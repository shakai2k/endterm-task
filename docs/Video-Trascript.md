# SIT742 Assignment 2 Group Video Script

## 0:00-1:00 - Introduction and forecasting objective

Hello. We are **Group 3 - Team 16**, and this is our SIT742 Assignment 2 forecasting project, represented by:

1. Swaminathan Babu Rao alias Swami
2. Biswadeep DasGupta alias Biswadeep
3. Rohit Jaiswal alias Rohit 

Our task is to forecast monthly Chinese outbound tourism demand for 20 destinations. We use the public TULIP Lab `ISF-TDF2023` dataset. The public history is available through July 2023, and our final forecast covers the 12 months from August 2023 to July 2024.

Our workflow has three main goals: produce accurate destination-level forecasts, prevent future-data leakage, and make the complete result reproducible from the submitted notebook. We compare simple baselines with recovery-aware statistical and foundation-model approaches, evaluate them using MASE and MAPE, select one final method, and export the final all-destination forecast directly from the notebook.

[SCREEN: Show the notebook title, group details, assignment overview, and dataset acknowledgement.]

## 1:00-5:00 - Data, cutoffs, and Q1-Q3 functions

Use these as simple talking points. Do not read every point word for word. Keep one main idea in each sentence and use one short example per question.

### Data foundation

- The data has 415 monthly rows and 21 columns.
- `Date` is the first column. The other 20 columns are destinations.
- This is a **wide table**. Each row is one month, and each destination has its own column.
- `Date` starts as text, such as `2023M07`.
- The code converts `Date` to a monthly pandas period, called `Period[M]`.
- This makes month sorting and subtraction safe across different years.
- Destination columns contain numeric monthly demand counts. They are not percentages.
- Some columns use floating-point values because they contain missing values.
- Demand scale is different across markets. Some are in thousands; others reach hundreds of thousands or millions.
- Missing values stay as `NaN`. They are not changed to zero because missing data does not mean zero demand.
- Every destination column is a separate forecast target.
- The dates are used to create the training, validation, and final forecast periods.

### Critical columns to mention

| Column or column group | Typical type | Why it is important |
| --- | --- | --- |
| `Date` | Text, then `Period[M]` | Controls sorting, cutoffs, validation months, forecast months, and alignment. It must be first in the final CSV. |
| `Australia` | Numeric | Required market for Q6 and used for market-level analysis. |
| `Japan` | Numeric | Required market for Q6 and used for market-level analysis. |
| `New Zealand` | Numeric | Extra market chosen because its pattern is similar to Australia. |
| `Taiwan China` | Numeric | Extra market chosen because its recovery pattern differs from Japan. |
| Other 16 destinations | Numeric | All are required in the validation tables and final forecast. |
| `_month_period` | Monthly period | Helper used for date filtering. It must not appear in the final CSV. |
| `destination` | Text | Market name in the long-format table. |
| `demand` | Numeric | Demand value in the long-format table. |

- **Emphasize:** the final CSV contains only `Date` and the exact 20 destination columns.
- Do not export helper columns, model labels, diagnostics, actual values, intervals, or an index column.
- The long table has three main columns: `Date`, `destination`, and `demand`.
- No external features are used. The forecasts use only past demand values.
- Validation uses demand observations only through `2023M02` and forecasts `2023M03` to `2023M07`.
- Final forecasting uses public history only through `2023M07` and forecasts `2023M08` to `2024M07`.
- The cutoff is applied before forecasting, so later actual values cannot leak into model inputs.

### NaN and null-value handling

