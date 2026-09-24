# Ad Creation SOP

Standard operating procedure for building a Meta Ads campaign end-to-end via the `meta-ads` MCP tools, for solo/freelance work across multiple client ad accounts. Scope: campaign creation through activation. Post-launch monitoring/reporting is out of scope for this SOP.

Every campaign is built **paused** at every level (campaign, ad set, ad) and only activated after an explicit go-ahead — never auto-launch.

---

## 0. Select the client / ad account

Before anything else, confirm which client this campaign is for.

1. Call `ads_get_ad_accounts` to list available ad accounts.
2. Confirm with the user which account/business this campaign belongs to (by client name, not just account ID).
3. Note the client's short code (see naming convention below) — used in every name from here on.
4. If the objective will need a Page, Instagram account, pixel, or custom audiences, resolve those now too:
   - `ads_get_ad_account_pages` / `ads_get_user_pages`
   - `ads_get_ig_accounts` (if boosting IG content or running IG placements)
   - `ads_get_datasets` (pixel, for conversion objectives)

---

## 1. Naming convention

All entity names use **kebab-case** (hyphen-separated) and follow a fixed field order so campaigns/ad sets/ads are identifiable at a glance across clients, without opening each one.

```
Campaign: {client}-{objective}-{yyyymmdd}-{variant}
Ad Set:   {client}-{objective}-{targeting-summary}-{variant}
Ad:       {client}-{objective}-{creative-id}-{cta}
```

Examples:
```
Campaign: acme-sales-20260924-v1
Ad Set:   acme-sales-wa-lookalike1pct-v1
Ad:       acme-sales-creative-a-shop-now
```

Rules:
- `client`: short lowercase kebab-case code for the client (agree on this once per client, reuse consistently).
- `objective`: one of `sales`, `leads`, `traffic`, `awareness` (see step 2).
- `variant`: `v1`, `v2`, ... — increment for A/B or iteration, never overwrite/rename an existing entity's variant number.
- No spaces, no camelCase, no underscores, no special characters beyond `-`.

---

## 2. Define the campaign objective

Ask the user which objective this campaign serves. Supported objectives:

| Objective label | Meta objective | Typical use |
|---|---|---|
| `sales` | Conversions / Sales | Website purchases, pixel-tracked conversion events |
| `leads` | Lead generation | On-platform Meta instant forms (no landing page required) |
| `traffic` | Traffic / Engagement | Link clicks, landing page views, post engagement |
| `awareness` | Awareness / Reach | Brand awareness, reach, video views |

For `sales`, confirm the pixel/dataset and the conversion event to optimize for (`ads_get_datasets`, `ads_pixel_event_read`) before proceeding.
For `leads`, confirm whether to use Meta's native instant form (no destination URL needed) vs. sending traffic to an external form.

---

## 3. Determine campaign budget structure

Ask: **CBO or ABO?**

- **CBO (Campaign Budget Optimization)** — one budget set at the campaign level; Meta's delivery system shifts spend across ad sets automatically toward best-performing audiences.
  - Pro: generally best ROI/efficiency, less manual rebalancing.
  - Con: less control over how much any single ad set gets — a weak ad set can get starved.
- **ABO (Ad Set Budget Optimization)** — fixed budget set per ad set.
  - Pro: guaranteed, even spend/control across each audience (e.g. testing multiple states/segments fairly).
  - Con: more manual work to rebalance based on performance; can leave spend on underperforming ad sets.

Also capture:
- Total daily or lifetime budget (confirm currency matches the ad account).
- Budget type: daily vs. lifetime.
- Schedule: start date/time, and end date/time if lifetime budget or a fixed flight.

Use `ads_create_campaign` with `status: PAUSED`.

---

## 4. Build ad sets (audience + placement + optimization)

Ad sets are children of the campaign and control **who** sees the ads and **how** delivery is optimized. Each distinct audience you want to test or isolate = one ad set.

> Analogy: one ad set per audience segment you want to control/measure independently — e.g. one per state (WA, SA), or one per lookalike vs. interest-based audience — not one ad set per creative.

For each ad set, ask fresh (per this SOP's decision — no saved presets):

1. **Location** — country/region/state/radius.
2. **Age range** and **gender** (if relevant to the objective).
3. **Audience type** — pick one or combine:
   - Interest/behavior targeting (detailed targeting).
   - Custom audience (check `ads_get_ad_account_custom_audiences` if the user mentions retargeting/an existing list).
   - Lookalike audience (built from a custom audience or pixel data).
4. **Placements** — automatic (recommended default) or manual platform/placement selection.
5. **Optimization & billing event** — tied to the objective chosen in step 2 (e.g. conversions, link clicks, reach, lead form submissions).
6. **Budget** (only if ABO — see step 3) and **schedule** if it differs from the campaign-level schedule.

Use `ads_create_ad_set` with `status: PAUSED`, parented to the campaign from step 3.

Repeat this step once per audience segment the campaign needs.

---

## 5. Build ads (creative + copy + CTA)

Ads are children of an ad set and carry the actual creative shown to the audience: asset(s), copy, and call-to-action.

For each ad, the user provides fresh assets/copy each time (no creative library reuse in this SOP):

1. **Asset(s)** — image or video file(s) the user hands over for this ad.
   - Upload via `ads_creative_upload_local_image` / `ads_creative_upload_media`, then `ads_finalize_local_ad_image_upload` as needed.
2. **Primary text / copy** — main ad copy, provided by the user.
3. **Headline** and **description** (if the placement/format uses them).
4. **Destination**:
   - `sales`/`traffic`: landing page URL.
   - `leads` (native): instant form (confirm the form exists or needs creating).
5. **Call to action (CTA)** button — e.g. Shop Now, Learn More, Sign Up — matched to the objective.
6. Build the creative with `ads_create_creative`, then the ad with `ads_create_ad`, both with `status: PAUSED`, parented to the ad set from step 4.

Repeat per creative variant (`-v1`, `-v2`, ...) if A/B testing copy/creative within an ad set.

---

## 6. Review checkpoint (mandatory pause before activation)

Before activating anything:

1. Summarize the full built structure for the user: campaign → ad sets → ads, with names, budget, targeting, and creative/CTA for each.
2. Optionally pull a preview: `ads_get_ad_preview` / `ads_get_ad_preview_screenshot`.
3. Get explicit confirmation from the user to proceed.
4. Only after confirmation, activate in order: campaign → ad sets → ads, via `ads_activate_entity`.

If the user requests changes at this checkpoint, update the relevant entity (`ads_update_entity` / `ads_creative_update`) and re-summarize before asking again — do not activate on an unconfirmed change.

---

## Open items / to confirm per client (fill in once, reuse)

- Client short codes for the naming convention (step 1).
- Default currency and typical budget range per client, if any.
- Whether each client's ad account has a pixel already connected (relevant for `sales` objective).
