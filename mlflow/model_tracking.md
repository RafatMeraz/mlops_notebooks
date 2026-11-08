# Model Tracking with MLflow

> **TL;DR:** MLflow Tracking is a logging API + storage backend + UI that records
> every experiment run as an immutable record of **params → metrics → artifacts**.
> You call the API during training; MLflow handles the rest.

---

## 1. Core Concepts

```mermaid
flowchart TD
    EXP["🗂️ Experiment\nA named container for related runs\ne.g. 'churn-prediction-v2'"]
    RUN["🧪 Run\nOne execution of training code\nhas a unique run_id (UUID)"]
    PARAM["🔧 Params\nScalar inputs that don't change mid-run\nlr=0.01, batch_size=32"]
    METRIC["📈 Metrics\nScalar outputs, can have steps\nf1=0.87 at epoch=10"]
    TAG["🏷️ Tag\nFree-form metadata\nengineer=rafat, dataset=v3"]
    ARTIFACT["📦 Artifact\nAny file — model.pkl, plot.png, data.csv"]

    EXP --> RUN
    RUN --> PARAM
    RUN --> METRIC
    RUN --> TAG
    RUN --> ARTIFACT
```

| Concept | Key rule |
|---|---|
| **Experiment** | Group runs by task, not by date. One experiment per model objective. |
| **Run** | Immutable once ended. Nested runs allowed for hyperparameter sweeps. |
| **Param** | Log once per run. Changing mid-run is an error. |
| **Metric** | Log many times with `step=` to get training curves. |
| **Tag** | Arbitrary string key-value. Use for metadata not in params/metrics. |
| **Artifact** | Any file or directory. Stored in artifact store (local or cloud). |

---

## 2. Tracking Server Modes

MLflow supports three backend configurations:

```mermaid
flowchart LR
    subgraph Local["1️⃣ Local (default)"]
        L1["mlruns/ folder\nin project root"]
    end

    subgraph DB["2️⃣ DB + local artifacts"]
        D1["SQLite / PostgreSQL\nfor run metadata"]
        D2["local path\nfor artifacts"]
    end

    subgraph Remote["3️⃣ Remote (production)"]
        R1["PostgreSQL\nfor run metadata"]
        R2["S3 / GCS / Azure\nfor artifacts"]
        R3["mlflow server\nHTTP API"]
    end
```

```bash
# 1. local — no setup, just run
mlflow ui

# 2. sqlite backend
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --default-artifact-root ./mlruns

# 3. remote — team-shared
mlflow server \
  --backend-store-uri postgresql://user:pass@host/mlflowdb \
  --default-artifact-root s3://my-bucket/mlflow-artifacts \
  --host 0.0.0.0 \
  --port 5000
```

```python
# point your code at any server
import mlflow
mlflow.set_tracking_uri("http://mlflow.internal:5000")

# or via environment variable (preferred for CI)
# export MLFLOW_TRACKING_URI=http://mlflow.internal:5000
```

---

## 3. Logging API — The Full Reference

### 3.1 Manual Logging

```python
import mlflow

mlflow.set_experiment("churn-prediction")

with mlflow.start_run(run_name="rf-baseline") as run:
    print(f"Run ID: {run.info.run_id}")

    # --- PARAMS ---
    mlflow.log_param("n_estimators", 100)
    mlflow.log_param("max_depth", 5)

    # log a dict of params in one call
    mlflow.log_params({
        "learning_rate": 0.01,
        "batch_size": 32,
        "optimizer": "adam",
    })

    # --- METRICS ---
    mlflow.log_metric("f1", 0.87)
    mlflow.log_metric("auc", 0.91)

    # log a dict of metrics in one call
    mlflow.log_metrics({"precision": 0.88, "recall": 0.86})

    # log metric with step (builds training curves in UI)
    for epoch in range(50):
        train_loss = train_one_epoch(model, loader)
        val_loss = validate(model, val_loader)
        mlflow.log_metric("train_loss", train_loss, step=epoch)
        mlflow.log_metric("val_loss", val_loss, step=epoch)

    # --- TAGS ---
    mlflow.set_tag("engineer", "rafat")
    mlflow.set_tag("dataset_version", "v3")
    mlflow.set_tags({"git_branch": "feature/rf-tuning", "env": "dev"})

    # --- ARTIFACTS ---
    mlflow.log_artifact("outputs/confusion_matrix.png")       # single file
    mlflow.log_artifact("outputs/report.html", artifact_path="reports")  # into subfolder
    mlflow.log_artifacts("outputs/")                          # entire directory

    # log a dict or dataframe as artifact without writing to disk first
    import json
    mlflow.log_dict({"threshold": 0.5, "classes": ["no", "yes"]}, "config.json")
    mlflow.log_text("experiment notes here", "notes.txt")
    mlflow.log_figure(fig, "feature_importance.png")          # matplotlib Figure
    mlflow.log_image(img_array, "sample_prediction.png")      # numpy array
```