- **Emphasize:** pandas represents empty numeric cells as `NaN`. The workflow treats `NaN`, `None`, and `pd.NA` as missing data; it never assumes that a missing observation means zero demand.
- We do not apply blanket zero-filling, interpolation, or historical-value imputation. The original missing values remain visible in the wide history table and in the EDA missingness summary.
- The missingness summary records both the number of missing months and the first observed month for every destination. Australia and Hawaii have complete histories, while Chile has the largest missing count at 291 months.
- Most missing values occur before a destination began reporting. However, Chile's first observation is `2012M01`; only 276 months precede that date, so its total indicates 15 additional missing observations after reporting began. We therefore avoid claiming that every missing value is only a pre-start value.
- Q1 returns `NaN` when a required lag source is unavailable rather than inventing a value. A later recursive forecast can remain `NaN` if it depends on that missing source.
- Q2 counts missing, nonnumeric, and nonfinite values separately. These checks make a forecast invalid instead of silently converting a bad value to zero.
- Q3 converts missing, nonnumeric, and infinite inputs to `NaN` for evaluation, excludes invalid forecast-actual pairs, excludes zero actuals from MAPE, and uses only valid exact-lag pairs for the MASE denominator. When no valid calculation is possible, it reports `NaN` with an availability or warning flag.
- The modelling workflow uses available observations with `dropna()` and deterministic fallbacks where needed. This removes unavailable observations from a calculation; it is not value imputation.
- One limitation is that TimesFM receives each destination after `dropna()`. If a series has an internal gap, such as Chile, this compresses the time sequence rather than explicitly representing the missing calendar month. The selected blended-recovery model does not rely on TimesFM for the final submission.
- **Emphasize the final result:** the final audit reports zero missing, nonnumeric, and nonfinite forecast values. The exported CSV has 12 data rows, `Date` plus 20 destination columns, and no blank, `NaN`, null, or infinite forecast cells.

[SCREEN: Show `missingness_summary`, briefly point to Chile and the two complete series, then show the Q2 value-count fields and the final zero-count audit fields.]

### Q1 - Naive-lag forecast generation

- **Emphasize the implementation:** the function first converts month labels to monthly periods and removes every row after the cutoff.
- It creates a month-to-value lookup for all required destinations.
- It generates future months in chronological order so an earlier forecast can be reused by a later forecast.
- For lag 1, the first future month uses the last actual value. Later months reuse generated values recursively.
- For lag 12, each target month uses the same calendar month from the previous year.
- If a source month is unavailable, the function returns `NaN` instead of creating an unsupported value.
- The final step restores the requested month order and returns only `Date` plus the destination columns.
- **Example to explain:** with a February cutoff and value 100, lag-1 forecasts for March, April, and May are all 100 because April uses the generated March value and May uses generated April.
- **Emphasize the result:** both lag-1 and lag-12 toy checks pass.
- The real validation run produces five forecast months for all 20 destinations for both baselines.
- Lag-1 gives a stable flat forecast across the recursive horizon, while lag-12 keeps the previous year's seasonal pattern.

### Q2 - Forecast and actual validation

- **Emphasize the implementation:** the function first defines the required month and destination scope.
- It checks forecast months, row counts, duplicate dates, missing destinations, and extra columns.
- It scans required values and counts missing, nonnumeric, and nonfinite cells separately.
- When actual data is supplied, it keeps only the required evaluation months before checking alignment.
- Structural checks are summarized by `can_align`.
- Structure and value-quality checks are combined into the stricter `is_valid` result.
- The function always returns the same 20 audit fields, which makes every audit easy to compare.
- In forecast-only mode, actual-related fields and `can_align` remain `None` because no actual table is available.
- **Emphasize the result:** both baseline validation audits report `can_align=True` and pass the required checks.
- The final submission audit reports `is_valid=True` and `can_align=None`.
- It confirms 12 rows, 20 destinations, no duplicate dates, no missing values, and no nonnumeric or infinite forecasts.

### Q3 - Forecast accuracy evaluation

