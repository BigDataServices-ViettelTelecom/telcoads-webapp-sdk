# TelcoAds — Integration Guide

**Version:** 2.2 (Web SDK v2.0.1) · **Updated:** 2026-11-11 · [Tiếng Việt](integration-guide.md)

This guide explains how partners integrate TelcoAds into their advertising stack: the Web SDK on websites and the backend APIs. SDK v1 (`POST /identify`) has been retired; partners still on v1 must upgrade as described in [Upgrading from v1](#upgrading-from-v1).

## Contents

- [Overview](#overview)
  - [Ad targeting flow](#ad-targeting-flow)
  - [Roles](#roles)
  - [Coverage](#coverage)
  - [What's new in v2](#whats-new-in-v2)
- [Website integration with the Web SDK](#website-integration-with-the-web-sdk)
  - [Installing the SDK](#installing-the-sdk)
  - [Embedding and initialization](#embedding-and-initialization)
  - [`init()` options](#init-options)
  - [SDK behavior](#sdk-behavior)
  - [Integration notes](#integration-notes)
  - [Upgrading from v1](#upgrading-from-v1)
  - [Post-integration checklist](#post-integration-checklist)
  - [SDK console messages](#sdk-console-messages)
- [Backend integration with the TelcoAds API](#backend-integration-with-the-telcoads-api)
- [API reference](#api-reference)
  - [Identification API — GET /ident](#identification-api--get-ident)
  - [Fallback result — GET /ident/result](#fallback-result--get-identresult)
  - [Identification API error codes](#identification-api-error-codes)
  - [Segment Catalog API — GET /v1/segments](#segment-catalog-api--get-v1segments)
  - [User Matching API — POST /v1/match](#user-matching-api--post-v1match)
- [Error handling and retries](#error-handling-and-retries)

## Overview

### Ad targeting flow

TelcoAds provides 3 APIs for the ad targeting flow (`GET /ident/result` is an additional fallback API called by the SDK itself):

- **Identification API — GET /ident** (called by the SDK): issues a temporary identifier (`sid`) for the user/subscriber for the current visit.
- **Segment Catalog API — GET /v1/segments** (called from your backend): returns the segment catalog for campaign setup (outside the real-time path).
- **User Matching API — POST /v1/match** (called from your backend): matches the user's segments against the list of segments the Adtech is interested in.

```
User's browser                       TelcoAds                    Adtech / DSP (server)
──────────────                       ────────                    ─────────────────────
TelcoAds.getSid() ── GET /ident ──►  identification via the
                  ◄── sid ────────   carrier authentication
                                     gateway
ad request + sid ─────────────────────────────────────────────►  POST /v1/match
                                                                 ◄── matched_smg_ids
```

The APIs never return the subscriber's phone number or any directly identifying information. The phone number is encrypted into the `sid`, a temporary session identifier that may only be used within the ad targeting flow.

### Roles

| Term | Definition | Role in TelcoAds | Example |
|---|---|---|---|
| **Publisher** | Owner of the user touchpoint. | Embeds the Web SDK; the SDK calls `GET /ident` when a user visits to obtain a `sid`. Each Publisher receives a `client_id`. | News sites, content websites. |
| **DSP** | Platform serving the advertising buy side. | Calls `GET /v1/segments` to get the segment catalog and `POST /v1/match` for matching. A DSP uses its own credentials. | A partner's programmatic buying system. |
| **Site** | A specific website owned by a Publisher. | Identified by `site_id`; a `sid` is issued per site. | Publisher A owns `news.example` and `sport.example`: two different Sites. |

### Coverage

Identification works only for **Viettel mobile subscribers browsing over the Viettel mobile data network in Vietnam**. For every other visit (Wi-Fi, other mobile networks, networks outside Vietnam), `getSid()` returns `null`. This is expected behavior, not an error: send the ad request without a `sid`.

### What's new in v2

- SDK v2 identifies subscribers through the **carrier authentication gateway** (`GET /ident`) instead of the IP-address lookup (`POST /identify`) used by v1.
- **Publisher-side SDK calls are unchanged:** still `TelcoAds.init()` followed by `TelcoAds.getSid()`, which returns a `sid` or `null`. Note the additional HTTPS, domain registration and CSP requirements.
- No endpoint configuration needed: the SDK uses the Production environment by default.
- For parameter changes see [Upgrading from v1](#upgrading-from-v1).

## Website integration with the Web SDK

The TelcoAds JavaScript SDK is a single vanilla JS file (~4.2 KB) with no dependencies and no build step. It calls `GET /ident`, opens the authentication gateway in an iframe when needed (never in a pop-up window), and returns the `sid` to the page. `POST /v1/match` is a server-to-server call made by the Adtech; the SDK does not call it.

### Installing the SDK

Load `telcoads-sdk-v2.min.js` from release `v2.0.1` of this repository via jsDelivr. Always pin a release tag (never `@main`) and keep the `integrity` attribute so the browser refuses to run the file if its content has been altered:

```html
<script src="https://cdn.jsdelivr.net/gh/BigDataServices-ViettelTelecom/telcoads-webapp-sdk@v2.0.1/dist/telcoads-sdk-v2.min.js"
        integrity="sha384-zyIFGucuXHVBakv1SMNADojUXbENS8WxfVKkyKt5qzLI2IYxhdMJyZx8qSK0Ldmj"
        crossorigin="anonymous"></script>
```

To self-host, download `dist/telcoads-sdk-v2.min.js` at tag `v2.0.1` and keep the same `integrity` value.

### Embedding and initialization

```html
<script src="https://cdn.jsdelivr.net/gh/BigDataServices-ViettelTelecom/telcoads-webapp-sdk@v2.0.1/dist/telcoads-sdk-v2.min.js"
        integrity="sha384-zyIFGucuXHVBakv1SMNADojUXbENS8WxfVKkyKt5qzLI2IYxhdMJyZx8qSK0Ldmj"
        crossorigin="anonymous"></script>
<script>
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
</script>
```

- `getSid()` **never throws or rejects**: every error resolves to `null`, so the Publisher's ad flow is never interrupted.
- `init()` **throws a `TypeError` immediately on invalid configuration** (missing `siteId`/`clientId`, `siteId` containing `~`, empty `identUrl`). These are integration errors meant to surface during development. If configuration values come from a dynamic source (environment variables, CMS…), wrap `init()` in `try/catch`.

How the `sid` is attached to the ad request follows the specification of your Adtech (field/key name defined by the Adtech); TelcoAds does not prescribe it.

### `init()` options

Only `siteId` and `clientId` are required; all other options have defaults suitable for Production.

| Option | Required | Default | Description |
|---|---|---|---|
| `siteId` | Yes | — | Website ID; a `sid` is issued per site. Must not contain `~`. |
| `clientId` | Yes | — | Publisher ID issued by TelcoAds. An invalid or inactive value returns `401`. |
| `identUrl` | No | Production | Full URL of `/ident` (e.g. `https://<host>/…/ident`). Only set it when TelcoAds provides another environment (e.g. UAT). The SDK derives `/ident/result` from this value. |
| `authTimeoutMs` | No | `8000` | Maximum time to wait for the authentication result from the iframe. |
| `timeoutMs` | No | `2000` | Maximum time to wait for the `/ident` request; on timeout the SDK returns `null`. Use `10000` during the pilot phase to avoid losing identifications when the server is slow. Does not include the iframe authentication time (`authTimeoutMs`); see [Integration notes](#integration-notes). |
| `negativeTtlS` | No | `60` | Number of seconds the SDK remembers a "not identified" result before retrying. |
| `debug` | No | `false` | Print `console.warn` messages on errors. |

### SDK behavior

- The `sid` and its expiry are stored in localStorage under the key `telcoads:sid`. The SDK subtracts a 5-second safety margin and re-identifies automatically when the `sid` expires or the stored record belongs to a different site.
- Multiple ad slots calling `getSid()` at the same time share a single identification session (single-flight).
- When a result is already available, `/ident` may return the `sid` immediately without going through the authentication gateway.
- Otherwise the SDK opens the authentication gateway in an iframe (never a pop-up window) and receives the result via `postMessage`. The SDK only accepts messages from the TelcoAds server that carry the matching session code. If no message arrives, the SDK polls `/ident/result` every 1.5 seconds.
- Not identified (`no_match`: Wi-Fi, outside the Viettel network) or invalid `clientId` (401): returns `null` and remembers the result for `negativeTtlS` seconds. An invalid `clientId` always prints a console warning, even with `debug: false`.
- Temporary errors (429, 5xx, timeout, network error, expired session): returns `null` and pauses further calls for 10 seconds; nothing is written to localStorage.
- localStorage blocked by the browser (private mode, quota exceeded): the SDK falls back to in-memory storage for the lifetime of the page.

### Integration notes

- **HTTPS:** pages embedding the SDK must be served over HTTPS. TelcoAds rejects (400) identification requests from `http://` pages (during the pilot phase `http://localhost` is accepted for development).
- **Domains:** send TelcoAds the list of domains that will embed the SDK (including `www`, subdomains and staging, one origin per line) to receive your `siteId`/`clientId` and get registered before integrating; TelcoAds confirms the `siteId` for each origin.
- **When to call:** call `getSid()` after the page has a `<body>` element (the SDK may need to attach an iframe).
- **Waiting time:** in the worst case `getSid()` resolves after about `timeoutMs` + `authTimeoutMs` (with the pilot configuration: 10 + 8 = 18 seconds); while the `sid` is still valid, `getSid()` resolves immediately. Do not block page rendering on `getSid()`; if your ad request has a time budget, send it without a `sid` when the budget runs out.
- **Content-Security-Policy:** if your website uses CSP, add `script-src https://cdn.jsdelivr.net` (when loading from jsDelivr); `connect-src https://affapi.viettel.vn`; `frame-src https://affapi.viettel.vn https://developers-private.viettel.vn`.
- SDK v1 (`POST /identify`) has been retired. Remove v1 from the page before embedding v2 (both versions use the same global `TelcoAds` variable).
- Never embed `DSP_CLIENT_SECRET` or any DSP credential in a web page. DSP APIs must always be called from your backend.
- A `sid` is short-lived (300 seconds by default); once expired it can no longer be used for segment matching.
- Treat `sid = null` as a normal state (Wi-Fi, outside the Viettel network…): still send the ad request, just without segment data.

### Upgrading from v1

SDK v1 has been retired: `POST /identify` is no longer served, and partners still on v1 must move to v2.

| Item | v1.0.0 | v2.0.1 |
|---|---|---|
| File | `dist/telcoads-sdk.min.js` | `dist/telcoads-sdk-v2.min.js` |
| API called by the SDK | `POST /identify` | `GET /ident` (+ `GET /ident/result` fallback) |
| `endpoint` | Required | **Removed** — replaced by `identUrl` (optional, defaults to Production) |
| `channelType` | Yes | **Removed** |
| `devIp` | Yes | **Removed** |
| New options | — | `identUrl`, `authTimeoutMs` |
| `getSid()`, localStorage key | Unchanged | Unchanged |

Upgrade steps:

1. Change the `<script>` tag to `telcoads-sdk-v2.min.js` at tag `v2.0.1`, with the new `integrity` value.
2. Remove `endpoint`, `channelType`, `devIp` from `init()`.
3. Add `timeoutMs: 10000` to `init()` during the pilot phase.
4. Check your CSP (if any) as described in [Integration notes](#integration-notes).

### Post-integration checklist

| Scenario | Expected result |
|---|---|
| Open the page on **Viettel mobile data in Vietnam** (phone with Wi-Fi off, or a computer tethered to a Viettel mobile hotspot) | `getSid()` returns a `sid` string (not `null`) |
| Call `getSid()` again while the `sid` is still valid (under 5 minutes) | Returns the same `sid` with no network request |
| Open the page on **Wi-Fi**, another network, or **from outside Vietnam** | `getSid()` returns `null`; the page and ads keep working normally |
| Invalid `clientId` | `getSid()` returns `null`; the Console shows a warning starting with `[TelcoAds] ident 401` |

Test environment: a registered HTTPS staging page, or a computer tethered to a Viettel mobile hotspot opening the page via `http://localhost` (during the pilot phase, Production accepts `http://localhost` for development). After changing the configuration (e.g. fixing a wrong `clientId`), clear the result remembered by the SDK with `localStorage.removeItem('telcoads:sid')` and reload the page.

**Testing from outside Vietnam:** only the `null` scenarios (Wi-Fi/other networks, invalid `clientId`) can be verified from abroad. To verify the scenario that returns a `sid`, contact your TelcoAds integration contact to arrange a test on the Viettel network.

Enable `debug: true` while testing to see SDK warnings in the Console. When you need help from TelcoAds, include the `request_id` from the response (see the Network tab in DevTools).

### SDK console messages

Console messages from SDK v2.0.1 are written in Vietnamese and start with `[TelcoAds]`. Main messages and their meaning:

| Message (excerpt) | Meaning |
|---|---|
| `TelcoAds.init: siteId bắt buộc, không được chứa "~"` (`TypeError`) | `siteId` is required and must not contain `~`. |
| `TelcoAds.init: clientId bắt buộc …` (`TypeError`) | `clientId` is required. |
| `TelcoAds.init: identUrl phải là URL tới /ident …` (`TypeError`) | `identUrl` must be the URL of `/ident`; omit it to use Production. |
| `[TelcoAds] ident 401 — clientId không hợp lệ/chưa kích hoạt` | `clientId` is invalid or not activated (always printed). |
| `[TelcoAds] ident no_match — negative cache 60s` | User not identified; result remembered for `negativeTtlS` seconds. |
| `[TelcoAds] ident lỗi tạm HTTP <status>` | Temporary HTTP error from `/ident`; calls paused for 10 seconds. |
| `[TelcoAds] ident lỗi mạng/timeout: …` | Network error or `/ident` exceeded `timeoutMs`. |
| `[TelcoAds] ident timeout` | No authentication result within `authTimeoutMs`. |
| `[TelcoAds] ident lỗi: <error_code>` | Error inside the identification result (see [error codes](#identification-api-error-codes)). |
| `[TelcoAds] không mở được authorize_url: …` | Could not attach the iframe (usually `getSid()` was called before `<body>` existed). |
| `[TelcoAds] ident 200 thiếu authorize_url` | Unexpected `/ident` response; contact TelcoAds with the `request_id`. |
| `[TelcoAds] chưa init` | `getSid()` was called before `init()`. |

All messages except the `TypeError`s and `ident 401` appear only with `debug: true`.

## Backend integration with the TelcoAds API

This section applies when an Adtech/DSP integrates directly from its backend (calling `GET /v1/segments` or `POST /v1/match`). Fill in the following values for the environment provided by TelcoAds:

| Variable | Example | Notes |
|---|---|---|
| `TELCOADS_BASE_URL` | `https://{{api-host}}` | Base URL for the environment. No trailing `/`. |
| `PUBLISHER_CLIENT_ID` | `{{publisher-client-id}}` | Publisher ID for the Identification API. |
| `DSP_CLIENT_ID` | `{{dsp-client-id}}` | DSP ID for the Segment Catalog API. |
| `DSP_CLIENT_SECRET` | `{{dsp-client-secret}}` | DSP secret. Keep it in a secret manager, never in source code. |
| `SITE_ID` | `{{site-id}}` | Website ID registered with TelcoAds. |

### General conventions

- All traffic uses HTTPS; requests and responses use UTF-8 JSON.
- For POST requests, send `Content-Type: application/json` and `Accept: application/json`.
- Tokens/credentials are not interchangeable between environments.
- All APIs return errors in the same JSON shape:

```json
{
  "error_code": "INVALID_INPUT",
  "error_message": "dữ liệu đầu vào không hợp lệ",
  "request_id": "04622c5a1a2c427783c100e6e77ea65c"
}
```

`error_code` is the value your application should act on. `error_message` is human-readable text, **currently in Vietnamese**, and may change without notice; do not parse it. `request_id` is used when asking TelcoAds to investigate.

### Endpoints

| API | Caller | Purpose | Authentication |
|---|---|---|---|
| `GET /ident` | Publisher's SDK | Opens an identification session through the authentication gateway, or returns a `sid` immediately when a result is already available | Valid Publisher `client_id` + Origin |
| `GET /ident/result` | Publisher's SDK | Retrieves the identification result when no `postMessage` was received (fallback) | Matching Origin and session `nonce` |
| `GET /v1/segments` | DSP | Retrieves the segment catalog for campaign setup | DSP HTTP Basic Auth |
| `POST /v1/match` | Adtech/DSP server | Matches a `sid` against target segments | Valid `sid` |

## API reference

### Identification API — GET /ident

Opens an identification session for a visit. **The SDK performs all of this automatically**; the details below help partners understand the flow, verify their integration and investigate incidents.

The flow has three steps:

1. The SDK calls `GET /ident`.
2. The browser opens the gateway's `authorize_url` in an iframe; the gateway identifies the subscriber and redirects back to TelcoAds.
3. TelcoAds sends the result to the Publisher page via `postMessage`, and keeps it for 60 seconds so the SDK can fetch it from `GET /ident/result` if needed.

**Endpoint**

```
GET {{TELCOADS_BASE_URL}}/ident?site_id={{site-id}}&client_id={{publisher-client-id}}&nonce={{nonce}}
```

**Headers**

| Name | Required | Example | Description |
|---|---|---|---|
| `Origin` | Yes (set by the browser) | `https://news.example.vn` | Publisher page origin; must be HTTPS. The identification result is only delivered to this origin. |

**Query parameters**

| Parameter | Type | Required | Validation | Description |
|---|---|---|---|---|
| `site_id` | string | Yes | Not empty; must not contain `~` | Registered Publisher website ID. |
| `client_id` | string | Yes | Not empty; active Publisher | Publisher ID issued by TelcoAds. |
| `nonce` | string | Yes | Exactly 24 hex characters | Random session value generated by the SDK; used to match the result to the right session. |

**Success response — 200 OK**

There are two kinds of 200 response.

*Type 1 — authentication required* (no `status` field):

| Field | Type | Description |
|---|---|---|
| `authorize_url` | string | Authentication gateway URL for the browser to open. Single use, valid for about 30 seconds. |
| `state_id` | string | Identification session ID, used with `GET /ident/result`. |
| `expire_in` | integer | Seconds the identification session remains valid (default 120). |
| `error_code`, `error_message` | string | Empty strings on success. |
| `request_id` | string | Request trace ID. |

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

*Type 2 — immediate result* (has a `status` field): an `ok` or `no_match` result is returned immediately when already available. Same structure as the [Identification result](#identification-result).

#### Identification result

The JSON object TelcoAds sends to the Publisher page via `postMessage`; the same object is returned by type-2 `/ident` responses and by `/ident/result`.

| Field | Type | Description |
|---|---|---|
| `source` | string | Always `telcoad-ident`; used by the SDK to filter messages. |
| `status` | string | `ok`: identified · `no_match`: not identified (Wi-Fi, outside the Viettel network) · `error`: processing error, see `error_code` · `pending` / `expired`: `/ident/result` only. |
| `ident_token` | string | Only when `status = ok`. This is the `sid`, used with `POST /v1/match`. |
| `expire_time` | integer | Only when `status = ok`. `sid` expiry time (Unix time, seconds); 300 seconds after issue by default. |
| `nonce` | string | The `nonce` sent by the SDK to `/ident`. |
| `error_code`, `error_message` | string | Empty strings when there is no error. |
| `request_id` | string | Trace ID; include it when asking TelcoAds for help. |

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

`no_match` **is not an error**: it is a valid result when the user is outside identification coverage. Fall back to non-personalized ads.

### Fallback result — GET /ident/result

Used when the page did not receive the `postMessage` (iframe blocked, message lost…). The result is kept for 60 seconds, **can be read only once**, and is only returned to the same origin and `nonce` that opened the session.

```
GET {{TELCOADS_BASE_URL}}/ident/result?state_id={{state-id}}&nonce={{nonce}}
```

| Parameter | Type | Required | Description |
|---|---|---|---|
| `state_id` | string | Yes | Value received from `/ident`. |
| `nonce` | string | Yes | Same `nonce` sent to `/ident`. |

A 200 response is an [Identification result](#identification-result) object, where `status` may also be:

- `pending` — no result yet, ask again later.
- `expired` (`error_code = STATE_EXPIRED`) — the session has expired or the result was already read; stop polling.

```json
{
  "source": "telcoad-ident",
  "status": "pending",
  "error_code": "",
  "error_message": "",
  "request_id": "0a1b2c3d4e5f60718293a4b5c6d7e8f9"
}
```

### Identification API error codes

**Errors returned as HTTP status codes** (`GET /ident`, `GET /ident/result`):

| HTTP | error_code | Common causes | Action |
|---|---|---|---|
| 400 | `INVALID_INPUT` | Missing or empty `site_id`, `client_id` · `site_id` contains `~` · `nonce` is not 24 hex characters · calling page is not HTTPS (missing/invalid Origin) · malformed `state_id` | Fix the integration; do not retry with the same data. |
| 401 | `UNAUTHORIZED` | `client_id` does not exist, is inactive, or lacks the Publisher role | Check `clientId` and the environment. |
| 403 | `FORBIDDEN` | `/ident/result` called from another origin, without `nonce`, or with a `nonce` from a different session | Do not retry. |
| 404 | `IDENT_DISABLED` | Identification is not enabled in the environment being called | Contact TelcoAds. |
| 429 | `RATE_LIMIT_EXCEEDED` | Too many identification requests | The SDK pauses for 10 seconds. |
| 500 | `INTERNAL_ERROR` | System error | The SDK pauses for 10 seconds. |

**Errors inside the Identification result** (`status = error` or `expired`):

| error_code | Common causes |
|---|---|
| *(authentication error codes)* | Errors raised while authenticating the subscriber (session rejected, invalid response, slow response…). The list of codes may change; no per-code handling is needed. |
| `STATE_EXPIRED` | The identification session expired (120 seconds by default) or the result was already read. |
| `INTERNAL_ERROR` | System error. |

The SDK handles all of the errors above and always returns `null`; partners do not need to handle them:

- `401 UNAUTHORIZED`: the result is remembered for `negativeTtlS` seconds (default 60) and a console warning is always printed.
- `403 FORBIDDEN` from `/ident/result`: ignored; the SDK keeps waiting for the `postMessage` until `authTimeoutMs` elapses.
- All other errors (400, 404, 429, 500, errors inside the Identification result): calls are paused for 10 seconds and nothing is written to localStorage.

When asking for help, send TelcoAds the `request_id`.

### Segment Catalog API — GET /v1/segments

DSPs use this endpoint to download the segment catalog for campaign setup. Sync the catalog periodically as your operations require; do not call this endpoint in the real-time ad serving path.

**Endpoint and authentication**

```
GET {{TELCOADS_BASE_URL}}/v1/segments
Authorization: Basic base64({{dsp-client-id}}:{{dsp-client-secret}})
Accept: application/json
```

Only credentials registered with the DSP role can call this endpoint. Never pass `client_secret` in the query string or write it to application logs.

**Request**

No query parameters and no request body.

| Header | Type | Required | Validation | Description |
|---|---|---|---|---|
| `Authorization` | string | Yes | Format `Basic <base64(client_id:secret)>`; active DSP | DSP credentials. |
| `Accept` | string | Recommended | `application/json` | Expected response type. |

**Success response — 200 OK**

| Field | Type | Description |
|---|---|---|
| `segments` | array\<object\> | Available segments. May be an empty array. |
| `segments[].segment_id` | string | Segment ID; use this value in `requested_smg_ids`. |
| `segments[].segment_name` | string | Display name of the segment. |
| `segments[].segment_group` | string | Segment group. |
| `segments[].is_active` | boolean | Whether the segment is active. |
| `error_code` | string | Empty string on success. |
| `error_message` | string | Empty string on success. |

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

Segment names (`segment_name`) may be in Vietnamese; use `segment_id` as the key.

**Error codes**

| HTTP | error_code | Common causes | Action |
|---|---|---|---|
| 401 | `DSP_UNAUTHORIZED` | Missing `Authorization` header · not a valid Basic Auth header · `client_id` does not exist or is inactive · wrong secret · credential lacks the DSP role | Check the base URL and credentials; do not retry with the same credentials. |
| 500 | `INTERNAL_ERROR` | System error | Limited retries with exponential backoff; use the last synced catalog if appropriate. |

### User Matching API — POST /v1/match

Adtech/DSP servers use this API to check which segments in a target list match the user holding the `sid`. The API only returns `segment_id`s from the submitted list, in the same order.

Never call this endpoint directly from a browser: it is a server-to-server call. Do not log the `sid` or send it to third-party analytics systems.

**Endpoint**

```
POST {{TELCOADS_BASE_URL}}/v1/match
```

**Headers**

| Name | Required | Example | Description |
|---|---|---|---|
| `Content-Type` | Yes | `application/json` | Request content type. |
| `Accept` | Recommended | `application/json` | Expected response type. |

**Request body**

| Field | Type | Required | Validation | Description |
|---|---|---|---|---|
| `sid` | string | Yes | Not empty; a valid, unexpired `sid` issued by TelcoAds | Value from `TelcoAds.getSid()` (the `ident_token` field of the Identification API). |
| `requested_smg_ids` | array\<string\> | Yes | Non-empty list of target segments | Segment IDs (smg = segment) to match. We recommend using only `segment_id`s with `is_active: true`. |

```json
{
  "sid": "{{opaque-sid}}",
  "requested_smg_ids": ["001", "003", "004"]
}
```

**Success response — 200 OK**

| Field | Type | Description |
|---|---|---|
| `sid` | string | The `sid` sent in the request. |
| `matched_smg_ids` | array\<string\> | Matching segments. An empty array is a valid result: the user belongs to none of the requested segments. Ordered as in `requested_smg_ids`. |
| `error_code` | string | Empty string on success. |
| `error_message` | string | Empty string on success. |

```json
{
  "sid": "{{opaque-sid}}",
  "matched_smg_ids": ["003"],
  "error_code": "",
  "error_message": ""
}
```

**Error codes**

| HTTP | error_code | Common causes | Action |
|---|---|---|---|
| 400 | `INVALID_INPUT` | Invalid JSON · missing or empty `sid` · missing or empty `requested_smg_ids` | Fix the request; do not retry with the same data. |
| 401 | `UNAUTHORIZED` | Invalid `sid` · tampered `sid` | Do not retry. |
| 401 | `SID_NOT_FOUND` | `sid` has expired | Get a new `sid` (the SDK re-identifies on the next `getSid()` call), then call match again. |
| 404 | `HASHID_NOT_FOUND` | No session data left for this `sid` | Treat as not matchable at this time. |
| 500 | `INTERNAL_ERROR` | System error | Limited retries with exponential backoff; if still failing, use the non-personalized flow. |

## Error handling and retries

| Error group | Examples | Retry? | Recommended action |
|---|---|---|---|
| Request errors | `400 INVALID_INPUT` | No | Fix the data or integration code. |
| Authentication/authorization errors | `401 UNAUTHORIZED`, `401 DSP_UNAUTHORIZED`, `403 FORBIDDEN` | No | Check credentials, environment and integration permissions. |
| No data | `status no_match`, `404 HASHID_NOT_FOUND`, `401 SID_NOT_FOUND` | Not the same request | Fall back to non-personalized ads; `SID_NOT_FOUND` requires a new `sid`. |
| Temporary identification errors | `429`, authentication errors, `STATE_EXPIRED`, timeout | Handled by the SDK | The SDK returns `null` and pauses for 10 seconds. **Do not write your own loop calling `getSid()`.** |
| Temporary backend errors | `500 INTERNAL_ERROR` on `/v1/segments`, `/v1/match` | Yes, limited | Exponential backoff, at most 2–3 attempts, within your ad processing time budget. |

Backoff example: wait a random time around 100 ms, 300 ms, 900 ms; stop as soon as little ad processing time remains.

This document describes the public integration contract. Internal processing, data sources and storage are out of scope.
