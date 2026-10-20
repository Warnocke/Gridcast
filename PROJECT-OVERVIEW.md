# Grid Demand Forecasting Pipeline

Oct 17, 2026 · @Evan Warnock

## The idea

Build a live system that forecasts next-day hourly electricity demand for JEA, Jacksonville's grid operator, and checks itself every day against both reality and JEA's own forecast.

Grid operators commit power plants a day ahead based on a demand forecast. Over-forecast and they pay to keep units idle; under-forecast and they buy expensive peak power on short notice. Utilities, grid operators, energy traders and retail power providers all pay for better load forecasts, so the problem passes the "who would pay for this" test.

What makes this more than a notebook:

- **A professional baseline.** EIA publishes each grid operator's own day-ahead forecast next to actual demand. Few student projects can measure themselves against the incumbent.
- **A live track record.** The system forecasts before the day happens and scores itself after, so results can't be tuned after the fact.
- **Production concerns.** Scheduled ingestion, data that gets revised, leakage rules, retraining with a promotion gate, and monitoring.

**One-line pitch:** a production pipeline that forecasts JEA's next-day hourly load and has tracked its accuracy against JEA's own forecast every day since launch.

## Data sources

Three free sources cover everything: EIA for demand, Open-Meteo for weather forecasts, and a holiday calendar.

