# MLOps Workflow — End-to-End Machine Learning Production Pipeline
### A Complete Industry Reference (Whiteboard-Style Notes)

---

## PART 1 — MLOps FUNDAMENTALS

### What is MLOps?
**Simple:** MLOps = the set of practices that get a machine learning model out of a notebook and into a real product, safely, repeatedly, and without breaking things.

**Technical:** MLOps (Machine Learning Operations) is the discipline of applying DevOps principles — automation, CI/CD, monitoring, versioning — to the ML lifecycle: data, code, and models. It covers everything from data ingestion to training, deployment, monitoring, and retraining.

### Why do we need MLOps?
- Models decay (data drifts, world changes) — a model that worked in Jan may be wrong by June.
- Notebooks aren't reproducible or deployable.
- Manual retraining doesn't scale to 50+ models.
- Regulators (banking/insurance) need audit trails: which model, which data, which version made this decision?
- Teams need to collaborate: DS builds, ML engineers deploy, SRE monitors.

### ML vs MLOps vs DevOps

| | ML | DevOps | MLOps |
|---|---|---|---|
| Focus | Building models | Shipping software | Shipping + maintaining models |
| Artifact | Model (weights) | Code binary | Code + Model + Data |
| Testing | Accuracy metrics | Unit/integration tests | Both + data/model tests |
| Failure mode | Silent (bad predictions) | Loud (crash) | Silent AND loud |
| Versioning | Notebooks (rare) | Git | Git + DVC + Model Registry |
| Change trigger | New idea | Code change | Code, data, OR model change |

### SDLC vs ML Lifecycle
```
SDLC:  Requirements → Design → Code → Test → Deploy → Maintain
ML   :  Problem → Data → EDA → Feature Eng → Train → Evaluate → Deploy → Monitor → Retrain (loop)
```
The key difference: ML lifecycle is a **loop**, not a line. Data changes, so the model must be revisited continuously — SDLC assumes code is mostly static once shipped.

### Data Science workflow vs MLOps workflow
- **DS workflow:** exploratory, notebook-driven, one-off, optimizes for insight/accuracy on a fixed dataset.
- **MLOps workflow:** productionized, script/pipeline-driven, repeatable, optimizes for reliability, latency, and maintainability over time.

### Problems with notebook-only ML projects
- No reproducibility (cells run out of order, hidden state).
- No tests.
- Hard to deploy (no clean function boundaries).
- No versioning of data/model artifacts.
- Not scalable to team collaboration.
- Credentials/paths hardcoded.

### Training vs Inference
- **Training:** learning parameters from historical data — batch, resource-heavy, offline usually.
- **Inference:** using a trained model to predict on new data — must be fast, often online, resource-constrained.

### Experimentation vs Production
| | Experimentation | Production |
|---|---|---|
| Goal | Best accuracy | Reliable, maintainable, fast |
| Code quality | Loose | Tested, typed, logged |
| Data | Static sample | Live, changing |
| Failure cost | Low | High (money/compliance/trust) |

### Reproducibility
Same code + same data + same config ⇒ same model, every time. Achieved via: fixed seeds, pinned dependency versions, versioned data (DVC), versioned code (Git), logged parameters (MLflow).

### Automation
Manual steps (run this notebook, copy that file) are replaced by pipelines (Airflow/GitHub Actions) so nothing depends on a human remembering a sequence.

### Scalability
Pipeline must handle 10x data, 10x models, 10x requests without re-architecting — achieved via containers, cloud infra, feature stores, batch/streaming design.

### Reliability
The system keeps serving correct predictions even under failure — rollback strategies, health checks, retries, redundancy.

### Model Governance
Formal control over what models exist, who approved them, what data they use, and how decisions can be audited/explained — critical in regulated industries.

### Model Lifecycle
```
Ideation → Data Collection → Training → Validation → Registration →
Staging → Production → Monitoring → Drift → Retraining → Archival
```

### Real-world examples
- **Banking:** loan approval models — must be explainable (SHAP) for regulators, versioned for audits.
- **Insurance:** claim fraud scoring — retrained quarterly as fraud patterns shift (concept drift).
- **Credit risk:** PD (probability of default) models — heavy governance, model risk management (MRM) sign-off before production.
- **Fraud detection:** real-time inference (<100ms), heavy monitoring for concept drift as fraudsters adapt.
- **E-commerce:** demand forecasting — batch retraining nightly, feature store for SKU-level features.
- **Recommendation systems:** online inference + feature store for real-time user behavior signals.
- **Healthcare:** diagnostic models — strict governance, explainability mandatory, human-in-the-loop.
- **Manufacturing:** predictive maintenance — streaming sensor data, drift detection on sensor distributions.

**Common interview Q:** *"What is MLOps and why is it different from DevOps?"*
**Strong answer:** MLOps extends DevOps by treating data and models as first-class versioned artifacts alongside code, and by explicitly handling model decay through monitoring, drift detection, and automated retraining — problems that don't exist in traditional software.

---

## PART 2 — COMPLETE MLOps WORKFLOW (Stage by Stage)

```
DATA SOURCE → INGESTION → VALIDATION → CLEANING → PREPROCESSING →
FEATURE ENGINEERING → DATA VERSIONING → MODEL TRAINING →
EXPERIMENT TRACKING → MODEL EVALUATION → MODEL REGISTRY → CI/CD →
CONTAINERIZATION → DEPLOYMENT → INFERENCE (API/BATCH/STREAM) →
MONITORING → DRIFT DETECTION → RETRAINING → RE-REGISTRATION → REDEPLOYMENT (loop)
```

For each stage below: **What / Why / Input / Output / Tools / Libraries / Files / Industry Example / Common Mistakes.** Full code for each is given in Parts 4–23 and the complete project in Part 31 (to avoid repeating identical code twice).

### 1. Data Source
- **What:** Where raw data originates — DBs, APIs, files, streams.
- **Why:** Foundation of the whole pipeline; garbage in → garbage out.
- **Tools:** PostgreSQL, S3, Kafka, REST APIs.
- **Mistake:** Not documenting source schema/ownership.

### 2. Data Ingestion
- **What:** Pulling raw data into the pipeline's storage.
- **Input:** External source. **Output:** raw data files/tables.
- **Tools:** pandas, SQLAlchemy, boto3, PySpark. **File:** `data_ingestion.py`
- **Mistake:** No retry/backoff on network failures; overwriting raw data (should be immutable).

### 3. Data Validation
- **What:** Checking incoming data matches expected schema/ranges before use.
- **Why:** Prevents "silent" pipeline corruption — a null column shouldn't crash training three stages later.
- **Tools:** Great Expectations, Pandera, Pydantic. **File:** `data_validation.py`
- **Mistake:** Validating only at training time, not at inference time (training-serving skew).

### 4. Data Cleaning
- **What:** Fixing/removing bad records — duplicates, corrupt rows, impossible values.
- **Mistake:** Silently dropping too many rows without logging the drop rate.

### 5. Data Preprocessing
- **What:** Transforming cleaned data into a model-ready numeric form — imputation, encoding, scaling.
- **File:** `preprocessing.py`
- **Mistake:** Fitting scalers/encoders on the full dataset before splitting (data leakage).

### 6. Feature Engineering
- **What:** Creating new predictive signals from raw features.
- **File:** `feature_engineering.py`
- **Mistake:** Using future information (leakage) — e.g., using "total lifetime purchases" to predict churn at time T when that total includes purchases after T.

### 7. Data Versioning
- **What:** Snapshotting datasets so any model can be tied back to the exact data it was trained on.
- **Tools:** DVC, Git LFS, cloud object versioning (S3 versioning).
- **Mistake:** Versioning code but not data — makes results irreproducible.

### 8. Model Training
- **What:** Fitting an algorithm to the processed training data.
- **File:** `train.py`. **Tools:** scikit-learn, XGBoost, LightGBM.
- **Mistake:** Not fixing random seeds; tuning hyperparameters on the test set.

### 9. Experiment Tracking
- **What:** Logging every run's parameters, metrics, and artifacts so experiments are comparable.
- **Tool:** MLflow.
- **Mistake:** Not logging the exact preprocessing pipeline alongside the model (skew at inference time).

### 10. Model Evaluation
- **What:** Scoring the model on held-out data with metrics that matter for the business, not just accuracy.
- **File:** `evaluate.py`
- **Mistake:** Using accuracy on an imbalanced fraud dataset (99% "not fraud" gives 99% accuracy doing nothing).

### 11. Model Registry
- **What:** Central catalog of model versions with stage labels (Staging/Production/Archived).
- **Tool:** MLflow Model Registry.
- **Mistake:** No promotion approval process — anyone can push to Production.

### 12. CI/CD
- **What:** Automated pipeline that tests and deploys code+model changes on every push.
- **Tool:** GitHub Actions. **File:** `.github/workflows/ci-cd.yml`
- **Mistake:** No automated tests before deploy — CI that only builds, never tests.

### 13. Containerization
- **What:** Packaging the app + model + dependencies into a portable image.
- **Tool:** Docker. **File:** `Dockerfile`
- **Mistake:** Huge images (no multi-stage build), baking secrets into the image.

### 14. Model Deployment
- **What:** Making the trained model available to serve predictions.
- **Tools:** FastAPI + Docker + AWS ECS/SageMaker/K8s.
- **Mistake:** Deploying without a rollback plan.

### 15. Inference (API / Batch / Streaming)
- **What:** The actual serving pattern — synchronous API call, scheduled batch job, or continuous stream processing.
- **Mistake:** Choosing real-time API for a use case that only needed nightly batch (unnecessary cost/complexity).

### 16. Monitoring
- **What:** Tracking system health (latency, errors) AND model health (drift, accuracy decay) in production.
- **Tools:** Prometheus, Grafana, CloudWatch, Evidently AI.
- **Mistake:** Monitoring infra metrics only, ignoring prediction distribution.

### 17. Drift Detection
- **What:** Statistically detecting when incoming data or model performance has shifted from training conditions.
- **Tools:** PSI, KS-test, Evidently.
- **Mistake:** No defined threshold/action — drift is detected but nothing happens.

### 18. Retraining
- **What:** Automatically or semi-automatically producing a new model version when drift/decay is detected.
- **Mistake:** Retraining on data that already contains the drift-causing bug (garbage back in).

