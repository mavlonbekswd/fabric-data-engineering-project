# Fabric Data Engineering Project

\*\*Mavlonbek Sultonbekov —

An end-to-end data engineering pipeline built on Microsoft Fabric that ingests, transforms, models, and visualises real-world data from four independent sources — NYC Yellow Taxi trips, OpenAQ air quality, World Bank GDP, and ECB exchange rates — answering the question: _how does urban taxi mobility relate to air quality and macroeconomic conditions in New York City?_

---

## Architecture

The project follows the **medallion architecture** — a three-layer pattern used by data engineering teams at Databricks, Uber, and Microsoft.

```
Raw Sources → Bronze Lakehouse → Silver Lakehouse → Gold Warehouse → Power BI
```

| Layer  | Purpose                                    | Format             |
| ------ | ------------------------------------------ | ------------------ |
| Bronze | Raw data, exactly as received — no changes | Parquet, JSON, CSV |
| Silver | Cleaned, typed, validated Delta tables     | Delta Lake         |
| Gold   | Star schema aggregated for analytics       | SQL Warehouse      |

---

## Data Sources

| Dataset         | Source                   | Volume                | Purpose                          |
| --------------- | ------------------------ | --------------------- | -------------------------------- |
| NYC Yellow Taxi | NYC TLC / AWS CloudFront | 2.96M rows (Jan 2024) | Urban mobility patterns          |
| OpenAQ PM2.5    | OpenAQ v3 API            | 1,000 hourly readings | Air quality at CCNY station      |
| World Bank GDP  | World Bank API           | 10 years annual       | US macroeconomic context         |
| ECB FX Rates    | ECB Data API             | 7,054 daily rates     | USD/EUR conversion               |
| NYC Weather     | Open-Meteo Archive API   | 744 hourly records    | Temperature, precipitation, wind |

---

## Notebooks

### Notebook 1 — Bronze Ingestion

Ingests all five data sources into the Bronze Lakehouse using Python's `requests` library inside Fabric Notebooks.

Key discovery: Fabric's Copy Job cannot ingest Parquet files over HTTP because Parquet requires random file access. Solution was to download directly to `/lakehouse/default/Files/` using `requests.get()`.

Also handles:

- **Weather integration** — fetches hourly NYC weather from Open-Meteo (free, no key required) for January 2024, saves 744 records to Bronze, pushes to InfluxDB time-series database in line protocol format
- **Great Expectations validation** — validates `silver_taxi`, `silver_air_quality`, and `silver_weather` with null checks, range checks, row count checks, and date alignment checks
- **Discord bot trigger** — sends a formatted alert to a Discord webhook on validation failure or pass

### Notebook 3 — Silver Transformation

Reads raw Bronze files and produces clean, typed Delta tables in the Silver Lakehouse.

| Table              | Input Rows | Output Rows | Key Transformation                          |
| ------------------ | ---------- | ----------- | ------------------------------------------- |
| silver_taxi        | 2,964,624  | 2,723,805   | Remove nulls, invalid fares, zero distances |
| silver_air_quality | 1,000      | 1,000       | Flatten nested JSON, extract date/hour      |
| silver_gdp         | 10         | 10          | Cast types, filter nulls                    |
| silver_fx          | 7,054      | 6,992       | Select 2 columns from 32, cast types        |
| silver_weather     | 744        | 744         | Cast timestamp, extract date/hour           |

240,819 invalid taxi rows removed (8.1% of total) with documented business reasons for each filter rule.

---

## Gold Layer — Star Schema

All Silver tables are aggregated into a star schema inside **GoldWarehouse** using T-SQL CTAS statements. The schema reduces 2.7M individual taxi trips into 6,679 daily zone summaries — a 400x reduction with no loss of analytical granularity.

```
FactTaxiDaily ──── DimDate
FactAirQualityDaily ──── DimDate
DimFX
DimGDP
```

Power BI connects via DirectQuery — dashboards always show live data.

---

## Dashboards

Four Power BI dashboard pages:

