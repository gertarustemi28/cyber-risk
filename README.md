# Data-Driven Cyber Risk Analytics and Decision Support Framework for Digital Banking Environments

## Overview

Digital banks are under growing pressure from both cyber threats and regulation. The EU's Digital Operational Resilience Act (DORA) and ISO 31000 require financial institutions to run ICT risk management that is systematic, evidence-based, and auditable — yet most machine learning intrusion detection work stops at reporting accuracy, without explaining *why* a model flagged a flow or *what a bank should actually do about it*.

This project closes that gap. It builds and evaluates a full pipeline that:

1. Detects and classifies network intrusions using four ML architectures spanning supervised, sequential, and unsupervised paradigms.
2. Explains model predictions with SHAP, satisfying the transparency requirements of DORA Article 13.
3. Maps model outputs onto a four-tier (Critical / High / Medium / Low) risk decision framework aligned with ISO 31000 and DORA, complete with suggested response actions.

The framework is designed to be usable as an actual ICT risk management capability for a digital banking institution, not just a benchmark exercise.

## Repository Contents

| File | Description |
|---|---|
| `Thesis_Phase1_EDA_Preprocessing.ipynb` | Data loading, cleaning, exploratory data analysis, and the full preprocessing pipeline |
| `Thesis_Phase2_Models.ipynb` | Model training, evaluation, SHAP explainability, and the risk-tiering decision framework |
| `CICIDS2017_Banking_FINAL.csv` | Working dataset (see [Dataset](#dataset) below) |

## Dataset

Built from the [CICIDS2017](https://www.unb.ca/cic/datasets/ids-2017.html) network intrusion detection benchmark (Canadian Institute for Cybersecurity), subsetting five days of the original capture chosen for relevance to digital banking threat scenarios:

| Source capture | Records |
|---|---|
| Tuesday (brute force) | 445,909 |
| Wednesday (DoS variants) | 692,703 |
| Thursday (web attacks) | 170,366 |
| Thursday (infiltration) | 288,602 |
| Friday (DDoS) | 225,745 |

Combined into `CICIDS2017_Banking_FINAL.csv`: **1,048,575 raw records × 79 columns** (78 network flow features + label).

After removing nulls, infinite values (a known `Flow Bytes/s` / `Flow Packets/s` CICFlowMeter artefact), and duplicates, the cleaned working set used throughout has **975,899 records** across **10 classes**:

- `BENIGN` (74.8%)
- `DDoS`, `DoS Hulk`, `DoS Slowhttptest`, `DoS Slowloris`
- `FTP-Patator`, `SSH-Patator`
- `Web Attack – Brute Force`, `Web Attack – SQL Injection` (21 records — the most severe imbalance in the dataset), `Web Attack – XSS`

> The dataset file is provided for reproducibility. It is a large file (~1M rows); see [Environment](#environment--how-to-run) below for handling it.

## Methodology

A benchmark network intrusion detection dataset is used to train and evaluate four machine learning architectures under a common preprocessing pipeline, model performance is compared using standard multi-class classification metrics, the best-performing supervised models are subjected to post-hoc SHAP explainability analysis, and model outputs are then mapped onto a risk-tiering decision framework.

### Phase 1 — Preprocessing (`Thesis_Phase1_EDA_Preprocessing.ipynb`)
- Label encoding fixes (UTF-8/Latin-1 artefacts in "Web Attack" labels)
- Data quality audit and cleaning (nulls, infinities, duplicates)
- Class distribution / imbalance analysis
- Feature-level EDA and Pearson correlation heatmap
- Stratified train/test split
- `StandardScaler` normalisation (fit on training data only — no leakage)
- **SMOTE** oversampling of the training set to address the ~75/25 benign/attack imbalance
- **PCA** dimensionality reduction (for training efficiency and 2D risk-cluster visualisation)
- **RFE** (Recursive Feature Elimination, Random Forest-based) for interpretable feature selection
- Saves all preprocessed arrays and fitted objects (scaler, PCA, label encoder) for Phase 2

### Phase 2 — Models, Explainability & Risk Framework (`Thesis_Phase2_Models.ipynb`)
Four models trained on the SMOTE-balanced data, each representing a different detection paradigm:

| Model | Paradigm | Notes |
|---|---|---|
| Random Forest | Supervised, bagging ensemble | Trained on full 78 features |
| XGBoost | Supervised, gradient boosting | Trained on full 78 features; validation split carved out for early stopping |
| LSTM | Supervised, sequential (RNN) | Trained on 16 PCA components reshaped as a 16-timestep sequence |
| Isolation Forest | Unsupervised, anomaly detection | Binary normal/anomaly output; targets zero-day/novel attacks |

Also included:
- **SHAP** explainability analysis for Random Forest and XGBoost
- Comparative evaluation (precision, recall, F1, AUC-ROC, precision-recall curves, false positive rate — accuracy alone is intentionally avoided given the class imbalance)
- **Risk Tier Decision Framework**: maps model outputs to Critical / High / Medium / Low risk tiers with response recommendations, aligned to ISO 31000 and DORA Article 13

## Results Summary

| Model | Macro F1 | AUC-ROC |
|---|---|---|
| **XGBoost** | **0.906** | **0.9999** |
| Random Forest | 0.881 | — |
| LSTM | 0.742 | — |
| Isolation Forest | 0.700 | — |

- XGBoost was the top performer overall; tree-based ensembles clearly outperformed the sequential (LSTM) and unsupervised (Isolation Forest) models on this structured, tabular network flow data.
- Isolation Forest's default false-positive rate (26.9%) was shown to stem from unoptimised hyperparameters, not a hard ceiling — a contamination-parameter sensitivity sweep reduced false positives by nearly two-thirds (at a recall cost).
- SHAP identified `Destination Port`, `Init_Win_bytes_backward`, and packet-timing features as the most influential predictors for both Random Forest and XGBoost.
- The rarest class, `Web Attack – SQL Injection` (21 records), remained the most volatile across all models even after SMOTE — consistent with known limits of SMOTE's effectiveness under severe high-dimensional imbalance.

## Environment & How to Run

Both notebooks were built and run in **Google Colab** with a **T4 GPU runtime**.

1. Upload `CICIDS2017_Banking_FINAL.csv` to Google Drive (e.g. `My Drive/Thesis/CICIDS2017_Banking_FINAL.csv`).
2. Run `Thesis_Phase1_EDA_Preprocessing.ipynb` top to bottom. This mounts Drive, cleans the data, and saves preprocessed arrays and fitted objects (`scaler.pkl`, `pca.pkl`, `label_encoder.pkl`, `X_train_sm.npy`, etc.) to `My Drive/Thesis/preprocessed/`.
3. Run `Thesis_Phase2_Models.ipynb` top to bottom. It loads Phase 1's outputs, trains all four models, runs SHAP, and produces the comparative evaluation and risk-tier reports.

**Approximate runtimes on a T4:**
- SMOTE (Phase 1): 5–15 minutes
- Random Forest: 15–30 minutes
- XGBoost: 20–40 minutes
- LSTM: 20–40 minutes (GPU required)
- Isolation Forest: 5–10 minutes

**Key libraries:** `scikit-learn`, `imbalanced-learn` (SMOTE), `xgboost`, `tensorflow`/`keras` (LSTM), `shap`, `pandas`, `numpy`, `matplotlib`/`seaborn`.

## Scope & Limitations

- Evaluated exclusively on the CICIDS2017 benchmark; no production banking network data was used.
- Risk-tier scoring weights follow CVSS-style severity logic and general banking operational risk taxonomy, not a specific institution's formal risk appetite — the framework is a template, not an as-deployed system.
- CICIDS2017's dataset age and severe imbalance in rare web-attack classes are acknowledged limitations; future work should validate against live banking network telemetry.

## Regulatory & Standards Alignment

- **DORA (EU) Article 13** — automated ICT risk detection tools must be explainable and auditable
- **ISO 31000:2018** — risk evaluation against defined criteria
- **ISO/IEC 27001:2022**, **NIST Cybersecurity Framework**, **Basel III**, **IIA Three Lines Model** — referenced as supplementary governance context

## Confidentiality

All references to "a digital banking institution" throughout this work are anonymised and generalised — no confidential, proprietary, or institution-specific information is disclosed.

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.
