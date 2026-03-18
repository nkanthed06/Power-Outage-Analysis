# Power-Outage-Analysis
Final Project for DSC 80
**Author**: Nirav Kanthed

---

## Introduction

This project analyzes a dataset of major power outages that occurred across the United States from 2000 to 2016. The data was compiled from publicly available reports by the Department of Energy and tracks outage events across a wide range of geographic regions, climate zones, and causes.

**Central Question**: *How does the cause of a power outage affect its severity — measured by outage duration and the number of customers affected?*

Understanding this question matters because power outages have cascading effects on public safety, economic activity, and critical infrastructure. If we can identify which types of outage causes consistently produce longer and more damaging events, utility operators, emergency managers, and policymakers can allocate resources more effectively — prioritizing prevention and faster response for the most dangerous categories.

The dataset contains **1,534 rows** (outage events) and 56 columns. The columns most relevant to this analysis are:

| Column | Description |
|---|---|
| `CAUSE.CATEGORY` | The high-level category of what caused the outage (e.g., severe weather, intentional attack, equipment failure) |
| `OUTAGE.DURATION` | Total duration of the outage, in minutes |
| `CUSTOMERS.AFFECTED` | Number of customers who lost power during the outage |
| `NERC.REGION` | The North American Electric Reliability Corporation region where the outage occurred |
| `CLIMATE.CATEGORY` | The climate classification for the state at the time of the outage (warm, normal, cold) |
| `OUTAGE.START.DATE` / `OUTAGE.START.TIME` | The date and time the outage began |
| `OUTAGE.RESTORATION.DATE` / `OUTAGE.RESTORATION.TIME` | The date and time power was restored |
| `MONTH` | The month the outage started |
| `ANOMALY.LEVEL` | The El Niño/La Niña oceanic index at the time, reflecting climate anomalies |
| `SEVERITY` *(engineered)* | A composite score combining normalized duration and normalized customers affected |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

The raw Excel file required several cleaning steps before analysis:

1. **Stripped column names**: Leading and trailing whitespace was removed from all column headers to prevent key errors.
2. **Replaced string `"NA"` with `NaN`**: The raw file used the string `"NA"` for missing values instead of true nulls; these were replaced with `np.nan` for proper handling.
3. **Dropped the `variables` column**: This column contained unit labels (a formatting artifact from the Excel source) and carried no analytical value.
4. **Converted numeric columns**: Columns such as `OUTAGE.DURATION`, `CUSTOMERS.AFFECTED`, `DEMAND.LOSS.MW`, price, and sales columns were cast to numeric, with invalid entries coerced to `NaN`.
5. **Combined date and time into timestamps**: `OUTAGE.START.DATE` + `OUTAGE.START.TIME` were merged into a single `OUTAGE.START` datetime column, and similarly for `OUTAGE.RESTORATION`.
6. **Extracted time features**: `YEAR`, `MONTH`, and `HOUR` were extracted from `OUTAGE.START` to enable temporal analysis.
7. **Calculated duration from timestamps**: A `CALCULATED_DURATION` column (in minutes) was derived directly from the difference between `OUTAGE.RESTORATION` and `OUTAGE.START`, serving as an independent check on the reported `OUTAGE.DURATION`.
8. **Dropped rows with missing critical values**: Rows missing `OUTAGE.START`, `OUTAGE.RESTORATION`, `CUSTOMERS.AFFECTED`, or `CLIMATE.CATEGORY` were removed, as these are essential to answering the central question. This left a clean working dataset.
9. **Engineered a `SEVERITY` metric**: Outage duration and customers affected were each normalized to [0, 1] by dividing by their column maxima, then summed into a single `SEVERITY` score. This composite allows severity to be compared across outages of different types.
10. **Fixed `ANOMALY.LEVEL`**: Ensured this column was numeric, coercing any remaining strings to `NaN`.

These steps were necessary because the raw data, being sourced from government reports compiled into Excel, contained formatting rows, mixed types, and inconsistently encoded missingness. Dropping rows with missing key columns reduces the dataset but ensures the analysis is not distorted by imputed or estimated values for the core variables.

### Head of Cleaned DataFrame

