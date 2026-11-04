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
| **Expo Go** trên điện thoại | ❌ không bao giờ | redirect URI trong Expo Go là `exp://192.168.1.20:8081/--/`, và thân phận gói là `host.exp.exponent` nên không khớp package name + SHA-1 mà client Android đòi. Từ bản này app **không mở luồng Google trong Expo Go nữa**: `GoogleLoginButton` kiểm `Constants.executionEnvironment === ExecutionEnvironment.StoreClient` và toast giải thích thay vì đưa người dùng vào trang lỗi của Google. Đây là giới hạn của Expo Go, không phải bug của app. |
| **Development build** trên điện thoại | ⏳ code app đã sẵn (mục 6.B), còn thiếu client Android + một lượt build | Trên máy, `useIdTokenAuthRequest` chuyển sang luồng authorization-code + PKCE và **tự đổi code lấy id_token mà không cần client secret** (`node_modules/expo-auth-session/build/providers/Google.js`, `useAuthRequest` + `AccessTokenRequest`). Client loại **Web application** thì chết ở bước đổi code đó, nên app phải xin thêm client loại Android/iOS — đây chính là 6.B đã cài. Redirect mặc định của thư viện là `com.playtogether.app:/oauthredirect`; app gửi redirect dạng AppAuth derive từ client ID (`com.googleusercontent.apps.<id>:/oauth2redirect`) và `app.config.ts` khai scheme đó. |

## 6. Chạy trên máy thật — đã chốt 6.B (04/11/2026), 6.A ghi lại để dành

Điểm chung cả hai: **Expo Go không bao giờ chạy được** (mục 5), và đều cần một *development
build* do chính tay bạn dựng, vì nó đòi tài khoản Expo và quyền sửa project Google Cloud.
`eas.json` đã nằm trong repo với bốn profile: `development` (dev client — APK Android / app simulator
iOS), `development-device` (dev client dựng cho máy thật iOS), `preview` (APK nội bộ chia tay ngắn
hạn), `production` (bản submit store).

```bash
cd play-together-fe
npx expo install expo-dev-client   # profile "development" cần nó (docs Expo, 03/11/2026)
npx eas-cli login                  # tài khoản expo.dev của bạn
npx eas-cli init                   # ghi extra.eas.projectId vào app.json
npx eas-cli build --profile development --platform android
npx expo start --dev-client        # hoặc --go nếu vẫn muốn chạy Expo Go
```

Chú ý cho đúng một hệ quả: docs Expo (sửa ngày 03/11/2026) ghi *"`npx expo start`, vì project đã có
`expo-dev-client` nên lệnh này nhắm development build thay vì Expo Go*. Muốn tiếp tục chạy Expo Go cho
những màn không liên quan tới Google thì thêm `-g` — `npx expo start --help` của bản này có đủ
`-d, --dev-client` và `-g, --go`.

### 6.A. Thêm thư viện Google native — cách Expo hướng dẫn (guide ngày 02/11/2026)

`docs.expo.dev/guides/google-authentication/` không còn hướng dẫn dùng `expo-auth-session` cho
Google nữa; trang đó (và chú thích `@deprecated` trên `GoogleAuthRequestConfig` của bản
`expo-auth-session@58.0.14` đang cài) trỏ sang một trong hai thư viện có native code:

| Thư viện | Bản mới nhất | ghi chú |
| --- | --- | --- |
| `@react-native-google-signin/google-signin` | 17.0.0 | phổ biến nhất; Android Credential Manager nằm trong gói trả phí |
| `react-native-nitro-google-signin` | 2.3.1 | dùng API Android Credential Manager miễn phí; dựa trên Nitro Modules |

Cả hai đều xin id_token bằng `serverClientId` = **client Web đang dùng hôm nay**, nên token vẫn
đúng `aud` và **API không phải đổi gì**. Web (`react-native-web`) vẫn đi `expo-auth-session` như cũ
vì hai thư viện kia không chạy trên trình duyệt — nghĩa là `GoogleLoginButton` sẽ tách thành hai
nhánh theo `Platform.OS`.

**Học phí phải trả**: native code mới đồng nghĩa Expo Go không nạp được app nữa (kể cả các màn không
liên quan tới Google — import thất bại là crash cả màn hình đăng nhập). Mọi lượt test máy từ nay phải
đi qua dev build, mà máy này không có JDK/Android Studio (`java` và `adb` đều không có trên PATH) nên
mỗi lượt là một lần EAS build. **Đây là lý do 6.A bị gác**: anh cần vòng test Expo Go cho mọi sửa đổi
khác, và cả hai đường đều phải tạo client Android + khai SHA-1 như nhau — 6.A không tiết kiệm việc phía
tài khoản, chỉ đổi lấy "đúng hướng Expo". Quay lại 6.A khi nào cần lên store hoặc cần iOS.

### 6.B. Giữ nguyên `expo-auth-session`, xin token bằng client Android (không thêm dependency)

`useAuthRequest` hỗ trợ sẵn từ lâu: trên Android nó dùng `androidClientId` (nếu có) và luồng code + PKCE,
rồi tự đổi code lấy `id_token` (`fullResult.params.id_token`). **Phía code app đã làm xong ngày 04/11/2026**:

- `src/components/auth/GoogleLoginButton.tsx` đọc thêm `EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID` /
  `EXPO_PUBLIC_GOOGLE_IOS_CLIENT_ID`, vẫn dùng đúng một hook `useIdTokenAuthRequest` (web xin
  `id_token` trực tiếp, máy chạy code + PKCE — hook tự chọn theo `Platform.OS`), và chặn luồng trong
  Expo Go bằng `Constants.executionEnvironment`.
- `src/features/auth/googleNativeRedirect.ts` dẫn xuất redirect AppAuth từ client ID
  (`com.googleusercontent.apps.<id-not-suffix>:/oauth2redirect`); `app.config.ts` khai chính scheme đó
  vào `scheme` lúc build, nên `app.json` vẫn giữ `playtogether` cho deep-link của router.
- Không có client Android/iOS thì nút tự về dạng hint nói thiếu biến nào — không mở một luồng
  không thể quay về.

**Việc còn lại là phía tài khoản của anh** (tôi không tự làm được):

1. Google Cloud → *Create Client* → **Application type: Android**, Package name
   `com.playtogether.app` (`app.json` → `android.package`) + **SHA-1 của keystore mà bản dựng dùng**.
   Build bằng EAS thì lấy SHA-1 ở `npx eas-cli credentials` (hoặc Play Console → Release → Setup →
   App Integrity). Profile `development` và `production` có thể dùng hai keystore khác nhau — một
   client Android cho phép khai nhiều SHA-1, cứ thêm cả hai.
2. `play-together-fe/.env`: `EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID=<client Android>`;
   `play-together-be/.env`: `GOOGLE_ANDROID_CLIENT_ID=<cùng một giá trị>` — `GoogleAuthService` đã nhận
   danh sách audience sẵn (`src/modules/auth/google-auth.service.ts:29-34`), **API không sửa code**.
3. `npx eas-cli build --profile development --platform android` rồi bấm nút trên bản dựng đó.

Điểm cần nhìn nếu build đầu tiên không quay về được: app gửi redirect dạng AppAuth vì tài liệu OAuth
cho installed app của Google buộc dạng đó, còn **mặc định của thư viện** lại là
`${Application.applicationId}:/oauthredirect` (`providers/Google.js:142-148`). Nếu Google trả
`redirect_uri_mismatch` (toast sẽ nói rõ, xem mục 8), chỉ cần bỏ dòng `redirectUri` trong
`GoogleLoginButton` và đổi `app.config.ts` thành khai `com.playtogether.app` — một chỗ duy nhất, có
test giữ cho hai bên khớp nhau.

Chi phí so với 6.A: không thêm dependency, Expo Go vẫn dùng được cho mọi thứ khác, chỉ riêng nút Google
là cần bản dựng. Về rủi ro API: `@deprecated` chỉ gắn trên **type** `GoogleAuthRequestConfig`
(`expo-auth-session@58.0.14`), kiểm trên bản đang cài (`build/providers/Google.d.ts`) thì
`useAuthRequest` (dòng 83) và `useIdTokenAuthRequest` (dòng 68) **không** bị đánh dấu — nên 6.B vẫn chạy
trên code đang được duy trì, chỉ kiểu cấu hình hai hook đó nhận là Expo đã ghi chú khuyên chuyển hướng.
iOS cần thêm một client *iOS* (Bundle ID `com.playtogether.app`) và tài khoản Apple Developer, nên
không bật `EXPO_PUBLIC_GOOGLE_IOS_CLIENT_ID` cho tới lúc đó.

### 6.C. Những việc chỉ chủ tài khoản làm được

| Việc | Ở đâu | Vì sao tôi không tự làm |
| --- | --- | --- |
| `eas login` + `eas init` | expo.dev | cần tài khoản và quyền thanh toán của bạn |
| Create OAuth client **Android** (+ **iOS** nếu cần) | Google Cloud Console | project Google của bạn, tôi không có quyền sửa |
| Lấy SHA-1 từ bản dựng và dán vào client Android | `npx eas-cli credentials` | chỉ tồn tại sau lần build đầu |
| Điền `EXPO_PUBLIC_GOOGLE_*` vào `play-together-fe/.env`, `GOOGLE_ANDROID_CLIENT_ID`/`_IOS_` vào `play-together-be/.env` | máy bạn | file env local, không commit |
| Bật lại Metro/API sau khi đổi env | máy bạn | biến `EXPO_PUBLIC_*` bị nhúng vào bundle lúc khởi động |
| Bấm một lượt `eas build --profile development --platform android` | expo.dev | build tốn hạn mức tài khoản của bạn; máy này không có JDK/Android SDK nên không build cục bộ được |
| ~~Chọn 6.A hay 6.B~~ | — | **đã chốt 6.B (04/11/2026)** vì vòng test máy vẫn cần Expo Go; còn mở: Android thôi hay cả iOS |

Client ID **không** phải bí mật (nó hiện ra trong bundle), nên việc điền vào `.env` là an toàn; điều
kiêng tuyệt đối là **client secret** — không đưa vào app, không commit, không gửi trong chat.


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
