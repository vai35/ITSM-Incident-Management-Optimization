# 🖥️ ITSM Incident Management — ML-Driven Optimization

> An end-to-end Machine Learning project solving **4 real-world ITSM use cases** for ABC Tech using 46K IT incidents extracted from a MySQL database — covering priority prediction, time-series forecasting, auto-tagging, and asset risk scoring.

---

## 📌 Table of Contents

- [Business Problem](#business-problem)
- [Dataset](#dataset)
- [Project Pipeline](#project-pipeline)
- [Use Cases Overview](#use-cases-overview)
- [UC-1: High Priority Ticket Prediction](#uc-1-high-priority-ticket-prediction)
- [UC-2: Incident Volume Forecasting](#uc-2-incident-volume-forecasting)
- [UC-3: Auto-Tagging & Department Routing](#uc-3-auto-tagging--department-routing)
- [UC-4: RFC Prediction & Asset Risk Scoring](#uc-4-rfc-prediction--asset-risk-scoring)
- [Data Preprocessing](#data-preprocessing)
- [Results Summary](#results-summary)
- [Installation](#installation)
- [Project Structure](#project-structure)
- [Conclusion](#conclusion)

---

## 🧩 Business Problem

**ABC Tech** is a mid-size IT-enabled services organization receiving **22–25K IT incidents per year**. Despite following ITIL best practices, a recent customer survey rated incident management as **poor**. Management identified 4 areas where ML could drive measurable improvement:

| # | Use Case | Business Goal |
|---|----------|--------------|
| 1 | 🚨 High Priority Prediction | Detect P1/P2 tickets early for preventive action |
| 2 | 📈 Incident Volume Forecasting | Forecast quarterly/annual ticket load for resource planning |
| 3 | 🏷️ Auto-Tagging & Routing | Route tickets to correct departments, reduce reassignments |
| 4 | 🔧 RFC & Asset Risk Prediction | Predict change requests and flag misconfiguration-prone assets |

---

## 📂 Dataset

| Property | Value |
|----------|-------|
| Source | MySQL Database (live extraction via SQLAlchemy) |
| Total Records | 46,606 |
| Time Period | 2012 – 2014 |
| Total Columns (raw) | 25 |
| Total Columns (after FE) | 32 |
| Target Variables | Priority, WBS, Has_Related_Change |

### Key Columns

| Column | Type | Description |
|--------|------|-------------|
| `CI_Name` | Categorical | Configuration Item name |
| `CI_Cat` / `CI_Subcat` | Categorical | Category and subcategory of CI |
| `Impact` / `Urgency` / `Priority` | Numeric | ITIL priority matrix inputs |
| `Open_Time` / `Close_Time` | Datetime | Ticket lifecycle timestamps |
| `Handle_Time_hrs` | Numeric | Time to resolve the incident |
| `No_of_Reassignments` | Numeric | Count of department reassignments |
| `WBS` | Categorical | Department/work breakdown code |
| `Alert_Status` | Categorical | System alert state |

### ITIL Priority Matrix

| | Urgency 1 | Urgency 2 | Urgency 3 | Urgency 4 | Urgency 5 |
|---|-----------|-----------|-----------|-----------|-----------|
| **Impact 1** | P1 | P2 | P3 | P3 | P3 |
| **Impact 2** | P2 | P2 | P2 | P3 | P3 |
| **Impact 3** | P2 | P2 | P3 | P3 | P4 |
| **Impact 4** | P3 | P3 | P3 | P4 | P4 |
| **Impact 5** | P3 | P3 | P4 | P4 | P5 |

---

## 🔄 Project Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                        END-TO-END PIPELINE                           │
└──────────────────────────────────────────────────────────────────────┘

  ┌─────────────────┐
  │  MySQL Database │
  │  (46,606 rows)  │
  └────────┬────────┘
           │ SQLAlchemy + mysql-connector
           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                     DATA PREPROCESSING                          │
  │                                                                 │
  │  • Handle '', #N/B  →  NaN                                      │
  │  • Handle #MULTIVALUE  →  Binary indicator columns              │
  │  • Parse Handle_Time_hrs  →  comma-decimal fix + aggregation    │
  │  • Type casting  →  int/float/datetime                          │
  │  • Null imputation  →  0 for counts, 'Unknown' for categoricals │
  │  • Feature engineering  →  Is_Resolved, Was_Reopened, etc.      │
  └────────────────────────────┬────────────────────────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
            ▼                  ▼                  ▼
     ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐
     │   UC-1      │   │   UC-2      │   │   UC-3 & UC-4   │
     │  Priority   │   │  Forecast   │   │  Auto-Tag &     │
     │  Prediction │   │  ARIMA      │   │  RFC Predict    │
     └──────┬──────┘   └──────┬──────┘   └────────┬────────┘
            │                 │                   │
            ▼                 ▼                   ▼
     CatBoost +         ARIMA(1,1,1)        CatBoost
     Threshold           + Moving            Multi-class
     Tuning              Average             + Risk Score
```

---

## 📋 Use Cases Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        4 USE CASES AT A GLANCE                      │
├──────────┬──────────────────────┬──────────────┬────────────────────┤
│ Use Case │ Problem Type         │ Model        │ Key Metric         │
├──────────┼──────────────────────┼──────────────┼────────────────────┤
│  UC-1    │ Binary Classification│ CatBoost     │ ROC-AUC: 0.893     │
│  UC-2    │ Time Series Forecast │ ARIMA(1,1,1) │ MAE: ~746          │
│  UC-3    │ Multi-class Classify │ CatBoost     │ Top-3 Acc: 96.4%   │
│  UC-4A   │ Binary Classification│ CatBoost     │ ROC-AUC: 0.744     │
│  UC-4B   │ Risk Scoring         │ Weighted Agg │ Asset Risk Index   │
└──────────┴──────────────────────┴──────────────┴────────────────────┘
```

---

## 🚨 UC-1: High Priority Ticket Prediction

**Goal:** Predict Priority 1 & 2 tickets early so the team can take preventive action before the issue escalates.

### Class Distribution

| Class | Label | Count | % |
|-------|-------|-------|---|
| Normal Priority (P3/P4/P5) | 0 | 44,526 | 98.5% |
| **High Priority (P1/P2)** | **1** | **700** | **1.5%** |

> ⚠️ Severe class imbalance — accuracy alone is misleading. ROC-AUC and Recall are the key metrics.

### Feature Selection & Leakage Handling

```
Initial Features (caused data leakage):
┌──────────────────────────────────────────────────────────┐
│  Impact ──┐                                               │
│  Urgency ─┼──▶ Directly define Priority via ITIL matrix  │
│           │    → ROC = 1.0 (Leakage!)                    │
│  No_of_Reassignments ──┐                                  │
│  No_of_Related_Changes─┼──▶ Post-assignment features      │
│  Handle_Time_is_multi ─┘    → Also leakage               │
└──────────────────────────────────────────────────────────┘

Final Clean Features (available at ticket creation time):
  CI_Cat | CI_Subcat | Category | Alert_Status
```

### Model Results (After Leakage Removal)

| Model | ROC-AUC | Accuracy | Recall (P1/P2) | Notes |
|-------|---------|----------|----------------|-------|
| Logistic Regression | 0.857 | 98% | 0% | Fails minority class |
| **CatBoost** | **0.893** | **95%** | **73%** | Best balance |

### Threshold Tuning

| Threshold | Precision (P1/P2) | Recall (P1/P2) | Accuracy |
|-----------|------------------|----------------|----------|
| 0.10 | 0.020 | 0.993 | 26.4% |
| 0.20 | 0.025 | 0.950 | 42.3% |
| 0.35 | 0.083 | 0.786 | 86.2% |
| **0.40** ✅ | **0.084** | **0.786** | **86.9%** |
| 0.50 | 0.200 | 0.730 | 95.0% |

> ✅ **Chosen threshold: 0.4** — detects ~1 real critical alert per 12 flagged (practical for ops teams)

### Feature Importance (SHAP)

| Feature | Importance (%) | Interpretation |
|---------|---------------|----------------|
| `CI_Subcat` | 52.75% | Subcategory is the strongest risk signal |
| `CI_Cat` | 27.32% | Category adds significant context |
| `Category` | 19.93% | Incident type matters |
| `Alert_Status` | 0.00% | Negligible when other features present |

---

## 📈 UC-2: Incident Volume Forecasting

**Goal:** Forecast monthly incident volumes to enable quarterly and annual resource & infrastructure planning.

### Data Preparation Flow

```
Raw Timestamps (Open_Time)
         │
         ▼
Monthly Aggregation
         │
         ▼
┌────────────────────────────────┐
│  Full Dataset (2012–2014)      │
│  → Spike detected in Sep 2013  │
│  → Pre-spike data distorts     │
│     forecast                   │
└────────────┬───────────────────┘
             │
             ▼
Post-Stabilization Dataset
(Sep 2013 onwards)
             │
     ┌───────┴────────┐
     │                │
     ▼                ▼
3-Month Moving    ARIMA(1,1,1)
Average (Baseline) (Main Model)
```

### Model Configuration

| Parameter | Value |
|-----------|-------|
| Model | ARIMA |
| Order (p,d,q) | (1, 1, 1) |
| Train Set | All months except last 3 |
| Test Set | Last 3 months |
| Evaluation Metric | MAE |

### Forecast Results

| Model | MAE | Notes |
|-------|-----|-------|
| 3-Month Moving Average | — | Baseline, trend smoothing only |
| **ARIMA (Post-Stabilization)** | **~746 incidents** | Best forecast |
| ARIMA (Full Data) | Higher error | Distorted by pre-spike data |

> 📌 Monthly forecasts are aggregated to derive **quarterly** and **annual** estimates for planning.

---

## 🏷️ UC-3: Auto-Tagging & Department Routing

**Goal:** Automatically route tickets to the correct WBS department code to reduce manual reassignments and resolution delays.

### Target Variable

- **`WBS`** — Department code (273 unique departments in raw data)
- Filtered to departments with ≥ 500 tickets → **top departments with sufficient data**

### Model Flow

```
Input Features:
  CI_Cat | CI_Subcat | Category | Impact | Urgency | Alert_Status
                          │
                          ▼
               Label Encoding (categorical)
                          │
                          ▼
           CatBoostClassifier (MultiClass)
           iterations=500 | depth=8 | lr=0.1
                          │
                ┌─────────┴──────────┐
                ▼                    ▼
          Top-1 Prediction     Top-3 Predictions
          (Direct routing)     (Suggested routing)
```

### Results by Department Filter

| Min Tickets per Dept | Departments | Top-1 Accuracy | Top-3 Accuracy |
|---------------------|-------------|----------------|----------------|
| ≥ 200 tickets | 38 depts | 64.9% | 88.4% |
| **≥ 500 tickets** | **Fewer, high-volume** | **75.3%** | **96.4%** ✅ |

### Business Impact Analysis

| Routing Outcome | Avg. Reassignments | Avg. Handle Time (hrs) |
|----------------|-------------------|----------------------|
| ✅ Correctly Routed | **1.00** | **427.6** |
| ❌ Incorrectly Routed | 1.77 | 506.5 |
| **Improvement** | **↓ 43%** | **↓ 15.6%** |

> 💡 **Interpretation:** Top-3 accuracy of 96.4% means the correct department appears in the model's top 3 suggestions for 96 out of 100 tickets — highly practical for a dropdown-based routing UI.

---

## 🔧 UC-4: RFC Prediction & Asset Risk Scoring

### UC-4A: RFC (Request for Change) Prediction

**Goal:** Predict whether an incident will trigger a Request for Change, enabling proactive change management.

#### Class Distribution

| Class | Count | % |
|-------|-------|---|
| No RFC | 46,041 | 98.8% |
| **Has RFC** | **536** | **1.1%** |

#### Model Results

| Model | ROC-AUC | Recall (RFC) | Notes |
|-------|---------|--------------|-------|
| Logistic Regression | 0.634 | 69% | Weak discrimination |
| **CatBoost** | **0.744** | **62%** | Better overall |

#### Threshold Tuning (CatBoost)

| Threshold | Recall (RFC) | Precision (RFC) | Accuracy |
|-----------|-------------|-----------------|----------|
| 0.05 | 0.96 | 0.013 | 17% |
| 0.20 | 0.93 | 0.015 | 34% |
| **0.40** ✅ | **0.91** | **0.025** | **71%** |
| 0.70 | 0.60 | 0.050 | 93% |

> ✅ **Threshold 0.4 chosen** — detects 91% of RFC cases, balancing early detection with operational feasibility.

---

### UC-4B: Asset Risk Scoring (Misconfiguration Detection)

**Goal:** Score each ITSM asset (CI_Name) by its historical risk profile to flag those most prone to failure or misconfiguration.

#### Risk Score Formula

```
Risk_Score = 0.35 × Incident_Count (normalized)
           + 0.35 × RFC_Count (normalized)
           + 0.20 × Avg_Handle_Time (normalized)
           + 0.10 × Avg_Reassignments (normalized)
```

#### Feature Weights Rationale

| Feature | Weight | Reasoning |
|---------|--------|-----------|
| `Incident_Count` | 35% | High frequency = structurally unstable |
| `RFC_Count` | 35% | Many changes = misconfiguration-prone |
| `Avg_Handle_Time` | 20% | Long resolution = complex/poorly documented |
| `Avg_Reassignments` | 10% | Many reassignments = unclear ownership |

> 🔧 Weights are configurable based on business priorities.

---

## 🧹 Data Preprocessing

### Anomaly Handling Strategy

| Anomaly Type | Handling Strategy |
|-------------|------------------|
| `''` (empty string) | → `NaN` (field unfilled) |
| `#N/B` | → `NaN` (not applicable) |
| `#MULTIVALUE` | → Binary indicator column created, then `NaN` |
| `Handle_Time_hrs` (comma-decimal) | → Custom parser: single comma = decimal point |
| `Handle_Time_hrs` (multi-value) | → Aggregated (sum) + binary flag column |

### Null Imputation Strategy

| Column Type | Strategy | Reasoning |
|-------------|----------|-----------|
| Count columns (`No_of_Reassignments`, etc.) | Fill with `0` | Null = no event occurred |
| Categorical (`CI_Cat`, `Closure_Code`) | Fill with `'Unknown'` | Preserve row, flag as missing |
| `Handle_Time_hrs` | Fill with `0` + missing flag | Preserve information |
| `Reopen_Time`, `Resolved_Time` | Keep `NaN` | Valid business states |

### Engineered Features

| Feature | Derived From | Description |
|---------|-------------|-------------|
| `Is_Resolved` | `Resolved_Time` | 1 if ticket was resolved |
| `Was_Reopened` | `Reopen_Time` | 1 if ticket was reopened |
| `Is_Closed` | `Close_Time` | 1 if ticket was closed |
| `Has_Related_Interaction` | `Related_Interaction` | 1 if interaction exists |
| `Has_Related_Change` | `Related_Change` | 1 if change request linked |
| `Related_Interaction_Is_Multi` | `#MULTIVALUE` flag | Multiple interactions present |
| `Related_Change_Is_Multi` | `#MULTIVALUE` flag | Multiple changes present |
| `Handle_Time_is_multi` | Comma count > 1 | Multi-entry handling time |
| `Handle_Time_hrs_missing` | Null flag | Handle time was missing |

---

## 📊 Results Summary

| Use Case | Model | Key Metric | Value |
|----------|-------|------------|-------|
| UC-1: Priority Prediction | CatBoost (threshold=0.4) | ROC-AUC | **0.893** |
| UC-1: Priority Prediction | CatBoost | Recall (P1/P2) | **78.6%** |
| UC-2: Volume Forecasting | ARIMA(1,1,1) | MAE | **~746 incidents** |
| UC-3: Auto-Routing | CatBoost MultiClass | Top-1 Accuracy | **75.3%** |
| UC-3: Auto-Routing | CatBoost MultiClass | Top-3 Accuracy | **96.4%** |
| UC-3: Business Impact | — | Reassignment Reduction | **↓ 43%** |
| UC-3: Business Impact | — | Handle Time Reduction | **↓ 15.6%** |
| UC-4A: RFC Prediction | CatBoost (threshold=0.4) | ROC-AUC | **0.744** |
| UC-4A: RFC Prediction | CatBoost | Recall (RFC) | **91%** |
| UC-4B: Asset Risk | Weighted Score | Top Risk Assets Flagged | ✅ |

---

## ⚙️ Installation

```bash
# Clone the repository
git clone https://github.com/vai35/itsm-ml-optimization.git
cd itsm-ml-optimization

# Install dependencies
pip install pandas numpy matplotlib seaborn scikit-learn catboost statsmodels sqlalchemy mysql-connector-python shap
```

### Requirements

| Library | Purpose |
|---------|---------|
| `pandas`, `numpy` | Data manipulation |
| `scikit-learn` | ML models, preprocessing, metrics |
| `catboost` | Gradient boosting classifier |
| `statsmodels` | ARIMA time-series forecasting |
| `sqlalchemy` + `mysql-connector-python` | MySQL data extraction |
| `shap` | Feature importance (SHAP values) |
| `matplotlib`, `seaborn` | Visualization |

---

## 📁 Project Structure

```
itsm-ml-optimization/
│
├── 📓 PR_0012.ipynb                  # Main notebook (all 4 use cases)
├── 📄 README.md                      # Project documentation
├── 📄 PRCL-0012.pdf                  # Problem statement (DataMites)
│
├── 📂 data/
│   ├── raw_data.csv                  # Extracted from MySQL
│   └── cleaned_df.csv                # Post-preprocessing dataset
│
└── 📂 outputs/
    ├── uc1_priority_model.pkl        # Saved CatBoost model (UC-1)
    ├── uc3_routing_model.pkl         # Saved CatBoost model (UC-3)
    └── uc4_asset_risk_scores.csv     # Asset risk rankings (UC-4B)
```

---

## 🔮 Future Scope

- 🔤 **NLP on ticket descriptions** — Use text embeddings to improve all classifiers
- ⚡ **Real-time deployment** — Integrate models into ITSM tools (ServiceNow, Jira)
- 🔁 **Continuous learning** — Retrain models as new tickets are resolved
- 📊 **Dynamic thresholds** — Auto-adjust classification thresholds based on ops load
- 🗺️ **Grad-CAM equivalent for tabular** — Deeper SHAP-based explainability dashboard

---

## 👤 Author

**[Vaishnavi Shidling]**
- 🔗 LinkedIn: [linkedin.com/in/vaishnavi-shidling/]
- 💻 GitHub: [https://github.com/vai35]
- 📧 Email: [vaishnavishidling74@gmail.com]

---

*Built as part of the DataMites Capstone Project — PR-0012 | Client: ABC Tech | Category: ITSM - ML*