- **Emphasize the implementation:** the function converts dates to monthly periods and keeps only the evaluation window.
- It rejects duplicate evaluation dates and joins forecast and actual rows one-to-one by month.
- It converts invalid numeric values to `NaN` and uses only valid forecast-actual pairs.
- MAE is calculated first from the absolute errors.
- For MASE, the training history is sorted and paired by the exact calendar lag for each destination.
- The official denominator uses lag 1, including when the forecast being evaluated is lag 12.
- MAPE uses valid rows with nonzero actual values, so the code never divides by zero.
- The function records pair counts and denominator warnings before creating unweighted aggregate results.
- **Emphasize the result:** lag-1 achieved mean MASE `2.67`, median MASE `1.70`, and mean MAPE `48.1%`.
- Lag-12 was weaker, with mean MASE `4.41`, median MASE `3.83`, and mean MAPE `77.9%`.
- These tables connect the Q1 forecasts, Q2 audits, and Q3 measures in one visible workflow.

[SCREEN: First show `raw_tourism_data.head()`, its shape, and its dtypes or schema. Then show `tourism_series_long.head()`, the Q1 toy checks, a successful Q2 audit, and both baseline evidence tables. Pause on important fields rather than reading every field.]

## 5:00-7:00 - Exploratory analysis and modelling implications

Use these as prompts and connect each observation to a modelling decision. Keep the section close to two minutes, prioritising the points marked **emphasize** and allowing time to pause on the most useful figures.

### EDA purpose and coverage

- **Emphasize:** EDA was used to understand whether the 20 destinations could reasonably be forecast using one common set of model assumptions.
- The public table contains monthly demand histories with different starting dates and numbers of valid observations.
- Missing periods are retained as missing rather than interpreted as zero demand. Most are before reporting began, while Chile also has 15 missing observations after its first reported month.
- Summary statistics and coefficients of variation show large differences in scale and volatility across destinations. This is why raw MAE alone is not suitable for comparing markets.

### Univariate findings

- **Emphasize:** the time-series plots show trend, annual seasonality, and a major COVID-19 structural break followed by uneven recovery.
- Several distributions are strongly skewed, especially where long low-demand periods are followed by rapid reopening.
- The monthly seasonal view shows that demand changes by calendar month, supporting the inclusion of annual seasonal models.
- The ADF tests indicate that none of the 20 level series is stationary at the five-percent significance level. This suggests that models must account for changing levels or differences rather than assuming a constant mean.
- Month-on-month change flags highlight unusually sharp movements. These observations motivate robust caps and safeguards instead of allowing a single recovery jump to dominate a forecast.

### Multivariate findings

- Most destination correlations are positive because markets experienced common tourism shocks; correlation is used as supporting interpretation, not as proof of causation.
- New Zealand has the strongest reported correlation with Australia, approximately 0.95, which supports using it as a related comparison market in Q6.
- Taiwan China provides a contrasting case for Japan, with a weaker reported correlation of approximately 0.56 and a different recovery path.
- The final models still forecast each destination separately, so no destination's future value is copied from another market.

### EDA-to-model decisions

- **Emphasize:** lag-1 is retained as a stable latest-level benchmark.
- **Emphasize:** lag-12 and seasonal models are retained because annual patterns are visible.
- Recovery-aware models are added because pre-COVID or previous-year levels alone do not represent the reopening period well.
- Non-negative floors, clipped recovery ratios, historical growth caps, and deterministic fallbacks are used to control unstable forecasts.
- Destination-level metrics and plots remain necessary because an aggregate winner can still perform poorly for an individual market.

[SCREEN: Show the all-destination history plot first, then one seasonal figure and one concise statistics or ADF table. Avoid spending time reading every destination row.]

## 7:00-10:30 - Candidate models

Use these points to compare the models in your own words. Focus on the forecast logic and the measured result. Do not spend time reading function parameters.

### Common comparison design

- **Emphasize:** all eight candidates use the same cutoff-safe training table, validation months, 20 destinations, wide output schema, and evaluation functions.
- Every candidate is evaluated with MASE using lag-1 scaling and with MAPE using zero-actual handling.
- Fixed settings are applied consistently across destinations to keep the comparison reproducible and avoid hidden destination-specific tuning.

### Model 1 - Naive lag-1