### 19. Model Re-registration
- **What:** New model version logged to the registry, evaluated against the current production model (champion/challenger).
- **Mistake:** Auto-promoting new model without human sign-off in regulated domains.

### 20. Redeployment
- **What:** Rolling the new model into production — ideally via blue/green or canary deployment.
- **Mistake:** Full traffic cutover with no canary — one bad model version affects 100% of users immediately.

**Interview Q:** *"Walk me through what happens when you push new code to your ML repo."*
**Answer shape:** code push → CI triggers → lint + unit tests + data/model tests → build Docker image → push to registry (ECR) → CD deploys to staging → smoke tests → promote to production → monitoring picks up new version's metrics.

---

## PART 3 — PROJECT FOLDER STRUCTURE

```
mlops-project/
│
├── data/
│   ├── raw/            # immutable, exactly as ingested
│   ├── processed/      # cleaned/transformed, reproducible from raw
│   └── external/       # third-party reference data
│
├── notebooks/           # EDA / experimentation ONLY — never production logic
│   ├── EDA.ipynb
│   └── experimentation.ipynb
│
├── src/                 # production Python modules
│   ├── data_ingestion.py
│   ├── data_validation.py
│   ├── data_preprocessing.py
│   ├── feature_engineering.py
│   ├── train.py
│   ├── evaluate.py
│   └── predict.py
│
├── tests/
│   ├── test_data.py
│   ├── test_model.py
│   └── test_api.py
│
├── configs/
│   └── config.yaml       # all tunables — no hardcoding
│
├── models/                # serialized trained models (often git-ignored, DVC/registry-tracked)
├── artifacts/              # plots, reports, encoders, scalers
│
├── app/
│   └── main.py             # FastAPI serving app
│
├── Dockerfile
├── requirements.txt
├── .gitignore
├── README.md
├── .github/workflows/ci-cd.yml
└── docker-compose.yml
```

**Why each matters:**
- `data/raw` vs `data/processed`: raw is a source of truth, never edited in place — this alone prevents most "why don't my numbers reproduce" bugs.
- `notebooks/` separate from `src/`: notebooks are for exploration; anything that runs in production must be a tested `.py` module. Mixing the two is the #1 sign of an unmature ML project.
- `configs/config.yaml`: single place to change hyperparameters/paths without touching code — enables CI to inject different configs for staging vs prod.
- `tests/`: mirrors `src/` — every production module should have a corresponding test file.
- `artifacts/`: holds the *fitted preprocessing objects* (scaler.pkl, encoder.pkl) — these must travel with the model to avoid training-serving skew.

**Alternative industry structures:**
- **Cookiecutter Data Science** layout — similar, adds `references/`, `reports/figures/`.
- **Monorepo per-model** — `models/credit_risk/`, `models/fraud/`, each with its own src/tests/config, shared `common/` library.
- **Kubeflow-style** — pipeline defined as a DAG of containerized "components," each with its own Dockerfile, no single `src/` folder.

---

## PART 4 — DATA INGESTION

**Sources:** CSV/Excel (local batch), SQL (transactional data), APIs (third-party/live), cloud storage (S3/GCS/Blob — data lakes), data warehouses (Snowflake/BigQuery — curated analytics data), streaming (Kafka/Kinesis — continuous events).

**Flow:** `Database → Python (pandas/SQLAlchemy) → Processed data (parquet/csv) → Cloud storage (S3)`

```python
# src/data_ingestion.py
"""Ingest raw data from multiple sources into data/raw/."""
import logging
import pandas as pd
from sqlalchemy import create_engine
import boto3
import requests
import yaml

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def load_config(path="configs/config.yaml"):
    with open(path) as f:
        return yaml.safe_load(f)


def ingest_from_csv(path: str) -> pd.DataFrame:
    logger.info(f"Reading CSV from {path}")
    return pd.read_csv(path)


def ingest_from_sql(conn_string: str, query: str) -> pd.DataFrame:
    engine = create_engine(conn_string)
    logger.info("Querying database")
    with engine.connect() as conn:
        return pd.read_sql(query, conn)


def ingest_from_api(url: str, params: dict = None) -> pd.DataFrame:
    resp = requests.get(url, params=params, timeout=30)
    resp.raise_for_status()
    return pd.DataFrame(resp.json())


def ingest_from_s3(bucket: str, key: str, local_path: str) -> pd.DataFrame:
    s3 = boto3.client("s3")
    s3.download_file(bucket, key, local_path)
    return pd.read_parquet(local_path)


def save_raw(df: pd.DataFrame, output_path: str):
    df.to_parquet(output_path, index=False)
    logger.info(f"Saved {len(df)} rows to {output_path}")


if __name__ == "__main__":
    cfg = load_config()
    df = ingest_from_csv(cfg["data"]["raw_csv_path"])
    save_raw(df, cfg["data"]["raw_output_path"])
```
**Industry example:** a credit-risk pipeline pulls applicant data nightly from a PostgreSQL transactional DB (`ingest_from_sql`) and merges with bureau data pulled from a third-party API.

**Common mistakes:** mutating `data/raw` in place; no logging of row counts (silent data loss goes unnoticed); hardcoded credentials instead of env vars.

