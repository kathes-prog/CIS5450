# Geopolitical Conflict and Oil Price Returns

**CIS 5450 Final Project — Elijah Jeoung, Diego Salamanca Murphy, Katherine Smith, Luciano Reyes Botello**

Can daily conflict activity in the Middle East predict next-day WTI crude oil returns? We joined 12 years of ACLED conflict event data with WTI futures prices and ran the results through five regression and three classification models to find out. The short answer: oil markets are remarkably efficient at pricing publicly observable geopolitical risk. Conflict features explain less than 1% of return variance — lagged oil returns dominate everything else.

---

## How to Run It

The project runs as a Google Colab notebook. You'll need a Google account and a copy of the ACLED dataset (see below).

**1. Clone or download the notebook**
```bash
git clone https://github.com/your-repo/cis5450-oil-conflict.git
```

**2. Upload to Google Colab**

Open [colab.research.google.com](https://colab.research.google.com), click *File → Upload notebook*, and select `CIS_5450_Final_Project.ipynb`.

**3. Mount Google Drive and add the ACLED data**

The notebook reads ACLED data from `/content/ACLED Data_2026-04-28.csv`. Export your own filtered dataset from [acleddata.com](https://acleddata.com) with these filters:

- **Date range:** January 2013 – January 2025
- **Region:** Middle East
- **Countries:** Iran, Iraq, Jordan, Qatar, Saudi Arabia, Syria, UAE, Yemen
- **Event types:** Battles, Explosions/Remote violence, Violence against civilians, Riots, Protests

Place the CSV in your Google Drive and update the file path in cell 1b if needed.

**4. Run all cells**

*Runtime → Run all*. WTI oil price data is fetched automatically via `yfinance` — no manual download required.

**Dependencies** are all pre-installed in Colab. If running locally, install via:
```bash
pip install numpy pandas matplotlib seaborn duckdb plotly yfinance scikit-learn scipy
```

---

## Tech Stack

| Layer | Tools |
|---|---|
| Data ingestion | `yfinance` (WTI futures), ACLED CSV export |
| Data wrangling | `pandas`, `numpy`, `re` (regex normalization) |
| SQL joins | `duckdb` (in-process SQL for the oil × conflict LEFT JOIN) |
| Visualization | `matplotlib`, `seaborn`, `plotly` |
| Modeling | `scikit-learn` — Linear Regression, Ridge, Lasso, Random Forest, Gradient Boosting |
| Validation | `TimeSeriesSplit`, `RandomizedSearchCV`, bootstrap resampling (10,000 iterations) |
| Environment | Google Colab + Google Drive |

---

## Development Process

**Data pipeline.** The two datasets live on different calendars — ACLED records events on any day, while WTI futures only trade on weekdays. We used DuckDB to perform a SQL `LEFT JOIN` on date, filling zero for trading days with no recorded conflict events. Getting this merge right (without leaking future data or inflating the event-count signal) took more iteration than expected.

**Feature engineering.** We aggregated ACLED events to the country-day level, one-hot encoded the five event categories, added three lags each for conflict signals and oil returns, and included day-of-week controls. Keeping the feature set interpretable was a deliberate choice — we wanted any signal to be traceable back to something concrete.

**Modeling.** We ran two parallel tracks: regression (predicting the continuous next-day return) and classification (predicting direction — up or down). All models used `TimeSeriesSplit` to respect temporal ordering; standard k-fold would have let future data leak into training folds.

**Problems we hit.**
- *Lasso zeroing everything.* At the cross-validated optimal α=1.0, Lasso dropped all 33 features. This was striking — and ultimately informative — but it meant we had to be careful about interpreting the Lasso MAE as a modeling success rather than a prediction of the training mean.
- *Gradient Boosting overfitting on returns.* GBR's test R² was −11.22 despite a train R² of 0.71. Even with shallow trees (`max_depth=3`) found by `RandomizedSearchCV`, the model memorized return-scale noise. We report it only on directional metrics (AUC, accuracy) where it genuinely performs best.
- *Calendar mismatch.* Weekend days in the ACLED data needed to be mapped onto the following Monday's trading session. The DuckDB LEFT JOIN handled this cleanly, but constructing the complete country-day panel first was a necessary intermediate step.

---

## Results Summary

All five regression models produced negative test R² — none beat the trivial baseline of predicting the mean return every day. Directional accuracy across the board was statistically indistinguishable from a coin flip (bootstrap 95% CI for the best model: [0.4953, 0.5150]).

Random Forest feature importances told the clearest story: the three lagged oil-return features account for ~80% of total importance; all conflict features combined contribute less than 9%. The model is basically saying that the best predictor of tomorrow's oil return is what oil did recently — not how many battles happened in Yemen.

This isn't a modeling failure. It's evidence that oil markets efficiently price publicly observable geopolitical information.

---

## Wishlist / Future Work

- **Intraday data.** The biggest limitation is the one-day lag. If markets react to events within hours, the signal is already priced in by the next close. Matching ACLED timestamps to intraday return windows would directly address this.
- **Broader geographic scope.** The current model is Middle East–only. Expanding to Russia, Venezuela, West Africa, and adding OPEC production decision data would make the feature set far more representative of actual supply-side risk.
- **Interaction features.** Lasso zeroed individual features, but interaction terms (fatalities × event type, country × event type) might recover linear structure that raw counts miss. The natural next step is engineering interactions, then running L1 selection again.
- **Regime-aware modeling.** EDA revealed clear volatility clustering across the 12-year window. A Hidden Markov Model approach would let conflict features carry different weights in calm vs. turbulent regimes, rather than averaging across them in a single static model.
- **Sentiment signals.** Layering in NLP-based news sentiment alongside ACLED event data could capture the *market interpretation* of conflict events, not just their raw occurrence.
