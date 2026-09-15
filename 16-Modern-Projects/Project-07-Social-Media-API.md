---
title: "Project 07 — Social Media API (Echo)"
module: 16-Modern-Projects
project_number: 7
estimated_time: "14–18 hours build, 40–60 hours to polish"
difficulty: advanced
prerequisites:
  - "Complete the core course (Lessons 01–24)"
  - "[[Lesson-12-SQLAlchemy-Setup]] — models with `Mapped` columns"
  - "[[Lesson-14-Relationships]] — Follow graph, Post → Comment → Like"
  - "[[Lesson-18-Blueprints]] — auth, posts, feed, social, search blueprints"
  - "[[Lesson-19-Building-JSON-API]] — REST + cursor pagination"
  - "[[Lesson-20-RESTful-Resources]] — resource modeling"
  - "[[Lesson-22-Email-And-Tasks]] — Celery for notification fanout"
  - "[[Modern-Databases-And-Caching]] — Redis sorted sets, denormalized counters"
  - "[[Modern-Flask-And-Python]] — Pydantic v2, JWT patterns"
  - "[[Project-03-SaaS-Starter-Kit]] — JWT + API keys (same primitives)"
tags:
  - flask
  - flask3
  - python313
  - project
  - portfolio
  - social
  - api
  - jwt
  - redis
  - celery
  - pydantic
  - s3
  - 2026
related_lessons:
  - "[[Lesson-13-CRUD-Operations]]"
  - "[[Lesson-18-Blueprints]]"
  - "[[Lesson-19-Building-JSON-API]]"
  - "[[Lesson-20-RESTful-Resources]]"
  - "[[Lesson-22-Email-And-Tasks]]"
  - "[[Modern-Databases-And-Caching]]"
  - "[[Modern-Flask-And-Python]]"
  - "[[Modern-Deployment-And-DevOps]]"
  - "[[Project-03-SaaS-Starter-Kit]]"
---

# Project 07 — Social Media API ("Echo")

> [!success] Build the backend behind a Twitter / Instagram clone.
> By the end of this project you'll have a production-grade social API: users register, log in with JWT, post text and images, follow each other, and read a personalized timeline feed built with **fanout-on-write** using Redis sorted sets. Likes, comments, hashtags, search, and notifications are all there. Rate limiting protects the API. Pre-signed S3 / Cloudflare R2 URLs handle image uploads. Celery workers fan out notifications asynchronously. This is the project that proves you can build a *backend* — recruiters specifically look for API-only projects.

#flask #api #social #jwt #redis #celery #pydantic #s3 #portfolio #2026

---

## 1. Project Overview

**Echo** is a REST API for a social network. Clients (a mobile app, a React web client, a CLI tool — you name it) talk to it over HTTPS with JSON. The server is stateless: every request carries a JWT in the `Authorization` header, and the feed is *pre-computed* in Redis so reads are O(log N) instead of a multi-table join.

The defining engineering decision is the **feed architecture**. A naive feed query — "give me the last 20 posts from people I follow, newest first" — runs as:

```sql
SELECT * FROM posts
WHERE author_id IN (SELECT followed_id FROM follows WHERE follower_id = $me)
ORDER BY created_at DESC LIMIT 20;
```

That works at 100 users. At 100k users with an average of 200 followings each, the `IN` clause becomes a 200-element scan and the `ORDER BY` forces a sort over a wide row set. The fix is **fanout-on-write**: when a user posts, push the post ID into each follower's pre-computed feed list (a Redis sorted set) at write time. Reads become `ZREVRANGEBYSCORE feed:{user_id}` — a single O(log N) call.

### Why this is a modern 2026 project

A "Flask + SQLite social app" tutorial from 2018 was a 200-line `app.py`. Modern social backends in 2026 look completely different:

1. **JWT with refresh tokens is the default.** Cookie sessions don't survive when your web client is on `app.example.com` and your API is on `api.example.com`. JWTs also work natively for mobile clients.
2. **Fanout-on-write is the standard feed algorithm.** Twitter's 2010-era blog post is required reading for any backend engineer. Knowing *when* to fanout-on-write vs fanout-on-read (and the celebrity hybrid) is the interview question.
3. **Pre-signed S3 / R2 URLs** offload image bytes from your app servers. The Flask app signs a URL; the client uploads directly to object storage; Flask only records the key.
4. **Pydantic v2 validates at the edge.** Request bodies are parsed and validated before they touch the ORM. Faster dev loop (the schema *is* the documentation) and fewer surprising 500s.
5. **Cursor-based pagination** replaces `?page=5` everywhere performance matters. New posts created while you're paginating shouldn't appear in two pages.
6. **Per-user rate limits** with tiered quotas (free vs pro) is how every public API works in 2026.
7. **Celery for notification fanout** — liking a post with 10k followers means writing 10k notification rows. That's not a 50 ms synchronous operation.

> [!info] Where this fits in the project series
> [[Project-03-SaaS-Starter-Kit]] taught you JWT, API keys, multi-tenancy, Stripe. **Echo reuses the JWT layer** but adds the social-graph workload (Follow + Feed + Notifications). Read P03 first — we won't re-explain access/refresh token rotation.

### Architecture at a glance

```mermaid
flowchart TB
    C[Client / Mobile App] -->|HTTPS + JWT| F[Flask API<br/>Blueprints: auth, posts, feed, social, search]
    F -->|SQLAlchemy 2.0| PG[(PostgreSQL 16<br/>users, posts, follows, likes, comments)]
    F -->|ZADD feed:uid| R[(Redis<br/>sorted sets, rate limits, cache)]
    F -->|presigned PUT| S3[(S3 / Cloudflare R2<br/>images)]
    F -->|enqueue fanout| Q[Celery Worker]
    Q -->|ZADD feed:follower| R
    Q -->|INSERT notifications| PG
    F -->|@token_required| JWT[JWT verify<br/>flask-jwt-extended]
    F -->|limiter.hit| RL[Flask-Limiter<br/>Redis backend]
    style F fill:#dbeafe,stroke:#2563eb
    style R fill:#dcfce7,stroke:#16a34a
    style Q fill:#fef3c7,stroke:#d97706
    style S3 fill:#fce7f3,stroke:#db2777
```

---

## 2. What You'll Learn

By the end you will have hands-on experience with:

- **Fanout-on-write vs fanout-on-read** — when to use each, why celebrities break naive fanout, the hybrid approach.
- **Redis sorted sets** (`ZADD`, `ZREVRANGEBYSCORE`, `ZRANGEBYSCORE`) for pre-computed feeds with timestamp scores.
- **JWT access + refresh tokens** — short-lived access tokens (15 min) rotated via a long-lived refresh token (30 days), with `jti` revocation.
- **Pre-signed S3 / R2 uploads** — the client uploads the bytes; Flask only signs URLs and stores the resulting key.
- **Pydantic v2 schemas** — `model_validate`, `model_dump`, `Field`, `model_config = ConfigDict(from_attributes=True)`.
- **Cursor-based pagination** — `?cursor=eyJpZCI6MTIz` instead of `?page=2`, base64-encoded JSON cursors.
- **Flask-Limiter with Redis storage** — per-user tiers (`free` vs `pro`), keyed on the authenticated user, not the IP.
- **Celery for async notification fanout** — liking a post enqueues a task that writes 10k notifications in batches.
- **API versioning** — `/api/v1/...` URL prefix, deprecation-friendly from day one.
- **Hashtag extraction** with regex, a `Hashtag` join table, and case-insensitive search.
- **Denormalized counters** (`like_count`, `comment_count`, `follower_count`) updated atomically to avoid count queries.

---

## 3. Prerequisites

| Lesson | Why |
|---|---|
| [[Lesson-12-SQLAlchemy-Setup]] | `Mapped` columns, model definition. |
| [[Lesson-14-Relationships]] | Follow graph (many-to-many self-join), Post → Comment. |
| [[Lesson-18-Blueprints]] | `auth`, `posts`, `feed`, `social`, `search` blueprints. |
| [[Lesson-19-Building-JSON-API]] | JSON responses, status codes. |
| [[Lesson-20-RESTful-Resources]] | Resource modeling for `/posts`, `/users`, `/feed`. |
| [[Lesson-22-Email-And-Tasks]] | Celery setup, async tasks. |
| [[Modern-Databases-And-Caching]] | Redis patterns, sorted sets. |
| [[Modern-Flask-And-Python]] | Pydantic v2, JWT ecosystem. |
| [[Project-03-SaaS-Starter-Kit]] | JWT + API key primitives (we reuse them). |

