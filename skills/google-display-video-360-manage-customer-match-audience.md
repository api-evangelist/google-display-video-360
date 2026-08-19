---
name: Create and maintain a Customer Match audience
description: Build a first-party audience and add or remove members, respecting the 5x write-quota multiplier on the two methods that do the work.
api: openapi/google-display-video-360-api-openapi.yml
operations:
  - firstPartyAndPartnerAudiencesList
  - firstPartyAndPartnerAudiencesCreate
  - firstPartyAndPartnerAudiencesEditCustomerMatchMembers
  - advertisersLineItemsBulkEditAssignedTargetingOptions
---

# Create and maintain a Customer Match audience

Customer Match audiences are first-party lists uploaded to DV360 and then targeted by line items.
Two of the methods involved are among the five most expensive calls in the entire API.

## Before you start

- OAuth 2.0 token on `https://www.googleapis.com/auth/display-video`, and a DV360 user profile
  with **Standard** or **Admin** permission on the advertiser.
- Customer Match has policy prerequisites that live outside the API — account eligibility and
  consent requirements for the data you upload. The API will accept the call; policy enforcement
  happens elsewhere. Confirm eligibility before building an integration around it.

## Steps

1. **Check whether it already exists.** `firstPartyAndPartnerAudiencesList` with a `filter` on
   the display name. Do this before every create — see the idempotency warning below.

2. **Create the audience.** `firstPartyAndPartnerAudiencesCreate` on
   `POST /v4/firstPartyAndPartnerAudiences`, with the `advertiserId` as a query parameter. Set
   `displayName`, `audienceType`, `membershipDurationDays` and `description`. Keep the returned
   `firstPartyAndPartnerAudienceId`.

3. **Add or remove members.** `firstPartyAndPartnerAudiencesEditCustomerMatchMembers` on
   `POST /v4/firstPartyAndPartnerAudiences/{id}:editCustomerMatchMembers`. Send
   `addedContactInfoList` / `removedContactInfoList` (or the mobile-device-ID equivalents) with
   the `advertiserId`. Batch as many members as the method accepts per call.

4. **Target it.** Assign the audience to a line item through
   `advertisersLineItemsBulkEditAssignedTargetingOptions` with
   `targetingType: TARGETING_TYPE_AUDIENCE_GROUP`.

## Rules that will bite you

- **These two methods cost 5x write quota.** `firstPartyAndPartnerAudiences.create` and
  `firstPartyAndPartnerAudiences.editCustomerMatchMembers` each consume **five** write units, not
  one. The project write budget is 700 per minute, so that is an effective ceiling of 140
  member-edit calls per minute for the whole project — shared with every other write your
  integration is doing. Batch members aggressively; a member-per-call loop will exhaust quota
  almost immediately.
- **There is no idempotency key.** A create that times out and is retried produces a duplicate
  audience, and duplicate audiences are not automatically merged. Always list-and-check first.
- **Infinite membership duration is gone.** Google removed the 10,000-day (effectively infinite)
  `membershipDurationDays` option effective 2025-04-07. Set a real finite duration.
- **Member edits are asynchronous in effect.** The call returns quickly; the audience size
  reflected in the UI and available for targeting updates later. Do not poll the audience size
  expecting an immediate change.

## Errors

`400 INVALID_ARGUMENT` on a member edit usually means malformed or unhashed contact info — read
`error.message` for the specific field. `429 RESOURCE_EXHAUSTED` here is most often the 5x
multiplier rather than raw call volume. See
`errors/google-display-video-360-problem-types.yml` and
`rate-limits/google-display-video-360-rate-limits.yml`.
