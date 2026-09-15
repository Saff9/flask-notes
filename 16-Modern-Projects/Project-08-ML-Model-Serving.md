---
title: "Project 08 — ML Model Serving API (Cortex)"
module: 16-Modern-Projects
project_number: 8
estimated_time: "12–16 hours build, 35–50 hours to polish"
difficulty: advanced
prerequisites:
  - "Complete the core course (Lessons 01–24)"
  - "[[Lesson-12-SQLAlchemy-Setup]] — models with `Mapped` columns"
  - "[[Lesson-14-Relationships]] — Model → ModelVersion"
  - "[[Lesson-18-Blueprints]] — models, predict, batch, admin blueprints"
  - "[[Lesson-19-Building-JSON-API]] — REST + API key auth"
  - "[[Lesson-22-Email-And-Tasks]] — Celery for batch inference"
  - "[[Modern-Databases-And-Caching]] — Redis caching, hash keys"
  - "[[Modern-Flask-And-Python]] — Pydantic v2, observability"
  - "[[Project-03-SaaS-Starter-Kit]] — API key auth + rate limits"
  - "[[Project-07-Social-Media-API]] — rate limits + Celery patterns"
tags:
  - flask
  - flask3
  - python313
  - project
  - portfolio
  - mlops
  - ml
  - onnx
  - joblib
  - prometheus
  - redis
  - celery
  - 2026
related_lessons:
  - "[[Lesson-13-CRUD-Operations]]"
  - "[[Lesson-18-Blueprints]]"
  - "[[Lesson-19-Building-JSON-API]]"
  - "[[Lesson-20-RESTful-Resources]]"
  - "[[Lesson-22-Email-And-Tasks]]"
  - "[[Modern-Databases-And-Caching]]"
  - "[[Modern-Deployment-And-DevOps]]"
  - "[[Project-07-Social-Media-API]]"
---

# Project 08 — ML Model Serving API ("Cortex")

> [!success] Deploy machine-learning models like a real MLOps team.
> By the end of this project you'll have a Flask API that loads `.joblib` and `.onnx` models, versions them, exposes `/predict` and `/batch-predict` endpoints, runs A/B tests between versions, caches inferences in Redis, exposes Prometheus metrics, and serves predictions with single-digit-millisecond latency at the cache. This is the project that proves you can build the *engineering* of ML — which is what most teams actually hire for in 2026.

#flask #mlops #ml-serving #onnx #joblib #prometheus #redis #celery #portfolio #2026

---

## 1. Project Overview

**Cortex** is a model-serving API. Data scientists upload trained model artifacts (a scikit-learn `.joblib` pickle or an ONNX export), the API stores them on S3/R2 with version metadata in Postgres, loads the model into memory, and exposes a `/predict` endpoint. Clients authenticate with API keys, send JSON inputs, get JSON predictions. Everything is observable: every request increments a Prometheus counter, every latency is a histogram bucket, every error is tagged with the model name and version.

The defining engineering decision is **in-memory model loading**. Loading a model from disk on every request takes 50-500 ms — a Python pickle for a 50-tree gradient-boosting model is 20 MB, and `joblib.load()` is dominated by deserialization. The trick: load each version once, hold it in a module-level dict, and serve many requests against the same in-memory object. With this, a 50 ms prediction becomes 2 ms (model is hot) plus 0.3 ms (Pydantic validation) plus 0.2 ms (Redis cache lookup for repeat inputs).

### Why this is a modern 2026 project

A "Flask ML model" tutorial from 2020 did `pickle.load(open('model.pkl','rb'))` on every request. Modern MLOps in 2026 looks completely different:

1. **Model versioning is first-class.** Production teams never deploy a model without an easy rollback. Cortex's `ModelVersion` table tracks every artifact with an `is_active` flag; flipping a boolean reverts a deployment in milliseconds.
2. **ONNX Runtime is the cross-framework standard.** PyTorch, TensorFlow, scikit-learn, XGBoost, LightGBM — all can export to ONNX. Serving one runtime means your inference path is uniform regardless of how the model was trained. ONNX Runtime is also 2-10× faster than native Python predict paths for many models.
3. **A/B testing is a checkbox, not a project.** Weighted routing between model versions (`v1` gets 90%, `v2` gets 10%) is how teams validate new models in production without breaking users.
4. **Inference caching pays for itself.** Many ML workloads have repeating inputs (recommendation systems with cold-start users, classification of common texts). Redis hash of `input_hash → output` skips the model entirely on cache hit.
5. **Prometheus metrics are the contract.** A `/metrics` endpoint scraped every 15 s gives you latency p50/p95/p99, QPS, error rate, cache hit rate — *per model, per version*. Grafana dashboards visualize it.
6. **API key auth with per-model quotas** is the commercial reality of model serving (OpenAI, Replicate, Baseten all work this way).
7. **Batch prediction with Celery** is the only way to score a million-row dataset in reasonable time without blocking web workers.

> [!info] Where this fits in the project series
> [[Project-03-SaaS-Starter-Kit]] taught you API keys + Stripe. [[Project-07-Social-Media-API]] taught you Celery fanout + Redis caching. **Cortex reuses both** and adds model lifecycle (upload, version, activate, rollback) and the inference runtime. Read P03 and P07 first — we won't re-explain API keys or Celery setup.

### Architecture at a glance

```mermaid
flowchart TB
    DS[Data Scientist] -->|POST /api/v1/models<br/>(multipart upload)| F[Flask API]
    F -->|PUT model.onnx| S3[(S3 / R2<br/>model artifacts)]
    F -->|INSERT models + versions| PG[(PostgreSQL<br/>metadata + predictions)]
    F -->|load artifact| MEM[In-Memory<br/>Registry]
    C[Client] -->|POST /predict<br/>X-API-Key| F
    F -->|cache hit?| R[(Redis<br/>input_hash → output)]
    F -->|route by A/B weights| MEM
    MEM -->|numpy → prediction| F
    F -->|/metrics| P[Prometheus<br/>scrape /15s]
    P --> G[Grafana]
    C -->|POST /batch-predict| F
    F -->|enqueue job| Q[Celery Worker]
    Q -->|poll /jobs/{id}| C
    style F fill:#dbeafe,stroke:#2563eb
    style MEM fill:#fce7f3,stroke:#db2777
    style R fill:#dcfce7,stroke:#16a34a
    style P fill:#fef3c7,stroke:#d97706
```

---

## 2. What You'll Learn

By the end you will have hands-on experience with:

- **In-memory model registry** — load-once, serve-many. A module-level dict of `{(model_name, version): runtime}` that survives across requests.
- **ONNX Runtime** for cross-framework inference — `onnxruntime.InferenceSession`, `session.run()`, input/output tensor management.
- **joblib** for scikit-learn models — the simpler path for tree-based models that don't need ONNX.
- **Model versioning with rollback** — an `is_active` flag per `ModelVersion`, atomic activation, instant revert.
- **A/B testing with weighted routing** — `random.random() < weight` selects version `v1` or `v2` per request.
- **Pydantic v2 input/output schemas** — each model declares its input shape (`features: list[float]`) and output shape; the API validates both directions.
- **Redis for inference caching** — `hash = sha256(canonical_json(input))`, `HGET inference:{model}:{version} {hash}`.
- **Prometheus metrics** with `prometheus_client` — `Counter`, `Histogram`, `Gauge`, labeled by model name and version.
- **Batch prediction via Celery** — long-running inference jobs return a `job_id`, client polls `/jobs/{id}` for results.
- **API key authentication** with per-key rate limits and per-model quotas.
- **GPU support** (optional) — `onnxruntime-gpu` and `SessionOptions` for CUDA execution providers.

---

## 3. Prerequisites