You also need:
- Python 3.13+, Docker, Docker Compose
- PostgreSQL 16 (Docker handles it)
- Redis 7 (Docker)
- An S3-compatible bucket — AWS S3, Cloudflare R2, MinIO (local), or Backblaze B2. The code is the same; only the endpoint URL changes.

---

## 4. Tech Stack

| Tool | Version | Why |
|---|---|---|
| Flask | 3.2+ | Web framework. |
| Flask-SQLAlchemy | 3.1+ | ORM with 2.0-style `Mapped` columns. |
| Flask-JWT-Extended | 4.7+ | Access + refresh JWTs, blocklist for revocation. |
| Flask-Limiter | 3.8+ | Per-user rate limits backed by Redis. |
| Flask-Migrate | 4.0+ | Alembic migrations. |
| SQLAlchemy | 2.0.36+ | ORM. |
| psycopg | 3.2+ | Postgres driver. |
| redis | 5.2+ | Sorted sets, rate limit counters, cache. |
| celery | 5.4+ | Async notification fanout. |
| pydantic | 2.9+ | Request/response validation. |
| boto3 | 1.35+ | S3 / R2 presigned URLs. |
| Pillow | 11.0+ | Image validation (not thumbnailing — Celery does that). |
| gunicorn | 23.0+ | WSGI server. |
| pytest | 8.3+ | Tests. |
| Docker Compose | 2.30+ | Local dev stack. |

> [!info] Why Cloudflare R2 over AWS S3?
> R2 is S3-API-compatible, charges **$0 for egress** (S3 charges ~$0.09/GB), and integrates with Cloudflare's CDN. For a social app where images are downloaded on every feed view, egress is the dominant cost line. The code in this project works identically against either — just set `S3_ENDPOINT_URL` to your R2 endpoint.

---

## 5. Project Structure

```
echo/
├── app/
│   ├── __init__.py            # create_app() factory
│   ├── config.py              # Config classes (Dev, Prod, Test)
│   ├── extensions.py          # db, jwt, limiter, celery, redis, s3
│   ├── models.py              # User, Follow, Post, Like, Comment, Notification, Media, Hashtag
│   ├── schemas.py             # Pydantic v2 request/response models
│   ├── auth.py                # Blueprint: register, login, refresh, logout
│   ├── posts.py               # Blueprint: create, list, get, delete, like, comment
│   ├── feed.py                # Blueprint: timeline (pre-computed feed)
│   ├── social.py              # Blueprint: follow/unfollow, profile, notifications
│   ├── search.py              # Blueprint: users, posts, hashtags
│   ├── media.py               # Blueprint: presigned upload URLs
│   ├── tasks.py               # Celery: fanout_post, fanout_notifications
│   ├── tokens.py              # JWT helpers: issue, revoke, refresh
│   ├── pagination.py          # Cursor encode/decode
│   └── decorators.py          # @require_auth, @tiered_limit
├── migrations/
│   └── versions/
├── tests/
│   ├── conftest.py
│   ├── test_auth.py
│   ├── test_posts.py
│   ├── test_feed.py
│   └── test_social.py
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

```bash
# pyproject.toml (excerpt)
[project]
name = "echo"
requires-python = ">=3.13"
dependencies = [
    "Flask==3.2.0",
    "Flask-SQLAlchemy==3.1.1",
    "Flask-JWT-Extended==4.7.1",
    "Flask-Limiter==3.8.0",
    "Flask-Migrate==4.0.7",
    "SQLAlchemy==2.0.36",
    "psycopg[binary]==3.2.3",
    "redis==5.2.0",
    "celery==5.4.0",
    "pydantic==2.9.2",
    "boto3==1.35.71",
    "Pillow==11.0.0",
    "python-dotenv==1.0.1",
    "gunicorn==23.0.0",
]
[project.optional-dependencies]
dev = ["pytest==8.3.4", "pytest-flask==1.3.0", "httpx==0.27.2"]
```

Install with `uv sync` (or `pip install -e .[dev]`).

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

    # JWT — short access, long refresh
    JWT_SECRET_KEY = os.environ["JWT_SECRET_KEY"]
    JWT_ACCESS_TOKEN_EXPIRES = 60 * 15          # 15 minutes
    JWT_REFRESH_TOKEN_EXPIRES = 60 * 60 * 24 * 30  # 30 days
    JWT_TOKEN_LOCATION = ["headers"]
    JWT_HEADER_NAME = "Authorization"
    JWT_HEADER_TYPE = "Bearer"

    # S3 / R2 — any S3-compatible endpoint
    S3_ENDPOINT_URL = os.environ.get("S3_ENDPOINT_URL")  # None for AWS, set for R2/MinIO
    S3_REGION = os.environ.get("S3_REGION", "auto")
    S3_BUCKET = os.environ["S3_BUCKET"]
    S3_ACCESS_KEY = os.environ["S3_ACCESS_KEY"]
    S3_SECRET_KEY = os.environ["S3_SECRET_KEY"]
    S3_PRESIGN_EXPIRES = 300  # 5 min — enough for a client upload

    # Rate limits per tier (requests per minute)
    RATELIMITS = {
        "free": "60/minute",
        "pro": "600/minute",
    }

    # Feed
    FEED_MAX_SIZE = 1000  # cap each user's pre-computed feed at 1000 items

class TestConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"
    WTF_CSRF_ENABLED = False
    S3_BUCKET = "test-bucket"
    JWT_SECRET_KEY = "test-secret"
    SECRET_KEY = "test-secret"
```

> [!warning] Don't reuse `SECRET_KEY` and `JWT_SECRET_KEY`
> They serve different purposes. `SECRET_KEY` signs Flask sessions and CSRF tokens; `JWT_SECRET_KEY` signs your JWTs. If one leaks, the other still protects its surface area. Generate both with `python -c "import secrets; print(secrets.token_urlsafe(48))"`.

### Step 3 — Extensions + factory

```python
# app/extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_jwt_extended import JWTManager
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from celery import Celery
from redis import Redis
import boto3

db = SQLAlchemy()
jwt = JWTManager()
limiter = Limiter(key_func=get_remote_address, default_limits=["1000/hour"])
celery = Celery("echo")
redis: Redis | None = None
s3_client = None  # lazily built so tests don't need credentials

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
from flask import Flask, jsonify
from app.config import Config
from app.extensions import db, jwt, limiter, celery, redis as _redis, init_s3
from redis import Redis

def create_app(config_class=Config) -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_class)

    db.init_app(app)
    jwt.init_app(app)
    limiter.init_app(app)

    import app.extensions as ext
    ext.redis = Redis.from_url(app.config["REDIS_URL"], decode_responses=True)

    if not app.config.get("TESTING"):
        init_s3(app)

    celery.conf.update(
        broker_url=app.config["CELERY_BROKER_URL"],
        result_backend=app.config["CELERY_RESULT_BACKEND"],
    )

    # JWT blocklist (for logout / refresh-token revocation)
    @jwt.token_in_blocklist_loader
    def _blocked(_jwt_header, jwt_data):
        jti = jwt_data["jti"]
        return bool(ext.redis.get(f"jwt:revoked:{jti}"))

    @jwt.revoked_token_loader
    def _revoked(_h, _d):
        return jsonify(error="token revoked"), 401

    from app.auth import auth_bp
    from app.posts import posts_bp
    from app.feed import feed_bp
    from app.social import social_bp
    from app.search import search_bp
    from app.media import media_bp

    v1 = "/api/v1"
    app.register_blueprint(auth_bp,    url_prefix=f"{v1}/auth")
    app.register_blueprint(posts_bp,   url_prefix=f"{v1}/posts")
    app.register_blueprint(feed_bp,    url_prefix=f"{v1}/feed")
    app.register_blueprint(social_bp,  url_prefix=f"{v1}/social")
    app.register_blueprint(search_bp,  url_prefix=f"{v1}/search")
    app.register_blueprint(media_bp,   url_prefix=f"{v1}/media")

    @app.errorhandler(429)
    def ratelimit_handler(e):
        return jsonify(error="rate_limited", retry_after=e.description), 429

    return app
```

> [!info] Why API versioning from day one?
> `v1` is cheap insurance. When you need to ship a breaking change (rename a field, change pagination), you add `v2` routes alongside `v1`. Clients migrate at their own pace. Without versioning, you're stuck choosing between breaking every client or never improving the API.

### Step 4 — Models

