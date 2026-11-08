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
| **Bản dựng thật (APK)`preview`** trên điện thoại | ✅ consent + callback chạy 06/11/2026; ⚠️ mọi request API bị Android chặn cho tới build #4 | Trên máy, `useIdTokenAuthRequest` chuyển sang luồng authorization-code + PKCE và **tự đổi code lấy id_token mà không cần client secret** (`node_modules/expo-auth-session/build/providers/Google.js`, `useAuthRequest` + `AccessTokenRequest`). Client loại **Web application** thì chết ở bước đổi code đó, nên app phải xin thêm client loại Android/iOS — đây chính là 6.B đã cài. Redirect mặc định của thư viện là `com.playtogether.app:/oauthredirect`; app gửi redirect dạng AppAuth derive từ client ID (`com.googleusercontent.apps.<id>:/oauth2redirect`) và `app.config.ts` khai scheme đó — đã đo: `npx expo config --type public` với client Android trong env trả `scheme = ["playtogether","com.googleusercontent.apps.606945851708-cj8l2b67qie9ougprndr7vn1k9uof7k1"]`. **Cái bẫy chỉ bản release mới có**: Android 9+ mặc định cấm HTTP không mã hoá, mà API dev là `http://<lan-ip>:3000`, nên app báo "Không thể kết nối máy chủ" dù server vẫn sống — Expo Go không bị vì manifest của chính Expo Go cho phép cleartext. Xem mục 6.D. |

## 6. Chạy trên máy thật — đã chốt 6.B (04/11/2026), 6.A ghi lại để dành

Điểm chung cả hai: **Expo Go không bao giờ chạy được** (mục 5), và đều cần một *bản dựng thật* do
chính tay chủ tài khoản dựng, vì nó đòi tài khoản Expo và quyền sửa project Google Cloud.
`eas.json` đã nằm trong repo với bốn profile: `development` (dev client — APK Android / app simulator
iOS), `development-device` (dev client dựng cho máy thật iOS), `preview` (APK nội bộ chia tay ngắn
hạn), `production` (bản submit store).

**Ngày 06/11/2026 đã chạy thật, và bản dùng được là `preview`, không phải `development`.** Lý do rất
đời thường: profile `development` đặt `developmentClient: true` nên EAS từ dựng với
`Install expo-dev-client manually` — thư viện đó **cố ý không cài** (mục 6.A giải thích: cài vào là
`npx expo start` nhắm dev build thay vì Expo Go). Profile `preview` cho ra một APK độc lập, cài là chạy,
không phải kéo Metro. Hai profile cũng dùng **chung một keystore** (`Build Credentials A6ccPXjA8H
(default)`), nên SHA-1 lấy từ bản `preview` khai cho client Android là đúng cho mọi bản sau này.

```bash
cd play-together-fe
npx eas-cli login                  # tài khoản expo.dev của bạn
npx eas-cli init                   # đã chạy 06/11/2026: tạo @manhnguyendev/play-together,
                                   # ghi extra.eas.projectId + owner vào app.json
npx eas-cli build --profile preview --platform android
```

`eas.json` hiện khai `environment` trong profile (`development` → `development`, `preview` →
`preview`): đó là cách EAS **Variables** biết bộ biến nào bơm vào lượt dựng. Bắt buộc phải đi đường này
vì `.env` nằm trong `.gitignore` nên **không** được upload lên máy dựng — build log in ra
`Environment variables … loaded from the "preview" environment on EAS` là bằng chứng duy nhất nhìn thấy
được từ máy local. Biến đang có trong environment `preview`: `EXPO_PUBLIC_API_URL`,
`EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID`, `EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID`.

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

**Việc phía tài khoản — cập nhật 06/11/2026.**

