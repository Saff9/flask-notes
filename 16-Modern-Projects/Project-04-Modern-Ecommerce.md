---
title: "Project 04 — Modern E-Commerce Platform (Mercado)"
module: 16-Modern-Projects
project_number: 4
estimated_time: "10–14 hours build, 45–60 hours to perfect"
difficulty: intermediate
prerequisites:
  - "Complete the core course (Lessons 01–24)"
  - "[[Lesson-16-User-Model-AndPasswords]] — password hashing, the User model"
  - "[[Lesson-17-Flask-Login]] — sessions, login_required"
  - "[[Lesson-12-SQLAlchemy-Setup]] and [[Lesson-13-CRUD-Operations]] — models & CRUD"
  - "[[Lesson-14-Relationships]] — one-to-many (User → Order → OrderItem)"
  - "[[Lesson-18-Blueprints]] — splitting shop, cart, checkout, admin blueprints"
  - "[[Lesson-21-File-Uploads]] — product image upload + Pillow thumbnails"
  - "[[Lesson-22-Email-And-Tasks]] — Celery + Flask-Mail for order emails"
  - "[[Modern-Frontend-Integration]] — HTMX, Tailwind 4, DaisyUI"
  - "[[Modern-Databases-And-Caching]] — Redis, PostgreSQL"
  - "[[Project-03-SaaS-Starter-Kit]] — Stripe webhook + Celery patterns reused"
tags:
  - flask
  - flask3
  - python313
  - project
  - portfolio
  - ecommerce
  - stripe
  - redis
  - celery
  - htmx
  - tailwind
  - daisyui
  - 2026
related_lessons:
  - "[[Lesson-13-CRUD-Operations]]"
  - "[[Lesson-17-Flask-Login]]"
  - "[[Lesson-18-Blueprints]]"
  - "[[Lesson-21-File-Uploads]]"
  - "[[Lesson-22-Email-And-Tasks]]"
  - "[[Lesson-23-Deployment]]"
  - "[[Modern-Frontend-Integration]]"
  - "[[Modern-Databases-And-Caching]]"
  - "[[Modern-Deployment-And-DevOps]]"
  - "[[Project-03-SaaS-Starter-Kit]]"
  - "[[Project-05-Live-Analytics-Dashboard]]"
---

# Project 04 — Modern E-Commerce Platform ("Mercado")

> [!success] Time to ship something you could actually sell.
> By the end of this project you'll have a working e-commerce store — product catalog, Redis-backed cart, Stripe Checkout, async order emails, real-time cart badge with HTMX, admin panel, reviews, and a wishlist. This is the project that proves you can build "the boring stuff that makes money" — and the boring stuff is exactly what hiring managers want to see.

#flask #ecommerce #stripe #redis #celery #htmx #tailwind #portfolio #2026

---

## 1. Project Overview

**Mercado** is a small-but-real e-commerce platform built with **Flask 3.2 + SQLAlchemy 2.0 + PostgreSQL + Redis + Stripe + Celery + HTMX**. Customers browse a product catalog, filter by category and price, drop items into a cart (which persists even if they close the browser), check out through Stripe's hosted Checkout page, and receive an order-confirmation email asynchronously through Celery. Admins get a dashboard to add products, manage inventory, and watch orders come in.

### Why this is a modern 2026 project

A "Flask shop" tutorial from 2018 had three routes, a SQLite database, and a hand-rolled `/checkout` form that posted to a fake payment gateway. Modern e-commerce in 2026 looks completely different:

1. **Stripe Checkout, not raw card forms.** PCI compliance is delegated to Stripe's hosted page — you never see a card number. Webhooks (not the redirect URL) are the source of truth for "did this order actually get paid?"
2. **Cart lives in Redis, not session cookies.** A cart with 20 items × 8 fields = 160 keys × 4 KB cookies = broken. Redis gives you a fast, schema-validated, server-side cart that survives logout.
3. **Async email + inventory via Celery.** Sending an SMTP email synchronously in the request handler means a slow mail server makes your `/pay` route take 8 seconds. Celery moves that off the request thread.
4. **HTMX for the cart badge.** No React, no Redux, no 200 KB of JavaScript. One `hx-post` attribute and the badge updates without a page reload.
5. **SQLAlchemy 2.0 `select()` style.** The legacy `Query` API is gone. We use `db.session.execute(select(Product).where(...))` everywhere — it's typed, composable, and matches the docs.

> [!info] Where this fits in the project series
> Project 03 ([[Project-03-SaaS-Starter-Kit]]) taught you Stripe webhooks, Celery tasks, and multi-tenancy. **Mercado reuses those patterns** but flips the lens: instead of selling a *subscription*, we're selling *physical products* with inventory, carts, and shipping. Read P03 first — we won't re-explain HMAC signature verification.

### Architecture at a glance

```mermaid
flowchart TB
    C[Customer<br/>HTMX + Tailwind] -->|browse, search| F[Flask App]
    F -->|catalog read| PG[(PostgreSQL<br/>products / orders / reviews)]
    F -->|cart state JSON| R[(Redis<br/>cart:session_id)]
    F -->|Checkout Session| ST[Stripe]
    ST -->|checkout.session.completed webhook| F
    F -->|enqueue email| Q[Celery Worker]
    Q -->|SMTP| C
    F -->|HTMX cart-badge swap| C
    F -->|admin: new order| WS[Flask-SocketIO]
    WS -->|live feed| A[Admin Dashboard]
    style F fill:#dbeafe,stroke:#2563eb
    style R fill:#fef3c7,stroke:#d97706
    style ST fill:#fce7f3,stroke:#db2777
    style Q fill:#dcfce7,stroke:#16a34a
```

---

## 2. What You'll Learn

By the end of this project you will have hands-on experience with:

- **Stripe Checkout Sessions** — the modern PCI-compliant redirect flow, plus webhook signature verification and idempotent event handling.
- **Redis as a cart store** — JSON-serialized cart state, per-session keys with TTLs, atomic add/remove operations.
- **Celery + Flask-Mail** — enqueue a transactional email in a request, retry on failure, keep the request handler fast.
- **HTMX 2.0** — `hx-post`, `hx-swap="outerHTML"`, `hx-target`, and out-of-band swaps (`hx-swap-oob`) for the cart badge.
- **SQLAlchemy 2.0** — `select()`, `where()`, `options(selectinload())`, `with_for_update()` for inventory reservation.
- **Pillow thumbnails** — generate `product_thumb.webp` on upload, serve next-gen formats.
- **Flask-Limiter** — rate-limit the auth and cart endpoints against scraping.
- **Docker Compose** — five services (web, worker, postgres, redis, stripe-cli) with one command.

---

## 3. Prerequisites

