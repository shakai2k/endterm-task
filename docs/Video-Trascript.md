# Wide Forecast Functions: Video Developer Guide

## Introduction

Hello, and welcome to this developer guide for the wide-format forecasting functions.

In this video, I will explain how the forecasting workflow is structured, what each function does, and how the functions work together. I will also highlight the main inputs, outputs, safeguards, and edge cases that developers should understand before using them.

The workflow contains eight core functions:

1. `generate_naive_forecast_wide`
2. `validate_forecast_actual_wide`
3. `evaluate_forecast_wide`
4. `generate_seasonal_recovery_forecast_wide`
5. `generate_blended_recovery_forecast_wide`
6. `generate_sarima_forecast_wide`
7. `generate_prophet_forecast_wide`
8. `generate_timesfm_forecast_wide`

We will begin with the shared data format, move through forecast generation, validation, and evaluation, and then look at the more advanced forecasting models.

## Shared data format

All eight functions work with monthly pandas DataFrames in wide format.

The date column is normally called `Date` and contains labels such as `2023M03`. Every destination has its own numeric column, and every row represents one month.

For example:

| Date | Australia | Japan |
| --- | ---: | ---: |
| `2023M03` | 100 | 200 |
| `2023M04` | 110 | 220 |

Before comparing dates or performing date arithmetic, the functions convert these labels into monthly `pandas.Period` values. This matters because calendar operations must be performed on dates rather than strings. For example, one month before January 2024 is December 2023, and a monthly period handles that transition correctly.

With that shared format established, let us begin with the simplest forecasting method.

## 1. Naive forecasting

The first function is `generate_naive_forecast_wide`.

```python
generate_naive_forecast_wide(
    historical_actual_wide,
    cutoff_label,
    forecast_months,
    required_destinations=None,
    lag=1,
    date_column="Date",
)
```

This function creates a naive lag forecast for one or more destinations. In simple terms, it forecasts a month by reusing a value from a fixed number of months earlier.

If the lag is one, it uses the previous month's value. If the lag is twelve, it uses the value from the same month in the previous year. Any positive integer lag is supported.

The historical table provides the observed data. The cutoff identifies the final month whose actual value may be used, which prevents future-data leakage. The forecast-month list identifies the months to generate, and its original order is preserved in the output.

If `required_destinations` is omitted, every column except the date column is forecast. The function checks that the lag is a positive integer, that the date column exists, and that every requested destination is present. Boolean lag values are rejected.

After converting all month labels to periods, the function removes observations after the cutoff. If a historical month is duplicated, it keeps the last supplied row. It then generates the required periods chronologically using this rule:

```text
forecast for month m = value from month m minus the lag
```

If the source month is at or before the cutoff, the observed value is used. If it is after the cutoff but was forecast earlier, that forecast is reused recursively.

For example, suppose the cutoff is February 2023, the last observed value is 100, and we need a lag-one forecast for March, April, and May. March uses February's actual value. April uses March's generated value, and May uses April's generated value. The forecast is therefore 100 for all three months.

The output is a wide DataFrame containing the date first, followed by the requested destinations in their original order. It contains no model labels, diagnostics, or index column. If no months are requested, it returns an empty DataFrame with the correct columns. A missing source period produces `NaN`, while a missing destination or invalid lag raises a clear `ValueError`.

One detail is worth emphasizing: a requested month at or before the cutoff still follows the lag rule. The function does not simply copy the actual value from the same month.

Now that we can generate a basic forecast, the next step is to confirm that the data is structurally sound.

## 2. Forecast and actual validation

The validation function is `validate_forecast_actual_wide`.

```python
validate_forecast_actual_wide(
    forecast_wide_df,
    actual_wide_df=None,
    required_months=None,
    required_destinations=None,
    date_column="Date",
)
```

This function audits the structure and value quality of a forecast. It can also check whether forecast and actual tables can be aligned safely.

It has two modes. In forecast-only mode, the actual table is left as `None`. This is useful before export or submission. In forecast-versus-actual mode, an actual table is supplied so that both datasets can be checked before evaluation.

If `required_months` is omitted, the forecast table defines the alignment scope. If `required_destinations` is omitted, every forecast column except the date is treated as a destination.

The validator checks the required date column, expected row count, missing or extra months, duplicate dates, missing destinations, extra columns, missing values, nonnumeric values, and infinite values.

Duplicate dates are counted only after their first occurrence. Cell-quality issues are counted only for required destinations that are present and for months inside the required scope. If an entire destination column is absent, its name is reported separately rather than producing an artificial count of missing cells.

When actual data is provided, the function first subsets it to the alignment months. The original actual source may therefore contain extra months without failing the audit. It then checks month and destination coverage, duplicate dates, value quality, and one-to-one alignment with the forecast.

