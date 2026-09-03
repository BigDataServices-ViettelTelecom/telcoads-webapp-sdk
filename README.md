# TelcoAds Web SDK

Web SDK được tích hợp trên website của Publisher để lấy `sid` tại thời điểm người dùng truy cập và chuyển `sid` cùng ad request sang hệ thống Adtech. SDK JavaScript của TelcoAds là 1 file vanilla JS, không dependency, không cần build. SDK chỉ gọi `POST /identify` để lấy `sid` cho lượt truy cập; `POST /v1/match` là luồng server-to-server của Adtech, SDK không gọi.

> Tài liệu tích hợp đầy đủ (API backend cho Adtech/DSP: `/identify`, `/v1/segments`, `/v1/match`, mã lỗi, retry...) xem tại **[docs/integration-guide.md](docs/integration-guide.md)**.

## Cài đặt SDK

Đối tác không cần và không nên cấu hình Base URL hoặc môi trường. SDK tự gọi đúng endpoint theo cấu hình của bản phát hành. Endpoint chính thức (`{{sdk-cdn-host}}`) do TelcoAds cấp riêng theo môi trường/hợp đồng tích hợp (ví dụ bản UAT hoặc Production) — dùng đúng giá trị TelcoAds cung cấp cho bạn.

Nếu chỉ cần link nhanh để tham khảo/thử nghiệm, có thể lấy trực tiếp file trong repo này qua jsDelivr (ghim theo tag phát hành, không dùng `@main` để tránh nhận thay đổi ngoài ý muốn):

```html
<script src="https://cdn.jsdelivr.net/gh/BigDataServices-ViettelTelecom/telcoads-webapp-sdk@v1.0.0/dist/telcoads-sdk.min.js"
        crossorigin="anonymous"></script>
```

Xem các bản phát hành tại [Releases](../../releases).

## Nhúng và khởi tạo

```js
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
```

`getSid()` không bao giờ throw/reject: mọi lỗi đều trả về `null` để luồng quảng cáo của Publisher không bị vỡ. Ngược lại `init()` throw ngay khi cấu hình sai — đây là lỗi tích hợp, cần lộ ra lúc phát triển.

Xem ví dụ chạy được tại [`examples/basic-website`](examples/basic-website).

## Tham số `init()`

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

## Hành vi SDK

- `sid` và thời hạn được lưu ở localStorage khoá `telcoads:sid`; SDK tự trừ hao 5 giây trước hạn và tự gọi lại identify khi hết hạn hoặc khi bản ghi thuộc site khác.
- Nhiều ad slot cùng gọi `getSid()` chỉ sinh 1 request identify (single-flight).
- Không định danh được (404) hoặc sai `client_id` (401): trả `null` và nhớ kết quả trong `negativeTtlS` giây.
- Lỗi tạm (429, 5xx, timeout, lỗi mạng): trả `null` và tạm dừng gọi lại 10 giây; không ghi vào localStorage.
- Trình duyệt chặn localStorage (chế độ riêng tư, hết quota): SDK tự chuyển sang bộ nhớ tạm trong phiên trang.

## Lưu ý khi tích hợp

- Không tự đổi endpoint, không tự thêm header `X-Forwarded-For`: địa chỉ IP dùng để định danh do hệ thống TelcoAds xác định từ kết nối thực tế.
- Không nhúng `DSP_CLIENT_SECRET` hoặc bất kỳ credential DSP nào vào trang web/ứng dụng. Các API dành cho DSP (`/v1/segments`, `/v1/match`) luôn gọi từ backend, xem [docs/integration-guide.md](docs/integration-guide.md).
- `sid` sau khi hết hạn (mặc định 300 giây) sẽ không thể dùng để match segment.
- Coi `sid = null` là trạng thái bình thường (người dùng ngoài mạng Viettel, dùng Wi-Fi…): vẫn gửi ad request, chỉ là không có dữ liệu segment.

## Changelog

Xem [CHANGELOG.md](CHANGELOG.md).

## Hỗ trợ

Liên hệ đầu mối tích hợp TelcoAds được cấp cho đối tác của bạn.

## Giấy phép

Xem [LICENSE](LICENSE).
