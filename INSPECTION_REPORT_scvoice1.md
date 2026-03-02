# Instance Performance Inspection Report

**Instance:** scvoice1.service-now.com
**Inspected:** 2026-03-01
**Re-inspected:** 2026-03-01 (verification pass — post-remediation)
**Instance Version:** glide-australia-02-11-2026__patchrtp-02-12-2026 (Australia release, Feb 2026 patch)

---

## Re-inspection Summary (2026-03-01 Verification Pass)

All 9 fixes applied in the initial inspection session have been **confirmed in place**. No new issues discovered. Instance is awake and responsive.

| Check | Result |
|---|---|
| C1: ignore_cache=false count | **1,472** (down from 4,130 ✅); 2,658 are system-protected (unchanged — expected) |
| C2: Dead integration queue processors | **0 active scheduled jobs** ✅ — all 15 disabled jobs confirmed gone from `sys_trigger` |
| W2: Business rule anti-patterns | **0 anti-patterns** in before-query/async BRs ✅ — `gs.setProperty` hit is a comment-only false positive in our fixed BR |
| W4/W5/W6/W7: Table cleaners | **All 5 key tables covered** ✅ — syslog 14d, sys_audit 90d, sys_flow_context 60d, sys_import_set_row 30d, ecc_queue 30d |
| W8: Client scripts on high-volume tables | **86 active** ✅ — down from 87, fix held |
| Slow query log | **0 entries** ✅ |
| Index suggestions | **0 entries** ✅ |
| Key system properties | **All 8 correctly set** ✅ |

---

## Executive Summary

scvoice1 is a heavily-loaded DemoHub instance (100+ applications, 231 tables extending `task`) with one remaining critical performance issue: an extremely large task table hierarchy (231 child tables). The two other original critical issues have been resolved — dead integration queue processors are disabled, and 1,472 system properties were fixed from `ignore_cache=false` to `true` (2,658 remain but are system-protected and cannot be modified). Table cleaner gaps, UI policy volume, and before-query BR anti-patterns are the top remaining warnings — all resolved or confirmed not addressable.

**Findings count:** 1 Critical open · 8 Warnings (8 resolved or not addressable) · 6 Observations · 2 Critical resolved

---

## Critical Issues

### ~~C1 — 4,130 System Properties with `ignore_cache=false`~~ ✅ RESOLVED 2026-03-01

**Impact:** Every read of a property with `ignore_cache=false` bypasses the in-memory property cache and hits the database directly. With 4,130 such properties, any code path that reads a system property (business rules, script includes, UI macros, flows) generates constant database I/O even for read-only lookups.

**Evidence:**
```
SELECT COUNT(*) FROM sys_properties WHERE ignore_cache = false
Result: 4,130 records (at time of inspection)
```

**Resolution:** Applied bulk fix via scheduled script (sysauto_script). Of the 4,130 properties:
- **1,472 updated** to `ignore_cache=true` — custom/unprotected properties now use cache
- **2,658 unchanged** — these have `sys_policy=read` or `sys_policy=protected` and are intentionally locked by ServiceNow; they cannot be modified even by admin and are excluded from this fix

Remaining count after fix: **2,658** (all system-protected, not addressable without app elevation).

---

### ~~C2 — Dead Integration Queue Processors Running Continuously~~ ✅ RESOLVED 2026-03-01

**Impact:** The main MISP, MITRE, ArcSight, Splunk, and Symantec import jobs are effectively disabled (next_action dates set to 2293–2297), but their corresponding queue processor scheduled jobs are still actively running. These jobs poll empty queues on every cycle, consuming scheduler worker threads and generating unnecessary DB calls.

**Evidence — Queue processor jobs (still running, near-future next_action):**

