# A Data-Driven Cyber Risk Analytics and Decision-Support Framework for Digital Banking Environments

MSc Computer Science dissertation project — Gerta Rustemi, UNYT

A machine learning pipeline for network intrusion detection that supports regulatory-aligned cyber risk decision-making in digital banking, built in response to the operational resilience requirements of the EU's **Digital Operational Resilience Act (DORA)** and the risk management principles of **ISO 31000**.

Four models — **Random Forest**, **XGBoost**, **LSTM**, and **Isolation Forest** — are trained and evaluated on the **CICIDS2017** network intrusion dataset, explained using **SHAP**, and mapped to a four-tier (**Critical / High / Medium / Low**) risk decision framework.

---

## Overview

Digital banks face an expanding range of advanced cyber threats at the same time regulators are demanding a more scientific, auditable approach to operational risk. This project investigates whether supervised and unsupervised machine learning models can detect network intrusions with enough accuracy and explainability to support a regulation-aligned risk tiering system — one that could plausibly sit inside a bank's incident response and governance workflow.

**Research pipeline:**
1. Exploratory data analysis and preprocessing of raw network flow data
2. Training and evaluation of four ML architectures for intrusion detection
3. SHAP-based explainability analysis for the top-performing tree-based models
4. Mapping model predictions to a four-tier, regulation-aligned risk decision framework

---

## Dataset

**CICIDS2017** (Canadian Institute for Cybersecurity, 2017) — labelled network flow data covering benign traffic and multiple attack categories.

| Stage | Rows | Notes |
|---|---|---|
| Raw dataset | 1,048,575 | 79 columns |
| After cleaning | 975,899 | nulls, infinite values, and 72,246 duplicate rows removed |
| Train / test split | 780,719 / 195,180 | stratified 80/20 split |
| Training set after SMOTE | 5,836,260 | balanced across all 10 classes |

**Classes (10):** BENIGN, DDoS, DoS Hulk, DoS Slowloris, DoS Slowhttptest, FTP-Patator, SSH-Patator, Web Attack – Brute Force, Web Attack – XSS, Web Attack – SQL Injection

**Class imbalance:** ~74.8% benign / ~25.2% attack traffic overall, with severe under-representation of web attack classes (as few as 21 SQL Injection samples) — the primary motivation for SMOTE oversampling.

> The raw CSV is not included in this repository due to its size. Download the original CICIDS2017 dataset from the [Canadian Institute for Cybersecurity](https://www.unb.ca/cic/datasets/ids-2017.html) and place it locally before running Phase 1.

---

## Methodology

### Preprocessing (Phase 1)
- Label-encoding fixes for corrupted Web Attack category names (UTF-8/Latin-1 mismatch in source CSVs)
- Removal of null values, infinite values (`Flow Bytes/s`, `Flow Packets/s`), and duplicate rows
- `StandardScaler` feature normalisation
- Stratified 80/20 train/test split (SMOTE applied to training data only, to avoid leakage)
- **SMOTE** oversampling to balance all 10 classes
- **PCA** dimensionality reduction (79 → 16 components, retaining 95% of variance)
- **RFE** (Recursive Feature Elimination) to identify the top 20 most informative features

### Models (Phase 2)
| Model | Type | Input features |
|---|---|---|
| Random Forest | Supervised, bagging ensemble | 78 features (SMOTE-balanced) |
| XGBoost | Supervised, gradient boosting | 78 features (SMOTE-balanced) |
| LSTM | Supervised, recurrent neural network | 16 PCA components |
| Isolation Forest | Unsupervised, anomaly detection | 78 features (trained on benign traffic only) |

### Explainability
SHAP `TreeExplainer` was applied to the Random Forest and XGBoost models to identify the network flow attributes most responsible for predictions (e.g. Destination Port, `Init_Win_bytes_backward`, packet timing features), supporting the auditability requirements of DORA Article 13 and ISO 31000.

### Risk Tiering Framework
Model predictions are mapped to a four-tier risk score (0–100) using a CVSS-style severity weighting per attack category, translating raw classifier output into governance-facing decisions:

| Tier | Score range | Example response |
|---|---|---|
| Critical | 85–100 | Immediate escalation, isolate systems, notify CISO/regulators (DORA Art. 19) |
| High | 65–84 | SOC alert within 15 minutes, block source IP, forensic logging |
| Medium | 40–64 | Log and monitor, escalate if pattern persists |
| Low | 0–39 | Record in risk register, review in daily report |

---

## Results

| Model | Macro F1 | AUC-ROC | Mean FPR |
|---|---|---|---|
| **XGBoost** | **0.906** | **0.9999** | — |
| Random Forest | 0.881 | — | — |
| LSTM | 0.742 | — | — |
| Isolation Forest | 0.700 | — | 0.269 (default), reduced ~⅔ with tuned contamination |

XGBoost and Random Forest (tree-based ensembles) substantially outperformed LSTM and Isolation Forest on structured network flow data, particularly on rare, semantically overlapping web attack classes. A sensitivity sweep on Isolation Forest's contamination parameter showed its default 26.9% false positive rate was a hyperparameter artefact, not a ceiling — tuning reduced false positives by nearly two-thirds at the cost of recall.

Full per-model metrics, confusion matrices, and SHAP plots are generated in the Phase 2 notebook.

---

## Repository Structure

```
.
├── notebooks/
│   ├── Thesis_Phase1_EDA_Preprocessing.ipynb   # Data cleaning, SMOTE, PCA, RFE
│   └── Thesis_Phase2_Model_Training.ipynb      # Model training, evaluation, SHAP, risk framework
├── figures/                                     # Generated plots (gitignored — regenerate by running notebooks)
├── models/                                      # Trained model artefacts (gitignored — regenerate by running notebooks)
├── .gitignore
├── LICENSE
└── README.md
```

---

## How to Reproduce

Both notebooks were built for **Google Colab** and expect Google Drive paths (`/content/drive/MyDrive/Thesis/...`). To run locally, replace the Drive mount cells with local file paths.

1. Download the CICIDS2017 dataset and place it at the path referenced in Phase 1, Cell 2
2. Run `Thesis_Phase1_EDA_Preprocessing.ipynb` end-to-end — this saves preprocessed arrays and fitted objects (scaler, PCA, label encoder, etc.)
3. Run `Thesis_Phase2_Model_Training.ipynb` — this loads Phase 1 outputs, trains all four models, runs SHAP analysis, and applies the risk tiering framework

**Key dependencies:** `scikit-learn`, `xgboost`, `tensorflow`, `imbalanced-learn`, `shap`, `pandas`, `numpy`, `matplotlib`, `seaborn`

```bash
pip install scikit-learn xgboost tensorflow imbalanced-learn shap pandas numpy matplotlib seaborn joblib
```

> Random Forest and XGBoost train in ~15–40 minutes each on the full SMOTE-balanced set (~5.8M rows); LSTM requires a GPU runtime.

---

## Limitations

- **Dataset age:** CICIDS2017 reflects attack patterns from 2017 and may not capture more recent threat techniques
- **Class imbalance:** despite SMOTE, rare web attack categories (as few as 21 raw SQL Injection samples) remain challenging to detect reliably
- **Simulated risk weights:** the risk-scoring weights in the tiering framework are illustrative, based on CVSS-style severity reasoning, not calibrated against live incident data
- **Future work:** validation against live banking network telemetry is recommended before any operational deployment

---

## Keywords

cyber risk analytics · network intrusion detection · machine learning · XGBoost · Random Forest · LSTM · Isolation Forest · SHAP explainability · DORA · ISO 31000 · digital banking · decision-support framework

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.