- **Implementation:** carries the latest available monthly level forward recursively.
- **Strength:** simple, deterministic, and stable during an uncertain recovery period.
- **Limitation:** produces a flat 12-month forecast and cannot represent annual seasonality.
- **Result:** mean MASE `2.67`, median MASE `1.70`, and mean MAPE `48.1%`.

### Model 2 - Seasonal naive lag-12

- **Implementation:** repeats demand from the same month in the previous year.
- **Strength:** preserves a transparent annual seasonal pattern.
- **Limitation:** assumes the previous year's level remains relevant, so it can under-forecast during rapid recovery.
- **Result:** mean MASE `4.41`, median MASE `3.83`, and mean MAPE `77.9%`. It was the weakest baseline overall.

### Model 3 - Seasonal recovery naive

- **Implementation:** begins with the lag-12 seasonal value and multiplies it by a recent recovery ratio.
- The ratio compares the latest six valid months with the same calendar months one year earlier.
- The recovery effect is strongest near the cutoff and decays toward ordinary seasonal naive over 12 months.
- **Strength:** combines annual shape with changing recovery levels.
- **Limitation:** a very small year-ago base can create an extreme ratio, so ratios are clipped and forecasts are growth-capped.
- **Result:** mean MASE `2.70`, median MASE `1.62`, and mean MAPE `45.4%`.

### Model 4 - Blended recovery

- **Implementation:** combines 60 percent lag-1 with 40 percent seasonal recovery.
- The lag-1 component anchors the latest level; the recovery component adds seasonal and reopening movement.
- **Emphasize:** `BLENDED_RECOVERY_WEIGHT = 0.4` is a fixed, validation-supported stability trade-off.
- **Strength:** more stable than full seasonal recovery while retaining a non-flat forecast shape.
- **Limitation:** one common weight may not be optimal for every destination.
- **Result:** it ranked first on the primary measures with mean MASE `2.49` and median MASE `1.53`; mean MAPE was `43.0%`.
- The weight was not obtained from an exhaustive continuous optimisation search.

### Model 5 - SARIMA

- **Why included:** SARIMA is a standard statistical benchmark for monthly data. It tests whether modelling autocorrelation, differencing, and yearly seasonality improves on the simpler recovery rules.
- **Implementation:** fits `SARIMAX(1,1,1)(1,1,1,12)` independently to each destination.
- Non-seasonal and seasonal differencing address changing levels and annual structure.
- **Strength:** established statistical model with explicit short-run and seasonal dynamics.
- **Limitation:** fixed orders may not fit every destination, and a COVID-scale structural break can weaken historical relationships.
- **Result:** mean MASE `2.98`, median MASE `2.78`, and mean MAPE `54.9%`.

### Model 6 - Prophet

- **Why included:** Prophet provides a different statistical approach. It tests whether flexible trend changes and yearly seasonality can handle the uneven recovery better than fixed seasonal relationships.
- **Implementation:** applies `log1p` to demand, allows candidate changepoints through 98 percent of the cutoff-safe history, and uses a fixed five-term yearly Fourier seasonality on the log scale. Daily and weekly seasonalities are disabled for monthly data.
- The extended changepoint range lets the COVID collapse and reopening influence the fitted trend instead of excluding the latest 20 percent of each history from candidate changepoints.
- **Strength:** flexible trend and changepoint representation with interpretable components.
- **Limitation:** even with the revised fixed settings, one common trend and seasonal specification cannot represent every destination's recovery path.
- **Result:** mean MASE `4.18`, median MASE `2.63`, and mean MAPE `74.7%`.

### Model 7 - TimesFM 2.5 zero-shot

