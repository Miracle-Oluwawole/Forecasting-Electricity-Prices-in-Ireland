# Forecasting Irish Day-Ahead Electricity Prices

Forecasting next day half-hourly prices in Ireland's Single Electricity Market (SEM). The project uses one year of SEMO auction results plus EirGrid wind, solar and demand data, and checks forecasts on both accuracy and trading value.

![Python](https://img.shields.io/badge/Python-3.11+-blue) ![scikit-learn](https://img.shields.io/badge/scikit--learn-1.x-orange) ![Azure](https://img.shields.io/badge/Storage-Azure%20Blob-0078D4)

---

## Overview

Every morning, before the 11:00 day-ahead auction closes, the model forecasts all 48 half-hourly prices for the next delivery day. It uses only information a trader would actually have at that time.

| | |
|---|---|
| **Market** | All-island SEM day-ahead auction (SEMOpx) |
| **Data** | 17,328 half-hours, 18 Oct 2025 – 13 Oct 2026 |
| **Models** | Naive benchmarks, Ridge regression, histogram gradient boosting, quantile gradient boosting |
| **Evaluation** | Untouched 8-week test set, weekly rolling backtest, battery trading test |

## Key results

Test period: 19 Aug – 13 Oct 2026 (2,688 half-hours).

| Model | MAE (€/MWh) | RMSE (€/MWh) | Skill vs naive |
|---|---|---|---|
| **Ridge regression (price history)** | **34.5** | 48.1 | **27%** |
| Naive: 7-day same-period mean | 38.9 | 55.9 | 17% |
| Gradient boosting (price history) | 41.7 | 55.5 | 11% |
| Naive: same half-hour yesterday | 47.0 | 69.7 | — |

**Effect of adding wind, solar and demand forecasts.** These were compared on identical rows with gradient boosting:

- RMSE fell 12% (53.1 → 46.8 €/MWh), so there were fewer large misses.
- MAE fell 1.2%.
- In the battery trading test, the forecasts captured **84% of the value available with perfect foresight**, against 44% without fundamentals and 35% for the naive rule.
- Forecast **net load** (demand − wind − solar) is the single most important input.

**Main finding.** All fitted models under-forecast by 15–31 €/MWh in the test period. Without gas and carbon prices, the models can't follow shifts in the overall price level. That's the priority next step.

## Pipeline

```mermaid
flowchart LR
    A[SEMO day-ahead results] --> C[Quality checks]
    B[EirGrid wind, solar, demand] --> C
    C --> D[Leakage-safe features]
    D --> E[Baselines and models]
    E --> F[Test set, rolling backtest, battery test]
    C --> G[(Azure Blob Storage)]
    F --> G
```

## Methodology

**No look-ahead (leakage controls)**
- Price lags of at least 48 half-hours only. Shorter lags point at the auction being forecast.
- The sterling price and same-day auction volumes are excluded. Volumes are used only with a one-day lag.
- Wind and solar enter as EirGrid *forecasts*. Actuals enter only up to the day before.
- An "oracle" model uses actual next-day wind and demand. It's labelled clearly and shows only the upper bound of what fundamentals can add.

**Features**
- Calendar in Irish local time: half-hour, weekday, public holidays, cyclical encodings
- Price history: same period 1, 2, 3 and 7 days earlier, plus the 7-day same-period mean
- Previous-day summary: mean, max, min, standard deviation, last price
- Fundamentals: wind and solar forecasts, demand basis, net load, wind share, daily net-load peak and trough

**Validation**
- Data split by whole delivery days in time order, never shuffled: 242 training days, 56 validation days, 56 test days.
- Settings tuned on validation only; the test set is scored once.
- Weekly rolling retrain to mimic live use.

**Metrics:** MAE (primary), RMSE, evening-peak MAE, bias, and skill vs naive. MAPE isn't used because zero and negative prices make it unstable.

## Repository structure

```
├── Forecasting Irish Electricity Price.ipynb   # Full pipeline: extraction to evaluation
├── auction_data/                               # Local outputs (git-ignored)
│   ├── irish_day_ahead_auction_results.csv
│   ├── irish_day_ahead_model_data.csv
│   ├── eirgrid_fundamentals_30min.csv
│   ├── test_forecasts*.csv
│   └── eirgrid_cache/                       # Daily EirGrid responses
├── Forecasting Irish Electricity Price.docx  # Analysis report
├── requirements.txt
└── README.md
```


## Limitations

- **No fuel prices.** There are no gas or carbon prices, so changes in the overall price level are missed.
- **Short test period.** Eight weeks in one season; the literature recommends at least one year.
- **Possibly revised forecasts.** EirGrid dashboard forecasts may have been updated after the 11:00 gate closure, so fundamentals results are slightly optimistic.
- **No demand forecast.** Last week's demand is used as a stand-in.
- **Under-confident ranges.** The P10–P90 intervals cover 48% of outcomes against an 80% target.

## Next steps

- [ ] Add daily gas (NBP/TTF) and carbon (EUA) prices
- [ ] Forecast the price relative to the 7day mean, so tree models follow level shifts
- [ ] Ensemble Ridge and gradient boosting
- [ ] Calibrate intervals with conformalised quantile regression
- [ ] Extend history to 2022+ and test over 12 months with daily retraining
- [ ] Diebold–Mariano significance tests between models

## Data sources and disclaimer

Price data comes from SEMO via the `irish-electricity-data` package. Wind, solar and demand data comes from the EirGrid Smart Grid Dashboard, through an unofficial endpoint. This project is for research and portfolio purposes only and is not trading advice.

## Author

**Miracle Oluwawole**