1. ✅ **Đã tạo** OAuth client loại Android trong Google Cloud project `play-together-513312`:
   Name `Play Together - Android`, Package name `com.playtogether.app`,
   SHA-1 `4D:86:07:82:4F:65:21:E3:CF:E7:13:51:D4:55:02:E9:49:DF:9C:10`,
   Client ID `606945851708-cj8l2b67qie9ougprndr7vn1k9uof7k1.apps.googleusercontent.com`.
   Giá trị này **không** phải bí mật (client ID loại Android không có secret), nên ghi công khai ở đây
   để lần sau khỏi phải dựng lại lấy SHA-1.
   **Bắt buộc thêm một bước mà Console không tự làm:** sau khi tạo client Android, mở
   *Advanced settings* và bật **Enable custom URI scheme** rồi Save. Client Android tạo mới mặc định **tắt**
   tuỳ chọn này (Google ghi "not recommended for Android clients" vì họ đang đẩy sang loopback IP / App
   Links), trong khi luồng AppAuth của `expo-auth-session` chỉ có thể dùng redirect
   `com.googleusercontent.apps.<id>:/oauth2redirect`. Thiếu bước này thì Google trả thẳng trang
   `400: invalid_request` — "Custom URI scheme is not enabled for your Android client" — và app không bao
   giờ nhận được callback, nên không có toast nào của app cho lỗi này. Bật xong không cần build lại:
   client ID giữ nguyên, scheme đã nằm sẵn trong `AndroidManifest.xml` của bản dựng.
   Tái hiện để xác nhận (không cần điện thoại): tự dựng URL `accounts.google.com/o/oauth2/v2/auth` với
   `response_type=code`, `client_id` = client Android, `redirect_uri` = đúng chuỗi AppAuth, `scope=openid
   profile email`, kèm `code_challenge` + `code_challenge_method=S256`; mở trong trình duyệt → trước khi
   bật là trang 400, sau khi bật là màn *Chọn tài khoản*. Lý do Google hiện ra là vì nó *không* redirect
   về app ở bước này, nên chỉ có dựng request bằng tay mới đọc được message gốc (nó nằm trong tham số
   `authError`, base64).
   *Cách lấy SHA-1 trên máy này* — để còn lặp lại, vì hai đường sách vở đều không chạy được:
   `npx eas-cli credentials` là menu **tương tác**, không có cờ nào in fingerprint; còn
   `keytool` thì không tồn tại (máy không có JDK). Thay vào đó, dựng một lượt `preview`, tải APK về rồi
   đọc **APK Signing Block**: APK ký bằng scheme v2 nên `META-INF/*.RSA` không tồn tại (không có chữ ký
   v1), phải tự tìm `EOCD` → offset Central Directory → magic `APK Sig Block 42` → cặp có ID
   `0x7109871a`, rồi cắt đúng 813 byte DER bắt đầu bằng `30 82 03 29` của certificate. Ba phép tính độc
   lập cho cùng một kết quả: `X509Certificate(...).fingerprint` của Node, `openssl x509 -inform DER
   -fingerprint -sha1`, và `openssl sha1` trên file DER. Lưu ý cạm bẫy khi tự parse: chuỗi `30 82` gặp
   đầu tiên trong section là **`tbsCertificate`** (bị cắt mất `signatureAlgorithm` + `signatureValue`),
   không phải certificate — nó vẫn `asn1parse` được nên nhìn tưởng đúng; bắt buộc phải thử
   *parse ra X509* rồi mới lấy fingerprint.
2. ⏳ **Vẫn còn** hai dòng `.env` (agent không sửa file env của bạn): `play-together-fe/.env` thêm
   `EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID=606945851708-cj8l2b67qie9ougprndr7vn1k9uof7k1.apps.googleusercontent.com`;
   `play-together-be/.env` thêm `GOOGLE_ANDROID_CLIENT_ID=<cùng một giá trị>` rồi khởi động lại API —
   `GoogleAuthService` đã nhận danh sách audience sẵn (`src/modules/auth/google-auth.service.ts:29-33`),
   **API không sửa code**. Thiếu dòng của API thì build vẫn mở được consent và vẫn nhận code, nhưng
   token mint ra có `aud` là client Android nên `verifyIdToken` từ chối → app báo lỗi audience.
3. ✅ **Đã dựng** bằng `npx eas-cli build --profile preview --platform android` (không phải
   `development`, xem mục 6). Build #1 `09a07d85…` chỉ để lấy keystore; build #2 `a13a9792…` là bản có
   client Android + scheme AppAuth biên dịch vào project native — **build lại là bắt buộc**, vì scheme
   return là cấu hình native, không phải thứ đổi được lúc chạy.

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

| Việc | Ở đâu | Tình trạng 06/11/2026 |
| --- | --- | --- |
| `eas login` | expo.dev | ✅ bạn tự đăng nhập (agent không nhận session); `eas whoami` = `manhnguyendev` |
| `eas init` | expo.dev | ✅ agent chạy khi đã có session — tạo `@manhnguyendev/play-together`, diff `app.json` được đưa ra trước khi kịp commit |
| Create OAuth client **Android** | Google Cloud Console | ✅ agent điền form dưới sự cho phép tường minh ("tạo OAuth client Android đó đi"); trước đó lần bấm nào cũng bị chặn cho tới khi có câu này |
| (+ **iOS** nếu cần) | Google Cloud Console | ⏳ mở: cần Bundle ID `com.playtogether.app` + tài khoản Apple Developer |
| Lấy SHA-1 từ bản dựng | APK + openssl | ✅ xem mục 6.B.1 — `eas credentials` tương tác và `keytool` đều không dùng được ở đây |
| Điền `EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID` vào EAS Variables | `eas env:set … --environment preview` | ✅ agent làm (client ID công khai, không phải secret) |
| Điền `EXPO_PUBLIC_GOOGLE_*` vào `play-together-fe/.env`, `GOOGLE_ANDROID_CLIENT_ID` vào `play-together-be/.env` | máy bạn | ⏳ **còn chờ** — file env local của bạn, agent không sửa |
| Bật lại API sau khi đổi env | máy bạn | ⏳ `EXPO_PUBLIC_*` nhúng vào bundle lúc dựng; `GOOGLE_ANDROID_CLIENT_ID` chỉ đọc lúc API khởi động |
| Bấm `eas build --profile preview --platform android` | expo.dev | ✅ 2 lượt (hạn mức Android 15 lượt/tháng của tài khoản Free; `eas account:usage manhnguyendev` là lệnh xem) |
| Cài APK lên máy và bấm nút Google | máy bạn | ⏳ vòng test của bạn |
| ~~Chọn 6.A hay 6.B~~ | — | **đã chốt 6.B (04/11/2026)** vì vòng test máy vẫn cần Expo Go; còn mở: Android thôi hay cả iOS |

