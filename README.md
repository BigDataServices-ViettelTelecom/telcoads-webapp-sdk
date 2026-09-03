# TelcoAds Web SDK

SDK JavaScript thuần (vanilla JS), một file, không phụ thuộc thư viện ngoài, dùng để nhúng vào website/app nhằm lấy `sid` (mã định danh tạm thời của phiên truy cập) và gắn vào ad request gửi sang hệ thống Adtech/DSP.

> Tài liệu tích hợp đầy đủ (API backend cho Adtech/DSP: `/identify`, `/v1/segments`, `/v1/match`, mã lỗi, retry...) xem tại **[docs/integration-guide.md](docs/integration-guide.md)**.

## Cài đặt

Endpoint SDK chính thức (`{{sdk-cdn-host}}`) do TelcoAds cấp riêng theo môi trường/hợp đồng tích hợp — dùng đúng giá trị TelcoAds cung cấp cho bạn, xem [docs/integration-guide.md](docs/integration-guide.md#11-tích-hợp-qua-sdk-webapp).

Nếu chỉ cần link nhanh để tham khảo/thử nghiệm, có thể lấy trực tiếp file trong repo này qua jsDelivr (ghim theo tag phát hành, không dùng `@main` để tránh nhận thay đổi ngoài ý muốn):

```html
<script src="https://cdn.jsdelivr.net/gh/BigDataServices-ViettelTelecom/telcoads-webapp-sdk@v1.0.0/dist/telcoads-sdk.min.js"
        crossorigin="anonymous"></script>
```

Xem các bản phát hành tại [Releases](../../releases).

## Sử dụng

```js
TelcoAds.init({
  siteId:   '{{site-id}}',              // mã website/app do TelcoAds cấp
  clientId: '{{publisher-client-id}}',  // mã Publisher do TelcoAds cấp
  endpoint: '{{TELCOADS_BASE_URL}}/identify' // do TelcoAds cấp theo môi trường
});

// tại thời điểm chuẩn bị gửi ad request:
TelcoAds.getSid().then(function (sid) {
  // sid là string  -> gắn vào ad request gửi Adtech
  // sid là null    -> gửi ad request KHÔNG kèm sid (quảng cáo không cá nhân hoá)
});
```

`getSid()` không bao giờ throw/reject — mọi lỗi đều resolve `null` để luồng quảng cáo không bị vỡ. `init()` throw ngay khi cấu hình sai (thiếu `siteId`/`clientId`/`endpoint`), đây là lỗi tích hợp cần sửa lúc phát triển.

Xem ví dụ chạy được tại [`examples/basic-website`](examples/basic-website).

## Tham số `init()`

| Tham số | Bắt buộc | Mặc định | Mô tả |
|---|---|---|---|
| `siteId` | Có | — | Mã website/app; `sid` được cấp theo từng site. Không chứa ký tự `~`. |
| `clientId` | Có | — | Mã Publisher do TelcoAds cấp. Sai hoặc chưa kích hoạt sẽ nhận `401`. |
| `endpoint` | Có | — | URL đầy đủ tới `/identify` theo môi trường TelcoAds cấp. |
| `channelType` | Không | `website` | Chỉ nhận `website` hoặc `app`. |
| `timeoutMs` | Không | `2000` | Thời gian chờ tối đa của request identify. |
| `devIp` | Không | — | Chỉ dùng dev/test cục bộ để gửi IP giả kèm request identify. **Không bật ở production.** |
| `debug` | Không | `false` | Bật cảnh báo console khi có lỗi. |

## Lưu ý khi tích hợp

- Không tự đổi `endpoint`, không tự thêm header `X-Forwarded-For` — địa chỉ IP dùng để định danh do hệ thống TelcoAds xác định từ kết nối thực tế.
- Không nhúng `DSP_CLIENT_SECRET` hay bất kỳ credential DSP nào vào trang web/ứng dụng. Các API dành cho DSP (`/v1/segments`, `/v1/match`) luôn gọi từ backend, xem [docs/integration-guide.md](docs/integration-guide.md).
- `sid` có thời hạn ngắn (mặc định 300 giây), không lưu/tái sử dụng sau khi hết hạn.
- Coi `sid = null` là trạng thái bình thường (người dùng ngoài mạng, dùng Wi-Fi…): vẫn gửi ad request, chỉ là không có dữ liệu segment.

## Changelog

Xem [CHANGELOG.md](CHANGELOG.md).

## Hỗ trợ

Liên hệ đầu mối tích hợp TelcoAds được cấp cho đối tác của bạn.

## Giấy phép

Xem [LICENSE](LICENSE).
