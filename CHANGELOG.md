# Changelog

Định dạng dựa theo [Keep a Changelog](https://keepachangelog.com/), versioning theo [SemVer](https://semver.org/).

## [Unreleased]

### Fixed

- Đối chiếu tài liệu với `dist/telcoads-sdk.min.js` thực tế và sửa lại cho khớp:
  - Response `POST /identify` trả `expire_time` (Unix timestamp), không phải `ttl_seconds`.
  - `negativeTtlS` và `devIp` đều là tham số `init()` có thật, không phải cái này thay cái kia.
  - Khoá localStorage đúng là `telcoads:sid` (chữ thường).
- README và `docs/integration-guide.md` viết lại rõ ràng, nhất quán hơn.

## [1.0.0] - 2026-08-22

### Added

- Phát hành đầu tiên `telcoads-sdk.min.js`: `TelcoAds.init()` và `TelcoAds.getSid()`.
- Tài liệu tích hợp API backend (`/identify`, `/v1/segments`, `/v1/match`) tại `docs/integration-guide.md`.
- Ví dụ tích hợp cơ bản tại `examples/basic-website`.
