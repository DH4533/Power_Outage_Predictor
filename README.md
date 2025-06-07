# Power_Outage_Predictor

# Introduction

This project explores patterns in major power outages across the U.S., using a dataset that contains hundreds of outage events along with weather, geographic, and economic attributes. Power outages can have major consequences — from public safety to economic losses — and understanding their causes helps improve infrastructure resilience.

Our main question is: **Do weather-related outages last longer than non-weather-related outages?** We also explore how well we can **predict whether an outage is caused by weather** based on information available at the time of the event.

- Number of rows: ~1900
- Relevant columns:
  - `OUTAGE.DURATION` (duration of the outage in minutes)
  - `CAUSE.CATEGORY` (general cause type)
  - `CLIMATE.CATEGORY` (e.g., Cold, Warm, Normal)
  - `NERC.REGION` (location of the outage)
  - `CUSTOMERS.AFFECTED` (impact size)
  - `OUTAGE.START` (timestamp when outage began)

# Data Cleaning and Exploratory Data Analysis

We performed the following key cleaning steps:

- Converted `OUTAGE.START.DATE` and `OUTAGE.RESTORATION.DATE` to datetime format.
- Combined date and time columns into single `OUTAGE.START` and `OUTAGE.RESTORATION` timestamp columns using `pd.to_datetime` and `pd.to_timedelta`.
- Created `OUTAGE.DURATION.HOURS` by dividing the duration in minutes by 60.
- Added derived time features like `OUTAGE.START.HOUR` and `OUTAGE.START.MONTH`.

| OUTAGE.START        |   OUTAGE.DURATION.HOURS | CAUSE.CATEGORY   | CLIMATE.CATEGORY   |   CUSTOMERS.AFFECTED |
|:--------------------|------------------------:|:-----------------|:-------------------|---------------------:|
| 2011-07-01 05:00:00 |                    51   | severe weather   | normal             |                70000 |
| 2010-10-26 08:00:00 |                    50   | severe weather   | cold               |                70000 |
| 2012-06-19 04:30:00 |                    42.5 | severe weather   | normal             |                68200 |
| 2015-07-18 02:00:00 |                    29   | severe weather   | warm               |               250000 |
| 2010-11-13 03:00:00 |                    31   | severe weather   | cold               |                60000 |

Below is a distribution of outage durations, showing a strong right skew.

<iframe src="assets/duration_hist.html" width="900" height="600" frameborder="0"></iframe>

This plot shows how outage duration varies across different cause categories. **Weather-related causes** and **fuel supply issues** tend to have the longest durations on average.

<iframe src="assets/duration_by_cause.html" width="900" height="600" frameborder="0"></iframe>

| Cause                 |   cold |   normal |   warm |
|:----------------------|-------:|---------:|-------:|
| equipment failure     |    5.1 |     53.4 |    8.4 |
| fuel supply emergency |  290.5 |    127.6 |  380   |
| intentional attack    |    8.3 |      7.1 |    5.2 |
| islanding             |    4.3 |      2.4 |    3.5 |
| public appeal         |   35.4 |     22.9 |    9.9 |

# Assessment of Missingness

We suspect that the `CAUSE.CATEGORY.DETAIL` column is **Not Missing At Random (NMAR)**.

The missingness of this column likely depends on the value itself. In many cases, a utility might not provide a detailed explanation of the cause if the event was ambiguous or uncertain. This means the missingness is tied to the nature of the outage itself — not just other columns. As a result, we believe this column is NMAR.

To confirm this, we would ideally want access to internal reporting policies from utilities, or more metadata about when and how detailed causes are logged.

We tested whether missingness in `CAUSE.CATEGORY.DETAIL` depends on `CLIMATE.CATEGORY` using a permutation test.

The observed difference in missingness rates was **0.0827**.  
The permutation test yielded a **p-value of 0.0220**, which is below our 0.05 significance level.

This suggests that missingness in `CAUSE.CATEGORY.DETAIL` **does depend** on `CLIMATE.CATEGORY`, and is therefore **not Missing Completely At Random (MCAR)**.

<iframe src="assets/missingness_test.html" width="800" height="600" frameborder="0"></iframe>