### 3.2 Auto-logging

MLflow can intercept framework training calls and log everything automatically:

```python
# one line before model.fit() — logs params, metrics, model
mlflow.sklearn.autolog()
mlflow.xgboost.autolog()
mlflow.lightgbm.autolog()
mlflow.tensorflow.autolog()
mlflow.pytorch.autolog()
mlflow.keras.autolog()
mlflow.statsmodels.autolog()

# fine-grained control
mlflow.sklearn.autolog(
    log_input_examples=True,   # saves a sample of X_train as artifact
    log_model_signatures=True, # infers input/output schema
    log_models=True,           # saves the model artifact
    max_tuning_runs=5,         # for GridSearchCV: log top N child runs
)
```

What sklearn autolog captures automatically:

| What | Example |
|---|---|
| All `__init__` params | `n_estimators=100`, `max_depth=5` |
| Training metrics | `training_f1_score`, `training_accuracy` |
| Cross-val metrics | if using `cross_val_score` |
| Model artifact | `model/model.pkl` + `MLmodel` manifest |
| Input example | sample rows from `X_train` |
| Model signature | input/output schema for serving |

### 3.3 Model Logging

```python
# framework-specific — preserves native serialization
mlflow.sklearn.log_model(model, "model")
mlflow.xgboost.log_model(model, "model")
mlflow.pytorch.log_model(model, "model")
mlflow.tensorflow.log_model(model, "model")

# generic pyfunc — for custom models
class MyModel(mlflow.pyfunc.PythonModel):
    def predict(self, context, model_input):
        return self.model.predict(model_input)

mlflow.pyfunc.log_model(
    artifact_path="model",
    python_model=MyModel(),
    artifacts={"model_path": "model.pkl"},
    conda_env={
        "channels": ["defaults"],
        "dependencies": ["python=3.11", "scikit-learn=1.3.2"],
    },
)
```

The logged model always includes:

```
model/
├── MLmodel           ← manifest: flavors, run_id, schema, timestamp
├── model.pkl         ← serialized model (framework-specific)
├── conda.yaml        ← full environment spec
├── requirements.txt  ← pip-installable deps
└── input_example.json  ← sample input (if log_input_examples=True)
```

---

## 4. Run Lifecycle

```mermaid
stateDiagram-v2
    [*] --> RUNNING : mlflow.start_run()
    RUNNING --> FINISHED : mlflow.end_run() or context exit
    RUNNING --> FAILED : unhandled exception inside with block
    RUNNING --> KILLED : manual client.set_terminated(KILLED)
    FINISHED --> [*]
    FAILED --> [*]
    KILLED --> [*]
```

```python
# context manager — preferred, handles exceptions automatically
with mlflow.start_run():
    ...  # FAILED status set automatically on exception

# manual — when you need the run_id before training
run = mlflow.start_run()
run_id = run.info.run_id
try:
    ...
    mlflow.end_run(status="FINISHED")
except Exception:
    mlflow.end_run(status="FAILED")
    raise

# resume an existing run — append more metrics/artifacts
with mlflow.start_run(run_id="abc123def456"):
    mlflow.log_metric("test_f1", 0.85)  # logged to the same run
```

---

## 5. Nested Runs (Hyperparameter Sweeps)