| Lesson | Why |
|---|---|
| [[Lesson-16-User-Model-AndPasswords]] | `User` model, password hashing. |
| [[Lesson-17-Flask-Login]] | `LoginManager`, `@login_required`. |
| [[Lesson-13-CRUD-Operations]] | `select()`, `db.session.add()`, `db.session.commit()`. |
| [[Lesson-14-Relationships]] | `User → Order → OrderItem → Product`. |
| [[Lesson-18-Blueprints]] | `shop`, `cart`, `checkout`, `admin`, `api` blueprints. |
| [[Lesson-21-File-Uploads]] | `request.files`, `werkzeug.secure_filename`, Pillow. |
| [[Lesson-22-Email-And-Tasks]] | Celery `@app.task`, `Flask-Mail`. |
| [[Modern-Databases-And-Caching]] | Redis patterns, PostgreSQL. |
| [[Project-03-SaaS-Starter-Kit]] | Stripe webhook + Celery templates. |

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
| Flask-Limiter | 3.8+ | Rate-limit `/cart/add` and `/auth/login`. |
| Flask-Mail | 0.10+ | Transactional email. |
| Flask-SocketIO | 5.4+ | Live admin order feed. |
| Celery | 5.4+ | Async email + inventory reconciliation. |
| Redis | 7 | Cart store, Celery broker, rate-limit backend. |
| PostgreSQL | 16 | Catalog, orders, reviews. |
| stripe (Python SDK) | 11+ | Checkout Sessions + webhook signature. |
| Pillow | 11+ | Image thumbnails (WebP). |
| HTMX | 2.0 (CDN) | Cart updates without SPA. |
| Tailwind CSS + DaisyUI | 4 / 5 (CDN) | Styling + component library. |
| Docker Compose | 2.30+ | Local dev stack. |

> [!tip] Why DaisyUI on top of Tailwind?
> Tailwind 4 gives you utilities; DaisyUI gives you `btn`, `card`, `badge`, `modal` — pre-styled components that look production-ready. For a portfolio project, this matters: nobody wants to read 600 lines of utility classes to see a "Buy" button.

---

## 5. Project Structure

```
mercado/
├── app/
│   ├── __init__.py            # create_app() factory + SocketIO init
│   ├── config.py              # Config classes
│   ├── extensions.py          # db, login_manager, mail, limiter, socketio, celery
│   ├── models.py              # User, Category, Product, Review, Order, OrderItem, Wishlist
│   ├── cart.py                # Redis-backed Cart class
│   ├── shop.py                # Blueprint: catalog, product detail, search
│   ├── cart_views.py          # Blueprint: HTMX add/remove/update
│   ├── checkout.py            # Blueprint: Stripe Checkout + webhook
│   ├── account.py             # Blueprint: order history, wishlist
│   ├── admin.py               # Blueprint: product CRUD, orders dashboard
│   ├── tasks.py               # Celery: send_order_email, refresh_inventory
│   ├── sockets.py             # SocketIO: admin order feed
│   ├── templates/
│   │   ├── base.html          # Tailwind + HTMX + cart badge (OOB swap)
│   │   ├── shop/{index,product}.html
│   │   ├── cart/{mini,badge}.html   # HTMX partials
│   │   ├── checkout/{success,canceled}.html
│   │   ├── account/{orders,wishlist}.html
│   │   └── admin/{dashboard,product_form,orders}.html
│   └── static/
│       └── uploads/           # Product images (Pillow-written)
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

We'll build this in 14 steps. Every step produces a runnable checkpoint — if you get lost, `git diff` against the prior step.

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
Pillow==11.0.0
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

    # Uploads
    UPLOAD_FOLDER = Path(__file__).parent / "static" / "uploads"
    MAX_CONTENT_LENGTH = 5 * 1024 * 1024  # 5 MiB

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
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from flask_socketio import SocketIO

db = SQLAlchemy()
login_manager = LoginManager()
csrf = CSRFProtect()
mail = Mail()
limiter = Limiter(key_func=get_remote_address, default_limits=["200/hour"])
socketio = SocketIO(cors_allowed_origins="*", async_mode="eventlet")

# Celery is created here so tasks can import it without circular deps
from celery import Celery
celery = Celery("mercado")
```

```python
# app/__init__.py
from flask import Flask
from app.config import Config
from app.extensions import db, login_manager, csrf, mail, limiter, socketio, celery

def create_app(config_class=Config) -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_class)

    # Init extensions
    db.init_app(app)
    login_manager.init_app(app)
    csrf.init_app(app)
    mail.init_app(app)
    limiter.init_app(app)
    socketio.init_app(app, async_mode="eventlet")

    # Celery config
    celery.conf.update(
        broker_url=app.config["CELERY_BROKER_URL"],
        result_backend=app.config["CELERY_RESULT_BACKEND"],
        task_serializer="json",
        accept_content=["json"],
        task_track_started=True,
    )

    # User loader
    from app.models import User
    @login_manager.user_loader
    def load_user(uid):
        return db.session.get(User, int(uid))

    # Register blueprints
    from app.shop import shop_bp
    from app.cart_views import cart_bp
    from app.checkout import checkout_bp
    from app.account import account_bp
    from app.admin import admin_bp
    app.register_blueprint(shop_bp)
    app.register_blueprint(cart_bp, url_prefix="/cart")
    app.register_blueprint(checkout_bp, url_prefix="/checkout")
    app.register_blueprint(account_bp, url_prefix="/account")
    app.register_blueprint(admin_bp, url_prefix="/admin")

    # SocketIO handlers
    from app import sockets  # noqa: F401 (registers handlers)

    # Ensure upload dir exists
    app.config["UPLOAD_FOLDER"].mkdir(parents=True, exist_ok=True)

    return app
```

> [!info] Why a factory, not module-level `app = Flask(__name__)`?
> The factory lets you create multiple apps in one process — critical for tests, Celery tasks, and CLI scripts. See [[Lesson-18-Blueprints]].

### Step 4 — Models