- **Why included:** TimesFM provides a modern pretrained comparison. It tests whether patterns learned from many external time series can improve these forecasts without fitting or tuning a separate local model.
- **Implementation:** uses each destination's cutoff-safe history as context for a pretrained time-series foundation model.
- No local fitting or fine-tuning is performed, and all eligible destinations are forecast in a batch.
- **Strength:** provides a modern model comparison and performed strongly in the longer backtest.
- **Limitation:** requires an external checkpoint and more computation, and official validation was slightly weaker than the selected blend.
- **Result:** mean MASE `2.60`, median MASE `2.00`, and mean MAPE `43.8%`. It ranked second on official mean MASE.

### Model 8 - Holt-Winters recovery ensemble

- **Why included:** Holt-Winters is a transparent classical method for monthly level, trend, and seasonality. Blending it with the existing recovery model tests whether learned seasonal smoothing adds value without losing the post-COVID anchor.
- **Implementation:** fits damped additive trend and additive 12-month seasonality independently by destination, then combines 30 percent Holt-Winters with 70 percent blended recovery. This is equivalent to 30 percent Holt-Winters, 42 percent lag-1, and 28 percent seasonal recovery.
- **Strength:** achieved the best official mean MAPE at `42.8%` and remained close to the selected model on mean MASE.
- **Limitation:** mean MASE `2.53` and median MASE `2.01` were still worse than blended recovery. Chile, Maldives, and Thailand contain internal gaps and therefore used the documented blended-recovery fallback for the Holt-Winters component.
- **Result:** mean MASE `2.53`, median MASE `2.01`, and mean MAPE `42.8%`. It is a strong challenger but was not selected because MASE is the primary assignment measure.

### Shared stability and fallback rules

- SARIMA and Prophet require sufficient history and fall back to seasonal recovery after fitting errors or nonfinite predictions.
- TimesFM and Holt-Winters fall back to blended recovery if loading, fitting, or prediction fails.
- Successful fitted forecasts are floored at zero and capped relative to the destination's cutoff-safe historical maximum.
- Prophet and Holt-Winters record fallback diagnostics and fixed settings. Prophet required no fallback; the Holt-Winters component used its deterministic fallback for the three internally incomplete series.

[SCREEN: Show one model-comparison diagram or the eight model headings. Pause on both blend formulas and the shared fallback safeguards, then move to the measured results rather than opening every function.]

## 10:30-13:30 - Validation results and model selection

Use this section to explain what the results mean, where each model is useful, and why the final model was selected. Do not only read the ranking table.

### Validation design

- The official validation period is `2023M03` to `2023M07`.
- Every model uses history only through `2023M02`.
- Every model forecasts the same five months and the same 20 destinations.
- Lower MASE and MAPE are better.
- Mean MASE is the main comparison measure. Median MASE shows whether a few difficult markets are affecting the mean.
- MAPE gives a percentage-based second view of error.

### Official validation comparison

| Model | Mean MASE | Median MASE | Mean MAPE | Where the model fits |
| --- | ---: | ---: | ---: | --- |
| Naive lag-1 | 2.67 | 1.70 | 48.1% | Useful when the latest level is the safest assumption and a flat forecast is acceptable. |
| Naive lag-12 | 4.41 | 3.83 | 77.9% | Useful when the yearly pattern is stable and the demand level has not changed greatly. |
| Seasonal recovery | 2.70 | 1.62 | 45.4% | Useful when annual seasonality remains important but the market is recovering to a new level. |
| Blended recovery | **2.49** | **1.53** | 43.0% | Useful when both stability and recovery movement are needed. This was the selected model. |
| SARIMA | 2.98 | 2.78 | 54.9% | Useful when a series has enough history and reasonably stable autocorrelation and seasonal structure. |
| Prophet | 4.18 | 2.63 | 74.7% | Useful when recent trend changes and proportional seasonal movement are more important than fixed lag relationships. |
| TimesFM 2.5 | 2.60 | 2.00 | 43.8% | Useful as a quick zero-shot model when strong forecasts are needed without local fitting or manual order selection. |
| Holt-Winters recovery ensemble | 2.53 | 2.01 | **42.8%** | Useful when smoothed level, trend, seasonality, and recovery anchoring are all desired. |

### What the results show for each model

