# Đăng nhập bằng Google — hướng dẫn cấu hình

Tài liệu này viết theo code đang có trong repo (không phải hướng dẫn chung chung):

| Chỗ | File | Biến |
| --- | --- | --- |
| API xác minh token | `play-together-be/src/modules/auth/google-auth.service.ts` | `GOOGLE_CLIENT_ID` (+ `GOOGLE_ANDROID_CLIENT_ID`, `GOOGLE_IOS_CLIENT_ID` — tuỳ chọn) |
| API đổi token Google lấy JWT | `play-together-be/src/modules/auth/auth.service.ts` → `loginWithGoogle()` | — |
| App gọi cửa hàng OAuth | `play-together-fe/src/components/auth/GoogleLoginButton.tsx` | `EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID` |
| App đổi code/id_token phía Google | `expo-auth-session` 58.0.14 + plugin `expo-web-browser` trong `app.json` | — |

## 1. Luồng chạy như thế nào

```
Người dùng bấm "ĐĂNG NHẬP BẰNG GOOGLE"
  → app mở trình duyệt (WebBrowser) tới trang consent của Google
  → Google quay lại app kèm id_token
  → app POST /api/v2/auth/google { idToken }
  → API verifyIdToken(): kiểm chữ ký + issuer + aud (client ID của mình) + exp
  → API tra user theo google_id; không có thì tra theo email (gắn liên kết);
    vẫn không có thì tạo tài khoản mới với password_hash = NULL
  → API trả accessToken/refreshToken/user như đăng nhập thường
```

Hai chỗ dễ sai nhất, và là lý do code đang viết như hiện tại:

- **`aud` phải khớp.** id_token mang client ID đã yêu cầu nó. App vì vậy luôn xin token
  bằng client **Web application** — xem chú thích `WEB_CLIENT_ID` trong `GoogleLoginButton.tsx`.
  Nếu app xin bằng client Android/iOS, API sẽ trả `401 invalid_google_token` dù người dùng
  đã đăng nhập Google thành công. (API vẫn chấp nhận client Android/iOS nếu bạn khai báo
  `GOOGLE_ANDROID_CLIENT_ID` / `GOOGLE_IOS_CLIENT_ID`, dành cho trường hợp dùng thư viện native.)
- **Không có client secret.** App là installed app nên không được chứa secret; secret cũng
  không dùng được với `response_type=id_token`.

## 2. Tạo project và OAuth consent screen

1. Mở <https://console.cloud.google.com/> → tạo project, ví dụ `play-together`.
2. **Google Auth Platform → Branding** (menu bên trái; project của bạn đã có sẵn mục này):
   - App name: `Play Together`; User support email + Developer contact: email của bạn.
   - Logo: tuỳ chọn, upload `play-together-fe/assets/icon_v2.png` (243 KB, ảnh vuông) là được.
     Đang ở chế độ Testing nên có logo cũng không bị đòi xác minh.
   - Application home page / privacy policy / terms: được phép để trống khi Testing; chỉ bắt buộc
     khi bạn xin xác minh.
   - **Audience** (cùng menu): giữ **External** — đây là lý do mục 3 bên dưới bắt buộc phải làm.
3. **Google Auth Platform → Audience → Test users**: thêm địa chỉ Gmail bạn dùng để test.
   Khi consent screen còn ở chế độ *Testing*, chỉ tài khoản trong danh sách này đăng nhập được;
   người khác sẽ gặp màn hình "app chưa được xác minh". Về sau bấm **Publish app** để bỏ giới hạn.
4. **Không phải vào Google Auth Platform → Data Access để thêm scope.** Trang đó (*Add or
   Remove Scopes*) chỉ cần khi app xin scope nhạy cảm; ba scope app này xin — `openid`, `email`,
   `profile` (`settings.minimumScopes` trong `node_modules/expo-auth-session/build/providers/Google.js`)
   — là scope không nhạy cảm, Google tự hiện trên màn consent theo chính request của app và không
   đòi xác minh. Để trống Data Access. Nếu khi test thật Google báo lỗi scope thì mới thêm đúng ba
   dòng đó vào.

## 3. Tạo OAuth client ID (Web application)

**Google Auth Platform → Clients → Create Client** (đường cũ: *Credentials → Create credentials →
OAuth client ID*) → **Application type: Web application**, đặt tên `play-together-web`.

- **Authorized JavaScript origins**: `http://localhost:8081`
- **Authorized redirect URIs**: `http://localhost:8081`

Sao **Client ID** (dạng `xxxxxxxx.apps.googleusercontent.com`). Đây là một giá trị duy nhất,
dùng cho cả app và API.

Vì sao là `localhost` mà không phải IP mạng LAN: `expo-auth-session` trên web lấy
`window.location.origin` làm redirect URI, và Google chỉ cho đăng ký `http://localhost`
(kèm cổng) cho client Web. Khi test Google trên web, hãy mở `http://localhost:8081`,
**không** mở `http://192.168.1.20:8081` — nếu mở bằng IP bạn sẽ nhận `redirect_uri_mismatch`.

## 4. Điền biến môi trường

`play-together-be/.env`:

```
GOOGLE_CLIENT_ID=xxxxxxxx.apps.googleusercontent.com
```

`play-together-fe/.env`:

```
EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID=xxxxxxxx.apps.googleusercontent.com
```

Rồi khởi động lại **cả hai**:

```bash
# API
npm run build && node dist/main.js

# Metro — biến EXPO_PUBLIC_* được nhúng thẳng vào bundle, phải build lại bundle
npx expo start --lan --clear
```