Two results are especially important: `can_align` and `is_valid`.

`can_align` tells us whether the tables have compatible months, destinations, row counts, and unique dates. It describes structural compatibility but does not guarantee usable values. In forecast-only mode, it is `None`.

`is_valid` is stricter. It is true only when every applicable structure, coverage, and value-quality check passes. Two tables can therefore be alignable while the overall audit is invalid because a value is missing or infinite.

Value problems are separated into three categories. A missing value is a null such as `None` or `NaN`. A nonnumeric value is present but cannot be converted into a number. A nonfinite value converts to a number but is positive infinity, negative infinity, or otherwise not finite.

The output is a fixed-schema dictionary with these fields:

```text
is_valid
can_align
forecast_row_count
actual_row_count
expected_row_count
missing_months_in_forecast
missing_months_in_actual
extra_months_in_forecast
extra_months_in_actual_source
missing_destinations_in_forecast
missing_destinations_in_actual
extra_columns_in_forecast
duplicate_forecast_dates
duplicate_actual_dates
missing_forecast_value_count
missing_actual_value_count
nonnumeric_forecast_value_count
nonnumeric_actual_value_count
nonfinite_forecast_value_count
nonfinite_actual_value_count
```

In forecast-only mode, the actual-related fields are `None`. Forecast tables are intentionally strict, so extra forecast months or columns make them invalid. Missing requirements, duplicates, missing values, nonnumeric values, and infinite values also cause the relevant audit to fail.

Once validation confirms that the tables can be aligned, we can measure forecast accuracy.

## 3. Forecast evaluation

The evaluation function is `evaluate_forecast_wide`.

```python
evaluate_forecast_wide(
    forecast_wide_df,
    actual_wide_df,
    training_actual_wide_df,
    start_month,
    end_month,
    required_destinations=None,
    naive_lag=1,
    date_column="Date",
)
```

This function aligns forecasts and actuals by month and destination. It then calculates MAE, MASE, and MAPE for each destination, followed by unweighted aggregate summaries.

The forecast and actual tables supply values for the evaluation period. A separate training table is used only to calculate the MASE denominator. This keeps the scale calculation cutoff-safe. The start and end months define the inclusive evaluation window.

The `naive_lag` controls the training pairs used for MASE. Official comparisons use a lag of one, even when evaluating a forecast that was generated with a lag of twelve.

After validating the inputs, the function converts dates to periods and subsets the forecast and actual data to the evaluation window. Duplicate dates are rejected because they make one-to-one alignment ambiguous. The tables are then inner-joined by month with one-to-one validation. Missing, nonnumeric, and infinite metric inputs become `NaN` so unusable pairs can be excluded safely.

### MAE

MAE means Mean Absolute Error. For each valid forecast-and-actual pair, the function calculates the absolute difference and then takes the mean.

```text
absolute error = absolute value of actual minus forecast
MAE = mean of the absolute errors
```

The field `n` records the number of valid pairs. If there are none, MAE is `NaN`.

### MASE

MASE means Mean Absolute Scaled Error. It compares forecast error with a naive error scale from the training history.

The function prepares the training data, keeps the last duplicated month, sorts it, and pairs every value with the value exactly `naive_lag` calendar months earlier. Exact calendar pairing prevents a missing month from turning two non-consecutive rows into a false lag pair.

```text
denominator = mean absolute difference between valid lagged training pairs
MASE = MAE divided by the denominator
```

The field `n_denominator_pairs` records how many pairs were used. MASE is unavailable when MAE is unavailable, no denominator pairs exist, or the denominator is missing or zero. In that case, `mase` is `NaN`, `mase_available` is false, and `denominator_warning` is true.

### MAPE

MAPE means Mean Absolute Percentage Error. It starts with the valid MAE rows but excludes zero actual values to avoid division by zero.

```text
MAPE = mean of the absolute percentage errors, multiplied by 100
```

If no valid nonzero actual values remain, MAPE is `NaN`.

### Evaluation output

The function returns two objects:

```python
destination_metrics, aggregate_metrics
```

The destination DataFrame contains the destination, valid-pair count, MAE, MASE, MAPE, denominator, denominator-pair count, MASE availability, and denominator warning.

The aggregate dictionary contains the number of destinations, mean MASE, median MASE, and mean MAPE. It includes only available metric values and gives every destination equal weight.

Lower metric values are better. MAE is expressed in the original units. MAPE expresses error as a percentage. A MASE below one is better than the chosen naive scale, one is equal to it, and above one is worse.

These first three functions give us the basic workflow: generate, validate, and evaluate. We will now look at the candidate models.

## 4. Seasonal recovery forecasting

The next function is `generate_seasonal_recovery_forecast_wide`.