- **Lag-1:** this was a strong and stable baseline, but its flat forecast cannot show annual peaks and troughs.
- **Lag-12:** this was the weakest overall model because the previous year's depressed values did not represent the speed of recovery.
- **Seasonal recovery:** this improved MAPE compared with lag-1 and restored seasonal shape, but full recovery scaling was less stable for fast-reopening destinations.
- **Blended recovery:** this produced the lowest mean MASE and lowest median MASE in official validation.
- **SARIMA:** this captured statistical and seasonal structure, but the fixed order did not handle the COVID disruption as well as the recovery-aware blend.
- **Prophet:** extending changepoints toward the cutoff and fitting log demand improved stability and percentage error, but it remained weak in aggregate relative to the recovery-aware blend.
- **TimesFM:** this was very close to the selected model and clearly outperformed several traditional candidates.
- **Holt-Winters ensemble:** this produced the lowest mean MAPE and competitive mean MASE, but its median and mean MASE did not beat blended recovery.

### Why blended recovery was selected

- **Best primary results:** it ranked first on mean MASE and median MASE. The Holt-Winters ensemble led MAPE by only about 0.4 percentage points.
- **Balanced logic:** 60 percent lag-1 controls unstable recovery values, while 40 percent seasonal recovery adds movement and annual shape.
- **Stable output:** it produced finite, non-negative, destination-specific forecasts for all 20 markets.
- **Simple reproduction:** it is deterministic and does not depend on an optimiser, random seed, or downloaded model during final generation.
- **Longer-horizon support:** its mean MASE was `2.19` in the 12-month recovery-era backtest, compared with `2.24` for lag-1.
- **Limitation:** one fixed weight is not best for every destination, and the 0.4 weight was not selected through an exhaustive search.

### Why TimesFM was still useful

- TimesFM ranked second in official validation with mean MASE `2.60` and mean MAPE `43.8%`.
- It achieved the best 12-month backtest mean MASE of `2.06`.
- It is a newer foundation-model approach and provides evidence that pretrained time-series models can be competitive on this tourism dataset.
- It is simple from a modelling point of view: we pass each destination's history to the pretrained model and request the forecast horizon.
- It does not require local training, SARIMA order selection, Prophet configuration tuning, or a separate fitted model for every destination.
- It can forecast all eligible destination histories together in one batch.
- This makes it useful for quick benchmarking and for future work with more rolling validation windows.
- However, simple model use does not mean zero operational cost. TimesFM still needs the external checkpoint, compatible packages, more memory, and more computation than blended recovery.
- We did not select it because blended recovery was better on both official MASE summaries and was easier to reproduce as the final submission model.

### Supporting backtest and caution

- The additional backtest uses a July 2022 cutoff and forecasts the next 12 months.
- TimesFM ranked first with mean MASE `2.06`.
- The Holt-Winters ensemble ranked second with mean MASE `2.18`, followed closely by blended recovery at `2.19`.
- The backtest overlaps part of the official validation period, so it is supporting evidence rather than a fully independent test.
- The official validation period remains the main basis for final model selection.

[SCREEN: Display `validation_model_comparison`, then `backtest_summary`, and finally `model_summary`. Pause on the blended-recovery, Holt-Winters ensemble, and TimesFM rows. Highlight the single `selected=True` value and explain the MASE-primary decision in one clear sentence.]

## 13:30-16:00 - Final forecast and selected markets

The final model is trained using public history through July 2023. It produces 12 rows from August 2023 to July 2024 for all 20 destinations.

For Australia, the blend's validation MASE is approximately 1.04, compared with 1.67 for lag-1 and 4.07 for lag-12. The final forecast ranges from about 63 thousand to 106 thousand and ends near 93 thousand. It retains seasonal movement but remains below the historical peak. The main uncertainty is the speed of recovery relative to the recent six-month pattern.