**Interview Q:** *"How do you ingest data reliably from an unstable third-party API?"* → retries with exponential backoff, timeouts, schema validation immediately after ingestion, idempotent writes (don't duplicate on re-run).

---

## PART 5 — DATA VALIDATION

Checks: schema (column names/types), missing values, duplicates, outliers, range checks (age between 0–120), category checks (categorical values in an allowed set), null checks, distribution checks (compare to a reference), overall data quality score.

```python
# src/data_validation.py
import pandera as pa
from pandera import Column, Check, DataFrameSchema
import pandas as pd

schema = DataFrameSchema({
    "age": Column(int, Check.in_range(18, 100), nullable=False),
    "income": Column(float, Check.greater_than(0), nullable=False),
    "loan_amount": Column(float, Check.greater_than(0), nullable=False),
    "employment_type": Column(str, Check.isin(["salaried", "self_employed", "unemployed"])),
    "default": Column(int, Check.isin([0, 1]), nullable=False),
})


def validate(df: pd.DataFrame) -> pd.DataFrame:
    """Raises SchemaError if data violates the contract."""
    return schema.validate(df, lazy=True)  # lazy=True collects ALL errors, not just the first


# --- Great Expectations style (conceptual) ---
# import great_expectations as ge
# ge_df = ge.from_pandas(df)
# ge_df.expect_column_values_to_not_be_null("income")
# ge_df.expect_column_values_to_be_between("age", 18, 100)
# results = ge_df.validate()

# --- Pydantic for single-record (API request) validation ---
from pydantic import BaseModel, Field, field_validator

class LoanApplication(BaseModel):
    age: int = Field(ge=18, le=100)
    income: float = Field(gt=0)
    loan_amount: float = Field(gt=0)
    employment_type: str

    @field_validator("employment_type")
    @classmethod
    def check_employment(cls, v):
        allowed = {"salaried", "self_employed", "unemployed"}
        if v not in allowed:
            raise ValueError(f"employment_type must be one of {allowed}")
        return v
```
**Why Pandera vs Pydantic:** Pandera validates whole DataFrames (batch/training time); Pydantic validates single JSON records (API/inference time) — you typically need **both**, using the same rules, to avoid training-serving skew.

**Mistake:** validating only training data — the #1 cause of production 422/500 errors is unvalidated inference input.

---

## PART 6 — DATA PREPROCESSING

Covers: missing value treatment, encoding, scaling, outlier treatment, feature selection, train/test split, cross-validation, data leakage.

```python
# src/data_preprocessing.py
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
import joblib

NUMERIC_FEATURES = ["age", "income", "loan_amount"]
CATEGORICAL_FEATURES = ["employment_type"]


def build_preprocessing_pipeline() -> ColumnTransformer:
    numeric_pipeline = Pipeline([
        ("imputer", SimpleImputer(strategy="median")),
        ("scaler", StandardScaler()),
    ])
    categorical_pipeline = Pipeline([
        ("imputer", SimpleImputer(strategy="most_frequent")),
        ("encoder", OneHotEncoder(handle_unknown="ignore")),
    ])
    return ColumnTransformer([
        ("num", numeric_pipeline, NUMERIC_FEATURES),
        ("cat", categorical_pipeline, CATEGORICAL_FEATURES),
    ])


def split_data(df: pd.DataFrame, target: str, test_size=0.2, random_state=42):
    X = df.drop(columns=[target])
    y = df[target]
    # stratify on target for classification — keeps class ratio consistent
    return train_test_split(X, y, test_size=test_size, random_state=random_state, stratify=y)


if __name__ == "__main__":
    df = pd.read_parquet("data/processed/clean.parquet")
    X_train, X_test, y_train, y_test = split_data(df, target="default")

    preprocessor = build_preprocessing_pipeline()
    # FIT ONLY ON TRAIN — this is the #1 leakage prevention rule
    X_train_transformed = preprocessor.fit_transform(X_train)
    X_test_transformed = preprocessor.transform(X_test)   # transform only, never fit

    joblib.dump(preprocessor, "artifacts/preprocessor.pkl")
```
**Why reproducibility matters here:** the *fitted* preprocessor (with learned means/std-devs/categories) must be saved and reused at inference — refitting it on new data at inference time is a classic bug that silently changes feature scales.

**Data leakage — the concept, explained simply:** any information the model sees during training that it wouldn't legitimately have at prediction time. Classic case: scaling on the full dataset before splitting means the test set's statistics "leak" into the training transformation, giving optimistic — and wrong — evaluation numbers.

---

## PART 7 — FEATURE ENGINEERING

```
Raw Data → Transformations → Features → Model
```

Types: numerical (ratios, log transforms, binning), categorical (encoding, target encoding), date/time (day-of-week, recency), aggregations (rolling averages, group-by stats), interaction features (feature A × feature B).

```python
# src/feature_engineering.py
import pandas as pd
import numpy as np


def add_features(df: pd.DataFrame) -> pd.DataFrame:
    df = df.copy()
    # Ratio feature
    df["loan_to_income_ratio"] = df["loan_amount"] / df["income"].replace(0, np.nan)
    # Log transform (handles skewed income distributions)
    df["log_income"] = np.log1p(df["income"])
    # Binning
    df["age_bucket"] = pd.cut(df["age"], bins=[18, 25, 35, 50, 65, 100],
                               labels=["18-25", "26-35", "36-50", "51-65", "65+"])
    # Date/time feature (example: application_date column)
    if "application_date" in df.columns:
        df["application_date"] = pd.to_datetime(df["application_date"])
        df["application_month"] = df["application_date"].dt.month
        df["application_dow"] = df["application_date"].dt.dayofweek
    return df


# Full pipeline combining feature engineering + preprocessing in ONE sklearn object
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import FunctionTransformer
from src.data_preprocessing import build_preprocessing_pipeline

full_pipeline = Pipeline([
    ("feature_engineering", FunctionTransformer(add_features)),
    ("preprocessing", build_preprocessing_pipeline()),
])
```
**Why wrap in a `Pipeline`:** a single serialized object (`full_pipeline`) can be `fit()` once and `predict()`-safely reused at both training and inference — eliminating an entire class of skew bugs where feature-engineering code diverges between the training script and the serving app.

---

## PART 8 — DATA VERSIONING (DVC)

**Why needed:** Git handles code well but is bad at large binary/data files. Without data versioning, you can reproduce the code for a model but not the exact data it saw — which makes debugging a production issue ("why did v3 behave differently from v2?") nearly impossible, and is often a compliance requirement in banking.

```bash
dvc init                       # sets up .dvc/ folder, like git init
dvc add data/raw/dataset.csv   # creates dataset.csv.dvc pointer file, moves real data to cache
git add data/raw/dataset.csv.dvc .gitignore
git commit -m "Track dataset v1 with DVC"

dvc remote add -d storage s3://my-bucket/dvc-store
dvc push                       # uploads actual data to S3 remote
dvc pull                       # downloads data referenced by the .dvc pointer file
```
**What happens internally:** `dvc add` computes a content hash (MD5) of the file, stores the real file in `.dvc/cache` (and later a remote like S3), and writes a tiny human-readable `.dvc` pointer file (containing the hash + size) that Git *can* track cheaply. Checking out an old Git commit checks out the old `.dvc` pointer, and `dvc checkout`/`dvc pull` fetches the matching data version — giving you Git-like history for multi-GB datasets without bloating the Git repo.

**Dataset lineage:** DVC pipelines (`dvc.yaml`) can chain stages (`ingest → validate → preprocess → train`) so you can trace any model back to the exact raw data and code that produced it.

---

## PART 9 — GIT & VERSION CONTROL

| Concept | Meaning |
|---|---|
| Repository | Project's tracked folder + full history |
| Commit | A saved snapshot with a message |
| Branch | An independent line of development |
| Merge | Combining branch history back together |
| Pull Request | Proposed merge, reviewed before accepted |
| `.gitignore` | Files Git should never track (secrets, `__pycache__`, large data) |
| Tag | A named pointer to a specific commit (e.g., `v1.0.0`) |
| Release | A packaged, published version, usually tied to a tag |

```bash
git init
git add .
git commit -m "Initial commit"
git branch feature/add-fraud-model
git checkout feature/add-fraud-model
git push origin feature/add-fraud-model
git pull origin main
git merge feature/add-fraud-model
```

**Typical ML project Git workflow:**
```
main (protected, always deployable)
  └── develop
        └── feature/train-xgboost-model   → PR → code review → CI passes → merge → develop
        └── feature/add-drift-monitor
```
Model files themselves are usually **not** committed to Git directly — they're tracked via DVC or logged to the MLflow Model Registry, with Git holding only the training *code* and a pointer/run ID.

---

## PART 10 — MODEL TRAINING

Key terms: **parameters** are learned from data (e.g., regression coefficients); **hyperparameters** are set before training (e.g., `n_estimators`, `learning_rate`); **cross-validation** estimates generalization by training/testing on multiple folds; **baseline model** is a simple reference (e.g., "always predict majority class") every real model must beat.

```python
# src/train.py
import joblib
import yaml
import pandas as pd
from xgboost import XGBClassifier
from sklearn.model_selection import cross_val_score
from src.feature_engineering import full_pipeline

def load_config(path="configs/config.yaml"):
    with open(path) as f:
        return yaml.safe_load(f)

def train():
    cfg = load_config()
    df = pd.read_parquet("data/processed/clean.parquet")
    X = df.drop(columns=[cfg["target"]])
    y = df[cfg["target"]]

    X_transformed = full_pipeline.fit_transform(X)

    model = XGBClassifier(
        n_estimators=cfg["model"]["n_estimators"],
        max_depth=cfg["model"]["max_depth"],
        learning_rate=cfg["model"]["learning_rate"],
        random_state=42,          # reproducibility
        eval_metric="logloss",
    )

    scores = cross_val_score(model, X_transformed, y, cv=5, scoring="roc_auc")
    print(f"CV ROC-AUC: {scores.mean():.4f} +/- {scores.std():.4f}")

    model.fit(X_transformed, y)

    joblib.dump(model, "models/model.pkl")           # joblib: efficient for numpy-heavy sklearn/xgboost objects
    joblib.dump(full_pipeline, "artifacts/pipeline.pkl")
    return model

if __name__ == "__main__":
    train()
```
**`joblib` vs `pickle`:** both serialize Python objects; `joblib` is optimized for objects containing large numpy arrays (most sklearn/xgboost models) — faster and more memory-efficient for that case. Use `pickle` for generic small Python objects; use `joblib` for ML models/pipelines.

**Pipeline:** `load data → preprocess (fit on train) → train → evaluate → save model + save pipeline together` — always save them together, never separately.

---

## PART 11 — EXPERIMENT TRACKING (MLflow)

**Why:** Without tracking, "which run gave the best F1 with which hyperparameters" lives only in someone's memory or a messy spreadsheet. MLflow logs every run automatically and makes runs comparable/searchable.

**MLflow core concepts:** an **Experiment** groups related **Runs**; each Run logs **parameters** (hyperparameters), **metrics** (accuracy, AUC), and **artifacts** (plots, the model file itself); the **Model Registry** sits on top, versioning models promoted from good runs.

```python
# train_with_mlflow.py
import mlflow
import mlflow.sklearn
from xgboost import XGBClassifier
from sklearn.metrics import roc_auc_score, f1_score

mlflow.set_experiment("credit-risk-model")

with mlflow.start_run(run_name="xgb-v1"):
    params = {"n_estimators": 200, "max_depth": 5, "learning_rate": 0.05}
    mlflow.log_params(params)

    model = XGBClassifier(**params, random_state=42)
    model.fit(X_train, y_train)

    preds = model.predict_proba(X_test)[:, 1]
    auc = roc_auc_score(y_test, preds)
    f1 = f1_score(y_test, model.predict(X_test))

    mlflow.log_metric("roc_auc", auc)
    mlflow.log_metric("f1", f1)

    mlflow.sklearn.log_model(model, artifact_path="model",
                              registered_model_name="credit_risk_xgb")
```
**MLflow architecture:** a **Tracking Server** (backend store: SQL DB for metadata + artifact store: S3/local for files) receives logs from client runs; the **Model Registry** is a layer on the tracking server's DB that adds versioning + stage transitions (None → Staging → Production → Archived); a **UI** lets you compare runs visually.

---

## PART 12 — MODEL EVALUATION

**Regression:** MAE (average absolute error, interpretable in original units), MSE (penalizes large errors more), RMSE (same units as target, penalizes large errors), R² (variance explained), MAPE (percentage error, unstable near zero).

**Classification:** Accuracy (correct/total — misleading on imbalance), Precision (of predicted positives, how many were right — matters when false positives are costly), Recall (of actual positives, how many were caught — matters when false negatives are costly), F1 (harmonic mean of precision/recall), ROC-AUC (ranking quality across thresholds), PR-AUC (better than ROC-AUC for imbalanced classes), Log Loss (penalizes confident wrong predictions).

```python
# src/evaluate.py
from sklearn.metrics import (confusion_matrix, classification_report,
                              roc_auc_score, average_precision_score)
import numpy as np

def evaluate(model, X_test, y_test, threshold=0.5):
    probs = model.predict_proba(X_test)[:, 1]
    preds = (probs >= threshold).astype(int)

    print(confusion_matrix(y_test, preds))
    print(classification_report(y_test, preds))
    print("ROC-AUC:", roc_auc_score(y_test, probs))
    print("PR-AUC:", average_precision_score(y_test, probs))
    return {"roc_auc": roc_auc_score(y_test, probs)}


def find_optimal_threshold(y_test, probs, cost_fp=1, cost_fn=5):
    """Business-metric-driven threshold tuning — for credit risk, a missed
    default (false negative) usually costs far more than an unnecessary
    rejection (false positive)."""
    best_threshold, best_cost = 0.5, float("inf")
    for t in np.arange(0.1, 0.9, 0.01):
        preds = (probs >= t).astype(int)
        fp = ((preds == 1) & (y_test == 0)).sum()
        fn = ((preds == 0) & (y_test == 1)).sum()
        cost = fp * cost_fp + fn * cost_fn
        if cost < best_cost:
            best_cost, best_threshold = cost, t
    return best_threshold
```
**Why accuracy misleads on fraud/credit risk:** if 1% of transactions are fraud, a model that predicts "not fraud" for everything scores 99% accuracy while catching zero fraud. **PR-AUC and recall at a fixed precision (or a cost-weighted threshold)** are the metrics that actually matter for the business.

---

## PART 13 — MODEL REGISTRY

```
Model v1 → Model v2 → Evaluation (v2 beats v1 on holdout) → Best model → Production
```
Registry states: **Development/None** (just logged) → **Staging** (candidate under test) → **Production** (actively serving) → **Archived** (retired, kept for audit/rollback).

```python
from mlflow import MlflowClient
client = MlflowClient()

# Register a run's model
result = mlflow.register_model("runs:/<run_id>/model", "credit_risk_xgb")

# Promote to staging, then production after validation
client.transition_model_version_stage(
    name="credit_risk_xgb", version=result.version, stage="Staging"
)
client.transition_model_version_stage(
    name="credit_risk_xgb", version=result.version, stage="Production",
    archive_existing_versions=True   # old prod version auto-archived, enabling rollback
)

# Rollback: just transition an older archived version back to Production
client.transition_model_version_stage(
    name="credit_risk_xgb", version=3, stage="Production"
)
```
**Rollback** is exactly this: the registry keeps every version, so reverting to a known-good model is a metadata change, not a retraining job.

---

## PART 14 — CONTAINERIZATION (Docker)

**Concepts:** an **Image** is a read-only template (app + dependencies + OS layer); a **Container** is a running instance of an image; a **Dockerfile** is the recipe to build an image; a **Port** exposes a container's network endpoint to the host; a **Volume** persists data outside the container's writable layer; **environment variables** inject config/secrets at runtime without baking them into the image.

```dockerfile
# Dockerfile
FROM python:3.11-slim AS base

WORKDIR /app

# Install deps first (better layer caching — only reinstalls when requirements.txt changes)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/
COPY app/ ./app/
COPY models/ ./models/
COPY artifacts/ ./artifacts/
COPY configs/ ./configs/

ENV PYTHONUNBUFFERED=1
EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```
Dockerfile → docker build → Docker Image → docker run → Container → ML API (listening on port 8000)
```

```bash
docker build -t credit-risk-api:latest .
docker run -p 8000:8000 -e MODEL_PATH=/app/models/model.pkl credit-risk-api:latest
docker ps                 # list running containers
docker stop <container_id>
docker images             # list local images
docker push myrepo/credit-risk-api:latest
```

**Common Docker errors:**
| Error | Cause | Fix |
|---|---|---|
| `port is already allocated` | Another process/container using the port | `docker ps` to find it, stop it, or map a different host port |
| Huge image size | Copying `.git`, `data/`, `notebooks/` into image | Use `.dockerignore`; multi-stage builds |
| `ModuleNotFoundError` inside container | requirements.txt not fully synced | Rebuild without cache: `docker build --no-cache` |
| Container exits immediately | CMD process crashes on start | `docker logs <id>` to see the traceback |

---

## PART 15 — REST API FOR ML MODEL (FastAPI)

**Concepts:** API = a contract for programs to talk to each other; Endpoint = a specific URL+method combo; GET = read data; POST = send data (used for predictions since input is a payload, not a URL param); JSON = the data format exchanged; status codes: `200` OK, `422` validation error, `500` server error.

```
JSON input → FastAPI → Preprocessing (load saved pipeline) → Model → Prediction → JSON response
```

```python
# app/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
import joblib
import pandas as pd
import logging

logger = logging.getLogger("credit_risk_api")
app = FastAPI(title="Credit Risk Prediction API")

# Load model + pipeline ONCE at startup, not per-request (huge latency saver)
model = joblib.load("models/model.pkl")
pipeline = joblib.load("artifacts/pipeline.pkl")


class LoanRequest(BaseModel):
    age: int = Field(ge=18, le=100)
    income: float = Field(gt=0)
    loan_amount: float = Field(gt=0)
    employment_type: str


class PredictionResponse(BaseModel):
    default_probability: float
    risk_band: str


@app.get("/health")
def health():
    return {"status": "ok"}


@app.post("/predict", response_model=PredictionResponse)
def predict(request: LoanRequest):
    try:
        df = pd.DataFrame([request.model_dump()])
        X = pipeline.transform(df)
        prob = float(model.predict_proba(X)[0, 1])
        band = "high" if prob > 0.6 else "medium" if prob > 0.3 else "low"
        return PredictionResponse(default_probability=prob, risk_band=band)
    except Exception as e:
        logger.exception("Prediction failed")
        raise HTTPException(status_code=500, detail=str(e))
```
**Pydantic schemas** (`LoanRequest`) do double duty: automatic request validation (bad input → `422` before your code even runs) and automatic OpenAPI docs generation at `/docs`.

---

## PART 16 — CI/CD FOR MACHINE LEARNING

```
Developer pushes code → GitHub → CI starts → Install deps → Lint → Unit tests →
Build Docker image → Push image → Deploy → Production
```

```yaml
# .github/workflows/ci-cd.yml
name: CI-CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4               # pulls repo code onto the runner

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Lint
        run: flake8 src/ app/

      - name: Run tests
        run: pytest tests/ --maxfail=1 --disable-warnings

  build-and-deploy:
    needs: test                                   # only runs if tests passed
    if: github.ref == 'refs/heads/main'            # only deploy from main
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t credit-risk-api:${{ github.sha }} .

      - name: Log in to ECR
        run: aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}

      - name: Push image
        run: |
          docker tag credit-risk-api:${{ github.sha }} ${{ secrets.ECR_REGISTRY }}/credit-risk-api:latest
          docker push ${{ secrets.ECR_REGISTRY }}/credit-risk-api:latest

      - name: Deploy
        run: aws ecs update-service --cluster ml-cluster --service credit-risk-api --force-new-deployment
