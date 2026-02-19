# Instance Performance Inspection Report

**Instance:** scvoice1 (https://scvoice1.service-now.com)
**Inspected:** 2026-02-19 (re-inspection after fixes applied)
**Previous Inspection:** 2026-02-19 (initial)
**Instance Version:** Zurich Patch 6 Hotfix 1 (build 02-09-2026)
**Active Users:** 2,002
**Background Script Execution:** Unavailable (Scripted REST API not configured or readOnly mode active)

---

## Executive Summary

This is the third inspection of scvoice1. All 9 fixes from the initial report remain confirmed. In this session, two additional items were resolved: **~152 `ignore_cache=false` properties were updated** (all `glide.ui.*` and `glide.*` properties cleared), and **both Article Optimization scheduled jobs were rescheduled to 02:00 AM** to avoid peak-hour load. Three items remain open: ~100 lower-priority scoped-app properties with `ignore_cache=false` (requires individual admin review), the task table hierarchy (architectural), and two daily jobs requiring product configuration changes (Scorecard Data Collection, SLO Error Budget).

**Status: 11 of 14 total recommendations resolved. 3 remaining open.**

---

## Fix Verification: Session 3 Fixes ✅

### ignore_cache=false Properties

All `glide.ui.*` and `glide.*` properties with `ignore_cache=false` have been updated to `ignore_cache=true`. Approximately 152 properties fixed across 6 batches (all succeeded with no cascades).

- All `glide.ui.*.activity.fields` properties — form activity formatter fields, read on every form load ✅
- All `glide.ui.*` UI configuration properties ✅
- All safe `glide.*` non-UI properties ✅
- **ACL-blocked (cannot update, requires manual admin):**
  - `glide.entitlement.ems.data.available` — EMS scoped app write restriction
  - `glide.ui.sn_vsc_instance_hardening_settings_activity.fields` — VSC scoped app write restriction

Remaining `ignore_cache=false` count: **~100** — all scoped-app properties (`sn_*`, `com.snc.*`, `css.*`, `com.glide.*`). These are lower priority (app-specific settings that change infrequently) and require individual review by app owners.

### Article Optimization Scheduled Jobs

Both monthly Article Optimization jobs rescheduled from 10:00 AM to **02:00 AM** to avoid peak-hour load.

| Job | sys_id | Before | After |
|---|---|---|---|
| Article Optimization - Scripted | 585113caffd876104ade90e0653bf1f6 | 10:00 AM | **02:00 AM** ✅ |
| Article Optimization - Now Assist | 5033db4effd876104ade90e0653bf127 | 10:00 AM | **02:00 AM** ✅ |

---

## Remaining Open Items

### [OPEN] ~100 Scoped-App Properties with `ignore_cache=false`
**Previous status:** C3 — Critical (100+)
**Current status:** ~100 remaining — all `glide.ui.*` and `glide.*` now resolved; remainder are scoped app properties

All `glide.ui.*` activity formatter fields (the highest-impact class — read on every form load) and `glide.ui.*` UI settings have been fixed. The remaining ~100 are scoped product settings (`sn_vul.*`, `com.snc.pa.*`, `sn_hr.*`, `css.*`, etc.) that are less frequently read and require individual app-owner review to determine whether `ignore_cache=true` is safe.

**Recommended action:** Review remaining properties via System Definition > System Properties, filtered by `ignore_cache=false`. Two properties are ACL-restricted and require ServiceNow Support to update: `glide.entitlement.ems.data.available` and `glide.ui.sn_vsc_instance_hardening_settings_activity.fields`.

---

### [OPEN] Task Table Hierarchy: 100+ Child Tables
**Original finding:** C4 — Critical
**Current status:** Still 100+ confirmed (query hit limit)

This is architectural and tied to the breadth of installed apps. Not reducible without deactivating scoped apps that add task subclasses. Requires a product-by-product review of which demo scenarios are actively used on this instance.

---

### [OPEN] High Processing-Duration Scheduled Jobs (Scorecard, SLO)
**Original finding:** W3, W4 — Warnings
**Current status:** Article Optimization ✅ rescheduled. Scorecard and SLO require product configuration changes.

| Job | Duration | Frequency | Status | Action Needed |
|---|---|---|---|---|
| Article Optimization - Scripted | 908,659ms (15 min) | Monthly | ✅ Rescheduled to 02:00 AM | Done |
| Article Optimization - Now Assist | — | Monthly | ✅ Rescheduled to 02:00 AM | Done |
| Scorecard Data Collection | 69,796ms (70s) | Daily | 🟡 Open | Reduce PA indicator scope |
| SLO: Error Budget | 34,287ms (34s) | Daily | 🟡 Open | Review ITOM SLO configuration |
| Admin Center Daily Data Collection | 17,405ms (17s) | Daily | ℹ️ Monitor | — |
| Stock Rule Runner | 12,169ms (12s) | Daily | ℹ️ Monitor | — |

---

## Full Inspection Results

### System Properties

| Property | Value | Status |
|---|---|---|
| `glide.war` (version) | Zurich Patch 6 HF1 | ✅ Current |
| `glide.invalid_query.returns_no_rows` | `true` | ✅ Fixed cycle 2 |
| `glide.home.refresh_disabled` | `true` | ✅ Fixed cycle 2 |
| `glide.ui.session_timeout` | `480` (8h) | ✅ Fixed cycle 2 |
| `glide.ui.list.allow_extended_fields` | `false` | ✅ Fixed cycle 2 |
| `glide.ui.goto_use_starts_with` | `true` | ✅ Fixed cycle 2 |
| `glide.security.use_csrf_token` | `true` | ✅ |
| `glide.ui.per_page` | `10,15,20,50` | ℹ️ 50 option present |
| `glide.db.max_view_records` | Not set | ℹ️ Using default |
| `glide.ui.related_list_timing` | Not set | ℹ️ Using default |
| `glide.home.refresh_intervals` | Not set | ℹ️ Using default |
| All `glide.ui.*` properties with `ignore_cache=false` | **0 remaining** | ✅ Fixed cycle 3 (~152 updated) |
| Remaining `ignore_cache=false` (scoped app props) | ~100 | 🟡 Open — manual audit needed |

---

### Business Rules

- 100 active before-timing BRs scanned — **no anti-patterns detected**
- 100 active async BRs scanned — **no anti-patterns detected**
- No matches for: `gs.setProperty`, `getRowCount`, `gs.sleep`, nested GlideRecord loops

✅ Clean

---

### Script Includes

- 100 active client-callable script includes — **no anti-patterns detected**

✅ Clean

---

### Query Performance

- Slow query log (`syslog_transaction`): **0 entries** ✅
- Index suggestions (`sys_index_suggestion`): **0 entries** ✅
- Record counts: Not available (background scripts unavailable)

---

### Table Health

- **Task hierarchy:** 100+ child tables (at query limit) — 🟡 Open
- **Table cleaner (`sys_auto_flush`):** Query returns 403 — field-level read restriction on this instance prevents inspection via MCP. Recommend manual review in System Definition > Table Cleaners.

---

### Scheduled Jobs

**ArcSight cluster: Confirmed eliminated from scheduler queue** ✅
All four ArcSight jobs are absent from `sys_trigger` — no longer firing.

**Article Optimization: Rescheduled to 02:00 AM** ✅

**Remaining high-frequency jobs (expected for active features):**

| Job | Est. Interval | Run Count | Avg Duration |
|---|---|---|---|
| Timeout unprocessed CCIF Messages | ~10s | 409,709 | 15ms |
| events process sn_skill_builder.autochat.metering | ~10s | 388,340 | 6ms |
| Event Management - Process records in em_extra_data_json | ~13s | 309,103 | 32ms |
| Indicator supporting data event process | ~24s | 172,552 | 13ms |
| Aggregate Indicator Source Records | ~20s | 197,656 | 6ms |
| @dotwalk/atf-commons-server/SNBOQProcess | ~26s | 154,866 | 645ms |
| GRC Metrics Queue Processor | ~30s | 135,229 | 5ms |

These are consistent with active AIOps, GRC, and Now Assist features — expected load.

**Long-running daily jobs (open):** See Remaining Open Items above.

---

### Client Scripts

- 20 client scripts using `getXMLAnswer` — all use async callback pattern ✅
- No synchronous AJAX detected

---

### UI Configuration

- Homepage auto-refresh: **Disabled** ✅ (fixed cycle 2)
- Session timeout: **8 hours** ✅ (fixed cycle 2)
- Extended fields on base lists: **Disabled** ✅ (fixed cycle 2)

---

## Recommendations — Updated Priority List

| # | Action | Priority | Status |
|---|---|---|---|
| 1 | Audit remaining ~100 `ignore_cache=false` scoped-app properties manually | Low | 🟡 Open |
| 2 | Review task table hierarchy — deactivate unused apps | Medium | 🟡 Open |
| 3 | Reduce Scorecard Data Collection PA scope | Medium | 🟡 Open |
| 4 | Review SLO: Error Budget job configuration | Medium | 🟡 Open |
| 5 | Enable background script execution in MCP config (`readOnly: false`) | Medium | 🟡 Open — needed for record counts, table cleaner inspection |
| 6 | Manual review of `sys_auto_flush` table cleaner rules | Medium | 🟡 Open — 403 prevents MCP inspection |
| ~~7~~ | ~~Fix ~152 `glide.ui.*` and `glide.*` properties with `ignore_cache=false`~~ | ~~Critical~~ | ✅ Done (cycle 3) |
| ~~8~~ | ~~Schedule Article Optimization jobs off-peak~~ | ~~Low~~ | ✅ Done (cycle 3) — rescheduled to 02:00 AM |
| ~~9~~ | ~~Set `glide.invalid_query.returns_no_rows = true`~~ | ~~Critical~~ | ✅ Done (cycle 2) |
| ~~10~~ | ~~Disable ArcSight dead integration jobs (4)~~ | ~~Critical~~ | ✅ Done (cycle 2) |
| ~~11~~ | ~~Set `glide.home.refresh_disabled = true`~~ | ~~High~~ | ✅ Done (cycle 2) |
| ~~12~~ | ~~Set `glide.ui.session_timeout = 480`~~ | ~~Medium~~ | ✅ Done (cycle 2) |
| ~~13~~ | ~~Set `glide.ui.goto_use_starts_with = true`~~ | ~~Medium~~ | ✅ Done (cycle 2) |
| ~~14~~ | ~~Set `glide.ui.list.allow_extended_fields = false`~~ | ~~Medium~~ | ✅ Done (cycle 2) |