| Lesson | Why |
|---|---|
| [[Lesson-12-SQLAlchemy-Setup]] | Models with `Mapped` columns. |
| [[Lesson-14-Relationships]] | `Model → ModelVersion` (one-to-many). |
| [[Lesson-18-Blueprints]] | `models`, `predict`, `batch`, `admin` blueprints. |
| [[Lesson-19-Building-JSON-API]] | REST + JSON. |
| [[Lesson-20-RESTful-Resources]] | Resource modeling for `/models`, `/predict`. |
| [[Lesson-22-Email-And-Tasks]] | Celery setup. |
| [[Modern-Databases-And-Caching]] | Redis hash caching. |
| [[Modern-Deployment-And-DevOps]] | Prometheus, observability. |
| [[Project-03-SaaS-Starter-Kit]] | API keys + rate limits. |
| [[Project-07-Social-Media-API]] | Celery fanout + rate limits. |

You also need:
- Python 3.13+, Docker, Docker Compose
- PostgreSQL 16 + Redis 7 (Docker)
- An S3-compatible bucket for model artifacts
- Optional: a scikit-learn model to test with (we provide a small training script)

---

## 4. Tech Stack

| Tool | Version | Why |
|---|---|---|
| Flask | 3.2+ | Web framework. |
| Flask-SQLAlchemy | 3.1+ | ORM. |
| Flask-Limiter | 3.8+ | Rate limiting per API key. |
| SQLAlchemy | 2.0.36+ | ORM. |
| psycopg | 3.2+ | Postgres driver. |
| redis | 5.2+ | Inference cache, rate limit counters. |
| celery | 5.4+ | Batch prediction jobs. |
| pydantic | 2.9+ | Input/output validation. |
| joblib | 1.4+ | scikit-learn model loading. |
| onnxruntime | 1.20+ | ONNX model inference. |
| scikit-learn | 1.5+ | (Optional) for the demo training script. |
| numpy | 2.1+ | Tensor manipulation. |
| prometheus-client | 0.21+ | `/metrics` endpoint. |
| boto3 | 1.35+ | S3 / R2 client. |
| gunicorn | 23.0+ | WSGI server. |
| Docker Compose | 2.30+ | Local dev stack. |

> [!info] Why ONNX Runtime over native frameworks?
> Loading PyTorch in production is a 600 MB dependency. Loading TensorFlow is 800 MB. ONNX Runtime is 50 MB and runs models exported from either — plus XGBoost, LightGBM, scikit-learn. The inference path is C++ under the hood, often 2-10× faster than the native Python predict loop. For a model-serving platform, ONNX is the right default. joblib stays around for the cases where ONNX export isn't clean (some scikit-learn pipelines).

---

## 5. Project Structure

```
cortex/
├── app/
│   ├── __init__.py            # create_app() factory + metrics endpoint
│   ├── config.py              # Config classes
│   ├── extensions.py          # db, limiter, celery, redis, s3
│   ├── models.py              # User, ApiKey, Model, ModelVersion, Prediction, Job
│   ├── schemas.py             # Pydantic v2 input/output schemas
│   ├── registry.py            # In-memory model registry (load, get, evict)
│   ├── runtime.py             # joblib + onnx runtimes behind one interface
│   ├── metrics.py             # Prometheus counters / histograms
│   ├── auth.py                # API key auth decorator
│   ├── models_api.py          # Blueprint: upload, list, activate, rollback
│   ├── predict.py             # Blueprint: /predict, /batch-predict
│   ├── jobs.py                # Blueprint: poll async jobs
│   ├── admin.py               # Blueprint: A/B test config, model management
│   └── tasks.py               # Celery: batch inference, model warmup
├── training/
│   └── train_iris.py          # Trains a tiny scikit-learn model for testing
├── tests/
│   ├── conftest.py
│   ├── test_models_api.py
│   ├── test_predict.py
│   └── test_batch.py
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
├── celery_worker.py
└── wsgi.py
```

---

## 6. Step-by-Step Build

### Step 1 — Dependencies

```toml
# pyproject.toml (excerpt)
[project]
name = "cortex"
requires-python = ">=3.13"
dependencies = [
    "Flask==3.2.0",
    "Flask-SQLAlchemy==3.1.1",
    "Flask-Limiter==3.8.0",
    "SQLAlchemy==2.0.36",
    "psycopg[binary]==3.2.3",
    "redis==5.2.0",
    "celery==5.4.0",
    "pydantic==2.9.2",
    "joblib==1.4.2",
    "onnxruntime==1.20.1",
    "scikit-learn==1.5.2",
    "numpy==2.1.3",
    "prometheus-client==0.21.0",
    "boto3==1.35.71",
    "python-dotenv==1.0.1",
    "gunicorn==23.0.0",
]
[project.optional-dependencies]
gpu = ["onnxruntime-gpu==1.20.1"]
dev = ["pytest==8.3.4", "pytest-flask==1.3.0", "httpx==0.27.2"]
```

### Step 2 — Configuration

```python
# app/config.py
import os
from dotenv import load_dotenv
load_dotenv()

class Config:
    SECRET_KEY = os.environ["SECRET_KEY"]
    SQLALCHEMY_DATABASE_URI = os.environ["DATABASE_URL"]
    SQLALCHEMY_ENGINE_OPTIONS = {"pool_pre_ping": True, "pool_size": 20}

    REDIS_URL = os.environ.get("REDIS_URL", "redis://localhost:6379/0")
    CELERY_BROKER_URL = os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/1")
    CELERY_RESULT_BACKEND = os.environ.get("CELERY_RESULT_BACKEND", "redis://localhost:6379/2")

    S3_ENDPOINT_URL = os.environ.get("S3_ENDPOINT_URL")
    S3_REGION = os.environ.get("S3_REGION", "auto")
    S3_BUCKET = os.environ["S3_BUCKET"]
    S3_ACCESS_KEY = os.environ["S3_ACCESS_KEY"]
    S3_SECRET_KEY = os.environ["S3_SECRET_KEY"]

    # Per-API-key rate limits (requests per minute)
    DEFAULT_RPM = 60
    PRO_RPM = 600

    # Inference cache TTL — short, since model versions change
    CACHE_TTL_SECONDS = 3600  # 1 hour

    # Model warmup at boot: load every active version
    WARMUP_ON_START = True

class TestConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"
    S3_BUCKET = "test-bucket"
    SECRET_KEY = "test"
    WARMUP_ON_START = False
```

### Step 3 — Extensions + factory

```python
# app/extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from celery import Celery
from redis import Redis
import boto3

db = SQLAlchemy()
limiter = Limiter(key_func=get_remote_address, default_limits=["1000/hour"])
celery = Celery("cortex")
redis: Redis | None = None
s3_client = None

def init_s3(app):
    global s3_client
    s3_client = boto3.client(
        "s3",
        endpoint_url=app.config["S3_ENDPOINT_URL"],
        region_name=app.config["S3_REGION"],
        aws_access_key_id=app.config["S3_ACCESS_KEY"],
        aws_secret_access_key=app.config["S3_SECRET_KEY"],
    )
```

```python
# app/__init__.py
from flask import Flask, jsonify, Response
from prometheus_client import generate_latest, CONTENT_TYPE_LATEST
from app.config import Config
from app.extensions import db, limiter, celery, redis as _redis, init_s3
from redis import Redis

def create_app(config_class=Config) -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_class)

    db.init_app(app)
    limiter.init_app(app)

    import app.extensions as ext
    ext.redis = Redis.from_url(app.config["REDIS_URL"], decode_responses=True)
    if not app.config.get("TESTING"):
        init_s3(app)

    celery.conf.update(
        broker_url=app.config["CELERY_BROKER_URL"],
        result_backend=app.config["CELERY_RESULT_BACKEND"],
    )

    from app.models_api import models_bp
    from app.predict import predict_bp
    from app.jobs import jobs_bp
    from app.admin import admin_bp
    v1 = "/api/v1"
    app.register_blueprint(models_bp, url_prefix=f"{v1}/models")
    app.register_blueprint(predict_bp, url_prefix=f"{v1}/models")
    app.register_blueprint(jobs_bp,   url_prefix=f"{v1}/jobs")
    app.register_blueprint(admin_bp,  url_prefix=f"{v1}/admin")

    # Prometheus metrics endpoint — scrape target
    @app.get("/metrics")
    def metrics():
        return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

    # Warmup: pre-load every active model version into memory
    if app.config.get("WARMUP_ON_START"):
        from app.registry import warmup_all
        with app.app_context():
            warmup_all()

    return app
```

