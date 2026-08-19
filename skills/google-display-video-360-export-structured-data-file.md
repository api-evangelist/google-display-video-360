---
name: Export a Structured Data File
description: Run DV360's only asynchronous workflow — start an SDF download task, poll the long-running Operation, and retrieve the file, handling the fact that failures surface on the Operation rather than on the request that started it.
api: openapi/google-display-video-360-api-openapi.yml
operations:
  - sdfdownloadtasksCreate
  - sdfdownloadtasksOperationsGet
  - mediaDownload
---

# Export a Structured Data File

Structured Data Files are DV360's bulk export format — the way to pull an entire advertiser's
campaign structure out in one pass instead of walking the resource tree with hundreds of list
calls. It is also the only asynchronous surface in this API, and the only place where a request
can succeed and the work behind it still fail.

## Before you start

- OAuth 2.0 token on `https://www.googleapis.com/auth/display-video`, with a DV360 user profile
  holding at least **Read only** on the advertiser or partner in scope.
- Decide the SDF version. Google enabled SDF v10 on 2026-06-10; SDF v6 was sunset on
  2025-04-30. Pin a version explicitly rather than relying on a default that moves.

## Steps

1. **Start the task.** `sdfdownloadtasksCreate` on `POST /v4/sdfdownloadtasks`. Supply the
   `version`, the scope (`partnerId` or `advertiserId`), and the filter describing what to
   include. The response is a `google.longrunning.Operation` — **not** the file. Keep
   `operation.name`.

2. **Poll the Operation.** `sdfdownloadtasksOperationsGet` on
   `GET /v4/{+name}`, passing the `operation.name` from step 1. Poll until `done: true`.
   Back off between polls: a tight poll loop is a read against the same 1500-per-minute project
   quota as everything else, and there is no push notification to wait on — this API has no
   webhooks or callbacks of any kind.

3. **Read the outcome from the finished Operation.** When `done` is true, exactly one of two
   fields is set:
   - `response` — the task succeeded and carries the resource name of the generated file.
   - `error` — the task failed, with a `google.rpc.Status` describing why.
   This is the step integrations get wrong. Step 1 returning `200` means the task was *accepted*,
   nothing more. If you treat that 200 as success you will report exports as complete that never
   produced a file.

4. **Download the file.** Fetch the media resource named in `operation.response` via
   `mediaDownload`. The result is a ZIP of CSVs.

## Rules that will bite you

- **No completion callback exists.** Polling is the only mechanism. Choose an interval that
  respects quota — seconds, not milliseconds — and give large exports minutes, not seconds.
- **Failure is asynchronous.** Always branch on `operation.error` before touching
  `operation.response`.
- **SDF versions are deprecated on a schedule.** Track
  `lifecycle/google-display-video-360-lifecycle.yml`; Google announces SDF version sunsets on the
  deprecations page and emits no `Sunset` header, so nothing in your traffic will warn you.
- **Targeting columns at the insertion-order and campaign level became non-writable on
  2026-02-23.** An SDF upload round-trip that used to modify them will now fail.
- **No idempotency key.** A retried `sdfdownloadtasksCreate` starts a second task and burns quota
  twice. Track the `operation.name` you already have before starting another.

## Errors

Request-level failures follow the standard envelope — branch on `error.status`. Task-level
failures arrive inside `operation.error` with the same `google.rpc.Code` vocabulary. See
`errors/google-display-video-360-problem-types.yml`.
