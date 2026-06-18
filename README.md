# bKash Presents NSUCEC Cybernauts Datathon — FictiPay Customer Churn Prediction

🏆 **Private Leaderboard Rank: Top 20**
🎯 **Qualified Among the Top 25 Teams**
📈 **Final OOF AUC: 0.98535**

An end-to-end machine learning pipeline developed for the **bKash Presents NSUCEC Cybernauts Datathon** to predict customer churn in a large-scale mobile financial service ecosystem.

The solution combines large-scale data processing, behavioral feature engineering, survival-style inactivity modeling, explainable machine learning, and ensemble learning to identify customers at risk of becoming inactive.

---

## Competition Overview

The objective of the competition was to build a scalable customer churn prediction system for **FictiPay**, a fictional mobile financial service provider operating in an emerging-market environment.

Unlike a typical Kaggle-style prediction challenge, the competition evaluated both:

* Predictive performance
* Scalability of the data pipeline
* Feature engineering quality
* Explainability
* Business recommendations
* Documentation and presentation quality

The challenge simulated real-world fintech conditions where model performance, interpretability, and operational feasibility are equally important.

---

## Scale of the Problem

| Dataset                 | Size         |
| ----------------------- | ------------ |
| Transaction Ledger      | 200M+ rows   |
| KYC Records             | ~2 Million   |
| Day-End Balance Records | ~360 Million |
| Training Accounts       | 595,000      |
| Test Accounts           | 255,000      |
| Churn Rate              | 12.68%       |

The raw data volume exceeded available memory, requiring an efficient out-of-core processing strategy.

---

## What This Project Does

The system predicts the probability that a customer will churn after a given reference date.

Rather than relying solely on traditional RFM metrics, the pipeline models:

* Transaction activity patterns
* Customer inactivity behavior
* Inter-transaction timing
* Balance trajectory changes
* Historical engagement trends
* Demographic and account-level characteristics

The final output is a probability score (`CHURN_PROB`) that ranks customers by churn risk.

---

## Solution Architecture

```text
Transaction Ledger
Balance History
KYC Profiles
        │
        ▼
DuckDB Feature Engineering
        │
        ├── Recency Features
        ├── Frequency Features
        ├── Rolling Window Features
        ├── Poisson Inactivity Features
        ├── Survival / Gap Features
        ├── Balance Features
        └── KYC Features
        │
        ▼
LightGBM
CatBoost
Neural Network
        │
        ▼
Rank-Normalized Ensemble
        │
        ▼
Churn Probability
```

---

## Large-Scale Data Processing

The competition dataset was intentionally designed to exceed memory limits.

To handle this scale efficiently:

* DuckDB was used as the primary analytical engine
* Out-of-core aggregations were performed directly in SQL
* Spill-to-disk processing enabled handling of datasets larger than RAM
* Transaction tables were processed without loading full datasets into memory
* Feature engineering was executed through optimized aggregation pipelines

This approach allowed hundreds of millions of records to be transformed into account-level features efficiently.

---

## Feature Engineering

A total of **194 engineered features** were generated.

### Recency & Frequency Features

Examples:

```python
s_cnt_7d
s_cnt_14d
s_cnt_30d
s_cnt_90d
s_recency_days
```

These features capture how recently and how frequently customers interact with the platform.

---

### Rolling Window Features

Transaction activity was aggregated across multiple windows:

```text
1, 2, 3, 5, 7, 10, 14, 21, 30, 45, 60, 90 days
```

This allows the model to detect sudden behavioral shifts and engagement decline.

---

### Poisson Inactivity Features

One of the strongest feature families in the project.

Examples:

```python
pois_p0_sent_7
pois_p0_sent_14
pois_p0_sent_30
pois_p0_any_30
```

These estimate the probability of observing no future activity given a customer's historical transaction rate.

The strongest individual signal achieved a standalone AUC of approximately **0.975**.

---

### Survival & Transaction Gap Features

Examples:

```python
s_recency_vs_maxgap
s_recency_vs_avggap
s_gap_zscore
days_to_next_expected
gap_cv
s_broke_pattern
```

These features model deviations from a customer's normal transaction rhythm and identify customers whose inactivity exceeds historical expectations.

---

### Trend & Decay Features

Examples:

```python
decay_m3_vs_m1
decay_m3_vs_m2
trend_30_vs_prev30
rate_accel
```

