# Build prompt: Launchpoint public Buyer API (2026-07)

Build the public, API-key-authenticated Buyer API described by the OpenAPI spec in the docs
repo at `api-reference/openapi.json` (base URL `https://api.launchpoint.sh/2026-07`). Work in
`/Users/adam/Developer/Launchpoint/launchpoint-backend`.

## Guiding principle: one shared core, a thin second adapter — do NOT duplicate logic

The dashboard routes under `src/app/api/buy/*` are already thin: they authenticate, parse
input, call a service in `src/lib/*`, and serialize. Examples to model on and REUSE:
- `GET /api/buy/campaigns` -> `getCampaignsList()` (src/lib/economics/campaigns/projections)
- `POST /api/buy/campaigns` -> `mutateCampaign()` (src/lib/campaigns/editor/mutate-campaign.ts)
- offers/videos -> `getCampaigns()` (src/lib/data/campaigns.server), `getVideosForOffer()`
- auth wrappers + principal: `withBuyClient`/`withAdmin` (src/lib/auth/auth-service.ts),
  `AuthUser` (src/lib/auth/auth-user.ts) with `client` + `clientInfo` (the brand/agency).

The public API is a SECOND adapter over those same services. Add only three public-only
layers; never copy business logic out of `src/lib`.

1. AUTH ADAPTER — `withApiKey(handler, { requiredScope })`, parallel to `withBuyClient`.
   Reads `Authorization: Bearer <key>`, verifies it, and builds the SAME AuthUser/clientInfo
   object the lib services expect, so every downstream service works unchanged. Enforce
   scopes and agency-vs-brand visibility here (see below). If both wrappers can't cleanly
   produce one identical principal type, refactor the principal into a small shared builder
   that each wrapper calls — that shared builder is the only new coupling allowed.

2. PRESENTER LAYER — `src/lib/api/public/2026-07/serializers/*`. The ONLY place internal
   domain objects become the external contract. All the external vocabulary lives here and
   nowhere else. Keep it pure (domain object in, spec DTO out) and versioned by folder.

3. INGRESS SCHEMAS — `src/lib/api/public/2026-07/schemas/*`. Zod parsers matching the
   OpenAPI request bodies/params; parse once at the boundary, map external->internal, hand
   trusted typed args to the services. No `unknown` past this line.

## Routing / versioning
- Put routes at `src/app/api/public/2026-07/...` mirroring the spec paths (campaigns, tasks,
  creators, submissions, posts, metrics, targeting).
- The public host `api.launchpoint.sh/2026-07/*` maps to `/api/public/2026-07/*`. Prefer a
  reverse-proxy/CDN rewrite; if done in-app, add a root `middleware.ts` that rewrites the
  subdomain — keep it to routing only.

## New infrastructure to build (none exists today — verify with a search first)
- `api_keys` table: id, client_id (owning brand/agency), key_hash (store only a hash;
  show a `lp_...` prefix once at creation), scopes text[], created_at, last_used_at,
  revoked_at, expires_at. Lookup = hash the presented key, match un-revoked/un-expired.
  ASK THE HUMAN to run any migration — do not apply DB changes yourself.
- Scopes: `campaigns:read`, `campaigns:write` (implies read), `approvals:write`. Enforce in
  `withApiKey`; on failure return 403 `missing_scope`.
- Agency vs brand keys: a brand key is scoped to one client_id and sees only its campaigns.
  An agency key resolves the managed brand from `brand_id` (body/query) or the resource's
  owner; if the key doesn't manage that brand, 403 `brand_not_managed`; unknown/foreign
  resource -> 404. Reuse existing agency-management/visibility helpers
  (src/lib/auth/dashboard-access-context.ts and the money-hiding rules) rather than
  reinventing them.
- Idempotency: header-based helper. On `Idempotency-Key` (Create Campaign, Add Task), store
  (key, request-hash, response) for 24h; replay returns the stored response with
  `Idempotent-Replay: true`; same key + different body -> 409 `idempotency_conflict`. Build
  on the existing UNIQUE-index idempotency pattern used in payouts.
- Rate limiting: 120 req/min per key (burst 20/s). Copy the sliding-window UPSERT approach
  from src/lib/conversion-events/public-rate-limit.ts, keyed by api_key id. On exceed, 429
  `rate_limited` with `Retry-After`; always set `X-RateLimit-Limit`/`X-RateLimit-Remaining`.