```python
# app/models.py
from datetime import datetime, timezone
from sqlalchemy import (
    Integer, String, DateTime, ForeignKey, Boolean, Text, BigInteger,
    UniqueConstraint, Index,
)
from sqlalchemy.orm import Mapped, mapped_column, relationship
from werkzeug.security import generate_password_hash, check_password_hash
import re
from app.extensions import db

def utcnow() -> datetime:
    return datetime.now(timezone.utc)

FOLLOW_LIMIT = 7_500  # Twitter's old cap; reasonable default

class User(db.Model):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(30), unique=True, index=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    bio: Mapped[str | None] = mapped_column(String(160), nullable=True)
    avatar_key: Mapped[str | None] = mapped_column(String(255), nullable=True)  # S3 key
    tier: Mapped[str] = mapped_column(String(10), default="free")  # "free" | "pro"
    is_verified: Mapped[bool] = mapped_column(default=False)

    # Denormalized counters — updated atomically, never counted live
    posts_count: Mapped[int] = mapped_column(BigInteger, default=0)
    followers_count: Mapped[int] = mapped_column(BigInteger, default=0)
    following_count: Mapped[int] = mapped_column(BigInteger, default=0)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    posts: Mapped[list["Post"]] = relationship(back_populates="author", cascade="all, delete-orphan")

    def set_password(self, raw: str) -> None:
        self.password_hash = generate_password_hash(raw)

    def check_password(self, raw: str) -> bool:
        return check_password_hash(self.password_hash, raw)

class Follow(db.Model):
    __tablename__ = "follows"
    __table_args__ = (
        UniqueConstraint("follower_id", "followed_id", name="uq_follow"),
        Index("ix_follow_followed", "followed_id"),  # for "who follows me?"
    )
    id: Mapped[int] = mapped_column(primary_key=True)
    follower_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    followed_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

class Post(db.Model):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    body: Mapped[str | None] = mapped_column(String(500), nullable=True)  # text posts
    media_key: Mapped[str | None] = mapped_column(String(255), nullable=True)  # image key in S3
    reply_to_id: Mapped[int | None] = mapped_column(ForeignKey("posts.id"), nullable=True, index=True)

    like_count: Mapped[int] = mapped_column(BigInteger, default=0)
    comment_count: Mapped[int] = mapped_column(BigInteger, default=0)
    repost_count: Mapped[int] = mapped_column(BigInteger, default=0)

    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow, index=True)

    author: Mapped[User] = relationship(back_populates="posts")
    hashtags: Mapped[list["Hashtag"]] = relationship(
        secondary="post_hashtags", back_populates="posts"
    )

class Like(db.Model):
    __tablename__ = "likes"
    __table_args__ = (UniqueConstraint("user_id", "post_id", name="uq_like"),)
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    post_id: Mapped[int] = mapped_column(ForeignKey("posts.id"), index=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

class Comment(db.Model):
    __tablename__ = "comments"
    id: Mapped[int] = mapped_column(primary_key=True)
    post_id: Mapped[int] = mapped_column(ForeignKey("posts.id"), index=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    body: Mapped[str] = mapped_column(String(500))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

class Notification(db.Model):
    __tablename__ = "notifications"
    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)  # recipient
    actor_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)  # who did it
    kind: Mapped[str] = mapped_column(String(20))  # "follow" | "like" | "comment" | "mention"
    post_id: Mapped[int | None] = mapped_column(ForeignKey("posts.id"), nullable=True)
    read: Mapped[bool] = mapped_column(default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow, index=True)

class Hashtag(db.Model):
    __tablename__ = "hashtags"
    __table_args__ = (UniqueConstraint("tag", name="uq_tag"),)
    id: Mapped[int] = mapped_column(primary_key=True)
    tag: Mapped[str] = mapped_column(String(60), index=True)  # lowercase, no '#'
    posts: Mapped[list["Post"]] = relationship(
        secondary="post_hashtags", back_populates="hashtags"
    )

# Association table for Post ↔ Hashtag (many-to-many)
post_hashtags = db.Table(
    "post_hashtags",
    db.Column("post_id", ForeignKey("posts.id"), primary_key=True),
    db.Column("hashtag_id", ForeignKey("hashtags.id"), primary_key=True),
    db.Column("created_at", DateTime(timezone=True), default=utcnow),
)
```

> [!tip] Why denormalized counters?
> `SELECT COUNT(*) FROM likes WHERE post_id = 123` is fast on a small table and brutal on a billion-row one — Postgres has to scan an index. Storing `like_count` on the `Post` row turns the read into a single integer lookup. The write is two operations (insert the Like row, increment the counter), but those happen in the same transaction so they stay consistent. The same trick powers `followers_count`, `comment_count`, and `repost_count`.

### Step 5 — ER diagram

```mermaid
erDiagram
    USER ||--o{ POST : writes
    USER ||--o{ FOLLOW : follower
    USER ||--o{ FOLLOW : followed
    USER ||--o{ LIKE : gives
    USER ||--o{ COMMENT : writes
    USER ||--o{ NOTIFICATION : receives
    USER ||--o{ NOTIFICATION : actor
    POST ||--o{ LIKE : receives
    POST ||--o{ COMMENT : receives
    POST ||--o{ NOTIFICATION : references
    POST ||--o{ POST_HASHTAG : tagged_in
    HASHTAG ||--o{ POST_HASHTAG : appears_in
    USER {
        int id PK
        string username UK
        string email UK
        string password_hash
        string tier
        bigint posts_count
        bigint followers_count
        bigint following_count
    }
    POST {
        int id PK
        int author_id FK
        string body
        string media_key
        bigint like_count
        bigint comment_count
        datetime created_at
    }
    FOLLOW {
        int id PK
        int follower_id FK
        int followed_id FK
        datetime created_at
    }
    LIKE {
        int id PK
        int user_id FK
        int post_id FK
    }
    COMMENT {
        int id PK
        int post_id FK
        int author_id FK
        string body
    }
    NOTIFICATION {
        bigint id PK
        int user_id FK
        int actor_id FK
        string kind
        int post_id FK
        bool read
    }
    HASHTAG {
        int id PK
        string tag UK
    }
```

### Step 6 — Pydantic v2 schemas

```python
# app/schemas.py
from datetime import datetime
from pydantic import BaseModel, Field, ConfigDict, field_validator
import re

USERNAME_RE = re.compile(r"^[a-z0-9_]{3,30}$", re.IGNORECASE)

class UserCreate(BaseModel):
    username: str = Field(min_length=3, max_length=30)
    email: str = Field(..., pattern=r"^[\w.+-]+@[\w-]+\.[\w.-]+$")
    password: str = Field(min_length=8, max_length=128)

    @field_validator("username")
    @classmethod
    def _username(cls, v: str) -> str:
        if not USERNAME_RE.match(v):
            raise ValueError("username must be 3-30 chars: letters, digits, underscore")
        return v.lower()

class UserLogin(BaseModel):
    email: str
    password: str

class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    username: str
    bio: str | None = None
    avatar_url: str | None = None
    is_verified: bool
    posts_count: int
    followers_count: int
    following_count: int
    created_at: datetime

class PostCreate(BaseModel):
    body: str | None = Field(default=None, max_length=500)
    media_key: str | None = Field(default=None, max_length=255)
    reply_to_id: int | None = None

    @field_validator("body", "media_key")
    @classmethod
    def _at_least_one(cls, v, info):
        # Custom root validation done in route — Pydantic v2 doesn't have
        # easy "at least one of two fields" without model_validator.
        return v

class PostOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    author: UserOut
    body: str | None
    media_url: str | None
    like_count: int
    comment_count: int
    repost_count: int
    liked_by_me: bool = False
    created_at: datetime

class CommentCreate(BaseModel):
    body: str = Field(min_length=1, max_length=500)

class CommentOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    author: UserOut
    body: str
    created_at: datetime

class NotificationOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    actor: UserOut
    kind: str
    post_id: int | None
    read: bool
    created_at: datetime

class PaginatedPosts(BaseModel):
    data: list[PostOut]
    next_cursor: str | None = None
```

> [!info] Why Pydantic v2 for the API layer?
> Three reasons. (1) **Validation lives in one place** — the schema is the spec. (2) `model_config = ConfigDict(from_attributes=True)` lets you build a response from a SQLAlchemy object directly (`PostOut.model_validate(post)`), no manual `dict` plumbing. (3) Pydantic v2 is Rust-backed — validation is ~10× faster than v1, which matters when you're parsing 1k requests/sec.

### Step 7 — JWT auth: register, login, refresh, logout

