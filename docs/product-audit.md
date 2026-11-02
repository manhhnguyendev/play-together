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


















