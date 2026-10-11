# When Confidence Meets Cost

**Timestamp-aware renewable forecasting, empirical interval calibration, and
stylized solar procurement — a corrected, reproducible analysis.**

This repository builds a leakage-free forecasting pipeline on a 15-minute
renewable-energy time series and connects it to downstream uncertainty
quantification and a stylized procurement decision. It also runs separate
experiments on SHAP attribution and a *simulated* seasonal-client federation.

Models are trained on data **before 2020**, tuned on **2020**, calibrated on
**2021**, and evaluated on **January–August 2022**, with additional
rolling-origin and repeated-seed variants.

## Authors

- **Sunawar Khan**
- **Habib Hamam**

---

## Repository structure

```
renewable-forecast-dispatch/
├── README.md
├── LICENSE
├── CITATION.cff
├── requirements.txt
├── .gitignore
├── notebooks/
│   └── renewable_forecast_dispatch.ipynb   # the full analysis
├── src/
│   ├── README.md
│   ├── daylight_calibration.py             # (user-supplied — see src/README.md)
│   └── daylight_reporting.py               # (user-supplied — see src/README.md)
├── data/
│   ├── README.md                           # dataset schema + provenance
│   └── Renewable.csv                        # (user-supplied — not committed)
└── output/
    ├── figures/                            # all generated .png figures
    └── tables/                             # all generated .csv tables
```

> `data/Renewable.csv` and the two `src/*.py` modules are **not** shipped with
> this repo. See `data/README.md` and `src/README.md` to supply them.

---

## Setup

```bash
# 1. clone, then create an environment (Python 3.12 recommended)
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 2. install dependencies
pip install -r requirements.txt

# 3. place the dataset
#    -> data/Renewable.csv   (schema in data/README.md)
```

## Run

```bash
jupyter notebook notebooks/renewable_forecast_dispatch.ipynb
# then: Run All
```

Or headless, from the repository root:

```bash
jupyter nbconvert --to notebook --execute --inplace \
    notebooks/renewable_forecast_dispatch.ipynb
```

Paths are resolved relative to the repository root, so the notebook works
whether launched from the root or from inside `notebooks/`. Every figure lands
in `output/figures/`, every table in `output/tables/`, and all other artefacts
(fitted models, prediction arrays, audit JSON) at the `output/` root.

---

## What the notebook produces

### Figures → `output/figures/`

| File                        | Section | Content                                                        |
| --------------------------- | ------- | -------------------------------------------------------------- |
| `image16.png`               | 3       | Contemporaneous-estimation & leakage diagnostic                |
| `image17.png`               | 4       | Model benchmark + feature-group ablation (solar, 1 h)          |
| `image18.png`               | 5       | Multi-horizon R² (solar & wind, endogenous vs NWP-oracle)      |
| `image19.png`               | 6       | 4-day probabilistic solar forecast (90% CQR interval)          |
| `image20.png`               | 7       | Empirical interval coverage + normalized width                 |
| `image21.png`               | 7       | Quantile reliability diagram (solar, 1 h)                      |
| `image22.png`               | 8       | SHAP summary — solar model (NWP-oracle)                        |
| `image23.png`               | 8       | SHAP summary — wind model (NWP-oracle)                         |
| `shap_solar_endogenous.png` | 8       | SHAP summary — solar model (endogenous)                        |
| `shap_wind_endogenous.png`  | 8       | SHAP summary — wind model (endogenous)                         |
| `image24.png`               | 9       | Simulated federated training curve + scheme comparison         |
| `image25.png`               | 10      | Stylized procurement cost sensitivity                          |

### Tables → `output/tables/`

| File                       | Section | Content                                              |
| -------------------------- | ------- | ---------------------------------------------------- |
| `diagnostic.csv`           | 3       | R² under GHI-only / random-split / temporal-split    |
| `benchmark.csv`            | 4       | RMSE / MAE / R² for 5 models                          |
| `ablation.csv`             | 4       | Feature-group ablation                               |
| `horizons.csv`             | 5       | Horizon sweep (R², RMSE, skill) solar & wind         |
| `probabilistic.csv`        | 6       | PICP, PINAW, Winkler, pinball, coverage failures     |
| `rolling_coverage.csv`     | 6       | Rolling-origin interval coverage (2020/2021/2022)    |
| `rolling_point.csv`        | 6       | Rolling-origin point scores (3 seeds)                |
| `quantile_reliability.csv` | 7       | Nominal vs empirical quantile coverage               |
| `federation_seeds.csv`     | 9       | Per-seed federation R²                               |
| `federation.csv`           | 9       | Federation scheme means ± std                        |
| `federation_clients.csv`   | 9       | Per-client (seasonal) R²                             |
| `dispatch.csv`             | 10      | Procurement cost per imbalance multiplier + policy   |

