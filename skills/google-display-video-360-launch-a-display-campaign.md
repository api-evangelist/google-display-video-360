---
name: Launch a Display & Video 360 display campaign
description: Build the campaign → insertion order → line item chain under an advertiser and put it live, without tripping DV360's per-advertiser write quota or its updateMask trap.
api: openapi/google-display-video-360-api-openapi.yml
operations:
  - partnersList
  - advertisersList
  - advertisersCampaignsCreate
  - advertisersInsertionOrdersCreate
  - advertisersLineItemsCreate
  - advertisersLineItemsBulkEditAssignedTargetingOptions
  - advertisersLineItemsPatch
---

# Launch a display campaign

DV360 buys media at the **line item**. Everything above it is structure and budget. You cannot skip
a level: a line item must belong to an insertion order, which must belong to a campaign, which must
belong to an advertiser.

## Before you start

- Get a Google OAuth 2.0 access token on `https://www.googleapis.com/auth/display-video`.
  There is no API key. `https://www.googleapis.com/auth/display-video-mediaplanning` is enough if
  you only touch campaign entities.
- The token is not sufficient by itself. The Google Account behind it must have a DV360 user
  profile with **Standard** or **Admin** permission on the advertiser you are writing to, or every
  call below returns `403 PERMISSION_DENIED`.
- Base URL: `https://displayvideo.googleapis.com`.

## Steps

1. **Find the advertiser.** `partnersList` gives the partners this account can see.
   `advertisersList` takes a required `partnerId` and returns advertisers under it. Keep the
   `advertiserId` — it is a path segment for everything that follows.

2. **Create the campaign.** `advertisersCampaignsCreate` on
   `POST /v4/advertisers/{advertiserId}/campaigns`. Set `displayName`, `entityStatus`,
   `campaignGoal`, and `campaignFlight` with the run dates. Keep the returned `campaignId`.

3. **Create the insertion order.** `advertisersInsertionOrdersCreate`. Reference the
   `campaignId`, then set `pacing`, `budget` and `bidStrategy`. The insertion order is where money
   is allocated; get the budget segments right here rather than trying to patch them later.

4. **Create the line item.** `advertisersLineItemsCreate`. Reference the `insertionOrderId`, set
   `lineItemType`, `flight`, `budget`, `pacing`, `bidStrategy`. Create it with
   `entityStatus: ENTITY_STATUS_DRAFT` so it cannot start serving before you have targeted it.

5. **Target it in one call.** `advertisersLineItemsBulkEditAssignedTargetingOptions` on
   `POST /v4/advertisers/{advertiserId}/lineItems/{lineItemId}:bulkEditAssignedTargetingOptions`.
   Send every `createRequests` entry — geography, audience, inventory, brand safety — in a single
   request. Do **not** loop
   `advertisersLineItemsTargetingTypesAssignedTargetingOptionsCreate` once per option: one bulk
   call costs one write against quota, N single calls cost N, and the per-advertiser write budget
   is only 150 per minute.

6. **Go live.** `advertisersLineItemsPatch` with body `{"entityStatus":"ENTITY_STATUS_ACTIVE"}` and
   query `updateMask=entityStatus`.

## Rules that will bite you

- **`updateMask` is required on every PATCH.** A field in the body but not in the mask is silently
  ignored. A field in the mask but not in the body is **cleared**. A patch that returns 200 and
  changes nothing almost always means a missing mask entry.
- **There is no idempotency key.** Retrying step 2, 3 or 4 after a timeout creates a *second*
  entity. Before retrying a create, list with a `filter` on the `displayName` you were about to
  use and check it is not already there.
- **Quota is per minute and cannot be raised.** 1500 total / 700 write per project, 300 total /
  150 write per advertiser. Failed requests still consume quota, so a retry storm makes the
  problem worse. On `429 RESOURCE_EXHAUSTED`, back off — do not immediately retry.
- **No rate-limit headers.** Nothing in the response tells you how much budget is left. Count
  your own writes.
- **Check for warnings after you activate.** A line item can be `ENTITY_STATUS_ACTIVE` and still
  not serve. Call `advertisersLineItemsGet` and read `warningMessages` — a bad flight, no assigned
  creative, or incomplete targeting shows up there, not as an error on the write.

## Errors

Branch on `error.status`, never on `error.message`.
`400 INVALID_ARGUMENT` fix the request · `403 PERMISSION_DENIED` fix the DV360 user profile ·
`404 NOT_FOUND` wrong id or wrong parent · `409 ABORTED` concurrent write, retry shortly ·
`429 RESOURCE_EXHAUSTED` slow down · `500 INTERNAL` / `504 DEADLINE_EXCEEDED` retry with backoff.

See `errors/google-display-video-360-problem-types.yml` and
`conventions/google-display-video-360-conventions.yml`.