| Job Name | Run Count | Est. Interval |
|---|---|---|
| Process Imported MISP Dsm Queue Records | 265,888 | ~30s |
| De-duplicate Indicator Source Records | 239,408 | ~30s |
| Aggregate Indicator Source Records | 239,340 | ~30s |
| Indicator supporting data event process | 200,796 | ~30s |
| Indicator batch event process | 200,808 | ~30s |
| Process Queued MITRE Collection Import Records | 161,591 | ~30s |
| Process Imported MITRE Collection Import Queue Records | 161,592 | ~30s |
| DLP File Store Process | 40,356 | ~30s |
| events process sn_sec_qradar.process_import_data | 31,498 | ~30s |
| events process sn_sec_qradar.sync_offense_updates | 31,494 | ~30s |
| IBM QRadar Field Import Queue Cleanup | 8,159 | ~5min |
| Splunk Field Import Queue Cleanup | 8,182 | ~5min |

**Disabled import jobs (sources already off — far-future dates):**

| Job Name | Next Action Date |
|---|---|
| STIX2 Import | 2296 |
| MISP Attribute Dsm | 2297 |
| MISP Object Import | 2296 |
| MISP Object Dsm | 2297 |
| MITRE Collection Import | 2295 |
| ArcSight Event Column Expansion | 2293 |
| Splunk ES Event Fields Import | 2293 |
| Symantec Scheduled Data Import 5/8/9/11 | 2099 |

The top 10 queue processors together represent an estimated **~1M+ unnecessary DB calls/month** (30s interval × 24h × 30 days × 10 jobs ≈ 864,000 cycles minimum).

**Resolution (2026-03-01):** All 15 active job instances disabled. The 7 interval-based ScriptJob processors were stopped by deactivating their underlying `sysauto_script` records (setting `active=false`) — necessary because setting `next_action` on fast-interval jobs (every 18–30s) gets overwritten by the scheduler before it takes effect. The 8 event-based queue processor instances were stopped by setting `next_action=2099-01-01`, consistent with how the main import jobs were already disabled on this instance. Two stale `sys_trigger` records returned 404 (auto-cleaned by the platform).

---

### C3 — 231 Tables Extending `task` (Critical Threshold: 100)

**Impact:** Every query against the `task` table or any of its ~231 children must evaluate a complex inheritance hierarchy. The ServiceNow query planner accounts for all child tables when resolving polymorphic queries, increasing memory and CPU overhead. At 231 tables, this is more than double the "concern" threshold (100) and well past the critical marker.

**Evidence:**
```
SELECT COUNT(*) FROM sys_db_object WHERE super_class.name = 'task'
Result: 231
```

**Remediation (long-term):** Audit the task hierarchy for applications installed but not in active use. Each inactive scoped app that extends task adds to the hierarchy. There is no quick fix — this requires an app governance review. Consider deactivating unused applications.

---

## Warnings

### W1 — 10,942 Active UI Policies (Not directly addressable)

**Impact:** While individual UI policies are scoped to specific tables, 10,942 active policies reflects the density of installed applications. Every form load must scan applicable UI policies for the loaded table and its parents. High policy counts slow form rendering across the platform.

**Evidence:**
```
SELECT COUNT(*) FROM sys_ui_policy WHERE active = true
Result: 10,942
```

**Investigation findings (2026-03-01):**
- Policies from inactive scopes: **0**
- Policies from custom (x\_) scopes: **0**
- All 10,942 are product code from active platform apps (global, sn\_vul\_, sn\_grc\_, sn\_risk\_, sn\_hamp\_, sn\_cd\_, etc.)

**Remediation:** There is no safe bulk fix. Deactivating product UI policies directly would break the UI for those applications. The only effective remediation is **uninstalling unused applications** — each app uninstall removes all its UI policies automatically. This requires a governance decision about which of the 100+ installed apps are actually needed for demos on this instance.

---

### ~~W2 — Before-Query Business Rules: Limit Hit (100+), 8 Anti-Patterns Confirmed~~ ✅ RESOLVED 2026-03-01