**Mobility Dashboard** — Daily trip volumes and average fare across January 2024. Key finding: clear weekly pattern with weekday peaks of 100,000+ trips/day dropping to 60,000–70,000 on weekends. Average fare: $30.14.

**Air Quality Dashboard** — PM2.5 trend at CCNY station. Average 7.75 μg/m³, peak 19.13 μg/m³ — well below WHO danger threshold of 35 μg/m³.

**Mobility vs Air Quality** — Hourly correlation between taxi trip counts and PM2.5 levels in January 2024. Pearson correlation computed in PySpark across date+hour joins.

**Economic Impact** — US GDP trend (2015–2024, reaching $28.75T) alongside taxi fare data for macroeconomic context.

---

## Integrations

### Weather + InfluxDB + Grafana

- Open-Meteo archive API fetches hourly NYC weather (temperature, precipitation, wind speed)
- Data written to InfluxDB Cloud (time-series database) in line protocol format
- Grafana Cloud connects to InfluxDB for time-series weather dashboard

### Great Expectations + Discord Bot

- `great-expectations==0.18.15` validates all three Silver tables on each run
- Checks include: null values, value ranges, row counts, date alignment
- Discord webhook fires automatically — green card on pass, red card on failure
- The date alignment check specifically catches the 2016–2017 vs 2024 data mismatch bug

---

## Governance

- **Automated schedule** — Notebook 3 runs daily at 12:00 UTC via Fabric's built-in scheduler
- **Data lineage** — Full lineage automatically tracked in OneLake catalog from HTTP source → Bronze → Silver → Gold → Power BI
- **Failure alerts** — Email notification configured for schedule failures

---

## Key Results

| Metric                           | Value                  |
| -------------------------------- | ---------------------- |
| Total raw rows ingested          | 2,964,624              |
| Invalid rows removed             | 240,819 (8.1%)         |
| Clean rows in Silver             | 2,723,805              |
| Gold fact rows                   | 6,679 daily aggregates |
| Data reduction Silver→Gold       | 400x                   |
| Average NYC taxi fare (Jan 2024) | $30.14                 |
| Average PM2.5 (Jan 2024)         | 7.75 μg/m³             |
| USA GDP (2024)                   | $28.75 trillion        |

---

## Tech Stack

| Technology                 | Role                                            |
| -------------------------- | ----------------------------------------------- |
| Microsoft Fabric Lakehouse | Bronze and Silver storage (OneLake Delta)       |
| Microsoft Fabric Warehouse | Gold star schema with T-SQL engine              |
| PySpark (Fabric Notebooks) | Data transformation and cleaning                |
| Python requests            | API ingestion from all five sources             |
| Power BI                   | Four-page interactive dashboard                 |
| InfluxDB Cloud             | Time-series weather data storage                |
| Grafana Cloud              | Weather time-series visualisation               |
| Great Expectations         | Data quality validation framework               |
| Discord Webhook            | Automated quality alert bot                     |
| GitHub                     | Version control for notebooks and documentation |

---

## Challenges Solved

| Problem                                         | Root Cause                                          | Fix                                             |
| ----------------------------------------------- | --------------------------------------------------- | ----------------------------------------------- |
| Copy Job failed for Parquet over HTTP           | Parquet needs random file access, HTTP is streaming | Used `requests.get()` with OneLake path         |
| OpenAQ v2 API returned 410 Gone                 | API permanently deprecated                          | Switched to v3 with API key auth                |
| PATH_NOT_FOUND reading across lakehouses        | `Files/` path is relative to current lakehouse      | Used full ABFS path for cross-lakehouse reads   |
| DATENAME() unsupported in Fabric Warehouse      | Returns nvarchar, unsupported type                  | Replaced with DATEPART() returning integers     |
| Dashboard showing 2016–2017 air quality data    | Join not filtering to Jan 2024                      | Added explicit date filter + GE alignment check |
| Grafana Cloud can't connect to Fabric Warehouse | Fabric requires Azure AD auth, not SQL login        | Switched to InfluxDB as time-series target      |

---

_Microsoft Fabric Data Engineering Project — Mavlonbek Sultonbekov — 