For Japan, the blend improves on both naive baselines but still has a relatively high validation MASE of about 3.41. Its final forecast rises from approximately 237 thousand to 407 thousand. The revised Prophet remains weaker for Japan, showing that flexible changepoints alone do not capture every destination's recovery path.

We selected New Zealand because its historical pattern is strongly correlated with Australia. Its final forecast ranges from roughly 15 thousand to 28 thousand. The unblended recovery model performs better for this market, so the selected common blend may understate some recovery months.

We selected Taiwan China as a contrasting market with weaker correlation to Japan and a slower recovery. Lag-1 and SARIMA outperform the blend for this destination. The final blend ranges from approximately 24 thousand to 40 thousand, illustrating the risk that a common recovery adjustment can overcorrect in a slow-recovery market.

These cases show why we report destination-level evidence as well as aggregate averages. The selected blend is strongest overall, but uncertainty and the best-performing model differ across markets.

[SCREEN: Show `forecast_submission_wide`, followed by the four market validation tables and plots. Do not scroll too quickly through all 20 columns.]

## 16:00-18:00 - Reproducibility and CSV demonstration

We now demonstrate how the submitted CSV is regenerated.

The final cell reads the selected model from `model_summary`; it does not hard-code a different model during export. It generates `forecast_submission_wide` using only history through July 2023 and then runs `validate_forecast_actual_wide` in forecast-only mode.

[SCREEN: Restart or use a clean kernel where practical. Run the final-generation cell. Zoom in on the following output.]

The audit reports `is_valid=True` and `can_align=None`. It confirms 12 rows, all required months, all 20 destinations, no duplicate dates, no missing values, and numeric finite forecasts.

The export cell writes this exact audited object with `index=False` to:

`SIT742-2026T2-A2-<ConfirmedGroupID>-Forecast.csv`

[SCREEN: Run the export cell, show the printed filename, and open the generated CSV briefly. Confirm that its first column is `Date`, it has no index column, and its first and last months are `2023M08` and `2024M07`.]

No forecast values are manually edited after export. The repository records package requirements, and the workflow uses relative paths. The deterministic selected model gives the same forecasts when rerun with the same input data and parameters.

## 18:00-19:00 - Limitations and possible improvements

Our main limitation is the structural uncertainty of forecasting during tourism recovery. The official validation window contains only five months, and performance varies considerably by destination. MAPE can also become unstable for small actual values, while MASE depends on a reliable historical denominator.

Historical missingness is another limitation. We preserve missing values and exclude invalid pairs rather than treating them as zero, but Chile contains internal gaps after reporting began. In addition, TimesFM's `dropna()` input preparation compresses internal gaps. A future version could reindex every destination to a complete monthly calendar and use a model-specific missing-data strategy or an explicit mask.

The fixed 0.4 recovery weight and 0.3 Holt-Winters ensemble weight were not selected through exhaustive optimisation. Future work could use several rolling-origin validation windows to tune recovery weights and caps without relying heavily on one short period. We could also use regularised destination-specific weights, allowing markets such as Japan or Taiwan China to use different model combinations while limiting overfitting.

Additional improvements could include forecast intervals or recovery scenarios, formal residual diagnostics, and sensitivity analysis for the recovery window, decay horizon, and growth cap. These additions would improve uncertainty communication while keeping the submitted point-forecast CSV in the required schema.

## 19:00-19:30 - Contributions and collaboration

Our work was completed collaboratively through shared review of the notebook, forecast outputs, and written interpretation. We used a common workflow and checked that every model followed the same cutoffs, destination coverage, and evaluation measures.

Each participating member should briefly state one verified contribution, such as Q1-Q3 implementation, EDA, candidate-model development, validation analysis, notebook integration, market interpretation, reproducibility checks, final forecast review, or video coordination.

We reviewed the final notebook and submission files together. **<Briefly describe the actual collaboration method, such as meetings, Git branches, peer review, or division of notebook sections.>**

[SCREEN: Show the completed `GROUP_INFO` contribution fields or a concise contribution slide.]