```python
generate_seasonal_recovery_forecast_wide(
    historical_actual_wide,
    cutoff_label,
    forecast_months,
    required_destinations=None,
    date_column=FINAL_FORECAST_DATE_COLUMN,
    recovery_window=6,
    decay_horizon=12,
    ratio_clip_bounds=(0.1, 10.0),
    max_growth_multiple=1.5,
)
```

This method extends a lag-twelve seasonal-naive forecast with an estimate of recent demand recovery. It is designed for data with both a repeating annual pattern and a changing overall level, such as demand recovering after the COVID-19 disruption.

It preserves the previous year's seasonal shape and adds a recovery adjustment that gradually decreases over the horizon. The method is deterministic and uses only observations at or before the cutoff.

For each destination, it averages the latest valid observations inside the recovery window and compares them with the same calendar months one year earlier. Dividing the recent average by the earlier average gives the recovery ratio.

If there is no valid history, or if the earlier average is missing, zero, or negative, the ratio becomes a neutral value of one. It is also clipped to the configured bounds, which are 0.1 and 10 by default.

For every forecast step, the seasonal source comes from exactly twelve months earlier. If it is unavailable, the function carries forward the last valid cutoff-safe observation.

```text
decay weight = maximum of zero and 1 minus forecast step divided by decay horizon
applied ratio = 1 plus recovery ratio minus 1, multiplied by the decay weight
forecast = seasonal source multiplied by the applied ratio
```

The full recovery ratio is applied at the first step. Each later step uses less of it. At or after the decay horizon, the ratio becomes one, moving the result back toward an ordinary seasonal-naive forecast.

The output is capped at `max_growth_multiple` times the cutoff-safe historical maximum. Missing seasonal history falls back to the last valid value, while a destination with no usable history produces `NaN`.

This model is responsive, but its projections can sometimes be strong. The next function balances that responsiveness with stability.

## 5. Blended recovery forecasting

The function `generate_blended_recovery_forecast_wide` combines the recursive lag-one forecast with the seasonal recovery forecast.

```python
generate_blended_recovery_forecast_wide(
    historical_actual_wide,
    cutoff_label,
    forecast_months,
    required_destinations=None,
    date_column=FINAL_FORECAST_DATE_COLUMN,
    recovery_weight=0.4,
)
```

The default recovery weight is 0.4. This gives 60 percent weight to the stable lag-one forecast and 40 percent to seasonal recovery.

```text
blended forecast =
    1 minus recovery weight, multiplied by the lag-one forecast,
    plus recovery weight, multiplied by the seasonal recovery forecast
```

A weight of zero returns lag-one values. A weight of one returns seasonal recovery values. Intermediate values trade stability for seasonal responsiveness. The default is fixed rather than tuned by destination.

The weight must be between zero and one, or the function raises `ValueError`. The output preserves the date values and column order, while `NaN` in either component normally propagates to the blend.

So far, the candidate methods have been deterministic. We will now move to a statistical time-series model.

## 6. SARIMA forecasting

The SARIMA function is `generate_sarima_forecast_wide`.

```python
generate_sarima_forecast_wide(
    historical_actual_wide,
    cutoff_label,
    forecast_months,
    required_destinations=None,
    date_column=FINAL_FORECAST_DATE_COLUMN,
    order=(1, 1, 1),
    seasonal_order=(1, 1, 1, 12),
    min_history_months=36,
    max_growth_multiple=1.5,
    fallback_generator=generate_seasonal_recovery_forecast_wide,
    diagnostic_log=None,
)
```

This function fits one `statsmodels` SARIMAX model independently for each destination. Its defaults define a SARIMAX 1, 1, 1 model with a seasonal 1, 1, 1 component and a twelve-month cycle. The same fixed orders are used for every destination, avoiding a separate tuning search for each series.

Forecast months must be consecutive and begin immediately after the cutoff. The function removes later observations and generates a complete fallback forecast before fitting any model.

If a destination has fewer than 36 valid observations by default, it immediately uses the seasonal recovery fallback. Otherwise, the series begins at its first valid observation and is converted to monthly frequency, leaving internal missing months visible. The function then fits SARIMA and forecasts the exact horizon.

Stationarity and invertibility enforcement are disabled to support a wider range of series. A fitting error or any nonfinite prediction causes that destination to use fallback values.

Successful model forecasts are floored at zero and capped at 1.5 times the cutoff-safe historical maximum by default. If a dictionary is supplied as `diagnostic_log`, `fallback_destinations` records which destinations used fallback. A failure for one destination does not stop the others.

Next, we will look at a model that handles trend and seasonality differently.

## 7. Prophet forecasting

The Prophet function is `generate_prophet_forecast_wide`.