- Outbound webhooks (new): a delivery system that signs the raw JSON body with HMAC-SHA256
  (`X-Launchpoint-Signature`), retries with exponential backoff up to 24h, at-least-once.
  Use Trigger.dev for the delivery/retry queue. Events: campaign.status_changed,
  creator.applied/approved/rejected, submission.created/approved/rejected,
  post.created/metrics_updated. Fire them from the domain events the dashboard already
  emits, so API and dashboard actions both notify. One webhook URL + secret per account.

## Cross-cutting contract details (match the spec exactly)
- Error shape `{ error: <http status>, message, code }`; codes: invalid_request,
  unauthenticated, missing_scope, brand_not_managed, not_found, invalid_state_transition,
  field_locked_for_status, already_decided, already_reviewed, idempotency_conflict,
  validation_failed, rate_limited.
- Pagination: `{ data, has_more, next_cursor }`, opaque cursor via `starting_after`,
  `limit` 1-100 default 25, newest first.

## Presenter projections (internal -> external) — implement in the serializer layer
- Campaign status: from offers.state + offers.publish_approval_status. draft+pending ->
  `pending_approval`; state complete -> `completed`; else state maps directly. status_detail
  from auto_paused / publish_denied.
- Task status: active/archived only (no other external value).
- Submission status -> { in_review, approved, rejected, failed, post_deleted } + a
  `failure_reason` string. Proposed map (confirm against where creator money is committed):
    in_review    <- pending, validating, updated, active
    approved     <- completed, paid, tracking
    rejected     <- rejected
    failed       <- validation_failed, error   (set failure_reason)
    post_deleted <- post_deleted
- Post status -> { live, deleted } from the social_posts validation status
  (passed -> live, deleted -> deleted). Deleted posts stay listable.
- Media: derive submission media_type video|carousel from video_url vs carousel_media_urls;
  post.is_story is inherited from the submission.
- Creator identity: expose a STABLE opaque `creator_id` (`cr_...`) + `username`. The backend
  only has bare Firebase UIDs and no creator username column. Build a stable-external-ID
  mapping (mint + persist a `cr_` id per creator) and source username from the creator's
  primary social handle (social_accounts.username). Never expose Firebase UIDs.
- Targeting: property is an opaque id (backend uses both slugs and UUIDs — expose whatever
  List Targeting Properties returns, consistently). Operators IN/NOT_IN, AND/OR groups.
  number properties take "min-max" range strings.

