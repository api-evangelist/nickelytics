---
name: Authenticate to the R-Ads platform and scope a session to an advertiser
description: >-
  Sign in to the Nickelytics / R-Ads platform with email and password, confirm
  the session, pin it to the right advertiser organization, and mint a verifiable
  JWT for downstream calls.
api: openapi/nickelytics-rads-auth-openapi.json
base_url: https://api-ads.robot.com/api/auth
operations: [signInEmail, getSession, setActiveOrganization, getJSONWebToken, getJSONWebKeySet,
  signOut]
generated: '2026-08-12'
method: generated
source: openapi/nickelytics-rads-auth-openapi.json
---

# Authenticate to the R-Ads platform and scope a session

Nickelytics ships as **R-Ads by Robot.com**. The only public, machine-readable
contract is the platform authentication API. Everything below is grounded in
operations that exist verbatim in `openapi/nickelytics-rads-auth-openapi.json`.

## Before you start

- Base URL: `https://api-ads.robot.com/api/auth` (the legacy host
  `https://api.nickelytics.com/api/auth` serves the identical application from
  the same load balancer).
- Credentials come from an R-Ads Ad Manager account created at
  <https://admanager.robot.com/>. There is no self-service API key program and no
  developer portal.
- Auth is `Authorization: Bearer <token>` or the `apiKeyCookie` session cookie.
- **There is no idempotency contract.** No operation accepts an
  `Idempotency-Key`. Do not blind-retry a POST — re-read state with
  `getSession` first.

## Steps

1. **Sign in** — `signInEmail` (`POST /sign-in/email`) with `email` and
   `password`. A success returns `{ redirect: false, token, user }`; the `token`
   is the session token. If the account is configured for social or SAML SSO the
   response redirects instead, and you must complete that flow in a browser —
   an unattended agent cannot finish it.
2. **Confirm the session** — `getSession` (`GET /get-session`). Verify the
   `user` and read `session.activeOrganizationId`. Never assume the sign-in
   response and the live session agree.
3. **Pin the advertiser** — `setActiveOrganization`
   (`POST /organization/set-active`) when the account belongs to more than one
   organization. Every organization-scoped read afterwards resolves against
   this value, so an unset active organization is the most common cause of an
   empty result set rather than an error.
4. **Mint a token for downstream use** — `getJSONWebToken` (`GET /token`)
   returns a JWT for the current session. Unauthenticated calls return
   `{"message":"Unauthorized","code":"UNAUTHORIZED"}` with HTTP 401.
5. **Verify the token** — `getJSONWebKeySet`
   (`GET /.well-known/jwks.json`, i.e.
   `https://api-ads.robot.com/api/auth/.well-known/jwks.json`) returns the
   platform JWKS. Keys are `EdDSA` over `Ed25519`; select by `kid`.
6. **Finish cleanly** — `signOut` (`POST /sign-out`) invalidates the session.
   Do not leave long-lived sessions open; `listUserSessions`
   (`GET /list-sessions`) will show every one you left behind.

## Error handling

Errors are a flat `{"message": "..."}` JSON body — **not** RFC 9457. The same
seven statuses are declared on every operation:

| Status | Meaning | What to do |
|---|---|---|
| 400 | Missing or invalid parameters | Fix the request; do not retry as-is |
| 401 | Missing or invalid authentication | Re-run step 1 |
| 403 | Not permitted for this user/role | Stop; escalate to an org owner |
| 404 | Resource not found | Check the active organization (step 3) |
| 429 | Rate limited | Back off — no `Retry-After` or `RateLimit-*` header is returned, so use exponential backoff with jitter |
| 500 | Server error | Retry idempotent reads only |

## What this API is not

This API manages **identity and tenancy only**. Campaigns, placements,
creatives, robot routes and proof-of-play reporting are not exposed here. That
surface lives at `https://api.nickelytics.com/v1`, which answers
`401 Authorization header required` and publishes no specification — buy and
manage media through the R-Ads Ad Manager UI instead.
