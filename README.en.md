# TelcoAds Web SDK

[Tiếng Việt](README.md) · **English**

The Web SDK runs on a Publisher's website to obtain a `sid` when a user visits and pass it, with the ad request, to the Adtech system. The SDK is a single vanilla JS file (~4.2 KB) with no dependencies and no build step.

**Current version: v2.0.1** — identification through the carrier authentication gateway (`GET /ident`). SDK calls are unchanged from v1 (`TelcoAds.init()` then `TelcoAds.getSid()`); note the HTTPS, domain registration and CSP requirements below.

> Full integration guide (Web SDK, backend APIs for Adtech/DSP: `/ident`, `/v1/segments`, `/v1/match`, error codes, retries…): **[docs/integration-guide.en.md](docs/integration-guide.en.md)**.

## Coverage

Only Viettel mobile subscribers browsing over the Viettel mobile data network in Vietnam can be identified. For all other visits (Wi-Fi, other mobile networks, networks outside Vietnam) `getSid()` returns `null`, which is expected.

## Installing the SDK

Load from jsDelivr, pinned to a release tag (never `@main`), keeping the `integrity` attribute:

```html
<script src="https://cdn.jsdelivr.net/gh/BigDataServices-ViettelTelecom/telcoads-webapp-sdk@v2.0.1/dist/telcoads-sdk-v2.min.js"
        integrity="sha384-zyIFGucuXHVBakv1SMNADojUXbENS8WxfVKkyKt5qzLI2IYxhdMJyZx8qSK0Ldmj"
        crossorigin="anonymous"></script>
```

To self-host: download `dist/telcoads-sdk-v2.min.js` at tag `v2.0.1` and keep the same `integrity` value.

## Embedding and initialization

```js
TelcoAds.init({
  siteId:   '{{site-id}}',              // website ID registered with TelcoAds
  clientId: '{{publisher-client-id}}',  // Publisher ID issued by TelcoAds
  timeoutMs: 10000                      // wait up to 10 s for /ident (pilot phase)
});

// when sending an ad request:
TelcoAds.getSid().then(function (sid) {
  // sid is a string -> attach it to the ad request sent to the Adtech
  // sid is null     -> send the ad request WITHOUT a sid (non-personalized ads)
});
```

- `getSid()` never throws or rejects — every error resolves to `null`, so the ad flow is never interrupted.
- `init()` throws a `TypeError` immediately on invalid configuration (missing `siteId`/`clientId`, `siteId` containing `~`, empty `identUrl`). This is an integration error to catch during development; if configuration comes from a dynamic source, wrap `init()` in `try/catch`.

See the example in [`examples/basic-website`](examples/basic-website) (replace `siteId` and `clientId` with yours).

## `init()` options

| Option | Required | Default | Description |
|---|---|---|---|
| `siteId` | Yes | — | Website ID; a `sid` is issued per site. Must not contain `~`. |
| `clientId` | Yes | — | Publisher ID issued by TelcoAds. Invalid or inactive returns `401`. |
| `identUrl` | No | Production | Full URL of `/ident`. Only set it when TelcoAds provides another environment (e.g. UAT). |
| `authTimeoutMs` | No | `8000` | Maximum time to wait for the authentication result. |
| `timeoutMs` | No | `2000` | Maximum time to wait for the `/ident` request; on timeout returns `null`. Use `10000` during the pilot phase. Does not include `authTimeoutMs`: in the worst case `getSid()` takes about `timeoutMs` + `authTimeoutMs` (pilot: 18 seconds). |
| `negativeTtlS` | No | `60` | Seconds to remember a "not identified" result before retrying. |
| `debug` | No | `false` | Print console warnings on errors. |

## SDK behavior

- The `sid` and its expiry are stored in localStorage under `telcoads:sid`; the SDK subtracts a 5-second margin and re-identifies when the `sid` expires or the record belongs to another site.
- Multiple ad slots calling `getSid()` together share one identification session (single-flight).
- When a result is already available, `/ident` may return the `sid` immediately. Otherwise the SDK opens the authentication gateway in an iframe (never a pop-up), receives the result via `postMessage`, and falls back to polling `/ident/result`.
- Not identified (Wi-Fi, outside the Viettel network) or invalid `clientId` (401): returns `null` and remembers the result for `negativeTtlS` seconds.
- Temporary errors (429, 5xx, timeout, network error): returns `null` and pauses calls for 10 seconds.
- localStorage blocked: the SDK falls back to in-memory storage for the page session.

## Integration notes

- Pages embedding the SDK must use HTTPS. Send TelcoAds your list of domains (including `www`, subdomains, staging) to be registered and receive your `siteId`/`clientId`.
- Call `getSid()` after the page has a `<body>`. If your website uses Content-Security-Policy: `script-src https://cdn.jsdelivr.net`, `connect-src https://affapi.viettel.vn`, `frame-src https://affapi.viettel.vn https://developers-private.viettel.vn`.
- SDK v1 (`POST /identify`) has been retired. Remove v1 from the page before embedding v2.
- Never embed `DSP_CLIENT_SECRET` or any DSP credential in a web page. Backend APIs (`/v1/segments`, `/v1/match`) are always called from your server.
- A `sid` expires after 300 seconds (default). Treat `sid = null` as normal: still send the ad request, just without segment data.
- SDK console messages are in Vietnamese; see [SDK console messages](docs/integration-guide.en.md#sdk-console-messages) for their meaning. API `error_message` values are also in Vietnamese; always act on `error_code`.

## Upgrading from v1 (required)

1. Change the `<script>` tag to `dist/telcoads-sdk-v2.min.js` at tag `v2.0.1`, with the new `integrity` value.
2. Remove `endpoint`, `channelType`, `devIp` from `init()` (not used in v2).
3. Add the CSP entries above (if you use CSP).

Details: [Upgrading from v1](docs/integration-guide.en.md#upgrading-from-v1). SDK v1 has been retired; partners still on v1 must upgrade.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) (Vietnamese).

## Support

Contact the TelcoAds integration contact assigned to your organization. When reporting an issue, include the `request_id` from the TelcoAds response.

## License

See [LICENSE](LICENSE).
