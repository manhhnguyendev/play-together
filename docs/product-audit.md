# Audit & kế hoạch cải tạo sản phẩm Play Together

Tài liệu này là **Phase 1 (Product Audit) + Phase 2 (Redesign proposal)** của chương trình
nâng cấp toàn sản phẩm. Nó mô tả hiện trạng bằng chứng cụ thể (`file:dòng`), những gì **đã tốt
và không được đụng vào**, và kế hoạch cải tạo theo thứ tự ưu tiên.

Cách thực hiện: 4 luồng đọc code song song (engine đơn hàng + ví, schema + migration, mobile
journeys, admin + platform/security) trên toàn bộ 53.600 dòng TypeScript của ba app
(BE 14.314 · FE 28.934 · Admin 10.391 trong đó 6.875 là code generated).

Ký hiệu: **✓** = tôi đã tự mở file đọc lại và xác nhận bằng chứng. Không có **✓** = phát hiện
từ luồng audit, cần kiểm lại trước khi sửa.

---

## 1. Hiện trạng kiến trúc

| Tầng | Công nghệ | Cấu trúc |
| --- | --- | --- |
| API | NestJS + Drizzle ORM, Postgres (dev: PGlite) | 18 module: `admin auth behavior catalog charity chat feedback health jobs notifications orders ratings recommendations reports search-history system-config users wallet` |
| DB | 35 bảng, 3 migration, sinh bằng `drizzle-kit` | `src/db/schema/{users,orders,wallet,catalog,moderation,social,charity,config,auth,enums}.ts` |
| Mobile | Expo SDK 58, expo-router, RN 0.88 | 55 screen + 3 layout, design system riêng (token, floating input, dark mode, transition) |
| Admin | Vite + React + Ant Design | 12 page, client API sinh bằng Orval |

**Thứ tự tiền thật** (đã kiểm chứng đường đi):

```
deposit  wallet.controller.ts:81 → wallet.service.ts:278 → credit :354
hold     orders.service.ts:279   → wallet.service.ts:580  (escrowHolds :599)
release  wallet.service.ts:643   → refund :674 / payout :704 / fee :741
pending→available  wallet.service.ts:863
withdraw wallet.service.ts:406   → admin approve :496
```

## 2. Không được "sửa" những thứ sau

Toàn bộ phần này đang đúng và là nền tảng tốt nhất của hệ thống. Phá nó là lỗ thật:

- **Sổ cái double-entry** với cặp DEBIT/CREDIT cân bằng cho mỗi biến động, `CHECK (amount > 0)`
  (`wallet.ts:64`), money là `bigint` VND — **không có một cột float/numeric nào** ở toàn schema.
- **Escrow chống release hai lần**: `escrowHolds_order_key` (UNIQUE) + điều kiện `status !== 'HELD'`
  dưới `FOR UPDATE` (`wallet.ts:86`, `wallet.service.ts:658-661`).
- **IPN deposit replay-safe**: `deposits_provider_txn_key` UNIQUE (`wallet.ts:134`) + early-return
  khi đã `SUCCEEDED` (`wallet.service.ts:364`).
- **`splitFee`** đảm bảo `fee + payout = gross` (`money.ts:16-25`).
- **`feeBps` snapshot theo đơn** (`orders.service.ts:139`) — đổi fee không làm lệch đơn cũ.
- **Guard model mặc định-chặn**: Throttler → JwtAuthGuard → RolesGuard bind toàn cục
  (`app.module.ts:71-73`), muốn public phải `@Public()` rõ ràng.
- **Ownership lọc ngay trong WHERE** ở chat / notifications / search-history.
- **`UNIQUE(order_id, rater_id)`** chặn đánh giá trùng; **partial unique index** chặn trùng email
  (`lower(email)`) và một Google ID dùng chung.
- **Ràng buộc Google login**: chỉ liên kết tài khoản khi `email_verified` (`auth.service.ts:168-202`).
- **Schema và migration khớp nhau 100%**: đối chiếu từng một trong 106 tên cột + toàn bộ
  index/constraint/check — không phát hiện drift.
- **Shutdown hooks**, env validation lúc boot, helmet, `ValidationPipe` whitelist +
  forbidNonWhitelisted, error model RFC 7807 nhất quán.

## 3. P0 — Tiền và trạng thái đơn hàng (phải sửa trước tiên)

### 3.1 ✓ Trần hoàn tiền của report đặt sai chỗ (không phải lỗ hổng đang mở)

`reports.service.ts:254-268` lấy cap là `order.playerPayout` cho **mỗi lần** xử lý report:

```ts
const maxRefund = order.playerPayout ?? 0;
if (dto.refundAmount > maxRefund) throw …
```

**Đính chính so với bản nháp đầu:** tôi từng mô tả đây là "hai report hoàn 200% tiền, tiền miễn
phí". Đọc lại toàn bộ đường tạo report thì **bất biến đó hiện không vượt qua được**:
`reports.service.ts:78` cho mỗi reporter đúng một report trên một đơn, và `:258-263` chặn mọi
report không nhắm vào player — nên mỗi đơn chỉ tối đa **một** report có khả năng hoàn tiền,
`reports.service.ts:212` lại chặn xử lý report lần hai. Con số 200% là suy diễn từ một đoạn,
không phải từ toàn bộ luồng.

Vấn đề thật, và vẫn đáng sửa: **trần luỹ kế nằm ở module báo cáo, cách xa chỗ tiền dịch chuyển**.
`refundFromPlayerToHirer` nhận `amount` và trả lời "vâng" với mọi con số. Chỉ cần thêm một đường
gọi mới (admin hoàn tiền độc lập, đối soát khiếu nại, script migrate) là invariant vỡ ngay.

**Đã sửa**: trần luỹ kế giờ tính trong `wallet.service.ts` từ chính các dòng ledger
`REPORT_REFUND` của đơn đó, và khoá `escrow_holds` bằng `FOR UPDATE` trước khi đọc — hai lần xử lý
đồng thời không thể cùng đọc một tổng.

### 3.2 ✓ `pendingBalance` và `pendingCredits` là hai nguồn sự thật → tiền biến mất lặng lẽ

`wallet.service.ts:784-806` (claw-back khi hoàn tiền) trừ thẳng `wallets.pendingBalance`
nhưng **không đụng các dòng `pending_credits`**. Đến hạn, `releaseDuePendingCredits`
(`:889-905`) làm:

```ts
const amount = Math.min(locked.amount, wallet.pendingBalance);
if (amount > 0) { …cộng available… }
await tx.update(pendingCredits).set({ releasedAt: now })   // chạy vô điều kiện
```

→ credit bị đánh dấu "đã release" trong khi **0 đồng** được chuyển, không có dòng ledger nào.
Người chơi mất thu nhập và không có cách nào phát hiện.

**Sửa (khuyến nghị)**: bỏ hẳn cột `wallets.pendingBalance`, suy ra từ `SUM(pending_credits)`
chưa release. Một nguồn sự thật duy nhất, hết cửa lệch.

### 3.3 ✓ Không có đường thoát khỏi `ACCEPTED`, và auto-complete trả 100%

- `cancel` bắt buộc `PROCESSING` (`orders.service.ts:362-372`), thông báo lỗi đá sang
  `endEarly`.
- `endEarly` bị chặn cho tới khi 10% thời lượng trôi qua (`:440-449`) — đơn 12h là **72 phút**.
- Hết `scheduledEndAt`, `autoCompleteDue` (`:588-609`) tự settle với `earned: order.totalPrice`
  = **trả đủ 100% không cần bên nào xác nhận**.

Kịch bản lừa: nhận đơn → để yên → nhận đủ tiền. Người thuê không có cách nào thoát trong 72 phút.

**Sửa**: (a) cho huỷ trước `startsAt`; (b) auto-settle phải có cửa sổ "không khiếu nại"
(ví dụ 30 phút sau giờ kết thúc, hirer được quyền báo sự cố), hoặc settle theo thời gian
thực đo được chứ không theo `totalPrice`.

### 3.4 ✓ Rating 3 sao không cập nhật bộ đếm

`behavior.service.ts:30-35` khai `3: {}`, rồi `:100-104`:

```ts
if (effect.points === undefined && effect.satisfaction === undefined) return;
await this.adjust(…); await this.stats.recompute(userId, executor);
```

Dòng 5 sao → 1 sao đều `undefined` → **return sớm, bỏ qua `stats.recompute`**. Rating được ghi
vào bảng `ratings` nhưng `ratingSum/ratingCount` đứng nguyên cho tới khi một sự kiện khác vô
tình recompute. Xếp hạng và tìm kiếm theo rating vì thế sai silenciously.

**Sửa**: tách riêng `recompute` khỏi `adjust` — luôn chạy recompute khi có rating.

### 3.5 ✓ Một request chưa trả lời khoá toàn bộ thị trường 5 phút

`orders.ts:46-51` đặt partial unique index để `PROCESSING` **và** `ACCEPTED` cùng chiếm một slot
duy nhất của player. Hirer cũng không bao giờ giữ được 2 đơn. Nghĩa là một người chơi nhận được
1 request trong 5 phút sẽ **biến mất khỏi search** với mọi khách khác.

**Sửa**: tách "một phiên đang chạy" (`ACCEPTED`) khỏi "N lời mời đang mở" (`PROCESSING`).

### 3.6 ✓ Cấu hình phí không có trần

`config.dto.ts:24-28`: trường `value` chỉ có `@Min(0)`, **không có `@Max`**, trong khi
`orders.fee_bps` có `CHECK between 0 and 5000` (`orders.ts:57`). Admin gõ `10000` → mọi lệnh
tạo đơn từ đó đổ vỡ bằng CHECK failure thô.

**Sửa**: validate theo từng key (map key → min/max ở BE), hoặc thêm `min_value/max_value`
vào `system_configs` và sinh `CHECK` động.

### 3.7 ✓ Hết hạn request phạt ba lần, và client tự quyết định hết hạn

- `orders.service.ts:689-701`: −1 behaviour point, −2 satisfaction, **và** tắt `isPlayer`.
  Người chơi ngủ quên 5 phút mất luôn gian hàng.
- `play-together-fe/src/app/receive-request/[orderId].tsx:100-108` tự gọi `reject()` khi
  countdown về 0, còn `reject` ở BE (`:321-330`) **không kiểm tra `respondBy`** → server chấp
  nhận một cú reject đến muộn và phạt nhầm.

**Sửa**: giữ đúng một cơ chế phạt; `reject` phải assert `respondBy > now`; countdown chỉ hiển thị.

### 3.8 ✓ `rotate()` refresh token race

`common/security/token.service.ts:62-91`: SELECT → kiểm tra → UPDATE, không transaction, không
`FOR UPDATE`, **không đọc số dòng bị ảnh hưởng** của UPDATE (`:82-85`). Hai request refresh đồng
time với cùng một token đều mint pair mới.

**Đính chính:** hệ quả "đăng xuất mọi thiết bị" tôi nêu ban đầu cần một lần replay *sau* cú race,
không phải bản thân cú race. Cái race gây ra là **mint đúp** — hai cặp token hợp lệ từ một lần cấp.

**Đã sửa**: toàn bộ exchange nằm trong một transaction với `FOR UPDATE`.

**Đã thử và bỏ**: phương án "ân hạn 10 giây" cho rotate lại một token vừa rotate — về lý sẽ cứu
kịch bản hai tab cùng refresh. Hai test e2e có thật (`auth.e2e-spec` "revokes the family on reuse",
`marketplace.e2e-spec` "lock the account") bắt được rằng nó **phá luôn kiểm soát đánh cắp token**:
kẻ cắp dùng token bị lấy trong 10 giây đầu sẽ trót lọt. Codebase này đã chọn rõ "dùng lại là thu
hồi cả nhà" và có test cho điều đó, nên tính nghiêm ngặt được giữ nguyên; chỉ tính atomic là mới.
Race hai tab đúng nghĩa nên xử ở tầng client (chuỗi hoá refresh) — ghi ở S4.

### 3.9 Các lỗi nhỏ dọn trong cùng đợt

- ✓ `wallet.service.ts:302` **xoá cứng** dòng deposit khi gateway lỗi → mất dấu vết để webhook
  đối chiếu sau này. **Đã đổi** thành `status='FAILED'`; `creditDeposit:371` đã chặn FAILED từ
  trước nên không có đường nào nạp tiền vào một deposit đã hỏng.
- ✓ Quy tắc số giờ khai báo **4 nơi**: `orders.dto.ts:40-44`, `orders.ts:55`,
  `users.ts:36`/`orders.service.ts:113`, `users.ts:55`. Một nguồn ở BE, còn lại đọc từ config — dời
  sang S3 vì nó dính lịch hẹn.
- ✓ `money.ts:12,17,40-41` và `wallet.service.ts:655` `throw new Error` thô.
  **Đính chính**: tôi từng ghi đây là "lọt ra 500, phá vỡ mô hình RFC 7807" — sai.
  `ProblemDetailsFilter` là `@Catch()` toàn cục và đã biến mọi exception lạ thành `problem+json` đủ
  trường. Đây là invariant nội bộ (đối số do server tính, không phải input người dùng), nên 500 là
  đáp án đúng. **Không sửa**: thay bằng AppError chỉ là bôi son.
- ✓ `behavior.service.ts:114-139` read-modify-write không khoá → mất điểm khi có rating đồng thời.
  **Đã sửa** bằng upsert-then-`FOR UPDATE`, cùng hình mẫu `getOrCreateWallet`/`lockWallet`.
- ✓ `wallet.service.ts:372` trả `invalid_order_state` cho xung đột **deposit** → client đọc nhầm
  thành lỗi đơn hàng. **Đã sửa**, thêm mã `deposit_already_settled`.

## 4. P1 — Hình thái sản phẩm (mobile)

### 4.1 ✓ Không có push notification — người chơi không thể nhận lời mời

`grep expo-notifications play-together-fe/package.json` → **0 kết quả**. Màn `receive-request`
chỉ đến được qua `useOngoingOrderGuard`. Hệ quả thật sự: **một người chơi đang mở app ở nền
không bao giờ thấy lời mời**, và cửa sổ 5 phút tự hết hạn. Đây không phải lỗi UI, đây là tính
năng cốt lõi chưa tồn tại.

### 4.2 ✓ Guard điều hướng đặt sai chỗ, chạy 4 lần

`useOngoingOrderGuard()` được mount ở `home.tsx:29`, `personal.tsx:57`, `notifications.tsx:33`,
`charity.tsx:40`. Bottom-tabs mount tất cả một lúc → **16 query đơn hàng và 4 `router.push`
đua nhau** ngay khi mở app. Phải đặt một lần ở `(tabs)/_layout.tsx`.

### 4.3 ✓ Guard không chạy lại khi app trở lại foreground

Deps `[refreshUser, router, userId]` — tabs không unmount, và **không có `AppState` listener nào
trong toàn app**. Mở lại app sau khi bị nền ⇒ phải tắt hẳn và bật lại mới thấy đơn đang chạy.

### 4.4 IA hiện tại là IA một chiều, không có dashboard cho Player

Tabs: `home / history / charity / notifications / personal`. Charity chiếm nguyên một tab trong
khi toàn bộ nghiệp vụ Player bị rải ra `manage-hiring`, `statistics`, `behavior-history`,
`unactive-balance`, `transactions`, `withdraw*` và một segment "Được thuê" trong `history`.

- Hirer: home → booking xác nhận = **6 tap / 5 màn**; filter là một màn hình riêng rồi `router.back()`
  (`filter.tsx:102`) — một back-track không cần thiết.
- Player: từ home đến lúc thấy tiền về ví = **8 tap**.
- `hiring-negotiating/[orderId].tsx:107-137` **không thương lượng gì cả** — chỉ là màn chờ poll 5s.
- Player **không đánh giá được Hirer**: `rating/[orderId]` chỉ được mở khi `isHirer`
  (`end-order/[orderId].tsx:70`). Review một chiều thì không xây được niềm tin hai phía.
- `send-hiring-request.tsx:213` đá sang màn nạp tiền khi thiếu số dư, và `deposit.tsx:80`
  `dismissTo('/(tabs)/personal')` — **toàn bộ draft đặt đơn mất theo**.
- `deposit.tsx:67,77` mở MoMo rồi toast, **không poll lại số dư**; mock tự xác nhận thành công.
- Màn Player-only **không có route guard**: `manage-hiring`, `receive-request/[id]`, `withdraw`
  mở được bằng mọi role (`_layout.tsx:105-120`).

### 4.5 Gộp màn hình (giảm 55 screen xuống ~40 mà không mất chức năng)

| Gộp | Vì sao |
| --- | --- |
| `hiring-negotiating` + `receive-request` → 1 màn `order/[id]` theo role | hai mặt của cùng một trạng thái |
| `hiring-stage` + `end-order` + `end-order-early` vào `order/[id]` | một đơn = một màn, hành đổi theo status |
| `end-order` + `history/[orderId]` | cùng payload, chỉ khác nhánh role |
| `withdraw-amount` → `withdraw-method` → `withdraw` thành 1 màn | 3 màn + 2 lần chuyển params cho **một** mutation |
| `deposit-method` + `withdraw-method` | danh sách 3 dòng gần như trùng nhau, 2/3 dòng là "(Sắp ra mắt)" chết |
| `search-history` vào `search` | màn chắn không có giá trị khi history trống |
| `filter` thành bottom sheet | không phải route |
| `onboarding/take-categories` + `update-hobbies` + `favorite-games` | ba bộ chọn catalogue cùng kiểu |

`player-list.tsx:10` đọc singleton `playerListDraft` ở module scope, 0 API call → không deep-link
được và trắng xoá nếu mất state. Phải hoặc làm thật, hoặc bỏ.

### 4.6 Trạng thái lỗi/thiếu

Không có `ErrorView` ở: `home.tsx` (5 query, fail im lặng), `personal.tsx`, `deposit.tsx`,
`filter.tsx`, `user-profile.tsx`, và nặng nhất là **`withdraw-amount.tsx`** — query số dư lỗi thì
màn hình hiển thị "0 đ" như sự thật. `manage-hiring.tsx:203-212` toggle bật/tắt chế độ Player
không có `disabled`/pending → tap nhanh race `updateIsPlayer`. `notifications.tsx:80-84` đánh dấu
đã đọc nhưng **không điều hướng đi đâu** — mọi notification đơn hàng là ngõ cụt.
`useCountdown.ts:9` dùng `setInterval` trần, đóng băng khi app ở nền.

## 5. P2 — Admin: đang là "bảng xem", chưa phải nền tảng vận hành

### 5.1 ✓ Không có audit log

`grep -rn audit be/src` → **rỗng**. Ngoài `orderStatusHistory.actorId` và các cột `resolvedBy`
trên report/rating/charity-withdrawal, **không có dấu vết nào** cho: đổi role, khoá user, sửa
config, broadcast, bật bảo trì, đổi catalogue.

### 5.2 ✓ Một role ADMIN duy nhất, toàn năng

`enums.ts:3` → `['USER','ADMIN','CHARITY']`. Mọi admin đều: tự thăng cấp user bất kỳ lên ADMIN
(`admin.service.ts:177-185`), khoá admin khác. Kiểm tra "còn ≥1 admin active" (`:150-158`) chỉ chạy
khi đổi `role`, **không chạy khi set `isActive:false`** → có thể khoá sạch toàn bộ admin.

### 5.3 ✓ Duyệt rút tiền không ghi được ai duyệt

`admin.service.ts:329` `resolveWithdrawal(id, status, note)` — **không nhận `adminId`**;
`admin.controller.ts:136` cũng không truyền. `wallet.ts:158` chỉ có `processedAt`. Tiền rời nền
tảng mà không biết ai làm, vì sao.

### 5.4 Tài chính trong admin đang mù

Không có endpoint admin nào cho: số dư ví, dòng ledger, deposit, danh sách escrow, đơn hàng.
Dashboard chỉ có `escrowTotal` và `platformFees`. Tệ hơn: `pendingUserWithdrawals` là **`count(*)`**
(`admin.service.ts:82-86`) nhưng hiển thị dưới thẻ "Chi trả đang chờ" (`DashboardPage.tsx:89-99`)
— đếm số lệnh rồi trình bày như số tiền.

Thiếu hoàn toàn: GMV, take rate, tỉ lệ hoàn thành, tỉ lệ khiếu nại, tỉ lệ hoàn tiền, retention,
top player, khoảng ngày, time series, xuất CSV.

### 5.5 Thao tác

Không có `sorter` nào trong **toàn bộ app**; không `rowSelection` → không bulk action;
`usePagedQuery` giữ page/filter trong `useState` → **không URL nào deep-link/chia sẻ được**;
`ConfigsPage.tsx:206` và `CatalogPage.tsx:63` nạp tối đa 100 bản ghi vào memory. Xử lý một report
là đi săn 3 trang: report chỉ hiện `orderId` thô (`ReportsPage.tsx:158`), mức trần hoàn tiền
(`playerPayout`) không nhìn thấy, lịch sử đối tượng nằm ở `/users` khác. Nút khoá user
(`UsersPage.tsx:184`), bật bảo trì (`MaintenancePage.tsx:38`), broadcast
(`BroadcastPage.tsx:77`) **không có confirm**. `LoginPage.tsx:77` in thẳng credentials seed
`admin@playtogether.local / Admin@12345` lên giao diện.

## 6. P3 — Nền tảng, bảo mật, khả năng mở rộng

| # | Vấn đề | Bằng chứng | Mức |
| --- | --- | --- | --- |
| ✓ | `value` config không có trần → phá mọi lệnh tạo đơn | `config.dto.ts:24-28` | high |
| ✓ | OTP & reset token lưu SHA-256 **không salt** — không gian 10^6, dò offline được khi lộ DB | `auth.service.ts:288,362` | med |
| ✓ | Không throttle riêng cho `refresh / google / password-reset`; không có khoá theo **tài khoản** chống brute force | `auth.controller.ts`, `auth.service.ts:128-154` | med |
| ✓ | Account enumeration: 409 `emailTaken` trước khi chứng minh sở hữu | `auth.service.ts:67,263-264` | med |
| ✓ | User bị khoá vẫn dùng access token tới hết 15 phút; role đóng trong JWT nên đổi role không có hiệu lực ngay | `jwt-auth.guard.ts:37-44` | med |
| ✓ | Throttler in-memory, **không `trust proxy`** → sau load balancer mọi khách chung một bucket | `main.ts:21-24` | med |
| ✓ | PGlite là **mặc định** của `DATABASE_DRIVER`, kể cả image production | `env.ts:15`, `Dockerfile` | high |
| ✓ | Jobs dùng `setInterval` + cờ trong process, không advisory lock → nhiều instance xử lý đúp | `background-jobs.service.ts:31-36` | med |
| ✓ | Cache config 15 phút trong từng process, `invalidate()` chỉ cục bộ → maintenance/fee lệch giữa các instance | `system-config.service.ts:10,137` | med |
| ✓ | Search toàn bộ là `ILIKE '%q%'` không index được, 5 module, không escape `%`/`_` | `users.service.ts:259-261`, `catalog.service.ts:62` | med |
| ✓ | Chat: bất kỳ ai đã đăng nhập cũng nhắn cho bất kỳ ai (chỉ chặn block-list), không theo đơn, không kiểm duyệt, không throttle | `chat.service.ts:21-40,30` | med |
| ✓ | Index tìm player không dùng được: index `(is_player,status,rating_sum)` vs WHERE luôn thêm `is_active` và bỏ `status`, `ORDER BY ranking_points`, `rating_sum::numeric/rating_count` không index được | `users.ts:52` vs `users.service.ts:255,345,271` | med |
| ✓ | `withdrawals.provider_txn_id` **không unique** → callback trùng lặp trả tiền hai lần | `wallet.ts:154` | high |
| ✓ | Cascade không nhất quán: xoá user CASCADE ledger? Không — `ledger_entries.walletId` chỉ SET NULL → sổ cái sống không còn chủ tài khoản | `wallet.ts:51` vs `:122,149` | med |
| ✓ | `user_status='MAINTAIN'` được đọc (`auth.service.ts:469`) nhưng không nơi nào ghi; `MAINTENANCE_REFUND`, `ORDER_EARLY_END` không bao giờ được insert | `enums.ts:33,51` | low |
| ✓ | Test: 3 spec unit + 3 file e2e (29 test). **Không có spec nào** cho auth/token/jobs/search; không coverage threshold; CI không chạy đường postgres e2e | `.github/workflows/ci.yml` | med |
| ✓ | `/docs` Swagger mở công khai ở production; `traceId` tin `x-request-id` do client gửi lên | `main.ts:44`, `problem-details.filter.ts:41` | low |
| ✓ | `charities.balance` là cột tiền **duy nhất không đối soát được** — stats tự suy ra từ `donations` (`charity.service.ts:592`) | `charity.ts:17` | med |

Điểm tốt cần giữ: recommendations **không phải** danh sách dựng sẵn — là content-based scorer thật
(hobby + 10 search gần nhất → overlap kỹ năng, tôn trọng block-list, fallback theo ranking)
(`recommendations.service.ts:22-141`).

## 7. Business rules đề xuất (Phase 2)

### 7.1 Vòng đời đơn hàng mới

```
PROPOSED ──accept──> LIVE ──> SETTLED
    │                  │
    ├─ player reject ──> CLOSED(reason=rejected)
    ├─ hirer cancel ──> CLOSED(reason=cancelled_by_hirer)
    ├─ timeout ──────> CLOSED(reason=expired)
    └─ admin ────────> CLOSED(reason=interrupted)

LIVE ──end early──> SETTLED(earned = elapsed)
LIVE ──scheduled end──> OPEN_TO_DISPUTE ──24h không khiếu nại──> SETTLED(100%)
                                          └─ có khiếu nại ──> admin quyết định
```

- Bỏ `ENDED_EARLY_BY_HIRER/_BY_PLAYER` → `ENDED_EARLY`; ai làm đã có trong
  `order_status_history.actorId` (`orders.ts:85`).
- Gộp `REJECTED/CANCELLED/EXPIRED` → `CLOSED` + `closedBy` + `closedReason`. Ba status, ba
  nhánh phạt, cùng một kết quả — không đáng là ba trạng thái.
- **Hủy được trước `startsAt`**, hoàn 100%, không phạt.
- **Không auto-pay 100% khi không có ai xác nhận.**
- Một player có thể nhận **nhiều lời mời mở**, nhưng chỉ **một phiên LIVE**.

### 7.2 Tiền

- Bỏ `wallets.pendingBalance`; mọi con số pending suy từ `pending_credits`.
- Cộng dồn hoàn tiền phải cap theo `playerPayout − đã hoàn`.
- `withdrawals` + `deposits`: unique `(provider, provider_txn_id)`, append-only, thêm bảng
  `payment_events` (idempotency key) trước khi cắm cổng thanh toán thật.
- Mọi biến động số dư viết bằng atomic SQL (`SET x = x + n`) + `CHECK (x >= 0)` chặn, thay vì
  số học phía JS dựa vào `lockWallet`.
- Behaviour points: `SET points = LEAST(100, GREATEST(0, points + d)) RETURNING`.

### 7.3 Feature đang tồn tại nhưng không có tác dụng — cần quyết định

`availability_windows` + 3 màn (`update-hours`, `add-hour`, `update-hour`) + overlap checks
(`users.service.ts:477-540`) **không bao giờ được đơn hàng tham chiếu**, vì `startsAt` = lúc accept
(`orders.service.ts:285`). Nghĩa là người chơi khai giờ online mỗi ngày và chẳng để làm gì.
Hoặc bỏ, hoặc biến nó thành lịch hẹn thật (hirer chọn khung giờ, BE chặn trùng). Không có phương
án thứ ba hợp lý.

## 8. Lộ trình đề xuất

| Đợt | Nội dung | Rủi ro dữ liệu | Cần bạn duyệt |
| --- | --- | --- | --- |
| **S1 — Stop the bleeding** | 3.1 → 3.9, 5.3, 6 (`withdrawals` unique, `@Max` config), `rotate()` atomic, rating recompute | migration thêm cột, không mất dữ liệu | không |
| **S2 — Money model** | bỏ `pendingBalance`, ledger hoá claw-back, atomic balance writes, `payment_events` | **backfill + đối soát** | **có** |
| **S3 — Order lifecycle** | status mới + migration map status cũ, cancel trước startsAt, cửa sổ khiếu nại, nhiều request/một phiên | **viết lại `orders.status`** | **có** |
| **S4 — Mobile IA** | tabs mới + dashboard Player, gộp 15 screen, push notification, role guard, trạng thái lỗi | không | một phần (push) |
| **S5 — Admin ops** | audit log, RBAC 3 mức, trang Orders + Money, detail route, sorter/bulk/deep-link, KPI thật | không | không |
| **S6 — Platform** | postgres mặc định, throttler ra redis + trust proxy, job lock, pg_trgm, chat scoping, test auth/token/jobs | không | không |

S1 và S5 không đụng mô hình kinh doanh → làm được ngay. S2/S3 thay đổi tiền và trạng thái đơn trên
dữ liệu đã tồn tại → cần bạn chốt trước.

## 9. Quyết định đã chốt (2026-10-31)

| Câu hỏi | Quyết định |
| --- | --- |
| Hai nguồn sự thật cho tiền pending | **Bỏ cột `wallets.pending_balance`**, suy từ `pending_credits` (S2) |
| Vòng đời đơn hàng | **Tái cấu trúc đầy đủ** + cửa sổ khiếu nại (S3) |
| Tính năng "giờ online" | **Làm thành lịch hẹn thật** — hirer chọn khung giờ, BE chặn trùng (S3b) |
| Push notification | **Làm foreground guard trước** — `AppState` + re-sync, chưa đổi cấu trúc build (S4) |

## 10. S1 — đã thực hiện

Phạm vi: chỉ các defect đã kiểm chứng, không đổi mô hình kinh doanh.

| # | Thay đổi | File |
| --- | --- | --- |
| 1 | Trần hoàn tiền luỹ kế tính từ ledger, khoá `escrow_holds` khi đọc | `wallet/wallet.service.ts` |
| 2 | Rating 3 sao luôn đi qua `stats.recompute` | `behavior/behavior.service.ts` |
| 3 | Behaviour points: upsert-then-`FOR UPDATE` thay cho read-modify-write | `behavior/behavior.service.ts` |
| 4 | `CONFIG_DEFAULTS` thêm `min`/`max` theo key; chặn ở create + update | `db/schema/config.ts`, `system-config.service.ts` |
| 5 | `rotate()` chạy trong một transaction với `FOR UPDATE` | `common/security/token.service.ts` |
| 6 | Deposit gateway lỗi → `status='FAILED'`, không xoá dòng | `wallet/wallet.service.ts` |
| 7 | `reject` sau hạn bị phân loại lại thành **expiry**, server là quyền quyết | `orders/orders.service.ts` |
| 8 | `withdrawals` unique `(provider, provider_txn_id)` + cột `processed_by` | `db/schema/wallet.ts`, migration `0003_windy_shaman.sql` |
| 9 | Duyệt rút tiền ghi nhận **admin nào** | `admin.controller.ts`, `admin.service.ts`, `wallet.service.ts` |
| 10 | Mã lỗi riêng `deposit_already_settled` cho xung đột deposit | `error-codes.ts`, `wallet.service.ts` |

### Kiểm chứng

Gate ở `play-together-be`: `tsc --noEmit` 0 · `eslint .` 0 · unit **25/25** · e2e **32/32**
(trước đó 29; thêm 3).

Test hồi quy mới `test/money-guards.e2e-spec.ts`, kèm **positive control** — mỗi test được chạy lại
với code cũ để chắc chắn nó đỏ:

| Test | Code cũ | Code mới |
| --- | --- | --- |
| Rating 3 sao cập nhật bộ đếm | `ratingCount: 0, rating: 0` | `ratingCount: 1, rating: 3` |
| Chặn `platform_fee_bps = 10000` | 200, lưu được | 422 `validation_failed` |
| Reject sau hạn = expiry | 200, `status: REJECTED` | 200, `status: EXPIRED` |

Chạy thật trên dev API (`AUTO_MIGRATE=true`, migration 0003 áp thành công):

```
PATCH /admin/configs/:fee  {value: 10000} -> 422
  detail: "\"order.platform_fee_bps\" must be between 0 and 3000"
  details: { key, value: 10000, min: 0, max: 3000 }
PATCH {value: 2500} -> 200   (đã restore về 1000)
```

### Việc rút ra trong lúc làm

- Cửa sổ ân hạn 10 giây cho rotate refresh **đã viết, đã chạy, và bị hai test e2e hiện có bác bỏ**
  vì nó nới lỏng kiểm soát đánh cắp token. Đã bỏ. Bài học: hợp đồng bảo mật có test là hợp đồng có
  chủ đích.
- Hai phát hiện (3.1 "hoàn 200%", 3.9 "raw Error phá RFC 7807") **tôi đã nói quá** và đã đính chính
  ngay trong tài liệu. 3.1 vẫn sửa vì invariant phải nằm cạnh chỗ tiền dịch chuyển, nhưng nó không
  phải một lỗ hổng đang mở.
- `useOngoingOrderGuard` nên chuỗi hoá refresh ở client — ghi ở S4.

## 11. S2 — đã thực hiện: một nguồn sự thật cho tiền pending

`wallets.pending_balance` **không còn tồn tại**. Pending giờ là tập hợp dòng
`pending_credits` chưa release, không phải một con số có thể lệch khỏi tập đó.

| # | Thay đổi | File |
| --- | --- | --- |
| 1 | Bỏ cột `pending_balance` + CHECK `wallets_pending_check` | `db/schema/wallet.ts`, migration `0004_milky_weapon_omega.sql` |
| 2 | `pendingTotal()` — `SUM(amount) WHERE released_at IS NULL`, là định nghĩa duy nhất của "pending" | `wallet.service.ts` |
| 3 | `getSummary` trả `pendingBalance` **suy ra**, nên hợp đồng API với mobile/admin không đổi | `wallet.service.ts` |
| 4 | Claw-back của report giờ **tiêu thụ chính các dòng credit** (xoá hoặc giảm `amount`), không trừ số nữa | `wallet.service.ts` |
| 5 | `releaseDuePendingCredits` đi theo thứ tự khoá ví→credits, cộng một lần cho cả ví | `wallet.service.ts` |
| 6 | Mọi biến động `available_balance` viết bằng SQL (`x + n` / `x - n WHERE x >= n`), không còn số học JS | `addToAvailable` / `tryDebitAvailable` / `debitLocked` |

### Vì sao thứ tự khoá đổi

Bản cũ khoá **credit rồi mới ví**; đường claw-back khoá **ví rồi mới credit**. Hai đường cùng chạm
một tập dòng với thứ tự ngược nhau là công thức deadlock. Giờ mọi nơi đều ví→credit, và mỗi ví chỉ
bị ghi một lần thay vì một lần cho mỗi credit.

### Kiểm chứng

Trước khi bỏ cột, chạy đối soát trên dev DB: **10 ví, 0 lệch**, tổng pending hai phía đều 0 →
không phải backfill.

Gate: `tsc --noEmit` 0 · `eslint .` 0 · unit **25/25** · e2e **32/32**.

Bất biến mới cài trong `marketplace.e2e-spec` (cùng một câu truy vấn, chạy trước và sau cú
claw-back):

```
trước report : pendingBalance 54_000, pendingCredits 1 dòng
sau report   : credit mở còn lại 0 dòng, availableBalance không đổi
```

Chạy thật sau khi migration 0004 áp lên dev:

```
GET /wallet (demo.player1) -> 200
{ availableBalance: 244000, pendingBalance: 0, pendingCredits: [] }
```

`pendingBalance` vẫn là số **tính từ credits**, không còn là cột. Phía mobile
(`api/v2/mappers.ts:166` cộng `availableBalance + pendingBalance`) và admin generated client không
phải sửa gì — hợp đồng response giữ nguyên.

### Ghi nhận cho đợt sau

Hoàn tiền của một withdrawal bị từ chối đang ghi ledger `type='ADJUSTMENT'` thay vì
`'WITHDRAWAL'`, nên nó hiện lẫn vào nhóm điều chỉnh trong lịch sử giao dịch. Không đổi ở S2 vì sẽ
làm đổi bộ lọc `?type=` mà mobile đang dùng — chuyển sang S6.

## 12. S3a — đã thực hiện: vòng đời đơn hàng

9 status thành 7. Lý do chính: `REJECTED` / `CANCELLED` / `EXPIRED` / `INTERRUPTED` là **một đơn
đóng**, chỉ khác nhau ở vì sao — bốn status, bốn nhánh xử lý, cùng một kết quả. `ENDED_EARLY_BY_HIRER`
và `_BY_PLAYER` cũng vậy: ai bấm đã có trong `order_status_history.actorId`.

```
trước : PROCESSING ACCEPTED COMPLETED CANCELLED REJECTED EXPIRED
        ENDED_EARLY_BY_HIRER ENDED_EARLY_BY_PLAYER INTERRUPTED
sau   : PROPOSED  SCHEDULED*  LIVE  DISPUTED*  SETTLED  ENDED_EARLY  CLOSED
        (* thêm ở S3b/S3c, khai báo trước vì Postgres cấm ADD VALUE trong transaction)

lý do giờ là dữ liệu, không phải trạng thái:
  orders.close_reason   = REJECTED_BY_PLAYER | CANCELLED_BY_HIRER | EXPIRED | INTERRUPTED_BY_ADMIN
  orders.ended_early_by = HIRER | PLAYER
  CHECK: status='CLOSED' ⇒ close_reason not null
  CHECK: status='ENDED_EARLY' ⇒ ended_early_by not null
```

**Một request để ngỏ không còn chiếm chỗ ai.** Index unique cũ chặn
`PROCESSING`+`ACCEPTED` mỗi bên một suất, nghĩa là một lời mời chưa trả lời làm người chơi **biến
mất khỏi toàn bộ thị trường hết cửa sổ 5 phút**. Index mới chỉ chặn `SCHEDULED`/`LIVE`/`DISPUTED`:
nhiều lời mời mở, một phiên đang chạy.

**Hết phạt ba lần một lỗi.** Hết hạn cửa sổ trả lời trước đây vừa trừ 1 điểm hành vi, vừa trừ 2
điểm hài lòng, **vừa tắt `isPlayer`** — ngủ quên 5 phút mất luôn gian hàng — và còn ép
`status='ONLINE'` dù người chơi đang chủ động để Offline. Giờ chỉ còn một đòn phạt.

| # | Thay đổi | File |
| --- | --- | --- |
| 1 | Enum 7 giá trị + `order_close_reason` + `order_party` | `db/schema/enums.ts` |
| 2 | `close_reason`, `ended_early_by`, index session mới, 2 CHECK | `db/schema/orders.ts` |
| 3 | `ORDER_STATUS_GROUPS` — tập trạng thái dùng chung, một nguồn | `db/schema/orders.ts` |
| 4 | Toàn bộ chuyển đổi đổi tên + ghi lý do | `orders/orders.service.ts` |
| 5 | Rating/report/stats đọc từ `FINISHED_ORDER_STATUSES` | `ratings`, `reports`, `users/stats` |
| 6 | `byStatus.PROCESSING + ACCEPTED` → reduce theo nhóm | `admin/admin.service.ts` |
| 7 | Tắt player mode chỉ còn bị chặn bởi phiên thật | `users/users.service.ts` |
| 8 | `OrderView` hết bị định nghĩa hai lần | `orders.dto.ts` + `orders.service.ts` |
| 9 | Mobile: adapter hấp thụ phép gộp, 55 screen không phải đổi | `api/v2/mappers.ts`, `types.ts`, `features/orders/status.ts` |
| 10 | Admin: `StatusTag` từ vựng mới, giữ `PENDING/REJECTED/CANCELLED` cho withdrawal/report | `components/StatusTag.tsx` |
| 11 | Migration viết lại tay + rewrite dữ liệu | `db/migrations/0005_flawless_hairball.sql` |

### Migration: bản drizzle sinh ra **không chạy được**

Bản tự động (i) cast `orders.status` sang enum mới **trước khi** đổi giá trị, nên `'PROCESSING'`
không tồn tại trong type mới → chết; (ii) thêm CHECK `CLOSED ⇒ close_reason not null` lên dữ liệu
chưa có cột; (iii) `DROP TYPE` ở dòng 7 trong khi hai partial index còn tham chiếu type đó phải tới
dòng 13 mới bị drop. Đã viết lại theo thứ tự: drop index → mở rộng `text` → **drop default** →
đổi type → thêm cột → map dữ liệu → mới cast → rồi mới ràng buộc.

Diễn tập trên **bản copy của dev DB** bắt được thêm hai lỗi nữa trước khi động vào DB thật:
`CASE` trên cột text bị suy ra text nên không gán vào cột enum (cần cast tường minh), và default
cũ `'PROCESSING'::order_status` không tự cast sang type mới.

### Kiểm chứng

Đối soát trước khi migrate: 2 đơn (`COMPLETED` 1, `EXPIRED` 1), 5 dòng history. Sau migrate trên
**cả bản diễn tập lẫn DB thật**: `SETTLED` 1, `CLOSED` 1 với `close_reason='EXPIRED'`, history map
đủ 5 dòng, **tổng số dòng không đổi** — không mất dữ liệu.

Gate: root `typecheck` 0 (cả 3 app) · root `lint` 0 · unit **25/25** · e2e **32/32**.
Sáu e2e đỏ ngay sau khi đổi tên; đọc từng cái để xác nhận đó là **assertion lỗi thời**, không phải
code sai, rồi mới sửa test.

Smoke qua HTTP thật trên dev (`GET /orders`, `POST /orders`, `POST /orders/:id/cancel`,
`GET /admin/stats`):

```
existing orders        : SETTLED
POST /orders           -> 201 PROPOSED
POST /cancel           -> 200 CLOSED closeReason: CANCELLED_BY_HIRER
admin orders           : {total:3, active:0, completed:1, byStatus:{SETTLED:1, CLOSED:2}}
re-booking same player -> 201 PROPOSED
```

Lưu ý: smoke để lại 2 đơn `CLOSED` trong dev DB.

### Chưa kiểm chứng bằng cách nhìn

Các màn đơn hàng của mobile chỉ được xác minh ở mức type và qua adapter, chưa render thật với một
phiên đã đăng nhập (lịch sử, chi tiết đơn, thẻ trạng thái). Cần mở `http://localhost:8081` đăng
nhập và xem mục Lịch sử.

## 13. S3c — đã thực hiện: bỏ auto-pay 100%, thêm cửa sổ đối soát

Mô hình cũ: đúng khoảnh khắc `scheduled_end_at` đi qua, job `autoCompleteDue()` settle toàn bộ
`totalPrice` cho người chơi — người thuê đã trả tiền nhưng **không hề được hỏi** phiên đó có diễn ra
không. Mobile còn tự bấm hộ: `useOngoingOrderGuard` thấy đơn `Starting` quá giờ là
`POST /complete` (`useOngoingOrderGuard.ts:25`).

Mô hình mới, khớp với 7.1:

```
LIVE ──hết giờ──> DISPUTED ──xác nhận của một bên──────> SETTLED(100%)
                    │        ──hết cửa sổ, không khiếu nại─> SETTLED(100%)
                    └── report còn PENDING ─────────────> đứng lại, admin quyết định
```

**Lời phản đối chính là report đã tồn tại**, không dựng thêm hệ thống khiếu nại song song. Vì vậy
`REPORTABLE_STATUSES` và `RATEABLE_STATUSES` chuyển sang nhóm `feedback` (DISPUTED + SETTLED +
ENDED_EARLY): hết phiên là rate/report được ngay, không phải chờ tiền chạy.

### Ba quyết định thiết kế

1. **Không dịch chuyển tiền khi vào DISPUTED.** Job chỉ đổi status, ghi `endedAt`/`disputedAt`, trả
   player về `ONLINE` và thông báo cho cả hai. `escrow_holds` vẫn `HELD`, `playerPayout` vẫn `null`,
   ledger không có dòng nào.
2. **DISPUTED không còn chiếm lịch của hai bên.** Nếu không, mỗi phiên vừa kết thúc sẽ giam người
   chơi khỏi marketplace thêm 24 giờ — đây là loại hỏng sản phẩm mà migration an toàn cũng không cứu
   được. Tách thành hai khái niệm trong `ORDER_STATUS_GROUPS`: `session` (SCHEDULED, LIVE — đang giữ
   người) và `escrowed` (SCHEDULED, LIVE, DISPUTED — đang giữ tiền). `interruptActiveForUser` dùng
   nhóm escrowed nên admin vẫn ngắt được đơn đang đối soát và hoàn đủ tiền cho hirer.
3. **Điều kiện chặn nằm trong job, không nằm trong status.** `settleDueDisputes()` khoá đơn, rồi mới
   hỏi "có report PENDING nào trên đơn này không" ngay trong transaction. Một report gửi lúc
   23 giờ 59 phút vẫn chặn được lần tick kế tiếp — không có race giữa cửa sổ và đơn khiếu nại.

Cửa sổ là config `order.dispute_window_hours` (mặc định 24, min 0 – max 168). `0` nghĩa là nhả tiền
ngay ở tick kế tiếp, tức lối thoát khi vận hành muốn tắt tính năng mà không deploy lại.

### Migration 0006

`ALTER TABLE orders ADD COLUMN disputed_at timestamptz` + `orders_disputed_at_check` + **drop rồi
tạo lại hai partial unique index** vì predicate đổi từ `('SCHEDULED','LIVE','DISPUTED')` xuống
`('SCHEDULED','LIVE')`. Diễn tập trên bản copy dev DB trước khi chạy thật: sau migrate vẫn
`SETTLED` 1, `CLOSED` 3, `nullDisputedAt` 0 nên CHECK mới không hề va dữ liệu cũ.

Một lỗi chỉ lộ ra khi nhìn SQL sinh ra: helper dựng predicate từ `ORDER_STATUS_GROUPS` bằng
`sql.join` cho ra `in ($1, $2)` trong câu DDL — index sẽ hỏng. Phải `sql.raw` để ra literal; nguồn
giá trị vẫn là group đã được `satisfies` ràng buộc với enum, nên không quay lại thời literal trôi
nở. Nhân tiện, index giờ **suy từ group** thay vì viết tay, vì S3c chứng minh hai bản sao là hai
lỗi.

### Kiểm chứng

Gate sau đợt: BE `typecheck` 0 · `lint` 0 · unit **25/25** · e2e **37/37** (thêm 5 test mới trong
`test/dispute-window.e2e-spec.ts`):

- hết giờ → `DISPUTED`, `playerPayout` null, `escrowStatus` HELD, **0 dòng `ORDER_EARNING`, 0 dòng
  `pending_credits`**, player về ONLINE;
- đơn đang đối soát **không** chặn việc tạo đơn mới;
- một bên bấm complete → `SETTLED` ngay, có đúng 1 earning + 1 pending credit;
- report PENDING chặn settle dù đã quá 25 giờ; admin xử lý xong thì tick kế tiếp nhả tiền, escrow
  `SETTLED`;
- `admin/stats`: `active` = LIVE+SCHEDULED, `completed` = SETTLED+ENDED_EARLY — DISPUTED không bị
  đếm hai lần và không bị bỏ quên.

`money-guards` và `marketplace` vẫn xanh nguyên vẹn, nên S3c không làm hỏng hai cổng tiền đã vá.

FE: `typecheck` 0 · `expo lint` 0 · admin `tsc -b` 0 · `oxlint` 0. Orval chạy lại cho cả hai app,
`openapi.json` có `disputedAt`/`disputeDeadlineAt`. API trên `:3000` đã được khởi động lại bằng bản
build mới (tiến trình cũ chạy code trước S3c và `docs-json` không có field mới).

### Mobile

`/end-order/[orderId]` bây giờ là màn đối soát: card giải thích theo vai trò (người thuê: "tiền vẫn
đang được giữ"), countdown tới `disputeDeadlineAt`, và hai nút **Có vấn đề** (→ `/report/[orderId]`)
/ **Xác nhận hoàn thành**. `useOngoingOrderGuard` không còn tự settle hộ ai nữa — hết giờ thì đưa
người dùng vào màn đơn. Màn này poll 10 s vì scheduler tự chuyển LIVE → DISPUTED ngoài đồng hồ của
người dùng.

### GAP ghi nhận

- **Admin chưa thể quyết định một phần khi đơn còn DISPUTED.** `refundFromPlayerToHirer` tính theo
  `playerPayout`, mà DISPUTED thì payout còn null, nên biên bản xử lý report chỉ có "buông" hoặc
  "ngắt toàn phần (hoàn 100%)". Direction đúng là cho admin settle với `earned` do admin chọn — nằm ở
  S5 (trang Orders detail), không phải hotfix.
- **DISPUTED gộp hai ý nghĩa**: "chờ hai bên xác nhận" và "đang có khiếu nại". FE hiển thị
  "Chờ đối soát" cho cả hai. Tách được thì cần thêm cột hoặc enum value, mà `ALTER TYPE ADD VALUE`
  không chạy trong transaction — để nguyên, ghi ở comment của enum.
- **Guard chưa "resume" đơn DISPUTED khi mở app.** `resumeStarting` chỉ quét đơn `Starting`; ai đóng
  app suốt cửa sổ sẽ chỉ thấy qua notification và tab Lịch sử. S4 viết lại guard (một chỗ, có
  `AppState`) nên không muốn nhân quadruple query lên thành sáu.
- **Chưa render thật.** Toàn bộ phần mobile trên chỉ xác minh ở mức type + adapter. Cần một phiên
  đăng nhập trên `http://localhost:8081` hoặc device để xem card đối soát, countdown và hai nút.
- `referenceUrl` của notification vẫn là `/orders/${id}` — không phải route nào của app. Vấn đề có
  từ trước S3c, chuyển vào S4 cùng lúc làm deep-link.

## 14. S3b — đã thực hiện: "giờ online" thành lịch hẹn thật

`availability_windows` từng được ba màn mobile đọc và cho sửa, nhưng **không có đơn hàng nào tham
chiếu**: `startsAt` luôn bằng lúc player bấm accept. Người chơi khai giờ mỗi ngày để… không làm gì.
Giờ thì giờ khai quyết định đặt được hay không.

```
PROPOSED ──accept──> SCHEDULED ──đúng giờ──> LIVE ──hết giờ──> DISPUTED ──> SETTLED
   │                    │
   │                    └──hirer cancel──> CLOSED (hoàn 100%, escrow REFUNDED)
   └── player reject / timeout ──> CLOSED
```

- `POST /orders` nhận thêm `startsAt` (ISO, optional). **Không gửi = thuê ngay**, giữ nguyên hành vi
  cũ nên mọi luồng đang chạy không đổi.
- Slot phải nằm trọn trong các window đã công bố (`isSlotCovered`, đi từng ngày theo giờ địa phương
  vì window không được vắt qua nửa đêm), không được quá khứ, không quá 14 ngày.
- `startsAt` được lưu ngay từ PROPOSED để người chơi **thấy mình đang nhận cái gì** (màn
  receive-request trước đó chỉ hiện "Thời lượng").
- Accept → `SCHEDULED` (escrow vẫn giữ, player **không** bị BUSY). Job `startDueSessions()` lật sang
  `LIVE` đúng giờ hẹn và chỉ lúc đó player mới BUSY.
- Hủy trước giờ bắt đầu: `cancel()` nhận thêm `SCHEDULED`, hoàn escrow 100%, không phạt ai, và **lưu
  lý do** vào `orders.reason`.

### Trói buộc lịch bằng advisory lock, không bằng unique index

"Một party chỉ có một phiên" là ràng buộc đúng khi chưa có lịch hẹn, nhưng sai khi có: một player
chính đáng nên giữ được vài phiên tương lai. Vấn đề là **trùng lịch là câu hỏi about range, không
phải about equality** — unique index không trả lời được.

Nên: `orders_one_session_per_*` → `orders_one_live_per_*` (chỉ `LIVE`, vẫn là sàn cứng để không ai có
hai phiên chạy cùng lúc), còn trùng slot do `assertSlotFree()` quyết trong transaction, dưới
`pg_advisory_xact_lock(hashtext(userId))` lấy theo hai id đã sort. Lý do không dùng row lock trên
`users`: create sẽ theo thứ tự users→orders trong khi accept theo orders→users — đúng công thức
ABBA deadlock. Advisory lock không có hướng, và cả hai đường đều xin nó trước.

Cả hai nhánh accept (ngay và hẹn) đều check trùng: nếu chỉ check nhánh hẹn, một player đang có lịch
lúc 20:00 vẫn có thể bị hire "ngay bây giờ" lúc 19:59 và đến 20:00 thì không phiên nào chạy được.

### Kiểm chứng

`test/booking.e2e-spec.ts` — 8 test, mỗi test lấy một slot khác nhau trong danh sách slot do chính
window của player seed sinh ra (không hard-code giờ, không phụ thuộc múi giờ máy chạy test):

- slot được echo lại khi PROPOSED, escrow `NONE`; accept → `SCHEDULED`, escrow `HELD`,
  `settledAmount = 0`, player **không** BUSY;
- tick trước giờ → `started: 0`; tick sau giờ → `started: 1`, `LIVE`, player BUSY;
- ngoài window → 422 `slot_outside_availability`; giờ đã trôi qua → 409 `slot_conflict`;
- **hai request cùng một slot trống đều 201**, ai accept trước thắng, cái sau 409 và vẫn PROPOSED —
  đúng nguyên tắc "proposal không giữ chỗ" của S3a;
- hủy booking → `CLOSED`/`CANCELLED_BY_HIRER`, escrow `REFUNDED`, lý do được lưu, và **slot đó đặt lại
  được ngay**;
- player đang BUSY vẫn nhận được booking tương lai, nhưng hire ngay thì 400 `player_unavailable`.

Gate: BE typecheck 0 · lint 0 · unit **25/25** · e2e **45/45** (6 suite) · FE typecheck 0 · expo lint
0 (không còn warning nào) · admin `tsc -b` 0 · `oxlint` 0. Migration 0007 diễn tập trên bản copy dev
DB trước, rồi xác minh trên DB thật sau khi server chạy: `orders_one_live_per_{hirer,player}`,
`orders_{hirer,player}_starts_idx`, `orders_slot_check` đều có mặt.

`openapi.json` + orval của FE và admin đã generate lại.

### Mobile

Màn **Thuê người chơi** thêm khối "Thời gian bắt đầu": chip `Ngay bây giờ` / `Chọn giờ hẹn`, dãy chip
ngày (Hôm nay / Ngày mai / T3 5/11…) và `SelectField` giờ. Danh sách giờ sinh từ window của player
(`buildBookableDays`), chỉ những giờ mà **cả phiên** nằm trong window; player chưa công bố giờ thì chỉ
còn lựa chọn "Ngay bây giờ" kèm dòng giải thích. `send-hiring-request` cũng hiện "Bắt đầu lúc".

Màn **Chi tiết thuê** hiển thị trạng thái `SCHEDULED` bằng card "Lịch hẹn đã được giữ" và nút **Hủy
lịch hẹn** (chỉ hirer, có dialog lý do) / **Nhắn tin**; người chơi chỉ thấy Nhắn tin.

### GAP ghi nhận

- **Player không tự hủy được một booking đã nhận.** `order_close_reason` chỉ có 4 giá trị đã khai
  trước; thêm `CANCELLED_BY_PLAYER` cần `ALTER TYPE ADD VALUE` chạy ngoài transaction, tức migration
  không do drizzle sinh. Không làm ẩu trong một đợt đã có hai migration; hướng xử lý là cho admin
  quyết (S5) và player dùng report. Ghi ở đây để không ai tưởng đây là oversight.
- **Guard chưa "resume" đơn SCHEDULED.** Mở app vẫn chỉ quét `Starting`/`Processing`, nên một lịch
  hẹn chưa tới không tự hiện ra; người dùng vào qua tab Lịch sử hoặc notification. S4 viết lại guard.
- **Slot dựng theo múi giờ máy**, còn BE kiểm theo `APP_TIMEZONE` (Asia/Ho_Chi_Minh). Với người dùng
  hiện tại hai cái là một; nếu không, BE trả 422 rõ ràng chứ không đặt sai. Múi giờ của player chưa
  phải là dữ liệu có trong DB — nếu app mở ra thị trường khác thì đây là việc phải làm, không phải
  lỗi hiển thị.
- **Chưa render thật.** Slot picker, card lịch hẹn và nút hủy mới chỉ đi tới mức type + test HTTP.
- Booking giữ escrow từ lúc accept tới lúc hủy/hết phiên, nên một hirer có nhiều booking sẽ bị giữ
  nhiều lần tiền. Đúng về mặt an toàn tiền, nhưng cần hiển thị rõ "đang giữ" bao nhiêu trong màn ví
  (S4/S5).

## 15. S4 (phần 1) — đã thực hiện: foreground sync + guard một chỗ

### Guard chuyển thành một mountpoint

`useOngoingOrderGuard()` được gọi trong **bốn** tab. Mỗi lần đổi tab, effect chạy lại: một lần
`refreshUser()` (network) + **4 truy vấn đơn hàng**, tổng cộng 16 truy vấn cho một lượt điều hướng,
và bốn push có thể đua nhau. Giờ guard chỉ nằm ở `src/app/(tabs)/_layout.tsx` — verification:
`grep -rln useOngoingOrderGuard src/` còn đúng hai file (layout + chính hook).

Thêm trong hook:
- `AppState` listener: quay lại app là lúc phiên có thể đã bắt đầu/kết thúc/bên kia accept, nên guard
  chạy lại. `running` flag chặn hai lần chạy chồng nhau.
- `pushOnce(route, orderId)`: mỗi order chỉ được push một lần trong một lượt, vì tab layout không
  unmount khi người dùng thoát màn hình — không có chốt này thì mỗi lần foreground lại chồng thêm một
  màn y hệt.
- Không tự settle hộ nữa (đã ghi ở S3c), và `resumeStarting` không còn `async` vì không gọi API.

### Foreground = focus của React Query

`refetchOnWindowFocus` đang bị tắt, và trên điện thoại thì "window focus" không tồn tại — nên app
quay lại từ background là **không refetch gì cả**. Số dư do gateway xác nhận, đơn đối tác vừa accept,
phiên scheduler vừa lật sang LIVE tất cả đứng hình cho tới khi user pull-to-refresh (đa số không biết
thao tác đó). Đây là cái "foreground guard" đã chốt ở §9, chưa từng có push notification:

- `AppProviders` mount thêm `ForegroundSync`: `focusManager.setFocused(status === 'active')` từ
  `AppState`, áp cả trạng thái hiện tại lúc boot, bỏ qua web (web có visibilitychange sẵn).
- `refetchOnWindowFocus: true` trở lại mặc định.
- Riêng query số dư ở tab Cá nhân đặt `staleTime: 0`: tiền thì không được phục vụ từ cache 30 giây.

Không thêm dependency nào (`@react-native-community/netinfo` không có trong project nên
`onlineManager` chưa nối — ghi thành GAP).

### Kiểm chứng

FE `typecheck` 0 · `expo lint` 0 warning. Đường đi của guard và sync chỉ xác minh được bằng tương tác
thật (xem GAP).

### GAP của đợt này

- **Chưa test trên máy.** Foreground/background, quay lại từ MoMo, chuyển tab 5 lần — toàn bộ phải
  chạy trên device hoặc ít nhất là Metro với một phiên đăng nhập. Đây cũng là lần đầu tiên các thay đổi
  S3b/S3c ở mobile được nhìn thấy thật.
- **`onlineManager` chưa nối** vì cần thêm package. Khi nào có dependency thì thêm 6 dòng.
- **SCHEDULED không được guard resume.** Lịch hẹn chưa tới thì không có gì để "vào lại"; nó hiện qua
  tab Lịch sử và notification. Sau khi scheduler lật sang LIVE thì guard bắt được như thường.
- **Item "giữ bản nháp đặt đơn qua màn nạp tiền": kiểm tra thấy đã ổn sẵn** —
  `router.push('/deposit-method')` giữ màn `send-hiring-request` mounted bên dưới nên state còn nguyên;
  không cần lưu storage. Ghi lại để lần sau không sửa thứ không hỏng.
- **Role guard trên route của player: chủ đích không làm.** `manage-hiring` chính là màn bật/tắt
  player mode, `update-hours`/`add-hour` là nơi khai giờ để *trở thành* player. Chặn theo
  `isPlayer` sẽ nhốt người dùng ngoài cửa đúng cái phòng họ cần vào. BE đã chặn ở tầng API.
- **Phần IA của S4 (dashboard cho player, gộp ~15 màn trùng nhau, deep-link notification) chưa làm**
  vì nó đổi cấu trúc điều hướng hàng loạt và cần xem thật trên điện thoại trước khi quyết — chuyển
  thành S4b.

## 16. S4 (phần 2) — đã thực hiện: tiếng Việt thật cho user-facing copy

Bốn lỗi user báo khi test máy (2026-10-31) đều là cùng một gốc: **app nói tiếng Việt nhưng backend
sinh text tiếng Anh**, và text đó được lưu thẳng vào DB rồi hiển thị nguyên văn.

### (a) Công tắc "Nhận yêu cầu thuê" bấm không ăn

Nguyên nhân xác định bằng cách gọi đúng payload mà app gửi:

```
PATCH /api/v2/users/me/player-mode {"enabled": false}
409 {"code":"active_order_exists","detail":"Finish your active order before leaving player mode"}
```

Tài khoản demo đang có đơn LIVE + SCHEDULED (chính hai đơn seed cho bài test), nên BE từ chối cho tắt
chế độ player. App bắt lỗi rồi toast một câu vô dụng — 'Cập nhật trạng thái thất bại' — nên trên máy
nó trông như nút bị liệt. Switch là controlled component: PATCH fail → `draft` bị reset → switch nảy
กลับ, không có gì giải thích.

Sửa:
- `src/lib/errorMessages.ts` (mới): bảng map **error code → tiếng Việt** cho toàn bộ code mobile có thể
  nhận. Nối vào `toApiError()` trong `src/api/client.ts:131`, ưu tiên cao nhất, nên *mọi* toast trong
  app (`error instanceof Error ? error.message : ...`) nói tiếng Việt mà không phải sửa từng màn.
  BE vẫn giữ `detail` tiếng Anh — đó là string debug/log, và code mới là hợp đồng.
- `manage-hiring.tsx`: chỉ PATCH mới quyết thành công. Trước đây `refreshUser()` nằm chung trong
  `try`, một lần refresh fail sẽ rolling lại switch dù thay đổi đã được lưu. Giờ tách ra, cộng
  `invalidateQueries(userService)` để switch đọc lại từ server.
- Nguyên hàng "Nhận yêu cầu thuê" thành `Pressable` (mục tiêu chạm ~44dp thay vì chỉ cái switch),
  có `accessibilityRole="switch"` + `accessibilityState`.

### (b) + (c) Notification tiếng Việt, không còn ISO

Toàn bộ `notifications.send*/sendMany` trong BE viết lại bằng tiếng Việt: `orders` (10 chỗ: yêu cầu mới,
accept, từ chối, hủy, kết thúc sớm, đơn bị tạm dừng, phiên tự bắt đầu, cửa sổ đối soát, hoàn thành, hết
hạn), `ratings` (2), `reports` (4), `charity` (2), `admin` (1).

Định dạng thời gian: `formatClockInZone` / `formatDateInZone` / `formatDateTimeInZone` trong
`src/common/time/time.ts` (dùng `zonedParts` + `APP_TIMEZONE`), và `formatVnd` trong `money.ts` thay cho
`toLocaleString('en-US')`. `orders.service` dùng `this.timeZone` đã có sẵn từ S3b.

Cùng gốc nhưng chưa ai báo: **timeline trạng thái đơn** (`order_status_history.reason`) và **note hành
vi** (`behavior_history.note`) cũng là text hiển thị trong app — 8 + 4 chuỗi đã dịch. Notification khi bị
báo cáo còn rò cả UUID đơn (`A report was filed on order ${orderId}`) → bỏ.

Kiểm chứng trên server thật (không phải test mock): đặt một lịch hẹn qua HTTP rồi đọc notification
của player. Cái instant đúng bằng cái user nhìn thấy:

```
startsAt = 2026-11-02T11:00:00.000Z
PLAYER: "Yêu cầu thuê mới" — "Hoàng An muốn hẹn bạn 1 giờ vào lúc 18:00 02/11/2026."
HIRER : "Đã giữ lịch hẹn"   — "Lượt thuê của bạn bắt đầu lúc 18:00 02/11/2026, kéo dài 1 giờ."
```

e2e mới trong `test/booking.e2e-spec.ts` chốt luôn hai bất biến (đúng định dạng, và không có `\dT\d{2}:\d{2}`);
unit test mới cho `formatDateTimeInZone` (kể cả trường hợp offset vượt nửa đêm) và `formatVnd`.

FE: bỏ AM/PM. `formatTimeAmPm` → `formatTime` (HH:mm), `formatDateTimeAmPm` → nhập vào `formatDateTime`
(HH:mm dd/MM/yyyy) — 12 file đổi theo, có rewrite bằng sed và diff lại từng dòng.

### (d) Ô trắng góc phải màn Thông báo

`ScreenHeader` luôn render một ô để cân xứng với logo, và khi không có `right` thì ô đó mang style
`logoWrap` — tức `backgroundColor: colors.plaque`. `plaque` = `neutral.white` ở sáng và
`darkNeutral.white` = `#FFFFFF` ở tối, nên trong dark mode nó là **một ô vuông trắng 34×34 có viền, trống
không**. Mọi chỗ khác dùng `plaque` đều là nền cho *image* logo nên hợp lệ; chỉ ô này không có gì bên
trong. Sửa: placeholder thành spacer trong suốt (`logoSpacer`, chỉ render khi `showLogo`), màn
Lịch sử cũng hết cùng lỗi.

### Kiểm chứng

BE: `typecheck` 0 · `lint` 0 · unit **29/29** (thêm 4 test format) · e2e **46/46** (thêm 1 test
notification). FE: `typecheck` 0 · `expo lint` 0. Gọi HTTP thật lên server đã build lại để lấy hai
notification ở trên.

## 16b. Đơn bị hủy không được phép đánh giá

User báo: sau khi hủy lịch hẹn thì app mở màn **"Kết thúc thuê"** với 5 sao + ô nhập + nút Gửi.
Validate bằng HTTP trên đúng luồng đó (tạo booking → accept → cancel):

```
created   PROPOSED   canRate=false
accepted  SCHEDULED  canRate=false
cancel    CLOSED / CANCELLED_BY_HIRER  canRate=false
refetch   CLOSED     canRate=false hasRated=false
POST /orders/:id/ratings  -> 409 invalid_order_state "Only finished orders can be rated"
POST /orders/:id/reports  -> 409 invalid_order_state "Only finished orders can be reported"
```

Vậy **BE đã đúng từ đầu** (`canRate` = là người trong cuộc + status ∈ {DISPUTED, SETTLED, ENDED_EARLY}
+ còn trong cửa sổ + chưa đánh giá). Lỗi hoàn toàn ở mobile: màn `end-order` có một nhánh footer
"Kết thúc" render cho *mọi* trạng thái không phải confirmable/booked — nên đơn CLOSED (và cả LIVE) vẫn
có nút, và `finish()` đẩy hirer sang form đánh giá mà server sẽ từ chối.

Sửa:
- `canRate` được đưa qua adapter (`OrderModel.canRate` + `toLegacyOrder`) thay vì app tự đoán trạng thái
  — một nguồn sự thật vẫn là BE.
- `end-order`: nút thứ ba chỉ hiện khi `isHirer && canRate`, đổi label "Kết thúc" → **"Đánh giá"** (nút
  không còn "kết thúc" cái gì cả), và footer không render nếu không có hành động nào.
- Bỏ nút đó để lại một lỗ hổng khác: tab Lịch sử mở đơn LIVE vào thẳng `end-order` (`openOrder` route
  đơn chưa có feedback tới đây), nên màn này phải có hành động cho phiên đang chạy → thêm nhánh
  `isLiveSession` với nút **"Vào phiên đang thuê"** dẫn sang `hiring-stage` (nơi thật sự có countdown và
  "Kết thúc sớm"). Trước đây nút "Kết thúc" trên đơn LIVE chỉ đưa sang form đánh giá bị server từ chối.
- `rating/[orderId]`: guard ngay trong màn — `!canRate` thì hiện EmptyState "Không đánh giá được lượt
  thuê này" + nút về Lịch sử, không có form. Chặn cả đường deep-link và state cũ.
- Cùng họ lỗi: nút **báo cáo** trên AppBar của `end-order` cũng luôn hiện, kể cả đơn đã hủy → gate bằng
  `isFeedbackOpenOrder` (helper mới, mirror `FEEDBACK_ORDER_STATUSES`). Hai chỗ còn lại ("Có vấn đề"
  trong nhánh DISPUTED, nút báo cáo trong màn rating) vốn đã nằm trong tập feedback.
- Label "Lý do kết thúc sớm" đổi thành "Lý do kết thúc" vì cùng field đó chứa lý do hủy/tạm dừng.

Test chốt lại: `booking.e2e-spec` thêm `canRate === false` + 409 `invalid_order_state` cho đơn vừa hủy;
`dispute-window.e2e-spec` thêm positive control `canRate === true` sau khi SETTLED (để assertion kia
không thành hiển nhiên đúng). e2e **46/46**.

### GAP của đợt này

- **Notification cũ vẫn tiếng Anh.** Text đã lưu vào `notifications` nên sửa code không sửa được dữ
  liệu; dev DB còn nguyên các row cũ (dùng nút "Xoá đã đọc" trên màn Thông báo để dọn). Không backfill
  vì đó là UPDATE hàng loạt trên dữ liệu user.
- **Deep-link notification vẫn chưa đi.** `referenceUrl` là `/orders/:id` (đường dẫn API) trong khi app
  cần `/end-order/:id` — bấm notification chỉ đánh dấu đã đọc. Thuộc S4b.
- **User withdrawal không có notification.** Admin duyệt/từ chối rút tiền của user thì user không nhận
  được gì (charity thì có). Cần thêm một `notifications.send` trong luồng duyệt withdrawal.
- **Ledger `note` vẫn tiếng Anh** (`Top-up via MoMo`, `Escrow released`, ...) — **đã sửa ở §22** (đổi
  chuỗi tại nguồn + backfill `0009`). Mobile không hiển thị field này (`toLegacyTransaction` bỏ qua),
  chỉ admin/log thấy.
- **Player chưa đánh giá được Hirer.** `POST /orders/:id/ratings` cho cả hai bên và `canRate` cũng đúng
  cho player, nhưng form `/rating/[orderId]` viết nguyên văn theo ngôi người thuê ("Nhập vào đánh giá
  người chơi của bạn", "Tố cáo đơn này"). Nên `end-order` chỉ mở form cho hirer; player thấy lịch sử.
  Cần một lượt copy theo chiều (suy ra từ `isHirer`) rồi mới mở cho player — không làm âm thầm ở đợt này.
- **`end-order` và `history/[orderId]` trùng nhau gần hết** (cùng một đơn, hai màn, hai tập hành động).
  Thuộc phần "gộp màn" của S4b.
- **`active_order_exists` chặn cả DISPUTED.** Một đơn đang chờ đối soát (phiên đã kết thúc) vẫn khiến
  player không tắt được chế độ nhận thuê trong suốt cửa sổ khiếu nại. Đã giải thích bằng tiếng Việt,
  nhưng về nghiệp vụ thì chỉ LIVE/SCHEDULED mới thật sự "còn chạy".
- **Vẫn còn text tiếng Anh lộ ra khi BE trả `validation_failed` cho field lạ**, vì map chỉ COVER các
  code đã biết; code mới thêm sau này sẽ rơi về `detail` tiếng Anh.

## 17. S5.1 — đã thực hiện: ba bậc admin + nhật ký thao tác

Bối cảnh: mọi tài khoản `role=ADMIN` đều cầm được toàn bộ console — khoá user, duyệt rút tiền, bật
bảo trì, sửa fee. Không có lịch sử nào trả lời được câu "ai đã đổi số tiền này và vì sao".

### Thiết kế đã chốt

Thêm **cột mới `users.admin_role`**, không đụng `users.role` (cột đó đang là điều kiện của nhiều
JOIN + check trong luồng tiền). `admin_role` là **varchar + CHECK**, không phải enum:
`ALTER TYPE ADD VALUE` không chạy được trong transaction nên không an toàn cho migration tự động.

| Bậc | Được làm |
| --- | --- |
| `MODERATOR` | users (đọc + sửa + chỉnh điểm), feedback, reports, ratings, broadcast |
| `FINANCE` | withdrawals, charity payout, charity account, config |
| `SUPER_ADMIN` | tất cả, cộng maintenance và catalog |

`RolesGuard` đọc thêm metadata `AdminRoles`; `SUPER_ADMIN` là wildcard nên mỗi route chỉ khai các bậc
chuyên biệt. Route nào chỉ gắn `@Roles('ADMIN')` (stats, các endpoint đọc) thì mọi admin đều vào được.

`admin_role` chỉ lộ ra trên đúng những view đã có email (`UserSummary`/`UserProfileView` khi
`includeEmail`), nên thẻ player công khai không tiết lộ tài khoản đó là staff. Token cũ cấp trước
migration không có claim → `adminRole = null` → **fail closed**, phải refresh pair mới dùng được console.

### Guard "còn lại ít nhất một super admin"

`PATCH /admin/users/:id` từ chối (409 `last_super_admin`) nếu thao tác đó khiến không còn super admin
nào đang hoạt động — bao gồm cả trường hợp tự hạ bậc chính mình và trường hợp một MODERATOR khoá tài
khoản admin cuối cùng (đường này tồn tại thật vì route sửa user thuộc bậc MODERATOR). Guard đặt trước
transaction nên không để lại audit row ảo.

### Lý do bắt buộc + nhật ký

| Hành động | Field bắt buộc | Error khi thiếu |
| --- | --- | --- |
| khoá / mở khoá / đổi `role` / đổi `admin_role` | `reason` | 422 `reason_required` |
| broadcast toàn hệ thống | `reason` | 400 (DTO) |
| chỉnh điểm thủ công | `note` | 400 (DTO) |
| duyệt rút tiền user | `note` | 400 (DTO) |
| duyệt payout charity | `note` | 400 (DTO) |
| hoàn tiền / khoá tài khoản từ báo cáo | `note` | 422 `reason_required` |
| xoá / sửa config | `reason` (tuỳ chọn) | — |

`AuditService` (@Global) ghi `admin_audit_logs`: ai (`adminId` + `adminRole`), làm gì (`action` trong
bảng từ vựng `AUDIT_ACTIONS`), tác động lên entity nào, vì sao (`reason`), và `detail` jsonb trước/sau.
**Write nằm trong transaction của thao tác nghiệp vụ**, nên mutation rollback thì không còn bản ghi
"đã làm".

### Migration + seed

`0008_ordinary_randall.sql` thêm cột + CHECK + index và tạo `admin_audit_logs`, cộng một câu backfill:

```sql
UPDATE "users" SET "admin_role" = 'SUPER_ADMIN' WHERE "role" = 'ADMIN' AND "admin_role" is null;
```

Diễn tập trên bản copy `.data/pgdata` trong TEMP trước khi chạm DB dev. Kết quả trên cả hai:

| Kiểm tra | Giá trị |
| --- | --- |
| `information_schema.columns` | `admin_role` character varying(16) |
| `pg_constraint` | `CHECK (admin_role IS NULL OR admin_role IN ('SUPER_ADMIN','MODERATOR','FINANCE'))` |
| `pg_indexes` | `users_admin_role_idx` + 3 index của `admin_audit_logs` |
| `users` sau backfill | 1 SUPER_ADMIN · (seed thêm) 1 MODERATOR · 1 FINANCE |
| `drizzle.__drizzle_migrations` | 9 |

Seed tạo thêm `demo.moderator@playtogether.local` và `demo.finance@playtogether.local` (mật khẩu
`Demo@12345`, in ra ở `db:seed`) để thử được RBAC mà không phải hạ bậc tài khoản thật.

### Kiểm chứng

e2e mới `test/admin-rbac.e2e-spec.ts` — 13 test, tất cả qua: ma trận route × bậc (kèm 403
`admin_role_required`), SUPER_ADMIN đi lọt mọi cổng (positive control cho wildcard), `reason_required`,
422 khi gắn bậc cho tài khoản không phải ADMIN, 409 `last_super_admin` cho cả hai đường (tự hạ bậc và
MODERATOR khoá admin), **rồi chính thao tác đó thành công 200 sau khi có super admin thứ hai** để chứng
minh guard không phải lệnh cấm mù. Các luồng ghi audit chạy hết đường thật: duyệt rút tiền của user
(`wallet/withdrawals` → `admin/withdrawals/:id`, kèm 409 khi quyết định lại lần hai), từ chối payout
charity (tiền quay lại số dư), MODERATOR đóng một góp ý, MODERATOR chỉnh điểm thủ công, và một vòng
trọn catalog (game → rank → xoá cả hai) do SUPER_ADMIN làm, MODERATOR bị 403 ở cùng route. Test cuối
đếm audit row theo action và đối chiếu `reason` + `adminRole` + `detail` trước/sau của từng hành động —
các request bị từ chối (không có lý do, sai bậc, hạ super admin cuối) **không để lại row nào**. **20/23 action** trong
`AUDIT_ACTIONS` đã được HTTP cover.

Gate: BE `tsc` 0 · `eslint` 0 · unit **29/29** · e2e **59/59** · build 0. Admin web `tsc` 0 ·
`oxlint` 0 · build 0. Mobile `tsc` 0 · `expo lint` 0.

Probe trực tiếp lên API dev sau khi chạy migration + seed:

```
login super 200 "SUPER_ADMIN"      GET /admin/users (moderator)      -> 200
login mod   200 "MODERATOR"        GET /admin/withdrawals (moderator) -> 403 admin_role_required
login fin   200 "FINANCE"          GET /admin/withdrawals (finance)   -> 200
                                   GET /admin/users (finance)         -> 403 admin_role_required
                                   GET /admin/withdrawals (super)     -> 200
```

### GAP của đợt này

- **Access token sống thêm tối đa 15 phút sau khi đổi bậc.** `revokeAll` chỉ thu refresh token;
  access token mang claim cũ vẫn dùng được tới hết TTL. Đã có hành vi tương tự với khoá tài khoản từ
  trước, nên đợt này giữ nguyên và ghi lại; muốn triệt để cần thêm `token_version` trên `users` và đối
  chiếu mỗi request.
- **Chưa có endpoint đọc nhật ký.** `admin_audit_logs` mới ghi, chưa có `GET /admin/audit-logs` —
  thuộc S5.2, kèm trang nhật ký trong console ở S5.3.
- **Admin web mới chỉ buộc nhập lý do ở các chỗ làm hỏng build** (broadcast, chỉnh điểm, duyệt rút
  tiền, sửa user). Configs và Maintenance vẫn chưa có ô "Lý do" dù BE đã nhận field này.
- **ReportsPage chưa báo trước** yêu cầu `note` khi tick hoàn tiền/khoá tài khoản — người dùng thấy
  422 từ server thay vì validator của form.
- **`Object.keys(dto)` trong detail của config/charity/catalog** liệt kê field theo tên wire tiếng Anh;
  trang nhật ký (S5.3) cần bản nhãn tiếng Việt cho các tên này.
- **Ba action chưa có e2e riêng**: `charity.create`, `charity.update` (cần fixture tài khoản từ thiện
  mới) và `rating.violation_resolve` (cần fixture đánh giá bị flag). Cùng khuôn code với 20 action đã
  được HTTP cover, nhưng chưa được chạy thật — ghi lại để không nói quá.
- **Catalog bị khoá ở `SUPER_ADMIN`** theo thiết kế đã chốt; nếu sau này MODERATOR phải tự thêm game
  thì chỉ cần bớt một decorator.
- **`eslint.config.mjs` phải tắt type-checked lint cho `**/*.mts`** vì `scripts/seed-lifecycle.mts`
  (scratch dev, chưa commit) làm gate `npm run lint` fail với "not found by the project service"; đây
  là cùng cách xử lý đã áp dụng cho `.js`/`.mjs`.

## 18. S5.2 — đã thực hiện: trang đọc cho console (đơn, tiền, nhật ký, KPI theo kỳ)

Mảng ghi đã có bậc + audit ở §17; mảng đọc thì vẫn chỉ có `count(*)`. Không trả lời được ba câu mà
người vận hành hỏi hằng ngày: "đơn này tiền đi đâu", "ai đã sửa gì và vì sao", "hôm nay doanh thu bao
nhiêu". Thêm 8 route mới trong `admin-ops.*` (controller/service/dto riêng, không phình
`admin.service.ts`).

| Route | Bậc | Nội dung |
| --- | --- | --- |
| `GET /admin/audit-logs` | mọi ADMIN | nhật ký thao tác, lọc theo `adminId/action/entity/entityId/from/to` |
| `GET /admin/orders` | MODERATOR, FINANCE | lọc `status/hirerId/playerId/q/sort/from/to`, kèm tên hai bên + game |
| `GET /admin/orders/:id` | MODERATOR, FINANCE | timeline trạng thái, escrow, ledger, pending credit, rating, báo cáo, cửa sổ đối soát |
| `GET /admin/kpis` | mọi ADMIN | số liệu theo kỳ (mặc định = hôm nay) |
| `GET /admin/money/ledger` | FINANCE | sổ cái kép, lọc theo loại/tài chiều/khoảng tiền/đơn/ví + tổng theo cả tập |
| `GET /admin/money/wallets` | FINANCE | số dư + tiền đang chờ release, lọc `pendingOnly` |
| `GET /admin/money/deposits` | FINANCE | nạp tiền theo trạng thái/khoảng thời gian/số tiền |
| `GET /admin/money/escrows` | FINANCE | tiền giữ hộ, tách settled/refunded/còn giữ |

### Ba quyết định định nghĩa số liệu

1. **KPI đếm theo chuyển trạng thái, không theo trạng thái hiện tại.** Nếu đếm `status = SETTLED` thì
   một đơn kết thúc tuần trước vẫn được tính là "chốt hôm nay" mỗi ngày. `orderStatusHistory` có sẵn
   thời điểm chuyển, nên `settled/endedEarly/disputed/closed` lấy từ bảng đó; `created` lấy từ
   `orders.createdAt`. Vì lý do này `disputed` là **số lượt bước vào đối soát**, còn số tiền đang thực
   sự giữ nằm ở `queue.escrowHeld` (snapshot).
2. **Tiền hoàn chỉ tính phía CREDIT.** Một hoàn tất escrow ghi hai bút toán (ESCROW chi / ví hirer thu);
   cộng cả hai sẽ báo cáo gấp đôi số tiền đã hoàn. Tỷ lệ vấn đề đổi thành
   `endedEarlyRateBps = endedEarly / completed` — `problemRateBps` cũ cộng `disputed` vào tử số với mẫu
   là đơn hoàn tất, ra 200% trên chính dữ liệu test.
3. **Tổng theo tập đang lọc, không theo trang.** `money/ledger` trả `totals.{debits,credits,entries}`
   tính bằng `sum(...) filter (where ...)` trên toàn bộ WHERE, để tài khoản đối soát được với trang khác
   mà không phải cộng thủ công từng trang.

### Bug mà test bắt được

Test "cộng mọi trang rồi so với `totals`" fail ngay lần chạy đầu: `debits` nhận 1.970.000 trong khi
endpoint báo 2.050.000. Nguyên nhân không phải tổng sai mà là **phân trang không ổn định** — các dòng
seed dùng chung một `createdAt`, nên `ORDER BY created_at DESC` không khoá thứ tự và một dòng rơi vào
hai trang, dòng khác bị bỏ sót. Sửa bằng cách thêm khoá phụ `desc(id)` vào cả 6 query phân trang mới.
Test giữ nguyên (cộng toàn trang + assert không trùng id) nên lỗi này không quay lại được.

### Kiểm chứng

`test/admin-ops.e2e-spec.ts` — 7 test, dựng fixture thật: một phiên xác nhận xong (SETTLED, có rating),
một phiên hết giờ đang đối soát (HELD), một phiên kết thúc sớm (prorated + ESCROW_REFUND), một nap tiền
đã xác nhận, một rút tiền đã duyệt, một broadcast. Điểm mạnh là các assertion **chéo endpoint**:

- `kpi.money.refunds` so với `credits` của ba loại_refund lấy từ `money/ledger`;
- `queue.escrowHeld` so với số giữ hộ tính từ `money/escrows?status=HELD`;
- `holder` của bút toán USER phải là tên một tài khoản có ví thật (`money/wallets`);
- `escrow.settledAmount` bằng đúng `totalPrice` của đơn.

Cộng toàn trang ledger: `debits == credits == 2.770.000` trên DB dev — sổ cái kép cân bằng.

Gate: BE `tsc` 0 · `eslint` 0 · unit 29/29 · e2e **66/66** · build 0 — admin `tsc`/`oxlint`/build 0 —
mobile `tsc`/`expo lint` 0. `openapi.json` lên **122 paths**, client orval của cả hai app đã regenerate
(thư mục mới `endpoints/admin-ops`).

Probe bằng HTTP thật trên DB dev (chỉ đọc):

```
GET /admin/kpis            200  created=6 settled=0 endedEarly=1 closed=5 · platformFees=1600 refunds=384000
GET /admin/orders          200  total=8
GET /admin/orders/:id      200  history=3 ledger=4 escrow=REFUNDED(80000/0/80000) ratings=0 reports=0
GET /admin/audit-logs      200  total=0  (DB dev chưa có thao tác nào qua bản build mới)
GET /admin/money/ledger    200  total=34 debits=2.770.000 credits=2.770.000
GET /admin/money/wallets   200  pendingOnly=true -> total=0 (khớp pendingCredits=0)
GET /admin/money/escrows   200  total=5 totalHeld=0 (khớp queue.escrowHeld)
moderator -> /admin/money/ledger 403 admin_role_required · anonymous -> /admin/kpis 401
```

### GAP của đợt này

- **Console chưa dùng các route này.** Dashboard vẫn gọi `/admin/stats`; trang Orders/Money/Audit và
  bộ gate menu theo `admin_role` thuộc S5.3.
- **Phân trang chưa ổn định ở các query cũ.** Hơn chục chỗ khác vẫn `ORDER BY created_at DESC` một mình
  (`admin/feedback`, `behavior/history`, `charities`, `charity-withdrawals`, `donations`, `chats`,
  notifications, transactions...). Cùng loại lỗi trên chỉ chưa bộc phát vì dữ liệu dev chưa đụng ngưỡng
  2 trang. Đã ghi thành task riêng (#102) để sửa một lượt kèm test, không âm thầm vá rải rác — **đã làm ở §20**.
- **`GET /admin/orders?q=` chỉ tìm theo tên hai bên**, chưa tìm theo email/mã đơn/game.
- **KPI chưa có chuỗi theo ngày** (biểu đồ) và chưa có so sánh kỳ trước; hiện chỉ có số tổng trong kỳ.
- **`/admin/stats` vẫn giữ nguyên** để dashboard cũ không vỡ; khi S5.3 đổi sang `/admin/kpis` thì cân
  nhắc bỏ bớt các `count(*)` trùng.
- **Múi giờ mặc định của kỳ KPI** là nửa đêm theo giờ tiến trình Node, không phải `APP_TIMEZONE`. Trên
  máy dev Việt Nam thì trùng nhau; server đặt ở TZ khác nên truyền `from/to` tường minh từ console.

## 19. S5.3 — đã thực hiện: console phân quyền thật + trang vận hành

§18 giao 8 route đọc nhưng console vẫn chưa dùng. Đợt này biến bảng điều khiển thành nơi làm việc: menu
và đường dẫn bị khoá theo bậc, thêm trang Đơn / Sổ tiền / Nhật ký thao tác / Hồ sơ tài khoản, bảng nào
cũng có bộ lọc trên URL + sắp xếp + chọn dòng, và mọi thao tác mà API đòi lý do đều đi qua hộp thoại.

### Quyền: một bảng, FE và BE khớp nhau

`src/auth/permissions.ts` là bảng `SECTION_TIERS` duy nhất; `AdminLayout` dựng menu từ nó và chặn đường
dẫn trực tiếp bằng `Result 403`, `PartyLink` chỉ hiện tên dạng liên kết khi bậc mở được trang đó.

| Đường dẫn | Bậc được mở | Decorator phía API |
| --- | --- | --- |
| `/` Tổng quan | mọi ADMIN (kể cả chưa phân quyền) | `GET /admin/stats`, `/admin/kpis` — chỉ `@Roles('ADMIN')` |
| `/orders`, `/orders/:id` | MODERATOR, FINANCE | `@AdminRoles('MODERATOR','FINANCE')` |
| `/users` (danh sách) | MODERATOR | `GET/PATCH /admin/users(…)`: MODERATOR |
| `/users/:id` (hồ sơ) | MODERATOR, FINANCE | `@AdminRoles('MODERATOR','FINANCE')` |
| `/feedback`, `/reports`, `/ratings`, `/broadcast` | MODERATOR | MODERATOR |
| `/withdrawals`, `/money`, `/charities`, `/configs` | FINANCE | FINANCE |
| `/catalog`, `/maintenance` | SUPER_ADMIN | SUPER_ADMIN |
| `/audit` | SUPER_ADMIN | **mới siết** — xem dưới |

`SUPER_ADMIN` là wildcard nên không cần liệt kê. Hồ sơ một tài khoản tách khỏi danh sách vì API mở
`GET /admin/users/:id` cho cả FINANCE: duyệt rút tiền cần nhìn ví và đơn còn mở của người được trả.

### Hai lỗ đặc quyền bịt trong lúc kiểm chứng

1. **MODERATOR tự nâng bậc.** `PATCH /admin/users/:id` nhận `role` và `adminRole` từ mọi bậc MODERATOR;
   FE chỉ làm mờ ô chọn, nên kẻ chặn được nó gọi thẳng API là tự gán `SUPER_ADMIN`. Thêm guard trong
   `admin.service.ts: updateUser`: nếu một trong hai cột đổi thật sự mà người gọi không phải
   `SUPER_ADMIN` → 403 `admin_role_required`. FE khoá thêm ô "Vai trò" cho đúng bậc. Test mới
   `keeps role and tier assignment to SUPER_ADMIN` chứng minh cả hai hướng: MODERATOR tự nâng bậc → 403,
   tự kết nạp ADMIN cho người khác → 403, nhưng vẫn đổi được trạng thái hiện diện (200), và **403 không
   để lại dòng audit nào**.
2. **`GET /admin/audit-logs` để ngỏ.** Route chỉ có `@Roles('ADMIN')`, trong khi menu giữ nhật ký cho
   SUPER_ADMIN — nghĩa là mọi tài khoản admin (kể cả chưa được gán bậc) đọc được tên người, lý do và số
   tiền trong bút toán. Thêm `@AdminRoles('SUPER_ADMIN')`; test quyền trong `admin-ops.e2e-spec.ts` đảo
   kỳ vọng sang 403 cho MODERATOR.

### Regression do AntD v6 gây ra, chỉ thấy khi render thật

`Modal destroyOnHidden` (tên mới của `destroyOnClose`) unmount luôn cả `Form`, nên `setFieldsValue()` gọi
trong `onClick` **mất tác dụng**: form mount sau đó và reset store. Hệ quả thật sự, đo trên trình duyệt:

| Before (form mount lại) | Giá trị thực tế khi mở modal |
| --- | --- |
| Điều chỉnh điểm · ô "Điểm hành vi" | `""` → báo "Hãy nhập thông tin cho trường…" dù cây a11y ghi `value="0"` |
| Điều chỉnh điểm · ô "Điểm hài lòng" | `""` → cùng lỗi |
| Xử lý rút tiền / feedback / báo cáo / đánh giá · "Quyết định" | không lựa chọn nào được tick |
| Cập nhật tài khoản · "Vai trò", "Trạng thái hiện diện" | select trống |

Khuôn sửa thống nhất: bỏ hết `setFieldsValue` trong hàm mở modal, thay bằng `initialValues` tính từ dòng
đang chọn (`destroyOnHidden` mount lại form mỗi lần mở nên `initialValues` luôn đúng, và không còn dư dữ
liệu của dòng trước). Áp cho 11 form ở 8 trang: Users (sửa, điểm), Withdrawals (user + charity),
Feedback, Reports, FlaggedRatings, Configs (sửa, tạo từ default), Charities, Catalog (game, thể loại).
Ngoại lệ có chủ đích: form thêm/sửa thứ hạng trong ngăn kéo Catalog là form nội tuyến luôn mount sẵn
(không có `destroyOnHidden`), nên `setFieldsValue` ở đó vẫn đúng — đừng "chỉnh cho đồng nhất" mà làm hỏng.
Đã render lại để xác nhận: mở "Thứ hạng · Arena of Valor" (9 hạng), bấm sửa hạng bậc 3 → form nội tuyến
hiện `Thứ tự = 3`, `Tên = Vàng`, nút đổi thành "Lưu / Huỷ"; bấm Huỷ trả form về rỗng và bảng giữ nguyên 9
dòng.
Bằng chứng sau sửa — modal "Điều chỉnh điểm · Lê Thu Hà" trả `numbers: ["0","0"]`, modal
"Cập nhật Quỹ Ánh Sáng" trả đủ 6 trường + công tắc `true`, modal "Cập nhật Age of Empires" trả
`Tên game = Age of Empires`, `Tên hiển thị = Đế Chế`, `Thể loại = RTS, Strategy` (multi-select).

### Trang mới, số liệu thật đã nhìn trên màn hình

| Trang | Nội dung kiểm chứng được (DB dev) |
| --- | --- |
| `/orders` | 8 dòng, nhãn tiếng Việt ("Đã đóng" + "Người thuê huỷ"), phí `10%`, hai bên là liên kết hồ sơ; bấm sắp xếp ghi `?sort=startsAt` và API trả đúng thứ tự NULLS LAST |
| `/orders/:id` | timeline 3 bước, escrow "Đã hoàn", 3 bảng (ledger, đánh giá, báo cáo), mã đơn copyable |
| `/money` | 4 tab `?tab=`; tổng server tính: Chi 2.770.000 đ = Thu 2.770.000 đ trên 34 bút; chọn một trang ra "20 bút toán đang chọn · Chi ra 900.000 đ · Thu vào 800.000 đ · Cân đối -100.000 đ" |
| `/audit` | 1 dòng: Người thao tác "Play Together Admin" + bậc "Quản trị cao nhất", Hành động "Cộng trừ điểm thủ công", Đối tượng "Tài khoản", Lý do tiếng Việt; mở rộng ra `points 0 / satisfaction 0` |
| `/users/:id` | khối tài khoản (bậc, khu vực, giá/giờ), khối điểm (2 thanh + lịch sử), khối tiền (dùng được, chưa tới hạn, rút đang chờ), đơn (đã thuê/đã chơi/còn mở + bảng gần đây), khối kiểm duyệt (4 số đếm) |
| `/` (Dashboard) | card "Trong kỳ" hôm nay: tạo mới 6, hoàn tất 0, kết thúc sớm 1 (100%), trả cho người chơi 14.400 đ, phí 1.600 đ, hoàn 384.000 đ; card "Việc cần làm ngay" chỉ hiện liên kết mà bậc đó mở được |

### những thứ đã bỏ để khỏi tự bắn vào chân

- Trang đăng nhập không còn in tài khoản demo + mật khẩu (menu quyền đã chia, để credential sẵn trên
  màn hình là mời gọi).
- Không còn "bấm là lưu": bật/tắt bảo trì, khoá tài khoản, gửi broadcast, xoá cấu hình đều qua
  `ConfirmReasonModal` (lý do bắt buộc 3–500 ký tự); broadcast còn hiện lại tiêu đề + nội dung và cảnh
  báo "Không thể thu hồi sau khi gửi."
- Không có control nào dẫn tới 403: liên kết tên, nút "Xem đơn", shortcut dashboard đều dựng theo bậc.

### Kiểm chứng

Gate: BE `tsc` 0 · `eslint` 0 · unit **29/29** · e2e **68/68** (8 suite) · `nest build` 0 — admin `tsc -b`
0 · `oxlint` 0/41 file · `vite build` 0. Không đổi schema nên `openapi.json` vẫn 122 paths, không cần
regenerate client.

Render thật bằng browser (login từng bậc, không đọc doc): SUPER_ADMIN 14 mục menu · MODERATOR 7 mục
("Tổng quan, Đơn thuê, Tài khoản, Báo cáo, Đánh giá gắn cờ, Phản hồi, Thông báo") · FINANCE 6 mục
("Tổng quan, Đơn thuê, Chi trả, Sổ tiền, Từ thiện, Cấu hình"). GATE: FINANCE vào `/users`, `/audit`,
`/maintenance` → "Trang này nằm ngoài quyền của bạn", không fetch bảng; MODERATOR vào `/money`, `/audit`
→ cùng màn 403. FINANCE mở `/users/:id` → hồ sơ render, nút "Danh sách tài khoản" ẩn, `/money` vẫn 20
dòng + cột chọn. Trong modal "Cập nhật Tài Chính" ở bậc MODERATOR: `Vai trò disabled=true`,
`Quyền vận hành disabled=true`, `Trạng thái hiện diện disabled=false`.

Probe HTTP trên DB dev, chạy lại sau khi API được khởi động lại bằng bản build mới (token từng bậc):

```
GET   /admin/audit-logs                       MODERATOR     403 admin_role_required
GET   /admin/audit-logs                       SUPER_ADMIN   200 (3 dong)
PATCH /admin/users/:id  {role:ADMIN}          MODERATOR     403 admin_role_required
PATCH /admin/users/:self{adminRole:SUPER}     MODERATOR     403 admin_role_required
PATCH /admin/users/:id  {status:ONLINE}       MODERATOR     200   <- khong chan oan
PATCH /admin/users/:id  {isActive,+reason}    SUPER_ADMIN   200
```

Đọc lại hồ sơ sau probe: bậc của kiểm duyệt vẫn là `MODERATOR` (403 không ghi non một cột nào), và
`audit-logs` có đúng 3 dòng — 1 dòng `user.behavior_adjust` cộng thêm 2 dòng `user.update` của hai lệnh
200; hai lệnh 403 không để lại dòng nào.

### GAP của đợt này

- **Copy tiếng Anh ngay tại nguồn, hai chỗ — đã làm ở §22.** (a) `note` của bút toán sổ cái:
  `Funds held for order`, `Platform commission`, `Order earnings (pending release)`, `Escrow released`,
  `Unused escrow refunded to hirer`, `Top-up via MoMo` — hiện cả trong console lẫn ví trên app. (b) 6/6
  tiêu đề cấu hình do seed sinh: `Payout objection window (hours)`, `Platform fee (basis points)`… cùng
  mô tả. Trái với quy tắc ở §16 (user-facing copy phải là tiếng Việt ngay chỗ ghi). Hướng sửa là đổi chuỗi
  ở nguồn **và** backfill `ledger_entries.note` / `system_configs` đã ghi sẵn — đụng dữ liệu tiền đang có
  nên đã hỏi và được chủ dự án duyệt trước khi chạy (§22).
- **Counter do seed đặt sẵn không khớp số thật.** `demo.player3` hiện "Đơn 35" và "4.9 (31)" trong danh
  sách, còn hồ sơ `/users/:id` đếm đúng 2 đơn thật và 0 dòng đánh giá. Không phải bug truy vấn (cột
  `ordersCount`/`rating` là denormalized), nhưng DB dev cần seed lại tạo đơn + đánh giá thật nếu muốn
  console tự đối chiếu được.
- **Hành vi cộng/trừ điểm chạm trần không để lại lịch sử.** Đã ghi ở §17; trang "Điểm hành vi" trong hồ
  sơ vì thế có thể hiện "Chưa có biến động điểm nào" với tài khoản seed.
- **`/admin/stats` vẫn là nguồn của dashboard cũ**; hai số trùng với `kpis` (đơn theo trạng thái) nên cân
  nhắc bỏ dần.
- **Chưa có test tự động cho `permissions.ts`.** Bảng quyền mới được chứng minh bằng render từng bậc;
  một unit test thuần (tier × section) sẽ rẻ hơn nhiều so với mở trình duyệt mỗi lần đổi.
- **Phân trang theo timestamp ở query cũ**: đã sửa kèm test ở §20.
- **`/audit` vẫn in `Mã bản ghi` (UUID)** trong phần mở rộng — cố ý, vì đó là khoá để tra bản ghi gốc;
  danh sách thì không hiện id nào.

## 20. #102 — đã thực hiện: mọi danh sách phân trang đều có khoá phụ

§18 phát hiện một trang ledger đọc trùng dòng vì `ORDER BY created_at DESC` không khoá được thứ tự khi
nhiều dòng trùng giờ. Đây là quét toàn bộ để tìm đúng lớp lỗi đó, không vá từng triệu chứng.

### Quét

Script tạm đi qua mọi `*.service.ts`, với mỗi dòng `.orderBy(` xem 8 dòng kế tiếp có `offsetOf(` hay không
(= có phân trang) và dòng orderBy có khoá thứ hai chưa. Kết quả: **23 query phân trang thiếu khoá phụ**,
6 query đã có (làm ở §18), 30 điểm `orderBy` không phân trang.

### Đã sửa (khoá phụ always là id của bảng, cùng chiều với cột chính)

| Module | Query | Khoá phụ |
| --- | --- | --- |
| notifications | danh sách thông báo | `desc(notifications.id)` |
| orders | đơn của tôi | `desc(orders.id)` |
| chat | danh sách cuộc trò chuyện + `conversations()` | `desc(chats.id)` |
| feedback | của người dùng + bảng admin | `desc(feedback.id)` ×2 |
| reports | của tôi + bảng admin | `desc(reports.id)` ×2 |
| ratings | bảng theo người + builder sort động | `desc(ratings.id)` ×2 |
| behavior | lịch sử điểm | `desc(behaviorHistory.id)` |
| users | tìm kiếm người chơi (builder sort động) + bảng admin | `asc(users.id)` / `desc(users.id)` |
| wallet | transactions, deposits, withdrawals ×2 | `desc(<table>.id)` ×4 |
| charity | charities ×2, charity-withdrawals ×2, donations | `desc(<table>.id)` ×5 |
| catalog | games theo tên, game types theo tên | `asc(games.id)`, `asc(gameTypes.id)` |

Hai chỗ sort động không sửa ở `.orderBy()` mà sửa ở **nơi dựng danh sách cột**
(`users.service.ts: playerOrderBy`, mảng `orderBy` trong `ratings.service.ts: listForUser`) để mọi nhánh
sort đều được hưởng, không lặp lại từng `case`.

`chat.conversations()` không phân trang nhưng cũng được thêm khoá: nó chọn tin mới nhất của từng người đối
diện, nên khi hai tin trùng giờ thì "mới nhất" là do may mắn — cùng lớp lỗi, cùng một dòng sửa.

Không sửa: 24 điểm `orderBy` không phân trang còn lại (OTP lấy cái mới nhất, `pendingCredits` nhả theo
`availableAt`, xếp hạng top player, `gameRanks.no`…). Không có ranh giới trang thì không có trùng/xổ.

### Test mới: `test/pagination-stability.e2e-spec.ts` (9 test)

- 7 test `it.each` đi hết từng trang với `pageSize=3` trên notifications, orders, wallet transactions,
  tìm kiếm người chơi (ranking + newest), admin users, games; assert (a) không id nào lặp lại, (b) số dòng
  đọc được bằng `meta.total`.
- 1 test " còn đủ danh sách multi-page": phải có ≥5 danh sách `total > pageSize`, để suite không âm thầm
  trở nên vô nghĩa khi dữ liệu seed gọn lại.
- 1 test dựng fixture **6 dòng notifications cùng một timestamp** rồi assert thứ tự API trả về đúng bằng
  id **giảm dần**, và đi từng trang 2 vẫn ra đúng chuỗi đó.

**Positive control (quan trọng nhất).** Đã tạm bỏ `, desc(notifications.id)` và chạy lại: test "cùng
timestamp ⇒ xếp theo id" **fail**, trong khi cả 7 test đi trang vẫn **pass**. Trên PGlite, bản thân việc đi
trang không phát hiện được lỗi vì heap order ổn định; assertion có răng thật là assertion thứ tự. Giữ cả
hai vì trên Postgres thật (có song song, có plan khác) thì kiểu đi trang vẫn có thể lộ lỗi.

### Kiểm chứng

Gate: `tsc` 0 · `eslint` 0 · unit **29/29** · e2e **77/77** (9 suite) · `nest build` 0. Không đổi schema
nên `openapi.json` giữ nguyên và client hai app không cần regenerate.

Probe live trên DB dev với bản build mới (đi trang `pageSize=3` so với `pageSize=50`, ba tiêu chí đều OK):

```
/users?sort=newest       total=  5   /users?sort=ranking  total=  5   /users?sort=price_desc total= 5
/games                   total= 26   /game-types          total= 32   /notifications         total=  5
/wallet/transactions     total=  0   /admin/users         total= 12   /admin/audit-logs      total=  3
/admin/money/ledger      total= 34   /admin/orders        total=  8
```

Đã loại trừ một nghi ngờ từ chính probe: `/wallet/transactions` của `demo.player3` trả 0 dòng **không phải
lỗi** — ví cô ấy `availableBalance = 0` và cả hai đơn của cô ấy đều "Người thuê huỷ", nên không có bút toán
nào được ghi.

### GAP còn lại

- 24 điểm `orderBy` đơn cột chưa phân trang vẫn phụ thuộc thứ tự heap khi trùng giá trị; chỉ thành vấn
  đề nếu sau này chúng được phân trang hoặc dùng để chọn "dòng đầu tiên" như `conversations()` vừa sửa.
- `orders.service.ts: orderHistory` và `ratings.service.ts: ratings của một đơn` mới sắp một cột, an toàn
  vì số dòng trong một đơn rất nhỏ — cùng khuôn, sửa được nếu muốn tất định tuyệt đối.
- Các danh sách phía app mobile dùng `pageSize` lớn nên chưa từng lộ lỗi này; nếu sau này mobile chuyển
  sang infinite-scroll thật (pageSize nhỏ), test ở §20 đã giữ trật tự cho các route đó.

---

## 21. S4b (phần 1) — đã thực hiện: onlineManager, deep-link thông báo, trạng thái lỗi, dashboard Player

Bốn việc nhỏ của S4b (#110–#113). Mục tiêu chung: app phải biết mình đang offline, thông báo phải đưa
người dùng tới đúng màn, màn tiền/khám phá phải nói thật khi dữ liệu không tải được, và tài khoản Player
phải mở app ra bằng công việc của mình chứ không bằng danh sách người để thuê.

### #110 · `onlineManager` — React Query phải biết là đang mất mạng

`src/providers/AppProviders.tsx` thêm `OnlineSync`: lấy trạng thái mạng ban đầu bằng
`Network.getNetworkStateAsync()` và nghe `Network.addNetworkStateListener`, đẩy kết quả vào
`onlineManager.setOnline()`.

Không có nó thì React Query mặc định thiết bị luôn online: nó bắn request vào một sóng radio đã chết,
hiện spinner thay vì lỗi, và — nặng nhất — **không tự refetch khi mạng về lại**, kể cả với
`refetchOnReconnect`. `expo-network` (~58.0.4) chạy được trong Expo Go nên việc này không đòi development
build; trên web thì `Platform.OS === 'web'` trả sớm để không phá trình duyệt.

### #111 · Deep-link từ thông báo — bấm vào là tới nơi

`referenceUrl` của API trước đây bị mapper bỏ quên. Nay `NotificationModel.referenceUrl`
(`src/api/types.ts`) giữ nó, `toLegacyNotification` (`src/api/v2/mappers.ts`) truyền qua, và module mới
`src/navigation/deepLink.ts` đổi đường dẫn resource thành màn hình:

| `referenceUrl` | Màn hình |
| --- | --- |
| `/orders/:id` | `openOrderById` → `/receive-request`… qua `/end-order/:id` hoặc `/history/:id` tuỳ trạng thái |
| `/charities/:id` | `/charity/[charityId]` |
| `/feedback/me`, `/users/me` | `/feedback`, `/personal` |
| `/notifications`, `/charity`, `/history` | từng màn tương ứng |

`openOrderById` là bản chuyển của logic vốn nằm trong `(tabs)/history.tsx` — giờ hai nơi (danh sách đơn và
danh sách thông báo) đi chung một đường, không ai tự đoán lại nữa. `(tabs)/notifications.tsx` đánh dấu
đã đọc rồi điều hướng trong một lần bấm.

Phía BE, `reports.service.ts` (3 chỗ) từng trỏ thông báo tới `/reports/:id` — một đường dẫn **app mobile
không có màn nào đọc**. Đổi thành `/orders/${report.orderId}`: người trong cuộc cần thấy đơn, không cần
trang tố cáo.

### #112 · Trạng thái lỗi ở màn tiền và màn khám phá

Nguyên tắc áp dụng chung: **chỉ hiện lỗi khi không còn gì để hiển thị**. Một lần refresh lỗi khi danh sách
cũ vẫn đang trên màn hình thì giữ danh sách đó — xoá đi là mất thông tin người dùng đang đọc.

| Màn | Trước | Sau |
| --- | --- | --- |
| `(tabs)/home.tsx` | ray fail → "Chưa có dữ liệu"; cả ba ray fail vẫn hiện "Chưa có người chơi nào" | ray có `error`/`onRetry` → "Không tải được mục này. [Thử lại]"; cả ba fail từ đầu → một `ErrorView` "Không tải được danh sách người chơi…" |
| `(tabs)/personal.tsx` | ví fail âm thầm, số dư cũ trong `user.userBalance` hiện như thể vừa đọc | hàng nhắc trong thẻ ví: "…con số có thể đã cũ. Nhấn để thử lại" (bấm là refetch) |
| `withdraw-amount.tsx` | ví fail → hiện số cũ, bấm "Rút tiền" báo "Số dư khả dụng không đủ!" — sai nguyên nhân | vẫn hiện số cuối cùng biết + "Không kiểm tra được số dư mới nhất. [Thử lại]"; khi `isError` thì chặn nút và nói đúng lý do |
| `filter.tsx` | game fail → lưới trống không lời | "Không tải được danh sách game. Bạn vẫn có thể áp dụng các bộ lọc khác. [Thử lại]" |
| `user-profile.tsx` | ảnh fail → "Chưa có hình ảnh nào" (một lời nói dối, người dùng có thể tưởng mình mất ảnh) | "Đang tải ảnh..." khi pending; "Không tải được ảnh của bạn. [Thử lại]" khi fail trắng |
| `search-history.tsx` | mục "Các tựa game hot" fail → "Chưa có dữ liệu" | "Không tải được danh sách game hot. [Thử lại]" |

`PlayerRail` nhận thêm `error` + `onRetry`; `transactions.tsx`, `statistics.tsx`, `history.tsx`,
`notifications.tsx` đã có `ErrorView` từ trước nên chỉ được dùng làm khuôn.

### #113 · `PlayerDashboard` — nửa Player của tab Trang chủ

`src/components/player/PlayerDashboard.tsx`, chỉ render khi `user.isPlayer`, **không xoá bớt** nội dung
cũ (ray khám phá vẫn còn, vì Player cũng đi thuê). Mỗi khối đọc API thật:

| Khối | Nguồn dữ liệu | Bấm vào |
| --- | --- | --- |
| "Doanh thu hôm nay / Tháng này" | `/wallet/earnings/summary` qua `userService.getStatistic()` | `/statistics` |
| "Xđ đang chờ về ví · n phiên" | `/wallet` → `pendingCredits` | `/unactive-balance` |
| Trạng thái nhận thuê + sao | `user.isPlayer`, `user.rate`, `user.numOfRate` | `/manage-hiring` |
| "Phiên đang diễn ra" | `/orders/requests?status=PROPOSED→LIVE` (`getRequestsOfPlayer('Starting')`) | `/hiring-stage/:id` |
| "Yêu cầu chờ phản hồi" | `getRequestsOfPlayer('Processing')` + hạn trả lời `respondBy` | `/receive-request/:id` |
| "Phiên sắp tới" | `getRequestsOfPlayer('Scheduled')` + `startsAt` | `/history/:id` |
| "Đánh giá gần đây" | `/users/:id/ratings?sort=newest` (3 dòng) | `/ratings/:id?canReport=1` |

Ba danh sách đơn là ba query riêng vì API lọc theo trạng thái; gộp thành một trang 10 đơn rồi lọc phía
client sẽ làm rơi đúng những phiên đã hẹn từ lâu. Key của chúng nằm dưới `queryKeys.playerWork.root` để
kéo-tuần-cập nhật ở tab Trang chủ (`refreshAll`) vô hiệu hoá cả dashboard — không thì pull-to-refresh chỉ
làm mới phần dành cho người thuê.

### Kiểm chứng

Gate: FE `tsc --noEmit` 0 · `npx expo lint` 0 · BE `tsc` 0 · `eslint` 0 · e2e **77/77 (9 suite)**.

Render thật bằng `expo start --web` + harness chặn mạng (chặn XHR/fetch theo URL `/api/v2`, chỉ mở
`/users/me` và `/auth/*` để phiên sống), trên tài khoản `demo.hirer1` và `demo.player1`:

```
home (tất cả ray fail)   → "Không tải được danh sách người chơi. Kiểm tra kết nối rồi thử lại." + [Thử lại]
home (1 ray sống)        → 2 ray fail hiện "Không tải được mục này. [Thử lại]", ray sống vẫn nguyên thẻ
personal                 → số 674.000 đ + "Không tải được số dư mới nhất, con số có thể đã cũ…", bấm [Thử tải lại số dư] → /wallet refetch (log 2 lần)
withdraw-amount          → "Số dư khả dụng 674.000 đ" + "Không kiểm tra được số dư mới nhất. [Thử lại]", bấm [Thử lại] → /wallet refetch
user-profile             → "Không tải được ảnh của bạn. [Thử lại]" thay cho "Chưa có hình ảnh nào"
transactions (đã có sẵn) → "Không thể tải dữ liệu. [Thử lại]" (positive control cho harness)
home, Player account     → "Ca trực của bạn · Đang nhận yêu cầu thuê · Doanh thu hôm nay 0 đ · Doanh thu
                            tháng này 14.400 đ · Chưa có khoản nào chờ thanh toán · Chưa có đánh giá nào"
hàng "Yêu cầu chờ phản hồi" → "Hoàng An · 2 giờ · 144.000 đ · trả lời trước 20:44 02/11/2026", bấm →
                               /receive-request/68fd652f… (màn hiển thị đúng đơn, đủ lời nhắn và game)
```

Hàng "Yêu cầu chờ phản hồi" được kiểm bằng một đơn **thật**: tạo `POST /orders` từ `demo.hirer1` cho
`demo.player1` (2 giờ, 160.000 đ), quay lại app bằng đúng tài khoản Player, rồi **huỷ đơn** khi xong — ví
người thuê trở về `availableBalance = 674000`, `pendingBalance = 0` như trước khi kiểm.

### GAP còn lại

- **Gộp ~15 màn trùng nhau** (§15) vẫn chưa làm: đây là thay đổi cấu trúc điều hướng, cần xem thật trên
  điện thoại rồi mới quyết; sẽ trình bày một bảng đề xuất "màn nào dồn về đâu" để duyệt, không làm âm thầm.
- Hai hàng lỗi mới (`filter.tsx`, `search-history.tsx`) chưa kịp thấy trên màn hình: harness chặn mạng làm
  **request đầu tiên của một screen mới bị abort** nên query treo ở `pending` thay thành `error`. Cùng
  khuôn component đã render thật ở ba nơi khác (ray, user-profile, withdraw) nên phần hiển thị không phải
  là điều chưa biết — nhưng nói thẳng là hai chỗ này mới chỉ chắc về code, chưa chắc về ảnh.
- Nút "Rút tiền" khi `isError`: logic chặn đã typecheck và đọc được, nhưng chưa bấm thật tới được dòng
  toast (synthetic press không đi tới `Pressable` của nút đó trong harness).
- ~~**`axios` không đặt `timeout`**~~ — **dòng này sai, đã kiểm lại bằng grep**: cả hai instance đều đã
  đặt `timeout: 20000` (`src/api/client.ts:47` và `src/api/client.ts:55`). Mạng nửa mở vì thế fail sau
  20 giây rồi rơi đúng vào nhánh lỗi vừa làm ở mục này, không treo spinner vô hạn. Hiện tượng "query treo
  ở `pending`" mà harness gặp là do chính nó abort request đầu tiên, không phải do thiếu timeout.
- "Phiên sắp tới" và "Đánh giá gần đây" chưa render được hàng thật vì DB dev không có đơn đã chấp nhận và
  không có đánh giá nào (đã kiểm bằng API: 0 dòng PROPOSED/SCHEDULED/LIVE cho cả 6 tài khoản demo).
- `useOngoingOrderGuard` (§15) vẫn tự đẩy Player đang có request treo vào thẳng `/receive-request/:id`,
  nên dashboard chỉ hiện hàng đó sau khi bấm "Quay lại". Đó là hành vi có chủ ý của guard; nếu muốn
  dashboard là nơi thấy hết mọi yêu cầu, cần quyết định guard chỉ chạy khi app từ background lên.

---

## 22. #114 — đã thực hiện: tiếng Việt ở đúng nguồn, kể cả dữ liệu đã có

§19 và §20 đều để lại cùng một GAP: hai chỗ mà hệ thống **tự ghi chữ tiếng Anh** vào dữ liệu mà con
người đọc — `ledger_entries.note` và `system_configs.title/description`. Đây là phần đã duyệt làm.

### Hiện trạng trước khi sửa

| Chỗ | Số điểm ghi | Vì sao nó lộ ra |
| --- | --- | --- |
| `wallet.service.ts` | 16 note tiếng Anh (`Top-up via MoMo`, `Funds held for order`, `Platform commission`…) | cột "Ghi chú" của trang Tiền in nguyên văn (`MoneyPage.tsx:169`) |
| `charity.service.ts` | 5 note (`Donation to X`, `Charity withdrawal requested`…), trong khi dòng kế cận đã là `Ủng hộ từ X` — một cặp bút toán nói hai thứ tiếng | cùng trang Tiền / Từ thiện |
| `db/schema/config.ts` | 6/6 `title` + `description` tiếng Anh | trang Cấu hình hiển thị cho operator |
| `db/seed.ts` | 4 note `Seed demo …` | như trên |

Ràng buộc cần nhớ: **`detail` của API vẫn tiếng Anh** (client đổi mã → tiếng Việt). Ledger note không phải
`detail`: nó không có mã, không ai parse, chỉ có người đọc. Vì vậy nó thuộc nhóm "user-facing copy phải
viết bằng tiếng Việt tại nguồn" (mục 16), không thuộc nhóm giữ tiếng Anh.

### Đã làm

1. `LEDGER_NOTES` — một bảng hằng số duy nhất ở đầu `wallet.service.ts`, kèm comment nói rõ vì sao dịch
   được an toàn: không code nào đọc lại note, mobile tự đặt tiêu đề dòng từ `type` của bút toán.
2. `Withdrawal ${status.toLowerCase()}` (sinh ra `Withdrawal rejected` / `Withdrawal cancelled`) đổi thành
   phân nhánh theo trạng thái → `Rút tiền bị từ chối, đã hoàn về ví` / `Rút tiền bị huỷ, đã hoàn về ví`.
3. Năm note từ thiện, bốn note seed, và `withdrawals.note = 'Cancelled by user'` → tiếng Việt.
4. `CONFIG_DEFAULTS`: 6 tiêu đề + 6 mô tả tiếng Việt; `key` vẫn là identifier tiếng Anh vì code đọc nó —
   hai thứ khác nhau có chủ ý, và comment trong file nói thẳng điều đó.
5. **Backfill** `src/db/migrations/0009_vi_money_copy.sql` (sinh bằng `drizzle-kit generate --custom`, nên
   journal + snapshot đúng khuôn): 21 UPDATE trên `ledger_entries`, 1 trên `withdrawals`, 6 trên
   `system_configs`. Mỗi câu chỉ khớp đúng một chuỗi cũ nên chạy lại là no-op; hai chuỗi có tham số
   (`Donation to %`, `Withdrawal payout for %`) ghép `LIKE` + `substring(... from '…(.*)$')` để giữ tên
   tổ chức.
6. Test mới trong `admin-ops.e2e-spec.ts`: note của DEPOSIT / WITHDRAWAL / ESCROW_HOLD / ORDER_EARNING /
   PLATFORM_FEE phải đúng chuỗi tiếng Việt; 9 loại do hệ thống ghi phải **có ít nhất một chữ có dấu**;
   mọi `title`/`description` trong `/admin/configs` cũng phải có dấu.

### Ảnh hưởng tới dữ liệu tiền — nêu theo quy tắc

Migration chỉ đổi `note` / `title` / `description`. Không câu nào chạm `amount`, `direction`,
`account_type`, cặp DEBIT/CREDIT, `wallets`, `pending_credits`, hay bất kỳ cột số nào; tổng
nợ–có mà trang Tiền tính ra vì thế không đổi. Không app nào đọc note để quyết định nghiệp vụ. Chạy một
lần khi khởi động (dev: `AUTO_MIGRATE`), prod: `npm run db:migrate`.

### Kiểm chứng

Gate: `tsc` 0 · `eslint` 0 (cả `src` và `test`) · unit **29/29** · e2e **78/78** (9 suite) · `nest build` 0
và `dist/db/migrations/0009_…sql` có thật (asset được copy).

- **Positive control:** tạm đổi `platformFee` về `'Platform commission'` → test mới fail, và dòng lỗi in
  ra đúng `Received array: ["Platform commission"]`; đổi lại → pass. Test có răng thật.
- **Cú pháp regex của migration:** chạy PGlite thật, `substring('Donation to Vina Charity' from
  'Donation to (.*)$')` → `Ủng hộ cho Vina Charity`, nên vế `||` không sinh chuỗi cụt.
- **Live trên DB dev** (rebuild → restart API → đọc bằng `demo.finance`):

```
ledger rows: 34
  ESCROW_HOLD    -> ["Tạm giữ tiền cho đơn thuê"]
  ESCROW_RELEASE -> ["Giải ngân tiền đơn thuê"]
  ESCROW_REFUND  -> ["Hoàn phần tiền thuê chưa dùng cho người thuê"]
  ORDER_EARNING  -> ["Doanh thu đơn thuê (chờ về ví)"]
  PLATFORM_FEE   -> ["Phí nền tảng"]
  DEPOSIT        -> ["Nạp tiền qua MoMo", "Số dư ban đầu (dữ liệu mẫu)"]
  DONATION       -> ["Quyên góp dữ liệu mẫu"]
english-left (ledger) : []
configs: 6/6 có title + description tiếng Việt (value giữ nguyên 24 / 72 / 1000 / 5 / 0 / 6)
english-left (configs): []
```

### GAP còn lại

- Chú thích do **admin tự gõ** vẫn có thể là tiếng Anh (hai spec đang gửi `'Reviewed: no fault found.'`,
  `'Verified against session logs.'`). Console không kiểm soát ngôn ngữ người nhập; muốn chặt thì thêm
  nhắc "ghi chú tiếng Việt" ở hộp thoại, không phải việc của migration.
- `config.dto.ts:18` giữ `example: 'Platform fee (basis points)'` — ví dụ trong tài liệu Swagger dành cho
  dev, không phải màn operator. Sửa nó sẽ làm `openapi.json` đổi và phải regenerate client của hai app,
  nên để nguyên.
- DB dev chưa có dòng nào mang note từ thiện/`Donation to %`, nên hai câu `substring` mới chỉ chắc về cú
  pháp (đã kiểm bằng PGlite) chứ chưa được kiểm trên dữ liệu thật.

---

## 23. #115 — ĐỀ XUẤT gộp màn trùng nhau (chờ duyệt, chưa thi công)

§15 chuyển phần "gộp ~15 màn" lại vì nó đổi cấu trúc điều hướng và cần xem thật trên điện thoại.
Dưới đây là bảng sau khi đã kiểm từng màn: màn nào đọc gì, ghi gì, thoát bằng gì, và ai đang đẩy nó vào.
**Chưa file nào bị xoá.** Duyệt nhóm nào thì làm nhóm đó.

### Nhóm A — Ví: 5 màn → 2 (bỏ 3). Rủi ro: thấp–trung

| Hiện tại | Vấn đề đã xác minh | Đề xuất |
| --- | --- | --- |
| `deposit-method.tsx` + `withdraw-method.tsx` | Hai file là **bản sao gần từng dòng** cùng tiêu đề "Phương thức thanh toán", 3 hàng mà 2 hàng ghi "Sắp ra mắt" | Thay bằng một bước chọn/điền gộp vào màn tiền |
| `deposit-method` + `deposit` | `deposit` mới là màn thật (gọi `POST /wallet/deposits`, `Linking.openURL`, rồi `dismissTo('/(tabs)/personal')`) | **một màn "Nạp tiền"**: số tiền + nút MoMo |
| `withdraw-amount` + `withdraw-method` + `withdraw` | `withdraw-amount` **không POST gì cả** — nó chỉ push số tiền qua route param; `withdraw` mới là màn gửi | **một màn "Rút tiền"**: số tiền + số MoMo + gửi |

Phải giữ khi gộp: 3 màn này đang nằm trong `SHEET_ROUTES` (`_layout.tsx`) nên tên route mới phải được khai
báo lại, nếu không mất hiệu ứng sheet và cử chỉ vuốt-down; nhánh "thiếu số dư" của
`send-hiring-request.tsx` đang push `/deposit-method`; `withdraw.tsx` kết thúc bằng `replace('/transactions')`
(thay vì push) để không quay lại một luồng đã nộp — giữ nguyên; guard số dư stale vừa thêm ở §21 phải ở lại
màn sau khi gộp.

### Nhóm B — Giờ và kỹ năng: 4 màn → 2 (bỏ 2). Rủi ro: thấp

| Cặp | Bằng chứng | Đề xuất |
| --- | --- | --- |
| `add-hour.tsx` ≡ `update-hour.tsx` | cùng `TimeRangeFields` + cùng kiểm tra khoảng giờ; khác nhau đúng một động từ (POST so với PATCH); `update-hours` truyền record thành 4 param chuỗi phẳng | một màn `hour-form.tsx?id=…` (không id = thêm mới) |
| `add-skill.tsx` → `update-skills.tsx` | `add-skill` là lưới checkbox + `POST /users/me/skills`, rồi `router.back()` về đúng màn cha | chọn ngay trong `update-skills` bằng một lớp phủ; giữ nguyên guard `canDelete = !isPlayer \|\| games.length > 1` |

Lưu ý: `manage-hiring` là nơi người dùng thật sự tới (tab Cá nhân + dashboard Player §21), còn
`update-hours`/`update-skills` chỉ có một inbound là chính nó — nên giữ hai màn list, bỏ hai màn "thêm".

### Nhóm C — Một hành động "lý do": `report-comment.tsx` gộp vào `report/[orderId].tsx` (bỏ 1). Rủi ro: thấp

Hai màn đều là "chọn lý do có sẵn + Lý do khác + gửi". Khác duy nhất endpoint
(`POST /ratings/{id}/flag` so với `POST /orders/{id}/reports`). Đề xuất: một component `ReasonSheet` dùng
chung, `report-comment` trở thành tham số của nó. Kết thúc bằng `router.back()` của `report-comment` phải
giữ nguyên vì nó mở từ `ratings/[userId]`.

### Nhóm C2 — `rating/[orderId].tsx` thành lớp phủ trong `end-order/[orderId].tsx` (bỏ 1). Rủi ro: trung bình

`rating` chỉ có **một** inbound: `replace('/rating/[orderId]')` từ `end-order`. Nó là form 5 sao + lời nhắn,
đứng sau `end-order` vốn đã hiển thị đúng thông tin đó. Gộp thành lớp phủ "Đánh giá" trong `end-order` thì
bớt một lần chuyển màn và hết phải sửa tiêu đề sai ("Kết thúc thuê" nhưng nội dung là đánh giá).
PHẢI giữ: nhánh `!order.canRate` (EmptyState) và việc `end-order` polling 10s để bắt nhịp đối soát.

### Nhóm D — Tìm kiếm: 2 màn → 1 (bỏ 1). Rủi ro: trung bình

`search-history.tsx` và `search.tsx` mỗi màn tự dựng một ô nhập riêng (trùng UI rõ rệt);
`search-history` là **điểm vào duy nhất** (từ `HomeAppBar`), còn `search` là trang kết quả.
Đề xuất: một màn `search.tsx` — khi chưa có từ khoá thì hiện 3 mục (gần đây / top / game hot), có từ khoá
thì hiện kết quả. Giữ: tự ghi lịch sử khi có `?q=` (`lastRecorded` ref), `POST /search-history`, và hợp
đồng `useDiscoveryFilter` với `filter.tsx`. Sâu hơn: `player-list.tsx` đọc `playerListDraft` — module
singleton **không ai clear**, nên nó không deep-link được và có thể hiện dữ liệu cũ; nên đổi sang
`?title=&ids=` hoặc một query, nhưng đó là việc riêng, không bắt buộc trong đợt gộp.

### Nhóm E — Phản hồi: `feedback/[feedbackId].tsx` (bỏ 1). Rủi ro: trung bình

Màn chi tiết chỉ đọc, không có router, và **copy nguyên** `statusOf()` + `TYPE_LABELS` từ `feedback.tsx`;
nội dung hiển thị gần đúng phần tử trong danh sách. Đề xuất: mở rộng hàng trong danh sách (accordion) và
xoá màn detail. Rủi ro duy nhất: `routeOfReference` (§21) đang map `/feedback/me` → `/feedback`, không
vào detail, nên deep-link không hỏng.

### Nhóm F — "Số dư khả dụng" thật ra là tiền chờ (sửa nhãn, có thể bỏ 1). Rủi ro: liên quan tiền

`unactive-balance.tsx` mang tiêu đề "Số dư khả dụng" nhưng **liệt kê tiền chưa về ví**, và không có nút
nào: hành động nhả tiền (`POST /wallet/pending-credits/release`) nằm ở tab Cá nhân. Người dùng đọc tên rồi
vào màn thì thấy thứ ngược lại. Hai hướng: (a) chỉ đổi nhãn thành "Tiền đang chờ về ví" + chuyển nút "Thu
hồi về khả dụng" vào màn này (sửa tên trong `SHEET_ROUTES` không cần thiết vì route giữ nguyên), hoặc (b)
gộp vào `transactions.tsx` như một chip lọc. **Tôi thiên về (a)** vì đây là màn tiền, và nó cũng là đích của
hàng "chờ về ví" trên dashboard Player mới (§21).

### Nhóm G — KHÔNG đề xuất gộp lúc này (nêu để khỏi bị hỏi lại)

- **`receive-request/[orderId]` ≡ `hiring-negotiating/[orderId]`**: đúng là một xương sống (poll 5s,
  countdown `processExpired`, tự hết hạn, latch `expiredHandled`) nhìn từ hai phía, gộp được về lý thuyết.
  Nhưng đây là hai màn mỏng manh nhất app: mỗi màn có **hai effect tự điều hướng**, `sessionRouteFor()`
  quyết định sang `/end-order` hay `/hiring-stage`, và cả `useOngoingOrderGuard` (§15) lẫn
  `openOrderById` (§21) đều push thẳng vào tên route. Gộp hai màn này là sửa UI của machines trạng thái
  tiền — chỉ nên làm **sau khi đã test máy** và đã có danh sách đường đi thật.
- **`end-order/[orderId]` vs `history/[orderId]`**: cùng một đơn, một bên là bảng điều khiển đang chạy,
  một bên là biên lai lưu trữ; `deepLink` chọn cái này hay cái kia theo điều kiện đã ghi ở §21. Không gộp
  màn — chỉ nên bỏ lần gọi trùng: `history/[orderId]` dùng `orderDetail`, `end-order` dùng `order`; và
  `statistics.tsx` đang gọi `/users/me` **hai lần** (`statistic` + `profile`), cùng lúc `item3 === item4`
  được vẽ hai lần. Sửa code, không đổi điều hướng.
- **`update-hobbies.tsx` vs `onboarding/take-categories.tsx`**: cùng lưới chọn game, nhưng một bên là khai
  báo lần đầu (`replace`, không đường quay lại) một bên là cài đặt. Nên trích component dùng chung, giữ cả
  hai route.
- **`personal-change-password.tsx` vs `(auth)/change-password.tsx`**: biểu mẫu giống nhau nhưng cái sau ăn
  `passwordResetDraft {email, token}` và kết thúc bằng `replace('/(auth)/login')`; cái trước phải lưu token
  đã xoay (`applyAuthResponse`). Gộp là nhập hai ngữ cảnh phiên vào nhau — không nên.

### Tổng kết theo con số

| | Số màn bỏ được | Rủi ro | Cần test máy trước? |
| --- | --- | --- | --- |
| A · Ví | 3 | thấp–trung | không (nhưng phải test lại MoMo) |
| B · Giờ/kỹ năng | 2 | thấp | không |
| C · Lý do + đánh giá | 2 | thấp–trung | không |
| D · Tìm kiếm | 1 | trung bình | nên |
| E · Phản hồi | 1 | trung bình | không |
| F · Tiền chờ | 0–1 | liên quan tiền | không |
| **Cộng** | **9–10** (cộng 3 chỗ trùng code ở G) | | |

Thứ tự tôi kiến nghị: **B → A → C → E → D → F**, vì B/C/E gần như chỉ là nhập file, A động vào luồng tiền
nhưng chỉ là biểu mẫu, D đổi điểm vào tìm kiếm, F liên quan nhãn tiền. G (receive-request ↔
hiring-negotiating) để riêng, chỉ làm khi anh xác nhận sau lượt test điện thoại.

---

## 24. #116, #117 — đã thực hiện: bỏ một con số bịa, bỏ một lần gọi trùng, bỏ một bộ nhớ dai

Ba việc này tìm ra khi liệt kê danh sách còn nợ, và đều sửa được trong phạm vi nhỏ. Cùng lúc, một dòng
GAP ở §21 hoá ra là **sai sự thật** — đã gạch và ghi lại đúng chỗ.

### Dòng GAP sai ở §21

§21 viết "`axios` không đặt `timeout`, màn hình sẽ giữ spinner vô hạn". Kiểm lại bằng grep thì cả hai
instance đều đã có `timeout: 20000` (`src/api/client.ts:47` và `src/api/client.ts:55`). Hiện tượng "query
treo ở `pending`" mà harness gặp là do chính nó abort request đầu tiên, không phải do thiếu timeout.
Đã sửa ngay dòng đó trong §21 thay vì để nó nằm ở đây.

### #116 — màn Thống kê vẽ hai ô giống hệt nhau dưới hai nhãn khác nhau

| | Trước | Sau |
| --- | --- | --- |
| `getStatistic` (`src/api/services/users.ts:127`) | gọi `/users/me` **+** `/wallet/earnings/summary`, trả `item1..item4` với `item3 = item4 = completion` | chỉ gọi `/wallet/earnings/summary`, trả `item1..item2` |
| `statistics.tsx:75-80` | hai ô `CompletionStat`: "Theo ngày" và "Theo tháng" — **cùng một con số**, không có số nào theo ngày hay theo tháng cả | một ô: `40 %` kèm dòng `2/5 đơn đã hoàn thành` |
| Số lần `/users/me` cho một lần mở màn | 2 (một của `queryKeys.statistic`, một của `queryKeys.profile`) | 1 |

Tỷ lệ hoàn thành giờ tính từ chính hồ sơ mà màn đã tải: `users.ts` không còn cần `/users/me`, còn màn hình
dùng `user.numOfFinishedOrder / user.numOfOrder`. Muốn có hai con số "theo ngày / theo tháng" thật thì API
phải trả theo kỳ — việc đó chưa làm, nên thà hiển thị một con số đúng còn hơn hai con số giống nhau.

Thay đổi nhỏ phải có: `UserModel` thêm `numOfFinishedOrder` (`src/api/types.ts:108`) và mapper nối
`profile.completedOrdersCount` vào nó (`src/api/v2/mappers.ts:223`). `PlayerDashboard` chỉ đọc `item1`/
`item2` nên không bị ảnh hưởng — đã kiểm lại bằng ảnh chụp màn hình.

### #117 — `playerListDraft` là module singleton không ai dọn

`clear()` có tồn tại nhưng không nơi nào gọi; màn `player-list` đọc `get()` và giữ nguyên dữ liệu đó cho
tới khi có người ghi đè. Hệ quả: mở lại màn này từ một lối vào khác (deep link, mount lại) sẽ thấy đúng cái
danh sách của ray trước đó, mà tiêu đề thì vẫn là danh sách mới.

Sửa: `playerListDraft.take()` **đọc và xoá trong một nhịp**, `player-list.tsx` gọi nó đúng một lần lúc mount
(`useState(() => playerListDraft.take())`), và màn rỗng phân biệt hai trường hợp — "chưa có ai trong danh mục
này" khi thật sự có danh sách rỗng, "Danh sách chỉ hiện khi mở từ một mục trên trang Chủ." khi không có
payload nào được giao.

Đây là bản sửa nhỏ. Bản đúng nghĩa (màn dùng query theo `?list=` để deep-link được thật) vẫn để ngỏ, vì
nó cần quyết định cách truyền danh sách "Đề xuất cho bạn" — khoá query của nó chứa cả `topPlayerIds`.

### Kiểm chứng

Metro web + API dev đang chạy, account `demo.player1` (đã kiểm cả hai vai):

```
/statistics (load thẳng)     → "Tỷ lệ hoàn thành · 40 % · 2/5 đơn đã hoàn thành"
                              "Số đơn thuê đã nhận: 5" khớp mẫu số, "…đúng giờ: 1" là số khác, không bị trộn
/network xhr trên màn đó     → /users/me + /wallet một cặp, /wallet/earnings/summary một lần
/player-list (load thẳng)    → "Không có dữ liệu · Danh sách chỉ hiện khi mở từ một mục trên trang Chủ."
home → [Xem tất cả] ray "Các người chơi mới" → tiêu đề "Các người chơi mới", 5 dòng
home → [Xem tất cả] ray "Đề xuất cho bạn"    → tiêu đề "Đề xuất cho bạn", 2 dòng  ← không lẫn dữ liệu
home (dashboard Player)      → "Doanh thu hôm nay 0 đ · Doanh thu tháng này 14.400 đ" vẫn đúng
gate                          → FE `typecheck` 0, FE `lint` 0
```

### GAP còn lại của đợt này

- Bản trên điện thoại chưa test; cả ba sửa đều là mã nguồn mở, Metro web mới là nơi đã thấy.
- Chưa có lối vào thật nào deep-link vào `/player-list`, nên đường "mở từ ngoài vào" mới chỉ chắc ở phần màn
  rỗng — chưa chứng minh được tình huống "danh sách của ray khác" trên máy thật.
- Phát hiện thêm, **chưa sửa**: tab Cá nhân gọi `/users/me` hai lần khi mở app (một của `refreshUser()`
  trong `useOngoingOrderGuard`, một của query `queryKeys.profile` mà chính màn đó dùng). Không hiện ra lỗi
  nào, chỉ thừa một round-trip; gộp được nhưng đụng tới thứ tự khởi động phiên nên để riêng.

---

## 25. Lưới chip sụp thành một ô mỗi dòng trên điện thoại — đã sửa

Anh chụp màn "Chi tiết đánh giá" trên máy: sáu nút lọc (Tất cả, 5 sao … 1 sao) mỗi nút một dòng, nửa màn
hình bên phải trống. Đây là lỗi hình học, không lỗi component.

### Nguyên nhân

`gridItemWidth()` chia chiều rộng cho số cột rồi làm tròn **lên** theo `PixelRatio`:

| Khổ | DPR | ô tính ra | 2 ô + gap | container | kết quả |
| --- | --- | --- | --- | --- | --- |
| 375 | 3 | 167.5 → **167.667** | 343.33 | 343 | **tràn → wrap thành 1 ô/dòng** |
| 411 | 3 | 185.5 → **185.667** | 379.33 | 379 | **tràn → 1 ô/dòng** |
| 360 | 2 | 160 | 328 | 328 | vừa |
| 390 | 3 | 175 | 358 | 358 | vừa |
| 428 | 2 | 194 | 396 | 396 | vừa |

Thừa **0,33 điểm ảnh** là đủ để flexbox đẩy ô thứ hai xuống dòng. đúng là lớp iPhone X/Pixel — vì những khổ
chia hết thì không bị, nên lượt kiểm trên web và trên tablet trước đây không phát hiện ra.

### Đã sửa

1. `src/theme/responsive.ts:67` — `gridItemWidth` dùng `Math.floor` thay vì làm tròn lên; tám chỗ gọi nó
   (lưới game, danh mục onboarding, game hot, thẻ giờ/kỹ năng…) tự hết tràn, thiết kế lưới giữ nguyên.
2. Hai **hàng lọc** bỏ width cố định để chip ôm lấy nhãn: `ratings/[userId].tsx:85` và `transactions.tsx:88`.
   Bộ lọc không phải lưới dữ liệu — ép mỗi ô rộng bằng nhau mới là thứ khiến nó trông hỏng.
3. Nhân tiện: nhãn lọc `Donate` → `Quyên góp` (`transactions.tsx:30`) — chữ Anh còn sót trên đúng màn đang
   xem; `type: 'Donate'` gửi lên API giữ nguyên vì đó là tham số, không phải chữ hiển thị.
4. Quét luôn chữ `MOMO` in hoa trong câu hiển thị: 6 chỗ ở `transactions.tsx:34-35`, `deposit.tsx:78,82,90`,
   `deposit-method.tsx:24`, `withdraw-method.tsx:27`, `withdraw.tsx:67` → `MoMo` cho khớp với note sổ tiền
   phía API ("Nạp tiền qua MoMo"). `partnerCode: 'MOMO'` ở `src/api/v2/mappers.ts:504` **giữ nguyên** vì đó
   là mã nhà cung cấp gửi đi, không phải chữ người dùng đọc.

### Kiểm chứng

```
node mô phép chia (7 khổ, gap 8, min 150): overflow trước = 2/7, sau = 0/7
đo thật trong iframe 375 px (cùng khổ máy anh):
  /ratings        → 4 chip dòng 1 (y=68) + 2 chip dòng 2 (y=116), mỗi chip 66–74 px, bắt đầu từ lề 16
  /transactions   → 3 + 3, chip cuối đọc "Quyên góp"
  /search-history → lưới game 2 ô/dòng, hàng rộng nhất 338 px ≤ 343 px container
gate: FE typecheck 0, FE lint 0
```

### Điều chưa chắc

Phép đo trong trình duyệt chạy `devicePixelRatio = 1`, nên nó chứng minh **bố cục** (mỗi dòng mấy ô, có tràn
không) chứ không tái hiện đúng con số 0,33 px gây tràn — số đó nằm ở bảng mô phỏng phía trên. Cần mắt anh
trên máy xác nhận lần cuối, và đây cũng là lý do nên đổi `roundToNearestPixel` thành `floor` ở gốc thay vì
chỉnh từng màn.

---

## 26. #118 — đã thực hiện: card "Ca trực" ở lại khi tắt chế độ người chơi

### Vì sao trước đây tắt là mất hết

`isPlayer` không phải cờ "rảnh/bận" mà là **chế độ hai chiều**: [orders.service.ts:109](../play-together-be/src/modules/orders/orders.service.ts) chặn tài khoản đang bật chế độ player đi thuê người khác,
còn [:125](../play-together-be/src/modules/orders/orders.service.ts) chặn nhận thuê khi đang tắt. Vì vậy tab Nhà
gate cả `PlayerDashboard` bằng `user?.isPlayer` — tắt là mất luôn thẻ, và muốn bật lại phải đi
Cá nhân → Quản lý nhận thuê (hai màn).

### Sau khi sửa

| Trạng thái | Tab Nhà hiển thị | Query chạy |
| --- | --- | --- |
| Đang bật | dashboard đầy đủ như cũ | tất cả |
| Đang tắt, tài khoản **có kỹ năng** | một card gọn: "Đang ngừng nhận yêu cầu thuê" + **[Bật lại]** + dòng giải thích + "Quản lý nhận thuê" | chỉ `Scheduled` và `Starting` |
| Đang tắt, không có kỹ năng | không có card (đúng như trước) | không |

Ca **đã hẹn / đang diễn ra vẫn được liệt kê** khi đang tắt, vì API không huỷ đơn lúc đổi chế độ — để mất
thứ người dùng đã cam kết là hỏng việc. Doanh thu, khoản chờ về ví, yêu cầu chờ phản hồi và đánh giá thì
ẩn, và query của chúng cũng không gọi.

Tín hiệu "có thể bật lại" lấy miễn phí: `/users/me` vốn đã trả `skills[]`
([users.service.ts:147](../play-together-be/src/modules/users/users.service.ts)), nên chỉ thêm
`hasSkills: profile.skills.length > 0` vào mapper (`src/api/v2/mappers.ts:220`, kiểu ở `src/api/types.ts:106`)
— không thêm request, không đổi API.

Cùng đợt: `player_mode_disabled` dịch lại cho đúng nghĩa. BE bắn mã này khi **chính người dùng** đang ở chế
độ player mà bấm đi thuê, nhưng app lại nói "Người chơi này đang ngừng nhận thuê." — ngược hẳn. Nay là
"Bạn đang ở chế độ nhận thuê. Tắt chế độ này trước khi đi thuê người khác." (`src/lib/errorMessages.ts:31`).

### Kiểm chứng (Metro web + API dev, account `demo.player1`)

```
đăng nhập bằng UI → /home            → "Ca trực của bạn · Đang nhận yêu cầu thuê · Doanh thu hôm nay"
tắt bằng công tắc ở /manage-hiring   → "Ngừng nhận yêu cầu thuê"
mở lại /home (tải lại trang, tức là vào lạnh)
                                     → "Đang ngừng nhận yêu cầu thuê"
                                       "Khi đang tắt, bạn không nhận đơn mới và cũng không đi thuê được.
                                        Ca đã hẹn vẫn còn hiệu lực."
                                       [Bật lại]  +  "Quản lý nhận thuê"
                                       (không còn dòng doanh thu / yêu cầu chờ nào)
bấm [Bật lại]                        → dashboard đầy đủ trở lại ngay ("Đang nhận yêu cầu thuê",
                                       "Doanh thu hôm nay", "Yêu cầu chờ phản hồi")
trạng thái cuối = như lúc bắt đầu kiểm (isPlayer = true)
gate: FE typecheck 0, FE lint 0
```

### Sự cố môi trường gặp phải khi kiểm — anh cần biết trước khi test máy

1. **API chết và DB dev cũ không mở được nữa.** Trong lúc kiểm, tiến trình API (khởi động từ 21:05) chết;
   thử khởi động lại thì PGlite báo `RuntimeError: Aborted()`. Đã loại trừ nguyên nhân "hai process cùng mở":
   liệt kê toàn bộ `node.exe` không còn tiến trình nào thuộc repo này, và mở `.data/pgdata` bằng một script
   PGlite độc lập **cũng abort**. Kết luận: cluster ở `.data/pgdata` hỏng sau lần bị giết cứng.
  Cách xử lý: **không xoá gì cả** — dựng DB mới ở `.data/pgdata-fresh` (chạy migration 0001→0009 + seed),
   còn thư mục cũ nằm nguyên đó nếu muốn xem lại.
2. **Metro hỏng module graph** ("Requiring unknown module \"892\"") → khởi động lại với `--clear`.
3. **Bẫy đáng nhớ nhất:** PGlite là tiến trình đang mở, nên dữ liệu do một process **khác** ghi vào sau đó
   (ở đây là `seed-cli`) sẽ không hiện ra với API. Hệ quả là mọi lần đăng nhập trả `invalid_credentials`
   dù mật khẩu đúng — đã xác định bằng cách mở thẳng DB và `bcrypt.compare` cho `true`. Chữa: chạy seed
   **trước**, rồi mới bật API.
4. Vì DB dev là bản seed mới: các đơn/ledger từng tạo trong những đợt kiểm trước **không còn**. Tài khoản
   demo (`demo.player1..6`, `demo.hirer1..2`, admin, charity) hoạt động lại bình thường, điện thoại vẫn trỏ
   `http://192.168.1.20:3000/api/v2`.

### GAP còn lại

- Chưa kiểm trên máy thật (mọi thứ ở đây là Metro web). **Đã sai ở chính chỗ này**: ảnh chụp từ máy cho thấy
  card khi tắt bị vỡ bố cục dù `innerText` trên web đọc ra đủ mọi dòng — xem §27.
- Card khi tắt chưa cho đổi **giá/giờ online** tại chỗ — vẫn phải vào "Quản lý nhận thuê". Có chủ đích: thêm
  hai ô nhập vào thẻ sẽ biến thẻ thành một màn hình.
- `hasSkills` chỉ đúng với tài khoản đã có kỹ năng. Một người từng bật player mode rồi **xoá hết kỹ năng**
  sẽ mất luôn card — khi đó họ cũng không thể bật lại, nên API vẫn đúng; chỉ là đường thoát duy nhất còn lại
  là thêm kỹ năng. Chưa xử lý, cần quyết định nếu anh thấy đáng.

---

## 27. Ảnh chụp từ máy bắt lỗi §26 + redesign dòng đánh giá

Anh chụp hai ảnh lúc 22:19: card "Ca trực" khi tắt chiếm gần hết màn hình, **mất tiêu đề và mất dòng trạng
thái** (chỉ còn cái chấm tròn), nút "Bật lại" nằm sát mép dưới và bị nút cài đặt nổi đè lên. §26 báo "đã
kiểm" vì kiểm bằng `innerText` — chữ có trong DOM nhưng bố cục thì sai. Bài học: đọc chữ không phải là nhìn
màn hình.

### Nguyên nhân

`Button` của hệ mặc định **`fullWidth = true`** (`src/components/ui/Button.tsx:70`). Đặt nó vào `headRow`
(một hàng `row` mà nhãn bên cạnh là `flex: 1, minWidth: 0`) nghĩa là đòi `width: 100%` trong hàng đó: Yoga
nén `headLabelWrap` về gần 0 → tiêu đề và nhãn trạng thái bị cắt tới mức vô hình, hàng tự cao lên, và vì
thẻ cao bất thường nên nút rơi đúng chỗ FAB cài đặt.

### Đã sửa

- Nút "Bật lại" ra khỏi `headRow`, đặt ở hàng riêng `pausedActionRow` (tự căn giữa, `minWidth: 160`,
  `fullWidth={false}`). "Quản lý" quay về đúng chỗ cũ ở góc phải hàng tiêu đề — cùng khuôn với thẻ khi đang
  bật, thay vì một đường link rời ở đáy thẻ.
- Dòng đánh giá dưới "Chưa có khoản nào chờ thanh toán": trước đây là một `Text` vô danh
  `4.8 ★ từ 24 đánh giá` (sao vẽ bằng ký tự, không bấm được). Nay là một hàng dẫn đường giống hàng số dư chờ:
  viên thuốc nền `goldSoft` chứa ngôi sao + điểm, nhãn "24 đánh giá từ người thuê", dấu › ở cuối, và bấm vào
  mở **`/ratings/{userId}`** của chính mình (không kèm `canReport` — báo bình luận là việc của người thuê).

### Kiểm chứng (iframe 375 px, account `demo.player1`, DB seed mới có 24 đánh giá)

```
thẻ khi BẬT   : cao 249 px; hàng đánh giá x=33 y=444 w=309 h=34, đọc "4.8 | 24 đánh giá từ người thuê"
tắt bằng API  : PATCH /users/me/player-mode {enabled:false} -> 200
thẻ khi TẮT   : cao 196 px (trên máy trước đây ~500 px toàn trắng)
                tiêu đề "Ca trực của bạn" 228x23 tại x=33  <- trước đây bị nén xuống 0
                "Đang ngừng nhận yêu cầu thuê" + "Quản lý" + dòng giải thích: đủ
                [Bật lại] 160x44, căn giữa (x=108 trên bề rộng 375)
bấm [Bật lại] : thẻ đầy đủ trở lại
bấm hàng đánh giá: /ratings/beef64a4-… -> "Chi tiết đánh giá · Tất cả · 5 sao"
trạng thái cuối = isPlayer bật lại như đầu; gate typecheck 0, lint 0
```

### GAP

- Vẫn chưa có ảnh từ máy cho bản sửa này — mới chắc ở số đo bố cục, không chắc ở cảm giác. Anh chụp lại giúp
  một tấm là đủ đóng mục này.
- FAB cài đặt nằm ở góc dưới phải mọi màn; khi thẻ ngắn lại thì hết đè, nhưng nó là nút nổi toàn cục — nếu
  sau này có hàng bấm được sát đáy thẻ, vẫn có thể đè. Chưa đổi.

---

## 28. #120, #119, #121 — đã thực hiện: gỡ bẫy DB dev, quét bàn phím, thêm CI cho FE và admin

### #120 — `npm run start:prod` không lên được nữa

`play-together-be/.env:8` vẫn ghi `PGLITE_DIR=.data/pgdata` trong khi API dev chạy nhờ env override trỏ
`.data/pgdata-fresh` (§26). nghĩa là lệnh thường dùng của anh gặp đúng cluster đã hỏng.
Đổi `.env` sang `.data/pgdata-fresh`, **không xoá** `.data/pgdata`, build lại (`nest build`, exit 0) rồi khởi
động đúng bằng `npm run start:prod`, không truyền env nào:

```
GET  /api/v2/health -> 200 {"status":"ok","database":"up"}
LOG  [Database] Connected using pglite (migrated)
POST /api/v2/auth/login (demo.player1) -> 200 accessToken
GET  /api/v2/users/me                  -> 200 isPlayer=true, rate=4.79, skills=2, price=80000
```

`.env` đang gitignored nên không có gì phải commit; `.env.example` giữ nguyên giá trị mặc định
`.data/pgdata` vì clone mới sẽ tự tạo thư mục đó.

### #119 — con số "16 màn" trước đây là sai, số thật là 5

Quét bằng grep trên toàn `src`, không nhớ từ ghi chú cũ: 23 màn có ô nhập, 15 màn tự bọc
`KeyboardAvoidingView`, 5 màn `(auth)` dùng KAV của `components/auth/AuthShell.tsx`.
**`softwareKeyboardLayoutMode` mặc định của Expo là `resize`** (xác nhận trong tài liệu Expo, không đoán) —
Android tự co cửa sổ khi bàn phím hiện, nên phần việc còn lại là iOS và là `keyboardShouldPersistTaps`.

Đã sửa:

| Màn | Thiếu gì | Đã làm |
| --- | --- | --- |
| `app/(tabs)/charity.tsx` | FlatList để mặc định `never`: chạm thẻ/✕ khi bàn phím đang mở bị nuốt nhịp đầu | thêm `keyboardShouldPersistTaps="handled"` |
| `app/filter.tsx` | hai ô "Từ / Đến" nằm **giữa** trang, iOS che là mất chỗ nhập | bọc ScrollView + footer vào KAV (`padding` cho riêng iOS) |
| `app/manage-hiring.tsx` | ô giá giữa trang, vừa thiếu KAV vừa thiếu persistTaps | bọc KAV + `keyboardShouldPersistTaps="handled"` |
| `components/auth/AuthShell.tsx` | `behavior="padding"` cứng → trên Android cộng dồn với `adjustResize`, thành khoảng trắng thừa | `Platform.OS === 'ios' ? 'padding' : undefined` |
| `app/personal-change-password.tsx` | y hệt AuthShell | như trên |

Không cần sửa (đã kiểm từng file): `app/search.tsx` và `app/search-history.tsx` — ô nhập nằm ở thanh trên
cùng nên bàn phím không che được, và list đã có `handled`. Quét lại lần cuối: file có `TextInput` mà thiếu
`keyboardShouldPersistTaps` giờ chỉ còn đúng `components/ui/TextField.tsx` (là định nghĩa component, chỗ
cuộn thuộc về màn dùng nó); mọi màn dùng `TextField` đều đã có `handled`, trừ 5 màn `(auth)` — chúng nhận từ
AuthShell.

Kiểm chứng bố cục (iframe cùng gốc 375×740, Metro web, tài khoản player):

```
/filter        : [Đặt lại][Áp dụng] y=674 h=50 -> đáy 724 < 740, footer vẫn ghim đúng;
                 hai ô giá ở y=819 (dưới vùng hiển thị, nằm trong ScrollView) -> đúng chỗ KAV lo trên iOS
/manage-hiring : render đủ 9 nút, "Chi phí mỗi giờ" + [Cập nhật] y=410
/charity       : danh sách quỹ render bình thường sau khi thêm prop
gate: tsc --noEmit exit 0, expo lint exit 0
```

### #121 — FE và admin trước đây không có pipeline nào

`ls */.github/workflows` chỉ trả về `play-together-be/.github/workflows/ci.yml`. Thêm
`play-together-fe/.github/workflows/ci.yml` (npm ci → `expo lint` → `tsc --noEmit` →
`npx expo export --platform web`) và `play-together-admin/.github/workflows/ci.yml`
(npm ci → `oxlint` → `tsc -b` → `vite build`), cùng khung với ci.yml của API: `permissions: contents: read`,
`concurrency` theo ref, Node 22, cache npm.
Chạy trước từng lệnh ở máy để không hứa hão: FE `expo export --platform web` exit 0 (đã bundle hết route),
admin `lint`/`typecheck`/`build` đều exit 0. `dist/` của FE và `dist` của admin đều gitignored nên artifact
CI không lẫn vào repo. `EXPO_PUBLIC_API_URL` thiếu trong CI cũng không làm hỏng bundle —
`src/api/endpoints.ts:2` có giá trị dự phòng `http://localhost:3000/api/v2`.

### GAP còn lại của đợt này

- Bàn phím là câu chuyện của thiết bị thật: ở đây không tạo được bàn phím ảo, nên mới chứng minh được **bố
  cục không vỡ** và **iOS sẽ có chỗ thở**, chưa chứng minh được cảm giác gõ trên Android. Cần anh bật bàn
  phím ở `/filter`, `/manage-hiring` và một màn đăng nhập rồi chụp.
  **→ ĐÃ XONG trên máy thật (04/11/2026, #122)**, qua Expo Go với Metro bind LAN (`exp://192.168.1.9:8081`) —
  anh xác nhận cả 5 màn mục #119, không còn khoảng trắng thừa ở `(auth)` và `/personal-change-password`.
  Lượt này cũng lộ một cái bẫy môi trường đáng nhớ: Metro cũ chỉ bind `[::]:8081` nên máy tính gọi
  `localhost:8081` vẫn 200 mà điện thoại thì treo vô thời hạn — symptom giống hệt "API chết".
- Hai file workflow mới nằm trong `.github/workflows/`: đã đẩy thành công (FE `5416abb`), GitHub Actions
  accept cả hai — không cần thêm scope nào cho PAT hiện tại.
- Vài màn có input vẫn dùng `ScrollView` thường mà không bọc KAV nhưng ô nhập của chúng nằm sát thanh tiêu
  đề (`/chat/[orderId]` dùng KAV riêng) — đã quét, không còn trường hợp giữa trang nào thiếu.

---

## 29. §23 nhóm B — đã thi công: 4 màn giờ/kỹ năng còn 2

Anh duyệt thứ tự **B → A → C → E → D → F** và mục tiêu §23 là **giảm màn, không giảm chức năng**.

### Giờ: `add-hour` + `update-hour` → `hour-form.tsx`

Hai màn là bản sao gần từng dòng (cùng `TimeRangeFields`, cùng kiểm tra khoảng giờ, cùng invalidate
`queryKeys.datings`), chỉ khác một động từ. Gộp thành `app/hour-form.tsx`:

- `/hour-form` = thêm mới (chọn ngày trong tuần, `POST /users/me/availability`).
- `/hour-form?id=…` = chỉnh sửa (`PATCH /users/me/availability/{id}`).
- **Hợp đồng route đổi thật**: `update-hours` trước đây truyền record thành 4 param chuỗi phẳng
  (`dayInWeek`, `fromHour`, `toHour` bằng string); nay chỉ còn `?id=` và màn hình tự đọc bản ghi từ
  `queryKeys.datings(userId)` — đúng key mà danh sách đang dùng nên không thêm request, và deep-link vào
  `?id=` cũng mở được.
- Thoát sau khi lưu: `router.back()` → `router.dismissTo('/update-hours')`, vì màn giờ có thể tới thẳng theo
  id chứ không chỉ từ danh sách.
- Không có đường dẫn cũ nào còn sót: `grep add-hour|update-hour` trong `src` trả về 0; hai màn không nằm
  trong `SHEET_ROUTES` nên không phải đăng ký lại animation.

### Kỹ năng: `add-skill` → lớp phủ trong `update-skills`

`components/skills/SkillPickerSheet.tsx` (Modal + lớp phủ, cùng khuôn `CancelOrderDialog`): lưới checkbox các
game chưa có, đếm "Đã chọn n/5", `POST /users/me/skills` rồi invalidate `gamesOfUser` và tự đóng. Giữ nguyên
guard `canDelete = !isPlayer || games.length > 1` và chặn `existing + selected > 5`.

**Một self-correction đáng ghi lại**: bản đầu mount Modal thường trực với `visible={pickerOpen}`. Đo trên Metro web
thì sau khi đóng, node `[aria-modal="true"]` vẫn nằm trong DOM ở 375×740, `pointer-events: none` nên không
chặn chạm (xóa kỹ năng vẫn chạy), nhưng nội dung cũ vẫn đọc được — với app gắn nhãn accessibility khắp nơi
thì đó là rác trong cây tiếp cận. Đổi thành mount có điều kiện `{pickerOpen ? <SkillPickerSheet/> : null}`;
đo lại: khi đóng còn **0** node `[aria-modal]` và **0** checkbox. Đổi lại là không còn animation khi đóng —
nhận thấy và chấp nhận.

### Hai lỗi phụ bắt được khi đi qua màn này

1. **Kỹ năng mới không bao giờ chọn được rank**: `SkillRow` bật SelectField theo `hasRank = rankId !== 'None'`
   và chỉ query danh sách rank khi `hasRank` — nên kỹ năng vừa thêm (rank rỗng) không có query, không có ô
   chọn, vĩnh viễn không set được rank. Đã đổi: luôn query rank theo `gameId`, ô chọn hiện khi game có rank.
2. **Tên game không nhất quán giữa hai nửa luồng**: lưới chọn hiển thị `displayName` ("Liên Quân Mobile") còn
   thẻ trong danh sách hiển thị `name` ("Arena of Valor"). Nay cả hai dùng `displayNameOfGame`.

### Kiểm chứng (Metro web + API dev trên `.data/pgdata-fresh`, tài khoản `demo.player1`)

```
đọc  : /update-hours -> 7 khung giờ, gộp đúng theo ngày
thêm : chọn Thứ 5, điền 07:00-08:00, [Thêm mới] -> toast "Thêm thành công",
       /update-hours hiện "Thứ 5: • 07:00 - 08:00"
sửa  : chạm hàng -> URL /hour-form?id=183f2f6f… (chỉ id), ô "Từ"=18:00 "Đến"=23:00 đọc từ cache
       đổi Đến=22:00 -> [Cập nhật] -> về /update-hours, hàng thành 18:00 - 22:00
       mở lại hàng đó: "Đến"=22:00 (cache đã được invalidate nên đọc số mới) -> đặt lại 23:00
xóa  : [thùng rác] hàng 07:00-08:00 -> "Xóa thành công", không còn hàng
kỹ năng: [+ ] -> sheet 25 game, "Đã chọn 0/5"; chọn Đế Chế -> "Đã chọn 1/5" -> [Thêm]
         -> thẻ "Arena of Valor" xuất hiện, sheet đóng, 0 node aria-modal
         -> [xóa] restore; Liên Quân (rank trống) nay hiện "Chọn rank", chọn Cao Thủ
         -> toast "Cập nhật thành công", thẻ hiển thị "Liên Quân Mobile / Cao Thủ"
dữ liệu dev sau cùng: 2 kỹ năng (Arena of Valor=Cao Thủ, League of Legends=Kim Cương),
       7 khung giờ như seed ban đầu
gate: tsc --noEmit exit 0, expo lint exit 0 (0 error, 0 warning)
```

Số dòng: 1.053 dòng (4 màn cũ + `update-hours`) → 1.116 dòng (2 màn + sheet + scaffold). Nhiều hơn ~3% vì
tách scaffold và header sheet, đổi lại ít hơn hai màn và hết một ngõ cụt.

### GAP

- Chưa bấm thử trên máy: animation mở sheet (fade) và chiều lùi của `dismissTo` trên stack native.
- `dismissTo('/update-hours')` là đích cứng: nếu sau này nhóm D/E đổi tên route thì phải sửa theo.
- Nhóm A (ví) chưa đụng; 4 dòng "(Sắp ra mắt)" sẽ ẩn trong đợt đó.

---

## 30. §23 nhóm A — đã thi công: 5 màn ví còn 2, hết tương tác giả

Anh duyệt "ẩn đi, chỉ để kênh đang chạy được" cho 4 dòng "(Sắp ra mắt)", và nhóm A làm ngay sau B.

### Cấu trúc mới

| Trước | Sau |
| --- | --- |
| `deposit-method` → `deposit` | `deposit` (số tiền + hint giới hạn + hàng "Ví MoMo" có logo) |
| `withdraw-amount` → `withdraw-method` → `withdraw` | `withdraw` (số tiền + số dư khả dụng + số điện thoại nhận) |

`deposit-method.tsx`, `withdraw-method.tsx`, `withdraw-amount.tsx` đã xoá. Component
`components/wallet/PaymentMethodRow.tsx` trở thành code chết nên xoá theo, cùng hai asset chỉ phục vụ dòng
"Sắp ra mắt": `assets/images/paypallogo.png`, `assets/images/onlinebankinglogo.png`. `momologo.png` vẫn dùng.
Không còn chỗ nào tham chiếu: `grep deposit-method|withdraw-method|withdraw-amount|PaymentMethodRow src` trả
về rỗng.

Điểm vào đổi theo: `personal.tsx` (Nạp/Rút), `send-hiring-request.tsx` (nhánh thiếu số dư) nay push thẳng
`/deposit` và `/withdraw`. Trong `_layout.tsx`, `SHEET_ROUTES` giảm từ 8 còn 5 tên — `deposit` và `withdraw`
vẫn là sheet nên giữ nguyên hiệu ứng trượt và vuốt xuống.

Giữ nguyên như §23 yêu cầu: guard "số dư đọc lỗi thì không cho rút" (kèm hàng chữ đỏ + nút Thử lại), kết
thúc rút bằng `replace('/transactions')` để không vuốt lùi lại một lệnh đã nộp, và body gọi API
(`moneyWithdraw` + `phoneNumberMomo`) đúng như màn cũ.

Rút gộp thêm một thứ màn cũ không có: nút ở trạng thái `loading` trong lúc gửi, vì giờ một màn duy nhất vừa
kiểm tra vừa POST.

### Kiểm chứng (Metro web + API dev, `demo.player1`, ví bắt đầu 0đ)

```
/deposit  : "Nạp tiền" + ô Số tiền + hint "Nạp tối thiểu 10.000đ, tối đa 50.000.000đ" + hàng "Ví MoMo" (img alt=MoMo)
            nhập 10.000 -> [Nạp tiền] -> toast "Nạp tiền thành công (thanh toán thử nghiệm)"
            GET /wallet -> availableBalance 10000
/withdraw : màn gộp đủ 4 khối (số tiền, số dư khả dụng 0đ, nhãn "Nhận qua MoMo", số điện thoại)
            bỏ trống  -> "Vui lòng nhập số tiền!"
            5.000     -> "Số tiền rút tối thiểu là 10.000đ"
            99.999.999-> "Số tiền rút tối đa là 50.000.000đ"
            10.000 khi số dư 0 -> "Số dư khả dụng không đủ!"
            10.000 + 0901234567 sau khi nạp -> "Rút tiền thành công" rồi tới /transactions
            GET /wallet -> availableBalance 0
            GET /wallet/withdrawals -> [{amount:10000, status:PENDING, provider:MOMO, note:"0901234567"}]
            POST /wallet/withdrawals/{id}/cancel -> CANCELLED, GET /wallet -> availableBalance 10000
gate: tsc --noEmit exit 0, expo lint exit 0
```

Số dư khả dụng 0 khiến nhánh "thiếu số điện thoại" không tới được — đó là thứ tự kiểm tra cố ý (tiền trước,
số sau), nhưng cũng có nghĩa là nhánh đó mới chỉ đọc code, chưa bấm thật.

### Hệ quả để lại trên DB dev (nói rõ, không giấu)

`demo.player1` hiện có 10.000đ và hai bản ghi test: một DEPOSIT mock, một WITHDRAWAL đã CANCELLED. Ledger
không xoá được nên đây là dữ liệu thật của lần kiểm này; reseed sẽ sạch.

### GAP

- Chưa chạy trên máy: chiều `replace('/transactions')` từ trong sheet và cảm giác khi sheet "Rút tiền" dài
  hơn (bốn khối thay vì hai).
- BE nhét số điện thoại MoMo vào cột `note` của withdrawals (code cũ, không đổi ở đây) — nếu sau này cần
  thống kê kênh rút thì nên có cột riêng.

---

## 31. §23 nhóm F — đã làm; nhóm C — tiền đề trong §23 sai, chưa thi công

### F · "Số dư khả dụng" thật ra là tiền chờ (đổi nhãn + chuyển hành động)

Đúng như §23 mô tả: màn `unactive-balance` mang tên "Số dư khả dụng" nhưng liệt kê tiền **chưa** về ví, còn
nút nhả tiền lại nằm ở tab Cá nhân. Đã làm theo hướng (a):

- `unactive-balance.tsx`: tiêu đề thành **"Tiền đang chờ về ví"**; thêm dòng tổng "Tổng đang chờ: X" và nút
  **"Thu hồi về số dư khả dụng"** (`POST /wallet/pending-credits/release` → invalidate `unActiveBalances` +
  `userBalance` + `refreshUser`); mỗi hàng ghi rõ "Về ví từ {dateActive}"; hết dữ liệu thì EmptyState nói
  thẳng tiền đó đến từ đâu.
- `personal.tsx`: nút icon "Cập nhật số dư" **không còn gọi release** nữa — nó chỉ tải lại số dư, đúng như
  nhãn. Hành động liên quan tiền giờ chỉ còn một chỗ, ở màn liệt kê tiền chờ.
- Hàng dưới số dư tách thành hai: "Khả dụng: X" (chữ thường, không bấm) và **"Đang chờ về ví: Y ›"** (mở
  màn tiền chờ). Trước đây hàng "Khả dụng: X" dẫn vào màn tiền chờ — tên một đường, nội dung một nẻo.
  `pendingBalance` = `balance - activeBalance` vì `toLegacyUserBalance` cộng hai thứ lại trong `balance`.

Kiểm chứng (Metro web, `demo.player1`):

```
/unactive-balance : "Tiền đang chờ về ví" + "Không có khoản nào chờ" + dòng giải thích
/personal         : hàng nút đọc "Xem tiền đang chờ về ví, 0 đ"; "Cập nhật số dư" vẫn còn nhưng không còn POST release
gate: tsc --noEmit 0, expo lint 0
```

**Nhánh "Thu hồi" chưa bấm được thật**: DB dev seed mới không có khoản chờ nào — cả `demo.player1..6` đều
`pendingBalance: 0`, vì tiền chờ sinh ra từ vòng đời đơn hàng đã hoàn thành. Nút chỉ hiện khi có hàng, nên
muốn test thật phải chạy một đơn từ thuê tới hết cửa sổ khiếu nại. Ghi rõ ở đây để không nhầm là đã kiểm.

### C · Phát hiện khi đọc code: hai màn tố cáo **không** phải bản sao

§23 ghi `report-comment` và `report/[orderId]` "khác duy nhất endpoint". Đọc lại từng dòng thì không đúng:

| | `report-comment.tsx` | `report/[orderId].tsx` |
| --- | --- | --- |
| Nội dung | 4 lý do cố định + "Lý do khác" + ô 500 ký tự | thẻ người bị tố (Avatar + tên) + một ô tự do 1.000 ký tự, **không có lý do cố định** |
| Endpoint | `POST /ratings/{id}/flag` | `POST /orders/{id}/reports` |
| Kết thúc | `router.back()` (mở từ `ratings/[userId]`) | `router.replace('/(tabs)/history')` |

Gộp thành một màn thì hoặc phải bịa ra danh sách lý do cho tố cáo đơn hàng (đổi sản phẩm, không phải sửa UI),
hoặc phải nhét hai chế độ `if (mode)` xuyên từ layout tới submit tới điều hướng — thứ mà chính §23 cảnh báo
ở nhóm G. Nên **tạm gác nhóm C phần report-comment**, chờ anh quyết: (1) để hai màn như hiện tại, hay
(2) cho tố cáo đơn hàng một danh sách lý do riêng rồi mới gộp. Phần C2 (`rating/[orderId]` → lớp phủ trong
`end-order`) vẫn còn hiệu lực (đúng một inbound, đã kiểm lại bằng grep) nhưng động vào màn có polling 10s và
hai effect tự điều hướng — để sau khi đã test máy.

### Sự cố môi trường đáng nhớ

Metro lại hỏng module graph y như §26 ("Requiring unknown module \"892\"", `/unactive-balance` trả 500 kèm
`_expo/static-tmp/_error.bundle`). Chữa như cũ: giết tiến trình đang giữ cổng 8081 rồi khởi động lại với
`--clear`. Đây là lần thứ hai, nên coi `--clear` là bước đầu tiên khi Metro trả 500 trên một route lẻ.

## 32. §23 nhóm E — đã thi công: màn phản hồi chi tiết thành accordion trong danh sách

§23 liệt kê cặp `feedback` (danh sách) + `feedback/[feedbackId]` (chi tiết) là hai màn cho một việc đọc.
Gộp theo hướng bỏ hẳn màn thứ hai:

- `feedback.tsx`: mỗi hàng là một `FeedbackRow` tự quản trạng thái `expanded`. Phần thân (thể loại + nội dung)
  chỉ tồn tại ở endpoint chi tiết nên gọi `useQuery` với `enabled: expanded` — lần mở đầu mới
  `GET /feedback/{id}`, đóng/mở lại không gọi nữa vì cache dùng chung `queryKeys.feedbackDetail(id)`.
  Hàng có `accessibilityRole="button"`, `accessibilityState={{ expanded }}`, `accessibilityHint` đổi theo
  ("Xem nội dung" / "Ẩn nội dung"), và `renderItem` thay vì render inline trong `keyExtractor`.
- Ba trạng thái trong vùng mở: đang tải ("Đang tải nội dung…"), lỗi (một nút "Không tải được nội dung. Nhấn
  để thử lại." gọi `refetch` — không im lặng hiện ô trống), và thành công.
- `statusOf` và `TYPE_LABELS` trước bị nhân bản ở cả hai màn; giờ chỉ còn một bản trong `feedback.tsx`.
- Xoá `src/app/feedback/[feedbackId].tsx`. Deep-link `'/feedback/me': '/feedback'` trong
  `navigation/deepLink.ts:30` vẫn đúng (nó trỏ vào danh sách, chưa từng trỏ vào màn chi tiết), nên không
  đường nào gãy. Không còn màn nào `push` tới `/feedback/<id>` — kiểm bằng grep, chỉ service và query key nhắc.

Kiểm chứng (Metro web, `demo.player1`):

```
DB dev không có phản hồi nào (/feedback/mine trả total:0) → tạo một hàng qua chính endpoint màn dùng:
POST /api/v2/feedback {title:"Kiem tra accordion", type:"FEATURE"} → 201, id c0733063-…, status PENDING

/feedback : "Trung tâm phản hồi" + hàng "Kiem tra accordion / Chờ xét duyệt / 23:58 02/11/2026"
tap lần 1 : nội dung hiện ngay trong hàng — "Thể loại" → "Đề xuất tính năng hoặc thêm mới danh mục game"
            + "Nội dung" → đúng message đã ghi
tap lần 2 : sập về đúng trạng thái ban đầu, không còn chữ "Nội dung"
network   : GET /api/v2/feedback/c0733063-… [200] chỉ xuất hiện SAU cú tap đầu;
            /feedback/mine gọi 2 lần (lượt tải + lượt focus invalidate), không có request nào khác
gate      : tsc --noEmit 0, expo lint 0 (không warning nào)
```

Còn lại của §23: nhóm D (search + search-history → một màn, nên test máy trước), nhóm C chờ anh quyết phần
`report-comment` (§31), C2 gác tới sau khi test máy.

## 33. §23 nhóm D — đã thi công: hai màn tìm kiếm còn một

`search-history.tsx` (điểm vào, có ô nhập) và `search.tsx` (kết quả, cũng có ô nhập) giờ là một màn
`search.tsx`:

- **Chưa có từ khoá** → ba mục "Tìm kiếm gần đây / Top tìm kiếm / Các tựa game hot" (thay cho một
  `EmptyState` tĩnh trước đây). **Có từ khoá** → danh sách kết quả như cũ. Nút "Xoá từ khoá" ngay trong ô
  nhập đưa người dùng về đúng phần gợi ý, không cần quay lại màn trước.
- Gợi ý không còn `push('/search?q=…')` mà gọi `applyKeyword(value)` tại chỗ: set `input` + `keyword`,
  reset `pageSize` về 20 và `record(value)` (`POST /search-history`). `lastRecorded` vẫn chặn ghi trùng.
  Nhánh `?q=` vẫn giữ cho deep-link sau này, dù hiện không còn ai push.
- Toàn bộ phần gợi ý nằm trong `src/components/search/SearchSuggestions.tsx` **mới**, và chỉ mount khi
  chưa có từ khoá — ba request (`/search-history`, `/search-history/hot`, `/games/most-favorited`) không
  còn chạy phía sau một danh sách kết quả. Hai toast xoá lịch sử và confirm "Xoá tất cả" giữ nguyên như
  màn cũ.
- `HomeAppBar.tsx:31` chuyển điểm vào sang `/search`; `search-history.tsx` bị xoá. Không có `SHEET_ROUTES`,
  `deepLink` hay route nào khác nhắc tới tên cũ (kiểm bằng grep — chỉ còn chuỗi endpoint và query key).
- Ô nhập giờ `autoFocus` khi mở màn không kèm `?q=` — đúng hành vi màn lịch sử cũ, vì `/search` đã là
  điểm vào.

Kiểm chứng (Metro web, `demo.player1`, hết một vòng):

```
Home → bấm ô tìm kiếm : path /search, hiện đủ 3 mục, "Chưa có tìm kiếm nào"
network khi mới mở    : GET /search-history?limit=5, /search-history/hot, /games/most-favorited (3 request)
bấm chip "Valorant"   : input lên "Valorant", GET /users?q=Valorant&pageSize=20 [200] + POST /search-history [201]
                       → "Không có dữ liệu" (dev DB không có player nào game này — đúng, không phải lỗi)
bấm "Xoá từ khoá"     : về lại gợi ý, "Valorant" đã xuất hiện ở cả "gần đây" và "Top"
bấm X một hàng        : hàng mất ở cả hai mục (query key top là con của ['search-history'] nên invalidate kèm)
"Xoá tất cả"          : toast "Đã xoá lịch sử tìm kiếm", hai mục trở về "Chưa có…"
gõ tay + Enter        : submit chạy, "a" trả về 3 card player thật (Nguyễn Minh Anh / Phạm Gia Huy / …)
/filter               : mở được, chọn chữ cái N + Áp dụng → về /search, giữ từ khoá "a",
                       hiện "Đang áp dụng bộ lọc" và danh sách còn đúng 1 card
gate                  : tsc --noEmit 0, expo lint 0
dọn dev data          : lịch sử tìm kiếm đã xoá sạch, bộ lọc đã reset về mặc định
```

Điểm để ý, chưa sửa: thanh "Đang áp dụng bộ lọc" vẫn hiển thị khi từ khoá trống (gợi ý hiện bên dưới).
Trước khi gộp cũng vậy, và nó nói thật trạng thái đang lưu trong `useDiscoveryFilter` — bộ lọc sẽ áp cho
lần tìm kế tiếp — nên giữ. Nếu anh muốn nó biến mất khi chưa có kết quả thì một điều kiện là xong.

Còn lại của §23: nhóm C (`report-comment`, chờ anh quyết), C2 (gác tới sau khi test máy), và phần G
`history/[orderId]` gọi trùng — chỉ là sửa code, không đổi điều hướng.

## 34. Ba ô soạn thảo cho phép dài hơn mức API chấp nhận (đã sửa)

Khi đọc code để quyết nhóm C, tình cờ thấy `report/[orderId].tsx` giới hạn lời nhắn 1.000 ký tự trong khi
`reports.message` là `varchar(500)`. Quét lại **toàn bộ** `maxLength` của FE và admin đối chiếu từng DTO phía
API, kết quả:

| Màn | Ô | FE trước | Giới hạn thật (`@MaxLength` + cột DB) |
| --- | --- | --- | --- |
| `send-hiring-request.tsx:365` | Lời nhắn cho người chơi | 1000 | `CreateOrderDto.message` 500, `orders.message` varchar(500) |
| `rating/[orderId].tsx:184` | Nội dung đánh giá | 1000 | `CreateRatingDto.comment` 500, `ratings.comment` varchar(500) |
| `report/[orderId].tsx:126` | Lý do tố cáo | 1000 | `CreateReportDto.message` 500, `reports.message` varchar(500) |

Cả ba cùng một kiểu lỗi: người dùng viết thoải mái, bấm Gửi, và **nhận HTTP 400** ngay sau khi đã soạn xong
— đúng loại "nút bấm không ăn" mà §17 đã kết luận là do app bỏ mất lý do server trả về.

- Sửa phía client, không đổi schema: ba màn giờ dùng chung `MAX_MESSAGE_LENGTH = 500`
  (`src/lib/validators.ts`), có chú thích ghi rõ nó soi gương `varchar(500)` + DTO để lần sau không lệch nữa.
- Những chỗ còn lại đều **chặt hơn** API chứ không lỏng hơn, nên không thể gây lỗi: chat 500/2000,
  feedback 1000/2000, tên người dùng 30/100. Các màn khớp sẵn: `report-comment` 500/500, bio 1000/text(DTO 1000),
  lời nhắn từ thiện 1000/varchar(1000).
- **Admin web không lệch nào**: mọi `maxLength` (`ReportsPage` 1000 = `ResolveReportDto.note` 1000,
  `FeedbackPage` 2000, `BroadcastPage` 200/1000, `CatalogPage` 100/50/1000, `CharitiesPage` 150/5000/200/255/20/1000,
  `WithdrawalsPage` 255) đều bằng hoặc thấp hơn DTO.

Kiểm chứng:

```
boundary thật qua HTTP (token demo.player1, POST /orders/<uuid>/reports):
  message 501 ký tự → 400 validation_failed, details.errors =
    ["message must be shorter than or equal to 500 characters"]
  message 500 ký tự → 404 order_not_found (tức đã qua validation, vào tới service)
render: /send-hiring-request?playerId=b7ad586f-… → TEXTAREA "Lời nhắn cho người chơi" maxlength="500"
gate: tsc --noEmit 0, expo lint 0
```

Lưu ý môi trường: ngay sau khi sửa, Metro trả 500 cho route `/send-hiring-request` (y như §26 và §31), nhưng
`src/app/send-hiring-request.bundle` vẫn request được 200 và tải lại trang là màn hình render bình thường —
lần này **không** cần khởi động lại Metro, không có gì hỏng thật.

## 35. §23 nhóm C — đã thi công: một màn tố cáo, tố cáo đơn hàng có bộ lý do riêng

Anh quyết: gộp một màn và thêm bộ lý do riêng cho tố cáo đơn hàng. §31 từng chứng minh hai màn **không** phải
bản sao, nên lần này không nhập khung mà viết lại thành **một màn có hai chế độ thật sự**:

- Route duy nhất `src/app/report/[target].tsx`, phân nhánh bằng `?kind=order|rating` (mặc định `order`).
  Xoá `report/[orderId].tsx` và `report-comment.tsx`; bốn điểm vào đổi theo:
  `end-order/[orderId]` (2 chỗ), `rating/[orderId]`, `ratings/[userId]`.
- Nhánh order: vẫn nạp `GET /orders/:id` để lấy thẻ người bị tố (giữ nguyên như màn cũ, không đổi hình dạng
  đã duyệt trên điện thoại), gửi `POST /orders/:id/reports`, thoát bằng `replace('/(tabs)/history')`.
- Nhánh rating: **không gọi API nào** (query `enabled: kind === 'order'`), danh tính người đánh giá đi qua
  param `name`/`avatar` như màn cũ vẫn làm, gửi `POST /ratings/:id/flag`, thoát bằng `back()`.
- Bộ lý do. Đơn hàng (5 + "Lý do khác"): *Không vào phiên đã hẹn · Bỏ giữa phiên · Lời lẽ xúc phạm, thái độ
  không phù hợp · Yêu cầu thanh toán ngoài ứng dụng · Nội dung chơi không đúng như đã thoả thuận*.
  Đánh giá: giữ 4 lý do cũ. **Nội dung gửi đi chính là nhãn đã chọn**, nên `reports.message` giờ đọc được
  ngay thay vì một đoạn văn tự do — kiểm: hàng trong `/reports/mine` có `message: "Bỏ giữa phiên"`.
- Ô "Lý do khác" đổi từ `TextInput` tự vẽ sang `TextField variant="floating" multiline` — cùng kiểu hộp với
  phần còn lại của app (mọi input đã ở `floating` từ pha redesign). Lý do đổi: anh gửi ảnh chụp và báo
  "ô text không đều". Đo bằng chứng: hộp cũ `marginLeft: 30` mà không có `marginRight`, nên trong card rộng
  343px nó lệch **30px trái / 0px phải** so với mép trong, và mép phải còn **trôi ra ngoài** dải tint của hàng
  chọn. Hộp mới đo trong viewport 375px: hàng chọn `33→342`, hộp `33→342` (thẳng hàng tuyệt đối), cao 118px,
  `maxlength="500"` khớp `varchar(500)` ở §34.

Kiểm chứng (Metro web, đơn thật `fd8fe34f…` do tôi tạo ở dưới, rating thật `d4976d1d…`):

```
nhánh order  : /report/fd8fe34f…?kind=order → header "LH / Lê Thu Hà" + 5 lý do mới
               chọn "Bỏ giữa phiên" → hàng đổi nền primaryFaint → Gửi
               → POST /orders/fd8fe34f…/reports 201, toast "Tố cáo thành công!", path trở thành /history
               → GET /reports/mine?scope=submitted: message="Bỏ giữa phiên", status PENDING
nhánh rating : mở bằng đường thật: /ratings/beef64a4…?canReport=1 → bấm "Báo cáo bình luận của Lê Thu Hà"
               → /report/d4976d1d… (không còn /report-comment), header "LH / Lê Thu Hà", 4 lý do cũ
               → không có request đơn nào → chọn "Quảng cáo trái phép" → Gửi
               → POST /ratings/d4976d1d…/flag 200, toast "Báo cáo thành công", quay lại đúng danh sách đánh giá
chặn gửi     : chưa chọn gì mà bấm Gửi → toast "Vui lòng chọn lý do!", không phát request nào
gate         : tsc --noEmit 0, expo lint 0
```

**Phát hiện khi test — một quy tắc BE chưa từng được nói cho người dùng biết**: `ratings.service.ts:75-86` —
đã tố cáo đối tác của một đơn thì **không còn đánh giá đơn đó được nữa** (409). App dịch `conflict` thành
"Thao tác xung đột với trạng thái hiện tại." (`errorMessages.ts:82`), tức người dùng báo xong rồi quay lại
chấm sao thì gặp một câu vô nghĩa. Đây là quyết định sản phẩm (nên hay không cho chấm sao sau khi tố cáo)
cộng với một mã lỗi riêng, nên tôi ghi lại chờ anh quyết, không tự đổi.

**Dữ liệu dev đã động tới** (cần đơn thật để test): nạp mock 500.000đ cho `demo.player1` (`SUCCEEDED`),
tắt rồi **bật lại** chế độ người chơi của tài khoản này (`isPlayer=true` như cũ), thuê Lê Thu Hà (`demo.player3`)
1 giờ, player nhận, hirer `end-early` → đơn `ENDED_EARLY`, một rating 4★, một report PENDING, một flag.
Sổ cái không hoàn tác được; các hàng đơn/đánh giá/tố cáo này ở lại DB dev và thực ra có ích cho các lượt test
sau (trước đây DB seed không có đơn nào).

Nhóm C2 (`rating/[orderId]` → lớp phủ trong `end-order`) vẫn gác: màn đó có polling 10s và hai effect tự
điều hướng, để sau lượt test điện thoại. §23 coi như khép ở nhóm A–F.

## 36. Cắm máy test cho FE và admin — và một nhãn phí làm tròn lệch tiền

BE đã có jest từ đầu (`src/common/money/money.spec.ts`…); hai repo client thì không, cổng duy nhất là
typecheck + lint + build. Từ mục này mỗi repo client đều có test và test đó chạy trong CI.

**FE — jest-expo, làm theo tài liệu chính thức của Expo SDK 58.** AGENTS.md bắt buộc đọc docs trước khi đụng
API Expo, nên tôi lấy `https://docs.expo.dev/llms.txt` rồi trang `develop/unit-testing.md`; trang
`versions/v58.0.0/sdk/test/` mà trí nhớ model hay đoán **404 — không tồn tại**. Lệnh cài đúng của Expo:

```
npx expo install jest-expo jest @types/jest --dev   → jest ~30.5.2, jest-expo ~58.0.13
```

(SDK 58 trở đi yêu cầu Jest 30 — nằm trong chính trang docs đó, không phải mặc định từ training data.)

- `jest.config.js`: `preset: 'jest-expo'` và `moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1' }`.
  jest-expo tự khai báo `resolver` riêng, nên tôi không *đoán* là `moduleNameMapper` vẫn ăn — bằng chứng là
  4 file test đều import bằng `@/…` và pass hết.
- `tsconfig.json` thêm `"types": ["jest", "node"]`. Thiếu nó `tsc --noEmit` báo `Cannot find name 'describe'`
  trong khi jest vẫn chạy bình thường — hai bộ máy đọc cấu hình khác nhau.
- script: `test` = `jest --runInBand`, `test:watch` = `jest --watch`.

Bốn file, 35 test, toàn hàm thuần (chưa test render — xem "chưa làm" cuối mục):

- `src/theme/__tests__/responsive.spec.ts` (8) — biên breakpoint, gutter, clamp của `fontScale`; **ca bug §25**
  `gridItemWidth(375, 160, 10) === 166` (nếu làm tròn lên 167.67 thì hàng 343.33 chui trong hộp 343px và lưới
  gộp thành 1 cột), cộng một bất biến quét 8 chiều rộng màn (320→428) × 3 cặp `(minCard, gap)`:
  `cols * item + (cols - 1) * gap <= width - 2 * gutter`.
- `src/api/v2/__tests__/mappers.spec.ts` (8) — đẳng thức tiền `available + pending = total` và phép trừ khôi
  phục `pending`; ma nhãn trạng thái đơn, gồm rẻn `closeReason` và `endedEarlyBy` theo bên; `toWireOrderStatus`
  phủ mọi trạng thái trong union.
- `src/lib/__tests__/format.spec.ts` (10) — `formatVnd` (phân tách hàng nghìn, làm tròn, số âm, null),
  `formatNumber`, `parseAmount`, thứ tự `dd/mm/yyyy`, `isEmptyDate` bắt sentinel `0001-01-01` của Postgres,
  `secondsBetween`, `addHours`.
- `src/lib/__tests__/validators.spec.ts` (9) — email, ma trận mật khẩu + `passwordRules` từng yếu tố, xác nhận
  mật khẩu, OTP, và `MAX_MESSAGE_LENGTH === 500` — chốt §34 lại thành test để lần sau không ai nâng cap client
  lên 1000 một lần nữa.

**Admin — vitest, chọn phiên bản có chủ đích.** Repo đang Vite 8, nên trước khi cài tôi kiểm peer range:
`vitest@5.1.0` khai báo `vite: '^6.4.0 || ^7.0.0 || ^8.0.0'` (cài xong xác nhận lại vite 8.3.5 + vitest 5.1.0,
`npm ci` sạch 0 lỗ hổng). `vite.config.ts` đổi sang `defineConfig` nhập từ `vitest/config` và thêm
`test: { environment: 'node', include: ['src/**/*.spec.ts'] }`. Specs import thẳng
`{ describe, expect, it } from 'vitest'` nên không phải sửa tsconfig; `tsc -b` vẫn biên dịch cả hai file test
(chứng minh bằng `tsc -p tsconfig.app.json --listFiles`, không phải đoán).

- `src/auth/__tests__/permissions.spec.ts` (12) — `sectionOf` gộp route chi tiết về section cha; ma trận phân
  quyền MODERATOR / FINANCE / SUPER_ADMIN (wildcard) và tài khoản **không** tier; `canOpenPath`,
  `canOpenDossier`, `tierOf`, và `visibleSections` (MODERATOR 7 section, FINANCE 6, không tier chỉ `/`).
- `src/lib/__tests__/labels.spec.ts` (18) — không map nào được để nhãn trùng chính cái mã (đúng quy tắc
  "màn hình vận hành không lộ mã/UUID"; xem feedback của anh về plain-language UI), phủ toàn bộ
  `CLOSE_REASON` và `LEDGER_TYPE` lấy theo enum BE ghi cứng để phát hiện drift khi BE thêm mã, nhãn đủ vòng đời
  đơn, fallback của `labelOf`/`statusMeta`/`statusLabel`.

**Một lỗi thật, tìm ra nhờ viết test** (không phải test sai): `formatBasisPoints` bo về 1 chữ số thập phân,
trong khi `orders.fee_bps` là số nguyên 0..5000 (`CONSTRAINT "orders_fee_check"` ở
`play-together-be/src/db/migrations/0000_init.sql:174`) và `feeOf = Math.round(gross * feeBps / 10000)`
(`src/common/money/money.ts:19`). Nghĩa là phí 25 bps hiện thành "0,3%" và 123 bps hiện thành "1,2%" — một tỷ
lệ mà người vận hành không cộng ra số tiền kèm đơn, ở cả `OrdersPage`, `OrderDetailPage`, `UserDetailPage` và
`DashboardPage`. Sửa `maximumFractionDigits: 1 → 2`: số nguyên chia 100 không bao giờ cần quá 2 chữ số nên mọi
giá trị hợp lệ đều hiển thị chính xác, số chẵn vẫn ra "10%" không thừa đuôi. Kèm test quét trọn 0..5000 khẳng
định đọc nhãn ngược lại vẫn ra đúng bps đã lưu.

**CI**: `- run: npm test` đứng trước bước bundle/build ở cả hai workflow (`play-together-fe/.github/workflows/ci.yml:38`,
`play-together-admin/.github/workflows/ci.yml:32`); tên job đổi thành "Lint, typecheck, test and bundle/build".

Cổng, chạy lại sau khi cài sạch từ lockfile:

```
FE     jest 35/35 · tsc --noEmit 0 · expo lint 0 · expo export --platform web 0
admin  vitest 30/30 · tsc -b 0 · oxlint 0 (43 files) · vite build 0
cả hai npm ci exit 0 từ lockfile đã commit — đúng lệnh CI dùng
```

**Sự cố đáng ghi lại để lần sau không lặp**: tôi định *xác minh lockfile mà không xoá tree* nên chạy
`npm ci --dry-run`. Hoá ra npm xoá `node_modules` **trước** rồi mới bỏ qua bước cài — cả FE và admin thành thư
mục rỗng, và ba tiến trình dev đang chạy trên đó (API, Metro, Vite) chết theo. Khôi phục bằng `npm ci` thật
(exit 0 — cũng chính là bằng chứng CI cài được từ lockfile đã commit). API đã bật lại và kiểm:
`GET http://localhost:3000/api/v2/health` → 200, `/docs` → 200. Muốn kiểm lockfile thì chạy thẳng `npm ci`;
`--dry-run` không phải probe an toàn. Metro và Vite dev **chưa** bật lại (terminal của anh), nên trước khi test
máy #122 anh chạy lại `npx expo start` (và `npm run dev` cho admin nếu cần).
*Cập nhật sau đó*: Vite admin đã bật lại trên 5174 (200, đăng nhập admin + moderator đều thông); Metro web
8081 tôi bật bằng tiến trình riêng, kèm `EXPO_PUBLIC_API_URL` của tiến trình đó — xem §37.

**Chưa làm, có chủ đích**: chưa có test render component — muốn bắt được kiểu lỗi lệch ô input hay bàn phím thì
cần thêm `@testing-library/react-native` + jest setup, và chỉ đáng làm khi một màn đổi hình dạng; chưa đặt
coverage threshold; test chạy trên Node nên không đụng code native, nghĩa là các defect device-only (§25, #122)
máy test không tự phát hiện.

## 37. #6 — `player-list` chuyển từ bộ nhớ tạm sang `?source=`

Màn "Xem tất cả" của năm rail khám phá trước đây nhận dữ liệu qua một module singleton
(`playerListDraft.set(...)` rồi `take()`). §14/#117 mới vá được nửa đường: `take()` đọc-một-lần nên không còn
render nhầm danh sách của rail khác, nhưng bản thân route vẫn **không tự chứa gì cả** — sao chép link, mở từ
thông báo, hay app bị nạp lại đều ra "Danh sách chỉ hiện khi mở từ một mục trên trang Chủ", kể cả khi API đang
trả về đúng dữ liệu đó.

Giờ URL là nguồn sự thật: `/player-list?source=new-accounts|same-hobbies|top-rating|recent-orders|recommend`.

- **Một bảng định nghĩa, dùng cho cả rail và màn danh sách** — `src/features/discovery/playerListSources.ts`:
  mỗi nguồn có `title`, `queryKey` và `queryFn` của chính nó. Home không còn tự viết năm `useQuery` nữa mà gọi
  `usePlayerListSource('new-accounts', …)`…, nên rail và màn "Xem tất cả" **buộc** phải trùng key và trùng
  fetcher; trước đây hai chỗ này là hai đoạn code giống nhau và rất dễ sửa lệch một bên.
- `recommend` vẫn cần năm id của rail top-rating làm seed, cả về key (`home.recommend(userId, ids)`) lẫn giới
  hạn `pageSize`. Màn danh sách dựng lại đúng chuỗi đó: chạy `top-rating` (enabled chỉ khi `source=recommend`)
  rồi mới gọi `recommendations/players?limit=<ids.length>` — và giữ nguyên điều kiện của Home là phải đăng
  nhập + đủ 5 id, để link mở lạnh không phát một request mà Home không bao giờ phát.
- Tiêu đề lấy từ bảng, nên nhãn rail và nhãn màn danh sách không thể lệch nhau nữa.
- Xoá `playerListDraft.ts`; `README.md` ghi lại tham số của route.
- **5 test mới** (`src/features/discovery/__tests__/playerListSources.spec.ts`, nâng tổng số lên 40): key của
  từng nguồn phải `toEqual` đúng `queryKeys.home.*` (một key nào đó lệch đi là mở "Xem tất cả" âm thầm
  thành request thứ hai cho cùng danh sách, và hai lần gọi có thể trả kết quả khác nhau), `noSource` không
  được trùng với nguồn thật,
  và `resolvePlayerListSource` từ chối `?source=bogus`, hoa/thường sai, cũng như dạng mảng `?source=a&source=b`.

Kiểm chứng (Metro web, tài khoản `demo.hirer1`, API `192.168.1.9:3000`):

```
từ rail      : Home → "Xem tất cả" (rail đầu) → URL /player-list?source=new-accounts, tiêu đề
               "Các người chơi mới", 5 thẻ người chơi
mở lạnh      : nạp lại hẳn trang ở /player-list?source=top-rating (JS context mới, không còn gì trong bộ nhớ)
               → tiêu đề "Top người chơi ưa thích" + 5 thẻ  〔trước đây: EmptyState "chỉ hiện khi mở từ trang Chủ"〕
link lạ      : /player-list?source=bogus → "Danh sách người chơi" + hướng dẫn mở từ trang Chủ, không gọi API
               nào cho danh sách
chuỗi recommend: /player-list?source=recommend (mở lạnh) → network cho thấy đúng hai request theo thứ tự
               GET /users?sort=rating&pageSize=5 → GET /recommendations/players?limit=5; tiêu đề
               "Đề xuất cho bạn"; tài khoản này không có đề xuất nào nên màn hiện empty state thật, không treo
quay lại     : nút "Quay lại" từ màn danh sách → /home
gate         : tsc --noEmit 0 · expo lint 0 · jest 40/40
```

**Phát hiện ngoài lề, chặn cả lượt test điện thoại #122**: máy đã đổi IP LAN — `192.168.1.8` → **`192.168.1.9`**
(`ipconfig`), trong khi `play-together-fe/.env` vẫn ghi `EXPO_PUBLIC_API_URL=http://192.168.1.8:3000/api/v2`.
Hệ quả đo được: mọi request từ app treo bất định (`POST /auth/login` và các `OPTIONS` ở trạng thái pending mãi),
không phải lỗi đăng nhập. API vẫn healthy trên `localhost:3000` và trên IP mới (`GET http://192.168.1.9:3000/api/v2/health`
→ 200). Tôi **không** sửa `.env` của anh (file cấu hình local, và một IP hay đổi như thế này thì nên đặt IP tĩnh
cho máy); toàn bộ kiểm chứng ở trên chạy với Metro mà tôi bật bằng `EXPO_PUBLIC_API_URL=http://192.168.1.9:3000/api/v2`
trên tiến trình đó. Anh chỉ cần đổi một dòng trong `.env` rồi `npx expo start` lại là điện thoại dùng được tiếp.

## 38. #8 — số liệu hoàn thành theo ngày và theo tháng (endpoint mới + biểu đồ trong app)

Màn Thống kê trước đây chỉ có hai con số tiền "Theo ngày / Theo tháng" lấy từ `wallet/earnings/summary` —
không có diễn biến, và phíaBE chưa từng bucket theo lịch nào (`grep date_trunc` = 0 kết quả). Thêm cả vòng:
một endpoint time-series và một biểu đồ thật trong app.

**Endpoint**: `GET /api/v2/users/me/session-stats?granularity=day|month&buckets=1..62` (mặc định `day` / 14),
trả `SessionStatsView` với `buckets[]` **oldest first**, mỗi ô gồm `bucket` (`2026-11-03` hoặc `2026-11`),
`startsAt`, `playerSessions`, `playerHours`, `earned`, `hirerSessions`, `spent`. Đặt ở `users` cạnh `me` vì
`StatsService` (users module) đã là nơi mọi con số công khai của user được suy ra từ bảng gốc.

- **"Hoàn thành" không tự định nghĩa lại**: dùng thẳng `FINISHED_ORDER_STATUSES` (`SETTLED`, `ENDED_EARLY`) từ
  `src/db/schema/orders.ts:40` — chính file đó ghi rõ các tập trạng thái "used to be re-typed inside the orders,
  ratings, reports and stats services, and the copies were plain string arrays", nên viết thêm một mảng thứ năm
  là tự tái tạo cái bug cũ.
- **Tiền**: `earned` = `playerPayout` (đã trừ phí nền tảng), `spent` = `totalPrice`. Hai series đi kèm
  `playerSessions/playerHours/hirerSessions` nên một lần đọc trả lời được cả vai trò Player lẫn Hirer; một người
  có thể giữ cả hai vai, app không phải đoán.
- **Một lần đọc, không N query**: `or(player_id = me, hirer_id = me)` + `status in finished` +
  `endedAt >= đầu cửa sổ`, rồi gom vào từng mốc ở JS. Không dùng `date_trunc(..., 'Asia/Ho_Chi_Minh')` vì DB dev là PGlite
  còn production là Postgres thật — hai bên không hứa như nhau về build ICU có những tên múi giờ nào; còn phép
  chia mốc lịch đã có sẵn trong `src/common/time` (`startOfDayInZone`) và có test.
- Helper mới trong `src/common/time/time.ts`: `startOfMonthInZone` và `periodStartsInZone`. Mỗi mốc được suy
  ra từ một điểm nằm *giữa* kỳ thay vì trừ một khoảng cố định khỏi biên — vì một kỳ không còn là số
  millisecond cố định khi múi giờ có DST, và tháng phải thăm giữa tháng cũng vì lý do đó.
- Không có phiên nào bị âm thầm bỏ: cả hai bước chuyển sang trạng thái khép đều đóng dấu `endedAt`
  (`orders.service.ts:611` cho `ENDED_EARLY`, `:930` cho `SETTLED`), đã kiểm trong code chứ không giả định.

**Test BE**: `time.spec.ts` 6 → 11 ca (mốc địa phương, lùi ngày, lùi tháng, sang năm, đứng đúng lúc giao thừa), và `src/modules/users/session-stats.spec.ts` 9 ca cho phần gom mốc thuần: cửa sổ trống vẫn trả đủ số
ô (để biểu đồ không nhầm "không có gì" với "không hỏi"), cắt theo nửa đêm giờ máy chứ không theo UTC, vai player
khác vai hirer, `playerPayout` null vẫn tính phiên mà không tính tiền, dữ liệu trước cửa sổ bị loại, mốc tương lai
rơi vào ô mới nhất chứ không biến mất. **43/43 test BE pass** (trước đó 34).
- Một ca e2e trong `test/v2-gaps.e2e-spec.ts` ("buckets my finished sessions per day and per month", suite
  12/12 pass) khoá thêm đúng cái hợp đồng mà app vẽ biểu đồ dựa vào: đủ số ô trả về, startsAt tăng dần, **ô cuối
  cùng phải là hôm nay theo giờ máy** (test tự tính bằng `Intl` với `Asia/Ho_Chi_Minh`, không copy giả định),
  `granularity=week` / `buckets=0` → 400 và không token → 401.

**App**: `src/components/stats/trendMath.ts` (pure: `buildTrendBars`, `barHeight`, `peakBar`, `shortBucketLabel`,
`METRIC_CONFIG`) + `src/components/stats/SessionTrendCard.tsx` (2 hàng Chip: Ngày/Tháng và Doanh thu/Phiên đã
chơi/Giờ đã chơi/Chi tiêu, caption tổng + cao nhất, `accessibilityRole="image"` kèm mô tả đầy đủ). Không thêm
thư viện biểu đồ nào — stack giữ nguyên, bar vẽ bằng `View`.
- Tách math khỏi component không phải cho đẹp: jest **không import được component nào** của app lúc này, vì
  `expo-asset` chỉ được cài nested ở `node_modules/expo/node_modules/expo-asset` (Metro hiểu, còn resolver theo
  chuẩn Node của jest thì chết ở `expo-font/build/FontLoader.js`). Đã đụng đúng lỗi đó khi thử test cái card.
  Cách sửa gọn nhất là để logic thuần ở module không import RN; sau này muốn test render thì mới cần map
  `expo-asset` trong `jest.config.js`.
- 12 ca mới (`src/components/stats/__tests__/trend.spec.ts`) → **52/52 test FE pass**, `tsc --noEmit` 0,
  `expo lint` 0, `expo export --platform web` 0.
- `userService.getSessionStats` + `queryKeys.sessionStats(granularity, buckets)`; đã chạy
  `npm run openapi:export` (123 paths, có `SessionStatsView`) và `npm run api:generate` để client orval commit
  không lệch hợp đồng (diff sinh đúng 5 model + 1 endpoint, không churn).

Kiểm chứng thật (API dev + Metro web, tài khoản có đơn thật trong DB dev):

```
API     demo.player1 (hirer của đơn fd8fe34f…): 2026-11-03 → hirerSessions 1, spent 100.000
        demo.player3 (player):                  2026-11-03 → playerSessions 1, playerHours 1, earned 9.000
        month x6 / month x12: bucket 2026-11 nhận đúng cùng một con số
        không query  → mặc định day/14; granularity=week → 400 validation_failed
                       buckets=0 / 999 / abc → 400; không token → 401
App     /statistics demo.hirer1 (chưa có phiên nào): "Chưa có doanh thu nào trong 14 ngày gần nhất."
        /statistics demo.player1, chọn "Chi tiêu":
          caption "100.000 đ đã trả cho người chơi trong 14 ngày gần nhất · Cao nhất 100.000 đ (03/11)"
          đo DOM: đúng 1 cột cao 132px (đỉnh, tô vàng) + 13 cột 2px (đáy) = 14 cột
        bấm "Theo tháng": 1 cột 132px + 5 cột 2px = 6 cột, axis 06/26 → 11/26
```

**Số liệu dev đang có rất ít**: DB chỉ có đúng một đơn khép (tạo ở §35), nên biểu đồ lúc này một cột vàng và
mười ba cột đáy — đó là dữ liệu thật, không phải placeholder. Muốn thấy nhiều cột thì cần seed thêm đơn lịch sử
hoặc test trên máy với vài phiên thật.


## 39. #7 — Google sign-in trên máy thật: đào tới tận nguồn, và một quyết định còn thiếu

Câu hỏi đặt ra là "cần cung cấp gì để đăng nhập Google chạy được trên bản dựng thật". Trước khi trả lời,
tôi đọc **code đang cài** thay vì nhớ lại hướng dẫn cũ — và hoá ra tài liệu trong repo (`docs/google-signin.md`,
mục 5) đang hứa sai.

**Bốn sự thật đo được, không suy đoán**:

1. `Google.useIdTokenAuthRequest` chỉ xin `id_token` trực tiếp **khi `Platform.OS === 'web'`**. Trên Android/iOS
   nó truyền `responseType: undefined` cho `useAuthRequest`, và hàm đó tự chọn `ResponseType.Code` ("installed
   apps can auto exchange") với redirect mặc định `<applicationId>:/oauthredirect`. Trích trong
   `node_modules/expo-auth-session/build/providers/google.js`.
2. Luồng code đó sau khi nhận `?code=` sẽ tự đổi code ở `https://oauth2.googleapis.com/token` **với
   `clientSecret: undefined`** (cùng file, `AccessTokenRequest`). Client loại *Web application* bắt buộc secret
   → `invalid_client`; mà secret thì không đời nào được nhúng vào app. Custom scheme cũng không phải redirect URI
   Google chấp nhận cho client Web. Tức là con đường "thêm `com.playtogether.app:/oauthredirect` vào client Web"
   mà mục 5 cũ ghi là ✅ **không tồn tại** — đã sửa lại trong `docs/google-signin.md`.
3. `expo-auth-session@58.0.14` không còn chứa thư viện Google native nào: `package.json` chỉ có
   expo-application / constants / crypto / linking / web-browser, không có `ios/`, `android/`, không có config
   plugin; và `GoogleAuthRequestConfig` bị đánh dấu `@deprecated` trỏ tới `guides/google-authentication`. Guide
   đó (sửa ngày 02/11/2026) bảo dùng `@react-native-google-signin/google-signin` (17.0.0) hoặc
   `react-native-nitro-google-signin` (2.3.1), và nói rõ **Expo Go không dùng được**.
4. Phía API **đã sẵn** cho client native: `google-auth.service.ts:29-34` gom `GOOGLE_CLIENT_ID` +
   `GOOGLE_ANDROID_CLIENT_ID` + `GOOGLE_IOS_CLIENT_ID` thành danh sách audience, `verifyIdToken({ audience })`
   nhận cả mảng — nên thêm client Android chỉ là thêm biến env, không sửa code.

**Đã chốt trong repo (không cần quyết định nào của anh)**:

- `play-together-fe/eas.json` — 4 profile dev/preview/production để dựng development build; `expo config
  --type introspect` chạy sạch với `app.json` hiện tại (exit 0). **Chưa** cài `expo-dev-client`: docs Expo
  (03/11/2026) nói từ lúc có nó thì `npx expo start` nhắm development build thay vì Expo Go, tức đổi luôn cách
  anh chạy #122. Lệnh cài đã ghi ở đầu mục 6 `docs/google-signin.md`, kèm `-g` cho ai vẫn muốn Expo Go.
- Nút Google không còn chết âm thầm: khi Google trả lỗi, toast nói **lý do** thay vì "Đăng nhập Google thất bại".
  `src/features/auth/googleFailureMessage.ts` (thuần, không import Expo nên jest chạy được trên Node) map 11 mã
  OAuth sang tiếng Việt, mã lạ thì giữ nguyên mã + mô tả của Google và cắt ở 140 ký tự vì toast tự ẩn sau 2,6 s.
  9 ca test mới → **61/61 FE pass**, `tsc --noEmit` 0, `expo lint` 0, `expo export --platform web` 0 và
  CI trên GitHub `success` với commit `630741b`. Bằng chứng nó thật sự nằm trong màn hình: chuỗi tiếng Việt
  mới xuất hiện đúng một lần, trong `dist/_expo/static/js/web/login-*.js` — các chunk route khác 0.
- `README.md` của app: bỏ câu "mọi module native đều có trong Expo Go nên demo không cần development build" —
  đúng với cả app trừ Google trên máy; viết lại đoạn Google (trước đây bắt biến `EXPO_PUBLIC_GOOGLE_IOS_CLIENT_ID`
  /`_ANDROID_` dù code không đọc hai biến đó). `.env.example` cũng nói thẳng: browser OK, Expo Go không bao giờ,
  máy thật cần một trong hai con đường.
- `docs/google-signin.md` mục 6 viết lại thành **6.A / 6.B / 6.C**.

**Quyết định còn thiếu, và vì sao tôi chưa tự làm**: hai con đường trên máy đều cần development build, nhưng chỉ
6.A phải trả thêm một học phí rất cụ thể — thêm native code nghĩa là **Expo Go không nạp nổi app nữa**, kể cả các
màn không liên quan tới Google (import thất bại là crash cả màn đăng nhập) — đúng lúc lượt test bàn phím #122 trên
máy đang chờ. Nên tôi dừng ở bước chuẩn bị, không `npx expo install` gì cả.

| | 6.A — lib Google native | 6.B — client Android + `expo-auth-session` |
| --- | --- | --- |
| Phụ thuộc mới | 1 lib native (+ config plugin) | không |
| Expo Go sau khi làm | **hết dùng được, toàn app** | vẫn dùng được cho mọi thứ khác |
| API | không đổi (`serverClientId` = client Web cũ) | thêm `GOOGLE_ANDROID_CLIENT_ID` |
| App | tách nhánh theo `Platform.OS`, web vẫn dùng hook cũ | thêm `androidClientId` + `redirectUri` dạng `com.googleusercontent.apps.<id>:/oauth2redirect`, khai scheme đó trong `app.json` |
| Trải nghiệm | nút Google thật của hệ điều hành | vẫn mở trình duyệt |
| Rủi ro | Expo Guide đang khuyến nghị, nhưng phải dựng bản dev để test | đi trên đường ray Expo đã đánh dấu `@deprecated` |

**Chỉ anh làm được** (chi tiết ở `docs/google-signin.md` §6.C): `npx eas-cli login` + `eas init` (tài khoản
expo.dev — hiện `npx eas-cli whoami` báo *Not logged in*, `app.json` chưa có `extra.eas.projectId`), tạo OAuth
client **Android** (package `com.playtogether.app` + SHA-1 của keystore mà bản dựng dùng, lấy từ
`npx eas-cli credentials`), tuỳ chọn client **iOS** (cần tài khoản Apple Developer), rồi điền client ID vào
`.env` của app và của API rồi khởi động lại cả hai. Client ID không phải bí mật; **client secret** thì có — không
đưa vào app, không commit.

**Chưa có bằng chứng đầu-cuối cho chính cái toast**: unit test khoá bảng map, bundle proof khoá việc nó nằm
trong chunk `login`, nhưng chưa lần nào thấy dòng chữ mới hiện trên màn hình. Lý do không phải thiếu cố gắng:
với lỗi ngay từ bước xin authorization (redirect URI không hợp lệ, client sai loại), Google hiển thị trang lỗi
**ngay trong popup** chứ không quay lại app, nên `response` của hook vẫn null và app không có gì để toast. Muốn
thấy câu chữ thật thì phải ở lượt dựng máy (6.A/6.B) — lúc đó các lỗi trả về kèm redirect (mã code hỏng, grant
bị thu…) mới đi qua hàm map này. Ghi lại để lần sau đừng tính đây là "đã test xong".

**Việc còn treo không phải do Google**: `play-together-fe/.env` vẫn giữ IP LAN cũ `192.168.1.8` trong khi máy đã
sang `192.168.1.9` (§37) — đổi một dòng đó là #122 test được tiếp.


## 40. C2 — màn Đánh giá thành lớp phủ trong `end-order`, và một defect BE lộ ra khi test nó

#122 đã xác nhận trên máy thật (§28), nên C2 — món cuối của §23 nhóm C — được thi công.

**Đã làm**: xoá hẳn route `src/app/rating/[orderId].tsx` (inbound duy nhất là `router.replace` từ `end-order`,
đã kiểm bằng grep trên cả ba repo: không notification deep-link nào trỏ vào đó). Form chuyển thành
`src/components/orders/RatingSheet.tsx` — `Modal` + `KeyboardAvoidingView` (`padding` riêng iOS, giống
`CancelOrderDialog` đang dùng), kéo từ đáy lên, nền `colors.overlay`, đóng bằng chạm nền / nút ✕ / `onRequestClose`.
`end-order` giữ nguyên mọi thứ nó đã có: vẫn `refetchInterval: 10_000`, vẫn `!canRate` → không hiện CTA, vẫn có
nhánh EmptyState trong sheet cho ca polling lật `canRate` thành false *khi sheet đang mở*.

**Lý do bỏ luôn nút tố cáo ở đầu màn đánh giá cũ**: màn đó hiển thị nút báo cáo vô điều kiện, còn `end-order` chỉ
hiện khi `isFeedbackOpenOrder` (đơn `DISPUTED`/`SETTLED`/`ENDED_EARLY`). Một đơn cho phép đánh giá thì luôn ở trạng
thái khép, nên điều kiện của `end-order` rộng bằng — không mất entry point nào; `end-order` còn giữ cả ô "Có vấn đề"
trong footer khi chờ đối soát.

Kiểm chứng bằng Metro web trong iframe cùng gốc 375×740 (đăng nhập `demo.player1`, đơn `fd8fe34f…`):

```
CTA        : "Đánh giá" (aria-label + nhãn) hiện ở footer end-order, cao 50px
Bấm CTA    : sheet mở, tiêu đề "Đánh giá người chơi", đủ 5 nút "Đánh giá N sao"
Bố cục 375 : sheet y=167..740, cao 573 (max-height 92% = 680, không tràn)
             sao kết thúc y=468 · ô nhập 484..644 (rộng 343, cao 160) · nút "Gửi" 668..724 (< 740, không phải cuộn)
             scrollWidth tài liệu = 375 → không vỡ ngang
Validate   : bấm "Gửi" khi chưa chọn sao → toast "Vui lòng đánh giá!" (giữ nguyên copy cũ)
Chọn sao   : nhãn tổng kết đổi thành "5.0 trên 5 sao" → state ăn thật
Polling    : network log trong lúc sheet mở cho thấy GET /orders/{id} lặp lại nhiều lần cách nhau 10s →
             lớp phủ KHÔNG chặn nhịp đối soát của màn dưới
```

**Ba thứ phải ghi lại để lần sau khỏi hiểu nhầm số liệu**:
- Surface của app dựng hình ở đây báo `viewport = 0x0` và `visibilityState = hidden`, nên **animation
  `slide` của Modal không chạy**: wrapper bị đóng băng ở `transform: matrix(...,0,740)`, nghĩa là sheet nằm
  hoàn toàn dưới mép màn hình. Đó là giả tạo của môi trường đo, không phải lỗi layout; muốn đo hình học thì
  tạm đặt `transform: none` rồi đo (số liệu ở trên lấy sau bước này).
- Toast tự ẩn sau 2,6s, còn mỗi lần gọi MCP tốn ~1s, nên "chờ rồi đọc toast" dễ trượt. Muốn chắc thì đọc
  `list_network_requests` hoặc bắt trạng thái ngay trong cùng một script.
- `performance.getEntriesByType('resource')` không liệt kê request còn pending; network log của MCP mới là
  chỗ thấy request thật.

**Defect lộ ra khi test (chính là #124, và nó không phải chuyện câu chữ)**: lần bấm "Gửi" có đủ sao + lời nhắn
bị server từ chối —

```
POST /api/v2/orders/fd8fe34f…/ratings → 409 application/problem+json
  code   : conflict
  detail : "You already reported the other party on this order, so you cannot rate it"
```

Trong khi cùng đơn đó, `GET /orders/fd8fe34f…` vẫn trả `canRate: true, hasRated: false`. Lý do ở code:
`orders.service.ts:1248` tính `canRate = isParticipant && FEEDBACK_STATUSES && withinWindow && !ratedSet`,
còn `ratings.service.ts:75-86` **chặn thêm** "đã tố cáo trong đơn này". Lá chắn của POST vì vậy chặt hơn cờ mà
GET quảng bá: app hiển thị một nút mà mọi lần bấm đều 409, và toast chỉ dịch được "Thao tác xung đột với trạng
thái hiện tại." vì code lỗi là `conflict` chung. Ghi nhận riêng: `canRate` còn dùng `endedAt` trong khi POST dùng
`endedAt ?? scheduledEndAt` — lệch thứ hai, ít gặp hơn nhưng cũng là hai quy tắc khác nhau.

**Chưa kiểm chứng**: nhánh `!canRate` bên trong sheet. Sau khi sửa #124 thì đúng đơn này sẽ thành `canRate: false`
nên càng không có đường bấm tới đó; nó chỉ tới được bằng race (polling lật cờ khi sheet đang mở), chưa tái tạo.
Đường **thành công** của sheet cũng mới chạy tới POST: lần bấm "Gửi" thật trong lượt này gặp 409 (defect §41), nên
`onDismiss` + `onRated()` chưa được thấy trên màn hình. Cả hai chỉ là đóng overlay và gọi lại một query đã có, nhưng
đừng ghi là "đã test xong".

**Gate**: `tsc --noEmit` 0 · `expo lint` 0 · jest 61/61 (không đổi — không có test nào import màn rating) ·
`expo export --platform web` 0.


## 41. #124 — một luật đánh giá duy nhất: cờ `canRate` phải khớp với POST

§40 tóm lại: không phải chuyện viết lại câu chữ, mà là hai chỗ trong BE đang diễn giải **cùng một luật** theo hai
cách. Nay luật đó nằm một chỗ.

**`src/modules/ratings/rating-eligibility.ts`** (mới, thuần): `ratingBlock(input)` trả về lý do chặn đầu tiên theo
đúng thứ tự mà POST vẫn báo lỗi từ trước tới nay — `notParticipant → orderNotFinished → noCompletionTime →
windowClosed → reportedOrder → alreadyRated`, và `canRateOrder()` là phủ boolean của nó. `ratingBlockedError()`
dịch block sang problem+json với message **giữ nguyên từng chữ** các lỗi cũ, riêng ca `reportedOrder` chuyển từ
`conflict` chung thành mã mới **`rating_blocked_by_report`** (`error-codes.ts`, type `ErrorCode` tự suy ra từ
object nên không phải sửa gì thêm).

**Hai chỗ đọc luật**:
- `ratings.service.create()` bỏ hẳn chuỗi `if … throw` thủ công, thay bằng một `Promise.all` lấy report + rating
  hiện có của người dùng này trên đơn này rồi gọi `ratingBlock`. Đổi nhỏ có chủ đích: hai query giờ chạy cả khi
  request sẽ bị chặn từ bước trạng thái — đổi lấy một định nghĩa duy nhất, và đây là route viết rất ít lượt.
- `orders.service.hydrate()` thêm query `reports.orderId … eq(reports.reporterId, viewerId)` (batch, không N+1) và
  tính `canRate` bằng `canRateOrder(...)`. Biến `withinWindow` cũ **đọc `endedAt` một mình**, còn POST dùng
  `endedAt ?? scheduledEndAt` — nay cả hai dùng chung phép đó. Xoá luôn alias `FEEDBACK_STATUSES` vì chỉ một chỗ
  dùng nó.

**App**: `errorMessages.ts` thêm `rating_blocked_by_report: 'Bạn đã tố cáo lượt thuê này nên không đánh giá được
nữa.'`. `client.ts` chọn `vietnameseMessageFor(code)` **trước** nên câu này thắng bản dịch chung của `conflict`
("Thao tác xung đột với trạng thái hiện tại.") — người dùng đọc được lý do thật thay vì một câu vô nghĩa.

Kiểm chứng:

```
Unit  (rating-eligibility.spec.ts, 17 ca): 3 trạng thái feedback được phép · PROPOSED/SCHEDULED/LIVE/CLOSED bị
          từ chối · outsider chặn trước mọi thứ · không có mốc kết thúc · đúng hẹn vẫn còn cửa (now == deadline),
          trễ 1ms thì hết · endedAt null nhưng scheduledEndAt còn hạn thì vẫn cho · report chặn dù chưa đánh giá ·
          report thắng when cả hai · đã đánh giá thì báo alreadyRated · mã + status của từng lỗi
E2E   (test/rating-block-by-report.e2e-spec.ts, 3 ca, DB seed mới): đơn DISPUTED chưa tố cáo → **cả hai bên đều
          canRate true** (đối chứng dương, DB dev không tạo được cảnh này) · hirer tố → canRate của hirer thành
          false và POST ratings trả 409 code `rating_blocked_by_report` · phía player vẫn canRate true rồi POST
          201, sau đó hasRated true và canRate false
API dev thật (đơn fd8fe34f…, người hirer đã tố từ §35):
          GET /orders/{id}   → canRate: false   (trước bản sửa này: true — chính là defect §40)
          POST /orders/{id}/ratings → 409 code=rating_blocked_by_report type=…/errors/rating_blocked_by_report
          người chơi của cùng đơn → canRate false vì hasRated true (cô ấy là người đã đánh giá 4★ ở §35,
          không phải vì luật mới)
Gate  : BE tsc 0 · eslint 0 · unit 60/60 · **e2e 82/82 (10 suite)** · FE tsc 0 · lint 0 · jest 61/61
```

OpenAPI không đổi (`openapi.json` không liệt kê danh sách mã lỗi, hợp đồng response giữ nguyên) nên không cần
chạy lại `npm run openapi:export` / `api:generate`.

**Chưa quan sát lại trên màn hình**: nút "Đánh giá" biến mất với người đã tố. nguyên nhân là surface trình diễn
của môi trường đo bị treo ngay sau phần kiểm sheet (§40) nên không bấm lại được. Đây là suy diễn từ một boolean:
`canGiveRating = isHirer && order.canRate` — cùng điều kiện đã **được thấy hiện thật** trên màn hình (khi `canRate`
true thì nút xuất hiện, §40) và API giờ trả false; không phải một lần bấm thật sau khi sửa.


## 42. Đồng bộ lại tài liệu trước khi sang task mới

Anh yêu cầu: cập nhật plan/spec/doc cho khớp code **trước** khi làm tiếp. Đây là biên bản của lượt đồng bộ đó —
mỗi dòng là một câu tài liệu đang nói sai so với code, kèm cách đã đo lại.

### Những chỗ đã lệch và đã sửa

| Tài liệu | Câu cũ nói gì | Code/đo thật | Đã sửa |
| --- | --- | --- | --- |
| `README.md` gốc, "Giới hạn đã biết" | "**136 operation** … 97 × `200`" | `play-together-be/openapi.json` tính lại: **123 path / 146 operation** — 107 × `200`, 17 × `201`, 1 × `202`, 21 × `204` | ghi số mới + kèm lệnh đếm lại, vì con số này đổi theo API |
| `README.md` gốc, "Giới hạn đã biết" | "**API có** GitHub Actions CI" | `.github/workflows/ci.yml` tồn tại ở **cả ba** app (kiểm bằng `ls` từng repo) | viết lại: mỗi repo một đoạn, liệt kê đúng bước từng pipeline |
| `README.md` gốc | (không nói test chạy ở đâu) | repo gốc **không** có script `test` — `package.json` chỉ có `check` = typecheck + lint | thêm bullet: test nằm ở từng repo, `npm run check` ở gốc không chạy test |
| `README.md` gốc, "Tùy chọn: đăng nhập Google" | "tạo client *Web application* rồi điền cùng giá trị vào API và app" là xong | đúng cho web, **sai cho điện thoại**: `useIdTokenAuthRequest` chỉ xin `id_token` khi `Platform.OS === 'web'`; native đi code + PKCE và tự đổi code, đổi đó cần client installed app | viết lại thành hai nhánh (web xong ngay / device cần 6.A hoặc 6.B) + nêu Expo Go không test được, theo `docs/google-signin.md` §6 |
| `play-together-be/README.md` | cây `test/` liệt kê **3** e2e spec; unit = "money, time zones, pagination" | `ls test` = **10** spec; unit còn `rating-eligibility.spec.ts` (luật đánh giá, §41) | bổ sung đủ 10 spec + đổi mô tả unit |
| `play-together-be/README.md` | bảng env chỉ có `GOOGLE_CLIENT_ID` | `GoogleAuthService` nhận danh sách audience (`google-auth.service.ts:29-34`), biến `GOOGLE_ANDROID_CLIENT_ID` / `GOOGLE_IOS_CLIENT_ID` đã dùng thật | thêm hàng riêng cho hai biến đó |
| `play-together-fe/README.md` | **không** nhắc tới test nào, dù CI chạy `npm test` | jest-expo: 7 suite / 61 ca, chỉ import được lớp thuần | thêm `npm test` / `test:watch` vào Scripts + mục **Tests** giải thích giới hạn `expo-asset` |
| `play-together-admin/README.md` | **không** nhắc vitest, dù CI chạy `npm test` | 2 file / 30 ca: `src/auth/permissions.ts` (RBAC) và `src/lib/labels.ts` (nhãn %, trong đó có phí theo basis point — §36) | thêm script + đoạn ghi rõ phạm vi test và caveat "không render component" |

Ba con số trong bảng trên đều đo lại bằng lệnh trong chính lượt này, không lấy từ trí nhớ.

### Trạng thái các đề án (kể từ §23)

- **§23 — đề án gộp màn: đóng**, trừ **nhóm G**. Từng nhóm một mục: B → §29, A → §30, F → §31, E → §32,
  D → §33, C → §35; **C2** (§40) thành `RatingSheet` — màn `rating/[orderId]` đã xoá, lớp phủ nằm trong
  `end-order`. Hai dòng "Còn lại của §23" ở cuối §32 (dòng 2224) và cuối §33 (dòng 2268) là mô tả
  **tại thời điểm viết**, không phải backlog — muốn biết còn gì thì đọc mục Backlog bên dưới.
- **Nhóm G** (hai màn đàm phán đơn) **gác có chủ đích**, không phải sót: gộp chúng là sửa UI của một
  state machine tiền tệ, chờ lượt test máy kế tiếp. Hai mục "trùng query" từng ghi ở nhóm G đã **bác bỏ**
  khi đối chiếu code (màn Thống kê dùng `/wallet/earnings/summary`; `history/[orderId]` là superset sau
  query key riêng).
- **#7 — Google sign-in trên bản dựng thật: anh chốt 6.B** (giữ `expo-auth-session` + client Android)
  vì vòng test máy vẫn cần Expo Go; 6.A gác có lý do. **Code app đã cài xong cùng ngày — xem §43**,
  còn thiếu client Android trong Google Cloud Console của anh và một lượt EAS build.

### Backlog còn mở sau lượt này

| Việc | Vì sao chưa làm ngay |
| --- | --- |
| Expo SDK 58.0.19 → 58.0.23 (+ `npx expo install --fix`) | `npm view expo dist-tags.latest` = **58.0.23** hôm nay, app đang khoá `~58.0.19`. Bản vá trong cùng SDK, nhưng mỗi lần nâng là một lần phải test lại trên máy; anh yêu cầu đồng bộ tài liệu trước nên chưa động |
| Test render component (`@testing-library/react-native` + map `expo-asset` trong jest) | thêm dependency vào harness vừa dựng; nong phạm vi test là quyết định của anh |
| Seed đơn lịch sử để biểu đồ "Diễn biến phiên" (§38) có nhiều cột | đo thật trên `GET /users/me/session-stats` của `demo.player1`: **14 bucket ngày, chỉ một bucket khác 0** (2026-11-03) — biểu đồ đang là một cột. Seed là viết vào PGlite mà lượt test máy đang dùng |
| Nhóm G (§42) | chờ lượt test máy |
| Cấu hình `SMTP_*` khi triển khai | việc của hạ tầng thật |


## 43. #7 — 6.B đã cài: đăng nhập Google trên bản dựng thật, không đổi vòng test Expo Go

Anh chốt 6.B với lý do quyết định: **"mình vẫn cần test app qua Expo Go"**. Bảng so sánh hai đường (đã
trình trước khi làm) rút ra ba sự thật dẫn tới lựa chọn:

1. **Cả hai đường đều phải tạo client Android + khai SHA-1** — nên 6.A không giảm việc phía tài khoản,
   chỉ tăng chi phí test.
2. Máy này **không có JDK / Android SDK / adb** (`java` và `adb` đều không trên PATH) và `eas-cli` chưa
   login → mọi bản dựng đều là EAS cloud, theo trang pricing hôm nay gói miễn phí có **15 lượt build
   Android**. 6.A biến mỗi lượt test máy thành một lượt build; 6.B chỉ cần build cho riêng nút Google.
3. `@deprecated` trong `expo-auth-session@58.0.14` chỉ gắn trên **type** `GoogleAuthRequestConfig`
   (`build/providers/Google.d.ts:8`), hai hook `useIdTokenAuthRequest` (dòng 68) và `useAuthRequest`
   (dòng 83) **không** bị đánh dấu → 6.B không phải code sắp bị bỏ.

### Đã cài (chỉ app, API không sửa dòng nào)

- `src/features/auth/googleNativeRedirect.ts` (mới, thuần): client ID → scheme trả về dạng AppAuth
  `com.googleusercontent.apps.<id-bỏ-hậu-tố>`, và redirect `<scheme>:/oauth2redirect`. ID không đúng
  dạng thì trả `null` để caller rơi về hint, không tự bịa scheme.
- `app.config.ts` (mới): đọc `EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID` / `..._IOS_CLIENT_ID` và **thêm
  scheme đó vào `scheme`**, giữ `playtogether` cho deep-link của router. Không có env thì config y hệt
  `app.json` — Expo Go và web không bị chạm. Lý do nằm ở config chứ không phải trong app code: scheme
  là cấu hình native, phải có trước lúc dựng.
  *Ghi chú kỹ thuật*: `app.config.ts` **không import được** code trong `src/` — Expo nạp file này bằng
  `require` thông thường (lỗi thật: `Cannot find module .../src/features/auth/googleNativeRedirect`,
  kể cả khi đã dùng đường dẫn tuyệt đối). Nên quy tắc derive tồn tại hai nơi, và
  `googleNativeRedirect.spec.ts` có ca test so `app.config.ts` sinh scheme gì với
  `googleInstalledScheme()` để hai bên lệch nhau là test đỏ.
- `GoogleLoginButton.tsx`: vẫn **một hook** `useIdTokenAuthRequest` — nó tự chọn kiểu luồng theo nền
  tảng (`Google.js:88-95`: `responseType: IdToken` chỉ khi `Platform.OS === 'web'`; `Google.js:108-121`:
  `androidClientId`/`iosClientId` được ưu tiên trên từng nền tảng; `Google.js:142-148`: `redirectUri`
  truyền tay thắng mặc định). Thêm `androidClientId`/`iosClientId` + `redirectUri` dẫn xuất, không tách
  component.
- Cũng trong nút đó: **Expo Go không còn mở luồng nữa**. `Constants.executionEnvironment ===
  ExecutionEnvironment.StoreClient` → toast nói "Expo Go không mở được liên kết Google trả về…".
  Trước đây Expo Go hiện một nút bấm rồi chết giữa trang Google — tương tác giả, đúng thứ brief cấm.
  Thiếu client cho nền tảng nào thì hint gọi đúng tên biến của nền tảng đó.
- `.env.example` khai `EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID` / `_IOS_` kèm hệ quả build; README của app
  viết lại đoạn Google.

### Bằng chứng

```
Unit  (googleNativeRedirect.spec.ts, 14 ca mới): derive từ client ID thật · ID đã ở dạng scheme thì giữ
          nguyên · trim · 6 input hỏng/empty → null · redirect = scheme + :/oauth2redirect ·
          **app.config.ts đăng ký đúng scheme mà helper tính**, và không đổi gì khi thiếu env
Gate  : FE tsc 0 · expo lint 0 · jest **8 suite / 75 ca** · `expo export --platform web` 0
Config: `npx expo config --type public` — không env → `scheme: 'playtogether'`;
          với client Android giả → `['playtogether','com.googleusercontent.apps.1234567890-abcdefg']`
Web   (đo thật, không suy diễn): vào `/login`, bấm nút, đọc URL Google nhận được —
          `response_type=id_token`, `redirect_uri=http://localhost:8081`,
          `scope=openid + userinfo.profile + userinfo.email`, `client_id` = client Web cũ
          → đường web **không đổi hành vi** sau bản sửa này
API   : `google-auth.service.ts:29-34` đã nhận ba audience, `.env.example` của BE đã khai đủ
          GOOGLE_ANDROID_CLIENT_ID / GOOGLE_IOS_CLIENT_ID → không sửa code, không đổi OpenAPI
```

### Chưa kiểm chứng được (và vì sao)

- **Luồng code + PKCE trên máy thật**: cần một client Android (Console của anh) và một bản dựng.
  Không có JDK/Android Studio nên không dựng cục bộ để thử sớm hơn được.
- **Đúng dạng redirect**: app gửi dạng AppAuth vì tài liệu OAuth cho installed app của Google buộc dạng
  đó, còn mặc định của thư viện là `com.playtogether.app:/oauthredirect`. Nếu build đầu báo
  `redirect_uri_mismatch` (toast dịch sẵn mã này, mục 8), đổi về mặc định chỉ mất hai chỗ
  (`GoogleLoginButton` + `app.config.ts`) — test giữ chúng khớp nhau.
- **Khoảng lặng khi đổi code thất bại**: `useAuthRequest` tự đổi code bên trong hook; nếu Google từ chối,
  promise nội bộ reject và **không có `response`** → nút đứng yên, không toast. Đây là giới hạn của
  đường 6.B, đã ghi vào `docs/google-signin.md` để lần build đầu biết mà nhìn network thay vì chờ toast.
- **Expo Go không còn là môi trường thử Google**: hành vi này chỉ quan sát được bằng code + tài liệu §5,
  chưa bấm thật trên máy (vì máy đang dùng Expo Go cho các màn khác — anh xác nhận giúp nếu thấy nút
  vẫn mở trình duyệt).

---

## 44. Seed lịch sử: 70 ngày phiên đã đóng, viết đúng theo đường tiền của service

Anh chốt: seed dữ liệu lịch sử trước, rồi mới test máy — "Làm luôn đi, tôi sẽ test sau khi seed xong" —
để một lượt trên điện thoại bao được nhiều màn thay vì phải bấm tạo từng đơn.

### Vì sao thiếu

`npm run db:seed` không tạo đơn nào. Mọi màn đọc phiên thật đều chỉ có một vạch hoặc bảng trống:
biểu đồ **Diễn biến phiên**, dashboard hai chiều Hirer/Player, danh sách **Đơn** và **Tiền** của admin,
ví, lịch sử đơn, và cả hàng đợi yêu cầu trong app.

### Đã viết — `src/db/demo-history.ts` (bật bằng `SEED_DEMO_HISTORY`, không bật theo `SEED_DEMO_DATA`)

Mười suite e2e seed với `demoData: true` rồi tự tạo đơn để kiểm chứng; nếu lịch sử đi theo cờ đó thì
chúng nhận thêm một đống đơn lạ, nên lịch sử có cờ riêng.

```
70 ngày trước   : mỗi ngày 2–3 phiên, 161 SETTLED + 18 ENDED_EARLY
hôm nay         : 2 phiên vừa kết thúc cách đây ~1 giờ → tiền còn đang chờ về ví
ngày mai / kia  : mỗi ngày 1 yêu cầu PROPOSED + 1–2 lịch SCHEDULED → lịch hẹn và hàng đợi không trống
```

Một phiên đi đúng chuỗi mà service ghi, không phải rải dòng vào bảng `orders`:

| Bước | Dòng ghi thêm | Tiền |
| --- | --- | --- |
| `PROPOSED` | hop tạo yêu cầu + notice "Yêu cầu thuê mới" gửi player | — |
| `SCHEDULED` (lúc accept) | hop + notice "Đã giữ lịch hẹn" gửi hirer | `holdEscrow`: escrow HELD + cặp ledger `ESCROW_HOLD`, trừ tiền hirer |
| `LIVE` | hop "Lịch hẹn tự động bắt đầu đúng giờ đã đặt" (actor null) + 2 notice "Lượt thuê đã bắt đầu" | — |
| hết giờ hẹn | hop `DISPUTED` (actor null) + 2 notice "Xác nhận lượt thuê" / "Lượt thuê đã kết thúc" | — |
| `SETTLED` | hop + 2 notice "Lượt thuê hoàn thành" | `splitFee(earned, feeBps)`: 3 dòng ledger ESCROW_RELEASE / ORDER_EARNING / PLATFORM_FEE + một `pending_credits` |
| `ENDED_EARLY` | hop + 2 notice "Lượt thuê kết thúc sớm" (kèm số tiền đã hoàn, format `formatVnd`) | `quoteEarlyEnd` tại **70% thời lượng đã đặt** → refund về hirer, escrow `PARTIALLY_SETTLED` |

Ba ràng buộc viết tay hồi trước hay lệch nên giờ số được **suy ra**, không gõ vào:

- `available_balance` của mỗi ví = Σ ledger USER − Σ `pending_credits` chưa nhả — đúng quy tắc mà
  `releaseDuePendingCredits` dùng, và bản seed không bao giờ cho ra số < 0.
- `player_payout` = số đã giải ngân − phí, và `orders.fee_bps` lấy từ `system_config` chứ không gõ cứng.
- Counter trên hồ sơ player (rating, số đơn, đúng giờ, giờ chơi, ranking points) tính lại bằng đúng câu
  aggregate của `StatsService.recompute`.

`released_at` chỉ được đóng dấu khi đã tới hạn, nên những phiên mới nhất để lại tiền chờ thật —
màn "chờ về ví" và job nhả tiền có việc để làm, thay vì danh sách trống.

### Số đo sau khi seed lại (cluster dev, 2026-11-04)

| Chỉ số | Trước | Sau |
| --- | --- | --- |
| đơn | 0 | **185** (PROPOSED 2 · SCHEDULED 4 · ENDED_EARLY 18 · SETTLED 161) |
| đánh giá | 0 | **205** |
| doanh thu player đã ghi sổ | 0 | **21.739.500 đ** |
| phí nền tảng | 0 | **2.415.500 đ** |
| escrow đang giữ | 0 | **650.000 đ** |
| đang chờ về ví | 0 | **234.000 đ** |
| thông báo (chưa đọc) | 0 | **1.406 (70)** |
| biểu đồ 14 ngày gần nhất | 1 vạch | có mặt số ở **cả 14 ngày** |

### Kiểm chứng

```
Audit : scripts/audit-demo-data.mjs → `npm run db:audit`, mở một BẢN COPY của cluster
        (API vẫn chạy được, không đụng file đang khoá) và suy lại từng số.
        21/21 PASS · exit 0
Unit  : 5 suite / 60 ca PASS
E2E   : 10 suite / 82 ca PASS (test tự seed, không bật lịch sử)
Gate  : tsc --noEmit 0 · eslint 0 · nest build 0
HTTP  : /wallet (player2 pendingBalance 108.000 + 1 pending credit còn hạn chờ),
        /wallet/earnings/summary, /wallet/transactions, /wallet/deposits,
        /orders?pageSize, /orders/requests, /users/me/session-stats (day + month),
        /notifications (+unread-count 8), /admin/stats (185 đơn, escrowHeld 650.000), /admin/kpis,
        /admin/money/{ledger,wallets,deposits,escrows}, /orders/:id/history,
        /orders/:id/ratings, `canRate=false` khi hirer đã đánh giá đơn đó
        hop admin thấy rõ: ∅→PROPOSED→SCHEDULED→LIVE→DISPUTED→SETTLED, `formatDateTimeInZone`
        in giờ đúng múi Á Đông (20:45 04/11/2026)
```

### Một defect phụ tìm ra khi chuẩn bị seed — đã sửa

`scripts/reset-db.mjs` **không đọc `.env`**, nên nó lấy mặc định `.data/pgdata` thay vì
`PGLITE_DIR=.data/pgdata-fresh` mà `.env` đang trỏ tới: lệnh `db:reset` xoá nhầm cluster đã nghỉ dùng,
không phải cluster đang chạy. Không mất dữ liệu (cluster thật còn nguyên, và bản backup
`.data/pgdata-fresh.bak-20261104*` đã có sẵn từ trước). Đã sửa ba điểm: nạp `dotenv/config`, in đường
sẽ xoá **trước khi** xoá, và từ chối mọi đường có `.bak` trong tên.

### Còn chờ

- **Test trên máy thật** — lượt của anh. Seed đã đủ dữ liệu cho: tìm player, tạo yêu cầu, nhận đơn,
  kết thúc sớm, đánh giá (lớp phủ `RatingSheet` §40), ví + chờ về ví, thông báo + deep-link, lịch hẹn,
  biểu đồ Diễn biến phiên, dashboard Player, và bảng Orders/Money/KPI của admin.
- **Tab Chat trống, có chủ đích**: production không tự mở chat (dòng `chats` chỉ sinh khi ai đó bấm
  Gửi tin), nên muốn test chat thì nhắn từ tài khoản hirer sang player ngay trên máy — đó đúng là
  luồng cần test.
- **Hàng đợi yêu cầu tự đóng sau 5 phút**: `order.response_window_minutes` = 5 là luật của sản phẩm,
  bản seed tôn trọng giá trị trong `system_config`. Muốn hàng đợi mở suốt buổi test thì nâng setting
  đó trong admin **rồi tạo** yêu cầu mới; seed lại sẽ kéo setting về mặc định.
- **Nạp/rút thật**: không seed withdrawal nào (`pendingUserWithdrawals = 0`); màn rút tiền vẫn cần
  bấm thật để test.

## 45. #7 (nối §43) — EAS đã link, SHA-1 lấy được từ APK, client Android đã tồn tại

§43 dừng ở "code app xong, còn chờ phía tài khoản". Ngày 06/11/2026 đã đi hết phần tài khoản, trừ hai
dòng `.env` mà chỉ chủ máy được sửa.

**Vì sao phải dựng tới hai lần.** Client Android cần SHA-1 của keystore, mà keystore EAS chỉ sinh ra
khi có lượt dựng đầu — vòng trứng/con gà, không cách nào rút xuống một build. Build #1
`09a07d85-6df4-46fc-9f54-e490669af26c` do đó chỉ để lấy fingerprint; build #2
`a13a9792-86e7-4e6f-a496-1d78f3fdf2cf` là bản thật: có `EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID` và
**scheme return AppAuth biên dịch vào project native** (`app.config.ts` thêm
`com.googleusercontent.apps.<id>` vào `scheme`) — đổi scheme là chuyện của project native nên không tồn
tại đường nào ngoài dựng lại.

**Profile dựng là `preview`, không phải `development`.** `eas build --profile development` chết ngay với
`Install expo-dev-client manually`; thư viện đó cố ý chưa cài vì §43 đã ghi rõ cài vào là
`npx expo start` nhắm dev build thay vì Expo Go, mà vòng test của cả app đang chạy qua Expo Go.
`preview` cho APK độc lập (không cần Metro), và quan trọng hơn: hai lượt dựng dùng **chung một
keystore** — log in `Using Keystore from configuration: Build Credentials A6ccPXjA8H (default)` — nên
SHA-1 khai một lần là đúng cho mọi bản sau.

**Lấy SHA-1 khi không có cả `eas credentials` lẫn `keytool`.** `eas credentials` là menu tương tác
(không có cờ in fingerprint), và máy này không có JDK nên `keytool` không tồn tại. Đường thực dụng: tải
APK về, đọc `APK Signing Block`. APK chỉ ký v2 (`0x7109871a`) nên `META-INF/*.RSA` không có gì cả — ký
v1 không tồn tại ở bản release. Duyệt theo cấu trúc: `EOCD` → offset Central Directory → xác
`magic = "APK Sig Block 42"` → `size` ngay trước magic → suy ra đầu block → tách các cặp
`(u64 length, u32 id)`. Cạm bẫy thật sự: chuỗi `30 82` gặp đầu tiên trong section là
**`tbsCertificate`** chứ không phải certificate — nó thiếu `signatureAlgorithm` + `signatureValue`,
nhưng `openssl asn1parse` vẫn đi hết nên nhìn tưởng thành công, còn `openssl x509` thì báo
`Could not find certificate`. Phải thử **parse ra X509** rồi mới tin. Certificate đúng là khối 813 byte
DER (`30 82 03 29`), RSA 2048, DN rỗng (`C= ST= L= O= OU= CN=` — keystore EAS sinh vậy),
serial `2A305AEA9513107A`, notBefore `2026-11-05 17:10:37 GMT` — trùng khít thời điểm build #1 tạo
keystore, chính là bằng chứng "đúng keystore của project này".

**Đã tạo trong Google Cloud (`play-together-513312`).** OAuth client loại Android: Name
`Play Together - Android`, Package name `com.playtogether.app`, SHA-1
`4D:86:07:82:4F:65:21:E3:CF:E7:13:51:D4:55:02:E9:49:DF:9C:10`, Client ID
`606945851708-cj8l2b67qie9ougprndr7vn1k9uof7k1.apps.googleusercontent.com`. Client ID loại Android đi
cùng PKCE nên **không có secret**; nó hiện trong bundle, vì vậy ghi công khai ở đây là hợp lệ và tiết
kiệm cho lần sau một lượt build. Danh sách clients sau khi tạo: 2 dòng (Web + Android), và trang chi
tiết hiển thị đúng package + SHA-1 đã nhập.

**Thay đổi trong repo.** `app.json` nhận `extra.eas.projectId` + `owner` (và `eas init` tự thêm
`extra.router.asyncRoutes.web` — không phải em viết); `eas.json` khai `environment` cho profile
`development` và `preview`. Khóa `environment` đó là mắt xích thật: **`.env` nằm trong `.gitignore` nên
không được upload lên máy dựng**, build log phải in ra `Environment variables with visibility "Plain
text" and "Sensitive" loaded from the "preview" environment on EAS: …` thì mới có bằng chứng. Ba biến
đang nằm trong environment `preview` (và hai biến trong `development`): `EXPO_PUBLIC_API_URL=http://<pc-ip>:3000/api/v2`,
`EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID`, `EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID`. **Chưa commit.**

```
Config: npx expo config --type public (override env per-process, không sửa .env)
        → scheme = ["playtogether","com.googleusercontent.apps.606945851708-cj8l…"]
        → android.package = com.playtogether.app · extra.eas.projectId khớp project đã tạo
Fingerprint: X509Certificate(...).fingerprint (Node) · openssl x509 -fingerprint -sha1 ·
             openssl sha1 trên file DER  →  ba giá trị trùng một
Build   : #1 09a07d85 FINISHED, APK 104.924.661 byte — bản này chỉ để lấy keystore
          #2 a13a9792 FINISHED, APK 104.925.037 byte, dùng chung keystore A6ccPXjA8H (default)
          Kiểm **trong file APK**, không tin log: unzip `assets/index.android.bundle` +
          `AndroidManifest.xml` rồi tìm chuỗi. build#2 bundle có androidClient=YES, webClient=YES,
          apiUrl=YES; build#2 manifest có appAuthScheme=YES (scheme return đã biên dịch vào native);
          build#1 cả hai đều `no` — control âm cho thấy phép thử thật sự phân biệt được, không phải
          khớp do chuỗi tình cờ. Bundle không chứa nguyên văn scheme (đúng như thiết kế: nó được nối
          lúc chạy từ client ID), nên phải soi manifest chứ không chỉ soi bundle.
Gate FE : tsc 0 · expo lint 0 · jest 8 suite / 75 ca (googleNativeRedirect.spec giữ app.config.ts
          và src/ khớp nhau — chính là mắt xích vừa dựng lại)
Hạn mức : eas account:usage manhnguyendev → Android 0/15 trước khi build, không phải hết credit
API     : /api/v2/health qua <pc-ip>:3000 → {"status":"ok","database":"up"}, /users trả player
          đã có ratingCount/ordersCount/rankingPoints từ seed §44
```

### 45.1 Bản dựng đầu trên máy báo `400 invalid_request` — và thủ phạm là một mặc định mới của Google

Anh cài APK build #2, bấm **Đăng nhập bằng Google**, Google chặn bằng trang *Lỗi 400: invalid_request*.
Không phải app sai: trang lỗi của Google ẩn lý do trong tham số `authError` (base64), giải mã ra câu
chặt chẽ: **"Custom URI scheme is not enabled for your Android client."**

Cách tái hiện không cần điện thoại (lặp lại được khi nghi ngờ): chính request mà
`useIdTokenAuthRequest` gửi lên `accounts.google.com/o/oauth2/v2/auth` dựng được bằng tay từ code —
`response_type=code` + `client_id` = client Android + `redirect_uri` =
`com.googleusercontent.apps.<id>:/oauth2redirect` + `scope=openid profile email` + PKCE
(`code_challenge_method=S256`). Mở URL đó trong trình duyệt: **trước** khi sửa thì Google trả trang lỗi,
**sau** khi sửa thì Google hiện màn hình *Chọn tài khoản — Tiếp tục tới Play Together* với đúng test user
`manhnguyendt309@gmail.com`. Khác biệt nằm ở duy nhất một cấu hình tài khoản.

**Nguyên nhân:** Google đã đổi mặc định cho OAuth client loại **Android** tạo mới — tuỳ chọn nhận
redirect bằng *custom URI scheme* (chính là dạng AppAuth mà mọi thư viện installed-app dùng) giờ **tắt
theo mặc định**, Console ghi thẳng "This setting is not recommended for Android clients" và đặt nó sau
*Advanced settings → Enable custom URI scheme*. Client Android em tạo chiều 06/11 nằm trong nhóm đó, nên
mọi request AppAuth bị từ chối trước khi tới được app. Bật công tắc + Save là hết; **không đổi client ID
nên không phải build lại app**, và giá trị có hiệu lực ngay trong phiên (Google quảng cáo "5 phút tới vài
giờ" nhưng phép thử tái hiện cho thấy nó bật lên tức thì).

**Bài học về hướng chính sách:** Google đang đẩy app Android sang loopback IP hoặc App Links (https +
`assetlinks.json`, cần tên miền công khai) và tự coi custom scheme là đường cũ. Nếu một ngày Google gỡ
hẳn checkbox này, 6.B hết đường và phải chuyển sang 6.A (thư viện Google native, dùng Play Services nên
không có redirect URI) — cái giá đã ghi ở §43 là mất Expo Go cho mọi màn khác. Ghi lại để lần sau khỏi
đi tìm lỗi trong app.

**Đã kiểm chứng thêm một điểm mà chuỗi trong manifest không trả lời được:** scheme có mặt trong
`AndroidManifest.xml` chỉ chứng tỏ nó nằm trong string pool, chưa chứng tỏ có `intent-filter` nào nhận
nó. Em chạy `npx expo prebuild --platform android --no-install` (có override
`EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID` theo process, không sửa `.env`) rồi đọc manifest sinh ra:
`MainActivity` có `intent-filter` với `<data android:scheme="playtogether"/>` **và**
`<data android:scheme="com.googleusercontent.apps.606945851708-cj8l…"/>` — Android sẽ trả callback về
đúng activity. Dọn dẹp sau đó: prebuild **tự đổi `package.json`** (`android`/`ios` từ `expo start --android`
thành `expo run:android` — sẽ làm hỏng vòng test Expo Go nếu để yên), em `git restore package.json` và xoá
thư mục `android/` vừa sinh (folder này nằm trong `.gitignore`, sinh ra chỉ để đọc rồi bỏ).

**Còn chờ** (cập nhật cuối mục sau §45.2).

- ✅ **06/11 đã đóng — cấu hình env hai phía**, nhưng không phải theo cách §45 dự kiến:
  `play-together-be/.env` có `GOOGLE_ANDROID_CLIENT_ID` (đối chiếu bằng sha256 của *cả chuỗi* với
  client ID + hậu tố `.apps.googleusercontent.com` — khớp tuyệt đối, không in giá trị ra log), còn
  `play-together-fe/.env` **không** có `EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID`: biến đó chỉ sống trong
  EAS Variables (environment `preview`) nên chỉ bản dựng trên mây thấy được. Vòng local thì Metro export
  đúng hai biến (`env: export EXPO_PUBLIC_API_URL EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID`), tức **chạy
  `expo prebuild` tại chỗ mà không tự set biến Android sẽ sinh ra native project thiếu scheme return** —
  bẫy này đã ghi ở §6.B, và là lý do mọi phép prebuild kiểm chứng trong các mục trên đều phải truyền
  override theo process. Thiếu dòng phía API thì consent vẫn mở, code vẫn đổi được id_token, nhưng `aud`
  là client Android nên `verifyIdToken` từ chối — lỗi nằm ở API, không phải app. Bằng chứng hiện tại:
  token giả gửi lên `/auth/google` trả **401** chứ không **503**, tức API đã đọc được client và đã tới
  bước verify.
- ✅ **06/11 đã đóng — cài APK và bấm nút**: đã cài thật, đã bấm thật, và hai lỗi chỉ lộ ra ở lượt đó
  (§45.2). `redirect_uri_mismatch` mà §45 lo lắng **không** xảy ra — Google chấp nhận redirect AppAuth
  ngay khi bật custom URI scheme.
- **Hàng đợi Free của EAS**: build #1 nằm `in queue` ~70 phút trước khi chạy. Không phải lỗi cấu hình,
  cũng không phải hết hạn mức — đừng dựng lại khi thấy `in queue`, chỉ cần `eas build:list`.
- **iOS** vẫn mở: cần client loại iOS (Bundle ID `com.playtogether.app`) + tài khoản Apple Developer.

### 45.2 Hai lỗi bản release trên máy thật: router nuốt callback, rồi Android chặn HTTP

Cài build #2 → **Đăng nhập bằng Google** → consent chạy hết, Google chấp nhận redirect AppAuth (§45.1),
nhưng thứ hiện lên là màn **"Unmatched Route — Page could not be found"** với URL
`playtogether:///oauth2redirect?state=…&code=…`. Fix xong, build #3 đi tiếp được một bước nữa rồi dừng ở
toast **"Không thể kết nối tới máy chủ: Kiểm tra EXPO_…"**. Hai lỗi này *đều không tồn tại* trong Expo Go,
và cả hai đều nằm ngoài mọi lời nhắc của thư viện — ghi lại đủ để lần sau không phải mất hai lượt dựng.

**(1) expo-router coi callback OAuth là một route.** Mọi URL hệ thống gửi vào app đều được router đưa qua
`redirectSystemPath` rồi điều hướng; `/oauth2redirect` không có màn hình nào đứng sau nên router kéo app
vào trang 404 nội bộ của nó, đè lên màn đăng nhập ngay lúc `expo-auth-session` đang đổi code — trình
`linking.js:154-176` guard bằng `if (href) listener(href)`, nên chỉ cần trả về falsy là router tự đứng
ngoài (thứ mà luồng này không cần: chính hook của `expo-auth-session` đã nghe sự kiện `url` rồi).
Fix: `play-together-fe/src/app/+native-intent.ts` — trả `null` khi path chứa `oauth2redirect`, giữ
nguyên mọi path khác. Kèm `src/features/auth/__tests__/nativeIntent.spec.ts`: 2 callback (một của AppAuth
scheme, một của scheme `playtogether` — đúng hai dạng mà máy có thể nhận) → `null`; 2 link thường
(`/orders/<id>`, `playtogether://profile`) → không đổi.
*Điều chỉnh lại thói quen kiểm chứng:* không được dùng chuỗi `redirectSystemPath` trong bundle làm bằng
chứng — bản thân expo-router chứa đúng định danh đó ở chỗ gọi, nên phép đếm chuỗi không phân biệt được app
có file fix hay không (build #3 đo ra `utf8=1`, con số đó vô nghĩa). Bằng chứng thật là **hành vi trên
máy**: build #3 hết trang 404 và tiến tới bước gọi API. Chuỗi binary chỉ đáng tin khi chuỗi đó *độc quyền*
của thay đổi (client ID, IP LAN, scheme AppAuth).

**(2) Android cấm HTTP rõ văn bản, và Expo SDK 58 không có chỗ khai báo.** Đo trực tiếp trong APK build #3
(`unzip` rồi đọc `AndroidManifest.xml` dạng binary + `assets/index.android.bundle`):

```
manifest  usesCleartextTraffic           utf8=0  utf16=0   ← vắng mặt, đúng giả thuyết
manifest  network_security_config        utf8=0  utf16=0
manifest  android.intent.action.VIEW     utf8=0  utf16=1   ← control dương: đúng file, đúng cách đọc
manifest  com.googleusercontent.apps     utf8=0  utf16=1   ← control dương thứ hai
bundle    192.168.1.8:3000               utf8=1  utf16=0   ← URL LAN đã biên dịch vào app
bundle    localhost:3000                 utf8=0  utf16=0   ← control âm: không phải fallback nào sót lại
bundle    android client id              utf8=1  utf16=0
```

String pool của AXML lưu chuỗi ở UTF-16, nên **mọi** phép đếm UTF-8 trên manifest trả 0 — hai control dương
(chuỗi `VIEW` và scheme AppAuth) chính là thứ biến cái `0` ở dòng đầu thành bằng chứng thay vì thành tiếng
sóng. TargetSdk ≥ 28 mặc định từ chối plain HTTP, app lại trỏ `http://<pc-ip>:3000`; Expo Go không bị vì
manifest của *chính Expo Go* cho phép cleartext — nghĩa là cả vòng test Expo Go chưa từng đi qua cổng này.
Expo SDK 58 không lộ key `app.json` nào cho việc đó (`@expo/config-plugins` chỉ *khai báo* thuộc tính
`android:usesCleartextTraffic` trong kiểu `build/android/Manifest.d.ts:80`, chứ không có chỗ nào trong
`app.json` đọc tới nó), nên buộc phải có config plugin: `plugins/withAndroidCleartext.js` (ghi thuộc tính vào
`<application>` qua `withAndroidManifest`) và nối vào `expo.plugins` trong `app.json`.
**Cái giá:** bật cleartext cho *mọi* domain của bản dựng, không riêng IP LAN — chấp nhận được với bản dev
nội bộ, và điều kiện gỡ đã ghi thẳng trong comment của plugin: API chạy HTTPS là xoá plugin.
Kiểm tại chỗ bằng `npx expo prebuild --platform android --no-install --clean`: manifest sinh ra có
`android:usesCleartextTraffic="true"` **và** vẫn còn 1 chuỗi `com.googleusercontent.apps` — hai thay đổi
không triệt tiêu nhau. Build #4 `e6ddd4b9` đang chạy để đo lại đúng hai chuỗi đó *trong APK*, không tin log.

**(3) Tầng thứ ba của "không gọi được API": chính cái PC này.** Lúc chuẩn bị giao build #3,
`curl http://192.168.1.8:3000/api/v2/health` treo (exit 28) trong khi tiến trình API **vẫn sống**. Dấu hiệu
lặp lại đúng lần thứ hai (instance `start:dev --watch` tối qua cũng treo kiểu đó): tiến trình còn, socket
vẫn nhận kết nối, nhưng event loop không trả lời; `netstat -ano` thì **không** liệt kê LISTENING nào — trên
máy này netstat không phải nguồn sự thật, curl mới là nguồn sự thật. Em tắt đúng PID mà log của *chính*
tiến trình đó ghi (`[Nest] 19160` trong `.data/api-run.log`) rồi khởi động lại ẩn, xuất log ra file; sau đó
health trả `{"status":"ok","database":"up","uptimeSeconds":3}`, `OPTIONS /auth/google` → 204, token giả →
401. **Không đụng tới PID 17216** (`node dist/main.js`): chuỗi cha đưa nó về
`pnpm --filter @budget-nest/server start` — cùng một mẫu lệnh, khác chủ; tắt nhầm là sập dự án khác trên
cùng máy.
Rút kinh nghiệm cho mọi báo lỗi "app không tới được server": có ba tầng phân biệt — API chết, Android chặn
HTTP, app sai URL — và thứ tự loại trừ luôn bắt đầu từ chính cái PC này vì phép đo đó rẻ nhất (`curl`
health). Nếu PC tự gọi được mà app vẫn báo lỗi thì hai nghi phạm còn lại đều là thứ *không bao giờ xuất
hiện trong log API*: chính sách cleartext của Android (mục (2) ngay trên) và URL đã biên dịch cứng vào
bundle.

**Còn chờ.**

- ✅ **06/11 đã đóng — build #4** (`e6ddd4b9`, FINISHED 14:50:50Z): đo lại trong APK đã ký, đúng hai chuỗi
  cần thấy, vẫn dùng control dương UTF-16 như bảng trên:

  ```
  manifest  usesCleartextTraffic      utf8=0  utf16=1   ← build #3 là 0/0; plugin đã vào bản dựng
  manifest  network_security_config   utf8=0  utf16=0
  manifest  com.googleusercontent.apps utf8=0 utf16=1   ← scheme return còn nguyên
  manifest  android.intent.action.VIEW utf8=0 utf16=1
  bundle    192.168.1.8:3000          utf8=1  utf16=0
  bundle    localhost:3000            utf8=0  utf16=0   ← control âm
  bundle    android client id         utf8=1  utf16=0
  ```
- **Group G / Docker / iOS / deploy công khai**: vẫn như các mục trước, không đổi.

### 45.3 "API tự treo" hoá ra là một deadlock thật: đọc config *bên trong* transaction trên PGlite

Đêm qua em ghi ở §45.2 rằng máy là tầng thứ ba gây lỗi kết nối. Đúng một nửa: máy chỉ là nơi *lộ ra*.
Sau ba lần dựng lại hiện trường, nguyên nhân là code, và nó nằm sẵn trong repo từ hồi làm cửa sổ đối soát
(§3), chỉ chờ dữ liệu seed (§44) để kích hoạt.

**Dấu hiệu.** Mỗi lần bật API bằng `node dist/main.js`, `/health` trả `database:"up"` được ~30–60 giây rồi
**mọi** địa chỉ (`127.0.0.1`, `localhost`, `192.168.1.8`) cùng timeout — curl exit 28 chứ không phải 7, tức
kernel vẫn nhận kết nối, còn ứng dụng thì không. Tiến trình sống, **CPU đứng nguyên** (`(Get-Process -Id
…).CPU` trả 5.234375 rồi không tăng sau 4 giây), RAM còn 4.9/16 GB. `netstat -ano` không liệt kê LISTENING
nào, kể cả các server khác đang chạy — trên máy này netstat không phải nguồn sự thật, đừng kết luận gì từ nó.

**Phép thử A/B.** `JOBS_ENABLED=false` → 8/8 probe sạch trong 178 giây. `JOBS_ENABLED=true,
JOBS_INTERVAL_MS=5000` → chết ngay sau một tick. Vậy thủ phạm là `BackgroundJobs`, không phải môi trường.

**Đo tới từng câu lệnh** (chỉ vá vào `dist/` đã build, `src/` không đổi; về sau `npm run build` đè lại):

```
[JOBDBG 15:03:32.688] start {}
[JOBDBG 15:03:32.725] expireDue          {expired:0, ms:38}
[JOBDBG 15:03:32.728] startDueSessions   {started:0, ms:41}
[ORDDBG 15:03:32.744] afterLockSelect  {id:"dd3ddb18-…"}
[ORDDBG 15:03:32.763] afterOrderUpdate {id:"dd3ddb18-…"}
[ORDDBG 15:03:32.793] afterUsersUpdate {id:"dd3ddb18-…"}
        … im lặng vĩnh viễn ở đây, và /health chết cùng lúc
```

Câu tiếp theo sau `afterUsersUpdate` chính là thủ phạm: `orders.service.ts` đọc
`this.configs.getNumber(CONFIG_KEYS.disputeWindowHours)` **không truyền `tx`**, dù hàm đã có tham số
executor (`getNumber(key, executor = this.db)`). Không truyền thì nó đi `all()` → `this.db.select()` —
một **kết nối thứ hai**, mà PGlite chỉ có đúng một kết nối và kết nối đó đang bị transaction bao ngoài
giữ. Nó chờ vô hạn, và mọi query sau đó (kể cả `/health`, vốn báo cáo `database`) xếp hàng phía sau.

**Vì sao suite e2e xanh từ trước tới giờ.** `CACHE_TTL_MS = 15 phút` trong `SystemConfigService`: khi cache
ấm, `getNumber` không chạm DB nên deadlock không xảy ra. `dispute-window.e2e-spec.ts` login hàng loạt trước
khi gọi `runJobs()` ⇒ cache luôn ấm. Còn dev thì **vừa khởi động là tick đầu tiên chạy với cache lạnh** ⇒
đúng 100% treo.

**Vì sao trước seed thì không treo.** `openDisputeWindow` chỉ vào transaction khi có đơn `LIVE` quá
`scheduledEndAt`. Trước §44 không có; sau seed dev DB còn **2** đơn như vậy (đo được).

**Đã sửa (4 file):**
- `src/modules/orders/orders.service.ts:838` — truyền `tx` cho `getNumber` trong `openDisputeWindow`.
- `src/modules/orders/orders.service.ts:592` — cùng lỗi trong `endEarly` (đường "Kết thúc sớm" của người
  thuê, liên quan tiền): một `getNumber` nữa thiếu `tx`.
- `src/modules/system-config/system-config.service.ts` + `src/common/security/token.service.ts:103` —
  `isMaintenance()` cũng đọc config qua kết nối dùng chung và đang nằm trong transaction của `rotate()`
  (đường refresh token). Thêm tham số executor vào `isMaintenance` rồi truyền `tx`; đây là bug *tiềm ẩn*
  (nó chỉ treo khi cache lạnh giữa lúc refresh), kiểm chứng bằng chính phép thử: trước khi sửa, gọi
  `/auth/refresh` vẫn 200 **vì** cache ấm — nên em không sửa kiểu "đoán", mà sửa vì cùng một cơ chế.
- `test/dispute-window.e2e-spec.ts` — thêm ca *"opens the window when the config cache is cold"*: tạo đơn
  quá hạn, gọi `harness.app.get(SystemConfigService).invalidate()` ngay trước `runJobs()`, rồi đòi đơn
  chuyển `DISPUTED`. **Đã chứng minh ca này bắt được bug**: bỏ đúng một chỗ truyền `tx` thì suite không
  in nổi gì sau 9 phút (jest không teardown được vì kết nối còn bị giữ), trong khi với code đã sửa nó xanh
  trong 75 ms và cả suite 6/6.

**Kiểm chứng bằng artifact, không tin suy luận.** `npm run build` → grep `ORDDBG|JOBDBG` trong `dist/` =
0 (dọn sạch instrument). Khởi động lại API với cấu hình mặc định (job 30 s), tick đầu log
`jobs: expired=0 started=0 disputed=2 completed=0 released=0` lúc 22:15:07 — job **chạy xong đúng phần việc
mà nó treo ở tick đầu tiên của mọi lần khởi động**, và 2 đơn đối soát thật sự được mở. Sau đó health
uptime 89 s rồi 116 s,
`/auth/login` 200, `/auth/refresh` 200, token Google giả 401, `err.log` 0 byte.

**Việc nên làm tiếp (chưa làm, ghi lại để khỏi quên).** Quét hết lớp lỗi "query không dùng tx bên trong
tx": em có hai script TEMP (`scan-tx-reentrancy.mjs`, `scan-tx-calls.mjs`) đếm block bằng brace — chạy
toàn `src/` ra đúng 3 chỗ, 2 đã sửa, 1 (`isMaintenance`) đã sửa; phần còn lại là dương giả vì `tx` nằm ở
dòng kế. Nếu muốn chốt lâu dài, đáng biến phép kiểm đó thành lint rule hoặc test tĩnh trong CI hơn là script
vặt ở TEMP.

**Một lần suýt phá dữ liệu, ghi lại để đừng lặp lại:** để đếm số đơn overdue, em đã mở *đúng cluster PGlite
của API* bằng một tiến trình node thứ hai (`new PGlite('.data/pgdata-fresh')`) trong khi API đang giữ nó.
Số đọc ra có vẻ đúng và API vẫn sống, nhưng đây là engine nhúng một người ghi — phép an toàn là gọi API
(admin endpoint) hoặc đọc khi API đã dừng, không phải mở song song.









---

## 46. Upload ảnh thật (lưu đĩa cục bộ) + avatar cho toàn bộ bản ghi, và bốn defect chỉ lộ ra khi bấm thật

**Yêu cầu.** "Làm tất cả các chỗ liên quan tới upload hình ảnh, giờ cứ cho lưu ảnh ở local đi, sau này
sẽ sử dụng bên thứ 3 sau" + "set all avatar cho các record hiện tại cho nó phong phú hơn, hình ảnh bạn
tự chọn rồi thêm".

### 46.1 Trước khi sửa: tính năng này chưa từng chạy được

`src/features/profile/pickImages.ts` (bản đã commit) nối `data:image/jpeg;base64,…` vào trường
`imageLink` rồi gửi lên `PATCH /users/me` — chú thích trong code nói thẳng là "không có Firebase
Storage nên inline base64". Nhưng `users.dto.ts` chặn `avatarUrl` bằng `@IsUrl()`, và `@IsUrl` **không**
chấp nhận data URI (đã đo, không đoán: ca control trong `test/uploads.e2e-spec.ts` gửi đúng chuỗi
`data:image/jpeg;base64,…` và nhận `400 validation_failed`). Nghĩa là mọi lần đổi ảnh đại diện từ app
bị API từ chối, còn app thì bắt lỗi chung chung nên người dùng chỉ thấy "Cập nhật thất bại".
Đây đúng là loại "tương tác giả" mà brief cấm để lại: có nút, có toast, không có ảnh nào được lưu.

### 46.2 Thiết kế: một interface, driver cục bộ, URL tương đối

- `src/common/storage/image-storage.ts` — `IMAGE_STORAGE` + `ImageStorage.saveImage(file)`;
  `StorageModule` (@Global) bind `LocalImageStorage`. Đổi sang S3/GCS/Cloudinary sau này là đổi
  provider, không đụng controller, DTO hay client.
- `LocalImageStorage` — kiểm tra theo thứ tự: có part `file` (`400 no_file_uploaded`) → kích thước
  (`422 image_too_large`, trần `MAX_UPLOAD_MB` mặc định 5 MB, interceptor còn chặn ở 20 MB trước khi
  nạp hết vào RAM) → MIME cho phép → **magic byte khớp MIME** (`422 unsupported_image_type`; một file
  chỉ đổi phần đuôi thành .png vẫn bị bắt). Ghi ra `UPLOAD_DIR/<yyyy-mm>/<uuid>.<ext>` và trả
  `/uploads/<yyyy-mm>/<uuid>.<ext>`.
- `originalname` của multer cố ý không dùng: đó là chuỗi do client đặt, dễ mang `../`.
- `serve-uploads.ts` đăng ký static **trước** `app.init()` — bản đầu dùng `@nestjs/serve-static` và
  nhận 404 `not_found` của Nest router chính vì thứ tự này (e2e bắt được). Header:
  `Cross-Origin-Resource-Policy: cross-origin` (app, bản web 8081 và admin 5174 là ba origin khác
  nhau; helmet mặc định để `same-origin` nên ảnh sẽ vỡ), `Cache-Control: public, max-age=86400`,
  `index:false`, `dotfiles:'deny'`.
- Lưu **đường dẫn gốc tương đối**, mỗi client tự nối origin khi hiển thị (`resolveAssetUrl()` ở FE và
  admin). Lý do: IP LAN của máy dev đổi (đã từng: .8), còn Google/CDN URL tuyệt đối thì giữ nguyên.
  `is-image-url.ts` thay `@IsUrl` để chấp nhận cả hai hình thái này.

### 46.3 Bốn defect mà chỉ bấm thật trên trình duyệt mới thấy

Em chạy hết luồng trên trình duyệt thật (Expo web :8081 + admin :5174, API :3000), và cả bốn chỗ sau
đều sai thật — không suy ra từ đọc code:

1. **Ảnh thư viện tải nhầm host.** `mappers.ts` đã resolve avatar, nhưng `userService.getImagesOfUser`
   trả `imageLink: image.url` nguyên trạng → browser xin `GET http://localhost:8081/uploads/…jpg` →
   **404**, màn Hình ảnh hiện hai ô vuông trắng. Đã sửa (`resolveAssetUrl`) + spec mới
   `src/api/services/__tests__/usersImages.spec.ts` giữ đúng quy tắc "đường dẫn tương đối phải trỏ về
   API host, URL tuyệt đối thì giữ nguyên".
2. **Ngày sinh bị lùi một ngày mỗi lần lưu.** Màn hình gửi `dateOfBirth.toISOString()`, service cắt
   `slice(0, 10)`. Ở +07, nửa đêm 01/01/2000 = `1999-12-31T17:00Z` → cắt ra `1999-12-31`. Cột
   `date_of_birth` là DATE nên mất luôn phần giờ. Sửa bằng `toCalendarDate()` / `parseCalendarDate()`
   trong `src/lib/format.ts` (dùng lịch địa phương, không dùng UTC) và 4 ca test. **Bằng chứng vòng
   ngoài:** DB đang đọc `2000-11-04` trước khi sửa; sau khi chọn 05/11/2000 và bấm CẬP NHẬT thì đọc
   `2000-11-05`.
3. **Lưu hồ sơ mà không chọn ảnh mới sẽ nhét URL tuyệt đối vào DB.** `save()` gửi lại `avatar` — mà
   `avatar` là *giá trị hiển thị* đã nối origin — nên `avatarUrl` thành
   `http://localhost:3000/uploads/…png`, đúng thứ thiết kế tương đối sinh ra để tránh. Sửa: không có
   ảnh mới thì **bỏ hẳn trường** (API đã có sẵn `...(body.avatar ? {avatarUrl} : {})`). Đã dọn lại bản
   ghi bị nhiễm.
4. **Màn "Thông tin tài khoản" mở từ tải lạnh là form rỗng.** `useState(user?.name ?? '')` chạy trước
   khi store hydrate xong, nên tên, giới tính, ngày sinh đều trắng và avatar hiện "?" — dù `/users/me`
   trả 200 đầy đủ. Sửa bằng effect khoá theo `user.id` (ref `hydratedFor`), chỉ mồi form khi có id mới,
   không ghi đè giữa lúc đang gõ.

Kèm một bài học kiểm chứng: Metro chạy với `CI=1` **không watch**, nên sửa xong bundle vẫn là code cũ và
phép thử đầu tiên cho kết quả sai. Phải grep `user-profile.bundle` thấy `toCalendarDate` (và không còn
`toISOString`) rồi mới được kết luận — đúng quy trình positive + negative control.

### 46.4 Avatar cho bản ghi hiện có

10 ảnh tự tạo (8 người + 2 huy hiệu tổ chức), resize 256×256 JPEG bằng System.Drawing, commit trong
`uploads/seed/` (thư mục duy nhất không bị gitignore) để `GET /uploads/seed/avatar-01.jpg` phục vụ y hệt
một ảnh user upload. `npm run db:avatars` (`src/db/seed-avatars.ts`) **từ chối chạy khi API còn sống**
(PGlite một người ghi) và chỉ đụng các dòng `avatar_url IS NULL` hoặc đang trỏ vào chính ảnh seed — ảnh
Google và ảnh user upload không bao giờ bị ghi đè. Lần chạy đầu gán theo thứ tự tạo nên hồ sơ nam nhận
ảnh nữ; đã đổi sang hai pool theo giới và chạy lại: `MALE=3, FEMALE=2, OTHER=5`, đối chứng qua
`GET /admin/users` (Trần Quốc Bảo → avatar-06, Phạm Gia Huy → avatar-04, Võ Thanh Tùng → avatar-02,
Đặng Khánh Linh → avatar-03, Lê Thu Hà → avatar-01; tài khoản Google vẫn giữ ảnh của họ).

### 46.5 Admin

`CharitiesPage` thêm nút "Chọn ảnh" trong dialog: file input ẩn → `POST /uploads` → ghi đường dẫn trả
về vào **trường nháp** + thông báo "Đã tải ảnh lên. Bấm Lưu để áp dụng." (đúng luật plain-language:
không tự lưu, một nút Lưu duy nhất). Validator riêng thay `rules={[{type:'url'}]}` vì rule đó từ chối
`/uploads/…`. Bảng Users/Charities render qua `resolveAssetUrl`. Đã bấm thật trong dialog: field nhận
`/uploads/2026-11/3ecd3cfe-….png`, toast hiện, "Đồng ý" → `PATCH` 200 → đọc lại `avatarUrl` đúng đường
dẫn đó; sau khi xác nhận em đã đặt lại về `charity-01.jpg` để dữ liệu demo không mang ảnh thử.

### 46.6 Cổng kiểm chứng (chạy sau cùng, tất cả xanh)

| Repo | Kết quả |
| --- | --- |
| BE | `lint` 0 lỗi · `tsc --noEmit` 0 · unit 5 suites / 60 · e2e **11 suites / 92** (gồm `uploads.e2e-spec.ts` 9 ca: 401 ẩn danh, ghi và đọc lại nguyên byte kèm header CORP, JPEG hợp lệ, PNG giả mạo 422, MIME rác 422, quá cỡ 422, thiếu part 400, nhận URL tương đối, control data-URI vẫn 400) |
| FE | `expo lint` 0 warning · `tsc --noEmit` 0 · jest **12 suites / 95** |
| Admin | `tsc -b` 0 · oxlint 0 · vitest **3 files / 36** · `vite build` OK; grep chunk `CharitiesPage-*.js` thấy lệnh `post` tới `/uploads` với `multipart/form-data` và nhãn tiếng Việt (negative control: `base64` chỉ xuất hiện trong vendor, không phải code em viết) |
| Artifact sống | `POST /uploads` 201 → `GET /uploads/2026-11/….png` 200 `image/png` → `PATCH /users/me` 200 → `/users/me` trả đúng đường dẫn tương đối; avatar seed `200 image/jpeg 14587` |

### 46.7 Việc còn để ngỏ (chưa làm, ghi lại)

- Đổi sang bên thứ ba: chỉ cần thêm driver `ImageStorage` mới; nhưng phải quyết định **di dời** các file
  đã nằm trong `uploads/` hay để nguyên (URL cũ vẫn phải sống).
- Chưa xoá file: user xoá ảnh (`DELETE /users/me/images/:id`) chỉ xoá dòng DB, byte vẫn trên đĩa; lên S3
  thì phải tính lifecycle. Ảnh upload thử trong `uploads/2026-11/` hiện là file mồ côi của dev DB.
- `POST /uploads` chưa có rate limit riêng (throttler toàn cục 120 req/60 s vẫn áp), và chưa chặn dung
  lượng theo từng user.
- Ảnh chưa được resize/cắt phía server — app gửi ảnh đã cắt 1:1 từ picker, nhưng admin upload nguyên
  file; driver thật nên kèm pipeline thumbnail.


---

## 47. Redesign toàn diện theo brand contract — phương pháp OpenDesign

**Nguồn phương pháp.** [OpenDesign](https://github.com/nexu-io/open-design) tổ chức mỗi design system
thành một package: `DESIGN.md` (brand contract) + tokens + components, và agent **refactor component
thật của repo theo contract** chứ không dán template đè lên sản phẩm. Em áp dụng đúng vòng đó: viết
[DESIGN.md](../DESIGN.md) ở gốc repo làm nguồn sự thật chung cho app và admin, rồi sửa token → component
→ màn hình, và kiểm chứng bằng render thật.

### 47.1 App di động

- **Thang chữ một nguồn.** `theme/typography.ts` đổi sang thang 11/12/14/15/16/18/20/24/30 (line đi
  kèm, tracking âm ở cỡ lớn). Script migrate **233 chỗ `fontSize`/`lineHeight` số trần trong 59 file**
  (`src/app` + `src/components`) về `type.*`; backup trước khi chạy, ba import bị thừa dấu phẩy được
  sửa tay và bắt lại bằng `tsc`. Sau cùng grep `fontSize: [0-9]` = **0**. Weight giữ nguyên ý đồ màn
  hình (chỉ size/line về token), nên không màn nào mất nhấn nhá.
- **Bề mặt xếp lớp.** `shadow.card` hạ xuống phẩy nhẹ (ios 0.04/12/4, android elevation 1); thẻ phân
  cấp bằng viền mảnh chứ không bằng bóng đậm. Component mới `Surface` (nền + viền + bo + padding theo
  contract) và `PlayerRowCard` chuyển sang dùng nó; 9 thẻ vẽ tay còn thiếu viền được thêm hairline.
- **Chrome.** `AppBar` bỏ viền đáy và nằm trên màu canvas (-screen liền mạch thay vì dải trắng cắt
  ngang); tab bar tăng vùng icon active lên 48×30 và đổi màu inactive sang `textMuted` cho dễ đọc.
- **Trạng thái rỗng có lối ra.** `EmptyState` nhận `actionLabel`/`onAction`; Trang chủ và Lịch sử dùng
  nó với nút "Tìm người chơi" thay vì câu chết "Không có dữ liệu".
- **Ngoại lệ có chủ đích.** Nhãn tab bar giữ bậc 11 px: năm tab ở 375 dp chỉ còn ~70 dp mỗi ô, bậc 12
  cắt cụt "Quyên góp"/"Thông báo" (đã thấy trên ảnh chụp). Ghi chú ngay trong `(tabs)/_layout.tsx`.

### 47.2 Console quản trị

Một agent riêng triển khai, em kiểm chứng lại bằng render và cổng:

- `ConfigProvider` mang đủ token contract: primary/info `#5B51D8`, success/warning/error khớp status
  app, radius 10, controlHeight 36, canvas `#F6F7FB`, hairline `#E4E6EF`; component tokens cho Menu
  (nền sáng, active tint violet), Table (header `#F0F1F6`, hover tint), Button (bỏ bóng).
- Sider **sáng** cùng tông nội dung với brand mark violet + wordmark; header trắng viền mảnh; giữ
  nguyên tier Tag, Avatar dropdown, cổng 403 và toàn bộ phân quyền.
- Khung trang thống nhất `PageHeader → FilterCard → TableCard`; **23 bảng** đều có `TableEmpty` tiếng
  Việt; hết hex cứng trong page (money ± và Progress đi qua token, tag trạng thái đi qua `ToneTag`).
- Login và Dashboard dựng lại theo contract (thẻ hairline, KPI có icon trong vòng tint); nghiệp vụ,
  query và mutation giữ nguyên.

### 47.3 Ba defect chỉ lộ ra khi chụp màn hình thật

1. **Nhãn tab tràn ở 375 dp** sau khi thang chữ nâng caption lên 12 — hạ về 11 như ngoại lệ ở 47.1.
2. **Cột "Người dùng" của bảng Tài khoản co còn ~144 px** khiến tên và email gãy từng chữ một
   (`Mạnh / Nguyễn / manhn / guyer…`). Sửa: `width: 260` cho cột nhận diện + `ellipsis` hai dòng
   ([UsersPage.tsx](../play-together-admin/src/pages/UsersPage.tsx)); chụp lại thấy mỗi người một dòng
   tên, một dòng email.
3. **Avatar đỏ rượu vang trên rail người chơi** không phải lỗi render: là file PNG **1×1 pixel**
   (alpha 127) còn sót từ phép thử upload, được lưu làm avatar của `demo.player1`. Đã thay bằng ảnh
   seed đúng giới qua chính `PATCH /users/me` của tài khoản đó — không sửa thẳng vào DB.

### 47.4 Kiểm chứng

Panel Browser của phiên đang ẩn nên không chụp bằng MCP được; em dựng **Chrome headless + CDP**
(node có WebSocket sẵn) để chụp pixel thật: app ở 375×812 và 768×1024, cả sáng lẫn tối
(`prefers-color-scheme`), admin ở 1280×860 cho dashboard, bảng Tài khoản, login và cả cổng 403 (seed
nhầm tier Tài Chính ở lần chụp đầu — tình cờ thành phép thử trạng thái ngoài quyền). Tiến trình
chrome chụp hình đã dọn sạch sau cùng.

Cổng: FE `tsc` 0 lỗi, `expo lint` 0 warning, jest **12 suites / 95**; admin `tsc -b` 0, oxlint 0,
vitest **36**, `vite build` xanh; grep chunk admin thấy `5B51D8`/`itemSelectedBg:#EFECFF` và không còn
`1677ff` trong các chunk page (negative control chỉ còn trong vendor AntD).

### 47.5 Còn để ngỏ

- Admin chưa có dark mode (contract chưa đặt yêu cầu); app vẫn resolve scheme một lần lúc import —
  đổi theme nóng sẽ phải đụng cả 46 màn, chưa làm.
- Skeleton mới phủ rail người chơi và gợi ý tìm kiếm; các danh sách khác vẫn `LoadingView`. Đáng lẽ
  nên quét tiếp một lượt `ListEmptyComponent`/`LoadingView` toàn app.
- 47 màn vẫn giữ `StyleSheet.create` riêng; bước tiếp theo của contract là gom các khối thẻ lặp lại
  về `Surface` dần dần, không phải một lần đập đi.

## 48. Admin dark mode — đóng mục để ngỏ 47.5

**Yêu cầu**: "làm thêm dark mode cho admin nha".

**Cách làm** (theo contract §5 `DESIGN.md`, giữ nguyên stack AntD v6):

- Tách theme khỏi `src/main.tsx` về `src/theme.ts`: hai bản `light` / `dark` đặt cạnh nhau nên mọi
  khác biệt đều đọc được trong một file. Bản light **không đổi một giá trị nào** so với block cũ.
- `src/app/Root.tsx` là component duy nhất đọc chế độ và đưa `theme` vào `ConfigProvider` (tách file
  để `oxlint` fast-refresh còn 0 warning; `main.tsx` chỉ còn bootstrap + render).
- `src/lib/themeMode.ts`: store zustand `mode: 'light' | 'dark' | 'system'` + `systemDark`, key
  `pt_admin_theme`, mặc định `system`; `resolveTheme()` là hàm thuần nên test được không cần DOM;
  listener `matchMedia` giúp đổi theo hệ điều hành ngay khi OS chuyển theme.
- Bản tối dùng `theme.darkAlgorithm` và nên khớp bảng dark của app: canvas `#101219`, container
  `#181B24`, bề mặt nổi `#20242F`, hairline `#2A2E3B`, viền ô nhập `#454C60`, `colorTextBase`
  `#F1F2F8`; màu hành động nâng lên violet 600 `#6E64F0` (thuật toán dark render ra xấp xỉ
  `#6158CF`), menu active `#2C2E52`/`#BCB3FF`, header bảng `#20242F`. Không đổi seed status vì
  darkAlgorithm tự sinh bảng màu tối từ cùng một seed, nên `StatusTag` dùng chung nhãn cho cả hai.
- Công tắc ở header (`AdminLayout`): Dropdown ba lựa chọn **Sáng / Tối / Theo hệ thống**, icon mặt
  trời / mặt trăng theo chế độ đang hiển thị; không có lựa chọn bị khoá.
- `src/index.css`: `color-scheme` đổi theo `data-theme`, `body` lấy nền `var(--pt-canvas)`.
  `data-theme` và biến canvas được đặt ở module scope **trước khi React mount**, nên tải lại ở chế độ
  tối không loá trắng một frame.

**Kiểm chứng bằng render thật** (Chrome headless + CDP, 1280×860, 12 assertion PASS):

- Dashboard sáng: body `rgb(246, 247, 251)`, ảnh `admin-light-dashboard-v2.png` trùng bản trước khi
  làm dark → không hồi quy, chỉ thêm icon mặt trời ở header.
- Bấm công tắc → `Tối`: `data-theme=dark`, `localStorage pt_admin_theme='dark'`, body
  `rgb(16, 18, 25)` (`admin-dark-dashboard.png`).
- Tải lại vẫn dark (persist hoạt động).
- Bảng Tài khoản tối: header bảng `rgb(32, 36, 47)`, avatar và tag tiền đều đọc được
  (`admin-dark-users.png`).
- 'Theo hệ thống': OS dark → dark; OS light → **light ngay không cần tải lại** (đường matchMedia live).
- Login tối (`admin-dark-login.png`): nút `Đăng nhập` `rgb(97, 88, 207)` — đúng giá trị darkAlgorithm
  sinh ra từ seed `#6E64F0`, chữ trắng đạt tương phản.
- Surface nổi trong tối: modal `Sửa` (`admin-dark-modal.png`) và dropdown Select
  (`admin-dark-select.png`) đều `rgb(32, 36, 47)` với chữ `rgba(241, 242, 248, .85)` — không rò
  panel trắng, không phải kiểu "nền tối mà popup trắng".
- Hai ghi chú về AntD v6 trong quá trình chụp: content modal là `.ant-modal-container` (không còn
  `.ant-modal-content`), và dropdown Select chỉ mở khi gửi `Input.dispatchMouseEvent` thật chứ
  `element.click()` không mở được — hai lần probe đầu fail vì selector/cách giả lập sự kiện, không
  phải lỗi giao diện.

**Vá dữ liệu phát hiện khi chụp**: `demo.hirer1` vẫn mang avatar là ảnh test upload một màu
400×400 (`uploads/2026-11/50458f27-….png`), hiện thành vòng magenta trong bảng. Sửa qua chính
`PATCH /users/me` của tài khoản đó (admin DTO không có quyền đổi avatar), sang
`/uploads/seed/avatar-04.jpg`, trả về 200 — cùng loại defect với `demo.player1` ở 47.3.

**Cổng**: admin `tsc -b` 0 lỗi, oxlint **0 warning** (54 files), vitest **40 passed** (thêm 4 test
`themeMode`: system theo OS, lựa chọn tường minh thắng, mặc định `system`, `setMode` ghi
localStorage), `vite build` xanh. Grep chunk build: `6E64F0`, `101219`, `pt_admin_theme` đều có mặt;
`1677ff` không còn trong `src/` (chỉ xuất hiện trong hằng số default của thư viện AntD ở vendor
chunk, tức negative control nằm ngoài code ta).

**Còn để ngỏ**: app di động vẫn resolve scheme một lần lúc import (đổi theme nóng chưa làm); admin
chưa có accent khác ngoài violet.
## 49. Dashboard admin — bố cục lại toàn bộ (R6)

**Yêu cầu**: "Sắp xếp refactor và improve lại UI của Dashboard cho thật đẹp và xịn xò hơn".

**Trước**: bốn `Card` + `Statistic` trần, một bảng trạng thái hai cột, card "Chi trả đang chờ" gần như
trống trắng, và một cột nút block cho hàng chờ; không có biểu đồ, không có ngữ cảnh kỳ ngoài một
Radio.Group nhỏ.

**Sau** (giữ stack AntD v6, không thêm thư viện chart):

- **Hero band** gradient violet → indigo với glow gold/magenta và lớp grain SVG: overline, tiêu đề,
  chấm live đập theo nhịp làm mới, giờ cập nhật thật; `Segmented` chọn kỳ dời lên hero và **mặc định
  7 ngày** để lần mở đầu tiên đã có dữ liệu thật (kỳ "Hôm nay" của DB dev trống trơn).
- **Bốn KPI surface** thay Card trần: icon tint theo hue riêng của từng chỉ số, số đếm động
  (`AnimatedNumber`, tôn trọng `prefers-reduced-motion`), sparkline đơn tạo mới thật, thanh cơ cấu
  người chơi / từ thiện / quản trị, meter cửa chờ giải ngân, và trạng thái "Hàng chờ sạch" khi không
  còn việc tồn.
- **Chart diễn biến** SVG tự vẽ (`TrendChart`): hai series thật — tạo mới theo `createdAt`, hoàn thành
  theo `endedAt` — vùng gradient, lưới bốn vạch tròn số, crosshair + tooltip khi hover. Dữ liệu bucket
  từ `/admin/orders` (`from` + `sort=createdAt`, tối đa 10 trang; tràn thì ghi chú rõ số đơn bị cắt).
  Đối chiếu API khớp tuyệt đối: 14 đơn trong kỳ = 3/3/2/6/0/0/0 theo ngày.
- **Donut cơ cấu** bảy hue tách nhau bằng cả luminance (`src/lib/chartHues.ts`), hover từng cung đổi
  số ở tâm; legend bên cạnh là link lọc thật `/orders?status=…`.
- **Trong kỳ** thành bốn tile có meter tỷ lệ kết thúc sớm / phí nền tảng / tỷ lệ hoàn và năm sao đánh
  giá; **hàng chờ** thành hàng link có badge số, tint vàng khi còn việc; cột trái xếp `Trong kỳ` +
  `Đơn mới nhất`, cột phải `Việc cần làm ngay` + `Hoạt động quản trị` (audit, chỉ SUPER_ADMIN) hoặc
  `Chi trả đang chờ` khi không có quyền audit. RBAC giữ nguyên: chart và đơn mới nhất chỉ hiện khi
  tier được mở `/orders`.
- Component mới: `src/components/charts.tsx` (Sparkline / TrendChart / Donut),
  `src/components/AnimatedNumber.tsx`, `src/lib/dashboardSeries.ts` (bucket giờ cho hôm nay, ngày cho
  kỳ dài + `niceCeiling`) kèm 5 test, `src/lib/chartHues.ts`. `WithdrawalsPage` nhận `?tab=charity`
  để hàng chờ từ thiện deep-link đúng tab.
- `src/index.css` thêm họ class `pt-*`: hero, KPI, meter, share bar, hàng chờ, hàng đơn, tile, tooltip
  chart, skeleton shimmer, entrance stagger; **scrollbar vẽ theo palette** cho cả hai chế độ; media
  ≤575px và `prefers-reduced-motion`.

**Ba defect chỉ lộ khi render và đã sửa**: (1) mặc định "Hôm nay" khiến chart và các tile kỳ trống
ngay lần mở đầu → đổi mặc định sang 7 ngày; (2) khoảng trắng lớn giữa hàng "Trong kỳ" và hàng "Đơn
mới nhất" → ghép thành hai cột xếp chồng; (3) mobile 390px: hero cắt mất nút làm mới, tên đơn xếp dọc
từng chữ, legend đè tiêu đề chart → hero wrap + segmented thu gọn, hàng đơn xếp hai tầng với
ellipsis, ẩn legend và tag thừa ở khổ hẹp.

**Kiểm chứng**: Chrome headless + CDP, 15 assertion PASS — hero, 4 KPI, chart SVG, 7 hàng chờ, 6 đơn
mới nhất, feed audit, tooltip hover, đổi kỳ 30 ngày sinh nhãn ngày, deep link hàng chờ ra
`/orders?status=DISPUTED`, dark mode, mobile không tràn ngang (0px), không console error. Ảnh:
`dash-light-1440.png`, `dash-light-30d.png`, `dash-dark-1440.png`, `dash-dark-390.png`. Cổng: `tsc -b`
0, oxlint 0 warning, vitest **45**, `vite build` xanh; grep artifact thấy `pt-hero` trong chunk CSS và
`dashboard-series` + `3FB97F` trong chunk `DashboardPage`.

**Còn để ngỏ**: chưa có nút xuất CSV/Ảnh của chart; cung donut chưa bấm được trực tiếp (legend thì có).

## 50. Sửa chớp trắng của bộ chọn kỳ ở hero (light mode)

**Triệu chứng**: bấm đổi kỳ trên hero, light mode chữ trong Segmented biến thành trắng suốt lúc
thumb trượt; dark mode bình thường.

**Nguyên nhân đo được bằng CDP ở frame giữa chuyển động**: khi motion chạy, AntD gỡ
`ant-segmented-item-selected` khỏi item rời đi và chỉ gắn `ant-segmented-item-selected-text` cho item
đích; màu chữ item rời đi được interpolate về màu unselected (`rgba(255,255,255,.85)`) trong khi thumb
nền trắng (`colorBgElevated` của light) vẫn trượt đè lên đúng chữ đó → trắng trên trắng. Dark không lộ
vì thumb nền `#20242F`, chữ trắng vẫn đọc được.

**Sửa** (`src/index.css`, đúng hướng "đổi màu chữ ở white mode"): light mode cho **mọi nhãn** màu tím
đậm `#2F2A72` trên track kính mờ `rgba(255,255,255,.42)` — nhãn rời đi vẫn đậm nên đọc được cả trên
thumb trắng lẫn trên track; pill selected giữ nền trắng + chữ tím đậm, hover thêm nền trắng `.35`.
Tương phản nhãn chưa chọn trên track ≈ 5.5:1. Dark giữ nguyên luật cũ; hai bộ luật tách scope bằng
`:root[data-theme='dark']` và `:root:not([data-theme='dark'])` để không đè nhau.

**Kiểm chứng**: frame giữa (90ms) light mode mọi nhãn đo được `rgb(47, 42, 114)`, không còn giá trị
nhạt trung gian; ảnh `seg-light-mid.png`, `seg-light-settled.png`, `seg-dark-mid.png` cho thấy chữ đọc
được ở mọi vị trí thumb của cả hai chế độ. Regression toàn dashboard: 15 assertion PASS, mobile 390
không tràn, 0 console error; oxlint 0 warning, `vite build` xanh.

## 51. Bốn chỉnh sửa dashboard theo phản hồi người dùng

**Phản hồi**: (1) hàng "Tiền đang giữ hộ" trong "Việc cần làm ngay" không giống các hàng còn lại;
(2) khoảng trống lớn bên dưới "Cơ cấu trạng thái đơn"; (3) thiếu viết hoa đầu câu ở "· tự làm mới mỗi
phút" và "tổng đơn"; (4) hai cột số của legend donut không có tiêu đề nên không biết cột nào là gì.

**Sửa**:

1. Hàng giữ hộ bỏ dòng phụ dưới nhãn, chuyển số tiền vào **pill bên phải** cùng kiểu badge với các
   hàng khác (`.pt-queue-money`), chevron luôn nằm cuối hàng — mọi hàng chờ giờ cùng một khuôn:
   chấm màu → nhãn → pill giá trị → chevron.
2. Khoảng trống sinh ra vì `TrendChart` chỉ có viewBox nên trên màn rộng (máy người dùng 2559px)
   chart cao tới ~533px và kéo card donut giãn theo. Nay chart **đo chiều rộng thật bằng
   ResizeObserver** và giữ chiều cao cố định 240px (`preserveAspectRatio="none"`), nên hai card cùng
   hàng luôn cao 321px ở mọi khổ màn; donut phóng lên 184px và thân card căn giữa, hết dải trống.
3. Viết hoa: "Cập nhật lúc … · **Tự** làm mới mỗi phút"; tâm donut "**Tổng** đơn".
4. Legend donut thành lưới bốn cột có **hàng tiêu đề** "Trạng thái / Số đơn / Tỷ lệ" (chữ nhỏ hoa,
   gạch chân mảnh), cột số và cột phần trăm căn phải thẳng hàng nhờ `grid-template-columns` cố định.

**Kiểm chứng**: headless ở 2559×1400, 1440×1000 và 390×844, cả sáng lẫn tối: chiều cao chart đúng
240px ở cả ba khổ, hai card cùng hàng lệch nhau 0px, thân donut không tràn, tiêu đề legend đúng
"Trạng thái|Số đơn|Tỷ lệ", tâm donut "Tổng đơn", hero chứa "Tự làm mới mỗi phút", hàng giữ hộ có pill
"320.000 đ" và chevron là phần tử cuối, mobile không tràn ngang, 0 console error. Ảnh
`fix-light-2559.png`, `fix-dark-2559.png`, `fix-dark-1440.png`, `fix-dark-390.png`. Cổng: `tsc -b` 0,
oxlint 0 warning, vitest 45, `vite build` xanh.

## 52. Ba chỉnh sửa dashboard: chữ dòng tiền, cột số hàng chờ, bố cục dòng audit

**Phản hồi**: (1) "297.000 đ trong cửa chờ giải ngân" khó hiểu, muốn dùng từ khác; (2) "Việc cần làm
ngay": cột bên phải phải là **số đơn (record)** cần làm, mục nào dính tiền thì **tổng tiền nằm ngay
dưới nhãn**; (3) "Hoạt động quản trị": dòng trên ghi account thao tác + update mục gì, dòng dưới là
chi tiết update cái gì, còn thời gian đẩy sang bên phải hàng.

**Sửa**:

1. **Dòng tiền KPI**: "… trong cửa chờ giải ngân" → "**… đang chờ trả cho người chơi**". Đồng thời bỏ
   meter nằm trên dòng đó: `pendingCredits` (thu nhập người chơi chưa release) và `escrowHeld` (tiền
   đơn còn HELD) là **hai pool rời nhau** — đối chiếu `admin-ops.service.ts` thấy escrow chỉ cộng các
   row `status = HELD` còn credit chỉ cộng row `releasedAt IS NULL` — nên thanh tỷ lệ cũ hàm ý sai
   quan hệ bộ phận / toàn thể. Card giữ hai dòng chữ thuần, chân card vẫn dóng hàng với ba card kia.
2. **Hàng chờ**: mọi hàng cùng một khuôn chấm màu → nhãn (+ tổng tiền dưới nhãn, tiền tố "Tổng") →
   **pill số record** → chevron. Hàng "Tiền đang giữ hộ" hết pill tiền (`.pt-queue-money` bị xoá):
   số record lấy thật từ `/admin/money/escrows?status=HELD&pageSize=1` qua `meta.total` (query riêng,
   bật theo quyền `/money`, tự làm mới mỗi phút và ăn nút Làm mới), tổng tiền xuống dòng phụ.
   Nhãn và dòng phụ ellipsis một dòng (`.pt-queue-label` / `.pt-queue-sub`) để không đẩy pill ra ngoài.
3. **Feed audit**: hàng thành lưới ba cột `8px | phần thân | thời gian`. Dòng 1 = tên người thao tác
   (đậm) + nhãn hành động (đã gọi tên mục bị sửa); dòng 2 = **chi tiết thay đổi** dựng bởi
   `src/lib/auditSummary.ts`: cặp `before/after` đọc thành "Trường: giá trị cũ → giá trị mới", enum đi
   qua nhãn chung (`STATUS_META`, role, tier), tiền ra `đ`, phí cấu hình ra `%`, giờ/phút ra đơn vị,
   UUID và id bị giấu, lần lưu không đổi giá trị nào thì kể tên các mục đã sửa ("Đã lưu: …", tối đa
   bốn mục rồi "…"); hết chỗ thì dùng lý do người thao tác nêu, không có gì thì "Không có chi tiết kèm
   theo". Thời gian sang cột phải, hover ra giờ tuyệt đối; cả hàng là link tới `/audit?action=…` nên
   bấm vào là bảng nhật ký lọc sẵn theo hành động đó. Hai dòng thân clamp 2 dòng + tooltip đủ chữ.
4. **Dọn warning lộ ra khi đi kiểm chứng**: AntD v6 báo `Typography.Text` không nhận `ellipsis.rows`
   (dòng 1 audit đổi sang `Paragraph`), và `Space direction` đã deprecated → đổi sang `orientation`
   ở sáu chỗ (AuditLog, Orders, OrderDetail, UserDetail ×3). Console sạch hẳn.

**Kiểm chứng**: Chrome headless + CDP ở 1440×1000 và 390×844, cả sáng lẫn tối — 27 assertion PASS:
hết chữ "trong cửa chờ giải ngân", có chữ mới, còn đúng 3 meter ở tile kỳ; không còn `.pt-queue-money`,
mọi pill phải khớp `^[\d.]+$`, pill giữ hộ = `meta.total` của API (2), ba dòng phụ đều "Tổng … đ";
feed đủ 4 hàng, mỗi hàng 3 cột với `.pt-audit-time` nằm sát mép phải (lệch < 1.5px), thân không tràn
dưới cột giờ, không UUID/enum trần trong chữ, thời gian không lẫn vào thân; bấm hàng audit mở
`/audit?action=user.update` và bảng hiện đúng 2 hàng; mobile tràn ngang 0px; 0 console error.
Ảnh: `r7-light-1440.png`, `r7-dark-1440.png`, `r7-queue-light.png`, `r7-queue-dark.png`,
`r7-light-390.png`, `r7-dark-390.png`, `r7-audit-filtered.png`. Cổng: `tsc -b` 0, oxlint 0 warning,
vitest **59** (thêm 14 test `auditSummary`), `vite build` xanh.

**Dữ liệu kiểm chứng**: để feed có cặp before→after thật, đổi trạng thái hiện diện của
`demo.player2` sang BUSY rồi **trả lại ONLINE ngay** bằng cùng endpoint quản trị (đã đọc lại
`account.status = ONLINE` sau cùng); không đụng tiền, quyền hay khoá tài khoản. Hai bản ghi audit
của lần đổi và lần trả là thao tác thật, nằm lại trong nhật ký đúng như thiết kế của audit trail.

## 53. Ô "Dòng tiền" mới, KPI bấm được, đổi chữ "sạch" và nhãn "Đã huỷ" (R8)

**Phản hồi**: (1) design lại ô tiền — tách hay update tuỳ chọn, nhưng tiêu đề chỉ là **"Dòng tiền"**
(ô cũ để trống một khoảng lớn dưới số sau khi bỏ meter ở §52); (2) bấm mỗi ô KPI phải vào thẳng trang
của phần đó; (3) đổi chữ "sạch" trong "Hàng chờ sạch…" sang từ cùng nghĩa; (4) nhãn status "Đã đóng"
đổi thành "Đã huỷ" cho hợp lý.

**Sửa**:

1. **Ô "Dòng tiền"** (`DashboardPage`): headline = **tổng tiền đang nằm trong nền tảng** =
   `escrowHeld + pendingCredits` (617.000 đ ở DB dev) — hai pool rời nhau nhưng cùng là tiền nền tảng
   đang giữ hộ người dùng, nên cộng lại thành một tổng đúng nghĩa. Bên dưới là **thanh hai dải**
   (vàng = giữ hộ, lục = chờ trả) tách đúng tổng đó, mỗi dải có hover title nêu tên pool và số tiền;
   dòng legend "Giữ hộ 320.000 đ · Chờ trả 297.000 đ"; dòng chữ mờ giữ phí / quyên góp luỹ kế. Hết
   khoảng trống: chân ô lại đủ hai dòng như ba ô kia, và thanh cơ cấu dóng ngang hàng với thanh của ô
   "Người dùng" (đo lệch < 2px).
2. **KPI thành link**: `KpiSurface` nhận `to`; có `to` thì cả ô là `Link` (`.pt-kpi-link`: cursor
   pointer, hover nhấc 2px + viền tint violet, focus ring cho bàn phím, `aria-label="Mở trang …"`),
   không có `to` thì là `div` tĩnh không pointer. Đích: `/users`, `/orders`, `/money`, và ô kiểm duyệt
   mở hàng đầu tiên tier được mở trong `/reports?status=PENDING` → `/ratings?status=FLAGGED` →
   `/feedback?status=PENDING`. SUPER_ADMIN bấm đủ bốn ô; FINANCE chỉ `/orders` + `/money`; MODERATOR
   chỉ `/users` + `/orders` + `/reports` — ô không có quyền không giả vờ bấm được.
3. **Chữ "sạch" → "trống"**: tag hàng chờ "Hàng chờ trống", câu KPI "Hàng chờ trống, không việc tồn".
   Grep toàn admin không còn chữ "sạch".
4. **`CLOSED` → "Đã huỷ"** trong `STATUS_META` (donut legend, bảng đơn, filter, StatusTag dùng chung
   một nhãn nên đổi một chỗ là hết mọi chỗ cũ); `CANCELLED` của rút tiền cũng đọc "Đã huỷ" nhưng nằm
   domain khác, không dropdown nào trộn hai giá trị. Thêm assertion khoá nhãn trong `labels.spec`.
   App mobile vốn đã xếp CLOSED vào họ huỷ (`Cancel`/`Reject`/`OverTime` trong mapper) nên admin giờ
   khớp app.

**Kiểm chứng**: Chrome headless + CDP, 35 assertion PASS ở 1440×1000 và 390×844 cả sáng lẫn tối:
tiêu đề bốn ô đúng "Người dùng|Đơn thuê|Dòng tiền|Hàng chờ kiểm duyệt", headline tiền = tổng API
(617.000 đ), thanh hai dải có title, legend đủ hai pool, chân ô hai dòng đơn và thanh dóng hàng;
bốn href đúng `/users|/orders|/money|/reports?status=PENDING`, cursor pointer, hover đo được
`matrix(1,0,0,1,0,-2)`, **bấm lần lượt từng ô mở đúng trang đó và trang render thật**; hết chữ "sạch",
tag "Hàng chờ trống", legend donut "Đã huỷ", không còn "Đã đóng"; mobile tràn ngang 0px, ô tiền không
cắt chữ; FINANCE/MODERATOR chỉ link những trang có quyền và ô tĩnh không pointer; 0 console error.
Ảnh: `r8-light-1440.png`, `r8-dark-1440.png`, `r8-kpis-light.png`, `r8-money-light.png`,
`r8-money-dark.png`, `r8-light-390.png`, `r8-dark-390.png`, `r8-finance.png`, `r8-moderator.png`.
Cổng: `tsc -b` 0, oxlint 0 warning, vitest **59**, `vite build` xanh; grep chunk thấy `Dòng tiền`,
`pt-kpi-link`, `Hàng chờ trống`, `Đã huỷ` và vắng `Dòng tiền · đang giữ hộ`, `Hàng chờ sạch`, `Đã đóng`.
## 54. Refactor chrome + mọi màn còn lại theo chuẩn Dashboard (R9)

**Phản hồi**: dashboard refactor quá đẹp, yêu cầu làm tương tự cho **tất cả màn còn lại**, gồm cả
header, footer, phần profile, phần chỉnh độ sáng và thanh điều hướng bên trái.

**Sửa**:

1. **Một nguồn cho vị trí màn** — `src/lib/sections.ts` khai `SECTION_META` (14 route → nhóm + tên) và
   `NAV_GROUPS` (Vận hành / Kiểm duyệt / Quản trị). Sider, overline `PageHeader` và breadcrumb header
   cùng đọc từ bảng đó; trước đây ba nơi tự viết chữ riêng nên thêm một trang là phải sửa ba chỗ.
2. **Sider** (`AdminLayout`): brand mark + tên app, nhóm điều hướng để chữ overline, mỗi mục có ô icon
   tint (`.pt-nav-icon`), mục đang chọn = nền tint + **thanh violet 3px sát mép trái** thay cho gạch
   dưới mặc định. Chân sider là chip trạng thái API thật (query `['ops','health']` gọi
   `/api/v2/health`, polling 60s, `retry: 1`) — "API đang hoạt động" / "Đang nối API…" / "Mất kết nối
   API" — cạnh nút thu. `width 232` / `collapsedWidth 68`, bỏ trigger gầm của AntD.
3. **Header** dính 60px + blur: trái là breadcrumb "Nhóm › Tên màn"; phải là **bộ chọn độ sáng** giờ là
   một Segmented ba icon (sáng / tối / theo hệ thống) có tooltip, vạch chia, rồi khối profile. Bấm
   profile mở thẻ: avatar tint, tên, email, **vai trò đọc bằng tiếng Việt** (không còn SUPER_ADMIN trần),
   rồi "Nhật ký thao tác" (chỉ khi tier được mở) và đăng xuất.
4. **Footer** mới: brand mark + "Play Together · Bảng điều hành", chip trạng thái API, bản quyền.
5. **Lớp design dùng chung** trong `index.css`: 14 biến `--pt-*` khai hai lần (sáng/tối) cho mặt nền,
   surface, hairline, thead, hover, overline, ba mức chữ, accent, tint, track Segmented, nền popover.
   Từ đó sinh ra chrome (`.pt-sider/.pt-brand/.pt-nav/.pt-sider-foot/.pt-header/.pt-crumb/.pt-theme-seg/
   .pt-profile/.pt-content/.pt-footer`), khung trang (`.pt-page-head`, `.pt-toolbar`, `.pt-table-card`
   with thead in hoa 11px + ô 12px), pill trạng thái (`.pt-status[data-tone]` 9 tone × 2 bản, thay hết
   `Tag` màu mặc định), trạng thái rỗng (`.pt-empty` icon + câu tiếng Việt) và dải nhận dạng
   (`.pt-identity`). `FilterCard`/`TableCard`/`TableEmpty`/`StatusTag`/`PageHeader` chỉ còn là lớp mỏng
   bọc các class đó, nên 17 màn đổi theo mà không sửa từng màn.
6. **Màn chi tiết** (`UserDetailPage`, `OrderDetailPage`): mở đầu bằng dải nhận dạng (avatar, tên, hai
   pill trạng thái, ba số liệu tóm tắt) và **bỏ những dòng `Descriptions` lặp lại đúng số liệu đó**;
   sáu bảng phụ trong màn chi tiết chuyển sang `scroll={{ x: 'max-content' }}`.
7. **Đăng nhập** (`LoginPage`): trước đây là một thẻ cô độc giữa nền trống; nay là bố cục hai cột —
   trái brand panel (gradient + grain + overline + tiêu đề + ba điểm bán thật), phải thẻ đăng nhập;
   hẹp hơn 991px bỏ brand panel. Dòng chân "Mật khẩu sẽ được đặt lại qua email" là câu tự bịa nên thay
   bằng "Mọi thao tác sau đăng nhập đều được ghi vào nhật ký vận hành." — điều hệ thống thật làm được.
8. **Dọn API cũ của AntD v6** lộ ra khi đi hết màn: `Space direction` → `orientation` (hết chỗ nào dùng
   từ cũ), `Drawer width` → `size`, `Timeline items.children` → `items.content`, `Dropdown dropdownRender`
   → `popupRender`. Console warning biến mất hoàn toàn trong tour.
9. **Năm defect còn lại sau khi tour đã xanh, bắt được bằng cách soi ảnh kết quả**:
   (a) cột Game ở `/orders` không có `width`, `scroll={{ x: 1500 }}` chia nốt 80px còn lại nên tag
   "Age of Empires" vẽ đè chữ cột "Lịch hẹn" — cột Game nay `width: 220`, `x` nâng lên 1720, và
   `.pt-tag-row`/`.pt-tag-fit` quy định **cuộn dòng trước, chỉ ellipsis khi một giá trị đã rộng hơn
   cột**; (b) sider thu còn 68px thì tên nhóm bị cắt thành "V…", "K…", "Q…" đọc như lỗi — ở chế độ thu
   title về `font-size: 0` và thay bằng hairline; (c) `/users/:id` là key RBAC riêng nhưng không có trong
   `SECTION_META`, nên overline và breadcrumb rơi về nhãn dự phòng và màn "Hồ sơ tài khoản" tự giới
   thiệu là **"Bảng điều hành › Tổng quan"** — thêm chính key đó vào bảng với nhãn "Hồ sơ tài khoản";
   (d) sider cũng dùng đúng key đó nên **không mục nào được sáng** khi đang xem hồ sơ — giờ hồ sơ khớp
   về mục "Tài khoản", và tier đọc được hồ sơ mà không có mục đó trong menu (FINANCE) thì **không sáng
   gì cả** thay vì sáng nhầm vào "Tổng quan"; (e) bảng phụ trong hai màn chi tiết vẫn mang header mặc
   định của AntD trong khi mọi bảng danh sách đã dùng chữ in hoa 11px — quy tắc thead/tbody giờ áp cho
   mọi bảng trong `.pt-content`, một ngôn ngữ bảng toàn console.
10. **Khoá bằng test** (`permissions.spec`): mọi đường dẫn thật của router phải phân giải ra một
    `SECTION_META` có cả nhóm lẫn nhãn, và danh sách key trong `NAV_GROUPS` đúng bằng tập key có mục
    menu. Control âm: xoá dòng `'/users/:id'` khỏi bảng là hai test đỏ ngay.

**Kiểm chứng**: tour Chrome headless + CDP qua **17 URL** — quét sáng 1440, quét tối 1440 và quét
khổ 375 — **84 assertion PASS / 0 FAIL**: mọi route render, bấm sider điều hướng đúng trang, thu/mở
232↔68, Segmented đổi sáng↔tối thật, thẻ profile đủ tên–email–vai trò, dòng audit mở được, phân trang
và tab chạy, modal sửa cấu hình mở/đóng được, dải nhận dạng hai màn chi tiết hiện, **tràn ngang 0px ở
42 phép đo**, 0 console error. Hai lượt đo riêng cho phần vừa sửa: 12 assertion cho breadcrumb + overline
+ thead hai màn chi tiết (sáng, tối, 375) và 6 assertion cho màu sáng của sider — SUPER_ADMIN vào hồ sơ
thấy "Tài khoản" sáng, vào đơn thấy "Đơn thuê" sáng, FINANCE vào hồ sơ **không sáng mục nào** nhưng hồ
sơ vẫn render đủ. Scan riêng `.ant-table-cell` trên 32 màn (1440 và 390) bắt mọi ô có con em vẽ ra ngoài
cột: sạch, và control dương chứng minh scanner bắt được overflow cố tình 227px. Ảnh: `r9-login-light`,
`r9-chrome-light`, `r9-chrome-dark`, `r9-profile`, `r9-orders-light`, `r9-orders-dark`, `r9-money-light`,
`r9-config-modal`, `r9-order-detail`, `r9-user-detail`, `r9-users-light`, `r9-maintenance`, `r9-broadcast`,
`r9-mobile-dark`, `r9-mobile-orders-light`, `r9-mobile-login`, `r9b-orders-*`, `r9c-rail-collapsed`,
`r9c-mobile-*`, `r9d-user-detail*`, `r9d-order-detail`.
Cổng: `tsc -b` 0, oxlint 0 warning / 62 file, vitest **61**, `vite build` xanh; grep chunk dist thấy
`pt-crumb`, `pt-sider-foot`, `pt-identity`, `pt-login-art`, `pt-tag-row`, `pt-content .ant-table-thead`,
`Hồ sơ tài khoản`, `ant-menu-inline-collapsed .ant-menu-item-group-title` và vắng `Hàng chờ sạch`,
`pt-queue-money`, `direction:"vertical"`, `items.children`; `dropdownRender` chỉ còn trong code nội bộ
AntD, không còn trong `src/`.

## 55. Chân sider bị bỏ, nút thu/giãn lên header (R10)

**Phản hồi**: hai chỗ ở thanh điều hướng vẫn chưa đẹp — chip "API đang hoạt động" nằm ở **cuối** sider
và nút thu/giãn nằm **dưới đáy**, phải tìm mới thấy; yêu cầu bỏ status ở cuối và đưa nút expand lên
trên cùng để nhìn thấy.

**Sửa**:

1. **Xoá chân sider** (`AdminLayout`): khối `.pt-sider-foot` (chip trạng thái + nút thu, có `border-top`)
   biến mất cùng hai class `.pt-sider-foot` / `.pt-sider-health` trong CSS. Lý do không chỉ là thẩm mỹ:
   một cột điều hướng kết thúc bằng hai phần tử rời nhau, không liên quan gì tới danh sách bên trên,
   đọc như khối chưa làm xong. Trạng thái nền tảng vẫn còn đó nhưng chỉ ở **footer**, nơi nó thuộc về.
   Sider giờ kết thúc ngay sau mục cuối và phần còn lại là nền trống — đúng vai của một cột menu.
2. **Nút thu/giãn sang header**, phần tử đầu tiên của `.pt-header-left` ngay trước breadcrumb, nên nó
   ở trên cùng và nhìn thấy được trong cả hai trạng thái (232px và 68px). `aria-label` vẫn đổi theo
   ("Thu gọn thanh điều hướng" / "Mở rộng thanh điều hướng"), hover có nền tint violet như các nút
   icon khác trong header.

**Kiểm chứng**: tour cũ chạy lại — **86 assertion PASS / 0 FAIL** sau khi thay ba assertion mới thay
cho assertion "sider health" đã hết nghĩa: footer vẫn đọc "API đang hoạt động", `.pt-sider-foot` và
`.pt-sider-health` không còn trong DOM, `.pt-header .pt-collapse-btn` có thật. Probe riêng 12 assertion
đo hình học: nút nằm ở top 15px, cạnh trái breadcrumb 40px ở cả hai trạng thái (expanded left 256 /
crumb 296, collapsed left 92 / crumb 132, ô 30×30), bấm từ header vẫn thu 232→68 và giãn lại, mobile
375px nút vẫn hiển thị, sáng/tối không tràn ngang, 0 console error.
Ảnh: `r10-top-expanded`, `r10-sider-expanded`, `r10-sider-collapsed`, `r10-dark-top`, `r10-mobile-top`.
Cổng: `tsc -b` 0, oxlint 0 warning / 62 file, vitest **61**, `vite build` xanh; grep chunk dist có
`pt-header-left` và **vắng** `pt-sider-foot` / `pt-sider-health`.

## 56. Ô icon trong thanh điều hướng bị lệch so với nền tint (R11)

**Phản hồi**: "icon chỗ này với màu đằng sau đang bị lệch, không cân đối" — ảnh chụp khoanh cả cột icon
của sider.

**Đo trước khi sửa** (`getBoundingClientRect` của tile / `.anticon` / `svg`, 14 mục): tile 26×26 nhưng
glyph 14×14 nằm **sát mép trái** — `dLeft 0 / dRight 12`, tâm lệch ngang **6px**; chiều dọc thì đúng
(6/6). Nguyên nhân không phải `margin`: AntD áp `display: flex` lên slot icon nên `place-items: center`
của `.pt-nav-icon` **không có tác dụng**, icon bị đẩy về đầu dòng theo `justify-content: flex-start`.

**Sửa** ([index.css](file:///C:/Users/manhk/Documents/Projects/play-together/play-together-admin/src/index.css)):

1. Khai lại cách căn giữa bằng cặp tính chất mà flex cũng hiểu — `.pt-nav.ant-menu .ant-menu-item
   .pt-nav-icon { align-items: center; justify-content: center; flex: none; }` — và giữ `place-items`
   cho trường hợp lưới. `flex: none` để ô không bị co khi item là flex container.
2. **Rail thu 68px lệch theo một nguyên nhân khác**: AntD tính bề rộng item theo glyph 16px, nên item chỉ
   34–42px và nằm lệch phải trong rail (tâm tile 38 so với tâm rail 34). `.pt-nav.ant-menu-inline-collapsed`
   bỏ inset của menu, item `width: auto; margin-inline: 8px` đối xứng + `justify-content: center`, vì vậy
   pill selected vẫn thụt vào đều hai bên như bản mở rộng thay vì tràn sát mép.

**Kiểm chứng**: probe hình học chạy trên **14 mục × 2 trạng thái × 2 bản màu** — mọi tile có
`|dLeft − dRight| ≤ 0.5` và `|dTop − dBottom| ≤ 0.5` (kết quả trả về mảng rỗng cả bốn lượt), tâm tile
trùng tâm vùng trong của rail thu (sai lệch < 0.75px). Tour cũ **86 assertion PASS / 0 FAIL**, 0 console
error. Ảnh: `r11-rail-light`, `r11-rail-dark`, `r11-rail-light-collapsed`, `r11-rail-dark-collapsed`.
Cổng: `tsc -b` 0, oxlint 0 warning / 62 file, vitest **61**, `vite build` xanh; grep CSS chunk thấy
`ant-menu-inline-collapsed .ant-menu-item .pt-nav-icon` và `margin-inline:8px`.

## 57. Donut "Cơ cấu trạng thái đơn" thiếu hai trạng thái, và hai nhãn trục ngày chồng nhau (R12)

**Phản hồi**: bộ lọc trạng thái ở `/orders` có bảy lựa chọn nhưng thẻ donut trên dashboard chỉ liệt kê
năm; người dùng hỏi hai trạng thái còn lại có cần hiển thị không, quyền quyết định thuộc về bên làm.

**Kết luận trước khi sửa**: đây **không phải mất dữ liệu**. `GET /admin/stats` trả `orders.byStatus` dựng
bằng `GROUP BY`, nên trạng thái không có đơn nào **không xuất hiện thành khoá** — đo trên DB dev kỳ 30
ngày: `{SCHEDULED:1, LIVE:1, ENDED_EARLY:18, SETTLED:163, CLOSED:2}`, và `PROPOSED` / `DISPUTED` có
`meta.total = 0` khi lọc thật.

**Quyết định: nên hiển thị**, vì (1) donut là chỗ người vận hành đọc vòng đời đơn, thiếu hai tên khiến
họ nghi hệ thống ẩn trạng thái; (2) "Chờ đối soát: 0" **là thông tin** — không có đơn kẹt ở khâu quyết
toán; (3) bảng legend thành danh sách cố định, không nhảy hàng mỗi lần đổi kỳ.

**Sửa** ([DashboardPage.tsx](file:///C:/Users/manhk/Documents/Projects/play-together/play-together-admin/src/pages/DashboardPage.tsx)):

1. `orderRows` dựng từ **`ORDER_STATUSES`** (đúng enum bộ lọc đang dùng, sinh từ OpenAPI) với
   `count = byStatus[status] ?? 0`, thay vì `Object.entries(byStatus)`. Legend và dropdown giờ **trùng
   nhau theo cấu trúc**, không thể lệch nhau vì hai nguồn số.
2. Dòng 0 đơn mờ hơn (`.pt-legend-zero`, opacity 0.55, số không in đậm) nhưng **vẫn bấm được** và vẫn
   link `/orders?status=…` — lọc ra bảng rỗng là thao tác hợp lệ, không phải ô điếc.
3. Donut không vẽ cung 0% (component đã bỏ `value === 0` từ trước) nên hình không đổi; tổng ở giữa vẫn
   đúng. Điều kiện rỗng đổi từ `orderRows.length === 0` (không còn bao giờ đúng) sang `orderTotal === 0`
   với câu "Kỳ này chưa có đơn nào ở mọi trạng thái."
4. **Defect bắt được khi soi ảnh kết quả**: trục ngày của "Diễn biến đơn thuê" in `7/11` sát `8/11` thành
   một chuỗi không đọc được. Nguyên nhân: nhãn cách đều `ceil(n/8)` theo số thứ tự **và** ép nhãn cuối
   cùng, nên hai nhãn cách nhau đúng một bước. Sửa thành đếm bước **ngược từ ngày cuối**
   (`(n-1-i) % labelEvery === 0`) — tám nhãn đều nhau 93,2px và ngày cuối vẫn có nhãn.

**Kiểm chứng**: 17 assertion — legend đủ **7 dòng**, mỗi dòng khớp `byStatus` từng số, đúng hai dòng mờ
là `Chờ chấp nhận` và `Chờ đối soát` với `0 / 0%`, donut vẫn vẽ **5 cung**, bấm dòng 0 đơn chuyển sang
`/orders?status=PROPOSED` và bảng hiện "Chưa có đơn thuê nào khớp bộ lọc" chứ không đứng im; khoảng
nhỏ nhất giữa hai nhãn trục **93,2px** (control âm: luật cũ cho ra `7/118/11`), nhãn cuối đúng ngày hôm
nay, tổng số nhãn 8. Chạy ở 1440 sáng, 1440 tối và 375px, tràn ngang 0. Tour cũ **86 PASS / 0 FAIL**.
Ảnh: `r12-donut-light`, `r12-donut-dark`, `r12-donut-mobile`.
Cổng: `tsc -b` 0, oxlint 0 warning / 62 file, vitest **61**, `vite build` xanh; grep chunk dist thấy
`pt-legend-zero`.

## 58. Quét a11y/motion do extension Chrome báo: 2 "AI tell" + 17 lỗi chất lượng (R13)

**Phản hồi**: người dùng dùng một extension quét page và dán nguyên danh sách — 2 "AI tells" (easing
nảy `cubic-bezier` overshoot, dot nhấp nháy vô hạn) và 17 "quality issues" (chữ tương phản thấp, chữ
10,5px, `transition: width`, chữ trắng trên nền vàng, chữ trắng trên gradient tím, Segmented trong
hero…). Yêu cầu: sửa **tất cả**.

**Cách làm**: không tin kết quả quét một trang, nên dựng một scanner riêng
(`scan-a11y.cjs`) đi qua **mọi node chữ** trên **16 route × 2 giao diện + trang login × 2 = 34 trạng
thái**, tự tính relative luminance theo sRGB, cộng màu alpha lên đúng mặt nền thật rồi so với ngưỡng
4,5:1 (miễn trừ chữ lớn ≥24px và control bị khoá theo WCAG 1.4.3), đồng thời quét `transition` trên thuộc tính layout,
`animation-iteration-count: infinite` và mọi `cubic-bezier` có control point ngoài [0,1] trong CSSOM. Con số đi xuống theo từng đợt sửa: **291 → 158 → 52 → 37 → 2 → 0**.

**Sửa — motion**:

1. Hai đường cong nảy trong `theme.ts` (`motionEaseOutBack: …,1.46)` / `motionEaseInBack: …,-0.46)`) đổi
   sang `cubic-bezier(0.22, 1, 0.36, 1)` và `cubic-bezier(0.64, 0, 0.78, 0)`; scanner đếm **0** đường cong
   overshoot còn sót trong CSSOM.
2. Bỏ `@keyframes pt-pulse` và khỏi danh sách `prefers-reduced-motion`: dot trạng thái đứng yên.
3. Bỏ hẳn animation trên thuộc tính layout do **code mình** viết: thanh rail thu/giãn dùng
   `transition: none` (kể cả icon/nhãn menu vốn interpolate `margin`), `Meter` chuyển từ
   `transition: width` sang `transform: scaleX()` — mỗi frame trước đây reflow cả lưới KPI.
   Scanner hiện báo **0** layout-animation trong dòng chảy. 12 mục còn lại là của AntD và **bị giới hạn
   trong hộp riêng** (ink bar `position: absolute`, `left` trên `th` không định vị, icon Steps) nên tách
   sang mục "contained" thay vì gọi là lỗi — có control chứng minh luật mới vẫn bắt được lỗi thật.

**Sửa — cỡ chữ**: `.pt-nav .ant-menu-item-group-title` và `.pt-crumb-group` 10,5px → **11px**; số chữ
dưới 11px trên toàn bộ 34 trạng thái = **0**.

**Sửa — tương phản** (phần nặng nhất). Phát hiện quyết định: **thuật toán tối của AntD kéo mọi seed màu
xuống ~0,87 lần** trước khi ra pixel — đo bằng probe: seed `#8882DB` render thành `#7772BD`, seed
`#9C95FC` render thành `#8882D9`. Vì vậy seed phải đặt = giá trị 4,6:1 chia ngược lại, và kiểm chứng trên
màu **đã render**, không trên mã nguồn.

| Chỗ | Trước | Sau | Đo lại |
|---|---|---|---|
| `colorSuccess` sáng (dấu "+", pill "Hoàn thành", "Hàng chờ trống, không việc tồn") | `#188A54` 4,37 | `#157B4B` | 5,29 |
| `colorWarning` sáng | `#B4740F` 3,86 | `#95600C` | 5,31 |
| 5 tone pill sáng (green/gold/blue/orange/volcano) | 3,4–4,5 trên **chính nền tint** | giải lại từng cặp | ≥4,6 |
| `colorLink` tối | render `#6158CF` 3,12 | seed `#9C95FC` → `#8882D9` | 5,11 |
| `colorError` tối (nút "Khoá") | render `#A62E34` 2,51 | seed `#F67C82` | 5,11 |
| `colorSuccess`/`colorWarning` tối | 3,13 / 4,46 | seed `#24B972` / `#DF9012` | 5,19 / 5,11 |
| Tab đang chọn + số trang hiện tại (tối) | `colorPrimary` 3,12 | `Tabs.itemSelectedColor` / `Pagination.itemActiveColor` = `#A9A2F7` | 6,6 |
| Chữ trên nút danger đặc (tối) | trắng trên `#D66C71` = 3,36 | `Button.dangerColor` = `#171632` | 5,2 |
| `.pt-avatar-initial` (chữ cái đầu tên, danh sách "Đơn mới nhất") | tím `#6E64F0` 3,81–4,43 | token `--pt-avatar-fg` theo giao diện (`#5D51EE` / `#A9A2F7`) | 4,66 / 6,0 |
| 2 `Tag` định sẵn màu trong head card | nền tint của AntD, 2,56–3,39 | đổi sang pill `.pt-status` của thương hiệu | ≥4,6 |
| Sao đánh giá trung bình | `#F4B740` trên trắng = 1,80 | `--pt-star` (`#B4740F` sáng / `#F4B740` tối) | 3,86 (đồ hoạ, ngưỡng 3) |
| Overline, `--pt-text-3`, hero + login gradient, Segmented trong hero | 1,9–4,2 | đã siết ở đợt trước của vòng này | ≥4,5 |

**Lỗi bắt được mà extension không thấy**: công tắc "Bật bảo trì để tạm khoá hệ thống" ở `/maintenance`
vẽ nhãn trắng trên nền `rgba(0,0,0,0.25)` = **1,83:1**. AntD lấy màu nền track từ `colorTextQuaternary`,
nên override trong phạm vi component: `Switch.colorTextQuaternary = #6F7484` (trắng trên đó 4,66),
hover `#5A5F73`.

**Có chủ đích không sửa**: nút "Khoá" ở dòng tài khoản của chính người đang đăng nhập bị disable và chỉ
đạt 1,83:1 — WCAG 1.4.3 miễn trừ control không dùng được, và làm cho nó *sẫm* hơn sẽ khiến nó trông như
bấm được. Scanner loại trừ `[disabled]`, và test có guard để việc loại trừ đó không làm mờ lỗi thật.

**Control (chứng minh bộ đo còn sống)**:
- Luật layout-animation: tiêm `.pt-kpi { transition: width }` → scanner bắt 4 phần tử; chuyển đúng các
  phần tử đó sang `position: absolute` → **0** (đúng tinh thần "chỉ tính khi reflow anh em hàng xóm");
  gỡ tiêm → 0. 4/4 PASS.
- Bộ đo tương phản: trên `/users` sáng, số lỗi = 0 dù **có** control bị disable trên màn hình; tiêm
  `.pt-crumb-group { color: #EFEFEF }` → scanner bắt ngay.
- Test mới `src/__tests__/designContract.spec.ts` (**28** assertion: 18 tone pill đọc thẳng từ
  `index.css`, 4 seed tối sau khi nhân hệ số re-shade, 3 mực chữ sáng, 2 đường cong motion). Control âm:
  làm hỏng `dark/green` + `colorLink` + hai easing → **4 fail**; khôi phục → 28 pass.

**Vướng mắc ghi lại cho lần sau**: `?raw` trong Vitest trả về **chuỗi rỗng** vì Vitest stub CSS mặc định;
phải bật `test.css: true` trong `vite.config.ts`. Nếu không có assertion `toHaveLength(18)`, test sẽ im
lặng xanh trên dữ liệu rỗng.

**Kiểm chứng cuối vòng**: scanner `{"contrast":0,"tiny":0,"anim":0,"ease":0}` (12 contained); tour cũ
**86 PASS / 0 FAIL** (17 URL × 3 khổ, 0 console error); probe màu 12 assertion
`R13 PROBE GREEN`; cổng `tsc -b` 0, oxlint **0 warning / 63 file**, vitest **89**, `vite build` xanh.
Ảnh đối chiếu không thoái hoá: `r13-dash-light`, `r13-dash-dark`, `r13-money-dark`, `r13-maint-light`,
`r13-users-light`.

## 59. Chọn "Hôm nay": chữ trống đè lên trục toạ độ, và một `transition: width` còn sót (R14)

**Phản hồi**: người dùng quét lại bằng extension và báo 2 mục — (1) *text occluded by an overlapping
element* ở thẻ "Diễn biến đơn thuê", câu "Chưa có đơn thuê nào trong kỳ này" bị che **100%** bởi svg,
chỉ hiện khi chọn kỳ **Hôm nay**; (2) *layout property animation* cấp trang: `transition: width`.

**Đo trước khi sửa** (`probe-r14.cjs`, kỳ = Hôm nay): `div` câu trống và `svg` biểu đồ **trùng khít
726×240px**, `overlapPctOfMessage = 100`, và trong svg vẫn vẽ **5 đường lưới + nhãn trục 0 1 2 3 4 +
8 nhãn giờ**. Nghĩa là extension đúng theo nghĩa sâu hơn nó nói: kỳ không có đơn thì biểu đồ chỉ còn
một bộ trục vô nghĩa, còn câu chữ in đè lên đó (nền suốt).

**Sửa**:

1. [charts.tsx](file:///C:/Users/manhk/Documents/Projects/play-together/play-together-admin/src/components/charts.tsx):
   khi `empty`, component **trả về trạng thái rỗng của nhà** (`TableEmpty` icon + chữ) ở đúng chiều cao
   biểu đồ, thay vì `position:absolute; inset:0` phủ lên svg. Không vẽ trục, không overlay → hết đè,
   và chiều cao giữ nguyên nên đổi kỳ không làm lưới thẻ nhảy.
2. Cùng kỳ rỗng, legend "Tạo mới / Hoàn thành" ở đầu thẻ cũng **ẩn theo** (`points.some(...)`) — nó mô
   tả hai đường không còn được vẽ.
3. `transition: width` của Segmented: AntD cho cái thumb trượt **vừa interpolate `transform` vừa
   `width`** (rule `.ant-segmented .ant-segmented-thumb-motion-appear-active`), nên mỗi lần đổi kỳ cả
   control reflow. Override giữ `transform`, bỏ `width` ([index.css](file:///C:/Users/manhk/Documents/Projects/play-together/play-together-admin/src/index.css)).
   Đo giữa animation: thumb `transitionProperty = transform`, `width` đã bằng **92px = khổ đích** trong
   khi `x` còn ở 1242px (vị trí cũ) → trượt đúng, không co giãn.
4. **Bắt được khi soi ảnh cùng kỳ**: donut "Cơ cấu trạng thái đơn" báo 185 đơn ngay cạnh biểu đồ "kỳ
   này chưa có đơn". Nguyên nhân không phải lỗi số liệu: `GET /admin/stats` **không nhận tham số nào
   cả** (`admin.controller.ts:55 overview()` — không query DTO), nên KPI + donut là **toàn bộ thời
   gian**, còn biểu đồ + thẻ "Trong kỳ" lấy từ `/admin/orders?from=` và `/admin/kpis?from=` nên đúng là
   theo kỳ. Thẻ donut nay nói rõ phạm vi của mình: `extra` = pill "Toàn bộ thời gian"; câu rỗng ở legend
   donut (đã viết "Kỳ này chưa có đơn nào ở mọi trạng thái." ở §57) đổi thành "Chưa có đơn thuê nào ở
   bất kỳ trạng thái nào."

**Còn để ngỏ (đã đo, chưa sửa vì đổi ý nghĩa số liệu)**: thẻ KPI "Đơn thuê" vẽ **số lớn toàn bộ thời
gian** (185) cạnh **sparkline theo kỳ** — hai con số trong cùng một thẻ không cùng phạm vi. Cách sửa
sạch là cho `/admin/stats` nhận `from`/`to` (thay đổi hành vi API, cần người dùng duyệt) hoặc bỏ
sparkline khỏi thẻ KPI tổng quan.

**Kiểm chứng**: `probe-r14b.cjs` **12/12 PASS** — kỳ rỗng: không còn `text`/`line` nào trong svg thẻ
(`texts=0 lines=0`), legend ẩn, pill phạm vi hiện; kỳ có dữ liệu: vẽ lại (`texts=12 lines=5`), legend
hiện lại 2 mục; thumb `transform`-only và khớp khổ đích; rail vẫn `transition: none`; bản tối cũng bỏ
trục; 375px@3x tràn ngang **0** và empty state nằm gọn trong viewport (mép phải 338/375). Tour cũ chạy
lại sau khi đổi cấu trúc dashboard: **86 PASS / 0 FAIL**; scan a11y 34 trạng thái vẫn
`contrast 0 / tiny 0 / anim 0 / ease 0` (12 contained). Cổng: `tsc -b` 0, oxlint 0 warning / 63 file,
vitest **89**, `vite build` xanh; grep chunk dist thấy rule thumb `transition:transform` và chuỗi
"Toàn bộ thời gian". Ảnh: `r14-empty-today.png`, `r14-empty-today-dark.png`, `r14-empty-today-375.png`,
`r14-thumb-mid.png`.

## 60. "7 ngày qua" vẽ to bất thường sau khi load page: viewBox đóng băng ở giá trị placeholder (R15)

**Phản hồi**: người dùng gửi hai ảnh thẻ "Diễn biến đơn thuê" cùng kỳ **7 ngày qua**. Ảnh 1 (lỗi, "lâu
lâu mới bị một lần"): nét đậm hơn, chữ trục **béo và ngang** hơn, nhãn 0/2/4/6/8 dạt sát mép trái trong
khi vạch lưới bắt đầu sâu hơn. Ảnh 2 là dạng đúng.

**Vì sao nhìn ảnh là ra bệnh**: svg vẽ bằng `viewBox="0 0 W 240"` + `preserveAspectRatio="none"`, nên
**mọi thứ trong svg bị kéo ngang theo `box/W`**; chiều dọc giữ nguyên 240px. Ảnh 1 có mép phải và các
nhãn ngày gần như không đổi nhưng **chữ và lề thì nở ra** → `W` nhỏ hơn bề rộng thật, không phải lỗi
font. Con số khớp chính xác: `W = 760` (placeholder của `useState`) trong box **1046px** ở viewport
1920 → tỉ lệ **1,376**.

**Nguyên nhân** (chính là lỗi do vòng R14 gây ra): [charts.tsx](file:///C:/Users/manhk/Documents/Projects/play-together/play-together-admin/src/components/charts.tsx)
đo bề rộng container bằng **`useLayoutEffect(..., [])`** — chạy đúng một lần lúc mount — trong khi R14
thêm nhánh `if (empty) return <div …>` **không gắn ref**. Chuỗi sau làm `measured` đóng băng vĩnh viễn:

1. load page → bấm **Hôm nay** khi request series đầu tiên chưa về → lần mount **đầu tiên** của
   `TrendChart` rơi vào nhánh trống → `wrapRef.current === null` → **không đo, không observe**;
2. bấm lại **7 ngày qua** → dữ liệu kỳ này **đã cache** nên không hiện skeleton → component **không
   mount lại** → effect một-lần không chạy tiếp → vẽ `viewBox 760` trong box 1046px.

Kỳ "Hôm nay" thật sự rỗng: `GET /admin/orders?from=<00:00+07 hôm nay>` trả `total = 0`, trong khi toàn
bộ thời gian là 185 đơn.

**Sửa**: thay effect mount-only bằng **callback ref** `attach` — phần tử nào được gắn vào DOM thì ngắt
observer cũ, đọc `clientWidth` **ngay** (trước paint) và observe chính phần tử đó; `useEffect(() =>
() => observer.current?.disconnect(), [])` dọn dẹp khi unmount; `onMove` dùng `event.currentTarget`
thay vì đọc ref. Điều kiện cần: **phần tử được đo phải tồn tại ở cả hai nhánh render** (đã ghi vào
DESIGN.md §5.5).

**Control dương tính** — dựng lại nguyên văn code trước sửa (useLayoutEffect `[]` + `wrapRef`, nhánh
trống không ref) rồi chạy đúng gesture *load → bấm Hôm nay → bấm 7 ngày qua*:

| bản | `probe-r15i` (dev 5174) | `probe-r15j` (dist + `vite preview` 4178) |
|---|---|---|
| trước sửa | **6/6 STRETCHED** — 1440: `vbW 760 / boxW 726` = 0,955; 1920: `vbW 760 / boxW 1046` = **1,376** | **6/6 STRETCHED**, cùng cặp số (bấm được "Hôm nay" ở 182–291ms sau navigate) |
| sau sửa | **6/6 `ratio = 1,000`** (726/726, 1046/1046) | **6/6 `ratio = 1,000`** |

Kiểm chứng qua **artifact** (không chỉ qua dev server): build bản lỗi → probe → build bản sửa → probe,
trên cùng một server preview; ảnh `shots/r15-fixed-card-1920.png` (thẻ chụp từ dist đã sửa,
`vbW=1046 boxW=1046`) đặt cạnh `shots/r15-repro-1920-0.png` (dist bản lỗi). Probe phụ
`probe-r15c.cjs` **4/4 PASS**: kỳ trống → đổi khổ cửa sổ → quay lại, viewBox vẫn khớp, cả hai chiều.

**Vòng control hỏng trước khi có bảng trên, ghi lại để lần sau khỏi lặp**:
- Control đầu tiên **xanh giả** vì chạy `node probe…` và `cp khôi phục` trong **cùng một lô song song**;
  bản sửa đã được ghi đè lên file trước khi Chrome kịp tải trang. Control phải chạy **tuần tự**.
- Control thứ hai ("cho `attach` chỉ nối observer một lần") **không tái hiện được** lỗi, vì đổi kỳ là
  `TrendChart` **mount lại** (skeleton 240px xen giữa ở `DashboardPage.tsx:353`) → state `measured` reset
  → nhánh nào cũng đo lại. Chỉ hình thái **mount-only effect + nhánh trống không ref** mới sinh bệnh.
- Bộ đo "mọi frame" (MutationObserver + rAF, `probe-r15e/f/g`) báo `chartFrames = 0` **không phải** vì hết
  lỗi: probe quên ghi `pt_admin_user`, nên Chrome nằm ở trang login và không có svg nào để soi. Một máy
  đo phải tự chứng minh nó thấy được đối tượng (assert `chartFrames > 0`) trước khi được tin.

**Kiểm chứng cuối vòng**: `tsc -b` 0 lỗi, oxlint **0 warning / 63 file**, vitest **89/89**, `vite build`
xanh, scan a11y `{"contrast":0,"tiny":0,"anim":0,"soft":12,"ease":0}`. Tour `shoot-r9.cjs` **86 PASS /
0 FAIL** ở 7 lượt liên tiếp; **một** lượt sớm hơn báo `TOUR FAILURES 2` nhưng tên hai assertion bị mất vì
output bị pipe qua `tail` — chưa loại được flaky, cần giữ log đầy đủ khi chạy lại.

## 61. Màn Đơn thuê: kicker "Vận hành", bảng áp sát viền card, và transition trên thuộc tính layout (R16)

**Phản hồi**: `/orders` còn 1 AI tell (kicker "Vận hành" nằm trên h1 "Đơn thuê") và 2 quality issues
(thẻ `div.ant-card.ant-card-bordered` có con áp sát border+bg trên/trái, không inset; `transition: height`
cấp trang).

**Đo trước khi sửa** (`probe-r16.cjs`, `/orders` @1440):
- Kicker `.pt-page-overline` = "Vận hành", 11px, `letter-spacing 1.54px`, màu `rgb(91,81,216)`. Cùng màn,
  **breadcrumb header đã in "Vận hành › Đơn thuê"** ([AdminLayout.tsx:167](file:///C:/Users/manhk/Documents/Projects/play-together/play-together-admin/src/app/AdminLayout.tsx#L167))
  và **tiêu đề nhóm ở sider** cũng in "Vận hành" → một chữ xuất hiện ba lần, hai lần chỉ để trang trí.
- `.ant-card-body` của thẻ bảng có `padding: 0px`; `ant-table-wrapper` so với card chỉ cách **top 1px /
  left 1px** (đúng 1px viền). Đo 4 màn `/orders`, `/users`, `/money`, `/audit`: cùng số 1/1 → extension
  gọi là "children flush against border+bg on top/left" là có thật.
- Phần tử **đang thật sự** transition thuộc tính layout: **2 node** `th.ant-table-cell.ant-table-column-has-sorters`
  với `transition: all 0.3s, left 0s` — `all` ôm trọn height/padding/width.

**Sửa**:

1. **Bỏ kicker** ([PageHeader.tsx](file:///C:/Users/manhk/Documents/Projects/play-together/play-together-admin/src/components/PageHeader.tsx)):
   component thôi đọc `SECTION_META`/`sectionOf`, không còn in overline; rule `.pt-page-overline` xoá khỏi
   CSS và `margin-top` của `.pt-page-title` từ `4px` → `0` (khoảng trống đó vốn để đẩy chữ khỏi kicker).
   Kiểm tra không mất thông tin: với 5 route, `.pt-crumb-group` vẫn là "Vận hành"/"Quản trị"/"Bảng điều
   hành" và nhóm menu vẫn đủ ["Vận hành","Kiểm duyệt","Quản trị"].
2. **Lót lưới vào trong card** ([TableCard.tsx](file:///C:/Users/manhk/Documents/Projects/play-together/play-together-admin/src/components/TableCard.tsx)):
   `styles={{ body: { padding: 0 } }}` → padding nằm trong CSS `.pt-table-card.ant-card > .ant-card-body`
   = **12px 16px 16px**; bản ≤575px thu còn **10px 12px 12px** vì card điện thoại chỉ rộng 275px, 16px
   mỗi bên sẽ ăn mất một cột. **Đưa padding từ inline style về CSS là bắt buộc**: inline thắng media query.
   Số đo sau sửa: inset **top 12 / left 16 / right 16**; chữ cột đầu "NGƯỜI THUÊ" từ **17px → 33px** tính
   từ viền card, trong khi hàng điều khiển phía trên ở 29px → lệch 4px (trước lệch 12px). Phân trang
   `margin: 12px 16px` → `12px 0 0` vì body đã có 16px.
3. **Cắt transition thuộc tính layout** ([index.css](file:///C:/Users/manhk/Documents/Projects/play-together/play-together-admin/src/index.css)):
   - `th.ant-table-column-has-sorters`: `all` → chỉ `background-color, color` (đúng cái nó cần: fade khi hover);
   - `.ant-select-multiple .ant-select-content .ant-select-selection-item`: `height, line-height, padding`
     → `opacity`. Đây là tag nhiều chọn ở drawer "Thêm game" (Danh mục game) — mỗi lần chọn là cả toolbar
     bị reflow theo chiều cao tag. Đo lại trên node thật sau override: `transitionProperty = opacity, 0.3s`.

Vì sao override thắng AntD: AntD inject `<style>` lúc chạy nên **thứ tự nguồn không bảo đảm**, nên các rule
trên tăng specificity (`.pt-content .ant-table-wrapper ...` = 4 class; `... .ant-select-selection-item.ant-select-selection-item`
lặp một class).

**Control của máy đo** — `scan-a11y.cjs` có thêm bộ đếm **`layRule`**: quét **CSSOM** (đúng cách extension
đọc) mọi rule transition `width/height/margin/padding` mà selector **thật sự khớp node đang render**, thay
vì chỉ nhìn computed style. Chuỗi control 3 bước (`probe-r16ctl.cjs`): baseline `/orders` = **4** rule
(tất cả là `.ant-menu` nội bộ, đã bị khối rail `transition: none` vô hiệu ở computed) → tiêm
`.pt-toolbar { transition: height 0.3s ease }` → **5** rule, bắt đúng 1 (`.pt-toolbar :: height`) → gỡ tiêm
→ **4**. PASS.

**Điểm tự sửa lại lời cũ**: bộ đếm `anim` ở §58/§59 loại `transition: all` với lý do "AntD khai all để
đổi màu, không reflow" — đúng với nút, nhưng **sai** với header bảng có sort: `all 0.3s` ở đó animate cả
height/padding. Vì vậy `anim: 0` hồi đó **không** chứng minh CSSOM sạch; nay `layRule` phủ cả hai cách đọc.

**Còn để ngỏ (cần quyết định, chưa làm)**: 4 rule `.ant-menu` (width/padding/font-size/margin) vẫn nằm
nguyên trong **text** của CSSOM, dù computed style của mọi node menu đã là `transition: none`. Nếu extension
đọc text thay vì giá trị computed, nó vẫn có thể báo "transition: width/height" cấp trang. Cách duy nhất
tắt hẳn là hạ token motion của Menu (`motionDurationSlow: '0s'` hoặc `motion: false`) — mất luôn fade khi
hover menu, nên chờ bạn chọn.

**Kiểm chứng cuối vòng**: `tsc -b` 0 lỗi, oxlint **0 warning / 63 file**, vitest **89/89**, `vite build`
xanh; grep **artifact** `dist/assets/*.css` thấy cả 4 rule mới (`padding:12px 16px 16px`,
`padding:10px 12px 12px`, header `transition:background-color...,color...`, tag `transition:opacity...`)
và **không còn** chuỗi `pt-page-overline` trong CSS lẫn JS (control âm); scan a11y 16 route × 2 giao diện
`{"contrast":0,"tiny":0,"anim":0,"soft":10,"layRule":134,"ease":0}` — `soft` giảm 12 → 10 vì hai header
sort không còn `all, left`, `layRule` là con số CSSOM đã giải thích ở trên; tour `shoot-r9.cjs` **86 PASS /
0 FAIL** (log đầy đủ `tour-r16.log`, `tour-r16b.log`). Ảnh: `r16-orders-before.png` / `r16-orders-after.png`
(cùng kỳ, cùng khổ), `r16-orders-light.png`, `r16-audit-dark.png`, `r16-orders-375.png` (tràn ngang 0).

## 62. Gạch dưới tab trên /money và /withdrawals animate `width` (R17)

**Phản hồi**: extension Chrome còn một mục *Layout property animation* tại
`div.ant-tabs-ink-bar.ant-tabs-ink-bar-animated`: `transition: width`, trên `/money` và `/withdrawals`.

**Đo trước khi sửa** (`probe-r17.cjs`, 1440): ink bar có `position: absolute`, computed
`transitionProperty = "width, left, right"` với `0.3s` cho cả ba; AntD ghi inline `width: 52.53px;
left: 26.27px; transform: translateX(-50%)`. Ba màn thật sự có Tabs: `/money` (4 tab), `/withdrawals` (2),
`/catalog` (2).

**Đây là chỗ lời bào chữa cũ sai**: §58 xếp ink bar vào nhóm 12 mục "contained" với lý do *phần tử absolute
thì không reflow hàng xóm*. Đúng về mặt hiệu năng, nhưng **thuộc tính được animate vẫn là thuộc tính
layout**, và máy quét đọc chính cái đó. Nên lần này bỏ hẳn ngoại lệ thay vì giải thích tiếp.

**Vì sao không giữ chuyển động bằng `transform`**: AntD định vị gạch bằng inline `left` + `width`
(`transform` chỉ dùng `translateX(-50%)` để căn giữa, không đổi theo tab), nên override
`transition: transform` sẽ không animate gì cả. Kiểm tra tín hiệu còn lại trước khi chấp nhận gạch đứng
yên: **nhãn tab đang chọn đổi màu violet** `rgb(91,81,216)` trong khi tab không chọn là
`rgba(0,0,0,0.88)` → gạch không phải tín hiệu duy nhất, mất trượt không mất ngữ nghĩa.

**Sửa** ([index.css](file:///C:/Users/manhk/Documents/Projects/play-together/play-together-admin/src/index.css)):
`transition: none` cho `.ant-tabs .ant-tabs-nav .ant-tabs-ink-bar.ant-tabs-ink-bar-animated` — bốn class
để thắng rule của AntD (ba class, inject lúc chạy nên không thể dựa thứ tự nguồn).

**Kiểm chứng** (`probe-r17ink.cjs`, **9/9 PASS**): duyệt **từng tab** trên cả ba màn, so rect của gạch với
rect của tab đang chọn → `barCenterOffset = 0`, `widthDelta = 0`, `visible = true`, `dur = 0s` ở cả 8 tab;
switch sang "Giữ hộ" trên `/money` đổi bảng thật (header `|Loại|Chiều|Tài khoản|…` →
`Người thuê|Người chơi|Đơn|…`). Ảnh `r17-money-tabs.png` (trước) và `r17-money-tabs-after.png` (sau).
Scan a11y sau sửa: `{"contrast":0,"tiny":0,"anim":0,"soft":4,"ease":0}` — **soft 10 → 4**, đúng 6 mục
ink bar (3 màn × 2 giao diện) biến mất; 4 mục còn lại là `div.ant-steps-item-icon` ở màn chi tiết đơn,
transition `background-color, border, color, inset, transform` — `inset` không đổi giá trị trong thực tế nên
để lại, đã ghi rõ để vòng sau không phải đoán.
Cổng: `tsc -b` 0, oxlint 0 warning / 63 file, vitest **89/89**, `vite build` xanh, tour `shoot-r9.cjs`
**86 PASS / 0 FAIL** (`tour-r17.log`), grep artifact thấy
chuỗi `ant-tabs-ink-bar.ant-tabs-ink-bar-animated{transition:none}` trong chunk CSS.

**Rule đã cập nhật**: DESIGN.md §5.7 viết lại — *không animate thuộc tính layout, kể cả trên phần tử
absolute*; indicator cần trượt thì dùng `transform`, không làm được thì đứng yên.

## 63. CI đỏ sau push: container không tạo được `/app/uploads`, API chết ngay khi boot (lỗi do mình push)

**Triệu chứng** (run `42490839858`, job `Build and smoke-test the image`): 30 lần probe `curl
http://localhost:3000/api/v2/health` đều `curl: (7) ... Could not connect`, sau đó `docker logs` in ra

```
Application failed to start: Error: EACCES: permission denied, mkdir '/app/uploads'
    at mkdirSync (node:fs:1370:26)
    at serveUploadsFrom (/app/dist/common/storage/serve-uploads.js:13:29)
    at bootstrap (/app/dist/main.js:37:42)
```

Job `Typecheck, build and test` (unit 60/60 + e2e 11 suites 92/92) **xanh** — nghĩa là toàn bộ cổng tôi
chạy lokal đều đúng, và vẫn bỏ sót lỗi này.

**Nguyên nhân**: §46 thêm `serveUploadsFrom()` gọi `mkdirSync(UPLOAD_DIR, { recursive: true })` lúc boot,
mặc định `uploads`resolve theo cwd. Dockerfile runtime `FROM node:22-bookworm-slim` có `WORKDIR /app` do
**root** sở hữu, rồi `USER node`; `COPY --chown=node:node` chỉ đổi chủ ba thư mục được copy
(`package.json`, `node_modules`, `dist`), không đổi chủ `/app`. User `node` vì thế không được phép tạo
thư mục con mới trong `/app`. `VOLUME` cũng không cứu: khai báo volume không làm thư mục bên trong thuộc
về `node`.

Máy này không có Docker (`docker: command not found`), nên `compose up`/`docker run` chưa từng được chạy
ở lokal — đúng cái đã ghi trong §Docker của README. Bài học thật: **mọi thay đổi đụng đường đi của file
trên đĩa phải có ít nhất một cổng chạy trong môi trường nó deploy**, không phải môi trường dev ghi log
`Connected using pglite` là đủ.

**Sửa**:
- `Dockerfile`: `RUN mkdir -p /app/.data /app/uploads && chown -R node:node /app/.data /app/uploads` đặt
  **trước** `USER node`, thêm `UPLOAD_DIR=/app/uploads` vào `ENV` và `/app/uploads` vào `VOLUME`.
- `play-together-be/docker-compose.yml` và `docker-compose.yml` (repo gốc): mount volume named `uploads`
  vào `/app/uploads` cho service `api` + khai báo trong `volumes:`. Không mount thì ảnh avatar nằm ở lớp
  container / volume ẩn, `down -v` là mất hết.
- README BE + README gốc ghi lại đường nào cần volume.

**Kiểm chứng**: YAML hai file compose parse bằng `node_modules/yaml` → `api.volumes=["uploads:/app/uploads"]`,
`topVolumes=["pgdata","uploads"]` cả hai file. Máy này không có Docker (`docker: command not found`) nên
bằng chứng quyết định là CI chạy lại, và nó đã chạy: run `42491626825` trên commit `ecfe1a5` cho
**`conclusion=success`** ở cả hai job, log step smoke test in ra

```
API healthy after 2 attempt(s)
{"status":"ok","database":"up","uptimeSeconds":0,"version":"2.0.0"}
```

(cùng step đó ở run hỏng in ra 30 dòng `curl: (7) Could not connect` rồi `API did not become healthy in
time`). Hai run FE `42490842981` và admin `42490846641` vẫn `success`, nên phần code app của đợt push này
không liên quan tới hỏng hóc.

## 64. Một cache key, một shape: ô "đang chờ về ví" mở ra màn "Không có khoản nào chờ" (R19)

**Triệu chứng** (người dùng báo, tái hiện bằng probe trên Expo web, tài khoản `demo.player2` có đúng một
khoản chờ): ô ở dashboard in `108.000 đ đang chờ về ví · 1 phiên`, bấm vào là màn `/unactive-balance`
nội dung

```
Tiền đang chờ về ví | Không có khoản nào chờ | Tiền của các đơn đã hoàn thành sẽ xuất hiện ở đây trước khi về số dư khả dụng.
```

Không có lỗi nào trong console (`=== errors (0)`), vì app không crash — nó chỉ tin vào dữ liệu rỗng.

**Nguyên nhân**: hai nơi dùng chung `queryKeys.unActiveBalances = ['users','un-active-balances']` nhưng
cache hai shape khác nhau. `PlayerDashboard.tsx:51-54` lưu mảng đã bóc (`(await ...).content`), còn
`unactive-balance.tsx:29-34` lưu nguyên envelope rồi đọc `data?.content`. React Query không phân biệt
được: key giống nhau, value khác nhau. Màn nào mount trước thì ghi kết quả của nó, màn sau đọc lại
`.content` trên một mảng → `undefined` → `balances = []` → EmptyState phủ định đúng con số người dùng
vừa bấm. Chiều ngược còn nặng hơn: dashboard làm `pending.reduce(...)` trên envelope → `TypeError`.
Envelope đó do `legacyArray()` (mappers.ts:83-95) sinh ra khi service adapter bọc lại `/wallet`.

Quét cả app bằng chính bộ quét đó tìm thêm hai điểm cùng loại:
- `queryKeys.allGames`: 4 màn cache mảng, riêng `onboarding/take-categories.tsx:44-49` cache envelope
  và đọc `data?.content`. Đi onboarding xong mở "Game yêu thích" trong cùng phiên là nhận phải giá trị
  sai shape (`games` là object, `.map`/`.filter` gãy ngay).
- `queryKeys.ratings(userId, vote, pageSize)`: màn `ratings/[userId].tsx` giữ envelope, dashboard bóc
  mảng. Chưa va chạm vì dashboard dùng `pageSize=3` còn màn kia bước 10, nhưng đây là mìn: đổi số 3
  thành 10 là hai bên đọc value của nhau.

**Sửa** — một key một shape, không đổi API:
- `unactive-balance.tsx`: `queryFn` bóc `.content`, màn đọc `data ?? []` (giống dashboard và giống
  convention của `statistic`, `playerWork.requests`).
- `onboarding/take-categories.tsx`: cùng chiều bóc `.content`.
- `PlayerDashboard.tsx:76-80, 87`: phần đánh giá dùng envelope đúng như màn `ratings` vẫn cache.
- Cổng mới `src/lib/__tests__/queryKeyShapes.spec.ts`: đọc toàn bộ `src`, với mỗi key có ≥2 nơi dùng thì
  các `queryFn` phải cùng shape. Nó tự kiểm mình: `scannedBlocks === declaredCalls` và không block nào
  bị bỏ qua, để cảm biến không âm thầm thành vô dụng khi cú pháp parse kém đi.

**Kiểm chứng**:
- Render thật (probe `probe-r19-shape.cjs`, 390×844 @2x, đăng nhập bằng form thật): 5/5 — tile mở đúng
  `/unactive-balance`, màn **không** còn là EmptyState, có dòng `Tổng đang chờ: 108.000 đ`, và con số
  của ô tile khớp từng ký tự với tổng của màn (`tile="108.000đ" screen="108.000đ"`), 0 lỗi runtime.
  Ảnh: `shots/r19-pass1-unactive.png` (card `+ 108.000 đ`, pill "Chưa xử lý", "Về ví từ 03:51 11/11/2026").
- Control âm: revert tạm hai file về shape cũ → cổng jest FAIL, liệt kê đúng hai conflict
  `queryKeys.unActiveBalances` và `queryKeys.allGames`; restore → PASS. Lần chạy đầu của chính cổng này
  từng FAIL-KHÔNG-PHÁT-HIỆN (parse cắt nhầm ở `() =>`), và control là thứ bắt nó quay lại.
- Gate: `npx tsc --noEmit` sạch, `npx expo lint` sạch, `npx jest` 13 suites / 97 tests xanh.