```python
# app/tokens.py
from flask_jwt_extended import create_access_token, create_refresh_token, get_jti
from app.extensions import redis

def issue_tokens(user_id: int) -> dict:
    access = create_access_token(identity=str(user_id))
    refresh = create_refresh_token(identity=str(user_id))
    # Track refresh jti so we can detect reuse (refresh-token rotation)
    refresh_jti = get_jti(refresh)
    redis.setex(f"jwt:refresh:{user_id}:{refresh_jti}", 60 * 60 * 24 * 30, "valid")
    return {"access_token": access, "refresh_token": refresh}

def revoke_access(jti: str, exp: int) -> None:
    # Store until the token would have expired, then auto-cleanup
    ttl = max(exp - int(__import__("time").time()), 1)
    redis.setex(f"jwt:revoked:{jti}", ttl, "1")

def revoke_refresh(user_id: int, jti: str) -> None:
    redis.delete(f"jwt:refresh:{user_id}:{jti}")
```

```python
# app/auth.py
from flask import Blueprint, request, jsonify, current_app
from flask_jwt_extended import (
    jwt_required, get_jwt, get_jwt_identity,
    create_access_token, get_jti,
)
from pydantic import ValidationError
from sqlalchemy import select
from app.extensions import db, redis
from app.models import User
from app.schemas import UserCreate, UserLogin, UserOut
from app.tokens import issue_tokens, revoke_access, revoke_refresh

auth_bp = Blueprint("auth", __name__)

@auth_bp.post("/register")
def register():
    try:
        data = UserCreate.model_validate(request.get_json())
    except ValidationError as e:
        return jsonify(error="validation_error", details=e.errors()), 422

    if db.session.scalar(select(User).where(User.email == data.email)):
        return jsonify(error="email_taken"), 409
    if db.session.scalar(select(User).where(User.username == data.username)):
        return jsonify(error="username_taken"), 409

    user = User(username=data.username, email=data.email)
    user.set_password(data.password)
    db.session.add(user)
    db.session.commit()
    return jsonify(user=UserOut.model_validate(user).model_dump(),
                   **issue_tokens(user.id)), 201

@auth_bp.post("/login")
def login():
    try:
        data = UserLogin.model_validate(request.get_json())
    except ValidationError as e:
        return jsonify(error="validation_error", details=e.errors()), 422

    user = db.session.scalar(select(User).where(User.email == data.email))
    if not user or not user.check_password(data.password):
        return jsonify(error="invalid_credentials"), 401

    return jsonify(user=UserOut.model_validate(user).model_dump(),
                   **issue_tokens(user.id)), 200

@auth_bp.post("/refresh")
@jwt_required(refresh=True)
def refresh():
    user_id = int(get_jwt_identity())
    jti = get_jwt()["jti"]
    # Refresh-token rotation: old refresh must be valid AND then we revoke it
    if not redis.get(f"jwt:refresh:{user_id}:{jti}"):
        return jsonify(error="refresh_reused"), 401  # possible token theft!
    revoke_refresh(user_id, jti)
    return jsonify(**issue_tokens(user_id)), 200

@auth_bp.delete("/logout")
@jwt_required()
def logout():
    user_id = int(get_jwt_identity())
    jti = get_jwt()["jti"]
    revoke_access(jti, get_jwt()["exp"])
    # Also revoke the caller's refresh if they sent it
    return jsonify(ok=True), 200
```

> [!warning] Refresh-token rotation detects theft
> When a refresh token is used, we immediately revoke it and issue a new one. If an attacker stole the original token and tries to reuse it *after* the legitimate user refreshed, the lookup `jwt:refresh:{user_id}:{jti}` misses — a strong signal of token theft. Production systems trigger a "log out all sessions" cascade at this point. We just return 401 here for clarity.

### Step 8 — Decorators: `@require_auth` and `@tiered_limit`

```python
# app/decorators.py
from functools import wraps
from flask import jsonify, g
from flask_jwt_extended import verify_jwt_in_request, get_jwt_identity
from sqlalchemy import select
from app.extensions import db, limiter
from app.models import User

def require_auth(fn):
    """Like @jwt_required but also loads the User into g.current_user."""
    @wraps(fn)
    def wrapper(*args, **kwargs):
        verify_jwt_in_request()
        user_id = int(get_jwt_identity())
        user = db.session.get(User, user_id)
        if not user:
            return jsonify(error="user_not_found"), 401
        g.current_user = user
        return fn(*args, **kwargs)
    return wrapper

def tiered_limit(action: str):
    """Rate limit per user, with the limit picked by the user's tier."""
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            from flask import g, current_app
            # Use the authenticated user's id as the key (falls back to IP)
            try:
                user_id = g.current_user.id
                key = f"user:{user_id}:{action}"
            except AttributeError:
                key = f"ip:{get_remote_address()}:{action}"
            user = getattr(g, "current_user", None)
            tier = user.tier if user else "free"
            limit = current_app.config["RATELIMITS"].get(tier, "60/minute")
            # We use limiter.limit dynamically with the resolved key
            decorator_ = limiter.limit(limit, key_func=lambda: key)
            return decorator_(fn)(*args, **kwargs)
        return wrapper
    return decorator
```

> [!info] Per-user vs per-IP rate limits
> Per-IP is the only thing you can do for anonymous endpoints (login, register). For authenticated endpoints, per-user is correct — a school's NAT may have 1,000 students behind one IP. Flask-Limiter's `key_func` lets you switch the key based on whether the request is authenticated.

### Step 9 — Cursor-based pagination

```python
# app/pagination.py
import base64, json
from datetime import datetime

def encode_cursor(post_id: int, created_at: datetime) -> str:
    """Opaque cursor = base64(JSON with id + timestamp)."""
    raw = json.dumps({"id": post_id, "ts": created_at.isoformat()}).encode()
    return base64.urlsafe_b64encode(raw).decode()

def decode_cursor(cursor: str) -> dict:
    try:
        return json.loads(base64.urlsafe_b64decode(cursor.encode()))
    except Exception:
        raise ValueError("invalid cursor")

def build_page(items, next_id, next_ts):
    """Build a paginated response. `items` already serialized to dicts."""
    return {
        "data": items,
        "next_cursor": encode_cursor(next_id, next_ts) if items else None,
    }
```

Why cursors beat `?page=5`:

```mermaid
flowchart LR
    A["Page 1<br/>posts 1–20"] --> B{new post arrives?}
    B -->|Yes — offset shifts| C["Page 2<br/>shows post 20 AGAIN"]
    B -->|No| D["Page 2<br/>posts 21–40"]
    E["Cursor 1<br/>(id=20, ts=12:00)"] --> F{new post arrives?}
    F -->|Yes| G["Cursor 2<br/>WHERE id < 20"]
    F -->|No| G
    G --> H["Page 2<br/>posts 21–40 (always)"]
    style C fill:#fecaca,stroke:#dc2626
    style H fill:#dcfce7,stroke:#16a34a
```

With offset pagination, a new post shifts everything right — the user sees a duplicate. With cursor pagination (`WHERE (created_at, id) < (cursor_ts, cursor_id) ORDER BY created_at DESC, id DESC`), the boundary is absolute: new posts just don't appear in the *current* browsing session, which is what users want.

### Step 10 — Posts blueprint (create, get, delete, like, comment)