> [!info] Why warmup at boot?
> The first request to a cold model takes 100-500 ms (load artifact, deserialize, allocate tensors). If you have 4 gunicorn workers and a load balancer health check that hits `/predict` every 10 s, the first 4 user requests after deploy each hit a different cold worker and feel slow. Warmup loads every active version into every worker's memory at boot. Subsequent requests find the model hot.

### Step 4 — Models

```python
# app/models.py
from datetime import datetime, timezone
from sqlalchemy import (
    Integer, String, DateTime, ForeignKey, Boolean, Float, Text, BigInteger, JSON,
    UniqueConstraint,
)
from sqlalchemy.orm import Mapped, mapped_column, relationship
import secrets, hashlib
from app.extensions import db

def utcnow() -> datetime:
    return datetime.now(timezone.utc)

class User(db.Model):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    is_admin: Mapped[bool] = mapped_column(default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    api_keys: Mapped[list["ApiKey"]] = relationship(back_populates="user",
                                                     cascade="all, delete-orphan")

class ApiKey(db.Model):
    __tablename__ = "api_keys"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    key_hash: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    key_prefix: Mapped[str] = mapped_column(String(20))  # for display
    name: Mapped[str] = mapped_column(String(80))        # "production", "staging"…
    tier: Mapped[str] = mapped_column(String(10), default="free")  # "free" | "pro"
    is_active: Mapped[bool] = mapped_column(default=True)
    last_used: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    user: Mapped[User] = relationship(back_populates="api_keys")

    @classmethod
    def generate(cls) -> tuple[str, str]:
        raw = "ctx_" + secrets.token_urlsafe(32)
        h = hashlib.sha256(raw.encode()).hexdigest()
        return h, raw

class Model(db.Model):
    """A logical model — e.g. 'iris-classifier'. Has many versions."""
    __tablename__ = "models"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(60), unique=True, index=True)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    versions: Mapped[list["ModelVersion"]] = relationship(
        back_populates="model", cascade="all, delete-orphan"
    )

class ModelVersion(db.Model):
    __tablename__ = "model_versions"
    __table_args__ = (UniqueConstraint("model_id", "version", name="uq_model_version"),)
    id: Mapped[int] = mapped_column(primary_key=True)
    model_id: Mapped[int] = mapped_column(ForeignKey("models.id"), index=True)
    version: Mapped[int] = mapped_column(Integer)  # 1, 2, 3…
    artifact_key: Mapped[str] = mapped_column(String(255))  # S3 key
    framework: Mapped[str] = mapped_column(String(20))      # "joblib" | "onnx"
    input_schema: Mapped[dict] = mapped_column(JSON)        # Pydantic schema as dict
    output_schema: Mapped[dict] = mapped_column(JSON)
    metrics: Mapped[dict] = mapped_column(JSON, default=dict)  # {accuracy: 0.94, ...}
    is_active: Mapped[bool] = mapped_column(default=False, index=True)
    # A/B test weight 0..1 — only meaningful if is_active=True
    weight: Mapped[float] = mapped_column(Float, default=1.0)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    activated_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)

    model: Mapped[Model] = relationship(back_populates="versions")

class Prediction(db.Model):
    """Audit log of predictions (optional — disable in prod for throughput)."""
    __tablename__ = "predictions"
    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    model_id: Mapped[int] = mapped_column(ForeignKey("models.id"), index=True)
    version_id: Mapped[int] = mapped_column(ForeignKey("model_versions.id"), index=True)
    api_key_id: Mapped[int] = mapped_column(ForeignKey("api_keys.id"), index=True)
    input_hash: Mapped[str] = mapped_column(String(64), index=True)
    output: Mapped[dict] = mapped_column(JSON)
    latency_ms: Mapped[float] = mapped_column(Float)
    cache_hit: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow, index=True)

class Job(db.Model):
    """Async batch prediction job."""
    __tablename__ = "jobs"
    id: Mapped[int] = mapped_column(primary_key=True)
    model_id: Mapped[int] = mapped_column(ForeignKey("models.id"), index=True)
    version_id: Mapped[int | None] = mapped_column(ForeignKey("model_versions.id"), nullable=True)
    api_key_id: Mapped[int] = mapped_column(ForeignKey("api_keys.id"))
    status: Mapped[str] = mapped_column(String(20), default="pending")  # pending|running|done|failed
    total: Mapped[int] = mapped_column(Integer, default=0)
    done: Mapped[int] = mapped_column(Integer, default=0)
    result: Mapped[dict | None] = mapped_column(JSON, nullable=True)
    error: Mapped[str | None] = mapped_column(Text, nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    finished_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
```

### Step 5 — ER diagram

```mermaid
erDiagram
    USER ||--o{ API_KEY : owns
    USER ||--o{ MODEL : owns
    MODEL ||--o{ MODEL_VERSION : has
    MODEL_VERSION ||--o{ PREDICTION : produces
    API_KEY ||--o{ PREDICTION : makes
    API_KEY ||--o{ JOB : submits
    MODEL ||--o{ JOB : runs
    MODEL_VERSION ||--o{ JOB : uses
    USER {
        int id PK
        string email UK
        string password_hash
        bool is_admin
    }
    API_KEY {
        int id PK
        int user_id FK
        string key_hash UK
        string key_prefix
        string name
        string tier
        bool is_active
    }
    MODEL {
        int id PK
        string name UK
        int owner_id FK
        string description
    }
    MODEL_VERSION {
        int id PK
        int model_id FK
        int version
        string artifact_key
        string framework
        json input_schema
        json output_schema
        json metrics
        bool is_active
        float weight
    }
    PREDICTION {
        bigint id PK
        int model_id FK
        int version_id FK
        int api_key_id FK
        string input_hash
        json output
        float latency_ms
        bool cache_hit
    }
    JOB {
        int id PK
        int model_id FK
        int version_id FK
        int api_key_id FK
        string status
        int total
        int done
        json result
        string error
    }
```

### Step 6 — The runtime abstraction (joblib + ONNX behind one interface)

```python
# app/runtime.py
from __future__ import annotations
import io, json, os, tempfile
from pathlib import Path
import numpy as np
from typing import Any, Protocol
from app.extensions import s3_client
from app.config import Config
from flask import current_app

class ModelRuntime(Protocol):
    """Common interface for joblib and ONNX runtimes."""
    def predict(self, input_data: list[list[float]]) -> list[Any]: ...

class JoblibRuntime:
    """Wraps a scikit-learn (or any pickle-able) model."""
    def __init__(self, model_obj):
        self.model = model_obj

    def predict(self, input_data: list[list[float]]) -> list[Any]:
        arr = np.asarray(input_data, dtype=np.float64)
        preds = self.model.predict(arr)
        # Coerce numpy types to JSON-serializable Python
        return [p.tolist() if hasattr(p, "tolist") else float(p) for p in preds]

class OnnxRuntime:
    """Wraps an ONNX Runtime InferenceSession."""
    def __init__(self, session):
        self.session = session
        # ONNX requires the input tensor name from the model graph
        self.input_name = session.get_inputs()[0].name
        self.output_name = session.get_outputs()[0].name

    def predict(self, input_data: list[list[float]]) -> list[Any]:
        arr = np.asarray(input_data, dtype=np.float32)
        if arr.ndim == 1:
            arr = arr.reshape(1, -1)
        outputs = self.session.run([self.output_name], {self.input_name: arr})
        preds = outputs[0]
        # Classification models often return labels + probabilities; return both
        return preds.tolist()

def load_runtime(framework: str, artifact_key: str) -> ModelRuntime:
    """Download artifact from S3, instantiate the right runtime."""
    bucket = current_app.config["S3_BUCKET"]
    with tempfile.NamedTemporaryFile(suffix=".bin", delete=False) as tmp:
        local_path = tmp.name
    try:
        s3_client.download_file(bucket, artifact_key, local_path)
        if framework == "joblib":
            import joblib
            return JoblibRuntime(joblib.load(local_path))
        elif framework == "onnx":
            import onnxruntime as ort
            # Disable session-level telemetry in production with env vars
            session = ort.InferenceSession(local_path, providers=["CPUExecutionProvider"])
            return OnnxRuntime(session)
        else:
            raise ValueError(f"unsupported framework: {framework}")
    finally:
        os.unlink(local_path)
```

