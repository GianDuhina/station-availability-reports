# DFMG Station Availability

Automated daily / weekly / monthly reporting on DFMG partner station availability —
pulled from Grafana/Thanos, rendered to HTML + PNG + CSV, published to GitHub Pages,
posted to Slack `#testing-scripts`.

This README captures everything we know about the system as of today: scripts, data
sources, hardware tiers, the precision architecture, the cache+live hybrid layer,
and the data-quality findings we've surfaced.

---

## Scripts

All Python scripts live in `scripts/` after the 2026-06-19 reorg. Generated
reports, cache, and `station_availability.log` continue to land at the project
root (the `BASE_DIR` walk-up in `station_availability.py` handles this).

| Script | Scope | What it covers |
|---|---|---|
| **`scripts/stationavailability_peraccount.py`** | Any one customer · SAR + path + power | **Production (V3.1).** Generates a single-customer report. CLI `--account <NAME>` with `--list-accounts` for the live roster. Default = dry-run; opt-in to publish via `--publish` + `--post-slack`. Includes V3.3 partner-feedback features (non-prod removal, stratified path table, sign-aware text restate). |
| **`scripts/stationavailability_allaccount.py`** | All 10 customers · one report | **Production (V3.1).** Overview rollup + native `<select>` dropdown for per-account drill-down. Same V3.3 partner-feedback features. Reads per-account caches written by peraccount. |
| **`scripts/dfmg_sar_availability.py`** | DFMG only · SAR metric only | Original DFMG production. Scheduled (daily/weekly/monthly + drain-queue plists). Hybrid cache. Now shares the V3.3 renderer improvements with the new account scripts via `render_path_connection_section`. |
| **`scripts/regions_stationavailability.py`** | All partners · grouped by region | NA / EU / AP rollup with dropdown selector + "All Regions - Overview" aggregate. Predecessor to the V3.0 account-dimension reports — kept for the regional-cut view. |
| `scripts/station_availability.py` | All partners · multi-region | Shared plumbing module (fetch / push / Slack / logo / cluster-failover). Imported by every other script. |
| `scripts/BETA_TESTING.py` | DFMG experiments | Permanent BETA twin of `dfmg_sar_availability.py`. Outputs use `BETA_` prefix so production reports aren't overwritten. |
| `scripts/BETA_TESTING_regions.py` | Regional experiments | Permanent BETA twin of `regions_stationavailability.py`. Same `BETA_` prefix isolation. |
| `scripts/BETA_TESTING_per_account.py` | Per-account experiments | **V3.0 BETA twin** of `stationavailability_peraccount.py`. Same isolation pattern. |
| `scripts/BETA_TESTING_all_accounts.py` | All-account experiments | **V3.0 BETA twin** of `stationavailability_allaccount.py`. Same isolation pattern. |
| `scripts/BETA_gcim_helper.py` | Jira GCIM fetcher + section renderer | **V3.4 (2026-06-27).** Single source of truth for the GCIM ticket-history section. Imported by both `stationavailability_peraccount.py` and `stationavailability_allaccount.py`. Auth via `JIRA_USER_EMAIL` + `JIRA_API_TOKEN` env vars. See [GCIM Ticket History integration](#gcim-ticket-history-integration-v34-shipped-2026-06-27). |
| `scripts/BETA_TESTING_gcim_render.py` | GCIM section visual sandbox | Mock-data harness for layout iteration without live Jira. Produces `BETA_dfmg_gcim_mock.{html,png}` under `output/accounts/`. |
| `scripts/dfmg_availability.py` | DFMG only · 3-metric variant (SAR + DCUPS + Orion) | Deprecated. Logs a `[DEPRECATED]` warning on startup. Not scheduled. Kept for reference. **One live edit (2026-06-27):** `render_station_row` gained an optional `gcim_info=None` kwarg consumed by the GCIM-augmented station rows. Backward-compatible — default None = original behaviour. |
| `scripts/station_check.py` | Per-station ad-hoc CLI diagnostic | Interactive; not a periodic report. |

> **Default targets:** for **partner-facing** changes, edit `scripts/stationavailability_peraccount.py` + `scripts/stationavailability_allaccount.py` (V3.1 production). For the legacy DFMG-only path, edit `scripts/dfmg_sar_availability.py`. BETA twins exist for experimental work; production stays untouched.

---

## DFMG EU station fleet (the population we report on)

Pulled live at run time from `edge_sitetracker_site_status{customer="DFMG"}`. Counts as of last audit:

| Hardware tier | All sites | Production (`noc_status==1`) | Continuous telemetry available? |
|---|---|---|---|
| **Adel UPS** (Full INFRA · GC11/GC13) | 174 | 170 | ✅ `power_management_dcups`, `ac_input_voltage`, `battery_*` |
| **APC Back-UPS Pro** (INFRA LITE · GC08 · IBR600C) | 88 | 80 | ❌ event-driven only (`sar_alert_power_source_*` counters) |
| **TOTAL** | **262** | **250** | |

### Geographic split

APC INFRA LITE is concentrated in **Western Europe**:
- 100% APC: France (29), UK (16), Austria (6), Denmark (4), Netherlands (3)
- 96% APC: Germany (26 of 27)

Adel Full INFRA dominates **Southern / Eastern Europe**: Spain (27), Italy (24), Norway (16), Finland (15), Greece (15), Poland (13), Sweden (20), Romania (10), Portugal (8), Bulgaria (6), Ireland (6), Croatia (4), Hungary (4), Czechia (2), Slovakia (2), Latvia (1).

### Why this matters

The two tiers are measured differently:
- **Adel sites**: continuous gauges (battery voltage, AC input voltage, state-of-charge). Direct measurement of outage duration via `ac_input_voltage` drop into the outage range.
- **APC sites**: no continuous telemetry exists in Prometheus — only NetCloud router GPIO transition alerts. Outage duration is *reconstructed* by pairing `backup_battery_total` (counter increments when grid lost) with `grid_power_total` (counter increments when grid restored).

The router itself knows more (GPIO 4 reads "APC UPS is Connected", GPIO 3 reads "Battery State is Normal/Charging") but `swift-nav/sar` doesn't currently scrape those pins into Prometheus. Closing that gap would require a PR to the SAR repo.

---

## Metrics and authority — the data sources powering the report

### Primary metrics (priority order)

| # | Metric | What it measures | How the script uses it |
|---|---|---|---|
| 1 | `edge_sitetracker_site_status{customer="DFMG"}` | Swift asset tracker's DFMG roster | **Authoritative partner filter** — defines the 262-station set |
| 2 | `sar_status_connection_online` | NetCloud router connection (gauge 1/0/-1) | **The uptime/downtime signal**. Day-by-day fetch @ 60s step, OR-across-instances via `max by (name)` |
| 3 | `orion_num_obs_cors_sum` | GNSS correction observations received | **Silent-station cross-match** — sites missing here aren't producing GNSS data |
| 4 | `orion_ntrip_client_bytes_sum` | NTRIP client bytes flowing | **Verification cross-check** — confirms offline vs hidden-failure diagnosis |
| 5 | `edge_sitetracker_noc_status` | Deployment lifecycle (1=prod, -10=decom, 4=pending, 2=degraded) | **Production vs test classification** |
| 6 | `power_management_dcups` | Adel UPS controller presence | **Tier detection** — sites in this set are Adel; complement is APC INFRA LITE |
| 7 | `ac_input_voltage` (Adel only) | Grid voltage at cabinet AC input | **Continuous Adel outage measurement** — drop into (-5V, 50V) range = outage |
| 8 | `sar_alert_*_total` (12 counters) | NetCloud event log | **APC outage reconstruction + silent-station diagnostics** |

### `sar_status_connection_online` value semantics (per [swift-nav/sar](https://github.com/swift-nav/sar))

| Value | Meaning | Script behaviour |
|---|---|---|
| `1` | Online | Counts as uptime second |
| `0` | Offline | Counts as downtime second |
| `-1` | Unknown / not yet observed | Treated as **missing** — does NOT count as downtime |

### The 12 SAR alert counters (event log)

Each counter increments when SAR sees a specific NetCloud alert message. Confirmed against the SAR source code (`station_monitoring/lib/metrics.py` + `netcloud/alert.py`).

| Counter | Triggered by NetCloud alert message |
|---|---|
| `sar_alert_connection_state_offline_total` | `'The device entered the "offline" state'` |
| `sar_alert_connection_state_online_total` | `'The device entered the "online" state'` |
| `sar_alert_connection_state_mdm_total` | `'WAN interface changed from ethernet-wan to mdm'` |
| `sar_alert_connection_state_wan_total` | `'Wan interface changed from mdm to ethernet-wan'` |
| `sar_alert_power_source_backup_battery_total` | `'Power Source is Backup Battery'` |
| `sar_alert_power_source_battery_exhausted_total` | `'Battery State is Exhausted'` |
| `sar_alert_power_source_battery_good_condition_total` | `'Battery State is Good Condition'` |
| `sar_alert_power_source_grid_power_total` | `'Power Source is Grid Power'` |
| `sar_alert_cabinet_door_open_total` | `'Cabinet Door is OPEN'` |
| `sar_alert_cabinet_door_closed_total` | `'Cabinet Door is CLOSED'` |
| `sar_alert_other_total` | Any unrecognised alert string |

Event-pattern → headline mapping (silent-station diagnostics):

| Pattern in last 30d | Headline |
|---|---|
| `battery_exhausted ≥ 1` | Power outage event — battery exhausted during window |
| `backup_battery ≥ 1` AND `grid_power ≥ 1` | Power blip — switched to backup battery and recovered |
| `backup_battery ≥ 1` only | Power issue — switched to battery (no grid-recovery alert) |
| `offline ≥ 1` AND `online ≥ 1` | Connection flapping — N offline, N online |
| `offline ≥ 1` only | Went offline N time(s) without recovery alert |
| `mdm ≥ 1` | Switched to mobile backup (WAN→MDM) N time(s) |
| `door_open` / `door_closed ≥ 1` | Site visit — cabinet door events |
| no events in 30d | No alert events in last 30d — silent before lookback window |

### Authority chain — what we trust vs what we infer

| Signal | Source | Level |
|---|---|---|
| Station belongs to DFMG | `edge_sitetracker_site_status{customer="DFMG"}` | **Authoritative** |
| Station is in production | `edge_sitetracker_noc_status == 1` | **Authoritative** |
| NetCloud router is online | `sar_status_connection_online` | **Authoritative** (SAR polls NetCloud directly) |
| Station produced GNSS obs | `orion_num_obs_cors_sum` series present | **Authoritative** |
| Power / connection events happened | `sar_alert_*_total` increments | **Authoritative** (NetCloud event log) |
| Site has Adel UPS | `power_management_dcups` series present | **Authoritative** |
| Site has APC UPS | NOT in `power_management_dcups` set | **Inferred** (by exclusion — no direct APC presence metric) |
| Trimble vs Septentrio fault | per-`antenna_instance` (0=Trimble, 1=Septentrio) obs presence | **Inferred** from convention |
| DMVPN status | inferred from "both receivers silent" | **Inferred** — `frr_bgp_peer_state` exists but no peer↔site mapping |

---

## Precision architecture (the hybrid cache+live layer)

### The problem we solved

The APC INFRA-LITE outage reconstruction queries SAR alert counters over the report window. Prometheus' query-point cap (~11,000 points per query) constrained the step:

| Window | At 60s step | At 300s step | 11k cap |
|---|---|---|---|
| Daily | 1,440 points | 288 points | ✅ both fit |
| Weekly | 10,080 points | 2,016 points | ✅ both fit |
| Monthly | **43,200 points** | 8,640 points | ❌ 60s fails, 300s fits |

So monthly was stuck at 5-minute precision — outages shorter than 5 minutes were invisible, and the coarse step inflated unpaired-event durations to phantom multi-day spans.

### The fix: autotune + per-day cache

**Layer 1 — autotune step (`_estimate_backup_pro_durations`).** Replaced the hardcoded `step_s=300` default with a call to `_autotune_step(start_dt, end_dt)`, which picks the smallest step that fits under the cap:

```python
def _estimate_backup_pro_durations(non_adel_names, name_to_site, start_dt, end_dt, step_s=None):
    if step_s is None:
        step_s = _autotune_step(start_dt, end_dt)
    ...
```

Result: daily → 60s, weekly → 60s, monthly → ~240–300s. No regression on monthly; daily/weekly get 5× sharper.

**Layer 2 — cache + live hybrid.** Daily runs persist their per-station emergency-power data to disk; weekly/monthly runs read the cached daily files and only gap-fill from Prometheus for missing dates. This means weekly/monthly reports get **1-minute precision at every window size** because the cache was generated at 60s step.

### Cache directory layout

```
station-availability/cache/
├── sar_daily_dfmg_2026-05-01.json   ← per-day, ~26 KB
├── sar_daily_dfmg_2026-05-02.json
├── ...
└── sar_daily_dfmg_2026-05-31.json
```

### Cache schema (v2)

```json
{
  "schema_version": 2,
  "partner": "DFMG",
  "window": {"start": "2026-05-01T00:00:00+00:00", "end": "2026-05-02T00:00:00+00:00"},
  "sar_agg_prod": { "<site>": {"name", "country", "region", "up_s", "down_s", "missing_s"} },
  "emergency_data": {
    "<site>": {
      "source": "adel" | "backups-pro",
      "duration_s": ...,
      "events": ...,
      "currently_on_battery": ...,
      "outage_windows": [[ts_start, ts_end], ...],
      "battery_runtime_s": ...,
      "raw_events": {
        "backup":    [ts, ts, ...],   // INFRA-LITE counter increments (timestamps)
        "grid":      [ts, ts, ...],
        "exhausted": [ts, ts, ...],
        "good":      [ts, ts, ...]
      }
    }
  },
  "adel_sites": [...],
  "nonprod_global": [...],
  "metadata": {"generated_at", "n_production_sites", "n_emergency_data_sites", "step_seconds_used"}
}
```

**What changed in v2 (2026-06-13):**
- `raw_events` added — stores per-site counter-increment timestamps so the weekly/monthly merger can **re-pair backup→grid events across day boundaries** (fixes the cross-midnight outage undercount).
- `duration_uncertain` removed from cache — spurious correction is now applied at **merge/render time** using window-end `recent_silent`, not baked in per-day. This means a site flagged "uncertain" on day N can be correctly reclassified on a weekly/monthly view if window-end evidence differs.
- Schema-version reader is strict: any v1 cache is logged as schema-drift and ignored — run `--rebuild-cache` to refresh.

### Storage footprint

| Scope | Bytes/day | Per year | Per 10 years |
|---|---|---|---|
| Typical (~30 KB) | 30,000 | 11 MB | 110 MB |
| Worst case (~40 KB) | 40,000 | 14.6 MB | 146 MB |

Trivial. Local-only storage; no remote backup needed.

### What the hybrid actually does at run time

For **daily** mode:
1. Fetch raw event timestamps + per-day outage windows
2. Compute `battery_runtime_s`
3. Write RAW v2 cache (no spurious correction baked in)
4. Apply spurious-correction + station-dark classifier in-memory using day-end `recent_silent`
5. Render with the corrected view

For **weekly / monthly** mode (`hybrid_fetch_emergency_power`):
1. Walk every day in the window
2. For each cached date → load from disk (~<1 ms each)
3. For each missing date → query Prometheus for that day, write a gap-fill v2 cache
4. **Re-pair INFRA-LITE backup/grid events across the FULL window** (so a 23:55→02:30 outage is one continuous duration, not two split sub-day fragments)
5. Merge contiguous Adel outage windows (catches Adel cross-midnight too)
6. Apply spurious-correction + station-dark classifier using **window-end** `recent_silent`

### Battery-exhausted + station-dark classification (new 2026-06-13)

Two distinct cases used to collapse into `duration_uncertain`:

| Pattern | OLD classification | NEW classification |
|---|---|---|
| Unpaired `backup_battery` alert + station still producing observations | `duration_uncertain` (excluded from totals) | `duration_uncertain` (excluded from totals) — unchanged |
| `battery_exhausted` ≥ 1 alert AND station is in `recent_silent` (no obs in last 24h) | `duration_uncertain` (excluded — data lost) | **`station_dark = True`** — duration capped at UPS battery life (Adel ~4 h / APC ~1 h), counted in totals |

So an outage that exceeded UPS capacity (battery died, station went dark) now contributes a realistic duration based on hardware battery-life limits, instead of being silently dropped. Recovers signal we previously threw away.

### What is NOT cached (still hits Prometheus on every weekly/monthly run)

The cache is scoped narrowly to the slowest single piece (INFRA-LITE counter timeline reconstruction). Everything else still queries live:

| Query | Why not cached |
|---|---|
| `sar_status_connection_online` day-by-day fetches | Already fast; drives availability % |
| `edge_sitetracker_site_status` (roster) | Single instant query, very cheap |
| `edge_sitetracker_noc_status != 1` (non-prod) | Cheap |
| `orion_num_obs_cors_sum` (silent cross-match) | Window-wide, depends on full window |
| Silent diagnostics (obs, NTRIP, NOC tracker, alerts) | Per-site, only for silent sites |
| `power_management_dcups` (Adel detection) | Cheap |
| `last_over_time(orion_num_obs_cors_sum[24h])` (recent-silent) | Must be fresh |

The log line `[HEADLINE] Emergency power: N sites affected · D total on-battery (hybrid cache+live)` confirms which path was taken.

### Performance

| Run | Wall-clock | What hit Prometheus |
|---|---|---|
| Daily | ~30–40 s | All standard daily queries + raw events fetch + cache write |
| Weekly (all 7 days cached) | ~30 s | Everything except emergency_power |
| Weekly (with gap-fill) | ~60 s | Standard + per-day emergency_power for missing days |
| Monthly (all 31 days cached) | ~65–70 s | Everything except emergency_power |
| Monthly (single-query, pre-hybrid) | ~3–5 min | Everything including 30-day SAR counter range query |

---

## Cache freshness & recovery (T-1 regen + `--rebuild-cache`)

### Automatic T-1 sanity check

At the start of every **daily** run (unless `--dry-run` or `--no-t1-regen`), the script checks yesterday's cache:

- If the cache file is **missing** OR has fewer than **50 production sites** (suggests Prometheus had errors during yesterday's run), the script spawns a daily subprocess for yesterday with publish + Slack enabled.
- The subprocess regenerates yesterday's cache, re-renders the HTML/CSV/PNG, and republishes — surfacing the "🔄 RE-RUN — corrected" version to Pages and Slack.
- Recursion is prevented by passing `--no-t1-regen` to spawned subprocesses, so the chain caps at T-1 (one day). Deeper holes require the manual `--rebuild-cache` flag below.

### Manual rebuild — `--rebuild-cache <SPEC>`

Use when a known date range is bad (Prometheus retention rollover, cluster failover tail, lost upstream data, etc.).

```bash
# One specific day
python3 dfmg_sar_availability.py --rebuild-cache 2026-06-10

# A whole month
python3 dfmg_sar_availability.py --rebuild-cache 2026-05

# Last N days
python3 dfmg_sar_availability.py --rebuild-cache 7d

# Also republish the regenerated reports (default is cache-only, silent)
python3 dfmg_sar_availability.py --rebuild-cache 2026-05 --publish-rebuilds
```

Behaviour: deletes the existing cache for each affected date, spawns a daily subprocess with `--dry-run` (cache-only by default — no Slack spam), regenerates the v2 cache file. Adding `--publish-rebuilds` opts into Pages + Slack for each date (useful for backfilling one or two specific reports).

---

## Cluster-failover resilience

### What happened 2026-06-10

Swift's edge Thanos infrastructure flipped active/passive: **eu-prod-2 → passive, eu-prod-1 → active**. Confirmed via the AWS Route 53 mTLS casters dashboard at the time of the change. The passive cluster's Thanos receivers (`cluster="eu2"`) remained reachable for historical queries but with occasional `rpc error: code = Unavailable` warnings for adjacent timestamps.

### Symptoms we hit

- `power_management_dcups` instant queries returning **0 series** during certain windows, causing all Adel sites to silently misclassify as INFRA-LITE.
- Roster + non-prod queries similarly flickered around the failover boundary.

### Fixes (active 2026-06-13)

| Query | Old | New | Why safe |
|---|---|---|---|
| `fetch_adel_equipped_sites` | instant query | `last_over_time[7d]` | Adel hardware-tier transitions are months-to-years scale |
| `fetch_global_nonprod_sites` | instant query | `last_over_time[24h]` | `noc_status` changes are administrative (days-to-weeks scale) |

### `[PROM-PASSIVE]` and `[PROM-WARN]` log tags

`_instant_query` and the v2 raw-events range query both extract Prometheus' `warnings` field from the response and log them as `WARNING`:

- `[PROM-PASSIVE]` — warnings that reference a known passive Thanos cluster (`cluster="eu2"` in the current topology). Flagged distinctly so operators can spot them in the monitor.
- `[PROM-WARN]` — any other Prometheus warning (typically dev or transient backend issues).

If the passive cluster set changes, update `PASSIVE_THANOS_CLUSTERS = {"eu2"}` at the top of the cache section in `dfmg_sar_availability.py`.

---

## Data-quality finding — phantom outages eliminated

The pre-hybrid monthly (single 30-day query at 5-min step) had a bug class we didn't notice until the hybrid surfaced it:

**Symptom (OLD monthly for May 2026):**
- FRMG0xFRA: 2 events, **5d 9h 50m** duration
- FRBE0xFRA: 5 events, **1d 18h 40m** duration
- Total fleet on-battery: **9d 23h 35m**

**Same window, NEW hybrid+autotune:**
- FRMG0xFRA, FRBE0xFRA: classified as `duration_uncertain` (excluded from totals)
- Total fleet on-battery: **2d 8h 12m**

**Root cause:** an unpaired `backup_battery` event (NetCloud alert sent, recovery alert lost or never arrived) gets extended to "end of window" by the script. In a 30-day window, that becomes a multi-day phantom. The per-day cache approach naturally caps any unpaired event at 24 hours and the spurious-correction logic reclassifies it as uncertain.

**Net effect:** monthly reports are now ~4× more conservative for fleet on-battery totals, and the previously inflated single-site multi-day durations are gone. The 7d 15h delta between OLD and NEW totals is almost entirely accounted for by FRMG (5d 9h) + FRBE (1d 18h).

**Action item carried forward:** the two sites that consistently produce these phantom events (FRMG, FRBE) likely have a quirky NetCloud alert delivery pattern. Worth investigating router-side whether recovery alerts are being lost upstream.

---

## How the report is constructed (data flow)

```
1. Fetch authoritative DFMG roster                ──► 262 sites with customer="DFMG"
        │
        ▼
2. Fetch sar_status_connection_online day-by-day ──► raw uptime samples @ 60s step
        │
        ▼
3. Rollup per station: up_s, down_s, missing_s
   (ignore -1 unknown samples · keep only sites in roster)
        │
        ▼
4. Compute headline KPIs:
     availability = up_s / (up_s + down_s)
     affected     = count of stations with down_s > 0
        │
        ▼
5. For each silent station (no obs in window):
     a. noc_status            ─► if ≠ 1, short-circuit as "non-prod"
     b. NetCloud alerts (30d) ─► event story (the human remark)
     c. Orion per-receiver    ─► Trimble/Septentrio/DMVPN inference
     d. NTRIP bytes           ─► verification of offline diagnosis
        │
        ▼
6. Emergency power section:
     daily mode   ─► fetch + correct + battery_runtime + write cache
     weekly/mthly ─► read cache + gap-fill missing days + merge
        │
        ▼
7. Render:
     · Row 1     — 4 KPI panels (Avail, Had Downtime, Downtime %, Data Window)
     · Row 1B    — SAR Station Health (uptime % green / downtime % red + lists)
     · Emergency power — 3 KPI tiles + Power Infrastructure Health + per-site table
     · Path Connections — 6 KPI tiles + per-station table (status · uptime · downtime · primary · secondary · current) + DATA ACCURACY + STATUS GUIDE
     · Grid      — Active (worst-first) → SILENCED (amber) → NON-PROD (grey)
     · Footer    — Silent Stations remark table with per-station diagnostic
```

---

## CLI

Default behaviour: **publish to GitHub Pages AND post to Slack `#testing-scripts`** on every run.

```bash
# Default = previous full calendar month
python3 dfmg_sar_availability.py

# Specific period
python3 dfmg_sar_availability.py --mode daily   --date 2026-05-29
python3 dfmg_sar_availability.py --mode weekly  --date 2026-W21
python3 dfmg_sar_availability.py --mode monthly --date 2026-04

# Opt-outs
python3 dfmg_sar_availability.py --mode daily --dry-run            # skip Pages push + Slack
python3 dfmg_sar_availability.py --mode daily --no-slack           # push to Pages, skip Slack
python3 dfmg_sar_availability.py --mode daily --no-t1-regen        # skip T-1 auto-regen

# Manual cache recovery
python3 dfmg_sar_availability.py --rebuild-cache 2026-06-10        # one day, cache-only
python3 dfmg_sar_availability.py --rebuild-cache 2026-05           # whole month, cache-only
python3 dfmg_sar_availability.py --rebuild-cache 7d                # last 7 days, cache-only
python3 dfmg_sar_availability.py --rebuild-cache 2026-06-10 --publish-rebuilds  # republish too
```

### Flag reference

| Flag | Effect |
|---|---|
| `--mode {daily,weekly,monthly}` | Report mode (default `monthly`) |
| `--date SPEC` | Anchor date in mode-specific format |
| `--dry-run` | Skip both GitHub Pages push AND Slack post |
| `--no-slack` | Push to Pages but skip Slack |
| `--no-t1-regen` | Disable the auto T-1 cache health check for this run (used internally by spawned subprocesses to prevent recursion) |
| `--rebuild-cache SPEC` | Regenerate cache for `YYYY-MM-DD` / `YYYY-MM` / `Nd` — cache-only by default |
| `--publish-rebuilds` | When used with `--rebuild-cache`, also publish each regenerated daily to Pages + Slack |

### Date format per mode

| Mode | `--date` accepts | Cache behaviour |
|---|---|---|
| `daily` | `YYYY-MM-DD` (one calendar day) | **Writes** cache after run |
| `weekly` | `YYYY-MM-DD` (any day in week) OR `YYYY-Www` | **Reads** cache + gap-fills missing |
| `monthly` | `YYYY-MM` OR `YYYY-MM-DD` (snaps to first of month) | **Reads** cache + gap-fills missing |

### What gets generated per run

Local files at `/Users/gian.duhina/projectclaude/station-availability/`:

```
dfmg_sar_<period>_<stamp>.html
dfmg_sar_<period>_<stamp>.png
dfmg_sar_<period>_<stamp>.csv
cache/sar_daily_dfmg_<YYYY-MM-DD>.json   (daily mode only, or gap-fill in weekly/monthly)
```

Pushed to GitHub Pages at https://gianduhina.github.io/station-availability-reports/ in three places:

| Path | Purpose |
|---|---|
| `dfmg_sar_<period>_<stamp>.<ext>` | Canonical dated archive |
| `latest_dfmg_sar_<mode>.<ext>` | Convenience alias (overwritten per run) |
| `snapshots/dfmg_sar_<period>_<stamp>_<utcstamp>.<ext>` | Unique-per-publish (Slack link target — no stale cache) |

---

## Scheduled runs (launchd)

Times are local PST (UTC+8). Plists live in `~/Library/LaunchAgents/`.

| Job | Schedule (local) | UTC | What it does |
|---|---|---|---|
| `com.station-availability.dfmg-sar.daily` | every day 23:00 | 15:00 UTC | Previous day window + writes cache |
| `com.station-availability.dfmg-sar.weekly` | Monday 23:15 | Mon 15:15 UTC | Previous full ISO week (reads cache) |
| `com.station-availability.dfmg-sar.monthly` | day-1 of month 23:30 | 15:30 UTC day-1 | Previous calendar month (reads cache) |
| `com.station-availability.monitor` | RunAtLoad + KeepAlive | — | Keeps tmux monitor session alive |

⚠ **launchd does NOT catch up missed schedules.** If the Mac is off at fire time, that run is skipped. Panel 3 of the monitor flags this (daily >25h, weekly >8d, monthly >35d).

Force-fire a scheduled job manually:

```bash
launchctl start com.station-availability.dfmg-sar.daily
launchctl start com.station-availability.dfmg-sar.weekly
launchctl start com.station-availability.dfmg-sar.monthly
```

Reload a plist after editing:

```bash
launchctl unload ~/Library/LaunchAgents/com.station-availability.dfmg-sar.daily.plist
launchctl load   ~/Library/LaunchAgents/com.station-availability.dfmg-sar.daily.plist
```

---

## Monitor (tmux session: `station-availability`)

Three live panels. Auto-restarts on login via launchd.

| Panel | Content | Refresh |
|---|---|---|
| 1 (top) | launchd job status — which jobs loaded, last exit code | every 10s |
| 2 (middle) | **Live structured tail of `station_availability.log`** — colour-coded by tag | streams real-time |
| 3 (bottom) | Failure & stale-content checks: errors, partial fetches, Pages-vs-local md5 mismatch, schedule freshness | every 30s |

```bash
tmux attach -t station-availability      # view live
Ctrl+B then D                            # detach (keeps running)
bash monitor.sh --kill                   # nuke the session
bash monitor.sh --detached               # (re-)create session, don't attach
```

### Structured log format (panel 2 colour coding)

Each line: `YYYY-MM-DD HH:MM:SS UTC  LEVEL    [TAG] message`

| Tag | Colour | Phase |
|---|---|---|
| `[START]` / `[COMPLETE]` / `====` | bold green | run boundaries |
| `[FETCH]` / `[AGGREGATE]` | cyan | data pipeline |
| `[CACHE]` | bright cyan | cache read/write |
| `[HEADLINE]` | bold yellow | computed KPIs |
| `[OUTPUT]` | green | file generation |
| `[GITHUB]` | magenta | Pages push |
| `[SLACK]` | bright blue | Slack post |
| `[DRY-RUN]` | grey | dry-run skip |
| `ERROR` / `WARNING` | red / yellow | failure |

---

## Output / Pages URLs

| Local file | Pages URL |
|---|---|
| `dfmg_sar_daily_<YYYYMMDD>.{html,png,csv}` | `https://gianduhina.github.io/station-availability-reports/dfmg_sar_daily_<YYYYMMDD>.html` |
| `dfmg_sar_weekly_<YYYYWww>.{html,png,csv}` | `.../dfmg_sar_weekly_<YYYYWww>.html` |
| `dfmg_sar_monthly_<YYYYMM>.{html,png,csv}` | `.../dfmg_sar_monthly_<YYYYMM>.html` |
| (aliases) | `.../latest_dfmg_sar_<mode>.<ext>` |

**Slack `Full report` link** always points to a unique `snapshots/.../<utcstamp>.html` path so browser/proxy/CDN caches cannot serve stale content.

---

## Path Connections section (shipped 2026-06-18, refined 2026-06-19)

A per-station section that surfaces **wired↔cellular failover behavior** alongside uptime/downtime. Sits between Emergency Power and the Grid section.

### What the columns mean

| Column | Source | Confidence |
|---|---|---|
| **STATION** | `name` label from `sar_status_connection_online` | Direct |
| **STATUS** | rules-based classifier on event counters + uptime (see below) | Derived |
| **UPTIME** | `sar_status_connection_online == 1` time + inline bar + trend arrow vs T-7 | Direct |
| **DOWNTIME** | `sar_status_connection_online == 0` time | Direct |
| **PRIMARY (WAN)** | `up_s − cellular_uptime_s` | Inferred (depends on #6) |
| **SECONDARY (cellular · est)** | per-event session timing inferred from `sar_alert_connection_state_mdm_total` events, capped at 4 h/session | **Estimated** — best-effort, no NetCloud session-data access |
| **CURRENT** | live `sar_status_connection_online` value + last-2h event window | Direct (WAN/Offline) + inferred (Cellular detection) |

Surfaced at the bottom of the section as a **DATA ACCURACY** reference panel so partners see explicit confidence levels per column.

### Confidence labels — what they mean

The DATA ACCURACY panel labels every column with one of four confidence levels. Understanding the difference is critical for interpreting the report:

| Label | What it means | Where the answer comes from |
|---|---|---|
| **✓ DIRECT** | Observed | Read straight from a Prometheus gauge or counter at query time. Highest trust. |
| **↔ INFERRED** | Deduced | Derived via fixed logic from other DIRECT measurements. Logically correct *if* its inputs are correct — trace the source chain to gauge trust. |
| **≈ ESTIMATED** | Approximated | Numeric best-effort when no direct metric exists. Carries an unknown error margin. |
| **✗ UNAVAILABLE** | Not knowable | The data point isn't exposed by any source we can read. Requires upstream work (SAR PR, NetCloud API) before it can be surfaced. |

#### Why split INFERRED from ESTIMATED

| | Inferred | Estimated |
|---|---|---|
| **Method** | Boolean / arithmetic logic | Numeric approximation |
| **Failure mode** | Wrong only if input is wrong (deterministic chain) | Wrong by some unknown magnitude even with perfect input (sampling, capping, assumed averages) |
| **Example** | Primary duration = uptime − cellular | Cellular session = mdm event → next event, capped at 4 h |
| **What to do if it looks off** | Audit the input chain | Check whether the sampling/cap assumption holds |

#### Two worked examples from this report

**1. Primary (WAN) duration — INFERRED**

There is no Prometheus metric `wan_uptime_total`. But we know:
- `up_s` — total time online (gauge==1) — DIRECT
- `cellular_uptime_s` — cellular session time — ESTIMATED

So we deduce: `primary_uptime_s = up_s − cellular_uptime_s`. Pure arithmetic. Reliable *if* the cellular estimate is right.

**2. "Currently on Cellular" detection — INFERRED**

There is no `currently_on_cellular_now` metric. We reason from three DIRECT signals at query time:
```
IF gauge == 1 (online right now)
AND mdm_2h > 0 (failed over within last 2h)
AND mdm_2h > online_2h (more failovers than recoveries)
THEN currently_on_cellular = True
```
The deduction is exact, but it can be a false positive if the station already recovered to WAN between the mdm event and report time. That's the kind of trust-chain audit INFERRED data invites.

#### Practical takeaway

When you see an **INFERRED** label on a column, treat the number as "logically correct if its inputs are correct" — look at what the source row says feeds it. The DATA ACCURACY table deliberately names the source for each row so readers can trace inferred → direct dependencies and decide how much to trust the final number.

### Status classifier rules (first match wins)

| Rule | Status | Why |
|---|---|---|
| `up_s + down_s > 0` AND `up_s < 60s` | `offline-entire-window` | Long-silent station — counters look quiet only because router was unreachable to fire any event. Critical fix vs falsely flagging as "stable" |
| `mdm_events == 0` AND `offline_events == 0` | `stable` | No transition events at all |
| `mdm_events == 0` AND `offline_events > 0` | `outage-no-cellular` | Went offline but no failover happened |
| `currently_on_cellular` heuristic fires | `on-cellular-now` | Online + mdm event in last 2h + more mdm than online events |
| `mdm_events ≥ 5` AND `online_events ≥ mdm_events - 1` | `flapping` | Many failovers WITH matching recoveries — chronic cycling |
| `mdm_events ≥ 5` AND `online_events < mdm_events - 1` | `stuck-degraded` | Failovers WITHOUT paired recoveries — sticky on cellular |
| else | `brief-failover` | 1–4 failover events, recovered |

### Caveats and current limitations

1. **Per-SIM split (SIM1 vs SIM2) is not exposed by Prometheus.** Requires a `swift-nav/sar` PR to scrape per-slot gauges. Until then, "cellular" is a binary.
2. **Cellular session durations are estimated.** Each mdm event is paired with the next online/offline event, capped at 4 h. Sessions longer than the cap are truncated; phantom-long sessions are not surfaced.
3. **Finland anomaly (May 2026 monthly).** 14 of 15 Finnish stations classified `stuck-degraded` while showing **100% availability**. The mdm-event rate in Finland is 5–10× the rest of the fleet for the same observed uptime. Two interpretations: either Finland really is on cellular most of the time (cost story), or NetCloud emits spurious mdm alerts there (counter-semantics story). Distinguishing requires NetCloud Data Usage API access.
4. **SAR `sar_status_connection_online` ≠ NTRIP service availability.** SAR measures whether the **NetCloud router** is reachable for management. The station hardware may still be streaming NTRIP via an alternate network path (direct internet, alt-VPN, etc.). For IRCO/HRPL in May 2026, the Grafana `orion_ntrip_client_bytes_sum` panel shows the station serving data most of the window even though SAR shows ~70%+ downtime. Adel's "Network Unreachable" corroborates SAR (both read the same NetCloud channel). For partner-facing service availability, cross-check with the Orion-bytes panel.

### Headline accounting (sanity invariant)

For each station: `up_s + down_s = window_seconds`. The split is then:
- `primary_uptime_s = up_s − cellular_uptime_s`  (time online via WAN)
- `cellular_uptime_s` ≤ `up_s`  (cellular only when ALSO online)
- `down_s` = fully offline (both paths failed)

CURRENT-column distribution should equal the production-station count exactly (e.g., May 2026 monthly: 244 WAN + 5 Offline + 1 Cellular = 250 prod).

---

## Regional report (shipped 2026-06-20, polished 2026-06-21)

A multi-region twin of the DFMG production report covering **NA / EU / AP +
an All Regions overview**. Default dropdown view = "All Regions - Overview";
drill into a specific region via the tab selector at the top.

### Why it exists

The DFMG report covers 250 production stations in EU. Swift has ~700
production stations across NA + EU + AP. Before this report there was no
single view that let an executive scan "how healthy is the fleet by region?"
without running each region's roster manually.

### Region inventory (Production filter applied, 2026-06-20 audit)

| Region | Stations | Customers (count) |
|---|---|---|
| **NA** | 330 | Swift 261 (USA) · Telus 69 (CAN) — Swift's CAN station `ABCG` overridden to Telus |
| **EU** | 251 | DFMG 250 (22 European countries) · Swift 1 (LVA test). UAE/Etisalat folds into EU (EMEA convention) — currently 0 prod |
| **AP** | 116 | KDDI 62 (JPN) · KT 20 + SKT 20 (KOR) · TaiwanMobile 18 (TWN) · Singtel 4 (SGP) · Airtel 5 (IND, currently non-prod) |
| **All Regions** | 697 | Aggregated overview only |

### Country → region mapping

| Region | ISO-3 codes |
|---|---|
| NA | USA, CAN, MEX, BRA, PER, ARG, ECU, URY, GUF, CUW (Americas) |
| EU | AUT, BGR, CZE, DNK, DEU, ESP, FIN, FRA, GBR, GRC, HRV, HUN, IRL, ITA, LVA, LTU, EST, NLD, NOR, POL, PRT, ROU, SVK, SVN, SWE, CHE, BEL, LUX, ISL, MLT, CYP, ALB, BIH, MKD, MNE, SRB, UKR, BLR, MDA, **ARE** (UAE folded in), **ZAF/GHA/MOZ/NAM/CPV** (Africa folded in via EMEA convention) |
| AP | JPN, KOR, TWN, SGP, IND, CHN, HKG, MAC, MYS, THA, VNM, PHL, IDN, AUS, NZL, MNG, NPL, UZB, MUS, FJI, NCL, PYF |

One-off overrides (until upstream Prometheus labels are corrected):
- `ABCG` (Swift CAN station): customer remapped Swift → Telus
- `ATGJ` (Airtel station with missing country label): country defaulted to IND

### Filters

A station enters the report only if it has BOTH:
1. SAR (`sar_status_connection_online`) OR Orion (`orion_num_obs_cors_sum`) telemetry in the last 30 days
2. `edge_sitetracker_noc_status == 1` (production rotation)

This drops 240 of 949 roster sites — mostly Swift reference receivers
(`RX-*`, `CDDIS-*`, `SAPOS-*`) that don't have NetCloud monitoring.

### Hybrid per-region cache

Mirrors production's pattern with region-specific filenames:

```
cache/sar_daily_region_NA_<date>.json
cache/sar_daily_region_EU_<date>.json
cache/sar_daily_region_AP_<date>.json
cache/sar_daily_region_ALL_<date>.json
```

The NA/EU/AP caches are shared between `regions_stationavailability.py` and
`BETA_TESTING_regions.py`. The ALL cache aggregates everything.

Rebuild a date range with `--rebuild-cache YYYY-MM-DD` / `YYYY-MM` / `Nd`.

### All Regions overview view

Default view when the report opens. Shows:
- Brand bar + dropdown selector (sticky)
- Executive summary line (1 sentence — "All Regions sustained X% availability across 697 stations …")
- 4 KPI tiles (Availability, Had Downtime, Downtime %, Data Window)
- Station Health panel (Uptime/Downtime split + station lists)
- Emergency Power 3-tile grid + Power Infrastructure Health
- Path Connections 6 KPI tiles

**Detail sections intentionally stripped from the overview** (per partner UX):
- Per-station uptime/downtime bars
- Silent Stations table
- EMERGENCY POWER · affected sites table
- ADVISORY bar
- Trend legend + per-station Path Connections table
- DATA ACCURACY panel
- STATUS GUIDE table

Per-region tabs (NA / EU / AP) keep the full detail.

---

## Account-dimension reports (V3.0 → V3.3, shipped 2026-06-25/26/27)

Successor to the regional report. Where the regional cut groups stations into NA / EU / AP buckets, the **account-dimension** cut groups them by **customer** — exactly matching how partners and Customer Success think about the fleet.

### Authoritative customer roster (live as of 2026-06-25)

Pulled from `count by (customer) (last_over_time(edge_sitetracker_site_status[30d]))` at run time:

| Customer | Unique sites | Country footprint |
|---|---:|---|
| Swift | 463 | 49 countries (USA-dominated) |
| DFMG | 262 | 22 EU countries |
| Telus | 73 | CAN=72, USA=1 |
| KDDI | 72 | JPN |
| KT / SKT | 20 each | KOR |
| TaiwanMobile | 19 | TWN |
| Etisalat | 10 | ARE |
| Airtel | 6 | IND |
| Singtel | 4 | SGP |
| **TOTAL** | **949** | **10 customers** |

The order is **fleet-size descending** — drives both the dropdown order and how accounts iterate during runs.

### V3.0 — BETA scaffolds (2026-06-25)

Two new BETA scripts built and smoke-tested:
- `BETA_TESTING_per_account.py` — single-customer report
- `BETA_TESTING_all_accounts.py` — Overview button + native `<select>` dropdown across all customers

Decisions locked at brainstorm time:
- Dropdown rows show **customer name only** (no site count clutter)
- Order is **fleet size descending**
- Small partners listed **individually**, not bucketed
- **Simple native `<select>`** dropdown (not custom combobox) — free accessibility + keyboard nav
- For accounts with 100% non-prod sites (Etisalat, Airtel currently): render an amber "DATA QUALITY" advisory card instead of fake KPIs

### V3.1 — Production promotion (2026-06-25)

Promoted to:
- `scripts/stationavailability_peraccount.py` (706 lines)
- `scripts/stationavailability_allaccount.py` (614 lines)

Key production-vs-BETA differences:
- **Default is dry-run** (inverted from `dfmg_sar_availability.py`). Pass `--publish` to upload to Pages and `--post-slack` to post Slack — both opt-in until accounts are vetted externally.
- **Per-invocation log file** alongside each output (`stationavailability_<ACCOUNT>_<period>.log`) — every line still mirrors to the shared `station_availability.log` for monitor.sh continuity.
- **CSV export** added (`write_sar_csv` shared with prod DFMG; allaccount writes a consolidated CSV with an `account` column for filter/pivot in Excel).
- **Caches shared with BETA** (`account_<CUSTOMER>` CACHE_PARTNER) — no duplicate Thanos fetches.

### V3.2 — First live monthly publish (2026-06-25)

8 monthly reports for May 2026 landed in `#testing-scripts`. Surprises captured for follow-up:
- Singtel monthly aborts with exit 2 — sitetracker shows all 4 Singtel sites as non-prod in May 2026 even though they're prod in June. `noc_status` is **time-varying**; daily reports filter against today's sitetracker, monthly filters against end-of-month's. A station can flip prod↔non-prod between months without any code change.
- GitHub API timeout wedge: 2-hour Pages-API outage on 2026-06-25 caused per-account publish to spin for ~30 min/account on retries before giving up. `_gh_put` has 3× retries at 60 s timeout — worth tightening for next-blip resilience.

### V3.3 — DFMG partner-feedback features (2026-06-26)

Triggered by DFMG (Thierry) feedback after they reviewed the May 2026 report:

1. **Non-prod sites removed from rendered sections.** The orange "NON-PRODUCTION" banner section is gone for partner-facing reports. Excluded site list goes to:
   - `station_availability.log` (each line tagged `[NON-PROD-EXCLUDED]` with site / name / country)
   - A side CSV `stationavailability_<ACCOUNT>_<period>_excluded.csv` (audit-only, not pushed to Pages)
   - A subtle italic footer note inside the HTML: *"N decommissioned or non-production site(s) excluded from this report. Full list available in the operator log."*
   - A Slack post addendum: `:wastebasket: N decommissioned sites excluded — see operator log.`
   - Silenced **production** stations stay visible (real ops signal, not decommissioned noise).

2. **Path Connection table stratified.** "0s and 0s" rows (`down_s == 0 AND cellular_uptime_s == 0`) are hidden behind a green ✓ NO IMPACT summary card:
   ```
   ┌─[ ✓ NO IMPACT ] N stations with no measurable downtime or cellular usage ──┐
   │ Primary path served traffic the entire window.                              │
   │ ▸ Show all N no-impact station(s)                                           │
   └─────────────────────────────────────────────────────────────────────────────┘
   ```
   The collapsible toggle reveals the full hidden table for anyone who wants to verify nothing's hidden. Main table now only shows stations with actual events to investigate.

3. **Sign-aware text restate.** Literal `0s` replaced with semantic text that carries the news:
   - `DOWNTIME = 0` → "no downtime" (green) — good news
   - `PRIMARY = 0` → "offline entire window" (red) — bad news, station never came online via wired
   - `SECONDARY = 0` when `PRIMARY > 0` → "primary only" (muted green) — wired path held
   - `SECONDARY = 0` when `PRIMARY = 0` → em-dash (avoids the "primary only" lie for offline-entire-window stations)

4. **Root-cause fix for the 400 Bad Request errors** that had quietly truncated path-connection event data on the first May 2026 publish. The fix wasn't URL length (which was the initial suspicion) — it was Thanos' 11 000-point-per-series cap being violated by a monthly window × 60 s step query. Solution: every range-query fetcher now uses `_autotune_step(start, end)` to pick a step that fits under the cap. Also switched the same fetchers GET→POST as defensive prophylaxis against future URL-bloat regressions. Affected fetchers:
   - `fetch_path_raw_events` (path connection alert counters)
   - `fetch_raw_alert_events` (Back-UPS Pro power-source counters)
   - `_estimate_backup_pro_durations` (counter-pair reconstruction)

### File-naming convention (V3.1 production)

```
output/peraccount/stationavailability_<ACCOUNT>_<period>.{html,png,csv,log}
output/peraccount/stationavailability_<ACCOUNT>_<period>_excluded.csv      (V3.3 side file)
output/allaccount/stationavailability_allaccount_<period>.{html,png,csv,log}
```

Examples:
- `stationavailability_DFMG_monthly_202605.html`
- `stationavailability_SWIFT_daily_20260624.csv`
- `stationavailability_allaccount_monthly_202605.png`

ACCOUNT is uppercased in filenames. Period suffix is always present so daily / weekly / monthly outputs never collide on disk.

### Caches

- BETA + V3.1 production share `cache/sar_daily_account_<CUSTOMER>_<date>.json` (no duplicate Thanos fetches between them).
- All-accounts union uses `account_PROD_ALL` to stay distinct.

### Smoke-test invocations

```bash
# List accounts (live roster)
python3 scripts/stationavailability_peraccount.py --list-accounts

# Single account (dry-run by default)
python3 scripts/stationavailability_peraccount.py --account DFMG --mode daily --date 2026-06-24

# Single account, live publish
python3 scripts/stationavailability_peraccount.py --account DFMG --mode monthly --date 2026-05 --publish --post-slack

# All accounts (dry-run by default)
python3 scripts/stationavailability_allaccount.py --mode monthly --date 2026-05
```

---

## GCIM Ticket History integration (V3.4, shipped 2026-06-27)

Triggered by DFMG asking to see their Jira incident timeline inline with the
station-availability report. Now lives in both `stationavailability_peraccount.py`
and `stationavailability_allaccount.py`, plus the BETA twin for experimentation.

### Why this exists

DFMG saw the V3.3 report and asked: *"Could be nice to have the GCIM tickets in
the same report. Incident detected, issue, ticket escalated, ticket solved."* The
GCIM Jira project is the source of truth for customer-facing incidents — pulling
it inline removes the click-out to Jira and gives partners a per-station
timeline alongside the availability stats.

> **Critical detail:** SiteTracker's `/v2/tickets` endpoint also carries a
> `Ticket_Number__c` field that *should* mirror the Jira key — but only ~5% of
> SiteTracker tickets had it populated when we audited (range frozen at
> GCIM-50→300, nothing more recent). So we drive the integration FROM Jira, not
> from SiteTracker. Inverse direction works reliably: every GCIM ticket has a
> `Swift Station ID` custom field that joins to the 4-char station code.

### What gets rendered

**Inline on every "Uptime and downtime" row:**
clickable `GCIM-####` link + status pill (OPEN / CLOSED / ESCAL / CANCEL / …).
Stations with no GCIM tickets show "no incidents" — single source of truth
across both columns of the stations grid. Bars are uniformly aligned because
the inline cells use fixed widths (`90px 1fr 70px 200px 90px 100px`).

**Bottom of report (collapsed `<details>` by default):**
"GCIM Ticket History — `<account>`" summary header with chevron + counts.
Click to expand. Inside: per-station blocks (one heading + table each), full
10-column timeline:

```
Ticket │ Status │ Description │ ST # │ SKU │ Start │ Detect │ Escalated │ End │ Closed
```

- **Description** wraps in `<details>/<summary>` when over 110 chars (stripped).
  Closed by default — partner clicks to open the ones they want to read.
  Short descriptions render inline without a toggle.
- Each `<tr>` has `id="gcim-row-GCIM-1097"` so the inline GCIM-# link in the
  Uptime/Downtime section can scroll-to-it AND auto-open the parent `<details>`
  via a tiny inline JS snippet (cross-browser; Chrome handles this natively
  for `<details>` containing the target, Safari needs the polyfill).
- Newest first per station (JQL `ORDER BY created DESC`, preserved by the
  per-station grouping in `fetch_gcim_for_account`).
- Stations with zero tickets are still listed for completeness as
  `— no incidents —`.

### Two-PNG output (peraccount only)

The main PNG renders with the `<details>` section closed (height-capped at
4000 px so the at-a-glance overview is preserved). A second
`*_gcim.png` re-renders the same HTML with `<details>` forced open and
`height=30000 px` so the full timeline is captured as a single image for
partners who want it. Both are pushed to GitHub Pages by the `--publish` flow.

The all-account variant generates only a single PNG (one tab per account =
forcing every section open at once would produce an unmanageable height).

### Auth — Atlassian Personal API Token

The integration uses Basic auth with a classic Personal API Token:

```bash
export JIRA_USER_EMAIL=your.name@swift-nav.com
export JIRA_API_TOKEN=<token>          # ATATT3xFfGF0…
```

**Get a token** at <https://id.atlassian.com/manage-profile/security/api-tokens>
→ "Create API token" → label it (e.g. `stationavailability-jiraticket`) → copy
the single string. No admin approval needed for self-issued personal tokens.

We tested OAuth 2.0 Client Credentials grant against `auth.atlassian.com/oauth/token`
on 2026-06-27 — it returned `invalid_client: failed to retrieve client`, which
is expected: those credentials are for Atlassian Connect/JWT apps, not REST
API token use. The classic Personal API Token is the supported path.

### Custom-field discovery

`BETA_gcim_helper.py` does NOT hard-code the `customfield_NNNNN` IDs — it
discovers them on first call via `GET /rest/api/3/field`, caches the
name→id mapping for the process lifetime, and resolves each of the 10
expected labels:

```
Swift Station ID, Station Owner, Sitetracker Ticket Number,
Sitetracker Ticket Link, Station Model Number / SKU, Incident Start Time,
Incident Detect Time, Escalated to Partner, Incident End Time, Incident Closed
```

If any of them are renamed in Jira, the affected fields just render as `—` —
the rest of the table stays intact.

### Pagination — new search endpoint

Atlassian deprecated `GET /rest/api/3/search` in 2025 (it now returns HTTP 410
Gone for Cloud customers). We use the replacement
`POST /rest/api/3/search/jql` with token-based pagination (`nextPageToken`),
100 issues per page. DFMG's 519 tickets fetch in 6 paginated calls (~6 s).

### Inputs the report exposes

| ctx key | Source | Where it renders |
|---|---|---|
| `gcim_section_html` | `render_gcim_section_html(...)` | Body slot after `path_connection_html` |
| `gcim_summary` (passed to `render_sar_block`) | `build_gcim_summary(...)` | Inline 5th + 6th cells in every station row |

`render_station_row` (in `dfmg_availability.py`) and `render_sar_block` (in
`dfmg_sar_availability.py`) each gained one optional kwarg with `None` default.
When `None`, behaviour is identical to pre-V3.4 — so any code path that
doesn't pass them stays exactly as it was.

### Failure mode

Jira fetch wrapped in `try/except`: missing env vars, expired token, 410/401
from API — operators see a `[GCIM]` log line, partners see the report
WITHOUT the new section. Nothing else breaks.

### Outstanding follow-up

- ~16 of DFMG's 519 GCIMs have NO `Swift Station ID` set — invisible to this
  integration. Worth raising with whoever triages GCIM so the field gets
  populated going forward.
- A service-account API token (currently using Gian's personal token; valid
  until 2026-07-27) is the proper long-term path for scheduled launchd runs.
- The `--publish` loop currently pushes only `{base}.{html,png,csv}`; the
  `*_gcim.{html,png}` second-PNG variant is rendered locally but not pushed
  to Pages — partners viewing the Slack post must click the HTML link to
  expand the GCIM section. Easy follow-up.

---

## Light / dark theme + Swift Navigation branding (2026-06-21)

Both `dfmg_sar_availability.py` and `regions_stationavailability.py` now ship
with a Swift-branded report:

| Element | Detail |
|---|---|
| **Dark navy header band** | gradient `#0f1a2a → #1a2332 → #232f44` with 4px Swift-orange bottom border |
| **Swift Navigation logo** | embedded as base64 PNG from `assets/swift_nav_logo.png` (58 px tall) |
| **Report title** | "DFMG Station Availability" / "Regional Station Availability" · 24 px / weight 800 / white |
| **Dark navy footer band** | mirrors header, includes generation timestamp |
| **Theme toggle** | sun/moon button in top-right of brand bar — switches dark ↔ light for the current session only |
| **Hard-defaulted to dark** | every page load starts in dark mode. `localStorage` intentionally NOT used so partners see consistent default regardless of prior viewer's toggle. |
| **Sentence-case headers** | JS `HEADER_REWRITES` table rewrites production's ALL-CAPS strings on `DOMContentLoaded` (e.g. "DURATION OF EMERGENCY POWER · per station · ..." → "Emergency power — time on backup battery, per station") |
| **Partner-friendly metric names** | "SAR Station Health" → "Station Health". "SAR Connection Online" → "Connectivity Status". Technical metric-name chips hidden via CSS. |
| **Methodology banner** | dropped from display. Technical jargon belongs in this README, not the partner-facing summary. |

### Light-mode contrast hardening

In light mode, ALL text is forced to **pure `#000000`** with extra-heavy
font-weights:
- `h1` / `h2`: weight 900
- `h3` / section titles / KPI labels: weight 800
- Body text / KPI subtitles / table cells: weight 700
- Station name bars: weight 800
- KPI big numbers: weight 900 (semantic accent color preserved)

Three layers of CSS specificity ensure these rules always win:
1. `body.light-mode .X` (when JS has applied the class)
2. `body:not(.dark-mode) .X` (default state before JS runs)
3. Direct selector fallbacks

WCAG verified: `#000000` on white ≈ 21:1 contrast (AAA-grade).

### Dark-mode palette

Background `#0a0e15` · card `#14202e` · text `#f0f4f8`. Status colors
brightened for proper contrast on dark: green `#4ade80`, red `#f87171`,
blue `#60a5fa`, yellow `#fcd34d`. Trend arrows, badges, code spans all
re-themed.

---

## Known limitations and open issues

### ✅ Resolved 2026-06-13 (schema v2 cycle)

1. **Cross-midnight outage boundary handling** — fixed. v2 cache stores raw event timestamps; merger re-pairs backup→grid events across the full multi-day window. Adel outage_windows separated by less than 120 s are merged automatically. A 23:55→02:30 outage now reports as one continuous ~2 h 35 min event.

2. **`recent_silent` per-day boundary** — fixed. Spurious correction + station-dark classifier are applied at merge/render time using **window-end** `recent_silent`, not baked in per-day.

3. **No cache invalidation policy** — partially addressed via T-1 auto-regen (catches yesterday's bad caches automatically) + `--rebuild-cache` manual flag (covers any range). For deeper-history corruption beyond Prometheus retention, no recovery is possible regardless of script behaviour.

### Still open

4. **No continuous APC presence/health metric.** Adel sites are observable continuously; APC sites are only observable on transitions. Closing this gap requires a PR to `swift-nav/sar` to scrape GPIO 4 ("APC UPS is Connected") and GPIO 3 ("Battery State is Normal/Charging") from the IBR600C router.

5. **NetCloud delivery latency floor.** Real timing accuracy is bounded by NetCloud's own alert delivery (~30–60 s), so even 1-min sampling can't claim sub-30 s precision. The 60s step approaches this floor; finer steps would not help.

6. **`stations_exporter_endpoint_alive` is a dead metric.** Registered in the Prometheus catalog but has produced no samples in 90+ days; the exporter's endpoint-probing subsystem appears disabled. Not used by the script.

7. **Active Thanos cluster topology is hard-coded.** `PASSIVE_THANOS_CLUSTERS = {"eu2"}` at the top of the cache section. If the active/passive flips again (or the topology changes to include more clusters), update this set so `[PROM-PASSIVE]` warnings are correctly tagged.

8. **Cross-midnight Adel outage merge threshold is fixed at 120 s.** Two distinct Adel outages that happen to start within 2 minutes of each other across a midnight boundary would be incorrectly fused. Extremely rare in practice; acceptable tradeoff vs. complexity of finer heuristics.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Slack post failed: 400 invalid_blocks` | Pages CDN didn't propagate before Slack tried to embed | Script auto-waits via `wait_for_url` up to 2h; otherwise text-only post |
| `Full report` link in Slack shows old content | Browser/proxy/CDN cache | Should not happen now — snapshot URLs are unique per publish |
| Headline `Availability 0.0000% · 0/0 affected` | Grafana historical query returning empty (Thanos hiccup) | Wait + retry; script doesn't crash |
| Recent stations missing from report | Filter mismatch | Authoritative roster fetched at run time — should always be current |
| Panel 3 schedule-freshness ⚠ | Mac was off during fire time | `launchctl start <job>` manually |
| Weekly/monthly emergency-power total looks suspiciously high | Pre-hybrid run (single 30-day query) inflated by phantom unpaired events | Re-run with hybrid; cross-check FRMG, FRBE |
| `[CACHE] N day(s) cached · M day(s) need gap-fill` | Some daily caches missing | Normal during backfill; gap-fill writes them |

---

## Setup (one-time)

### Prerequisites

```bash
pip3 install requests
# Headless Chrome (already present if you have Google Chrome.app)
```

### Tokens (already embedded as defaults; override via env)

| Variable | Purpose |
|---|---|
| `GRAFANA_API_KEY` | Thanos read access via Grafana proxy |
| `GITHUB_TOKEN` | Push commits to `GianDuhina/station-availability-reports` |
| `SLACK_WEBHOOK` | `#testing-scripts` incoming webhook |

### Loading the launchd schedules

```bash
for p in ~/Library/LaunchAgents/com.station-availability.*.plist; do
  launchctl unload "$p" 2>/dev/null
  launchctl load "$p"
done
launchctl list | grep station-availability    # verify
```

### Starting the monitor

```bash
bash monitor.sh
# Detach with Ctrl+B D — session stays alive in the background.
# Re-attach later: tmux attach -t station-availability
```

---

## Architecture (file layout, post 2026-06-19 reorg)

```
station-availability/
├── README.md                                ← this file
├── monitor.sh                               ← tmux session bootstrap
├── _panel_realtime.sh                       ← panel 2 — live log tail w/ colour
├── _panel_failures.sh                       ← panel 3 — failures + stale-content checks
├── station_availability.log                 ← structured UTC + [TAG] log
├── assets/
│   └── swift_nav_logo.png                   ← Swift Navigation brand logo (base64-embedded in reports)
├── cache/                                   ← hybrid cache layer
│   ├── sar_daily_dfmg_<YYYY-MM-DD>.json     ← DFMG (production)
│   ├── sar_daily_region_NA_<YYYY-MM-DD>.json
│   ├── sar_daily_region_EU_<YYYY-MM-DD>.json
│   ├── sar_daily_region_AP_<YYYY-MM-DD>.json
│   ├── sar_daily_region_ALL_<YYYY-MM-DD>.json
│   └── publish_queue.json                   ← failed-push retry queue
├── scripts/                                 ← ALL Python scripts (since 2026-06-19)
│   ├── dfmg_sar_availability.py             ← DFMG production
│   ├── regions_stationavailability.py       ← Regional production
│   ├── station_availability.py              ← shared plumbing module
│   ├── BETA_TESTING.py                      ← BETA twin for DFMG experiments
│   ├── BETA_TESTING_regions.py              ← BETA twin for regional experiments
│   ├── dfmg_availability.py                 ← deprecated 3-metric variant
│   └── station_check.py                     ← ad-hoc CLI diagnostic
├── dfmg_sar_<mode>_<stamp>.{html,png,csv}   ← DFMG generated reports
└── regions_stationavailability_<mode>_<stamp>.{html,png,csv}
```

The `BASE_DIR` walk-up in `scripts/station_availability.py` ensures generated
files, caches, and logs land at the project root regardless of where the
script is run from.

---

## Reference: swift-nav/sar source

`https://github.com/swift-nav/sar` (private) is the actual SAR exporter codebase, renamed
internally to **Station Monitoring Service**. Key files for our purposes:

- `station_monitoring/lib/metrics.py` — `class Status: TRUE=1, FALSE=0, UNKNOWN=-1`
- `station_monitoring/netcloud/alert.py` — NetCloud alert message string constants
- `helm/station_monitoring/dashboards/common/*.json` — canonical Grafana dashboards
  (router-status, single-router-overview, etc.)

The recording rules used by the canonical `router-connectivity-30-day-report.json`
(`name:sar_status_connection_online:avg_over_30d_by_1m_max` etc.) live only in the
dedicated SAR Grafana instance — not in our edge Thanos — so we can't piggyback on them.

---

## Project journey — what we've tackled

| Phase | Outcome |
|---|---|
| Initial build | Multi-partner script (`station_availability.py`), then DFMG-specific 3-metric (`dfmg_availability.py`), then SAR-only production script (`dfmg_sar_availability.py`) |
| Authoritative roster | Replaced country-suffix heuristic (`DFMG-EU`, 92% accurate) with `edge_sitetracker_site_status{customer="DFMG"}` (262 sites, authoritative) |
| Silent-station diagnostics | NetCloud alert event counters mapped to human-readable headlines |
| Production vs non-prod split | `edge_sitetracker_noc_status` drives header/silenced/non-prod sectioning |
| Hardware-tier awareness | `power_management_dcups` separates Adel (174) from APC INFRA LITE (88); each tier measured via its own data source |
| Precision investigation | Identified that APC outage durations were stuck at 5-min precision on monthly due to Prometheus' 11k-point cap |
| **Autotune step fix** | Replaced hardcoded `step_s=300` with `_autotune_step()` — daily/weekly now 60s, monthly stays safe |
| **Hybrid cache + live** | Daily runs write per-day JSON cache; weekly/monthly read cache + gap-fill from Prometheus → 1-min precision at all window sizes |
| Phantom outage discovery | Pre-hybrid monthly inflated unpaired events into multi-day phantoms (FRMG 5d 9h, FRBE 1d 18h). Hybrid eliminates this class of bug. |
| Validated end-to-end | May 2026 backfill: 31 dailies cached (818 KB total), 5 weeklies (W18 used 4-day gap-fill), monthly stitched from 31 cached days in 70 s |
| **Schema v2** (2026-06-13) | Raw event timestamps stored per-day so weekly/monthly merger re-pairs backup→grid events **across midnight boundaries**. Spurious correction moved to merge time. Fixes cross-midnight outage undercount. |
| **Battery-exhausted-dark classifier** | New 3rd classification: when `battery_exhausted ≥ 1` AND station is in `recent_silent`, duration is capped at UPS battery life instead of being thrown away as uncertain. Recovers signal we previously lost. |
| **T-1 auto-regen + `--rebuild-cache`** | Daily mode sanity-checks yesterday's cache; missing or undersized cache auto-spawns a regen subprocess. Manual `--rebuild-cache` flag covers any range (YYYY-MM-DD / YYYY-MM / Nd). |
| **Cluster-failover resilience** | After 2026-06-10 eu-prod-1↔eu-prod-2 active/passive flip exposed instant-query fragility, `fetch_adel_equipped_sites` now uses `last_over_time[7d]` and `fetch_global_nonprod_sites` uses `[24h]`. Prometheus warnings surface as `[PROM-WARN]` / `[PROM-PASSIVE]` in logs. |
| **Methodology banner** | Beta-marked banner now rendered at the top of every report explaining the precision improvements + reclassification logic, until stakeholders are aligned and the banner is opted-out via `DFMG_HIDE_METHODOLOGY=1`. |
| **Monitor panel 3 cache section** | `_panel_failures.sh` extended with yesterday's cache presence, 7-day coverage, schema-drift detection, and `[PROM-PASSIVE]`/`[PROM-WARN]` tail. |
| Validated end-to-end (v2) | May 2026 re-rebuilt to v2 (31 dailies, 828 KB). Launchd-fired daily for 2026-06-11: T-1 sanity check (healthy → skipped), main daily completed in ~3 min including Pages publish + Slack post, methodology banner visible on Pages. |
| **Path Connections section** (2026-06-18) | Per-station wired↔cellular failover surfacing. Status classifier (7 categories), trend arrows vs T-7 baseline, primary/secondary duration columns, CURRENT (WAN/Cellular/Offline) column. Cellular durations estimated via per-mdm-event session timing capped at 4h. |
| **Publish queue + morning drain** | GitHub API timeouts at the 23:00 PHT slot (VPN session expiry) caused 6 stuck reports between 2026-06-15 and 2026-06-17. Added a `cache/publish_queue.json` write-when-fail mechanism + `--drain-publish-queue` flag + 09:00 PHT launchd job that retries pending pushes when the VPN is fresh. Recovered all 6 backlogged reports automatically. |
| **`offline-entire-window` classifier fix** (2026-06-19) | Stations with <60s of uptime in the window were falsely labelled "stable" because their counters looked quiet (router never reachable to fire events). Added an explicit pre-check on `up_s` so silent stations correctly surface as warning-color OFFLINE rows. Caught 4 stations in 2026-06-18 daily (FIJM0xFIN, FRMG0xFRA, HUVE0xHUN, RONA0xROU). |
| **UX refinements** (2026-06-19) | STATUS GUIDE moved to bottom of Path Connections (reference position vs upfront). Removed redundant exec summary banner (info already in KPI tiles). Added trend-arrow legend above per-station table. Zero-uptime cells show `0s` instead of `—` (disambiguates "no data" vs "nothing was up"). DATA ACCURACY panel added — explicit confidence label (direct / inferred / estimated / unavailable) per visible column. |
| **SAR vs Orion-bytes mismatch surfaced** (2026-06-19) | Cross-checking IRCO/HRPL May 2026 against Grafana's `orion_ntrip_client_bytes_sum` revealed the two metrics measure different observation points: SAR sees the NetCloud control channel, Orion sees actual NTRIP byte flow. A station can be "down" per SAR while still serving NTRIP via an alternate network path. Documented in the Path Connections caveats. |
| Validated end-to-end (Phase 3) | May 2026 monthly republished 2026-06-19 with new layout. 250 prod stations, 97.6572% availability, current-connection distribution (244 WAN / 5 Offline / 1 Cellular = 250) matches production count exactly. STATUS GUIDE counts sum correctly after subtracting the legend rows. |
| **Scripts reorg** (2026-06-19) | All 5 `.py` files moved into `scripts/` subfolder for clean repo layout. `BASE_DIR` walk-up added to `station_availability.py` so reports / cache / log still land at project root. 4 launchd plists updated to point at `scripts/dfmg_sar_availability.py`. Backups saved as `*.bak.20260619`. |
| **Regional Station Availability** (2026-06-20) | New `regions_stationavailability.py` produces NA / EU / AP / All-Regions rollup with full feature parity to DFMG production. Country → region mapping with overrides (ABCG → Telus, ATGJ → IND). SAR/orion + `noc_status==1` filters applied. Per-region hybrid cache (`sar_daily_region_<ALL\|NA\|EU\|AP>_<date>.json`). May 2026 rebuild took 1h 18m for 93 region-day caches (~3.5× DFMG's 22m). May headline: NA 330 / EU 251 / AP 116 production stations. EU matches DFMG exactly (84 affected, 259879 min down). |
| **BETA sandbox split** (2026-06-19/20) | `BETA_TESTING.py` (DFMG) + `BETA_TESTING_regions.py` (regional). Outputs use `BETA_` filename prefix so production reports are never overwritten. Title badge shows "· BETA"; logger names distinct. Pattern: experiment in BETA → smoke-test → merge to production once proven. |
| **Multi-disciplinary lens commitment** (2026-06-20) | Foundational instruction: every session and every process is approached through ALL professional lenses — web dev, software dev, graphic designer, graphic artist, UX, accessibility, technical writer, brand — not just code-correctness. Visual + brand + UX checks before shipping any change. Saved as `feedback_multi_disciplinary_lens.md` in auto-memory. |
| **All-Regions overview view** (2026-06-20) | Default dropdown option in `regions_stationavailability.py`. Aggregates NA + EU + AP into one rollup (~697 production stations) and STRIPS per-station detail (uptime/downtime bars, silent table, emergency per-site table, ADVISORY bar, trend legend, Path Connections per-station table, DATA ACCURACY, STATUS GUIDE) — keeps only the high-level KPI tiles + Power Infrastructure Health for executive scan. Per-region tabs (NA / EU / AP) retain full detail. |
| **Swift Navigation light theme** (2026-06-21) | Light theme with Swift brand colors (orange `#f26522`, red `#e63027`, navy `#1a2332`). Dark navy header band with embedded real PNG logo (58 px) + report title (24 px / weight 800). Sentence-case headers via JS rewrite. Partner-friendly metric renames (Station Health, Connectivity Status). Methodology banner dropped from display. |
| **Dark mode + toggle** (2026-06-21) | Sun/moon toggle button in brand bar. Dark default = page bg `#0a0e15`, cards `#14202e`, text `#f0f4f8`. Brightened status colors for dark-bg contrast (green `#4ade80`, red `#f87171`, blue `#60a5fa`, yellow `#fcd34d`). Pre-`<body>` inline script prevents flash-of-wrong-theme. Hard-defaulted to dark on every page load — `localStorage` not used, so partners see consistent default. Toggle is session-only. Applied to both DFMG and regional scripts. |
| **Pure-black + extra-bold light mode** (2026-06-21) | After multiple iterations of "still too light" feedback: ALL light-mode text forced to `#000000` with weights 700-900 (was `#0a0e15` at 500-700). Three-layer CSS specificity insurance (`body.light-mode`, `body:not(.dark-mode)`, direct selectors) guarantees rules win cascade. WCAG AAA verified (21:1 contrast on white). |
| **V3.0 — Account-dimension BETAs** (2026-06-25) | `BETA_TESTING_per_account.py` + `BETA_TESTING_all_accounts.py` shipped. Customer-pivoted reports (10 customers / 949 sites) with Overview rollup + native `<select>` dropdown. Caches shared with later V3.1 production. Decisions locked: simple native dropdown, fleet-size-desc order, customer name only. |
| **V3.1 — Account-dimension production** (2026-06-25) | Promoted to `stationavailability_peraccount.py` (706 lines) + `stationavailability_allaccount.py` (614 lines). Default = dry-run; opt-in to publish via `--publish` + `--post-slack`. Per-invocation log file alongside HTML/PNG/CSV. File naming `stationavailability_<ACCOUNT>_<period>.{html,png,csv,log}`. |
| **V3.2 — First live monthly publish** (2026-06-25) | 8 May 2026 monthly reports posted to `#testing-scripts`. Two surprises captured: (1) Singtel monthly aborts because `noc_status` was non-prod in May but prod in June (sitetracker is time-varying), (2) GitHub Pages-API outage 15:40–17:30 UTC wedged retries for ~30 min/account — recovery: kill + verify + retry from warm cache (~5 min/account). |
| **V3.3 — DFMG partner feedback features** (2026-06-26) | Driven by Thierry's review of the May 2026 report: (a) decommissioned/non-prod sites removed from rendered sections, full list in `_excluded.csv` + Slack `:wastebasket:` line + footer note; (b) path-connection table stratified — "0s and 0s" rows hidden behind a green ✓ NO IMPACT card with collapsible "Show all" reveal; (c) sign-aware text restate — `0s` becomes "no downtime" (green), "offline entire window" (red), or "primary only" (muted green) with edge-case conditional for offline-entire-window stations. |
| **`_autotune_step` lesson** (2026-06-26) | Yesterday's `400 Bad Request` on `sar_alert_connection_state_*` queries during the May 2026 allaccount publish was misdiagnosed as URL-length overflow. Real cause: Thanos rejects range queries above 11 000 points/series. Monthly window × 60 s step = 44 640 points → fails. Fix: every range-query fetcher (`fetch_path_raw_events`, `fetch_raw_alert_events`, `_estimate_backup_pro_durations`) now uses `_autotune_step(start, end)`. Same fetchers also switched GET→POST as defensive prophylaxis against future URL bloat. |
| **DFMG May 2026 republished with all V3.3 changes** (2026-06-27) | Full 8-account + allaccount publish. Restate counts (per DFMG report): 167 "no downtime" cells, 107 "primary only", 21 "offline entire window". All-accounts overview shows 697 stable / 782 production = 89% stratified by the green NO IMPACT card — fleet-wide stability headline lands first, the 85 actionable stations follow. |

---

_Last reviewed: 2026-06-27 · V3.3 partner-feedback shipped (non-prod removal · stratified path table · sign-aware restate · autotune_step fix) · V3.1 account-dimension production (`stationavailability_peraccount.py` + `stationavailability_allaccount.py`) live with dry-run default · 10 customers / 949 sites authoritative roster · Regional + DFMG legacy production still active · Light/dark themes with Swift branding · Sentence-case headers · Hard-dark default · Pure-black light mode · Scripts in `scripts/` subfolder · schema v2 · cluster-failover-resilient · T-1 auto-regen on_