```python
generate_prophet_forecast_wide(
    historical_actual_wide,
    cutoff_label,
    forecast_months,
    required_destinations=None,
    date_column=FINAL_FORECAST_DATE_COLUMN,
    min_history_months=36,
    max_growth_multiple=1.5,
    fallback_generator=generate_seasonal_recovery_forecast_wide,
    diagnostic_log=None,
)
```

This function also fits one model per destination. Prophet represents trend changes together with annual seasonality.

Its fixed configuration enables yearly seasonality, disables weekly and daily seasonality, and uses multiplicative seasonality so seasonal effects scale with the forecast level. Setting `mcmc_samples` to zero uses MAP estimation, improving repeatability and runtime.

The months must again be consecutive and immediately follow the cutoff. For each destination, missing history is removed. If fewer than 36 valid observations remain, the fallback is used. Otherwise, monthly periods become Prophet timestamps, dates map to `ds`, values map to `y`, and the point forecast is read from `yhat`.

A fitting error, prediction error, or nonfinite value activates the fallback for that destination. Successful forecasts are floored at zero and capped at 1.5 times the historical maximum by default. Fallback destinations can be captured in the diagnostic log.

No COVID-specific regressor or manual intervention is included. Prophet must represent structural change through its standard trend mechanism.

This brings us to the final candidate: a pretrained time-series foundation model.

## 8. TimesFM forecasting

The final function is `generate_timesfm_forecast_wide`.

```python
generate_timesfm_forecast_wide(
    historical_actual_wide,
    cutoff_label,
    forecast_months,
    required_destinations=None,
    date_column=FINAL_FORECAST_DATE_COLUMN,
    max_growth_multiple=1.5,
    fallback_generator=generate_blended_recovery_forecast_wide,
    diagnostic_log=None,
)
```

This function uses Google's pretrained TimesFM 2.5, 200-million-parameter PyTorch model, with the pinned checkpoint:

```text
google/timesfm-2.5-200m-pytorch
```

TimesFM runs in zero-shot mode. It is not fitted or fine-tuned on this dataset. Instead, each destination's cutoff-safe history is supplied directly as context. The model is loaded and compiled once per runtime and then reused.

The configuration supports up to 512 context observations and a twelve-month forecast horizon. Input normalization, continuous quantile output, flip invariance, positive-series inference, and quantile-crossing correction are enabled. The per-core batch size is 20, and float32 matrix multiplication precision is set to high.

The cache location comes from `TIMESFM_CACHE_DIR`, with `.cache/timesfm` as the default.

Forecast months must be consecutive and immediately follow the cutoff. After cutoff filtering, the function prepares blended recovery fallback values. Missing history is removed, valid histories become `float32`, and all eligible destinations are forecast together. A destination with no history uses fallback immediately.

Every prediction is checked for finite values. Valid outputs are floored at zero and capped at 1.5 times the historical maximum by default. If model loading or batch prediction fails, all eligible destinations use their fallback forecasts.

The diagnostic log can record the sorted fallback destinations, pinned model ID, and any exception text. The intended horizon is no more than twelve months. The first call may be slower if the checkpoint must be downloaded, while later calls reuse the cached model.

## End-to-end workflow

We have now covered all eight functions. Let us bring them together into one workflow.

First, generate cutoff-safe forecasts with the naive function and the candidate models.

Second, validate each forecast with `validate_forecast_actual_wide` to confirm the correct months, destinations, structure, and values.

Third, after confirming alignment readiness, evaluate the forecasts with `evaluate_forecast_wide`.

Fourth, compare destination-level and aggregate MASE and MAPE results across the candidate methods.

Finally, run the validator in forecast-only mode before exporting the selected forecast. This final audit ensures that the submission has the exact required shape and no invalid values.

## Closing summary

To summarize each function in one question:

`generate_naive_forecast_wide` asks: what would the forecast be if earlier values repeated at a chosen lag?

`validate_forecast_actual_wide` asks: are these tables complete, clean, and safe to align?

`evaluate_forecast_wide` asks: after alignment, how accurate is the forecast for each destination and overall?

`generate_seasonal_recovery_forecast_wide` asks: what if last year's seasonal pattern repeats at the current recovery level?

`generate_blended_recovery_forecast_wide` asks: what if we combine a stable latest-level forecast with the seasonal recovery pattern?

`generate_sarima_forecast_wide` asks: what does a fixed statistical model of trend changes, autocorrelation, and yearly seasonality predict?

`generate_prophet_forecast_wide` asks: what does a trend-and-seasonality model with changepoints predict?

And `generate_timesfm_forecast_wide` asks: what does a pretrained foundation model predict from each destination's history without local training?

Together, these functions provide a consistent workflow for forecast generation, quality checking, accuracy evaluation, model comparison, and final export.

That concludes this developer guide.
