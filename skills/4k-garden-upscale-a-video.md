---
name: 4k-garden-upscale-a-video
description: >-
  Submit a video to 4K Garden's Diebian AI platform for AI super-resolution, track the
  job to completion, and retrieve the output — while keeping control of the prepaid
  credit balance the job spends.
generated: '2026-09-05'
method: generated
source: openapi/4k-garden-diebian-ai-openapi.json
api: Diebian AI Super-Resolution API
base_url: https://video-cn.fly4k.com
operations:
  - loginUsingPOST
  - meUsingGET
  - getCreditRulesUsingGET
  - getBalanceUsingGET
  - createTaskUploadTestUsingPOST
  - createTaskUploadUsingPOST
  - listTasksUsingGET
  - getTaskUsingGET
  - cancelTaskUsingPOST
  - retryTaskUsingPOST
  - createDownloadUsingPOST
  - getDownloadUsingPOST
---

# Upscale a video with Diebian AI

Diebian AI (蝶变 AI) is 4K Garden's video super-resolution service. You give it a video
and a target output resolution; it returns a higher-resolution render. Work is paid for
from a prepaid credit balance, so the cost of a job is knowable *before* you submit it.

## Before you start — read this

The provider publishes **no developer documentation** for this API. Everything below is
grounded in the operationIds of its published OpenAPI contract. Two consequences matter:

- **The auth header name is not published.** `loginUsingPOST` returns a `token`, but no
  public document states which header carries it. Confirm it against the application's
  own traffic before scripting anything.
- **There is no idempotency mechanism.** None of the 43 mutating operations accepts an
  `Idempotency-Key`. If `createTaskUploadUsingPOST` times out ambiguously, **do not
  blindly retry** — call `listTasksUsingGET` first and check whether the task landed.
  Retrying a task creation charges you twice.

## Step 1 — Authenticate

Call `loginUsingPOST` (`POST /api/auth/login`). It returns `ApiResponse<LoginResultVo>`,
whose `data` carries `token` and a `user` object. Confirm the session with `meUsingGET`
(`GET /api/auth/me`).

Registration is invite-gated: `registerConfigUsingGET` reports
`invite_code_required: true`, so `registerUsingPOST` needs an invite code you already
hold.

## Step 2 — Price the job before you spend anything

Call `getCreditRulesUsingGET` (`GET /api/user/credit-rules`, readable anonymously). It
returns the credit rate bands, which are charged on **output frame rate**:

| Band | Output fps | Rate |
|---|---|---|
| low | up to 30 | 25 |
| medium | up to 60 | 50 |
| high | 61 and above | 100 |

Then call `getBalanceUsingGET` (`GET /api/user/balance`) and confirm the balance covers
the job. As a reference point, 50,000 credits is roughly 33 minutes of 30fps
super-resolution. If the balance is short, `productListUsingGET`
(`GET /api/user/productList`) lists the purchasable credit packages and their CNY prices.

## Step 3 — Rehearse

Call `createTaskUploadTestUsingPOST` (`POST /api/user/tasks/uploadTest`) first. This is
the only rehearsal affordance on the surface. Its exact semantics are undocumented, so
treat it as a smoke test of your request shape rather than proof of the final cost.

## Step 4 — Submit the real job

Call `createTaskUploadUsingPOST` (`POST /api/user/tasks/upload`). The `CreateTaskAO`
body carries `video_name`, `video_size`, `original_duration`, `original_fps`,
`original_resolution`, `output_resolution`, `remark`, and `use_enterprise_credits`.

Set `output_resolution` deliberately — it is what drives the frame-rate band and
therefore the price. Set `use_enterprise_credits` only if you intend to spend the team's
shared balance rather than your own.

**This call is billable and cannot be replayed safely.** Record the returned task id.

## Step 5 — Track it

Poll `getTaskUsingGET` (`GET /api/user/tasks/{taskId}`). `TaskDetailVo` carries
`status`, `progress`, `cost_credits`, `cost_rmb` and `work_duration`, so you can watch
both progress and realised cost. `listTasksUsingGET` (`GET /api/user/tasks`) lists your
tasks with `page` / `page_size` pagination and `total` in the response.

## Step 6 — Collect the output

Call `createDownloadUsingPOST` (`POST /api/user/download/create`), then
`getDownloadUsingPOST` (`POST /api/user/download/get`) to retrieve it.

## Reversing or repairing a job

- **Cancel:** `cancelTaskUsingPOST` (`POST /api/user/tasks/{taskId}/cancel`). **No
  cancellation window is published.** Do not assume a job is cancellable at any
  particular stage — check `status` first, and do not promise a refund you cannot
  evidence.
- **Retry:** `retryTaskUsingPOST` (`POST /api/user/tasks/{taskId}/retry`). **No public
  document states whether a retry re-debits credits.** Treat it as billable until the
  provider confirms otherwise; check `getBalanceUsingGET` before and after.
- **Dispute:** `appealTaskUsingPOST` (`POST /api/user/tasks/appeal`) raises an appeal
  against a completed task. This is a dispute path, not a cancellation.

## Reading responses and errors

Every response is the same envelope: `{code, msg, data}`. `code` is `0` and `msg` is
`"success"` on the success path. This is **not** RFC 9457 — there is no `type`, `title`,
`detail` or `instance`.

Every operation declares `401` and `403`; most declare `404`. `500` is **not** declared
in the contract but does occur in practice, and at least two catalogue endpoints
currently return it. There are no rate-limit headers and no `429`, so there is no
runtime backoff signal — pace your polling conservatively.