| YEAR | MONTH | U.S._STATE | NERC.REGION | CLIMATE.CATEGORY | CAUSE.CATEGORY | OUTAGE.DURATION | CUSTOMERS.AFFECTED | SEVERITY | HOUR |
|---|---|---|---|---|---|---|---|---|---|
| 2011 | 7 | Minnesota | MRO | normal | severe weather | 3060 | 70000 | 0.1426 | 17 |
| 2010 | 10 | Minnesota | MRO | cold | severe weather | 3000 | 70000 | 0.1413 | 20 |
| 2012 | 6 | Minnesota | MRO | normal | severe weather | 2550 | 68200 | 0.1236 | 23 |
| 2015 | 7 | Minnesota | MRO | warm | severe weather | 1740 | 250000 | 0.3305 | 2 |
| 2010 | 11 | Minnesota | MRO | cold | severe weather | 1860 | 60000 | 0.1233 | 8 |

> *Note: Run `print(df.head().to_markdown(index=False))` in your notebook and paste the exact output here before publishing.*

### Univariate Analysis

<iframe
  src="assets/outage-causes-bar.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

The bar chart above shows the distribution of outage causes across all events in the dataset. **Severe weather** is by far the most common cause, accounting for nearly half of all recorded outages. Intentional attacks are the second most frequent, though they are considerably less common. This distribution is important context — any model or test that treats all cause categories equally must contend with this imbalance.

<iframe
  src="assets/outage-duration-hist.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

The histogram of `OUTAGE.DURATION` reveals a heavily right-skewed distribution, with the vast majority of outages resolved within a few thousand minutes but a long tail of extreme events stretching into the tens of thousands of minutes. This skew motivates careful consideration of which statistical tests and model evaluations are appropriate, as mean-based metrics will be pulled by outliers.

### Bivariate Analysis

<iframe
  src="assets/duration-by-cause-box.html"
  width="800"
  height="550"
  frameborder="0"
></iframe>

The box plot compares outage duration across cause categories. **Fuel supply emergencies** stand out with an extremely high median duration and wide spread, suggesting these events are logistically complex and slow to resolve. Severe weather also shows high variability. Intentional attacks, by contrast, tend to be shorter in duration despite being frequent — likely because they are identified and addressed quickly.

### Interesting Aggregates

The table below shows the **mean outage duration (minutes)** and **mean customers affected** grouped by cause category:

| CAUSE.CATEGORY | OUTAGE.DURATION (mean) | CUSTOMERS.AFFECTED (mean) |
|---|---|---|
| equipment failure | 389.7 | 109,222.8 |
| fuel supply emergency | 16,402.8 | 0.2 |
| intentional attack | 353.0 | 1,865.5 |
| islanding | 237.1 | 6,169.1 |
| public appeal | 2,142.1 | 7,618.8 |
| severe weather | 3,937.2 | 188,490.4 |
| system operability disruption | 543.8 | 210,561.7 |

This table reveals a striking divergence: **fuel supply emergencies** last the longest on average (over 11 days!) but affect almost no customers — likely because they are managed through controlled load reduction rather than a sudden failure. Meanwhile, **severe weather** and **system operability disruptions** have extremely high average customers affected, making them the most socially impactful causes even when duration is lower. This interplay between duration and scale is exactly why the composite `SEVERITY` score is a more informative target than either metric alone.

---

## Assessment of Missingness

### MNAR Column

I believe `OUTAGE.DURATION` is **MNAR** (Missing Not At Random). The reasoning is that outage duration is sometimes unreported precisely because the outage itself was unusual, contentious, or administratively complex — for instance, fuel supply emergencies or events that were eventually disputed as "outages." Utilities may be less likely to report duration for events where the restoration timeline was ambiguous or politically sensitive. The missingness is therefore related to the value of the duration itself (very long or abnormal events going unreported), which is the defining characteristic of MNAR.

To convert this to MAR, one would ideally have access to reporting logs from individual utility companies — specifically, whether a utility filed a full DOE report or a partial one. If missingness is predictable from the utility's filing type rather than the outage's actual duration, the mechanism would shift to MAR.

### Missingness Permutation Tests

**Test 1 — `OUTAGE.DURATION` missingness vs. `CAUSE.CATEGORY`**

