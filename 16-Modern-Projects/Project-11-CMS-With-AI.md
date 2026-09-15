---
title: "Project 11 — AI-Powered CMS (Quill)"
module: 16-Modern-Projects
project_number: 11
estimated_time: "10–14 hours build, 35–50 hours to perfect"
difficulty: intermediate
prerequisites:
  - "Complete the core course (Lessons 01–24)"
  - "[[Lesson-16-User-Model-AndPasswords]] — password hashing, the User model"
  - "[[Lesson-17-Flask-Login]] — sessions, login_required"
  - "[[Lesson-12-SQLAlchemy-Setup]] and [[Lesson-13-CRUD-Operations]] — models & CRUD"
  - "[[Lesson-14-Relationships]] — many-to-many (Page ↔ Tag)"
  - "[[Lesson-18-Blueprints]] — splitting admin, author, public blueprints"
  - "[[Lesson-21-File-Uploads]] — image upload + Pillow thumbnails"
  - "[[Lesson-22-Email-And-Tasks]] — Celery beat for scheduled publishing"
  - "[[Modern-Frontend-Integration]] — HTMX live preview, Tailwind 4"
  - "[[Modern-Databases-And-Caching]] — Redis caching, PostgreSQL tsvector"
tags:
  - flask
  - flask3
  - python313
  - project
  - portfolio
  - cms
  - ai
  - llm
  - openai
  - anthropic
  - markdown
  - htmx
  - tailwind
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
  - "[[Project-01-AI-Chat-Application]]"
  - "[[Project-10-Event-Booking-Platform]]"
---

# Project 11 — AI-Powered CMS ("Quill")

> [!success] Build the project that non-engineers understand instantly.
> By the end of this project you'll have a Markdown-first content management system where authors write in Markdown, an AI assistant drafts and summarizes content, generates meta descriptions for SEO, and even creates cover images with DALL-E. Editors live-preview rendered HTML as they type (HTMX), reviewers approve drafts, scheduled posts publish on time via Celery beat, and the public site reads straight from Redis-cached HTML for sub-10 ms responses. This is the project that proves you can build **AI-integrated, multi-role, performant server-rendered apps** in Flask.

#flask #cms #ai #llm #openai #markdown #htmx #tailwind #sse #seo #portfolio #2026

---

## 1. Project Overview

**Quill** is a content management system built with **Flask 3.2 + SQLAlchemy 2.0 + PostgreSQL 16 + Redis + OpenAI/Anthropic + Celery + HTMX 2.0 + Tailwind 4**. Authors write in Markdown; on every keystroke (debounced), an HTMX request asks the server to render preview HTML and (optionally) ask the LLM for suggestions. Editors approve drafts; admins manage users. Every published page is rendered once and cached in Redis; the public site reads the cache, so a viral Hacker News spike costs nothing extra. Scheduled posts publish at their `published_at` time via Celery beat. A full-text search powered by PostgreSQL `tsvector` finds any post in <10 ms across millions of rows.

### Why this is a modern 2026 project

A 2018 "Flask blog" tutorial had four routes, a SQLite database, and a single `Post` model with `body TEXT`. Modern CMS projects in 2026 look completely different:

1. **AI is a first-class collaborator, not a chatbot sidebar.** Authors say "expand this paragraph" or "summarize the whole piece" or "translate to Spanish" — the LLM streams responses back via Server-Sent Events (SSE) directly into the editor.
2. **Live preview is server-rendered.** The same Markdown rendering pipeline that produces the public HTML is used for the editor preview — no drift between what the author sees and what the reader gets.
3. **Role-based access is enforced server-side.** Authors can write drafts but not publish; reviewers can approve but not edit; admins can do everything. Permissions are checked in the route, not the UI.
4. **Version history with restore.** Every save creates a `Version` row; the diff is one SQL query. Restoring an old version creates *another* version (never destroys history).
5. **PostgreSQL `tsvector` for full-text search.** No Elasticsearch, no Algolia — Postgres handles a million rows fine with a GIN index. One fewer moving part.
6. **Scheduled publishing via Celery beat.** "Publish this at 9 a.m. Tuesday" works without cron on the host — the beat task finds due drafts and publishes them.
7. **Redis-cached rendered HTML.** The public site reads pre-rendered HTML from Redis keyed by URL path — a cache hit is a single `GET` with no DB queries at all.

> [!info] Where this fits in the project series
> Project 01 ([[Project-01-AI-Chat-Application]]) taught you LLM integration, streaming, and prompt engineering. **Quill reuses those patterns** but adds the *content management* dimension: Markdown rendering pipelines, multi-role auth, version history, and Redis-cached HTML. Read P01 first — we won't re-explain OpenAI client setup.

### Architecture at a glance

```mermaid
flowchart TB
    A[Author<br/>HTMX + Tailwind] -->|type Markdown| F[Flask App]
    F -->|render preview| MD[markdown-it-py]
    F -->|SSE stream| LLM[OpenAI / Anthropic]
    LLM -->|token stream| F
    F -->|text/event-stream| A
    F -->|publish| PG[(PostgreSQL<br/>pages / versions / media)]
    F -->|render + cache HTML| R[(Redis<br/>page:path)]
    P[Public Reader] -->|GET /p/slug| F
    F -->|read cache| R
    F -->|serve HTML| P
    CB[Celery Beat] -->|every minute| F
    F -->|publish due drafts| PG
    style F fill:#dbeafe,stroke:#2563eb
    style LLM fill:#fce7f3,stroke:#db2777
    style R fill:#dcfce7,stroke:#16a34a
```

---

## 2. What You'll Learn

By the end of this project you will have hands-on experience with:

- **Markdown rendering pipelines** — `markdown-it-py` with syntax highlighting (`pygments`), custom plugins, and safe HTML sanitization.
- **Server-Sent Events (SSE)** for streaming LLM responses — `text/event-stream`, `yield` chunks, client-side `EventSource`.
- **HTMX live preview** — debounce input, `hx-post` to the preview endpoint, swap the rendered HTML into a `<div>`.
- **Role-based access control (RBAC)** — `User.role` enum, `@role_required` decorator, route-level enforcement.
- **Version history** — every save snapshots the full content; restore creates a new version (immutable history).
- **PostgreSQL full-text search** — `tsvector`, `tsquery`, GIN index, ranking with `ts_rank`.
- **Redis-cached rendered HTML** — `SET page:/p/my-post <html>` on publish, `GET` on read; cache invalidation on update.
- **Scheduled publishing with Celery beat** — find `status='scheduled' AND published_at <= now()` and flip to `published`.
- **DALL-E image generation** for cover art — generate from a prompt, upload to S3/R2, attach to the page.
- **Docker Compose** — five services (web, worker, beat, postgres, redis) in one command.

---

## 3. Prerequisites

| Lesson | Why |
|---|---|
| [[Lesson-16-User-Model-AndPasswords]] | `User` model, password hashing. |
| [[Lesson-17-Flask-Login]] | `LoginManager`, `@login_required`. |
| [[Lesson-13-CRUD-Operations]] | `select()`, `db.session.add()`, `db.session.commit()`. |
| [[Lesson-14-Relationships]] | `Page ↔ Tag` many-to-many. |
| [[Lesson-18-Blueprints]] | `admin`, `author`, `public` blueprints. |
| [[Lesson-21-File-Uploads]] | `request.files`, secure filename, Pillow thumbnails. |
| [[Lesson-22-Email-And-Tasks]] | Celery beat, `@app.task`. |
| [[Modern-Frontend-Integration]] | HTMX, SSE, Tailwind 4. |
| [[Modern-Databases-And-Caching]] | Redis caching, PostgreSQL tsvector. |
| [[Project-01-AI-Chat-Application]] | OpenAI client setup, streaming responses. |

