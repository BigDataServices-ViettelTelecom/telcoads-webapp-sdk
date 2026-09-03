# TelcoAds — Tài liệu hướng dẫn tích hợp

**Phiên bản:** 1.0 · **Cập nhật:** 25/08/2026

Tài liệu này hướng dẫn đối tác tích hợp giải pháp TelcoAds vào hệ thống quảng cáo, bao gồm phương thức tích hợp qua SDK trên website và kết nối API từ backend. Tài liệu cung cấp các thông tin cần thiết để cài đặt, cấu hình, kết nối và sử dụng các endpoint của TelcoAds trong luồng nhắm quảng cáo.

## Mục lục

- [Tổng quan](#tổng-quan)
  - [Luồng nhắm quảng cáo](#luồng-nhắm-quảng-cáo)
  - [Phân biệt các vai trò](#phân-biệt-các-vai-trò)
  - [Tích hợp Website với Web SDK](#tích-hợp-website-với-web-sdk)
  - [Tích hợp Backend với TelcoAds API](#tích-hợp-backend-với-telcoads-api)
  - [Quy ước chung](#quy-ước-chung)
  - [Danh sách endpoint](#danh-sách-endpoint)
- [Đặc tả API](#đặc-tả-api)
  - [API Định danh — POST /identify](#api-định-danh--post-identify)
  - [API Danh mục Segment — GET /v1/segments](#api-danh-mục-segment--get-v1segments)
  - [API Đối ứng người dùng — POST /v1/match](#api-đối-ứng-người-dùng--post-v1match)
- [Xử lý lỗi và retry](#xử-lý-lỗi-và-retry)

## Tổng quan

### Luồng nhắm quảng cáo

Giải pháp TelcoAds cung cấp 3 API cho luồng nhắm quảng cáo:

- **API Định danh — POST /identify** (gọi qua SDK): Cấp mã định danh tạm thời của người dùng/thuê bao cho phiên truy cập.
- **API Danh mục Segment — GET /v1/segments** (gọi qua Back-end): Lấy danh mục segment để cấu hình chiến dịch (ngoài luồng real time).
- **API Đối ứng người dùng — POST /v1/match** (gọi qua Back-end): Đối ứng segment của người dùng với danh sách segment mà Adtech quan tâm.

Các API không trả số thuê bao hay thông tin nhận dạng trực tiếp của người dùng. Số thuê bao được mã hoá và tạo thành `sid`; đây là mã định danh phiên tạm thời, chỉ sử dụng trong luồng nhắm quảng cáo.

### Phân biệt các vai trò

| Khái niệm | Định nghĩa | Vai trò trong TelcoAds | Ví dụ |
|---|---|---|---|
| **Publisher** | Đơn vị sở hữu điểm tiếp xúc với người dùng. | SDK của Publisher gọi `POST /identify` khi người dùng truy cập website/app để nhận `sid`. Một Publisher được nhận `client_id`. | Trang tin, ứng dụng nội dung hoặc ứng dụng tích hợp SDK. |
| **DSP** | Nền tảng phục vụ bên mua quảng cáo. | Gọi `GET /v1/segments` để lấy danh mục segment và dùng các mã segment đó khi cấu hình chiến dịch. DSP dùng credential riêng. | Hệ thống mua quảng cáo tự động của đối tác. |
| **Site** | Một website hoặc ứng dụng cụ thể thuộc Publisher. | Được nhận diện bằng `site_id` trong request identify; giúp TelcoAds biết lượt truy cập thuộc điểm tiếp xúc nào. | Publisher A có thể có `news.example` và app *Example News* là hai Site khác nhau. |

### Tích hợp Website với Web SDK

Web SDK được tích hợp trên website của Publisher để lấy `sid` tại thời điểm người dùng truy cập và chuyển `sid` cùng ad request sang hệ thống Adtech.

SDK JavaScript của TelcoAds là 1 file vanilla JS, không dependency, không cần build. SDK chỉ gọi `POST /identify` để lấy `sid` cho lượt truy cập; `POST /v1/match` là luồng server-to-server của Adtech, SDK không gọi.

#### Cài đặt SDK

Đối tác không cần và không nên cấu hình Base URL hoặc môi trường. SDK tự gọi đúng endpoint theo cấu hình của bản phát hành.

TelcoAds sẽ quản lý endpoint/môi trường theo SDK được cấp (ví dụ bản UAT hoặc Production). Đối tác chỉ cần làm theo hướng dẫn khởi tạo SDK và dùng các định danh được TelcoAds cấp, chẳng hạn `site_id` và `client_id` nếu SDK yêu cầu truyền khi khởi tạo.

Không tự thay đổi endpoint, không chèn `X-Forwarded-For`, và không nhúng credential bí mật DSP vào SDK/browser. Các API DSP và đối ứng server-to-server vẫn phải được gọi từ backend.

#### Nhúng và khởi tạo

```html
<script src="https://{{sdk-cdn-host}}/telcoads-sdk.min.js"
        integrity="{{sdk-integrity-hash}}"
        crossorigin="anonymous"></script>
<script>
  TelcoAds.init({
    siteId:   '{{site-id}}',              // mã website/app đã đăng ký
    clientId: '{{publisher-client-id}}',      // mã Publisher do TelcoAds cấp
    endpoint: 'https://{{api-host}}/identify' // do TelcoAds cấp theo môi trường
  });

  // tại thời điểm gửi ad request:
  TelcoAds.getSid().then(function (sid) {
    // sid là chuỗi  -> gắn vào ad request gửi Adtech
    // sid là null   -> gửi ad request KHÔNG kèm sid (quảng cáo không cá nhân hoá)
  });
</script>
```

`getSid()` không bao giờ throw/reject: mọi lỗi đều trả về `null` để luồng quảng cáo của Publisher không bị vỡ. Ngược lại `init()` throw ngay khi cấu hình sai — đây là lỗi tích hợp, cần lộ ra lúc phát triển.

#### Tham số `init()`

| Tham số | Bắt buộc | Mặc định | Mô tả |
|---|---|---|---|
| `siteId` | Có | — | Mã website/app; `sid` được cấp theo từng site. Không chứa ký tự `~`. |
| `clientId` | Có | — | Mã Publisher do TelcoAds cấp. Sai hoặc chưa kích hoạt sẽ nhận `401`. |
| `endpoint` | Có | — | URL đầy đủ tới `/identify` theo môi trường TelcoAds cấp. |
| `channelType` | Không | `website` | Chỉ nhận `website` hoặc `app`. |
| `timeoutMs` | Không | `2000` | Thời gian chờ tối đa của request identify, tránh làm chậm tải trang. |
| `negativeTtlS` | Không | `60` | Số giây SDK nhớ kết quả không định danh được (HTTP 404 hoặc 401) trước khi gọi lại `/identify`. |
| `devIp` | Không | — | Chỉ dùng dev/test cục bộ để gửi IP giả kèm request identify. Không bật ở production. |
| `debug` | Không | `false` | Bật cảnh báo console khi có lỗi. |

#### Hành vi SDK

- `sid` và thời hạn được lưu ở localStorage khoá `telcoads:sid`; SDK tự trừ hao 5 giây trước hạn và tự gọi lại identify khi hết hạn hoặc khi bản ghi thuộc site khác.
- Nhiều ad slot cùng gọi `getSid()` chỉ sinh 1 request identify (single-flight).
- Không định danh được (404) hoặc sai `client_id` (401): trả `null` và nhớ kết quả trong `negativeTtlS` giây.
- Lỗi tạm (429, 5xx, timeout, lỗi mạng): trả `null` và tạm dừng gọi lại 10 giây; không ghi vào localStorage.
- Trình duyệt chặn localStorage (chế độ riêng tư, hết quota): SDK tự chuyển sang bộ nhớ tạm trong phiên trang.

#### Lưu ý khi tích hợp

- Không tự đổi endpoint, không tự thêm header `X-Forwarded-For`: địa chỉ IP dùng để định danh do hệ thống TelcoAds xác định từ kết nối thực tế.
- Không nhúng `DSP_CLIENT_SECRET` hoặc bất kỳ credential DSP nào vào trang web/ứng dụng. Các API dành cho DSP luôn gọi từ backend.
- `sid` sau khi hết hạn (mặc định 300 giây) sẽ không thể dùng để match segment.
- Coi `sid = null` là trạng thái bình thường (người dùng ngoài mạng Viettel, dùng Wi-Fi…): vẫn gửi ad request, chỉ là không có dữ liệu segment.

### Tích hợp Backend với TelcoAds API

Phần này chỉ áp dụng khi Adtech/DSP tích hợp trực tiếp từ backend (ví dụ gọi `GET /v1/segments` hoặc `POST /v1/match`). Điền các giá trị sau theo môi trường được TelcoAds cấp:

| Biến | Ví dụ | Ghi chú |
|---|---|---|
| `TELCOADS_BASE_URL` | `https://{{api-host}}` | Base URL theo môi trường. Không thêm dấu `/` ở cuối. |
| `PUBLISHER_CLIENT_ID` | `{{publisher-client-id}}` | Mã Publisher cho API Định danh. |
| `DSP_CLIENT_ID` | `{{dsp-client-id}}` | Mã DSP cho API Danh mục Segment. |
| `DSP_CLIENT_SECRET` | `{{dsp-client-secret}}` | Bí mật DSP cho API Danh mục Segment. Lưu trong secret manager, không đưa vào mã nguồn. |
| `SITE_ID` | `{{site-id}}` | Mã website/app đã đăng ký với TelcoAds. |

### Quy ước chung

- Giao tiếp qua HTTPS; request và response dùng JSON UTF-8, trừ API `GET /v1/segments` không có body request.
- Với POST, gửi header `Content-Type: application/json` và `Accept: application/json`.
- Với tích hợp backend trực tiếp, dùng đúng base URL TelcoAds đã cấp. Token/credential giữa các môi trường không hoán đổi được.
- `sid` có thời hạn ngắn (mặc định 300 giây). Sau khi hết thời hạn sẽ không thể dùng để match segment nữa.
- Các API trả lỗi theo cùng dạng JSON:

```json
{
  "error_code": "INVALID_INPUT",
  "error_message": "Dữ liệu đầu vào không hợp lệ"
}
```

Nội dung `error_message` có thể được cải thiện mà không báo trước; `error_code` là mã để ứng dụng xử lý.

### Danh sách endpoint

| API | Ai gọi | Mục đích | Xác thực |
|---|---|---|---|
| `POST /identify` | Publisher/SDK | Cấp `sid` tạm thời cho lượt truy cập | `client_id` Publisher hợp lệ |
| `GET /v1/segments` | DSP | Lấy danh mục segment để cấu hình chiến dịch | HTTP Basic Auth của DSP |
| `POST /v1/match` | Adtech/DSP phía server | Đối ứng `sid` với segment mục tiêu | `sid` còn hợp lệ |

## Đặc tả API

### API Định danh — POST /identify

Cấp `sid` tạm thời cho một lượt truy cập. Publisher/SDK gọi API này trước khi chuyển sang luồng quảng cáo. Thành công không đồng nghĩa người dùng chắc chắn sẽ có segment phù hợp; kết quả segment được lấy ở API match.

**Endpoint**

```
POST {{TELCOADS_BASE_URL}}/identify
```

**Header**

| Tên | Bắt buộc | Ví dụ | Mô tả |
|---|---|---|---|
| `Content-Type` | Có | `application/json` | Kiểu dữ liệu request. |
| `Accept` | Khuyến nghị | `application/json` | Kiểu dữ liệu response mong muốn. |
| `X-Forwarded-For` | Do hạ tầng cung cấp | `203.0.113.10` | IP người dùng. Production ưu tiên giá trị này do proxy/gateway tin cậy gắn. Không tự giả lập header này từ browser/client. |

**Body request**

| Trường | Kiểu | Bắt buộc | Validation | Mô tả |
|---|---|---|---|---|
| `ip` | string | Có trong dev/test; dự phòng ở production | Địa chỉ IP hợp lệ | IP người dùng. Trong production, giá trị từ `X-Forwarded-For` được ưu tiên; field này dùng làm dự phòng khi header không có. |
| `site_id` | string | Có | Không rỗng; không chứa ký tự `~` | Mã website/app Publisher đã đăng ký. |
| `client_id` | string | Có | Không rỗng; phải là Publisher đang hoạt động | Mã Publisher được TelcoAds cấp. |
| `channel_type` | string | Có | Một trong: `website`, `app` | Kênh phát sinh yêu cầu. |

```json
{
  "ip": "203.0.113.10",
  "site_id": "{{site-id}}",
  "client_id": "{{publisher-client-id}}",
  "channel_type": "website"
}
```

**Response thành công — 200 OK**

| Trường | Kiểu | Mô tả |
|---|---|---|
| `sid` | string | Token định danh tạm thời, dùng cho `POST /v1/match`. |
| `expire_time` | integer | Unix timestamp (giây) tại thời điểm `sid` hết hạn. SDK trừ hao 5 giây so với giá trị này để tự làm mới `sid` trước khi thực sự hết hạn. |
| `error_code` | string | Chuỗi rỗng khi thành công. |
| `error_message` | string | Chuỗi rỗng khi thành công. |

```json
{
  "sid": "{{opaque-sid}}",
  "expire_time": 1787654317,
  "error_code": "",
  "error_message": ""
}
```

**Bảng mã lỗi**

| HTTP | error_code | Nguyên nhân thường gặp | Xử lý |
|---|---|---|---|
| 400 | `INVALID_INPUT` | JSON không hợp lệ · thiếu hoặc rỗng `ip`, `site_id` hoặc `client_id` · `ip` sai định dạng · `site_id` chứa ký tự `~` · `channel_type` không phải `website` hoặc `app` | Sửa request, không retry cùng dữ liệu. |
| 401 | `UNAUTHORIZED` | `client_id` không tồn tại · `client_id` không hoạt động · `client_id` không có quyền Publisher | Kiểm tra credential/môi trường. Liên hệ TelcoAds nếu cần. |
| 404 | `IDENTIFY_FAILED` | Không thể định danh IP tại thời điểm gọi · IP không thuộc phạm vi hỗ trợ · chưa có thông tin nhận diện tương ứng với IP | Tiếp tục luồng quảng cáo không cá nhân hóa. |
| 503 | `SIGNAL_TIMEOUT` | Nguồn dữ liệu tín hiệu tạm thời không phản hồi | Có thể retry với exponential backoff. Nếu vẫn lỗi, dùng luồng không cá nhân hóa. |
| 500 | `INTERNAL_ERROR` | Lỗi hệ thống | Retry giới hạn với exponential backoff. Ghi nhận request ID nếu được cung cấp và liên hệ Viettel khi lỗi kéo dài. |

### API Danh mục Segment — GET /v1/segments

DSP dùng endpoint này để tải danh mục segment phục vụ cấu hình chiến dịch. Nên đồng bộ danh mục định kỳ theo nhu cầu vận hành của bạn; không gọi endpoint này trong đường xử lý quảng cáo thời gian thực.

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
| 401 | `DSP_UNAUTHORIZED` | Thiếu header `Authorization` · header không đúng định dạng Basic Auth · `client_id` không tồn tại hoặc không hoạt động · secret không khớp · credential không có vai trò DSP | Kiểm tra base URL (nếu gọi trực tiếp) và credential, không retry cùng credential. |
| 500 | `INTERNAL_ERROR` | Lỗi hệ thống | Retry giới hạn với exponential backoff; sử dụng bản danh mục đã đồng bộ gần nhất nếu phù hợp. |

### API Đối ứng người dùng — POST /v1/match

Adtech/DSP phía server dùng API này để kiểm tra segment nào trong danh sách mục tiêu khớp với người dùng đang có `sid`. API chỉ trả các `segment_id` nằm trong danh sách gửi lên và giữ nguyên thứ tự của danh sách đó.

Không gọi endpoint này trực tiếp từ browser: đây là luồng server-to-server. Không gửi `sid` vào log, URL, hoặc hệ thống phân tích bên thứ ba.

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
| `sid` | string | Có | Không rỗng; là `sid` TelcoAds đã cấp và còn hiệu lực | Token nhận từ `POST /identify`. |
| `requested_smg_ids` | array\<string\> | Có | Danh sách segment mục tiêu, không rỗng | Các mã segment cần đối ứng. Khuyến nghị chỉ dùng `segment_id` có `is_active: true` từ danh mục. |

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
| 401 | `UNAUTHORIZED` | `sid` không hợp lệ · `sid` đã bị sửa đổi | Không retry, thực hiện lại luồng định danh nếu còn cần thiết. |
| 401 | `SID_NOT_FOUND` | `sid` đã hết hạn | Gọi lại `POST /identify` để lấy `sid` mới, sau đó gọi lại match. |
| 404 | `HASHID_NOT_FOUND` | Không còn dữ liệu phiên tương ứng với `sid` | Coi là không thể đối ứng tại thời điểm đó; có thể định danh lại nếu phù hợp với luồng của bạn. |
| 500 | `INTERNAL_ERROR` | Lỗi hệ thống | Retry giới hạn với exponential backoff; nếu không thành công, dùng luồng không cá nhân hóa. |

## Xử lý lỗi và retry

| Nhóm lỗi | Ví dụ | Có retry? | Hành động khuyến nghị |
|---|---|---|---|
| Lỗi request | `400 INVALID_INPUT` | Không | Sửa dữ liệu hoặc mã tích hợp. |
| Lỗi xác thực/ủy quyền | `401 UNAUTHORIZED`, `401 DSP_UNAUTHORIZED` | Không | Kiểm tra credential, môi trường và quyền tích hợp. |
| Kết quả nghiệp vụ không có dữ liệu | `404 IDENTIFY_FAILED`, `404 HASHID_NOT_FOUND`, `401 SID_NOT_FOUND` | Không cùng request | Rẽ sang quảng cáo không cá nhân hóa; riêng `SID_NOT_FOUND` cần định danh lại để lấy `sid` mới. |
| Lỗi tạm thời | `503 SIGNAL_TIMEOUT`, `500 INTERNAL_ERROR` | Có, giới hạn | Retry exponential backoff, ví dụ tối đa 2–3 lần. Không để retry làm chậm đường quảng cáo quá ngân sách thời gian của bạn. |

Ví dụ backoff: chờ ngẫu nhiên quanh 100 ms, 300 ms, 900 ms; dừng ngay khi còn ít thời gian xử lý quảng cáo. Không retry `POST /identify` như cách thay thế cho một kết quả `IDENTIFY_FAILED` hợp lệ.
