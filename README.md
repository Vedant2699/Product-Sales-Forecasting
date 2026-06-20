# Product Sales Forecasting

A time-series forecasting pipeline that takes 36 months of historical sales (Jan 2022 – Dec 2024) across 10 products and produces 6-month-ahead forecasts, with model selection handled per product rather than applied uniformly.

## Why model selection matters here

Ten products rarely share one demand pattern. In this dataset, a quick review of the series showed at least three distinct behaviors: steady or strong upward trends (Product_1, Product_10), seasonality-like fluctuation (Product_2), and sparse or intermittent demand with frequent zeros (Product_5, Product_9). A single model fit to all ten would be misspecified for at least some of them. The pipeline instead evaluates multiple candidate models against each product's own history and lets the data decide.

## Pipeline

**1. Data loading and reshaping**
Raw data arrives in wide format (one row per product, one column per month) with metadata rows above the actual table. It's parsed, the header row located, and reshaped into long format (`Product`, `Month`, `Sales`) — the standard layout for per-group time-series work in pandas.

**2. Data quality checks**
Missing values and zero counts are checked per product before any modeling, since zero-heavy series (intermittent demand) need different treatment than continuous ones and silently feeding them into ARIMA would produce misleading fits.

**3. Feature engineering**
Lag features (t-1, t-2, t-3), a 3-month rolling mean, a 3-month rolling standard deviation (volatility), and a time index are generated per product. These aren't fed directly into the statistical models used here, but they characterize each series' trend, memory, and volatility, and are the basis for the model-selection reasoning and for any future move to a regression-based or gradient-boosted approach.

**4. Model comparison**
For each product, the last 6 months are held out as a test set. Three candidates are fit on the remaining history and scored on the holdout using Mean Absolute Percentage Error (MAPE):
- **Naive** — carries the last observed value forward; the baseline every other model has to beat
- **Holt's Exponential Smoothing** (additive trend) — for products with a consistent directional trend
- **ARIMA(1,1,1)** — for products with more complex autocorrelation structure

The lowest-MAPE model per product is recorded in the comparison table, making the model choice auditable rather than assumed.

**5. Final forecasting**
ARIMA(1,1,1) is fit on the full history for each product and used to generate the 6-month-ahead forecast. Where ARIMA fails to converge (a real risk for short or sparse series), the pipeline falls back to a last-value carry-forward rather than erroring out, so every product still gets a usable forecast.

## Files

| File | Description |
|---|---|
| `Product_Sales_Forecasting.ipynb` | Full pipeline: load → reshape → quality checks → feature engineering → model comparison → final forecast |
| `interview_question.xlsx` | Raw input: monthly sales by product, Jan 2022–Dec 2024 |
| `forecast_output.csv` | Output: 6-month-ahead forecast per product (`Product`, `Month_Ahead`, `Forecast`) |

## Honest limitations

- ARIMA(1,1,1) is applied as a fixed order across all products in the final step rather than tuned per series (e.g. via `auto_arima` or a grid search on AIC); the order was reasonable for this dataset but isn't guaranteed optimal for each product individually.
- The Naive/Holt/ARIMA comparison and the final ARIMA-only forecast are run as separate steps — the comparison step doesn't yet feed its winning model back into the forecast step, so the "best model" label is currently diagnostic, not automatically wired into production.
- Intermittent-demand products (frequent zeros) are still passed through ARIMA in the final step; a dedicated method (e.g. Croston's method) would likely serve them better than the fallback used here.
- No confidence intervals are reported alongside the point forecasts.

## Requirements

```
pandas
numpy
scikit-learn
statsmodels
openpyxl
```
