# IEX Day-Ahead Price Forecaster — Project Document

*As of 2026-11-21 · Author: BK*

## Summary

The system forecasts tomorrow's electricity price for each of the 96 fifteen-minute blocks on the Indian Energy Exchange (IEX) day-ahead market. It runs every morning before bidding closes and measures how much money those forecasts would save a buyer who can shift some of their load.

It runs daily on AWS and keeps a public live track record. It's judged by backtested and live error against simple baselines, and by rupees saved, not by a demo alone. This is a personal GitHub portfolio project using only public data.

## The problem

Factories with open-access connections, power distribution companies and traders buy electricity on IEX's day-ahead market. Each morning they bid for the next day's 96 blocks. Getting the timing right is hard:

- **Prices swing hard within a day.** They're cheap when solar output peaks around noon and expensive in the evening peak, when demand is high and solar is gone.
- **Weather drives everything.** Heatwaves push demand up, and cloud cover and wind change renewable supply, so yesterday's prices are a weak guide.
- **The market keeps changing.** Solar capacity grows every year, and rule changes such as price caps shift the price pattern.
- **Only some information exists at bidding time.** A forecast that quietly uses data from after the bidding deadline looks great in testing and fails in real use.

A buyer with flexible load, such as a factory that can move a batch process by a few hours, saves real money by scheduling that load into tomorrow's cheapest blocks. That needs a forecast they can trust, with an honest range around it.

## Goals and non-goals

**Goals (version 1):**

1. A daily pipeline that loads prices, grid and weather data into a store you can query with SQL, in three layers: raw → clean → features.
2. Forecast all 96 blocks of day D by 09:00 IST on day D-1, with a low, middle and high value (P10 / P50 / P90).
3. A 12-month backtest that replays each morning using only data available at that morning's cutoff.
4. Turn forecast accuracy into rupees with a simple flexible-load scheduling strategy.
5. Run live on AWS every day from week 3, with monitoring and alerts.
6. A dashboard showing forecast vs. actual, accuracy over time, and savings.

**Stretch goal (only after the core is live):**

7. An **"ask your data" agent**: ask questions about prices, forecasts and accuracy in plain English. It writes SQL against the project's own tables, runs it read-only, and explains the answer with the SQL shown.

**Non-goals:**

- No automated bidding or trading. Output is information, not advice.
- Only the all-India price. Regional prices, when grid congestion splits the market, go on a "later" list.
- No real-time, high-price or term-ahead markets in version 1.
- No paid data sources.

## Data

All inputs are public. No employer data is used.