> [!info] Why a `Protocol` (duck typing) instead of an ABC?
> `ModelRuntime` is a structural type — anything with a `predict` method matches. We don't need inheritance; we just need both runtimes to expose the same method. Protocol is the Python 3.8+ way to express that without forcing a base class.

### Step 7 — The in-memory model registry

```python
# app/registry.py
"""In-memory model registry. Survives across requests within a gunicorn worker."""
import threading
from typing import Any
from sqlalchemy import select
from app.extensions import db
from app.models import Model, ModelVersion
from app.runtime import load_runtime

_lock = threading.Lock()
# Key: (model_name, version)  →  (runtime, framework)
_cache: dict[tuple[str, int], tuple[Any, str]] = {}

def get_runtime(model_name: str, version: int | None = None) -> tuple[Any, ModelVersion]:
    """Return (runtime, version_row). Loads from S3 if not in memory."""
    with _lock:
        # Resolve version: explicit, or active (with A/B weighting done by caller)
        if version is None:
            # Active version(s) — there may be more than one for A/B tests
            versions = db.session.scalars(
                select(ModelVersion).join(Model)
                .where(Model.name == model_name, ModelVersion.is_active == True)
                .order_by(ModelVersion.version.desc())
            ).all()
            if not versions:
                raise KeyError(f"no active version for model {model_name!r}")
            # Pick by weight (single active → weight=1.0, returns itself)
            version_row = _pick_weighted(versions)
        else:
            version_row = db.session.scalar(
                select(ModelVersion).join(Model)
                .where(Model.name == model_name, ModelVersion.version == version)
            )
            if not version_row:
                raise KeyError(f"model {model_name!r} version {version} not found")

        key = (model_name, version_row.version)
        if key not in _cache:
            runtime = load_runtime(version_row.framework, version_row.artifact_key)
            _cache[key] = (runtime, version_row.framework)
        return _cache[key][0], version_row

def _pick_weighted(versions: list[ModelVersion]) -> ModelVersion:
    import random
    if len(versions) == 1:
        return versions[0]
    r = random.random()
    cumulative = 0.0
    for v in versions:
        cumulative += v.weight
        if r <= cumulative:
            return v
    return versions[-1]  # fallback for rounding

def evict(model_name: str, version: int) -> None:
    """Drop a version from memory — used on rollback / delete."""
    with _lock:
        _cache.pop((model_name, version), None)

def warmup_all() -> None:
    """Pre-load every active version. Called from create_app()."""
    versions = db.session.scalars(
        select(ModelVersion).where(ModelVersion.is_active == True)
    ).all()
    for v in versions:
        try:
            model = db.session.get(Model, v.model_id)
            get_runtime(model.name, v.version)
        except Exception as e:
            print(f"warmup failed for {model.name} v{v.version}: {e}")
```

> [!warning] One registry per worker process
> `_cache` lives in module scope — that means each gunicorn worker has its own. With 4 workers, the same model is loaded 4 times into RAM. That's the tradeoff for stateless workers (you can scale horizontally). If your model is 2 GB and you have 8 workers, that's 16 GB of RAM — consider an external model server (TorchServe, Triton) or fewer, larger workers.

### Step 8 — Pydantic v2 schemas

```python
# app/schemas.py
from pydantic import BaseModel, Field, ConfigDict
from datetime import datetime

class PredictionInput(BaseModel):
    """Generic input — list of feature rows."""
    instances: list[list[float]] = Field(
        ..., min_length=1, description="List of feature rows to predict on"
    )

class PredictionOutput(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    model: str
    version: int
    predictions: list
    latency_ms: float
    cache_hit: bool = False

class BatchJobRequest(BaseModel):
    instances: list[list[float]] = Field(..., min_length=1, max_length=100_000)

class BatchJobOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    status: str
    total: int
    done: int
    result: dict | None = None
    error: str | None = None
    created_at: datetime
    finished_at: datetime | None = None

class ModelOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    description: str | None
    versions: list["VersionOut"] = []

class VersionOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    version: int
    framework: str
    is_active: bool
    weight: float
    metrics: dict
    activated_at: datetime | None
    created_at: datetime

ModelOut.model_rebuild()
```

### Step 9 — Prometheus metrics

```python
# app/metrics.py
from prometheus_client import Counter, Histogram, Gauge

# Counter: monotonic increase — total count of events
PREDICTIONS_TOTAL = Counter(
    "cortex_predictions_total",
    "Total predictions served",
    ["model", "version", "cache_hit"],
)

# Histogram: latency distribution — p50, p95, p99
PREDICTION_LATENCY = Histogram(
    "cortex_prediction_latency_seconds",
    "Inference latency in seconds",
    ["model", "version"],
    buckets=(0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0),
)

# Counter: errors
PREDICTION_ERRORS = Counter(
    "cortex_prediction_errors_total",
    "Prediction errors",
    ["model", "version", "type"],
)

# Gauge: current value — number of models loaded in memory
MODELS_LOADED = Gauge(
    "cortex_models_loaded",
    "Number of model versions currently loaded in memory",
)

# Gauge: batch jobs by status
BATCH_JOBS = Gauge(
    "cortex_batch_jobs",
    "Batch jobs by status",
    ["status"],
)
```

> [!info] Counter vs Histogram vs Gauge
> **Counter** goes up only (total requests served, total errors). **Histogram** samples a distribution (latency buckets → compute p99 later). **Gauge** goes up or down (current loaded models, queue depth). Pick the right one — a `Counter` for "current connections" will jump down at restart and confuse Prometheus's `rate()` function.

### Step 10 — API key auth + rate limits

```python
# app/auth.py
import hashlib, time
from functools import wraps
from flask import request, jsonify, g
from sqlalchemy import select
from app.extensions import db
from app.models import ApiKey, User

def _hash(raw: str) -> str:
    return hashlib.sha256(raw.encode()).hexdigest()

def require_api_key(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        raw = request.headers.get("X-API-Key", "")
        if not raw.startswith("ctx_"):
            return jsonify(error="missing_or_invalid_api_key"), 401
        h = _hash(raw)
        key = db.session.scalar(select(ApiKey).where(ApiKey.key_hash == h))
        if not key or not key.is_active:
            return jsonify(error="invalid_api_key"), 401
        # Update last_used (deferred commit so it doesn't slow the request)
        key.last_used = time.time()
        db.session.commit()
        g.api_key = key
        g.user = db.session.get(User, key.user_id)
        return fn(*args, **kwargs)
    return wrapper
```

For rate limits, we use Flask-Limiter keyed on the API key:

```python
# app/decorators.py
from functools import wraps
from flask import g, current_app
from app.extensions import limiter

def per_key_limit():
    """Apply per-API-key rate limit, scaled by tier."""
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            from flask import request
            key_id = getattr(g, "api_key", None)
            if key_id:
                tier = g.api_key.tier
                rpm = current_app.config["PRO_RPM"] if tier == "pro" \
                      else current_app.config["DEFAULT_RPM"]
                key_fn = lambda: f"akey:{key_id.id}"
                return limiter.limit(f"{rpm}/minute", key_func=key_fn)(fn)(
                    *args, **kwargs
                )
            return fn(*args, **kwargs)
        return wrapper
    return decorator
```

### Step 11 — Models API: upload, list, activate, rollback