These capture whether customer activity is accelerating or declining over time.

---

### Balance Features

Examples:

```python
bal_mean
bal_std
bal_last
days_since_bal_change
days_since_pos_bal
```

Balance dynamics provide additional evidence of disengagement and declining platform usage.

---

### KYC Features

Examples:

```python
account_type
gender
region
account_age_days
```

These account for demographic and lifecycle differences between customer segments.

---

## Modeling Approach

### LightGBM

Primary model used in the ensemble.

Configuration:

* Learning Rate: 0.02
* Num Leaves: 128
* Feature Fraction: 0.75
* Bagging Fraction: 0.85
* L1 Regularization: 0.5
* L2 Regularization: 2.0
* GPU Training

**OOF AUC: 0.98534**

---

### CatBoost

Used for robust handling of categorical variables and complementary decision boundaries.

Configuration:

* Learning Rate: 0.03
* Depth: 8
* L2 Leaf Regularization: 6
* Native Categorical Support
* GPU Training

**OOF AUC: 0.98518**

---

### Neural Network

Architecture:

```text
256 → 128 → 64
BatchNorm
Dropout
Adam Optimizer
QuantileTransformer
```

The neural network provides additional model diversity that improves ensemble performance.

**OOF AUC: 0.98506**

---

## Ensemble Strategy

All models were trained on the same 5-fold StratifiedKFold split.

Pipeline:

```text
Model Predictions
       │
       ▼
Per-Fold Rank Normalization
       │
       ▼
OOF Blending
       │
       ▼
Final Churn Probability
```

A weight-optimization search selected the final blend:

| Model          | Weight |
| -------------- | ------ |
| LightGBM       | 0.745  |
| CatBoost       | 0.115  |
| Neural Network | 0.140  |

Final Ensemble OOF AUC:

```text
0.98535
```

---

## Validation & Explainability

### SHAP Analysis

Global SHAP analysis revealed that the model primarily relies on:

* Recent transaction counts
* Transaction recency
* Poisson inactivity probabilities
* Balance staleness indicators
* Transaction periodicity features

These findings align closely with expected customer churn behavior.

---

### Leakage Audit

Three independent leakage probes were evaluated:

* Account ID numeric suffix
* Dataset row order
* Account age

Maximum observed probe AUC:

```text
0.5016
```

No meaningful leakage or exploitable artifact was detected.

This indicates that model performance is driven by genuine behavioral signals rather than synthetic dataset shortcuts.

---

## Final Results

| Metric                   | Score        |
| ------------------------ | ------------ |
| LightGBM OOF AUC         | 0.98534      |
| CatBoost OOF AUC         | 0.98518      |
| Neural Network OOF AUC   | 0.98506      |
| Final Ensemble OOF AUC   | 0.98535      |
| Precision @ Top 10% Risk | 0.9173       |
| Recall @ Top 10% Risk    | 0.7235       |
| Private Leaderboard Rank | Top 20       |
| Qualification Status     | Top 25 Teams |

At a 10% intervention budget, the model successfully identifies approximately **72% of churning customers** while maintaining **91.7% precision**.

---

## Repository Structure

```text
fictipay-churn-prediction/
├── notebooks/
│   └── churn_prediction.ipynb
├── docs/
│   ├── report.pdf
│   ├── presentation.pdf
│   └── features.md
├── explainability/
│   ├── shap_summary.png
│   ├── shap_beeswarm.png
│   └── feature_importance.csv
├── assets/
│   └── leaderboard_screenshot.png
├── requirements.txt
├── README.md
└── .gitignore
```

---

## Future Work

Potential future improvements include:

* Transaction-network analysis
* Graph Neural Networks (GNNs)
* Temporal validation strategies
* Sequence modeling with Transformers
* Self-supervised customer embeddings
* Survival-specific deep learning approaches

---

## Tech Stack

* Python
* DuckDB
* Pandas
* NumPy
* Scikit-Learn
* LightGBM
* CatBoost
* TensorFlow / Keras
* SHAP
* Matplotlib

---

## Acknowledgements

Built for the **bKash Presents NSUCEC Cybernauts Datathon**.

This solution achieved a **Top 20 Private Leaderboard finish** and qualified among the **Top 25 teams** advancing from the assessment round.