```
**Line-by-line intent:** the `test` job gates everything — nothing gets built or deployed if tests fail; `needs: test` enforces that ordering; `if: github.ref == 'refs/heads/main'` prevents feature branches from deploying to production; secrets (`ECR_REGISTRY`) are injected from GitHub's encrypted secrets store, never hardcoded.

---

## PART 17 — TESTING IN MLOPS

```python
# tests/test_data.py
import pandas as pd
from src.data_validation import schema

def test_schema_accepts_valid_data():
    df = pd.DataFrame({
        "age": [30], "income": [50000.0], "loan_amount": [10000.0],
        "employment_type": ["salaried"], "default": [0],
    })
    schema.validate(df)  # should not raise

def test_schema_rejects_negative_income():
    df = pd.DataFrame({
        "age": [30], "income": [-1.0], "loan_amount": [10000.0],
        "employment_type": ["salaried"], "default": [0],
    })
    import pytest
    with pytest.raises(Exception):
        schema.validate(df)
```
```python
# tests/test_model.py
import joblib
import numpy as np

def test_model_predicts_probability_range():
    model = joblib.load("models/model.pkl")
    pipeline = joblib.load("artifacts/pipeline.pkl")
    import pandas as pd
    df = pd.DataFrame([{"age": 35, "income": 60000, "loan_amount": 15000,
                         "employment_type": "salaried"}])
    X = pipeline.transform(df)
    probs = model.predict_proba(X)
    assert probs.shape[1] == 2
    assert np.all((probs >= 0) & (probs <= 1))

def test_model_beats_baseline():
    # regression test: a new model must not underperform the last known-good AUC
    from sklearn.metrics import roc_auc_score
    # ... load X_test, y_test, model ...
    # auc = roc_auc_score(y_test, model.predict_proba(X_test)[:, 1])
    # assert auc > 0.75   # minimum acceptable bar
    pass
```
```python
# tests/test_api.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_health():
    resp = client.get("/health")
    assert resp.status_code == 200

def test_predict_valid_input():
    payload = {"age": 35, "income": 60000, "loan_amount": 15000, "employment_type": "salaried"}
    resp = client.post("/predict", json=payload)
    assert resp.status_code == 200
    assert 0.0 <= resp.json()["default_probability"] <= 1.0

def test_predict_invalid_input_returns_422():
    payload = {"age": 5, "income": 60000, "loan_amount": 15000, "employment_type": "salaried"}
    resp = client.post("/predict", json=payload)
    assert resp.status_code == 422
```
**Testing taxonomy:** unit (a single function), integration (multiple components together, e.g., pipeline→model), API testing (HTTP contract), data testing (schema/quality), model testing (predictions sane, performance above bar — "regression testing" for models), smoke testing (does the deployed service even start and answer `/health`).

---

## PART 18 — CLOUD DEPLOYMENT (AWS)

| Service | Role |
|---|---|
| S3 | Object storage — raw data, model artifacts, DVC remote |
| IAM | Access control — who/what can touch which resource |
| EC2 | Raw virtual machines — full control, more ops overhead |
| ECR | Docker image registry (private) |
| Lambda | Serverless functions — great for low-traffic/event-driven inference |
| API Gateway | Exposes Lambda/services as public REST endpoints |
| CloudWatch | Logs, metrics, alarms |
| SageMaker | Managed ML platform — training, hosting, pipelines, built-in monitoring |

```
Data → S3 → Training (SageMaker/EC2) → Model → ECR/Model Registry →
SageMaker Endpoint or EC2/ECS → API Gateway → Prediction → CloudWatch
```

**When to use each:** small/simple, infrequent inference → **Lambda + API Gateway** (pay per call, zero idle cost); steady moderate traffic, need full control → **EC2/ECS + Docker**; heavy ML-specific tooling (built-in training jobs, endpoints, monitoring, autoscaling) → **SageMaker**, at higher cost/complexity.

---

## PART 19 — MODEL SERVING PATTERNS

| | Batch | Real-time | Streaming |
|---|---|---|---|
| Trigger | Scheduled job | Synchronous API call | Continuous event stream |
| Latency need | Minutes-hours OK | <100ms-1s | Near-continuous, per-event |
| Example | Nightly churn scoring | Credit application decision | Fraud scoring per transaction |
| Infra | Airflow + batch compute | FastAPI + load balancer | Kafka + streaming consumer |

**Credit scoring example (real-time):**
```
Customer application → API (FastAPI) → Model → Credit risk score → Decision engine (approve/reject/review)
```

---

## PART 20 — MONITORING

**Why:** a model's accuracy at deployment time tells you nothing about its accuracy six months later — the world (and the data) keeps changing. Monitoring is how you find out *before* it costs real money.

**System metrics:** latency, throughput (requests/sec), error rate, CPU, memory.
**Model metrics:** prediction distribution (are outputs suddenly all "low risk"?), data drift, concept drift, feature drift, live performance (once ground-truth labels arrive, e.g., did the loan actually default?).

**Definitions:**
- **Data Drift** — the distribution of input features (`P(X)`) changes.
- **Concept Drift** — the relationship between inputs and target (`P(Y|X)`) changes, even if inputs look the same.
- **Prediction Drift** — the distribution of the model's *output* shifts, often an early warning sign of one of the above.

---

## PART 21 — DATA DRIFT

**Example:** Training data average customer age = 35. Production average customer age = 47. That shift alone doesn't prove the model is wrong, but it means the model is now extrapolating outside the range it learned well — worth investigating.

**Methods:** PSI (Population Stability Index — standard banking-industry drift metric), KS-test (Kolmogorov-Smirnov, compares two distributions), simple distribution comparison (histograms, means/stds side by side).

```python
import numpy as np