```python
# app/models_api.py
import io, time, uuid
from flask import Blueprint, request, jsonify, g, current_app
from sqlalchemy import select
from pydantic import ValidationError
from app.extensions import db, s3_client
from app.models import Model, ModelVersion, User
from app.schemas import ModelOut, VersionOut
from app.auth import require_api_key
from app.decorators import per_key_limit
from app.registry import evict

models_bp = Blueprint("models", __name__)

ALLOWED_FRAMEWORKS = {"joblib", "onnx"}

@models_bp.post("", endpoint="upload_model")
@require_api_key
def upload_model():
    """Multipart upload: file + JSON metadata."""
    if "file" not in request.files:
        return jsonify(error="missing_file"), 422
    file = request.files["file"]
    framework = request.form.get("framework", "").lower()
    if framework not in ALLOWED_FRAMEWORKS:
        return jsonify(error="unsupported_framework",
                       allowed=list(ALLOWED_FRAMEWORKS)), 422
    name = request.form.get("name", "").strip().lower()
    if not name or not name.replace("-", "").replace("_", "").isalnum():
        return jsonify(error="invalid_name"), 422

    # Read input/output schemas (sent as JSON strings in the form)
    import json
    try:
        input_schema = json.loads(request.form.get("input_schema", "{}"))
        output_schema = json.loads(request.form.get("output_schema", "{}"))
        metrics = json.loads(request.form.get("metrics", "{}"))
    except json.JSONDecodeError:
        return jsonify(error="invalid_schema_json"), 422

    # Upload artifact to S3 / R2
    ext = "joblib" if framework == "joblib" else "onnx"
    key = f"models/{name}/{int(time.time())}-{uuid.uuid4().hex}.{ext}"
    s3_client.upload_fileobj(file.stream, current_app.config["S3_BUCKET"], key)

    # Get or create Model
    model = db.session.scalar(select(Model).where(Model.name == name))
    if not model:
        model = Model(name=name, owner_id=g.user.id)
        db.session.add(model); db.session.flush()
    elif model.owner_id != g.user.id and not g.user.is_admin:
        return jsonify(error="not_model_owner"), 403

    # New version = max(existing) + 1
    max_version = db.session.scalar(
        select(ModelVersion.version)
        .where(ModelVersion.model_id == model.id)
        .order_by(ModelVersion.version.desc()).limit(1)
    ) or 0
    new_version = max_version + 1

    mv = ModelVersion(
        model_id=model.id,
        version=new_version,
        artifact_key=key,
        framework=framework,
        input_schema=input_schema,
        output_schema=output_schema,
        metrics=metrics,
        is_active=False,  # explicitly activate after testing
    )
    db.session.add(mv)
    db.session.commit()
    return jsonify(ModelOut.model_validate(model).model_dump(mode="json")), 201

@models_bp.get("", endpoint="list_models")
@require_api_key
def list_models():
    models = db.session.scalars(select(Model)).all()
    return jsonify({"data": [ModelOut.model_validate(m).model_dump(mode="json")
                             for m in models]})

@models_bp.post("/<name>/versions/<int:version>/activate",
                endpoint="activate_version")
@require_api_key
def activate_version(name: str, version: int):
    model = db.session.scalar(select(Model).where(Model.name == name))
    if not model:
        return jsonify(error="not_found"), 404
    if model.owner_id != g.user.id and not g.user.is_admin:
        return jsonify(error="not_model_owner"), 403
    mv = db.session.scalar(
        select(ModelVersion).where(ModelVersion.model_id == model.id,
                                   ModelVersion.version == version)
    )
    if not mv:
        return jsonify(error="version_not_found"), 404
    mv.is_active = True
    mv.weight = 1.0
    from datetime import datetime, timezone
    mv.activated_at = datetime.now(timezone.utc)
    db.session.commit()
    # Pre-load into memory
    from app.registry import get_runtime
    get_runtime(model.name, mv.version)
    return jsonify(ok=True), 200

@models_bp.post("/<name>/versions/<int:version>/rollback",
                endpoint="rollback")
@require_api_key
def rollback(name: str, version: int):
    """Deactivate all newer versions, activate the named one."""
    model = db.session.scalar(select(Model).where(Model.name == name))
    if not model:
        return jsonify(error="not_found"), 404
    if model.owner_id != g.user.id and not g.user.is_admin:
        return jsonify(error="not_model_owner"), 403
    target = db.session.scalar(
        select(ModelVersion).where(ModelVersion.model_id == model.id,
                                   ModelVersion.version == version)
    )
    if not target:
        return jsonify(error="version_not_found"), 404

    newer = db.session.scalars(
        select(ModelVersion).where(ModelVersion.model_id == model.id,
                                   ModelVersion.version > version)
    ).all()
    for v in newer:
        v.is_active = False
        evict(model.name, v.version)  # drop from in-memory cache
    target.is_active = True
    target.weight = 1.0
    db.session.commit()
    return jsonify(ok=True, rolled_back_to=version,
                   deactivated=[v.version for v in newer]), 200
```

### Step 12 — Predict endpoint with caching + A/B routing

```python
# app/predict.py
import hashlib, json, time, random
from flask import Blueprint, request, jsonify, g, current_app
from sqlalchemy import select
from pydantic import ValidationError
from app.extensions import db, redis
from app.models import Model, ModelVersion, Prediction
from app.schemas import PredictionInput, PredictionOutput
from app.auth import require_api_key
from app.decorators import per_key_limit
from app.registry import get_runtime
from app.metrics import PREDICTIONS_TOTAL, PREDICTION_LATENCY, PREDICTION_ERRORS

predict_bp = Blueprint("predict", __name__)

def _hash_input(model_name: str, instances: list) -> str:
    canonical = json.dumps({"m": model_name, "i": instances}, sort_keys=True)
    return hashlib.sha256(canonical.encode()).hexdigest()

@predict_bp.post("/<name>/predict", endpoint="predict")
@require_api_key
@per_key_limit()
def predict(name: str):
    try:
        data = PredictionInput.model_validate(request.get_json())
    except ValidationError as e:
        return jsonify(error="validation_error", details=e.errors()), 422

    model = db.session.scalar(select(Model).where(Model.name == name))
    if not model:
        return jsonify(error="model_not_found"), 404

    # A/B routing: if multiple active versions, pick by weight
    active = db.session.scalars(
        select(ModelVersion).where(ModelVersion.model_id == model.id,
                                   ModelVersion.is_active == True)
    ).all()
    if not active:
        return jsonify(error="no_active_version"), 409
    if len(active) == 1:
        version_row = active[0]
    else:
        # Weighted pick
        r = random.random()
        cumulative = 0.0
        for v in active:
            cumulative += v.weight
            if r <= cumulative:
                version_row = v
                break
        else:
            version_row = active[-1]

    # Cache check
    input_hash = _hash_input(name, data.instances)
    cache_key = f"inference:{name}:{version_row.version}"
    cached = redis.hget(cache_key, input_hash)
    if cached:
        PREDICTIONS_TOTAL.labels(model=name, version=version_row.version,
                                 cache_hit="true").inc()
        out = json.loads(cached)
        # Record cached prediction latency (very fast)
        start = time.perf_counter()
        result = PredictionOutput(
            model=name, version=version_row.version,
            predictions=out["predictions"], latency_ms=0.2, cache_hit=True,
        )
        return jsonify(result.model_dump(mode="json")), 200

    # Cache miss: call the model
    start = time.perf_counter()
    try:
        runtime, _ = get_runtime(name, version_row.version)
        predictions = runtime.predict(data.instances)
    except Exception as e:
        PREDICTION_ERRORS.labels(model=name, version=version_row.version,
                                 type=type(e).__name__).inc()
        return jsonify(error="inference_failed", detail=str(e)), 500
    latency_ms = (time.perf_counter() - start) * 1000

    PREDICTION_LATENCY.labels(model=name, version=version_row.version) \
        .observe(latency_ms / 1000)
    PREDICTIONS_TOTAL.labels(model=name, version=version_row.version,
                             cache_hit="false").inc()

    # Store in cache
    redis.hset(cache_key, input_hash,
               json.dumps({"predictions": predictions}))
    redis.expire(cache_key, current_app.config["CACHE_TTL_SECONDS"])

    # Optional audit log (disable for max throughput)
    if not current_app.config.get("TESTING"):
        db.session.add(Prediction(
            model_id=model.id, version_id=version_row.version,
            api_key_id=g.api_key.id, input_hash=input_hash,
            output={"predictions": predictions}, latency_ms=latency_ms,
            cache_hit=False,
        ))
        db.session.commit()

    result = PredictionOutput(
        model=name, version=version_row.version,
        predictions=predictions, latency_ms=round(latency_ms, 3),
        cache_hit=False,
    )
    return jsonify(result.model_dump(mode="json")), 200
```

