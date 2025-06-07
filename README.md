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

Below is a distribution of outage durations, showing a strong right skew.

<iframe src="imgs/duration_hist.html" width="800" height="500" frameborder="0"></iframe>

This plot shows how outage duration varies across different cause categories. **Weather-related causes** and **fuel supply issues** tend to have the longest durations on average.

<iframe src="imgs/duration_by_cause.html" width="900" height="550" frameborder="0"></iframe>

# Assessment of Missingness

We suspect that the `CAUSE.CATEGORY.DETAIL` column is **Not Missing At Random (NMAR)**.

The missingness of this column likely depends on the value itself. In many cases, a utility might not provide a detailed explanation of the cause if the event was ambiguous or uncertain. This means the missingness is tied to the nature of the outage itself — not just other columns. As a result, we believe this column is NMAR.

To confirm this, we would ideally want access to internal reporting policies from utilities, or more metadata about when and how detailed causes are logged.

We tested whether missingness in `CAUSE.CATEGORY.DETAIL` depends on `CLIMATE.CATEGORY` using a permutation test.

The observed difference in missingness rates was **0.0827**.  
The permutation test yielded a **p-value of 0.0220**, which is below our 0.05 significance level.

This suggests that missingness in `CAUSE.CATEGORY.DETAIL` **does depend** on `CLIMATE.CATEGORY`, and is therefore **not Missing Completely At Random (MCAR)**.

<iframe src="imgs/missingness_test.html" width="800" height="500" frameborder="0"></iframe>

# Hypothesis Testing

We wanted to test whether **weather-related outages** tend to last longer than **non-weather-related outages**.

We created a new binary column `IS_WEATHER` where 1 indicates a weather-related cause. We then performed a permutation test on `OUTAGE.DURATION.HOURS` to test the following hypotheses:

- **Null Hypothesis (H₀):** Weather-related and non-weather-related outages have the same average duration.
- **Alternative Hypothesis (H₁):** Weather-related outages last longer, on average.

The observed difference in mean duration was **42.30 hours**.  
The permutation test yielded a **p-value of 0.0000**.

This very small p-value provides strong evidence against the null hypothesis, suggesting that **weather-related outages are significantly longer** than non-weather outages.

<iframe src="imgs/weather_duration_test.html" width="850" height="500" frameborder="0"></iframe>
