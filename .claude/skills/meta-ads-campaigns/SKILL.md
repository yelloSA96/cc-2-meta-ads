---
name: meta-ads-campaigns
description: Build a Meta Ads campaign (campaign -> ad set -> ad) end-to-end for a client ad account, staying PAUSED until the user explicitly confirms activation.
context: fork
disable-model-invocation: true
model: sonnet
effort: medium
---

# Meta Ads Campaign Creation

Full procedure lives in `.claude/requirements/ad-creation-sop.md` — read it first if this session hasn't already. This file is the operational version: it maps each SOP step to the exact `meta-ads` MCP tool calls and field values.

**Hard rule:** every entity (campaign, ad set, ad) is created in `PAUSED` state. Nothing is activated without an explicit go-ahead from the user at step 6. Never call `ads_activate_entity` before that checkpoint.

**Every `meta-ads` tool call requires `client_conversation_id`.** Generate one random 20-character alphanumeric id at the start of this skill run and reuse it on every call for the rest of the session.

---

## Step 0 — Select the client / ad account

1. `ads_get_ad_accounts` — list accounts, confirm with the user which client this campaign belongs to (by name, not just ID). Skip accounts where `is_ads_mcp_enabled` is false.
2. Note the client's kebab-case short code for naming (Step 1 below).
3. Resolve what the objective will need, before building anything:
   - `ads_get_ad_account_pages` (Page + `leadgen_tos_accepted` flag — required if objective is `leads`) or `ads_get_user_pages` if not scoped to this account yet.
   - `ads_get_ig_accounts` — only if placements will include Instagram / boosting IG content.
   - `ads_get_datasets` (ad_account_id scoped) — pixel/dataset, required for `sales`.

---

## Step 1 — Naming convention

Kebab-case, fixed field order, so entities are identifiable across clients at a glance.

```
Campaign: {client}-{objective}-{yyyymmdd}-{variant}
Ad Set:   {client}-{objective}-{targeting-summary}-{variant}
Ad:       {client}-{objective}-{creative-id}-{cta}
```

Example: `acme-sales-20260924-v1` / `acme-sales-wa-lookalike1pct-v1` / `acme-sales-creative-a-shop-now`

Rules: lowercase, hyphens only, no underscores/camelCase/spaces. `variant` increments (`v1`, `v2`, ...) — never reuse or overwrite a number.

---

## Step 2 — Define the campaign objective

Ask the user which objective this campaign serves, then map to the **ODAX outcome value** `ads_create_campaign` requires — legacy objectives (LINK_CLICKS, REACH, etc.) are rejected.

| SOP label | `objective` value | Typical use |
|---|---|---|
| `sales` | `OUTCOME_SALES` | Website purchases, pixel-tracked conversions |
| `leads` | `OUTCOME_LEADS` | Meta instant forms or external lead forms |
| `traffic` | `OUTCOME_TRAFFIC` | Link clicks, landing page views, engagement |
| `awareness` | `OUTCOME_AWARENESS` | Reach, brand awareness, video views |

- `sales`: confirm the pixel/dataset via `ads_get_datasets`, and the event to optimize for via `ads_pixel_event_read`. `OUTCOME_SALES` + `WEBSITE` destination **requires** `promoted_object` with a `pixel_id` on the ad set (Step 4) — the API rejects the ad set without it.
- `leads`: confirm native instant form vs. external form. If native and `optimization_goal` will be `LEAD_GENERATION`/`QUALITY_LEAD`, the promoted Page must have `leadgen_tos_accepted: true` (checked in Step 0) — if not, tell the user to accept ToS at https://www.facebook.com/legal/leadgen/tos before proceeding.

---

## Step 3 — Campaign budget structure + create the campaign

Ask: **CBO or ABO?**

- **CBO (recommended default, and what Meta recommends unless told otherwise)** — set `campaign_daily_budget` or `campaign_lifetime_budget` (cents) on `ads_create_campaign`. Meta shifts spend across ad sets automatically.
- **ABO** — only if the user explicitly asks for per-ad-set budget control. Leave `campaign_daily_budget`/`campaign_lifetime_budget`/`campaign_bid_strategy` **unset** on the campaign; budget goes on each ad set instead (Step 4). Setting any campaign-level budget field silently switches to CBO, so don't set them if the user asked for ABO.

