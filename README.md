# TelcoAds Web SDK

Web SDK được tích hợp trên website của Publisher để lấy `sid` tại thời điểm người dùng truy cập và chuyển `sid` cùng ad request sang hệ thống Adtech. SDK là 1 file vanilla JS (~4,7 KB), không dependency, không cần build.

**Phiên bản hiện tại: v2.0.1** — định danh qua cổng xác thực của nhà mạng (`GET /ident`). Mã gọi SDK không đổi so với v1 (`TelcoAds.init()` rồi `TelcoAds.getSid()`); cần lưu ý thêm HTTPS, đăng ký domain và CSP (xem bên dưới).

> Tài liệu tích hợp đầy đủ (Web SDK, API backend cho Adtech/DSP: `/ident`, `/v1/segments`, `/v1/match`, mã lỗi, retry…) xem tại **[docs/integration-guide.md](docs/integration-guide.md)**.

## Cài đặt SDK

Nhúng qua jsDelivr, ghim theo tag phát hành (không dùng `@main`) và giữ thuộc tính `integrity`:

```html
<script src="https://cdn.jsdelivr.net/gh/BigDataServices-ViettelTelecom/telcoads-webapp-sdk@v2.0.1/dist/telcoads-sdk-v2.min.js"
        integrity="sha384-zyIFGucuXHVBakv1SMNADojUXbENS8WxfVKkyKt5qzLI2IYxhdMJyZx8qSK0Ldmj"
        crossorigin="anonymous"></script>
```

Muốn tự host: tải `dist/telcoads-sdk-v2.min.js` tại tag `v2.0.1`, giữ nguyên `integrity` ở trên.

## Nhúng và khởi tạo

```js
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
```

- `getSid()` không bao giờ throw hay reject — mọi lỗi đều trả về `null`, để luồng quảng cáo không bị gián đoạn.
- `init()` throw `TypeError` ngay nếu cấu hình sai (thiếu `siteId`/`clientId`, `siteId` chứa `~`, `identUrl` rỗng). Đây là lỗi tích hợp, cần phát hiện khi phát triển; nếu cấu hình lấy từ nguồn động, nên bọc `init()` trong `try/catch`.


Ví dụ tại [`examples/basic-website`](examples/basic-website) (thay `siteId`, `clientId` của bạn).

## Tham số `init()`

| Tham số | Bắt buộc | Mặc định | Mô tả |
|---|---|---|---|
| `siteId` | Có | — | Mã website; `sid` được cấp theo từng site. Không chứa ký tự `~`. |
| `clientId` | Có | — | Mã Publisher do TelcoAds cấp. Sai hoặc chưa kích hoạt sẽ nhận `401`. |
| `identUrl` | Không | Production | URL đầy đủ tới `/ident`. Chỉ truyền khi TelcoAds cấp môi trường khác (ví dụ UAT). |
| `authTimeoutMs` | Không | `8000` | Thời gian chờ tối đa kết quả xác thực. |
| `timeoutMs` | Không | `2000` | Thời gian chờ tối đa của request `/ident`; quá thời gian trả `null`. Giai đoạn pilot dùng `10000`. |
| `negativeTtlS` | Không | `60` | Số giây nhớ kết quả "không định danh được" trước khi thử lại. |
| `debug` | Không | `false` | Bật cảnh báo console khi có lỗi. |

## Hành vi SDK

- `sid` và thời hạn được lưu ở localStorage khoá `telcoads:sid`; SDK tự trừ hao 5 giây trước hạn và tự định danh lại khi hết hạn hoặc khi bản ghi thuộc site khác.
- Nhiều ad slot cùng gọi `getSid()` chỉ sinh 1 phiên định danh (single-flight).
- Khi đã có sẵn kết quả, `/ident` có thể trả `sid` ngay. Trường hợp còn lại: SDK mở cổng xác thực qua iframe (không mở cửa sổ bật lên), nhận kết quả qua `postMessage`, dự phòng hỏi lại `/ident/result`.
- Không định danh được (Wi-Fi, ngoài mạng Viettel) hoặc sai `clientId` (401): trả `null` và nhớ kết quả trong `negativeTtlS` giây.
- Lỗi tạm (429, 5xx, timeout, lỗi mạng): trả `null` và tạm dừng gọi lại 10 giây.
- Trình duyệt chặn localStorage: SDK tự chuyển sang bộ nhớ tạm trong phiên trang.

## Lưu ý khi tích hợp

- Trang nhúng SDK phải chạy HTTPS. Gửi TelcoAds danh sách domain (gồm `www`, subdomain, staging) để đăng ký và nhận `siteId`/`clientId`.
- Gọi `getSid()` sau khi trang đã có `<body>`. Nếu website dùng Content-Security-Policy: `script-src https://cdn.jsdelivr.net`, `connect-src https://affapi.viettel.vn`, `frame-src https://affapi.viettel.vn https://developers-private.viettel.vn`.
- SDK v1 (`POST /identify`) đã ngừng hoạt động. Gỡ bản v1 khỏi trang trước khi nhúng v2.
- Không nhúng `DSP_CLIENT_SECRET` hoặc credential DSP nào vào trang web. Các API backend (`/v1/segments`, `/v1/match`) luôn gọi từ server.
- `sid` hết hạn sau 300 giây (mặc định). Coi `sid = null` là trạng thái bình thường: vẫn gửi ad request, chỉ là không có dữ liệu segment.

## Nâng cấp từ v1 (bắt buộc)

1. Đổi thẻ `<script>` sang `dist/telcoads-sdk-v2.min.js` tại tag `v2.0.1`, kèm `integrity` mới.
2. Bỏ `endpoint`, `channelType`, `devIp` khỏi `init()` (v2 không dùng).
3. Bổ sung CSP (nếu có) như mục Lưu ý ở trên.

Chi tiết: [Nâng cấp từ v1](docs/integration-guide.md#nâng-cấp-từ-v1). SDK v1 đã ngừng hoạt động, đối tác đang dùng v1 cần nâng cấp.

## Changelog

Xem [CHANGELOG.md](CHANGELOG.md).

## Hỗ trợ

Liên hệ đầu mối tích hợp TelcoAds được cấp cho đối tác của bạn. Khi báo lỗi, gửi kèm `request_id` trong response của TelcoAds.

## Giấy phép

Xem [LICENSE](LICENSE).