```python
# app/posts.py
import re
from flask import Blueprint, request, jsonify, g, current_app
from sqlalchemy import select, desc, func, and_
from pydantic import ValidationError
from app.extensions import db, redis
from app.models import Post, User, Like, Comment, Hashtag, post_hashtags
from app.schemas import PostCreate, PostOut, CommentCreate, CommentOut
from app.decorators import require_auth, tiered_limit
from app.pagination import decode_cursor, build_page
from app.tasks import fanout_post_created, fanout_notifications

posts_bp = Blueprint("posts", __name__)
HASHTAG_RE = re.compile(r"#([a-z0-9_]{1,60})", re.IGNORECASE)

def _post_to_out(post: Post, viewer: User | None) -> dict:
    out = PostOut.model_validate(post).model_dump(mode="json")
    if viewer:
        out["liked_by_me"] = bool(db.session.scalar(
            select(Like.id).where(Like.user_id == viewer.id, Like.post_id == post.id)
        ))
    return out

@posts_bp.post("")
@require_auth
@tiered_limit("create_post")
def create_post():
    try:
        data = PostCreate.model_validate(request.get_json())
    except ValidationError as e:
        return jsonify(error="validation_error", details=e.errors()), 422

    if not data.body and not data.media_key:
        return jsonify(error="post_must_have_body_or_media"), 422

    post = Post(
        author_id=g.current_user.id,
        body=data.body,
        media_key=data.media_key,
        reply_to_id=data.reply_to_id,
    )
    db.session.add(post)
    db.session.flush()  # get post.id without committing

    # Extract hashtags, upsert them, link to the post
    if data.body:
        tags = {t.lower() for t in HASHTAG_RE.findall(data.body)}
        for tag in tags:
            h = db.session.scalar(select(Hashtag).where(Hashtag.tag == tag))
            if not h:
                h = Hashtag(tag=tag)
                db.session.add(h)
                db.session.flush()
            db.session.execute(
                post_hashtags.insert().values(post_id=post.id, hashtag_id=h.id)
            )

    # Bump author's post count atomically
    db.session.query(User).filter_by(id=g.current_user.id).update(
        {"posts_count": User.posts_count + 1}
    )
    db.session.commit()

    # Async: push post_id into each follower's feed (sorted set)
    fanout_post_created.delay(post.id)
    return jsonify(_post_to_out(post, g.current_user)), 201

@posts_bp.get("/<int:post_id>")
@require_auth
def get_post(post_id: int):
    post = db.session.get(Post, post_id)
    if not post:
        return jsonify(error="not_found"), 404
    return jsonify(_post_to_out(post, g.current_user))

@posts_bp.delete("/<int:post_id>")
@require_auth
def delete_post(post_id: int):
    post = db.session.get(Post, post_id)
    if not post or post.author_id != g.current_user.id:
        return jsonify(error="not_found"), 404
    db.session.delete(post)
    db.session.query(User).filter_by(id=g.current_user.id).update(
        {"posts_count": User.posts_count - 1}
    )
    db.session.commit()
    # Remove from every cached feed
    redis.delete(*[f"feed:{uid}" for uid in _follower_ids(g.current_user.id)])
    return "", 204

@posts_bp.post("/<int:post_id>/like")
@require_auth
@tiered_limit("like")
def like_post(post_id: int):
    post = db.session.get(Post, post_id)
    if not post:
        return jsonify(error="not_found"), 404
    existing = db.session.scalar(
        select(Like).where(Like.user_id == g.current_user.id, Like.post_id == post_id)
    )
    if existing:
        return jsonify(ok=True, liked=True), 200  # idempotent
    like = Like(user_id=g.current_user.id, post_id=post_id)
    db.session.add(like)
    db.session.query(Post).filter_by(id=post_id).update(
        {"like_count": Post.like_count + 1}
    )
    db.session.commit()
    # Async: notify the post's author (skip if liking own post)
    if post.author_id != g.current_user.id:
        fanout_notifications.delay(
            actor_id=g.current_user.id,
            recipient_id=post.author_id,
            kind="like",
            post_id=post_id,
        )
    return jsonify(ok=True, liked=True), 201

@posts_bp.delete("/<int:post_id>/like")
@require_auth
def unlike_post(post_id: int):
    like = db.session.scalar(
        select(Like).where(Like.user_id == g.current_user.id, Like.post_id == post_id)
    )
    if like:
        db.session.delete(like)
        db.session.query(Post).filter_by(id=post_id).update(
            {"like_count": Post.like_count - 1}
        )
        db.session.commit()
    return jsonify(ok=True, liked=False), 200

@posts_bp.post("/<int:post_id>/comments")
@require_auth
@tiered_limit("comment")
def create_comment(post_id: int):
    post = db.session.get(Post, post_id)
    if not post:
        return jsonify(error="not_found"), 404
    try:
        data = CommentCreate.model_validate(request.get_json())
    except ValidationError as e:
        return jsonify(error="validation_error", details=e.errors()), 422

    comment = Comment(post_id=post_id, author_id=g.current_user.id, body=data.body)
    db.session.add(comment)
    db.session.query(Post).filter_by(id=post_id).update(
        {"comment_count": Post.comment_count + 1}
    )
    db.session.commit()

    if post.author_id != g.current_user.id:
        fanout_notifications.delay(
            actor_id=g.current_user.id,
            recipient_id=post.author_id,
            kind="comment",
            post_id=post_id,
        )
    return jsonify(CommentOut.model_validate(comment).model_dump(mode="json")), 201

def _follower_ids(user_id: int) -> list[int]:
    rows = db.session.execute(
        select(Follow.follower_id).where(Follow.followed_id == user_id)
    ).scalars().all()
    return rows
```

### Step 11 — The feed blueprint (fanout-on-write, the heart of the project)

```python
# app/feed.py
import time
from flask import Blueprint, request, jsonify, g, current_app
from sqlalchemy import select
from app.extensions import db, redis
from app.models import Post, User
from app.decorators import require_auth
from app.pagination import decode_cursor, build_page
from app.posts import _post_to_out

feed_bp = Blueprint("feed", __name__)

@feed_bp.get("")
@require_auth
def timeline():
    """Pre-computed feed: read post IDs from Redis sorted set."""
    user = g.current_user
    cursor = request.args.get("cursor")
    limit = min(int(request.args.get("limit", 20)), 50)

    feed_key = f"feed:{user.id}"

    if cursor:
        try:
            c = decode_cursor(cursor)
            # ZREVRANGEBYSCORE returns items with score <= cursor_ts
            # (we use post_id as the score, since ids are monotonic)
            post_ids = redis.zrevrangebyscore(
                feed_key, int(c["id"]), -1, start=0, num=limit
            )
        except ValueError:
            return jsonify(error="invalid_cursor"), 400
    else:
        post_ids = redis.zrevrange(feed_key, 0, limit - 1)

    if not post_ids:
        return jsonify({"data": [], "next_cursor": None})

    # Hydrate: fetch posts + authors in two queries (no N+1)
    posts = db.session.scalars(
        select(Post)
        .where(Post.id.in_([int(pid) for pid in post_ids]))
        .order_by(Post.id.desc())
    ).all()
    # Map by id so we can sort by Redis order
    by_id = {p.id: p for p in posts}
    ordered = [by_id[int(pid)] for pid in post_ids if int(pid) in by_id]

    items = [_post_to_out(p, user) for p in ordered]
    last_id = int(post_ids[-1])
    return jsonify(build_page(items, last_id, None))

@feed_bp.get("/mine")
@require_auth
def my_posts():
    """The authenticated user's own posts (fanout-on-read for self)."""
    user = g.current_user
    cursor = request.args.get("cursor")
    limit = min(int(request.args.get("limit", 20)), 50)

    q = select(Post).where(Post.author_id == user.id).order_by(Post.id.desc())
    if cursor:
        try:
            c = decode_cursor(cursor)
            q = q.where(Post.id < int(c["id"]))
        except ValueError:
            return jsonify(error="invalid_cursor"), 400
    q = q.limit(limit)
    posts = db.session.scalars(q).all()
    items = [_post_to_out(p, user) for p in posts]
    next_cursor = items[-1]["id"] if len(items) == limit and items else None
    return jsonify({"data": items, "next_cursor": next_cursor})
```

> [!info] Why use `post_id` as the score, not `created_at`?
> Postgres `id` columns are monotonic — every new post has a higher ID than the previous one. That gives us a strict ordering by creation time *for free*. Using `created_at` as the score works too, but two posts created within the same millisecond would tie, and Redis would break ties by lex order of the member (the post ID, as a string — which sorts `"10"` before `"2"`). Integer IDs dodge this entirely.

### Step 12 — Celery tasks: feed fanout + notification fanout

```python
# app/tasks.py
from celery import shared_task, current_task
from sqlalchemy import select
from app.extensions import db, redis
from app.models import Follow, Post, Notification, User

FEED_MAX_SIZE = 1000

@shared_task
def fanout_post_created(post_id: int) -> None:
    """Push post_id into every follower's feed (sorted set)."""
    # Find the post (it must be committed before this task runs)
    post = db.session.get(Post, post_id)
    if not post:
        return

    # Find all follower IDs
    follower_ids = db.session.execute(
        select(Follow.follower_id).where(Follow.followed_id == post.author_id)
    ).scalars().all()

    if not follower_ids:
        return

    # Hybrid: celebrities (>10k followers) skip fanout-on-write.
    # Their posts are merged at read time. (See "celebrity problem" below.)
    author_followers = len(follower_ids)
    if author_followers > 10_000:
        # Mark this post for fanout-on-read instead
        redis.zadd(f"celebrity_posts:{post.author_id}", {str(post_id): post_id})
        return

    # Batch ZADD: {member: score, ...}
    pipe = redis.pipeline()
    for fid in follower_ids:
        key = f"feed:{fid}"
        pipe.zadd(key, {str(post_id): post_id})
        pipe.zremrangebyrank(key, 0, -FEED_MAX_SIZE - 1)  # cap at 1000 newest
    pipe.execute()

@shared_task
def fanout_notifications(actor_id: int, recipient_id: int, kind: str,
                          post_id: int | None = None) -> None:
    """Insert a notification row. Bulk-fanout for likes on big posts
    would batch many recipients — for simplicity we do one at a time."""
    if actor_id == recipient_id:
        return  # never notify self
    n = Notification(user_id=recipient_id, actor_id=actor_id,
                     kind=kind, post_id=post_id)
    db.session.add(n)
    db.session.commit()
    # Increment unread counter in Redis for fast badge reads
    redis.incr(f"notifications:unread:{recipient_id}")
```