```python
# app/models.py
from datetime import datetime, timezone
from flask_login import UserMixin
from sqlalchemy import Integer, String, Text, Float, DateTime, ForeignKey, Boolean, UniqueConstraint
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
    is_admin: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    orders: Mapped[list["Order"]] = relationship(back_populates="user")
    reviews: Mapped[list["Review"]] = relationship(back_populates="user")
    wishlist: Mapped[list["Wishlist"]] = relationship(back_populates="user")

    def set_password(self, pw: str) -> None:
        self.password_hash = generate_password_hash(pw)
    def check_password(self, pw: str) -> bool:
        return check_password_hash(self.password_hash, pw)

class Category(db.Model):
    __tablename__ = "categories"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(80), unique=True)
    slug: Mapped[str] = mapped_column(String(80), unique=True, index=True)
    products: Mapped[list["Product"]] = relationship(back_populates="category")

class Product(db.Model):
    __tablename__ = "products"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200), index=True)
    slug: Mapped[str] = mapped_column(String(200), unique=True, index=True)
    description: Mapped[str] = mapped_column(Text)
    price_cents: Mapped[int] = mapped_column(Integer)  # store money as cents, never float
    stock: Mapped[int] = mapped_column(Integer, default=0)
    image_path: Mapped[str | None] = mapped_column(String(255), nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    category_id: Mapped[int] = mapped_column(ForeignKey("categories.id"))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    category: Mapped[Category] = relationship(back_populates="products")
    reviews: Mapped[list["Review"]] = relationship(back_populates="product")

    @property
    def price(self) -> float:
        return self.price_cents / 100
    @property
    def avg_rating(self) -> float | None:
        if not self.reviews:
            return None
        return sum(r.stars for r in self.reviews) / len(self.reviews)

class Review(db.Model):
    __tablename__ = "reviews"
    __table_args__ = (UniqueConstraint("product_id", "user_id", name="uq_one_review_per_user"),)
    id: Mapped[int] = mapped_column(primary_key=True)
    product_id: Mapped[int] = mapped_column(ForeignKey("products.id"))
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    stars: Mapped[int] = mapped_column(Integer)  # 1..5
    body: Mapped[str] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    product: Mapped[Product] = relationship(back_populates="reviews")
    user: Mapped[User] = relationship(back_populates="reviews")

class Order(db.Model):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int | None] = mapped_column(ForeignKey("users.id"), nullable=True)
    stripe_session_id: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    email: Mapped[str] = mapped_column(String(255))
    total_cents: Mapped[int] = mapped_column(Integer)
    status: Mapped[str] = mapped_column(String(20), default="paid")  # paid, fulfilled, canceled
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    user: Mapped[User | None] = relationship(back_populates="orders")
    items: Mapped[list["OrderItem"]] = relationship(back_populates="order", cascade="all, delete-orphan")

class OrderItem(db.Model):
    __tablename__ = "order_items"
    id: Mapped[int] = mapped_column(primary_key=True)
    order_id: Mapped[int] = mapped_column(ForeignKey("orders.id"))
    product_id: Mapped[int] = mapped_column(ForeignKey("products.id"))
    name: Mapped[str] = mapped_column(String(200))   # snapshot at purchase time
    price_cents: Mapped[int] = mapped_column(Integer)  # snapshot
    quantity: Mapped[int] = mapped_column(Integer)

    order: Mapped[Order] = relationship(back_populates="items")
    product: Mapped[Product] = relationship()

class Wishlist(db.Model):
    __tablename__ = "wishlist"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    product_id: Mapped[int] = mapped_column(ForeignKey("products.id"))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    user: Mapped[User] = relationship(back_populates="wishlist")
    product: Mapped[Product] = relationship()
```

> [!tip] Snapshot `name` and `price_cents` on `OrderItem`
> If you only stored `product_id`, an admin renaming a product or raising its price would silently rewrite history. Snapshots preserve what the customer actually saw and paid. This is "event sourcing light."

### Step 5 — Entity-relationship map

```mermaid
erDiagram
    USER ||--o{ ORDER : places
    USER ||--o{ REVIEW : writes
    USER ||--o{ WISHLIST : has
    CATEGORY ||--o{ PRODUCT : contains
    PRODUCT ||--o{ REVIEW : receives
    PRODUCT ||--o{ ORDER_ITEM : appears_in
    PRODUCT ||--o{ WISHLIST : wished_for
    ORDER ||--|{ ORDER_ITEM : contains
    USER {
        int id PK
        string email
        string password_hash
        bool is_admin
    }
    PRODUCT {
        int id PK
        string name
        int price_cents
        int stock
        bool is_active
    }
    ORDER {
        int id PK
        string stripe_session_id
        int total_cents
        string status
    }
    ORDER_ITEM {
        int id PK
        string name
        int price_cents
        int quantity
    }
    REVIEW {
        int id PK
        int stars
        text body
    }
```

### Step 6 — The Redis-backed cart

The cart is the heart of any store. We store it in Redis as a JSON blob keyed by session ID. Guests get a random session ID; logged-in users get `cart:user:{user_id}` so their cart follows them across devices.

```python
# app/cart.py
import json
from dataclasses import dataclass, asdict
from typing import Iterable
from redis import Redis
from flask import session, current_app, g

@dataclass
class CartLine:
    product_id: int
    name: str
    price_cents: int
    quantity: int
    image_path: str | None

class Cart:
    """Redis-backed shopping cart. One instance per request."""

    def __init__(self, redis: Redis, key: str):
        self._r = redis
        self._key = key

    @classmethod
    def current(cls) -> "Cart":
        r = current_app.extensions["redis"]
        if g.get("user_id"):
            key = f"cart:user:{g.user_id}"
        else:
            sid = session.setdefault("cart_sid", secrets_token())
            key = f"cart:session:{sid}"
        return cls(r, key)

    def _load(self) -> list[CartLine]:
        raw = self._r.get(self._key)
        if not raw:
            return []
        return [CartLine(**d) for d in json.loads(raw)]

    def _save(self, lines: list[CartLine]) -> None:
        self._r.set(self._key, json.dumps([asdict(l) for l in lines]), ex=60 * 60 * 24 * 30)

    def add(self, product, qty: int = 1) -> None:
        lines = self._load()
        for ln in lines:
            if ln.product_id == product.id:
                ln.quantity += qty
                break
        else:
            lines.append(CartLine(product.id, product.name, product.price_cents, qty, product.image_path))
        self._save(lines)

    def remove(self, product_id: int) -> None:
        lines = [l for l in self._load() if l.product_id != product_id]
        self._save(lines)

    def set_qty(self, product_id: int, qty: int) -> None:
        lines = self._load()
        for ln in lines:
            if ln.product_id == product_id:
                ln.quantity = max(0, qty)
        self._save([l for l in lines if l.quantity > 0])

    def clear(self) -> None:
        self._r.delete(self._key)

    @property
    def lines(self) -> list[CartLine]:
        return self._load()

    @property
    def total_cents(self) -> int:
        return sum(l.price_cents * l.quantity for l in self._load())

    @property
    def count(self) -> int:
        return sum(l.quantity for l in self._load())

def secrets_token() -> str:
    import secrets
    return secrets.token_urlsafe(24)
```

> [!info] Why a dataclass, not a dict?
> Type hints catch typos. `cart.add(product, qty=2)` is fine; `cart.add(product, "two")` is not. Pydantic would also work, but a dataclass is zero-overhead and stdlib.

### Step 6.5 — Base template + auth blueprint