### Step 13 — Inference request flow

```mermaid
sequenceDiagram
    participant C as Client
    participant F as Flask
    participant R as Redis cache
    participant PG as PostgreSQL
    participant MEM as In-memory registry
    participant S3 as S3 / R2

    C->>F: POST /api/v1/models/iris/predict<br/>X-API-Key: ctx_...<br/>{"instances": [[5.1, 3.5, 1.4, 0.2]]}
    F->>PG: SELECT * FROM api_keys WHERE key_hash = ...
    PG-->>F: ApiKey row
    F->>F: Pydantic validate input
    F->>PG: SELECT active versions WHERE model_id = ...
    PG-->>F: [v1, v2] (both active for A/B)
    F->>F: Weighted pick → v2
    F->>R: HGET inference:iris:2 {input_hash}
    alt Cache hit
        R-->>F: cached predictions
        F-->>C: 200 (latency_ms ~0.2)
    else Cache miss
        F->>MEM: get_runtime("iris", 2)
        alt Not loaded yet
            MEM->>S3: GET model artifact
            S3-->>MEM: bytes
            MEM->>MEM: joblib.load() / ort.InferenceSession()
        end
        MEM-->>F: runtime object
        F->>MEM: runtime.predict([[5.1, ...]])
        MEM-->>F: [0] (setosa)
        F->>R: HSET inference:iris:2 {hash: '{"predictions":[0]}'}
        F->>PG: INSERT prediction audit log
        F-->>C: 200 (latency_ms ~50 cold, ~2 hot)
    end
```

### Step 14 — Batch prediction with Celery

```python
# app/tasks.py
import time
from celery import shared_task
from sqlalchemy import select
from app.extensions import db
from app.models import Job, Model, ModelVersion
from app.registry import get_runtime
from datetime import datetime, timezone

@shared_task
def run_batch(job_id: int) -> None:
    job = db.session.get(Job, job_id)
    if not job:
        return
    job.status = "running"
    db.session.commit()

    try:
        model = db.session.get(Model, job.model_id)
        version_id = job.version_id
        if version_id:
            version = db.session.get(ModelVersion, version_id)
        else:
            # Active version
            version = db.session.scalar(
                select(ModelVersion).where(
                    ModelVersion.model_id == job.model_id,
                    ModelVersion.is_active == True,
                ).order_by(ModelVersion.version.desc())
            )
        runtime, _ = get_runtime(model.name, version.version)

        # The full input set is stored in the job's result field as a placeholder,
        # OR passed via Redis to avoid DB bloat. Here we read from a Redis list.
        import json
        from app.extensions import redis
        instances = json.loads(redis.get(f"batch:input:{job_id}") or "[]")
        job.total = len(instances)
        db.session.commit()

        # Batch in chunks of 1000 to keep memory bounded
        results = []
        CHUNK = 1000
        for i in range(0, len(instances), CHUNK):
            chunk = instances[i:i + CHUNK]
            preds = runtime.predict(chunk)
            results.extend(preds)
            job.done = min(i + CHUNK, len(instances))
            db.session.commit()

        job.result = {"predictions": results}
        job.status = "done"
        job.finished_at = datetime.now(timezone.utc)
        db.session.commit()
        # Cleanup Redis input
        redis.delete(f"batch:input:{job_id}")
    except Exception as e:
        job.status = "failed"
        job.error = str(e)
        job.finished_at = datetime.now(timezone.utc)
        db.session.commit()
```

```python
# app/predict.py (continued) — batch endpoint
from app.extensions import redis
from app.models import Job
from app.tasks import run_batch

@predict_bp.post("/<name>/batch-predict", endpoint="batch_predict")
@require_api_key
def batch_predict(name: str):
    from pydantic import ValidationError
    try:
        data = BatchJobRequest.model_validate(request.get_json())
    except ValidationError as e:
        return jsonify(error="validation_error", details=e.errors()), 422

    model = db.session.scalar(select(Model).where(Model.name == name))
    if not model:
        return jsonify(error="model_not_found"), 404
    # Use active version
    version = db.session.scalar(
        select(ModelVersion).where(ModelVersion.model_id == model.id,
                                   ModelVersion.is_active == True)
        .order_by(ModelVersion.version.desc())
    )
    if not version:
        return jsonify(error="no_active_version"), 409

    job = Job(model_id=model.id, version_id=version.id,
              api_key_id=g.api_key.id, status="pending", total=len(data.instances))
    db.session.add(job); db.session.commit()

    # Stash input in Redis (avoids a giant JSON in Postgres)
    import json
    redis.setex(f"batch:input:{job.id}", 3600 * 24,
                json.dumps(data.instances))

    # Enqueue
    run_batch.delay(job.id)
    return jsonify({"job_id": job.id, "status": "pending",
                    "total": len(data.instances)}), 202
```

```python
# app/jobs.py
from flask import Blueprint, jsonify
from app.extensions import db
from app.models import Job
from app.auth import require_api_key
from app.schemas import BatchJobOut

jobs_bp = Blueprint("jobs", __name__)

@jobs_bp.get("/<int:job_id>", endpoint="get_job")
@require_api_key
def get_job(job_id: int):
    job = db.session.get(Job, job_id)
    if not job or job.api_key_id != g.api_key.id:
        return jsonify(error="not_found"), 404
    return jsonify(BatchJobOut.model_validate(job).model_dump(mode="json"))
```

> [!info] Why stash batch input in Redis, not Postgres?
> A 100k-row batch is ~10 MB of JSON. Storing it in the `Job.result` column means every `GET /jobs/{id}` poll transfers 10 MB. Stashing the input in Redis with a 24-hour TTL keeps Postgres lean and lets the client poll status (small row) cheaply. Final predictions go to `Job.result` — those are usually smaller (one float per row, not a feature vector).

### Step 15 — Admin blueprint: A/B test configuration

```python
# app/admin.py
from flask import Blueprint, request, jsonify, g
from sqlalchemy import select
from app.extensions import db
from app.models import Model, ModelVersion
from app.auth import require_api_key

admin_bp = Blueprint("admin", __name__)

@admin_bp.post("/models/<name>/ab-test", endpoint="set_ab_test")
@require_api_key
def set_ab_test(name: str):
    """Configure an A/B test between two (or more) active versions.

    Body: {"versions": [{"version": 1, "weight": 0.9},
                        {"version": 2, "weight": 0.1}]}
    """
    if not g.user.is_admin:
        return jsonify(error="admin_only"), 403
    model = db.session.scalar(select(Model).where(Model.name == name))
    if not model:
        return jsonify(error="not_found"), 404
    body = request.get_json() or {}
    versions = body.get("versions", [])
    if not versions:
        return jsonify(error="versions_required"), 422
    total_weight = sum(v.get("weight", 0) for v in versions)
    if abs(total_weight - 1.0) > 0.001:
        return jsonify(error="weights_must_sum_to_1",
                       got=total_weight), 422

    # Deactivate all current versions, then activate the named ones
    db.session.query(ModelVersion).filter_by(model_id=model.id).update(
        {"is_active": False, "weight": 0.0}
    )
    for v in versions:
        mv = db.session.scalar(
            select(ModelVersion).where(ModelVersion.model_id == model.id,
                                       ModelVersion.version == v["version"])
        )
        if not mv:
            return jsonify(error=f"version_not_found: {v['version']}"), 404
        mv.is_active = True
        mv.weight = v["weight"]
    db.session.commit()
    return jsonify(ok=True), 200
```

