# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Context

Financial Econometrics I final project at FSV CUNI — volatility forecasting on a single assigned asset (Group 17, asset `20.RData`). Authors: Markéta Ondřejová, Pavel Bittnar, Aneta Mudruňková. Deliverable per the brief (`../FE_I_Project_2026.pdf`) is the notebook exported to HTML; teams are paired for peer review at the oral exam.

## Running the Notebook

The analysis lives entirely in `Project.ipynb` (R kernel, R 4.4.2+). Cells must run sequentially — many later cells depend on objects (`df`, `arma_ic_table`, `garch_ic_table`, fitted models) built once in the long "Setup" cell under `## In-sample Fit`.

**Data setup** — `data_project.zip` is gitignored. Place it at the project root before running; the first data cell does `unzip("data_project.zip", exdir = "data_project")` then `load("data_project/20.RData")`. To switch assets, change `20.RData` only.

**R package dependencies:**
```r
install.packages(c("xts", "zoo", "dplyr", "tseries", "KernSmooth",
                   "rugarch", "modelsummary", "sandwich", "lmtest",
                   "knitr", "tidyr"))
```
`tidyr` is used (`tidyr::pivot_wider`) but not in the `library()` block — it must be installed.

## Assignment Scope vs. Current State

The brief requires four parts. The notebook currently covers parts 1–2 only:

1. **Data description** — done (summary stats, time-series + ACF plots, written commentary).
2. **In-sample fit** — done for all six required models: `AR(1)-RV`, `HAR`, `HAR-RSV`, `HAR-Rskew-RKurt`, `AR(1)-GARCH(1,1)`, `AR(1)-Realized-GARCH(1,1)`. Newey-West SEs for the lm models; MSE/MAE comparison table.
3. **Forecasts** — **not yet implemented.** Needs expanding-window (start = 750) and rolling-window (length = 750) out-of-sample forecasts for all six models, forecast-error plots, MSE/MAE, pairwise Diebold–Mariano (MSE loss), and per-model Mincer–Zarnowitz regressions.
4. **Summary** — **not yet written** (≤ ½ page comparing model performance).

Final deliverable must be an HTML export of the notebook, file name including all team member names.

## Architecture & Conventions

**Single notebook, top-to-bottom flow.** All helper functions, data prep, model fits, and result tables are defined in the early cells; later cells just print pre-computed objects (`arma_ic_table`, `estimates_part1`, `comparison_table_IC`, …). When adding the forecasting section, follow the same pattern: do the heavy computation in one setup-style cell, then have small display cells.

**Plotting helpers** (defined in the third code cell) — use these instead of base `plot()` to keep figure styling consistent:
- `plot_ts(date, y, main, ylab)` — single series with yearly x-axis.
- `plot_multi_ts(date, y_list, labels, main, ylab, cols)` — overlay; expects equal-length series and `date`.
- `plot_acf(x, lag.max, main)` / `plot_pacf(...)` — drop lag 0, draw 95% CI bands.
- `plot_fit_vs_rv(date, rv, fit, model_label, model_col)` — observed RV vs one model.

**Color palette** — `my_cols` (navy, red, green, purple, brown, ochre, gray, black). The in-sample fit comparison already assigns these per model; reuse the same mapping in the forecasting section so the same model is the same color throughout.

**Plot device sizing** — `options(repr.plot.width = 20, repr.plot.height = 8.5)` and `lecture_par()` (large `cex`, thick `lwd`) are set once and apply to all subsequent plots. Don't reset `par()` mid-notebook unless you restore `lecture_par()` after.

**Summary tables** — `summary_table()` for descriptive stats of a data.frame; `garch_summary_table(fit, name)` extracts `fit@fit$matcoef` from a `ugarchfit` object into a tidy data.frame with significance stars. `modelsummary(..., output = "data.frame")` is wrapped through `kable()` rather than letting `modelsummary` render directly — needed because the notebook prints via `knitr::kable`.

## Important Implementation Notes

- **Data scale.** `RV`, `RV_p`, `RV_n` are reported on the **realized-volatility (sqrt) scale**, not the variance scale. So `RV_p + RV_n ≠ RV`, but `RV_p^2 + RV_n^2 ≈ RV^2`. Forecasting code that compares against `df$RV` must compare on the same scale — `sigma()` from `ugarchfit` returns conditional standard deviation, which is the right scale here.
- **GARCH fits use raw returns**, not the lag-trimmed `df`. `ret_xts` is built from `df$date`/`df$ret` after the `na.omit(df)` step, so all six models share the same 1478-obs index — exploit this when computing forecast errors.
- **Realized GARCH** requires both `data = ret_xts` and `realizedVol = rv_xts` (the latter on the volatility scale), with `solver = "hybrid"`. Other solvers tend to fail to converge on this data.
- **Lag bug already in setup.** Lines computing `df$RS_lag1` / `df$RK_lag1` appear twice (once with `dplyr::lag`, once with bare `lag`). The bare `lag` after `library(dplyr)` resolves to `dplyr::lag`, so it's harmless — but if you refactor, remove the duplicate.
- **GARCH order choice.** IC table favors `GARCH(1,2)` on AIC; notebook text justifies picking `GARCH(1,1)` on parsimony. Keep this justification if you re-run with a different asset that may flip the ranking.
