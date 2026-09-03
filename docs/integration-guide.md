# TelcoAds API — Hướng dẫn tích hợp

**Dành cho:** Publisher/SDK, Adtech và DSP tích hợp TelcoAds.
**Phiên bản tài liệu:** 1.0 (bản tích hợp khách hàng) · **Cập nhật:** 22/08/2026

TelcoAds cung cấp ba API cho một luồng quảng cáo đơn giản:

- Cấp mã định danh tạm thời của người dùng/thuê bao cho phiên truy cập.
- Lấy danh mục segment để cấu hình chiến dịch.
- Đối ứng segment của người dùng với danh sách segment mà Adtech quan tâm.

API không trả số thuê bao hay thông tin nhận dạng trực tiếp của người dùng. Số thuê bao được mã hoá và tạo thành `sid`; hãy coi đây là mã tạm thời, một token ngắn hạn và chỉ sử dụng trong luồng cần thiết.

## Mục lục

1. [Bắt đầu nhanh](#1-bắt-đầu-nhanh)
2. [Tổng quan endpoint](#2-tổng-quan-endpoint)
3. [API Định danh — POST /identify](#3-api-định-danh--post-identify)
4. [API Danh mục Segment — GET /v1/segments](#4-api-danh-mục-segment--get-v1segments)
5. [API Đối ứng người dùng — POST /v1/match](#5-api-đối-ứng-người-dùng--post-v1match)
6. [Xử lý lỗi và retry](#6-xử-lý-lỗi-và-retry)

## 1. Bắt đầu nhanh

### 1.1. Tích hợp qua SDK Web/App

TelcoAds cung cấp SDK JavaScript thuần, một file, không phụ thuộc thư viện ngoài, dùng để nhúng trực tiếp vào website hoặc ứng dụng. SDK gọi API Định danh, lưu `sid` vào bộ nhớ trình duyệt và tự làm mới khi hết hạn; khách hàng không cần và không nên tự cấu hình Base URL hoặc môi trường, SDK tự gọi đúng endpoint theo bản phát hành được cấp.

```html
<script src="https://{{sdk-cdn-host}}/telcoads-sdk.min.js"
        integrity="{{sdk-integrity-hash}}"
        crossorigin="anonymous"></script>
```

Gọi `TelcoAds.init()` một lần khi trang tải xong, truyền các định danh TelcoAds đã cấp:

```js
TelcoAds.init({
  siteId: '{{site-id}}',
  clientId: '{{publisher-client-id}}',
  endpoint: '{{TELCOADS_BASE_URL}}/identify',
  channelType: 'website'
});
```

Tham số tùy chọn: `channelType` (mặc định `website`, chỉ nhận `website` hoặc `app`), `timeoutMs` (mặc định `2000`, thời gian chờ tối đa trước khi huỷ request identify), `debug` (mặc định `false`, bật `console.warn` khi có lỗi). `init()` throw ngay nếu thiếu `siteId`/`clientId`/`endpoint` hoặc `channelType` sai giá trị; đây là lỗi cấu hình cần sửa lúc tích hợp, không phải lỗi runtime.

Tại thời điểm chuẩn bị gửi ad request, gọi `TelcoAds.getSid()`:

```js
TelcoAds.getSid().then(function (sid) {
  // sid là string: gắn vào ad request
  // sid là null: gửi ad request không kèm sid
});
```

`getSid()` không bao giờ throw hay reject; mọi lỗi (404, 401, 429, 5xx, timeout, mất mạng) đều resolve `null` để luồng quảng cáo của publisher không bị vỡ. SDK tự cache theo site, gộp nhiều lời gọi `getSid()` đồng thời thành một request, và tạm dừng gọi lại vài giây sau lỗi tạm thời; publisher không cần tự cài retry cho API Định danh.

> Không tự thay đổi endpoint, không chèn `X-Forwarded-For`, và không nhúng credential bí mật DSP vào SDK/browser. Các API DSP và đối ứng server-to-server vẫn phải được gọi từ backend. Tham số `devIp` (nếu SDK hỗ trợ) chỉ dùng lúc dev/test cục bộ; không bật ở production vì sẽ gửi IP giả kèm mọi request định danh.

### 1.2. Gọi API từ backend trực tiếp

Phần này chỉ áp dụng khi Adtech/DSP tích hợp trực tiếp từ backend (ví dụ gọi `GET /v1/segments` hoặc `POST /v1/match`). Điền các giá trị sau theo môi trường được TelcoAds cấp:

| Biến | Ví dụ | Ghi chú |
|---|---|---|
| `TELCOADS_BASE_URL` | `https://{{api-host}}` | Base URL theo môi trường. Không thêm dấu `/` ở cuối. |
| `PUBLISHER_CLIENT_ID` | `{{publisher-client-id}}` | Mã Publisher cho API Định danh. |
| `DSP_CLIENT_ID` | `{{dsp-client-id}}` | Mã DSP cho API Danh mục Segment. |
| `DSP_CLIENT_SECRET` | `{{dsp-client-secret}}` | Bí mật DSP cho API Danh mục Segment. Lưu trong secret manager, không đưa vào mã nguồn. |
| `SITE_ID` | `{{site-id}}` | Mã website/app đã đăng ký với TelcoAds. |

### 1.3. Quy ước chung

- Giao tiếp qua HTTPS; request và response dùng JSON UTF-8, trừ API `GET /v1/segments` không có body request.
- Với POST, gửi header `Content-Type: application/json` và `Accept: application/json`.
- Với tích hợp backend trực tiếp, dùng đúng base URL TelcoAds đã cấp. Token/credential giữa các môi trường không hoán đổi được.
- `sid` có thời hạn ngắn (mặc định 300 giây). Không lưu hoặc tái sử dụng sau khi hết hạn.
- Ở môi trường production, địa chỉ IP thực được nhận từ lớp gateway/proxy tin cậy. Client không nên tự đặt header `X-Forwarded-For`.
- Luôn kiểm tra cả HTTP status và `error_code`. `error_message` nhằm hỗ trợ hiển thị/tra cứu; logic tích hợp nên dựa vào `error_code`.
- Không có rate limit hay cơ chế idempotency được công bố trong phiên bản này. Chỉ retry lỗi tạm thời theo hướng dẫn ở phần [Xử lý lỗi](#6-xử-lý-lỗi-và-retry).

### 1.4. Mẫu response lỗi

Các API trả lỗi theo cùng dạng JSON:

```json
{
  "error_code": "INVALID_INPUT",
  "error_message": "Dữ liệu đầu vào không hợp lệ"
}
```

Nội dung `error_message` có thể được cải thiện mà không báo trước; `error_code` là mã để ứng dụng xử lý.

## 2. Tổng quan endpoint

| API | Ai gọi | Mục đích | Xác thực |
|---|---|---|---|
| `POST /identify` | Publisher/SDK | Cấp `sid` tạm thời cho lượt truy cập | `client_id` Publisher hợp lệ |
| `GET /v1/segments` | DSP | Lấy danh mục segment để cấu hình chiến dịch | HTTP Basic Auth của DSP |
| `POST /v1/match` | Adtech/DSP phía server | Đối ứng `sid` với segment mục tiêu | `sid` còn hợp lệ |

### Phân biệt các vai trò

| Khái niệm | Hiểu đơn giản | Vai trò trong TelcoAds | Ví dụ |
|---|---|---|---|
| **Publisher** | Đơn vị sở hữu điểm tiếp xúc với người dùng. | SDK của Publisher gọi `POST /identify` khi người dùng truy cập website/app để nhận `sid`. Một Publisher được nhận `client_id`. | Trang tin, ứng dụng nội dung hoặc ứng dụng tích hợp SDK. |
| **DSP** (Demand-Side Platform) | Nền tảng phục vụ bên mua quảng cáo. | Gọi `GET /v1/segments` để lấy danh mục segment và dùng các mã segment đó khi cấu hình chiến dịch. DSP dùng credential riêng. | Hệ thống mua quảng cáo tự động của đối tác. |
| **Site** | Một website hoặc ứng dụng cụ thể thuộc Publisher. | Được nhận diện bằng `site_id` trong request identify; giúp TelcoAds biết lượt truy cập thuộc điểm tiếp xúc nào. | Publisher A có thể có `news.example` và app *Example News* là hai Site khác nhau. |

Một Publisher có thể sở hữu nhiều Site. `client_id` xác định Publisher, còn `site_id` xác định website/app cụ thể của Publisher đó.

## 3. API Định danh — POST /identify

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
| `ttl_seconds` | integer | Số giây `sid` còn hiệu lực; mặc định là 300. |
| `error_code` | string | Chuỗi rỗng khi thành công. |
| `error_message` | string | Chuỗi rỗng khi thành công. |

```json
{
  "sid": "{{opaque-sid}}",
  "ttl_seconds": 300,
  "error_code": "",
  "error_message": ""
}
```

**Lỗi có thể gặp**

| HTTP | error_code | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|---|
| 400 | `INVALID_INPUT` | JSON không hợp lệ · thiếu hoặc rỗng `ip`, `site_id` hoặc `client_id` · `ip` sai định dạng · `site_id` chứa ký tự `~` · `channel_type` không phải `website` hoặc `app` | Sửa request; không retry cùng dữ liệu. |
| 401 | `UNAUTHORIZED` | `client_id` không tồn tại · `client_id` không hoạt động · `client_id` không có quyền Publisher | Kiểm tra credential/môi trường; liên hệ TelcoAds nếu cần. |
| 404 | `IDENTIFY_FAILED` | Không thể định danh IP tại thời điểm gọi · IP không thuộc phạm vi hỗ trợ · chưa có thông tin nhận diện tương ứng với IP | Đây là kết quả nghiệp vụ dự kiến; tiếp tục luồng quảng cáo không cá nhân hóa. |
| 503 | `SIGNAL_TIMEOUT` | Nguồn dữ liệu tín hiệu tạm thời không phản hồi | Có thể retry với exponential backoff; nếu vẫn lỗi, dùng luồng không cá nhân hóa. |
| 500 | `INTERNAL_ERROR` | Lỗi hệ thống | Retry giới hạn với exponential backoff; ghi nhận request ID nếu được cung cấp và liên hệ TelcoAds khi lỗi kéo dài. |

## 4. API Danh mục Segment — GET /v1/segments

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

**Lỗi có thể gặp**

| HTTP | error_code | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|---|
| 401 | `DSP_UNAUTHORIZED` | Thiếu header `Authorization` · header không đúng định dạng Basic Auth · `client_id` không tồn tại hoặc không hoạt động · secret không khớp · credential không có vai trò DSP | Kiểm tra base URL (nếu gọi trực tiếp) và credential; không retry cùng credential. |
| 500 | `INTERNAL_ERROR` | Lỗi hệ thống | Retry giới hạn với exponential backoff; sử dụng bản danh mục đã đồng bộ gần nhất nếu phù hợp. |

## 5. API Đối ứng người dùng — POST /v1/match

Adtech/DSP phía server dùng API này để kiểm tra segment nào trong danh sách mục tiêu khớp với người dùng đang có `sid`. API chỉ trả các `segment_id` nằm trong danh sách gửi lên và giữ nguyên thứ tự của danh sách đó.

> Không gọi endpoint này trực tiếp từ browser: đây là luồng server-to-server. Không gửi `sid` vào log, URL, hoặc hệ thống phân tích bên thứ ba.

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

**Lỗi có thể gặp**

| HTTP | error_code | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|---|
| 400 | `INVALID_INPUT` | JSON không hợp lệ · thiếu hoặc rỗng `sid` · thiếu hoặc rỗng `requested_smg_ids` | Sửa request; không retry cùng dữ liệu. |
| 401 | `UNAUTHORIZED` | `sid` không hợp lệ · `sid` đã bị sửa đổi | Không retry; thực hiện lại luồng định danh nếu còn cần thiết. |
| 401 | `SID_NOT_FOUND` | `sid` đã hết hạn | Gọi lại `POST /identify` để lấy `sid` mới, sau đó gọi lại match. |
| 404 | `HASHID_NOT_FOUND` | Không còn dữ liệu phiên tương ứng với `sid` | Coi là không thể đối ứng tại thời điểm đó; có thể định danh lại nếu phù hợp với luồng của bạn. |
| 500 | `INTERNAL_ERROR` | Lỗi hệ thống | Retry giới hạn với exponential backoff; nếu không thành công, dùng luồng không cá nhân hóa. |

## 6. Xử lý lỗi và retry

| Nhóm lỗi | Ví dụ | Có retry? | Hành động khuyến nghị |
|---|---|---|---|
| Lỗi request | `400 INVALID_INPUT` | Không | Sửa dữ liệu hoặc mã tích hợp. |
| Lỗi xác thực/ủy quyền | `401 UNAUTHORIZED`, `401 DSP_UNAUTHORIZED` | Không | Kiểm tra credential, môi trường và quyền tích hợp. |
| Kết quả nghiệp vụ không có dữ liệu | `404 IDENTIFY_FAILED`, `404 HASHID_NOT_FOUND`, `401 SID_NOT_FOUND` | Không cùng request | Rẽ sang quảng cáo không cá nhân hóa; riêng `SID_NOT_FOUND` cần định danh lại để lấy `sid` mới. |
| Lỗi tạm thời | `503 SIGNAL_TIMEOUT`, `500 INTERNAL_ERROR` | Có, giới hạn | Retry exponential backoff, ví dụ tối đa 2–3 lần. Không để retry làm chậm đường quảng cáo quá ngân sách thời gian của bạn. |

Ví dụ backoff: chờ ngẫu nhiên quanh 100 ms, 300 ms, 900 ms; dừng ngay khi còn ít thời gian xử lý quảng cáo. Không retry `POST /identify` như cách thay thế cho một kết quả `IDENTIFY_FAILED` hợp lệ.

---

Tài liệu này mô tả hợp đồng tích hợp công khai. Các cơ chế xử lý, nguồn dữ liệu và lưu trữ nội bộ không nằm trong phạm vi tài liệu.
