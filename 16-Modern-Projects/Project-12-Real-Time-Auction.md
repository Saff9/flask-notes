---
title: "Project 12 — Real-Time Auction Platform (Gavel)"
module: 16-Modern-Projects
project_number: 12
estimated_time: "14–18 hours build, 45–65 hours to perfect"
difficulty: advanced
prerequisites:
  - "Complete the core course (Lessons 01–24)"
  - "[[Lesson-16-User-Model-AndPasswords]] — password hashing, the User model"
  - "[[Lesson-17-Flask-Login]] — sessions, login_required"
  - "[[Lesson-12-SQLAlchemy-Setup]] and [[Lesson-13-CRUD-Operations]] — models & CRUD"
  - "[[Lesson-14-Relationships]] — one-to-many (Auction → Bid), (User → Bids)"
  - "[[Lesson-18-Blueprints]] — splitting auctions, bids, seller, buyer blueprints"
  - "[[Lesson-22-Email-And-Tasks]] — Celery + Flask-Mail for winner email"
  - "[[Modern-Frontend-Integration]] — HTMX, Tailwind 4, Socket.IO"
  - "[[Modern-Databases-And-Caching]] — Redis rate limiting, PostgreSQL row locks"
  - "[[Project-10-Event-Booking-Platform]] — atomic inventory + Stripe webhook patterns"
tags:
  - flask
  - flask3
  - python313
  - project
  - portfolio
  - auction
  - real-time
  - websocket
  - socketio
  - stripe
  - redis
  - celery
  - htmx
  - tailwind
  - 2026
related_lessons:
  - "[[Lesson-13-CRUD-Operations]]"
  - "[[Lesson-17-Flask-Login]]"
  - "[[Lesson-18-Blueprints]]"
  - "[[Lesson-22-Email-And-Tasks]]"
  - "[[Lesson-23-Deployment]]"
  - "[[Modern-Frontend-Integration]]"
  - "[[Modern-Databases-And-Caching]]"
  - "[[Modern-Deployment-And-DevOps]]"
  - "[[Project-04-Modern-Ecommerce]]"
  - "[[Project-10-Event-Booking-Platform]]"
---

# Project 12 — Real-Time Auction Platform ("Gavel")

> [!success] The hardest combination in web engineering: real-time + money.
> By the end of this project you'll have a live auction platform: sellers list items with start prices, reserve prices, and end times; buyers place bids that every other viewer sees within milliseconds via WebSocket; auto-bid (proxy bidding) bids up to a user-configured maximum on their behalf; a server-authoritative countdown avoids client clock drift; the moment the timer hits zero, a Celery task settles the auction (notifies the winner, charges their saved payment method via Stripe). Rate limiting prevents bid spam. This is the project that proves you can build **concurrency-safe, real-time, financially-integrated systems** under load.

#flask #auction #real-time #websocket #socketio #stripe #redis #celery #htmx #tailwind #portfolio #2026

---

## 1. Project Overview

**Gavel** is a real-time auction platform built with **Flask 3.2 + SQLAlchemy 2.0 + PostgreSQL 16 + Redis 7 + Stripe + Celery + Flask-SocketIO + HTMX 2.0 + Tailwind 4**. Sellers create auctions with a start price, optional reserve price, optional buy-it-now price, and an end time. Buyers browse active auctions, watch a real-time bid stream, place bids (or configure auto-bid), and see the countdown timer tick down — synced to the server, not the client clock. When the auction ends, a Celery task determines the winner, sends notification emails, and creates a Stripe PaymentIntent for the winner to complete. A live dashboard shows the hottest auctions ending soon.

### Why this is a modern 2026 project

A 2018 "Flask auction" tutorial had three routes, a SQLite database, and a `current_bid` integer that two concurrent bids could race on — resulting in lost bids and angry users. Modern auction platforms in 2026 look completely different:

1. **Server-authoritative countdowns.** The client doesn't compute "time remaining" from `Date.now()` — clock drift, time zones, and malicious clients all break that. The server sends the canonical end time; the client renders a countdown from `Date.now()` *only* as a display hint, and the WebSocket delivers the authoritative "auction ended" event.
2. **Atomic bid placement with `SELECT FOR UPDATE`.** Two bids arriving in the same millisecond both lock the auction row; one wins, the other gets a "you've been outbid" message immediately.
3. **Proxy bidding (auto-bid).** Each bidder sets a maximum willingness to pay. The system bids the minimum increment on their behalf up to that limit — eBay-style. Implementing this correctly requires a tiny rules engine.
4. **WebSocket fan-out per auction room.** Each auction is its own SocketIO room; only viewers on that page receive bid updates. No global broadcast storms.
5. **Rate limiting via Redis.** A user can place at most 10 bids per minute — prevents a bot from spamming the bid endpoint to grief other users or probe for auto-bid limits.
6. **Celery beat for auction settlement.** Every minute, a beat task finds auctions whose end time has passed and enqueues a settlement task per auction. The settlement task determines the winner, sends emails, and creates a Stripe PaymentIntent.
7. **HTMX for the bid form.** Clicking "Bid $105" posts via HTMX; the new current bid swaps into the page without a reload. The WebSocket handles everyone *else's* bids.

> [!info] Where this fits in the project series
> Project 10 ([[Project-10-Event-Booking-Platform]]) taught you atomic inventory via Redis Lua scripts. **Gavel reuses that mindset** but the "inventory" is now a *single high-stakes item with a running price*. The concurrency story is different: instead of N seats, you have one auction row that every bid must `SELECT FOR UPDATE`. Read P10 first — we won't re-explain Stripe webhook idempotency.

### Architecture at a glance

```mermaid
flowchart TB
    B1[Bidder A] -->|POST /bid| F[Flask App]
    B2[Bidder B] -->|WS subscribe| WS[Socket.IO Room<br/>auction-123]
    B3[Bidder C] -->|WS subscribe| WS
    F -->|SELECT FOR UPDATE| PG[(PostgreSQL<br/>auctions + bids)]
    F -->|commit bid| PG
    F -->|broadcast new high bid| WS
    WS -->|live update| B1
    WS -->|live update| B2
    WS -->|live update| B3
    CB[Celery Beat<br/>every 1 min] -->|find ended auctions| F
    F -->|enqueue settle| Q[Celery Worker]
    Q -->|create PaymentIntent| ST[Stripe]
    Q -->|email winner + seller| E[SMTP]
    W[Winner] -->|pay via Stripe| ST
    ST -->|webhook payment_intent.succeeded| F
    F -->|mark paid| PG
    style F fill:#dbeafe,stroke:#2563eb
    style WS fill:#fce7f3,stroke:#db2777
    style PG fill:#dcfce7,stroke:#16a34a
```

---

## 2. What You'll Learn

By the end of this project you will have hands-on experience with:

- **PostgreSQL `SELECT FOR UPDATE`** — pessimistic row locks for atomic bid placement. Two concurrent bids on the same auction serialize; one wins, the other is rejected.
- **Flask-SocketIO rooms** — each auction is its own broadcast room; subscribers see bid updates only for the auction they're viewing.
- **Server-authoritative time** — the auction's `ends_at` is the canonical truth; clients render a *display* countdown, but the "ended" event comes from the server (Celery beat).
- **Proxy bidding (auto-bid) algorithm** — Vickrey-style incremental bidding: when a new bid arrives, the system raises the current bid to `min(new_bid, prior_high + increment)` and updates the high bidder accordingly.
- **Rate limiting via Redis** (`Flask-Limiter` with a Redis backend) — prevent bid spam and auto-bid probing.
- **Stripe PaymentIntents + webhooks** — the winner pays after the auction ends; the webhook marks the auction as "paid."
- **Celery beat for settlement** — a periodic task finds ended auctions and processes them.
- **HTMX 2.0** — bid form submission swaps the bid panel without a page reload.
- **Docker Compose** — six services (web, worker, beat, postgres, redis, stripe-cli) in one command.

---

## 3. Prerequisites

| Lesson | Why |
|---|---|
| [[Lesson-16-User-Model-AndPasswords]] | `User` model, password hashing. |
| [[Lesson-17-Flask-Login]] | `LoginManager`, `@login_required`. |
| [[Lesson-13-CRUD-Operations]] | `select()`, `db.session.add()`, `db.session.commit()`. |
| [[Lesson-14-Relationships]] | `Auction → Bid`, `User → Bid`. |
| [[Lesson-18-Blueprints]] | `auctions`, `bids`, `seller`, `buyer` blueprints. |
| [[Lesson-22-Email-And-Tasks]] | Celery beat, `@app.task`, Flask-Mail. |
| [[Modern-Databases-And-Caching]] | Redis rate limiting, PostgreSQL row locks. |
| [[Modern-Frontend-Integration]] | HTMX, Socket.IO, Tailwind 4. |
| [[Project-04-Modern-Ecommerce]] | Stripe webhook patterns. |
| [[Project-10-Event-Booking-Platform]] | Concurrency + Stripe + Celery patterns. |