```mermaid
flowchart TD
    PARENT["Parent Run\nhyperparameter-sweep-001"]
    C1["Child Run\nlr=0.01, depth=3\nF1=0.81"]
    C2["Child Run\nlr=0.01, depth=5\nF1=0.87 ✅"]
    C3["Child Run\nlr=0.001, depth=5\nF1=0.79"]

    PARENT --> C1
    PARENT --> C2
    PARENT --> C3
```

```python
with mlflow.start_run(run_name="hyperparameter-sweep") as parent:
    mlflow.log_param("sweep_method", "grid")
    mlflow.log_param("n_trials", 9)

    best_f1 = 0
    best_params = {}

    for lr in [0.01, 0.001, 0.0001]:
        for depth in [3, 5, 10]:
            with mlflow.start_run(run_name=f"lr={lr}-depth={depth}", nested=True):
                mlflow.log_params({"lr": lr, "max_depth": depth})
                model = train(lr=lr, max_depth=depth)
                f1 = evaluate(model)
                mlflow.log_metric("f1", f1)
                mlflow.sklearn.log_model(model, "model")

                if f1 > best_f1:
                    best_f1 = f1
                    best_params = {"lr": lr, "max_depth": depth}

    # log summary on parent
    mlflow.log_metric("best_f1", best_f1)
    mlflow.log_params({f"best_{k}": v for k, v in best_params.items()})
```

---

## 6. Querying Runs Programmatically

```python
import mlflow
import pandas as pd

client = mlflow.MlflowClient()

# --- search runs ---
runs = mlflow.search_runs(
    experiment_names=["churn-prediction"],
    filter_string="metrics.f1 > 0.85 AND params.max_depth = '5'",
    order_by=["metrics.f1 DESC"],
    max_results=10,
)
# returns a pandas DataFrame
print(runs[["run_id", "params.max_depth", "metrics.f1"]])

# --- get a specific run ---
run = client.get_run("abc123def456")
print(run.data.params)   # dict of all params
print(run.data.metrics)  # dict of latest metric values
print(run.data.tags)     # dict of all tags
print(run.info.status)   # FINISHED / FAILED / RUNNING

# --- get metric history (all steps) ---
history = client.get_metric_history("abc123def456", "train_loss")
steps  = [h.step for h in history]
values = [h.value for h in history]

# --- list artifacts ---
artifacts = client.list_artifacts("abc123def456")
for a in artifacts:
    print(a.path, a.file_size)

# --- download artifact ---
local_path = client.download_artifacts("abc123def456", "model/model.pkl", "/tmp/")

# --- load model directly from run ---
model = mlflow.sklearn.load_model("runs:/abc123def456/model")
predictions = model.predict(X_test)
```

---

## 7. Run Comparison in the UI

The MLflow UI provides four comparison views out of the box:

```mermaid
flowchart LR
    subgraph UI["MLflow UI  → Experiment → Compare Runs"]
        TABLE["Runs Table\nSort/filter by any param or metric\nHighlight best value per column"]
        PARALLEL["Parallel Coordinates\nEach axis = one param or metric\nEach line = one run\nVisualize correlations"]
        SCATTER["Scatter Plot\nparam X vs metric Y\nSpot sweet spots quickly"]
        CONTOUR["Contour Plot\ntwo params → one metric\nFind optimal region"]
    end
```

**Workflow:**
1. Check the runs you want to compare in the table.
2. Click **Compare**.
3. Use **Parallel Coordinates** to find which param combination clusters near high F1.
4. Use **Scatter Plot** to confirm the relationship between one param and the metric.

---

## 8. Model Signature and Input Example

A **signature** defines the expected input/output schema of the model.
Prevents silent type mismatches between training and serving.

```python
from mlflow.models.signature import infer_signature

# infer from training data
signature = infer_signature(X_train, model.predict(X_train))
input_example = X_train.iloc[:5]  # first 5 rows as sample

mlflow.sklearn.log_model(
    model,
    "model",
    signature=signature,
    input_example=input_example,
)
```

The stored `MLmodel` then contains:

