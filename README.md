# Teacher–Student Distillation with Confidence-Based Intrusion Detection System

A hybrid intrusion detection system for CICIDS2017 that combines:

- a **high-capacity CatBoost teacher**
- a **shallow MLP student**
- a **confidence-based routing mechanism** that sends high-confidence cases to the student and falls back to the teacher when the student is uncertain

The goal is to keep most predictions fast and lightweight while preserving strong detection quality on hard samples.

## Overview

This project implements a teacher–student distillation pipeline for **multi-class intrusion detection** on the **CICIDS2017** dataset. The system learns a compact student model from a stronger teacher, calibrates the student’s probabilities with temperature scaling, and then routes predictions dynamically using a validation-tuned confidence threshold.

### Core idea

For each sample:

1. the **student** predicts class probabilities;
2. the maximum student probability is treated as its confidence;
3. if confidence is **greater than or equal to** the tuned threshold, the system keeps the student prediction;
4. otherwise, it **falls back to the CatBoost teacher**.

This yields a practical trade-off between inference cost and detection performance.

---

## Dataset

The pipeline is built on CICIDS2017, using the processed dataset artifacts produced by the preprocessing stage.

### Final modeling setup

- **9 classes**
- **34 selected features**
- **Stratified train/validation split**
- **QuantileTransformer scaling**
- **Class rebalancing / handling of rare classes**
- **Label encoding saved as artifacts**

### Class labels

- Benign
- DDoS
- DoS GoldenEye
- DoS Hulk
- DoS Slowhttptest
- DoS slowloris
- FTP-Patator
- Other
- SSH-Patator

---

## Repository workflow

### 1) Dataset preprocessing

The preprocessing pipeline:

- loads the CICIDS2017 parquet sources
- removes invalid / unusable values
- keeps only numeric features
- removes low-variance features
- removes highly correlated features
- optionally uses feature importance filtering
- skips PCA because the resulting feature count is already manageable
- scales features using **QuantileTransformer(output_distribution="normal")**
- encodes labels
- saves train/test parquet files and preprocessing artifacts

### Saved preprocessing artifacts

- `train_processed.parquet`
- `test_processed.parquet`
- `train_labels_original.parquet`
- `test_labels_original.parquet`
- `label_encoder.joblib`
- `scaler.joblib`
- `variance_selector.joblib`
- `preprocess_settings.json`
- `manifest.json`

---

## Model architecture

### Teacher: CatBoost

The teacher is a CatBoost multiclass classifier configured with:

- `iterations=800`
- `learning_rate=0.03`
- `depth=8`
- `l2_leaf_reg=3.0`
- `border_count=128`
- early stopping with `od_wait=50`
- optional class weights

### Student: shallow MLP

The student is a compact feed-forward network:

- input dimension: **34**
- hidden layers: **[128, 64]**
- BatchNorm
- ReLU
- Dropout (`0.15`)
- output layer: 9 classes

### Distillation loss

The student is trained with a blended objective:

- hard-label cross-entropy
- soft distillation loss via KL divergence against teacher probabilities
- temperature-controlled softening
- `alpha=0.5`
- `temperature=4.0`

### Calibration

After training, the student is calibrated using **temperature scaling** on the validation set.

The learned temperature in the reported run was:

- **2.079877**

---

## Confidence-based routing

Routing is tuned on the validation set by scanning a confidence grid and selecting the threshold that maximizes routed accuracy, with teacher usage used as a tie-breaker.

### Validation-selected threshold

- **Threshold = 0.9662**
- **Routed validation accuracy = 0.996299**
- **Teacher usage on validation = 9.3375%**
- **Routed validation macro-F1 = 0.949214**

### Inference behavior

- high-confidence student samples are predicted by the student
- low-confidence samples are passed to the teacher
- the final output is the routed probability distribution

---

## Reported results

### Teacher

- **Test accuracy:** 0.995490
- **Test macro-F1:** 0.936430

### Student (raw / calibrated)

