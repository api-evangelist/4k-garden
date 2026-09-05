---
name: 4k-garden-manage-enterprise-team
description: >-
  Set up and administer a Diebian AI enterprise account on 4K Garden — upgrade to a
  team, invite and remove members, set member levels, and monitor shared credit spend.
generated: '2026-09-05'
method: generated
source: openapi/4k-garden-diebian-ai-openapi.json
api: Diebian AI Super-Resolution API
base_url: https://video-cn.fly4k.com
operations:
  - loginUsingPOST
  - upgradeToEnterpriseUsingPOST
  - getMyEnterpriseUsingGET
  - inviteMemberUsingPOST
  - acceptInvitationUsingPOST
  - rejectInvitationUsingPOST
  - listMembersUsingGET
  - changeMemberLevelUsingPUT
  - removeMemberUsingDELETE
  - enterpriseStatsUsingGET
  - listEnterprisePlansUsingGET
  - getConsumptionUsingGET
---

# Manage a Diebian AI enterprise team

The Diebian AI API carries a full team surface: an enterprise account with members,
invitations, member levels, shared credits and usage statistics.

## Before you start — read this

These are **consequential organisational writes with no published reversal window**.
Removing a member or changing a level takes effect immediately, and nothing in the
contract or any public document states whether or how it can be undone. There is also
no idempotency mechanism, so a retried invite may create a duplicate.

Note also that `listEnterprisePlansUsingGET` currently returns an **empty** plan list,
so there is no published enterprise tier to buy even though this administrative surface
exists. Confirm commercial terms with 4K Garden directly (bd@4kgarden.com) before
building on it.

## Step 1 — Authenticate

`loginUsingPOST` (`POST /api/auth/login`) → `token`. The transport header name is not
published by the provider; confirm it before scripting.

## Step 2 — Create or find the team

- `upgradeToEnterpriseUsingPOST` (`POST /api/enterprise/upgrade`) converts the current
  account into an enterprise account. **Treat this as one-way** — no downgrade
  operation exists anywhere in the contract.
- `getMyEnterpriseUsingGET` (`GET /api/enterprise/my`) returns the enterprise you
  already belong to, as `EnterpriseDetailVo`. Call this first; do not upgrade blindly.

## Step 3 — Invite members

`inviteMemberUsingPOST` (`POST /api/enterprise/{enterpriseId}/invite`).

The invitee then resolves it with either `acceptInvitationUsingPOST`
(`POST /api/enterprise/invitations/{invitationId}/accept`) or
`rejectInvitationUsingPOST` (`.../reject`). Rejection is the reversal path for an
invitation; once accepted, the reversal is removal, which is a heavier action.

## Step 4 — Administer members

- `listMembersUsingGET` (`GET /api/enterprise/{enterpriseId}/members`) — the roster.
- `changeMemberLevelUsingPUT`
  (`PUT /api/enterprise/{enterpriseId}/members/{memberUserId}/level`) — takes a
  `ChangeMemberLevelAO`. Levels are not documented; read current values from the roster
  before setting one rather than guessing.
- `removeMemberUsingDELETE`
  (`DELETE /api/enterprise/{enterpriseId}/members/{memberUserId}`) — **irreversible in
  one call.** Re-adding requires a fresh invite and acceptance by the person.

Because `403` is declared on every operation, expect it whenever the calling principal
is a member rather than an administrator.

## Step 5 — Watch the shared spend

- `enterpriseStatsUsingGET` (`GET /api/enterprise/{enterpriseId}/stats`) returns
  `EnterpriseStatsVo`.
- `getConsumptionUsingGET` (`GET /api/user/consumption`) is the credit-spend ledger,
  paginated with `page` / `page_size` and a `total`.

Members spend shared credits by setting `use_enterprise_credits: true` when they create
a task — see the `4k-garden-upscale-a-video` skill. That flag is the only control point
between an individual job and the team balance, so audit it in the consumption ledger
rather than assuming it.
