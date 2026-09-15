---
title: "Project 10 — Event Booking Platform (Reserve)"
module: 16-Modern-Projects
project_number: 10
estimated_time: "12–16 hours build, 40–55 hours to perfect"
difficulty: intermediate
prerequisites:
  - "Complete the core course (Lessons 01–24)"
  - "[[Lesson-16-User-Model-AndPasswords]] — password hashing, the User model"
  - "[[Lesson-17-Flask-Login]] — sessions, login_required"
  - "[[Lesson-12-SQLAlchemy-Setup]] and [[Lesson-13-CRUD-Operations]] — models & CRUD"
  - "[[Lesson-14-Relationships]] — one-to-many (Event → TicketType → Booking → Ticket)"
  - "[[Lesson-18-Blueprints]] — splitting events, bookings, organizer blueprints"
  - "[[Lesson-22-Email-And-Tasks]] — Celery + Flask-Mail for ticket emails + reminders"
  - "[[Modern-Frontend-Integration]] — HTMX, Tailwind 4, Socket.IO"
  - "[[Modern-Databases-And-Caching]] — Redis atomic ops, PostgreSQL"
  - "[[Project-04-Modern-Ecommerce]] — Stripe webhook patterns reused"
tags:
  - flask
  - flask3
  - python313
  - project
  - portfolio
  - events
  - booking
  - stripe
  - redis
  - celery
  - socketio
  - htmx
  - tailwind
  - qrcode
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
  - "[[Project-12-Real-Time-Auction]]"
---

# Project 10 — Event Booking Platform ("Reserve")

> [!success] Time to build the Ticketmaster problem — without the $50 convenience fees.
> By the end of this project you'll have a working event ticketing platform: organizers publish seated or general-admission events, attendees see real-time seat availability (with Redis atomic decrements so two users can never book the same seat), Stripe Checkout for payment, QR-coded PDF tickets emailed via Celery, a 24-hour-before-event reminder beat, and a waitlist for sold-out shows. This is the project that proves you can build **concurrency-safe, real-time, payment-integrated systems** in Flask.

#flask #booking #real-time #redis #stripe #celery #socketio #qrcode #htmx #tailwind #portfolio #2026

---

## 1. Project Overview

**Reserve** is an event ticketing platform built with **Flask 3.2 + SQLAlchemy 2.0 + PostgreSQL 16 + Redis 7 + Stripe + Celery + Flask-SocketIO + HTMX 2.0**. Organizers create events with capacity limits and tiered pricing. Attendees browse events, see live seat maps (for seated shows) or general-admission counters, hold seats for 10 minutes while they check out, pay via Stripe, and receive a QR-coded ticket by email. Twenty-four hours before each event, a Celery beat job fires a reminder email. If a show sells out, hopeful attendees join a waitlist and get notified automatically when seats free up.

### Why this is a modern 2026 project

A 2018 "Flask ticket app" tutorial had three routes, a SQLite database, and a `tickets_sold` integer that two users could increment simultaneously — resulting in overselling. Modern event platforms in 2026 look completely different:

1. **Real-time seat maps over WebSocket.** When user A clicks seat 12A, every other browser viewing the event sees it turn yellow within ~50 ms. No polling. No React.
2. **Atomic inventory via Redis Lua scripts.** A single Redis operation either reserves N seats or returns "not enough" — there's no TOCTOU window where two users both think they got the last seat.
3. **Time-limited holds with Redis TTL.** Seats return to the pool automatically 10 minutes after a hold if the user abandons checkout. No cron job scanning the database.
4. **Stripe PaymentIntents, not raw card forms.** PCI compliance is delegated to Stripe. Webhooks — not the redirect URL — are the source of truth for "did this booking actually get paid?"
5. **QR-coded tickets, signed and unique per booking.** Each ticket has a JWT-signed QR code; the day-of-event check-in page scans the QR, verifies the signature, and marks the attendee as arrived.
6. **Celery beat for scheduled reminders.** A 24-hour-before-event email is queued by a periodic task, not by a request handler.

> [!info] Where this fits in the project series
> Project 04 ([[Project-04-Modern-Ecommerce]]) taught you Stripe webhooks, Celery tasks, and Redis-backed carts. **Reserve reuses those patterns** but adds the *real-time + concurrency* dimension: WebSocket broadcasting, atomic seat holds, and time-limited TTLs. Read P04 first — we won't re-explain HMAC webhook signature verification.

### Architecture at a glance

```mermaid
flowchart TB
    U[Attendee<br/>HTMX + Tailwind] -->|select seats| F[Flask App]
    F -->|EVAL Lua hold seats| R[(Redis<br/>holds + availability)]
    F -->|broadcast hold| WS[Flask-SocketIO<br/>room: event-123]
    WS -->|seat map update| U2[Other attendees]
    F -->|Stripe PaymentIntent| ST[Stripe]
    ST -->|webhook payment_intent.succeeded| F
    F -->|INSERT booking + tickets| PG[(PostgreSQL<br/>events / bookings / tickets)]
    F -->|enqueue email| Q[Celery Worker]
    Q -->|SMTP + QR PDF| U
    CB[Celery Beat] -->|24h before event| Q
    Q -->|reminder email| U
    CB -->|every minute| F
    F -->|expire abandoned holds| R
    style F fill:#dbeafe,stroke:#2563eb
    style R fill:#fef3c7,stroke:#d97706
    style ST fill:#fce7f3,stroke:#db2777
    style Q fill:#dcfce7,stroke:#16a34a
```

---

## 2. What You'll Learn

By the end of this project you will have hands-on experience with:

- **Redis Lua scripts for atomic inventory** — the canonical pattern for "decrement N atomically, return the new value, or fail without side effects."
- **Time-limited holds with TTL** — `SET hold:event-123:seat-12A <user> EX 600` gives you a 10-minute reservation that auto-expires; no cleanup cron needed.
- **Flask-SocketIO rooms** — each event is its own broadcast room; subscribers see seat updates only for the event they're viewing.
- **Stripe PaymentIntents + webhooks** — the modern PCI-compliant flow with idempotent booking completion.
- **QR code generation** (`qrcode` + `Pillow`) and **JWT signing** for tamper-proof check-in.
- **Celery beat** — periodic tasks for reminders and for expiring abandoned holds.
- **HTMX 2.0** — `hx-post` for "hold these seats", `hx-swap-oob` for live seat map updates triggered by WebSocket.
- **SQLAlchemy 2.0 `select()` with `with_for_update()`** — pessimistic row locks for the rare case where we need DB-level concurrency control.
- **Docker Compose** — six services (web, worker, beat, postgres, redis, stripe-cli) in one command.

---

## 3. Prerequisites

| Lesson | Why |
|---|---|
| [[Lesson-16-User-Model-AndPasswords]] | `User` model, password hashing. |
| [[Lesson-17-Flask-Login]] | `LoginManager`, `@login_required`. |
| [[Lesson-13-CRUD-Operations]] | `select()`, `db.session.add()`, `db.session.commit()`. |
| [[Lesson-14-Relationships]] | `Event → TicketType → Booking → Ticket`. |
| [[Lesson-18-Blueprints]] | `events`, `bookings`, `organizer`, `account` blueprints. |
| [[Lesson-22-Email-And-Tasks]] | Celery `@app.task`, `Flask-Mail`, Celery beat. |
| [[Modern-Databases-And-Caching]] | Redis patterns, Lua scripts. |
| [[Modern-Frontend-Integration]] | HTMX, Socket.IO, Tailwind 4. |
| [[Project-04-Modern-Ecommerce]] | Stripe webhook + Celery templates. |

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
| Flask-SQLAlchemy | 3.1+ | ORM with 2.0-style `select()`. |
| Flask-Login | 0.6+ | Session auth. |
| Flask-WTF | 1.2+ | CSRF on forms. |
| Flask-Mail | 0.10+ | Transactional email. |
| Flask-SocketIO | 5.4+ | Real-time seat availability. |
| Celery | 5.4+ | Async ticket emails, reminders, beat. |
| Redis | 7 | Seat holds, availability counters, Celery broker. |
| PostgreSQL | 16 | Events, bookings, tickets, payments. |
| stripe (Python SDK) | 11+ | PaymentIntents + webhook signature. |
| `qrcode` + `Pillow` | 7.4+ / 11+ | QR code ticket images. |
| `PyJWT` | 2.9+ | Signed QR payloads (tamper-proof). |
| HTMX | 2.0 (CDN) | Seat selection without SPA. |
| Tailwind CSS | 4 (CDN) | Styling. |
| Docker Compose | 2.30+ | Local dev stack. |