The base template sets up Tailwind 4 + DaisyUI 5 from CDNs (no build step), pulls in HTMX 2.0, and renders the navbar with the cart badge. Every other template `{% extends "base.html" %}`.

```html
<!-- templates/base.html -->
<!doctype html>
<html lang="en" data-theme="light">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}Mercado{% endblock %}</title>
  <!-- Tailwind 4 + DaisyUI 5 (CDN, no build) -->
  <script src="https://cdn.tailwindcss.com?plugins=forms,typography"></script>
  <link href="https://cdn.jsdelivr.net/npm/daisyui@5.0.0/dist/full.min.css" rel="stylesheet">
  <!-- HTMX 2.0 -->
  <script src="https://unpkg.com/htmx.org@2.0.3"></script>
</head>
<body class="min-h-screen flex flex-col">
  <nav class="navbar bg-base-100 shadow sticky top-0 z-50">
    <div class="flex-1">
      <a class="btn btn-ghost text-xl" href="{{ url_for('shop.index') }}">Mercado</a>
    </div>
    <div class="flex-none gap-2">
      <form action="{{ url_for('shop.index') }}" method="get" class="join">
        <input type="search" name="q" placeholder="Search…"
               value="{{ request.args.get('q','') }}"
               class="input input-bordered join-item">
        <button class="btn btn-primary join-item">Go</button>
      </form>
      <a href="{{ url_for('cart.view') }}" class="btn btn-ghost btn-circle">
        {% include "cart/badge.html" with context %}
      </a>
      {% if current_user.is_authenticated %}
        <div class="dropdown dropdown-end">
          <button class="btn btn-ghost btn-circle avatar" tabindex="0">
            <div class="w-10 rounded-full">{{ current_user.email[0]|upper }}</div>
          </button>
          <ul class="menu dropdown-content bg-base-100 rounded-box shadow w-52">
            <li><a href="{{ url_for('account.orders') }}">Orders</a></li>
            <li><a href="{{ url_for('account.wishlist_view') }}">Wishlist</a></li>
            {% if current_user.is_admin %}
              <li><a href="{{ url_for('admin.dashboard') }}">Admin</a></li>
            {% endif %}
            <li><a href="{{ url_for('auth.logout') }}">Sign out</a></li>
          </ul>
        </div>
      {% else %}
        <a href="{{ url_for('auth.login') }}" class="btn btn-ghost">Sign in</a>
      {% endif %}
    </div>
  </nav>

  <main class="flex-1 container mx-auto p-6">
    {% with messages = get_flashed_messages(with_categories=true) %}
      {% for cat, msg in messages %}
        <div class="alert alert-{{ cat }} mb-4">{{ msg }}</div>
      {% endfor %}
    {% endwith %}
    {% block content %}{% endblock %}
  </main>

  <footer class="footer footer-center p-6 bg-base-200">
    <p>Mercado · Built with Flask 3.2 · {{ now().year if now is defined else 2026 }}</p>
  </footer>
</body>
</html>
```

Auth blueprint — straightforward registration + login, writing into the `User` model from [[Lesson-16-User-Model-AndPasswords]]:

```python
# app/auth.py  (omitted from structure for brevity — register as auth_bp)
from flask import Blueprint, render_template, request, redirect, url_for, flash
from flask_login import login_user, logout_user, login_required, current_user
from sqlalchemy import select
from app.extensions import db, limiter
from app.models import User

auth_bp = Blueprint("auth", __name__)

@auth_bp.route("/register", methods=["GET", "POST"])
@limiter.limit("5/hour")
def register():
    if request.method == "POST":
        email = request.form["email"].lower().strip()
        if db.session.execute(select(User).where(User.email == email)).scalar_one_or_none():
            flash("That email is already registered.", "error")
            return redirect(url_for("auth.register"))
        u = User(email=email)
        u.set_password(request.form["password"])
        db.session.add(u); db.session.commit()
        login_user(u)
        return redirect(url_for("shop.index"))
    return render_template("auth/register.html")

@auth_bp.route("/login", methods=["GET", "POST"])
@limiter.limit("10/hour")
def login():
    if request.method == "POST":
        u = db.session.execute(select(User).where(User.email == request.form["email"].lower())).scalar_one_or_none()
        if u and u.check_password(request.form["password"]):
            login_user(u)
            return redirect(request.args.get("next") or url_for("shop.index"))
        flash("Invalid credentials.", "error")
    return render_template("auth/login.html")

@auth_bp.post("/logout")
@login_required
def logout():
    logout_user()
    return redirect(url_for("shop.index"))
```

Don't forget to register it: `app.register_blueprint(auth_bp, url_prefix="/auth")` in `create_app`.

> [!warning] Rate-limiting auth routes
> Without `@limiter.limit("10/hour")` on `/auth/login`, a botnet will hammer you with credential stuffing in week one. Flask-Limiter with a Redis backend gives you per-IP caps cheaply. See [[Modern-Deployment-And-DevOps]].

### Step 7 — Shop blueprint: catalog, product detail, search

```python
# app/shop.py
from flask import Blueprint, render_template, request, abort
from sqlalchemy import select, func, or_
from app.extensions import db
from app.models import Product, Category, Review

shop_bp = Blueprint("shop", __name__)

@shop_bp.route("/")
def index():
    q = request.args.get("q", "").strip()
    cat = request.args.get("cat", type=int)
    sort = request.args.get("sort", "newest")

    stmt = select(Product).where(Product.is_active.is_(True))
    if q:
        stmt = stmt.where(Product.name.ilike(f"%{q}%"))
    if cat:
        stmt = stmt.where(Product.category_id == cat)
    if sort == "price_asc":
        stmt = stmt.order_by(Product.price_cents.asc())
    elif sort == "price_desc":
        stmt = stmt.order_by(Product.price_cents.desc())
    else:
        stmt = stmt.order_by(Product.created_at.desc())

    products = db.session.execute(stmt).scalars().all()
    categories = db.session.execute(select(Category).order_by(Category.name)).scalars().all()
    return render_template("shop/index.html", products=products, categories=categories, q=q, cat=cat, sort=sort)

@shop_bp.route("/product/<slug>")
def product(slug: str):
    stmt = (
        select(Product)
        .where(Product.slug == slug, Product.is_active.is_(True))
        .options(selectinload(Product.reviews).selectinload(Review.user))
    )
    product = db.session.execute(stmt).scalar_one_or_none()
    if not product:
        abort(404)
    return render_template("shop/product.html", product=product)
```

> [!tip] `selectinload` beats the N+1
> Without `selectinload(Product.reviews)`, rendering a product page issues 1 query for the product + 1 query per review's user. With it, the ORM fires 3 queries total: product, reviews, users — and joins them in Python. See [[Modern-Databases-And-Caching]].

Need a `selectinload` import:

```python
from sqlalchemy.orm import selectinload
```