### Other artefacts → `output/`

Fitted LightGBM boosters (`*.txt`), scalers/baselines (`point_baselines.joblib`),
federated/central/local weights (`*.npz`), prediction arrays (`*.csv`), and audit
files (`data_audit.json`, `split_counts.json`, `duplicate_rates.json`,
`paired_bootstrap.json`, `demand_audit.json`). `data_audit.json` includes the
SHA-256 of the dataset actually read — record it for reproducibility.

---

## Method at a glance

1. **Data + synthetic wind** — physics-derived wind energy from `wind_speed`,
   `pressure`, `temp` (uncapped cubic aerodynamic proxy, *not* a rated curve).
2. **Causal features + blocked temporal split** — 15-min elapsed-time grid;
   windows that cross data gaps are rejected; no target/history interpolation.
3. **Leakage diagnostic** — shows how much of the reported contemporaneous
   R² ≈ 0.91 comes from GHI alone and from random-split leakage.
4. **Benchmark + ablation** — Persistence, Seasonal-naive, Ridge, Random
   Forest, LightGBM; paired moving-block bootstrap for GB vs RF.
5. **Multi-horizon forecasting** — endogenous vs NWP-oracle at 15 min / 1 h /
   3 h / 6 h for solar and wind.
6. **Conformalized Quantile Regression (CQR)** — quantile grid, monotone
   rearrangement, split-conformal calibration, probabilistic metrics.
7. **Reliability diagram** — quantile calibration, overall vs daytime.
8. **SHAP attribution** — endogenous and NWP-oracle point models.
9. **Simulated federation** — five seasonal partitions of **one** site;
   Local / FedAvg (non-IID & IID) / Centralized across three seeds.
10. **Stylized procurement** — economic-fractile policy vs median / raw-quantile
    / CQR-lower / one-sided / risk-cap / perfect-information references.
11. **Daylight calibration & causal online updates** — optional, needs the two
    `src/` modules (see below).

---

## Important caveats

These are carried over from the analysis and matter for interpretation:

- **Synthetic wind** is an uncapped aerodynamic proxy, not a measured or rated
  turbine output.
- **The "federation" is simulated** — five *seasonal partitions of a single
  site*, not five measured plants. It implements **no** secure aggregation and
  **no** differential privacy; shared moments and weights are not privacy
  protected.
- **CQR** gives marginal calibration only. Two-sided endpoints are **not** a
  calibrated CDF, and marginal coverage is **not** a per-decision conditional
  guarantee.
- **The 2022 window was inspected before** the daylight method was proposed, so
  Section 11's assessment is **exploratory**, not a fresh-test result.
- **Dataset provenance, availability and licence are unresolved** — document
  them in `data/README.md` before relying on or publishing results.

---

## Citation

If you use this code or analysis, please cite it as:

**Plain text**

> Khan, S., & Hamam, H. (2026). *When Confidence Meets Cost: Timestamp-aware
> renewable forecasting, empirical interval calibration, and stylized solar
> procurement* [Computer software]. GitHub.
> https://github.com/SunawarKhan/Daylight-Adaptive-Forecast-Calibration-for-Tariff-Aware-Solar-Procurement

**BibTeX**

```bibtex
@software{khan_hamam_2026_confidence_cost,
  author    = {Khan, Sunawar and Hamam, Habib},
  year      = {2026},
  title     = {When Confidence Meets Cost: Timestamp-aware renewable
               forecasting, empirical interval calibration, and stylized
               solar procurement},
  type      = {Computer software},
  publisher = {GitHub},
  url       = {https://github.com/SunawarKhan/Daylight-Adaptive-Forecast-Calibration-for-Tariff-Aware-Solar-Procurement}
}
```

A machine-readable version is provided in `CITATION.cff`
(GitHub shows a "Cite this repository" button automatically).

---

## License

Code is released under the MIT License (see `LICENSE`). The dataset is **not**
covered by this license and is not distributed here; see `data/README.md`.