> [!tip] Why Lua scripts instead of `DECR`?
> Redis `DECRBY` is atomic for a *single key*, but a booking often needs to decrement the event's global availability **and** mark specific seats as held **and** record the user — all atomically. A Lua script runs as a single Redis command, so no other client can interleave. This is the same pattern Redis itself uses internally for `MULTI/EXEC`, but with conditional logic that `MULTI` can't express.

---

## 5. Project Structure

```
reserve/
├── app/
│   ├── __init__.py            # create_app() factory + SocketIO init
│   ├── config.py              # Config classes
│   ├── extensions.py          # db, login_manager, mail, socketio, celery, redis
│   ├── models.py              # User, Event, TicketType, Booking, Ticket, Payment, Refund
│   ├── inventory.py           # Redis Lua scripts: hold_seats, release_seats, confirm_seats
│   ├── sockets.py             # SocketIO: join event room, broadcast seat holds
│   ├── events.py              # Blueprint: list, detail, search
│   ├── bookings.py            # Blueprint: hold, checkout, success, cancel
│   ├── account.py             # Blueprint: my tickets, upcoming events
│   ├── organizer.py           # Blueprint: create events, view bookings, scan QR
│   ├── tasks.py               # Celery: send_tickets, send_reminder, expire_holds
│   ├── tickets.py             # QR generation + JWT signing
│   ├── templates/
│   │   ├── base.html          # Tailwind + HTMX + Socket.IO client
│   │   ├── events/{list,detail}.html
│   │   ├── bookings/{cart,checkout,success}.html
│   │   ├── account/{tickets,event}.html
│   │   └── organizer/{dashboard,event_form,scan}.html
│   └── static/
│       └── tickets/           # Generated QR PNGs (cached)
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

We'll build this in **14 steps**. Every step produces a runnable checkpoint — if you get lost, `git diff` against the prior step.

### Step 1 — Install dependencies

```bash
# requirements.txt
Flask==3.2.0
Flask-SQLAlchemy==3.1.1
Flask-Login==0.6.3
Flask-WTF==1.2.1
Flask-Mail==0.10.0
Flask-SocketIO==5.4.1
SQLAlchemy==2.0.36
celery==5.4.0
redis==5.2.0
stripe==11.3.0
qrcode==7.4.2
Pillow==11.0.0
PyJWT==2.9.0
psycopg[binary]==3.2.3
python-dotenv==1.0.1
eventlet==0.37.0
gunicorn==23.0.0
pytest==8.3.4
```

Install:

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

### Step 2 — Configuration

```python
# app/config.py
import os
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

class Config:
    SECRET_KEY = os.environ["SECRET_KEY"]
    SQLALCHEMY_DATABASE_URI = os.environ["DATABASE_URL"]
    SQLALCHEMY_ENGINE_OPTIONS = {"pool_pre_ping": True, "pool_size": 10}
    REDIS_URL = os.environ.get("REDIS_URL", "redis://localhost:6379/0")

    # Stripe
    STRIPE_SECRET_KEY = os.environ["STRIPE_SECRET_KEY"]
    STRIPE_WEBHOOK_SECRET = os.environ["STRIPE_WEBHOOK_SECRET"]
    STRIPE_CURRENCY = "usd"

    # Mail (Mailtrap in dev, SES in prod)
    MAIL_SERVER = os.environ.get("MAIL_SERVER", "smtp.mailtrap.io")
    MAIL_PORT = int(os.environ.get("MAIL_PORT", 587))
    MAIL_USERNAME = os.environ.get("MAIL_USERNAME")
    MAIL_PASSWORD = os.environ.get("MAIL_PASSWORD")
    MAIL_USE_TLS = True

    # Booking
    HOLD_TTL_SECONDS = 600          # 10 minutes
    QR_SECRET = os.environ["QR_SECRET"]   # for signing QR JWTs

    # Celery
    CELERY_BROKER_URL = os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/1")
    CELERY_RESULT_BACKEND = os.environ.get("CELERY_RESULT_BACKEND", "redis://localhost:6379/2")

    # Rate limits
    RATELIMIT_STORAGE_URI = os.environ.get("REDIS_URL", "redis://localhost:6379/0")
    RATELIMIT_DEFAULT = "200/hour"

class TestConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"
    RATELIMIT_ENABLED = False
    WTF_CSRF_ENABLED = False
```

> [!warning] Don't commit `.env`
> Use `.env.example` as a template. Real secrets live in your deployment secret manager (Vault, AWS Secrets Manager, Doppler). See [[Modern-Deployment-And-DevOps]].

### Step 3 — Extensions + factory

```python
# app/extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager
from flask_wtf import CSRFProtect
from flask_mail import Mail
from flask_socketio import SocketIO
from celery import Celery
import redis

db = SQLAlchemy()
login_manager = LoginManager()
csrf = CSRFProtect()
mail = Mail()
socketio = SocketIO(cors_allowed_origins="*", async_mode="eventlet")
celery = Celery("reserve")

# Shared Redis client (decoded responses for easier Python use)
def make_redis(app):
    return redis.Redis.from_url(app.config["REDIS_URL"], decode_responses=True)

# We'll attach `app.redis` inside create_app so tasks can use current_app.redis.
```

```python
# app/__init__.py
from flask import Flask
from app.config import Config
from app.extensions import db, login_manager, csrf, mail, socketio, celery, make_redis

def create_app(config_class=Config) -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_class)

    db.init_app(app)
    login_manager.init_app(app)
    csrf.init_app(app)
    mail.init_app(app)
    socketio.init_app(app, async_mode="eventlet")

    app.redis = make_redis(app)

    # Celery config
    celery.conf.update(
        broker_url=app.config["CELERY_BROKER_URL"],
        result_backend=app.config["CELERY_RESULT_BACKEND"],
        task_serializer="json",
        accept_content=["json"],
        task_track_started=True,
        beat_schedule={
            # 24h-before-event reminders
            "send-reminders-every-hour": {
                "task": "app.tasks.send_reminders",
                "schedule": 3600.0,            # every hour
            },
            # Expire abandoned holds (belt-and-suspenders; TTL handles it too)
            "expire-holds-every-minute": {
                "task": "app.tasks.cleanup_holds",
                "schedule": 60.0,
            },
        },
    )

    # User loader
    from app.models import User
    @login_manager.user_loader
    def load_user(uid):
        return db.session.get(User, int(uid))

    # Register blueprints
    from app.events import events_bp
    from app.bookings import bookings_bp
    from app.account import account_bp
    from app.organizer import organizer_bp
    app.register_blueprint(events_bp)
    app.register_blueprint(bookings_bp, url_prefix="/bookings")
    app.register_blueprint(account_bp, url_prefix="/account")
    app.register_blueprint(organizer_bp, url_prefix="/organizer")

    # SocketIO handlers
    from app import sockets  # noqa: F401 (registers handlers)

    return app
```

> [!info] Why a factory, not module-level `app = Flask(__name__)`?
> The factory lets you create multiple apps in one process — critical for tests, Celery tasks, and CLI scripts. See [[Lesson-18-Blueprints]].

### Step 4 — Models

```python
# app/models.py
from datetime import datetime, timezone
from flask_login import UserMixin
from sqlalchemy import (
    Integer, String, Text, DateTime, ForeignKey, Boolean, UniqueConstraint, Index
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
    is_organizer: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    bookings: Mapped[list["Booking"]] = relationship(back_populates="user")
    events: Mapped[list["Event"]] = relationship(back_populates="organizer")
    waitlist_entries: Mapped[list["Waitlist"]] = relationship(back_populates="user")

    def set_password(self, pw: str) -> None:
        self.password_hash = generate_password_hash(pw)
    def check_password(self, pw: str) -> bool:
        return check_password_hash(self.password_hash, pw)

class Event(db.Model):
    __tablename__ = "events"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200), index=True)
    slug: Mapped[str] = mapped_column(String(200), unique=True, index=True)
    description: Mapped[str] = mapped_column(Text)
    starts_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), index=True)
    ends_at: Mapped[datetime] = mapped_column(DateTime(timezone=True))
    venue_name: Mapped[str] = mapped_column(String(200))
    venue_address: Mapped[str] = mapped_column(String(300))
    is_seated: Mapped[bool] = mapped_column(Boolean, default=False)  # True = seat map
    is_published: Mapped[bool] = mapped_column(Boolean, default=True)
    organizer_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    organizer: Mapped[User] = relationship(back_populates="events")
    ticket_types: Mapped[list["TicketType"]] = relationship(
        back_populates="event", cascade="all, delete-orphan"
    )
    bookings: Mapped[list["Booking"]] = relationship(back_populates="event")
    waitlist: Mapped[list["Waitlist"]] = relationship(back_populates="event")

    @property
    def is_upcoming(self) -> bool:
        return self.starts_at > utcnow()

