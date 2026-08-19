---
name: Audit line item targeting across an advertiser
description: Read every assigned targeting option across an advertiser's line items and resolve the opaque option IDs to human-readable values, using the bulk list method so the audit does not eat the quota.
api: openapi/google-display-video-360-api-openapi.yml
operations:
  - advertisersLineItemsList
  - advertisersLineItemsBulkListAssignedTargetingOptions
  - advertisersLineItemsTargetingTypesAssignedTargetingOptionsList
  - targetingTypesTargetingOptionsList
  - targetingTypesTargetingOptionsSearch
---

# Audit line item targeting

Targeting in DV360 is a join. A line item holds **assigned** targeting options, each of which
carries a `targetingOptionId` that means nothing on its own; the human-readable value lives in the
**targeting option** catalog, which is a separate top-level resource. An audit that only reads
assigned options produces a list of opaque numbers.

## Before you start

- OAuth 2.0 token on `https://www.googleapis.com/auth/display-video`. Read-only work needs only a
  DV360 user profile with **Read only** permission on the advertiser.
- Base URL: `https://displayvideo.googleapis.com`.

## Steps

1. **List the line items.** `advertisersLineItemsList` on
   `GET /v4/advertisers/{advertiserId}/lineItems`. Narrow with `filter`, e.g.
   `entityStatus="ENTITY_STATUS_ACTIVE"`, and page with `pageSize` + `pageToken` until the
   response has no `nextPageToken`.

2. **Read the assigned targeting in bulk.** `advertisersLineItemsBulkListAssignedTargetingOptions`
   on `GET /v4/advertisers/{advertiserId}/lineItems:bulkListAssignedTargetingOptions`. Pass the
   line item IDs and get every targeting type back in one paginated stream. This is the whole
   point of the skill: the per-type alternative,
   `advertisersLineItemsTargetingTypesAssignedTargetingOptionsList`, requires one call **per line
   item per targeting type**, and DV360 has dozens of targeting types. On an advertiser with a
   hundred line items that is thousands of calls against a 300-per-minute per-advertiser ceiling.
   Use the per-type method only when you are inspecting one known type on one known line item.

3. **Resolve the option IDs.** For each distinct `targetingType` you saw, call
   `targetingTypesTargetingOptionsList` on
   `GET /v4/targetingTypes/{targetingType}/targetingOptions` with the advertiser context, and
   build a `targetingOptionId → displayName` map once. Reuse the map across all line items rather
   than resolving per assignment. When you are looking for a specific value rather than the whole
   catalog, `targetingTypesTargetingOptionsSearch` is the cheaper call.

4. **Join and report.** Every assigned option now has a name, a targeting type, and the line item
   it belongs to. Inherited targeting — assigned at the advertiser or partner level rather than on
   the line item — will not appear here; read it separately from
   `advertisersTargetingTypesAssignedTargetingOptionsList` and
   `partnersTargetingTypesAssignedTargetingOptionsList`.

## Rules that will bite you

- **Reads consume the same quota as writes** against the 1500/minute project and 300/minute
  advertiser total limits. An audit is a read-heavy workload and is the most common way to hit
  `429 RESOURCE_EXHAUSTED` on this API.
- **Failed requests still consume quota.** Back off on 429; do not retry immediately.
- **Nothing tells you how much budget is left.** No `X-RateLimit-*`, no `RateLimit-*`, no
  `Retry-After`. Pace yourself.
- **Insertion-order and campaign-level targeting methods now return 404.** Google retired them
  effective 2026-02-23. If an old audit script 404s on IO-level targeting, that is the reason —
  see `lifecycle/google-display-video-360-lifecycle.yml`.
- **`pageToken` is opaque.** Pass it back verbatim; never construct or parse one.

## Errors

Branch on `error.status`. `403 PERMISSION_DENIED` means the Google Account has no DV360 user
profile on that advertiser, not that the token is bad — a bad token is `401 UNAUTHENTICATED`.
See `errors/google-display-video-360-problem-types.yml`.
