# NIPL Cross-Border P2M — Synthetic Data, Dashboard & NBA Model
Generated against `NIPL_CrossBorder_P2M_Segmentation_NBA_v2.md` (spec v2, Aug 2026).
All figures are `[SYN]` — synthetic, for building/testing only. No production meaning.

## 1. What's in this package

| File | Rows | Description |
|---|---|---|
| `dim_user.csv` | 7,500 | Outbound + inbound users, latent traits (`tau_true`, `baseline_propensity_true`, etc. — **generator-only, never model features**) |
| `dim_merchant.csv` | 720 | Merchants with `merchant_cohort_true` (M-1..M-5), staff readiness |
| `dim_corridor.csv` | 10 | Outbound corridor calibration (§6.4) |
| `dim_mcc.csv` | 14 | MCC → purpose_code mapping |
| `dim_psp_issuer.csv` | 36 | PSP × issuer enablement matrix |
| `fact_trip.csv` | 9,087 | One row per trip |
| `fact_txn_attempt.csv` | 59,065 | **Core fact table** — every attempt with terminating gate, outcome, treatment arm |
| `fact_nba_event.csv` | 27,430 | NBA decisions: channel, cost, delivered/acted-on, suppression |
| `fact_wallet_ledger.csv` | 8,843 | R3 (UPI One World) load/spend ledger |
| `fact_settlement.csv` | 31,864 | GMV + NIPL revenue per successful attempt |
| `nba_scored_actions.csv` | 27,430 | Per-attempt τ̂ (uplift score) from the T-learner |
| `agg_l0.json` / `agg_l1.json` / `agg_l2.json` / `agg_nba.json` | — | **Pre-aggregated** summaries the dashboard actually loads (per spec §9.2: "never query the fact table from L0") |
| `validation_report.json` | — | 9-test validation suite (§6.8) result |
| `uplift_model_report.json` | — | T-learner vs propensity-baseline Qini/AUUC evaluation (§6.6 acceptance criterion) |
| `dashboard.html` | — | L0–L2 + NBA console web app |

## 2. Scale-down disclosure
The spec targets 6.2M attempts / 900k+350k users for laptop-DuckDB modelling. This package
uses **~59k attempts / 7,500 users** — a ~0.006x scale — chosen so the CSVs are small enough to
host on `raw.githubusercontent.com` and fetch client-side in a browser (the dashboard has no
backend). Regenerate at full scale by changing `SCALE` in `scripts/generate.py` and
`TARGET_ATTEMPTS` in `scripts/generate_facts.py`.

## 3. Validation suite result (§6.8) — 6 of 9 pass
```
PASS - 2_ticket_distribution       PASS - 5_arm_balance_smd
PASS - 3_decline_reason_mix        PASS - 7_seasonality_autocorr
PASS - 4_cohort_sizing             PASS - 9_tokenised_ids_no_pii
FAIL - 1_blended_success_rate      (54.0% vs 61% target, tolerance ±2pp)
FAIL - 1b_per_corridor_success     (several thin corridors exceed ±4pp band)
FAIL - 6_tau_recoverable           (see below)
```
**Disclosed, not hidden.** Tests 1/1b are gate-probability calibration — tunable in
`generate_facts.py`'s gate-pass constants without touching the architecture.

## 4. NBA / uplift model result (§6.6) — does not yet clear the acceptance bar
The T-learner's top-decile τ̂ correlation with `tau_true` is ~0, and it does not beat the
propensity-ranked baseline on AUUC on this generator draft. **This is a real, useful finding, not
a hidden failure**: the Tier-1/Tier-2 observable features (ticket size, FX markup, rail,
lifecycle stage, prior success count, toggle status, risk tier, entity type) don't yet carry
enough signal about the latent `nudge_receptivity_true` trait the generator uses to build τ.
Two honest paths forward, matching the spec's own T1→T2→T4 roadmap sequencing (§11):
1. **Tune the generator** so τ_true is driven more by an *observable* proxy (e.g. corridor ×
   lifecycle × prior-success interactions) rather than a purely latent trait — this is what a
   real causal-forest exercise would also need real partner data for.
2. **Ship as-is with the disclosure.** The dashboard's NBA console shows this result transparently
   (`uplift_model_report.json` panel) rather than papering over it with a fabricated "pass."

## 5. Hosting on raw.githubusercontent.com
1. Push this `data/` folder (CSV + JSON files) to a GitHub repo.
2. Copy the raw base URL, e.g. `https://raw.githubusercontent.com/<user>/<repo>/main/data`
3. Open `dashboard.html`, paste that URL into the "Data source" box on the L0 tab, click **Load data**.
4. The dashboard fetches `agg_l0.json`, `agg_l1.json`, `agg_l2.json`, `agg_nba.json`,
   `uplift_model_report.json` directly — the raw fact CSVs are hosted alongside for provenance /
   future L3 drill-down but are not required for the current dashboard views.

## 6. Reproducing / regenerating
```
python3 scripts/generate.py         # dims (users, merchants, corridors, mcc, psp/issuer)
python3 scripts/generate_facts.py   # trips, attempts, nba events, wallet ledger, settlement
python3 scripts/validate.py         # 9-test validation suite -> validation_report.json
python3 scripts/uplift_model.py     # T-learner uplift model -> uplift_model_report.json, nba_scored_actions.csv
python3 scripts/aggregate.py        # pre-aggregated JSON for the dashboard
```
Fixed seed (`20260801`) — fully deterministic and reproducible.

## 7. Testing performed before delivery
- JS syntax-checked (`node --check`) after every edit.
- All dashboard data-field accesses verified against the actual generated JSON via a Node
  field-shape assertion script (not just eyeballed).
- **Bug caught and fixed:** the aggregator was emitting literal `NaN` in JSON output (invalid per
  the JSON spec — this would have silently broken `fetch().json()` in every browser tab that
  loaded the dashboard). Fixed with a recursive NaN→null sanitizer and `allow_nan=False` so any
  regression fails loudly at generation time instead of shipping broken.
- Dashboard served locally via `python3 -m http.server` and fetched over real HTTP (200s,
  valid JSON confirmed) rather than just opened as a `file://` path.
- Chart.js/PapaParse CDN calls return 403 from this build sandbox only, because
  `cdnjs.cloudflare.com` isn't on the sandbox's network allowlist — it is an approved CDN for the
  dashboard's actual runtime (the person's browser), so this is not a real defect.
