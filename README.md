# Anomaly-Based Intrusion Detection System for Zero-Day Attack Detection Using ML

Traditional signature-based IDS can't catch zero-day exploits — they only know attacks they've seen before. This project instead learns a strict "normalcy baseline" from benign network traffic and flags anything that deviates from it, using a **Spatio-Temporal Hybrid CNN-LSTM** pipeline (with a **Deep Autoencoder** for feature compression and anomaly thresholding) on the **CIC-IDS2017** dataset.

Full writeup: [`MLA_Paper(final) (1) (2).pdf`](MLA_Paper(final)%20(1)%20(2).pdf).

## Architecture

```
Raw PCAP-derived CSVs (CIC-IDS2017)
        │
        ▼
Phase 1 — Data Prep          clean, cast, scale (StandardScaler fit on BENIGN traffic only)
        │
        ▼
Phase 2 — Feature Selection  ANOVA F-test → top 25 (of 78) features, 80/20 split, SMOTE-balanced
        │                    training copy also produced (see Results — SMOTE was ultimately rejected)
        ▼
Phase 3 — Autoencoder        Deep Autoencoder (25 → 16 → 8 → 16 → 25), trained to reconstruct
        │                    normal traffic; encoder reused as a feature compressor
        ▼
Phase 4 — Threshold          Reconstruction error (MSE) on held-out BENIGN samples → statistical
        │                    anomaly threshold τ = mean + 3·std
        ▼
Phase 5 — CNN-LSTM           Encoder output windowed into sequences (window=5) → 1D-CNN
        │                    (spatial/header correlations) + LSTM (temporal/burst patterns),
        │                    trained twice — with and without the SMOTE-balanced data
        ▼
Phase 6 — Evaluation         Classification report, confusion matrix, precision/recall/accuracy,
                             compared with vs. without SMOTE at a 0.85 confidence threshold
```

## Dataset

[CIC-IDS2017](https://www.unb.ca/cic/datasets/ids-2017.html) (Canadian Institute for Cybersecurity) — labeled network flow data covering benign traffic and common attacks (DDoS, PortScan, Web Attacks, Infiltration) across a full working week.

**Raw CSVs and generated intermediate artifacts (`.npy`, `.parquet`, trained model weights) are not included in this repo** due to size (~1.8GB) — see `.gitignore`. To reproduce:

1. Download the CIC-IDS2017 `MachineLearningCSV` files from the link above.
2. Place them under `MLA_Final_Project/` (matching the filenames referenced in `PHASE_1_Data_Prep (1).ipynb`).
3. Run the notebooks in order, Phase 1 → Phase 6.

## Repo Structure

```
├── PHASE_1_Data_Prep (1).ipynb                              # Load, clean, cast, scale raw CSVs
├── PHASE_2_Feature_Selection (1).ipynb                       # ANOVA feature selection, split, SMOTE
├── PHASE_3_Autoencoder_Training (1).ipynb                    # Deep Autoencoder training
├── PHASE_4_Threshold_Calculation (1).ipynb                   # Reconstruction-error anomaly threshold
├── PHASE_5_CNN_LSTM_Integration (1).ipynb                    # Hybrid CNN-LSTM sequence classifier
├── PHASE_6_Final_Evaluation_&_Performance_Metrics (1).ipynb  # Final metrics & confusion matrix
├── MLA_Paper(final) (1) (2).pdf                              # Project writeup
└── MLA_Final_Project/                                        # Data & model artifacts (mostly gitignored)
```

## Tech Stack

- **Data processing**: pandas, NumPy, scikit-learn (`StandardScaler`, `SelectKBest`/ANOVA F-test, train/test split), imbalanced-learn (SMOTE)
- **Modeling**: PyTorch (Deep Autoencoder, hybrid Conv1D + LSTM classifier)
- **Evaluation**: scikit-learn metrics, Matplotlib/Seaborn for the confusion matrix
- **Environment**: Google Colab (GPU), Google Drive for artifact persistence between notebooks

## Key Design Choices

- **Scaler fit on benign traffic only** — defines "normal" before any attack data influences the baseline.
- **Autoencoder as a feature compressor and independent anomaly signal** — its 8-dimensional latent space feeds the CNN-LSTM, while its reconstruction error separately defines a statistical anomaly threshold (mean + 3σ on normal traffic).
- **Windowed sequences (size 5)** — the CNN-LSTM sees short temporal windows of flows rather than single flows, to capture bursty attack patterns (e.g. port scans, DDoS) that a per-flow model would miss.
- **Precision over accuracy** — the project explicitly optimizes for minimizing false alarms ("alert fatigue") over raw accuracy, since a real SOC will ignore an IDS that cries wolf.

## Results — SMOTE was rejected

The model was evaluated both with and without SMOTE-balanced training data:

| Metric | Baseline (No SMOTE) | With SMOTE |
|---|---|---|
| Overall Accuracy | 0.61 | 0.86 |
| Attack Precision | **0.98** | 0.15 |
| Attack Recall | 0.32 | 0.02 |

SMOTE raises overall accuracy but **destroys attack precision** — it introduces synthetic noise that dilutes the model's decision boundary, causing the vast majority of real attacks to be misclassified as normal. The project's core finding is this inverse relationship between class balancing and precision reliability: the **baseline model (no SMOTE)**, run at a 0.85 confidence threshold, is the one recommended for deployment — a 0.98 precision means an alert is almost always a genuine threat, even though ~68% of attacks go undetected (0.32 recall). Full discussion in §4–6 of the paper.

## Limitations (from the paper)

- **Static threshold** — the MSE anomaly boundary doesn't adapt as "normal" traffic drifts over time.
- **No encrypted payload inspection** — only flow-level metadata is analyzed, so HTTPS-borne application-layer exploits are out of scope.
- **LSTM latency** — sequential processing may bottleneck on very high-speed (10Gbps+) links.
- The paper also describes integrating a live Scapy packet sniffer and real-time dashboard as a deployment-gap contribution; that component isn't part of the notebooks in this repo.

## Authors

Vedant Desai, Kennith Kujur, Anvay Khuperkar — Department of Information Technology, NMIMS MPSTME
