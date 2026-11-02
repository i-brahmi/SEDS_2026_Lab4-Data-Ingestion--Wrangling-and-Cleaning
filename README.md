# Data Ingestion, Wrangling and Cleaning with Pandas

![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB?logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/pandas-data%20wrangling-150458?logo=pandas&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-imputation-F7931E?logo=scikitlearn&logoColor=white)
![Jupyter](https://img.shields.io/badge/Jupyter-notebook-F37626?logo=jupyter&logoColor=white)
![Status](https://img.shields.io/badge/status-completed-brightgreen)

Practical lab for the **Software Engineering for Data Science (SEDS)** course at the Higher School of Computer Science, Sidi Bel Abbes (ESI SBA). The notebook walks through the complete data preparation workflow on four real-world style datasets: ingestion from heterogeneous sources, exploratory data analysis (EDA), systematic data cleaning, and export of analysis-ready datasets.

---

## Table of Contents

- [Overview](#overview)
- [Challenges at a Glance](#challenges-at-a-glance)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Detailed Methodology](#detailed-methodology)
  - [Challenge 1: SQL Aggregation on the Chinook Database](#challenge-1-sql-aggregation-on-the-chinook-database)
  - [Challenge 2: iTunes Multi-Source Integration](#challenge-2-itunes-multi-source-integration)
  - [Challenge 3: Oil and Gas Production Dataset](#challenge-3-oil-and-gas-production-dataset)
  - [Challenge 4: Bitcoin Price Data](#challenge-4-bitcoin-price-data)
- [Outputs](#outputs)
- [Design Decisions and Known Limitations](#design-decisions-and-known-limitations)
- [Tech Stack](#tech-stack)
- [Author and Acknowledgements](#author-and-acknowledgements)
- [License](#license)

---

## Overview

Raw data is rarely ready for modelling. This project applies the standard data cleaning checklist taught in the course to produce clean datasets that can later feed machine learning models (recommendation systems, production forecasting, and price forecasting).

The workflow applied in every challenge:

1. **Ingestion**: load data from SQLite, CSV, Excel, JSON and a live REST-backed API (Yahoo Finance).
2. **Exploratory Data Analysis**: inspect shape, datatypes, missing values, descriptive statistics, correlations and distributions.
3. **Cleaning**: remove irrelevant data, handle missing values, treat outliers, drop duplicates, fix datatypes and standardize formats.
4. **Validation**: assert data integrity before saving.
5. **Export**: write the cleaned dataset to CSV or Excel.

---

## Challenges at a Glance

| # | Challenge | Sources | Key techniques | Output |
|---|-----------|---------|----------------|--------|
| 1 | Longest average song length per genre | SQLite (`chinook.db`) | SQL joins, `AVG`, `GROUP BY`, `pd.read_sql_query` | In-notebook result |
| 2 | iTunes song catalogue | SQLite + CSV + Excel | `pd.concat`, mode and median imputation, IQR outlier removal | `data/itunes_clean.xlsx` |
| 3 | Oil and gas daily production | Semicolon-separated CSV (Latin-1) | Unit stripping, date parsing, range validation, IQR + KNN imputation | `data/cleaned_oil_gas_data.csv` |
| 4 | Bitcoin (BTCUSD) price history | JSON file + Yahoo Finance API | Schema alignment, time-series reindexing and interpolation | `data/cleaned_bitcoin_price.csv` |

---

## Repository Structure

```
SEDS_2026_Lab4/
|-- data/
|   |-- chinook.db                    # SQLite database (Challenges 1 and 2)
|   |-- chinook_data.csv              # iTunes data, CSV source (Challenge 2)
|   |-- chinook_data.xlsx             # iTunes data, Excel source (Challenge 2)
|   |-- itunes_data.csv               # Reference iTunes data
|   |-- itunes_clean.xlsx             # Cleaned iTunes dataset (output)
|   |-- oil_gas_data.csv              # Raw oil and gas dataset (Challenge 3)
|   |-- cleaned_oil_gas_data.csv      # Cleaned oil and gas dataset (output)
|   |-- bitcoin_price.json            # Raw Bitcoin history (Challenge 4)
|   `-- cleaned_bitcoin_price.csv     # Cleaned Bitcoin dataset (output)
|-- figures/                          # Images used by the notebook
|-- SEDS_2026_Lab4.ipynb              # Main notebook with all solutions
|-- SEDS_2026_Lecture 04.pdf          # Course lecture (reference material)
`-- README.md
```

---

## Getting Started

### Prerequisites

- Python 3.9 or later
- Jupyter Notebook, JupyterLab, or VS Code with the Jupyter extension
- Internet access for Challenge 4 (Yahoo Finance download)

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/<your-username>/<your-repository>.git
cd <your-repository>

# 2. (Recommended) create an isolated environment
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install pandas numpy matplotlib scikit-learn yfinance openpyxl jupyter
```

With conda:

```bash
conda install pandas numpy matplotlib scikit-learn openpyxl jupyter
pip install yfinance
```

### Run

```bash
jupyter notebook SEDS_2026_Lab4.ipynb
```

Execute the cells in order. Each challenge is self-contained, and every solution cell starts with its own imports. All file paths are relative to the repository root, so launch Jupyter from there.

---

## Detailed Methodology

### Challenge 1: SQL Aggregation on the Chinook Database

**Goal:** find the genres with the longest average track length.

- Connects to `data/chinook.db` with `sqlite3`.
- Joins `genres` and `tracks`, computes `AVG(Milliseconds)` per genre, converts it to minutes, and sorts in descending order.
- Loads the query result directly into a DataFrame with `pd.read_sql_query`.

### Challenge 2: iTunes Multi-Source Integration

**Goal:** build one clean catalogue from three different sources for a future song recommendation model.

**Ingestion**

- `pd.read_csv` for the CSV, `pd.read_excel` for the workbook, and a four-table SQL join (`tracks`, `genres`, `albums`, `artists`) for the database.
- The three frames are combined with `pd.concat`.

**EDA**

- `head`, `tail`, `shape`, `info`, `isna().sum()`, `describe`, `value_counts`, correlation matrix, a histogram of track duration, a duration versus size scatter plot, and a bar chart of genre frequency.

**Cleaning**

| Step | Action |
|------|--------|
| Irrelevant data | Dropped `Composer` (many missing values); removed non-music genres (Drama, TV Shows, Sci Fi & Fantasy, Science Fiction, Comedy) |
| Duplicates | Detected with `duplicated()` and removed with `drop_duplicates()` |
| Missing values | `UnitPrice` filled with the mode; `Milliseconds` and `Bytes` filled with the median |
| Outliers | IQR rule (1.5 x IQR) applied to `Milliseconds` |
| Datatypes | `Milliseconds` and `Bytes` cast to `int` |
| Standardization | Whitespace stripped from text columns; `Seconds` feature derived from `Milliseconds` |

**Export:** `data/itunes_clean.xlsx`

### Challenge 3: Oil and Gas Production Dataset

**Goal:** clean a daily production dataset (Hassi Messaoud complex) to be used for time series forecasting models.

**Data quality issues found**

| Issue | Treatment |
|-------|-----------|
| File is `;` separated and Latin-1 encoded | `read_csv(sep=';', encoding='latin-1')` |
| Numeric columns stored as text with units (`'8352.41 feet'`, `'135.24 °C'`) | Regex extraction of the leading number, cast to `float` |
| `date` stored as `dd/mm/yyyy` text | `pd.to_datetime(format='%d/%m/%Y')`, sorted chronologically |
| `well_id` has a single constant value | Dropped as irrelevant |
| `drilling_success` contains the invalid value `2` | Set to `NaN`, then filled with the mode |
| `daily_oil_production` has missing values | Imputed with `KNNImputer` |
| `daily_gas_production` has extreme spikes | Detected with the IQR rule, set to `NaN`, then imputed |
| Duplicates | None found |

**Why KNN imputation:** oil production is almost perfectly correlated with drilling depth and gas production is strongly correlated with it, so a simple mean would ignore real structure in the data. Features are standardized before KNN because the algorithm is distance based.

**Why outliers are not dropped:** the dataset is a continuous daily series for forecasting. Removing rows would create gaps in time, so outliers are re-estimated instead.

**Validation:** assertions confirm no missing values remain and `drilling_success` is strictly binary.

**Export:** `data/cleaned_oil_gas_data.csv`

### Challenge 4: Bitcoin Price Data

**Goal:** combine a historical JSON file with live Yahoo Finance data into a single, clean BTCUSD daily dataset.

**Ingestion**

- JSON history loaded with `json.load` and `DataFrame.from_dict`.
- Data from 2020-11-28 to 2022-10-31 downloaded with `yfinance` (`auto_adjust=False`). The `end` parameter of `yfinance` is exclusive, so `2022-11-01` is used to include the last day.

**Schema alignment**

- JSON `time` column renamed to `Date`, converted from epoch milliseconds, and set as the index.
- Yahoo columns flattened (recent `yfinance` versions return a MultiIndex) and lowercased to match the JSON schema.

**EDA**

- Time series plots of `open`, `close`, `high`, `low` and `volume`, descriptive statistics, and a correlation matrix.

**Cleaning**

| Issue | Treatment |
|-------|-----------|
| Overlapping date: the JSON file actually ends on 2020-11-28, the day the Yahoo data starts | Duplicate index removed, keeping the Yahoo row |
| `adj close` is missing for the whole JSON part and not needed | Column dropped |
| `symbol` is constant (`btcusd`) and missing for the Yahoo part | Column dropped |
| Impossible `low` values in early 2013 (for example `0.01` while the price is about 84 USD) | Replaced with `min(open, close)` |
| Missing calendar days in the JSON history | Reindexed to a continuous daily range and interpolated in time |

**Export:** `data/cleaned_bitcoin_price.csv` (daily index from 2013-03-31 to 2022-10-31, columns `open, close, high, low, volume`).

---

## Outputs

| File | Description | Shape |
|------|-------------|-------|
| `data/itunes_clean.xlsx` | Clean music-only song catalogue | Generated by the notebook |
| `data/cleaned_oil_gas_data.csv` | Clean daily production series | 1,064 rows x 7 columns |
| `data/cleaned_bitcoin_price.csv` | Clean BTCUSD daily OHLCV series | About 3,500 rows x 5 columns |

---

## Design Decisions and Known Limitations

- **Outliers on time series are imputed, not dropped.** Dropping rows from a daily series breaks the time axis. For the iTunes catalogue, which is not a time series, rows are removed as taught in the lecture.
- **Mixed volume units in the Bitcoin dataset.** The historical JSON volume appears to be expressed in BTC, whereas Yahoo Finance reports volume in USD. The two parts are concatenated as-is, which affects the `volume` plot and its correlations. A possible harmonization is to divide the Yahoo volume by the closing price to convert it to BTC.
- **Live data is not frozen.** The Yahoo Finance download depends on the provider at run time, so minor differences between runs are possible.
- **Imputation is an estimate.** KNN-imputed and interpolated values are model-based approximations and should be treated as such in downstream forecasting.

---

## Tech Stack

| Area | Tools |
|------|-------|
| Language | Python |
| Data wrangling | pandas, NumPy |
| Databases | SQLite (`sqlite3`) |
| Machine learning utilities | scikit-learn (`KNNImputer`, `StandardScaler`) |
| Visualization | Matplotlib |
| Market data | yfinance |
| File formats | CSV, Excel (`openpyxl`), JSON, SQLite |
| Environment | Jupyter Notebook |

---

## Author and Acknowledgements

**Author:** Your Name ([github.com/your-username](https://github.com/your-username))

**Course:** Software Engineering for Data Science (SEDS), Higher School of Computer Science, 08 May 1945 of Sidi Bel Abbes (ESI SBA).

**Instructor:** Dr. Belkacem Khaldi. The lab statements and lecture material are the property of the instructor and the institution; the solutions in this repository are the author's own work.
