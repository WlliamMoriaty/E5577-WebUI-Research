# Huawei E5577s-321 WebUI Analysis & Build Plan

> Status: research-first; do not flash or write router settings during analysis.
> Target observed by the project owner: Huawei E5577s-321, WebUI 17.100.11.03.1367, firmware 21.328.01.00.00, hardware CL1E5573SM02.

## 1. What is actually in the repository

The repository contains the original WebUI firmware package and two generated text reports:
- `webui-api-inventory.txt`: searchable index of 124 JavaScript-related assets and API references (1316 lines, 118,969 bytes).
- `webui-audit-report.txt`: full local audit report (1,074,459 bytes). The GitHub connector can confirm the file/blob exists, but its content fetch currently returns an empty result, so this document does not pretend to have read all of its contents through that connector.
- Firmware WebUI binaries and updater/release-note materials.

Important distinction: the inventory is an endpoint/function index, not a copy of the source code. Some behavior is confirmed by earlier inspection of locally extracted JavaScript; details that depend on code not yet visible here remain open questions.

## 2. Confirmed high-level architecture

The stock UI is a browser application that makes HTTP API requests to the router and consumes XML responses. Its common JavaScript helpers include:
- `getAjaxData(url, callback, options)` for GET-style requests.
- `saveAjaxData(url, xmlData, callback, options)` for POST-style requests.
- Shared response/error handling and session/token handling.
- Page-specific scripts for SMS, USSD, Wi-Fi, mobile network, SIM/PIN, device status, and other router settings.

The inventory identifies 124 JS-related assets/sections and 752 lines containing API references. The most represented API groups include WLAN, SMS, USSD, phonebook (`pb`), user/authentication, monitoring, device, network/dial-up, security, and SD-card functions. Counts are inventory matches, not unique API counts or proof every endpoint works on this hardware.

## 3. Authentication: what is confirmed vs. unknown

### Confirmed from the inventory/code references
- `api/webserver/token` is requested during startup.
- `api/webserver/publickey` is requested by the common main script.
- `api/user/login` is submitted by the login flow.
- `api/user/state-login` is queried repeatedly to check the login state.
- `api/user/session` appears in request/error handling.
- `api/user/logout` is submitted by the logout flow.
- The common AJAX helpers add the `__RequestVerificationToken` header when the shared request-token variable is set.
- Response handling includes special logic for login success and missing response-token conditions.

### Not yet proven; do not assume
- The exact password transformation/encryption algorithm and its input format.
- Whether the public key is RSA or another format for this exact firmware build, and its key size/encoding.
- Whether the token returned by `api/webserver/token` is identical to the token carried in `__RequestVerificationToken`, how it rotates, and its expiry.
- Cookie/session identifiers, cookie flags, timeout policy, and whether sessions are IP-bound.
- Whether the router serves the UI only over HTTP or can enforce HTTPS on this firmware.
- Lockout/rate-limiting behavior and server-side password-storage details.

### Authentication flow to implement only after verification
1. Bootstrap: request the stock token and public-key metadata.
2. Parse the XML strictly; fail closed if required fields are absent or malformed.
3. Prepare the login payload using the exact algorithm confirmed from the stock login code, not a guessed hash/encryption method.
4. POST to `api/user/login` with the required token/header semantics.
5. Confirm success by parsing the response and checking `api/user/state-login`.
6. Store only the minimum transient state needed in memory; do not persist passwords or tokens in localStorage/sessionStorage.
7. Attach the current request-verification token to protected calls exactly as the stock client does; handle token rotation and session-expiry responses.
8. Logout through `api/user/logout`, clear in-memory state, and return to the locked/login view.

Never log credentials, session cookies, or verification tokens. Do not commit captures containing them.

## 4. API areas found and planned modules

| Module | Observed API examples | Initial implementation scope |
|---|---|---|
| Dashboard/status | `api/monitoring/status`, `api/device/information`, `api/device/signal` | Read-only status cards and refresh |
| Authentication | `api/webserver/token`, `api/webserver/publickey`, `api/user/login`, `api/user/state-login`, `api/user/session`, `api/user/logout` | Login/session manager |
| SMS | `api/sms/sms-count`, `api/sms/sms-list`, `api/sms/set-read`, `api/sms/send-sms`, `api/sms/delete-sms` | Read-only inbox first; mutations behind explicit confirmation |
| USSD | `api/ussd/send`, `api/ussd/get`, `api/ussd/status`, `api/ussd/release` | Dialog + polling state machine; never auto-send |
| Phonebook | `api/pb/pb-list`, `api/pb/pb-match` | Read-only contacts after response schema is confirmed |
| Wi-Fi | Many `api/wlan/*` endpoints | Read current settings first; editing comes later with validation |
| Network/SIM | `api/dialup/*`, `api/pin/*`, `api/net/*` | Status first; avoid PIN/PUK actions without a deliberate user confirmation |
| System/security | `api/device/*`, `api/security/*`, `api/online-update/*` | Display-only until each endpoint and rollback path are understood |