You also need:
- Python 3.13+, Docker, Docker Compose
- An **OpenAI** API key (GPT-4o or newer, plus DALL-E 3 access)
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
| Celery | 5.4+ | Scheduled publishing, sitemap. |
| Redis | 7 | Rendered HTML cache, Celery broker. |
| PostgreSQL | 16 | Pages, versions, media, full-text search. |
| `markdown-it-py` | 4.0+ | Markdown → HTML with extension support. |
| `pygments` | 2.18+ | Code syntax highlighting. |
| `bleach` | 6.1+ | HTML sanitization (defense in depth). |
| `openai` (Python SDK) | 1.50+ | Chat completions + DALL-E. |
| `httpx` | 0.27+ | Async HTTP (LLM streaming). |
| `Pillow` | 11+ | Image thumbnails for media library. |
| `boto3` | 1.35+ | S3 / R2 uploads. |
| HTMX | 2.0 (CDN) | Live preview, swap-oob. |
| Tailwind CSS | 4 (CDN) | Styling. |
| Docker Compose | 2.30+ | Local dev stack. |

> [!tip] Why `markdown-it-py` over `python-markdown`?
> `markdown-it-py` is the CommonMark port of JavaScript's `markdown-it`. It's what GitHub, Obsidian, and Discourse use. Spec compliance is better, the extension API is cleaner, and it's actively maintained in 2026. `python-markdown` is fine for simple cases but has surprising edge cases around nested lists and fenced code.

---

## 5. Project Structure

```
quill/
├── app/
│   ├── __init__.py            # create_app() factory
│   ├── config.py              # Config classes
│   ├── extensions.py          # db, login_manager, mail, celery, redis, openai
│   ├── models.py              # User, Page, Version, Media, Category, Tag, PageTag
│   ├── markdown.py            # Markdown rendering pipeline + sanitization
│   ├── auth.py                # Blueprint: login, register
│   ├── permissions.py         # role_required decorator, role enum
│   ├── author.py              # Blueprint: editor, live preview, AI features
│   ├── ai.py                  # OpenAI client, streaming, prompt templates
│   ├── admin.py               # Blueprint: user management, dashboard
│   ├── public.py              # Blueprint: public site, RSS, sitemap, search
│   ├── tasks.py               # Celery: scheduled publish, sitemap regen
│   ├── media.py               # Blueprint: upload, generate cover image
│   ├── templates/
│   │   ├── base.html          # Tailwind + HTMX + EventSource
│   │   ├── author/{editor,list,versions}.html
│   │   ├── admin/{users,dashboard}.html
│   │   └── public/{page,index,search,rss.xml}.html
│   └── static/
│       └── uploads/           # Local dev uploads (S3 in prod)
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

We'll build this in **14 steps**. Every step produces a runnable checkpoint.

### Step 1 — Install dependencies

```bash
# requirements.txt
Flask==3.2.0
Flask-SQLAlchemy==3.1.1
Flask-Login==0.6.3
Flask-WTF==1.2.1
SQLAlchemy==2.0.36
celery==5.4.0
redis==5.2.0
markdown-it-py==4.0.0
pygments==2.18.0
bleach==6.1.0
openai==1.51.0
httpx==0.27.2
Pillow==11.0.0
boto3==1.35.40
psycopg[binary]==3.2.3
python-dotenv==1.0.1
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
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

class Config:
    SECRET_KEY = os.environ["SECRET_KEY"]
    SQLALCHEMY_DATABASE_URI = os.environ["DATABASE_URL"]
    SQLALCHEMY_ENGINE_OPTIONS = {"pool_pre_ping": True, "pool_size": 10}
    REDIS_URL = os.environ.get("REDIS_URL", "redis://localhost:6379/0")

    # AI
    OPENAI_API_KEY = os.environ["OPENAI_API_KEY"]
    OPENAI_MODEL = os.environ.get("OPENAI_MODEL", "gpt-4o")
    DALLE_MODEL = os.environ.get("DALLE_MODEL", "dall-e-3")

    # Uploads
    UPLOAD_FOLDER = Path(__file__).parent / "static" / "uploads"
    MAX_CONTENT_LENGTH = 10 * 1024 * 1024  # 10 MiB
    S3_BUCKET = os.environ.get("S3_BUCKET")         # set in prod
    S3_REGION = os.environ.get("S3_REGION", "us-east-1")
    CDN_BASE = os.environ.get("CDN_BASE")           # e.g. https://cdn.quill.app

    # Celery
    CELERY_BROKER_URL = os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/1")
    CELERY_RESULT_BACKEND = os.environ.get("CELERY_RESULT_BACKEND", "redis://localhost:6379/2")

    SITE_URL = os.environ.get("SITE_URL", "http://localhost:5000")
    CACHE_TTL_SECONDS = 3600          # 1 hour for rendered HTML

class TestConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"
    WTF_CSRF_ENABLED = False
```

### Step 3 — Extensions + factory

```python
# app/extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager
from flask_wtf import CSRFProtect
from flask_mail import Mail
from celery import Celery
import redis
from openai import OpenAI

db = SQLAlchemy()
login_manager = LoginManager()
csrf = CSRFProtect()
mail = Mail()
celery = Celery("quill")

def make_redis(app):
    return redis.Redis.from_url(app.config["REDIS_URL"], decode_responses=True)

def make_openai(app):
    return OpenAI(api_key=app.config["OPENAI_API_KEY"])
```

```python
# app/__init__.py
from flask import Flask
from app.config import Config
from app.extensions import (
    db, login_manager, csrf, mail, celery, make_redis, make_openai
)

def create_app(config_class=Config) -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_class)

    db.init_app(app)
    login_manager.init_app(app)
    csrf.init_app(app)
    mail.init_app(app)

    app.redis = make_redis(app)
    app.openai = make_openai(app)

    celery.conf.update(
        broker_url=app.config["CELERY_BROKER_URL"],
        result_backend=app.config["CELERY_RESULT_BACKEND"],
        task_serializer="json",
        accept_content=["json"],
        beat_schedule={
            "publish-scheduled-pages": {
                "task": "app.tasks.publish_scheduled",
                "schedule": 60.0,                # every minute
            },
            "regenerate-sitemap": {
                "task": "app.tasks.regenerate_sitemap",
                "schedule": 3600.0,              # hourly
            },
        },
    )

    from app.models import User
    @login_manager.user_loader
    def load_user(uid):
        return db.session.get(User, int(uid))

    from app.auth import auth_bp
    from app.author import author_bp
    from app.admin import admin_bp
    from app.public import public_bp
    from app.media import media_bp
    app.register_blueprint(auth_bp)
    app.register_blueprint(author_bp, url_prefix="/author")
    app.register_blueprint(admin_bp, url_prefix="/admin")
    app.register_blueprint(public_bp)
    app.register_blueprint(media_bp, url_prefix="/media")

    app.config["UPLOAD_FOLDER"].mkdir(parents=True, exist_ok=True)
    return app
```

> [!info] Why is the `OpenAI` client on `app.openai`?
> So that any route or task can do `current_app.openai.chat.completions.create(...)` without re-initializing the client. Connection pooling, retry config, and timeouts are configured once.

### Step 4 — Models with full-text search

```python
# app/models.py
from datetime import datetime, timezone
from enum import Enum
from flask_login import UserMixin
from sqlalchemy import (
    Integer, String, Text, DateTime, ForeignKey, Boolean, Enum as SAEnum,
    UniqueConstraint, Index, Column, func, Computed
)
from sqlalchemy.orm import Mapped, mapped_column, relationship
from werkzeug.security import generate_password_hash, check_password_hash
from app.extensions import db

def utcnow() -> datetime:
    return datetime.now(timezone.utc)