def calculate_psi(expected, actual, buckets=10):
    """PSI < 0.1: no significant shift. 0.1-0.25: moderate shift, watch.
    > 0.25: major shift, investigate/retrain."""
    breakpoints = np.percentile(expected, np.linspace(0, 100, buckets + 1))
    breakpoints[0], breakpoints[-1] = -np.inf, np.inf

    expected_pct = np.histogram(expected, breakpoints)[0] / len(expected)
    actual_pct = np.histogram(actual, breakpoints)[0] / len(actual)

    # avoid division/log by zero
    expected_pct = np.where(expected_pct == 0, 1e-6, expected_pct)
    actual_pct = np.where(actual_pct == 0, 1e-6, actual_pct)

    psi = np.sum((actual_pct - expected_pct) * np.log(actual_pct / expected_pct))
    return psi


from scipy.stats import ks_2samp

def ks_drift_test(expected, actual, alpha=0.05):
    stat, p_value = ks_2samp(expected, actual)
    return p_value < alpha   # True = statistically significant drift detected
```

---

## PART 22 — CONCEPT DRIFT

```
P(X) changes         → Data Drift      (inputs look different)
P(Y|X) changes        → Concept Drift   (same inputs, different correct answer)
```
**Examples:**
- **Fraud detection:** fraudsters adapt their tactics — a transaction pattern that was safe last quarter is now a common fraud pattern (`P(Y|X)` shifted, `P(X)` may look identical).
- **Credit risk:** a recession changes what "safe" income/debt ratios mean — same applicant profile, different true default risk.
- **Insurance:** new regulation changes claim behavior.
- **E-commerce:** a competitor's price war changes what "will this customer churn" looks like for the same behavioral signals.

**Detect:** monitor live model performance against ground truth as labels arrive (delayed feedback loop); monitor prediction confidence/distribution as an early proxy when labels are delayed.
**Respond:** trigger retraining; in the interim, may need to widen decision thresholds or add human review for the affected segment.

---

## PART 23 — RETRAINING PIPELINE

```
Monitoring → Drift detected → Trigger → New data pulled → Validation →
Retraining → Evaluation (vs current prod model) → Model Registry →
Approval (human sign-off in regulated domains) → Deployment
```

```python
# retrain_pipeline.py (simplified orchestration logic)
def retraining_pipeline(psi_threshold=0.25):
    drift_score = calculate_psi(reference_data["income"], live_data["income"])
    if drift_score < psi_threshold:
        print("No significant drift — skipping retrain")
        return

    new_data = pull_latest_labeled_data()
    validate(new_data)                      # same schema checks as Part 5

    new_model = train_on(new_data)           # same train.py logic
    new_metrics = evaluate(new_model, holdout_data)

    current_prod_metrics = get_production_model_metrics()
    if new_metrics["roc_auc"] > current_prod_metrics["roc_auc"]:
        register_and_promote(new_model)       # champion/challenger promotion
    else:
        flag_for_manual_review(new_model, new_metrics)
```

---

## PART 24 — ORCHESTRATION

**Concepts:** a **Workflow** is a set of tasks with dependencies; a **DAG** (Directed Acyclic Graph) represents that as tasks + arrows with no cycles; **Task** = one unit of work; **Scheduling** = when a DAG runs (cron-like); **Retries** = automatic re-attempt on transient failure; **Failure handling** = alerting/branching logic when a task fails permanently.

**Tools:** Airflow (mature, Python-based, huge ecosystem), Prefect (more modern Python-native API, dynamic workflows), Kubeflow Pipelines (Kubernetes-native, container-per-step, ML-specific).

```python
# Airflow DAG example
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG("credit_risk_retraining", schedule_interval="@weekly",
         start_date=datetime(2026, 1, 1), catchup=False) as dag:

    ingest = PythonOperator(task_id="ingest_data", python_callable=ingest_from_sql)
    validate = PythonOperator(task_id="validate_data", python_callable=validate)
    train = PythonOperator(task_id="train_model", python_callable=train)
    evaluate = PythonOperator(task_id="evaluate_model", python_callable=evaluate)

    ingest >> validate >> train >> evaluate   # defines the DAG edges (dependencies)
```
```
Task A (ingest) → Task B (validate) → Task C (train) → Task D (evaluate)
```

---

## PART 25 — KUBERNETES (as much as MLOps needs)

| Concept | Meaning |
|---|---|
| Pod | Smallest deployable unit — one or more containers sharing network/storage |
| Node | A physical/virtual machine running pods |
| Cluster | A set of nodes managed together |
| Deployment | Declarative spec: "keep N replicas of this pod running" |
| Service | Stable network endpoint routing to a set of pods |
| Replica | One running copy of a pod, for scaling/redundancy |
| Scaling | Adding/removing replicas based on load (HPA = Horizontal Pod Autoscaler) |
| ConfigMap | Non-secret config injected into pods |
| Secret | Sensitive config (API keys, DB passwords) injected securely |

**ML model on K8s (conceptual):** Docker image (Part 14) → pushed to a registry → a `Deployment` YAML tells K8s to run N replicas of that image → a `Service` load-balances traffic across those replicas → an `Ingress`/API Gateway exposes it externally → `HorizontalPodAutoscaler` scales replicas up during traffic spikes.

---

## PART 26 — KUBEFLOW

Kubeflow = ML-specific orchestration built on Kubernetes. **Kubeflow Pipelines** define ML workflows as DAGs of containerized **Components** (each step is its own Docker image — reproducible in isolation); an **Experiment** groups pipeline **Runs**; each run's outputs are tracked as **Artifacts** (similar spirit to MLflow, but pipeline-native).

```
Data preprocessing (component) → Training (component) → Evaluation (component) → Deployment (component)
```
Each arrow is an artifact handoff (e.g., preprocessed data path) passed between containers, and each box is an independently versioned/buildable Docker image — useful when different steps need very different environments (e.g., Spark for preprocessing, GPU PyTorch for training).

---

## PART 27 — FEATURE STORE

**What:** a centralized system that computes, stores, and serves features consistently for both training (offline, batch) and inference (online, low-latency) — solving the classic problem where the training pipeline and serving pipeline compute "customer average spend" slightly differently.

- **Offline features:** large-scale historical features, used for training (stored in a data warehouse/lake).
- **Online features:** the same feature definitions, but served with millisecond latency for real-time inference (stored in a low-latency store like Redis).
- **Feature consistency:** the *same* transformation logic backs both, avoiding training-serving skew.
- **Feature reuse:** once a good feature (e.g., "30-day rolling avg transaction amount") is built, any team/model can reuse it instead of recomputing it inconsistently.

**Tool:** Feast (open-source feature store).

**Credit-risk example:** a `customer_avg_balance_90d` feature is computed daily in a batch job and written to both an offline store (for retraining) and an online store (Redis) so the real-time loan-decision API can fetch it in <10ms — without the API needing to do the aggregation itself.

---

## PART 28 — MODEL SECURITY

- **Secrets:** never hardcode API keys/DB passwords — use environment variables, AWS Secrets Manager, or Vault.
- **API authentication/authorization:** require API keys/OAuth tokens on `/predict`; role-based access for who can hit admin endpoints.
- **Data privacy:** mask/anonymize PII before it reaches logs or non-production environments.
- **Encryption:** TLS in transit, encryption at rest for S3/DB.
- **Dependency vulnerabilities:** scan `requirements.txt`/Docker images (e.g., `pip-audit`, Trivy) in CI.
- **Model poisoning:** validate/monitor training data sources, especially if any data is user-submitted — an attacker who can influence training data can bias the model.
- **Input validation:** the Pydantic schema at the API layer isn't just for correctness — it also blocks malformed/malicious payloads before they hit model code.
- **Secure containers:** run as non-root user in Dockerfile, use minimal base images (`python:3.11-slim`), don't bake secrets into image layers.

---

## PART 29 — MODEL GOVERNANCE

- **Model lineage:** trace any production model back to its exact training data version (DVC hash), code commit (Git SHA), and run (MLflow run ID).
- **Model versioning:** every trained model is a distinct, retrievable version — never overwrite `model.pkl` in place.
- **Auditability:** who approved this model for production, when, based on what evaluation results — logged, not verbal.
- **Explainability:** for any individual prediction, can you say *why* the model made that call (SHAP/LIME — Part 30)?
- **Approval workflows:** in banking/healthcare, a model risk management (MRM) team or compliance officer signs off before Staging → Production promotion — not fully automated.
- **Reproducibility:** given the lineage info above, could a new engineer rebuild the exact same model from scratch?
- **Documentation:** model cards — intended use, training data description, known limitations, fairness evaluation.
- **Responsible AI:** bias/fairness testing across protected groups, especially for credit/hiring/healthcare models where discriminatory outcomes carry legal risk.

**Why it matters in banking/insurance/healthcare:** these are the industries where a wrong or unexplainable model decision has legal, financial, and human consequences — regulators (e.g., SR 11-7 in US banking) explicitly require documented model risk management, and "the model said so" is not an acceptable answer to "why was this person denied?".

---

## PART 30 — MODEL EXPLAINABILITY

```python
import shap

explainer = shap.TreeExplainer(model)          # fast, exact for tree models (XGBoost/LightGBM)
shap_values = explainer.shap_values(X_test)