Metro đọc `.env` **một lần lúc khởi động**, không theo dõi file. Nếu bạn thêm
`EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID` trong lúc Metro đang chạy thì bundle vẫn thiếu biến đó và
nút Google vẫn toast "Chưa cấu hình Google…" dù `.env` đã đúng. Kiểm chứng nhanh: tải
`http://localhost:8081/node_modules/expo-router/entry.bundle?platform=web&dev=true&hot=false`
rồi tìm `HMR env vars from Expo CLI` — biến nào không nằm trong đoạn đó là biến Metro chưa thấy.

Không cấu hình thì nút Google vẫn hiện, chỉ toast nhắc "Chưa cấu hình Google…"; API trả
`503 google_not_configured`.

## 5. Test được gì ở đâu

| Nền tảng | Chạy được chưa | Ghi chú |
| --- | --- | --- |
| Web (`http://localhost:8081`) | ✅ ngay sau khi làm mục 2–4 | đường test nhanh nhất |
| **Expo Go** trên điện thoại | ❌ | redirect URI trong Expo Go là `exp://192.168.1.20:8081/--/`; Google không cho đăng ký scheme lạ ở client Web, nên bấm nút sẽ rơi vào trang lỗi của Google. Đây là giới hạn của Expo Go, không phải bug của app. |
| **Development build** trên điện thoại | ✅ sau khi build bản dev | `expo-auth-session` lúc này dùng `com.playtogether.app:/oauthredirect`; thêm chuỗi đó vào *Authorized redirect URIs* của client Web. Máy này chưa có JDK/Android SDK, nên build bằng EAS: `npx eas-cli build --profile development -p android` (hoặc cài Android Studio + JDK 17 rồi `npx expo run:android`). |

Nếu Google không cho lưu redirect URI dạng custom scheme, hoặc bạn muốn trải nghiệm
"nút Google" đúng bản native (đệm đăng nhập của hệ điều hành, không mở trình duyệt), thì
dùng thư viện native thay cho `expo-auth-session` — xem mục 6.

## 6. Client ID Android / iOS (chỉ khi dùng thư viện native)

Làm thêm **một client ID cho mỗi nền tảng** trong cùng project:

- **Android**: Application type *Android*, Package name `com.playtogether.app`
  (theo `app.json` → `android.package`), SHA-1 của keystore bạn build bằng.
  SHA-1 debug: `keytool -list -v -keystore "$HOME/.android/debug.keystore" -alias androiddebugkey -storepass android`
  hoặc `npx eas-cli credentials` nếu build trên EAS.
- **iOS**: Application type *iOS*, Bundle ID `com.playtogether.app`
  (theo `app.json` → `ios.bundleIdentifier`).

Điền client ID của hai bản này vào API (`GOOGLE_ANDROID_CLIENT_ID`, `GOOGLE_IOS_CLIENT_ID`)
để API còn chấp nhận token do chúng xin. App **vẫn** xin token bằng client Web.

Thư viện native tham khảo: `@react-native-google-signin/google-signin` (Android Credential
Manager + GoogleSignIn cho iOS). Lưu ý nó có native code nên **Expo Go sẽ không chạy được app
nữa** — chỉ thêm khi bạn đã làm dev build.

## 7. Tài khoản Google rồi mới đặt mật khẩu

Đây chính là kịch bản "đăng nhập Google → Quên mật khẩu → đặt mật khẩu → login bằng cả hai",
và app đã xử lý được vì:

- Tài khoản tạo từ Google có `password_hash = NULL` (bảng `users` để trống được) nên không
  đăng nhập bằng mật khẩu được — API trả `invalid_credentials`.
- Màn **Quên mật khẩu** chỉ cần email đã tồn tại, không hỏi mật khẩu cũ → tài khoản Google
  vẫn nhận được OTP.
- `POST /auth/password/reset` ghi `password_hash` mới, **không đụng tới `google_id`** →
  lần sau đăng nhập Google, API tìm thấy đúng user cũ theo `google_id`, không tạo tài khoản thứ hai.
- Email đó không đăng ký trùng được: `POST /auth/register` trả `409 email_taken`.
- Đặt lại mật khẩu thì mọi phiên đang đăng nhập bị thu hồi (refresh token bị revoke),
  người dùng phải đăng nhập lại — đây là hành vi chủ đích.
- API từ chối tạo hoặc liên kết tài khoản khi Google báo email chưa xác minh
  (`email_verified != true`), vì luồng quên mật khẩu lấy email làm danh tính.

Kiểm chứng nhanh bằng curl (dev đang bật `EXPOSE_OTP=true` và chưa có SMTP nên mã OTP trả
ngay trong response, và app cũng toast mã này ra màn hình):

```bash
EMAIL=me@gmail.com
BASE=http://localhost:3000/api/v2/auth

curl -s -X POST $BASE/otp/request -H 'content-type: application/json' \
  -d "{\"email\":\"$EMAIL\",\"purpose\":\"PASSWORD_RESET\"}"   # → devCode
curl -s -X POST $BASE/otp/verify -H 'content-type: application/json' \
  -d "{\"email\":\"$EMAIL\",\"purpose\":\"PASSWORD_RESET\",\"code\":\"<devCode>\"}"   # → resetToken
curl -s -o /dev/null -w '%{http_code}\n' -X POST $BASE/password/reset -H 'content-type: application/json' \
  -d '{"token":"<resetToken>","newPassword":"NewPass123"}'      # → 204
curl -s -X POST $BASE/login -H 'content-type: application/json' \
  -d "{\"email\":\"$EMAIL\",\"password\":\"NewPass123\"}"       # → 200 + JWT
```