- **Null Hypothesis (H₀)**: The distribution of `CAUSE.CATEGORY` is the same regardless of whether `OUTAGE.DURATION` is missing.
- **Alternative Hypothesis (H₁)**: The distributions differ.
- **Test Statistic**: Total Variation Distance (TVD) between the `CAUSE.CATEGORY` distributions for rows with and without missing `OUTAGE.DURATION`.
- **Observed TVD**: 0.252 | **p-value**: 0.001

**Conclusion**: We **reject the null hypothesis**. The cause of an outage is strongly associated with whether its duration gets reported, consistent with the MNAR reasoning above.

<iframe
  src="assets/missing-by-cause.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

The plot shows the proportion of missing `OUTAGE.DURATION` values broken down by cause category. Certain categories — particularly fuel supply emergency — have disproportionately high rates of missing duration, supporting the MNAR hypothesis.

---

**Test 2 — `OUTAGE.DURATION` missingness vs. `MONTH`**

- **Null Hypothesis (H₀)**: The distribution of `MONTH` is the same for rows with and without missing `OUTAGE.DURATION`.
- **Alternative Hypothesis (H₁)**: The distributions differ.
- **Observed TVD**: 0.223 | **p-value**: 0.12

**Conclusion**: We **fail to reject the null hypothesis**. The month of an outage does not appear to be related to whether duration is missing — missingness is not a seasonal phenomenon.

---

## Hypothesis Testing

**Question**: Does the cause of a power outage significantly affect its severity?

- **Null Hypothesis (H₀)**: The distribution of outage severity is the same across all cause categories. Any observed differences are due to random chance.
- **Alternative Hypothesis (H₁)**: The distribution of outage severity differs across cause categories.
- **Test Statistic**: The difference between the maximum and minimum group mean severity across all cause categories (i.e., the range of group means). This statistic is appropriate because it directly captures how much the "best" and "worst" cause categories diverge in terms of severity.
- **Significance Level**: α = 0.05
- **p-value**: 0.001

**Conclusion**: Since the p-value (0.001) is well below 0.05, we **reject the null hypothesis**. The data suggests that outage cause and severity are not independent — different causes are associated with meaningfully different severity levels. This does not prove a causal mechanism; rather, it provides statistical evidence that the relationship between cause and severity is unlikely to be explained by chance alone.

<iframe
  src="assets/hypothesis-permutation.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

The histogram shows the permutation distribution of the test statistic (range of group means) under the null hypothesis. The red vertical line marks the observed statistic, which falls far into the right tail — visually confirming how extreme the result is under the null.

---

## Framing a Prediction Problem

**Prediction Problem**: Predict the **severity** of a power outage given information available at or near the time the outage begins.

- **Type**: Regression (severity is a continuous numeric value)
- **Response Variable**: `SEVERITY` — a composite of normalized outage duration and normalized customers affected. This was chosen over either individual metric because it captures both the time dimension and the social impact dimension of an outage in a single value, making it a more complete proxy for "how bad was this event?"
- **Evaluation Metric**: Root Mean Squared Error (RMSE). RMSE is preferred over raw MSE because it is expressed in the same units as severity, making it easier to interpret. It is preferred over MAE because it penalizes large errors more heavily, which is appropriate here: severely under-predicting a major outage is much worse than under-predicting a small one.

**Justification of features used at prediction time**: The features used — cause category, NERC region, climate category, month, and hour — are all known at or shortly after the outage begins. Duration and customers affected are *outcomes* that unfold over the course of the event, so they are appropriately excluded from the model. This respects the real-world prediction setting where a dispatcher needs an early severity estimate to allocate resources.

---

## Baseline Model

The baseline model is a **Linear Regression** pipeline using only two quantitative features:

- `month` (quantitative): The calendar month the outage started
- `hour` (quantitative): The hour of day the outage started

Both features were standardized using `StandardScaler` before being passed to the regressor. No categorical encoding was necessary at this stage since no nominal features were included.

**Baseline Performance (Test Set)**:

| Metric | Value |
|---|---|
| MSE | 0.017 |
| RMSE | 0.131 |
| R² | -0.016 |

The baseline model performs poorly. An R² of -0.016 means the model actually does *worse* than simply predicting the mean severity for every outage — a sign that month and hour alone carry almost no predictive signal for severity. This is not surprising: the time an outage starts has little inherent connection to how severe it will be; what matters more is what caused it and where. The baseline establishes a floor to beat and makes clear that richer features are needed.

---

## Final Model

