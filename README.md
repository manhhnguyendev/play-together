# Play Together

Capstone project: nền tảng kết nối người thuê và người chơi game.

Workspace gồm **ba ứng dụng độc lập**, mỗi app là một git repo riêng, và **repo gốc này chỉ giữ tài liệu + script điều phối**.

```text
play-together/
├── play-together-be/      NestJS API v2  (port 3000)  · repo riêng
├── play-together-fe/      Expo / React Native app     · repo riêng
├── play-together-admin/   Vite + React admin console (port 5174) · repo riêng
└── package.json           script chạy chung (chỉ có concurrently)
```

## Kiến trúc

| App | Vai trò | Stack | Port |
| --- | --- | --- | --- |
| `play-together-be` | REST API `/api/v2`, JWT access + refresh xoay vòng, Drizzle ORM + PGlite | NestJS 11 · TypeScript | 3000 |
| `play-together-fe` | Ứng dụng di động cho người dùng cuối | Expo SDK 58 · Expo Router · TanStack Query · Zustand | 8081 |
| `play-together-admin` | Bảng điều khiển kiểm duyệt và vận hành | React 19 · Vite · Ant Design 6 | 5174 |

Hai client đều nói chuyện với **một** API v2. FE giữ nguyên các envelope cũ nhờ lớp adapter ở `play-together-fe/src/api/v2`, còn admin dùng thẳng kiểu dữ liệu của v2.

## Yêu cầu

- Node.js **>= 20** (khai báo trong `engines` của cả ba app)
- npm (cả ba app đều dùng `package-lock.json`)
- Không cần cài Postgres: mặc định API chạy PGlite với dữ liệu trong `play-together-be/.data/pgdata`.
  `docker-compose.yml` trong repo BE chỉ dùng khi muốn chuyển sang Postgres thật.
- Docker Desktop (tuỳ chọn) nếu muốn chạy stack bằng container — xem mục Docker bên dưới.

## Chạy nhanh

```bash
npm install        # cài concurrently cho repo gốc
npm run setup      # npm ci cho cả ba app
cp play-together-be/.env.example play-together-be/.env   # bật migrate + seed demo
npm run dev        # API (3000) + admin (5174)
npm run dev:app    # Expo dev server, chạy ở terminal riêng
```

`play-together-be/.env.example` bật sẵn `AUTO_MIGRATE=true` và `SEED_DEMO_DATA=true`, nên lần chạy
đầu API tự migrate và seed dữ liệu demo. Nếu bỏ qua bước copy `.env`, API vẫn khởi động với secret
dev mặc định nhưng **không** seed — khi đó chạy `npm run db:seed`.

Hai client đọc URL API từ `.env` riêng (FE: `EXPO_PUBLIC_API_URL`, admin: `VITE_API_URL`); giá trị
mặc định đã trỏ về `http://localhost:3000/api/v2` nên bước này là tuỳ chọn.

Hoặc dựng cả hệ thống bằng Docker (API + admin console + PostgreSQL 17, không cần Node trên máy):

```bash
cp .env.example .env      # PowerShell: Copy-Item .env.example .env
npm run docker:up         # = docker compose up --build
```

- Admin: <http://localhost:8080> · API: <http://localhost:3000/api/v2>
- **Điều kiện**: bốn repo phải nằm cạnh nhau (`play-together/`, `play-together-be/`, `play-together-fe/`,
  `play-together-admin/`). `docker-compose.yml` build trực tiếp từ thư mục của hai repo app, mà repo gốc
  chỉ chứa tài liệu — code của ba app bị `.gitignore` và nằm ở repo riêng, nên clone lẻ repo gốc sẽ
  không dựng được.
- Seed dữ liệu demo sau lần chạy đầu: `docker compose exec api node dist/db/seed-cli.js`
- App mobile **không** nằm trong compose (cần thiết bị/simulator hoặc Expo Go): chạy `npm run dev:app`
  và trỏ `EXPO_PUBLIC_API_URL` về IP LAN của máy, cổng 3000.
- `JWT_ACCESS_SECRET` / `JWT_REFRESH_SECRET` bắt buộc có trong `.env` (compose dừng nếu thiếu);
  `SMTP_*` để trống thì API ghi mã OTP vào log thay vì gửi mail.
- Chỉ cần API + Postgres: `docker compose -f play-together-be/docker-compose.yml up --build`.

Admin được build với `VITE_API_URL=/api/v2` và nginx trong container proxy `/api/` sang service `api`,
nên trình duyệt gọi cùng origin và không vướng CORS. Muốn trỏ sang API khác thì đổi `ADMIN_API_URL`
trong `.env` rồi build lại (`docker compose build admin`).

### Tài khoản seed

| Vai trò | Email | Mật khẩu |
| --- | --- | --- |
| Admin | `admin@playtogether.local` | `Admin@12345` |
| Tổ chức từ thiện | `charity@playtogether.local` | `Charity@12345` |
| Người chơi demo | `demo.player1..6@playtogether.local` | `Demo@12345` |
| Người thuê demo | `demo.hirer1..2@playtogether.local` | `Demo@12345` |

## Script gốc

| Lệnh | Tác dụng |
| --- | --- |
| `npm run dev` | API watch mode + admin, tắt cùng nhau nếu một bên lỗi |
| `npm run dev:api` / `dev:admin` / `dev:app` | Chạy riêng từng app |
| `npm run build` | Build API và admin |
| `npm run typecheck` | `tsc --noEmit` / `tsc -b` cho cả ba app |
| `npm run lint` | ESLint của BE + FE, oxlint của admin |
| `npm run check` | typecheck + lint |
| `npm run db:seed` / `db:reset` | Seed lại / reset dữ liệu PGlite |
| `npm run docker:up` / `docker:down` | Dựng / dừng stack Docker (API + admin + Postgres) |