### Step 16 — Training a test model

```python
# training/train_iris.py
"""Train a tiny iris classifier, export as both .joblib and .onnx.
Run: python training/train_iris.py
Output: /tmp/iris_v1.joblib and /tmp/iris_v1.onnx
"""
import joblib, os
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split

# Train
X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=42)
clf = RandomForestClassifier(n_estimators=20, random_state=42).fit(X_train, y_train)
acc = clf.score(X_test, y_test)
print(f"Accuracy: {acc:.4f}")

# Save joblib
joblib.dump(clf, "/tmp/iris_v1.joblib")

# Export to ONNX
from skl2onnx import convert_sklearn
from skl2onnx.common.data_types import FloatTensorType
initial_type = [("float_input", FloatTensorType([None, 4]))]
onx = convert_sklearn(clf, initial_types=initial_type)
with open("/tmp/iris_v1.onnx", "wb") as f:
    f.write(onx.SerializeToString())
print("Saved: /tmp/iris_v1.joblib and /tmp/iris_v1.onnx")
```

> [!warning] `skl2onnx` is a separate package
> Add it to your dev dependencies: `pip install skl2onnx`. It's a build-time tool, not a runtime one — you only need it when exporting models.

---

## 7. Running the Project

### `.env.example`

```bash
SECRET_KEY=change-me
DATABASE_URL=postgresql+psycopg://cortex:cortexpass@postgres:5432/cortex
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=redis://redis:6379/1
CELERY_RESULT_BACKEND=redis://redis:6379/2

S3_ENDPOINT_URL=https://<account>.r2.cloudflarestorage.com
S3_REGION=auto
S3_BUCKET=cortex-models
S3_ACCESS_KEY=<key>
S3_SECRET_KEY=<secret>
```

### `docker-compose.yml`

```yaml
services:
  web:
    build: .
    command: gunicorn -w 4 -b 0.0.0.0:5000 wsgi:app
    ports: ["5000:5000"]
    env_file: .env
    depends_on: [postgres, redis]
  worker:
    build: .
    command: celery -A celery_worker.celery worker --loglevel=info --concurrency=2
    env_file: .env
    depends_on: [redis, postgres]
  prometheus:
    image: prom/prometheus:v3.0.0
    ports: ["9090:9090"]
    volumes: ["./prometheus.yml:/etc/prometheus/prometheus.yml:ro"]
  grafana:
    image: grafana/grafana:11.4.0
    ports: ["3000:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: cortex
      POSTGRES_USER: cortex
      POSTGRES_PASSWORD: cortexpass
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
    volumes: ["redisdata:/data"]
volumes: { pgdata: {}, redisdata: {} }
```

### `prometheus.yml`

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: "cortex"
    static_configs:
      - targets: ["web:5000"]
```

### First run

```bash
cp .env.example .env
# Fill in S3 credentials
docker compose up --build -d postgres redis
docker compose run --rm web flask --app wsgi db upgrade
docker compose up --build -d web worker prometheus grafana

# Train a test model:
pip install scikit-learn skl2onnx joblib
python training/train_iris.py

# Create an admin user + API key (helper script):
docker compose run --rm web python -c "
from app import create_app
from app.extensions import db
from app.models import User, ApiKey
app = create_app()
with app.app_context():
    db.create_all()
    u = User(email='admin@x.com', is_admin=True)
    u.password_hash = 'x'  # set real hash in prod
    db.session.add(u); db.session.commit()
    h, raw = ApiKey.generate()
    k = ApiKey(user_id=u.id, key_hash=h, key_prefix=raw[:14],
               name='admin', tier='pro')
    db.session.add(k); db.session.commit()
    print('API_KEY=' + raw)
"

# Upload the model:
export KEY=ctx_...
curl -X POST http://localhost:5000/api/v1/models \
  -H "X-API-Key: $KEY" \
  -F "name=iris" \
  -F "framework=onnx" \
  -F "input_schema={\"features\":4}" \
  -F "output_schema={\"classes\":3}" \
  -F "metrics={\"accuracy\":0.92}" \
  -F "file=@/tmp/iris_v1.onnx"

# Activate it:
curl -X POST http://localhost:5000/api/v1/models/iris/versions/1/activate \
  -H "X-API-Key: $KEY"

# Predict:
curl -X POST http://localhost:5000/api/v1/models/iris/predict \
  -H "X-API-Key: $KEY" \
  -H "Content-Type: application/json" \
  -d '{"instances": [[5.1, 3.5, 1.4, 0.2]]}'
# → {"model":"iris","version":1,"predictions":[0],"latency_ms":2.3,"cache_hit":false}

# Predict again (same input) → cache_hit:true, latency_ms:0.2

# Check metrics:
curl http://localhost:5000/metrics | grep cortex_predictions_total
```

---

## 8. Testing

```python
# tests/conftest.py
import pytest
from app import create_app
from app.config import TestConfig
from app.extensions import db as _db
from app.models import User, ApiKey

@pytest.fixture
def app():
    app = create_app(TestConfig)
    with app.app_context():
        _db.create_all()
        # Seed admin user + API key
        u = User(email="admin@x.com", is_admin=True, password_hash="x")
        _db.session.add(u); _db.session.flush()
        h, raw = ApiKey.generate()
        k = ApiKey(user_id=u.id, key_hash=h, key_prefix=raw[:14],
                   name="admin", tier="pro")
        _db.session.add(k); _db.session.commit()
        yield app, raw
        _db.session.remove()
        _db.drop_all()

@pytest.fixture
def client(app):
    return app[0].test_client()

@pytest.fixture
def api_key(app):
    return app[1]
```

```python
# tests/test_predict.py
def test_predict_requires_api_key(client):
    r = client.post("/api/v1/models/iris/predict", json={"instances": [[1,2,3,4]]})
    assert r.status_code == 401

def test_predict_validates_input(client, api_key):
    r = client.post("/api/v1/models/iris/predict",
                    headers={"X-API-Key": api_key},
                    json={"instances": "not-a-list"})
    assert r.status_code == 422

def test_predict_unknown_model(client, api_key):
    r = client.post("/api/v1/models/no-such-model/predict",
                    headers={"X-API-Key": api_key},
                    json={"instances": [[1,2,3,4]]})
    assert r.status_code == 404
```

```python
# tests/test_batch.py
def test_batch_returns_job_id(client, api_key, app):
    # Need an active model — seed a stub Model + ModelVersion
    from app.models import Model, ModelVersion
    from app.extensions import db
    with app[0].app_context():
        m = Model(name="iris"); db.session.add(m); db.session.flush()
        mv = ModelVersion(model_id=m.id, version=1, artifact_key="x",
                          framework="joblib", input_schema={}, output_schema={},
                          is_active=True, weight=1.0)
        db.session.add(mv); db.session.commit()
        # Monkeypatch get_runtime to return a fake
        import app.predict as P
        class FakeRT:
            def predict(self, x): return [0] * len(x)
        import app.registry as reg
        reg.get_runtime = lambda n, v: (FakeRT(), None)

    r = client.post("/api/v1/models/iris/batch-predict",
                    headers={"X-API-Key": api_key},
                    json={"instances": [[1,2,3,4], [5,6,7,8]]})
    assert r.status_code == 202
    assert "job_id" in r.get_json()
