---
name: mdu-high-density-analysis
description: "Analyze the eero High Density MDU wifi feature (High Density ACS + Reduced 5GHz Bandwidth) rollout on any set of MDU properties using Looker dashboard 3021. Given a list of (subset_id, property name) and an enablement date, produces per-MDU before/after notes and flags only anomalies that survive rigorous daily-time-series methodology, cross-checked against the WAN-speed grid and the 'MDU ACS - Partner Testing Tracker' Google Sheet. Use whenever asked to analyze, monitor, or report on MDU high-density feature-flag deployments — channel distribution/concentration, 80MHz bandwidth reduction, WAN speed distribution, radar strikes, ACS ringing/churn, or updating the MDU tracking sheet."
---

# MDU High Density Feature Analysis

Repeatable procedure for analyzing rollouts of the **High Density MDU** wifi feature across MDU (Multi-Dwelling Unit) properties, using eero's MDU Looker dashboard **3021**. Built from the feature PRR, the data-analysis methodology, and the corrected time-series method that avoids the common false-positive traps.

Use this when someone gives you a batch of MDU communities (subset IDs) that got the feature flag enabled and wants per-property before/after notes plus honest anomaly flags.

---

## 0. How to invoke this skill (feeding it MDU IDs)

Give me two things and I'll run the whole analysis:
1. **The MDU list** — as `(subset_id, property_name)` pairs. A subset_id is the UUID that goes in the dashboard's `Organization Subset ID` filter (e.g. `34d45828-b476-4cc1-9471-36285c9eae2f`). Paste them however you like — a Python-style list of tuples, a CSV, or "here are the 20 IDs …". Property names are optional but make the output readable.
2. **The enablement date** — when the FF was turned on for this batch (e.g. "Jun 22"). If you don't give one, I'll ask, or pull it from the tracking sheet (§8).

