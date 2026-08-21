# Cross-Workload Flow-Size Prediction for AI Data Center Networks

Artifacts for the paper **“Cross-Workload Flow-Size Prediction for AI Data Center Networks”** by Leonardo Alberro, Pedro Casas, Matías Richart, and Eduardo Grampín (CNSM 2026).

The work replaces oracle flow-size knowledge in flow-aware data-center transport with predictions derived from end-host telemetry. A single XGBoost model is trained over the union of partially overlapping feature spaces from several AI workloads. The paper evaluates both continuous flow-size regression and threshold-based short/long-flow classification, then validates the predictions in the Fork congestion-control protocol.

The revised manuscript is available at [`atlas_cnsm2026_revised.pdf`](atlas_cnsm2026_revised.pdf).

## Repository contents

```text
.
├── atlas_cnsm2026_revised.pdf   # revised paper
├── classification/              # binary short/long-flow XGBoost models
├── regression/                  # continuous flow-size XGBoost models
└── datasets/                    # dataset download instructions
```

This snapshot contains the paper, dataset link, and serialized trained models. It does **not** currently contain the data-preparation, training, evaluation, or ns-3/Fork simulation source code. Consequently, the supplied models can be inspected and used with identically prepared feature matrices, but the paper's results cannot be regenerated end to end from this repository alone (yet).

## Data

Download the dataset archive from [Google Drive](https://drive.google.com/file/d/1PUQBHa7oqHzD-HoWrY8vRRS6PpUoRL98/view?usp=sharing). See [`datasets/README.md`](datasets/README.md) for instructions.

The paper combines:

- traces from the public Flux dataset;
- a GPT fine-tuning workload collected with INTFusion; and
- updated Spark KMeans, PageRank, and SGD workloads collected with INTFusion.

Each workload consists of multiple tabular files, with one file corresponding to one independent run. The experiments split complete runs—not individual flows—into training, validation, and test partitions using a 70/20/10 ratio. This prevents flows from the same run leaking across partitions.

A flow is treated as a flowlet: a burst identified by the conventional 5-tuple plus a gap identifier. The target is the total payload size of that burst. Features available before the target flow begins include temporal context, the previous five flows, host activity, memory and storage activity, network activity, and transport state. `flow_size0`, the target flow size, must never be included as an input feature.

## Trained models

All `.pkl` files serialize an `xgboost.core.Booster`.

### Regression

The regression models use the `reg:squarederror` objective and predict a continuous, max-normalized flow size.

| File | Scope |
| --- | --- |
| `regression/model-unified.pkl` | Cross-workload model |
| `regression/model-GPT.pkl` | GPT-specific model |
| `regression/model-KMeans.pkl` | KMeans-specific model |
| `regression/model-PageRank.pkl` | PageRank-specific model |
| `regression/model-SGD.pkl` | SGD-specific model |
| `regression/model-tensorflow.pkl` | TensorFlow-specific model |

Each feature column and the regression target were divided by the corresponding maximum computed on the training set. Predictions therefore need to be multiplied by the saved training-target maximum to recover bytes. That preprocessing metadata is not included in this snapshot.

### Classification

The classifiers use the `binary:logistic` objective. A model predicts class `0` for a short flow and class `1` for a long flow.

| Files | Scope | Threshold encoded in filename |
| --- | --- | --- |
| `classification/model-unified-1024.pkl` | Cross-workload | 1 KiB (1,024 bytes) |
| `classification/model-unified-10240.pkl` | Cross-workload | 10 KiB (10,240 bytes) |
| `classification/model-unified-102400.pkl` | Cross-workload | 100 KiB (102,400 bytes) |
| `classification/model-unified-100000.pkl` | Cross-workload | 100,000 bytes |
| `classification/model-classification-<workload>-10240.pkl` | Workload-specific | 10 KiB (10,240 bytes) |
| `classification/model-classification-<workload>-102400.pkl` | Workload-specific | 100 KiB (102,400 bytes) |

`<workload>` is one of `GPT`, `KMeans`, `PageRank`, `SGD`, or `tensorflow`. Note that 100,000 bytes and 100 KiB are different thresholds; select the artifact that matches the preprocessing and evaluation convention you intend to use.

## Loading a model

Install Python and XGBoost in an isolated environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install xgboost pandas
```

Then load a trusted artifact and predict from a feature matrix whose columns, order, missing-value representation, and normalization exactly match training:

```python
import pickle

import xgboost as xgb

with open("classification/model-unified-102400.pkl", "rb") as handle:
    model = pickle.load(handle)

# X must reproduce the training feature schema and preprocessing.
dmatrix = xgb.DMatrix(X)
probability_long = model.predict(dmatrix)
predicted_class = (probability_long >= 0.5).astype("int8")
```

Python pickle files can execute code while loading. Only load these artifacts if you trust their origin. Pickle compatibility can also depend on the XGBoost version used to create the model; a version manifest is not included here.

## Method summary

The unified representation uses the union of the workload feature sets. Features unavailable for a workload are left missing, allowing XGBoost's sparsity-aware split logic to learn default branches. The selected history length is five preceding flows.

The evaluation addresses three questions:

1. Whether one cross-workload regressor can preserve the quality of workload-specific regressors.
2. Whether one cross-workload classifier can reliably distinguish short and long flows.
3. Whether learned predictions can replace Fork's oracle flow-size information without materially degrading flow completion time.

For regression, the paper reports R², MAE, RMSE, and threshold-crossing errors. For classification, it reports accuracy, precision, recall, F1, false-positive rate, and false-negative rate. The transport experiments use Fork in ns-3 with a 100 KB short-flow threshold, 10 Gbit/s links, 60 µs RTT, 250 KB buffers, and offered loads from 30% to 90%.

## Main results

- The unified classifier correctly identifies 99.87% of short flows and 98.38% of long flows at the evaluated 100 KB boundary.
- The unified regressor remains close to specialized models for most workloads, although the effect of unification is workload dependent.
- Threshold-crossing errors are more informative for flow-aware transport than aggregate regression error alone.
- Misclassifying a long flow as short is substantially more damaging to average flow completion time than misclassifying a short flow as long.
- Learned models retain most of Fork's oracle-assisted transport benefit without requiring a separate predictor per workload.

See the paper for the complete methodology, confidence intervals, workload-level results, and limitations.

## Citation

Publication metadata is not yet included in this repository. The paper is under single blind review.