| Source                                                                                                                                                                  | What                                                                                           | Granularity   | Period               | Notes                                           |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ------------- | -------------------- | ----------------------------------------------- |
| [IEX day-ahead market data](https://iexindia.com/market-data/day-ahead-market/market-snapshot)                                                                           | Market clearing price, purchase and sell bids, cleared volume                                  | 15 min        | Apr 2022 → today    | Check export options and terms of use in week 1 |
| [Grid-India-based dataset (Mendeley)](https://data.mendeley.com/datasets/y58jknpgs8/2)                                                                                   | National demand, solar and wind generation                                                     | 1 hour        | Sep 2021 – Jun 2025 | History for training                            |
| [India generation dataset (IEEE DataPort)](https://ieee-dataport.org/documents/indian-energy-generation-observatory-multi-source-dataset-energy-generation-power-supply) | Daily and 15-minute grid observations from CEA and Grid-India reports                          | Daily, 15 min | Oct 2025 – Oct 2026 | Check access and licence                        |
| Grid-India daily reports                                                                                                                                                | Demand and supply position                                                                     | Daily         | Ongoing              | Live updates; parse format in week 1            |
| Open-Meteo                                                                                                                                                              | Historical weather and forecasts: temperature, solar radiation, wind, for major demand centres | 1 hour        | Ongoing              | Free for non-commercial use                     |
| Python `holidays` library + festival list                                                                                                                             | National and state holidays, major festivals                                                   | Daily         | —                   | Calendar features                               |

**The information cutoff.** The forecast for day D is made at 09:00 IST on day D-1. At that moment:

- **Known:** IEX prices up to the end of D-1 (cleared the day before), weather forecasts for D, and the calendar.
- **Not known:** anything that happens on D.

Every feature must respect this, and a test enforces it.

**Open question:** is there a steady live source of demand and renewable data after June 2025? Check in week 1. If not, use weather-only features, since weather drives demand, solar and wind anyway, and report what that costs in accuracy.

## How it works

The pipeline runs once a day, plus a weekly retrain. Only step 4 is machine learning; everything else is plain code, so it's easy to test.

| # | Step      | What it does                                                                        | ML or plain code | Main tools                    |
| - | --------- | ----------------------------------------------------------------------------------- | ---------------- | ----------------------------- |
| 1 | Ingest    | Fetch new prices, grid data and weather forecasts; save raw files                   | Plain code       | httpx, boto3 → S3            |
| 2 | Validate  | Check 96 blocks present, prices within bounds, no duplicates; stop and alert if not | Plain code       | Pandera                       |
| 3 | Transform | Raw → clean Parquet → feature tables, written in SQL                              | Plain code (SQL) | DuckDB locally, Athena on AWS |
| 4 | Forecast  | P10 / P50 / P90 for all 96 blocks of day D                                          | ML               | LightGBM (quantile), PyTorch  |
| 5 | Publish   | Save forecasts; serve them through an API                                           | Plain code       | S3, FastAPI                   |
| 6 | Reconcile | Next day, compare with actual prices; update rolling accuracy                       | Plain code       | DuckDB                        |
| 7 | Monitor   | Alert if the run fails, misses 09:00, or accuracy drifts                            | Plain code       | CloudWatch, SNS email         |
| 8 | Show      | Forecast vs. actual, accuracy over time, savings                                    | Plain code       | Streamlit                     |

**Weekly retrain.** A new model replaces the live one only if it beats  on the last 8 weeks of backtest. Every model is versioned, so any forecast can be traced to the model that made it.

**Models, from simplest to most complex:**

| Model                | Idea                                                                               | Role                     |
| -------------------- | ---------------------------------------------------------------------------------- | ------------------------ |
| Same block last week | Tomorrow 18:00 = last week's 18:00                                                 | Main baseline            |
| Same block yesterday | Tomorrow 18:00 = today's 18:00                                                     | Second baseline          |
| LightGBM, quantile   | Recent prices, weather forecast, calendar → P10 / P50 / P90 per block             | Main model               |
| PyTorch network      | Last 7 days of prices + tomorrow's weather → all 96 blocks × 3 quantiles at once | Deep-learning comparison |

### Stretch: the ask-your-data agent

**Why it's here.** Teams that own large business data increasingly put agents on top of it, so people can get answers without writing SQL. This agent uses the tables the pipeline already builds, so it needs no new data.

**How it works:**

1. The user asks a question in the dashboard's chat panel, e.g. *"How accurate were the evening forecasts on days above 40°C?"*
2. The agent checks which tables and columns exist, writes a SQL query, and runs it.
3. If the query fails or returns something odd (no rows, impossible values), it reads the error and tries again, up to 3 times.
4. It answers with the numbers, shows the SQL it ran, and draws a chart when the result is a time series.
5. Follow-up questions keep the conversation's context ("and for the afternoon?").

**Tools it can call:**

| Tool               | What it does                                | Guardrail                                            |
| ------------------ | ------------------------------------------- | ---------------------------------------------------- |
| `list_tables`    | Table names with one-line descriptions      | Only the clean and feature tables are visible        |
| `describe_table` | Columns, types, units and example values    | —                                                   |
| `run_sql`        | Runs one query and returns at most 500 rows | Read-only connection; SELECT only; 10-second timeout |
| `plot`           | Draws a line or bar chart from a result     | —                                                   |

**Guardrails.**

- **The database connection is read-only,** so the agent can't change or delete data even if it tries.
- **Every answer shows the SQL it ran,** so the person asking can check it.
- **Questions the data can't answer** get a plain "the data doesn't cover that", not a guess.

## Output format

One record per block. The code rejects any record that doesn't match this shape.

```json
{
  "delivery_date": "2026-12-15",
  "block": 73,
  "block_start": "18:00",
  "issued_at": "2026-12-14T09:00:00+05:30",
  "model_version": "lgbm-q-2026-12-10",
  "p10_rs_mwh": 6120.0,
  "p50_rs_mwh": 8450.0,
  "p90_rs_mwh": 9800.0,
  "actual_rs_mwh": null
}
```

The values are made up to show the shape. `actual_rs_mwh` is filled in by the reconcile step the next day.

## Tech stack

| Part                | Choice                                                      | Why                                                           |
| ------------------- | ----------------------------------------------------------- | ------------------------------------------------------------- |
| Language            | Python 3.12, uv                                             | Same as the RBI project                                       |
| Storage             | Parquet on S3 in three layers (raw / clean / features)      | Cheap, columnar, the standard lakehouse layout                |
| SQL                 | DuckDB locally, Athena on AWS                               | Same SQL in both places; Athena costs pennies at this size    |
| Data checks         | Pandera                                                     | Fails loudly on bad data                                      |
| Models              | LightGBM, PyTorch                                           | Strong tabular baseline, plus a deep-learning comparison      |
| Experiment tracking | MLflow                                                      | Versions every model and backtest                             |
| API                 | FastAPI                                                     | Small, typed                                                  |
| Dashboard           | Streamlit                                                   | Quick to build for internal-style data apps                   |
| Agent (stretch)     | LangGraph + one API model, on a read-only DuckDB connection | LangGraph is widely used in industry; read-only keeps it safe |
| Scheduling          | EventBridge Scheduler → ECS Fargate task (Docker)          | Runs a few minutes a day; no server left running              |
| Monitoring          | CloudWatch logs and alarms, SNS email                       | Failure and drift alerts                                      |
| CI/CD               | GitHub Actions: tests → build image → push to ECR         | Every change tested and deployed the same way                 |
| Tests               | pytest                                                      | Includes a test that no feature uses data after the cutoff    |

**Cost.** S3, Athena and a short daily Fargate task should stay well under ₹500 a month. Set an AWS billing alert at ₹500 on day one.

## Evaluation

Every number is compared with the simple baselines, so the results show what the ML actually adds. The targets are starting goals, to be adjusted once the first results are in.

| What is measured              | Metric                                                                                                             | Baseline to beat                                          | Starting target                              |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------- | -------------------------------------------- |
| Point accuracy                | Mean absolute error in ₹/MWh, broken down by hour and month                                                       | Same block last week                                      | 20%+ lower error                             |
| Evening peak                  | Error on 18:00–23:00 blocks only                                                                                  | Same                                                      | Report separately                            |
| Uncertainty                   | Share of actual prices inside P10–P90 (should be about 80%)                                                       | —                                                        | 75–85%                                      |
| Money                         | ₹ saved by scheduling X MWh a day of flexible load into the forecast's cheapest blocks                            | Naive schedule; perfect-foresight schedule as the ceiling | Capture 60%+ of the perfect-foresight saving |
| Live vs. backtest             | Rolling 30-day live error                                                                                          | Backtest error                                            | Within 15%                                   |
| Reliability                   | Days the forecast was published by 09:00                                                                           | —                                                        | 95%+                                         |
| Cost                          | AWS cost per month                                                                                                 | —                                                        | Under ₹500                                  |
| Ask-your-data agent (stretch) | Share of 30 test questions answered correctly, checked against hand-written queries; failures broken down by cause | —                                                        | 80%+                                         |

**Money matters most.** A model with a slightly higher average error can still save more if it gets the cheapest and most expensive blocks right. The README leads with rupees, then error.

## Timeline

Six weeks to a public repo. The live pipeline starts in week 3, so the track record grows from then on.

| Week    | Phase      | Work                                                                       | Gate before moving on                                   |
| ------- | ---------- | -------------------------------------------------------------------------- | ------------------------------------------------------- |
| 1       | Data       | Confirm sources, exports and terms; build history; write the cutoff rules  | 3+ years of clean 15-minute prices aligned with weather |
| 2       | Backtest   | Rolling backtest harness, baselines, LightGBM                              | Harness reproducible; LightGBM beats both baselines     |
| 3       | Go live    | AWS daily pipeline: ingest → validate → forecast → publish → reconcile | 5 days in a row published by 09:00                      |
| 4       | Models     | Quantile forecasts, PyTorch model, money metric                            | Results table complete                                  |
| 5       | Operations | Alerts, weekly retrain with promotion check, dashboard                     | An alert fires on a test failure                        |
| 6       | Write-up   | README, demo GIF, cleanup                                                  | —                                                      |
| Stretch | Agent      | Ask-your-data agent and its 30-question test set                           | Start only after the week 5 gate passes                 |

## Repository layout

```
iex-price-forecaster/
├── README.md              # problem, dashboard GIF, results table, live record
├── pyproject.toml         # managed with uv
├── src/iexcast/
│   ├── ingest/            # step 1: one module per source
│   ├── validate.py        # step 2 (Pandera schemas)
│   ├── sql/               # step 3: SQL for clean and feature tables
│   ├── features.py        # cutoff-aware feature builder
│   ├── models/            # baselines, LightGBM, PyTorch
│   ├── forecast.py        # step 4
│   ├── publish.py         # step 5
│   └── reconcile.py       # step 6
├── backtest/
│   ├── run_backtest.py    # rolling replay of past mornings
│   └── results/           # one JSON per model and period
├── strategy/              # flexible-load scheduling and ₹ saved
├── api/                   # FastAPI app
├── dashboard/             # Streamlit app
├── agent/                 # stretch: ask-your-data agent (tools, prompts, 30 test questions)
├── infra/                 # Dockerfile, ECS task, EventBridge schedule
├── .github/workflows/     # CI/CD
└── tests/                 # includes the no-future-data test
```

## Risks

| Risk                                         | What happens                       | How to handle it                                                                                      |
| -------------------------------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------- |
| IEX doesn't allow bulk or automated download | Price history is hard to collect   | Check terms in week 1; use the site's own export manually if needed; never scrape against its terms   |
| Grid data stops after mid-2025               | Fewer features for live runs       | Fall back to weather-only features and report the accuracy cost                                       |
| Market rule changes                          | Old patterns stop holding          | Mark rule-change dates; report accuracy before and after separately                                   |
| Future data leaks into features              | Backtest looks better than reality | Cutoff rules plus an automated test; live vs. backtest error shows any gap                            |
| AWS costs creep up                           | Surprise bill                      | ₹500 billing alert; scheduled tasks only; nothing left running                                       |
| Someone trades on the forecasts              | Real money lost                    | Clear disclaimer: research only, not trading advice                                                   |
| The agent writes wrong SQL                   | Confident wrong answers            | SQL shown with every answer; 30-question test set; read-only connection so mistakes can't damage data |
| Scope creep                                  | Project never ships                | Version 1 is the six goals only; anything else goes in a "later" list                                 |
| Overlap with day job                         | Employer IP questions              | Public energy data only; nothing from GRL                                                             |

## What the README should show

1. **One-line problem statement** and a 20-second GIF of the dashboard.
2. **Money first:** ₹ saved by the strategy vs. naive and vs. perfect foresight.
3. **Results table:** each metric for the baselines, LightGBM and PyTorch.
4. **Live record:** days running, rolling error, days published on time.
5. **Architecture diagram:** sources → S3 layers → SQL → model → API and dashboard, with the AWS services named.
6. **Design choices:** the information cutoff, why quantiles, why LightGBM beat or lost to PyTorch.
7. **Ask-your-data agent** (if built): a short GIF of a question, the SQL it wrote and the answer, plus its test-set score.
8. **Known limits** and **how to run it** in three commands.