### Step 8 — HTMX cart blueprint

This is the modern part. The "Add to cart" button is just:

```html
<button hx-post="/cart/add/{{ product.id }}"
        hx-target="#cart-badge"
        hx-swap="outerHTML"
        class="btn btn-primary">Add to cart</button>
```

The server returns the **cart badge HTML fragment**, and HTMX swaps it in place. No JSON, no `fetch`, no React.

```python
# app/cart_views.py
from flask import Blueprint, render_template, request, jsonify
from sqlalchemy import select
from app.extensions import db, limiter
from app.models import Product
from app.cart import Cart

cart_bp = Blueprint("cart", __name__)

@cart_bp.post("/add/<int:pid>")
@limiter.limit("60/minute")
def add(pid: int):
    product = db.session.execute(select(Product).where(Product.id == pid)).scalar_one_or_none()
    if not product or product.stock < 1:
        return render_template("cart/badge.html", count=0, error="Out of stock"), 409
    Cart.current().add(product, qty=int(request.form.get("qty", 1)))
    # Return the badge fragment — HTMX swaps it in place
    return render_template("cart/badge.html", count=Cart.current().count)

@cart_bp.post("/remove/<int:pid>")
def remove(pid: int):
    Cart.current().remove(pid)
    return _mini_cart()

@cart_bp.post("/update/<int:pid>")
def update(pid: int):
    qty = int(request.form.get("qty", 1))
    Cart.current().set_qty(pid, qty)
    return _mini_cart()

@cart_bp.get("/")
def view():
    return _mini_cart()

def _mini_cart():
    cart = Cart.current()
    return render_template("cart/mini.html", lines=cart.lines, total_cents=cart.total_cents)
```

The badge partial:

```html
<!-- templates/cart/badge.html -->
<span id="cart-badge" class="badge badge-primary badge-lg">
  🛒 {{ count }}
</span>
```

The mini-cart partial (also HTMX-swapped):

```html
<!-- templates/cart/mini.html -->
<div id="mini-cart" class="card bg-base-100 shadow">
  <div class="card-body">
    <h2 class="card-title">Your cart</h2>
    {% if not lines %}
      <p class="text-base-content/60">Cart is empty.</p>
    {% else %}
      <ul class="divide-y">
        {% for ln in lines %}
          <li class="py-2 flex items-center gap-2">
            <span class="flex-1">{{ ln.name }} × {{ ln.quantity }}</span>
            <span class="tabular-nums">${{ '%.2f'|format(ln.price_cents * ln.quantity / 100) }}</span>
            <button hx-post="/cart/remove/{{ ln.product_id }}"
                    hx-target="#mini-cart"
                    hx-swap="outerHTML"
                    class="btn btn-ghost btn-xs">✕</button>
          </li>
        {% endfor %}
      </ul>
      <div class="flex justify-between mt-2 font-semibold">
        <span>Total</span>
        <span class="tabular-nums">${{ '%.2f'|format(total_cents / 100) }}</span>
      </div>
      <a href="/checkout" class="btn btn-primary btn-block mt-3">Checkout</a>
    {% endif %}
  </div>
</div>
```

> [!success] Out-of-band swaps for the navbar badge
> If you want the navbar badge to update *and* a "toast" to appear, return both fragments and mark the toast with `hx-swap-oob="true"`. HTMX will swap both targets from one response.

### Step 9 — Stripe Checkout + webhook

```python
# app/checkout.py
import stripe
from flask import Blueprint, render_template, request, redirect, url_for, current_app, flash, abort
from sqlalchemy import select
from app.extensions import db, socketio
from app.models import Product, Order, OrderItem
from app.cart import Cart
from app.tasks import send_order_confirmation

checkout_bp = Blueprint("checkout", __name__)

@checkout_bp.route("/")
def checkout():
    cart = Cart.current()
    if not cart.lines:
        flash("Your cart is empty.")
        return redirect(url_for("shop.index"))
    return render_template("checkout/form.html", lines=cart.lines, total_cents=cart.total_cents)

@checkout_bp.post("/pay")
def pay():
    cart = Cart.current()
    if not cart.lines:
        abort(400)

    stripe.api_key = current_app.config["STRIPE_SECRET_KEY"]
    line_items = [{
        "quantity": ln.quantity,
        "price_data": {
            "currency": current_app.config["STRIPE_CURRENCY"],
            "unit_amount": ln.price_cents,
            "product_data": {"name": ln.name},
        },
    } for ln in cart.lines]

    session = stripe.checkout.Session.create(
        mode="payment",
        line_items=line_items,
        success_url=request.host_url + "checkout/success?sid={CHECKOUT_SESSION_ID}",
        cancel_url=request.host_url + "checkout/canceled",
        customer_email=request.form.get("email") or None,
        metadata={"cart_total_cents": str(cart.total_cents)},
    )
    return redirect(session.url, code=303)

@checkout_bp.route("/success")
def success():
    sid = request.args.get("sid")
    return render_template("checkout/success.html", sid=sid)

@checkout_bp.route("/canceled")
def canceled():
    return render_template("checkout/canceled.html")

# Webhook — no CSRF (Stripe signs the payload instead)
@checkout_bp.post("/webhook")
def webhook():
    payload = request.get_data(as_text=True)
    sig = request.headers.get("Stripe-Signature", "")
    try:
        event = stripe.Webhook.construct_event(
            payload, sig, current_app.config["STRIPE_WEBHOOK_SECRET"]
        )
    except (ValueError, stripe.error.SignatureVerificationError):
        abort(400)

    if event["type"] == "checkout.session.completed":
        _fulfill(event["data"]["object"])
    return "", 200

def _fulfill(session_obj):
    # Idempotency: skip if we already processed this session
    exists = db.session.execute(
        select(Order).where(Order.stripe_session_id == session_obj["id"])
    ).scalar_one_or_none()
    if exists:
        return

    cart = Cart.current()  # ← in production, reconstruct from session.metadata
    order = Order(
        stripe_session_id=session_obj["id"],
        email=session_obj.get("customer_details", {}).get("email", ""),
        total_cents=int(session_obj["amount_total"]),
        status="paid",
    )
    db.session.add(order)
    db.session.flush()  # populate order.id

    for ln in cart.lines:
        # Decrement stock with a row lock
        prod = db.session.execute(
            select(Product).where(Product.id == ln.product_id).with_for_update()
        ).scalar_one()
        prod.stock = max(0, prod.stock - ln.quantity)
        db.session.add(OrderItem(
            order_id=order.id,
            product_id=prod.id,
            name=ln.name,
            price_cents=ln.price_cents,
            quantity=ln.quantity,
        ))
    db.session.commit()
    cart.clear()

    # Fire async email + notify admin via WebSocket
    send_order_confirmation.delay(order.id)
    socketio.emit("new_order", {"id": order.id, "total_cents": order.total_cents}, namespace="/admin")
```

