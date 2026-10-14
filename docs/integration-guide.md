# TelcoAds — Tài liệu hướng dẫn tích hợp

**Phiên bản:** 2.1 (Web SDK v2.0.1) · **Cập nhật:** 14/10/2026

Tài liệu này hướng dẫn đối tác tích hợp giải pháp TelcoAds vào hệ thống quảng cáo, bao gồm tích hợp Web SDK trên website và kết nối API từ backend. SDK v1 (`POST /identify`) đã ngừng hoạt động; đối tác đang dùng v1 cần nâng cấp theo mục [Nâng cấp từ v1](#nâng-cấp-từ-v1).

## Mục lục

- [Tổng quan](#tổng-quan)
  - [Luồng nhắm quảng cáo](#luồng-nhắm-quảng-cáo)
  - [Phân biệt các vai trò](#phân-biệt-các-vai-trò)
  - [Có gì mới ở v2](#có-gì-mới-ở-v2)
- [Tích hợp Website với Web SDK](#tích-hợp-website-với-web-sdk)
  - [Cài đặt SDK](#cài-đặt-sdk)
  - [Nhúng và khởi tạo](#nhúng-và-khởi-tạo)
  - [Tham số `init()`](#tham-số-init)
  - [Hành vi SDK](#hành-vi-sdk)
  - [Lưu ý khi tích hợp](#lưu-ý-khi-tích-hợp)
  - [Nâng cấp từ v1](#nâng-cấp-từ-v1)
  - [Tự kiểm tra sau khi tích hợp](#tự-kiểm-tra-sau-khi-tích-hợp)
- [Tích hợp Backend với TelcoAds API](#tích-hợp-backend-với-telcoads-api)
- [Đặc tả API](#đặc-tả-api)
  - [API Định danh — GET /ident](#api-định-danh--get-ident)
  - [Lấy kết quả dự phòng — GET /ident/result](#lấy-kết-quả-dự-phòng--get-identresult)
  - [Mã lỗi API Định danh](#mã-lỗi-api-định-danh)
  - [API Danh mục Segment — GET /v1/segments](#api-danh-mục-segment--get-v1segments)
  - [API Đối ứng người dùng — POST /v1/match](#api-đối-ứng-người-dùng--post-v1match)
- [Xử lý lỗi và retry](#xử-lý-lỗi-và-retry)

## Tổng quan

### Luồng nhắm quảng cáo

Giải pháp TelcoAds cung cấp 3 API cho luồng nhắm quảng cáo (ngoài ra `GET /ident/result` là API dự phòng do SDK tự gọi):

- **API Định danh — GET /ident** (gọi qua SDK): Cấp mã định danh tạm thời (`sid`) của người dùng/thuê bao cho phiên truy cập.
- **API Danh mục Segment — GET /v1/segments** (gọi từ backend): Lấy danh mục segment để cấu hình chiến dịch (ngoài luồng real time).
- **API Đối ứng người dùng — POST /v1/match** (gọi từ backend): Đối ứng segment của người dùng với danh sách segment mà Adtech quan tâm.

```
Trình duyệt người dùng               TelcoAds                    Adtech / DSP (server)
──────────────────────               ────────                    ─────────────────────
TelcoAds.getSid() ── GET /ident ──►  định danh qua cổng
                  ◄── sid ────────   xác thực nhà mạng
ad request + sid ─────────────────────────────────────────────►  POST /v1/match
                                                                 ◄── matched_smg_ids
```

Các API không trả số thuê bao hay thông tin nhận dạng trực tiếp của người dùng. Số thuê bao được mã hoá và tạo thành `sid`; đây là mã định danh phiên tạm thời, chỉ sử dụng trong luồng nhắm quảng cáo.

### Phân biệt các vai trò

| Khái niệm | Định nghĩa | Vai trò trong TelcoAds | Ví dụ |
|---|---|---|---|
| **Publisher** | Đơn vị sở hữu điểm tiếp xúc với người dùng. | Nhúng Web SDK; SDK gọi `GET /ident` khi người dùng truy cập để nhận `sid`. Một Publisher được nhận `client_id`. | Trang tin, website nội dung. |
| **DSP** | Nền tảng phục vụ bên mua quảng cáo. | Gọi `GET /v1/segments` để lấy danh mục segment và `POST /v1/match` để đối ứng. DSP dùng credential riêng. | Hệ thống mua quảng cáo tự động của đối tác. |
| **Site** | Một website cụ thể thuộc Publisher. | Được nhận diện bằng `site_id`; `sid` được cấp theo từng site. | Publisher A có `news.example` và `sport.example` là hai Site khác nhau. |

### Có gì mới ở v2

- SDK v2 định danh thuê bao qua **cổng xác thực của nhà mạng** (`GET /ident`) thay cho tra theo địa chỉ IP (`POST /identify`) của v1.
- **Mã gọi SDK phía Publisher không đổi:** vẫn `TelcoAds.init()` rồi `TelcoAds.getSid()`, nhận về `sid` hoặc `null`; cần lưu ý thêm HTTPS, đăng ký domain và CSP.
- Không cần cấu hình endpoint: SDK mặc định dùng môi trường Production.
- Chi tiết thay đổi tham số xem tại [Nâng cấp từ v1](#nâng-cấp-từ-v1).

## Tích hợp Website với Web SDK

SDK JavaScript của TelcoAds là 1 file vanilla JS (~4,7 KB), không dependency, không cần build. SDK gọi `GET /ident`, khi cần thì mở cổng xác thực qua iframe (không mở cửa sổ bật lên), rồi trả `sid` cho trang. `POST /v1/match` là luồng server-to-server của Adtech, SDK không gọi.

### Cài đặt SDK

Nhúng file `telcoads-sdk-v2.min.js` từ bản phát hành `v2.0.1` của repository này qua jsDelivr. Luôn ghim theo tag phát hành (không dùng `@main`) và giữ thuộc tính `integrity` để trình duyệt từ chối chạy nếu nội dung file bị thay đổi:

```html
<script src="https://cdn.jsdelivr.net/gh/BigDataServices-ViettelTelecom/telcoads-webapp-sdk@v2.0.1/dist/telcoads-sdk-v2.min.js"
        integrity="sha384-zyIFGucuXHVBakv1SMNADojUXbENS8WxfVKkyKt5qzLI2IYxhdMJyZx8qSK0Ldmj"
        crossorigin="anonymous"></script>
```

Nếu muốn tự host, tải `dist/telcoads-sdk-v2.min.js` tại tag `v2.0.1` và giữ nguyên thuộc tính `integrity` ở trên.

### Nhúng và khởi tạo

```html
<script src="https://cdn.jsdelivr.net/gh/BigDataServices-ViettelTelecom/telcoads-webapp-sdk@v2.0.1/dist/telcoads-sdk-v2.min.js"
        integrity="sha384-zyIFGucuXHVBakv1SMNADojUXbENS8WxfVKkyKt5qzLI2IYxhdMJyZx8qSK0Ldmj"
        crossorigin="anonymous"></script>
<script>
  TelcoAds.init({
    siteId:   '{{site-id}}',              // mã website đã đăng ký với TelcoAds
    clientId: '{{publisher-client-id}}',  // mã Publisher do TelcoAds cấp
    timeoutMs: 10000                      // chờ /ident tối đa 10 giây (giai đoạn pilot)
  });

  // tại thời điểm gửi ad request:
  TelcoAds.getSid().then(function (sid) {
    // sid là chuỗi  -> gắn vào ad request gửi Adtech
    // sid là null   -> gửi ad request KHÔNG kèm sid (quảng cáo không cá nhân hoá)
  });
</script>
```

- `getSid()` **không bao giờ throw/reject**: mọi lỗi đều trả về `null` để luồng quảng cáo của Publisher không bị gián đoạn.
- `init()` **throw `TypeError` ngay khi cấu hình sai** (thiếu `siteId`/`clientId`, `siteId` chứa `~`, `identUrl` rỗng). Đây là lỗi tích hợp, cần phát hiện lúc phát triển. Nếu giá trị cấu hình lấy từ nguồn động (biến môi trường, CMS…), nên bọc `init()` trong `try/catch`.

`sid` được gắn vào ad request theo đặc tả của hệ thống Adtech bạn sử dụng (tên trường/khoá do Adtech quy định); TelcoAds không quy định vị trí này.

### Tham số `init()`

Chỉ `siteId` và `clientId` là bắt buộc; các tham số khác đã có giá trị mặc định phù hợp cho Production.

| Tham số | Bắt buộc | Mặc định | Mô tả |
|---|---|---|---|
| `siteId` | Có | — | Mã website; `sid` được cấp theo từng site. Không chứa ký tự `~`. |
| `clientId` | Có | — | Mã Publisher do TelcoAds cấp. Sai hoặc chưa kích hoạt sẽ nhận `401`. |
| `identUrl` | Không | Production | URL đầy đủ tới `/ident` (ví dụ `https://<host>/…/ident`). Chỉ truyền khi TelcoAds cấp môi trường khác (ví dụ UAT). SDK tự suy ra `/ident/result` từ giá trị này. |
| `authTimeoutMs` | Không | `8000` | Thời gian chờ tối đa kết quả xác thực qua iframe. |
| `timeoutMs` | Không | `2000` | Thời gian chờ tối đa của request `/ident`; quá thời gian SDK trả `null`. Giai đoạn pilot khai báo `10000` để không bỏ sót lượt định danh khi máy chủ phản hồi chậm. |
| `negativeTtlS` | Không | `60` | Số giây SDK nhớ kết quả "không định danh được" trước khi thử lại. |
| `debug` | Không | `false` | Bật cảnh báo `console.warn` khi có lỗi. |

### Hành vi SDK

- `sid` và thời hạn được lưu ở localStorage khoá `telcoads:sid`; SDK tự trừ hao 5 giây trước hạn và tự định danh lại khi hết hạn hoặc khi bản ghi thuộc site khác.
- Nhiều ad slot cùng gọi `getSid()` chỉ sinh 1 phiên định danh (single-flight).
- Khi đã có sẵn kết quả, `/ident` có thể trả `sid` ngay mà không cần qua cổng xác thực.
- Trường hợp còn lại: SDK mở cổng xác thực qua iframe (không mở cửa sổ bật lên) và nhận kết quả qua `postMessage`. SDK chỉ chấp nhận thông điệp từ đúng máy chủ TelcoAds và đúng mã phiên. Nếu không nhận được thông điệp, SDK tự hỏi lại kết quả qua `/ident/result` mỗi 1,5 giây.
- Không định danh được (`no_match`: Wi-Fi, ngoài mạng Viettel) hoặc sai `clientId` (401): trả `null` và nhớ kết quả trong `negativeTtlS` giây. Sai `clientId` luôn có cảnh báo console, kể cả khi `debug: false`.
- Lỗi tạm (429, 5xx, timeout, lỗi mạng, phiên hết hạn): trả `null` và tạm dừng gọi lại 10 giây; không ghi vào localStorage.
- Trình duyệt chặn localStorage (chế độ riêng tư, hết quota): SDK tự chuyển sang bộ nhớ tạm trong phiên trang.

### Lưu ý khi tích hợp

- **HTTPS:** trang nhúng SDK phải chạy trên HTTPS. TelcoAds từ chối (400) request định danh từ trang `http://` (trong giai đoạn pilot chấp nhận `http://localhost` để phát triển).
- **Domain:** gửi TelcoAds danh sách domain sẽ nhúng SDK (bao gồm `www`, subdomain và trang staging, mỗi origin một dòng) để được cấp `siteId`/`clientId` và đăng ký trước khi tích hợp; TelcoAds xác nhận `siteId` tương ứng với từng origin.
- **Thời điểm gọi:** gọi `getSid()` sau khi trang đã có thẻ `<body>` (SDK có thể cần gắn iframe).
- **Content-Security-Policy:** nếu website dùng CSP, bổ sung `script-src https://cdn.jsdelivr.net` (nếu nhúng qua jsDelivr); `connect-src https://affapi.viettel.vn`; `frame-src https://affapi.viettel.vn https://developers-private.viettel.vn`.
- SDK v1 (`POST /identify`) đã ngừng hoạt động. Gỡ bản v1 khỏi trang trước khi nhúng v2 (hai bản dùng chung biến toàn cục `TelcoAds`).
- Không nhúng `DSP_CLIENT_SECRET` hoặc bất kỳ credential DSP nào vào trang web. Các API dành cho DSP luôn gọi từ backend.
- `sid` có thời hạn ngắn (mặc định 300 giây); sau khi hết hạn không thể dùng để match segment.
- Coi `sid = null` là trạng thái bình thường (người dùng dùng Wi-Fi, ngoài mạng Viettel…): vẫn gửi ad request, chỉ là không có dữ liệu segment.

### Nâng cấp từ v1

SDK v1 đã ngừng hoạt động: `POST /identify` không còn phục vụ, đối tác đang dùng v1 bắt buộc chuyển sang v2.

| Hạng mục | v1.0.0 | v2.0.1 |
|---|---|---|
| File | `dist/telcoads-sdk.min.js` | `dist/telcoads-sdk-v2.min.js` |
| API SDK gọi | `POST /identify` | `GET /ident` (+ `GET /ident/result` dự phòng) |
| `endpoint` | Bắt buộc | **Bỏ** — thay bằng `identUrl` (không bắt buộc, mặc định Production) |
| `channelType` | Có | **Bỏ** |
| `devIp` | Có | **Bỏ** |
| Tham số mới | — | `identUrl`, `authTimeoutMs` |
| `getSid()`, khoá localStorage | Giữ nguyên | Giữ nguyên |

Các bước nâng cấp:

1. Đổi thẻ `<script>` sang `telcoads-sdk-v2.min.js` tại tag `v2.0.1`, kèm `integrity` mới.
2. Bỏ `endpoint`, `channelType`, `devIp` khỏi `init()`.
3. Thêm `timeoutMs: 10000` vào `init()` trong giai đoạn pilot.
4. Kiểm tra CSP (nếu có) theo mục [Lưu ý khi tích hợp](#lưu-ý-khi-tích-hợp).

### Tự kiểm tra sau khi tích hợp

| Tình huống | Kết quả mong đợi |
|---|---|
| Mở trang bằng **4G Viettel** (điện thoại tắt Wi-Fi, hoặc máy tính bắt hotspot 4G Viettel) | `getSid()` trả về một chuỗi `sid` (khác `null`) |
| Gọi lại `getSid()` khi `sid` còn hạn (dưới 5 phút) | Trả cùng `sid`, không phát sinh request mạng |
| Mở trang bằng **Wi-Fi** hoặc mạng khác | `getSid()` trả `null`; trang và quảng cáo vẫn hoạt động bình thường |
| `clientId` sai | `getSid()` trả `null`, Console có cảnh báo `[TelcoAds] ident 401` |

Môi trường kiểm tra: trang staging HTTPS đã đăng ký, hoặc máy tính bắt hotspot 4G Viettel và mở trang qua `http://localhost` (trong giai đoạn pilot, môi trường Production chấp nhận `http://localhost` để phát triển). Sau khi sửa cấu hình (ví dụ `clientId` sai), xoá kết quả SDK đã nhớ bằng `localStorage.removeItem('telcoads:sid')` rồi tải lại trang.

Bật `debug: true` khi kiểm tra để xem cảnh báo của SDK trong Console. Khi cần TelcoAds hỗ trợ, gửi kèm `request_id` trong response (xem tab Network của DevTools).

## Tích hợp Backend với TelcoAds API

Phần này áp dụng khi Adtech/DSP tích hợp trực tiếp từ backend (gọi `GET /v1/segments` hoặc `POST /v1/match`). Điền các giá trị sau theo môi trường được TelcoAds cấp:

| Biến | Ví dụ | Ghi chú |
|---|---|---|
| `TELCOADS_BASE_URL` | `https://{{api-host}}` | Base URL theo môi trường. Không thêm dấu `/` ở cuối. |
| `PUBLISHER_CLIENT_ID` | `{{publisher-client-id}}` | Mã Publisher cho API Định danh. |
| `DSP_CLIENT_ID` | `{{dsp-client-id}}` | Mã DSP cho API Danh mục Segment. |
| `DSP_CLIENT_SECRET` | `{{dsp-client-secret}}` | Bí mật DSP. Lưu trong secret manager, không đưa vào mã nguồn. |
| `SITE_ID` | `{{site-id}}` | Mã website đã đăng ký với TelcoAds. |

### Quy ước chung

- Giao tiếp qua HTTPS; request và response dùng JSON UTF-8.
- Với POST, gửi header `Content-Type: application/json` và `Accept: application/json`.
- Token/credential giữa các môi trường không hoán đổi được.
- Các API trả lỗi theo cùng dạng JSON:

```json
{
  "error_code": "INVALID_INPUT",
  "error_message": "dữ liệu đầu vào không hợp lệ",
  "request_id": "04622c5a1a2c427783c100e6e77ea65c"
}
```

Nội dung `error_message` có thể được cải thiện mà không báo trước; `error_code` là mã để ứng dụng xử lý. `request_id` dùng khi cần TelcoAds hỗ trợ tra soát.

### Danh sách endpoint

| API | Ai gọi | Mục đích | Xác thực |
|---|---|---|---|
| `GET /ident` | SDK của Publisher | Mở phiên định danh qua cổng xác thực, hoặc trả `sid` ngay khi đã có sẵn kết quả | `client_id` Publisher hợp lệ + Origin |
| `GET /ident/result` | SDK của Publisher | Lấy kết quả định danh khi không nhận được `postMessage` (dự phòng) | Đúng Origin và `nonce` của phiên |
| `GET /v1/segments` | DSP | Lấy danh mục segment để cấu hình chiến dịch | HTTP Basic Auth của DSP |
| `POST /v1/match` | Adtech/DSP phía server | Đối ứng `sid` với segment mục tiêu | `sid` còn hợp lệ |

## Đặc tả API

### API Định danh — GET /ident

Mở một phiên định danh cho lượt truy cập. **SDK tự thực hiện toàn bộ phần này**; nội dung dưới đây giúp đối tác hiểu luồng, kiểm tra khi tích hợp và đối soát khi có sự cố.

Luồng gồm ba bước:

1. SDK gọi `GET /ident`.
2. Trình duyệt mở `authorize_url` của cổng xác thực trong iframe; cổng xác thực nhận diện thuê bao rồi chuyển hướng về TelcoAds.
3. TelcoAds gửi kết quả về trang Publisher qua `postMessage`, đồng thời giữ kết quả 60 giây để SDK lấy qua `GET /ident/result` nếu cần.

**Endpoint**

```
GET {{TELCOADS_BASE_URL}}/ident?site_id={{site-id}}&client_id={{publisher-client-id}}&nonce={{nonce}}
```

**Header**

| Tên | Bắt buộc | Ví dụ | Mô tả |
|---|---|---|---|
| `Origin` | Có (trình duyệt tự gắn) | `https://news.example.vn` | Trang gốc của Publisher, phải là HTTPS. Kết quả định danh chỉ trả về cho đúng trang gốc này. |

**Tham số query**

| Tham số | Kiểu | Bắt buộc | Validation | Mô tả |
|---|---|---|---|---|
| `site_id` | string | Có | Không rỗng; không chứa ký tự `~` | Mã website Publisher đã đăng ký. |
| `client_id` | string | Có | Không rỗng; Publisher đang hoạt động | Mã Publisher được TelcoAds cấp. |
| `nonce` | string | Có | Đúng 24 ký tự hex | Mã ngẫu nhiên của phiên do SDK sinh; dùng để khớp kết quả với đúng phiên. |

**Response thành công — 200 OK**

Có hai dạng response 200.

*Dạng 1 — cần qua cổng xác thực* (không có trường `status`):

| Trường | Kiểu | Mô tả |
|---|---|---|
| `authorize_url` | string | Địa chỉ cổng xác thực để trình duyệt mở. Chỉ dùng một lần, hiệu lực khoảng 30 giây. |
| `state_id` | string | Mã phiên định danh, dùng cho `GET /ident/result`. |
| `expire_in` | integer | Số giây phiên định danh còn hiệu lực (mặc định 120). |
| `error_code`, `error_message` | string | Chuỗi rỗng khi thành công. |
| `request_id` | string | Mã truy vết của request. |

```json
{
  "authorize_url": "https://{{auth-host}}/...",
  "state_id": "Zk3vQ0c1hXW2mN8pR5tYbA",
  "expire_in": 120,
  "error_code": "",
  "error_message": "",
  "request_id": "7bfaae7e8e67f34a71a8b7250c999f78"
}
```

*Dạng 2 — có kết quả ngay* (có trường `status`): kết quả `ok` hoặc `no_match` được trả ngay khi đã có sẵn. Cấu trúc giống [Kết quả định danh](#kết-quả-định-danh).

#### Kết quả định danh

Đối tượng JSON TelcoAds gửi về trang Publisher qua `postMessage`; cùng đối tượng này được trả ở response dạng 2 của `/ident` và ở `/ident/result`.

| Trường | Kiểu | Mô tả |
|---|---|---|
| `source` | string | Luôn là `telcoad-ident`; SDK dùng để lọc thông điệp. |
| `status` | string | `ok`: định danh thành công · `no_match`: không định danh được (Wi-Fi, ngoài mạng Viettel) · `error`: lỗi xử lý, xem `error_code` · `pending` / `expired`: chỉ có ở `/ident/result`. |
| `ident_token` | string | Chỉ có khi `status = ok`. Đây chính là `sid`, dùng cho `POST /v1/match`. |
| `expire_time` | integer | Chỉ có khi `status = ok`. Thời điểm `sid` hết hạn (Unix time, giây); mặc định 300 giây kể từ lúc cấp. |
| `nonce` | string | Giá trị `nonce` SDK đã gửi ở `/ident`. |
| `error_code`, `error_message` | string | Chuỗi rỗng khi không có lỗi. |
| `request_id` | string | Mã truy vết; gửi kèm khi cần TelcoAds hỗ trợ. |

```json
{
  "source": "telcoad-ident",
  "status": "ok",
  "ident_token": "{{opaque-sid}}",
  "expire_time": 1787654017,
  "nonce": "9f2c4a7be01d35c68a4b17e2",
  "error_code": "",
  "error_message": "",
  "request_id": "5d0f6c1e9a3b4c7d8e2f1a0b9c8d7e6f"
}
```

`no_match` **không phải lỗi**: đó là kết quả hợp lệ khi người dùng không thuộc phạm vi định danh. Hãy rẽ sang quảng cáo không cá nhân hoá.

### Lấy kết quả dự phòng — GET /ident/result

Dùng khi trang không nhận được `postMessage` (iframe bị chặn, mất thông điệp…). Kết quả được giữ 60 giây, **chỉ đọc được một lần** và chỉ trả cho đúng trang gốc (Origin) cùng `nonce` đã mở phiên.

```
GET {{TELCOADS_BASE_URL}}/ident/result?state_id={{state-id}}&nonce={{nonce}}
```

| Tham số | Kiểu | Bắt buộc | Mô tả |
|---|---|---|---|
| `state_id` | string | Có | Giá trị nhận từ `/ident`. |
| `nonce` | string | Có | Trùng `nonce` đã gửi ở `/ident`. |

Response 200 là đối tượng [Kết quả định danh](#kết-quả-định-danh), trong đó `status` có thêm:

- `pending` — chưa có kết quả, hỏi lại sau.
- `expired` (`error_code = STATE_EXPIRED`) — phiên đã hết hạn hoặc kết quả đã được đọc; dừng hỏi.

```json
{
  "source": "telcoad-ident",
  "status": "pending",
  "error_code": "",
  "error_message": "",
  "request_id": "0a1b2c3d4e5f60718293a4b5c6d7e8f9"
}
```

### Mã lỗi API Định danh

**Lỗi trả bằng mã HTTP** (`GET /ident`, `GET /ident/result`):

| HTTP | error_code | Nguyên nhân thường gặp | Xử lý |
|---|---|---|---|
| 400 | `INVALID_INPUT` | Thiếu hoặc rỗng `site_id`, `client_id` · `site_id` chứa `~` · `nonce` không phải 24 ký tự hex · trang gọi không phải HTTPS (thiếu/sai Origin) · `state_id` sai định dạng | Sửa tích hợp, không retry cùng dữ liệu. |
| 401 | `UNAUTHORIZED` | `client_id` không tồn tại, không hoạt động hoặc không có quyền Publisher | Kiểm tra `clientId` và môi trường. |
| 403 | `FORBIDDEN` | Gọi `/ident/result` từ trang gốc khác, thiếu `nonce` hoặc `nonce` khác phiên đã mở | Không retry. |
| 404 | `IDENT_DISABLED` | Luồng định danh chưa được bật ở môi trường đang gọi | Liên hệ TelcoAds. |
| 429 | `RATE_LIMIT_EXCEEDED` | Vượt số lượng request định danh cho phép | SDK tự tạm dừng 10 giây. |
| 500 | `INTERNAL_ERROR` | Lỗi hệ thống | SDK tự tạm dừng 10 giây. |

**Lỗi nằm trong Kết quả định danh** (`status = error` hoặc `expired`):

| error_code | Nguyên nhân thường gặp |
|---|---|
| Lỗi xác thực | Lỗi phát sinh trong quá trình xác thực thuê bao (phiên bị từ chối, phản hồi không hợp lệ, phản hồi chậm…). |
| `STATE_EXPIRED` | Phiên định danh quá hạn (mặc định 120 giây) hoặc kết quả đã được đọc. |
| `INTERNAL_ERROR` | Lỗi hệ thống. |

SDK xử lý tất cả các lỗi trên bằng cách trả `null` và tạm dừng 10 giây; đối tác không cần tự xử lý. Khi cần hỗ trợ, gửi kèm `request_id` cho TelcoAds.

### API Danh mục Segment — GET /v1/segments

DSP dùng endpoint này để tải danh mục segment phục vụ cấu hình chiến dịch. Nên đồng bộ danh mục định kỳ theo nhu cầu vận hành; không gọi endpoint này trong đường xử lý quảng cáo thời gian thực.

**Endpoint và xác thực**

```
GET {{TELCOADS_BASE_URL}}/v1/segments
Authorization: Basic base64({{dsp-client-id}}:{{dsp-client-secret}})
Accept: application/json
```

Chỉ credential được đăng ký với vai trò DSP mới gọi được endpoint này. Không truyền `client_secret` qua query string hoặc log ứng dụng.

**Request**

Endpoint không có query parameter và không có request body.

| Header | Kiểu | Bắt buộc | Validation | Mô tả |
|---|---|---|---|---|
| `Authorization` | string | Có | Định dạng `Basic <base64(client_id:secret)>`; DSP đang hoạt động | Thông tin xác thực DSP. |
| `Accept` | string | Khuyến nghị | `application/json` | Kiểu dữ liệu response mong muốn. |

**Response thành công — 200 OK**

| Trường | Kiểu | Mô tả |
|---|---|---|
| `segments` | array\<object\> | Danh sách segment hiện có. Có thể là mảng rỗng. |
| `segments[].segment_id` | string | Mã segment; sử dụng giá trị này trong `requested_smg_ids`. |
| `segments[].segment_name` | string | Tên hiển thị của segment. |
| `segments[].segment_group` | string | Nhóm segment. |
| `segments[].is_active` | boolean | Trạng thái kích hoạt của segment. |
| `error_code` | string | Chuỗi rỗng khi thành công. |
| `error_message` | string | Chuỗi rỗng khi thành công. |

```json
{
  "segments": [
    {
      "segment_id": "001",
      "segment_name": "Người quan tâm thể thao",
      "segment_group": "interest",
      "is_active": true
    }
  ],
  "error_code": "",
  "error_message": ""
}
```

**Bảng mã lỗi**

| HTTP | error_code | Nguyên nhân thường gặp | Xử lý |
|---|---|---|---|
| 401 | `DSP_UNAUTHORIZED` | Thiếu header `Authorization` · header không đúng định dạng Basic Auth · `client_id` không tồn tại hoặc không hoạt động · secret không khớp · credential không có vai trò DSP | Kiểm tra base URL và credential, không retry cùng credential. |
| 500 | `INTERNAL_ERROR` | Lỗi hệ thống | Retry giới hạn với exponential backoff; dùng bản danh mục đã đồng bộ gần nhất nếu phù hợp. |

### API Đối ứng người dùng — POST /v1/match

Adtech/DSP phía server dùng API này để kiểm tra segment nào trong danh sách mục tiêu khớp với người dùng đang có `sid`. API chỉ trả các `segment_id` nằm trong danh sách gửi lên và giữ nguyên thứ tự của danh sách đó.

Không gọi endpoint này trực tiếp từ browser: đây là luồng server-to-server. Không ghi `sid` vào log hoặc gửi sang hệ thống phân tích bên thứ ba.

**Endpoint**

```
POST {{TELCOADS_BASE_URL}}/v1/match
```

**Header**

| Tên | Bắt buộc | Ví dụ | Mô tả |
|---|---|---|---|
| `Content-Type` | Có | `application/json` | Kiểu dữ liệu request. |
| `Accept` | Khuyến nghị | `application/json` | Kiểu dữ liệu response mong muốn. |

**Body request**

| Trường | Kiểu | Bắt buộc | Validation | Mô tả |
|---|---|---|---|---|
| `sid` | string | Có | Không rỗng; là `sid` TelcoAds đã cấp và còn hiệu lực | Giá trị nhận từ `TelcoAds.getSid()` (trường `ident_token` của API Định danh). |
| `requested_smg_ids` | array\<string\> | Có | Danh sách segment mục tiêu, không rỗng | Các mã segment (smg = segment) cần đối ứng. Khuyến nghị chỉ dùng `segment_id` có `is_active: true`. |

```json
{
  "sid": "{{opaque-sid}}",
  "requested_smg_ids": ["001", "003", "004"]
}
```

**Response thành công — 200 OK**

| Trường | Kiểu | Mô tả |
|---|---|---|
| `sid` | string | Giá trị `sid` đã gửi trong request. |
| `matched_smg_ids` | array\<string\> | Các segment khớp. Mảng rỗng là kết quả hợp lệ: người dùng không thuộc segment nào bạn yêu cầu. Thứ tự theo `requested_smg_ids`. |
| `error_code` | string | Chuỗi rỗng khi thành công. |
| `error_message` | string | Chuỗi rỗng khi thành công. |

```json
{
  "sid": "{{opaque-sid}}",
  "matched_smg_ids": ["003"],
  "error_code": "",
  "error_message": ""
}
```

**Bảng mã lỗi**

| HTTP | error_code | Nguyên nhân thường gặp | Xử lý |
|---|---|---|---|
| 400 | `INVALID_INPUT` | JSON không hợp lệ · thiếu hoặc rỗng `sid` · thiếu hoặc rỗng `requested_smg_ids` | Sửa request, không retry cùng dữ liệu. |
| 401 | `UNAUTHORIZED` | `sid` không hợp lệ · `sid` đã bị sửa đổi | Không retry. |
| 401 | `SID_NOT_FOUND` | `sid` đã hết hạn | Lấy `sid` mới (SDK tự định danh lại ở lần `getSid()` kế tiếp), sau đó gọi lại match. |
| 404 | `HASHID_NOT_FOUND` | Không còn dữ liệu phiên tương ứng với `sid` | Coi là không thể đối ứng tại thời điểm đó. |
| 500 | `INTERNAL_ERROR` | Lỗi hệ thống | Retry giới hạn với exponential backoff; nếu không thành công, dùng luồng không cá nhân hoá. |

## Xử lý lỗi và retry

| Nhóm lỗi | Ví dụ | Có retry? | Hành động khuyến nghị |
|---|---|---|---|
| Lỗi request | `400 INVALID_INPUT` | Không | Sửa dữ liệu hoặc mã tích hợp. |
| Lỗi xác thực/ủy quyền | `401 UNAUTHORIZED`, `401 DSP_UNAUTHORIZED`, `403 FORBIDDEN` | Không | Kiểm tra credential, môi trường và quyền tích hợp. |
| Kết quả không có dữ liệu | `status no_match`, `404 HASHID_NOT_FOUND`, `401 SID_NOT_FOUND` | Không cùng request | Rẽ sang quảng cáo không cá nhân hoá; riêng `SID_NOT_FOUND` cần `sid` mới. |
| Lỗi tạm thời phía định danh | `429`, lỗi xác thực, `STATE_EXPIRED`, timeout | SDK tự xử lý | SDK trả `null` và tự tạm dừng 10 giây. **Không tự viết vòng lặp gọi `getSid()`.** |
| Lỗi tạm thời phía backend | `500 INTERNAL_ERROR` ở `/v1/segments`, `/v1/match` | Có, giới hạn | Retry exponential backoff, tối đa 2–3 lần, không vượt ngân sách thời gian xử lý quảng cáo. |

Ví dụ backoff: chờ ngẫu nhiên quanh 100 ms, 300 ms, 900 ms; dừng ngay khi còn ít thời gian xử lý quảng cáo.

Tài liệu này mô tả hợp đồng tích hợp công khai. Cơ chế xử lý, nguồn dữ liệu và lưu trữ nội bộ không thuộc phạm vi tài liệu.