# Hypothesis Testing

We wanted to test whether **weather-related outages** tend to last longer than **non-weather-related outages**.

We created a new binary column `IS_WEATHER` where 1 indicates a weather-related cause. We then performed a permutation test on `OUTAGE.DURATION.HOURS` to test the following hypotheses:

- **Null Hypothesis (H₀):** Weather-related and non-weather-related outages have the same average duration.
- **Alternative Hypothesis (H₁):** Weather-related outages last longer, on average.

The observed difference in mean duration was **42.30 hours**.  
The permutation test yielded a **p-value of 0.0000**.

This very small p-value provides strong evidence against the null hypothesis, suggesting that **weather-related outages are significantly longer** than non-weather outages.

<iframe src="assets/weather_duration_test.html" width="850" height="600" frameborder="0"></iframe>

# Framing a Prediction Problem

Our prediction problem is:

> **Can we predict whether a power outage was caused by weather, based only on features available at the time of the event?**

This is a **binary classification** problem, where:
- `1` = Weather-related outage
- `0` = Non-weather-related outage

We created a column `IS_WEATHER` as the target variable.

We chose this problem because accurately predicting weather-related outages could help utility companies allocate resources and issue alerts faster in real time.

We selected the following features to predict `IS_WEATHER`:
- `MONTH`: Indicates the time of year (seasonal patterns)
- `NERC.REGION`: Captures regional variability in weather and infrastructure
- `CUSTOMERS.AFFECTED`: Size of the impact (log-transformed)
- `CLIMATE.CATEGORY`: Captures climate episodes like El Niño
- `OUTAGE.START.HOUR`: Time of day

### Evaluation Metric

Since both false positives and false negatives are important (we don’t want to miss weather events, but also don’t want to falsely predict them), we use the **F1-score** as our evaluation metric. It balances **precision** and **recall**.

# Baseline Model

For our baseline model, we used two features:
- `MONTH` (quantitative): The time of year the outage occurred
- `NERC.REGION` (categorical): Regional code that can capture geographic variation

We trained a **logistic regression classifier** using a scikit-learn `Pipeline`. We used:
- `StandardScaler` on `MONTH`
- `OneHotEncoder` on `NERC.REGION`

We evaluated the model using 5-fold cross-validation and computed the **F1-score** for each fold:

- Fold scores: 0.701, 0.629, 0.726, 0.538, 0.585
- **Average F1-score**: **0.636**

This simple model serves as our baseline and will be compared against a more complex model in the next step.

# Final Model

For our final model, we expanded the feature set to better capture patterns in weather-related outages. In addition to `MONTH` and `NERC.REGION`, we added:

- `LOG_CUSTOMERS_AFFECTED`: Log-transformed number of customers affected (impact size)
- `CLIMATE.CATEGORY`: Seasonal climate condition (e.g., Warm, Cold)
- `OUTAGE.START.HOUR`: Time of day the outage began

We used a **Random Forest Classifier**, which can capture nonlinear interactions between features. We tuned the following hyperparameters using `GridSearchCV`:
- `n_estimators`: [50, 100]
- `max_depth`: [5, 10, None]

### 🔍 Best Parameters
```text
max_depth = None  
n_estimators = 100
```

# Fairness Analysis

To assess fairness, we asked:

> **Does our model perform equally well across different seasons — specifically Winter vs. Summer outages?**

We used **precision** as our metric, since it's important to know whether the weather-related predictions we make are actually correct. We compared:

- **Winter** = December–February
- **Summer** = June–August

We conducted a **permutation test** with the following hypotheses:

- **Null Hypothesis (H₀):** Precision is the same for Winter and Summer outages.
- **Alternative Hypothesis (H₁):** Precision differs between the two groups.

### 📊 Results
- **Observed precision difference**: 0.0263
- **P-value**: 1.0000

Since the p-value is very high, we **fail to reject the null hypothesis**, meaning there is **no significant evidence** of unfairness in model precision between Winter and Summer outages.

<iframe src="assets/fairness_precision_test.html" width="800" height="600" frameborder="0"></iframe>
