# Fadior Fork Modifications

This is a Fadior-internal fork of [`AdsMCP/tiktok-ads-mcp-server`](https://github.com/AdsMCP/tiktok-ads-mcp-server).
Upstream ships a high-quality TikTok Ads MCP server, but at the time of forking
(2026-04-29 Beijing time) it was **read-mostly**: 4 write methods existed in the
client but were not registered as MCP tools, and 2 tool layers returned mock
placeholders instead of calling the live API. This fork closes those gaps and
adds 6 additional write methods needed for end-to-end campaign management from
inside an LLM agent.

All changes live under `src/tiktok_ads_mcp/`.

## 1. OAuth redirect URL — Fadior-aligned default + env override

**File:** `src/tiktok_ads_mcp/oauth_simple.py`

- Default `redirect_uri` changed from `"https://adsmcp.com"` to
  `"https://www.fadiorhome.com/"` (the URL Leo registered when applying for the
  TikTok Marketing API app).
- Added `TIKTOK_OAUTH_REDIRECT_URI` environment-variable override, with the
  resolution order: explicit arg → env var → Fadior default.
- No local listener was added — OAuth remains a one-shot flow where the user
  pastes the `code` query-string parameter back into the
  `tiktok_ads_complete_auth` tool.

## 2. Register the 4 already-implemented write tools

**File:** `src/tiktok_ads_mcp/server.py`

Upstream `tiktok_client.py` already had `create_campaign`, `create_adgroup`,
`upload_image`, and `create_report_task`, and `call_tool()` already routed them.
But `list_tools()` did not advertise them, so MCP clients never saw them.
This fork registers `Tool` objects for:

- `tiktok_ads_create_campaign`
- `tiktok_ads_create_adgroup`
- `tiktok_ads_upload_image`
- `tiktok_ads_generate_report`

…with `inputSchema` shapes that match the underlying tool-class signatures.

## 3. Add 6 new write methods + tool wrappers

**File:** `src/tiktok_ads_mcp/tiktok_client.py`

Appended six new async methods on `TikTokAdsClient`:

| Method | Endpoint |
| --- | --- |
| `update_campaign(campaign_data)` | `POST campaign/update/` |
| `update_adgroup(adgroup_data)` | `POST adgroup/update/` |
| `update_status(level, ids, status)` | `POST {level}/status/update/` |
| `create_ad(ad_data)` | `POST ad/create/` |
| `upload_video(video_path, upload_type)` | `POST file/video/ad/upload/` (multipart, mirrors `upload_image`) |
| `pause_campaign(campaign_id)` | thin wrapper around `update_status('campaign', [id], 'DISABLE')` |

A seventh helper `create_custom_audience(audience_data)` was also added
(`POST dmp/custom_audience/create/`) so audience tools can call a real endpoint
instead of the previous mock.

**File:** `src/tiktok_ads_mcp/tools/campaign_tools.py`

Added wrapper methods (`update_campaign`, `update_adgroup`, `update_status`,
`pause_campaign`, `create_ad`) so MCP tools have a clean callable surface.

**File:** `src/tiktok_ads_mcp/tools/creative_tools.py`

Added `upload_video` wrapper with file-existence / extension / size validation
parallel to the existing `upload_image` helper.

**File:** `src/tiktok_ads_mcp/server.py`

Registered the 6 new MCP tools and added matching `call_tool()` routing
branches:

- `tiktok_ads_update_campaign`
- `tiktok_ads_pause_campaign`
- `tiktok_ads_update_adgroup`
- `tiktok_ads_update_status`
- `tiktok_ads_create_ad`
- `tiktok_ads_upload_video`

## 4. Replace the 2 mock placeholders with real API calls

**File:** `src/tiktok_ads_mcp/tools/audience_tools.py`

`create_custom_audience` previously fabricated an audience ID
(`f"audience_{int(time.time())}"`) and returned a fake success. It now calls
`client.create_custom_audience(audience_data)` (the new method above) and
surfaces the real `custom_audience_id` from TikTok's response.

**File:** `src/tiktok_ads_mcp/tools/creative_tools.py`

`create_ad_creative` previously fabricated a creative ID and returned a fake
success. It now calls `client.create_ad(creative_data)` so the creative is
actually persisted on the advertiser account. The function still keeps the
caller-friendly response shape but now also returns `ad_ids` and the raw
`api_result` for debugging.

## 5. Compatibility / non-goals

- Tool wire names remain stable for the 9 readonly tools and for the 4 already-
  routed write tools — no breaking changes for existing AdsMCP users.
- No changes to authentication flow beyond the redirect-URI default.
- No new dependencies were introduced.
- No upstream tests existed; syntax was validated via `python -m py_compile` on
  every modified file, and the OAuth precedence (arg > env > default) was
  verified by direct module-load.

## Why fork instead of PR upstream?

Leo's TikTok app is mid-review (1–3 day turnaround) and the Fadior team needs
deterministic write coverage on day-one. Upstream may want subtly different
defaults (e.g. their own redirect URI). Once the in-house pipeline is stable,
the write-tool registration + mock-replacement portions are good candidates for
upstream PRs.