> [!warning] The celebrity problem
> Lady Gaga has 80M followers. Fanout-on-write for her would push each post into 80M Redis sorted sets. That's 80M writes per tweet. Twitter's solution: celebrities get **fanout-on-read** — their posts are *not* pushed; instead, the timeline query merges their posts in at read time. The hybrid threshold (10k followers in our code) is what makes the system work at scale. Pure fanout-on-write is fine for 99% of users; the top 1% need a different path.

### Step 13 — Fanout-on-write sequence

```mermaid
sequenceDiagram
    participant U as User A (author)
    participant F as Flask
    participant PG as PostgreSQL
    participant Q as Celery worker
    participant R as Redis
    participant B as Follower B's feed request

    U->>F: POST /api/v1/posts (body="hello world")
    F->>F: Pydantic validate
    F->>PG: INSERT INTO posts (RETURNING id=42)
    F->>PG: UPDATE users SET posts_count = posts_count + 1
    F->>PG: COMMIT
    F->>Q: fanout_post_created.delay(42)
    F-->>U: 201 Created (post object)

    Note over Q: Async fanout (out of request cycle)
    Q->>PG: SELECT follower_id FROM follows WHERE followed_id = A
    PG-->>Q: [B, C, D, E, F, ...]
    Q->>R: ZADD feed:B {42: 42}
    Q->>R: ZADD feed:C {42: 42}
    Q->>R: ZADD feed:D {42: 42}
    Q->>R: ZREMRANGEBYRANK feed:B 0 -1001 (cap at 1000)

    Note over B: 200 ms later
    B->>F: GET /api/v1/feed
    F->>R: ZREVRANGE feed:B 0 19
    R-->>F: ["42", "41", "39", ...]
    F->>PG: SELECT * FROM posts WHERE id IN (42, 41, 39, ...)
    F->>PG: SELECT * FROM users WHERE id IN (...)
    F-->>B: 200 OK (20 posts, next_cursor=...)
```

### Step 14 — Social blueprint: follow, profile, notifications

```python
# app/social.py
from flask import Blueprint, request, jsonify, g
from sqlalchemy import select, func
from pydantic import ValidationError
from app.extensions import db, redis
from app.models import User, Follow, Notification, Post
from app.schemas import UserOut, NotificationOut
from app.decorators import require_auth, tiered_limit
from app.tasks import fanout_notifications
from app.posts import _post_to_out

social_bp = Blueprint("social", __name__)

@social_bp.post("/follow/<username>")
@require_auth
@tiered_limit("follow")
def follow(username: str):
    target = db.session.scalar(select(User).where(User.username == username))
    if not target:
        return jsonify(error="user_not_found"), 404
    if target.id == g.current_user.id:
        return jsonify(error="cannot_follow_self"), 400

    existing = db.session.scalar(
        select(Follow).where(Follow.follower_id == g.current_user.id,
                             Follow.followed_id == target.id)
    )
    if existing:
        return jsonify(ok=True, following=True), 200  # idempotent

    f = Follow(follower_id=g.current_user.id, followed_id=target.id)
    db.session.add(f)
    # Update denormalized counters atomically
    db.session.query(User).filter_by(id=g.current_user.id).update(
        {"following_count": User.following_count + 1}
    )
    db.session.query(User).filter_by(id=target.id).update(
        {"followers_count": User.followers_count + 1}
    )
    db.session.commit()

    fanout_notifications.delay(
        actor_id=g.current_user.id, recipient_id=target.id, kind="follow"
    )
    return jsonify(ok=True, following=True), 201

@social_bp.delete("/follow/<username>")
@require_auth
def unfollow(username: str):
    target = db.session.scalar(select(User).where(User.username == username))
    if not target:
        return jsonify(error="user_not_found"), 404
    f = db.session.scalar(
        select(Follow).where(Follow.follower_id == g.current_user.id,
                             Follow.followed_id == target.id)
    )
    if f:
        db.session.delete(f)
        db.session.query(User).filter_by(id=g.current_user.id).update(
            {"following_count": User.following_count - 1}
        )
        db.session.query(User).filter_by(id=target.id).update(
            {"followers_count": User.followers_count - 1}
        )
        db.session.commit()
    return jsonify(ok=True, following=False), 200

@social_bp.get("/profile/<username>")
@require_auth
def profile(username: str):
    target = db.session.scalar(select(User).where(User.username == username))
    if not target:
        return jsonify(error="user_not_found"), 404
    out = UserOut.model_validate(target).model_dump(mode="json")
    # Is the current user following this person?
    out["following"] = bool(db.session.scalar(
        select(Follow.id).where(Follow.follower_id == g.current_user.id,
                                Follow.followed_id == target.id)
    ))
    return jsonify(out)

@social_bp.get("/notifications")
@require_auth
def notifications():
    rows = db.session.scalars(
        select(Notification)
        .where(Notification.user_id == g.current_user.id)
        .order_by(Notification.id.desc())
        .limit(50)
    ).all()
    # Hydrate actor users in one query
    actor_ids = {n.actor_id for n in rows}
    actors = {u.id: u for u in db.session.scalars(
        select(User).where(User.id.in_(actor_ids))
    ).all()} if actor_ids else {}
    items = []
    for n in rows:
        n_dict = NotificationOut.model_validate({
            "id": n.id, "kind": n.kind, "post_id": n.post_id,
            "read": n.read, "created_at": n.created_at,
            "actor": UserOut.model_validate(actors[n.actor_id]).model_dump(mode="json"),
        }).model_dump(mode="json")
        items.append(n_dict)
    # Mark all as read
    db.session.query(Notification).filter_by(
        user_id=g.current_user.id, read=False
    ).update({"read": True})
    db.session.commit()
    redis.delete(f"notifications:unread:{g.current_user.id}")
    return jsonify({"data": items})

@social_bp.get("/notifications/unread_count")
@require_auth
def unread_count():
    n = redis.get(f"notifications:unread:{g.current_user.id}")
    return jsonify(count=int(n or 0))
```

### Step 15 — Media uploads with pre-signed S3 / R2 URLs

```python
# app/media.py
import uuid, time
from flask import Blueprint, request, jsonify, g, current_app
from app.extensions import s3_client
from app.decorators import require_auth, tiered_limit

media_bp = Blueprint("media", __name__)
ALLOWED = {"image/jpeg", "image/png", "image/webp", "image/gif"}

@media_bp.post("/upload-url")
@require_auth
@tiered_limit("upload")
def create_upload_url():
    """Return a pre-signed PUT URL the client uses to upload directly to S3/R2."""
    body = request.get_json() or {}
    content_type = body.get("content_type", "")
    if content_type not in ALLOWED:
        return jsonify(error="unsupported_media_type",
                       allowed=list(ALLOWED)), 422

    ext = {"image/jpeg": "jpg", "image/png": "png",
           "image/webp": "webp", "image/gif": "gif"}[content_type]
    key = f"uploads/{g.current_user.id}/{int(time.time())}-{uuid.uuid4().hex}.{ext}"

    url = s3_client.generate_presigned_url(
        "put_object",
        Params={"Bucket": current_app.config["S3_BUCKET"],
                "Key": key,
                "ContentType": content_type},
        ExpiresIn=current_app.config["S3_PRESIGN_EXPIRES"],
    )
    # Read URL — what the client will use to display the image later
    read_url = s3_client.generate_presigned_url(
        "get_object",
        Params={"Bucket": current_app.config["S3_BUCKET"], "Key": key},
        ExpiresIn=3600 * 24 * 7,  # 7 days — or use a public bucket + CDN
    )
    return jsonify(key=key, upload_url=url, read_url=read_url,
                   expires_in=current_app.config["S3_PRESIGN_EXPIRES"]), 201
```