shap.summary_plot(shap_values, X_test)          # global: which features matter most overall
shap.force_plot(explainer.expected_value, shap_values[0], X_test.iloc[0])  # local: this one prediction
```
**SHAP (SHapley Additive exPlanations):** game-theory-based, tells you each feature's exact contribution to a single prediction, relative to a baseline — the gold standard for "why did the model reject this loan?" (e.g., "loan_to_income_ratio contributed +0.22 to the default probability, age contributed -0.05").

**LIME:** approximates the model locally with a simple interpretable model around one prediction — model-agnostic (works on anything, even non-tabular), faster but less theoretically grounded than SHAP.

**Partial Dependence Plots (PDP):** shows how the model's average prediction changes as one feature varies, holding others constant — good for understanding a feature's overall directional effect (e.g., "does default probability rise monotonically with loan-to-income ratio?").

**"Why did the model classify this transaction as fraud?"** → pull that transaction's SHAP values → report the top 3 contributing features (e.g., "unusual merchant category (+0.31), transaction at 3am (+0.18), amount 5x above account average (+0.15)") — this becomes the human-readable explanation given to a fraud analyst or customer.

---

## PART 31 — COMPLETE MLOPS PROJECT: Credit Risk / Loan Default Prediction

```
Raw Dataset → Data Validation → Preprocessing → Feature Engineering →
Model Training → MLflow Tracking → Model Registry → Docker → FastAPI →
GitHub Actions → AWS Deployment → Monitoring → Drift Detection → Retraining
```

**How the files communicate:**
`config.yaml` is read by every script for paths/hyperparameters →
`data_ingestion.py` writes `data/raw/*.parquet` →
`data_validation.py` reads that and raises on schema violation, else passes through →
`preprocessing.py` + `feature_engineering.py` combine into `full_pipeline`, fit on validated data, save `artifacts/pipeline.pkl` →
`train.py` uses `full_pipeline` + logs to MLflow, saves `models/model.pkl`, registers to MLflow Model Registry →
`evaluate.py` scores the registered model against a holdout set, gating promotion to "Production" stage →
`app/main.py` loads `models/model.pkl` + `artifacts/pipeline.pkl` at startup, serves `/predict` →
`Dockerfile` packages `app/` + `models/` + `artifacts/` + `src/` into a deployable image →
`.github/workflows/ci-cd.yml` runs `tests/` against all of the above on every push, then builds/pushes/deploys the image →
`docker-compose.yml` runs the API (and optionally a local MLflow server) together for local dev/testing.

*(Full code for `data_ingestion.py`, `data_validation.py`, `preprocessing.py`, `feature_engineering.py`, `train.py`, `evaluate.py`, `app/main.py` (as `predict.py`'s logic), `Dockerfile`, `ci-cd.yml`, and `tests/*.py` is given in full in Parts 4, 5, 6, 7, 10, 12, 14, 15, 16, 17 above — this section is the index tying them into one coherent pipeline.)*

```yaml
# configs/config.yaml
target: default
data:
  raw_csv_path: data/raw/loans.csv
  raw_output_path: data/raw/loans.parquet
  processed_path: data/processed/clean.parquet
model:
  n_estimators: 200
  max_depth: 5
  learning_rate: 0.05
mlflow:
  experiment_name: credit-risk-model
  registered_model_name: credit_risk_xgb
api:
  host: 0.0.0.0
  port: 8000
```

```yaml
# docker-compose.yml
version: "3.9"
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - MODEL_PATH=/app/models/model.pkl
    volumes:
      - ./models:/app/models
      - ./artifacts:/app/artifacts

  mlflow:
    image: ghcr.io/mlflow/mlflow:latest
    ports:
      - "5000:5000"
    command: mlflow server --host 0.0.0.0 --backend-store-uri sqlite:///mlflow.db
```

---

## PART 32 — REQUIREMENTS.TXT

```
pandas==2.2.2
numpy==1.26.4
scikit-learn==1.5.0
xgboost==2.0.3
mlflow==2.14.1
fastapi==0.111.0
uvicorn[standard]==0.30.1
pydantic==2.7.4
joblib==1.4.2
pytest==8.2.2
boto3==1.34.131
pandera==0.19.3
sqlalchemy==2.0.31
pyyaml==6.0.1
requests==2.32.3
shap==0.45.1
dvc[s3]==3.51.2
```
**Dependency management:** pin exact versions (`==`) in production `requirements.txt` to guarantee reproducible builds — "it works on my machine" is very often an unpinned-version problem. Use separate `requirements-dev.txt` for dev-only tools (pytest, flake8, jupyter). **Version conflicts** happen when two libraries need incompatible versions of a shared dependency — resolve with a virtual environment per project and tools like `pip-tools`/`poetry` to lock a consistent dependency graph.

---

## PART 33 — CONFIGURATION

- **`config.yaml`:** non-secret tunables — hyperparameters, file paths, feature lists. Checked into Git.
- **`.env`:** secrets and environment-specific values (DB passwords, API keys, S3 bucket names) — **never** checked into Git (must be in `.gitignore`).
- **Environment variables:** read at runtime (`os.environ["DB_PASSWORD"]`), injected differently per environment (local `.env` file, CI secrets, cloud secret manager in production).
- **Secrets management in production:** AWS Secrets Manager / HashiCorp Vault, injected into the container at deploy time — not baked into the image.

**Never hardcode:** database credentials, API keys, S3 bucket names/paths that differ per environment, model file paths that differ per environment.

---

## PART 34 — LOGGING

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(name)s | %(levelname)s | %(message)s",
)
logger = logging.getLogger(__name__)

logger.debug("Detailed diagnostic info — dev only")
logger.info("Normal operational event, e.g., 'Model loaded successfully'")
logger.warning("Something unexpected but not fatal, e.g., 'Missing values imputed for 3% of rows'")
logger.error("Something failed, e.g., 'Prediction request failed validation'")
```
**Why `print()` isn't enough in production:** no severity levels (can't filter noise from real problems), no timestamps/structure (hard to correlate with an incident), doesn't integrate with log aggregation tools (CloudWatch/ELK), and can't be turned on/off per environment — proper `logging` gives you all of that, plus the ability to ship structured logs (JSON) that monitoring dashboards can parse automatically.

---

## PART 35 — MLOPS TRICKS & BEST PRACTICES (with WHY)

- **Never train directly from production data without validation** — production data can contain corrupted/adversarial rows that silently poison the model.
- **Never hardcode credentials** — leaked secrets in Git history are a permanent security hole (Git remembers forever).
- **Version datasets** — without it you can't reproduce or debug a specific model version.
- **Version models** — enables rollback and A/B comparison.
- **Version dependencies** — prevents "worked yesterday, broke today" from an upstream library update.
- **Save preprocessing pipeline with the model** — prevents training-serving skew, the single most common silent production bug.
- **Avoid training-serving skew** — feature logic must be identical at train and inference time (this is *why* sklearn `Pipeline` objects and feature stores exist).
- **Automate tests** — a human forgetting to run tests before deploying is a matter of "when," not "if."
- **Track experiments** — otherwise good results are unreproducible/unexplainable in a review or audit.
- **Monitor production** — a model's offline metrics don't predict how it behaves as the world drifts.
- **Keep rollback models** — deploying is a bet; you need an exit if it loses.
- **Log predictions carefully** — needed later for both debugging and computing live accuracy once ground truth arrives (but respect PII/privacy rules).
- **Separate development and production** — a bug in dev must never be able to touch real customers' data/decisions.
- **Use environment variables** — same codebase, different config per environment, with no code changes.
- **Keep notebooks for exploration, scripts for production** — notebooks lack the structure/testability required for reliable systems.
- **Make pipelines idempotent** — re-running a pipeline on the same input should give the same result, not duplicate/corrupt data.
- **Set reproducible random seeds** — without it, "did my change actually help?" is unanswerable — you can't tell signal from noise.
- **Validate schema before inference** — catches malformed requests before they reach (and possibly crash, or worse, silently mis-predict on) the model.
- **Monitor drift** — the earliest, cheapest signal that a model needs attention, well before performance visibly collapses.

---

## PART 36 — COMMON ERRORS & DEBUGGING

| Problem | Why it happens | How to diagnose | How to fix |
|---|---|---|---|
| `ModuleNotFoundError` | Package missing from environment/image | Check `pip list`, compare to `requirements.txt` | `pip install -r requirements.txt`; rebuild image without cache |
| Version mismatch | Different library versions locally vs CI vs prod | Compare `pip freeze` outputs across environments | Pin exact versions in `requirements.txt` |
| Docker build failure | Syntax error in Dockerfile, missing file in build context | Read the build log line where it stopped | Fix the failing `RUN`/`COPY` line; check `.dockerignore` isn't excluding a needed file |
| Port already in use | Another process/container bound to that port | `docker ps` / `lsof -i :8000` | Stop the conflicting process or map a different host port |
| Model file not found | Wrong path, file not copied into image, `.gitignore`'d and never pulled from DVC | Check the exact path the code is loading vs what actually exists (`ls`) | Fix `MODEL_PATH`; ensure `dvc pull` or `COPY` step runs before load |
| Pickle errors (`UnpicklingError`, class not found) | Model pickled with a different library version, or from a class defined in a script not importable in the new environment | Check library versions match between save and load environments | Repin versions; use `joblib` consistently; keep the model class importable in the serving environment |
| API `422` error | Request payload doesn't match Pydantic schema | Read FastAPI's detailed error response body | Fix client payload; loosen/clarify schema if the constraint was wrong |
| API `500` error | Unhandled exception in prediction logic | Check application logs/traceback | Add try/except with logging; fix root cause; add regression test |
| Data schema mismatch | Upstream data source changed columns/types | Data validation step raises with a clear message (if you built one) | Update schema definition or fix the upstream source; add alerting on validation failures |
| Feature mismatch | Model expects N features, gets M (e.g., encoder saw new category) | Compare `model.n_features_in_` (or equivalent) to input shape | Use `handle_unknown="ignore"` in encoders; ensure the same saved pipeline is used at inference |
| Missing environment variables | `.env` not loaded, or var not set in deploy environment | App crashes with `KeyError`/`None` config value | Add startup validation that fails fast with a clear error if required env vars are missing |
| AWS permission errors | IAM role/policy doesn't grant the needed action | Read the `AccessDenied` error — it names the exact missing permission | Attach the specific IAM policy needed (principle of least privilege, not `*`) |
| S3 access errors | Wrong bucket/key, bucket policy blocking access, wrong region | `aws s3 ls s3://bucket/` to test access directly | Fix bucket name/region/policy; check credentials are valid |
| Model drift | Real-world data distribution shifted from training data | PSI/KS-test on live vs training data (Part 21) | Trigger retraining pipeline |
| Training-serving skew | Feature computation differs between training code and serving code | Compare feature values for the same raw record computed by both paths | Use one shared pipeline/feature store for both, never duplicate logic |

---

## PART 37 — INTERVIEW PREPARATION (50 Questions)

### Beginner
1. **What is MLOps?** — practices for reliably taking ML models from experimentation to production and maintaining them over time.
2. **Why is MLOps different from DevOps?** — MLOps must version and monitor *data and models*, not just code, and must handle silent decay (drift), which traditional software doesn't experience.
3. **What is CI/CD?** — Continuous Integration (auto build/test on every change) + Continuous Deployment (auto release of passing changes) — reduces manual error and speeds delivery.
4. **What is model drift?** — a broad term for any degradation caused by the production environment diverging from training conditions (covers data drift + concept drift).
5. **Data drift vs concept drift?** — data drift = `P(X)` changes (inputs look different); concept drift = `P(Y|X)` changes (same inputs, different true answer).
6. **Why Docker?** — portability ("works on my machine" solved), reproducible environments, easy horizontal scaling, isolation.
7. **What is MLflow?** — an experiment tracking + model registry tool that logs params/metrics/artifacts per run and manages model versions/stages.
8. **What is a model registry?** — a versioned catalog of trained models with lifecycle stages (Staging/Production/Archived) enabling controlled promotion and rollback.
9. **Why use DVC?** — Git-like version control for large data files, tying data versions to code commits for reproducibility.
10. **What is a feature store?** — a system that computes and serves consistent features for both training and real-time inference, preventing skew.
11. **Batch vs real-time inference?** — batch = scheduled, high-latency-tolerant, cheap; real-time = on-demand, low-latency, more infra.
12. **What's the difference between a parameter and a hyperparameter?** — parameters are learned from data; hyperparameters are set before training.
13. **What is a Dockerfile?** — a text recipe describing how to build a Docker image, layer by layer.
14. **What does `joblib.dump` do differently from `pickle.dump`?** — joblib is optimized for objects with large numpy arrays, common in sklearn models.
15. **What's the purpose of a `.gitignore`?** — prevents committing files that shouldn't be tracked (secrets, large data, caches).

### Intermediate
16. **How would you deploy an ML model?** — package model + preprocessing pipeline behind a versioned API (FastAPI), containerize (Docker), deploy via CI/CD to cloud infra (ECS/K8s/SageMaker), with health checks and monitoring from day one.
17. **How would you monitor a model in production?** — track system metrics (latency/errors) and model metrics (input distribution/drift, prediction distribution, and live performance once labels arrive), alert on thresholds.
18. **How would you automatically retrain a model?** — orchestrated pipeline (Airflow) triggered by drift detection or schedule, retrains on fresh validated data, evaluates against current production model, only promotes if it wins (champion/challenger), often gated by human approval.
19. **How would you roll back a bad model?** — model registry keeps prior versions; transition the last known-good version back to "Production" stage — a metadata operation, not a retrain.
20. **How would you handle data leakage?** — fit all transformations (scalers/encoders) only on training data, split before any feature computation that uses aggregate statistics, and audit for features containing "future" information relative to the prediction point.
21. **How would you ensure reproducibility?** — fixed seeds, pinned dependency versions, versioned data (DVC), versioned code (Git), and every run's parameters logged (MLflow).
22. **Explain training-serving skew and how to prevent it.** — occurs when feature computation differs between training and inference code paths; prevent with a single shared pipeline object or feature store used by both.
23. **What's the difference between model evaluation offline and online?** — offline = metrics on a static holdout set before deployment; online = live performance monitored against real (often delayed) outcomes in production.
24. **How do you choose a threshold for a classifier in production?** — based on the business cost of false positives vs false negatives, not a default 0.5 — often optimized against a cost function (Part 12).
25. **What is a canary deployment and why use it for ML models?** — route a small % of traffic to the new model version first, compare its live metrics to the current model, then ramp up — limits blast radius of a bad model.
26. **What's the difference between Great Expectations/Pandera and Pydantic?** — the former validate whole DataFrames (batch), the latter validates single records (API requests) — both enforce the same underlying data contract at different stages.
27. **What is idempotency and why does it matter in pipelines?** — re-running the same pipeline on the same input produces the same result without side-effect duplication — critical for safe retries after failures.
28. **What's a champion/challenger setup?** — the current production model (champion) is compared against a newly trained candidate (challenger) on live or holdout data before any promotion.
29. **How do feature stores prevent skew?** — by centralizing the *definition and computation* of a feature so both the offline training job and the online serving path pull from the same source of truth.
30. **What's the difference between horizontal and vertical scaling for a model API?** — horizontal = more replica pods/instances behind a load balancer; vertical = bigger single instance — horizontal is generally preferred for resilience and elasticity.

### Advanced
31. **How would you design an end-to-end MLOps pipeline for a fraud detection system requiring <100ms latency?** — real-time feature store (Redis) for low-latency feature lookups, a lightweight model (or optimized serving runtime), autoscaled API behind a load balancer, async logging of predictions for later evaluation, streaming drift monitors on prediction distribution.
32. **How do you handle label delay in retraining pipelines (e.g., loan default known only months later)?** — use proxy/early-warning signals (prediction drift, input drift) for near-term monitoring; schedule true performance evaluation and retraining on a delayed cadence matched to when labels actually arrive.
33. **How would you implement model governance in a regulated bank?** — full lineage (data hash + code commit + run ID per model), mandatory MRM sign-off gate before Production promotion, SHAP-based explainability reports attached to every registered model, immutable audit logs of every stage transition.
34. **Explain blue/green vs canary deployment for ML models.** — blue/green: two full environments, instant full cutover with instant rollback capability; canary: gradual traffic ramp with live comparison — canary is generally safer for catching subtle model regressions that only surface at scale.
35. **How do you detect concept drift when ground truth is delayed by months?** — monitor proxy signals: prediction confidence distribution, feature drift, and any available partial/early-outcome signals; combine with periodic manual sampling/labeling for a delayed but ground-truth-based check.
36. **How would you architect a multi-model MLOps platform serving 50+ models?** — shared feature store, shared CI/CD templates parameterized per model, a common model registry, standardized monitoring dashboards per model, and per-model config (not per-model bespoke infra).
37. **What's your strategy for testing a data pipeline, not just a model?** — schema/contract tests at every stage boundary, row-count/null-rate assertions, statistical distribution checks against a reference, and integration tests running the full pipeline on a small fixture dataset.
38. **How do you prevent a poisoned or adversarial input from corrupting a live model via feedback loops (e.g., online learning)?** — validate/rate-limit inputs used for any online updates, monitor for anomalous shifts triggered by a narrow source, and prefer periodic batch retraining with human-reviewed data over unchecked continuous online learning for high-stakes models.
39. **How would you reduce Docker image size and build time for a large ML serving image?** — multi-stage builds, minimal base images, `.dockerignore` excluding data/notebooks, layer ordering so dependencies (rarely changing) are cached separately from code (frequently changing).
40. **What trade-offs would you consider between SageMaker and a self-managed K8s deployment?** — SageMaker: faster to stand up, built-in monitoring/autoscaling/endpoints, higher managed cost, less infra control; K8s: more control/portability/cost optimization at scale, higher operational burden.

### Scenario-based
41. **Your model's accuracy dropped 15% overnight — walk me through your investigation.** — first check for a pipeline/infra bug (bad deploy, schema change, feature computation error) before assuming drift, since sudden overnight drops are usually a bug, not gradual real-world drift; check recent deploys/commits, compare live input distribution to training distribution, check for upstream data source changes.
42. **A stakeholder wants to know why a specific customer was denied a loan.** — pull that record's SHAP values from the registered model, translate the top contributing features into plain language, and document it per the governance/explainability process.
43. **You need to add a new feature to a model already in production — how do you do it safely?** — build and test the feature in the shared feature/preprocessing pipeline, retrain a challenger model with it, evaluate offline then via canary, only promote if it beats the champion on the metrics that matter.
44. **The CI pipeline is green but the model in production is performing worse than what was tested — why might that happen?** — training-serving skew (different feature computation at inference vs training), or the test set not representative of live traffic, or a silent data source change post-deployment not covered by tests.
45. **How would you design monitoring for a model you don't have ground truth labels for at all (e.g., real-time recommendations with no explicit "correct answer")?** — track proxy business metrics (click-through rate, conversion), prediction distribution stability, and input drift as leading indicators in place of direct accuracy.
46. **Your Docker build works locally but fails in CI — what do you check?** — differences in `.dockerignore` scope, cached layers/build context, base image version pinning, and CI runner architecture (e.g., ARM vs x86) mismatches.
47. **A retrained model has better offline metrics but the team is nervous about promoting it — what's your process?** — canary deployment with live A/B comparison over a defined period against real traffic and business metrics, not just offline holdout numbers, before full promotion.
48. **How would you design a system to prevent training-serving skew across a team of 10 data scientists?** — enforce all feature logic go through a shared feature store/pipeline library (not per-notebook code), code review gates on any new feature, and automated tests comparing training-time vs serving-time feature values on sample records.
49. **Regulators ask you to reproduce a model decision from 8 months ago exactly — can you?** — yes, if data (DVC hash), code (Git commit), and run (MLflow run ID/params) were all versioned and linked at registration time; this is exactly why lineage tracking (Part 29) is mandatory, not optional, in regulated ML.
50. **How do you balance model complexity against explainability requirements in a regulated use case?** — often accept a small performance trade-off for an inherently more interpretable model family (e.g., gradient-boosted trees with SHAP over a deep neural net), or invest more heavily in post-hoc explainability tooling if the complex model's uplift materially matters, documenting the trade-off decision itself for audit.

---

## PART 38 — REAL-LIFE CASE STUDIES

### 1. Credit Risk
**Business problem:** decide loan approval/pricing based on default probability. **Data:** applicant demographics, income, credit bureau data, loan history. **Model:** XGBoost classifier predicting probability of default. **Pipeline:** batch ingestion from core banking + bureau API → strict schema validation → feature engineering (debt-to-income, utilization ratios) → MLflow-tracked training → MRM-gated registry promotion. **Deployment:** real-time API embedded in the loan origination system. **Monitoring:** PSI on key financial ratios monthly. **Drift:** economic shifts (recession) cause concept drift. **Retraining:** quarterly scheduled, plus triggered retraining if PSI exceeds threshold, always with human governance sign-off.

### 2. Fraud Detection
**Business problem:** flag fraudulent transactions in real time. **Data:** transaction stream (amount, merchant, location, device, velocity features). **Model:** gradient boosting or a lightweight neural net, optimized for sub-100ms inference. **Pipeline:** streaming ingestion (Kafka) → real-time feature store lookups → low-latency API. **Deployment:** autoscaled containers behind a load balancer. **Monitoring:** prediction-rate anomalies, PSI on transaction feature streams, live precision/recall as confirmed-fraud labels trickle in. **Drift:** concept drift is constant — fraudsters actively adapt. **Retraining:** frequent (weekly or more), often semi-automated with fast-track review given the adversarial, time-sensitive nature.

### 3. Insurance Claim Prediction
**Business problem:** predict claim likelihood/severity for pricing and fraud screening. **Data:** policyholder profile, claim history, external risk data. **Model:** GLM or gradient boosting depending on interpretability requirements set by regulators. **Pipeline:** batch nightly ingestion → validation → feature engineering (claim frequency, severity trends) → registry with actuarial sign-off. **Deployment:** batch scoring feeding underwriting/pricing systems. **Monitoring:** distribution of claim risk scores vs actuarial expectations. **Drift:** new regulation or catastrophic events (e.g., a pandemic) causing sudden concept drift. **Retraining:** scheduled annually/semi-annually, with emergency retraining triggers for major external shocks.

### 4. Customer Churn
**Business problem:** predict which customers are likely to cancel, to trigger retention offers. **Data:** usage behavior, support tickets, billing history. **Model:** logistic regression or gradient boosting. **Pipeline:** batch ingestion from product analytics warehouse → feature store for behavioral aggregates → training with cross-validation. **Deployment:** batch scoring feeding a marketing automation system weekly. **Monitoring:** feature drift on usage patterns, model lift vs a holdout control group. **Drift:** product changes or new competitor offerings shift churn drivers. **Retraining:** monthly scheduled retrain.

### 5. Recommendation System
**Business problem:** recommend products/content to maximize engagement/conversion. **Data:** user interaction history, item metadata, real-time session events. **Model:** collaborative filtering + gradient boosting re-ranker, or embedding-based retrieval. **Pipeline:** feature store critical here — online features (recent clicks) must be millisecond-fresh. **Deployment:** real-time API within the app's serving path. **Monitoring:** click-through rate, prediction distribution, latency (recommendations must be near-instant). **Drift:** trending content/seasonality causes fast-moving concept drift. **Retraining:** frequent, sometimes continuous/online updates for the retrieval layer, batch retraining for the re-ranker.

### 6. Demand Forecasting
**Business problem:** predict future product demand for inventory/supply chain planning. **Data:** historical sales, promotions, seasonality, external factors (weather, holidays). **Model:** gradient boosting or time-series models (Prophet/ARIMA) per SKU/store. **Pipeline:** large-scale batch ingestion → heavy feature engineering (lags, rolling windows, holiday flags) → distributed training (possibly PySpark) for thousands of SKU-level models. **Deployment:** nightly batch scoring feeding inventory systems. **Monitoring:** forecast error (MAPE) tracked per SKU/segment. **Drift:** demand shocks (supply chain disruption, viral trends) causing sudden data drift. **Retraining:** scheduled weekly/monthly, with alerting on segments where forecast error spikes.

---

## PART 39 — MLOPS vs DEVOPS vs DATA ENGINEERING

| Dimension | DevOps | Data Engineering | MLOps |
|---|---|---|---|
| Purpose | Ship & operate software reliably | Build & maintain data pipelines/infra | Ship & operate ML models reliably |
| Primary artifact | Application code | Datasets/pipelines | Code + Data + Model |
| Testing | Unit/integration/E2E tests | Data quality/schema tests | All of DevOps + data tests + model tests |
| Deployment | App servers/containers | Pipeline schedulers | Model-serving infra (API/batch/stream) |
| Monitoring | Uptime, latency, errors | Pipeline SLAs, data freshness/quality | System metrics + drift + model performance |
| Versioning | Git (code) | Git + pipeline configs | Git (code) + DVC (data) + Registry (model) |
| Infrastructure | CI/CD, cloud compute | ETL/ELT platforms, warehouses/lakes | CI/CD + containers + feature stores + registries |
| Automation focus | Build/test/deploy | Extract/transform/load | Train/evaluate/deploy/retrain (closed loop) |
| Typical tools | Jenkins/GitHub Actions, Docker, K8s | Airflow, Spark, dbt, Snowflake | MLflow, DVC, Kubeflow, FastAPI, SageMaker + the DevOps toolset |

---

## PART 40 — COMPLETE TOOL MAP

| Stage | Tools | Python Libraries | Purpose |
|---|---|---|---|
| Version control | Git, GitHub | — | Code history, collaboration |
| Data versioning | DVC | dvc | Version large datasets, tie to code |
| Data ingestion | — | pandas, SQLAlchemy, boto3, requests, PySpark | Pull raw data in |
| Data validation | Great Expectations, Pandera | pandera, pydantic | Enforce data contracts |
| Preprocessing/FE | — | scikit-learn, numpy, pandas | Clean + engineer features |
| Experiment tracking | MLflow | mlflow | Log params/metrics/artifacts |
| Model registry | MLflow Model Registry | mlflow | Version + stage models |
| Containerization | Docker | — | Portable, reproducible environments |
| API serving | FastAPI, Uvicorn | fastapi, uvicorn, pydantic | Real-time inference endpoint |
| Testing | pytest | pytest | Unit/integration/API/model tests |
| CI/CD | GitHub Actions | — | Automated test/build/deploy |
| Orchestration | Airflow, Prefect, Kubeflow | apache-airflow, prefect, kfp | Schedule & chain pipeline steps |
| Container orchestration | Kubernetes | — | Scale & manage containers in production |
| Feature store | Feast | feast | Consistent train/serve features |
| Cloud storage/compute | AWS S3, EC2, ECR, Lambda, API Gateway, SageMaker | boto3 | Storage, compute, managed ML infra |
| Monitoring (infra) | CloudWatch, Prometheus, Grafana | — | System health metrics/dashboards |
| Monitoring (model) | Evidently AI | evidently | Drift and data quality dashboards |
| Explainability | SHAP, LIME | shap, lime | Model/prediction interpretability |

---

## PART 41 — COMPLETE MLOPS ARCHITECTURE

```
                         ┌─────────────────┐
                         │   DATA SOURCES   │  (DBs, APIs, S3, streams)
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │ DATA INGESTION   │  pandas / SQLAlchemy / boto3
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │ DATA VALIDATION  │  Pandera / Great Expectations
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │ DATA VERSIONING  │  DVC + Git
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │FEATURE ENGINEERING│ scikit-learn Pipeline / Feast
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │ MODEL TRAINING   │  XGBoost / LightGBM / sklearn
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │EXPERIMENT TRACKING│ MLflow
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │ MODEL EVALUATION │  ROC-AUC / PR-AUC / SHAP
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │  MODEL REGISTRY  │  MLflow Model Registry
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │      CI/CD       │  GitHub Actions
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │     DOCKER       │  Dockerfile → Image
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │      CLOUD       │  AWS ECR/ECS/SageMaker/K8s
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │  MODEL SERVING   │  FastAPI: API / Batch / Stream
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │    MONITORING    │  Prometheus / Grafana / CloudWatch
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │ DRIFT DETECTION  │  PSI / KS-test / Evidently
                         └────────┬─────────┘
                                  ▼
                         ┌─────────────────┐
                         │   RETRAINING     │  Airflow-orchestrated loop
                         └────────┬─────────┘
                                  │
                                  └──────────────► back to MODEL TRAINING (↺)
```

---

## PART 42 — FINAL CHEAT SHEET

**1. MLOps Lifecycle:** Data → Validate → Preprocess → Feature Engineer → Version → Train → Track → Evaluate → Register → CI/CD → Containerize → Deploy → Serve → Monitor → Detect Drift → Retrain → Re-register → Redeploy (loop).

**2. Key Definitions:**
- *Data drift:* `P(X)` changes. *Concept drift:* `P(Y|X)` changes.
- *Training-serving skew:* feature logic differs between train and inference code.
- *Idempotency:* re-running a pipeline with the same input gives the same result.
- *Champion/challenger:* comparing a new candidate model against the current production model before promoting.
- *Model lineage:* the traceable chain of data version → code commit → training run → registered model.

**3. Important Commands:**
```
git init / add / commit / branch / checkout / push / pull / merge
dvc init / add / push / pull
docker build / run / ps / stop / images / push
```

**4. Important Python Libraries:** pandas, numpy, scikit-learn, xgboost/lightgbm, mlflow, dvc, fastapi, uvicorn, pydantic, pandera, pytest, boto3, shap, evidently.

**5. Important Files:** `config.yaml`, `requirements.txt`, `Dockerfile`, `docker-compose.yml`, `.github/workflows/ci-cd.yml`, `train.py`, `evaluate.py`, `app/main.py`, `.dvc` pointer files.

**6. Important Docker Commands:** `docker build -t <name> .`, `docker run -p 8000:8000 <name>`, `docker ps`, `docker logs <id>`, `docker exec -it <id> bash`.

**7. Important Git Commands:** `git status`, `git diff`, `git log --oneline`, `git tag v1.0.0`, `git stash`.

**8. Important MLflow Commands/API:** `mlflow.start_run()`, `mlflow.log_param/metric()`, `mlflow.sklearn.log_model()`, `mlflow.register_model()`, `client.transition_model_version_stage()`.

**9. Important AWS Services:** S3 (storage), IAM (access), ECR (image registry), ECS/EC2 (compute), Lambda (serverless), API Gateway (public endpoint), SageMaker (managed ML), CloudWatch (observability).

**10. Interview Points to Nail:** explain training-serving skew unprompted; know PSI/KS-test by name and formulae intuition; know why accuracy is misleading on imbalanced problems; be able to describe a full pipeline end-to-end without notes; know the difference between data and concept drift cold.

**11. Common Mistakes (top 10 to never make):**
1. Fitting scalers/encoders before train/test split.
2. Not saving the preprocessing pipeline with the model.
3. Hardcoding secrets/paths.
4. No tests before deploy.
5. Using accuracy on imbalanced classification.
6. No rollback plan.
7. Monitoring infra only, not model behavior.
8. Retraining without evaluating against the current production model.
9. Notebooks in production.
10. No data/model versioning — irreproducible results.

**12. Production Readiness Checklist:**
- [ ] Data validated at both training and inference time
- [ ] Preprocessing pipeline saved and versioned with the model
- [ ] Random seeds fixed, dependencies pinned
- [ ] Experiment tracked (MLflow), model registered
- [ ] Automated tests: data, model, API
- [ ] CI/CD pipeline: lint → test → build → deploy, gated on main branch
- [ ] Dockerized, minimal image, no secrets baked in
- [ ] Health check endpoint (`/health`)
- [ ] Logging (structured, not `print`)
- [ ] Monitoring: system metrics + drift + live performance
- [ ] Rollback plan (model registry versions kept)
- [ ] Governance: lineage, explainability, approval sign-off (if regulated)
- [ ] Retraining pipeline defined with a clear promotion gate

---

*End of reference. This document is designed to be read top-to-bottom as a course, or used as a lookup reference by Part number for interview prep or project work.*