> [!danger] Never trust the redirect URL
> Customers can hit `/checkout/success` manually without paying. The redirect is for UX only — the **webhook is the source of truth**. Always create the `Order` row inside `_fulfill()`, gated by Stripe's signature.

Exempt the webhook from CSRF:

```python
# app/__init__.py (inside create_app, after csrf.init_app)
csrf.exempt(checkout_bp)  # or specifically the `webhook` route
```

> [!danger] Stripe webhook deep-dive — why we sign, verify, and idempotent-dispatch
> Stripe's webhook signature is `t=timestamp,v1=HMAC-SHA256(payload, webhook_secret)`. The `t` lets you reject replay attacks older than ~5 minutes. The `v1` proves the payload came from Stripe (only Stripe and you know the secret). If you skip `construct_event`, anyone who finds your webhook URL can forge a `checkout.session.completed` and get free goods.
>
> Stripe **redelivers** events for up to 3 days if your endpoint returns non-2xx. So your handler *must* be idempotent — running it twice for the same `session.id` must produce the same final state as running it once. That's why `_fulfill()` short-circuits with a `SELECT … WHERE stripe_session_id = …` check. Without it, a slow response on your side could lead to two `Order` rows for the same checkout.
>
> Stripe also delivers events **out of order**. A `charge.refunded` might arrive before the corresponding `checkout.session.completed` if your fulfillment endpoint was slow. Design your handlers to look up the order by `stripe_session_id` (the source of truth) and not by `event.created` timestamp.

> [!tip] Pillow deep-dive — why WebP and 400×400
> Product images from phones are 4000×3000 JPEGs at 4 MB. Serving them as-is means your homepage ships 80 MB of images. Pillow fixes this in two lines:
>
> ```python
> Image.open(full).convert("RGB").resize((400, 400)).save(thumb, "WEBP", quality=82)
> ```
>
> Why 400×400? Catalog cards on retina screens look crisp at 400 px logical, 800 px physical — and we set the `<img>` `width="400" height="400"` to avoid layout shift.
>
> Why WebP quality 82? At equal quality, WebP is ~30% smaller than JPEG. Quality 82 is the visual sweet spot — dropping to 70 introduces visible banding; pushing past 88 doubles file size with no perceptible improvement. Set `Cache-Control: public, max-age=31536000, immutable` on `/static/uploads/` so browsers cache forever — the filename includes a hash or timestamp, so busting is automatic.

### Step 10 — Celery tasks

```python
# celery_worker.py
from app import create_app
from app.extensions import celery

app = create_app()
app.app_context().push()
```

```python
# app/tasks.py
from celery import shared_task
from flask import render_template, current_app
from flask_mail import Message
from app.extensions import db, mail, celery
from app.models import Order
from sqlalchemy import select

@celery.task(bind=True, max_retries=3, default_retry_delay=30)
def send_order_confirmation(self, order_id: int):
    order = db.session.execute(
        select(Order).where(Order.id == order_id)
    ).scalar_one_or_none()
    if not order:
        return  # order was deleted; nothing to do

    msg = Message(
        subject=f"Mercado — Order #{order.id}",
        sender=current_app.config.get("MAIL_USERNAME"),
        recipients=[order.email],
    )
    msg.body = render_template("email/order_confirmation.txt", order=order)
    msg.html = render_template("email/order_confirmation.html", order=order)
    try:
        mail.send(msg)
    except Exception as exc:
        raise self.retry(exc=exc)

@celery.task
def reconcile_inventory():
    """Nightly job: cross-check actual stock against a stock ledger."""
    # Omitted for brevity — fetch from an ERP, write a StockAdjustment row, etc.
    pass
```

Start the worker:

```bash
celery -A celery_worker.celery worker --loglevel=info
```

### Step 11 — Order-confirmation email templates

```
<!-- templates/email/order_confirmation.txt -->
Hi,

Thanks for your order #{{ order.id }} on Mercado!

Total: ${{ '%.2f'|format(order.total_cents / 100) }}

Items:
{% for item in order.items %}
  - {{ item.name }} × {{ item.quantity }} — ${{ '%.2f'|format(item.price_cents * item.quantity / 100) }}
{% endfor %}

We'll ship to the address you provided at checkout.

— Mercado
```

### Step 12 — Account & wishlist blueprint

```python
# app/account.py
from flask import Blueprint, render_template, request, redirect, url_for, abort
from flask_login import login_required, current_user
from sqlalchemy import select
from app.extensions import db
from app.models import Order, Product, Wishlist

account_bp = Blueprint("account", __name__)

@account_bp.get("/orders")
@login_required
def orders():
    stmt = (
        select(Order)
        .where(Order.user_id == current_user.id)
        .order_by(Order.created_at.desc())
    )
    return render_template("account/orders.html", orders=db.session.execute(stmt).scalars().all())

@account_bp.post("/wishlist/<int:pid>")
@login_required
def wishlist_add(pid: int):
    if not db.session.execute(select(Wishlist).where(Wishlist.user_id == current_user.id, Wishlist.product_id == pid)).scalar_one_or_none():
        db.session.add(Wishlist(user_id=current_user.id, product_id=pid))
        db.session.commit()
    return "", 204

@account_bp.get("/wishlist")
@login_required
def wishlist_view():
    stmt = select(Product).join(Wishlist, Wishlist.product_id == Product.id).where(Wishlist.user_id == current_user.id)
    return render_template("account/wishlist.html", products=db.session.execute(stmt).scalars().all())
```

### Step 13 — Admin blueprint + image upload