To create a post with an image, the client does this in **three steps**:

```mermaid
sequenceDiagram
    participant C as Client
    participant F as Flask API
    participant S3 as S3 / R2

    C->>F: POST /api/v1/media/upload-url<br/>{content_type: "image/png"}
    F-->>C: 201 {key, upload_url, read_url}
    C->>S3: PUT upload_url<br/>(raw bytes, Content-Type: image/png)
    S3-->>C: 200 OK
    C->>F: POST /api/v1/posts<br/>{body: "look at this", media_key: "uploads/1/..."}
    F->>F: store Post with media_key
    F-->>C: 201 Created
```

> [!info] Why presigned URLs and not multipart uploads through Flask?
> Two reasons. (1) **Throughput.** Image bytes don't traverse your Flask servers. Your app server has 4 workers; if each handles a 5 MB upload at 1 MB/s, that's 4 concurrent uploads maxing out your fleet. Presigned URLs let the client hit S3 directly with no Flask bottleneck. (2) **Cost.** S3 ingress is free; Flask servers cost CPU time per byte. For a social app where every post may have an image, this is the difference between $50/month and $500/month in compute.

### Step 16 — Search blueprint

```python
# app/search.py
from flask import Blueprint, request, jsonify, g
from sqlalchemy import select, or_, func
from app.extensions import db
from app.models import User, Post, Hashtag, post_hashtags
from app.schemas import UserOut, PostOut
from app.decorators import require_auth
from app.posts import _post_to_out

search_bp = Blueprint("search", __name__)

@search_bp.get("/users")
@require_auth
def search_users():
    q = request.args.get("q", "").strip().lower()
    if len(q) < 2:
        return jsonify(error="query_too_short"), 400
    users = db.session.scalars(
        select(User).where(
            or_(func.lower(User.username).like(f"{q}%"),
                func.lower(User.username).like(f"%{q}%"))
        ).limit(20)
    ).all()
    return jsonify({"data": [UserOut.model_validate(u).model_dump(mode="json")
                             for u in users]})

@search_bp.get("/posts")
@require_auth
def search_posts():
    q = request.args.get("q", "").strip()
    if len(q) < 2:
        return jsonify(error="query_too_short"), 400
    # Basic ILIKE — production would use Postgres pg_trgm or Elasticsearch
    posts = db.session.scalars(
        select(Post).where(Post.body.ilike(f"%{q}%"))
        .order_by(Post.id.desc()).limit(20)
    ).all()
    return jsonify({"data": [_post_to_out(p, g.current_user) for p in posts]})

@search_bp.get("/hashtags/<tag>")
@require_auth
def posts_for_hashtag(tag: str):
    tag = tag.lower().lstrip("#")
    h = db.session.scalar(select(Hashtag).where(Hashtag.tag == tag))
    if not h:
        return jsonify({"data": []})
    posts = db.session.scalars(
        select(Post)
        .join(post_hashtags, post_hashtags.c.post_id == Post.id)
        .where(post_hashtags.c.hashtag_id == h.id)
        .order_by(Post.id.desc()).limit(50)
    ).all()
    return jsonify({"data": [_post_to_out(p, g.current_user) for p in posts]})
```

> [!tip] Production search is bigger than `ILIKE`
> `ILIKE '%query%'` works at 100k posts. At 10M, switch to **Postgres `pg_trgm`** (trigram index — fast fuzzy matching) or **Elasticsearch / Meilisearch** (full-text with ranking, stemming, synonyms). Hashtag search is fine on `ILIKE` because hashtags are short and discrete.

---

## 7. Running the Project

### `.env.example`

```bash
SECRET_KEY=change-me
JWT_SECRET_KEY=change-me-too
DATABASE_URL=postgresql+psycopg://echo:echopass@postgres:5432/echo
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=redis://redis:6379/1
CELERY_RESULT_BACKEND=redis://redis:6379/2

# Cloudflare R2 example (S3-compatible)
S3_ENDPOINT_URL=https://<account>.r2.cloudflarestorage.com
S3_REGION=auto
S3_BUCKET=echo-media
S3_ACCESS_KEY=<r2-key-id>
S3_SECRET_KEY=<r2-secret>
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
    command: celery -A celery_worker.celery worker --loglevel=info --concurrency=4
    env_file: .env
    depends_on: [redis, postgres]
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: echo
      POSTGRES_USER: echo
      POSTGRES_PASSWORD: echopass
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
    volumes: ["redisdata:/data"]
volumes: { pgdata: {}, redisdata: {} }
```

### `wsgi.py` and `celery_worker.py`

```python
# wsgi.py
from app import create_app
app = create_app()
```

```python
# celery_worker.py
from app import create_app
from app.extensions import celery

app = create_app()
celery_app = celery
# Import tasks so Celery discovers them
import app.tasks  # noqa: F401
```

### First run

```bash
cp .env.example .env
# Edit .env — fill in your R2 / S3 credentials
docker compose up --build -d postgres redis
docker compose run --rm web flask --app wsgi db upgrade
docker compose up --build -d web worker

# Register a user:
curl -X POST http://localhost:5000/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","email":"alice@example.com","password":"hunter2hunter2"}'

# Response includes access_token + refresh_token. Save them:
export ACCESS="..."
# Create a post:
curl -X POST http://localhost:5000/api/v1/posts \
  -H "Authorization: Bearer $ACCESS" \
  -H 'Content-Type: application/json' \
  -d '{"body":"first post #hello"}'

# Read the feed:
curl http://localhost:5000/api/v1/feed -H "Authorization: Bearer $ACCESS"
```

---

## 8. Testing

```python
# tests/conftest.py
import pytest
from app import create_app
from app.config import TestConfig
from app.extensions import db as _db

@pytest.fixture
def app():
    app = create_app(TestConfig)
    with app.app_context():
        _db.create_all()
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
# tests/test_auth.py
def test_register_and_login(client):
    r = client.post("/api/v1/auth/register", json={
        "username": "alice", "email": "alice@example.com",
        "password": "hunter2hunter2"})
    assert r.status_code == 201
    assert "access_token" in r.get_json()

    r = client.post("/api/v1/auth/login", json={
        "email": "alice@example.com", "password": "hunter2hunter2"})
    assert r.status_code == 200

def test_register_rejects_bad_username(client):
    r = client.post("/api/v1/auth/register", json={
        "username": "a!", "email": "a@b.com", "password": "x" * 12})
    assert r.status_code == 422

def test_refresh_rotates_token(client):
    r = client.post("/api/v1/auth/register", json={
        "username": "alice", "email": "a@b.com", "password": "x" * 12})
    refresh = r.get_json()["refresh_token"]
    r1 = client.post("/api/v1/auth/refresh",
                     headers={"Authorization": f"Bearer {refresh}"})
    assert r1.status_code == 200
    # Old refresh should now be revoked
    r2 = client.post("/api/v1/auth/refresh",
                     headers={"Authorization": f"Bearer {refresh}"})
    assert r2.status_code == 401
```

```python
# tests/test_feed.py
def test_feed_picks_up_followed_post(client, app):
    from app.extensions import db
    from app.models import User, Follow, Post
    with app.app_context():
        a = User(username="alice", email="a@x.com"); a.set_password("x"*12)
        b = User(username="bob", email="b@x.com");   b.set_password("x"*12)
        db.session.add_all([a, b]); db.session.commit()
        db.session.add(Follow(follower_id=a.id, followed_id=b.id))
        db.session.commit()

        # Alice logs in, creates a post (no — bob posts, alice's feed should show it)
        post = Post(author_id=b.id, body="hello from bob")
        db.session.add(post); db.session.commit()

        from app.tasks import fanout_post_created
        fanout_post_created.apply(args=[post.id])  # run synchronously in tests

    # Alice logs in
    r = client.post("/api/v1/auth/login", json={
        "email": "a@x.com", "password": "x"*12})
    access = r.get_json()["access_token"]
    r = client.get("/api/v1/feed",
                   headers={"Authorization": f"Bearer {access}"})
    assert r.status_code == 200
    data = r.get_json()["data"]
    assert len(data) == 1
    assert data[0]["body"] == "hello from bob"
```

> [!warning] Celery tasks in tests
> Use `task.apply(args=[...])` (synchronous) instead of `task.delay(...)` (async via broker). The task body runs in-process, raises any exception to your test, and skips the Redis broker entirely. For end-to-end tests, set `task_always_eager = True` in the Celery config — every `.delay()` becomes a sync call.

---

## 9. Deployment

### Production checklist