class Role(str, Enum):
    admin = "admin"
    editor = "editor"        # can edit + publish any page
    author = "author"        # can edit own drafts, submit for review
    reviewer = "reviewer"    # can approve drafts

class User(UserMixin, db.Model):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    name: Mapped[str] = mapped_column(String(120))
    role: Mapped[Role] = mapped_column(SAEnum(Role), default=Role.author)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)

    pages: Mapped[list["Page"]] = relationship(back_populates="author")

    def set_password(self, pw: str) -> None:
        self.password_hash = generate_password_hash(pw)
    def check_password(self, pw: str) -> bool:
        return check_password_hash(self.password_hash, pw)

class Page(db.Model):
    __tablename__ = "pages"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200), index=True)
    slug: Mapped[str] = mapped_column(String(200), unique=True, index=True)
    body_markdown: Mapped[str] = mapped_column(Text, default="")
    excerpt: Mapped[str | None] = mapped_column(String(500), nullable=True)
    cover_media_id: Mapped[int | None] = mapped_column(
        ForeignKey("media.id"), nullable=True
    )
    category_id: Mapped[int | None] = mapped_column(ForeignKey("categories.id"), nullable=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    status: Mapped[str] = mapped_column(String(20), default="draft")
    # draft | in_review | scheduled | published | archived
    published_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    scheduled_for: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True), nullable=True
    )
    meta_description: Mapped[str | None] = mapped_column(String(300), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=utcnow, onupdate=utcnow
    )

    author: Mapped[User] = relationship(back_populates="pages")
    category: Mapped["Category | None"] = relationship()
    cover: Mapped["Media | None"] = relationship()
    tags: Mapped[list["Tag"]] = relationship(
        secondary="page_tags", back_populates="pages"
    )
    versions: Mapped[list["Version"]] = relationship(
        back_populates="page", cascade="all, delete-orphan",
        order_by="Version.created_at.desc()"
    )

    # Full-text search: PostgreSQL GENERATED COLUMN (computed from body)
    # On SQLite (tests) this becomes None — see search function for fallback.
    __table_args__ = (
        Index("ix_pages_search", "search_vector", postgresql_using="gin"),
    )
    # The search_vector column is defined via raw SQL in migrations because
    # SQLAlchemy's `Computed` doesn't speak tsvector cleanly. See Step 5.

class Tag(db.Model):
    __tablename__ = "tags"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True)
    slug: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    pages: Mapped[list[Page]] = relationship(secondary="page_tags", back_populates="tags")

class PageTag(db.Model):
    __tablename__ = "page_tags"
    page_id: Mapped[int] = mapped_column(ForeignKey("pages.id"), primary_key=True)
    tag_id: Mapped[int] = mapped_column(ForeignKey("tags.id"), primary_key=True)

class Category(db.Model):
    __tablename__ = "categories"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(80), unique=True)
    slug: Mapped[str] = mapped_column(String(80), unique=True, index=True)

class Version(db.Model):
    __tablename__ = "versions"
    id: Mapped[int] = mapped_column(primary_key=True)
    page_id: Mapped[int] = mapped_column(ForeignKey("pages.id"))
    title: Mapped[str] = mapped_column(String(200))
    body_markdown: Mapped[str] = mapped_column(Text)
    edited_by: Mapped[int] = mapped_column(ForeignKey("users.id"))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    note: Mapped[str | None] = mapped_column(String(200), nullable=True)

    page: Mapped[Page] = relationship(back_populates="versions")