- **Test accuracy:** 0.983594
- **Test macro-F1:** 0.834860

### Routed system

- **Test accuracy:** 0.996151
- **Test macro-F1:** 0.946932
- **Teacher usage:** 9.2641%
- **Student usage:** 90.7359%

### Additional test metrics

| Model | Accuracy | Balanced Acc. | Macro-F1 | Weighted-F1 | Macro Precision | Macro Recall | Log Loss | Brier Score | ECE |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Teacher/Test | 0.995490 | 0.996079 | 0.936430 | 0.995910 | 0.896550 | 0.996079 | 0.021084 | 0.008300 | 0.007469 |
| Student Raw/Test | 0.983594 | 0.949256 | 0.834860 | 0.985587 | 0.772524 | 0.949256 | 0.057634 | 0.025113 | 0.011215 |
| Student Calibrated/Test | 0.983594 | 0.949256 | 0.834860 | 0.985587 | 0.772524 | 0.949256 | 0.041393 | 0.022566 | 0.005678 |
| Routed/Test | 0.996151 | 0.990015 | 0.946932 | 0.996321 | 0.912670 | 0.990015 | 0.018076 | 0.007036 | 0.002768 |

---

## Project structure

A typical layout for the generated artifacts is:

```text
cicids2017_processed/
├── data/
│   ├── train_processed.parquet
│   ├── test_processed.parquet
│   ├── train_labels_original.parquet
│   └── test_labels_original.parquet
├── artifacts/
│   ├── label_encoder.joblib
│   ├── scaler.joblib
│   ├── variance_selector.joblib
│   └── preprocess_settings.json
└── manifest.json
```

Modeling artifacts are saved separately:

```text
tdr_ids/
├── teacher_catboost.cbm
├── student_mlp_best.pt
├── student_temperature.joblib
├── routing_threshold.joblib
├── metadata.json
└── eval/
    └── routing_behavior.png
```

---

## Requirements

Suggested Python packages:

- `numpy`
- `pandas`
- `scikit-learn`
- `catboost`
- `torch`
- `joblib`
- `matplotlib`
- `seaborn`

Optional, depending on the preprocessing notebook:

- `tensorflow`
- `umap-learn`
- `scikit-image`
- `pyarrow`

---

## Installation

```bash
git clone https://github.com/yacerak/Teacher-Student-Distillation-with-Confidence-Based-Intrusion-detection-System.git
cd your-repo

pip install -r requirements.txt
```

If no `requirements.txt` exists yet, install the core dependencies manually:

```bash
pip install numpy pandas scikit-learn catboost torch joblib matplotlib seaborn pyarrow
```

---

## How to run

### 1. Preprocess the dataset

Run the preprocessing notebook or script to generate:

- scaled train/test parquet files
- label encoder
- scaler
- feature selector
- preprocessing manifest

### 2. Train the teacher and student

Run the modeling notebook/script to:

- train CatBoost teacher
- train distilled student MLP
- fit temperature scaling
- tune routing threshold
- evaluate routed inference on the test set

### 3. Review outputs

Inspect:

- classification reports
- confusion matrix
- calibration metrics
- routing behavior plots
- saved model artifacts

---

## Key implementation details

- **Teacher model** is strong and stable on the full feature set.
- **Student model** is intentionally shallow to reduce inference cost.
- **Distillation** transfers teacher knowledge to the student using soft targets.
- **Temperature scaling** improves confidence calibration before routing.
- **Threshold optimization** is done on validation data only.
- **Final routed predictions** combine the best of both models.

---

## Notes

- The preprocessing stage reports **34 final features**, so PCA is not applied.
- Class imbalance is handled using a combination of preprocessing decisions and class weights.
- The routed system achieves better test macro-F1 than the teacher alone in the reported run, while using the teacher for only about **9.26%** of test samples.

---

## Citation / provenance

This README is derived from the uploaded preprocessing and modeling code notebooks for the CICIDS2017 Teacher–Student Distillation IDS pipeline.
