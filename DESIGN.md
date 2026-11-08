# Play Together — Design Contract

Nguồn sự thật duy nhất cho giao diện của **app di động** (`play-together-fe`) và **console quản trị**
(`play-together-admin`). Mọi màn hình, component và token mới phải dẫn chiếu về file này; khi đổi một
quyết định ở đây, đổi token trước rồi để component/màn hình hưởng theo — không sửa lẻ từng màn.

Phương pháp lấy từ [OpenDesign](https://github.com/nexu-io/open-design): mỗi design system là một
package gồm `DESIGN.md` (brand contract) + tokens + components, và agent refactor **component thật**
theo contract chứ không dán template đè lên sản phẩm.

---

## 1. Sản phẩm và người dùng

Play Together là chợ thuê người chơi game cùng: **Hirer** tìm và thuê **Player** theo giờ, trả tiền
qua ví ký quỹ (escrow), có khiếu nại/đánh giá; kèm khối từ thiện. Hai vai trò dùng chung một app,
chuyển chế độ bằng công tắc "Nhận yêu cầu thuê". Console quản trị là công cụ vận hành (duyệt chi trả,
khiếu nại, cấu hình) cho staff — mật độ thông tin cao, thao tác lặp lại nhiều.

Hệ quả thiết kế:

- App: ưu tiên một tay, vùng chạm ≥ 44 pt, chữ đọc được ngoài nắng, trạng thái tiền/đơn phải rõ ràng
  tuyệt đối (không giấu sau icon).
- Admin: ưu tiên quét nhanh — bảng đặc, nhãn tiếng Việt đời thường, một nút Lưu duy nhất cho mỗi
  biểu mẫu (không lưu ngầm khi bấm từng control), không hiện UUID/mã enum ra màn hình.

## 2. Nguyên tắc

1. **Một nguồn sự thật.** Màu, chữ, khoảng cách, bo góc, đổ bóng, chuyển động đều lấy từ token.
   Không hex cứng, không `fontSize` số trần trong màn hình/component.
2. **Bề mặt xếp lớp bằng độ sáng** (học Linear): nền màn hình → thẻ nổi → thẻ trong thẻ. Mỗi lớp
   chênh nhau một bậc sáng và một đường viền mảnh; đổ bóng chỉ phẩy nhẹ, không dùng bóng đậm để phân
   cấp.
3. **Phân cấp bằng chữ và khoảng cách, không bằng màu mè.** Một màn chỉ có một điểm nhấn màu thương
   hiệu; trạng thái dùng màu ngữ nghĩa riêng.
4. **Mọi danh sách có ba trạng thái thật**: đang tải (skeleton đúng hình dáng nội dung), rỗng (giải
   thích + một hành động), lỗi (thông điệp từ API + nút thử lại). Không spinner trần giữa màn hình.
5. **Chuyển động có trục.** Vào từ trái = đi sâu vào nội dung; sheet từ đáy = tác vụ của hub hiện tại;
   đổi nhóm = fade. Một giá trị điều khiển cả hai chiều, không tự chế animation mỗi màn.
6. **Phản hồi tức thì.** Mọi thứ bấm được có trạng thái nhấn (scale 0.97 / opacity 0.82, 120 ms);
   mọi thao tác mạng có trạng thái chờ trên chính control đó; lỗi nói rõ lý do bằng tiếng Việt.
7. **Không control chết.** Không disabled vô cớ, không khoá kèm tooltip thay cho lựa chọn thật;
   lựa chọn rỗng là hợp lệ nếu nghiệp vụ cho phép.
8. **Responsive bằng breakpoint, không bằng cảm giác.** `useResponsive()` quyết định gutter, số cột,
   cỡ chữ scale; màn hình không tự đoán chiều rộng.

## 3. Token — app di động (`src/theme`)

### 3.1 Màu

Giữ ramp thương hiệu hiện có (`palette.ts`): violet 50–900, gold, neutral, status; dark dùng
`darkNeutral`/`darkStatus`/`darkBrand`. Ngữ nghĩa (`colors.ts`):

| Nhóm | Token | Ghi chú |
| --- | --- | --- |
| Thương hiệu | `primary`, `primaryAction`, `primaryFill`, `primarySoft/Faint/Border` | `primaryFill`/`primaryAction` là màu hành động (đủ tương phản cho chữ trắng); `primary`/ramp nhạt chỉ dùng làm tint, không làm chữ trên nền trắng |
| Bề mặt | `background` (canvas), `surface` (thẻ), `surfaceMuted` (lớp lõm) | Thẻ luôn kèm `border` mảnh; dark: canvas `#101219` → card `#181B24` |
| Chữ | `text`, `textStrong`, `textSecondary`, `textMuted`, `textFaint` | Không dùng `textFaint` cho nội dung bắt buộc đọc |
| Viền | `border`, `borderStrong` | `border` = hairline của thẻ |
| Ngữ nghĩa | `success*`, `danger*`, `warning*`, `gold*` | Tiền vào/ra dùng success/danger, không dùng màu thương hiệu |

### 3.2 Chữ (`typography.ts`)

Montserrat 400/500/600/700. Thang chữ **mới** (size/line), tracking âm ở cỡ lớn:

| Token | Size/Line | Dùng cho |
| --- | --- | --- |
| `display` | 30/36 bold −0.5 | Số dư lớn, tiêu điểm một màn |
| `title1` | 24/30 bold −0.3 | Tiêu đề màn ngang hàng lớn |
| `title2` | 20/26 semibold −0.2 | Tiêu đề khối |
| `title3` | 17/23 semibold | Tiêu đề thẻ, tên người |
| `title4` | 18/24 semibold | Tiêu đề phụ lớn hơn title3 một nhịp |
| `bodyLarge` | 16/24 regular | Đoạn văn chính, mô tả dài |
| `body` / `bodyMedium` | 15/22 | Chữ mặc định |
| `bodySmall` / `bodySmallMedium` | 14/20 | Chữ phụ trong thẻ, danh sách |
| `caption` | 12/16 medium | Nhãn phụ, metadata |
| `overline` | 11/14 semibold +0.7 | Nhãn khối IN HOA |

Quy tắc: màn hình **không** viết `fontSize:` số trần — spread `...type.X` (hoặc `type.X` của
`useResponsive()` trên màn cần scale). Weight lấy từ `fonts.*`, không gõ `'600'` trần.

### 3.3 Khoảng cách / bo góc / chạm

`spacing` 4/8/12/16/20/24/32/40/48/64. Nhịp dọc mặc định giữa các khối: `lg` (20); trong thẻ: `md`
(16); giữa nhãn và giá trị: `xs` (8). `radius`: xs 6 (chip nhỏ), sm 10 (control), md 14 (thẻ),
lg 18 (thẻ lớn/sheet), xl 24 (khối anh hùng), pill 999 (avatar, badge). `touchTarget` 44;
`control.inputHeight` 50, button 44/50/56.

### 3.4 Độ nổi

`shadow.card` = phẩy nhẹ (ios opacity 0.04 / radius 12 / offset 4; android elevation 1) **cộng** viền
`border` 1 px từ component `Surface`. `raised`/`floating`/`sheet` giữ cho phần tử nổi thật (FAB,
toast, sheet). Không tự đặt `elevation`/`shadow*` trong màn hình.

### 3.5 Chuyển động (`motion.ts`)

`duration` press 120 / state 180 / enter 240 / exit 200 / sheet 320; `easing.standard`
`bezier(0.2,0,0,1)`; `press.scale` 0.97. Stack: `push='ios_from_left'`, `sheet='slide_from_bottom'`,
`group='fade'` — **đừng đổi trục** (quyết định đã ghi ở product-audit: forward vào từ trái, back vào
từ phải là cố ý).

## 4. Component dùng chung — app (`src/components/ui`)

| Component | Hợp đồng |
| --- | --- |
| `Surface` | Thẻ chuẩn: nền `surface`, viền `border`, bo `radius.md`, padding `md`, `shadow.card`. Biến thể `muted`, `padless`, `radius`. Mọi thẻ trong màn hình dùng nó, không tự vẽ View trắng |
| `Button` | 5 biến thể × 3 cỡ; loading hiện spinner **trong** nút và giữ nguyên nhãn; fullWidth; icon trái |
| `TextField` / `SelectField` / `DatePickerField` | variant `floating` là mặc định cho biểu mẫu; lỗi hiện dưới ô bằng chữ `caption` màu danger; focus đổi viền sang `primaryBorder` |
| `AppBar` / `ScreenHeader` | Cao 56, nền trong suốt trên canvas, tiêu đề `title3`, nút back vùng chạm 44 |
| `TabBar` (trong `(tabs)/_layout`) | Thanh nổi bo pill, mục active có nền tint + icon đậm, badge số thật. Nhãn giữ bậc 11 px: năm tab ở 375 dp không đủ chỗ cho bậc 12 |
| `EmptyState` | Icon trong vòng tròn tint 56 pt + tiêu đề `title3` + mô tả `bodySmall` + một Button hành động |
| `ErrorView` | Thông điệp API (tiếng Việt) + nút "Thử lại"; không lộ stack trace |
| `LoadingView` / `Skeleton` | Skeleton shimmer đúng hình dáng khối sẽ hiện; chỉ dùng spinner cho tác vụ ngắn |
| `Avatar` | Bo tròn, viền tuỳ chọn, fallback chữ cái đầu trên nền tint |
| `Chip`, `Checkbox`, `RatingStars`, `SectionHeader`, `Toast` | Giữ nguyên API; màu/chữ lấy token |

## 5. Token — console quản trị (AntD v6)

Theme khai báo một lần trong `src/theme.ts` (hai bản `light` / `dark`), `src/app/Root.tsx` chọn
bản theo chế độ đang bật rồi đưa vào `ConfigProvider` duy nhất.

### 5.1 Bản sáng

- `colorPrimary: '#5B51D8'` (violet 700 — cùng màu hành động với app), `colorInfo` cùng họ;
  `colorSuccess '#188A54'`, `colorWarning '#B4740F'`, `colorError '#C0333A'` (khớp status app).
- `borderRadius: 10`, `controlHeight: 36`, `fontFamily` hệ thống chữ không chân hiện có;
  `colorBgLayout: '#F6F7FB'` (canvas app), `colorBorderSecondary: '#E4E6EF'`.
- Component tokens: `Menu` (nền sáng, item active nền tint violet + chữ violet đậm), `Card`
  (viền mảnh, không bóng đậm), `Table` (header nền `surfaceMuted`, row hover tint), `Button`
  (bo 10, bỏ bóng mặc định).
- Sider **sáng** cùng tông nội dung, brand mark violet; không dùng sider đen lệch tông.

### 5.2 Bản tối (`theme.darkAlgorithm`)

- Nền lấy đúng bảng dark của app: canvas `#101219`, container `#181B24`, bề mặt nổi
  (modal, dropdown, popover) `#20242F`, hairline `#2A2E3B`, viền ô nhập `#454C60`,
  `colorTextBase: '#F1F2F8'`.
- Hành động nâng lên violet 600 `#6E64F0` (thuật toán dark render ra xấp xỉ `#6158CF`) để chữ
  trắng trên nút đạt tương phản; menu active nền `#2C2E52` chữ `#BCB3FF`, bảng header `#20242F`.
- Không đổi họ màu status: thuật toán dark tự sinh bảng màu tối từ cùng một seed, nên
  `StatusTag` dùng chung nhãn cho cả hai chế độ.

### 5.3 Chế độ giao diện

- Ba lựa chọn `Sáng / Tối / Theo hệ thống` trong menu ở header; mặc định là `Theo hệ thống`
  (bám `prefers-color-scheme`, đổi OS thì giao diện đổi ngay không cần tải lại).
- Lưu key `pt_admin_theme` trong localStorage; `data-theme` + biến `--pt-canvas` được đặt ở
  module scope trước khi React mount nên tải lại ở chế độ tối không loá trắng; `color-scheme`
  đổi theo để scrollbar và input native đúng nền.

### 5.4 Khung trang

- Mọi bảng có `locale.emptyText` chung (icon + câu tiếng Việt + hành động nếu có); mọi màu tiền /
  trạng thái đi qua `StatusTag` / token, không hex ngoài `src/theme.ts`.
- Mỗi trang danh sách theo một khung: `PageHeader` → `FilterCard` → `TableCard`; không lặp khối Card trần.

### 5.5 Dashboard

- Mở đầu bằng hero gradient violet→indigo (glow gold/magenta + grain), chấm live và giờ cập nhật thật;
  bộ chọn kỳ nằm ở hero, mặc định **7 ngày** để lần mở đầu không rơi vào kỳ trống.
- KPI là surface riêng (hairline + bo 14): icon tint theo hue chỉ số, số đếm động, và một hình thật
  bên dưới (sparkline / thanh cơ cấu) khi hình đó nói đúng một tỷ lệ bộ-phận/toàn-thể. Card
  **"Dòng tiền"** lấy headline là tổng tiền đang nằm trong nền tảng (giữ hộ + chờ trả người chơi)
  và tách đúng hai pool đó bằng thanh hai dải có hover title; phí / quyên góp luỹ kế nằm dòng chữ riêng.
- Mỗi ô KPI là **link mở thẳng trang của phần đó** (`/users`, `/orders`, `/money`, và hàng kiểm duyệt
  đầu tiên tier được mở); hover nhấc nhẹ + viền tint, focus ring cho bàn phím. Ô mà tier không có
  trang là khối tĩnh, không cursor pointer, để không giả vờ bấm được.
- Chữ vận hành: trạng thái `CLOSED` của đơn đọc là **"Đã huỷ"**; câu báo hết việc dùng **"Hàng chờ
  trống"**, không dùng chữ "sạch".
- Chart vẽ bằng SVG tự dựng (`src/components/charts.tsx`), không thêm thư viện: hai series thật
  (tạo mới / hoàn thành), tooltip hover, trần lưới tròn số; donut dùng bảy hue tách luminance ở
  `src/lib/chartHues.ts`, legend là link lọc thật. Nhãn trục ngày đếm bước **ngược từ ngày cuối** để
  mọi nhãn cách đều nhau và ngày cuối vẫn có nhãn — thêm nhãn cuối theo kiểu "modulo + ép cột cuối"
  từng in hai nhãn chồng lên nhau.
- Bảng "Cơ cấu trạng thái đơn" liệt kê **đủ bảy trạng thái vòng đời** (`ORDER_STATUSES`, cùng nguồn với
  dropdown lọc ở Đơn thuê), kể cả trạng thái đang có 0 đơn: dòng 0 mờ đi nhưng vẫn bấm ra bộ lọc của
  nó. Lý do: "Chờ đối soát: 0" là thông tin vận hành, và một bảng nhảy hàng theo kỳ thì không ai đọc
  được; donut vẫn chỉ vẽ cung có đơn.
- Hàng chờ là hàng link: chấm màu → nhãn (tổng tiền nếu có nằm **dưới nhãn**, tiền tố "Tổng") →
  **pill số record** bên phải → chevron; tint vàng khi còn việc. Không nhét số tiền vào pill.
- Feed audit mỗi hàng ba cột: chấm màu, thân hai dòng (dòng 1 người thao tác + hành động gọi tên mục
  bị sửa, dòng 2 chi tiết thay đổi dựng bởi `src/lib/auditSummary.ts` — before→after, nhãn tiếng Việt,
  giấu id), và thời gian ở cột phải (hover ra giờ tuyệt đối); cả hàng link tới nhật ký lọc theo hành
  động. Feed đơn mới nhất nằm cột đối diện; khối nào cũng có trạng thái rỗng và skeleton riêng,
  tôn trọng `prefers-reduced-motion`.
- Scrollbar toàn console vẽ theo palette (track trong, thumb bo tròn, hover sáng hơn) cho cả sáng lẫn tối.
- **Kỳ trống thì biểu đồ nhường chỗ, không làm nền**: `TrendChart` ở kỳ không có đơn trả về trạng thái
  rỗng của nhà (`TableEmpty`) đúng chiều cao biểu đồ — **không** vẽ trục 0-4 rồi in chữ `position:absolute`
  đè lên (đã từng che 100% câu chữ bằng 5 đường lưới + 12 nhãn trục). Legend "Tạo mới / Hoàn thành" cũng
  ẩn theo vì hai đường nó mô tả không tồn tại.
- **Mỗi số phải nói đúng phạm vi của nó**: `GET /admin/stats` không nhận tham số kỳ, nên KPI + donut là
  *toàn bộ thời gian*; biểu đồ và thẻ "Trong kỳ" lấy từ `/admin/orders?from=` nên *theo kỳ*. Thẻ donut vì
  vậy mang pill **"Toàn bộ thời gian"** ngay cạnh pill kỳ của thẻ "Trong kỳ". Đừng để một thẻ vừa in số
  toàn bộ thời gian vừa in sparkline theo kỳ — đó là điểm còn nợ ở KPI "Đơn thuê" (§59).
- **Phần tử được đo phải tồn tại ở mọi nhánh render.** `TrendChart` đặt `viewBox` theo bề rộng đo được
  của container, nên **nhánh biểu đồ và nhánh kỳ trống phải gắn cùng một callback ref**, không dùng
  `useLayoutEffect(..., [])`: effect một lần lúc mount không thấy gì để đo nếu lần mount đầu rơi đúng
  vào kỳ trống, `measured` đóng băng ở placeholder 760 và biểu đồ sau đó vẽ `viewBox` 760 trong box
  1046px — nét to, chữ béo, lề trái rộng hơn (đã xảy ra thật, đối chứng ở §60).

### 5.6 Khung điều hành (chrome) — mọi màn ngoài Dashboard

- **Một nguồn vị trí**: `src/lib/sections.ts` (`SECTION_META` + `NAV_GROUPS`) là chỗ duy nhất định nghĩa
  màn nào thuộc nhóm nào, tên ra sao. Sider và breadcrumb header cùng đọc từ đó — thêm trang mới là thêm
  một dòng. Màn chi tiết có route riêng (`/users/:id`) cũng phải có một dòng riêng, nếu không breadcrumb
  rơi về nhãn dự phòng và tự giới thiệu sai chỗ.
- **`PageHeader` không có kicker**: chỉ tiêu đề + một dòng mô tả + hành động bên phải. Chữ "màn này thuộc
  nhóm nào" đã có ở breadcrumb ("Vận hành › Đơn thuê") và ở nhãn nhóm trong sider; in thêm một overline
  IN HOA màu thương hiệu ngay trên h1 là dấu hiệu nhận ra ngay của giao diện sinh hàng loạt (và là mục
  extension báo ở §61).
- **Sider** 232px (thu 68px): brand mark + tên, nhóm điều hướng chữ overline nhỏ, mỗi mục có ô icon
  tint; mục đang chọn = nền tint + **thanh violet 3px sát mép trái**. Màn chi tiết khớp về mục danh
  sách của nó; tier đọc được màn đó mà không có mục nào trong menu thì **không sáng gì cả** — thà không
  sáng còn hơn sáng nhầm. Sider **không có chân**: một chip trạng thái + một nút lọt thỏm dưới đáy cột
  điều hướng nhìn như khối chưa làm xong, nên thanh này chỉ còn mỗi việc điều hướng. Ở 68px, tên nhóm
  **không cắt thành "V…"** mà chuyển thành hairline ngang.
- **Nút thu/giãn nằm ở mép trái header**, cạnh breadcrumb — luôn ở trên cùng và nhìn thấy được ở cả hai
  trạng thái; `aria-label` đổi theo ("Thu gọn…" / "Mở rộng…").
- **Ô icon trong sider** 26×26: glyph 14px phải **cách đều bốn cạnh (6px mỗi bên)**, đo được chứ không
  phải nhìn. Hai bẫy: AntD giữ `display: flex` trên slot icon nên `place-items: center` của ô không áp
  dụng (phải khai thêm `align/justify-content`), và ở rail thu 68px thì item co theo khổ glyph 16px nên
  cả ô bị lệch phải — rail thu bỏ inset của menu và cho item `margin-inline: 8px` đối xứng.
- **Header** dính (sticky) 60px, blur nền: breadcrumb "Nhóm › Tên màn" bên trái, bên phải là bộ chọn
  giao diện (Segmented ba icon kèm tooltip, không phải nút mặt trời/lửa rời rạc), vạch chia và khối
  profile. Bấm profile mở thẻ: avatar tint + tên + email + **vai trò bằng tiếng Việt**, rồi mục
  "Nhật ký thao tác" (chỉ hiện khi tier được mở) và đăng xuất.
- **Footer** nêu brand, chip trạng thái API ("API đang hoạt động" / "Đang nối API…" / "Mất kết nối API",
  polling `/api/v2/health` 60s — footer là chỗ duy nhất nói trạng thái này) và bản quyền.
- **Trạng thái** là pill chấm màu (`.pt-status[data-tone]`), không còn `Tag` mặc định của AntD; chín tone
  khai báo hai lần (sáng/tối) trong CSS, component chỉ việc truyền tone.
- **Bảng**: một ngôn ngữ cho **mọi** bảng trong `.pt-content` (kể cả bảng phụ trong màn chi tiết) —
  header in hoa 11px letter-spaced, ô 12px, phân trang cách mép; mọi ô đa giá trị (game tags)
  dùng `.pt-tag-row`/`.pt-tag-fit` — **cuộn xuống dòng trước, chỉ ellipsis khi một giá trị đã rộng hơn
  cột**, để không chữ nào vẽ đè cột kế tiếp.
- **Lưới không dính viền card**: thân `TableCard` lót **12px 16px 16px** (≤575px còn 10px 12px 12px vì
  card điện thoại chỉ 275px). Bảng áp sát border từng là chủ ý "viền card là khung của bảng", nhưng phép
  đo cho thấy dải header chỉ cách viền **1px** và chữ cột đầu lệch hàng điều khiển phía trên 12px; inset
  đưa cả hai về đúng. Padding phải nằm trong CSS chứ **không phải inline style**, vì inline thắng media
  query và breakpoint sẽ không giảm được nó.
- **Màn chi tiết** mở đầu bằng dải nhận dạng (avatar, tên, hai pill trạng thái, ba số liệu tóm tắt) rồi
  mới tới thẻ thông tin; số liệu đã có ở dải không lặp lại trong `Descriptions`.
- **Đăng nhập** hai cột: trái là brand panel (gradient + grain + ba điểm bán bằng chữ thật), phải là
  thẻ đăng nhập; bản hẹp hơn 991px bỏ brand panel. Dòng chân trang chỉ nói điều hệ thống thật làm được.

### 5.7 Sàn a11y bắt buộc (motion, chữ, tương phản)

Sàn này là **một phần của brand contract**: thêm màu/tone/animation mới là phải đo lại, không phải
trông ổn là đủ. Hai cổng tự động giữ nó (`docs/product-audit.md` §58).

- **Không easing nảy.** `motionEaseOutBack` / `motionEaseInBack` trong `theme.ts` là đường cong
  monotone (`0.22,1,0.36,1` và `0.64,0,0.78,0`), không có control point ngoài `[0,1]`; mặc định của
  AntD (1,46 / −0,46) làm popup và drawer nảy, dấu hiệu nhận ra ngay của giao diện sinh hàng loạt.
- **Không chuyển động trang trí vô hạn.** Dot trạng thái đứng yên; `@keyframes pt-pulse` đã bỏ.
- **Không animate thuộc tính layout — kể cả trên phần tử absolute.** `width/height/margin/padding`
  (và `left/right/top/bottom`) không được transition ở **bất kỳ** phần tử nào trong trang. Ngoại lệ cũ
  "ink bar của Tabs được, vì nó absolute" đã bỏ: đúng là nó không reflow hàng xóm, nhưng thuộc tính nó
  animate vẫn là thuộc tính layout, và đó chính là thứ máy quét đọc. Indicator cần trượt thì dùng
  `transform`; AntD ghi `width`/`left` inline nên không đổi sang transform được — thà để nó đứng yên
  (dấu hiệu thật của tab đang chọn là **nhãn đổi màu violet**, không phải gạch).
  Thanh rail thu/giãn dùng `transition: none`; `Meter` vẽ bằng `transform: scaleX()`, không bằng `width`.
  **`transition: all` cũng là một vi phạm**, vì nó ôm cả height/padding: hai chỗ đã phải ghi đè — header
  bảng có sort (`all` → `background-color, color`) và tag nhiều chọn của Select (`height, line-height,
  padding` → `opacity`). Override phải **tăng specificity** (thêm `.pt-content`, hoặc lặp lại một class)
  vì AntD inject `<style>` lúc chạy, thứ tự nguồn không bảo đảm.
- **Chữ chức năng ≥ 11px.** Nhãn nhóm ở sider, breadcrumb và overline trang đăng nhập đều 11px in hoa;
  nhỏ hơn là không đọc được trên màn 1080p sát mép.
- **Tương phản ≥ 4,5:1 cho chữ < 24px**, tính trên nền **đã cộng alpha thật** (nền pill tint, card,
  canvas), 3:1 cho đồ hoạ mang nghĩa (sao đánh giá, vạch Meter, dot legend). Miễn trừ đúng một trường
  hợp: control đang bị disable — WCAG 1.4.3 cho phép, và tô sẫm nó lên sẽ làm nó trông như bấm được.
- **Bẫy AntD bản tối**: thuật toán `darkAlgorithm` kéo mọi seed màu xuống ~0,87 lần trước khi ra pixel.
  Seed phải đặt bằng giá trị mục tiêu chia ngược (4,6:1), và kiểm chứng phải đọc **màu đã render** từ
  browser, không đọc hex trong `theme.ts`.
- **Cổng**: `src/__tests__/designContract.spec.ts` (18 tone pill đọc từ `index.css`, seed tối sau
  re-shade, mực chữ sáng, đường cong motion) chạy trong `npm test`; quét toàn màn bằng
  `scan-a11y.cjs` (mọi node chữ trên 16 route × 2 giao diện + login).

## 6. Kiểm chứng

- Đổi token/component xong phải **render thật**: app trên Expo web (đủ khổ 375/414/768) và admin trên
  trình duyệt; chụp lại màn chính trước khi báo xong.
- Cổng: `tsc --noEmit`, `expo lint` / `oxlint`, jest / vitest xanh; grep xác nhận không còn
  `fontSize:` số trần và không hex ngoài theme.
- Ghi quyết định và bằng chứng vào `docs/product-audit.md` theo mục mới, không sửa mục cũ.
