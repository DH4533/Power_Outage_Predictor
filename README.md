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