```yaml
signature:
  inputs: '[{"name": "age", "type": "long"}, {"name": "balance", "type": "double"}, ...]'
  outputs: '[{"type": "long"}]'
```

MLflow serving validates every prediction request against this schema.
Type mismatch → 400 error before it reaches the model.

---

## 9. Tags — Structured Metadata

Tags are free strings but using consistent keys makes runs queryable:

```python
# built-in MLflow tags (set automatically)
# mlflow.source.type     = "LOCAL" / "NOTEBOOK" / "PROJECT"
# mlflow.source.git.commit = current HEAD SHA
# mlflow.user            = OS username
# mlflow.runName         = run_name kwarg

# recommended custom tags
mlflow.set_tags({
    "mlflow.note.content": "Baseline RF. No feature engineering yet.",
    "dataset.version": "v3",
    "dataset.source": "s3://bucket/data/train_v3.parquet",
    "team": "growth",
    "ticket": "ML-142",
})
```

Filter by tag in search:

```python
mlflow.search_runs(
    filter_string="tags.dataset_version = 'v3' AND tags.team = 'growth'"
)
```

---

## 10. System Metrics (Resource Tracking)

MLflow 2.9+ can track CPU, GPU, memory, and disk automatically:

```python
mlflow.enable_system_metrics_logging()

with mlflow.start_run():
    train(model)
    # system.cpu.utilization, system.memory.usage,
    # system.gpu.0.utilization, system.gpu.0.memory_usage
    # → logged automatically every N seconds
```

Useful for cost estimation and detecting training bottlenecks.

---

## 11. Full End-to-End Example

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import f1_score, roc_auc_score, classification_report
from mlflow.models.signature import infer_signature
import pandas as pd

# --- setup ---
mlflow.set_tracking_uri("http://localhost:5000")
mlflow.set_experiment("churn-prediction")

# --- data ---
df = pd.read_parquet("data/features_v3.parquet")
X, y = df.drop("churn", axis=1), df["churn"]
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# --- train + track ---
params = {
    "n_estimators": 200,
    "learning_rate": 0.05,
    "max_depth": 4,
    "subsample": 0.8,
    "random_state": 42,
}

with mlflow.start_run(run_name="gbt-tuned-v1") as run:
    mlflow.set_tags({
        "dataset.version": "v3",
        "mlflow.note.content": "GBT with subsampling. Tuned depth and lr.",
    })

    mlflow.log_params(params)

    model = GradientBoostingClassifier(**params)

    # log loss curve during training
    for i, staged_pred in enumerate(model.staged_predict(X_train)):
        pass  # real curve: use staged_predict_ after fit or manual loop

    model.fit(X_train, y_train)

    # evaluation
    y_pred = model.predict(X_test)
    y_prob = model.predict_proba(X_test)[:, 1]

    mlflow.log_metrics({
        "f1":        f1_score(y_test, y_pred),
        "auc":       roc_auc_score(y_test, y_prob),
        "train_f1":  f1_score(y_train, model.predict(X_train)),
    })

    # log classification report as artifact
    report = classification_report(y_test, y_pred)
    mlflow.log_text(report, "classification_report.txt")

    # log model with signature + example
    signature = infer_signature(X_train, model.predict(X_train))
    mlflow.sklearn.log_model(
        model, "model",
        signature=signature,
        input_example=X_train.iloc[:3],
    )

    print(f"Run ID : {run.info.run_id}")
    print(f"F1     : {f1_score(y_test, y_pred):.4f}")
    print(f"AUC    : {roc_auc_score(y_test, y_prob):.4f}")
```

---

## 12. Key Takeaways

1. **Experiment → Run → {Params, Metrics, Artifacts, Tags}** is the entire data model.
2. `log_param` once per run. `log_metric` many times with `step=` for curves.
3. `autolog()` captures 90% of what you need with zero extra lines.
4. Model signature + input example = contract between training and serving.
5. Nested runs give structure to sweeps — parent holds summary, children hold per-trial detail.
6. Everything is queryable: `mlflow.search_runs()` returns a DataFrame you can filter and sort.
7. `runs:/<run_id>/<artifact_path>` URI works everywhere — load, serve, register.
