# Boston Parking Violations — Streamlit Demo Dashboard

This repository contains a small, interactive Streamlit dashboard built as an **academic exercise** to explore patterns in Boston parking-violation tickets.

## What this demo shows

- **Interactive filtering** (date range, violation categories, and optional “offender” drilldowns when enabled)
- A **map view** for spatial patterns (with an option to switch rendering modes)
- **Time-series trends** and a **weekday × hour heatmap** to reveal recurring enforcement/behavior patterns (e.g., commute-hour peaks)

## Data notes (important)

- This dashboard is **unofficial** and is **not** affiliated with the City of Boston.
- Any identifiers in the provided datasets are treated as **de-identified**.
- The dashboard is intended for **learning, exploratory analysis, and demo purposes** only.
- Do **not** use this tool to make decisions about individuals.

## Run locally (macOS/Linux)

From this folder:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
streamlit run app.py
```

Then open:

- http://localhost:8501

## Reproducibility

- Minimal dependencies are listed in [requirements.txt](requirements.txt).
- If you need a “known-working” set of exact versions, use [requirements-lock.txt](requirements-lock.txt) (generated from a working environment):

```bash
source .venv/bin/activate
pip install -r requirements-lock.txt
```

## Files

- [app.py](app.py): Streamlit application entry point
- [requirements.txt](requirements.txt): minimal runtime dependencies
- `images/`: assets used by the app (e.g., header)
- `*.parquet`, `*.csv`, `*.json`: local data artifacts used by the demo

## Demo talking points (for interviews)

- **Data management**: joined/filtered local datasets efficiently via DuckDB over Parquet.
- **Transparency**: clear labeling + a prominent “unofficial / academic exercise” disclaimer.
- **Interpretability**: adds an intuitive weekday/hour heatmap to make temporal patterns legible.
- **Responsible AI use**: demonstrates ability to iterate with AI assistance while validating outputs (e.g., compilation checks, query validation).
