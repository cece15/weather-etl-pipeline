# Weather Data ETL Pipeline

![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat&logo=python&logoColor=white)
![Google BigQuery](https://img.shields.io/badge/Google%20BigQuery-4285F4?style=flat&logo=googlebigquery&logoColor=white)
![GCP](https://img.shields.io/badge/Google%20Cloud-4285F4?style=flat&logo=googlecloud&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-003B57?style=flat&logo=sqlite&logoColor=white)
![Looker Studio](https://img.shields.io/badge/Looker%20Studio-4285F4?style=flat&logo=looker&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-150458?style=flat&logo=pandas&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-green?style=flat)

An ETL pipeline for ten years of hourly weather readings from Szeged, Hungary (2006–2016). It cleans the data, adds time-of-day features, loads the result into a local SQLite database and Google BigQuery, and feeds a Looker Studio dashboard.

---

## Live Project Page

> **[View Interactive Project Page](https://cece15.github.io/weather-etl-pipeline)**
> Full pipeline documentation with architecture diagram and technical challenges

## Dashboard Preview

> [View Notebook on Google Colab](https://colab.research.google.com/drive/19O1rZ1WewTlzk16MWzNl0k0uApxSMDyR?usp=sharing)
> 
> [View Live Looker Studio Dashboard](https://lookerstudio.google.com/reporting/7f2762fb-3187-4767-ad7e-ee9708269c04)

**4-page interactive dashboard built on BigQuery:**
- **Page 1 — Overview:** 10-year temperature trend + KPI scorecards
- **Page 2 — Temperature Analysis:** Avg temp by Part of Day & Weekday  
- **Page 3 — Weather Factors:** Monthly humidity & wind speed trends
- **Page 4 — Heatmap:** Avg temperature across Part of Day × Weekday

---

## Pipeline Architecture

```
┌─────────────────┐     ┌──────────────────────────┐     ┌────────────────────────┐
│                 │     │                          │     │                        │
│   CSV Source    │────▶│     TRANSFORMATION       │────▶│      DATA LOADING      │
│  (96K+ records) │     │                          │     │                        │
│                 │     │  ✦ Deduplication         │     │  ✦ SQLite (local)      │
└─────────────────┘     │  ✦ Null handling         │     │  ✦ Google BigQuery     │
                        │  ✦ Datetime normalization│     │     (GCP cloud)        │
                        │  ✦ Temp rounding         │     │                        │
                        │  ✦ Feature engineering   │     └──────────┬─────────────┘
                        │    (Part of Day,          │                │
                        │     Weekday, Hour)        │                ▼
                        │  ✦ BigQuery schema        │     ┌────────────────────────┐
                        │    sanitization (regex)   │     │   VERIFICATION         │
                        └──────────────────────────┘     │  ✦ Row count checks    │
                                                          │  ✦ Query validation    │
                                                          └──────────┬─────────────┘
                                                                     │
                                                                     ▼
                                                          ┌────────────────────────┐
                                                          │  VISUALIZATION         │
                                                          │  ✦ Looker Studio       │
                                                          │  ✦ Matplotlib charts   │
                                                          └────────────────────────┘
```

---

## Key Features

- **Two load targets** — the cleaned data goes to a local SQLite database and to Google BigQuery
- **Modular version** — the final notebook cell rewrites each ETL phase as its own function, run by `run_pipeline()`
- **BigQuery schema compatibility** — regex-based column sanitization handles special characters, spaces, and reserved keywords
- **Structured logging** — INFO / WARNING / ERROR levels written to both console and `etl_pipeline.log`
- **Error handling** — try/except blocks at each stage, with row-count checks and table existence checks
- **Feature engineering** — derives `Part of Day`, `Weekday`, and `Hour` from raw timestamps
- **Cloud authentication** — secure GCP OAuth via Google Colab's built-in auth flow

---

## Repository Structure

```
weather-etl-pipeline/
│
├── ETL_Pipeline_Project (1).ipynb   # Full pipeline: extract, transform, load, verify, visualize
├── index.html                       # Project page (GitHub Pages)
├── figures/                         # Charts from the analysis
└── README.md

The source CSV ("Weather in Szeged 2006–2016" on Kaggle) is not committed. Download it, save it
as `weatherHistory1.csv` and upload it to the notebook's working directory before running.
```

---

## Data Schema

**Source:** `weatherHistory1.csv` — historical weather readings with:

| Field | Type | Description |
|---|---|---|
| `Formatted Date` | datetime | Timestamp of reading |
| `Temperature (C)` | float | Air temperature in Celsius |
| `Apparent Temperature (C)` | float | Feels-like temperature |
| `Humidity` | float | Relative humidity (0–1) |
| `Wind Speed (km/h)` | float | Wind speed |
| `Precip Type` | string | Rain / Snow / None |
| `Summary` | string | Weather condition label |

**Engineered Features (added during transformation):**

| Feature | Logic |
|---|---|
| `Part_of_Day` | Morning (5–11), Afternoon (12–16), Evening (17–20), Night (21–4) |
| `Weekday` | Day of week extracted from datetime |
| `Hour` | Hour extracted from datetime |

**BigQuery Tables:**

| Table | Contents |
|---|---|
| `weather_cleaned` | Full transformed dataset |
| `daily_avg_temp` | Daily temperature aggregations |

---

## How to Run

### Option 1: Google Colab (Recommended)
Open the notebook directly:
[Open in Colab](https://colab.research.google.com/drive/19O1rZ1WewTlzk16MWzNl0k0uApxSMDyR?usp=sharing)

### Option 2: Run locally in Jupyter

You need Python 3.10+ and a Google Cloud project with the BigQuery API enabled.

```bash
git clone https://github.com/cece15/weather-etl-pipeline.git
cd weather-etl-pipeline
pip install pandas matplotlib seaborn google-cloud-bigquery pandas-gbq jupyter
export GOOGLE_APPLICATION_CREDENTIALS="path/to/your/service-account-key.json"
jupyter notebook "ETL_Pipeline_Project (1).ipynb"
```

The authentication cells use `google.colab.auth`; outside Colab, skip them and rely on
the credentials variable above.

---

## Row counts

| Stage | Rows |
|---|---|
| Extracted from CSV | 96,453 |
| After removing duplicates and missing values | 95,912 (541 removed) |
| Loaded to SQLite `weather_cleaned` | 95,912 |
| Loaded to BigQuery `weather_data.weather_cleaned` | 95,912 |

---

## Technical Challenges & Solutions

| Challenge | Solution |
|---|---|
| BigQuery rejects column names with spaces/special chars | Regex sanitization: `re.sub(r'[^a-zA-Z0-9_]', '_', name)` |
| Timestamps mix two UTC offsets (+01:00 winter, +02:00 summer) | Parse with `pd.to_datetime(..., utc=True)` and convert to `Europe/Budapest`. The first version used `errors='coerce'`, which silently emptied the 39,902 winter timestamps and labeled them all Night; this is now fixed |
| Secure GCP auth in Colab | `google.colab.auth.authenticate_user()` |
| Docker container specs insufficient for local Postgres | Pivoted to SQLite + BigQuery dual-target architecture |

---

## Insights from Analysis

Corrected results, after the time zone fix:

| Part of Day | Hours | Average temperature |
|---|---|---|
| Morning | 5–11 | 10.8 °C |
| Afternoon | 12–16 | 16.1 °C |
| Evening | 17–20 | 13.9 °C |
| Night | 21–4 | 9.4 °C |

- Monthly averages run from 0.8 °C in January to 23.0 °C in July.
- Wind is strongest from about 11:00 to 15:00, when humidity is lowest.
- Day of the week makes no meaningful difference to temperature.

The Looker Studio dashboard was built from the first run and still reflects the time zone error
until it is refreshed with the corrected data.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.10 |
| Data Processing | pandas, NumPy |
| Local Storage | SQLite (sqlite3) |
| Cloud Storage | Google BigQuery |
| Cloud Platform | Google Cloud Platform (GCP) |
| Visualization | Looker Studio, Matplotlib |
| Environment | Google Colab |
| Version Control | Git / GitHub |

---

## Author

**Cynthia Mutua** — MS Data Science & Analytics, Grand Valley State University

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat&logo=linkedin)](https://linkedin.com/in/cynthia-mutua)
[![GitHub](https://img.shields.io/badge/GitHub-cece15-181717?style=flat&logo=github)](https://github.com/cece15)
