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

Hoặc chạy backend bằng Docker (API + PostgreSQL 17, không cần Node trên máy):

```bash
docker compose -f play-together-be/docker-compose.yml up --build
```

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
- API đã có Dockerfile + GitHub Actions CI (`play-together-be/.github/workflows/ci.yml`) và ESLint
  type-aware.

## Đẩy lên GitHub

Bốn repo độc lập nên mỗi repo có một remote riêng:

```bash
# FE đang ở branch feat/api-v2; đổi tên trước khi push
git -C play-together-fe branch -m feat/api-v2 main

for repo in . play-together-be play-together-fe play-together-admin; do
  git -C "$repo" remote add origin <url-của-repo-đó>
  git -C "$repo" push -u origin main
done
```

Không push file `.env` (đã nằm trong `.gitignore` của từng repo); khi clone về máy mới thì copy
`.env.example` như bước chạy nhanh ở trên.

## Giấy phép

MIT — xem [`LICENSE`](./LICENSE) ở mỗi repo. Đây là đồ án sinh viên; game name, ảnh và nhãn hiệu
trong dữ liệu seed thuộc về chủ sở hữu tương ứng, cần kiểm tra lại trước khi dùng thương mại.