Optional: environment (defaults to `prod`), and whether you want the WAN grid pulled (default yes — it's needed to interpret bandwidth activation).

Then I fan out looker-api agents (~5 MDUs each, in parallel), one pass for the core metrics (§4 items 1–5) and one for the WAN grid (§3), apply the §5 methodology, and return per-MDU one-liners (§7). Where do the subset IDs come from? Usually the enablement notebook/ticket or the **tracking sheet (§8)** — its `MDU ID` column plus the EC/Insight community link give you the subset. If you only have property names, I can look them up from the sheet.

---

## 1. Feature background (what you are analyzing)

**"High Density MDU"** = two complementary eeroOS wifi features, shipped in v7.12.0, **off by default**, feature-flag gated, targeting dense apartment buildings where many collocated eero networks interfere on 5GHz. Owner: Connectivity WiFi ACS team (`@wifi-acs`, Jira component `wifi-acs`).

**The two features (enabled together via the `enable_high_density_optimizations` umbrella FF):**

1. **High Density ACS** (`enable_high_density_selection`) — When a network detects neighboring eero networks on its current channel, it picks a channel at *random* from the top score-improving set instead of the single best channel. Over a few nightly ACS cycles, networks self-spread across channels with no central coordination. Goal: **flatten the channel distribution** (lower peak occupancy, more distinct channels in use).

2. **MDU Reduced 5GHz Bandwidth** (`reduced_mdu_bandwidth`) — Reduces 5GHz width from 320/160 MHz down to 80/40 MHz, but **only** in eero-deploy MDUs **and** when WAN speed is below a threshold, so throughput isn't harmed. Halving width doubles the non-overlapping channel pool and narrows the DFS radar detection window (a strike hits only one 80 MHz half, so fewer networks are displaced to the fallback channel).

### How the combined FF actually works (code-level, nodelib @ Desktop/acs)

The MDUs are NOT given separate per-feature configs — they get the **umbrella `enable_high_density_optimizations` FF**, which fans out to *defaults* for three sub-features. Read once in `get_high_density_params()` (`acs/utils.py:2503`); when True it sets: `reduced_mdu_bandwidth = DEFAULT_REDUCED_MDU_BANDWIDTH`, `enable_high_density_selection = True`, and `acs_score = {"type":"util"}` (util-based scoring). Each sub-flag can still be **individually overridden** if explicitly set — the umbrella only supplies the fallback default (`utils.py:2516-2524`). Consumers fan out from this one function: `radio.py:1229` (bandwidth), `chan_selector.py:229,428` (selection/scoring). *(So `acs_score=util` is a third thing the combined flag turns on — worth noting the combined FF is not just "the two features.")*

**Bandwidth default & gate** — `DEFAULT_REDUCED_MDU_BANDWIDTH = {"band_lo5": {"600": 80}, "band_hi5": {"600": 80}}` (`constants.py:2847`). `get_reduced_mdu_bandwidth()` (`radio.py:1220`) decodes it: keys = WAN **ingress** thresholds in Mbps, values = MHz cap. The loop caps only when measured speed is *below* the key (`radio.py:1296`): `if int(speed_limit) > (max_ingress_mbps.value / 1000): min_bw = ...` — note `max_ingress_mbps` is kbps, hence `/1000` → Mbps. So `{"600":80}` = "if measured WAN ingress < 600 Mbps → cap 5GHz to 80 MHz; at ≥600 no key matches → no cap." Assume the `{"600":80}` default unless you verify a per-property override.

> **CRITICAL — 80 MHz has TWO independent sources; do NOT read 80 MHz share as "low-WAN readout" universally.** The WAN-gated cap above is only ONE way a node lands on 80 MHz. The umbrella FF *also* turns on `acs_score = util` (util-based scoring, see below), and util scoring picks whichever `(freq, width)` config scores best in the current RF environment. In a congested band a clean 80 MHz half legitimately out-scores a busy 160 MHz config, so **the selection path produces 80 MHz with no WAN check at all** (`chan_selector.py` scores candidate configs including width; `data.py:778` stores scores per freq+width). Consequence:
> - On **Trieste + low-WAN** properties, 80 MHz tracks the <600 Mbps WAN share cleanly — the cap path dominates.
> - On **fast-WAN / non-Trieste (hornbill, novo)** properties, you will see substantial *and climbing* 80 MHz (30–70%) despite 95–99% of networks being ≥600 Mbps. This is util-scoring, working as designed — **NOT** a gate violation or a config override. In this session Skyline (36% @ 0.2% <600), Country Club (61–70% @ 10% <600), Regency/Islander/Venture/Kampus/Southlake all showed this.
> - So "80 MHz appeared" ≠ "the WAN cap fired." Only equate the two on Trieste-cap-path properties. On everything else, 80 MHz is a congestion/util signal, not a WAN readout.

**No network-type (`is_mdu`) gate in the high-density code.** `is_mdu` exists (`radio.py:1386,2763`, = `cloud_config[network_customer_type] == "MDU"`) but it is consumed ONLY by the separate **tx-power reduction** path (`is_reduced_power = is_mdu and low_tx_power`, `radio.py:2778`). `get_high_density_params()` and `get_reduced_mdu_bandwidth()` contain **zero** references to `is_mdu` / `network_customer_type`. The feature is kept off retail networks purely by *operational* targeting (we only set the FF on MDU subsets), not by any in-code guardrail. So the PRR/§1 phrasing "only in eero-deploy MDUs" describes deployment convention, not an enforced gate — if the flag were set on any network it would engage subject only to the RF/WAN gates below. (A default-on-by-`is_mdu` plan would *introduce* the first real network-type check; it doesn't exist today.)

**All FOUR gates must hold for the 80 MHz cap** (each returns `None` = no cap, in `radio.py:1220-1307`): (a) FF-derived per-band config present; (b) model supports >80 MHz on 5GHz — `platform.has_160mhz()` (`radio.py:1237`, i.e. Trieste/160-320 models; reduction is meaningless on 80-max hw); (c) **dense-neighbor threshold met** (see below); (d) a valid stored speedtest exists (`radio.py:1284`; no speedtest → `None` → stays full-width). Consumed in `_get_max_bw` as "Priority 3" default, behind the `max_5g_bandwidth` FF and cloud `acs_radio_config` (`radio.py:1142`).

**High Density ACS selection** — triggered when `enable_high_density_selection and eero_neighbor_net_count > 0` (`chan_selector.py:430`; threshold is strictly **>0** neighbors on the current channel). `_select_channel_high_density` (`chan_selector.py:72`) builds a pool = current channel (if valid) + top `M` improving channels where `M = min(neighbors_on_current_chan, ceil(sqrt(L)))`, `L` = count of improving non-overlapping configs, then `random.choice(pool)` (`chan_selector.py:138-151`). More neighbors / more improving options → wider random pool.

**Dense-neighbor threshold** = **2** eero networks (default), from `DenseNetworkDetection.DEFAULT_THRESHOLDS` `{"rssi": -75, "count": 2, "ratio": 0.5}` (`constants.py:2834`), read via the `dense_network_detection` flag (`radio.py:1245`). NOTE: this count=2 gate is used for the **bandwidth** path; the **ACS-selection** path uses a separate, simpler `>0` neighbor trigger. They are different mechanisms.

**Neighbors = other eero NETWORKS, not nodes — and NOT your own siblings.** The gate counts distinct neighboring eero *networks* (`eero_neighbor_net_count`, `data.py:1000`), built by two collapses in `_count_eeros` (`data.py:1210`): (1) every BSSID → its `base_mac`, so one physical eero broadcasting main+guest+IoT+backhaul counts once; (2) sibling nodes of the same network merge into one entry via shared SSID or known base_mac set (`data.py:1242`). So a 5-node mesh next door with 20 BSSIDs on air counts as **1** toward your density. Your own network's nodes merge together the same way, so they never inflate your own count. Two corollaries that trip people up:
> - **Your own node count is irrelevant to activation.** A single-node network in a dense stack still trips the gate off the *other* units; "the property only has 1 node per network" is NOT a valid explanation for non-activation. What matters is neighboring *networks*, not siblings.
> - MDU carve-out (`data.py:1252`): separate units that share a common community/IoT SSID are kept as **distinct** networks (they'd otherwise wrongly merge to 1). Non-eero APs are tracked separately and do **not** feed `eero_neighbor_net_count`.

**Fallback / failure behavior** — Neighbor count 0 → standard best-score selection, tagged `"standard"` (`chan_selector.py:435-473`). All-zero/no-better scores (`L==0`) → `M=0`, pool is just the current channel → stays put. No speedtest, or speed above all thresholds → bandwidth returns `None` → node keeps full/standard bandwidth (`radio.py:1153`). So every failure mode degrades gracefully to standard ACS behavior.

*Tuning note (not a bug):* the default is safe/correct (symmetric lo5/hi5, correct MHz units, correct kbps→Mbps conversion), but `{"600":80}` is a single coarse tier vs the doc's graduated example (e.g. `{"100": 80, "50": 40}`). Two judgment calls if asked to tune: (a) an 80 MHz 2×2 link tops out ~500–700 Mbps real-world, so a unit measured right at ~600 can itself be bottlenecked at 80 MHz — a cutoff nearer ~700 removes that gray zone; (b) there's no 40 MHz step for very slow WAN. A graduated ladder needs the actual `max_ingress_mbps` distribution to set well.

**Why it matters:** In dense MDUs, wide bandwidths (160/320 MHz) leave only ~2-3 non-overlapping 5GHz channels, so nearly all networks pile onto the same channels → high channel utilization. DFS radar strikes then dump all of them onto the same fallback channel (usually ch36). The threshold for sensitivity is **~2+ nearby eero networks per unit** (data-analysis finding), which lines up with the code's dense-neighbor gate of `count=2` (see §1 code section). The two features together attack both the channel-diversity problem and the channel-availability problem.

**Deployment context:** Rolled out per-property via coordinated FF enablement on batches of network IDs (mapped to an MDU/subset identifier). Long-term target ~5,000 EC properties / up to 26k networks. Product approved incremental rollout (Jun 2026), conditional on revisiting alerting before mass enablement.

**Known risks / bugs to keep in mind:**
- **UNII-3 / high-power channel bias** (CONN-67667, open): networks cluster on the highest-tx-power channels (e.g. ch149-165 / ch5560 in the freq encoding, ~27 dBm) and coverage-hole logic then prevents them from leaving or selecting wider bandwidths. Watch for concentration pinned to a UNII-3 channel.
- **Ringing / non-convergence:** randomized selection does not *guarantee* a stable distribution. One earlier property (The Confluence) oscillated nightly on 2.4GHz (only 3 non-overlapping channels) and never settled until the FF was tuned/pulled. 2.4GHz is a fundamental constraint; 5GHz is the primary target and generally converges.
- **DFS fallback diversification was cut** (CONN-56262): networks still all fall back to the same default channel after radar.
- **Bandwidth reduction not activating:** if no speed test is stored, or WAN-speed gating blocks it, the feature returns None and stays on the wider bandwidth. This is *sometimes* by-design (high WAN tier) and *sometimes* a real gap — investigate.

**Reference docs:**
- PRR: Google Doc `1aaTJvRnWqLW4O_NCgZOCZk4yaT3w6CDZ2uySmW1tpLY` (read via `gws docs documents get` — see §6)
- Data analysis: Google Doc `1KgpShaVnKpv0MX3LmIapK2OCV2QHCWFc_Qefly3wvlg`
- Feature-flag docs: `eero-inc/documentation` → `cloud/api/node/flags.md` (read via the `github` MCP `get_file_contents`, owner `eero-inc`, repo `documentation`, path `cloud/api/node/flags.md`). The flag definitions (all introduced in v7.12.0):

| Flag | Type | Description (from flags.md) |
|---|---|---|
| `enable_high_density_optimizations` | boolean | **Combined flag** that enables high density ACS and reduced MDU bandwidth. Individual flags (`enable_high_density_selection`, `reduced_mdu_bandwidth`) override this if explicitly set. Default false. |
| `enable_high_density_selection` | boolean | Enable high density WiFi channel selection algorithm. |
| `reduced_mdu_bandwidth` | dictionary | Enables and configures parameters for reduced bandwidth feature. WAN speed threshold + eero neighbor density must be met; only applies to models with 5GHz 160/320MHz. |

`reduced_mdu_bandwidth` config shape — the **doc example** shows a *graduated* ladder (keys = WAN Mbps thresholds, values = MHz cap; set both `band_lo5` and `band_hi5` for full5):
```json
{"band_lo5": {"100": 80, "50": 40}, "band_hi5": {"100": 80, "50": 40}}
```
…but the **code default** the combined FF actually applies is a single coarse tier (`constants.py:2847`):
```json
{"band_lo5": {"600": 80}, "band_hi5": {"600": 80}}
```
(This doc-vs-default gap is exactly the "tuning note" in §1 — the illustrative doc values are graduated and low; the shipped default is one step at 600.)

---

## 2. Inputs required

Before starting, you need:
- **A list of `(subset_id, property_name)`** — the MDU communities to analyze.
- **The enablement date** — the day the feature flag was turned on. Feature effects typically appear with a **~3-day propagation lag** (e.g. flag set Jun 22 → bandwidth mix visibly flips ~Jun 25).
- **Environment** — essentially always `prod`.

Define the comparison windows from the enablement date:
- **Before** = the ~15 days *before* enablement.
- **After** = enablement day through today (allow for the ~3-day propagation lag when reading onset).

---

## 3. The dashboard

Looker dashboard 3021, one property at a time via the `Organization Subset ID` filter:

```
https://looker.eero.amazon.dev/dashboards/3021?Date=30+day&Env=prod&Organization+Subset+ID=<SUBSET_ID>&MDU+Community+ID=
```

Keep `Date=30 day` (gives ~2 weeks each side for a mid-window enablement) and `Env=prod`. Leave `MDU Community ID` blank. Substitute each subset id.

The dashboard is backed by the `phalanx_node_events_custom` Looker model, scoped per MDU via `organization_subsets_units.organization_subset_id`, filtered `environment = "prod"`. Underlying explores/views used:
- `channel_switch` — selected 5GHz frequency + bandwidth (filter to 5GHz, `fallback=No` for steady-state selection). Source for channel distribution, bandwidth mix, and genuine channel changes.
- `conn_radar_detect` — radar strikes.
- `conn_acs_performance` — 5GHz `primary_chan_util` (P50/P90/P99), AND per-node **operating** channel/width (what each node is actually running).
- `hw_emmc_usage` — reliable per-day network count, the **WAN speed grid**, AND per-node **hardware model** (verify Trieste vs hornbill/novo here — see §4).

**Measurement method — use OPERATING-channel data for bandwidth/concentration, not windowed switch counts (this caused big first-pass errors).** A window-summed `channel_switch` distribution over-weights nodes that *flap* (they emit many switch events) and under-weights stable nodes (few/no events), so it badly **under-measures** the steady-state 80 MHz share. In this session it reported ~22% where operating data showed ~85% (Lake Dexter) and ~8% vs ~88% (Paragon). Prefer each node's actual running `(primary_freq, width)` from `conn_acs_performance` as ground truth for "what width/channel is each node on right now." Two more gotchas:
- Measure concentration/bandwidth on **`band_full5`** (the main 5GHz radio). `band_hi5`/`band_lo5` are natively-80 MHz split-band segments that inject a false ~9% "80 MHz" floor even pre-enablement.
- `channel_switch` bandwidth share is event-weighted → double-counts intraday width changes. When you must use it, weight by distinct networks, not events.

**WAN speed grid (essential for interpreting bandwidth activation):** dashboard 3021 has a tile **"WAN Speed (past 10 days)"** (tile id 77996), backed by `hw_emmc_usage` field `speed_test_median_7d.download_speed_tier`, scoped by `organization_subsets_units.organization_subset_id`, bucketed in ~100 Mbps tiers. **Gotcha: this tile is hardcoded to `stage` — you MUST override to `hw_emmc_usage.environment=prod`** (without it, the prod subset IDs return zero rows). The 600 Mbps gate lands exactly on the "500 to 599" / "600 to 699" tier boundary, so the below/above-600 split is exact. Pull, per property: **% of networks below 600 Mbps (eligible) vs ≥600 Mbps (ineligible)**, median tier, and the native buckets.

Use the **looker-api agent** to query these explores directly (it can pull the tiles' underlying data and time-series); don't just eyeball the rendered dashboard. For many properties, fan out across several looker-api agents in parallel (~5 MDUs each).

---

## 4. Metrics to extract per MDU

1. **Network count** (from `hw_emmc_usage`) — is the property big enough to matter? Below ~60 networks the high-density benefit is marginal.
2. **Hardware model** (from `hw_emmc_usage`) — **verify Trieste vs hornbill/novo from the data; do NOT trust the tracker/prior notes.** In this session the sheet labeled several properties "all-Trieste, low-WAN" that were actually gigabit hornbill (Regency, Lantern, Islander). Hardware determines how to read 80 MHz (see item 3) and whether the cap path is even eligible (`has_160mhz`; all of Trieste/hornbill/novo pass, so hornbill IS cap-eligible — its 80 MHz is not "native 80 MHz hardware").
3. **Channel diversity** — distinct 5GHz channels in use per day, AND **peak channel occupancy** = share of networks on the single busiest 5GHz channel. Lower peak occupancy + more distinct channels after enablement = success.
4. **Bandwidth mix** — daily share at 320 / 160 / 80 / 40 MHz (from operating data, §3). The key signal is whether **80 MHz appeared or grew** after enablement. **Read it against hardware + WAN (two-path rule, §1):** on Trieste low-WAN it should track the <600 share; on fast-WAN/hornbill/novo, expect meaningful util-driven 80 MHz that does NOT track WAN — that's healthy, not a violation.
5. **Radar** — as a DAILY time series, plus the count of distinct **radar-days** before vs after.
6. **Stability / ringing** — genuine day-over-day channel changes per network, as a time series, with attention to the last few days.
7. **WAN speed distribution** — % below 600 Mbps vs ≥600 Mbps + median tier (from the WAN grid, §3). Determines how much 80 MHz the *cap path* can activate; always pull it before judging a low/zero 80 MHz activation. (Remember it does NOT bound the util-scoring 80 MHz path — see §1 two-path rule.)

Terminology to use in output (keep these DISTINCT — don't conflate):
- **"peak (5GHz) channel occupancy"** / **"max single-channel concentration"** = share of networks on the single busiest 5GHz channel — a *distribution* measure (who's sitting where). This is what High Density ACS moves.
- **"channel utilization"** / **congestion** = `primary_chan_util`, the *airtime* the channel is busy — a different thing. Concentration can drop while utilization stays flat (spreading networks removes structural risk, but airtime only falls where there was real contention). Don't call a concentration drop a "congestion drop."
- **"channel churn" / stability** = how often networks *change* 5GHz channel day-over-day (movement, not location). For non-jargon audiences, phrase as "networks still changing their 5GHz channel day-to-day / haven't settled into a stable distribution" rather than "churn."

---

## 5. Methodology — CRITICAL (this is the corrected method; do NOT regress)

An earlier pass produced ~4x too many false anomalies by using naive window-sums and a saturated metric. **Always use the time-series methods below.** Each rule includes the trap it avoids.

### Radar — never sum strikes over the window
Radar is **bursty**: a single bad day can dominate a window-sum and fake a "radar doubled" regression. Instead:
- Pull radar as a **daily series** for all 30 days.
- Count **distinct radar-days** before vs after enablement.
- Only call radar a regression if the **number of radar-days OR the sustained daily rate genuinely rose** across *multiple* post-enablement days that exceed the pre-period max — NOT if one post day spiked.
- Many MDUs are chronically radar-saturated (radar every single day, 15/15 both windows); that's a site-environment characteristic, not feature-induced.
- **Check for FLEET-WIDE synchronized radar before blaming the feature.** If a radar spike lands on the *same day(s)* across many unrelated properties at once (different partners, regions), it's an environmental/weather DFS event, not a per-property regression — it self-clears within days. In this session a Jul 11–16 spike hit ~all properties simultaneously; only genuine per-property regressions (rise starting on that property's own timeline, far above its own pre-max) survive. Pull the fleet daily total as a cross-check.
- *Real regression example (survives §5):* one property went from a pre-period max of ~4 strikes/day to sustained 73–990/day starting mid-window as channel-spreading pushed nodes onto DFS — sustained, multi-day, far above its own baseline, and NOT synchronized with the fleet event. That is a true feature-induced regression.
- *Trap example:* Skyline "radar 2889→4081" was ONE day (enablement day = 3172); every other day was single/double digits → not a regression.

### Bandwidth — a from-zero appearance is SUCCESS, not "inert"
- Use **operating-channel data** (§3), not windowed switch counts, or you will under-measure 80 MHz badly. Pull the **daily 80 MHz share** on `band_full5`. If 80 MHz went from ~0% pre to *any* meaningful, sustained share post, the feature is **working** — report it as such.
- Only call bandwidth **inert** if 80 MHz stayed ~0% on essentially **every** post-enablement day.
- Report pre→post 80 MHz share AND the trend/onset day.
- *Trap example:* Skyline "bandwidth inert (3.5% at 80MHz)" was wrong — 80 MHz went 0→present with a clean onset = working.
- **Interpret the 80 MHz share through the TWO-PATH rule (§1), gated on hardware + WAN — do not treat 80 MHz as a pure low-WAN readout:**
  - **Fast-WAN, mostly ≥600 Mbps → cap path SHOULD be near-zero.** Low 80 MHz here is correct-by-design, not a fault. *Example:* Kiahuna 80 MHz ~0% with **96.9%** of networks ≥600 Mbps (mass in the 600–699 tier) = correctly inert.
  - **But fast-WAN properties can STILL show substantial 80 MHz via util-scoring** (the second path, no WAN gate). *Example:* Islander is gigabit **hornbill** (96% ≥600) and runs ~30% 80 MHz; Regency is gigabit hornbill and runs ~26–33%. That is util-scoring, working — not a gate violation. (These two were first-pass-mislabeled as "low-WAN Trieste with 91%/1%"; the corrected read is hornbill, both ~30%. Verify hardware from data, §4.)
  - **The only genuine bandwidth anomaly is the cap-path gap:** Trieste-eligible AND high <600 share AND still ~0% (see §7). Suspect FF-not-live or missing speedtest — NOT node count.

### WAN vs activation reconciliation — measured ingress ≠ plan tier (cap path only)
- This reconciliation applies to the **cap path** (Trieste low-WAN). The endpoints match (very-low-WAN activate strongly; all-high-WAN stay inert on the cap path), but the **middle often shows MORE 80 MHz than the below-600 grid share predicts** (e.g. 7.6% below 600 but 34% at 80 MHz).
- Two compounding reasons, both consistent with the feature working: (1) the code gates on the node's **measured `max_ingress_mbps`**, while the WAN grid shows the **7-day median download tier** (≈ provisioned plan), and measured ingress routinely lands *below* plan (overhead, single sample, off-peak, upstream congestion), so more units clear the 600 gate than the plan tier suggests; (2) util-scoring adds 80 MHz on top with no WAN gate at all. So expect activation ≥ the grid's below-600 share. Only chase it further (read `max_ingress_mbps` on sample nodes) if you need it airtight.

### Channel diversity — read the daily trend
- Distinct channels/day and peak occupancy, before→after, watching the trend through the latest days. Peak occupancy trending down = load spreading. Note if concentration stays pinned to a single UNII-3 channel (high-power bias).

### Ringing / stability — the naive metric is saturated and USELESS
- The raw "% of networks that emitted a channel-switch/decision event per day" sits at **~90-100% every day, before and after** (ACS re-evaluates constantly). This produced the bogus "ringing 8→18%" style flags. **Do not use it.**
- Instead compute **genuine channel changes**: `previous_freq != attempted_freq`, both 5GHz, successful, non-fallback — as **switch-events-per-network-per-day**, normalized by networks present.
- **Distinguish two cases explicitly:**
  - *Genuine oscillation (ringing):* the per-network shift stays **HIGH at the tail** (last 3-4 days) AND movers bounce among the **same few channels** (check: of the heavy movers, how many confined moves to ≤3 channels?).
  - *Convergence-in-progress (healthy):* shift spikes right after enablement then **trends down and settles** to ~1 change/net/day by the latest days. This is expected and is NOT an anomaly.
- Watch for **fleet-wide radar bursts** (e.g. a bad radar day/two) that transiently spike switching across many properties at once — that's radar-forced, not the feature ringing; it converges within days.
- Beware weekend/reporting cadence: daily "networks that switched" counts thin out on weekends; normalize by networks present and note noise.

### General
- Compare like-for-like windows; account for the ~3-day propagation lag when identifying onset.
- Be honest: if a prior flag was a methodology artifact, say so explicitly.

---

## 6. Reading the reference docs (optional context refresh)

Both source docs are Google Docs, readable with the `gws` (Google Workspace CLI):

```bash
gws docs documents get --params '{"documentId": "<DOC_ID>"}' --format json
```

The JSON is verbose; extract just the text (paragraphs + tables) with a small script — see the walker used previously:

```bash
gws docs documents get --params "{\"documentId\": \"$1\"}" --format json 2>/dev/null | python3 -c '
import json,sys
d=json.load(sys.stdin)
def walk(content):
    out=[]
    for el in content:
        if "paragraph" in el:
            txt="".join(e.get("textRun",{}).get("content","") for e in el["paragraph"].get("elements",[]))
            if txt.strip(): out.append(txt.rstrip("\n"))
        elif "table" in el:
            for row in el["table"].get("tableRows",[]):
                out.append(" | ".join(" ".join(walk(c.get("content",[]))) for c in row.get("tableCells",[])))
    return out
print("\n".join(walk(d.get("body",{}).get("content",[]))))
'
```

(`WebFetch` and the Amazon internal reader CANNOT read these Google Docs — only `gws` is authenticated for docs.google.com.)

---

## 7. Deliverable format

For **each MDU**, give a one-line note in this style (numbers first, plain terms, no emoji):

> **CIG - Paragon Ranch** — Max 5GHz channel concentration dropped from 37.7% to 23.1%, radar strikes fell from 1,952 to 626, primary-util P90 improved from 16.5% to 11.9%, and bandwidth reduction partially activated (160MHz 99.9% to 92.1%, ~8% moved to 80MHz).

Include: peak channel occupancy before→after, distinct-channels change, radar (as days/trend, not a misleading sum), bandwidth 80MHz before→after, and (if relevant) primary-util percentiles and stability. Then, **only for issues that survive §5 methodology**, add a short flagged clause (e.g. "but bandwidth reduction did NOT activate — 80MHz stayed ~0% every post-day").

End with a short cross-cutting summary: how many properties are clean vs how many have confirmed anomalies, and list the confirmed anomalies with their type. Real anomaly categories that matter:
- **Cap-path activation gap on a genuinely ELIGIBLE property** — the real bandwidth anomaly, distinct from correctly-inert. A property that is Trieste-eligible AND has a **high <600 Mbps WAN share** (so the cap path *should* fire) but shows ~0% 80 MHz. Contrast with the non-anomaly: a fast-WAN property (≥600 Mbps mostly) showing ~0% is **correct by design** (Kiahuna: 96.9% ≥600 → correctly inert). Only flag when eligibility is confirmed and it still didn't activate — then suspect the FF isn't actually live on the subset, or a missing/failed speedtest. (Do NOT explain a gap by "few nodes per network" — node count doesn't gate activation; neighboring *networks* do, see §1.)
- **Sustained radar-rate rise** (survives the daily-series test across multiple days, AND is not the fleet-wide synchronized event — see §5).
- **Non-converged channel churn** (still high at the tail) — distinguish broad re-optimization (movers spread across many channels = healthy) from tight ≤3-channel ringing.
- **Concentration still pinned high (>25%) on a UNII-3 channel** (high-power bias).
- **Property too small** (<~60 networks) for the feature to matter.

Do not inflate the anomaly list. Most properties should read as "feature working as designed." Note that a *low* 80 MHz share is only an anomaly under the cap-path-eligible case above; on fast-WAN or util-driven properties, low OR high 80 MHz can both be correct.

---

## 8. The tracking sheet ("MDU ACS - Partner Testing Tracker")

The team maintains a Google Sheet tracking every MDU property. Read/write it with `gws sheets` (see §6 for the `gws` auth note — `WebFetch` cannot read it).

- **Spreadsheet ID:** `1TnDWdfCcefBjeilvKFW1ktQ1M4WCnI4pMjeZfnsu-GM`
- **Tabs:** `Property Tracker` (gid 0) — the main per-property tracker; `EC Links` (gid 2118081352) — community/Insight links.

`Property Tracker` columns (row 1 is the header; data starts row 2):

| Col | Field | Notes |
|---|---|---|
| A | Partner | ISP (Metronet, BAI Connect, ENCO, Hawaiian Telcom, Frontier, altafiber, …) |
| B | Property Name | e.g. "Country Club Towers II & III" |
| C | MDU ID | eero network id(s) — sometimes comma-separated; NOT the subset UUID |
| D | Model | hardware (Trieste = eero Pro 6E, etc.) |
| E | IOT SSID | Enabled/Disabled |
| F | Network Group | rollout group (Yaco, Default, …) |
| G | Property Role | e.g. "Sensitive to radar", "No test", control group |
| H | Radar Activity | data source (Looker) |
| I | EC | Insight |
| J | WAN speed distribution | source (Looker) — this is the §3 WAN grid |
| K | Status | e.g. "Both feature flags enabled", "Combined feature flag enabled" |
| L | Next step | |
| M | Notes | free-form (e.g. "Mostly <200Mbps") |
| N–Q | Monthly Notes | `December Notes`, `January Notes`, `February Notes`, `July Notes` — **this is where the per-analysis findings go**; add a new month column for each analysis round |

Read a range:
```bash
gws sheets spreadsheets values get --params '{"spreadsheetId": "1TnDWdfCcefBjeilvKFW1ktQ1M4WCnI4pMjeZfnsu-GM", "range": "Property Tracker!A1:Q44"}' --format json
```
Write a cell/column (only when the user asks — confirm first):
```bash
gws sheets spreadsheets values update \
  --params '{"spreadsheetId": "1TnDWdfCcefBjeilvKFW1ktQ1M4WCnI4pMjeZfnsu-GM", "range": "Property Tracker!Q<row>", "valueInputOption": "USER_ENTERED"}' \
  --json '{"values": [["<the one-liner note for that property>"]]}'
```

Uses:
- **Cross-reference / sanity-check:** match your analysis against the sheet's `Status`, `Model`, `WAN speed`, and prior monthly notes. If the sheet says a property is a control group (no FF), don't expect activation.
- **Look up subset/network IDs** by property name when the user only gives names.
- **Write findings back:** the natural home for the §7 one-liners is the current month's Notes column. When updating, keep the one-liner style already used in prior months (concise, numbers-first). Match on `Property Name` (col B) to find the right row.
- **Beware superseded numbers:** if a note carries pre-correction values (e.g. a first-pass Regency Townes row), flag it and offer the corrected numbers rather than silently trusting the sheet.

The MDU ID column holds network IDs, not subset UUIDs; the subset UUID for the dashboard filter comes from the enablement notebook/ticket or the `EC Links` tab.