Also capture: budget type (daily vs. lifetime — lifetime requires `end_time` downstream), start/stop time (`campaign_start_time`/`campaign_stop_time`, ISO 8601), and currency (must match the ad account's).

Call `ads_create_campaign`:
- `ad_account_id`, `campaign_name` (Step 1 convention), `objective` (Step 2 ODAX value), `buying_type: "AUCTION"` (always set explicitly; use `RESERVED` only if the user asks).
- CBO: `campaign_daily_budget` or `campaign_lifetime_budget`. Bid strategy defaults to `LOWEST_COST_WITHOUT_CAP`; only set `campaign_bid_strategy` if the user wants `COST_CAP`, `LOWEST_COST_WITH_BID_CAP`, or `LOWEST_COST_WITH_MIN_ROAS`.
- Campaign is created `PAUSED` automatically.
- The response includes `valid_optimization_goals` and `recommended_optimization_goal` for the chosen objective — carry these into Step 4.
- After creation, offer `ads_get_opportunity_score` on the ad account for best-practice recommendations.

---

## Step 4 — Build ad sets (audience + placement + optimization)

One ad set per audience segment to test/isolate independently — not one per creative.

For each ad set, ask fresh (no saved presets, per this SOP):

1. **Location** — country/region/state/radius → `targeting.geo_locations`.
2. **Age/gender** — `age_min`/`age_max` are soft suggestions under Advantage+ Audience (default on) unless the user wants a hard cap, in which case set `targeting_automation.advantage_audience: 0`.
3. **Audience type**:
   - Broad/interest — put `geo_locations` only for broad; only add `flexible_spec` interest IDs if the user supplies **real** numeric IDs (never invent one).
   - Custom audience — check `ads_get_ad_account_custom_audiences` if the user mentions retargeting/an existing list.
   - Lookalike — built from a custom audience or pixel data (same tool, `subtype_filter: LOOKALIKE`).
4. **Placements** — Advantage+ (default; omit `publisher_platforms` and all `*_positions` fields) unless the user explicitly restricts placements, in which case set `publisher_platforms` + matching position arrays inside `targeting`.
5. **`optimization_goal`** — pick from the campaign objective's compatible list (returned by `ads_create_campaign` as `valid_optimization_goals`/`recommended_optimization_goal`); when in doubt, use the recommended default.
6. **`billing_event`** — `IMPRESSIONS`, `LINK_CLICKS`, `POST_ENGAGEMENT`, or `VIDEO_VIEWS`, matched to the objective.
7. **`promoted_object`** — required whenever `optimization_goal` is `OFFSITE_CONVERSIONS`, `VALUE`, `LEAD_GENERATION`, `QUALITY_LEAD`, `APP_INSTALLS`, or `IN_APP_VALUE`. For `sales`: `{"pixel_id":"..."}` (add `custom_event_type` if optimizing a specific event). For `leads`: `{"page_id":"..."}` or pixel, per what the user confirmed in Step 2.
8. **Budget/schedule** — only set `daily_budget`/`lifetime_budget` here if ABO (Step 3); `lifetime_budget` requires `end_time`.

Call `ads_create_ad_set`: `ad_account_id`, `campaign_id`, `ad_set_name` (Step 1 convention), `billing_event`, `optimization_goal`, `targeting` (JSON string). Ad set is created `PAUSED` automatically.

Repeat once per audience segment.

---

## Step 5 — Build ads (creative + copy + CTA)

User supplies fresh assets/copy per ad each time — no creative-library reuse in this SOP.

1. **Upload asset(s)**: `ads_creative_upload_media` with `upload_source: URL` + `media_type` + public `media_url` (preferred path when the client doesn't support interactive MCP Apps), or `upload_source: LOCAL_FILE` if it does. For images specifically, `ads_creative_upload_local_image` + `ads_finalize_local_ad_image_upload` is the local-file path. Get an `image_hash` (images) or `video_id` (video) back.
2. **Copy**: primary text (`message`), `headline`, `description` as supplied by the user.
3. **Destination**: `link_url` for `sales`/`traffic`; for native `leads`, confirm the instant form is set up (page-based, no `link_url` needed).
4. **CTA**: `call_to_action_type`, matched to the objective (e.g. `SHOP_NOW`, `LEARN_MORE`, `SIGN_UP`, `BOOK_NOW`) — pick from what the user's intent maps to, don't guess a mismatched one.
5. Call `ads_create_creative`: `ad_account_id`, `page_id` (required — from Step 0), plus exactly one media source (`image_hash`/`image_url` for images; `video_id` + a thumbnail `image_hash`/`image_url` for video) and `link_url`, `message`, `headline`, `description`, `call_to_action_type` as gathered above. Note: `page_id` is required inside every path — its omission is the most common rejection ("Facebook Page is Missing").
6. Call `ads_create_ad`: `ad_account_id`, `ad_set_id`, `ad_name` (Step 1 convention), `creative: {"creative_id": "<id from step 5>"}`. Ad is created `PAUSED` automatically.

Repeat per creative variant (`-v1`, `-v2`, ...) for A/B testing within an ad set.

---

## Step 6 — Review checkpoint (mandatory pause before activation)

Before any `ads_activate_entity` call:

1. Summarize the full built structure: campaign → ad sets → ads, with names, budget, targeting, and creative/CTA for each.
2. Optionally preview: `ads_get_ad_preview` with `ad_id` (or `creative_id` if the ad hasn't fully materialized yet). **Always include the returned `preview_url` verbatim** in your reply so the user can open it.
3. Get explicit confirmation to proceed.
4. Only after confirmation, activate top-down: campaign → ad set → ad, via `ads_activate_entity` (`entity_type`: `campaign`/`ad_set`/`ad`). Activating a parent does not auto-activate children — each level needs its own call, and all levels must be `ACTIVE` for the ad to actually deliver.
5. If the response status is `PUBLISHING`, report it as handed off, not live — the publisher still runs further checks asynchronously.

If the user requests changes at this checkpoint: use `ads_update_entity` with the correct **API field names** (`name`, `daily_budget`, `lifetime_budget`, `status`, etc. — not the `ads_create_*` argument names) for campaign/ad-set edits, or build a new creative + new ad via `ads_create_creative`/`ads_create_ad` for creative changes (ad creatives are immutable — there is no update path for media/copy/CTA). Re-summarize before asking again. Do not activate on an unconfirmed change.

---

## Open items / to confirm per client (fill in once, reuse)

- Client short codes for the naming convention (Step 1).
- Default currency and typical budget range per client, if any.
- Whether each client's ad account has a pixel already connected (relevant for `sales`).