### Feature Engineering

Three new features were added for the final model:

1. **`cause`** (nominal): The cause category is arguably the single most important feature. Different causes (severe weather, fuel supply emergency, intentional attack) have vastly different severity profiles as seen in the EDA. Including this allows the model to learn severity expectations per cause type.
2. **`region`** (nominal): The NERC region captures geographic infrastructure differences — some grids are more robust or redundant than others. A model that knows the region can adjust predictions for underlying resilience.
3. **`climate`** (nominal): Climate category (warm/normal/cold) reflects the environmental stress on infrastructure at the time of the outage, which relates to both how outages start and how quickly crews can respond.
4. **`cause_region`** (interaction, nominal): A string concatenation of cause and region (e.g., `"severe weather_RFC"`). This interaction term lets the model learn that, say, severe weather in the Northeast has a different typical severity than severe weather on the West Coast — a relationship a linear model would miss but a tree-based model can exploit.

`month` and `hour` were retained from the baseline.

All nominal features were encoded with `OneHotEncoder(handle_unknown="ignore")`, and numeric features were passed through without scaling (unnecessary for Random Forest).

### Algorithm and Hyperparameter Tuning

The modeling algorithm is a **Random Forest Regressor**. Random Forests were chosen over linear regression because severity is almost certainly influenced by nonlinear interactions between cause, region, and climate — relationships that a linear model cannot capture. Random Forests also handle high-cardinality one-hot encoded features well and are robust to the skewed target distribution observed in EDA.

Hyperparameters were selected via **3-fold cross-validated Grid Search** over:

| Hyperparameter | Values Tried | Best Value |
|---|---|---|
| `n_estimators` | 300, 500 | **500** |
| `max_depth` | 10, 20 | **10** |
| `min_samples_leaf` | 1, 2, 4 | **4** |

The best configuration (max_depth=10, min_samples_leaf=4, n_estimators=500) prevents overfitting by capping tree depth and requiring a minimum leaf size, which is especially important given that the training set is moderately sized.

### Final Model Performance

| Metric | Baseline (Linear Regression) | Final (Random Forest) |
|---|---|---|
| MSE | 0.017 | **0.014** |
| RMSE | 0.131 | **0.120** |
| R² | -0.016 | **0.147** |

The final model improves on every metric. Most importantly, R² moves from negative (worse than a mean predictor) to 0.147 — meaning the model now explains roughly 15% of the variance in severity. While this is modest, it represents a meaningful structural improvement over the baseline. The RMSE drops by ~8%, reducing average prediction error in real units. The improvement is attributable to adding causal and geographic context that the time-only baseline entirely lacked.

---

## Fairness Analysis

**Goal**: Assess whether the final model performs equally well across different geographic groups — specifically comparing the **RFC region** (Reliability First Corporation, covering parts of the Midwest and Mid-Atlantic) and the **WECC region** (Western Electricity Coordinating Council, covering the Western U.S.).

- **Group X**: RFC | **Group Y**: WECC
- **Evaluation Metric**: RMSE (Root Mean Squared Error)
- **Null Hypothesis (H₀)**: The model is fair. The RMSE for RFC outages is the same as for WECC outages; any observed difference is due to random chance.
- **Alternative Hypothesis (H₁)**: The model is unfair. The RMSE for RFC outages is higher than for WECC outages.
- **Test Statistic**: Difference in RMSE (RFC − WECC)
- **Significance Level**: α = 0.05

**Observed values**:

| Group | Count | RMSE |
|---|---|---|
| RFC | 68 | 0.1370 |
| WECC | 50 | 0.1746 |
| Observed Difference (RFC − WECC) | — | -0.0376 |

**p-value**: 0.7490

**Conclusion**: We **fail to reject the null hypothesis**. The observed RMSE difference (-0.0376, meaning RFC actually had *lower* error than WECC) is not statistically significant under permutation testing. There is no strong evidence that the model performs unfairly between RFC and WECC regions — performance differences are consistent with what would arise by chance given the sample sizes.

<iframe
  src="assets/fairness-permutation.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

> *Note: The fairness permutation plot currently uses matplotlib. To embed it as an interactive iframe, re-generate it with plotly (`px.histogram`) and save with `fig.write_html("assets/fairness-permutation.html", include_plotlyjs="cdn")`.*