class TicketType(db.Model):
    __tablename__ = "ticket_types"
    id: Mapped[int] = mapped_column(primary_key=True)
    event_id: Mapped[int] = mapped_column(ForeignKey("events.id"))
    name: Mapped[str] = mapped_column(String(80))         # "GA", "VIP", "Balcony"
    price_cents: Mapped[int] = mapped_column(Integer)
    capacity: Mapped[int] = mapped_column(Integer)        # total available
    seats_json: Mapped[str | None] = mapped_column(Text, nullable=True)
    # For seated events: JSON list of seat IDs ["A1","A2",...] belonging to this tier

    event: Mapped[Event] = relationship(back_populates="ticket_types")
    tickets: Mapped[list["Ticket"]] = relationship(back_populates="ticket_type")

    @property
    def price(self) -> float:
        return self.price_cents / 100

class Booking(db.Model):
    __tablename__ = "bookings"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    event_id: Mapped[int] = mapped_column(ForeignKey("events.id"))
    ticket_type_id: Mapped[int] = mapped_column(ForeignKey("ticket_types.id"))
    quantity: Mapped[int] = mapped_column(Integer)
    seats_json: Mapped[str | None] = mapped_column(Text, nullable=True)  # seated: list
    total_cents: Mapped[int] = mapped_column(Integer)
    status: Mapped[str] = mapped_column(String(20), default="pending")
    # pending -> paid -> canceled | refunded
    stripe_payment_intent: Mapped[str | None] = mapped_column(String(255), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    paid_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    canceled_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)

    user: Mapped[User] = relationship(back_populates="bookings")
    event: Mapped[Event] = relationship(back_populates="bookings")
    ticket_type: Mapped[TicketType] = relationship(back_populates="bookings")
    tickets: Mapped[list["Ticket"]] = relationship(
        back_populates="booking", cascade="all, delete-orphan"
    )
    payment: Mapped["Payment | None"] = relationship(back_populates="booking")

class Ticket(db.Model):
    __tablename__ = "tickets"
    id: Mapped[int] = mapped_column(primary_key=True)
    booking_id: Mapped[int] = mapped_column(ForeignKey("bookings.id"))
    ticket_type_id: Mapped[int] = mapped_column(ForeignKey("ticket_types.id"))
    seat: Mapped[str | None] = mapped_column(String(20), nullable=True)  # seated only
    qr_token: Mapped[str] = mapped_column(String(500))   # signed JWT
    checked_in: Mapped[bool] = mapped_column(Boolean, default=False)
    checked_in_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )

    booking: Mapped[Booking] = relationship(back_populates="tickets")
    ticket_type: Mapped[TicketType] = relationship()

class Payment(db.Model):
    __tablename__ = "payments"
    id: Mapped[int] = mapped_column(primary_key=True)
    booking_id: Mapped[int] = mapped_column(ForeignKey("bookings.id"))
    stripe_payment_intent: Mapped[str] = mapped_column(String(255), index=True)
    amount_cents: Mapped[int] = mapped_column(Integer)
    status: Mapped[str] = mapped_column(String(20))  # succeeded, failed
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    booking: Mapped[Booking] = relationship(back_populates="payment")

class Refund(db.Model):
    __tablename__ = "refunds"
    id: Mapped[int] = mapped_column(primary_key=True)
    payment_id: Mapped[int] = mapped_column(ForeignKey("payments.id"))
    stripe_refund_id: Mapped[str] = mapped_column(String(255))
    amount_cents: Mapped[int] = mapped_column(Integer)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

class Waitlist(db.Model):
    __tablename__ = "waitlist"
    __table_args__ = (
        UniqueConstraint("event_id", "user_id", name="uq_waitlist_per_user"),
    )
    id: Mapped[int] = mapped_column(primary_key=True)
    event_id: Mapped[int] = mapped_column(ForeignKey("events.id"))
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    quantity: Mapped[int] = mapped_column(Integer)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    event: Mapped[Event] = relationship(back_populates="waitlist")
    user: Mapped[User] = relationship(back_populates="waitlist_entries")