Client ID **không** phải bí mật (nó hiện ra trong bundle), nên việc điền vào `.env` là an toàn; điều
kiêng tuyệt đối là **client secret** — không đưa vào app, không commit, không gửi trong chat.

### 6.D. Hai bẫy chỉ xuất hiện khi chạy bản release trên máy (06/11/2026)

Cả hai đều vô hình với Expo Go và vô hình với test web, nên chỉ hiện ra ở lượt bấm trên điện thoại.

**1) expo-router nuốt callback OAuth.** Triệu chứng: Google đồng ý, quay về app, và app hiện màn
`Unmatched Route / Page could not be found` bằng **tiếng Anh** (màn mặc định của framework, không phải
màn của app). Lý do: expo-router coi *mọi* URL hệ thống gửi vào app là một route, mà `/oauth2redirect`
không có screen nào. Callback vẫn tới nơi — chuỗi `playtogether:///oauth2redirect?state=…&code=…` in ra
là bằng chứng Google đã trả code thành công.
Fix: `src/app/+native-intent.ts` export `redirectSystemPath` (hook có sẵn của expo-router, xem
`node_modules/expo-router/build/types.d.ts`: trả về giá trị falsy thì **không điều hướng**), chặn đúng
path callback và trả nguyên văn mọi link khác để deep-link thông báo không bị liên luỵ. Có test
`src/features/auth/__tests__/nativeIntent.spec.ts` giữ cả hai chiều.

**2) Android cấm HTTP không mã hoá, và Expo không có khoá app.json để mở.** Triệu chứng: toast của
chính app — `Không thể kết nối máy chủ. Kiểm tra EXPO_PUBLIC_API_URL và kết nối mạng.` — trong khi
`curl http://<lan-ip>:3000/api/v2/health` từ máy tính vẫn `ok`. Đây **không** phải cấu hình sai và
**không** phải server chết: `EXPO_PUBLIC_API_URL` được nhúng lúc dựng (đã kiểm trong
`assets/index.android.bundle` của APK là có chuỗi `192.168.1.8:3000`), nên thứ chặn là nền tảng.
Bằng chứng đọc từ file: giải `AndroidManifest.xml` (AXML binary) trong APK đã ký thì
`usesCleartextTraffic`, `networkSecurityConfig`, `cleartextTrafficPermitted` **đều không có**, còn
`INTERNET` thì có — control cho thấy phép đọc đúng. Android 9 (API 28) trở lên mặc định từ chối
cleartext, và OkHttp mà RN dùng tuân thủ luật đó, nên request không rời khỏi thiết bị. Expo Go sống
sót vì manifest của *chính Expo Go* cho phép cleartext — đây là lý do defect này chưa từng xuất hiện
trong suốt các vòng test Expo Go.
Fix: `plugins/withAndroidCleartext.js` (một config plugin `withAndroidManifest` rất nhỏ) ghi
`android:usesCleartextTraffic="true"` vào `<application>`, khai trong `app.json → expo.plugins`.
**Vì sao phải viết plugin chứ không phải bật config:** docs SDK 58 tra từ khoá `cleartext` trả "not
present", và `grep` trong `@expo/config-types` + `@expo/config-plugins` + `@expo/prebuild-config` không
có chỗ nào đọc một property cleartext nào — nên không có đường cấu hình, chỉ có đường plugin.
Đã kiểm bằng `npx expo prebuild --platform android`: manifest sinh ra có thuộc tính và **vẫn còn**
intent-filter của scheme AppAuth (hai thay đổi không triệt tiêu nhau).
**Giá phải trả và điều kiện gỡ:** thuộc tính này mở cleartext cho *toàn bộ* app, không giới hạn host —
chấp nhận được với backend LAN của đồ án, nhưng là một bước lùi bảo mật nếu app ra store. Khi API chạy
HTTPS thật thì **xoá plugin và bỏ dòng khai trong `app.json`**; comment trong file plugin nhắc đúng điều
đó. (Con đường chặt hơn — `network_security_config.xml` cho phép cleartext theo từng host — không hợp ở
đây vì IP LAN của máy anh đã đổi vài lần trong project này.)

**Bài học quy trình:** hai lỗi này nối tiếp nhau và mỗi lần chỉ hiện một cái, nên thứ đáng tin là
**tầng gần nhất chưa được kiểm chứng**, không phải suy đoán từ triệu chứng. Màn tiếng Anh → lỗi của
framework; toast tiếng Việt của app → app đã chạy và đã gọi đi nơi khác; log server im lặng → hoặc
request không tới, hoặc API không log request thành công (đúng ra là vế sau, nên nó không phải bằng
chứng).


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
