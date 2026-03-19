# Weather Data ETL Pipeline

![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat&logo=python&logoColor=white)
![Google BigQuery](https://img.shields.io/badge/Google%20BigQuery-4285F4?style=flat&logo=googlebigquery&logoColor=white)
![GCP](https://img.shields.io/badge/Google%20Cloud-4285F4?style=flat&logo=googlecloud&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-003B57?style=flat&logo=sqlite&logoColor=white)
![Looker Studio](https://img.shields.io/badge/Looker%20Studio-4285F4?style=flat&logo=looker&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-150458?style=flat&logo=pandas&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-green?style=flat)

A production-pattern ETL pipeline that extracts historical weather data from CSV, applies multi-step transformations including feature engineering and schema normalization, and loads into both a local SQLite database and Google BigQuery on GCP, with interactive dashboards in Looker Studio.

---

## Dashboard Preview

>  [View Live Looker Studio Dashboard](https://lookerstudio.google.com/reporting/7f2762fb-3187-4767-ad7e-ee9708269c04)
> 
>  [View Notebook on Google Colab](https://colab.research.google.com/drive/19O1rZ1WewTlzk16MWzNl0k0uApxSMDyR?usp=sharing)

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

- **Dual-target loading** — data simultaneously persisted to local SQLite and Google BigQuery
- **Production-style modular code** — each ETL phase is an isolated function, orchestrated by `run_pipeline()`
- **BigQuery schema compatibility** — regex-based column sanitization handles special characters, spaces, and reserved keywords
- **Structured logging** — INFO / WARNING / ERROR levels written to both console and `etl_pipeline.log`
- **Comprehensive error handling** — try/except blocks at every stage with row-count validation and table existence checks
- **Feature engineering** — derives `Part of Day`, `Weekday`, and `Hour` from raw timestamps
- **Cloud authentication** — secure GCP OAuth via Google Colab's built-in auth flow
- **Version controlled** — full pipeline on GitHub with clean commit history

---

## Repository Structure

```
weather-etl-pipeline/
│
├── etl_pipeline.py          # Main pipeline: extract, transform, load, orchestrate
├── requirements.txt         # Python dependencies
├── data/
│   └── weatherHistory1.csv  # Source dataset (96K+ rows)
├── outputs/
│   └── etl_pipeline.log     # Auto-generated run log
└── README.md
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

### Option 2: Local Setup

**Prerequisites**
```bash
Python 3.10+
A Google Cloud project with BigQuery API enabled
```

**Install dependencies**
```bash
git clone https://github.com/cece15/weather-etl-pipeline.git
cd weather-etl-pipeline
pip install -r requirements.txt
```

**Configure GCP credentials**
```bash
export GOOGLE_APPLICATION_CREDENTIALS="path/to/your/service-account-key.json"
```

**Run the pipeline**
```bash
python etl_pipeline.py
```

---

## Sample Output

```
[INFO]  Starting ETL pipeline...
[INFO]  Extracted 96,453 rows from weatherHistory1.csv
[INFO]  Columns validated: all required fields present
[INFO]  Transformation complete: 96,453 → 95,812 rows (641 duplicates/nulls removed)
[INFO]  Feature engineering complete: Part_of_Day, Weekday, Hour added
[INFO]  SQLite load verified: 95,812 rows in weather_cleaned
[INFO]  BigQuery upload complete: weather_data.weather_cleaned
[INFO]  Pipeline completed successfully in 47.3s
```

---

## Technical Challenges & Solutions

| Challenge | Solution |
|---|---|
| BigQuery rejects column names with spaces/special chars | Regex sanitization: `re.sub(r'[^a-zA-Z0-9_]', '_', name)` |
| Datetime strings needed for temporal analysis | `pd.to_datetime()` with `errors='coerce'` for safe conversion |
| Secure GCP auth in Colab | `google.colab.auth.authenticate_user()` |
| Docker container specs insufficient for local Postgres | Pivoted to SQLite + BigQuery dual-target architecture |

---

## Insights from Analysis

- **Night** records the highest cumulative temperature — consistent with sustained heat retention
- **Wind speed peaks between 2–6 PM** — aligns with thermal mixing from surface heating
- **Inverse relationship** observed between afternoon humidity and wind speed — typical diurnal pattern

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