| Source | What you pull | Access and limits | Gotchas |
| --- | --- | --- | --- |
| [EIA Open Data API v2](https://www.eia.gov/opendata/), Form EIA-930 | Hourly demand, the operator's day-ahead demand forecast, net generation and interchange, per balancing authority (JEA, GVL, FPL, FPC, TEC and others) | Free key, issued automatically. Max 5,000 rows per JSON response, so page by date range. EIA's guidance is under about 9,000 requests per hour and 5 per second. History via the API starts in 2019 | Raw data includes outliers and gaps. Demand posts about an hour after the hour ends. Values can be revised later |
| [Open-Meteo](https://open-meteo.com/en/docs/previous-runs-api) | Hourly temperature, dewpoint, humidity, cloud cover, wind | No key for non-commercial use. Forecast API up to 16 days ahead. Previous Runs API gives forecasts at fixed lead times (1 to 7 days) from January 2024. Historical Forecast API goes back to 2021 | Use forecasts, never observed weather, for the target day. See the leakage rule below |
| Holiday calendar (`holidays` Python package, plus UF's academic calendar if you add GVL) | Federal holidays, observed days, school breaks | Static, no API | Holidays are rare, so the model sees few examples. Thanksgiving falls inside your first live month |

Store every raw response unchanged in an append-only area, tagged with the time you fetched it. That lets you rebuild exactly what the model knew on any past morning, even after EIA revises a number.

## Prediction task

Every morning at 9:00 ET, predict JEA's demand in MWh for each of the 24 hours of the next local day.

- **Target:** hourly demand (EIA series type D) for one balancing authority.
- **Horizon:** a 9:00 issue time for tomorrow's midnight-to-midnight window means 15 to 38 hours ahead.
- **Regions:** start with JEA only. Add GVL, FPL, Duke Energy Florida (FPC) and Tampa Electric (TEC) once the pipeline works. Several regions let you train one global model and compare it to per-region models.
- **Output:** a point forecast plus a P10 to P90 range per hour. Operators plan around uncertainty, and most student projects skip it.

**The leakage rule:** a forecast may only use what existed at 9:00 that morning.

- Demand through the latest hour EIA has posted, usually around 8:00.
- Weather forecasts issued before 9:00. Never observed weather for the target day.
- Calendar facts, which are known in advance.

Your backtest must follow the same rule, or its numbers will beat what the live system can do.

**Time zones:** store everything in UTC and convert to local time for features. Daylight saving days have 23 or 25 hours, and the first version of the pipeline will break on them if you don't plan for it. The next one is November 1, 2026.

## Features and models

Weather and recent demand will carry most of the signal; expect gradient-boosted trees to win and say so with numbers.

**Features**

- **Recent demand:** same hour yesterday, same hour last week, yesterday's daily peak and total, rolling 7-day mean by hour. Check each lag is available at 9:00.
- **Weather forecast:** temperature, dewpoint or humidity, apparent temperature, cloud cover. Add cooling and heating degree hours, since Florida demand is driven mostly by air conditioning.
- **Weather location:** average 2 to 4 points across JEA's service area instead of one airport. Weight by population if you want an interview talking point.
- **Calendar:** hour of day, day of week, month, holiday flags, the day before and after a holiday.
- **Interactions:** hour by temperature matters, because 95 °F at 3 PM and 95 °F at 9 AM produce different loads. Trees find this on their own; a linear model needs it spelled out.

**Model ladder** (build in this order; each must beat the one before to earn its complexity)

1. **Seasonal naive:** same hour last week. The floor.
2. **Linear regression** with temperature splines and calendar terms. This is close to what many utilities run, so it's a fair second baseline.
3. **LightGBM**, one model with hour as a feature, predicting all 24 hours. Likely your production model. Train extra quantile models for P10 and P90.
4. **Optional neural model**, such as N-BEATS or a small temporal fusion transformer through a forecasting library. Worth running as an experiment; report it honestly if it loses.

Track every run's parameters, features and backtest scores in MLflow so you can show how the model got better and why.

## Evaluation

The headline number is your error next to JEA's own day-ahead forecast, measured the same way on the same days.

**Backtest design:** rolling origin. For each day in the test window, train only on data before it, build features exactly as they looked at 9:00, forecast the next day, and step forward. A random train/test split would leak the future and is the most common mistake in forecasting projects.

**Backtest window:** start the honest backtest in January 2024, when Open-Meteo's fixed-lead forecasts begin. Earlier years can use the Historical Forecast API for training, but its stitched short-lead forecasts are more accurate than a real day-ahead forecast, so label those results as optimistic.

| Metric | Why it matters |
| --- | --- |
| MAPE | The industry's standard load-forecast metric, comparable across regions |
| MAE in MW | Readable for one region; easier to explain than a percentage |
| Daily peak error (size and hour) | Peaks set costs and reliability risk |
| Pinball loss and P10 to P90 coverage | Whether your ranges are honest: about 80% of actuals should land inside |

**Compare against:** seasonal naive, the linear baseline, and the operator's forecast (EIA series type DF).

**Slice the results:** by season, heat waves, holidays and DST days. An average can hide that you fail on the hottest days, which are the ones operators care about most.

**Retrain cadence experiment:** rerun the backtest with no retraining, monthly, weekly and daily retraining. Pick the cadence from the results, not from intuition.

## The daily loops

Predictions and scoring run every day; retraining runs on the schedule your backtest picks, probably weekly.

| When (ET) | Job | Output |
| --- | --- | --- |
| Hourly | Ingest EIA demand and forecast data; store raw responses with fetch time | Raw layer grows; late or revised values are kept as new versions |
| Daily, 8:45 | Pull the latest weather forecast for your points | Weather snapshot tagged with its issue time |
| Daily, 8:50 | Validate inputs: missing hours, spikes, stale weather | Pass, or an alert and a fallback to the last good model inputs |
| Daily, 9:00 | Build features as of now, load the production model, predict tomorrow | 24 hourly forecasts with P10 to P90, saved with model version |
| Daily, midday | Score yesterday: your forecast and the operator's against actuals | One row per day in a scores table; feeds the dashboard |
| Weekly | Retrain a challenger on all history; backtest it on the last 8 weeks | Promote only if it beats the current model; log both in MLflow |
| On alert | Rolling 14-day error passes a threshold you set from backtest variance | Triggers an early retrain and a notification |

Two rules keep this honest. Never overwrite a past forecast, and never let a retrain touch the days it will be scored on.

## System designs

Option B is the one to build: it is small enough to finish this semester and real enough to defend in an interview.

|  | A. Scripts and cron | B. Small production stack (recommended) | C. Platform scale |
| --- | --- | --- | --- |
| Scheduling | GitHub Actions scheduled workflows | Prefect flows in Docker on one small cloud VM | Airflow or Dagster on Kubernetes |
| Storage | Parquet files, queried with DuckDB | Raw Parquet in S3; Postgres (Supabase) for forecasts, scores and metadata | Data lake plus a feature store such as Feast |
| Models | Pickled model in the repo | MLflow tracking and model registry with a promotion gate | Same, plus a dedicated model-serving platform |
| Serving | Static dashboard | FastAPI service plus a Next.js dashboard | Autoscaled API behind a gateway |
| What it proves | You can automate a script | You can run, monitor and retrain a model in production | Tool familiarity |
| Main weakness | Scheduled workflows can start late or be skipped under load; reads as a hobby setup | You run and pay for a VM | Overbuilt for one hourly feed; interviewers will ask why |

Start with A's simplicity for the first week (local scripts, DuckDB) so you have backtest results fast. Then move the same code into B's structure. Treat C as an interview answer to "how would this scale to every grid operator in the US," not something to build.

&#91;embedded content: option B architecture · 10 components\]

The feature builder is the piece to protect: training and the daily forecast call the same code, so the model never sees inputs in production that differ from what it trained on. The scorer's drift alert can also trigger the training job early.

## Tech stack

Every item here should end up behind a resume bullet; anything you can't explain in an interview, leave out.

| Layer | Choice | Why | Fallback |
| --- | --- | --- | --- |
| Language and packaging | Python with uv | Fast, reproducible environments | pip and venv |
| Data wrangling | Pandas, DuckDB, Parquet | DuckDB is already on your resume; Parquet keeps raw pulls cheap | Polars |
| Validation | Pandera or Pydantic | Catches gaps and spikes before they reach the model | Hand-written checks |
| Models | scikit-learn, LightGBM | Linear baseline plus the likely winner | XGBoost |
| Tracking and registry | MLflow | Run history and a promotion gate | Weights & Biases |
| Orchestration | Prefect | Python-native, light to run, retries and logs built in | Cron in Docker |
| Storage | S3 for raw; Postgres via Supabase for serving tables | You already know Supabase | Postgres on the VM |
| Serving | FastAPI in Docker | Backs up the Docker line on your resume | Flask |
| Dashboard | Next.js | Your TypeScript strength, and a live link recruiters can click | Streamlit |
| CI | GitHub Actions: tests, lint, image build | Shows engineering hygiene | None |
| Cloud | AWS: one small EC2 instance and S3 | The most requested cloud in postings | Any VM provider |

Add to your skills line only after it ships: LightGBM, MLflow, Prefect, FastAPI, AWS. Docker and DuckDB stop being unsupported claims.

## Build plan

Getting the forecast live by week six matters most, because every day it runs adds to the track record.

&#91;embedded content: build plan · 4 phases, 3 gates\]

Dates assume a part-time pace starting Monday, October 19. If classes or applications eat a week, slip phase 4, not phase 3: a live system with a plain dashboard beats a polished one that started late.

## Resume and interview framing

The entry should lead with the live comparison to the operator's forecast; fill the brackets only with numbers the system actually produced.

**Draft bullets** (rewrite in your own voice once results exist)

- Built a production pipeline forecasting next-day hourly electricity demand for Jacksonville's grid, running unattended for \[N\] days with daily ingestion, scoring and weekly retraining
- Reached \[X\]% MAPE on a \[N\]-month rolling backtest, within \[Y\] points of the grid operator's own day-ahead forecast, using LightGBM over weather-forecast and calendar features
- Prevented lookahead leakage by rebuilding each forecast from data as it existed at issue time, including versioned EIA revisions and archived weather forecasts
- Served forecasts and P10 to P90 ranges through a Dockerized FastAPI service on AWS, with MLflow model promotion and drift-triggered retraining

**Stories to have ready**

- Why the backtest uses forecasted weather, and how much worse it got when you switched from observed weather.
- Why LightGBM won, or why the neural model lost.
- How you picked the retraining cadence from the backtest.
- What broke on the first DST day or the first missing EIA hour, and what you changed.
- How you'd scale it to all 65 US balancing authorities (option C).

## Risks and open questions

The biggest risk is scope: a working option B with honest numbers beats a half-built option C.

| Risk | Mitigation |
| --- | --- |
| Time, alongside classes, ACM and applications | Hold the scope in this doc; cut the neural model and extra regions first |
| Small regions are noisy (GVL is much smaller than JEA) | Start with JEA; add small regions only as a robustness check |
| EIA gaps, spikes and revisions | Validation step, versioned raw storage, fall back to the last good inputs |
| Unfair comparison with the operator's forecast | Confirm when JEA issues its forecast before claiming you beat it |
| Cloud cost creep | One small VM, budget alerts on day one, stop anything idle |

**Open questions**

- [ ] When is JEA's day-ahead forecast issued, and is 9:00 ET a fair issue time to compare against it?
- [ ] Which 2 to 4 weather points best represent JEA's service area?
- [ ] Does JEA's EIA history have long gaps or outliers that need a cleaning rule?
- [ ] Single region first, or JEA plus FPL from the start for a global model?

## Sources

- [EIA Open Data](https://www.eia.gov/opendata/)
- [About the EIA-930 data](https://www.eia.gov/electricity/930-content/about.html)
- [EIA Hourly Electric Grid Monitor timing](https://www.eia.gov/todayinenergy/detail.php?id=43295)
- [EIA-930 data notes (PUDL)](https://docs.catalyst.coop/pudl/en/stable/data_sources/eia930.html)
- [EIA API limits](https://www.oilpriceapi.com/eia-api)
- [Open-Meteo Previous Runs API](https://open-meteo.com/en/docs/previous-runs-api)
- [Open-Meteo Historical Forecast API](https://openmeteo.substack.com/p/introducing-the-historical-forecast)
- [Open-Meteo Single Runs API (data availability by API)](https://openmeteo.substack.com/p/single-runs-api)
- [Balancing authority list (GridInfo)](https://www.gridinfo.com/balancing-authorities?o=ba&d=asc)