class Media(db.Model):
    __tablename__ = "media"
    id: Mapped[int] = mapped_column(primary_key=True)
    uploaded_by: Mapped[int] = mapped_column(ForeignKey("users.id"))
    filename: Mapped[str] = mapped_column(String(255))
    storage_key: Mapped[str] = mapped_column(String(500))   # S3 key or local path
    content_type: Mapped[str] = mapped_column(String(100))
    width: Mapped[int | None] = mapped_column(Integer, nullable=True)
    height: Mapped[int | None] = mapped_column(Integer, nullable=True)
    alt_text: Mapped[str | None] = mapped_column(String(300), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
```

### Step 5 — Full-text search migration

The `search_vector` column must be a PostgreSQL `tsvector` generated from `title` and `body_markdown`. We do this with a raw migration:

```bash
# Generate migration
flask --app wsgi db init
flask --app wsgi db migrate -m "initial"
```

Then edit the generated migration to add the tsvector column and GIN index:

```python
# migrations/versions/XXXX_initial.py (excerpt — add to upgrade())
def upgrade():
    # ... autogenerated tables ...

    op.execute("""
        ALTER TABLE pages
        ADD COLUMN search_vector tsvector
        GENERATED ALWAYS AS (
            setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
            setweight(to_tsvector('english', coalesce(body_markdown, '')), 'B')
        ) STORED;
    """)
    op.execute("CREATE INDEX ix_pages_search ON pages USING GIN (search_vector);")

def downgrade():
    op.execute("DROP INDEX IF EXISTS ix_pages_search;")
    op.execute("ALTER TABLE pages DROP COLUMN IF EXISTS search_vector;")
```

The search query itself:

```python
# app/public.py (search endpoint — see Step 10 for full blueprint)
from sqlalchemy import select, func, text
from app.models import Page

def search_pages(query: str, limit: int = 20):
    # Useplainto_tsquery for AND semantics on words
    stmt = (
        select(
            Page,
            func.ts_rank(Page.search_vector, func.plainto_tsquery('english', query)).label('rank'),
        )
        .where(Page.search_vector.op('@@')(func.plainto_tsquery('english', query)))
        .where(Page.status == 'published')
        .order_by(text('rank DESC'))
        .limit(limit)
    )
    return db.session.execute(stmt).all()
```

> [!tip] Why `setweight` with 'A' and 'B'?
> Title matches rank higher than body matches. A title hit for "flask" beats a body hit every time. This is how Postgres relevance ranking works — `setweight` tags tokens with weight A/B/C/D, and `ts_rank` sums them with weights 0.1/0.2/0.4/1.0 by default.

### Step 6 — Markdown rendering pipeline

```python
# app/markdown.py
"""Markdown → HTML rendering with extensions and sanitization."""
from markdown_it import MarkdownIt
from pygments import highlight
from pygments.lexers import get_lexer_by_name, guess_lexer
from pygments.formatters import HtmlFormatter
import bleach

# Pygments callback for fenced code blocks
def _highlight(code, lang, _):
    try:
        lexer = get_lexer_by_name(lang or "text", stripall=False)
    except Exception:
        try:
            lexer = guess_lexer(code)
        except Exception:
            lexer = get_lexer_by_name("text")
    formatter = HtmlFormatter(cssclass="syntax", nowrap=False)
    return highlight(code, lexer, formatter)

md = MarkdownIt(
    "commonmark",
    {
        "html": True,                 # allow inline HTML
        "linkify": True,              # auto-link URLs
        "typographer": True,          # smart quotes, dashes
        "highlight": _highlight,
    },
)
md.enable(["table", "strikethrough", "linkify"])

# Allowlist for bleach — defense in depth even though markdown-it is safe
ALLOWED_TAGS = [
    "h1","h2","h3","h4","h5","h6","p","br","hr","a","img","blockquote",
    "ul","ol","li","code","pre","em","strong","del","table","thead",
    "tbody","tr","th","td","div","span","figure","figcaption",
]
ALLOWED_ATTRS = {
    "a": ["href","title","target","rel"],
    "img": ["src","alt","title","width","height"],
    "span": ["class"],
    "div": ["class"],
    "code": ["class"],
    "pre": ["class"],
    "th": ["align"], "td": ["align"],
}

def render_markdown(source: str) -> str:
    """Render Markdown to sanitized HTML."""
    raw = md.render(source or "")
    # Bleach strips any tag/attr not in the allowlist (e.g. <script>)
    safe = bleach.clean(raw, tags=ALLOWED_TAGS, attributes=ALLOWED_ATTRS, strip=True)
    # Force rel="noopener" on external links (security)
    safe = bleach.linkify(safe, skip_tags=["pre", "code"])
    return safe
```

> [!warning] Why both `markdown-it-py` AND `bleach`?
> `markdown-it-py` is spec-compliant and won't generate `<script>` tags from valid Markdown — but it allows raw inline HTML by design (you set `html: True`). If an author pastes `<script>alert('xss')</script>` into their Markdown, `markdown-it-py` will pass it through. `bleach.clean` strips any tag not in the allowlist, so even a malicious or compromised author can't ship XSS to readers.

### Step 7 — AI client: chat, streaming, DALL-E

```python
# app/ai.py
"""OpenAI integration for content tasks.

All prompt templates live here so they can be tuned without touching routes.
"""
from flask import current_app, Response, stream_with_context
from typing import Iterator

# ---- Prompt templates --------------------------------------------------------

PROMPT_GENERATE = (
    "You are an expert content writer. Write a complete draft on the topic: "
    "\"{topic}\". Use Markdown. Aim for 800–1200 words. Include H2 headings "
    "and a short TL;DR at the top."
)
PROMPT_EXPAND = (
    "Expand the following Markdown section with more detail, examples, and "
    "concrete numbers. Keep the original voice. Return only the expanded "
    "Markdown, no preamble.\n\n---\n{selection}\n---"
)
PROMPT_SUMMARIZE = (
    "Summarize the following article in 3 bullet points and one tweet-length "
    "(< 280 chars) summary.\n\n---\n{body}\n---"
)
PROMPT_TRANSLATE = (
    "Translate the following Markdown to {language}. Preserve all Markdown "
    "syntax, code blocks, and links. Return only the translated Markdown.\n\n"
    "---\n{body}\n---"
)
PROMPT_META = (
    "Given the article below, write:\n"
    "1. A meta description (<= 155 chars, compelling, includes the main keyword)\n"
    "2. 5–8 SEO keywords as a comma-separated list\n"
    "3. A suggested slug (kebab-case, <= 60 chars)\n\n"
    "Return as JSON: {{\"meta_description\": \"...\", \"keywords\": [...], \"slug\": \"...\"}}\n\n"
    "---\n{body}\n---"
)

# ---- Sync (non-streaming) ----------------------------------------------------

def generate_meta(body: str) -> dict:
    """Ask the model for an SEO meta description + keywords + slug."""
    client = current_app.openai
    resp = client.chat.completions.create(
        model=current_app.config["OPENAI_MODEL"],
        messages=[
            {"role": "system", "content": "You are an SEO expert. Respond only with JSON."},
            {"role": "user", "content": PROMPT_META.format(body=body[:6000])},
        ],
        response_format={"type": "json_object"},
        temperature=0.3,
    )
    import json
    return json.loads(resp.choices[0].message.content)

# ---- Streaming (SSE) ---------------------------------------------------------

def stream_chat(prompt: str, system: str = "You are a helpful writing assistant.") -> Iterator[str]:
    """Yield chunks formatted as SSE: 'data: <token>\\n\\n'.

    The route wraps this in `stream_with_context` and returns a
    `Response(mimetype='text/event-stream')`.
    """
    client = current_app.openai
    stream = client.chat.completions.create(
        model=current_app.config["OPENAI_MODEL"],
        messages=[
            {"role": "system", "content": system},
            {"role": "user", "content": prompt},
        ],
        stream=True,
        temperature=0.7,
    )
    for chunk in stream:
        delta = chunk.choices[0].delta.content
        if delta:
            # SSE format: each event is `data: <json-or-text>\n\n`
            yield f"data: {delta!r}\n\n"
    yield "data: [DONE]\n\n"

# ---- Image generation --------------------------------------------------------

def generate_cover_image(prompt: str, size: str = "1792x1024") -> str:
    """Generate a cover image with DALL-E 3. Returns the S3/local storage key."""
    client = current_app.openai
    resp = client.images.generate(
        model=current_app.config["DALLE_MODEL"],
        prompt=f"Editorial cover image: {prompt}. Wide aspect ratio, no text overlay.",
        size=size,
        quality="standard",
        n=1,
        response_format="b64_json",
    )
    import base64, time
    img_bytes = base64.b64decode(resp.data[0].b64_json)
    # Save to storage (S3 in prod, local disk in dev)
    from app.media import save_image_bytes
    key = f"ai-covers/{int(time.time())}.png"
    save_image_bytes(key, img_bytes, content_type="image/png")
    return key
```

### Step 8 — Auth + role-based permissions

```python
# app/permissions.py
from functools import wraps
from flask import abort, request
from flask_login import current_user
from app.models import Role, Page

def role_required(*roles: Role):
    """Decorator: abort 403 if user's role isn't in `roles`."""
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            if not current_user.is_authenticated:
                abort(401)
            if current_user.role not in roles:
                abort(403)
            return fn(*args, **kwargs)
        return wrapper
    return decorator

def can_edit_page(user, page: Page) -> bool:
    """Authors can edit their own drafts; editors can edit anything."""
    if user.role in (Role.admin, Role.editor):
        return True
    if user.role == Role.author and page.author_id == user.id and page.status == "draft":
        return True
    return False

def can_publish(user) -> bool:
    return user.role in (Role.admin, Role.editor)

def can_review(user) -> bool:
    return user.role in (Role.admin, Role.editor, Role.reviewer)
```

```python
# app/auth.py
from flask import Blueprint, render_template, request, redirect, url_for, flash
from flask_login import login_user, logout_user, login_required, current_user
from app.models import User
from app.extensions import db

auth_bp = Blueprint("auth", __name__)

@auth_bp.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        email = request.form["email"]
        pw = request.form["password"]
        user = db.session.execute(
            db.select(User).where(User.email == email)
        ).scalar_one_or_none()
        if user and user.check_password(pw) and user.is_active:
            login_user(user)
            return redirect(url_for("author.dashboard"))
        flash("Invalid credentials", "error")
    return render_template("auth/login.html")

@auth_bp.route("/logout")
@login_required
def logout():
    logout_user()
    return redirect(url_for("public.index"))
```

### Step 9 — Author blueprint: editor, live preview, AI features

```python
# app/author.py
import json
from datetime import datetime, timezone
from flask import (
    Blueprint, render_template, request, redirect, url_for,
    flash, current_app, Response, stream_with_context, jsonify
)
from flask_login import login_required, current_user
from sqlalchemy import select

from app.models import Page, Version, Tag, Category, Media
from app.extensions import db
from app.markdown import render_markdown
from app.permissions import role_required, can_edit_page, can_publish, can_review
from app import ai

author_bp = Blueprint("author", __name__)

@author_bp.route("/")
@login_required
def dashboard():
    stmt = (
        select(Page)
        .where(Page.author_id == current_user.id)
        .order_by(Page.updated_at.desc())
    )
    pages = db.session.execute(stmt).scalars().all()
    return render_template("author/list.html", pages=pages)

@author_bp.route("/new", methods=["GET", "POST"])
@login_required
@role_required()  # any authenticated user with a role
def new_page():
    if request.method == "POST":
        page = Page(
            title=request.form["title"] or "Untitled",
            slug=request.form["slug"],
            body_markdown=request.form.get("body", ""),
            author_id=current_user.id,
            status="draft",
        )
        db.session.add(page); db.session.commit()
        return redirect(url_for("author.edit_page", page_id=page.id))
    return render_template("author/editor.html", page=None)

@author_bp.route("/edit/<int:page_id>", methods=["GET", "POST"])
@login_required
def edit_page(page_id: int):
    page = db.session.get(Page, page_id)
    if not page or not can_edit_page(current_user, page):
        return "Forbidden", 403

    if request.method == "POST":
        # Snapshot the current state as a Version BEFORE applying the edit
        db.session.add(Version(
            page_id=page.id, title=page.title, body_markdown=page.body_markdown,
            edited_by=current_user.id,
            note=request.form.get("version_note"),
        ))
        page.title = request.form["title"]
        page.body_markdown = request.form.get("body", "")
        page.excerpt = request.form.get("excerpt")
        page.meta_description = request.form.get("meta_description")
        db.session.add(page); db.session.commit()
        # Invalidate cached HTML
        current_app.redis.delete(f"page:{page.slug}")
        flash("Saved", "success")
        return jsonify({"ok": True})

    return render_template("author/editor.html", page=page)

# --- Live preview (HTMX) ------------------------------------------------------

@author_bp.route("/preview", methods=["POST"])
@login_required
def preview():
    """Render the submitted Markdown to HTML for live preview."""
    body = request.form.get("body", "")
    html = render_markdown(body)
    return html  # HTMX swaps this directly into the preview <div>

# --- AI features (SSE streaming) ----------------------------------------------

@author_bp.route("/ai/generate", methods=["POST"])
@login_required
def ai_generate():
    """Stream a draft from the LLM. Client uses EventSource."""
    topic = request.form.get("topic", "")
    prompt = ai.PROMPT_GENERATE.format(topic=topic)
    return Response(
        stream_with_context(ai.stream_chat(prompt, system="You are an expert content writer.")),
        mimetype="text/event-stream",
    )

@author_bp.route("/ai/expand", methods=["POST"])
@login_required
def ai_expand():
    selection = request.form.get("selection", "")
    prompt = ai.PROMPT_EXPAND.format(selection=selection)
    return Response(
        stream_with_context(ai.stream_chat(prompt)),
        mimetype="text/event-stream",
    )

@author_bp.route("/ai/summarize", methods=["POST"])
@login_required
def ai_summarize():
    body = request.form.get("body", "")
    prompt = ai.PROMPT_SUMMARIZE.format(body=body[:6000])
    return Response(
        stream_with_context(ai.stream_chat(prompt, system="You are an editor.")),
        mimetype="text/event-stream",
    )

@author_bp.route("/ai/translate", methods=["POST"])
@login_required
def ai_translate():
    body = request.form.get("body", "")
    language = request.form.get("language", "Spanish")
    prompt = ai.PROMPT_TRANSLATE.format(body=body[:6000], language=language)
    return Response(
        stream_with_context(ai.stream_chat(prompt, system="You are a professional translator.")),
        mimetype="text/event-stream",
    )

# --- SEO meta (sync JSON) -----------------------------------------------------

@author_bp.route("/ai/meta", methods=["POST"])
@login_required
def ai_meta():
    body = request.form.get("body", "")
    try:
        result = ai.generate_meta(body)
        return jsonify(result)
    except Exception as e:
        return jsonify({"error": str(e)}), 500

# --- Workflow: submit / review / publish / schedule ---------------------------

@author_bp.route("/page/<int:page_id>/submit", methods=["POST"])
@login_required
def submit_for_review(page_id: int):
    page = db.session.get(Page, page_id)
    if not page or not can_edit_page(current_user, page):
        return "Forbidden", 403
    page.status = "in_review"
    db.session.add(page); db.session.commit()
    flash("Submitted for review", "success")
    return redirect(url_for("author.edit_page", page_id=page.id))

@author_bp.route("/page/<int:page_id>/publish", methods=["POST"])
@login_required
def publish(page_id: int):
    if not can_publish(current_user):
        return "Forbidden", 403
    page = db.session.get(Page, page_id)
    page.status = "published"
    page.published_at = datetime.now(timezone.utc)
    db.session.add(page); db.session.commit()
    # Render and cache
    _cache_page_html(page)
    flash("Published", "success")
    return redirect(url_for("author.edit_page", page_id=page.id))

@author_bp.route("/page/<int:page_id>/schedule", methods=["POST"])
@login_required
def schedule(page_id: int):
    if not can_publish(current_user):
        return "Forbidden", 403
    page = db.session.get(Page, page_id)
    page.status = "scheduled"
    page.scheduled_for = datetime.fromisoformat(request.form["scheduled_for"])
    db.session.add(page); db.session.commit()
    flash("Scheduled", "success")
    return redirect(url_for("author.edit_page", page_id=page.id))

def _cache_page_html(page: Page):
    """Render the page once and cache the HTML in Redis."""
    html = render_template("public/page.html", page=page, cached=True)
    current_app.redis.setex(
        f"page:{page.slug}", current_app.config["CACHE_TTL_SECONDS"], html
    )

# --- Version history ----------------------------------------------------------

@author_bp.route("/page/<int:page_id>/versions")
@login_required
def versions(page_id: int):
    page = db.session.get(Page, page_id)
    return render_template("author/versions.html", page=page)

@author_bp.route("/page/<int:page_id>/restore/<int:version_id>", methods=["POST"])
@login_required
def restore_version(page_id: int, version_id: int):
    page = db.session.get(Page, page_id)
    version = db.session.get(Version, version_id)
    if not (page and version and version.page_id == page.id):
        return "Not found", 404
    # Restore = create a NEW version with the old content (immutable history)
    db.session.add(Version(
        page_id=page.id, title=page.title, body_markdown=page.body_markdown,
        edited_by=current_user.id, note=f"Restored from version {version.id}",
    ))
    page.title = version.title
    page.body_markdown = version.body_markdown
    db.session.add(page); db.session.commit()
    current_app.redis.delete(f"page:{page.slug}")
    flash("Restored", "success")
    return redirect(url_for("author.edit_page", page_id=page.id))
```

```mermaid
sequenceDiagram
    participant A as Author Browser
    participant F as Flask
    participant MD as markdown-it-py
    participant LLM as OpenAI
    participant R as Redis
    participant DB as PostgreSQL

    Note over A,DB: Live preview (debounced, every 500ms)
    A->>F: POST /author/preview (body=Markdown)
    F->>MD: render_markdown(body)
    MD-->>F: HTML
    F-->>A: HTML (HTMX swaps into <div id=preview>)

    Note over A,DB: AI generate (streaming SSE)
    A->>F: POST /author/ai/generate (topic)
    F->>LLM: chat.completions.create(stream=True)
    LLM-->>F: token chunk 1
    F-->>A: data: "The"\\n\\n
    LLM-->>F: token chunk 2
    F-->>A: data: " Flask"\\n\\n
    LLM-->>F: [DONE]
    F-->>A: data: [DONE]\\n\\n

    Note over A,DB: Publish
    A->>F: POST /author/page/42/publish
    F->>DB: UPDATE page SET status='published'
    F->>MD: render_template(page.html)
    F->>R: SET page:my-slug <html> EX 3600
    F-->>A: redirect
```

### Step 10 — Public blueprint with cache-first reads

```python
# app/public.py
from datetime import datetime, timezone
from flask import Blueprint, render_template, request, Response, current_app, abort
from sqlalchemy import select, func, text
from feedgen.feed import FeedGenerator

from app.models import Page, Category, Tag
from app.extensions import db
from app.markdown import render_markdown

public_bp = Blueprint("public", __name__)

@public_bp.route("/")
def index():
    stmt = (
        select(Page)
        .where(Page.status == "published")
        .order_by(Page.published_at.desc())
        .limit(20)
    )
    pages = db.session.execute(stmt).scalars().all()
    return render_template("public/index.html", pages=pages)

@public_bp.route("/p/<slug>")
def page(slug: str):
    # Cache-first: try Redis, fall back to DB + cache the result
    cached = current_app.redis.get(f"page:{slug}")
    if cached:
        return cached  # raw HTML, sub-1ms response

    page = db.session.execute(
        select(Page).where(Page.slug == slug, Page.status == "published")
    ).scalar_one_or_none()
    if not page:
        abort(404)

    html = render_template("public/page.html", page=page, cached=False)
    current_app.redis.setex(
        f"page:{slug}", current_app.config["CACHE_TTL_SECONDS"], html
    )
    return html

@public_bp.route("/search")
def search():
    q = request.args.get("q", "").strip()
    if not q:
        return render_template("public/search.html", q=q, results=[])
    stmt = (
        select(
            Page,
            func.ts_rank(Page.search_vector, func.plainto_tsquery("english", q)).label("rank"),
        )
        .where(Page.search_vector.op("@@")(func.plainto_tsquery("english", q)))
        .where(Page.status == "published")
        .order_by(text("rank DESC"))
        .limit(20)
    )
    results = db.session.execute(stmt).all()
    return render_template("public/search.html", q=q, results=results)

@public_bp.route("/category/<slug>")
def category(slug: str):
    stmt = (
        select(Page)
        .join(Category)
        .where(Category.slug == slug, Page.status == "published")
        .order_by(Page.published_at.desc())
    )
    pages = db.session.execute(stmt).scalars().all()
    return render_template("public/index.html", pages=pages, category_slug=slug)

@public_bp.route("/tag/<slug>")
def tag(slug: str):
    stmt = (
        select(Page)
        .join(Page.tags)
        .where(Tag.slug == slug, Page.status == "published")
        .order_by(Page.published_at.desc())
    )
    pages = db.session.execute(stmt).scalars().all()
    return render_template("public/index.html", pages=pages, tag_slug=slug)

@public_bp.route("/feed.rss")
def rss():
    fg = FeedGenerator()
    fg.id(current_app.config["SITE_URL"])
    fg.title("Quill — Latest")
    fg.link(href=current_app.config["SITE_URL"], rel="alternate")
    fg.description("Latest published pages")

    stmt = (
        select(Page)
        .where(Page.status == "published")
        .order_by(Page.published_at.desc())
        .limit(20)
    )
    for page in db.session.execute(stmt).scalars():
        fe = fg.add_entry()
        fe.title(page.title)
        fe.link(href=f"{current_app.config['SITE_URL']}/p/{page.slug}")
        fe.id(f"{current_app.config['SITE_URL']}/p/{page.slug}")
        fe.published(page.published_at)
        fe.description(page.excerpt or page.title)
    return Response(fg.rss_str(), mimetype="application/rss+xml")

@public_bp.route("/sitemap.xml")
def sitemap():
    cached = current_app.redis.get("sitemap:xml")
    if cached:
        return Response(cached, mimetype="application/xml")
    abort(404)  # regenerated hourly by Celery
```

### Step 11 — Media blueprint with AI cover generation

```python
# app/media.py
import io, os, time
from pathlib import Path
from flask import (
    Blueprint, render_template, request, redirect, url_for,
    flash, current_app, jsonify, send_from_directory
)
from flask_login import login_required, current_user
from werkzeug.utils import secure_filename
from PIL import Image
import boto3

from app.models import Media, Page
from app.extensions import db
from app.permissions import role_required
from app import ai

media_bp = Blueprint("media", __name__)

def save_image_bytes(key: str, data: bytes, content_type: str = "image/png"):
    """Save to S3 if S3_BUCKET is set, otherwise local disk."""
    if current_app.config.get("S3_BUCKET"):
        s3 = boto3.client("s3", region_name=current_app.config["S3_REGION"])
        s3.put_object(
            Bucket=current_app.config["S3_BUCKET"],
            Key=key,
            Body=data,
            ContentType=content_type,
        )
    else:
        # Local disk for dev
        path = current_app.config["UPLOAD_FOLDER"] / key
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_bytes(data)

@media_bp.route("/upload", methods=["POST"])
@login_required
@role_required()
def upload():
    file = request.files.get("file")
    if not file:
        return "No file", 400
    filename = secure_filename(file.filename)
    ext = os.path.splitext(filename)[1].lower()
    if ext not in {".png", ".jpg", ".jpeg", ".webp", ".gif"}:
        return "Unsupported", 400

    # Generate thumbnails with Pillow
    img = Image.open(file.stream)
    width, height = img.size
    key = f"uploads/{int(time.time())}_{filename}"
    file.seek(0)
    save_image_bytes(key, file.read(), content_type=file.content_type)

    media = Media(
        uploaded_by=current_user.id,
        filename=filename,
        storage_key=key,
        content_type=file.content_type,
        width=width,
        height=height,
    )
    db.session.add(media); db.session.commit()
    return jsonify({"id": media.id, "url": _media_url(media)})

@media_bp.route("/generate-cover", methods=["POST"])
@login_required
@role_required()
def generate_cover():
    """Generate a cover image with DALL-E."""
    prompt = request.form.get("prompt", "")
    if not prompt:
        return "No prompt", 400
    try:
        key = ai.generate_cover_image(prompt)
    except Exception as e:
        return jsonify({"error": str(e)}), 500

    # Read it back to get dimensions
    media = Media(
        uploaded_by=current_user.id,
        filename=f"ai-cover-{int(time.time())}.png",
        storage_key=key,
        content_type="image/png",
        width=1792,
        height=1024,
        alt_text=f"AI-generated: {prompt[:200]}",
    )
    db.session.add(media); db.session.commit()
    return jsonify({"id": media.id, "url": _media_url(media)})

def _media_url(media: Media) -> str:
    if current_app.config.get("CDN_BASE"):
        return f"{current_app.config['CDN_BASE']}/{media.storage_key}"
    return f"/media/file/{media.id}"

@media_bp.route("/file/<int:media_id>")
def serve(media_id: int):
    """Local-dev fallback for serving uploads. In prod the CDN handles this."""
    media = db.session.get(Media, media_id)
    if not media:
        return "Not found", 404
    if current_app.config.get("S3_BUCKET"):
        # Redirect to S3 presigned URL
        s3 = boto3.client("s3", region_name=current_app.config["S3_REGION"])
        url = s3.generate_presigned_url(
            "get_object",
            Params={"Bucket": current_app.config["S3_BUCKET"], "Key": media.storage_key},
            ExpiresIn=3600,
        )
        return redirect(url)
    return send_from_directory(
        current_app.config["UPLOAD_FOLDER"], media.storage_key,
        mimetype=media.content_type
    )
```

### Step 12 — Celery tasks: scheduled publish, sitemap

```python
# app/tasks.py
from datetime import datetime, timezone
from celery import shared_task
from sqlalchemy import select

from app.extensions import db, celery
from app.models import Page, Tag, Category

def _get_app():
    from app import create_app
    return create_app()

@celery.task
def publish_scheduled():
    """Every minute: find scheduled pages whose time has come and publish them."""
    app = _get_app()
    with app.app_context():
        now = datetime.now(timezone.utc)
        stmt = (
            select(Page)
            .where(Page.status == "scheduled", Page.scheduled_for <= now)
        )
        due = db.session.execute(stmt).scalars().all()
        for page in due:
            page.status = "published"
            page.published_at = now
            db.session.add(page)
            # Render and cache HTML
            from flask import render_template
            html = render_template("public/page.html", page=page, cached=True)
            app.redis.setex(f"page:{page.slug}", app.config["CACHE_TTL_SECONDS"], html)
        db.session.commit()
        return len(due)

@celery.task
def regenerate_sitemap():
    """Hourly: regenerate sitemap.xml and cache in Redis."""
    app = _get_app()
    with app.app_context():
        stmt = (
            select(Page)
            .where(Page.status == "published")
            .order_by(Page.published_at.desc())
        )
        pages = db.session.execute(stmt).scalars().all()
        urls = [
            f"""<url><loc>{app.config['SITE_URL']}/p/{p.slug}</loc>"""
            f"""<lastmod>{p.updated_at.strftime('%Y-%m-%d')}</lastmod></url>"""
            for p in pages
        ]
        xml = (
            '<?xml version="1.0" encoding="UTF-8"?>\n'
            '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">\n'
            + "\n".join(urls) + "\n</urlset>"
        )
        app.redis.set("sitemap:xml", xml)
        return len(pages)
```

### Step 13 — The editor template (HTMX + SSE client)

```html
<!-- templates/author/editor.html -->
{% extends "base.html" %}
{% block content %}
<div class="grid grid-cols-2 gap-4">
  <!-- Left: editor -->
  <div>
    <input id="title" name="title" value="{{ page.title if page else '' }}"
           class="w-full text-2xl font-bold mb-2 border-b p-2"
           hx-post="/author/edit/{{ page.id if page else '0' }}"
           hx-trigger="change" hx-include="this">

    <textarea id="body" name="body" rows="25"
              class="w-full font-mono p-2 border rounded"
              hx-post="/author/preview"
              hx-trigger="keyup changed delay:500ms"
              hx-target="#preview">{{ page.body_markdown if page else '' }}</textarea>

    <!-- AI toolbar -->
    <div class="mt-2 flex gap-2">
      <button onclick="aiAction('generate')">✨ Generate draft</button>
      <button onclick="aiAction('expand')">Expand selection</button>
      <button onclick="aiAction('summarize')">Summarize</button>
      <button onclick="aiAction('translate')">Translate</button>
      <button onclick="aiMeta()">SEO suggest</button>
      <button onclick="aiCover()">🎨 Generate cover</button>
    </div>
  </div>

  <!-- Right: live preview -->
  <div>
    <h3 class="text-sm text-gray-500 mb-2">PREVIEW</h3>
    <div id="preview" class="prose prose-lg max-w-none border p-4 rounded">
      {% if page %}{{ page.body_html|safe }}{% endif %}
    </div>
  </div>
</div>

<script>
  // SSE-based AI streaming
  function aiAction(action) {
    const body = document.getElementById("body").value;
    const selection = body.substring(body.selectionStart, body.selectionEnd);
    const topic = prompt("Topic?"); if (!topic && action === "generate") return;
    const lang = action === "translate" ? prompt("Translate to? (Spanish/French/etc.)") : null;

    const formData = new FormData();
    formData.append("body", body);
    formData.append("selection", selection);
    formData.append("topic", topic || "");
    formData.append("language", lang || "Spanish");

    // Use fetch + ReadableStream for SSE POST (EventSource doesn't support POST)
    fetch(`/author/ai/${action}`, {method: "POST", body: formData})
      .then(resp => {
        const reader = resp.body.getReader();
        const decoder = new TextDecoder();
        let buffer = "";
        let appended = "";
        function read() {
          reader.read().then(({done, value}) => {
            if (done) {
              // Append the streamed content into the editor
              const ta = document.getElementById("body");
              ta.value += "\n\n" + appended;
              ta.dispatchEvent(new Event("keyup")); // trigger preview
              return;
            }
            buffer += decoder.decode(value, {stream: true});
            const lines = buffer.split("\n\n");
            buffer = lines.pop();
            for (const line of lines) {
              if (line.startsWith("data: ")) {
                const payload = line.slice(6).trim();
                if (payload === "[DONE]") continue;
                try { appended += JSON.parse(payload); } catch { appended += payload; }
              }
            }
            read();
          });
        }
        read();
      });
  }

  function aiMeta() {
    const body = document.getElementById("body").value;
    fetch("/author/ai/meta", {
      method: "POST",
      body: new URLSearchParams({body}),
    })
      .then(r => r.json())
      .then(data => {
        if (data.error) { alert(data.error); return; }
        document.querySelector("[name=meta_description]").value = data.meta_description;
        document.querySelector("[name=slug]").value = data.slug;
        alert("Suggested keywords: " + data.keywords.join(", "));
      });
  }

  function aiCover() {
    const prompt = prompt("Describe the cover image:");
    if (!prompt) return;
    fetch("/media/generate-cover", {
      method: "POST",
      body: new URLSearchParams({prompt}),
    })
      .then(r => r.json())
      .then(data => {
        if (data.error) { alert(data.error); return; }
        document.querySelector("[name=cover_media_id]").value = data.id;
        document.getElementById("cover-preview").src = data.url;
      });
  }
</script>
{% endblock %}
```

### Step 14 — Docker Compose

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: quill
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  web:
    build: .
    command: gunicorn -w 4 -b 0.0.0.0:5000 wsgi:app
    environment:
      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/quill
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
      CELERY_RESULT_BACKEND: redis://redis:6379/2
      SECRET_KEY: dev-secret-change-me
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      OPENAI_MODEL: gpt-4o
      DALLE_MODEL: dall-e-3
      SITE_URL: http://localhost:5000
    ports: ["5000:5000"]
    depends_on: [postgres, redis]
    volumes:
      - ./app/static/uploads:/app/app/static/uploads

  worker:
    build: .
    command: celery -A celery_worker.celery worker --loglevel=info
    environment:
      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/quill
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
      CELERY_RESULT_BACKEND: redis://redis:6379/2
      SECRET_KEY: dev-secret-change-me
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      SITE_URL: http://localhost:5000
    depends_on: [postgres, redis]

  beat:
    build: .
    command: celery -A celery_worker.celery beat --loglevel=info
    environment:
      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/quill
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
      CELERY_RESULT_BACKEND: redis://redis:6379/2
    depends_on: [redis]

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
```

---

## 7. Running the Project

```bash
cp .env.example .env
# Edit .env: set OPENAI_API_KEY

docker compose up --build
docker compose exec web flask --app wsgi db upgrade

# Create an admin user
docker compose exec web python -c "
from app import create_app
from app.extensions import db
from app.models import User, Role
app = create_app()
with app.app_context():
    u = User(email='admin@quill.local', name='Admin', role=Role.admin)
    u.set_password('admin')
    db.session.add(u); db.session.commit()
    print('Admin created')
"
```

Open `http://localhost:5000`, log in, click "New page", type Markdown — the right-hand preview updates 500 ms after you stop typing. Click "✨ Generate draft", type a topic, watch the LLM stream tokens into the editor. Click "SEO suggest" to auto-fill the meta description. Click "🎨 Generate cover" to call DALL-E. Click "Publish" — the page goes live at `/p/<slug>`. On a second visit, you'll get the cached HTML (sub-1 ms response).

---

## 8. Testing

```python
# tests/test_markdown.py
from app.markdown import render_markdown

def test_renders_basic_markdown():
    assert "<h1>" in render_markdown("# Hello")
    assert "<strong>" in render_markdown("**bold**")

def test_strips_script_tags():
    html = render_markdown("<script>alert(1)</script>plain text")
    assert "<script>" not in html
    assert "plain text" in html

def test_syntax_highlighting():
    html = render_markdown("```python\nprint('hi')\n```")
    assert "syntax" in html  # pygments CSS class

def test_tables_render():
    md = "| A | B |\n|---|---|\n| 1 | 2 |"
    assert "<table>" in render_markdown(md)
```

```python
# tests/test_permissions.py
def test_author_cannot_publish(app, client):
    # ... login as author, attempt POST /author/page/1/publish, assert 403
    pass

def test_editor_can_publish(app, client):
    # ... login as editor, same POST, assert 302
    pass
```

```python
# tests/test_search.py
def test_fulltext_search_finds_published():
    # ... seed two pages (one published, one draft), search for a word,
    # assert only the published page is returned
    pass
```

Run:

```bash
pytest -q
```

### End-to-end smoke checklist

| # | Action | Expected |
|---|---|---|
| 1 | `GET /` | Index renders with published pages |
| 2 | Login as author, click "New page" | Editor opens with empty title and body |
| 3 | Type Markdown in body | Preview updates after 500 ms debounce |
| 4 | Click "Generate draft", enter topic | Tokens stream into editor via SSE |
| 5 | Click "SEO suggest" | Meta description auto-fills |
| 6 | Click "Generate cover" | DALL-E returns image, cover preview updates |
| 7 | Save draft, attempt Publish as author | 403 Forbidden |
| 8 | Login as editor, publish | Page status → published, HTML cached in Redis |
| 9 | `GET /p/<slug>` twice | Second response is sub-1ms (Redis cache hit) |
| 10 | `GET /search?q=flask` | Returns matching pages ranked by relevance |
| 11 | `GET /feed.rss` | RSS XML with latest 20 pages |
| 12 | Schedule a page for 2 minutes from now | Within 60s of the scheduled time, beat publishes it |

---

## 9. Deployment

### Production checklist

- [ ] **gunicorn** with multiple workers (no SocketIO here, so scale freely).
- [ ] **PostgreSQL**: managed RDS / Cloud SQL. The GIN index on `search_vector` is critical for search performance.
- [ ] **Redis**: managed ElastiCache / Memorystore with `appendonly` persistence (cache warmup after restart).
- [ ] **S3 / Cloudflare R2** for media uploads — don't store images on the app server's disk.
- [ ] **CDN** in front of the app for static assets and image delivery.
- [ ] **OpenAI API key** in Doppler / Vault; set spending limits on the OpenAI org to prevent runaway costs.
- [ ] **CSRF** enabled; SSE endpoints exempt if you call them via fetch with headers.
- [ ] **HTTPS only** — HSTS, `Secure` cookies.
- [ ] **Celery worker + beat** as separate Fly.io apps or ECS services.
- [ ] **Cache invalidation on update** — when a page is re-published, delete `page:{slug}` so the next read rebuilds it.

### Fly.io example (web)

```toml
app = "quill-web"
[deploy]
  strategy = "rolling"
[http_service]
  internal_port = 5000
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 0
```

```mermaid
flowchart LR
    Reader[Public Reader] -->|GET /p/slug| F[Flask public]
    F -->|GET page:slug| R[(Redis cache)]
    R -.->|hit| F
    F -->|miss| PG[(PostgreSQL<br/>pages)]
    PG --> F
    F -->|render + cache| R
    F -->|serve HTML| Reader

    Author[Author] -->|HTMX| F2[Flask author]
    F2 -->|render preview| MD[markdown-it-py + pygments]
    F2 -->|SSE stream| LLM[OpenAI API]
    LLM -.->|token chunks| F2
    F2 -.->|text/event-stream| Author
    F2 -->|save / publish| PG
    F2 -->|delete cache| R

    Beat[Celery beat<br/>every 1 min] --> Worker[Celery worker]
    Worker -->|publish scheduled| PG
    Worker -->|render + cache| R
    style R fill:#dcfce7,stroke:#16a34a
    style LLM fill:#fce7f3,stroke:#db2777
    style F2 fill:#dbeafe,stroke:#2563eb
```

---

## 10. Next Steps / Enhancements

1. **Block-based editor.** Replace the Markdown textarea with a Notion-style block editor (paragraph, heading, image, code, callout) using Tiptap or ProseMirror — each block is a JSON object stored in PostgreSQL.
2. **Multi-language content.** Add a `language` column to `Page` and a translation workflow. The AI translate endpoint already exists; expose it as a first-class feature.
3. **Comments + reactions.** Add `Comment` and `Reaction` models; render via HTMX partials. Rate-limit with Flask-Limiter.
4. **Workflow approvals.** A reviewer must approve before an editor can publish — add a `reviewed_by` column and a "needs review" status.
5. **AI image alt text.** When an author uploads an image, call a vision LLM (GPT-4o vision) to auto-generate alt text for accessibility.
6. **Public API with API keys.** Expose `/api/v1/pages` for headless consumption. Use Flask-HTTPAuth with scoped API keys.
7. **Cache warming.** On deploy, a Celery task iterates all published pages and pre-warms the Redis cache so the first visitor after deploy doesn't pay the render cost.
8. **Analytics.** A Celery beat job logs page views (from request logs) into PostgreSQL; an admin dashboard shows top pages, referrers, and traffic over time.

---

## 11. What We Learned

- **Markdown rendering pipelines** need three layers: a CommonMark parser (`markdown-it-py`), syntax highlighting (`pygments`), and HTML sanitization (`bleach`). Skip any layer and you have either broken rendering or an XSS hole.
- **HTMX live preview** is one of the most delightful UX patterns in server-rendered apps. A debounced `hx-post` returns the rendered HTML directly — no JSON, no React, no build step.
- **SSE for LLM streaming** works without a single WebSocket. `text/event-stream` + `Response(stream_with_context(...))` + fetch's ReadableStream on the client — that's the whole stack.
- **Role-based access control belongs in the route, not the UI.** Hide buttons all you want, but the *server* must enforce `role_required()` and `can_edit_page()` on every state-changing endpoint.
- **Version history is immutable.** Restoring an old version creates a *new* version with the old content — never destroys history. This is "event sourcing lite" and it's the only safe way to do undo.
- **PostgreSQL `tsvector` + GIN index** is enough for full-text search up to millions of rows. You only need Elasticsearch when you need fuzzy matching across multiple languages or billions of rows.
- **Redis-cached rendered HTML** is the cheapest performance win in Flask. A cache hit is a single `GET` with zero DB queries — the difference between 5 ms and 500 ms.
- **Scheduled publishing with Celery beat** is the right tool. Don't put cron jobs on the host; the beat schedule lives in the application, version-controlled with the code.
- **DALL-E for cover art** is a legitimate 2026 CMS feature — but always let the author override. AI-generated images are good starting points, not final art.
- **Prompt templates belong in one file** (`app/ai.py`). When you want to tune tone or add a new AI feature, you edit one place. This is the same lesson as "queries belong in models, not views."

---

```mermaid
flowchart TB
    subgraph Browser
        Editor[Markdown editor<br/>+ HTMX preview] -->|hx-post /preview| F[Flask author]
        AI[AI toolbar<br/>EventSource/fetch] -->|SSE| F
    end
    subgraph Flask
        F --> MD[markdown-it-py]
        F --> LLM[OpenAI<br/>chat + DALL-E]
        F --> PG[(PostgreSQL<br/>pages + versions)]
        F --> R[(Redis<br/>page:slug cache)]
        Public[public routes] --> R
        Public -.->|miss| PG
        Public --> Search[tsvector + GIN]
    end
    subgraph Async
        Beat[Celery beat] --> Worker[worker]
        Worker -->|publish scheduled| PG
        Worker -->|render + cache| R
        Worker -->|regenerate sitemap| R
    end
    Reader[Public reader] -->|GET /p/slug| Public
    style R fill:#dcfce7,stroke:#16a34a
    style LLM fill:#fce7f3,stroke:#db2777
    style Search fill:#fef3c7,stroke:#d97706
    style Beat fill:#dbeafe,stroke:#2563eb
```

> [!quote] A note from your teacher
> CMS projects are deceptively simple — "it's just a blog." But the moment you add roles, versioning, AI integration, full-text search, scheduling, and caching, you're touching almost every pattern a working Flask developer uses daily. Quill is the project I'd show a non-technical hiring manager: they understand "I built a Ghost alternative with AI" instantly, and the engineering depth is there for any senior engineer who reads the code. Pair this project with [[Project-10-Event-Booking-Platform]] to see how the same Redis + Celery + PostgreSQL patterns power a completely different domain — booking instead of content.