Mỗi app vẫn có script riêng; xem README trong từng repo.

## Vì sao không dùng npm workspaces?

Ba app có bộ dependencies rất khác nhau (Metro/Expo, NestJS, Vite). Hoisting về `node_modules`
chung thường làm Metro resolve sai hoặc phải thêm cấu hình `watchFolders`/`nodeModulesPaths`,
và đường dẫn tương đối của PGlite (`PGLITE_DIR=.data/pgdata`) sẽ lệch theo thư mục chạy.
Với quy mô capstone, giữ ba `node_modules` riêng đổi lấy việc chạy `npm run dev` một lần là hợp lý.
Khi cần khoá phiên bản chéo app thì chuyển sang workspaces (hoặc submodule) sau.

## Quy ước

- **Commit message**: Conventional Commits (`feat:`, `fix:`, `refactor:`, `chore:`, `docs:`), mô tả *vì sao* thay đổi.
- **Không commit** file `.env`; cả ba app đều có `.env.example` (BE khai đủ biến, FE có thêm Client ID
  Google tuỳ chọn, admin chỉ có URL API).
- **Hợp đồng API**: mọi thay đổi wire format phải sửa DTO ở `play-together-be` trước, chạy
  `npm run openapi:export` khi API đang chạy, rồi chạy `npm run api:generate` ở FE và admin.
  Code sinh ra nằm trong `src/api/generated` (không sửa tay, được bỏ qua khi lint).
- **Không sửa** `ios/` và `android/` bằng tay — chúng được sinh bởi Expo CNG, cấu hình nằm trong `app.json`.

## Giới hạn đã biết

- OpenAPI đã bật Swagger CLI plugin: **136 operation**, không operation nào thiếu response thành
  công — 97 × `200`, 17 × `201`, 1 × `202` và 21 × `204` (`@ApiNoContentResponse`). `openapi.json`
  là đầu vào cho Orval (`npm run api:generate`) ở cả hai client.
- **Admin** gọi thẳng endpoint và model sinh ra trong `src/api/generated`. **FE** vẫn giữ lớp adapter
  v2 viết tay ở `src/api/v2` (dịch wire type về model cũ); bản sinh ra trong `src/api/generated` đã
  có sẵn nhưng chưa màn hình nào dùng, để dành cho bước chuyển đổi sau.
- Các tính năng FE từng phải giả lập đã có endpoint v2 và đã nối vào app: hot search, filter
  giới tính/chữ cái/sở thích, release pending credit, feedback detail. Game yêu thích có màn hình
  riêng (`/favorite-games`, vào từ tab Cá nhân) với nút bật/tắt và cập nhật lạc quan.
- OTP/email đã chạy thật: đăng ký và quên mật khẩu gọi `/auth/otp/request` + `/auth/otp/verify`,
  mã 6 số hạn 10 phút. Chưa cấu hình `SMTP_HOST` thì API ghi mã vào log và trả `devCode`
  (ngoài production) — cần SMTP thật khi triển khai.
- Docker: API image (2 stage, non-root, `HEALTHCHECK`) nằm ở `play-together-be/Dockerfile` và được CI
  build + smoke test; admin có `play-together-admin/Dockerfile` (Vite build → nginx, proxy `/api/`);
  `docker-compose.yml` ở repo gốc ghép ba service `db`/`api`/`admin`. FE không đóng gói container vì là
  app mobile.
- API có GitHub Actions CI (`.github/workflows/ci.yml` ở repo BE: lint, typecheck, unit + e2e, build
  image và smoke test `/health`) và ESLint type-aware.

## Mã nguồn trên GitHub

Bốn repo công khai, cùng nằm dưới `github.com/manhhnguyendev` (chú ý hai chữ `h`):

| Repo | URL |
| --- | --- |
| Gốc (tài liệu + compose) | <https://github.com/manhhnguyendev/play-together> |
| API | <https://github.com/manhhnguyendev/play-together-be> |
| App | <https://github.com/manhhnguyendev/play-together-fe> |
| Admin | <https://github.com/manhhnguyendev/play-together-admin> |

Vì code của ba app nằm ở repo riêng (repo gốc chỉ giữ tài liệu, `.gitignore` bỏ qua ba thư mục app),
muốn dựng đầy đủ — kể cả `docker compose` — thì clone cả bốn repo cạnh nhau:

```bash
mkdir play-together && cd play-together
git clone https://github.com/manhhnguyendev/play-together.git .
git clone https://github.com/manhhnguyendev/play-together-be.git
git clone https://github.com/manhhnguyendev/play-together-fe.git
git clone https://github.com/manhhnguyendev/play-together-admin.git
```

Không push file `.env` (đã nằm trong `.gitignore` của từng repo); khi clone về máy mới thì copy
`.env.example` (repo gốc và từng app) như bước chạy nhanh ở trên.

## Giấy phép

MIT — xem [`LICENSE`](./LICENSE) ở mỗi repo. Đây là đồ án sinh viên; game name, ảnh và nhãn hiệu
trong dữ liệu seed thuộc về chủ sở hữu tương ứng, cần kiểm tra lại trước khi dùng thương mại.
