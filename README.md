# DFMG Station Availability

Automated daily / weekly / monthly reporting on DFMG partner station availability —
pulled from Grafana/Thanos, rendered to HTML + PNG + CSV, published to GitHub Pages,
posted to Slack `#testing-scripts`.

This README captures everything we know about the system as of today: scripts, data
sources, hardware tiers, the precision architecture, the cache+live hybrid layer,
and the data-quality findings we've surfaced.

---

## Scripts

| Script | Scope | What it covers |
|---|---|---|
| **`dfmg_sar_availability.py`** | DFMG only · SAR metric only | The production script. Scheduled. Now hybrid cache+live. |
| `dfmg_availability.py` | DFMG only · SAR + Adel UPS + NTRIP throughput | 3-metric variant. Heavier; not scheduled. |
| `station_availability.py` | All partners · multi-region | General script + shared plumbing module. Not scheduled. |

> Every change to "the report" defaults to `dfmg_sar_availability.py`.

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

## Architecture (file layout)

```
station-availability/
├── README.md                              ← this file
├── dfmg_sar_availability.py               ← production script (DFMG · SAR-only · hybrid)
├── dfmg_availability.py                   ← DFMG · 3-metric variant
├── station_availability.py                ← shared plumbing (fetch, push, slack)
├── monitor.sh                             ← tmux session bootstrap
├── _panel_realtime.sh                     ← panel 2 — live log tail w/ colour
├── _panel_failures.sh                     ← panel 3 — failures + stale-content checks
├── station_availability.log               ← structured UTC + [TAG] log
├── cache/                                 ← daily JSON event log (hybrid layer)
│   └── sar_daily_dfmg_<YYYY-MM-DD>.json
└── dfmg_sar_<mode>_<stamp>.{html,png,csv} ← generated reports
```

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

---

_Last reviewed: 2026-06-19 · schema v2 active · Path Connections live · cluster-failover-resilient · T-1 auto-regen on_
