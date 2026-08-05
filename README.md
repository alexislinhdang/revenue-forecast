# Macro Sensitivity & Revenue-at-Risk Analysis
### How exposed is Walmart's store revenue to economic shocks, and where should FP&A focus on?

---

## Abstract

> Analyzed 2.7 years of weekly sales across 45 Walmart stores to quantify how sensitive revenue is to macroeconomic shocks. A pooled regression with clustered standard errors found a statistically significant relationship between unemployment changes and sales growth (p < 0.001). This is a finding that survived two rounds of methodological correction. A SARIMA forecast beat the seasonal naive baseline (1.65% vs. 2.12% MAPE), and scenario stress-testing on top of it projects **$33.6M–$94.2M in 12-week revenue at risk (6%–17% of baseline)** under mild-to-severe labor market downturns.

---

## 1. Business Problem

An FP&A team needs to understand how exposed network revenue is to macroeconomic deterioration (unemployment, rising living costs) to prioritize contingency planning, cost flexibility, and inventory buffers ahead of a potential downturn.

A standard sales forecast alone only shows the numbers under standard and normal economic conditions. This analysis, on the other hand, quantifies which macro variables actually impact sales (with statistical significance), which stores appear most exposed, and how much revenue is at stake* under defined adverse scenarios.

---

## 2. Data