```python
# app/admin.py
import os
from pathlib import Path
from PIL import Image
from flask import Blueprint, render_template, request, redirect, url_for, flash, current_app, abort
from flask_login import login_required, current_user
from werkzeug.utils import secure_filename
from sqlalchemy import select, func
from app.extensions import db
from app.models import Product, Category, Order, Review

admin_bp = Blueprint("admin", __name__)

def admin_required(fn):
    @login_required
    def wrapper(*args, **kwargs):
        if not current_user.is_admin:
            abort(403)
        return fn(*args, **kwargs)
    wrapper.__name__ = fn.__name__
    return wrapper

@admin_bp.get("/")
@admin_required
def dashboard():
    # Live stats
    total_orders = db.session.execute(select(func.count(Order.id))).scalar()
    revenue = db.session.execute(select(func.coalesce(func.sum(Order.total_cents), 0))).scalar()
    low_stock = db.session.execute(
        select(Product).where(Product.stock < 5, Product.is_active.is_(True))
    ).scalars().all()
    return render_template("admin/dashboard.html", total_orders=total_orders, revenue=revenue, low_stock=low_stock)

@admin_bp.route("/products/new", methods=["GET", "POST"])
@admin_required
def new_product():
    if request.method == "POST":
        file = request.files.get("image")
        image_path = None
        if file and file.filename:
            name = secure_filename(file.filename)
            upload_dir: Path = current_app.config["UPLOAD_FOLDER"]
            full = upload_dir / name
            file.save(full)
            # Generate a 400x400 WebP thumbnail
            thumb_name = Path(name).stem + "_thumb.webp"
            Image.open(full).convert("RGB").resize((400, 400)).save(upload_dir / thumb_name, "WEBP", quality=82)
            image_path = f"uploads/{thumb_name}"

        p = Product(
            name=request.form["name"],
            slug=request.form["slug"],
            description=request.form["description"],
            price_cents=int(float(request.form["price"]) * 100),
            stock=int(request.form["stock"]),
            category_id=int(request.form["category_id"]),
            image_path=image_path,
        )
        db.session.add(p)
        db.session.commit()
        flash(f"Created {p.name}")
        return redirect(url_for("admin.dashboard"))
    categories = db.session.execute(select(Category).order_by(Category.name)).scalars().all()
    return render_template("admin/product_form.html", categories=categories)
```

### Step 14 — SocketIO admin feed

```python
# app/sockets.py
from flask_socketio import join_room, emit
from flask_login import current_user
from app.extensions import socketio

@socketio.on("connect", namespace="/admin")
def admin_connect():
    if not current_user.is_authenticated or not current_user.is_admin:
        return False  # reject the connection
    join_room("admins")
    emit("ready", {"msg": "subscribed"}, namespace="/admin")
```

In `templates/admin/dashboard.html`:

```html
<script src="https://cdn.socket.io/4.7/socket.io.min.js"></script>
<script>
  const sock = io("/admin", { transports: ["websocket"] });
  sock.on("new_order", (data) => {
    const list = document.getElementById("order-feed");
    const li = document.createElement("li");
    li.className = "py-2";
    li.textContent = `Order #${data.id} — $${(data.total_cents / 100).toFixed(2)}`;
    list.prepend(li);
  });
</script>
```

---

## 7. Checkout data flow

```mermaid
sequenceDiagram
    participant C as Customer
    participant F as Flask
    participant S as Stripe
    participant DB as PostgreSQL
    participant Q as Celery Worker
    participant A as Admin (WS)

    C->>F: POST /checkout/pay
    F->>S: Create Checkout Session
    S-->>F: session.url
    F-->>C: 303 redirect to Stripe
    C->>S: Card details (hosted page)
    S-->>C: Redirect to /checkout/success
    S->>F: POST /checkout/webhook (signed)
    F->>F: Verify signature
    F->>DB: BEGIN; INSERT order + items; FOR UPDATE on products
    F->>DB: COMMIT
    F->>Q: send_order_confirmation.delay(order_id)
    F->>A: socketio.emit new_order
    Q->>C: SMTP email
    A-->>A: Live feed updates
```

---

## 8. Running the Project

`docker-compose.yml`:

```yaml
services:
  web:
    build: .
    command: gunicorn -k eventlet -w 1 -b 0.0.0.0:5000 wsgi:app
    ports: ["5000:5000"]
    env_file: .env
    depends_on: [postgres, redis]
  worker:
    build: .
    command: celery -A celery_worker.celery worker --loglevel=info
    env_file: .env
    depends_on: [redis]
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: mercado
      POSTGRES_PASSWORD: devpass
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
  stripe-cli:
    image: stripe/stripe-cli:latest
    command: listen --api-key ${STRIPE_SECRET_KEY} --forward-to web:5000/checkout/webhook
    env_file: .env
volumes: { pgdata: {} }
```

Run:

```bash
cp .env.example .env  # fill in STRIPE_*, MAIL_*, etc.
docker compose up --build
```

The `Dockerfile` is small and built for layer caching:

```dockerfile
# Dockerfile
FROM python:3.13-slim

# System deps for Pillow + psycopg
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev gcc libjpeg-dev zlib1g-dev libwebp-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Cache requirements layer
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Create a non-root user
RUN useradd -m mercado && chown -R mercado:mercado /app
USER mercado

EXPOSE 5000
CMD ["gunicorn", "-k", "eventlet", "-w", "1", "-b", "0.0.0.0:5000", "wsgi:app"]
```

> [!info] Why one gunicorn worker with `eventlet`?
> Flask-SocketIO needs an async worker class. With `eventlet`, one process can hold thousands of concurrent WebSocket connections through greenlets. Don't scale by adding gunicorn workers — scale by adding Fly machines behind a sticky-session load balancer (SocketIO breaks without sticky sessions). See [[Project-02-Real-Time-Collaborative-Editor]].

`.env.example`:

```
SECRET_KEY=change-me
DATABASE_URL=postgresql+psycopg://postgres:devpass@postgres:5432/mercado
REDIS_URL=redis://redis:6379/0
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
MAIL_SERVER=smtp.mailtrap.io
MAIL_PORT=587
MAIL_USERNAME=mailtrap-user
MAIL_PASSWORD=mailtrap-pass
CELERY_BROKER_URL=redis://redis:6379/1
CELERY_RESULT_BACKEND=redis://redis:6379/2
```

Open `http://localhost:5000`, browse products, add to cart — the badge updates without a reload. Click **Checkout**, fill in the Stripe test card `4242 4242 4242 4242`, and watch:

1. Stripe redirects you back to `/checkout/success`.
2. The Stripe CLI forwards the webhook to your Flask app.
3. An order row appears in PostgreSQL (verify with `docker compose exec postgres psql -U postgres mercado -c "SELECT * FROM orders;"`).
4. Within ~5 seconds, the order-confirmation email lands in Mailtrap.
5. The admin dashboard's live feed (open `/admin` in a second tab) shows the new order in real time.

---

## 9. Testing

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

# tests/test_cart.py
def test_add_to_cart_updates_badge(client):
    # seed a product
    from app.models import Product, Category
    from app.extensions import db
    cat = Category(name="Books", slug="books")
    p = Product(name="Flask Up", slug="flask-up", description="x", price_cents=1999,
                stock=10, category=cat)
    db.session.add_all([cat, p]); db.session.commit()

    resp = client.post(f"/cart/add/{p.id}")
    assert resp.status_code == 200
    assert "1" in resp.text  # badge shows count = 1

def test_out_of_stock_returns_409(client):
    from app.models import Product, Category
    from app.extensions import db
    cat = Category(name="Books", slug="books")
    p = Product(name="Sold Out", slug="sold-out", description="x", price_cents=1000,
                stock=0, category=cat)
    db.session.add_all([cat, p]); db.session.commit()
    resp = client.post(f"/cart/add/{p.id}")
    assert resp.status_code == 409