```

> [!warning] Monkeypatching the registry
> Tests should never hit S3 or load real models. We replace `app.registry.get_runtime` with a lambda returning a `FakeRT` whose `.predict` returns canned output. This is the unit-test boundary: everything *outside* the model is real (validation, auth, cache, routing); the model itself is a stub.

---

## 9. Deployment

### Production checklist

- [ ] **gunicorn with `--preload`** loads models once per worker spawn (4 workers → 4× the model in RAM, but only one load I/O burst at boot, not 4).
- [ ] **Celery worker `--concurrency=2`** for batch jobs — model inference is CPU-bound; 2 processes per container matches typical 2-CPU boxes.
- [ ] **Prometheus** with 30-day retention. Grafana dashboards for: QPS by model, latency p95 by model, error rate by version, cache hit ratio.
- [ ] **Alerting rules**: `rate(cortex_prediction_errors_total[5m]) > 0.01` → page on-call; `cortex_prediction_latency_seconds{quantile="0.99"} > 1` → warn.
- [ ] **S3 lifecycle policy**: move model artifacts to Glacier after 90 days, delete after 1 year. Old versions are for rollback, not daily use.
- [ ] **Per-model API quotas** in addition to per-key: free tier can call `gpt-4-equivalent` 10×/day max. Store in Redis, decrement on each request.
- [ ] **PII / data residency**: input hashes are SHA-256 (one-way) but feature values are stored in the prediction audit log. Disable audit logging for sensitive models, or encrypt at rest with column-level keys.
- [ ] **GPU inference** (optional): install `onnxruntime-gpu`, set `providers=["CUDAExecutionProvider", "CPUExecutionProvider"]` in `OnnxRuntime.__init__`. Add `--gpus all` to the Docker run command. Cost: ~3× CPU instance price for ~10× inference throughput on transformer models.
- [ ] **Cold-start mitigations**: `WARMUP_ON_START=True` (already set), `--preload` in gunicorn, and a `/healthz/ready` endpoint that returns 200 only after warmup completes (so the load balancer doesn't route to a still-warming worker).
- [ ] **Model versioning discipline**: never mutate a `ModelVersion` row's `artifact_key`. New artifact = new version. Mutating artifacts in place breaks rollback.

### Scaling the inference path

```mermaid
flowchart TB
    LB[Load Balancer] --> W1[Flask web 1<br/>4 workers × models in RAM]
    LB --> W2[Flask web 2<br/>4 workers × models in RAM]
    LB --> WN[Flask web N<br/>4 workers × models in RAM]
    W1 & W2 & WN --> R[(Redis<br/>inference cache)]
    W1 & W2 & WN --> PG[(PostgreSQL<br/>metadata + audit)]
    W1 & W2 & WN --> S3[(S3 / R2<br/>model artifacts)]
    W1 & W2 & WN --> MET[/metrics endpoint<br/>scraped by Prometheus/]
    QB[Celery queue] --> Q1[Worker 1<br/>batch inference]
    QB --> Q2[Worker 2]
    PROM[Prometheus] --> GRAF[Grafana]
    MET --> PROM
    style R fill:#dcfce7,stroke:#16a34a
    style S3 fill:#fce7f3,stroke:#db2777
    style MET fill:#fef3c7,stroke:#d97706
```

Each web worker is stateless and identical. Adding capacity means adding workers — the in-memory registry rebuilds itself lazily on first request. Redis cache is shared across all workers (cache hit on worker 2 after worker 1 missed).

---

## 10. Next Steps / Enhancements

1. **Drift detection.** A Celery beat job samples 1% of predictions, computes input feature distributions, alerts when they drift >2σ from training. Use `evidently` or `alibi-detect`.
2. **Canary deployments.** Instead of binary A/B (90/10), ramp gradually: 1% → 5% → 25% → 100% over 24 hours. Auto-rollback if error rate exceeds threshold.
3. **Shadow mode.** Send 100% of traffic to v1, *also* send to v2 silently (drop v2's response). Compare predictions in aggregate — a safe way to test v2 in prod without affecting users.
4. **Multi-model pipelines.** A `/pipelines/<name>/predict` endpoint that chains models: text → embedding model → classifier. Each step is a `ModelVersion`.
5. **Streaming inference.** For LLMs, use Server-Sent Events to stream tokens back. Flask 3.2 supports `stream_with_context`; pair with `transformers` for the model.
6. **TensorRT / OpenVINO** as alternative runtimes — same `ModelRuntime` interface, different backend. Useful when you need every last ms of latency.
7. **Per-model GPU pools.** Tag models with `requires_gpu=True`; route those to a dedicated GPU worker pool. CPU models stay on cheap instances.
8. **Model marketplace.** Public `/models` listing with descriptions, metrics, sample inputs. Clients can `POST /models/<name>/predict` without uploading — your team monetizes the platform.
9. **Webhooks for batch completion.** `POST /jobs/{id}` accepts a `callback_url`; Celery posts results there when done. Saves polling.
10. **Differential privacy.** Add noise to predictions on a per-API-key basis. Useful for sensitive models (medical, financial) where you don't want to leak training data through the API.

---

## 11. What We Learned

- **In-memory model registry is the single biggest performance lever** — load each version once, serve many requests against the same Python object. The 100× speedup vs `joblib.load()` per request is the difference between a 50 ms API and a 5 s API.
- **ONNX Runtime is the cross-framework default** — one runtime, every model format, 2-10× faster than native Python predict paths. `onnxruntime.InferenceSession` with `providers=["CPUExecutionProvider"]` is the production-grade path.
- **Model versioning with an `is_active` flag** makes rollback a single UPDATE — flip the flag, evict the old version from RAM, done. No redeploy.
- **A/B testing is weighted random selection** — `random.random() < cumulative_weight`. Two active versions with `weight=0.9` and `weight=0.1` route 10% of traffic to the challenger. No infrastructure needed; the routing decision is per-request.
- **Redis hash for inference caching** — `HSET inference:{model}:{version} {input_hash} {output}`. Same input → same hash → instant cache hit. Set a TTL so cache doesn't grow unbounded.
- **Pydantic v2 input/output validation** catches bad inputs before they hit the model. The schema doubles as API documentation.
- **Prometheus metrics labeled by model + version** give you per-version QPS, latency, error rate in Grafana. The `/metrics` endpoint is a ~20-line addition that pays off forever.
- **API key auth with `sha256` hashing** (never store the raw key) and a `ctx_` prefix is the same pattern as [[Project-03-SaaS-Starter-Kit]] and [[Project-07-Social-Media-API]].
- **Batch prediction with Celery + Redis for input stash** keeps the request cycle fast and Postgres lean. The client polls `/jobs/{id}` for status; final results land in the DB only.
- **Warmup at boot** turns cold-start latency into a one-time cost. `WARMUP_ON_START=True` + gunicorn `--preload` means the first user request after deploy is as fast as the millionth.
- **Model artifacts in S3/R2** keep Postgres small and make version rollbacks instant (no DB migration; just a metadata UPDATE).

> [!quote] A note from your teacher
> Cortex is the project that taught *me* what MLOps actually is. Most "ML engineers" in 2026 are not training models — they're *deploying* them. The skills that pay are: versioning without downtime, A/B testing without bugs, caching without staleness, monitoring without noise. Every one of those skills is an engineering skill, not a data science skill. Build Cortex once and you'll understand why SageMaker, Baseten, and Replicate are billion-dollar companies — they sell this exact primitive. You now know how to build it.

```mermaid
flowchart LR
    subgraph Upload Path
        DS[Data scientist] -->|POST /models| API1[Flask]
        API1 -->|PUT artifact| S3[(S3 / R2)]
        API1 -->|INSERT metadata| PG[(PostgreSQL)]
    end
    subgraph Inference Path
        C[Client] -->|POST /predict<br/>X-API-Key| API2[Flask]
        API2 -->|HGET cache| R[(Redis)]
        API2 -->|active versions| PG
        API2 -->|get_runtime| MEM[In-memory registry]
        MEM -.->|cold start| S3
        API2 -->|observe latency| MET[Prometheus]
        API2 -->|audit log| PG
    end
    subgraph Batch Path
        C2[Client] -->|POST /batch-predict| API3[Flask]
        API3 -->|SETEX input| R
        API3 -->|enqueue| Q[Celery worker]
        Q -->|poll status| C2
    end
    MET --> GRAF[Grafana dashboards]
    style R fill:#dcfce7,stroke:#16a34a
    style S3 fill:#fce7f3,stroke:#db2777
    style MEM fill:#fef3c7,stroke:#d97706
    style MET fill:#dbeafe,stroke:#2563eb
```
