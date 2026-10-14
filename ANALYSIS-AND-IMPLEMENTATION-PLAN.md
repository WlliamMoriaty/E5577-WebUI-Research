# Huawei E5577s-321 WebUI Research — Analysis & Build Plan

## Scope and evidence

Target: Huawei E5577s-321, reported by the owner as firmware 21.328.01.00.00, WebUI 17.100.11.03.1367, hardware CL1E5573SM02. The repository contains the WebUI firmware package, release-note files, and two extracted local reports:
- `webui-api-inventory.txt` — 1,316 lines, 124 file sections, roughly 752 API-related references.
- `webui-audit-report.txt` — the extracted HTML/page audit report (about 1.07 MB).

The API inventory is a reference inventory, not a full source-code listing. Counts are occurrences/references, not unique endpoints. The firmware binary and updater are preserved as research inputs. No router API calls are performed by this document.

## High-level findings

### WebUI structure
The package includes multiple versioned UI bundles plus shared compressed JavaScript (`.js.gz`), HTML and language resources. The same functions occur in more than one versioned bundle, so apparent duplicates should be deduplicated by path/version before making claims about behavior. The UI is an XML-oriented client for router endpoints, using common helpers such as `getAjaxData()` and `saveAjaxData()`.

### Authentication — verified references, remaining questions
The inventory identifies these relevant flows in `common/js/main.js.gz` and `common/js/checklogin.js.gz`:
- `GET api/webserver/publickey`
- `GET api/webserver/token`
- `POST api/user/login`
- `GET api/user/state-login`
- `POST api/user/session`
- `POST api/user/logout`
- `POST api/user/hilink_login`

The shared request helpers reference `__RequestVerificationToken` handling, and response handling treats the login and session endpoints specially. This is evidence of a request-verification token mechanism, but the inventory alone does **not** prove the exact password transformation, cryptographic algorithm/parameters, cookie name or lifetime, token rotation policy, transport security, or server-side authorization rules. Those must be confirmed from the actual extracted function bodies and, where appropriate, passive browser-network observations. Never publish real passwords, cookies, or tokens in the repo.

### API domains found
The inventory has the heaviest references in WLAN, SMS, USSD, phonebook, user/authentication, monitoring, device, network/dial-up, cradle, security, SD card, PIN, Bluetooth and virtual-SIM areas. The inventory contains many additional settings modules (DHCP, firewall, VPN, DDNS, update, backup/restore, and others). This is a map of code references, not proof that every endpoint is supported by this specific hardware/firmware or is safe to invoke.

### SMS
Observed UI endpoint references include:
- Read/count: `api/sms/sms-count`, `api/sms/sms-list`
- Contact matching: `api/pb/pb-match`
- Mutations: `api/sms/delete-sms`, `api/sms/backup-sim`, `api/sms/set-read`, `api/sms/save-sms`, `api/sms/send-sms`, `api/sms/cancel-send`

The UI distinguishes local storage, SIM storage, inbox, sent, drafts, trash, and mixed views. Exact supported box types and response fields should be documented from the extracted SMS source before implementing the adapter. Sending/deleting messages must require explicit user action and confirmation.

### USSD
Observed UI endpoint references:
- `POST api/ussd/send`
- `GET api/ussd/get`
- `GET api/ussd/status`
- `GET api/ussd/release`

The legacy UI builds an XML request containing USSD content, code type, and timeout, then polls for a result and handles pending/timeout responses. USSD can trigger carrier-side actions, so never auto-send or blindly replay a request. A release action should only run when the app owns an active USSD dialog.

### Status and notifications
The UI references `api/monitoring/status` and `api/monitoring/check-notifications`, and reads values such as unread-message count and SMS-storage-full state. This supports an in-app polling indicator. It does not establish that the router offers browser push notifications when the page is closed; that requires a separate local notification strategy.

## Proposed application architecture

1. **Local gateway/backend** on the laptop, bound to localhost by default. It owns the router communication and exposes a small, typed API to the frontend. It must not expose router credentials or verification tokens to third-party services.
2. **Frontend dashboard** with responsive layout: overview/status, signal/network, SMS, USSD, device/Wi-Fi settings, and diagnostics. Start read-only; add mutations after contract tests.
3. **Router adapter** with a single request layer for XML parsing/serialization, timeout handling, token refresh, session-state checks, and normalized errors. Avoid duplicating authentication logic in each screen.
4. **Session manager** keeps credentials out of logs and persistent browser storage. Keep secrets in memory where feasible, clear them on logout, and redact sensitive request/response fields.
5. **Capability registry** based on hardware/firmware checks, not merely on every endpoint found in the firmware.
6. **Tests and fixtures** built from sanitized sample XML responses. Unit tests must not require a live router; integration tests are opt-in and clearly label mutating actions.

## Implementation order

1. Confirm the precise source functions for login/public-key/token/session and record the exact request/response fields.
2. Build a read-only authentication/status adapter and test parsing against sanitized fixtures.
3. Implement overview and signal/status screens.
4. Add SMS listing and counts; only then add send/delete/read-state actions with confirmation.
5. Add USSD interactive dialog with timeout, polling, cancel/release handling and no automatic retries for side-effecting requests.
6. Add settings modules one at a time, each behind capability checks and explicit confirmation for destructive operations.
7. Add local/mobile notifications only after deciding how the local service remains available and what works while the UI is closed.
8. Add user-facing docs, supported firmware matrix, threat model, backup/recovery instructions and release packaging.

## Safety and reliability rules

- Do not execute firmware flashing, reset, reboot, restore, SMS-send, USSD-send, or settings-write endpoints during discovery.
- Do not assume endpoint names imply authorization or compatibility.
- Do not log passwords, public/private key material beyond public metadata, session cookies, or verification tokens.
- Prefer least privilege, local-only binding, explicit origin/CSRF protections on the new backend, strict input validation, and a clear confirmation step for state-changing actions.
- Keep a backup of the original firmware package and record hashes before any future firmware-related work.

## Open items before coding auth

- Read the actual extracted bodies around the public-key request, login payload construction, token response processing, session requests and logout.
- Determine whether the firmware's API requires cookie state in addition to the verification header.
- Confirm key format/algorithm and password encoding from code, not guesses.
- Verify which response codes indicate expired session, invalid credentials, token mismatch, pending USSD, and timeout.
- Confirm whether the app can communicate with the router from the intended browser/origin; if CORS blocks direct access, use the local gateway.
- Determine which versioned bundle matches the router's installed WebUI and avoid mixing assets from other hardware variants.