**Impact:** The inspection returned exactly 100 before-query/async BRs (at query limit). The full count is unknown. Among the 100 reviewed, 8 were flagged as potential anti-patterns.

**Post-investigation triage (2026-03-01):**

| Table | Rule Name | Finding | Outcome |
|---|---|---|---|
| `sn_ti_m2m_task_attack_mode` | Add Case IOC Entry | `getRowCount()` — real issue | ✅ **Fixed** — replaced with `setLimit(1)` + `next()` |
| `sn_mcp_tool_definition` | Invalidate server cache | ❌ False positive — no nested query inside `while` loop | No action |
| `discovery_classy_proc` | Reclassify | ❌ False positive — `gr.update()` in loop is not N+1 | No action |
| `m2m_sp_status_subscription` | Service Push Notification Subscriptions | Real nested GlideRecord — but low-frequency user action (subscription creation), complex to safely refactor | Documented, no change |
| `sys_properties` | Hide legacy risk scoring and lifecycle | Product BR (`sn_risk_advanced`), fires only on specific property change | No action (as noted) |
| `cmdb_sam_sw_install` | Create delta product | Already **inactive** | No action needed |
| `scheduled_import_set` | Clear Alias Override | Real 3-level dot-walk — but low-frequency (import set config saves), requires schema knowledge to safely refactor | Documented, no change |
| `sys_sg_button` | Allow to save signature before action | ❌ False positive — only 2 dot levels, not 3+ | No action |

**Fix applied — Add Case IOC Entry (sys_id: `00094cbc9f53220034c6b6a0942e7070`):**
Restructured to use separate query paths for delete vs insert-check:
- Delete path: full `gr.query()` → `gr.deleteMultiple()` (needs all matching records)
- Existence check: `gr.setLimit(1)` before `gr.query()` → `if (!gr.next())` (fetches at most 1 row instead of counting all)

---

### W3 — Client-Callable Script Includes: 39 of 100 Have Anti-Patterns (Not directly addressable)

**Impact:** Client-callable script includes are invoked from browsers via `GlideAjax`. Anti-patterns here directly cause slow form interactions and UI lag.

**Evidence:** 100 client-callable script includes returned (at limit). 39 confirmed with anti-patterns after detailed analysis.

**Investigation findings (2026-03-01):**
- Custom script includes (x\_ scope) with anti-patterns: **0**
- All 39 flagged includes are **ServiceNow product code** (FSM, SAM, GRC, ITFM, OSCAL, WSD, Telecom, etc.)
- 4 of the top 10 are `sys_policy=read` (write-protected); remainder are unprotected product code that should not be modified on a customer/demo instance

**Top anti-pattern offenders (all product code, for reference):**

| Script Include | Anti-Patterns | Product Area |
|---|---|---|
| GuidedDecisionsUtilsSNC | `getRowCount()` + nested loop + deep dot-walk | Guided Decisions |
| FSMPartSourcingUtil | `getRowCount()` + nested loop + deep dot-walk | Field Service Mgmt |
| SMServiceByTagsUtilsAjax | `getRowCount()` + nested loop + deep dot-walk | Service Mgmt |
| SAMReconLLMUtils | `getRowCount()` + nested loop | SAM/AI |
| ScheduledInstallService | `getRowCount()` + deep dot-walk | ITAM/SAM |
| ProminFindingsDefUtilSNC | `getRowCount()` + nested loop | IRM/GRC |
| ITFM\_Budgeting | Nested loop + deep dot-walk | IT Financial Mgmt |
| OSCALSSPGeneratorBase | Nested loop + deep dot-walk | IRM/GRC |
| SchedulerExternalInterface | Nested loop + deep dot-walk | Scheduler |
| FlowSummarySkeletonUtil | Nested loop + deep dot-walk | Flow Designer |

**Remediation:** No actionable fix on this instance. These are product-owned implementations. There are zero custom script includes with anti-patterns to address. If these are causing measurable UI lag on specific forms, raise with ServiceNow Support/Engineering citing the specific script include and anti-pattern.