- [ ] **Gunicorn behind nginx** — 4 workers per CPU core. Nginx terminates TLS, sets `X-Forwarded-For` (Flask-Limiter uses this if `RATELIMIT_HEADERS_ENABLED = True`).
- [ ] **Celery worker pool** — `--concurrency=4` per worker container, scale horizontally for big fanout bursts (likes on a viral post).
- [ ] **Single Celery beat instance** — schedule nothing here yet, but if you add a "clean up old notifications" job, run beat as a one-replica Deployment.
- [ ] **PostgreSQL** with connection pooling (PgBouncer in transaction mode) — Flask's `pool_size=20` × 4 workers = 80 connections; PgBouncer multiplexes them down to 10.
- [ ] **Redis** with `maxmemory-policy=allkeys-lru` — feeds are recreatable from Postgres; if Redis OOMs, oldest feed entries evict first.
- [ ] **S3 / R2 bucket policy** — block public access; only allow reads via presigned URLs or a CloudFront-signed-URL CDN in front.
- [ ] **Rate-limit headers** — `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` so client SDKs can back off gracefully.
- [ ] **JWT secret rotation** — store in a secrets manager (AWS Secrets Manager, Doppler, Vault). Rotate every 90 days; `JWT_SECRET_KEY` accepts a list to support overlap.
- [ ] **Cold-start fanout spikes** — when a celebrity with 1M followers posts, your Celery worker queue explodes. Set `worker_prefetch_multiplier=1` and `task_acks_late=True` so workers don't grab 100 fanout tasks at once and OOM.
- [ ] **Image scanning** — Pillow's `Image.open().verify()` on upload (or in a Celery task) rejects non-image payloads masquerading as JPEGs.
- [ ] **Observability** — `/metrics` endpoint (Prometheus), structured JSON logs (structlog), Sentry for error tracking.

### Scaling beyond a single Flask instance

```mermaid
flowchart TB
    LB[Load Balancer<br/>nginx / ALB] --> W1[Flask web 1]
    LB --> W2[Flask web 2]
    LB --> W3[Flask web N]
    W1 --> PG[(PostgreSQL<br/>+ PgBouncer)]
    W2 --> PG
    W3 --> PG
    W1 --> R[(Redis<br/>cluster mode)]
    W2 --> R
    W3 --> R
    QB[Celery queue] --> Q1[Worker 1]
    QB --> Q2[Worker 2]
    QB --> QN[Worker N]
    W1 --> S3[(S3 / R2<br/>+ CloudFront CDN)]
    style PG fill:#dcfce7,stroke:#16a34a
    style R fill:#fef3c7,stroke:#d97706
    style S3 fill:#fce7f3,stroke:#db2777
```

Stateless Flask + external Redis/Postgres/S3 means horizontal scaling is a knob — add workers, add Celery processes, add Redis replicas. The feed algorithm stays correct because Redis sorted sets are atomic operations.

---

## 10. Next Steps / Enhancements

1. **Reposts / quotes.** Add a `repost_of_id` FK on `Post`. Reposting a post pushes the same `post_id` into followers' feeds with the reposting user as the actor.
2. **Direct messages.** A `Conversation` model + a `Message` table. Use Flask-SocketIO for real-time delivery; reuse the JWT for auth.
3. **Trending hashtags.** A Celery beat job every 5 minutes counts hashtag occurrences in the last hour, stores the top 50 in Redis (`ZADD trending:hashtags`). The `/trending` endpoint reads from Redis.
4. **Image thumbnails.** A Celery task uses Pillow to generate 3 sizes (small, medium, large) on upload; stores them at `thumbnails/{key}/small.jpg` etc. Posts include `media_urls: {small, medium, large}`.
5. **GraphQL endpoint.** Add `strawberry-graphql` and expose `Query.posts`, `Query.user`, `Mutation.createPost`. Clients fetch exactly the fields they need — saves bandwidth on mobile.
6. **Push notifications.** Integrate Firebase Cloud Messaging. `fanout_notifications` enqueues a follow-up task that sends a push to each device token.
7. **Block / mute.** A `Block` table; fanout filters out blocked users' posts. A `Mute` table; soft-hide in the feed without unfollowing.
8. **Embeddings-based search.** Use `sentence-transformers` to embed every post body, store in `pgvector`, and serve semantic search (`"show me posts about distributed systems"` matches even without exact word overlap).
9. **Rate-limit per endpoint, not just per user.** Posting: 50/day. Liking: 1000/hour. Following: 100/hour. Different abuse surfaces, different limits.
10. **Webhooks for clients.** `POST /api/v1/webhooks` with a callback URL — Echo POSTs events (`post.created`, `post.liked`) to the client's server. Useful for bots.

---

## 11. What We Learned

- **Fanout-on-write is the standard feed algorithm** — push post IDs into each follower's Redis sorted set at write time. Reads become a single `ZREVRANGE`. The tradeoff is write amplification (1 post × N followers writes); we cap that with the celebrity hybrid (fanout-on-read for users with >10k followers).
- **Redis sorted sets** (`ZADD`, `ZREVRANGEBYSCORE`, `ZREMRANGEBYRANK`) are the right data structure for time-ordered feeds — atomic, O(log N) ops, naturally ordered by score.
- **JWT with refresh-token rotation** detects token theft: every refresh revokes the old `jti`, so a reused refresh is a strong signal of compromise.
- **Pre-signed S3 / R2 URLs** keep image bytes off your Flask servers — cheaper, faster, and removes the upload as a bottleneck.
- **Pydantic v2 with `from_attributes=True`** converts SQLAlchemy objects to API responses with one line: `PostOut.model_validate(post)`. The schema is the documentation.
- **Cursor-based pagination** with a `(created_at, id)` cursor beats offset pagination for two reasons: it's stable when new items arrive, and Postgres can use a `WHERE id < ?` index seek instead of an `OFFSET 5000` table scan.
- **Per-user rate limits keyed on the authenticated identity** (not IP) — NAT'd offices would otherwise throttle legit users.
- **Celery for notification fanout** keeps the request cycle fast. Liking a viral post may write 10k notification rows; do it in the background.
- **Denormalized counters** (`like_count`, `followers_count`) are an atomic `UPDATE ... SET count = count + 1` — never `SELECT COUNT(*)` live. Counters are eventually consistent with the truth table, and that's fine.
- **API versioning from day one** (`/api/v1/`) costs nothing and saves you when you need to ship a breaking change.
- **Hashtag extraction with regex** + an upsert pattern keeps the `Hashtag` table clean and case-insensitive. The `post_hashtags` association table is the many-to-many bridge.

> [!quote] A note from your teacher
> Echo is the project that taught *me* why Redis exists. Without it, the feed query is a four-table join that grows with your followings. With it, the feed is one `ZREVRANGE`. The fanout-on-write insight — *push the work to write-time so reads are cheap* — is the same insight behind every cache, every index, every materialized view in computer science. Build this once and you'll recognize the pattern everywhere: in Slack's channel reads, in Discord's message history, in your bank's transaction log. That's the worth of a portfolio project — it teaches you a *pattern*, not a feature.

```mermaid
flowchart LR
    subgraph Client
        MO[Mobile app]
        WE[Web client]
        CLI[CLI tool]
    end
    subgraph Echo API
        AUTH[/api/v1/auth/]
        POSTS[/api/v1/posts/]
        FEED[/api/v1/feed/]
        SOCIAL[/api/v1/social/]
        SEARCH[/api/v1/search/]
        MEDIA[/api/v1/media/]
    end
    subgraph State
        PG[(PostgreSQL<br/>users, posts, follows)]
        R[(Redis<br/>feeds + rate limits)]
        S3[(S3 / R2<br/>images)]
    end
    subgraph Async
        Q[Celery worker<br/>fanout + notifications]
    end
    MO --> AUTH & POSTS & FEED & SOCIAL & SEARCH & MEDIA
    WE --> AUTH & POSTS & FEED & SOCIAL & SEARCH & MEDIA
    CLI --> AUTH & POSTS & FEED & SEARCH
    AUTH --> PG
    POSTS --> PG
    POSTS -->|enqueue| Q
    FEED --> R
    FEED --> PG
    SOCIAL --> PG
    SOCIAL -->|enqueue| Q
    SEARCH --> PG
    MEDIA --> S3
    POSTS --> S3
    Q --> R
    Q --> PG
    style PG fill:#dcfce7,stroke:#16a34a
    style R fill:#fef3c7,stroke:#d97706
    style S3 fill:#fce7f3,stroke:#db2777
    style Q fill:#dbeafe,stroke:#2563eb
```
