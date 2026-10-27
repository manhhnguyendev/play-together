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
npm run dev        # API (3000) + admin (5174)
npm run dev:app    # Expo dev server, chạy ở terminal riêng
```

Lần đầu chạy, API tự migrate và seed dữ liệu demo (`AUTO_MIGRATE=true`, `SEED_DEMO_DATA=true`).

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

## Script gốc

| Lệnh | Tác dụng |
| --- | --- |
| `npm run dev` | API watch mode + admin, tắt cùng nhau nếu một bên lỗi |
| `npm run dev:api` / `dev:admin` / `dev:app` | Chạy riêng từng app |
| `npm run build` | Build API và admin |
| `npm run typecheck` | `tsc --noEmit` / `tsc -b` cho cả ba app |
| `npm run lint` | ESLint của FE + oxlint của admin (**BE chưa có ESLint**) |
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
- **Không commit** file `.env`; mỗi app có `.env.example` tương ứng.
- **Hợp đồng API**: mọi thay đổi wire format phải sửa DTO ở `play-together-be` trước, rồi cập nhật
  adapter (`play-together-fe/src/api/v2`) và types (`play-together-admin/src/api/types.ts`).
- **Không sửa** `ios/` và `android/` bằng tay — chúng được sinh bởi Expo CNG, cấu hình nằm trong `app.json`.

## Giới hạn đã biết

- OpenAPI đã bật Swagger CLI plugin và có response schema cho admin/catalog/auth; còn ~17 endpoint
  (users, wallet, notifications, order history) chưa khai báo response, nên chưa sinh client tự động được.
  Types của admin vẫn viết tay.
- Một số tính năng FE phải giả lập vì v2 chưa có endpoint: hot search, filter giới tính/chữ cái/sở thích,
  release pending credit, favorite games, feedback detail.
- API đã có Dockerfile + GitHub Actions CI (`play-together-be/.github/workflows/ci.yml`); BE vẫn chưa có
  ESLint, cả ba repo chưa cấu hình remote.
