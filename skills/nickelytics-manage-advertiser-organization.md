---
name: Manage an R-Ads advertiser organization and its members
description: >-
  Read an advertiser workspace in the Nickelytics / R-Ads platform, invite a
  teammate, and change a member's role — grounded in the published auth OpenAPI.
api: openapi/nickelytics-rads-auth-openapi.json
base_url: https://api-ads.robot.com/api/auth
operations: [getSession, setActiveOrganization, getOrganization, createOrganizationInvitation,
  updateOrganizationMemberRole]
generated: '2026-08-12'
method: generated
source: openapi/nickelytics-rads-auth-openapi.json
---

# Manage an R-Ads advertiser organization

An **Organization** is the advertiser or agency workspace in the R-Ads Ad
Manager. Members belong to it through a **Member** record carrying a role;
prospective members get an **Invitation** tied to an email address. All three
entities are declared in the published OpenAPI
(`data-model/nickelytics-data-model.yml` has the full graph).

## Prerequisites

Complete `nickelytics-authenticate-and-scope-session.md` first — you need an
authenticated session, and organization operations resolve against the session's
`activeOrganizationId`.

## Steps

1. **Confirm scope** — `getSession` (`GET /get-session`), then
   `setActiveOrganization` (`POST /organization/set-active`) if
   `session.activeOrganizationId` is not the workspace you mean to change.
   Skipping this is how invitations get sent into the wrong advertiser account.
2. **Read the workspace** — `getOrganization`
   (`GET /organization/get-full-organization`) returns the organization with its
   members and invitations. Check whether the person already holds a Member
   record or an outstanding Invitation before inviting them again.
3. **Invite a teammate** — `createOrganizationInvitation`
   (`POST /organization/invite-member`) with the email address and the role to
   grant. The Invitation carries `status` and `expiresAt`; it is not a
   membership until accepted.
4. **Adjust a role** — `updateOrganizationMemberRole`
   (`POST /organization/update-member-role`) for someone who is already a
   member. Roles are the platform's own strings, not a documented enum — read
   the existing members from step 2 rather than guessing a value.

## Rules that matter

- **No idempotency.** `POST /organization/invite-member` accepts no
  `Idempotency-Key`. If the call times out, re-read with `getOrganization`
  before sending it again or you will issue duplicate invitations.
- **403 is authoritative.** Role changes and invitations are permission-gated;
  a 403 means the acting user is not an owner or admin of that organization.
  Do not retry — escalate.
- **429 has no signal.** The API declares 429 on every operation but publishes
  no limit, window or `RateLimit-*`/`Retry-After` header. Use exponential
  backoff with jitter.
- **Admin operations are a different surface.** `/admin/*` and `/dash/*`
  (user creation, bans, impersonation, SSO providers, log drains, audit events)
  are platform-operator operations, and most of them declare no `operationId`
  at all. Treat them as out of scope for an agent.