The API names are taken from the repository's inventory. Their presence does not guarantee availability on every device or firmware variant.

## 5. Important SMS/USSD behavior already identified

### SMS
The stock UI uses box types:
- 1 local inbox, 2 sent, 3 drafts, 4 trash
- 5 SIM inbox, 6 SIM sent, 7 SIM drafts
- 8 mixed inbox, 9 mixed sent, 10 mixed drafts

The list request uses fields such as `PageIndex`, `ReadCount`, `BoxType`, `SortType`, `Ascending`, and `UnreadPreferred`. This is useful for matching the router's native pagination instead of inventing a different API contract.

Potentially destructive/action endpoints include delete, send, cancel-send, and SIM backup. They must never be called by a dashboard refresh or automatically on page load.

### USSD
The stock code sends a request to `api/ussd/send` with `content`, `codeType`, and a timeout value; then reads `api/ussd/get`, checks `api/ussd/status`, and may call `api/ussd/release`. The stock implementation recognizes result/timeout codes (including `111019` and `111020` in the inspected code). We should preserve the device's expected state machine and avoid overlapping requests. USSD sends can trigger carrier-side actions and must require a deliberate click.

## 6. Security and reliability requirements

- Keep the UI same-origin with the router where possible; a separately hosted UI can hit browser CORS and origin restrictions.
- Do not create a cloud relay or expose the router API to the public internet.
- Use HTTPS if the device genuinely supports it; do not claim transport confidentiality over plain HTTP.
- Use a single API client that handles XML parsing, timeouts, request-token attachment, session expiry, and consistent errors.
- Parse XML with explicit error detection; never treat malformed XML as a successful response.
- Separate read-only requests from mutations. No automatic reboot, reset, PIN/PUK, firmware update, SMS send/delete, or USSD send.
- Show a confirmation screen for every consequential action and require a second explicit confirmation for destructive/high-risk operations.
- Keep passwords and tokens in memory only; do not put them in logs, URLs, screenshots, analytics, or Git.
- Do not store SMS contents or contacts in persistent browser storage by default.
- Maintain compatibility with the stock XML schema and older browser/runtime constraints of the router.
- Test on extracted files or a local mock first. Back up the current WebUI/firmware and document a recovery route before any installation or flashing.

## 7. Proposed application structure

Keep the first build dependency-light and separable from the vendor UI:

```text
app/
  index.html
  assets/
    styles/
      tokens.css
      layout.css
      components.css
    js/
      app.js
      router-api.js       # same-origin requests + XML parsing
      auth-manager.js     # login/token/session/logout
      state-store.js      # in-memory state only
      modules/
        dashboard.js
        sms.js
        ussd.js
        wifi.js
        device.js
  tests/
    xml-fixtures/
    api-mock.js
docs/
  ANALYSIS-AND-BUILD-PLAN.md
```

This is a proposed structure, not a claim that these files already exist. Start with a responsive read-only dashboard; then authentication; then SMS inbox and USSD; then carefully gated settings modules.

## 8. Build phases and acceptance criteria

### Phase 0 — finish source verification
- Read the relevant login/token/session functions from the exact extracted WebUI version.
- Compare duplicate versioned scripts (e.g. `41601/js/main.js`, `common/js/main.js.gz`, `common/js/checklogin.js.gz`) and record which one is actually loaded by the target pages.
- Confirm XML schemas and token lifecycle from source, not guesswork.
- Resolve why the full audit report cannot currently be read through the GitHub connector.

### Phase 1 — safe UI shell
- Responsive mobile/desktop layout, Arabic RTL support, accessible focus/contrast.
- Mock data only; no router requests.
- No secrets, no writes, no firmware changes.

### Phase 2 — API client + authentication
- XML parser tests and mocked API responses.
- Correct login/token/session handling verified against stock source.
- Session expiry returns to login and clears transient state.

### Phase 3 — read-only modules
- Status dashboard, signal/network, SMS counts/list, phonebook read, USSD status/result display.
- Explicit loading/empty/error states and manual refresh.
- No background polling that could overload the router.

### Phase 4 — controlled actions
- SMS send/delete and USSD send only after request schemas and error behavior are verified.
- Confirmation and validation, prevent duplicate submissions, clear success/failure feedback.
- Settings writes only after per-endpoint review and a backup/rollback plan.

### Phase 5 — package and install
- Build static assets compatible with the router's actual serving/package format.
- Verify path/case sensitivity and compression requirements.
- Test offline/local, then on the router with read-only calls first.
- Install only after backup and a documented recovery procedure.

## 9. Immediate next work

The next useful code work is not the visual dashboard yet: first establish the exact source for the login flow and the API response contracts. The repository currently has the firmware binaries and inventories, but not a directly browseable set of extracted source files; the full audit report is present as a large blob but the current GitHub fetch tool returns no content for it. Until that is resolved, authentication details must stay marked as unverified.

Once the source evidence is available, create the API client and tests before connecting UI controls to the router.
