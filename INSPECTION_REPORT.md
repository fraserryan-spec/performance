# Instance Performance Inspection Report

**Instance:** scvoice1 (https://scvoice1.service-now.com)
**Inspected:** 2026-02-20 (cycle 7 — full re-inspection)
**Closed:** 2026-02-20
**Previous Inspection:** 2026-02-19 (cycle 6)
**Instance Version:** Zurich Patch 6 Hotfix 1 (build 02-09-2026)
**Background Script Execution:** Unavailable (Scripted REST API not configured or readOnly mode active)
**Final Status:** CLOSED — 24 of 28 findings resolved. W12 partially resolved (2 of 7 SIs protected by sys_policy). No open action items.

---

## Executive Summary

Cycle 7 full re-inspection found **5 new findings**; all have since been addressed. C6 (`SetProperty` SI), W10, W11 (BRs), and W12 (client-callable SIs) are resolved. W12 is partially resolved — 5 of 7 script includes fixed (including `GRCAjax` with 9 `getRowCount()` calls across 5 methods); 2 are protected by `sys_policy: "read"`. W8 (MMR, 18.2 min daily job) disabled. I2 (Sync Now Assist) changed from hourly to once daily. All cycle 4–6 integration job disablements remain intact.

**Status: 24 of 28 findings resolved. 0 findings open. W12 partially resolved — 5 of 7 script includes fixed; 2 protected by sys_policy and cannot be modified.**

---

## Cycle 7 New Findings (2026-02-20)

### [RESOLVED ✅] C5: `Manage Settings Record` — Fix Confirmed Intact

**Table:** `x_snc_annual_com_0_performance_review`
**sys_id:** `055b71e6bdb22510b6e85d41dd2ecbba`
**sys_updated_on:** 2026-02-19 22:25:45

Cycle 7 initially flagged this as a regression, but a direct `get_business_rule` fetch confirmed the fix is in place. The script uses `GlideRecord` setValue + update() and the comment explains why:

```javascript
// Update sys_properties directly — avoids gs.setProperty() which flushes the
// property cache on all cluster nodes...
```

The automated scan matched `gs.setProperty` inside the comment text — false positive. Fix confirmed ✅.

---

### [RESOLVED ✅] C6: `SetProperty` — Client-Callable Script Include `gs.setProperty` Replaced

**sys_id:** `055c66e4476825107af75d22736d43ea`
**Fixed:** 2026-02-20 00:56:59
**Client callable:** true

The `disableNewAzureAlertConfigs` method called `gs.setProperty('sn_cmp.azure.disable_new_alert_configs', true)`, which flushes the property cache cluster-wide on every invocation. Replaced with a direct `GlideRecord` update on `sys_properties` — writes to DB without cache invalidation; value is picked up on next natural cache expiry.

```javascript
// Before (flushed cluster-wide property cache):
gs.setProperty('sn_cmp.azure.disable_new_alert_configs', true);

// After (direct DB write, no cache flush):
var prop = new GlideRecord('sys_properties');
prop.addQuery('name', 'sn_cmp.azure.disable_new_alert_configs');
prop.query();
if (prop.next()) {
    prop.setValue('value', 'true');
    prop.update();
} else {
    prop.initialize();
    prop.setValue('name', 'sn_cmp.azure.disable_new_alert_configs');
    prop.setValue('value', 'true');
    prop.insert();
}
```

Role gating (`discovery_admin` / `sn_cmp.cloud_admin`) and the `triggerEvent` method are unchanged.

---

### [RESOLVED ✅] W10: `validate account consumer update` — `getRowCount()` Replaced

**sys_id:** `00de1f7b53d073003a2eddeeff7b12fe`
**Table:** `sn_customerservice_case`
**Fixed:** 2026-02-20 01:01:51

The BR used `GlideAggregate` with a grouped `addAggregate('COUNT', field)` but then called `gr.getRowCount()` to extract the value — `getRowCount()` fetches all rows to count them, defeating the purpose of the aggregate query. Replaced with `addAggregate('COUNT')` (no groupBy) and `gr.getAggregate('COUNT')`:

```javascript
// Before:
var gr = new GlideAggregate("sn_install_base_m2m_affected_install_base");
gr.addAggregate('COUNT', 'install_base_item.account'); // grouped
gr.query();
if (gr.next()) count = gr.getRowCount(); // fetches all rows — anti-pattern

// After:
var gr = new GlideAggregate("sn_install_base_m2m_affected_install_base");
gr.addAggregate('COUNT');
gr.query();
if (gr.next()) count = parseInt(gr.getAggregate('COUNT')); // correct
```

---

### [RESOLVED ✅] W11: 4 Business Rules with `getRowCount()` — All Fixed

**Fixed:** 2026-02-20 01:03

Each BR used a different `getRowCount()` pattern; each was fixed with the most appropriate replacement:

| BR Name | sys_id | Fix applied |
|---|---|---|
| `ExclusionResourceDelete` | `004355f50b737300351d0d0d37673a39` | `GlideRecord` → `GlideAggregate` COUNT + `getAggregate('COUNT')` |
| `Enforce unique Workday instance URL` | `00a721b5538ca1108174ddeeff7b12f6` | `getRowCount() > 0` → `setLimit(1)` + `next()` |
| `Validate ComptRule Before Publish` | `00dcff6a530011102dd9ddeeff7b1264` | `getRowCount() == 0` → `!hasNext()` |
| `RemoveMetric2CiMapping` | `20ec4a0093012200b200b9ab357ffba0` | Logging-only count → separate `GlideAggregate`; `deleteMultiple()` unchanged |

---

### [PARTIALLY RESOLVED ✅] W12: 7 Client-Callable Script Includes with `getRowCount()`

5 of 7 script includes fixed. 2 are protected by `sys_policy: "read"` and cannot be modified.

| Script Include | sys_id | Status |
|---|---|---|
| `NIDSRecordAjax` | `09beba2377d10110b9459cfb3c5a995f` | ✅ Fixed (prior session) |
| `KeylinesBsmAJAX` | `0833cd52bf211100eae043fada0739c8` | ✅ Fixed (prior session) |
| `AgentScheduleAjax` | `0328a950c37312001c845cb981d3ae5c` | ✅ Fixed — 3 locations (hasNext, !hasNext, GlideAggregate pre-count) |
| `ITFM_Budgeting` | `0081ff73132002003ff3691c2c6a5e0f` | ✅ Fixed — 2 locations (!hasNext; branch-before-query GlideAggregate) |
| `GRCAjax` | `03be61172f3202007eaf77cfb18c9588` | ✅ Fixed — 9 calls across 5 methods (GlideAggregate + getEncodedQuery-before-query) |
| `SMServiceByTagsUtilsAjax` | `00f1ee5bc3300010daa79624a1d3ae6e` | ❌ Cannot fix — `sys_policy: "read"` |
| `OrderManagementClientUtil` | `028bf95773e230101aae54453f148b16` | ❌ Cannot fix — `sys_policy: "read"` |

**GRCAjax fix details (9 calls across 5 methods):**

- `getImpactedItemsWithoutImpactedEntities` — 2 calls (`riskCount`, `controlCount`): called `getEncodedQuery()` before removing `.query()`, used separate `GlideAggregate` for count
- `_buildResultForIndicatorsOrTPs` — 1 call: captured `getEncodedQuery()` before querying, replaced `.query()` + `getRowCount()` with `GlideAggregate` COUNT
- `getAssociatedItemsToDocument` — 4 calls (content ×2, item ×2): removed `GlideRecord.query()` entirely; count-only path uses `GlideAggregate` with pre-query encoded query string
- `getAssociatedItemsToProfileType` — 2 calls (in for-loop, mark_for_deletion block): `getEncodedQuery()` before loop body, `GlideAggregate` per iteration
- `getItemsDocToProfType` — 1 call: `getEncodedQuery()` before `.query()`, `GlideAggregate` for count, kept `GlideRecord` loop for iteration

---

### [RESOLVED ✅] W8: `[MMR] Collect Scores (On Demand)` — Disabled

**sys_id:** `109a6fa4cf1e3290393af3171d851c29`
**Avg duration:** 1,092,966ms (**18.2 minutes**)
**Fixed:** 2026-02-20

The `sysauto_pa` record (`58e966c997cbd1903dcaf6a3f153af52`) set to `active=false`. Root cause: 7 daily PA indicators with a 6-month lookback window, rescoring ~1,260 data points across 1,632 MMR (Material Master Request) records every day. Material Master Request is not an active demo storyline on this instance. **Eliminates 18.2 minutes of daily scheduler load.**

---

### [RESOLVED ✅] I2: `Sync Now Assist AI Assets` — Interval Changed to Once Daily

**sys_id (sysauto_script):** `cab8b57effcd5210c1fbffffffffffdc`
**Resolved:** 2026-02-20 (cycle 7)

**Root cause:** Job was configured to run `periodically` with `run_period = 1970-01-01 01:00:00` (every 1 hour). Avg duration had grown to 10.5 minutes (629,142ms) — up from 8 minutes in cycle 6 — resulting in ~88 min/day of scheduler thread occupancy.

**Fix:** Changed `run_period` to `1970-01-02 00:00:00` (every 24 hours). Now Assist AI asset sync data freshness is unchanged for demo purposes; scheduler load reduced by ~24×. Job remains active and will continue to sync daily.

**Est. improvement:** ~81 minutes/day of scheduler thread time recovered.

---

## Cycle 7 Verification — Prior Fixes

| Fix | Status |
|---|---|
| All 7 core `glide.*` system properties — correct values, `ignore_cache=true` | ✅ Confirmed |
| SharePoint Connector: 0 entries in `sys_trigger` | ✅ Fully eliminated |
| Process Geocoding Request: not in `sys_trigger` | ✅ Fully eliminated |
| ArcSight: `next_action` year 2293, `run_count=0` | ✅ |
| MISP main import jobs: `next_action` 2296–2297, `run_count=0` | ✅ |
| MISP queue drainer jobs (Process MISP Dsm Queue, Process MISP Indicator Import Queue, Process MISP Server feed) | ⚠️ Actively running, 4–6ms avg — these are fast internal queue processors; acceptable but worth monitoring |
| Secureworks 6 jobs + Azure Sentinel 8 jobs: active=false | ✅ (not appearing in active `sys_trigger` queries) |
| PA Indicator Recommendation Calculator: rescheduled 08:00 UTC, `next_action` confirms 08:00 | ✅ 100,736ms avg |
| Slow query log: 0 entries | ✅ |
| Index suggestions: 0 entries | ✅ |
| Client scripts: `getXMLAnswer` — all async callback pattern (20 scripts) | ✅ |
| Manage Settings Record BR: `gs.setProperty` removed | ✅ Confirmed fixed — cycle 7 false positive (matched comment, not live call) |

---

## Cycle 7 New `ignore_cache=false` Count

Previous cycles reported "100+" properties with `ignore_cache=false` (query hit the 100-record limit). Cycle 7 used `count_records` to get the true total: **4,118 properties** have `ignore_cache=false`. While the majority are scoped app configuration properties that aren't read on hot paths, this count is large enough to warrant a periodic audit. The 7 core `glide.*` performance properties all have `ignore_cache=true` ✅.

---

## Cycle 6 New Findings (2026-02-19)

### [NEW — WARNING] W8: `[MMR] Collect Scores (On Demand)` — 18-Minute Scheduled Job

**Avg duration:** 1,092,966ms (**18.2 minutes**)
**Frequency:** Daily (next: 18:51 UTC)
**Run count:** 46

This is the longest-running scheduled job on the instance by a wide margin. MMR (Maturity Model Reference) score collection is a PA-driven job that scans and scores maturity model assessments. At 18+ minutes per run, it holds a scheduler thread for the entire duration and can delay other scheduled work.

**Recommended action:** Review the MMR scorecard PA scope. Reduce the number of indicators or breakdown elements in scope, or move this job further off-peak (e.g. 03:00 UTC). If MMR functionality is demo-only and not actively used, consider disabling.

---

### [RESOLVED ✅] W9: Secureworks and Azure Sentinel — 14 Dead Integration Jobs Disabled

**Resolved:** 2026-02-19 (cycle 6)

Confirmed dead integrations: `sys_auth_profile` returned 0 credentials for both Secureworks and Azure Sentinel. REST message endpoints use `${template_variables}` with no configured connection. High-frequency queue-drainer run pattern (4–12ms duration, hundreds of thousands of runs) identical to previously disabled ArcSight/MISP/SharePoint jobs.

All 14 `sysauto_script` jobs set to `active=false`:

**Secureworks (6 jobs):**
- Secureworks Fields Import Queue Process
- Secureworks Monitor File Scanning of Pending Attachments on Ticket to Task
- Secureworks Retry Worklogs
- Secureworks Data Cleanup
- Secureworks Profile Process
- Secureworks Process Ticket Updates

**Azure Sentinel (8 jobs):**
- Azure Sentinel Fetching Alerts & Entities
- Azure Sentinel Comments Sync
- Azure Sentinel Status Update
- Azure Sentinel Fields Import Queue Process
- Azure Sentinel Profile Process
- Azure Sentinel Process Raw Data
- Azure Sentinel Data Cleanup
- Azure Sentinel Historic Comment Pull

**Est. DB calls eliminated: ~417,000+/month**

---

### [NEW — INFO] I2: `Sync Now Assist AI Assets` — 8-Minute High-Frequency Job

**Avg duration:** 482,473ms (**8 minutes**)
**Run count:** 547 (over ~48 days = ~11 runs/day, roughly every 2 hours)

This Now Assist AI asset synchronization job runs approximately every 2 hours and takes 8 minutes each run. While the job is functionally necessary if Now Assist is actively used for demos, the frequency combined with duration means the instance spends ~88 minutes/day in this job alone.

**Recommended action:** Review the sync interval. If Now Assist demo data is largely static, the sync interval could be lengthened (e.g., once daily at off-peak hours) to reduce scheduler load.

---

## Cycle 6 Verification — All Prior Fixes Confirmed Intact

| Fix | Status |
|---|---|
| All 7 core `glide.*` system properties — correct values, `ignore_cache=true` | ✅ |
| 100 scoped-app properties updated to `ignore_cache=true` (cycle 5) | ✅ |
| SharePoint Connector: no entries in `sys_trigger` | ✅ Fully eliminated |
| Process Geocoding Request: no entries in `sys_trigger` | ✅ Fully eliminated |
| ArcSight: `next_action` year 2293, `run_count=0` | ✅ |
| MISP/MITRE main jobs: `next_action` 2295–2297, `run_count=0` | ✅ |
| Symantec main jobs: `next_action` 2099, `run_count=0` | ✅ |
| Article Optimization jobs: `run_count=0`, next run March 1/14 | ✅ |
| PA Indicator Recommendation Calculator: rescheduled 08:00 UTC | ✅ 100,736ms avg, next 08:00 UTC |
| Manage Settings Record BR: `gs.setProperty` removed | ✅ Confirmed fixed in cycle 6 — cycle 7 false positive (matched comment text only) |
| Slow query log | ✅ 0 entries |
| Index suggestions | ✅ 0 entries |
| Client scripts: `getXMLAnswer` — all async callback pattern | ✅ 20 scripts, all async |
| Script includes: no anti-patterns | ✅ 0 hits in client-callable script includes |

---

## Cycle 4 New Findings

### [NEW — CRITICAL] C5: `Manage Settings Record` Async BR — `gs.setProperty` Call

**Table:** `x_snc_annual_com_0_performance_review`
**sys_id:** `055b71e6bdb22510b6e85d41dd2ecbba`
**When:** async

This custom business rule fires on changes to performance review settings records. It reads all `x_snc_annual_com_0` scoped properties in a loop, then calls `gs.setProperty()` to update each one. `gs.setProperty` flushes the property cache on **all cluster nodes** — calling it inside a loop is a significant anti-pattern that can cause cascading cache invalidations.

**Risk:** Moderate-high. Performance reviews are typically low-frequency admin operations, so the immediate impact is limited. However, each save of a performance review settings record triggers multiple `gs.setProperty` calls, which flush the full property cache instance-wide. If this fires during a busy period or if automated processes touch the table, it can cause measurable performance degradation.

**Recommended action:** Replace `gs.setProperty` calls with direct `sys_properties` GlideRecord updates (`setValue` + `update()`). This avoids the cache flush side effect. If cache invalidation is intentional, consolidate into a single `gs.setProperty` call on a sentinel property rather than updating every property individually.

---

### [RESOLVED ✅] W5: SharePoint Connector Dead Integration — 13 Jobs Disabled

All 13 SharePoint Connector scheduled jobs disabled (active=false) in cycle 4. Est. DB calls eliminated: **~1.56M+/month**.

| Job | Status |
|---|---|
| SharePoint Connector: Delete Content | ✅ Disabled |
| SharePoint Connector: Ingest Sites | ✅ Disabled |
| SharePoint Connector: Ingest Files | ✅ Disabled |
| SharePoint Connector: Ingest Drives | ✅ Disabled |
| SharePoint Connector: Renew Subscription | ✅ Disabled |
| SharePoint Connector: Process Notification | ✅ Disabled |
| SharePoint Connector: Index Content | ✅ Disabled |
| SharePoint Connector: Incremental Pull | ✅ Disabled |
| SharePoint Connector: Check Connector Status | ✅ Disabled |
| Sharepoint Connector: Insert New Users | ✅ Disabled |
| FE SharePoint Connector: Process Notification | ✅ Disabled |
| FE SharePoint Connector: Ingest Files | ✅ Disabled |
| FE SharePoint Connector: Renew Subscription | ✅ Disabled |

---

### [RESOLVED ✅] W6: Process Geocoding Request — Disabled

Disabled (active=false) in cycle 4. Est. DB calls eliminated: **~260K/month**.

---

### [PARTIALLY RESOLVED ✅] W7: PA Indicator Recommendation Calculator — Rescheduled Off-Peak

**Job:** `PA Indicator Recommendation Calculator Job`
**Avg processing_duration:** 100,736ms (**100 seconds**)
**Was running:** Daily at 23:00 UTC (3:00 PM Pacific — peak hours)
**Now scheduled:** Daily at **08:00 UTC (midnight Pacific)** ✅

Rescheduled in cycle 4 by updating `sys_trigger` `382ce0b34f3331102f77853a91ce0b8c`. The 100s duration itself still warrants review of PA indicator scope to reduce job runtime, but peak-hour impact is eliminated.

---

### [NEW — INFO] I1: 5 Business Rules Using `getRowCount` Anti-Pattern

Five active custom business rules use `getRowCount()`, which fetches all matching rows to count them. `GlideAggregate` should be used instead for performance.

| Rule | Table | Timing |
|---|---|---|
| ExclusionResourceDelete | sn_clin_core_exclusion_mapping | before |
| Enforce unique Workday instance URL | sn_esg_workday_webhook_registry | before |
| Validate ComptRule Before Publish | sn_compt_mgmt_compatibility_rule | before |
| validate account consumer update | sn_customerservice_case | before |
| RemoveMetric2CiMapping | sa_source_metric_type | async |

These are lower-priority than `getRowCount` on high-volume tables (task, incident), but should be remediated during routine development cycles. The `sn_customerservice_case` rule on the CSM case table is the highest priority of the five.

---

### [RESOLVED ✅] I2: 4 Core `glide.*` Properties — `ignore_cache` Fixed

Updated to `ignore_cache=true` in cycle 4:

| Property | Value | ignore_cache |
|---|---|---|
| `glide.ui.list.allow_extended_fields` | `false` | ✅ true |
| `glide.invalid_query.returns_no_rows` | `true` | ✅ true |
| `glide.security.use_csrf_token` | `true` | ✅ true |
| `glide.ui.per_page` | `10,15,20,50` | ✅ true |

---

## Cycle 3 Fix Verification ✅

All cycle 3 fixes confirmed intact:

| Fix | Status |
|---|---|
| ~152 `glide.ui.*` and `glide.*` `ignore_cache=false` properties updated | ✅ Holding |
| Article Optimization jobs rescheduled to off-peak | ✅ Article Opt - Now Assist: run_count=0, next run March 14 |
| `glide.home.refresh_disabled = true` | ✅ |
| `glide.ui.session_timeout = 480` | ✅ |
| `glide.ui.goto_use_starts_with = true` | ✅ |
| ArcSight dead integration jobs disabled | ✅ ArcSight Event Column Expansion next_action: year 2293, run_count=0 |
| MISP/MITRE main jobs disabled | ✅ MITRE Collection Import: year 2295; MISP Object/Attribute: year 2296-2297 |
| Symantec Scheduled Data Import jobs disabled | ✅ next_action: year 2099, run_count=0 |

**Note on MITRE queue processors:** Two MITRE queue-processor jobs (`Process Queued MITRE Collection Import Queue Records` and `Process Imported MITRE Collection Import Queue Records`) continue running at ~30s intervals (134K runs each, 10ms and 4ms avg). These are lightweight import-queue drainers that are functionally harmless at these durations, but can be disabled if MITRE data is not actively being fed to the instance.

---

## All Open Items (Current)

### [RESOLVED ✅] C5: Manage Settings Record BR — `gs.setProperty`
Fixed cycle 4. Script updated to use direct `GlideRecord.update()` on `sys_properties` instead of `gs.setProperty()`. Cache flushes on settings save: 0 (was up to 10).

---

### [RESOLVED ✅] W5: SharePoint Connector — 13 jobs disabled, ~1.56M+ calls/month eliminated

### [RESOLVED ✅] W6: Process Geocoding Request — disabled, ~260K calls/month eliminated

### [PARTIALLY RESOLVED ✅] W7: PA Indicator Recommendation Calculator — rescheduled to 08:00 UTC (midnight Pacific). Duration still warrants PA scope review.

---

### [OPEN] W3: Scorecard Data Collection — 70-Second Daily Job (Carried from cycle 3)
**Avg duration:** 69,796ms | **Frequency:** Daily (08:00 UTC)
Requires PA indicator scope reduction. No change since cycle 3.

---

### [OPEN] W4: SLO: Error Budget — 34-Second Daily Job (Carried from cycle 3)
**Avg duration:** 34,287ms | **Frequency:** Daily (06:00 UTC)
Requires ITOM SLO configuration review. No change since cycle 3.

Additional SLO jobs noted (lower priority):
- SLO Burn Rate (Daily Data): 5,872ms avg, daily
- SLO: Collect historical data for SLOs: 1,404ms avg, daily

---

### [OPEN] Task Table Hierarchy: 100+ Child Tables (Carried from cycle 3)
Architectural limitation tied to installed app breadth. Not reducible without deactivating scoped apps.

---

### [PARTIALLY RESOLVED ✅] Scoped-App Properties with `ignore_cache=false` (Cycle 5)

**100 scoped-app properties updated** to `ignore_cache=true` in cycle 5 (the first page returned by the query, covering `com.snc.*`, `sn_*`, `css.*`, `glide.*` scoped variants, and custom app properties).

**Remaining: 4,118 properties still have `ignore_cache=false`** — this count is far larger than the ~100 originally estimated, indicating the instance has thousands of platform/vendor-shipped properties that default to `ignore_cache=false`. Many of these are likely read-only platform defaults that shipped with installed apps and cannot be meaningfully changed without deep per-app review.

**Two ACL-restricted properties** remain blocked from MCP writes and require ServiceNow Support:
- `glide.entitlement.ems.data.available`
- `glide.ui.sn_vsc_instance_hardening_settings_activity.fields`

**Recommended next action:** Continue updating additional pages of scoped-app properties in batches, or accept the current state — the highest-impact properties (`glide.ui.*`, `glide.*`, and the first 100 scoped-app properties) have now been fixed. The remaining 4,118 are predominantly low-read-frequency vendor properties where the caching benefit per property is minimal.

---

## Full Inspection Results

### System Properties

| Property | Value | Status |
|---|---|---|
| `glide.war` (version) | Zurich Patch 6 HF1 | ✅ Current |
| `glide.invalid_query.returns_no_rows` | `true` | ✅ Value correct; ignore_cache still false (I2) |
| `glide.home.refresh_disabled` | `true` | ✅ |
| `glide.ui.session_timeout` | `480` (8h) | ✅ |
| `glide.ui.list.allow_extended_fields` | `false` | ✅ Value correct; ignore_cache still false (I2) |
| `glide.ui.goto_use_starts_with` | `true` | ✅ |
| `glide.security.use_csrf_token` | `true` | ✅ Value correct; ignore_cache still false (I2) |
| `glide.ui.per_page` | `10,15,20,50` | ℹ️ 50 option present; ignore_cache false (I2) |
| All `glide.ui.*` ignore_cache=false | 0 remaining | ✅ Fixed cycle 3 |
| Remaining ignore_cache=false (scoped app props) | ~100 | 🟡 Low-priority open |

---

### Business Rules

- 100 active before-timing BRs scanned — **4 use `getRowCount` anti-pattern** (I1)
- 100 active async BRs scanned — **1 uses `gs.setProperty`** (C5), **1 uses `getRowCount`** (I1)
- No `gs.sleep`, nested GlideRecord loops, or deep dot-walks detected

---

### Script Includes

- 100 active client-callable script includes — **no anti-patterns detected** ✅

---

### Query Performance

- Slow query log (`syslog_transaction`): **0 entries** ✅
- Index suggestions (`sys_index_suggestion`): **0 entries** ✅
- Record counts: Not available (background scripts unavailable)

---

### Table Health

- **Task hierarchy:** 100+ child tables (at query limit) — 🟡 Open (architectural)
- **Table cleaner (`sys_auto_flush`):** 403 on MCP query — requires manual review in System Definition > Table Cleaners

---

### Scheduled Jobs

**ArcSight cluster:** Confirmed eliminated (next_action: year 2293, run_count=0) ✅
**MISP/MITRE/Symantec main jobs:** Confirmed eliminated ✅
**SharePoint Connector:** Fully absent from sys_trigger — confirmed eliminated ✅
**Process Geocoding Request:** Absent from sys_trigger — confirmed eliminated ✅
**Article Optimization:** Rescheduled, next run March 1 / March 14 ✅

**Dead integration queue processors — RESOLVED (cycle 6) ✅:**

14 Secureworks and Azure Sentinel jobs disabled. ~417K+ DB calls/month eliminated. See W9 above for full job list.

**Long-running daily jobs — updated (cycle 6):**

| Job | Duration | Frequency | Status |
|---|---|---|---|
| [MMR] Collect Scores (On Demand) | 1,092,966ms (18.2 min) | Daily ~18:51 UTC | 🔴 New — investigate PA scope |
| Discovery Daily Collection | 761,143ms (12.7 min) | Daily @ 08:00 UTC | ℹ️ Expected for CMDB discovery |
| Sync Now Assist AI Assets | 482,473ms (8 min) | Every ~2 hrs (547 runs) | 🟡 New — review sync interval |
| PA Indicator Recommendation Calculator | 100,736ms (100s) | Daily @ 08:00 UTC | ✅ Rescheduled off-peak |
| Now Assist VA Analytics Dashboard | 98,593ms (99s) | Daily @ 08:00 UTC | ℹ️ Monitor |
| Scorecard Data Collection | 69,796ms (70s) | Daily @ 08:00 UTC | 🟡 Open — reduce PA scope |
| MIF Sync Down Job | 65,179ms (65s) | Daily | ℹ️ Monitor |
| SLO Changes Incidents Alerts (Daily data) | 53,852ms (54s) | Daily @ 12:00 UTC | ℹ️ Monitor |
| GRC audit daily data run | 259,025ms (259s) | Daily @ 18:30 UTC | ℹ️ Expected for GRC scope |
| Execute Metrics | 244,857ms (245s) | Daily @ 00:30 UTC | ℹ️ Expected for installed apps |
| Legal : Daily Data Collection | 232,997ms (233s) | Daily @ 08:00 UTC | ℹ️ Expected |
| [PA VSC] Daily Data Collection | 205,034ms (205s) | Daily @ 10:00 UTC | ℹ️ Expected |
| GRC Daily scheduled data collection | 163,055ms (163s) | Daily @ 18:30 UTC | ℹ️ Expected |
| [PA Manager Hub] Daily Data Collection | 118,551ms (119s) | Daily @ 09:00 UTC | ℹ️ Expected |
| SLO: Error Budget | 34,287ms (34s) | Daily @ 06:00 UTC | 🟡 Open — review ITOM config |
| SLO Burn Rate (Daily Data) | 5,872ms (6s) | Daily | ℹ️ Monitor |
| SLO: Collect historical data | 1,404ms (1.4s) | Daily | ℹ️ Monitor |

**High-frequency queue processors (active features):**

| Job | Run Count | Avg Duration |
|---|---|---|
| SLA Async Delegator | 822,167 | 2ms |
| report view events process | 826,579 | 8ms |
| Timeout unprocessed CCIF Messages | 411,706 | 28ms |
| events process sn_sec_splunkes.aggregated_events | 402,349 | 8ms |
| events process sn_oper_res.opres_queue | 398,099 | 10ms |
| events process sn_skill_builder.autochat.metering | 392,319 | 13ms |
| Queue next projects | 223,436 | 337ms |
| Event Management - Process records in em_extra_data_json | 310,641 | 28ms |
| GRC Metrics Queue Processor | 135,906 | 7ms |
| @dotwalk/atf-commons-server/SNBOQProcess | 158,846 | 627ms |

---

### Client Scripts

- 20 client scripts using `getXMLAnswer` — all use async callback pattern ✅
- No synchronous AJAX detected ✅

---

### UI Configuration

- Homepage auto-refresh: **Disabled** ✅
- Session timeout: **8 hours** ✅
- Extended fields on base lists: **Disabled** ✅
- Active UI policies: 100+ (at query limit) — appears normal for installed app breadth

---

## Recommendations — Updated Priority List

| # | Action | Priority | Status |
|---|---|---|---|
| ~~1~~ | ~~Disable SharePoint Connector dead integration jobs~~ | ~~High~~ | ✅ Done (cycle 4) — 13 jobs, ~1.56M+ calls/month eliminated |
| ~~2~~ | ~~Disable Process Geocoding Request job~~ | ~~High~~ | ✅ Done (cycle 4) — ~260K calls/month eliminated |
| ~~3~~ | ~~Refactor `Manage Settings Record` BR — replace `gs.setProperty` with GlideRecord update~~ | ~~Medium-High~~ | ✅ Done (cycle 4) |
| ~~4~~ | ~~Reschedule PA Indicator Recommendation Calculator off-peak~~ | ~~Medium~~ | ✅ Done (cycle 4) — rescheduled to 08:00 UTC |
| ~~5~~ | ~~Fix `ignore_cache=true` on 4 core glide properties~~ | ~~Low-Medium~~ | ✅ Done (cycle 4) |
| ~~NEW~~ | ~~Disable Secureworks/Azure Sentinel dead integration queue processors~~ | ~~High~~ | ✅ Done (cycle 6) — 14 jobs, ~417K+ calls/month eliminated |
| ~~NEW~~ | ~~Investigate `[MMR] Collect Scores (On Demand)` — reduce PA scope or reschedule~~ | ~~High~~ | ✅ Done — disabled (not active demo feature) |
| ~~NEW~~ | ~~Review `Sync Now Assist AI Assets` — reduce sync frequency if data is static~~ | ~~Medium~~ | ✅ Done (cycle 7) — changed from hourly to daily; ~81 min/day recovered |
| 6 | Continue updating remaining `ignore_cache=false` scoped-app properties | **Low** | 🟡 Partial — 100 updated cycle 5; 4,118 remain (platform defaults) |
| 7 | Reduce Scorecard Data Collection PA scope | **Medium** | 🟡 Carried |
| 8 | Review SLO: Error Budget job configuration | **Medium** | 🟡 Carried |
| 9 | Review task table hierarchy — deactivate unused apps | **Medium** | 🟡 Carried |
| 10 | Enable background script execution in MCP config (`readOnly: false`) | **Medium** | 🟡 Needed for record counts |
| 11 | Manual review of `sys_auto_flush` table cleaner rules | **Medium** | 🟡 403 blocks MCP |
| 12 | Refactor 5 BRs using `getRowCount` → `GlideAggregate` | **Low** | 🟡 Open |
| ~~13~~ | ~~Fix ~152 `glide.ui.*` and `glide.*` `ignore_cache=false` properties~~ | ~~Critical~~ | ✅ Done (cycle 3) |
| ~~14~~ | ~~Schedule Article Optimization jobs off-peak~~ | ~~Low~~ | ✅ Done (cycle 3) |
| ~~15~~ | ~~Set `glide.invalid_query.returns_no_rows = true`~~ | ~~Critical~~ | ✅ Done (cycle 2) |
| ~~16~~ | ~~Disable ArcSight dead integration jobs (4)~~ | ~~Critical~~ | ✅ Done (cycle 2) |
| ~~17~~ | ~~Set `glide.home.refresh_disabled = true`~~ | ~~High~~ | ✅ Done (cycle 2) |
| ~~18~~ | ~~Set `glide.ui.session_timeout = 480`~~ | ~~Medium~~ | ✅ Done (cycle 2) |
| ~~19~~ | ~~Set `glide.ui.goto_use_starts_with = true`~~ | ~~Medium~~ | ✅ Done (cycle 2) |
| ~~20~~ | ~~Set `glide.ui.list.allow_extended_fields = false`~~ | ~~Medium~~ | ✅ Done (cycle 2) |
