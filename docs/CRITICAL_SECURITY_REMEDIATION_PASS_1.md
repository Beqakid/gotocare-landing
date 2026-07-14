# Critical Security Remediation — Pass 1

**Date:** 2026-07-14  
**Repository:** `Beqakid/gotocare-landing`  
**Scope:** URL-borne admin credential, session-token, and invite-token containment

## Findings addressed

- Removed the admin launch key from both page-entry and API-request query strings.
- Removed admin session tokens from privileged API query strings in all three admin dashboard variants.
- Changed the invite landing flow from a query token (`?t=…`) to a URL fragment (`#t=…`), which is not sent in HTTP requests or referrers. The fragment is removed from the address bar with `history.replaceState` immediately after it is read.
- Changed invite verification and acceptance to send the invite token in the `Authorization: Bearer …` header to the existing endpoints. The invite password remains in a POST JSON body and is not placed in a URL.
- Added `<meta name="referrer" content="no-referrer">` to every remediated admin page.
- Privileged request helpers fail closed when no admin credential is available.

No secret value was retrieved, copied, printed, or committed by this remediation.

## Files changed

- `admin-launch.html`
- `admin-panel.html`
- `admin.html`
- `admin/index.html`
- `admin-invite/index.html`
- `docs/CRITICAL_SECURITY_REMEDIATION_PASS_1.md`

## Tests and checks

This repository has no package manifest, automated test configuration, or existing test framework, so no reliable regression test could be added without introducing broad test scaffolding.

Checks actually run against the changed files:

- A static Python assertion checked all five remediated HTML files for credential-like query parameters (`token`, `key`, `password`, `session`, or invite `t`), reads of those credentials from `location.search`, and the required `no-referrer` meta policy: **passed**.
- Inline JavaScript was extracted and checked with `node --check`: `admin.html`, `admin-panel.html`, `admin-launch.html`, `admin-invite/index.html`, and the first two extracted script blocks from `admin/index.html` **passed**.
- The third extracted script block in `admin/index.html` could not be validated because the existing file contains HTML markup inside that extracted script region near the “PHASE 21A: KAI COPILOT TAB” marker. This remediation did not modify that structural defect, and a full-page browser smoke test remains required.

Owner/CI must run these deployment-coupled checks before release:

1. Exercise login, saved-session validation, every admin dashboard tab, mutations, launch metrics, and logout against staging.
2. Confirm every privileged endpoint accepts `Authorization: Bearer <credential>` and rejects missing, malformed, expired, revoked, and query-string credentials.
3. Confirm CORS allows the `Authorization` request header only from approved admin origins.
4. Generate a staging invite whose URL uses `#t=…`; verify it, accept it once, and confirm replay and expiry fail.
5. Confirm access, application, proxy, CDN, analytics, and error logs do not record credentials or credential-bearing URLs.

## Deployment coupling and risks

The static frontend and API must be deployed in a coordinated order. Deploy backend support first. The existing admin implementation already used Bearer authorization in an adjacent request helper, but every affected endpoint still must be verified. The existing invite endpoints are retained; they must be updated/configured to read the invite credential from the Authorization header. The invite generator must emit fragment links (`/admin-invite/#t=…`) rather than query links.

If the backend does not accept the header, admin requests will fail closed rather than fall back to URL credentials. Custom `Authorization` headers can trigger CORS preflight; the API must explicitly allow the approved origin and header. Old query-form invite links intentionally no longer work and must be reissued after invalidation.

## Exact owner actions

1. Coordinate the API and static-site release; verify Bearer-header handling and CORS in staging before publishing this frontend.
2. Rotate every affected admin key/password or equivalent credential and invalidate all active admin sessions. Do not reuse prior values.
3. Invalidate outstanding invite tokens and reissue only short-lived, single-use fragment links after backend support is live.
4. Inspect browser history and sync data, referrer records, reverse-proxy/server/CDN logs, Cloudflare logs, analytics, monitoring/error reports, support artifacts, screenshots, and caches for prior credential-bearing URLs. Preserve evidence and assess unauthorized use before applying retention-safe deletion.
5. Confirm invite tokens are cryptographically random, short-lived, single-use, invalidated atomically on acceptance, and never logged.
6. Verify query-string credentials are rejected server-side after migration; do not retain compatibility fallback.
7. Decide whether this repository should remain public only after a sanitized marketing-only and git-history review.
8. Perform a manual browser smoke test of all admin variants, especially `admin/index.html` because of its pre-existing malformed script/HTML region.

## Rollback

If the frontend must be rolled back, revert this commit only after pausing public access to the affected admin pages. **Do not restore query-string authentication.** If API header support is unavailable, keep the admin UI unavailable until the API is corrected. Rotated credentials, invalidated sessions, and invalidated invites must not be restored.

## Remaining findings and residual risk

- Historical leakage remains unassessed until the owner completes the log, browser-history, analytics, cache, and repository-history investigation.
- Admin session/key material is still stored in `localStorage` by the existing architecture, leaving it exposed to same-origin script execution. Migrating to a Secure, HttpOnly, SameSite cookie requires a coordinated same-origin server exchange and is deferred rather than invented in this static repository.
- Backend enforcement, CORS behavior, query-token rejection, invite expiry/single-use behavior, and invite-link generation are outside this repository and remain deployment blockers until verified.
- `admin/index.html` has a pre-existing structural script/HTML parsing defect that prevented complete standalone syntax validation.

## Next pass

- Move admin sessions to a Secure, HttpOnly, SameSite cookie using a real same-origin server exchange.
- Add server-side regression tests for header-only authentication, query-token rejection, invite expiry, one-time redemption, replay rejection, redacted logging, and CORS.
- Add a repository test harness or HTML/browser smoke-test workflow, then repair and test the `admin/index.html` structural defect.
- Complete sanitized source/history review and decide repository visibility.