You also need:
- Python 3.13+, Docker, Docker Compose
- A **Stripe** account in test mode (no real charges)
- A **Redis** 7+ instance (Docker is fine)
- PostgreSQL 16+

---

## 4. Tech Stack

| Tool | Version | Why |
|---|---|---|
| Flask | 3.2+ | Web framework. |
| Flask-SQLAlchemy | 3.1+ | ORM with 2.0-style `select()` and `with_for_update()`. |
| Flask-Login | 0.6+ | Session auth. |
| Flask-WTF | 1.2+ | CSRF on forms. |
| Flask-Limiter | 3.8+ | Rate-limit `/bid` (Redis backend). |
| Flask-Mail | 0.10+ | Transactional email. |
| Flask-SocketIO | 5.4+ | Real-time bid fan-out. |
| Celery | 5.4+ | Settlement task, beat. |
| Redis | 7 | Rate limiting, Celery broker, bid stream buffer. |
| PostgreSQL | 16 | Auctions, bids, users, payments. |
| stripe (Python SDK) | 11+ | PaymentIntents + webhook signature. |
| HTMX | 2.0 (CDN) | Bid form submission without SPA. |
| Tailwind CSS | 4 (CDN) | Styling. |
| Docker Compose | 2.30+ | Local dev stack. |

> [!tip] Why `SELECT FOR UPDATE` and not optimistic locking?
> Auctions have *hot* rows — when 50 users are bidding in the last 30 seconds, optimistic locking's abort-retry loop creates a thundering herd. Pessimistic locking with `SELECT FOR UPDATE` serializes bids for ~5 ms each, which is fine — even at 200 bids/sec the queue is manageable. The trade-off: a single bid holds the row for the duration of the transaction, so keep the transaction *short* (no network calls inside).

---

## 5. Project Structure

```
gavel/
├── app/
│   ├── __init__.py            # create_app() factory + SocketIO init
│   ├── config.py              # Config classes
│   ├── extensions.py          # db, login_manager, mail, socketio, celery, redis, limiter
│   ├── models.py              # User, Auction, Bid, Payment, Category
│   ├── sockets.py             # SocketIO: join auction room, broadcast bids
│   ├── bidding.py             # place_bid logic: validation, auto-bid, atomic lock
│   ├── auctions.py            # Blueprint: list, detail, search, categories
│   ├── bids.py                # Blueprint: place bid (HTMX), bid history
│   ├── account.py             # Blueprint: my bids, my wins, pay
│   ├── seller.py              # Blueprint: create auctions, view bids, see winners
│   ├── tasks.py               # Celery: settle_auction, send_winner_email
│   ├── templates/
│   │   ├── base.html          # Tailwind + HTMX + Socket.IO client + countdown
│   │   ├── auctions/{list,detail}.html
│   │   ├── bids/{panel,history}.html    # HTMX partials
│   │   ├── account/{bids,wins,pay}.html
│   │   └── seller/{dashboard,form}.html
│   └── static/
│       └── img/
├── migrations/
├── tests/
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── celery_worker.py
└── wsgi.py
```

---

## 6. Step-by-Step Build

We'll build this in **15 steps**. Every step produces a runnable checkpoint.

### Step 1 — Install dependencies

```bash
# requirements.txt
Flask==3.2.0
Flask-SQLAlchemy==3.1.1
Flask-Login==0.6.3
Flask-WTF==1.2.1
Flask-Limiter==3.8.0
Flask-Mail==0.10.0
Flask-SocketIO==5.4.1
SQLAlchemy==2.0.36
celery==5.4.0
redis==5.2.0
stripe==11.3.0
psycopg[binary]==3.2.3
python-dotenv==1.0.1
eventlet==0.37.0
gunicorn==23.0.0
pytest==8.3.4
```

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
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

    # Stripe
    STRIPE_SECRET_KEY = os.environ["STRIPE_SECRET_KEY"]
    STRIPE_PUBLISHABLE_KEY = os.environ["STRIPE_PUBLISHABLE_KEY"]
    STRIPE_WEBHOOK_SECRET = os.environ["STRIPE_WEBHOOK_SECRET"]
    STRIPE_CURRENCY = "usd"

    # Mail
    MAIL_SERVER = os.environ.get("MAIL_SERVER", "smtp.mailtrap.io")
    MAIL_PORT = int(os.environ.get("MAIL_PORT", 587))
    MAIL_USERNAME = os.environ.get("MAIL_USERNAME")
    MAIL_PASSWORD = os.environ.get("MAIL_PASSWORD")
    MAIL_USE_TLS = True

    # Auction
    BID_INCREMENT_CENTS = 100         # minimum $1.00 over current bid
    RATELIMIT_STORAGE_URI = os.environ.get("REDIS_URL")
    RATELIMIT_DEFAULT = "200/hour"

    # Celery
    CELERY_BROKER_URL = os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/1")
    CELERY_RESULT_BACKEND = os.environ.get("CELERY_RESULT_BACKEND", "redis://localhost:6379/2")
    SITE_URL = os.environ.get("SITE_URL", "http://localhost:5000")

class TestConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"
    RATELIMIT_ENABLED = False
    WTF_CSRF_ENABLED = False
```

### Step 3 — Extensions + factory

```python
# app/extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager
from flask_wtf import CSRFProtect
from flask_mail import Mail
from flask_socketio import SocketIO
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from celery import Celery
import redis

db = SQLAlchemy()
login_manager = LoginManager()
csrf = CSRFProtect()
mail = Mail()
socketio = SocketIO(cors_allowed_origins="*", async_mode="eventlet")
limiter = Limiter(key_func=get_remote_address, default_limits=["200/hour"])
celery = Celery("gavel")

def make_redis(app):
    return redis.Redis.from_url(app.config["REDIS_URL"], decode_responses=True)
```

```python
# app/__init__.py
from flask import Flask
from app.config import Config
from app.extensions import (
    db, login_manager, csrf, mail, socketio, limiter, celery, make_redis
)

def create_app(config_class=Config) -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_class)

    db.init_app(app)
    login_manager.init_app(app)
    csrf.init_app(app)
    mail.init_app(app)
    limiter.init_app(app)
    socketio.init_app(app, async_mode="eventlet")
    app.redis = make_redis(app)

    celery.conf.update(
        broker_url=app.config["CELERY_BROKER_URL"],
        result_backend=app.config["CELERY_RESULT_BACKEND"],
        task_serializer="json",
        accept_content=["json"],
        beat_schedule={
            # Find auctions that have ended and settle them
            "settle-ended-auctions": {
                "task": "app.tasks.settle_ended_auctions",
                "schedule": 60.0,              # every minute
            },
            # Broadcast countdown ticks to active auction rooms
            "broadcast-countdowns": {
                "task": "app.tasks.broadcast_countdowns",
                "schedule": 1.0,               # every second
            },
        },
    )

    from app.models import User
    @login_manager.user_loader
    def load_user(uid):
        return db.session.get(User, int(uid))

    from app.auctions import auctions_bp
    from app.bids import bids_bp
    from app.account import account_bp
    from app.seller import seller_bp
    from app.webhooks import webhooks_bp
    app.register_blueprint(auctions_bp)
    app.register_blueprint(bids_bp, url_prefix="/bids")
    app.register_blueprint(account_bp, url_prefix="/account")
    app.register_blueprint(seller_bp, url_prefix="/seller")
    app.register_blueprint(webhooks_bp, url_prefix="/webhooks")

    from app import sockets  # noqa: F401
    return app