## Metrics endpoint (GET /campaigns/{id}/metrics) — align to the new views
Back it with the unified performance views from commit f3188e6af
(mv_social_post_daily_metrics, src/lib/analytics/performance/*). Contract shape is schema
`CampaignMetrics`.
- ENGAGEMENT: the spec's engagement includes saves. Today
  src/lib/analytics/performance/types.ts defines engagement = likes+comments+shares and
  excludes saves from PERFORMANCE_METRIC_KEYS even though the MV has total_saves/saves_delta.
  Add saves as a first-class aggregate and compute engagement_rate =
  (likes+comments+shares+saves)/views (0 when views=0, be consistent).
- effective_cpm_cents = round(spent_cents/views*1000), null when views=0. Not currently
  exposed by the perf endpoint (internal payout math only) — compute in the read-model.
- group_by=platform: back with the series platform filter. group_by=task: the new views
  filter by platform/offer/creator-group but NOT task — add per-task attribution via
  offer_videos.task_id, or report back that it isn't feasible and we drop group_by=task.
- SPEND IS CAMPAIGN-LEVEL ONLY (and per-creator for List Creators). Do not attempt
  per-platform/per-group spend — payouts aren't attributable to a platform (see spend.ts).
- funnel counts come from the campaign workflow queries, not the perf views.
- attribution: the endpoint takes no attribution param; default to lifetime ("upload") totals
  for this all-time report and document it.

## Suggested build order
1. api_keys table + withApiKey + scope/visibility enforcement (reuse AuthUser principal).
2. Read endpoints (list/get campaigns, tasks, creators, submissions, posts, targeting) as
   serializer-wrapped calls to existing lib services. Prove the "thin adapter" pattern.
3. Write + lifecycle endpoints (create/update/delete, publish/pause/resume/complete, tasks,
   approvals) over mutateCampaign and the existing approval services.
4. Metrics endpoint alignment.
5. Idempotency + rate-limit middleware.
6. Outbound webhooks.

Deliver thin route handlers + serializers + schemas that reuse src/lib services verbatim.
Cents are integers. Verify numbers against a real campaign's dashboard values. Ask the human
to run any DB migration.

---

# RESOLVED — post-mapping corrections & authoritative build plan (added 2026-07-07)

Everything above is the original intent. This section is the authoritative plan after a full
mapping of the spec against the current backend (worktree off `main`, commit `f3188e6af` present).
**Where this section conflicts with the text above, this section wins.**

## Confirmed decisions

1. **Sequencing.** Correct the OpenAPI spec + docs first, get sign-off, then implement.
2. **`DELETE /campaigns/{id}` = SOFT delete.** Mark the offer deleted/hidden and keep the row +
   children; guard to `draft`/`pending_approval` only (anything live -> 409
   `invalid_state_transition`). No hard deletes.
3. **Agency keys use the SAME money rules as the dashboard.** An agency key acting on a managed
   brand sees the brand's campaign spend but the brand's saved card is hidden. This falls out for
   free from the shared principal (see Auth below) + the existing money-hiding helpers.
4. **Public `creator_id` (`cr_...`) is GLOBAL per creator** — one stable id per creator, same value
   for every brand/agency.

## Corrections to the text above (these were wrong)

- **`getVideosForOffer`** lives in `src/lib/campaigns/videos/listing.ts` (~:2890), NOT
  `src/lib/data/campaigns.server` (that file only exports `getCampaigns`).
- **"spend.ts"** means `src/lib/analytics/performance/spend.ts`.
- **Submission status map targeted the wrong table.** The Submission object the endpoints describe
  (has `feedback`, `revision_count`, `media_type`, approve/reject-with-feedback) is `offer_videos`
  (schema ~:2186), which has **NO status column** — review state is derived from
  `approved_at`/`rejected_at` + media presence via `src/lib/campaigns/videos/video-state.ts`.
  `offer_submissions.status` (schema ~:2464) is a **post-tracking** record keyed by (video_id,
  platform); one video has 0..n of them. Correct map:
  - `in_review`  <- offer_video submitted, not yet approved/rejected (video-state)
  - `approved`   <- offer_video `approved_at` set
  - `rejected`   <- offer_video `rejected_at` set (a revision returns it to `in_review`)
  - `failed`     <- fall through to offer_submissions.status in {validation_failed, error}; set `failure_reason`
  - `post_deleted` <- offer_submissions.status = post_deleted
  The serializer owns this as a declarative map (no if/else ladder).
- **`group_by=task` IS feasible** — `offer_videos.task_id` exists (NOT NULL FK) and `ov` is already
  joined in `ELIGIBLE_POSTS_FROM` (`src/lib/analytics/performance/eligible-scope.ts`). Add a
  `task_id` filter + `GROUP BY ov.task_id`; `group_label` = `offer_tasks.title`. Do not drop it.

## Migrations to hand to the human (write the Drizzle schema + generate SQL; do NOT apply)

- `api_keys` (id, client_id -> clients.id, key_hash UNIQUE, key_prefix, scopes text[], created_at,
  last_used_at, revoked_at, expires_at). Lookup = sha256(presented key) where not revoked/expired.
  Agency-ness is derived from `clients.client_type='agency'`, never a key column.
- `api_idempotency_keys` (key, api_key_id, request_hash, response_status, response_body jsonb,
  created_at, expires_at; UNIQUE(api_key_id, key)). Copy the `onConflictDoNothing` technique from
  `createWalletTransactionIdempotent` (`src/lib/wallet/queries.ts:212`) — only the technique, the
  storage is new.
- `api_key_rate_limits` — two windows per key (60s cap 120, 1s cap 20). Copy the atomic
  INSERT..ON CONFLICT DO UPDATE from `src/lib/conversion-events/public-rate-limit.ts`.
- `creator_external_ids` (cr_id text pk, uid varchar UNIQUE, created_at). Mint+persist on first
  exposure; never emit Firebase UIDs.
- `webhook_endpoints` (client_id, url, secret, created_at, disabled_at) + optional
  `webhook_deliveries` (status, attempts, next_attempt_at) for at-least-once.

## Authoritative reuse map (operation -> existing service : file)

- listCampaigns -> `getCampaignsList` (`src/lib/economics/campaigns/projections/list.ts:206`);
  map external statuses -> internal, add `created_after/before`, pass `limit` explicitly (service
  default is 20, spec default 25), and handle `pending_approval` (state=draft + publish_approval_status='pending').
- getCampaign -> `getCampaignsList` single, or `getCampaignEditorSnapshot` + `fetchTasksWithRequirements`
  + `getTargetingRules` for the embedded tasks/targeting.
- createCampaign -> `mutateCampaign` intent `save_draft` (`src/lib/campaigns/editor/mutate-campaign.ts:2358`).
- updateCampaign -> `mutateCampaign` with **load-merge**: load current snapshot, deep-merge the
  external PATCH over it (buildOfferValues writes scalars WHOLESALE — omitting a field would blank
  it), pass the loaded `updated_at` to use the optimistic lock. Only video_examples/links/targeting
  are legitimately replace-wholesale.
- deleteCampaign -> NEW `deleteDraftCampaign(campaignId, access)` (soft; see decision 2).
- publishCampaign -> `mutateCampaign` intent `publish`.
- pause/resume/complete -> `applyCampaignTransition` (`src/lib/campaigns/transitions.ts:155`).
  **resume adds a billing guard**: 409 `invalid_state_transition` when
  `auto_pause_reason in {billing_past_due, payment_failed}` (the transition service has no such guard today).
- listTasks -> `fetchTasksWithRequirements` (`src/lib/campaigns/task-management.ts:59`).
- create/update/archiveTask -> `mutateCampaign` **whole-array read-mutate-resave** (there is no
  per-task service; a task absent from the array is HARD-DELETED). Create = append; update =
  field-merge preserving omitted fields; archive = set `brief_stage='archived'` and KEEP the row.
  Reject writes on Canvas campaigns (check `offers.is_canvas`/`campaign_mode`) with 409/422.
- listCreators -> `getCampaignCreatorRowsPage` (`src/lib/campaigns/videos/listing.ts:1377`) + a NEW
  read model for ContentCounts, live-post count, split engagement (likes/comments/shares/saves),
  per-creator spend, and username via `social_accounts.username`. Not a 1:1 reuse.
- approve/rejectCreator -> `handleCreatorApproval` (`src/lib/campaigns/videos/creator-approval.ts:938`).
- listSubmissions/getSubmission -> NEW campaign-scoped query over `offer_videos` (+ video-state);
  reuse the SELECT shape from `getRecentSubmissions` (`src/lib/submissions/recent.ts:187`).
- approveSubmission -> `maybeRecordAgencyVideoReview` then `approveVideoWithSideEffects`
  (`src/lib/campaigns/videos/approval.ts:484`). Commits creator money -> cannot undo (`already_reviewed`).
- rejectSubmission -> `maybeRecordAgencyVideoReview` then `rejectVideoWithNotification`
  (`src/lib/campaigns/videos/rejection.ts:34`). It requires non-empty feedback; the adapter supplies
  a default note when the caller omits `feedback` (spec keeps it optional).
- listPosts -> NEW campaign-scoped `social_posts` query; pick one row per (parent, platform) via
  `selectCanonicalSocialPostForParentPlatform`. **Liveness: use `validation_status`** (passed->live,
  deleted->deleted); rows still in pending/validating/failed/cancelled/needs_review are NOT posts yet.
- getCampaignMetrics -> NEW lifetime read model over the perf views (force attribution='upload',
  drop getPerformance's previous-window comparison) + funnel from the campaign workflow queries.
  ADD: `total_saves` aggregate, `engagement_rate=(likes+comments+shares+saves)/views` (0 at 0 views),
  `effective_cpm_cents=round(spent_cents/views*1000)` (null at 0 views), `group_by=task`.
- listTargetingProperties -> `fetchPropertiesByIds` (`src/lib/properties/queries.ts:8`) over the
  fixed public catalog (the 6 UUID properties + `age`/`followers` that `/api/buy/properties` exposes).
- listTargetingPropertyValues -> NEW paginated+search query over `literals`
  (WHERE property_id, ILIKE string_value, ORDER BY string_value, LIMIT+cursor); 404 for `number` kinds.

## Presenter/serializer resolutions (external vocabulary lives ONLY here)

- **Campaign status**: presenter-only `deriveExternalCampaignStatus(state, publish_approval_status)` —
  do NOT reuse `computeCampaignStatus` (`src/lib/campaigns/utils.ts:38`, no `pending_approval`, uses
  `complete` not `completed`). Map: (draft|null)+pending -> `pending_approval`; complete -> `completed`;
  else state passes through (null state = active).
- **status_detail**: real columns are `offers.auto_pause_reason` (enum of 6:
  budget_exhausted, repurpose_media_exhausted, billing_past_due, creator_approval_waitlist_full,
  approval_inactivity, payment_failed) and `offers.publish_rejection_reason` +
  `publish_reviewed_at`. Map `billing_past_due|payment_failed -> reason "billing_issue"`, the other
  four -> `"other"`; `occurred_at` best-effort `offers.updated_at` (auto-pause) / `publish_reviewed_at`
  (denial). There is no dedicated auto-pause timestamp column.
- **spent_cents** (Campaign + metrics.spend): use ONE source — the analytics `spend.ts` path
  (`src/lib/analytics/performance/spend.ts`), because spend-visibility money-hiding is already wired
  to it. Verify the number against a real campaign's dashboard "spend" before shipping. Honor
  `brand_money_hidden` for brand-key callers.
- **invite_code**: not on offers — read the oldest `user_custom_codes` row per offer, only for
  `unlisted`; on write pass code + `invite_only=(visibility==='unlisted')` to `mutateCampaign`
  (there is no direct `offers.visibility` write; mutate recomputes it).
- **Task.autoreview_prompt**: not a column — it is the single `offer_task_requirements.requirement`
  row when `submission_review_mode='autoreview'`. Ingress upserts/clears that one row.
- **Task.status/archive**: `brief_stage` ('archived' = archived). `post_approval_mode` now includes
  `always` (auto-approve) in the spec; ingress accepts all four.
- **revision_count** = `MAX(offer_video_revisions.revision_number) - 1` (re-submissions after a
  rejection; 0 for a first submission), consistent with the spec text.
- **funnel**: `creators_applied` = pending applications
  (`creator_offer_task_slots.state='pending'` / `creator_offer_approval_decisions.state='pending'`),
  NOT `creatorsAccepted`. `creators_approved<-creatorsApproved`,
  `submissions_in_review<-awaitingApproval`, `submissions_approved<-approvals`, `posts<-submissions`.
- **Targeting**: echo property `id` and value tokens BYTE-FOR-BYTE both directions (ids are a mix of
  UUIDs and slugs per `src/lib/targeting/property-ids.ts`; values are `literals.id` UUIDs). On write,
  validate each property id is a known UUID in `properties` or a recognized slug, else 422, so a
  stale id can't persist an un-matchable condition. Number properties take one `"min-max"` string.

## Auth (the one shared coupling)

Extract the identity-agnostic tail of `getAuthUserFromSessionCookie`
(`src/lib/auth/auth-service.ts` ~:481-523) into `buildClientPrincipal(...)` and have the session,
token, AND `withApiKey` wrappers call it; export `rowToClientInfo`. For a key, build a synthetic
`clientUser {client_id: key.client_id, role: null}` (role MUST be null so keys never get the
platform-admin lens), run `resolveEffectiveClientUser` (pass the agency key's `brand_id` as
`simulatedClientId`), then `buildClientPrincipal`. Enforce scopes in `withApiKey`
(`campaigns:write` implies `campaigns:read`) -> 403 `missing_scope`. Resource-owner authorization is
two-tier: brand_id at principal-build time; the campaign_id owner check happens inside each handler
via `resolveCampaignWriteContext` + `canAgencyManageBrand`. Pre-check `brand_id` with
`getAgencyBrandIds` before `getCampaignsList` (an unmanaged brand silently yields an empty page,
not a 403).

## Build order

1. Migrations (human runs): api_keys, api_idempotency_keys, api_key_rate_limits,
   creator_external_ids, webhook_endpoints (+deliveries).
2. Shared principal: extract `buildClientPrincipal`, export `rowToClientInfo`, refactor wrappers
   (no behavior change).
3. `withApiKey` + scopes + agency/brand visibility + error/pagination/creator-id helpers.
4. Read endpoints (serializer-wrapped): campaigns, tasks, targeting, creators (new read model),
   submissions (offer_videos + video-state), posts.
5. Metrics endpoint (lifetime read model: saves, engagement_rate, effective_cpm, funnel pending
   count, group_by task/platform, spend-visibility).
6. Write + lifecycle: create/update(load-merge)/delete(soft), publish/pause/resume(billing guard)/
   complete, tasks (whole-array), approvals (agency-first-line, money-commit 409 state machine).
7. Idempotency + rate-limit wrapping the write routes.
8. Outbound webhooks: fire-and-forget enqueue at ~7 shared write points (no event bus exists;
   mirror the inline Slack notify) + HMAC delivery task with 24h retry.
9. Subdomain routing (CDN rewrite preferred; else root `middleware.ts`).
10. Verify every cents/spend/metrics number against a real campaign's dashboard values.