```

### Step 5 — Entity-relationship map

```mermaid
erDiagram
    USER ||--o{ EVENT : organizes
    USER ||--o{ BOOKING : places
    USER ||--o{ WAITLIST : joins
    EVENT ||--o{ TICKET_TYPE : has
    EVENT ||--o{ BOOKING : includes
    EVENT ||--o{ WAITLIST : for
    TICKET_TYPE ||--o{ TICKET : issues
    TICKET_TYPE ||--o{ BOOKING : of_type
    BOOKING ||--|{ TICKET : contains
    BOOKING ||--o| PAYMENT : paid_by
    PAYMENT ||--o{ REFUND : has
    USER {
        int id PK
        string email
        string password_hash
        bool is_organizer
    }
    EVENT {
        int id PK
        string title
        datetime starts_at
        bool is_seated
    }
    TICKET_TYPE {
        int id PK
        int event_id FK
        string name
        int price_cents
        int capacity
    }
    BOOKING {
        int id PK
        int user_id FK
        int event_id FK
        int total_cents
        string status
    }
    TICKET {
        int id PK
        int booking_id FK
        string seat
        string qr_token
        bool checked_in
    }
```

> [!tip] Two inventory systems, one source of truth
> For **general admission** events we track a per-`ticket_type` availability counter in Redis (`avail:event-123:ga`). For **seated** events we additionally track per-seat holds (`hold:event-123:seat-A1`). The database `Booking` row only appears after Stripe confirms payment; Redis is the source of truth during the "hold + checkout" window.

### Step 6 — Redis inventory with Lua scripts

This is the heart of the concurrency story. We need three atomic operations:

1. **`hold_seats`** — try to reserve N seats (general admission) or specific seats (seated). Either all succeed or all fail.
2. **`release_seats`** — return held seats to the pool (used on cancel, or by the TTL cleanup).
3. **`confirm_seats`** — move held seats to "sold" (decrement availability permanently).

```python
# app/inventory.py
"""
Redis-backed seat availability with atomic Lua scripts.

Keys (per event):
  avail:{event_id}:{ticket_type_id}  -> int  (remaining GA seats)
  hold:{event_id}:{ticket_type_id}:{seat} -> user_id  (with TTL)
  holdset:{event_id}:{ticket_type_id} -> SET of held seats (so we can list them)

A Lua script runs as a single atomic command in Redis — no other client
can interleave between the GET and the SET. This is how we prevent
overselling.
"""

# Try to hold N GA seats, or specific seats for a seated event.
# Returns: {ok=1, held={...}} or {ok=0, reason="..."}
HOLD_SCRIPT = """
local event_id = KEYS[1]
local tt_id = KEYS[2]
local user_id = ARGV[1]
local ttl = tonumber(ARGV[2])
local mode = ARGV[3]            -- "ga" or "seated"
local qty = tonumber(ARGV[4])
local seats = ARGV              -- ARGV[5..] = seat IDs (seated only)

local avail_key = "avail:" .. event_id .. ":" .. tt_id
local holdset_key = "holdset:" .. event_id .. ":" .. tt_id

if mode == "ga" then
    local avail = tonumber(redis.call("GET", avail_key) or "0")
    if avail < qty then
        return cjson.encode({ok=0, reason="not_enough", available=avail})
    end
    redis.call("DECRBY", avail_key, qty)
    -- Track the hold so we can release on cancel/TTL
    local hold_key = "hold:" .. event_id .. ":" .. tt_id .. ":" .. user_id
    redis.call("SET", hold_key, tostring(qty), "EX", ttl)
    return cjson.encode({ok=1, held=qty, available=avail - qty})
else
    -- Seated: each seat must be free
    local held = {}
    for i = 5, #ARGV do
        local seat = ARGV[i]
        local seat_key = "hold:" .. event_id .. ":" .. tt_id .. ":" .. seat
        if redis.call("EXISTS", seat_key) == 1 then
            -- Roll back any seats we already held in this call
            for _, s in ipairs(held) do
                local k = "hold:" .. event_id .. ":" .. tt_id .. ":" .. s
                redis.call("DEL", k)
                redis.call("SREM", holdset_key, s)
            end
            return cjson.encode({ok=0, reason="seat_taken", seat=seat})
        end
        redis.call("SET", seat_key, user_id, "EX", ttl)
        redis.call("SADD", holdset_key, seat)
        table.insert(held, seat)
    end
    return cjson.encode({ok=1, held=held})
end
"""

RELEASE_SCRIPT = """
local event_id = KEYS[1]
local tt_id = KEYS[2]
local user_id = ARGV[1]
local mode = ARGV[2]
local qty = tonumber(ARGV[3])
local seats = ARGV             -- ARGV[4..]

local avail_key = "avail:" .. event_id .. ":" .. tt_id
local holdset_key = "holdset:" .. event_id .. ":" .. tt_id

if mode == "ga" then
    redis.call("INCRBY", avail_key, qty)
    redis.call("DEL", "hold:" .. event_id .. ":" .. tt_id .. ":" .. user_id)
else
    for i = 4, #ARGV do
        local seat = ARGV[i]
        redis.call("DEL", "hold:" .. event_id .. ":" .. tt_id .. ":" .. seat)
        redis.call("SREM", holdset_key, seat)
    end
    redis.call("INCRBY", avail_key, #ARGV - 3)
end
return cjson.encode({ok=1})
"""

# Confirm: permanently remove from availability (no INCRBY back).
CONFIRM_SCRIPT = """
local event_id = KEYS[1]
local tt_id = KEYS[2]
local user_id = ARGV[1]
local mode = ARGV[2]
local seats = ARGV             -- ARGV[3..]

local holdset_key = "holdset:" .. event_id .. ":" .. tt_id
if mode == "ga" then
    redis.call("DEL", "hold:" .. event_id .. ":" .. tt_id .. ":" .. user_id)
else
    for i = 3, #ARGV do
        redis.call("DEL", "hold:" .. event_id .. ":" .. tt_id .. ":" .. ARGV[i])
        redis.call("SREM", holdset_key, ARGV[i])
    end
end
return cjson.encode({ok=1})
"""

def register_scripts(redis_client):
    """Register Lua scripts once; returns callables you can invoke."""
    return {
        "hold": redis_client.register_script(HOLD_SCRIPT),
        "release": redis_client.register_script(RELEASE_SCRIPT),
        "confirm": redis_client.register_script(CONFIRM_SCRIPT),
    }

# Helper used everywhere
from flask import current_app
def get_scripts():
    return current_app.config["SCRIPTS"]  # populated in create_app
```

Wire it into the app factory:

```python
# app/__init__.py — add at the end of create_app
from app.inventory import register_scripts
app.config["SCRIPTS"] = register_scripts(app.redis)

# When an event is created, seed availability counters
def seed_availability(event_id: int, ticket_type_id: int, capacity: int):
    from flask import current_app
    key = f"avail:{event_id}:{ticket_type_id}"
    current_app.redis.set(key, capacity)
```

> [!info] Why Lua and not `WATCH/MULTI/EXEC`?
> Optimistic locking with `WATCH` works for simple cases, but when 200 users are simultaneously hitting "Book 2 GA tickets" for a popular show, the abort-retry loop becomes pathological — most transactions fail and retry, killing throughput. A Lua script is *pessimistic* and serial, processing in microseconds.

### Step 7 — SocketIO real-time seat updates

Each event is its own SocketIO room. When a user holds seats, we broadcast to everyone else in the room so their seat maps update.

```python
# app/sockets.py
from flask_socketio import join_room, leave_room, emit
from app.extensions import socketio

@socketio.on("connect")
def on_connect():
    pass  # auth happens via cookie session

@socketio.on("subscribe_event")
def on_subscribe(data):
    """Client emits this when viewing an event detail page."""
    event_id = data["event_id"]
    join_room(f"event-{event_id}")

@socketio.on("unsubscribe_event")
def on_unsubscribe(data):
    leave_room(f"event-{data['event_id']}")

def broadcast_seat_update(event_id: int, ticket_type_id: int, payload: dict):
    """Called from the bookings blueprint after a hold/release/confirm."""
    socketio.emit(
        "seat_update",
        {"ticket_type_id": ticket_type_id, **payload},
        room=f"event-{event_id}",
    )
```

The client-side JavaScript (in `base.html`) listens for `seat_update` and either toggles a seat's CSS class or updates the "N seats remaining" counter. Here's the relevant snippet:

```html
<!-- templates/base.html (script section) -->
<script src="https://cdn.socket.io/4.7/socket.io.min.js"></script>
<script>
  const socket = io();
  const EVENT_ID = document.body.dataset.eventId;
  if (EVENT_ID) {
    socket.emit("subscribe_event", {event_id: parseInt(EVENT_ID)});

    socket.on("seat_update", (data) => {
      // data = {ticket_type_id, mode: "held"|"released"|"sold", seats?: [...], available?: N}
      if (data.mode === "held" && data.seats) {
        data.seats.forEach(seatId => {
          const el = document.querySelector(`[data-seat="${seatId}"]`);
          if (el) el.classList.replace("bg-green-200", "bg-yellow-200");
        });
      } else if (data.mode === "released" && data.seats) {
        data.seats.forEach(seatId => {
          const el = document.querySelector(`[data-seat="${seatId}"]`);
          if (el) el.classList.replace("bg-yellow-200", "bg-green-200");
        });
      } else if (data.available !== undefined) {
        const counter = document.querySelector(
          `[data-tt="${data.ticket_type_id}"] [data-available]`
        );
        if (counter) counter.textContent = data.available;
      }
    });
  }
</script>
```

### Step 8 — Events blueprint (list + detail)

```python
# app/events.py
from flask import Blueprint, render_template, request, current_app
from sqlalchemy import select
from app.models import Event, TicketType
from app.extensions import db

events_bp = Blueprint("events", __name__)

@events_bp.route("/")
def index():
    q = request.args.get("q", "").strip()
    stmt = select(Event).where(Event.is_published == True).order_by(Event.starts_at)
    if q:
        stmt = stmt.where(Event.title.ilike(f"%{q}%"))
    events = db.session.execute(stmt).scalars().all()
    return render_template("events/list.html", events=events, q=q)

@events_bp.route("/event/<slug>")
def detail(slug: str):
    event = db.session.execute(
        select(Event).where(Event.slug == slug)
    ).scalar_one_or_none()
    if not event:
        return "Not found", 404

    # Build availability map per ticket type (live from Redis)
    avail_map = {}
    held_seats_map = {}
    for tt in event.ticket_types:
        avail = current_app.redis.get(f"avail:{event.id}:{tt.id}")
        avail_map[tt.id] = int(avail) if avail is not None else tt.capacity
        if event.is_seated:
            held = current_app.redis.smembers(f"holdset:{event.id}:{tt.id}")
            held_seats_map[tt.id] = set(held)
    return render_template(
        "events/detail.html",
        event=event,
        avail_map=avail_map,
        held_seats_map=held_seats_map,
    )
```

The detail template renders either a seat map (for seated events) or a quantity picker (for general admission). Each seat button is an HTMX element that calls `/bookings/hold`.

```html
<!-- templates/events/detail.html (excerpt) -->
<body data-event-id="{{ event.id }}">
  <h1>{{ event.title }}</h1>
  <p>{{ event.starts_at }} · {{ event.venue_name }}</p>

  {% for tt in event.ticket_types %}
    <div data-tt="{{ tt.id }}" class="border p-4 my-2">
      <h3>{{ tt.name }} — ${{ '%.2f'|format(tt.price) }}</h3>
      <p><span data-available="{{ avail_map[tt.id] }}">{{ avail_map[tt.id] }}</span> seats left</p>

      {% if event.is_seated %}
        <div class="grid grid-cols-10 gap-2">
          {% for seat in tt.seats_list %}
            <button hx-post="/bookings/hold"
                    hx-vals='{"event_id": "{{ event.id }}", "tt_id": "{{ tt.id }}", "seat": "{{ seat }}"}'
                    hx-swap="outerHTML"
                    data-seat="{{ seat }}"
                    class="p-2 rounded {% if seat in held_seats_map[tt.id] %}bg-yellow-200{% else %}bg-green-200 hover:bg-green-300{% endif %}">
              {{ seat }}
            </button>
          {% endfor %}
        </div>
      {% else %}
        <form hx-post="/bookings/hold" hx-swap="outerHTML">
          <input type="hidden" name="event_id" value="{{ event.id }}">
          <input type="hidden" name="tt_id" value="{{ tt.id }}">
          <input type="number" name="qty" value="1" min="1" max="8">
          <button class="btn btn-primary">Hold 10 minutes</button>
        </form>
      {% endif %}
    </div>
  {% endfor %}
</body>
```

### Step 9 — Bookings blueprint (hold → checkout → success)

```python
# app/bookings.py
import json
from datetime import timedelta
from flask import (
    Blueprint, render_template, request, redirect, url_for,
    flash, current_app, jsonify
)
from flask_login import login_required, current_user
from sqlalchemy import select
import stripe

from app.models import Event, TicketType, Booking, Ticket, Payment
from app.extensions import db
from app.sockets import broadcast_seat_update
from app.tickets import generate_qr_token, render_qr_png

bookings_bp = Blueprint("bookings", __name__)
stripe.api_key = current_app.config["STRIPE_SECRET_KEY"] if current_app else None

@bookings_bp.route("/hold", methods=["POST"])
@login_required
def hold():
    """Hold seats (Redis Lua script). Broadcasts update via SocketIO."""
    event_id = request.form.get("event_id", type=int)
    tt_id = request.form.get("tt_id", type=int)
    tt = db.session.get(TicketType, tt_id)
    if not tt or tt.event_id != event_id:
        return "Bad request", 400

    mode = "seated" if tt.event.is_seated else "ga"
    ttl = current_app.config["HOLD_TTL_SECONDS"]

    if mode == "ga":
        qty = request.form.get("qty", type=int, default=1)
        seats = []
    else:
        seat = request.form["seat"]
        qty = 1
        seats = [seat]

    scripts = current_app.config["SCRIPTS"]
    args = [str(current_user.id), str(ttl), mode, str(qty)] + seats
    result = json.loads(scripts["hold"](keys=[event_id, tt_id], args=args))

    if not result.get("ok"):
        flash(f"Could not hold: {result.get('reason')}", "error")
        return redirect(url_for("events.detail", slug=tt.event.slug))

    # Broadcast the hold to everyone viewing this event
    broadcast_seat_update(event_id, tt_id, {
        "mode": "held",
        "seats": seats,
        "available": result.get("available"),
    })

    # Store pending booking in session so checkout knows what to charge
    pending = {
        "event_id": event_id,
        "tt_id": tt_id,
        "qty": qty,
        "seats": seats,
        "price_cents": tt.price_cents,
    }
    # In a real app, store this in Redis keyed by a UUID returned to the user.
    # For brevity we use Flask session:
    from flask import session
    session["pending_hold"] = pending
    return redirect(url_for("bookings.checkout"))

@bookings_bp.route("/checkout")
@login_required
def checkout():
    from flask import session
    pending = session.get("pending_hold")
    if not pending:
        return redirect(url_for("events.index"))
    event = db.session.get(Event, pending["event_id"])
    tt = db.session.get(TicketType, pending["tt_id"])
    total_cents = pending["qty"] * tt.price_cents
    return render_template(
        "bookings/checkout.html",
        event=event, tt=tt, pending=pending, total_cents=total_cents,
    )

@bookings_bp.route("/pay", methods=["POST"])
@login_required
def pay():
    """Create a Stripe PaymentIntent and a pending Booking row."""
    from flask import session
    pending = session.get("pending_hold")
    if not pending:
        return redirect(url_for("events.index"))

    tt = db.session.get(TicketType, pending["tt_id"])
    total_cents = pending["qty"] * tt.price_cents

    intent = stripe.PaymentIntent.create(
        amount=total_cents,
        currency=current_app.config["STRIPE_CURRENCY"],
        metadata={
            "user_id": current_user.id,
            "event_id": pending["event_id"],
            "tt_id": pending["tt_id"],
            "seats": ",".join(pending["seats"]),
            "qty": pending["qty"],
        },
    )

    booking = Booking(
        user_id=current_user.id,
        event_id=pending["event_id"],
        ticket_type_id=pending["tt_id"],
        quantity=pending["qty"],
        seats_json=json.dumps(pending["seats"]) if pending["seats"] else None,
        total_cents=total_cents,
        status="pending",
        stripe_payment_intent=intent.id,
    )
    db.session.add(booking)
    db.session.commit()

    # Client-side Stripe.js will confirm the card payment with intent.client_secret
    return render_template(
        "bookings/pay.html",
        client_secret=intent.client_secret,
        booking_id=booking.id,
        publishable_key=current_app.config["STRIPE_PUBLISHABLE_KEY"],
    )

@bookings_bp.route("/webhook", methods=["POST"])
def stripe_webhook():
    """Idempotent: if Stripe redelivers, we must not double-create tickets."""
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
    booking = db.session.execute(
        select(Booking).where(Booking.stripe_payment_intent == intent.id)
    ).scalar_one_or_none()

    if not booking or booking.status == "paid":
        # Already processed (idempotent) — return 200
        return "", 200

    # Atomically confirm seats in Redis (permanent removal)
    scripts = current_app.config["SCRIPTS"]
    seats = json.loads(booking.seats_json) if booking.seats_json else []
    mode = "seated" if seats else "ga"
    args = [str(booking.user_id), mode] + seats
    scripts["confirm"](keys=[booking.event_id, booking.ticket_type_id], args=args)

    # Mark booking paid
    from datetime import timezone
    booking.status = "paid"
    booking.paid_at = datetime.now(timezone.utc)
    db.session.add(Payment(
        booking_id=booking.id,
        stripe_payment_intent=intent.id,
        amount_cents=intent.amount,
        status="succeeded",
    ))

    # Generate one Ticket per quantity, each with a unique signed QR
    for _ in range(booking.quantity):
        seat = seats.pop(0) if seats else None
        ticket = Ticket(
            booking_id=booking.id,
            ticket_type_id=booking.ticket_type_id,
            seat=seat,
            qr_token=generate_qr_token(booking.id, seat, current_app.config["QR_SECRET"]),
        )
        db.session.add(ticket)
    db.session.commit()

    # Enqueue the email (async, off the request thread)
    from app.tasks import send_tickets_email
    send_tickets_email.delay(booking.id)

    # Broadcast "sold" to all viewers
    broadcast_seat_update(booking.event_id, booking.ticket_type_id, {
        "mode": "sold",
        "seats": json.loads(booking.seats_json) if booking.seats_json else [],
    })
    return "", 200

# Don't forget the imports at the top of bookings.py
from datetime import datetime
```

```mermaid
sequenceDiagram
    participant U as Attendee
    participant F as Flask
    participant R as Redis (Lua)
    participant WS as SocketIO room
    participant S as Stripe
    participant Q as Celery
    participant DB as PostgreSQL

    U->>F: POST /bookings/hold (seat 12A)
    F->>R: EVAL hold_script
    R-->>F: {ok:1, held:["12A"]}
    F->>WS: broadcast seat_update (held)
    WS-->>U: seat 12A → yellow (live)
    F-->>U: redirect /bookings/checkout
    U->>F: POST /bookings/pay
    F->>S: create PaymentIntent
    S-->>F: client_secret
    F->>DB: INSERT Booking (pending)
    F-->>U: Stripe.js confirm card
    U->>S: confirm card payment
    S->>F: webhook payment_intent.succeeded
    F->>R: EVAL confirm_script
    F->>DB: Booking.status=paid, INSERT tickets
    F->>Q: send_tickets_email.delay(booking_id)
    F->>WS: broadcast seat_update (sold)
    WS-->>U: seat 12A → red (sold)
    Q-->>U: email with QR PNG attachment
```

### Step 10 — QR code generation and signing

```python
# app/tickets.py
import io, json, jwt, qrcode
from datetime import datetime, timezone, timedelta

def generate_qr_token(booking_id: int, seat: str | None, secret: str) -> str:
    """Sign a JWT containing the booking ID, seat, and expiry (event start + 7d).
    The check-in scanner verifies the signature — nobody can forge a ticket."""
    payload = {
        "booking_id": booking_id,
        "seat": seat,
        "iat": datetime.now(timezone.utc),
        "exp": datetime.now(timezone.utc) + timedelta(days=14),
    }
    return jwt.encode(payload, secret, algorithm="HS256")

def render_qr_png(token: str) -> bytes:
    """Render the JWT as a QR code PNG (for email attachment)."""
    buf = io.BytesIO()
    img = qrcode.make(token)
    img.save(buf, format="PNG")
    return buf.getvalue()

def verify_qr_token(token: str, secret: str) -> dict | None:
    try:
        return jwt.decode(token, secret, algorithms=["HS256"])
    except jwt.PyJWTError:
        return None
```

> [!tip] Why JWT and not just `ticket_id`?
> If the QR code contained `ticket_id=42`, anyone could forge a ticket by generating their own QR with `ticket_id=43`. With a signed JWT, the scanner can verify authenticity *offline* (no DB lookup needed for the signature check) and the `exp` claim means stale tickets from last year's event don't scan.

### Step 11 — Celery tasks (tickets, reminders, cleanup)

```python
# app/tasks.py
import json, logging
from datetime import datetime, timezone, timedelta
from celery import shared_task
from flask import current_app
from flask_mail import Message
from sqlalchemy import select

from app.extensions import db, mail, celery
from app.models import Booking, Event, Waitlist, User, Ticket
from app.tickets import render_qr_png
from app.sockets import broadcast_seat_update

log = logging.getLogger(__name__)

def _get_app():
    """Celery tasks don't have an app context; create one."""
    from app import create_app
    app = create_app()
    return app

@celery.task(bind=True, max_retries=3, default_retry_delay=30)
def send_tickets_email(self, booking_id: int):
    app = _get_app()
    with app.app_context():
        booking = db.session.get(Booking, booking_id)
        if not booking or booking.status != "paid":
            return
        msg = Message(
            subject=f"Your tickets for {booking.event.title}",
            recipients=[booking.user.email],
        )
        msg.body = f"Hi {booking.user.name},\n\nYour booking is confirmed. " \
                   f"See attached QR codes for check-in.\n\nEvent: {booking.event.title}\n" \
                   f"When: {booking.event.starts_at}\nWhere: {booking.event.venue_name}\n"
        # Attach each ticket's QR PNG
        for ticket in booking.tickets:
            png = render_qr_png(ticket.qr_token)
            seat_str = f" (seat {ticket.seat})" if ticket.seat else ""
            msg.attach(
                filename=f"ticket-{ticket.id}.png",
                content_type="image/png",
                data=png,
            )
        try:
            mail.send(msg)
        except Exception as exc:
            log.warning("Email send failed: %s", exc)
            raise self.retry(exc=exc)

@celery.task
def send_reminders():
    """Every hour, find events starting in 23–25 hours and email attendees."""
    app = _get_app()
    with app.app_context():
        window_start = datetime.now(timezone.utc) + timedelta(hours=23)
        window_end = datetime.now(timezone.utc) + timedelta(hours=25)
        stmt = select(Event).where(
            Event.starts_at >= window_start,
            Event.starts_at <= window_end,
        )
        events = db.session.execute(stmt).scalars().all()
        for event in events:
            for booking in event.bookings:
                if booking.status != "paid":
                    continue
                msg = Message(
                    subject=f"Reminder: {event.title} tomorrow",
                    recipients=[booking.user.email],
                )
                msg.body = f"Hi {booking.user.name},\n\nJust a reminder: " \
                           f"{event.title} starts at {event.starts_at}.\n" \
                           f"Venue: {event.venue_name}\n\nSee you there!"
                mail.send(msg)

@celery.task
def cleanup_holds():
    """Belt-and-suspenders: any pending Booking older than 12 minutes
    should release its Redis hold. (Normally the TTL handles this, but
    if the user disconnected mid-checkout, we want DB + Redis consistent.)"""
    app = _get_app()
    with app.app_context():
        cutoff = datetime.now(timezone.utc) - timedelta(minutes=12)
        stmt = select(Booking).where(
            Booking.status == "pending",
            Booking.created_at < cutoff,
        )
        stale = db.session.execute(stmt).scalars().all()
        scripts = app.config["SCRIPTS"]
        for booking in stale:
            seats = json.loads(booking.seats_json) if booking.seats_json else []
            mode = "seated" if seats else "ga"
            args = [str(booking.user_id), mode, str(booking.quantity)] + seats
            scripts["release"](keys=[booking.event_id, booking.ticket_type_id], args=args)
            booking.status = "canceled"
            booking.canceled_at = datetime.now(timezone.utc)
            db.session.add(booking)
            # Notify waitlist
            notify_waitlist.delay(booking.event_id, booking.ticket_type_id, booking.quantity)
        db.session.commit()

@celery.task
def notify_waitlist(event_id: int, tt_id: int, qty: int):
    """When seats free up, email the first N users on the waitlist."""
    app = _get_app()
    with app.app_context():
        stmt = (
            select(Waitlist)
            .where(Waitlist.event_id == event_id)
            .order_by(Waitlist.created_at)
            .limit(qty)
        )
        entries = db.session.execute(stmt).scalars().all()
        for entry in entries:
            msg = Message(
                subject="Seats just opened up!",
                recipients=[entry.user.email],
            )
            msg.body = f"Hi {entry.user.name},\n\nSeats became available " \
                       f"for an event on your waitlist. Grab them fast:\n" \
                       f"{app.config.get('SITE_URL', 'http://localhost:5000')}/event/{entry.event.slug}\n"
            mail.send(msg)
```

### Step 12 — Organizer dashboard (events, bookings, QR scan)

```python
# app/organizer.py
from flask import Blueprint, render_template, request, redirect, url_for, flash, current_app
from flask_login import login_required, current_user
from sqlalchemy import select, func
import jwt

from app.models import Event, TicketType, Booking, Ticket
from app.extensions import db
from app.tickets import verify_qr_token

organizer_bp = Blueprint("organizer", __name__)

@organizer_bp.before_request
def require_organizer():
    if not (current_user.is_authenticated and current_user.is_organizer):
        return "Forbidden", 403

@organizer_bp.route("/")
def dashboard():
    stmt = select(Event).where(Event.organizer_id == current_user.id).order_by(Event.starts_at)
    events = db.session.execute(stmt).scalars().all()
    # Sales totals per event
    sales = {}
    for ev in events:
        total = db.session.execute(
            select(func.coalesce(func.sum(Booking.total_cents), 0))
            .where(Booking.event_id == ev.id, Booking.status == "paid")
        ).scalar()
        sales[ev.id] = total / 100
    return render_template("organizer/dashboard.html", events=events, sales=sales)

@organizer_bp.route("/event/new", methods=["GET", "POST"])
def event_form():
    if request.method == "POST":
        # Create event + ticket types, then seed Redis availability counters
        ev = Event(
            title=request.form["title"],
            slug=request.form["slug"],
            description=request.form["description"],
            starts_at=request.form["starts_at"],
            ends_at=request.form["ends_at"],
            venue_name=request.form["venue_name"],
            venue_address=request.form["venue_address"],
            is_seated="is_seated" in request.form,
            organizer_id=current_user.id,
        )
        db.session.add(ev); db.session.commit()
        # Parse ticket types from form (simplified)
        for i in range(int(request.form.get("tt_count", 1))):
            tt = TicketType(
                event_id=ev.id,
                name=request.form[f"tt_{i}_name"],
                price_cents=int(float(request.form[f"tt_{i}_price"]) * 100),
                capacity=int(request.form[f"tt_{i}_capacity"]),
                seats_json=request.form.get(f"tt_{i}_seats", ""),
            )
            db.session.add(tt); db.session.commit()
            # Seed Redis
            current_app.redis.set(f"avail:{ev.id}:{tt.id}", tt.capacity)
        flash("Event created", "success")
        return redirect(url_for("organizer.dashboard"))
    return render_template("organizer/event_form.html")

@organizer_bp.route("/event/<int:event_id>/scan")
def scan_page(event_id):
    ev = db.session.get(Event, event_id)
    return render_template("organizer/scan.html", event=ev)

@organizer_bp.route("/event/<int:event_id>/checkin", methods=["POST"])
def checkin(event_id):
    """Verify a scanned QR token. Idempotent: re-scanning returns 'already checked in'."""
    token = request.json.get("token", "")
    payload = verify_qr_token(token, current_app.config["QR_SECRET"])
    if not payload:
        return {"ok": False, "error": "invalid_token"}, 400

    booking_id = payload["booking_id"]
    booking = db.session.get(Booking, booking_id)
    if not booking or booking.event_id != event_id:
        return {"ok": False, "error": "wrong_event"}, 400

    # Find the ticket with this exact seat (or first ticket if GA)
    seat = payload.get("seat")
    stmt = select(Ticket).where(
        Ticket.booking_id == booking_id,
        Ticket.seat == seat if seat else True,
    )
    ticket = db.session.execute(stmt).scalars().first()
    if not ticket:
        return {"ok": False, "error": "ticket_not_found"}, 404
    if ticket.checked_in:
        return {"ok": True, "already": True, "name": booking.user.name}

    from datetime import datetime, timezone
    ticket.checked_in = True
    ticket.checked_in_at = datetime.now(timezone.utc)
    db.session.add(ticket); db.session.commit()
    return {"ok": True, "name": booking.user.name, "seat": seat}
```

### Step 13 — Account dashboard + refund flow

```python
# app/account.py
import stripe
from datetime import datetime, timezone
from flask import Blueprint, render_template, redirect, url_for, flash, current_app
from flask_login import login_required, current_user
from sqlalchemy import select

from app.models import Booking, Refund, Payment, Ticket
from app.extensions import db
from app.sockets import broadcast_seat_update
import json

account_bp = Blueprint("account", __name__)

@account_bp.route("/tickets")
@login_required
def my_tickets():
    upcoming = db.session.execute(
        select(Booking)
        .where(Booking.user_id == current_user.id, Booking.status == "paid")
        .order_by(Booking.event_id)
    ).scalars().all()
    return render_template("account/tickets.html", bookings=upcoming)

@account_bp.route("/booking/<int:booking_id>/cancel", methods=["POST"])
@login_required
def cancel(booking_id):
    booking = db.session.get(Booking, booking_id)
    if not booking or booking.user_id != current_user.id:
        return "Not found", 404
    if booking.status != "paid":
        return "Cannot cancel", 400

    # Refund via Stripe
    payment = booking.payment
    if not payment:
        return "No payment to refund", 400
    stripe.api_key = current_app.config["STRIPE_SECRET_KEY"]
    refund = stripe.Refund.create(payment_intent=payment.stripe_payment_intent)

    db.session.add(Refund(
        payment_id=payment.id,
        stripe_refund_id=refund.id,
        amount_cents=refund.amount,
    ))

    # Release seats back to the pool
    scripts = current_app.config["SCRIPTS"]
    seats = json.loads(booking.seats_json) if booking.seats_json else []
    mode = "seated" if seats else "ga"
    args = [str(booking.user_id), mode, str(booking.quantity)] + seats
    scripts["release"](keys=[booking.event_id, booking.ticket_type_id], args=args)

    booking.status = "refunded"
    booking.canceled_at = datetime.now(timezone.utc)
    db.session.add(booking); db.session.commit()

    broadcast_seat_update(booking.event_id, booking.ticket_type_id, {
        "mode": "released",
        "seats": seats,
    })
    # Notify waitlist
    from app.tasks import notify_waitlist
    notify_waitlist.delay(booking.event_id, booking.ticket_type_id, booking.quantity)

    flash("Booking refunded", "success")
    return redirect(url_for("account.my_tickets"))
```

### Step 14 — Docker Compose

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: reserve
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    ports: ["6379:6379"]

  web:
    build: .
    command: gunicorn -k eventlet -w 1 -b 0.0.0.0:5000 wsgi:app
    environment:
      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/reserve
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
      CELERY_RESULT_BACKEND: redis://redis:6379/2
      SECRET_KEY: dev-secret-change-me
      QR_SECRET: qr-secret-change-me
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
    command: celery -A celery_worker.celery worker --loglevel=info
    environment:
      # same as web
      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/reserve
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
      CELERY_RESULT_BACKEND: redis://redis:6379/2
      SECRET_KEY: dev-secret-change-me
      QR_SECRET: qr-secret-change-me
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
      # same as worker
      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/reserve
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
      CELERY_RESULT_BACKEND: redis://redis:6379/2
      SECRET_KEY: dev-secret-change-me
      QR_SECRET: qr-secret-change-me
    depends_on: [redis]

  stripe-cli:
    image: stripe/stripe-cli:latest
    command: listen --forward-to web:5000/bookings/webhook
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
celery_app = celery  # for `celery -A celery_worker.celery worker`
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
# 1. Clone, install deps, create .env from .env.example
cp .env.example .env
# Edit .env: set STRIPE_SECRET_KEY, STRIPE_PUBLISHABLE_KEY, STRIPE_WEBHOOK_SECRET, MAILTRAP_*

# 2. Start the stack
docker compose up --build

# 3. In a second terminal — initialize the database
docker compose exec web flask --app wsgi db upgrade
# (or: docker compose exec web python -c "from app import create_app; from app.extensions import db; app=create_app(); app.app_context().push(); db.create_all()")

# 4. In a third terminal — start Stripe CLI forwarding
docker compose logs -f stripe-cli
# The first run prints a webhook signing secret; paste it into .env and restart web
```

Open `http://localhost:5000`, browse events, click a seat — it turns yellow. In a second browser (incognito) viewing the same event, you'll see the seat turn yellow within ~50 ms. Click "Hold" in the first browser, go to checkout, fill in the Stripe test card `4242 4242 4242 4242`, and watch:

1. The Stripe webhook fires.
2. A `Booking` row appears in PostgreSQL (`SELECT * FROM bookings WHERE status='paid';`).
3. The seat turns red (sold) in both browsers via SocketIO.
4. Within ~5 seconds, the ticket email with QR PNG lands in Mailtrap.
5. Open `/organizer/event/1/scan` and scan the QR (or paste the JWT) — attendee checks in.

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
# tests/test_inventory.py
"""Concurrency test: 50 simultaneous holds should never oversell a 10-seat event."""
import threading
from app.models import Event, TicketType, User
from app.extensions import db

def test_no_oversell_under_contention(app):
    with app.app_context():
        u = User(email="org@test", name="Org"); u.set_password("x"); u.is_organizer = True
        ev = Event(title="T", slug="t", description="x", organizer=u)
        tt = TicketType(name="GA", price_cents=1000, capacity=10, event=ev)
        db.session.add_all([u, ev, tt]); db.session.commit()
        app.redis.set(f"avail:{ev.id}:{tt.id}", 10)

        results = []
        def hold(user_id):
            with app.test_request_context():
                scripts = app.config["SCRIPTS"]
                r = scripts["hold"](
                    keys=[ev.id, tt.id],
                    args=[str(user_id), "600", "ga", "1"]
                )
                import json
                results.append(json.loads(r)["ok"])

        threads = [threading.Thread(target=hold, args=(i,)) for i in range(50)]
        for t in threads: t.start()
        for t in threads: t.join()

        assert sum(results) == 10  # exactly 10 succeeded, 40 failed
        assert app.redis.get(f"avail:{ev.id}:{tt.id}") == "0"
```

```python
# tests/test_webhook_idempotency.py
def test_double_webhook_does_not_duplicate_tickets(app, client):
    """If Stripe redelivers the webhook, we must not create duplicate tickets."""
    from app.models import Booking, Ticket
    from app.extensions import db
    # ... seed booking with status=pending, fire webhook twice, assert only N tickets exist
```

Run:

```bash
pytest -q
```

### End-to-end smoke checklist

| # | Action | Expected |
|---|---|---|
| 1 | `GET /` | Event list renders |
| 2 | Open event detail in 2 browsers | Both show same availability |
| 3 | Click seat in browser 1 | Browser 2 shows seat yellow within ~100 ms |
| 4 | Wait 10 minutes | Seat returns to green (TTL) |
| 5 | Hold + checkout with `4242...` | Webhook fires, seat turns red, tickets created |
| 6 | Check Mailtrap inbox | Email with QR PNG arrives within ~5 s |
| 7 | Open `/organizer/event/1/scan`, scan QR | Returns `{"ok": true, "name": ...}` |
| 8 | Scan same QR again | Returns `{"ok": true, "already": true}` |
| 9 | Cancel booking from `/account/tickets` | Refund in Stripe, seat returns to pool, waitlist notified |

---

## 9. Deployment

### Production checklist

- [ ] **gunicorn** with `eventlet` worker (SocketIO needs it). One worker for now; scale horizontally behind a sticky-session load balancer.
- [ ] **PostgreSQL**: managed RDS / Cloud SQL, automated nightly snapshots.
- [ ] **Redis**: managed ElastiCache / Memorystore with `appendonly` persistence (so a Redis restart doesn't lose 10 minutes of holds).
- [ ] **Stripe webhook endpoint** registered in the Stripe dashboard with the production URL.
- [ ] **CSRF** enabled everywhere except `/bookings/webhook`.
- [ ] **Secrets** in Doppler / Vault / AWS Secrets Manager — never in the image.
- [ ] **Celery worker + beat** as separate Fly.io apps or ECS services.
- [ ] **HTTPS only** — HSTS, `Secure` cookies, `SameSite=Lax`.
- [ ] **QR secret** rotated periodically; old tickets remain valid until their JWT `exp` lapses.

### Fly.io example (web)

```toml
# fly.toml
app = "reserve-web"
[deploy]
  strategy = "rolling"
[http_service]
  internal_port = 5000
  auto_stop_machines = false        # SocketIO needs sticky sessions
  auto_start_machines = true
  min_machines_running = 1
```

For the worker and beat:

```toml
app = "reserve-worker"
[experimental]
  cmd = ["celery", "-A", "celery_worker.celery", "worker", "--loglevel=info"]
```

```mermaid
flowchart LR
    subgraph Browser
        UI[Tailwind + HTMX] -->|hx-post /bookings/hold| Hold[Hold endpoint]
        UI -.->|Socket.IO| WSRoom[event-123 room]
    end
    subgraph Flask
        Hold --> Lua[EVAL hold_script]
        Lua --> R[(Redis)]
        Hold --> Emit[socketio.emit]
        Emit -.-> WSRoom
        Checkout --> Stripe[stripe.PaymentIntent.create]
        Webhook --> Confirm[EVAL confirm_script]
        Confirm --> DB[(PostgreSQL<br/>Booking + Tickets)]
        Webhook --> Queue[send_tickets_email.delay]
    end
    Stripe -.->|signed webhook| Webhook
    Queue --> Worker[Celery worker]
    Worker --> SMTP[SMTP/Mailtrap]
    Beat[Celery beat] -->|hourly| Reminders[send_reminders task]
    style Lua fill:#fef3c7,stroke:#d97706
    style Emit fill:#fce7f3,stroke:#db2777
    style Stripe fill:#dbeafe,stroke:#2563eb
```

---

## 10. Next Steps / Enhancements

Once you have the core working, push it further:

1. **Reserved seating with seat categories.** Replace the flat `seats_json` with a 2D layout (`{row: 'A', col: 1, type: 'vip'}`) and render an actual venue map.
2. **Group discounts.** "Buy 10, get 10% off" — apply at the Stripe PaymentIntent via `amount` adjustment.
3. **Check-in analytics.** A live dashboard showing % of attendees checked in, by ticket type, by hour. Reuse the SocketIO pattern from [[Project-05-Live-Analytics-Dashboard]].
4. **iCal export.** Generate `.ics` files for booked events so they appear in the attendee's calendar.
5. **Stripe Connect for multi-organizer.** Each organizer connects their Stripe account; payments route directly to them, platform takes a fee. This is the "Eventbrite" model.
6. **Apple Wallet / Google Wallet passes.** Generate `.pkpass` files alongside the QR PNG — attendees tap their phone at the door.
7. **Anti-fraud.** Use Stripe Radar rules + IP rate limiting (Flask-Limiter) to detect bots buying up blocks of seats.
8. **Waitlist auto-promotion.** When seats free up, hold them for the first waitlisted user for 5 minutes before offering them publicly.

---

## 11. What We Learned

- **Redis Lua scripts are the right primitive for atomic inventory.** A single `EVAL` either reserves N seats or returns "not enough" — no TOCTOU window, no abort-retry storm. This is the pattern behind every real ticketing system from Ticketmaster to airlines.
- **Time-limited holds via Redis TTL** eliminate the cleanup cron. The key auto-expires; the only thing the beat task does is reconcile stale DB rows.
- **Stripe PaymentIntents + webhooks are the source of truth.** The user redirecting to `/success` doesn't mean anything — the webhook does. Idempotency via checking `booking.status == "paid"` before processing prevents double-ticketing on Stripe's at-least-once redelivery.
- **Flask-SocketIO rooms are the simplest real-time fan-out pattern in Flask.** One room per event, one `emit(..., room=...)` per state change. No message broker, no Kafka.
- **JWT-signed QR codes** make offline check-in possible — the scanner doesn't need DB access to verify authenticity, and the `exp` claim gives you automatic ticket expiry.
- **Celery beat** is the right tool for scheduled reminders and periodic cleanup. Don't put cron jobs on the host; put them in the application.
- **HTMX 2.0** keeps the frontend in HTML — `hx-post` for the hold button, no React, no SPA build. The WebSocket bridges the few cases where server-initiated updates are needed.
- **Snapshot `seats_json` on the Booking** so changing the venue layout later doesn't break historical records. Same lesson as snapshotting `price_cents` in [[Project-04-Modern-Ecommerce]].
- **Idempotency at every boundary** — Stripe webhook, QR scan, refund flow. The rule of thumb: any external callback may arrive twice; your handler must be safe to call N times.
- **Concurrency tests with `threading`** catch overselling bugs that unit tests miss. Always write at least one test that fires 50 concurrent holds at a 10-seat event.

---

```mermaid
flowchart TB
    subgraph Real-time Layer
        SocketIO[Flask-SocketIO<br/>room per event]
        Redis[(Redis<br/>Lua scripts + TTL)]
    end
    subgraph Persistence Layer
        PG[(PostgreSQL<br/>Bookings + Tickets)]
        Stripe[Stripe<br/>PaymentIntents]
    end
    subgraph Async Layer
        Worker[Celery worker<br/>ticket emails]
        Beat[Celery beat<br/>reminders + cleanup]
    end
    Browser[Browser<br/>HTMX + Tailwind] -->|HTTP| Flask[Flask routes]
    Flask <--> SocketIO
    Flask <--> Redis
    SocketIO <--> Browser
    Flask --> PG
    Flask <--> Stripe
    Stripe -->|webhook| Flask
    Flask -->|delay| Worker
    Beat -->|periodic| Worker
    Worker --> SMTP[SMTP]
    style Redis fill:#fef3c7,stroke:#d97706
    style Stripe fill:#fce7f3,stroke:#db2777
    style SocketIO fill:#dbeafe,stroke:#2563eb
```

> [!quote] A note from your teacher
> Booking systems look deceptively simple — "just decrement a counter." The moment two users race for the last seat, the simplicity evaporates. Building Reserve teaches you the *one* pattern that solves it: **push the contention into a single-threaded atomic primitive** (Redis Lua), and let everything else be eventually consistent. Once you internalize this, every inventory problem — concert tickets, restaurant reservations, doctor appointments, hotel rooms — becomes the same shape. Pair this project with [[Project-12-Real-Time-Auction]] to see how the same concurrency patterns apply when the "inventory" is a single high-stakes item with a running price.