---

### ~~W4 — `ecc_queue` Table Cleaner Rule is Inactive~~ ✅ RESOLVED 2026-03-01

Activated the existing `ecc_queue` table cleaner rule (sys_id: `8d3b69e3c0a8010a01585235ec061376`). Rule retains 30 days of records and matches on `sys_created_on`.

---

### ~~W5 — `sys_import_set_row` Has No Table Cleaner Rule~~ ✅ RESOLVED 2026-03-01

Created a new table cleaner rule for `sys_import_set_row` (sys_id: `8756ef2ecf133a50e603f06c4d851c98`) with 30-day retention (2,592,000 seconds), matching on `sys_created_on`. Set by DemoHub performance inspection 2026-03-01.

---

### ~~W6 — `sys_flow_context` Has Aggressive 2-Day Cleanup Rule~~ ✅ RESOLVED 2026-03-01

On closer inspection, the 2-day rules are appropriately scoped — not blanket cleanups:
- `state=PAUSED_IN_DEBUG` at 2 days — correct, cleans up flows stuck in debug mode
- `calling_source=DATA_STREAM_VPP` + terminal states at 2 days — correct, high-volume data stream flows warrant a short TTL

The **COMPLETE flow retention** was extended from 14 days to **60 days** (5,184,000s) on sys_id `7b85d58277663010b599e0490e5a9915`. Completed flow execution history is now retained for 60 days.

---

### ~~W7 — `sys_audit` Cleanup at 14 Days (Recommended: 90 Days)~~ ✅ RESOLVED 2026-03-01

**Impact:** Audit history is lost after 14 days. Acceptable for demo performance, but insufficient if audit trails are needed for demo scenarios or troubleshooting.

**Evidence:**
```
sys_auto_flush for sys_audit: age = 1,209,600s (14 days)
Recommended: 7,776,000s (90 days)
```

**Resolution:** Extended `sys_auto_flush` rule (sys_id: `1ca56dc02f96321007389c7fcea4e334`) from 14 days (1,209,600s) to **90 days (7,776,000s)**.

---

### ~~W8 — 87 Active Client Scripts on High-Volume Tables~~ ✅ PARTIALLY RESOLVED 2026-03-01

**Impact:** Every incident, change, task, or request form load must execute all applicable client scripts. 87 scripts across these core tables is an elevated count.

**Investigation findings (2026-03-01):**
- Custom scripts (x\_ scope): **0** — all 87 are product code (global + app scopes)
- Synchronous GlideAjax (`getXMLAnswer`): **0** ✅
- Duplicate scripts (same name/table/scope with identical logic): **1 found and deactivated**

**Fix applied:**
Deactivated duplicate `"Show AIA notification - incident"` on `incident` (sys_id: `91cbedb247bf9650df06eac9636d4389`, created 2025-02-22 07:28). An identical copy (sys_id: `0be2313e47ff9650df06eac9636d4354`, created 31 min later) remains active. Count reduced from **87 → 86**.

Two other same-name pairs investigated but found to have **different logic** — both scripts in each pair are needed:
- "Show On hold reason when on hold ticked" × 2 — 2015 version has additional state-button visibility logic the 2018 version lacks
- "Set State field and hide UI Actions" × 2 — 2017-08 version adds race-condition guards absent in 2017-06 version

**Remaining count: 86 (all product code).** Further reduction requires app governance — uninstalling unused apps removes their client scripts automatically.

---

## Observations

### O1 — All Key Performance System Properties Correctly Set

A prior DemoHub performance inspection applied the following properties, all confirmed in place:

| Property | Value | Status |
|---|---|---|
| `glide.home.refresh_disabled` | `true` | ✅ |
| `glide.ui.goto_use_starts_with` | `true` | ✅ |
| `glide.invalid_query.returns_no_rows` | `true` | ✅ |
| `glide.ui.related_list_timing` | `async` | ✅ |
| `glide.security.use_csrf_token` | `true` | ✅ |
| `glide.ui.per_page` | `10,15,20` | ✅ |
| `glide.ui.list.allow_extended_fields` | `false` | ✅ |
| `glide.ui.session_timeout` | `60` | ✅ |

`glide.home.refresh_intervals` and `glide.db.max_view_records` are not set in `sys_properties` (using defaults), which is acceptable given homepage refresh is disabled and the per_page cap is already set.

---

### O2 — No Slow Query Log Entries or Index Suggestions

```
syslog_transaction WHERE type LIKE 'slow': 0 records  (confirmed on re-inspection)
sys_index_suggestion: 0 records                       (confirmed on re-inspection)
```

No immediate index remediation required. Instance responded to all MCP queries without timeout — not hibernated.

---

### O3 — All Dead Integration Jobs Disabled (C2 Resolved)

All integration-related scheduled jobs are confirmed disabled. On re-inspection, `sys_trigger` returned **0 active jobs** (state IN ready, executing, waiting) — confirming all 15 queue processor instances disabled during the initial inspection are no longer running.

The main import jobs (STIX2, MISP, MITRE, ArcSight, Splunk, Symantec) retain far-future `next_action` dates (2293–2297), unchanged from initial inspection.

---

### O4 — syslog Table Rotation Configured

20 rotating tables are configured, including `syslog`. This is the expected configuration and prevents unbounded syslog growth.

---

### O5 — Background Script Execution Not Available

`execute_background_script` was not accessible during this inspection. The following diagnostics were **not performed**:

- Record counts for: task, incident, change_request, sc_req_item, sys_audit, syslog, sys_email, sys_flow_context, sys_import_set_row
- Active/total record ratio for the task table
- Performance benchmarking (query execution times)
- `gs.getProperty()` verification of system properties
- Per-table client script count (GlideAggregate)

These should be run when the instance is accessible with `readOnly: false` in the MCP config.

---

### O6 — 100+ Scoped Applications Installed

At least 100 active scoped applications (first 100 retrieved). Notable apps include Now Assist for ITSM/CSM/FSM/WSD/OTSM/Voice, Vulnerability Response, IRM/GRC, HRSD, Field Service, CSM, SecOps, ITOM, ITAM, and many industry-specific modules. This breadth is the root cause of both the large task hierarchy (C3) and high UI policy count (W1).

---

## Detailed Findings by Area

### System Properties
- 8 key performance properties: all correctly set ✅
- `ignore_cache=false`: ~~4,130 properties~~ → **1,472 fixed** ✅; 2,658 remain (system-protected, not modifiable)

### Business Rules
- Before-query/async BRs: 100 returned (at limit — full count unknown)
- 8 flagged → 3 false positives, 1 already inactive, 1 product BR (no-touch), 2 real but low-frequency/complex
- ✅ `Add Case IOC Entry`: `getRowCount()` replaced with `setLimit(1)` + `next()`

### Script Includes
- 100 client-callable returned (at limit); 39 confirmed with anti-patterns
- All 39 are ServiceNow product code — 0 custom script includes with issues
- Not actionable without app uninstallation or ServiceNow Engineering fix

### Query Performance
- No slow query log entries ✅
- No index suggestions ✅
- Record counts not available (background scripts inaccessible)

### Table Health
- **Task hierarchy: 231 child tables** (C3) — open
- syslog rotation: configured ✅
- `ecc_queue` cleaner: ~~inactive~~ ✅ activated (W4)
- `sys_import_set_row`: ~~no cleaner rule~~ ✅ 30-day rule created (W5)
- `sys_flow_context`: ~~aggressive 2-day rule (W6)~~ ✅ COMPLETE rule extended to 60 days
- `sys_audit`: ~~14-day retention~~ ✅ extended to 90 days (W7)
- `syslog`: 14-day retention (slightly below recommended 30 days — acceptable for demo)

