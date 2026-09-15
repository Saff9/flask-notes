---
title: "Project 09 — DevOps Dashboard (PulseOps)"
module: 16-Modern-Projects
project_number: 9
estimated_time: "14–18 hours build, 40–55 hours to polish"
difficulty: advanced
prerequisites:
  - "Complete the core course (Lessons 01–24)"
  - "[[Lesson-12-SQLAlchemy-Setup]] — models with `Mapped` columns"
  - "[[Lesson-14-Relationships]] — Service → Check → Incident"
  - "[[Lesson-18-Blueprints]] — dashboard, services, incidents, status blueprints"
  - "[[Lesson-22-Email-And-Tasks]] — Celery beat for periodic health checks"
  - "[[Modern-Frontend-Integration]] — SocketIO, HTMX, Chart.js"
  - "[[Modern-Databases-And-Caching]] — Redis ring buffer, metrics cache"
  - "[[Modern-Deployment-And-DevOps]] — Docker SDK, observability"
  - "[[Project-05-Live-Analytics-Dashboard]] — Chart.js patterns"
  - "[[Project-06-IoT-Data-Hub]] — Celery beat + alerting"
tags:
  - flask
  - flask3
  - python313
  - project
  - portfolio
  - devops
  - monitoring
  - docker
  - socketio
  - celery
  - chartjs
  - htmx
  - 2026
related_lessons:
  - "[[Lesson-13-CRUD-Operations]]"
  - "[[Lesson-18-Blueprints]]"
  - "[[Lesson-22-Email-And-Tasks]]"
  - "[[Lesson-23-Deployment]]"
  - "[[Modern-Frontend-Integration]]"
  - "[[Modern-Databases-And-Caching]]"
  - "[[Modern-Deployment-And-DevOps]]"
  - "[[Project-05-Live-Analytics-Dashboard]]"
  - "[[Project-06-IoT-Data-Hub]]"
---

# Project 09 — DevOps Dashboard ("PulseOps")

> [!success] Build your own Datadog.
> By the end of this project you'll have a real-time DevOps dashboard: every minute a Celery beat job pings your services, stores response times in Redis, pushes the result over SocketIO to any open browser, and fires a Slack alert after three consecutive failures. Docker containers show up as cards (running/stopped/unhealthy). GitHub Actions CI/CD runs are pulled in via the GitHub API. There's a public status page with incident history. This is the project that proves you can do observability, real-time UI, and external API integration in one system — exactly the SRE / DevOps skills that pay in 2026.

#flask #devops #monitoring #docker #socketio #celery #chartjs #htmx #portfolio #2026

---

## 1. Project Overview

**PulseOps** is a self-hosted DevOps monitoring dashboard. You register services (URLs to health-check), and a Celery beat job pings each one on its configured interval. Response times go into a Redis ring buffer (last 1,440 checks per service). The dashboard view shows live status via SocketIO — when a check completes, every browser viewing that service's card updates within ~200 ms. Failures trigger incidents: an `Incident` row is created, an alert is sent (Slack webhook + email), and the public status page flips from "All systems operational" to "Partial outage." SSL certificate expiry is checked weekly. Domain expiry monthly. GitHub Actions runs are polled every 5 minutes per repo.

The defining engineering decision is the **real-time push architecture**. A polling dashboard (refresh every 30 s) feels sluggish and lies — when a service goes down, you see it 30 s late. PulseOps uses Flask-SocketIO with per-service rooms; the Celery worker emits a `check.completed` event after each health check, and any browser viewing that service receives it instantly. The result is a dashboard that *feels* like Datadog or GitHub's status page — and that's the bar users have in 2026.

### Why this is a modern 2026 project

A "Flask + chart" monitoring tutorial from 2019 did `requests.get(url)` every 30 s and showed a Chart.js line. Modern DevOps monitoring in 2026 looks completely different:

1. **Real-time push is the default.** Users won't tolerate a 30 s refresh. SocketIO + per-resource rooms is the standard pattern — same as Linear, Vercel, GitHub Actions logs.
2. **Docker SDK for Python** lets you inspect container state programmatically — `docker.Client().containers.list()`, `container.attrs["State"]["Health"]["Status"]`. This is how Portainer and dokku work.
3. **GitHub API integration** pulls CI/CD status into the same dashboard as uptime — no context-switching between tools. `PyGithub` wraps the REST API; webhooks are an alternative for push-on-change.
4. **HTMX + Tailwind** is the server-first UI stack of 2026 — no React build step, no client state to desync, just server-rendered partials swapped into the DOM. Perfect for dashboards where every interaction is "fetch data, render it."
5. **Redis ring buffer for time-series** avoids a TimescaleDB dependency for "last N checks per service" use cases. `LPUSH` + `LTRIM` keeps a fixed-size list per service; reads are O(N).
6. **Celery beat for periodic checks** is the natural fit — declarative schedule, restartable workers, no in-process schedulers fighting gunicorn for the GIL.
7. **Status page with incident history** (status.github.com, status.stripe.com) is a 2026 product expectation, not a luxury. Customers want a public URL they can check during an outage.

> [!info] Where this fits in the project series
> [[Project-05-Live-Analytics-Dashboard]] taught you Chart.js + dashboard layouts. [[Project-06-IoT-Data-Hub]] taught you Celery beat + alerting. **PulseOps reuses both** and adds external integrations (Docker SDK, GitHub API) and real-time push (SocketIO). Read P05 and P06 first — we won't re-explain Chart.js or Celery beat setup.

### Architecture at a glance

```mermaid
flowchart TB
    CB[Celery Beat<br/>every 1 min] -->|enqueue check| Q[Celery Worker]
    Q -->|HTTP GET| S1[Service 1<br/>api.example.com]
    Q -->|HTTP GET| S2[Service 2<br/>app.example.com]
    Q -->|docker SDK| DOCKER[Docker Engine<br/>container health]
    Q -->|GitHub API| GH[GitHub Actions<br/>workflow runs]
    Q -->|ssl + socket| CERT[SSL cert<br/>expiry check]
    Q -->|LPUSH + LTRIM| R[(Redis<br/>ring buffer)]
    Q -->|emit check.completed| WS[Flask-SocketIO]
    WS -->|live update| B[Dashboard Browser]
    Q -->|3 failures| INC[Create Incident]
    INC -->|webhook| SLACK[Slack]
    INC -->|email| OPS[Ops inbox]
    B -->|HTMX partial| F[Flask]
    F -->|GET /dashboard| F
    F -->|SELECT| PG[(PostgreSQL<br/>services, checks, incidents)]
    style WS fill:#dbeafe,stroke:#2563eb
    style R fill:#dcfce7,stroke:#16a34a
    style Q fill:#fef3c7,stroke:#d97706
```

---

## 2. What You'll Learn

By the end you will have hands-on experience with:

- **Docker SDK for Python** (`docker` package) — list containers, inspect health, tail logs, stream events.
- **GitHub API** via `PyGithub` — list workflow runs, fetch last status, paginate efficiently.
- **Flask-SocketIO with per-service rooms** — `socketio.emit(event, room=f"service:{id}")` pushes only to browsers viewing that service.
- **Celery beat** for periodic health checks (every minute), cert checks (weekly), domain checks (monthly).
- **Redis ring buffer** (`LPUSH` + `LTRIM`) for bounded time-series storage — last 1,440 checks per service in 200 KB.
- **Percentile computation** (p50, p95, p99) over a Redis list with `numpy.percentile` — fast for small N.
- **Incident lifecycle** — ` Investigating → Identified → Monitoring → Resolved`, with an `update` log.
- **Alert rules** — consecutive-failure threshold, cooldown window (don't re-alert within 30 min).
- **SSL certificate checking** via `ssl.create_default_context()` + `socket.create_connection()` — pull the cert, parse `not_after`, warn at 14 days.
- **HTMX + Tailwind 4** for server-rendered dashboard — `hx-get="/services/42/checks"` swaps a partial, no JS framework.
- **Chart.js time-series** with downsampling for response-time charts over the last hour/day/week.
- **Public status page** with incident history — no auth, just `GET /status`.

---

## 3. Prerequisites

| Lesson | Why |
|---|---|
| [[Lesson-12-SQLAlchemy-Setup]] | Models with `Mapped` columns. |
| [[Lesson-14-Relationships]] | `Service → Check → Incident`. |
| [[Lesson-18-Blueprints]] | `dashboard`, `services`, `incidents`, `status` blueprints. |
| [[Lesson-22-Email-And-Tasks]] | Celery beat for periodic checks. |
| [[Lesson-23-Deployment]] | Docker, deployment patterns. |
| [[Modern-Frontend-Integration]] | SocketIO, HTMX, Chart.js. |
| [[Modern-Databases-And-Caching]] | Redis ring buffers, caching. |
| [[Modern-Deployment-And-DevOps]] | Observability, Docker. |
| [[Project-05-Live-Analytics-Dashboard]] | Chart.js patterns. |
| [[Project-06-IoT-Data-Hub]] | Celery beat + alerting. |

You also need:
- Python 3.13+, Docker, Docker Compose
- PostgreSQL 16 + Redis 7 (Docker)
- A Slack workspace (for webhook alerts) — optional
- A GitHub personal access token (for CI/CD status) — optional

---

## 4. Tech Stack

| Tool | Version | Why |
|---|---|---|
| Flask | 3.2+ | Web framework. |
| Flask-SQLAlchemy | 3.1+ | ORM. |
| Flask-Login | 0.6+ | Dashboard auth. |
| Flask-SocketIO | 5.4+ | Real-time push to dashboard. |
| Flask-WTF | 1.2+ | CSRF + forms. |
| Celery | 5.4+ | Periodic health checks, beat scheduler. |
| Redis | 7 | Ring buffer, alert cooldown, Celery broker. |
| PostgreSQL | 16 | Service/check/incident metadata. |
| psycopg | 3.2+ | Postgres driver. |
| docker | 7.1+ | Docker SDK for Python. |
| PyGithub | 2.5+ | GitHub Actions API. |
| httpx | 0.27+ | Async-friendly HTTP client (sync use here). |
| eventlet | 0.37+ | SocketIO async mode. |
| Chart.js | 4 (CDN) | Time-series charts. |
| HTMX | 2.0 (CDN) | Server-rendered partials. |
| Tailwind CSS | 4 (CDN) | Styling. |
| Docker Compose | 2.30+ | Local dev stack. |

> [!info] Why the Docker SDK and not `docker compose ps` shell-outs?
> Shelling out to the Docker CLI from a Flask app is fragile (parse text output, handle version drift, spawn subprocess per call). The Docker SDK (`pip install docker`) is the official Python binding — typed, versioned, returns Python objects. It connects to `/var/run/docker.sock` directly. Same engine, no subprocess overhead.

---

## 5. Project Structure

```
pulseops/
├── app/
│   ├── __init__.py            # create_app() factory + SocketIO + beat schedule
│   ├── config.py              # Config classes
│   ├── extensions.py          # db, login_manager, socketio, celery, redis
│   ├── models.py              # User, Service, Check, Incident, IncidentUpdate, Notification
│   ├── checks.py              # Health check primitives (HTTP, TCP, SSL, Docker, GitHub)
│   ├── tasks.py               # Celery beat tasks: run_checks, check_ssl, pull_github
│   ├── alerts.py              # Alert routing: Slack webhook, email, dedup
│   ├── docker_client.py       # Docker SDK wrapper
│   ├── github_client.py       # PyGithub wrapper
│   ├── sockets.py             # SocketIO join/leave per service
│   ├── dashboard.py           # Blueprint: dashboard, service detail
│   ├── services.py            # Blueprint: CRUD for services
│   ├── incidents.py           # Blueprint: incident management
│   ├── status.py              # Blueprint: public status page
│   ├── auth.py                # Blueprint: login/logout
│   ├── templates/
│   │   ├── base.html
│   │   ├── dashboard/{index,service}.html
│   │   ├── services/{list,form}.html
│   │   ├── incidents/{list,form,detail}.html
│   │   ├── status/{index,incidents}.html
│   │   └── partials/{service_card,check_row,incident_badge}.html
│   └── static/
│       └── js/dashboard.js    # Chart.js + SocketIO client
├── tests/
│   ├── conftest.py
│   ├── test_checks.py
│   ├── test_alerts.py
│   └── test_dashboard.py
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
name = "pulseops"
requires-python = ">=3.13"
dependencies = [
    "Flask==3.2.0",
    "Flask-SQLAlchemy==3.1.1",
    "Flask-Login==0.6.3",
    "Flask-WTF==1.2.1",
    "Flask-SocketIO==5.4.1",
    "Flask-Mail==0.10.0",
    "SQLAlchemy==2.0.36",
    "psycopg[binary]==3.2.3",
    "redis==5.2.0",
    "celery==5.4.0",
    "docker==7.1.0",
    "PyGithub==2.5.0",
    "httpx==0.27.2",
    "pydantic==2.9.2",
    "eventlet==0.37.0",
    "python-dotenv==1.0.1",
    "gunicorn==23.0.0",
]
[project.optional-dependencies]
dev = ["pytest==8.3.4", "pytest-flask==1.3.0", "responses==0.25.3"]
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

    # Docker SDK
    DOCKER_SOCKET = os.environ.get("DOCKER_SOCKET", "unix:///var/run/docker.sock")

    # GitHub
    GITHUB_TOKEN = os.environ.get("GITHUB_TOKEN", "")
    GITHUB_REPOS = os.environ.get("GITHUB_REPOS", "").split(",")  # "owner/repo,owner/repo"

    # Alerts
    SLACK_WEBHOOK_URL = os.environ.get("SLACK_WEBHOOK_URL", "")
    ALERT_COOLDOWN_MINUTES = 30
    ALERT_FAILURE_THRESHOLD = 3  # consecutive failures → alert

    # Mail
    MAIL_SERVER = os.environ.get("MAIL_SERVER", "smtp.mailtrap.io")
    MAIL_PORT = int(os.environ.get("MAIL_PORT", 587))
    MAIL_USERNAME = os.environ.get("MAIL_USERNAME")
    MAIL_PASSWORD = os.environ.get("MAIL_PASSWORD")
    MAIL_USE_TLS = True
    ALERT_EMAIL_TO = os.environ.get("ALERT_EMAIL_TO", "ops@example.com")

    # Ring buffer size per service (1 check/min × 1440 = 24 hours)
    CHECK_RING_SIZE = 1440

class TestConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"
    WTF_CSRF_ENABLED = False
    SECRET_KEY = "test"
    DOCKER_SOCKET = "unix:///nonexistent.sock"  # tests stub the client
    GITHUB_REPOS = []
```

### Step 3 — Extensions + factory

```python
# app/extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager
from flask_wtf import CSRFProtect
from flask_socketio import SocketIO
from flask_mail import Mail
from celery import Celery
from redis import Redis

db = SQLAlchemy()
login_manager = LoginManager()
csrf = CSRFProtect()
socketio = SocketIO(async_mode="eventlet", cors_allowed_origins="*")
mail = Mail()
celery = Celery("pulseops")
redis: Redis | None = None
```

```python
# app/__init__.py
from flask import Flask
from app.config import Config
from app.extensions import (
    db, login_manager, csrf, socketio, mail, celery, redis as _redis
)
from redis import Redis

def create_app(config_class=Config) -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_class)

    db.init_app(app)
    login_manager.init_app(app)
    csrf.init_app(app)
    mail.init_app(app)
    socketio.init_app(app, async_mode="eventlet", cors_allowed_origins="*")

    import app.extensions as ext
    ext.redis = Redis.from_url(app.config["REDIS_URL"], decode_responses=True)

    celery.conf.update(
        broker_url=app.config["CELERY_BROKER_URL"],
        result_backend=app.config["CELERY_RESULT_BACKEND"],
        beat_schedule={
            # Health checks run on the service's own interval (every minute
            # we enqueue a job that checks all services due now).
            "run-due-checks": {
                "task": "app.tasks.run_due_checks",
                "schedule": 60.0,  # every 60 seconds
            },
            "pull-github": {
                "task": "app.tasks.pull_github_status",
                "schedule": 300.0,  # every 5 minutes
            },
            "check-ssl": {
                "task": "app.tasks.check_ssl_certs",
                "schedule": 86400.0,  # daily
            },
        },
    )

    from app.models import User
    @login_manager.user_loader
    def load_user(uid):
        return db.session.get(User, int(uid))

    from app.auth import auth_bp
    from app.dashboard import dashboard_bp
    from app.services import services_bp
    from app.incidents import incidents_bp
    from app.status import status_bp

    app.register_blueprint(auth_bp,       url_prefix="/auth")
    app.register_blueprint(dashboard_bp,  url_prefix="/dashboard")
    app.register_blueprint(services_bp,   url_prefix="/services")
    app.register_blueprint(incidents_bp,  url_prefix="/incidents")
    app.register_blueprint(status_bp,     url_prefix="/status")

    from app import sockets  # noqa: F401  (registers SocketIO handlers)

    return app
```

### Step 4 — Models

```python
# app/models.py
from datetime import datetime, timezone
from flask_login import UserMixin
from sqlalchemy import (
    Integer, String, DateTime, ForeignKey, Boolean, Float, Text, BigInteger, Index,
)
from sqlalchemy.orm import Mapped, mapped_column, relationship
from werkzeug.security import generate_password_hash, check_password_hash
from app.extensions import db

def utcnow() -> datetime:
    return datetime.now(timezone.utc)

class User(UserMixin, db.Model):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    is_admin: Mapped[bool] = mapped_column(default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    def set_password(self, raw): self.password_hash = generate_password_hash(raw)
    def check_password(self, raw): return check_password_hash(self.password_hash, raw)

class Service(db.Model):
    __tablename__ = "services"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(120))
    url: Mapped[str] = mapped_column(String(500))
    kind: Mapped[str] = mapped_column(String(20), default="http")
    # "http" | "tcp" | "docker" | "github" | "ssl"
    expected_status: Mapped[int] = mapped_column(Integer, default=200)
    check_interval_seconds: Mapped[int] = mapped_column(Integer, default=60)
    timeout_seconds: Mapped[int] = mapped_column(Integer, default=10)
    is_active: Mapped[bool] = mapped_column(default=True)
    is_public: Mapped[bool] = mapped_column(default=True)  # show on status page

    # Last-known state (denormalized for dashboard speed)
    current_status: Mapped[str] = mapped_column(String(20), default="unknown")
    # "operational" | "degraded" | "down" | "unknown" | "paused"
    last_check_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    last_response_ms: Mapped[int | None] = mapped_column(Integer, nullable=True)
    last_error: Mapped[str | None] = mapped_column(Text, nullable=True)
    consecutive_failures: Mapped[int] = mapped_column(Integer, default=0)

    # Metadata for non-HTTP checks
    config: Mapped[dict] = mapped_column(default=dict)  # JSON column

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    checks: Mapped[list["Check"]] = relationship(back_populates="service",
                                                   cascade="all, delete-orphan")
    incidents: Mapped[list["Incident"]] = relationship(back_populates="service")

class Check(db.Model):
    __tablename__ = "checks"
    __table_args__ = (Index("ix_check_service_time", "service_id", "checked_at"),)
    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    service_id: Mapped[int] = mapped_column(ForeignKey("services.id"), index=True)
    checked_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    status: Mapped[str] = mapped_column(String(20))  # "ok" | "fail" | "timeout"
    response_ms: Mapped[int | None] = mapped_column(Integer, nullable=True)
    status_code: Mapped[int | None] = mapped_column(Integer, nullable=True)
    error: Mapped[str | None] = mapped_column(Text, nullable=True)
    service: Mapped[Service] = relationship(back_populates="checks")

class Incident(db.Model):
    __tablename__ = "incidents"
    id: Mapped[int] = mapped_column(primary_key=True)
    service_id: Mapped[int] = mapped_column(ForeignKey("services.id"), index=True)
    title: Mapped[str] = mapped_column(String(200))
    status: Mapped[str] = mapped_column(String(20), default="investigating")
    # "investigating" | "identified" | "monitoring" | "resolved"
    severity: Mapped[str] = mapped_column(String(20), default="minor")
    # "minor" | "major" | "critical"
    started_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    resolved_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    updates: Mapped[list["IncidentUpdate"]] = relationship(
        back_populates="incident", cascade="all, delete-orphan",
        order_by="IncidentUpdate.created_at.desc()"
    )
    service: Mapped[Service] = relationship(back_populates="incidents")

class IncidentUpdate(db.Model):
    __tablename__ = "incident_updates"
    id: Mapped[int] = mapped_column(primary_key=True)
    incident_id: Mapped[int] = mapped_column(ForeignKey("incidents.id"), index=True)
    body: Mapped[str] = mapped_column(Text)
    new_status: Mapped[str] = mapped_column(String(20))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    incident: Mapped[Incident] = relationship(back_populates="updates")

class Notification(db.Model):
    """Audit log of alerts sent (Slack, email)."""
    __tablename__ = "notifications"
    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    service_id: Mapped[int] = mapped_column(ForeignKey("services.id"), index=True)
    channel: Mapped[str] = mapped_column(String(20))  # "slack" | "email" | "webhook"
    message: Mapped[str] = mapped_column(Text)
    sent_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
```

### Step 5 — ER diagram

```mermaid
erDiagram
    USER ||--o{ SERVICE : owns
    SERVICE ||--o{ CHECK : monitored_by
    SERVICE ||--o{ INCIDENT : experiences
    INCIDENT ||--o{ INCIDENT_UPDATE : documented_in
    SERVICE ||--o{ NOTIFICATION : triggers
    USER {
        int id PK
        string email UK
        string password_hash
        bool is_admin
    }
    SERVICE {
        int id PK
        string name
        string url
        string kind
        int expected_status
        int check_interval_seconds
        string current_status
        datetime last_check_at
        int last_response_ms
        int consecutive_failures
        bool is_public
    }
    CHECK {
        bigint id PK
        int service_id FK
        datetime checked_at
        string status
        int response_ms
        int status_code
        string error
    }
    INCIDENT {
        int id PK
        int service_id FK
        string title
        string status
        string severity
        datetime started_at
        datetime resolved_at
    }
    INCIDENT_UPDATE {
        int id PK
        int incident_id FK
        string body
        string new_status
        datetime created_at
    }
    NOTIFICATION {
        bigint id PK
        int service_id FK
        string channel
        string message
        datetime sent_at
    }
```

### Step 6 — Health check primitives

```python
# app/checks.py
"""The actual checking logic — one function per service kind."""
import ssl, socket, time
from datetime import datetime, timezone
from urllib.parse import urlparse
import httpx
from app.extensions import redis
from flask import current_app

def check_http(url: str, expected: int, timeout: int) -> dict:
    start = time.perf_counter()
    try:
        r = httpx.get(url, timeout=timeout, follow_redirects=True)
        ms = int((time.perf_counter() - start) * 1000)
        ok = (r.status_code == expected)
        return {
            "status": "ok" if ok else "fail",
            "status_code": r.status_code,
            "response_ms": ms,
            "error": None if ok else f"expected {expected}, got {r.status_code}",
        }
    except httpx.TimeoutException:
        return {"status": "timeout", "status_code": None,
                "response_ms": timeout * 1000, "error": "timeout"}
    except Exception as e:
        return {"status": "fail", "status_code": None,
                "response_ms": None, "error": str(e)}

def check_tcp(host: str, port: int, timeout: int) -> dict:
    start = time.perf_counter()
    try:
        with socket.create_connection((host, port), timeout=timeout):
            ms = int((time.perf_counter() - start) * 1000)
            return {"status": "ok", "status_code": None,
                    "response_ms": ms, "error": None}
    except Exception as e:
        return {"status": "fail", "status_code": None,
                "response_ms": None, "error": str(e)}

def check_ssl(hostname: str, port: int = 443) -> dict:
    """Returns days_until_expiry in the error field."""
    try:
        ctx = ssl.create_default_context()
        with socket.create_connection((hostname, port), timeout=10) as sock:
            with ctx.wrap_socket(sock, server_hostname=hostname) as ssock:
                cert = ssock.getpeercert()
        # cert['notAfter'] = 'Jan 1 23:59:59 2026 GMT'
        expires = datetime.strptime(cert["notAfter"], "%b %d %H:%M:%S %Y %Z")
        expires = expires.replace(tzinfo=timezone.utc)
        days_left = (expires - datetime.now(timezone.utc)).days
        if days_left < 0:
            return {"status": "fail", "error": f"expired {-days_left} days ago"}
        if days_left < 14:
            return {"status": "fail", "error": f"expires in {days_left} days"}
        return {"status": "ok", "error": f"{days_left} days left",
                "response_ms": 0}
    except Exception as e:
        return {"status": "fail", "error": str(e)}

def check_docker_container(container_name: str) -> dict:
    """Use Docker SDK to inspect a container's health."""
    from app.docker_client import get_container_state
    state = get_container_state(container_name)
    if state is None:
        return {"status": "fail", "error": "container not found"}
    if state == "running":
        return {"status": "ok", "response_ms": 0, "error": None}
    if state == "unhealthy":
        return {"status": "fail", "error": "container unhealthy"}
    return {"status": "fail", "error": f"container {state}"}
```

> [!info] Why `httpx` and not `requests`?
> `httpx` is the modern Python HTTP client — same API as `requests` but with HTTP/2 support, async support, and a cleaner connection pool. For a monitoring tool that makes thousands of outbound requests per minute, the connection pool reuse is worth the upgrade. `requests` is fine; `httpx` is the future.

### Step 7 — Docker SDK + GitHub API wrappers

```python
# app/docker_client.py
"""Thin wrapper over docker SDK. Cached client per process."""
import docker
from flask import current_app

_client: docker.DockerClient | None = None

def _get_client() -> docker.DockerClient:
    global _client
    if _client is None:
        _client = docker.DockerClient(base_url=current_app.config["DOCKER_SOCKET"])
    return _client

def list_containers(all_states: bool = False) -> list[dict]:
    """Return [{name, state, status, health, image}, ...]."""
    client = _get_client()
    out = []
    for c in client.containers.list(all=all_states):
        health = "unknown"
        attrs = c.attrs
        state = attrs.get("State", {})
        if "Health" in state:
            health = state["Health"]["Status"]  # "starting" | "healthy" | "unhealthy"
        out.append({
            "name": c.name,
            "state": state.get("Status", "unknown"),
            "status": c.status,
            "health": health,
            "image": attrs.get("Config", {}).get("Image", ""),
        })
    return out

def get_container_state(name: str) -> str | None:
    """Return 'running' | 'exited' | 'unhealthy' | 'healthy' | None."""
    client = _get_client()
    try:
        c = client.containers.get(name)
    except docker.errors.NotFound:
        return None
    state = c.attrs.get("State", {})
    if state.get("Status") != "running":
        return state.get("Status", "unknown")
    if "Health" in state:
        return state["Health"]["Status"]  # "healthy" | "unhealthy"
    return "running"

def tail_logs(name: str, lines: int = 100) -> str:
    client = _get_client()
    try:
        c = client.containers.get(name)
    except docker.errors.NotFound:
        return ""
    return c.logs(tail=lines, timestamps=True).decode(errors="replace")
```

```python
# app/github_client.py
"""GitHub Actions CI/CD status via PyGithub."""
from github import Github
from flask import current_app

_client: Github | None = None

def _get_client() -> Github:
    global _client
    if _client is None:
        token = current_app.config["GITHUB_TOKEN"]
        _client = Github(token) if token else Github()
    return _client

def latest_runs(repo_full: str, limit: int = 5) -> list[dict]:
    """Return last N workflow runs: [{id, name, status, conclusion, html_url, ran_at}]."""
    g = _get_client()
    try:
        repo = g.get_repo(repo_full)
    except Exception:
        return []
    runs = []
    for run in repo.get_workflow_runs()[:limit]:
        runs.append({
            "id": run.id,
            "name": run.name,
            "status": run.status,        # "queued" | "in_progress" | "completed"
            "conclusion": run.conclusion,  # "success" | "failure" | "cancelled" | None
            "html_url": run.html_url,
            "ran_at": run.run_started_at.isoformat() if run.run_started_at else None,
        })
    return runs
```

> [!warning] Rate limits on the Docker SDK and GitHub API
> Docker SDK is local — no rate limit, but `containers.list()` walks every container on the host. With 200+ containers, that's ~50 ms per call. Cache it for 5 s in Redis if you poll from a dashboard. GitHub API is harder: 5,000 requests/hour per token. We poll every 5 minutes per repo — at 100 repos that's 1,200 requests/hour, well under the limit.

### Step 8 — Celery tasks (beat schedule)

```python
# app/tasks.py
"""Celery tasks invoked by the beat schedule."""
from datetime import datetime, timezone, timedelta
from celery import shared_task
from sqlalchemy import select
from app.extensions import db, redis, socketio
from app.models import Service, Check, Incident, Notification
from app.checks import check_http, check_tcp, check_ssl, check_docker_container
from app.alerts import maybe_alert
from app.github_client import latest_runs

@shared_task
def run_due_checks() -> None:
    """Every minute, check every service whose interval has elapsed."""
    now = datetime.now(timezone.utc)
    services = db.session.scalars(
        select(Service).where(Service.is_active == True)
    ).all()
    for svc in services:
        if svc.last_check_at and (now - svc.last_check_at).total_seconds() < svc.check_interval_seconds:
            continue  # not due yet
        run_check.delay(svc.id)

@shared_task
def run_check(service_id: int) -> None:
    svc = db.session.get(Service, service_id)
    if not svc or not svc.is_active:
        return

    # Dispatch by kind
    if svc.kind == "http":
        result = check_http(svc.url, svc.expected_status, svc.timeout_seconds)
    elif svc.kind == "tcp":
        from urllib.parse import urlparse
        u = urlparse(svc.url)
        result = check_tcp(u.hostname, u.port or 80, svc.timeout_seconds)
    elif svc.kind == "ssl":
        from urllib.parse import urlparse
        u = urlparse(svc.url)
        result = check_ssl(u.hostname)
    elif svc.kind == "docker":
        result = check_docker_container(svc.config.get("container_name", ""))
    else:
        result = {"status": "fail", "error": f"unknown kind: {svc.kind}"}

    # Persist the check row
    check = Check(
        service_id=svc.id,
        status=result["status"],
        response_ms=result.get("response_ms"),
        status_code=result.get("status_code"),
        error=result.get("error"),
    )
    db.session.add(check)

    # Update denormalized service state
    svc.last_check_at = datetime.now(timezone.utc)
    svc.last_response_ms = result.get("response_ms")
    svc.last_error = result.get("error")
    if result["status"] == "ok":
        svc.consecutive_failures = 0
        svc.current_status = "operational"
    else:
        svc.consecutive_failures += 1
        if svc.consecutive_failures >= 3:
            svc.current_status = "down"
        else:
            svc.current_status = "degraded"
    db.session.commit()

    # Push to Redis ring buffer (LPUSH + LTRIM keeps the last N)
    ring_key = f"checks:ring:{svc.id}"
    import json
    redis.lpush(ring_key, json.dumps({
        "at": check.checked_at.isoformat(),
        "status": check.status,
        "ms": check.response_ms,
        "code": check.status_code,
    }))
    redis.ltrim(ring_key, 0, 1439)  # keep last 1440 (24h at 1/min)

    # Real-time push to any browser viewing this service
    socketio.emit("check.completed", {
        "service_id": svc.id,
        "status": svc.current_status,
        "response_ms": svc.last_response_ms,
        "checked_at": svc.last_check_at.isoformat(),
        "error": svc.last_error,
    }, room=f"service:{svc.id}")

    # Open / escalate an incident if threshold crossed
    if svc.consecutive_failures == 3:  # exactly on the third failure
        _open_incident(svc)
    maybe_alert(svc)

@shared_task
def check_ssl_certs() -> None:
    """Daily SSL cert expiry check for every service with kind='http' on HTTPS."""
    services = db.session.scalars(
        select(Service).where(Service.kind == "http", Service.url.like("https://%"))
    ).all()
    for svc in services:
        from urllib.parse import urlparse
        host = urlparse(svc.url).hostname
        result = check_ssl(host)
        # Store as a special check row with kind=ssl metadata
        check = Check(service_id=svc.id, status=result["status"],
                      error=result["error"], response_ms=result.get("response_ms"))
        db.session.add(check)
        # Optionally alert on cert expiry
        if result["status"] == "fail" and "expired" in (result["error"] or ""):
            _open_incident(svc, kind="ssl",
                           title=f"SSL certificate expired for {svc.name}")
    db.session.commit()

@shared_task
def pull_github_status() -> None:
    """Every 5 minutes, pull latest GitHub Actions runs for configured repos."""
    from flask import current_app
    repos = current_app.config["GITHUB_REPOS"]
    for repo_full in repos:
        if not repo_full.strip():
            continue
        runs = latest_runs(repo_full.strip(), limit=1)
        if runs:
            r = runs[0]
            # Cache the last run per repo in Redis (cheap read for dashboard)
            import json
            redis.setex(f"github:last_run:{repo_full}", 600,
                        json.dumps(r))
            # If the latest run failed, surface it
            if r["conclusion"] == "failure":
                socketio.emit("github.failed", {
                    "repo": repo_full, "run": r,
                }, room="dashboard")

def _open_incident(svc: Service, kind: str = "service",
                   title: str | None = None) -> Incident:
    """Create an incident row if there isn't an open one for this service."""
    existing = db.session.scalar(
        select(Incident).where(Incident.service_id == svc.id,
                               Incident.status != "resolved")
    )
    if existing:
        return existing
    inc = Incident(
        service_id=svc.id,
        title=title or f"{svc.name} is {svc.current_status}",
        status="investigating",
        severity="major" if svc.current_status == "down" else "minor",
    )
    db.session.add(inc)
    db.session.commit()
    # Broadcast to dashboard
    socketio.emit("incident.opened", {
        "id": inc.id, "service_id": svc.id, "title": inc.title,
        "severity": inc.severity, "status": inc.status,
    }, room="dashboard")
    return inc
```

### Step 9 — Alert routing with cooldown

```python
# app/alerts.py
"""Send alerts (Slack, email) with cooldown so we don't spam."""
from datetime import datetime, timezone, timedelta
import json
import httpx
from flask import current_app, render_template_string
from app.extensions import db, redis, mail
from app.models import Notification
from flask_mail import Message

SLACK_TEMPLATE = """
*🚨 {{ svc.name }} is {{ svc.current_status }}*
Last error: `{{ svc.last_error or "n/a" }}`
Response time: {{ svc.last_response_ms or "—" }} ms
Dashboard: https://{{ domain }}/dashboard/services/{{ svc.id }}
"""

def maybe_alert(svc) -> None:
    """Fire an alert only if cooldown has elapsed."""
    if svc.current_status == "operational":
        # Recovery alert — short-circuit if no recent alert
        if redis.get(f"alert:recovery:{svc.id}"):
            return
        _send_recovery(svc)
        return
    cooldown_key = f"alert:cooldown:{svc.id}"
    if redis.get(cooldown_key):
        return  # alerted recently, suppress
    _send_alert(svc)
    redis.setex(cooldown_key,
                current_app.config["ALERT_COOLDOWN_MINUTES"] * 60, "1")
    # Track recovery so we don't send recovery alert on the first OK after start
    redis.setex(f"alert:recovery:{svc.id}",
                current_app.config["ALERT_COOLDOWN_MINUTES"] * 60, "1")

def _send_alert(svc) -> None:
    msg = render_template_string(SLACK_TEMPLATE, svc=svc,
                                 domain="pulseops.example.com")
    _send_slack(msg)
    _send_email(f"[PulseOps] {svc.name} is {svc.current_status}", msg)
    db.session.add(Notification(service_id=svc.id, channel="slack+email",
                                message=msg))
    db.session.commit()

def _send_recovery(svc) -> None:
    msg = f"✅ {svc.name} recovered (was {svc.current_status})"
    _send_slack(msg)
    db.session.add(Notification(service_id=svc.id, channel="slack",
                                message=msg))
    db.session.commit()

def _send_slack(text: str) -> None:
    url = current_app.config["SLACK_WEBHOOK_URL"]
    if not url:
        return
    try:
        httpx.post(url, json={"text": text}, timeout=5.0)
    except Exception:
        pass  # alerts should never break the check pipeline

def _send_email(subject: str, body: str) -> None:
    try:
        msg = Message(subject=subject, recipients=[current_app.config["ALERT_EMAIL_TO"]],
                      body=body)
        mail.send(msg)
    except Exception:
        pass
```

> [!info] Alert fatigue is the #1 monitoring failure
> A service that flaps (down → up → down every 30 s) will fire 60 alerts per minute without a cooldown. The team learns to ignore alerts; the next real outage is missed. PulseOps's `ALERT_COOLDOWN_MINUTES=30` means at most 2 alerts per service per hour — enough to page on-call without burning trust. The same logic powers PagerDuty's "dedup" feature.

### Step 10 — SocketIO handlers

```python
# app/sockets.py
from flask_socketio import join_room, leave_room
from app.extensions import socketio

@socketio.on("subscribe")
def on_subscribe(data):
    """Client subscribes to a service's room — receives its check.completed events."""
    service_id = data.get("service_id")
    if service_id:
        join_room(f"service:{service_id}")

@socketio.on("subscribe_dashboard")
def on_subscribe_dashboard():
    """Subscribe to dashboard-wide events (incident.opened, github.failed)."""
    join_room("dashboard")

@socketio.on("unsubscribe")
def on_unsubscribe(data):
    service_id = data.get("service_id")
    if service_id:
        leave_room(f"service:{service_id}")
```

### Step 11 — Real-time check flow

```mermaid
sequenceDiagram
    participant CB as Celery Beat
    participant Q as Celery Worker
    participant S as External Service
    participant PG as PostgreSQL
    participant R as Redis ring buffer
    participant WS as SocketIO
    participant B as Browser dashboard
    participant SL as Slack

    CB->>Q: run_due_checks (every 60s)
    Q->>PG: SELECT services WHERE is_active
    PG-->>Q: [svc1, svc2, svc3]
    loop for each due service
        Q->>Q: run_check.delay(svc.id)
    end

    Note over Q: in worker process
    Q->>S: HTTP GET svc.url (timeout=10s)
    S-->>Q: 200 OK in 142ms
    Q->>PG: INSERT check row
    Q->>PG: UPDATE service SET last_check_at, current_status
    Q->>R: LPUSH checks:ring:N + LTRIM 0 1439
    Q->>WS: emit check.completed room=service:N
    WS->>B: receive event → update card

    Note over Q: on 3rd consecutive failure
    Q->>PG: INSERT incident
    Q->>WS: emit incident.opened room=dashboard
    Q->>SL: POST webhook (with cooldown)
    WS->>B: incident banner appears
```

### Step 12 — Dashboard blueprint with HTMX partials

```python
# app/dashboard.py
import json
from flask import Blueprint, render_template, jsonify, g, request
from flask_login import login_required
from sqlalchemy import select, desc
from app.extensions import db, redis
from app.models import Service, Incident
from app.docker_client import list_containers

dashboard_bp = Blueprint("dashboard", __name__)

@dashboard_bp.get("/")
@login_required
def index():
    services = db.session.scalars(
        select(Service).where(Service.is_active == True)
        .order_by(Service.name)
    ).all()
    open_incidents = db.session.scalars(
        select(Incident).where(Incident.status != "resolved")
        .order_by(Incident.started_at.desc())
    ).all()
    return render_template("dashboard/index.html",
                           services=services, incidents=open_incidents)

@dashboard_bp.get("/services/<int:service_id>")
@login_required
def service_detail(service_id: int):
    svc = db.session.get(Service, service_id)
    if not svc:
        return "", 404
    # Read last 60 checks from Redis ring buffer
    ring = redis.lrange(f"checks:ring:{svc.id}", 0, 59)
    checks = [json.loads(r) for r in ring]
    # Reverse so oldest is first (chart x-axis goes left → right)
    checks.reverse()
    return render_template("dashboard/service.html",
                           service=svc, checks=checks)

@dashboard_bp.get("/services/<int:service_id>/card")
@login_required
def service_card_partial(service_id: int):
    """HTMX partial — just the card. Used by polls as a fallback to SocketIO."""
    svc = db.session.get(Service, service_id)
    if not svc:
        return "", 404
    return render_template("partials/service_card.html", service=svc)

@dashboard_bp.get("/containers")
@login_required
def containers():
    """Docker container monitoring view."""
    try:
        containers = list_containers(all_states=True)
    except Exception as e:
        containers = []
        g.docker_error = str(e)
    return render_template("dashboard/containers.html", containers=containers)

@dashboard_bp.get("/github")
@login_required
def github_view():
    """GitHub Actions CI/CD view across configured repos."""
    from flask import current_app
    repos = []
    for repo_full in current_app.config["GITHUB_REPOS"]:
        if not repo_full.strip():
            continue
        raw = redis.get(f"github:last_run:{repo_full.strip()}")
        last = json.loads(raw) if raw else None
        repos.append({"name": repo_full.strip(), "last_run": last})
    return render_template("dashboard/github.html", repos=repos)
```

### Step 13 — Dashboard template (HTMX + Tailwind + Chart.js)

```html
<!-- app/templates/dashboard/index.html -->
{% extends "base.html" %}
{% block content %}
<div class="max-w-7xl mx-auto px-4 py-6" id="dashboard-root"
     hx-get="/dashboard/" hx-trigger="every 30s" hx-select="#dashboard-root"
     hx-swap="outerHTML">

  <div class="flex items-center justify-between mb-6">
    <h1 class="text-2xl font-bold">Dashboard</h1>
    <span id="last-updated" class="text-sm text-gray-500">
      Updated <span x-text="new Date().toLocaleTimeString()"></span>
    </span>
  </div>

  {% if incidents %}
  <div class="mb-6 space-y-2">
    {% for inc in incidents %}
    <div class="p-4 rounded-lg border-l-4
                {{ 'border-red-500 bg-red-50' if inc.severity == 'critical'
                   else 'border-amber-500 bg-amber-50' if inc.severity == 'major'
                   else 'border-yellow-400 bg-yellow-50' }}">
      <div class="flex justify-between items-center">
        <div>
          <span class="font-medium">{{ inc.title }}</span>
          <span class="ml-2 text-xs uppercase tracking-wide
                       {{ 'text-red-700' if inc.severity == 'critical'
                          else 'text-amber-700' }}">
            {{ inc.severity }}
          </span>
        </div>
        <a href="/incidents/{{ inc.id }}" class="text-sm text-blue-600">View →</a>
      </div>
    </div>
    {% endfor %}
  </div>
  {% endif %}

  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
    {% for svc in services %}
      {% include "partials/service_card.html" %}
    {% endfor %}
  </div>
</div>

<script>
  // SocketIO client — receives real-time updates
  const socket = io();
  socket.emit("subscribe_dashboard", {});

  socket.on("check.completed", (data) => {
    const card = document.querySelector(`[data-service-id="${data.service_id}"]`);
    if (!card) return;
    // Update status badge
    card.querySelector("[data-status]").dataset.status = data.status;
    card.querySelector("[data-status]").textContent = data.status;
    card.querySelector("[data-latency]").textContent =
      data.response_ms ? data.response_ms + " ms" : "—";
    // Flash green/red border briefly
    card.classList.add(data.status === "operational" ? "ring-green-400" : "ring-red-400");
    setTimeout(() => card.classList.remove("ring-green-400", "ring-red-400"), 1000);
  });

  socket.on("incident.opened", (data) => {
    // Reload the page to show the incident banner — simplest correct behavior
    location.reload();
  });
</script>
{% endblock %}
```

```html
<!-- app/templates/partials/service_card.html -->
<a href="/dashboard/services/{{ svc.id }}"
   data-service-id="{{ svc.id }}"
   class="block p-4 rounded-lg border bg-white shadow-sm hover:shadow-md transition ring-2 ring-transparent"
   :class="{'border-red-300': svc.current_status === 'down'}">
  <div class="flex justify-between items-start">
    <div>
      <div class="font-medium">{{ svc.name }}</div>
      <div class="text-xs text-gray-500 truncate max-w-[200px]">{{ svc.url }}</div>
    </div>
    <span data-status="{{ svc.current_status }}"
          class="text-xs px-2 py-1 rounded-full
                 {{ 'bg-green-100 text-green-800' if svc.current_status == 'operational'
                    else 'bg-red-100 text-red-800' if svc.current_status == 'down'
                    else 'bg-amber-100 text-amber-800' if svc.current_status == 'degraded'
                    else 'bg-gray-100 text-gray-800' }}">
      {{ svc.current_status }}
    </span>
  </div>
  <div class="mt-3 flex justify-between text-xs text-gray-600">
    <span data-latency>
      {{ svc.last_response_ms ~ ' ms' if svc.last_response_ms else '—' }}
    </span>
    <span>
      {% if svc.last_check_at %}
        {{ svc.last_check_at.strftime('%H:%M:%S') }}
      {% else %}
        never
      {% endif %}
    </span>
  </div>
</a>
```

### Step 14 — Service detail with Chart.js

```html
<!-- app/templates/dashboard/service.html -->
{% extends "base.html" %}
{% block content %}
<div class="max-w-5xl mx-auto px-4 py-6">
  <h1 class="text-2xl font-bold mb-1">{{ service.name }}</h1>
  <p class="text-sm text-gray-500 mb-6">{{ service.url }}</p>

  <div class="bg-white rounded-lg shadow p-6 mb-6">
    <h2 class="font-semibold mb-4">Response time (last 60 checks)</h2>
    <canvas id="latencyChart" height="80"></canvas>
  </div>

  <div class="bg-white rounded-lg shadow p-6">
    <h2 class="font-semibold mb-4">Recent checks</h2>
    <table class="w-full text-sm">
      <thead><tr class="text-left text-gray-500 border-b">
        <th class="py-2">Time</th><th>Status</th><th>Latency</th><th>Code</th><th>Error</th>
      </tr></thead>
      <tbody>
        {% for c in checks %}
        <tr class="border-b">
          <td class="py-2">{{ c.at }}</td>
          <td>
            <span class="px-2 py-0.5 rounded text-xs
              {{ 'bg-green-100 text-green-800' if c.status == 'ok' else 'bg-red-100 text-red-800' }}">
              {{ c.status }}
            </span>
          </td>
          <td>{{ c.ms or '—' }} ms</td>
          <td>{{ c.code or '—' }}</td>
          <td class="text-gray-500">{{ c.error or '' }}</td>
        </tr>
        {% endfor %}
      </tbody>
    </table>
  </div>
</div>

<script src="https://cdn.jsdelivr.net/npm/chart.js@4"></script>
<script>
  const checks = {{ checks | tojson }};
  const labels = checks.map(c => new Date(c.at).toLocaleTimeString());
  const data = checks.map(c => c.ms || 0);

  const ctx = document.getElementById('latencyChart').getContext('2d');
  const chart = new Chart(ctx, {
    type: 'line',
    data: { labels, datasets: [{
      label: 'Latency (ms)', data, borderColor: '#3b82f6',
      backgroundColor: 'rgba(59,130,246,0.1)', fill: true, tension: 0.3,
    }] },
    options: { responsive: true, scales: { y: { beginAtZero: true } } },
  });

  // Subscribe to live updates for this service
  const socket = io();
  socket.emit("subscribe", { service_id: {{ service.id }} });
  socket.on("check.completed", (d) => {
    if (d.service_id !== {{ service.id }}) return;
    chart.data.labels.push(new Date(d.checked_at).toLocaleTimeString());
    chart.data.datasets[0].data.push(d.response_ms || 0);
    // Keep last 60 points
    if (chart.data.labels.length > 60) {
      chart.data.labels.shift();
      chart.data.datasets[0].data.shift();
    }
    chart.update('none');  // skip animation for snappy feel
  });
</script>
{% endblock %}
```

### Step 15 — Public status page

```python
# app/status.py
from flask import Blueprint, render_template, jsonify
from sqlalchemy import select, desc
from app.extensions import db
from app.models import Service, Incident

status_bp = Blueprint("status", __name__)

@status_bp.get("/")
def public_status():
    """Public-facing status page — no auth required."""
    services = db.session.scalars(
        select(Service).where(Service.is_active == True, Service.is_public == True)
        .order_by(Service.name)
    ).all()
    # Open + recent incidents (last 30 days)
    from datetime import datetime, timezone, timedelta
    cutoff = datetime.now(timezone.utc) - timedelta(days=30)
    incidents = db.session.scalars(
        select(Incident).where(
            (Incident.status != "resolved") | (Incident.resolved_at >= cutoff)
        ).order_by(desc(Incident.started_at)).limit(20)
    ).all()
    # Overall status: green if all operational, amber if any degraded, red if any down
    statuses = [s.current_status for s in services]
    overall = ("operational" if all(s == "operational" for s in statuses) and statuses
               else "down" if "down" in statuses
               else "degraded" if "degraded" in statuses
               else "unknown")
    return render_template("status/index.html",
                           services=services, incidents=incidents,
                           overall=overall)

@status_bp.get("/api")
def public_status_api():
    """JSON version for embedding in other tools."""
    services = db.session.scalars(
        select(Service).where(Service.is_active == True, Service.is_public == True)
    ).all()
    return jsonify({
        "services": [{"name": s.name, "status": s.current_status} for s in services],
    })
```

### Step 16 — Service CRUD blueprint

```python
# app/services.py
from flask import Blueprint, request, redirect, url_for, flash, render_template
from flask_login import login_required, current_user
from app.extensions import db
from app.models import Service
from app.forms import ServiceForm  # Flask-WTF form

services_bp = Blueprint("services", __name__)

@services_bp.get("/")
@login_required
def list_services():
    services = db.session.scalars(db.select(Service).order_by(Service.name)).all()
    return render_template("services/list.html", services=services)

@services_bp.route("/new", methods=["GET", "POST"])
@login_required
def new_service():
    form = ServiceForm()
    if form.validate_on_submit():
        svc = Service(
            name=form.name.data,
            url=form.url.data,
            kind=form.kind.data,
            expected_status=form.expected_status.data or 200,
            check_interval_seconds=form.check_interval_seconds.data or 60,
            timeout_seconds=form.timeout_seconds.data or 10,
            is_public=form.is_public.data,
        )
        db.session.add(svc)
        db.session.commit()
        flash(f"Service {svc.name} added — first check in ~60 s.", "success")
        return redirect(url_for("dashboard.index"))
    return render_template("services/form.html", form=form)

@services_bp.post("/<int:service_id>/delete")
@login_required
def delete_service(service_id: int):
    svc = db.session.get(Service, service_id)
    if svc:
        svc.is_active = False  # soft delete — preserve history
        db.session.commit()
        flash(f"Service {svc.name} paused.", "info")
    return redirect(url_for("services.list_services"))
```

### Step 17 — Incident management blueprint

```python
# app/incidents.py
from datetime import datetime, timezone
from flask import Blueprint, request, redirect, url_for, flash, render_template
from flask_login import login_required
from sqlalchemy import select
from app.extensions import db
from app.models import Incident, IncidentUpdate
from app.forms import IncidentUpdateForm

incidents_bp = Blueprint("incidents", __name__)

@incidents_bp.get("/")
@login_required
def list_incidents():
    incs = db.session.scalars(
        select(Incident).order_by(Incident.started_at.desc()).limit(50)
    ).all()
    return render_template("incidents/list.html", incidents=incs)

@incidents_bp.get("/<int:incident_id>")
@login_required
def incident_detail(incident_id: int):
    inc = db.session.get(Incident, incident_id)
    if not inc:
        return "", 404
    form = IncidentUpdateForm()
    return render_template("incidents/detail.html", incident=inc, form=form)

@incidents_bp.post("/<int:incident_id>/update")
@login_required
def add_update(incident_id: int):
    inc = db.session.get(Incident, incident_id)
    if not inc:
        return "", 404
    form = IncidentUpdateForm(request.form)
    if form.validate():
        upd = IncidentUpdate(
            incident_id=inc.id,
            body=form.body.data,
            new_status=form.new_status.data,
        )
        db.session.add(upd)
        inc.status = form.new_status.data
        if inc.status == "resolved":
            inc.resolved_at = datetime.now(timezone.utc)
        db.session.commit()
        flash("Incident updated.", "success")
    return redirect(url_for("incidents.incident_detail", incident_id=inc.id))
```

---

## 7. Running the Project

### `.env.example`

```bash
SECRET_KEY=change-me
DATABASE_URL=postgresql+psycopg://pulseops:pulsepass@postgres:5432/pulseops
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=redis://redis:6379/1
CELERY_RESULT_BACKEND=redis://redis:6379/2
DOCKER_SOCKET=unix:///var/run/docker.sock
GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
GITHUB_REPOS=yourorg/yourrepo,yourorg/anotherrepo
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
ALERT_EMAIL_TO=ops@example.com
MAIL_SERVER=smtp.mailtrap.io
MAIL_PORT=587
MAIL_USERNAME=
MAIL_PASSWORD=
```

### `docker-compose.yml`

```yaml
services:
  web:
    build: .
    command: gunicorn -k eventlet -w 1 -b 0.0.0.0:5000 wsgi:app
    ports: ["5000:5000"]
    env_file: .env
    volumes:
      # Mount the host's Docker socket so we can inspect host containers
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on: [postgres, redis]
  worker:
    build: .
    command: celery -A celery_worker.celery worker --loglevel=info --concurrency=4
    env_file: .env
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on: [redis, postgres]
  beat:
    build: .
    command: celery -A celery_worker.celery beat --loglevel=info
    env_file: .env
    depends_on: [redis]
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: pulseops
      POSTGRES_USER: pulseops
      POSTGRES_PASSWORD: pulsepass
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
    volumes: ["redisdata:/data"]
volumes: { pgdata: {}, redisdata: {} }
```

### `wsgi.py` and `celery_worker.py`

```python
# wsgi.py
from app import create_app, socketio
app = create_app()

if __name__ == "__main__":
    socketio.run(app, host="0.0.0.0", port=5000)
```

```python
# celery_worker.py
from app import create_app
from app.extensions import celery
app = create_app()
celery_app = celery
import app.tasks  # noqa: F401  (register tasks)
```

### First run

```bash
cp .env.example .env
# Fill in GITHUB_TOKEN, SLACK_WEBHOOK_URL (both optional)
docker compose up --build -d postgres redis
docker compose run --rm web flask --app wsgi db upgrade
docker compose up --build -d web worker beat

# Create an admin user:
docker compose run --rm web python -c "
from app import create_app
from app.extensions import db
from app.models import User
app = create_app()
with app.app_context():
    db.create_all()
    u = User(email='admin@x.com', is_admin=True)
    u.set_password('hunter2hunter2')
    db.session.add(u); db.session.commit()
    print('admin user created')
"

# Visit http://localhost:5000/auth/login, log in,
# add a service: GET https://example.com, expected 200, every 60s.
# Within ~60s the dashboard should show it as operational, with a green card.
# Stop your service (or change expected to 500) — within 3 minutes you'll
# see the card flip to red, an incident appear, and (if Slack is configured)
# a Slack message.
```

---

## 8. Testing

```python
# tests/conftest.py
import pytest
from app import create_app
from app.config import TestConfig
from app.extensions import db as _db
from app.models import User

@pytest.fixture
def app():
    app = create_app(TestConfig)
    with app.app_context():
        _db.create_all()
        u = User(email="admin@x.com", is_admin=True)
        u.set_password("x"*12)
        _db.session.add(u); _db.session.commit()
        yield app
        _db.session.remove()
        _db.drop_all()

@pytest.fixture
def client(app):
    return app.test_client()

@pytest.fixture
def db(app):
    return _db
```

```python
# tests/test_checks.py
from app.checks import check_http
import responses

@responses.activate
def test_check_http_success():
    responses.add(responses.GET, "https://api.example.com/health",
                  status=200, json={"ok": True})
    r = check_http("https://api.example.com/health", 200, 5)
    assert r["status"] == "ok"
    assert r["status_code"] == 200

@responses.activate
def test_check_http_wrong_status():
    responses.add(responses.GET, "https://api.example.com/health",
                  status=500)
    r = check_http("https://api.example.com/health", 200, 5)
    assert r["status"] == "fail"
    assert "500" in r["error"]

def test_check_tcp_unreachable():
    r = check_tcp("127.0.0.1", 1, 1)  # port 1 — nothing listening
    assert r["status"] == "fail"
```

```python
# tests/test_alerts.py
def test_cooldown_suppresses_repeat(app, db, monkeypatch):
    from app.models import Service
    from app import alerts
    sent = []
    monkeypatch.setattr(alerts, "_send_slack", lambda msg: sent.append(msg))
    monkeypatch.setattr(alerts, "_send_email", lambda s, b: None)

    svc = Service(name="t", url="https://t.example.com", kind="http",
                  current_status="down", last_error="boom",
                  last_response_ms=None, id=1)
    db.session.add(svc); db.session.commit()

    alerts.maybe_alert(svc)
    alerts.maybe_alert(svc)  # should be suppressed by cooldown
    alerts.maybe_alert(svc)
    assert len(sent) == 1  # only the first alert fired
```

```python
# tests/test_dashboard.py
def test_dashboard_requires_login(client):
    r = client.get("/dashboard/")
    assert r.status_code == 302  # redirect to login

def test_status_page_public(client):
    r = client.get("/status/")
    assert r.status_code == 200
    # Should show "all systems operational" or "unknown" with no services
    assert b"operational" in r.data.lower() or b"unknown" in r.data.lower()
```

> [!warning] Mocking Docker SDK in tests
> Don't try to spin up a Docker daemon in tests. Patch `app.docker_client._get_client` with a fake that returns canned `containers.list()` results. Same for `app.github_client._get_client` — patch with a fake `Github` returning canned workflow runs. Unit tests should never reach external services.

---

## 9. Deployment

### Production checklist

- [ ] **Mount Docker socket only on trusted hosts.** `/var/run/docker.sock` gives root-equivalent access to the host. In production, run PulseOps in a container with `--user` and use [socket-proxy](https://github.com/Tecnativa/docker-socket-proxy) to expose only read-only Docker API methods.
- [ ] **GitHub token with minimal scopes** — `repo:status` is enough for public repos; `repo` for private. Rotate every 90 days.
- [ ] **Slack webhook** stored in a secrets manager, not in `.env`. Use a dedicated `#alerts` channel; don't ping `@here` — that's how you burn trust.
- [ ] **Single Celery beat instance** — never run two beat schedulers, you'll get duplicate checks. Use a leader-election pattern or a single-replica Deployment.
- [ ] **eventlet for SocketIO** — `gunicorn -k eventlet -w 1`. Multiple workers won't share SocketIO state without a Redis/message-queue adapter. For >1 worker, configure `SOCKETIO_MESSAGE_QUEUE=redis://...`.
- [ ] **PostgreSQL** with autovacuum tuned — the `checks` table grows fast (1 row per service per minute). Add a retention policy: a Celery beat job that `DELETE FROM checks WHERE checked_at < now() - INTERVAL '30 days'` weekly.
- [ ] **Redis ring buffer is bounded by `LTRIM`** — never grows past 1,440 entries per service. But `notifications` and `incidents` tables do grow — same retention policy applies.
- [ ] **HTTPS everywhere** — the dashboard itself should be HTTPS; an attacker on the wire could read your auth cookie or inject fake status events. Use Caddy or nginx with Let's Encrypt.
- [ ] **Public status page** behind a CDN (Cloudflare) — during an outage, traffic to `/status/` spikes 100×. Static-cache the page with a 30 s TTL; only the API endpoint needs to be dynamic.
- [ ] **Alert routing tier** — `critical` → PagerDuty (page on-call); `major` → Slack `#alerts`; `minor` → email digest. Don't page for minor.
- [ ] **Synthetic checks from multiple regions** — a service might be up in us-east but down in eu-west. Deploy PulseOps workers in 2-3 regions; treat disagreement as `degraded`.

### Scaling beyond a single PulseOps instance

```mermaid
flowchart TB
    LB[Load Balancer] --> W1[Flask web 1<br/>eventlet + SocketIO]
    LB --> W2[Flask web 2<br/>eventlet + SocketIO]
    W1 <-->|Redis pub/sub<br/>message queue| W2
    W1 & W2 --> PG[(PostgreSQL<br/>+ PgBouncer)]
    W1 & W2 --> R[(Redis<br/>ring buffer + SocketIO MQ)]
    CB[Celery Beat<br/>single instance] --> Q[Celery worker pool]
    Q --> EXT[External services<br/>HTTP / Docker / GitHub]
    Q --> R
    Q --> W1 & W2
    style R fill:#dcfce7,stroke:#16a34a
    style Q fill:#fef3c7,stroke:#d97706
```

For multiple Flask web workers, SocketIO needs a shared message queue (Redis pub/sub) so an event emitted from a Celery worker reaches every browser, regardless of which web worker it's connected to. Flask-SocketIO supports this natively via `SOCKETIO_MESSAGE_QUEUE`.

---

## 10. Next Steps / Enhancements

1. **Log aggregation.** Use `docker_client.tail_logs()` to power a `/dashboard/containers/<name>/logs` view with auto-scroll and SocketIO streaming. Same pattern as `docker logs -f`.
2. **Synthetic transactions.** Beyond "is the URL up?", run a scripted flow: log in, create a resource, delete it. Use Playwright headless. This catches "page loads but the database is broken" outages.
3. **Multi-region checks.** Deploy PulseOps workers in 3 regions. A service is `operational` only if at least 2 of 3 regions see it green.
4. **Anomaly detection.** Replace static thresholds with a rolling z-score on response time: alert when `ms > mean + 3*stddev` over the last hour. Catches "still up but 10× slower" outages.
5. **On-call schedule.** A `Schedule` model mapping (day, hour) → user. Route alerts to the on-call user's phone via Twilio.
6. **Status page subscribers.** Let customers subscribe to incident emails. `Subscriber` table + Celery task that emails them on `incident.opened` and `incident.resolved`.
7. **Prometheus exporter.** Add a `/metrics` endpoint (`prometheus_client`) exposing `pulseops_service_up{service="api"} 1` and `pulseops_response_ms{service="api"} 142`. Let teams scrape PulseOps into their existing Prometheus.
8. **Maintenance windows.** A `MaintenanceWindow` model with `start_at`, `end_at`, `service_id`. Checks are paused during the window; no incidents fire. The status page shows "Scheduled maintenance" instead of "Down".
9. **Webhook ingestion.** Accept webhooks from GitHub, Sentry, Stripe — display them inline alongside internal checks. PulseOps becomes a unified event feed.
10. **Mobile app + push notifications.** A React Native app that connects to the same SocketIO endpoint and shows push notifications for `critical` incidents. Critical-team must-have.

---

## 11. What We Learned

- **Docker SDK for Python** is the right way to inspect container state from code — typed, versioned, no subprocess overhead. Mount `/var/run/docker.sock` carefully; it's root-equivalent.
- **GitHub API via PyGithub** integrates CI/CD status into the same dashboard as uptime — no context-switching. Watch the 5,000 req/hour token limit.
- **Flask-SocketIO with per-service rooms** is the real-time pattern: `socketio.emit(event, room=f"service:{id}")` pushes only to subscribed browsers. Browsers not viewing a service don't waste bandwidth on its updates.
- **Celery beat** is the natural scheduler for periodic checks. Single beat instance + N workers = horizontal scaling with no duplicate work.
- **Redis ring buffer** (`LPUSH` + `LTRIM`) is the right data structure for "last N time-series values" — bounded memory, O(1) writes, O(N) reads. Avoids a TimescaleDB dependency for the simple case.
- **Alert cooldowns** prevent fatigue — `SETEX alert:cooldown:{service_id} 1800 1` after every alert suppresses repeats for 30 min. A flapping service produces 2 alerts per hour, not 60.
- **Incident lifecycle** (`investigating → identified → monitoring → resolved`) with an updates log is the standard SRE pattern — same as status.github.com, status.stripe.com.
- **HTMX + Tailwind** is the server-rendered UI stack of 2026. No React build step, no client state to desync — `hx-get="/dashboard/" hx-trigger="every 30s"` falls back gracefully when SocketIO isn't available.
- **Public status page with incident history** is a customer expectation, not a luxury. `is_public=True` on services, plus an unauthenticated `/status/` route, gives you a Stripe-style status page in 50 lines.
- **SSL cert checking** via `ssl.create_default_context()` + `socket.create_connection()` is the standard pattern — pull the cert, parse `not_after`, warn at 14 days. Cheap to run daily.
- **Denormalized service state** (`current_status`, `last_check_at`, `consecutive_failures` on the `Service` row) makes the dashboard query a single `SELECT * FROM services` instead of a subquery over the `checks` table.
- **Real-time push beats polling** — SocketIO updates within ~200 ms of a check completing; polling every 30 s feels sluggish and lies during outages.

> [!quote] A note from your teacher
> PulseOps is the project that taught *me* what observability actually means. It's not "log all the things" — it's "tell me about problems before users do, and tell me in a way I can act on." The three core primitives — *check, alert, status page* — are the same primitives behind every monitoring tool from Nagios to Datadog to PagerDuty. Build them once and you understand the entire SRE discipline: detection (Celery beat + checks), noise reduction (cooldowns + thresholds), communication (incidents + status page), and integration (Docker SDK, GitHub API, Slack). That's a senior engineer's toolkit in one project. Build it, deploy it, point it at your own side projects — and watch how fast you find the next bug.

```mermaid
flowchart LR
    subgraph Inputs
        HTTP[HTTP check<br/>httpx]
        TCP[TCP check<br/>socket]
        SSL[SSL check<br/>ssl + socket]
        DOCKER[Docker check<br/>docker SDK]
        GH[GitHub check<br/>PyGithub]
    end
    subgraph PulseOps Core
        CB[Celery Beat<br/>every 60s] --> Q[Celery Worker]
        Q --> CHECKS[checks.py<br/>dispatch by kind]
        HTTP & TCP & SSL & DOCKER & GH --> CHECKS
        CHECKS --> PG[(PostgreSQL<br/>services, checks, incidents)]
        CHECKS --> R[(Redis<br/>ring buffer + cooldowns)]
        CHECKS --> SOCK[SocketIO<br/>emit per-service]
        CHECKS --> ALERT[alerts.py]
        ALERT --> SLACK[Slack webhook]
        ALERT --> MAIL[Email via SMTP]
    end
    subgraph UI
        SOCK --> DASH[Dashboard<br/>HTMX + Chart.js]
        SOCK --> DETAIL[Service detail<br/>live chart]
        PG --> PUB[/status/<br/>public status page]
    end
    style R fill:#dcfce7,stroke:#16a34a
    style SOCK fill:#dbeafe,stroke:#2563eb
    style Q fill:#fef3c7,stroke:#d97706
```
