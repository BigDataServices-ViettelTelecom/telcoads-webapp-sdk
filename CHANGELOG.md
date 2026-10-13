# Changelog

Định dạng dựa theo [Keep a Changelog](https://keepachangelog.com/), versioning theo [SemVer](https://semver.org/).

## [2.0.0] - 2026-10-13

### Added

- `dist/telcoads-sdk-v2.min.js`: định danh qua cổng xác thực của nhà mạng (`GET /ident`, dự phòng `GET /ident/result`).
- Tham số `init()` mới: `identUrl`, `popup`, `onAuthUrl`, `authTimeoutMs`.
- Hướng dẫn nâng cấp từ v1 và bảng tự kiểm tra trong `docs/integration-guide.md`.

### Changed

- `docs/integration-guide.md` cập nhật cho SDK v2.
- SDK v1 (`POST /identify`) ngừng hoạt động; đối tác đang dùng v1 cần nâng cấp.
- `examples/basic-website` chuyển sang SDK v2, ghim tag `v2.0.0` kèm `integrity`.

### Removed

- `dist/telcoads-sdk.min.js` (SDK v1).
- Tham số `endpoint`, `channelType`, `devIp` (chỉ có ở v1).

### Không đổi

- `TelcoAds.getSid()`, khoá localStorage `telcoads:sid`.

## [1.0.0] - 2026-08-22

### Added

- Phát hành đầu tiên `telcoads-sdk.min.js`: `TelcoAds.init()` và `TelcoAds.getSid()`.
- Tài liệu tích hợp API backend (`/identify`, `/v1/segments`, `/v1/match`) tại `docs/integration-guide.md`.
- Ví dụ tích hợp cơ bản tại `examples/basic-website`.