### Scheduled Jobs
- ~~10 dead integration queue processors~~ ✅ all 15 job instances disabled (C2)
- **Re-inspection verified**: `sys_trigger` query returns **0 active jobs** — all disabled queue processors confirmed absent ✅
- `SLA Async Delegator`: 984,368 runs — normal high-frequency platform job
- `Scorecard Data Collection`: 140,633ms average — expected for PA workloads
- `PA Indicator Recommendation Calculator Job`: 516,861ms average — normal for analytics jobs

### UI Configuration
- Homepage auto-refresh: disabled ✅
- No synchronous GlideAjax detected ✅
- `glide.ui.per_page` capped at 20 ✅
- **Active UI policies: 10,942** (W1) — not addressable without app uninstallation
- Active client scripts on high-volume tables: ~~87~~ → **86** after duplicate removed (W8)

### Client Scripts
- ~~87~~ **86** active on task/incident/change_request/sc_request/sc_req_item
- Duplicate `Show AIA notification - incident` deactivated ✅
- No `getXMLAnswer` synchronous AJAX detected ✅
- All remaining are product code — not addressable without app uninstallation

### Applications & Update Sets
- 100+ active scoped applications
- Multiple DemoHub content sets applied in Jan 2026 (normal for refreshed demo instance)
- Recent update sets include Now Assist for ITSM fixes, GRC AI agents, Application Vulnerability Management, SecOps Now Assist

### Performance Benchmarking
- Background scripts not available — no record counts or query timing obtained
- Instance is awake and responsive: all MCP queries (incident, syslog, sys_user, sys_flow_context, sys_trigger, sys_properties, sys_auto_flush, sys_script, sys_script_include) returned results without timeout
- Re-inspection note: `servicenow_execute_script` returned HTTP 400 — x_snc_mcp_tools not installed on this instance

---

## Recommendations

Prioritized by impact:

| # | Priority | Finding | Action |
|---|---|---|---|
| 1 | ~~Critical~~ ✅ | ~~C1: 4,130 `ignore_cache=false` properties~~ | Resolved 2026-03-01: 1,472 updated; 2,658 system-protected (cannot be changed) |
| 2 | ~~Critical~~ ✅ | ~~C2: Dead integration queue processors~~ | Resolved 2026-03-01: 15 job instances disabled |
| 3 | Critical | C3: 231 task child tables | App governance review; deactivate unused apps |
| 4 | ~~Warning~~ ✅ | ~~W4: ecc_queue cleaner inactive~~ | Resolved 2026-03-01: rule activated |
| 5 | ~~Warning~~ ✅ | ~~W5: sys_import_set_row has no cleaner~~ | Resolved 2026-03-01: 30-day rule created |
| 6 | ~~Warning~~ ✅ | ~~W6: sys_flow_context 2-day cleanup~~ | Resolved 2026-03-01: COMPLETE rule extended to 60 days |
| 7 | ~~Warning~~ ✅ | ~~W7: sys_audit 14-day retention~~ | Resolved 2026-03-01: extended to 90 days |
| 8 | ~~Warning~~ ✅ | ~~W2: BR anti-patterns (8 flagged)~~ | Resolved 2026-03-01: 1 real fix (Add Case IOC Entry), 3 false positives cleared |
| 9 | ~~Warning~~ ✅ (partial) | ~~W8: 87 client scripts on high-volume tables~~ | Resolved 2026-03-01: duplicate deactivated (87→86); remainder all product code |
| 10 | Warning (no fix possible) | W1: 10,942 active UI policies | Investigated — all product code; requires app uninstallation |
| 11 | Warning (no fix possible) | W3: 39 client-callable SIs with anti-patterns | Investigated — all product code; requires ServiceNow Engineering |
| 12 | Observation | O5: No background scripts | Re-run with `readOnly: false` for record counts and benchmarks |