- **Source**: Walmart weekly sales dataset ([Kaggle: yasserh/walmart-dataset](https://www.kaggle.com/datasets/yasserh/walmart-dataset))
- **Scope**: 45 stores, weekly sales from Feb 5, 2010 – Oct 26, 2012 (6,435 rows)
- **Fields used**: `Weekly_Sales`, `Holiday_Flag`, `Temperature`, `Fuel_Price`, `CPI`, `Unemployment`
- **Store-level scale**: average weekly sales ranged from ~$260K to ~$2.1M (8x spread), median ~$967K. Due to the huge range in store sizes,  all sensitivity analysis was run on percent change in sales, not raw dollars, to avoid size dominance in regression results
- **Holiday effect**: holiday weeks average ~8% higher sales than non-holiday weeks; Thanksgiving week specifically drives the largest single spike in the full time series (nearly double baseline)
- **Known limitations**: no product-level detail; no geographic labels beyond Store ID; unemployment/CPI reported monthly but repeated weekly (see Limitations); 2010–2012 window may not reflect current macro conditions

---

## 3. Methodology

### Phase 1 — Exploratory Data Analysis
- Data cleaning and date parsing
- Trend, seasonality, and holiday-effect visualization
- *[link to notebook: `01_exploration.ipynb`]*

### Phase 2 — Sensitivity Analysis
- Converted `Weekly_Sales` to week-over-week percent change per store, to control for the 8x scale difference between stores
- Initial approach: separate OLS regression per store (`Sales_Pct_Change ~ CPI + Unemployment + Fuel_Price + Holiday_Flag + C(Month)`) — found no statistically significant coefficients at p<0.05 for any store, likely due to limited observations per store (~143 weeks) and a mismatch between a fast-changing dependent variable (weekly sales) and slow-moving macro levels
- Corrected approach: pooled regression across all 45 stores (6,390 observations) with store fixed effects, using **week-over-week changes** in CPI/Unemployment/Fuel_Price (matching the sales variable's transformation) rather than raw levels, and **clustered standard errors by store** to correct for non-independent errors within each store's time series
- This correction revealed a statistically significant finding for Unemployment (p < 0.001) that the original per-store models were underpowered to detect
- Extracted each store's original (level-based) CPI/Unemployment/Fuel_Price coefficients into a sensitivity table for descriptive clustering — see caveat in Key Findings
- Clustered the 45 stores into 4 risk tiers via k-means (elbow method confirmed k=4 as a reasonable cutoff)
- *[link to notebook: `02_sensitivity_analysis.ipynb`]*

### Phase 3 — Forecasting & Scenario Stress-Testing
- Aggregated to network-wide weekly sales; held out the final 12 weeks (Aug–Oct 2012) as a test set
- Baseline: seasonal naive forecast ("same week last year") achieved 2.12% MAPE — a strong bar, typical of strongly seasonal retail data
- SARIMA (1,1,1)(1,1,0,52) outperformed the baseline at 1.65% MAPE (RMSE reported alongside for error-distribution context); order selection used a conventional low-complexity starting specification given only ~2.5 annual cycles of training data
- Scenario stress-tests: applied the validated pooled unemployment coefficient (-0.232) to the SARIMA baseline under three adverse scenarios (+0.5 / +1.0 / +1.5 pts unemployment over 12 weeks), with effects compounding weekly
- *[link to notebook: `03_forecasting_scenarios.ipynb`]*

---

## 4. Key Findings

**Unemployment has a statistically significant relationship with sales growth.** A pooled regression across all 45 stores (6,390 observations), using week-over-week changes and clustered standard errors to correct for non-independent errors across stores and time, found that a rise in unemployment is associated with a decline in weekly sales growth (p < 0.001). Fuel price changes showed no significant relationship (p = 0.51). CPI changes showed a marginal, less certain relationship (p = 0.034 after correction, p = 0.52 before), which is a nonconclusive result.

**Note on the per-store clustering below**: the original per-store regressions (used to build the risk-tier segmentation) did not reach statistical significance individually, most likely due to limited observations per store (~143 weeks each). The risk tiers should be read as a **descriptive/exploratory segmentation** rather than a statistically confirmed classification of store-level risk.

**Four descriptive risk tiers emerged from clustering (k=4):**
| Tier | # Stores | Unemployment sensitivity | Explanation |
|---|---|---|---|
| Stable | 25 | -0.009 | Barely reacts to any macro variable |
| Resilient | 9 | -0.034 | Mild sensitivity; slightly *benefits* from rising CPI |
| Moderately Exposed | 9 | -0.056 | Meaningful negative unemployment sensitivity |
| High-Risk | 2 | -0.194 (avg) | Dramatically more exposed than every other cluster |

**Counter-intuitive finding**: the two most exposed stores in this descriptive segmentation (26 and 40) are both almost exactly median-sized (~$1.00M and ~$964K weekly average, against a $967K median across all 45 stores) — not the smallest or most marginal stores. Even treated descriptively, this is worth noting: it suggests store size alone would be a poor proxy for risk exposure, which is directionally consistent with the validated pooled finding that macro sensitivity is a real, measurable phenomenon independent of scale.

**Revenue at risk under adverse scenarios (Phase 3):** Layering the validated unemployment coefficient onto a SARIMA baseline forecast (which beat the seasonal naive benchmark, 1.65% vs 2.12% MAPE):

| Scenario | 12-week revenue | At risk vs. baseline |
|---|---|---|
| Baseline (no change) | $558.3M | — |
| Mild downturn (+0.5 pts) | $524.7M | $33.6M (-6.0%) |
| Moderate downturn (+1.0 pts) | $493.3M | $65.0M (-11.6%) |
| Severe downturn (+1.5 pts) | $464.1M | $94.2M (-16.9%) |

---

## 5. Recommendation

Labor-market conditions are the macro variable that should be monitored for revenue planning. Unemployment was the only variable with a robust, statistically significant relationship to sales growth, where rising unemployment leads to lower sales on average. Finance leadership should (1) incorporate unemployment-linked scenario ranges into quarterly revenue guidance, using the stress-test bands above as planning tolerances; (2) treat the exploratory risk-tier segmentation as a starting watchlist while validating store-level exposure with longer data before committing resources; and (3) not assume store size proxies for economic resilience, since the segmentation suggests risk concentrates independently of scale.

---

## 7. Tools & Techniques

`Python` · `pandas` · `statsmodels` (OLS, clustered SEs, SARIMA) · `scikit-learn` (k-means) · `matplotlib` 

---

## 8. Repository Structure

```
revenue-forecast/
├── data/
│   ├── Walmart.csv
│   ├── sensitivity_results.csv
│   ├── pooled_model_results.csv
│   └── scenario_results.csv
├── notebooks/
│   ├── 01_exploration.ipynb
│   ├── 02_sensitivity_analysis.ipynb
│   └── 03_forecasting_scenarios.ipynb
│   └── app.py
└── README.md
```

---

## 9. Limitations & Next Steps

- **Per-store estimates are underpowered**: individual store regressions (~143 weeks each) did not reach statistical significance, which is why the risk-tier clustering is framed as descriptive. A formal test of whether sensitivity truly varies by store (interaction model: `Unemployment_Change × Store`) is the natural next step.
- **Unemployment data is monthly, repeated weekly**: ~92% of week-over-week unemployment changes are zero, so the effective variation behind the pooled estimate is smaller than the row count suggests. Clustered standard errors partially address this; monthly-frequency modeling would address it directly.
- **Linear extrapolation in scenarios**: the stress tests apply a coefficient estimated on small weekly fluctuations to larger sustained shifts. This is a standard but strong assumption; the relationship may be nonlinear at recession-scale moves.
- **Single-holdout evaluation**: model comparison used one 12-week window; a production version would use rolling-origin cross-validation (e.g., `TimeSeriesSplit`) and AIC-based order search (e.g., `auto_arima`) rather than a single conventional SARIMA specification.
- **Dataset constraints**: no product-level detail, no geographic labels beyond Store ID, and a 2010–2012 window that may not reflect current macro dynamics. Store-level geography would enable a true regional risk map; live macro data via API (e.g., FRED) would keep the scenario tool current.

---

## Contact

Linh (Alexis) Dang · [linkedin.com/in/alexislinhdang](https://linkedin.com/in/alexislinhdang) · alexisdang@berkeley.edu