```

### Step 4 — Models

```python
# app/models.py
from datetime import datetime, timezone
from flask_login import UserMixin
from sqlalchemy import (
    Integer, String, Text, DateTime, ForeignKey, Boolean, Numeric, Index
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
    name: Mapped[str] = mapped_column(String(120))
    is_seller: Mapped[bool] = mapped_column(Boolean, default=False)
    stripe_customer_id: Mapped[str | None] = mapped_column(String(255), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    auctions: Mapped[list["Auction"]] = relationship(back_populates="seller")
    bids: Mapped[list["Bid"]] = relationship(back_populates="bidder")

    def set_password(self, pw: str) -> None:
        self.password_hash = generate_password_hash(pw)
    def check_password(self, pw: str) -> bool:
        return check_password_hash(self.password_hash, pw)

class Category(db.Model):
    __tablename__ = "categories"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(80), unique=True)
    slug: Mapped[str] = mapped_column(String(80), unique=True, index=True)
    auctions: Mapped[list["Auction"]] = relationship(back_populates="category")

class Auction(db.Model):
    __tablename__ = "auctions"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200), index=True)
    slug: Mapped[str] = mapped_column(String(200), unique=True, index=True)
    description: Mapped[str] = mapped_column(Text)
    image_path: Mapped[str | None] = mapped_column(String(500), nullable=True)

    start_price_cents: Mapped[int] = mapped_column(Integer)
    reserve_price_cents: Mapped[int | None] = mapped_column(Integer, nullable=True)
    buy_now_price_cents: Mapped[int | None] = mapped_column(Integer, nullable=True)

    increment_cents: Mapped[int] = mapped_column(Integer, default=100)  # min overbid
    current_bid_cents: Mapped[int | None] = mapped_column(Integer, nullable=True)
    current_high_bidder_id: Mapped[int | None] = mapped_column(
        ForeignKey("users.id"), nullable=True
    )

    starts_at: Mapped[datetime] = mapped_column(DateTime(timezone=True))
    ends_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), index=True)

    status: Mapped[str] = mapped_column(String(20), default="pending")
    # pending | active | ended | sold | canceled | paid

    winner_id: Mapped[int | None] = mapped_column(ForeignKey("users.id"), nullable=True)
    winner_bid_id: Mapped[int | None] = mapped_column(ForeignKey("bids.id"), nullable=True)
    winning_price_cents: Mapped[int | None] = mapped_column(Integer, nullable=True)
    reserve_met: Mapped[bool] = mapped_column(Boolean, default=False)

    seller_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    category_id: Mapped[int | None] = mapped_column(ForeignKey("categories.id"), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    seller: Mapped[User] = relationship(
        back_populates="auctions", foreign_keys=[seller_id]
    )
    category: Mapped[Category | None] = relationship(back_populates="auctions")
    bids: Mapped[list["Bid"]] = relationship(
        back_populates="auction",
        foreign_keys="Bid.auction_id",
        cascade="all, delete-orphan",
        order_by="Bid.amount_cents.desc()"
    )
    winner: Mapped[User | None] = relationship(foreign_keys=[winner_id])
    winning_bid: Mapped["Bid | None"] = relationship(foreign_keys=[winner_bid_id])

    @property
    def is_active(self) -> bool:
        return (
            self.status == "active"
            and self.starts_at <= utcnow() < self.ends_at
        )
    @property
    def next_min_bid(self) -> int:
        base = self.current_bid_cents or self.start_price_cents
        return base + self.increment_cents

class Bid(db.Model):
    __tablename__ = "bids"
    __table_args__ = (
        Index("ix_bids_auction_amount", "auction_id", "amount_cents"),
    )
    id: Mapped[int] = mapped_column(primary_key=True)
    auction_id: Mapped[int] = mapped_column(ForeignKey("auctions.id"), index=True)
    bidder_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    amount_cents: Mapped[int] = mapped_column(Integer)
    is_auto_bid: Mapped[bool] = mapped_column(Boolean, default=False)
    max_auto_bid_cents: Mapped[int | None] = mapped_column(Integer, nullable=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=utcnow, index=True
    )

    auction: Mapped[Auction] = relationship(
        back_populates="bids", foreign_keys=[auction_id]
    )
    bidder: Mapped[User] = relationship(back_populates="bids", foreign_keys=[bidder_id])

class Payment(db.Model):
    __tablename__ = "payments"
    id: Mapped[int] = mapped_column(primary_key=True)
    auction_id: Mapped[int] = mapped_column(ForeignKey("auctions.id"), index=True)
    payer_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    stripe_payment_intent: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    amount_cents: Mapped[int] = mapped_column(Integer)
    status: Mapped[str] = mapped_column(String(20))  # pending | succeeded | failed
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
```

### Step 5 — Entity-relationship map

```mermaid
erDiagram
    USER ||--o{ AUCTION : sells
    USER ||--o{ BID : places
    USER ||--o{ AUCTION : wins
    CATEGORY ||--o{ AUCTION : has
    AUCTION ||--o{ BID : receives
    AUCTION ||--o| PAYMENT : settled_by
    USER ||--o{ PAYMENT : pays
    USER {
        int id PK
        string email
        string password_hash
        bool is_seller
        string stripe_customer_id
    }
    AUCTION {
        int id PK
        string title
        int start_price_cents
        int reserve_price_cents
        int buy_now_price_cents
        int current_bid_cents
        datetime starts_at
        datetime ends_at
        string status
        int winner_id FK
    }
    BID {
        int id PK
        int auction_id FK
        int bidder_id FK
        int amount_cents
        bool is_auto_bid
        int max_auto_bid_cents
    }
    PAYMENT {
        int id PK
        int auction_id FK
        int payer_id FK
        string stripe_payment_intent
        int amount_cents
        string status
    }
```

### Step 6 — Bidding logic with `SELECT FOR UPDATE`

This is the heart of the platform. Two rules:

1. **A bid must be at least `current_bid + increment`** (or `start_price` if no bids yet).
2. **If the new bid is higher than the current high bidder's auto-bid max**, the new bid wins at `prior_high_bidder.max_auto_bid + increment` (one tick above their max), and they get notified they've been outbid.

```python
# app/bidding.py
"""Atomic bid placement with SELECT FOR UPDATE.

The whole function runs inside a single DB transaction; the auction row
is locked for the duration so concurrent bids serialize.
"""
from datetime import datetime, timezone
from dataclasses import dataclass
from flask import current_app
from sqlalchemy import select
from sqlalchemy.orm import with_for_update

from app.extensions import db
from app.models import Auction, Bid, User

@dataclass
class BidResult:
    ok: bool
    reason: str = ""
    new_current_bid: int | None = None
    new_high_bidder_id: int | None = None
    outbid_user_id: int | None = None   # the user we displaced (if any)

def place_bid(auction_id: int, bidder: User, amount_cents: int,
              is_auto_bid: bool = False, max_auto_bid_cents: int | None = None) -> BidResult:
    """Place a bid atomically. Returns BidResult with the outcome."""
    now = datetime.now(timezone.utc)

    # Lock the auction row for the duration of this transaction.
    # Other bids on this auction wait until we commit (or rollback).
    auction = db.session.execute(
        select(Auction)
        .where(Auction.id == auction_id)
        .with_for_update()           # SELECT ... FOR UPDATE
    ).scalar_one_or_none()

    if not auction:
        return BidResult(ok=False, reason="auction_not_found")
    if auction.status != "active":
        return BidResult(ok=False, reason="auction_not_active")
    if not (auction.starts_at <= now < auction.ends_at):
        return BidResult(ok=False, reason="auction_not_in_window")
    if auction.seller_id == bidder.id:
        return BidResult(ok=False, reason="cannot_bid_on_own_auction")

    # The minimum acceptable bid
    min_bid = auction.next_min_bid
    if amount_cents < min_bid:
        return BidResult(ok=False, reason="below_minimum",
                         new_current_bid=auction.current_bid_cents,
                         new_high_bidder_id=auction.current_high_bidder_id)

    # Look up the current high bidder's auto-bid max (if any).
    # If the new bid exceeds their max, the new bid wins at one tick above.
    prior_high_id = auction.current_high_bidder_id
    prior_max = None
    if prior_high_id and prior_high_id != bidder.id:
        prior_auto = db.session.execute(
            select(Bid)
            .where(
                Bid.auction_id == auction_id,
                Bid.bidder_id == prior_high_id,
                Bid.is_auto_bid == True,
            )
            .order_by(Bid.max_auto_bid_cents.desc())
            .limit(1)
        ).scalar_one_or_none()
        if prior_auto:
            prior_max = prior_auto.max_auto_bid_cents

    increment = auction.increment_cents

    # ----- Proxy bidding logic ----------------------------------------------
    if is_auto_bid and max_auto_bid_cents:
        # The user wants the system to bid up to max_auto_bid_cents.
        # Effective ceiling: max_auto_bid_cents
        if prior_max is not None:
            # Two auto-bidders compete
            if max_auto_bid_cents > prior_max:
                # New bidder wins; price = prior_max + increment
                final_price = min(prior_max + increment, max_auto_bid_cents)
                outbid = prior_high_id
            else:
                # Prior bidder still wins; price = max_auto_bid_cents + increment
                final_price = min(max_auto_bid_cents + increment, prior_max)
                outbid = bidder.id
                # Insert a record of the new bidder's auto-bid (lost)
                db.session.add(Bid(
                    auction_id=auction_id, bidder_id=bidder.id,
                    amount_cents=max_auto_bid_cents, is_auto_bid=True,
                    max_auto_bid_cents=max_auto_bid_cents,
                ))
                auction.current_bid_cents = final_price
                db.session.add(auction); db.session.commit()
                return BidResult(ok=True, reason="auto_bid_lost",
                                 new_current_bid=final_price,
                                 new_high_bidder_id=prior_high_id,
                                 outbid_user_id=outbid)
        else:
            # No prior auto-bid; bid one increment above current (or start price)
            final_price = min(
                (auction.current_bid_cents or auction.start_price_cents - increment) + increment,
                max_auto_bid_cents,
            )
            outbid = prior_high_id
        # Cap at the actual amount if there's no competition
        final_price = max(final_price, min_bid)
        amount_cents = final_price
    else:
        # Manual bid: must be >= min_bid (already checked)
        # If this exceeds prior auto-bid max, the system re-prices to prior_max + increment
        if prior_max is not None and amount_cents > prior_max:
            # New manual bid wins; price = prior_max + increment (not full amount)
            amount_cents = prior_max + increment
            outbid = prior_high_id
        elif prior_max is not None and amount_cents <= prior_max:
            # Prior auto-bidder still wins; raise the price to amount + increment
            new_price = min(amount_cents + increment, prior_max)
            auction.current_bid_cents = new_price
            db.session.add(auction)
            # Insert a record of this (lost) manual bid
            db.session.add(Bid(
                auction_id=auction_id, bidder_id=bidder.id,
                amount_cents=amount_cents, is_auto_bid=False,
            ))
            db.session.commit()
            return BidResult(ok=False, reason="outbid_by_auto",
                             new_current_bid=new_price,
                             new_high_bidder_id=prior_high_id)
        else:
            outbid = prior_high_id

    # Insert the winning bid record
    bid = Bid(
        auction_id=auction_id, bidder_id=bidder.id,
        amount_cents=amount_cents, is_auto_bid=is_auto_bid,
        max_auto_bid_cents=max_auto_bid_cents if is_auto_bid else None,
    )
    db.session.add(bid)

    # Update auction state
    auction.current_bid_cents = amount_cents
    auction.current_high_bidder_id = bidder.id
    if (auction.reserve_price_cents is not None
            and amount_cents >= auction.reserve_price_cents):
        auction.reserve_met = True
    db.session.add(auction)
    db.session.commit()

    return BidResult(
        ok=True,
        new_current_bid=amount_cents,
        new_high_bidder_id=bidder.id,
        outbid_user_id=outbid,
    )

def buy_now(auction_id: int, buyer: User) -> BidResult:
    """Buy-it-now: immediately ends the auction at the buy-now price."""
    auction = db.session.execute(
        select(Auction).where(Auction.id == auction_id).with_for_update()
    ).scalar_one_or_none()
    if not auction or not auction.is_active:
        return BidResult(ok=False, reason="not_active")
    if not auction.buy_now_price_cents:
        return BidResult(ok=False, reason="no_buy_now")
    if auction.seller_id == buyer.id:
        return BidResult(ok=False, reason="cannot_buy_own")

    bid = Bid(
        auction_id=auction_id, bidder_id=buyer.id,
        amount_cents=auction.buy_now_price_cents, is_auto_bid=False,
    )
    db.session.add(bid)
    auction.current_bid_cents = auction.buy_now_price_cents
    auction.current_high_bidder_id = buyer.id
    auction.status = "ended"
    auction.ends_at = datetime.now(timezone.utc)   # ends immediately
    if (auction.reserve_price_cents is None
            or auction.buy_now_price_cents >= auction.reserve_price_cents):
        auction.reserve_met = True
    db.session.add(auction); db.session.commit()
    return BidResult(ok=True, new_current_bid=auction.buy_now_price_cents,
                     new_high_bidder_id=buyer.id)
```

> [!info] Why `with_for_update()` instead of version columns?
> Optimistic locking (version columns) lets the transaction proceed, then aborts at commit time if the row was modified. For a *hot* auction row with 50 concurrent bids, almost every transaction would abort and retry — wasteful and slow. `SELECT FOR UPDATE` acquires a row lock immediately; concurrent bids queue up but always commit on the first try. The trade-off is latency under load (each bid waits ~5 ms), but for a high-stakes item where correctness is non-negotiable, this is the right call.

### Step 7 — SocketIO real-time bid fan-out

```python
# app/sockets.py
from flask_socketio import join_room, leave_room, emit
from app.extensions import socketio

@socketio.on("connect")
def on_connect():
    pass  # auth via session cookie

@socketio.on("subscribe_auction")
def on_subscribe(data):
    auction_id = data["auction_id"]
    join_room(f"auction-{auction_id}")

@socketio.on("unsubscribe_auction")
def on_unsubscribe(data):
    leave_room(f"auction-{data['auction_id']}")

def broadcast_bid_update(auction_id: int, payload: dict):
    """Called after a successful bid. Fans out to all viewers of this auction."""
    socketio.emit(
        "bid_update",
        {"auction_id": auction_id, **payload},
        room=f"auction-{auction_id}",
    )

def broadcast_auction_ended(auction_id: int, payload: dict):
    socketio.emit(
        "auction_ended",
        {"auction_id": auction_id, **payload},
        room=f"auction-{auction_id}",
    )

def broadcast_countdown(auction_id: int, seconds_left: int):
    """Server-authoritative countdown tick. Sent every second by Celery beat."""
    socketio.emit(
        "countdown_tick",
        {"auction_id": auction_id, "seconds_left": seconds_left},
        room=f"auction-{auction_id}",
    )
```

### Step 8 — Auctions blueprint (list, detail, search)

```python
# app/auctions.py
from datetime import datetime, timezone
from flask import Blueprint, render_template, request, redirect, url_for, current_app
from sqlalchemy import select, or_
from app.models import Auction, Category
from app.extensions import db

auctions_bp = Blueprint("auctions", __name__)

@auctions_bp.route("/")
def index():
    now = datetime.now(timezone.utc)
    stmt = (
        select(Auction)
        .where(Auction.status == "active", Auction.starts_at <= now, Auction.ends_at > now)
        .order_by(Auction.ends_at.asc())
    )
    auctions = db.session.execute(stmt).scalars().all()
    return render_template("auctions/list.html", auctions=auctions, tab="active")

@auctions_bp.route("/upcoming")
def upcoming():
    stmt = (
        select(Auction)
        .where(Auction.status == "active", Auction.starts_at > datetime.now(timezone.utc))
        .order_by(Auction.starts_at.asc())
    )
    auctions = db.session.execute(stmt).scalars().all()
    return render_template("auctions/list.html", auctions=auctions, tab="upcoming")

@auctions_bp.route("/ended")
def ended():
    stmt = (
        select(Auction)
        .where(Auction.status.in_(["ended", "sold", "paid"]))
        .order_by(Auction.ends_at.desc())
    )
    auctions = db.session.execute(stmt).scalars().all()
    return render_template("auctions/list.html", auctions=auctions, tab="ended")

@auctions_bp.route("/search")
def search():
    q = request.args.get("q", "").strip()
    stmt = select(Auction).order_by(Auction.ends_at.asc())
    if q:
        stmt = stmt.where(
            or_(
                Auction.title.ilike(f"%{q}%"),
                Auction.description.ilike(f"%{q}%"),
            )
        )
    auctions = db.session.execute(stmt).scalars().all()
    return render_template("auctions/list.html", auctions=auctions, q=q, tab="search")

@auctions_bp.route("/category/<slug>")
def by_category(slug: str):
    stmt = (
        select(Auction)
        .join(Category)
        .where(Category.slug == slug)
        .order_by(Auction.ends_at.asc())
    )
    auctions = db.session.execute(stmt).scalars().all()
    return render_template("auctions/list.html", auctions=auctions, category_slug=slug)

@auctions_bp.route("/auction/<slug>")
def detail(slug: str):
    auction = db.session.execute(
        select(Auction).where(Auction.slug == slug)
    ).scalar_one_or_none()
    if not auction:
        return "Not found", 404
    bid_history = db.session.execute(
        select(Bid).where(Bid.auction_id == auction.id).order_by(Bid.created_at.desc()).limit(20)
    ).scalars().all() if False else auction.bids[:20]
    return render_template("auctions/detail.html", auction=auction, bid_history=bid_history)

# import Bid here at the bottom to avoid circular imports in the slice above
from app.models import Bid
```

### Step 9 — Bids blueprint (place bid via HTMX)

```python
# app/bids.py
from flask import Blueprint, render_template, request, redirect, url_for, flash, jsonify
from flask_login import login_required, current_user
from flask_limiter import Limiter
from app.extensions import db, limiter
from app.models import Auction, Bid
from app.bidding import place_bid, buy_now, BidResult
from app.sockets import broadcast_bid_update

bids_bp = Blueprint("bids", __name__)

@bids_bp.route("/place/<int:auction_id>", methods=["POST"])
@login_required
@limiter.limit("10/minute")         # prevent bid spam
def place(auction_id: int):
    auction = db.session.get(Auction, auction_id)
    if not auction:
        return "Not found", 404

    amount_str = request.form.get("amount", "")
    try:
        amount_dollars = float(amount_str)
        amount_cents = int(amount_dollars * 100)
    except (TypeError, ValueError):
        return "Invalid amount", 400

    is_auto = request.form.get("is_auto_bid") == "on"
    max_auto_cents = None
    if is_auto:
        max_str = request.form.get("max_auto_bid", "")
        try:
            max_auto_cents = int(float(max_str) * 100)
        except (TypeError, ValueError):
            return "Invalid max", 400

    result: BidResult = place_bid(
        auction_id, current_user, amount_cents,
        is_auto_bid=is_auto, max_auto_bid_cents=max_auto_cents,
    )

    if not result.ok:
        # Return the updated bid panel with an error message
        auction = db.session.get(Auction, auction_id)  # refresh
        return render_template(
            "bids/panel.html", auction=auction,
            error=result.reason, next_min_bid=auction.next_min_bid,
        ), 400 if result.reason in ("below_minimum",) else 200

    # Refresh and broadcast
    auction = db.session.get(Auction, auction_id)
    broadcast_bid_update(auction_id, {
        "current_bid_cents": auction.current_bid_cents,
        "high_bidder_id": auction.current_high_bidder_id,
        "next_min_bid_cents": auction.next_min_bid,
        "outbid_user_id": result.outbid_user_id,
    })

    # Return the updated bid panel (HTMX swaps it in for the bidder only)
    return render_template(
        "bids/panel.html", auction=auction,
        success="Bid placed!", next_min_bid=auction.next_min_bid,
    )

@bids_bp.route("/buy-now/<int:auction_id>", methods=["POST"])
@login_required
def buy_now_route(auction_id: int):
    result = buy_now(auction_id, current_user)
    if not result.ok:
        flash(f"Could not buy: {result.reason}", "error")
        return redirect(url_for("auctions.detail", slug=db.session.get(Auction, auction_id).slug))
    auction = db.session.get(Auction, auction_id)
    from app.sockets import broadcast_auction_ended
    broadcast_auction_ended(auction_id, {
        "winner_id": current_user.id, "price_cents": auction.current_bid_cents,
        "reason": "buy_now",
    })
    # Enqueue settlement immediately
    from app.tasks import settle_auction
    settle_auction.delay(auction_id)
    return redirect(url_for("account.pay", auction_id=auction_id))

@bids_bp.route("/history/<int:auction_id>")
def history(auction_id: int):
    auction = db.session.get(Auction, auction_id)
    return render_template("bids/history.html", auction=auction)
```

### Step 10 — Bid panel template (HTMX)

```html
<!-- templates/bids/panel.html -->
<div id="bid-panel" class="border p-4 rounded">
  {% if success %}
  <div class="bg-green-100 text-green-800 p-2 mb-2 rounded">{{ success }}</div>
  {% endif %}
  {% if error %}
  <div class="bg-red-100 text-red-800 p-2 mb-2 rounded">Error: {{ error }}</div>
  {% endif %}

  <p class="text-3xl font-bold">${{ '%.2f'|format(auction.current_bid_cents / 100) }}</p>
  <p class="text-sm text-gray-500">
    Next minimum bid: ${{ '%.2f'|format(next_min_bid / 100) }}
  </p>

  {% if auction.is_active %}
  <form hx-post="/bids/place/{{ auction.id }}"
        hx-target="#bid-panel"
        hx-swap="outerHTML"
        class="mt-2">
    <input type="number" name="amount" step="0.01"
           min="{{ next_min_bid / 100 }}"
           value="{{ '%.2f'|format(next_min_bid / 100) }}"
           class="border p-2 w-full">
    <label class="block mt-2 text-sm">
      <input type="checkbox" name="is_auto_bid" onchange="document.getElementById('max-field').classList.toggle('hidden')">
      Auto-bid up to a maximum
    </label>
    <input id="max-field" type="number" name="max_auto_bid" step="0.01"
           class="border p-2 w-full mt-2 hidden">
    <button class="btn btn-primary w-full mt-2">Place Bid</button>
  </form>

  {% if auction.buy_now_price_cents %}
  <form hx-post="/bids/buy-now/{{ auction.id }}" class="mt-2"
        hx-confirm="Buy now for ${{ '%.2f'|format(auction.buy_now_price_cents / 100) }}?">
    <button class="btn btn-secondary w-full">
      Buy Now ${{ '%.2f'|format(auction.buy_now_price_cents / 100) }}
    </button>
  </form>
  {% endif %}
  {% elif auction.status == "ended" or auction.status == "sold" or auction.status == "paid" %}
  <p class="text-lg">Auction ended.</p>
  {% else %}
  <p class="text-lg">Starts {{ auction.starts_at }}</p>
  {% endif %}
</div>
```

The countdown and bid stream client-side:

```html
<!-- templates/auctions/detail.html (script section) -->
<body data-auction-id="{{ auction.id }}">
  <h1>{{ auction.title }}</h1>

  <div class="grid grid-cols-2 gap-4">
    <div>
      <img src="{{ auction.image_path }}" alt="{{ auction.title }}">
      <div class="prose">{{ auction.description }}</div>
    </div>
    <div>
      <div id="countdown" class="text-2xl font-mono"
           data-ends-at="{{ auction.ends_at.isoformat() }}">
        Loading...
      </div>
      <div id="bid-panel">
        {% include "bids/panel.html" %}
      </div>
      <div id="bid-history" class="mt-4">
        {% include "bids/history.html" %}
      </div>
    </div>
  </div>

  <script src="https://cdn.socket.io/4.7/socket.io.min.js"></script>
  <script>
    const AUCTION_ID = document.body.dataset.auctionId;
    const socket = io();
    socket.emit("subscribe_auction", {auction_id: parseInt(AUCTION_ID)});

    // --- Server-authoritative countdown ---
    // Display-only; the actual "ended" event comes from the server.
    const endsAt = new Date(document.getElementById("countdown").dataset.endsAt);
    function tickCountdown() {
      const msLeft = endsAt - new Date();
      if (msLeft <= 0) {
        document.getElementById("countdown").textContent = "ENDED";
        return;
      }
      const s = Math.floor(msLeft / 1000);
      const h = Math.floor(s / 3600), m = Math.floor((s % 3600) / 60), sec = s % 60;
      document.getElementById("countdown").textContent =
        `${h}h ${m}m ${sec}s`;
    }
    tickCountdown();
    setInterval(tickCountdown, 1000);

    // --- Server-authoritative countdown tick (drift correction) ---
    socket.on("countdown_tick", (data) => {
      if (data.auction_id !== parseInt(AUCTION_ID)) return;
      const s = data.seconds_left;
      const h = Math.floor(s / 3600), m = Math.floor((s % 3600) / 60), sec = s % 60;
      document.getElementById("countdown").textContent = `${h}h ${m}m ${sec}s`;
    });

    // --- Real-time bid updates ---
    socket.on("bid_update", (data) => {
      if (data.auction_id !== parseInt(AUCTION_ID)) return;
      // Re-fetch the bid panel HTML (server-rendered with new state)
      fetch(`/bids/history/${AUCTION_ID}`)
        .then(r => r.text())
        .then(html => {
          document.getElementById("bid-history").innerHTML = html;
        });
      // Update the displayed current bid
      const newBid = (data.current_bid_cents / 100).toFixed(2);
      document.querySelector("#bid-panel .text-3xl").textContent = `$${newBid}`;
      if (data.outbid_user_id === {{ current_user.id if current_user.is_authenticated else 'null' }}) {
        const banner = document.createElement("div");
        banner.className = "bg-yellow-100 text-yellow-800 p-2 rounded mt-2";
        banner.textContent = "You've been outbid!";
        document.getElementById("bid-panel").prepend(banner);
      }
    });

    socket.on("auction_ended", (data) => {
      if (data.auction_id !== parseInt(AUCTION_ID)) return;
      location.reload();   // simplest: refresh to show "ended" state
    });
  </script>
</body>
```

```mermaid
sequenceDiagram
    participant A as Bidder A
    participant B as Bidder B
    participant F as Flask
    participant DB as PostgreSQL
    participant WS as Socket.IO room
    participant CB as Celery beat

    Note over A,B: Both bid at the same millisecond
    A->>F: POST /bids/place/1 ($105)
    B->>F: POST /bids/place/1 ($106)

    Note over F,DB: SELECT ... FOR UPDATE serializes
    F->>DB: BEGIN; SELECT * FROM auctions WHERE id=1 FOR UPDATE
    Note over F,DB: A's tx holds the lock; B's tx waits ~5ms
    F->>DB: INSERT bid $105; UPDATE auction SET current_bid=$105
    F->>DB: COMMIT (lock released)
    F->>WS: emit bid_update (current_bid=$105)
    WS-->>A: ack
    WS-->>B: bid_update (current_bid=$105, outbid!)

    Note over F,DB: B's tx now acquires the lock
    F->>DB: SELECT * FROM auctions WHERE id=1 FOR UPDATE
    F->>DB: INSERT bid $106; UPDATE auction SET current_bid=$106
    F->>DB: COMMIT
    F->>WS: emit bid_update (current_bid=$106)
    WS-->>A: outbid!
    WS-->>B: ack (you're high bidder)

    Note over CB: Every second, beat broadcasts countdown
    CB->>F: beat task: broadcast_countdowns
    F->>WS: emit countdown_tick (seconds_left=89)
    WS-->>A: tick
    WS-->>B: tick
```

### Step 11 — Celery tasks: countdown broadcast + settlement

```python
# app/tasks.py
import logging
from datetime import datetime, timezone, timedelta
from celery import shared_task
from sqlalchemy import select
import stripe

from app.extensions import db, celery, mail
from app.models import Auction, Bid, User, Payment
from flask_mail import Message

log = logging.getLogger(__name__)

def _get_app():
    from app import create_app
    return create_app()

# --- Countdown broadcaster (every second) -----------------------------------

@celery.task
def broadcast_countdowns():
    """For every active auction, broadcast seconds_left to its room."""
    app = _get_app()
    with app.app_context():
        now = datetime.now(timezone.utc)
        stmt = (
            select(Auction)
            .where(Auction.status == "active", Auction.ends_at > now)
        )
        auctions = db.session.execute(stmt).scalars().all()
        from app.sockets import broadcast_countdown
        for a in auctions:
            seconds = int((a.ends_at - now).total_seconds())
            broadcast_countdown(a.id, seconds)

# --- Settlement (every minute, find ended auctions) -------------------------

@celery.task
def settle_ended_auctions():
    """Find auctions that ended but aren't yet settled; enqueue settlement."""
    app = _get_app()
    with app.app_context():
        now = datetime.now(timezone.utc)
        stmt = (
            select(Auction)
            .where(
                Auction.status == "active",
                Auction.ends_at <= now,
            )
        )
        ended = db.session.execute(stmt).scalars().all()
        for auction in ended:
            # Mark as ended so we don't double-settle
            auction.status = "ended"
            db.session.add(auction)
            settle_auction.delay(auction.id)
        db.session.commit()
        return len(ended)

@celery.task(bind=True, max_retries=3)
def settle_auction(self, auction_id: int):
    """Determine the winner, notify parties, create a Stripe PaymentIntent."""
    app = _get_app()
    with app.app_context():
        auction = db.session.get(Auction, auction_id)
        if not auction or auction.status not in ("ended",):
            return  # already settled or canceled

        # Determine winner: highest bidder if reserve met, else None
        if auction.bids and auction.reserve_met:
            winner_bid = max(auction.bids, key=lambda b: b.amount_cents)
            auction.winner_id = winner_bid.bidder_id
            auction.winner_bid_id = winner_bid.id
            auction.winning_price_cents = winner_bid.amount_cents
            auction.status = "sold"
        else:
            # No winner (reserve not met or no bids)
            auction.status = "ended"  # final state, no sale

        db.session.add(auction); db.session.commit()

        from app.sockets import broadcast_auction_ended
        broadcast_auction_ended(auction_id, {
            "winner_id": auction.winner_id,
            "price_cents": auction.winning_price_cents,
            "reserve_met": auction.reserve_met,
        })

        # Notify winner + seller via email
        if auction.winner_id:
            winner = db.session.get(User, auction.winner_id)
            seller = auction.seller
            # Winner email
            msg = Message(
                subject=f"You won: {auction.title}",
                recipients=[winner.email],
            )
            msg.body = (
                f"Hi {winner.name},\n\n"
                f"You won the auction for {auction.title} at "
                f"${auction.winning_price_cents / 100:.2f}.\n"
                f"Pay within 7 days: {app.config['SITE_URL']}/account/pay/{auction.id}\n"
            )
            mail.send(msg)
            # Seller email
            msg2 = Message(
                subject=f"Your auction ended: {auction.title}",
                recipients=[seller.email],
            )
            msg2.body = (
                f"Hi {seller.name},\n\n"
                f"Your auction for {auction.title} sold for "
                f"${auction.winning_price_cents / 100:.2f}.\n"
                f"Winner: {winner.email}\n"
            )
            mail.send(msg2)

            # Create a Stripe PaymentIntent (or use a saved payment method in production)
            stripe.api_key = app.config["STRIPE_SECRET_KEY"]
            intent = stripe.PaymentIntent.create(
                amount=auction.winning_price_cents,
                currency=app.config["STRIPE_CURRENCY"],
                metadata={
                    "auction_id": auction.id,
                    "winner_id": winner.id,
                    "seller_id": seller.id,
                },
            )
            db.session.add(Payment(
                auction_id=auction.id,
                payer_id=winner.id,
                stripe_payment_intent=intent.id,
                amount_cents=auction.winning_price_cents,
                status="pending",
            ))
            db.session.commit()

# --- Outbid notification (called from bidding.place_bid when an auto-bidder loses) ---

@celery.task(bind=True, max_retries=3, default_retry_delay=30)
def send_outbid_email(self, auction_id: int, user_id: int):
    app = _get_app()
    with app.app_context():
        auction = db.session.get(Auction, auction_id)
        user = db.session.get(User, user_id)
        msg = Message(
            subject=f"You've been outbid on {auction.title}",
            recipients=[user.email],
        )
        msg.body = (
            f"Hi {user.name},\n\n"
            f"Someone outbid you on {auction.title}.\n"
            f"Current bid: ${auction.current_bid_cents / 100:.2f}\n"
            f"Place another bid: {app.config['SITE_URL']}/auction/{auction.slug}\n"
        )
        try:
            mail.send(msg)
        except Exception as exc:
            raise self.retry(exc=exc)
```

### Step 12 — Account blueprint (my bids, my wins, pay)

```python
# app/account.py
from flask import Blueprint, render_template, request, redirect, url_for, flash, current_app
from flask_login import login_required, current_user
from sqlalchemy import select
import stripe

from app.models import Auction, Bid, Payment, User
from app.extensions import db

account_bp = Blueprint("account", __name__)

@account_bp.route("/bids")
@login_required
def my_bids():
    stmt = (
        select(Auction)
        .join(Bid)
        .where(Bid.bidder_id == current_user.id)
        .distinct()
        .order_by(Auction.ends_at.desc())
    )
    auctions = db.session.execute(stmt).scalars().all()
    return render_template("account/bids.html", auctions=auctions)

@account_bp.route("/wins")
@login_required
def my_wins():
    stmt = (
        select(Auction)
        .where(Auction.winner_id == current_user.id)
        .order_by(Auction.ends_at.desc())
    )
    auctions = db.session.execute(stmt).scalars().all()
    return render_template("account/wins.html", auctions=auctions)

@account_bp.route("/pay/<int:auction_id>", methods=["GET", "POST"])
@login_required
def pay(auction_id: int):
    auction = db.session.get(Auction, auction_id)
    if not auction or auction.winner_id != current_user.id:
        return "Forbidden", 403
    payment = db.session.execute(
        select(Payment).where(Payment.auction_id == auction_id)
    ).scalar_one_or_none()
    if not payment:
        return "No payment due", 400

    if request.method == "POST":
        # Stripe.js confirms the payment client-side; we just hand back the client_secret
        pass

    stripe.api_key = current_app.config["STRIPE_SECRET_KEY"]
    intent = stripe.PaymentIntent.retrieve(payment.stripe_payment_intent)
    return render_template(
        "account/pay.html", auction=auction, payment=payment,
        client_secret=intent.client_secret,
        publishable_key=current_app.config["STRIPE_PUBLISHABLE_KEY"],
    )
```

### Step 13 — Seller blueprint (create auctions, view bids)

```python
# app/seller.py
from datetime import datetime, timezone, timedelta
from flask import Blueprint, render_template, request, redirect, url_for, flash
from flask_login import login_required, current_user
from sqlalchemy import select
from slugify import slugify

from app.models import Auction, Category, Bid
from app.extensions import db

seller_bp = Blueprint("seller", __name__)

@seller_bp.before_request
def require_seller():
    if not (current_user.is_authenticated and current_user.is_seller):
        return "Forbidden", 403

@seller_bp.route("/")
@login_required
def dashboard():
    stmt = (
        select(Auction)
        .where(Auction.seller_id == current_user.id)
        .order_by(Auction.ends_at.desc())
    )
    auctions = db.session.execute(stmt).scalars().all()
    return render_template("seller/dashboard.html", auctions=auctions)

@seller_bp.route("/new", methods=["GET", "POST"])
@login_required
def new_auction():
    if request.method == "POST":
        ends_at_str = request.form["ends_at"]   # ISO datetime-local
        ends_at = datetime.fromisoformat(ends_at_str).astimezone(timezone.utc)
        auction = Auction(
            title=request.form["title"],
            slug=slugify(request.form["title"]) + "-" + str(int(datetime.now().timestamp()))[-5:],
            description=request.form["description"],
            start_price_cents=int(float(request.form["start_price"]) * 100),
            reserve_price_cents=(
                int(float(request.form["reserve_price"]) * 100)
                if request.form.get("reserve_price") else None
            ),
            buy_now_price_cents=(
                int(float(request.form["buy_now_price"]) * 100)
                if request.form.get("buy_now_price") else None
            ),
            increment_cents=int(float(request.form.get("increment", "1.00")) * 100),
            starts_at=datetime.fromisoformat(request.form["starts_at"]).astimezone(timezone.utc),
            ends_at=ends_at,
            status="active",
            seller_id=current_user.id,
        )
        if request.form.get("category_id"):
            auction.category_id = int(request.form["category_id"])
        db.session.add(auction); db.session.commit()
        flash("Auction created", "success")
        return redirect(url_for("auctions.detail", slug=auction.slug))

    categories = db.session.execute(select(Category)).scalars().all()
    return render_template("seller/form.html", categories=categories)
```

### Step 14 — Stripe webhook

```python
# app/webhooks.py
import stripe
from flask import Blueprint, request, current_app
from sqlalchemy import select
from app.extensions import db
from app.models import Payment, Auction

webhooks_bp = Blueprint("webhooks", __name__)

@webhooks_bp.route("/stripe", methods=["POST"])
def stripe_webhook():
    payload = request.get_data(as_text=True)
    sig = request.headers.get("Stripe-Signature", "")
    try:
        event = stripe.Webhook.construct_event(
            payload, sig, current_app.config["STRIPE_WEBHOOK_SECRET"]
        )
    except (ValueError, stripe.error.SignatureVerificationError):
        return "Bad signature", 400

    if event["type"] != "payment_intent.succeeded":
        return "", 200

    intent = event["data"]["object"]
    payment = db.session.execute(
        select(Payment).where(Payment.stripe_payment_intent == intent.id)
    ).scalar_one_or_none()
    if not payment or payment.status == "succeeded":
        return "", 200  # idempotent

    payment.status = "succeeded"
    auction = db.session.get(Auction, payment.auction_id)
    if auction:
        auction.status = "paid"
        db.session.add(auction)
    db.session.add(payment); db.session.commit()
    return "", 200
```

### Step 15 — Docker Compose

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: gavel
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  web:
    build: .
    command: gunicorn -k eventlet -w 1 -b 0.0.0.0:5000 wsgi:app
    environment:
      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/gavel
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
      CELERY_RESULT_BACKEND: redis://redis:6379/2
      SECRET_KEY: dev-secret-change-me
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
      STRIPE_PUBLISHABLE_KEY: ${STRIPE_PUBLISHABLE_KEY}
      STRIPE_WEBHOOK_SECRET: ${STRIPE_WEBHOOK_SECRET}
      MAIL_SERVER: smtp.mailtrap.io
      MAIL_USERNAME: ${MAILTRAP_USER}
      MAIL_PASSWORD: ${MAILTRAP_PASS}
    ports: ["5000:5000"]
    depends_on: [postgres, redis]

  worker:
    build: .
    command: celery -A celery_worker.celery worker --loglevel=info --concurrency=4
    environment:
      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/gavel
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
      CELERY_RESULT_BACKEND: redis://redis:6379/2
      SECRET_KEY: dev-secret-change-me
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
      STRIPE_PUBLISHABLE_KEY: ${STRIPE_PUBLISHABLE_KEY}
      STRIPE_WEBHOOK_SECRET: ${STRIPE_WEBHOOK_SECRET}
      MAIL_SERVER: smtp.mailtrap.io
      MAIL_USERNAME: ${MAILTRAP_USER}
      MAIL_PASSWORD: ${MAILTRAP_PASS}
    depends_on: [postgres, redis]

  beat:
    build: .
    command: celery -A celery_worker.celery beat --loglevel=info
    environment:
      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/gavel
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
      CELERY_RESULT_BACKEND: redis://redis:6379/2
      SECRET_KEY: dev-secret-change-me
    depends_on: [redis]

  stripe-cli:
    image: stripe/stripe-cli:latest
    command: listen --forward-to web:5000/webhooks/stripe
    environment:
      STRIPE_API_KEY: ${STRIPE_SECRET_KEY}
    depends_on: [web]

volumes:
  pgdata:
```

```python
# celery_worker.py
from app import create_app
from app.extensions import celery
app = create_app()
celery_app = celery
```

```python
# wsgi.py
from app import create_app
app = create_app()
if __name__ == "__main__":
    from app.extensions import socketio
    socketio.run(app, host="0.0.0.0", port=5000)
```

---

## 7. Running the Project

```bash
cp .env.example .env
# Edit .env: set STRIPE_SECRET_KEY, STRIPE_PUBLISHABLE_KEY, STRIPE_WEBHOOK_SECRET, MAILTRAP_*

docker compose up --build
docker compose exec web flask --app wsgi db upgrade

# Create a seller + a buyer
docker compose exec web python -c "
from app import create_app
from app.extensions import db
from app.models import User
app = create_app()
with app.app_context():
    s = User(email='seller@gavel.local', name='Seller', is_seller=True); s.set_password('x')
    b = User(email='buyer@gavel.local', name='Buyer'); b.set_password('x')
    db.session.add_all([s, b]); db.session.commit()
    print('Created seller+buyer')
"
```

Open `http://localhost:5000`, log in as seller, click "New Auction" — set a start price of $10, end time 5 minutes from now, reserve $50. Log out, log in as buyer in two browsers (one incognito), open the auction page in both — both countdowns tick. Place a bid in browser 1 ($11): browser 2's current bid updates within ~50 ms. Place a bid in browser 2 ($15): browser 1 shows "you've been outbid." Enable auto-bid in browser 1 with max $100; place a manual bid of $20 in browser 2: browser 1's auto-bid fires, raising to $21 immediately. When the timer hits zero:

1. Within 60 seconds, the Celery beat task fires `settle_ended_auctions`.
2. `settle_auction` runs: determines the winner, marks auction `sold`, sends winner + seller emails.
3. A Stripe PaymentIntent is created; the winner gets an email with `/account/pay/<auction_id>` link.
4. The winner pays via Stripe.js; the webhook marks the auction `paid`.

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
```

```python
# tests/test_bidding.py
"""Critical: concurrent bids must never produce inconsistent state."""
import threading
from app.models import Auction, User, Bid
from app.extensions import db
from app.bidding import place_bid

def test_concurrent_bids_never_oversell(app):
    with app.app_context():
        seller = User(email="s@t", name="S", is_seller=True); seller.set_password("x")
        bidders = []
        for i in range(20):
            b = User(email=f"b{i}@t", name=f"B{i}"); b.set_password("x")
            bidders.append(b)
        db.session.add_all([seller, *bidders])
        from datetime import datetime, timezone, timedelta
        auction = Auction(
            title="T", slug="t", description="x",
            start_price_cents=1000, increment_cents=100,
            starts_at=datetime.now(timezone.utc) - timedelta(minutes=1),
            ends_at=datetime.now(timezone.utc) + timedelta(hours=1),
            status="active", seller_id=seller.id,
        )
        db.session.add(auction); db.session.commit()
        auction_id = auction.id

        results = []
        def bid(user_id, amount):
            with app.test_request_context():
                u = db.session.get(User, user_id)
                r = place_bid(auction_id, u, amount)
                results.append((amount, r.ok, r.new_current_bid))

        # All 20 bid different amounts simultaneously
        threads = [
            threading.Thread(target=bid, args=(bidders[i].id, 1100 + i * 100))
            for i in range(20)
        ]
        for t in threads: t.start()
        for t in threads: t.join()

        # The final current_bid must equal the highest winning amount
        a = db.session.get(Auction, auction_id)
        assert a.current_bid_cents == max(r[2] for r in results if r[1])
        # Exactly one bid should be the current high
        assert a.current_high_bidder_id is not None
        # All 20 bids should have produced a result (some ok=False, some ok=True)
        assert len(results) == 20

def test_below_minimum_bid_rejected(app):
    # ... bid $5 when minimum is $11, assert ok=False, reason="below_minimum"
    pass

def test_auto_bid_caps_at_max(app):
    # ... set auto-bid max $100; another manual bid at $50 should result
    # in current_bid = $51 (max + 1 increment), not $100
    pass

def test_cannot_bid_on_own_auction(app):
    # ... seller attempts to bid on own auction, assert ok=False
    pass
```

```python
# tests/test_settlement.py
def test_auction_no_sale_when_reserve_not_met(app):
    # ... start auction at $10, reserve $100, highest bid $50
    # run settle_auction, assert status="ended" (not "sold"), winner_id=None
    pass

def test_auction_sold_when_reserve_met(app):
    # ... highest bid $120 >= reserve $100
    # run settle_auction, assert status="sold", winner_id set, Payment created
    pass
```

Run:

```bash
pytest -q
```

### End-to-end smoke checklist

| # | Action | Expected |
|---|---|---|
| 1 | `GET /` | Active auctions render |
| 2 | Open auction detail in 2 browsers | Both countdowns tick synchronously |
| 3 | Place bid $11 in browser 1 | Browser 2 sees $11 within ~100 ms |
| 4 | Place bid $15 in browser 2 | Browser 1 sees "outbid" banner |
| 5 | Enable auto-bid (max $100) in browser 1 | Stored, no immediate effect |
| 6 | Place manual bid $20 in browser 2 | Browser 1's auto-bid raises to $21 instantly |
| 7 | Click "Buy Now $200" | Auction ends immediately, settlement email queued |
| 8 | Wait for auction end (timer reaches 0) | Within 60s, status → sold, winner + seller emails arrive |
| 9 | Winner opens `/account/pay/<id>` | Stripe payment form renders |
| 10 | Pay with `4242...` test card | Webhook fires, status → paid |
| 11 | Spam 20 bids in 1 minute | After 10, rate limiter returns 429 |

---

## 9. Deployment

### Production checklist

- [ ] **gunicorn** with `eventlet` worker (SocketIO needs it). One worker; scale horizontally behind a sticky-session load balancer (or use the Redis message queue for SocketIO).
- [ ] **PostgreSQL**: managed RDS / Cloud SQL with `pool_size=20` for the bid lock contention.
- [ ] **Redis**: managed ElastiCache / Memorystore. SocketIO uses it as a message queue when scaling beyond one web worker.
- [ ] **Stripe webhook endpoint** registered in the Stripe dashboard.
- [ ] **CSRF** enabled everywhere except `/webhooks/stripe`.
- [ ] **Rate limits** tuned: 10 bids/minute per user, 5 auctions/hour per seller.
- [ ] **Secrets** in Doppler / Vault.
- [ ] **Celery worker + beat** as separate services. Beat must be a singleton (one process only) — running two beats will double-fire scheduled tasks.
- [ ] **HTTPS only** — HSTS, `Secure` cookies.
- [ ] **Sticky sessions** at the load balancer for SocketIO (or configure the Redis adapter so any worker can emit to any client).

### Fly.io example (web)

```toml
app = "gavel-web"
[deploy]
  strategy = "rolling"
[http_service]
  internal_port = 5000
  auto_stop_machines = false        # SocketIO needs sticky sessions
  auto_start_machines = true
  min_machines_running = 1
```

For the worker:

```toml
app = "gavel-worker"
[experimental]
  cmd = ["celery", "-A", "celery_worker.celery", "worker", "--loglevel=info", "--concurrency=4"]
```

For beat (singleton):

```toml
app = "gavel-beat"
[experimental]
  cmd = ["celery", "-A", "celery_worker.celery", "beat", "--loglevel=info"]
```

```mermaid
flowchart LR
    subgraph Browser
        UI[Tailwind + HTMX] -->|hx-post /bids/place| BidRoute[bids.place]
        UI -.->|Socket.IO| WSRoom[auction-123 room]
    end
    subgraph Flask
        BidRoute --> Lock[SELECT FOR UPDATE]
        Lock --> BiddingLogic[place_bid<br/>proxy bidding]
        BiddingLogic --> DB[(PostgreSQL<br/>auctions + bids)]
        BiddingLogic --> Emit[socketio.emit]
        Emit -.-> WSRoom
    end
    subgraph Async
        Beat[Celery beat<br/>every 1 sec] --> Countdown[broadcast_countdowns]
        Beat -->|every 1 min| Settle[settle_ended_auctions]
        Settle --> SettleTask[settle_auction per auction]
        SettleTask --> Stripe[Stripe PaymentIntent]
        SettleTask --> SMTP[SMTP winner + seller]
    end
    WSRoom -.->|tick| UI
    WSRoom -.->|bid_update| UI
    WSRoom -.->|auction_ended| UI
    style Lock fill:#fef3c7,stroke:#d97706
    style Emit fill:#fce7f3,stroke:#db2777
    style Stripe fill:#dbeafe,stroke:#2563eb
```

---

## 10. Next Steps / Enhancements

1. **Anti-snipe (popcorn bidding).** If a bid is placed in the last 10 seconds, extend the auction end by 30 seconds. Implement by checking `ends_at - now < 10s` inside the bid transaction and updating `ends_at`.
2. **Watch list.** Users bookmark auctions they want to track; a Celery beat job emails them 1 hour before the auction ends.
3. **Image gallery.** Multiple images per auction (one-to-many); upload via HTMX with progress bar.
4. **Seller ratings.** After payment, the winner rates the seller 1–5 stars; aggregate on the seller's profile.
5. **Stripe Connect for marketplace.** Each seller connects their Stripe account; the platform takes a 5% fee and routes the rest to the seller via `transfer_data`.
6. **Live bidding video stream.** For high-end auctions, integrate a WebRTC stream so buyers can see the item in real time.
7. **Currency conversion.** Buyers see prices in their local currency (daily FX rate via Celery beat).
8. **Bid analytics dashboard.** A seller-side chart showing bid volume over time, unique bidders, and price trajectory — reuse the patterns from [[Project-05-Live-Analytics-Dashboard]].

---

## 11. What We Learned

- **`SELECT FOR UPDATE` is the right primitive for hot rows.** Optimistic locking's abort-retry loop is wasteful when 50 bids hit the same auction in the last 30 seconds. Pessimistic locking serializes them — each bid waits ~5 ms but always commits on the first try.
- **Server-authoritative time eliminates clock drift bugs.** Clients render a *display* countdown from `ends_at - Date.now()`, but the actual "ended" event comes from the Celery beat task. A user with a misconfigured clock never gets a stale "you can still bid!" UI.
- **Proxy bidding is a tiny rules engine.** The Vickrey-style increment logic fits in ~60 lines: if your max exceeds the prior high bidder's max, you win at `prior_max + increment`; otherwise, the prior bidder still wins at `your_max + increment`. Spend the time to write tests for every combination.
- **Flask-SocketIO rooms are the simplest fan-out pattern.** One room per auction, one `emit(..., room=...)` per state change. Scale beyond one worker with the Redis adapter (`socketio.init_app(app, message_queue=app.config['REDIS_URL'])`).
- **Rate limiting via Redis** (Flask-Limiter) prevents bid spam — a critical safety net when an attacker could probe auto-bid limits by spamming incremental bids. 10 bids/minute per user is generous for humans and tight enough for bots.
- **Idempotent Stripe webhooks** are non-negotiable. Stripe will redeliver for 3 days; checking `payment.status == "succeeded"` before processing prevents double-marking the auction as paid.
- **Settlement as a Celery task, not a request handler.** When the auction ends, the user who placed the last bid isn't necessarily online. The beat task finds ended auctions and processes them; the winner gets an email with a payment link.
- **HTMX 2.0 keeps the bid form server-rendered.** `hx-post` returns the updated bid panel HTML; the WebSocket handles everyone else's updates. No React, no SPA build, no state sync bugs.
- **Concurrency tests with `threading` are essential.** A unit test that bids once will pass even with a broken lock. A test that fires 20 concurrent bids at one auction catches the oversell bug immediately.
- **`with_for_update()` requires a real database.** SQLite ignores row locks silently — your concurrency tests must run against PostgreSQL (use `testcontainers-python` or a Docker service in CI).

---

```mermaid
flowchart TB
    subgraph Real-time Layer
        SocketIO[Flask-SocketIO<br/>room per auction]
        Beat1[Celery beat 1s<br/>countdown broadcast]
    end
    subgraph Persistence Layer
        PG[(PostgreSQL<br/>SELECT FOR UPDATE)]
        Stripe[Stripe<br/>PaymentIntents]
    end
    subgraph Async Layer
        Beat2[Celery beat 1m<br/>find ended auctions]
        Worker[Celery worker<br/>settle + email]
    end
    Browser[Browser<br/>HTMX + Tailwind] -->|HTTP POST /bid| Flask[Flask routes]
    Flask <--> SocketIO
    Flask <--> PG
    SocketIO <--> Browser
    Beat1 -->|emit tick| SocketIO
    Flask -->|delay| Worker
    Beat2 -->|find ended| Worker
    Worker --> Stripe
    Worker --> SMTP[SMTP]
    Stripe -->|webhook| Flask
    style PG fill:#dcfce7,stroke:#16a34a
    style Stripe fill:#fce7f3,stroke:#db2777
    style SocketIO fill:#dbeafe,stroke:#2563eb
```

> [!quote] A note from your teacher
> Real-time + money is the hardest combination in web engineering — and Gavel is the project that proves you can do it. The patterns here (`SELECT FOR UPDATE`, server-authoritative time, idempotent webhooks, proxy bidding) are the exact same patterns eBay, Sotheby's, and StockX use in production. Build this project well, deploy it, and put a live demo link on your résumé with a YouTube video of two browsers bidding against each other in real time. That single demo is worth more than any certification. Pair Gavel with [[Project-10-Event-Booking-Platform]] to see how the same concurrency primitives (atomic operations, Redis, Celery beat) solve two completely different domain problems — ticket inventory and auction settlement. Then go build the next thing. You're ready.