## 19:30-20:00 - Closing

### Points to emphasize

- **Complete task:** we forecast 12 months of tourism demand for all 20 destinations.
- **No leakage:** validation and final forecasts use strict February and July 2023 cutoffs.
- **Reliable workflow:** Q1 generates forecasts, Q2 validates the tables, and Q3 measures accuracy.
- **Fair comparison:** all eight models use the same months, destinations, and MASE/MAPE evaluation rules.
- **Final choice:** blended recovery achieved the best official mean and median MASE; the Holt-Winters ensemble achieved the best MAPE but did not win the primary MASE comparison.
- **Modern comparison:** TimesFM ranked close to the selected model and achieved the best longer backtest result, showing that simple zero-shot use can be valuable.
- **Submission readiness:** the final audit passed with 12 rows, 20 destinations, and numeric finite values.
- **Reproducibility:** the final CSV is generated directly from `forecast_submission_wide` with no manual forecast editing.
- **Honest limitation:** the validation window is short, and future work should use more rolling validation and destination-specific model combinations.

### Suggested final statement

In summary, we created a cutoff-safe and reproducible forecasting workflow for all 20 destinations. We used Q1 to generate the required baselines, Q2 to check table quality, and Q3 to compare forecast accuracy. Among eight candidate models, blended recovery gave the best official mean and median MASE while remaining stable and easy to reproduce. The new Holt-Winters recovery ensemble achieved the best MAPE, and TimesFM remained strongest in the longer backtest. Because MASE is the primary assignment measure, blended recovery remains the final model. Finally, our audited 12-month forecast passed all schema and value checks and can be exported directly from the notebook. The main next step would be wider rolling validation and more destination-specific model selection. Thank you.

---

## Recording emphasis checklist

The following points should be visibly demonstrated or clearly emphasized during recording:

1. **All members participate.** Each person should appear or speak and explain a meaningful technical component.
2. **Problem and data.** State that the task forecasts Chinese outbound tourism demand for 20 destinations using the TULIP Lab `ISF-TDF2023` dataset.
3. **Cutoff safety.** Emphasize the February 2023 validation cutoff and July 2023 final cutoff. Make clear that later actuals are never used as forecast inputs.
4. **EDA-to-model connection.** Do not only describe plots; explain how seasonality, nonstationarity, COVID disruption, and uneven recovery motivated the candidate models and safeguards.
5. **Candidate-model comparison.** Give the basic idea, one strength, and one limitation for each model; avoid reading function implementations or every parameter.
6. **Required baselines.** Show both lag-1 and lag-12 results rather than discussing only advanced models.
7. **Both performance measures.** Explain why MASE and MAPE provide different views and mention their limitations.
8. **Model-selection evidence.** Pause on the comparison table and clearly state why the 0.4 blended-recovery model was selected.
9. **Weight limitation.** Do not describe either 0.4 recovery weighting or 0.3 Holt-Winters weighting as mathematically optimal; call them fixed, validation-supported trade-offs.
10. **Destination evidence.** Show at least Australia, Japan, New Zealand, and Taiwan China, including one limitation or risk for each.
11. **Final schema.** Show that `forecast_submission_wide` contains 12 months and exactly 20 destination columns plus `Date`.
12. **Audit result.** Zoom in on `is_valid=True` and `can_align=None` before export.
13. **CSV regeneration.** Run the export cell, identify the exact submitted CSV, and state that it comes directly from `forecast_submission_wide` with `index=False`.
14. **Correct naming.** Use `SIT742`, not `SIG742`, and replace `<ConfirmedGroupID>` everywhere before recording.
15. **Reproducibility and stability.** Mention deterministic final forecasts, relative paths, documented dependencies, and model fallbacks.
16. **Honest limitations.** Mention the short validation window, overlapping supporting backtest, destination heterogeneity, and lack of exhaustive blend-weight tuning.
17. **Contributions.** Replace all contribution placeholders with accurate, specific statements that match the submitted work.