```

Run:

```bash
pytest -q
```

You should see something like:

```
===== 12 passed in 1.84s =====
```

### Stripe webhook test

Use the Stripe CLI to fire a fake event:

```bash
stripe trigger checkout.session.completed
```

Watch the worker logs to confirm `send_order_confirmation` fires.

### End-to-end smoke test

A quick Playwright-style checklist you can run by hand against your local stack:

| # | Action | Expected |
|---|---|---|
| 1 | `GET /` | Catalog renders with at least one product |
| 2 | `POST /cart/add/1` (HTMX) | Response body contains the badge HTML with count `1` |
| 3 | `POST /cart/add/1` again | Badge shows `2` |
| 4 | `POST /cart/remove/1` | Mini-cart re-renders, line removed |
| 5 | `POST /checkout/pay` | Redirect to `checkout.stripe.com` |
| 6 | `stripe trigger checkout.session.completed` | New row in `orders` table; Celery task enqueued |
| 7 | Check Mailtrap inbox | Email arrives within ~5s |
| 8 | `GET /admin/` (as admin) | Live feed socket connects, order appears in list |

---

## 10. Deployment

### Production checklist

- [ ] **gunicorn** with `eventlet` worker (SocketIO needs it). One worker for now; scale horizontally behind a sticky-session load balancer.
- [ ] **PostgreSQL**: managed RDS / Cloud SQL, automated nightly snapshots.
- [ ] **Redis**: managed ElastiCache / Memorystore with `appendonly` persistence for carts.
- [ ] **Stripe webhook endpoint** registered in the Stripe dashboard with the production URL.
- [ ] **CSRF** enabled everywhere except `/checkout/webhook`.
- [ ] **Secrets** in Doppler / Vault / AWS Secrets Manager — never in the image.
- [ ] **Image uploads**: switch from local disk to S3 + CloudFront in prod. Pillow can write to a `BytesIO` and `boto3.put_object`.
- [ ] **HTTPS only** — HSTS, `Secure` cookies, `SameSite=Lax`.
- [ ] **Celery worker** as a separate Fly.io app or ECS service. Don't co-locate with web — a long-running task shouldn't tie up a web worker.

### Fly.io example

```toml
# fly.toml
app = "mercado-web"
[deploy]
  strategy = "rolling"
[http_service]
  internal_port = 5000
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 0
```

For the worker:

```toml
app = "mercado-worker"
[deploy]
  strategy = "rolling"
[experimental]
  cmd = ["celery", "-A", "celery_worker.celery", "worker", "--loglevel=info"]
```

---

## 11. Next Steps / Enhancements

Once you have the core working, push it further:

1. **Abandoned-cart emails.** A Celery beat job checks every hour: "any cart last updated >24h ago with items still in it?" Send a reminder email. This is where e-commerce *actually* makes money.
2. **Coupons + discounts.** Add a `Coupon` model with `code`, `percent_off`, `expires_at`. Pass `discounts` to Stripe's Checkout Session.
3. **Product variants** (size, color). Replace `Product` flat fields with `Product → Variant` one-to-many.
4. **Faceted search with PostgreSQL `tsvector`.** Add a `search_vector` generated column and a GIN index. Full-text search in 5 lines.
5. **Recommendations.** A Celery job computes "users who bought X also bought Y" nightly using `pg_trgm` and `covar_samp`.
6. **Subscriptions.** Reuse Project 03's Stripe Billing code to sell a "subscribe & save" variant.
7. **Multi-currency.** Use Stripe's `currency` field and a daily FX rate pulled by a Celery beat job.

---

## 12. What We Learned

- **Stripe Checkout is the only PCI-compliant way for small teams** — never build a card form yourself. Webhooks are the source of truth, not the success URL.
- **Redis as a cart store** is faster and more flexible than cookie-session carts, and lets you do abandoned-cart recovery. A 30-day TTL means carts survive browser restarts but don't linger forever.
- **HTMX 2.0** lets you build a fully interactive cart without a single line of React — `hx-post` + `hx-swap="outerHTML"` is the whole magic. The server returns the *next* HTML state, not JSON.
- **SQLAlchemy 2.0 `select()` style** with `selectinload` and `with_for_update` gives you composable queries and safe inventory updates. The row lock on `with_for_update()` prevents two concurrent checkouts from overselling the last item.
- **Celery** keeps your request handlers fast: SMTP, image resizing, and inventory reconciliation happen off-thread. `bind=True` + `max_retries=3` + `default_retry_delay=30` is the right default for transactional email.
- **Pillow + WebP** gives you next-gen image formats with ~30% smaller thumbnails than JPEG. Browsers in 2026 all support WebP — there's no reason to ship JPEG.
- **Docker Compose** with a `stripe-cli` service means webhooks work in local dev with zero tunneling setup — a 2026 best practice that beats `ngrok` for project-local development.
- **Idempotency by `stripe_session_id`** prevents duplicate order rows if Stripe redelivers a webhook (which it will — they guarantee at-least-once delivery for 3 days).
- **Snapshot `name` and `price_cents` on `OrderItem`** so a future price change or rename doesn't rewrite history. This is "event sourcing light" — the order is a frozen view of the cart at purchase time.

---

```mermaid
flowchart LR
    subgraph Browser
        UI[Tailwind + HTMX] -->|hx-post /cart/add| Badge[Cart Badge]
    end
    subgraph Flask
        CartBP[cart_views] --> CartRedis[(Redis<br/>cart:*)]
        ShopBP[shop] --> Catalog[(PostgreSQL<br/>products)]
        CheckBP[checkout] --> StripeSDK[stripe SDK]
        CheckBP --> OrderTable[(orders + items)]
        CheckBP --> Emit[socketio.emit]
        AdminBP[admin] --> Pillow[Pillow thumb]
    end
    StripeSDK --> Stripe[Stripe Checkout]
    Stripe -->|signed webhook| CheckBP
    CheckBP -->|task.delay| Worker[Celery worker]
    Worker --> SMTP[SMTP / Mailtrap]
    Emit -.->|WS| AdminBrowser[Admin dashboard]
    style CheckBP fill:#dbeafe,stroke:#2563eb
    style Stripe fill:#fce7f3,stroke:#db2777
    style CartRedis fill:#fef3c7,stroke:#d97706
```

> [!quote] A note from your teacher
> E-commerce isn't glamorous. There's no LLM. No vector database. No streaming. But it's the project that *most reliably* teaches you the patterns that pay the bills: idempotent webhooks, atomic inventory updates, async email, and real-time UX without a frontend framework. Build it well, deploy it, put it on your résumé with a live demo link — and then go build [[Project-05-Live-Analytics-Dashboard]] so you can *see* how customers actually use your store.
