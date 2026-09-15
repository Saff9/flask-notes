---
title: "Modern Databases & Caching — The 2026 Landscape"
module: 15-Modern-Tooling
lesson_number: R4
estimated_time: "75–100 minutes (research reference)"
difficulty: intermediate
prerequisites:
  - "Lessons 12–15 (SQLAlchemy, models, relationships, migrations)"
  - "Lesson 22 (Celery background tasks — Redis was already introduced there)"
  - "Familiarity with SQL basics (SELECT, JOIN, WHERE)"
tags:
  - sqlalchemy2
  - postgresql
  - postgresql17
  - sqlite
  - litestream
  - turso
  - redis
  - valkey
  - caching
  - pgvector
  - vector-databases
  - timescaledb
  - alembic
  - pgbouncer
  - full-text-search
  - meilisearch
  - research
  - modern-tooling
related_lessons:
  - "[[Lesson-12-Database-Setup]]"
  - "[[Lesson-13-CRUD-And-Querying]]"
  - "[[Lesson-14-Relationships]]"
  - "[[Lesson-15-Flask-Migrate]]"
  - "[[Lesson-22-Email-And-Tasks]]"
  - "[[Lesson-23-Deployment]]"
---

# Modern Databases & Caching — The 2026 Landscape

> [!success] You built a blog with SQLite. Now meet the whole zoo.
> Lessons 12–15 taught you `db = SQLAlchemy(app)`, a `Post` model, and `flask db migrate`. That is *honestly* enough to ship most apps. But the database world has been on fire lately — PostgreSQL grew vector search, SQLite became production-ready at the edge, Redis forked, and a new generation of purpose-built stores (time-series, vector, full-text) arrived. This document maps the territory so you can pick the right tool without a 3-week rabbit hole.

#sqlalchemy #postgresql #redis #caching #2026 #databases

---

## 0. The Big Picture — What Changed Since You Learned `db.session`

If you learned databases from a 2020 Flask tutorial, you wrote something like:

```python
# Old (1.x) — the "Query API"
users = User.query.filter(User.name == 'alice').all()
```

That still works in 2026 — Flask-SQLAlchemy 3.x keeps `Model.query` alive for compatibility. But the *idiomatic* way now is:

```python
# New (2.x) — the "select() API"
from sqlalchemy import select
stmt = select(User).where(User.name == 'alice')
users = db.session.execute(stmt).scalars().all()
```

It looks longer. It is. But it composes, type-checks, works in async, and matches every other database library in the Python ecosystem (asyncpg, databases, SQLModel). **This is the single biggest shift in the Python data layer since 2010** — and it's just the headline. The same period gave us:

| Year | Event | Why it matters to you |
|------|-------|----------------------|
| 2023 | SQLAlchemy 2.0 released | New query API, `Mapped[T]` typing, async support |
| 2023 | Flask-SQLAlchemy 3.1 | Adopted SQLAlchemy 2.0 internally |
| 2023 | `pgvector` explosion | PostgreSQL became a vector DB overnight |
| 2023 | Valkey fork of Redis | Open-source Redis future after license change |
| 2024 | PostgreSQL 17 | Better logical replication, SQL/JSON, failover slots |
| 2024 | Turso matures | Distributed SQLite at the edge, for real |
| 2025 | SQLAlchemy 2.1 (preview) | Async improvements, removed legacy paths |
| 2026 | Everyone agrees: SQLite is production-ready | Litestream, Turso, Cloudflare D1 changed minds |

This document walks through each of these in turn. Buckle up.

---

## 1. Modern SQLAlchemy (2.x in 2026)

### 1.1 Why SQLAlchemy 2.0 was a big deal

SQLAlchemy 1.x had **two parallel APIs** running simultaneously:

- The **Core** API (`select()`, `table()`, `insert()`) — explicit, low-level, SQL-like
- The **ORM** `Query` API (`Session.query(User).filter(...)`) — Django-like, magical

They didn't compose. You couldn't take a Core `select()` and feed it into a `Query`, or vice versa. Code that mixed them was awkward. Async wasn't possible because sessions were deeply synchronous.

SQLAlchemy 2.0 **unified everything onto `select()`**. The ORM `Query` API was deprecated (and removed in 2.1). One statement style, one execution model, full async support, full type inference. It's the rewrite people had been asking for since 2015.

### 1.2 The new `select()` statement style

The mental model is now: **build a statement, execute it, unpack the result**.

```python
from sqlalchemy import select, func, and_, or_

# 1. Build the statement
stmt = (
    select(User)
    .where(User.active == True)
    .where(User.created_at > '2026-01-01')
    .order_by(User.name.asc())
    .limit(20)
)

# 2. Execute it
result = db.session.execute(stmt)

# 3. Unpack it
users = result.scalars().all()
```

The statement is **just an object** — you can pass it around, mutate it, log it, cache it. Compare to `Query`, which was bound to a session and lazy-evaluated at unpredictable times.

> [!teach] Teaching Tip — Show the pipeline, not just the syntax
> Students who only see `db.session.execute(select(...))` as "magic line" never internalise it. Draw the pipeline on the board:
> 1. **Statement** = a `Select` object (a query plan, not data)
> 2. **Execution** = `session.execute(stmt)` returns a `Result` (a cursor wrapper)
> 3. **Unpacking** = `.scalars()` strips the row tuple, `.all()` materialises to a list
>
> Each stage is a *different object*. Once students see the three stages, every variant (`scalar()`, `first()`, `one()`, `scalars().one_or_none()`) becomes obvious.

### 1.3 Type annotations with `Mapped[T]` and `mapped_column()`

The old way declared columns as class attributes:

```python
class User(db.Model):  # SQLAlchemy 1.x style
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(80), nullable=False)
    email = db.Column(db.String(120), nullable=True)
```

The new way uses **PEP 593 type annotations**:

```python
from typing import Optional
from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy import String

class User(db.Model):  # SQLAlchemy 2.x style
    __tablename__ = 'users'

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(80))
    email: Mapped[Optional[str]] = mapped_column(String(120), nullable=True)
```

Why this is better:

- **`Mapped[int]`** tells your IDE and mypy the Python type — autocomplete on `user.id` now works
- **`Optional[str]`** automatically sets `nullable=True` — no more forgetting the nullable flag
- **`mapped_column`** is one declarative entry point instead of `Column` vs `mapped_column` confusion
- **Relationships are typed**: `posts: Mapped[list["Post"]] = relationship(back_populates="author")` — now `user.posts` is `list[Post]` to your type checker, not `Any`

Relationships in 2.x:

```python
from sqlalchemy.orm import relationship
from typing import List

class User(db.Model):
    __tablename__ = 'users'
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(80))
    posts: Mapped[List["Post"]] = relationship(back_populates="author")

class Post(db.Model):
    __tablename__ = 'posts'
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(db.ForeignKey('users.id'))
    author: Mapped["User"] = relationship(back_populates="posts")
```

### 1.4 Async SQLAlchemy

This is the killer feature for 2026 — you can now write database code that *doesn't block the event loop*, which means Flask async views (`async def index()`) actually deliver concurrency.

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

# Async engine — note the driver: asyncpg for PostgreSQL
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/blog",
    pool_size=10,
    max_overflow=20,
)
AsyncSessionLocal = async_sessionmaker(engine, class_=AsyncSession)

# Async query
async def get_user(user_id: int) -> User | None:
    async with AsyncSessionLocal() as session:
        stmt = select(User).where(User.id == user_id)
        result = await session.execute(stmt)
        return result.scalar_one_or_none()
```

In a Flask async view:

```python
@app.route("/users/<int:user_id>")
async def show_user(user_id: int):
    user = await get_user(user_id)
    if user is None:
        abort(404)
    return render_template("user.html", user=user)
```

> [!warning] Async SQLAlchemy is not magic concurrency
> Async DB code only helps if *other* parts of your request are also async (HTTP calls to other services, async template rendering, etc.). If your view is `async` but only does one DB call, you've added complexity for no gain. Use async when you have **concurrent I/O** — multiple DB queries, HTTP calls, or fan-out work.

### 1.5 SQLAlchemy 2.0 + Flask-SQLAlchemy 3.x

Flask-SQLAlchemy 3.1+ fully supports the 2.0 API. Setup looks like:

```python
# app.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

def create_app():
    app = Flask(__name__)
    app.config["SQLALCHEMY_DATABASE_URI"] = "postgresql://user:pass@localhost/blog"
    app.config["SQLALCHEMY_ENGINE_OPTIONS"] = {
        "pool_size": 10,
        "max_overflow": 20,
        "pool_timeout": 30,
        "pool_recycle": 1800,  # 30 min — recycle connections before DB kills them
    }
    db.init_app(app)
    return app
```

### 1.6 The `db.session.execute()` pattern — and the helpers

The new execution surface has four entry points you'll use constantly:

| Method | Returns | Use when |
|--------|---------|----------|
| `db.session.execute(stmt)` | `Result` object | Full control — ORM entities, scalars, tuples |
| `db.session.scalars(stmt)` | `ScalarResult` | When you only want the entities (skip `.scalars()`) |
| `db.session.scalar(stmt)` | Single value | Aggregates like `count(*)`, max, min |
| `db.session.get(Model, pk)` | Entity or `None` | Primary key lookup (uses identity map cache) |

Examples:

```python
# Get one user by PK — uses the session identity map (cached!)
user = db.session.get(User, 42)

# Get a single scalar (e.g., count)
count = db.session.scalar(select(func.count()).select_from(User))

# Get all users as a list
users = db.session.scalars(select(User).order_by(User.name)).all()

# Get the first matching user, or None
user = db.session.scalars(select(User).where(User.email == email)).first()

# Get exactly one, error if not found or multiple found
user = db.session.scalars(select(User).where(User.id == 42)).one()

# Get one or None, error if multiple
user = db.session.scalars(select(User).where(User.email == email)).one_or_none()
```

### 1.7 Why the new API is better

| Old `Query` API | New `select()` API |
|---|---|
| Bound to a `Session` | Statement is a plain value — pass it, cache it, log it |
| `User.query.filter(...)` is implicit class state | `select(User)` is explicit and composable |
| No type inference on `.all()` | `scalars().all()` returns `list[User]` to mypy |
| No async support | Same statement works with `AsyncSession` |
| Two parallel APIs (Core + ORM) | One unified API across Core and ORM |
| Hard to test (session required) | Statement can be built & inspected without a session |

> [!teach] Teaching Tip — The migration is mostly mechanical
> When students panic at "all the queries changed!", show them the 1:1 mapping:
> - `User.query.all()` → `db.session.scalars(select(User)).all()`
> - `User.query.filter_by(name='x').first()` → `db.session.scalars(select(User).filter_by(name='x')).first()`
> - `User.query.get(42)` → `db.session.get(User, 42)`
> - `User.query.filter(...).count()` → `db.session.scalar(select(func.count()).select_from(User).where(...))`
>
> Run `pip install sqlalchemy[mypy]` and `ruff` will *auto-fix* most legacy queries with the `UP` rule set. Students see the migration is real but not scary.

### 1.8 Mermaid — SQLAlchemy 2.0 query flow

```mermaid
flowchart LR
    A["select(User)<br/>.where(...)<br/>.order_by(...)"] -->|"Statement<br/>(plain object)"| B["db.session.execute(stmt)"]
    B -->|"Result object<br/>(cursor wrapper)"| C{".scalars()<br/>or .first()<br/>or .one()"}
    C -->|"ScalarResult"| D["list[User]"]
    C -->|"single row"| E["User | None"]
    C -->|"aggregate"| F["int / str / etc."]

    style A fill:#e0f2fe,stroke:#0284c7
    style B fill:#fef3c7,stroke:#d97706
    style D fill:#dcfce7,stroke:#16a34a
    style E fill:#dcfce7,stroke:#16a34a
    style F fill:#dcfce7,stroke:#16a34a
```

#sqlalchemy #orm #async

---

## 2. Modern PostgreSQL (16/17 in 2026)

PostgreSQL has eaten the relational world. As of 2026 it's the default answer for "what database should I use?" — and the gap between "boring SQL database" and "everything you need" has closed almost completely. Vector search, full-text search, JSON, geospatial, time-series, pub/sub — Postgres does all of it natively now.

### 2.1 PostgreSQL 17 — what landed

PG 17 (released September 2024, in heavy production by 2026) brought:

- **SQL/JSON `JSON_TABLE`** — query JSON documents as if they were tables. The XML version has existed since 2016; JSON finally caught up.
- **Logical replication improvements** — failover slots keep replicas in sync during failover (no more "your replica broke during promotion" surprises)
- **Incremental backups** with `pg_basebackup` — only the changed pages
- **`MERGE ... RETURNING`** — finally, upserts that return rows
- **Faster vacuum** — index vacuum skips unchanged pages
- **Better `NULL` sorting** with `NULLS NOT DISTINCT` for unique indexes

```sql
-- PG 17: MERGE with RETURNING
MERGE INTO products p
USING incoming i ON p.sku = i.sku
WHEN MATCHED THEN UPDATE SET stock = p.stock + i.qty
WHEN NOT MATCHED THEN INSERT (sku, stock) VALUES (i.sku, i.qty)
RETURNING merge_action(), sku, stock;
```

### 2.2 JSONB — PostgreSQL's superpower

Every NoSQL database argument in 2016 ended with "but Postgres has `JSONB`". In 2026, that argument is over — Postgres won. `JSONB` is binary JSON with indexing, querying, and mutation:

```sql
CREATE TABLE events (id serial PRIMARY KEY, data jsonb);

-- Insert
INSERT INTO events (data) VALUES
  ('{"type": "signup", "user": {"id": 42, "plan": "pro"}, "ts": "2026-01-15"}');

-- Query with path operators
SELECT data->'user'->>'plan' AS plan
FROM events
WHERE data->>'type' = 'signup';

-- GIN index for fast JSON lookups
CREATE INDEX idx_events_data ON events USING GIN (data jsonb_path_ops);

-- Find all events where user.plan = 'pro'
SELECT * FROM events WHERE data @> '{"user": {"plan": "pro"}}';
```

In SQLAlchemy 2.x:

```python
from sqlalchemy.dialects.postgresql import JSONB

class Event(db.Model):
    __tablename__ = 'events'
    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[dict] = mapped_column(JSONB)

# Query JSONB
stmt = select(Event).where(Event.data['user']['plan'].astext == 'pro')
events = db.session.scalars(stmt).all()
```

### 2.3 Full-text search

PostgreSQL has had `tsvector` and `tsquery` for years, but in 2026 they're competitive with Elasticsearch for *most* use cases up to a few million documents:

```sql
-- Add a generated tsvector column
ALTER TABLE posts ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(body, ''))
  ) STORED;

-- GIN index for fast search
CREATE INDEX idx_posts_search ON posts USING GIN (search_vector);

-- Search
SELECT title, ts_rank(search_vector, query) AS rank
FROM posts, plainto_tsquery('english', 'flask tutorial') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

In SQLAlchemy:

```python
from sqlalchemy import func

stmt = (
    select(Post, func.ts_rank(Post.search_vector, func.plainto_tsquery('english', q)).label('rank'))
    .where(Post.search_vector.op('@@')(func.plainto_tsquery('english', q)))
    .order_by(text('rank DESC'))
    .limit(20)
)
```

### 2.4 PostGIS — geospatial

If you build anything with maps, locations, or "find me coffee shops within 2km", PostGIS is the standard:

```sql
CREATE EXTENSION postgis;

CREATE TABLE cafes (
  id serial PRIMARY KEY,
  name text,
  location geometry(Point, 4326)
);

-- Find cafes within 2km of a point
SELECT name, ST_Distance(location, ST_MakePoint(-122.42, 37.77)::geometry(Point, 4326)) AS dist_m
FROM cafes
WHERE ST_DWithin(location, ST_MakePoint(-122.42, 37.77)::geometry(Point, 4326), 2000)
ORDER BY dist_m;
```

### 2.5 pgvector — vectors for AI/ML (the hot one)

`pgvector` turned PostgreSQL into a vector database in 2022 and by 2026 it's everywhere. Store embedding vectors alongside your relational data — no second database needed.

```sql
CREATE EXTENSION vector;

CREATE TABLE documents (
  id serial PRIMARY KEY,
  content text,
  embedding vector(1536)  -- OpenAI text-embedding-3-small dimension
);

-- HNSW index for approximate nearest neighbour search
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- Semantic search: find documents similar to a query embedding
SELECT content, 1 - (embedding <=> $1) AS similarity
FROM documents
ORDER BY embedding <=> $1
LIMIT 10;
```

See **§9 Vector Databases** for the full Flask + OpenAI + pgvector example.

### 2.6 Materialized views

A materialized view is a *cached query* — useful when a report takes 30 seconds to compute but only needs to refresh hourly:

```sql
CREATE MATERIALIZED VIEW daily_sales AS
SELECT date_trunc('day', created_at) AS day,
       COUNT(*) AS order_count,
       SUM(total) AS revenue
FROM orders
GROUP BY 1;

CREATE UNIQUE INDEX ON daily_sales(day);

-- Refresh (CONCURRENTLY doesn't lock readers)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;
```

In Flask, schedule refreshes with a Celery beat task or a cron'd `psql` call.

### 2.7 CTEs and window functions

**Common Table Expressions (CTEs)** — named subqueries that make complex SQL readable:

```sql
WITH active_users AS (
  SELECT user_id, COUNT(*) AS post_count
  FROM posts
  WHERE created_at > NOW() - INTERVAL '30 days'
  GROUP BY user_id
)
SELECT u.name, au.post_count
FROM users u
JOIN active_users au ON u.id = au.user_id
ORDER BY au.post_count DESC;
```

**Window functions** — compute aggregates *across rows* without collapsing them:

```sql
-- Rank posts within each author by view count
SELECT title, author_id, views,
       RANK() OVER (PARTITION BY author_id ORDER BY views DESC) AS author_rank
FROM posts;
```

### 2.8 EXPLAIN ANALYZE — your best debugging friend

When a query is slow, `EXPLAIN ANALYZE` shows the actual plan and timings:

```sql
EXPLAIN ANALYZE
SELECT * FROM posts WHERE author_id = 42 ORDER BY created_at DESC LIMIT 10;
```

Look for:
- **Seq Scan** on a large table → missing index
- **Nested Loop** with high row estimates → missing join index
- **Sort** with `Sort Method: external merge Disk` → not enough `work_mem`
- **Hash Join** spilling to disk → same fix

### 2.9 Index types — when to use each

| Index type | Best for | Example |
|------------|----------|---------|
| **B-tree** (default) | Equality and range on sortable values | `WHERE created_at > '2026-01-01'` |
| **GIN** | Composite values (arrays, JSONB, tsvector) | `WHERE tags @> ARRAY['flask']` |
| **GiST** | Geospatial, range types, exclusion constraints | `WHERE location && box` |
| **BRIN** | Huge append-only tables (time-series) | `WHERE timestamp > '2026-01-01'` on a 1B-row log table |
| **Hash** | Equality only (rarely used; B-tree is usually better) | `WHERE id = 42` |
| **SP-GiST** | Non-balanced data (IPs, phone prefixes) | `WHERE ip <<= '192.168.0.0/16'` |
| **HNSW / IVFFlat** (pgvector) | Vector similarity search | `ORDER BY embedding <=> $1` |

#postgresql #postgres17 #jsonb #fulltext #postgis #pgvector

---

## 3. Modern SQLite (the hidden gem in 2026)

Here's the hottest take in databases right now: **SQLite is production-ready in 2026.** Three years ago, that sentence would get you laughed out of a hiring panel. Then Litestream, Turso, and Cloudflare D1 happened. SQLite quietly became the most-deployed database in the world (it's in every phone, every browser, every macOS app) and the *server* story caught up.

### 3.1 Why SQLite was "never used in production"

The traditional objections:

1. **Single writer** — only one connection can write at a time
2. **No network access** — it's a file, you can't connect to it remotely
3. **No replication** — single point of failure
4. **No horizontal scaling** — one file, one machine

All four were real. All four are now solvable.

### 3.2 WAL mode — concurrent reads while writing

**Write-Ahead Logging (WAL)** mode lets readers and writers coexist without blocking:

```python
# In SQLAlchemy:
app.config["SQLALCHEMY_ENGINE_OPTIONS"] = {
    "connect_args": {"timeout": 15},  # wait up to 15s on lock
}

# Or set it directly with a startup event:
from sqlalchemy import event

@event.listens_for(db.engine, "connect")
def set_sqlite_pragma(dbapi_conn, _):
    cursor = dbapi_conn.cursor()
    cursor.execute("PRAGMA journal_mode=WAL")
    cursor.execute("PRAGMA synchronous=NORMAL")  # safe in WAL, much faster
    cursor.execute("PRAGMA foreign_keys=ON")
    cursor.execute("PRAGMA busy_timeout=5000")
    cursor.close()
```

With WAL:
- Writers don't block readers
- Readers don't block writers
- Multiple readers can run concurrently
- Only writers serialise against each other (and `busy_timeout` handles contention)

### 3.3 Litestream — real-time replication

[Litestream](https://litestream.io/) runs as a sidecar that streams SQLite WAL changes to S3-compatible storage. If your server dies, you restore from S3 in seconds. It gives you the disaster-recovery story PostgreSQL has had for decades.

```bash
# Install
brew install litestream

# Configure (~/.litestream.yml)
dbs:
  - path: /var/data/blog.db
    replicas:
      - type: s3
        bucket: my-blog-backups
        path: db

# Run alongside your app
litestream replicate
```

For a Flask blog with 1000 daily readers, this is genuinely enough infrastructure. One server, one file, automatic off-site backups.

### 3.4 Turso — distributed SQLite at the edge

[Turso](https://turso.tech/) extends SQLite to be a *distributed* database — primary in one region, read replicas at the edge. Reads are sub-10ms anywhere in the world. The client library (`libsql`) is a SQLite fork with native HTTP replication built in.

```python
# Turso uses libsql — drop-in for sqlite3
import libsql_experimental as libsql

conn = libsql.connect(
    "libsql://my-blog.turso.io",
    auth_token="...",
)
# Use it like normal sqlite3 — but reads come from the closest edge replica
```

Turso's free tier includes 500 databases and 9GB of storage. **For multi-tenant SaaS**, where each customer gets their own database, this is a *ridiculous* deal — Postgres-per-customer gets expensive fast, SQLite-per-customer on Turso is basically free.

### 3.5 Cloudflare D1 — SQLite at the edge

Cloudflare D1 is SQLite running on Cloudflare's Workers runtime. Each query runs in the Cloudflare region closest to the user. Great for read-heavy edge apps:

```javascript
// In a Cloudflare Worker (Python workers in beta as of 2026)
export default {
  async fetch(request, env) {
    const { results } = await env.DB.prepare(
      "SELECT * FROM posts ORDER BY created_at DESC LIMIT 10"
    ).all();
    return Response.json(results);
  }
};
```

### 3.6 When SQLite beats PostgreSQL

| Use case | Pick SQLite when... |
|----------|---------------------|
| Personal blog / portfolio | One server, file backups are fine |
| Edge / IoT device | Local storage, occasional sync to cloud |
| Multi-tenant SaaS | Each tenant gets their own DB (Turso) |
| Read-heavy global app | D1 / Turso give edge reads for cheap |
| Development / testing | Zero-config, runs in-memory |
| Embedded in desktop/mobile | It's already there — use it |
| Single-region MVP | Skip Postgres ops entirely until you scale |

When to stick with PostgreSQL: heavy write concurrency (more than ~100 writes/sec sustained), complex replication topologies, native full-text search at scale, or you need a feature SQLite lacks (like materialized views).

### 3.7 SQLAlchemy + SQLite — production config

```python
import os
from sqlalchemy import event

def configure_sqlite(app):
    app.config["SQLALCHEMY_DATABASE_URI"] = os.environ["DATABASE_URL"]
    app.config["SQLALCHEMY_ENGINE_OPTIONS"] = {
        "connect_args": {"timeout": 15, "check_same_thread": False},
        "pool_pre_ping": True,
    }

    @event.listens_for(db.engine, "connect")
    def set_pragmas(dbapi_conn, _):
        cur = dbapi_conn.cursor()
        cur.execute("PRAGMA journal_mode=WAL")
        cur.execute("PRAGMA synchronous=NORMAL")
        cur.execute("PRAGMA temp_store=MEMORY")
        cur.execute("PRAGMA cache_size=-64000")  # 64MB cache
        cur.execute("PRAGMA mmap_size=268435456")  # 256MB memory-mapped I/O
        cur.execute("PRAGMA foreign_keys=ON")
        cur.execute("PRAGMA busy_timeout=10000")
        cur.close()
```

> [!teach] Teaching Tip — Don't apologise for SQLite
> For a long time, teachers introduced SQLite as "the toy database we'll replace with Postgres in production." Stop. For 80% of student projects — personal sites, hackathon apps, internal tools — SQLite + Litestream is genuinely the right answer in 2026. Teach it as a real choice, with the tradeoffs honestly stated, and your students will be better engineers.

#sqlite #wal #litestream #turso #cloudflare-d1 #edge

---

## 4. Modern Redis (7.x in 2026)

Redis started as "memcached with persistence." It's now a *category* of tool. Caching, sessions, pub/sub, queues, rate limiting, leaderboards, vector search (via RediSearch), time-series (via RedisTimeSeries) — Redis does all of it. In a Flask app, Redis is often your *second* infrastructure dependency after the database itself.

### 4.1 Redis is more than a cache

```mermaid
mindmap
  root((Redis 7.x))
    Cache
      GET/SET with TTL
      Cache-aside pattern
      Write-through
    Session store
      Flask-Session backend
      Distributed sessions
    Message broker
      Celery broker
      RQ queue
      Sidekiq-compatible
    Rate limiter
      INCR + EXPIRE
      Sliding window
      Token bucket
    Real-time
      Pub/Sub
      Streams
      Notifications
    Data structures
      Sorted sets (leaderboards)
      HyperLogLog (cardinality)
      Bitmaps (feature flags)
      Geospatial (GEORADIUS)
    Advanced
      Functions (Lua replacement)
      ACLs
      Cluster sharding
```

### 4.2 Each use case — Flask examples

**Cache** (the classic):

```python
from flask import Flask
import redis

app = Flask(__name__)
r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

@app.route("/posts/<int:post_id>")
def show_post(post_id):
    cache_key = f"post:{post_id}"
    cached = r.get(cache_key)
    if cached:
        return cached  # HTML string cached

    post = db.session.get(Post, post_id)
    html = render_template("post.html", post=post)
    r.setex(cache_key, 300, html)  # 5-minute TTL
    return html
```

**Session store**:

```python
from flask_session import Session

app.config["SESSION_TYPE"] = "redis"
app.config["SESSION_REDIS"] = redis.from_url("redis://localhost:6379/1")
Session(app)
# Now sessions persist across restarts and work with multiple Flask workers
```

**Rate limiter** (sliding window):

```python
def rate_limit(user_id: str, limit: int = 100, window: int = 60) -> bool:
    """Return True if request allowed, False if rate-limited."""
    key = f"rate:{user_id}:{int(time.time() // window)}"
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window)
    count, _ = pipe.execute()
    return count <= limit
```

Or use Flask-Limiter with Redis as the storage backend:

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app,
    key_func=get_remote_address,
    storage_uri="redis://localhost:6379/2",
    default_limits=["200 per hour", "50 per minute"],
)
```

**Pub/Sub for real-time**:

```python
# Publisher (in a request handler)
@app.route("/notify/<channel>", methods=["POST"])
def notify(channel):
    r.publish(channel, request.form["message"])
    return "ok"

# Subscriber (in a background worker, NOT in Flask request context)
pubsub = r.pubsub()
pubsub.subscribe("chat")
for msg in pubsub.listen():
    if msg["type"] == "message":
        print(msg["data"])
```

**Streams (Kafka-lite)**:

```python
# Producer
r.xadd("events", {"user": "alice", "action": "login"})

# Consumer group
r.xgroup_create("events", "workers", id="$")
while True:
    messages = r.xreadgroup("workers", "worker-1", {"events": ">"}, count=10, block=1000)
    for stream, msg_list in messages:
        for msg_id, data in msg_list:
            process(data)
            r.xack("events", "workers", msg_id)
```

**Sorted sets — leaderboards**:

```python
r.zadd("leaderboard", {"alice": 1500, "bob": 2000, "carol": 1800})
top_10 = r.zrevrange("leaderboard", 0, 9, withscores=True)
# [('bob', 2000.0), ('carol', 1800.0), ('alice', 1500.0)]
```

**HyperLogLog — cardinality estimation**:

```python
# Count unique visitors — uses ~12KB regardless of cardinality
r.pfadd("unique_visitors:2026-01-15", "user1", "user2", "user3", "user4")
estimated = r.pfcount("unique_visitors:2026-01-15")  # ≈ 4
```

### 4.3 Redis 7.x features

- **Functions** — replace ad-hoc Lua scripts with named, reloadable server-side functions
  ```lua
  --!lua name=mylib
  redis.register_function('my_hset', function(keys, args)
      -- custom logic here
  end)
  ```
- **ACLs** — fine-grained user permissions (`+@read -@write` etc.)
- **Sharding via Cluster** — built-in horizontal scaling
- **Multi-part AOF** — append-only file in multiple parts, faster to rewrite
- **Better `LATENCY` introspection**

### 4.4 Redis Cluster vs Sentinel

| Feature | Redis Sentinel | Redis Cluster |
|---------|----------------|---------------|
| Goal | High availability (failover) | HA + horizontal scaling |
| Sharding | No (single primary) | Yes (data split across nodes) |
| Clients | Any Redis client | Cluster-aware client required |
| Setup complexity | Medium | High |
| Use when | Need HA, traffic fits on one box | Need to scale beyond one box's RAM/CPU |

For most Flask apps: start with single-instance Redis + Sentinel for failover. Move to Cluster when you're caching more than ~50GB or have hit CPU ceiling on a single Redis.

### 4.5 Redis alternatives — the great fork era

In March 2024, Redis Ltd. changed their license from BSD to a more restrictive dual license. The community forked:

| Project | Story | Status in 2026 |
|---------|-------|----------------|
| **Valkey** | Linux Foundation fork, backed by AWS/Google/Oracle | Default choice for managed Redis on AWS ElastiCache |
| **DragonflyDB** | Multi-threaded C++ reimplementation, ~30× faster on some workloads | Strong for write-heavy, RAM-constrained setups |
| **KeyDB** | Multi-threaded Redis fork (Snap acquisition, now independent again) | Niche — being absorbed into Valkey ecosystem |
| **Redis (original)** | Still exists, still popular, license change applies above thresholds | Most on-prem deployments still on it; cloud is moving to Valkey |

**The practical answer**: if you're on AWS ElastiCache in 2026, you're probably already running Valkey. If you're on Upstash, it's Redis. If you self-host and want to stay open-source, use Valkey. The wire protocol is identical — your Flask code doesn't change.

### 4.6 Managed Redis options

| Provider | Pricing model | Notes |
|----------|---------------|-------|
| AWS ElastiCache | Per-instance-hour | Now offers Valkey; solid but expensive |
| Upstash | Per-request (serverless) | Great for low-traffic apps, HTTP API too |
| Redis Cloud | Per-shard + throughput | Official Redis; multi-AZ included |
| Aiven | Per-node | Open-source versions, multi-cloud |
| Memorystore (GCP) | Per-GB-hour | Simple pricing, tight GCP integration |
| Upstash Valkey | Same as Upstash | The Valkey equivalent on Upstash |

> [!teach] Teaching Tip — Show Redis as five tools, not one
> When introducing Redis, don't teach "Redis is a cache" first. Teach **five things Redis does**, with one example each — cache, session, rate limiter, queue, leaderboard. Students who see only the cache use case never reach for Redis when they need a rate limiter and end up building it badly with `dict`. The whole point of Redis is *it does lots of things well enough*.

#redis #caching #valkey #rate-limiting #pubsub

---

## 5. Modern ORMs and Database Tools (2026)

### 5.1 The ORM landscape

SQLAlchemy is the default — but it's no longer the only credible option. Here's the 2026 field:

| ORM | Style | Async? | Best when |
|-----|-------|--------|-----------|
| **SQLAlchemy 2.x** | Explicit, Core + ORM | Yes | Default choice, complex schemas, type safety |
| **Tortoise ORM** | Django-like, async-first | Native | Async web apps, simpler than SQLAlchemy |
| **SQLModel** | Pydantic + SQLAlchemy | Yes (via SQLAlchemy) | FastAPI apps, single source of truth for models |
| **Pony ORM** | DSL with Python-generator queries | No (sync) | Elegant queries, prototyping |
| **Prisma Client Python** | TypeScript's Prisma port | Yes | Full-stack teams sharing schema with TS frontend |
| **Peewee** | Small, Django-like | No | Small scripts, learning |

### 5.2 Tortoise ORM example

```python
from tortoise import Tortoise, fields
from tortoise.models import Model

class User(Model):
    id = fields.IntField(pk=True)
    name = fields.CharField(max_length=80)
    email = fields.CharField(max_length=120, null=True)

    class Meta:
        table = "users"

async def init():
    await Tortoise.init(db_url="postgres://user:pass@localhost/blog", modules={"models": ["app"]})

async def get_user(user_id: int):
    return await User.get_or_none(id=user_id)
```

### 5.3 SQLModel example (FastAPI style)

```python
from typing import Optional
from sqlmodel import SQLModel, Field, Session, select

class User(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    email: Optional[str] = None

# Same class works for API validation AND database persistence
with Session(engine) as session:
    user = User(name="alice", email="a@b.com")
    session.add(user)
    session.commit()
    session.refresh(user)
```

### 5.4 Pony ORM example

```python
from pony.orm import Database, Required, Optional, db_session

db = Database()
db.bind("postgres", user="user", password="pass", database="blog")

class User(db.Entity):
    name = Required(str, 80)
    email = Optional(str, 120)
    posts = Set("Post")

class Post(db.Entity):
    title = Required(str, 200)
    author = Required(User)

db.generate_mapping(create_tables=True)

@db_session
def alice_posts():
    return [p.title for p in select(p for p in Post if p.author.name == "alice")]
```

Pony's generator-expression syntax is the most readable query DSL in Python. The tradeoff: it's sync-only, slower, and not type-checked.

### 5.5 Decision matrix

| If you want... | Use |
|----------------|-----|
| Maximum control + mature ecosystem | SQLAlchemy 2.x |
| Async-first, Django ergonomics | Tortoise |
| FastAPI + single model source | SQLModel |
| Most readable queries, sync OK | Pony |
| TS/Python shared schema (full-stack team) | Prisma |
| Tiny script, learn SQL | raw `sqlite3` / `asyncpg` |

### 5.6 Database GUI tools

| Tool | Price | Best for |
|------|-------|----------|
| **DBeaver** | Free (Community) / paid (Pro) | Every database, every platform, swiss-army knife |
| **TablePlus** | Free tier / $59+ license | Beautiful, fast, low-friction, macOS-first |
| **DataGrip** | $89/yr (JetBrains) | Heavy DB work, integrates with PyCharm |
| **pgAdmin** | Free | PostgreSQL only, web UI, ugly but functional |
| **Studio for SQLite** | Free | SQLite browsing |
| **Redis Insight** | Free | Visualising Redis data, slowlog analysis |

> [!teach] Teaching Tip — Make students use a GUI from day 1
> Forcing students to use `psql` or `sqlite3` for everything creates a "databases are scary" association. Install DBeaver on day one, show them their tables as rows, let them click around. Save the CLI for when it's the right tool (scripting, server access).

#sqlalchemy #tortoise #sqlmodel #pony #orm

---

## 6. Modern Connection Pooling (2026)

### 6.1 Why pooling matters

Every PostgreSQL connection costs ~10MB of server RAM and ~50ms to establish. If you open a new connection per request, your app falls over at ~100 concurrent users. **Connection pooling reuses connections** across requests so the expensive setup happens once.

The problem compounds in modern setups:
- Serverless functions (AWS Lambda) spin up hundreds of parallel instances — each opens its own connection → PostgreSQL's `max_connections` (default 100) is exhausted instantly
- Containerised apps with 20 replicas × 10 pool size = 200 connections
- Long-running Flask processes need to recycle connections before DB-side timeouts kill them

### 6.2 SQLAlchemy's built-in pool

SQLAlchemy ships with `QueuePool` (the default for non-SQLite). Tune it via engine options:

```python
app.config["SQLALCHEMY_ENGINE_OPTIONS"] = {
    "pool_size": 10,         # persistent connections kept open
    "max_overflow": 20,      # extra connections allowed under load
    "pool_timeout": 30,      # seconds to wait for a free connection
    "pool_recycle": 1800,    # recycle connections older than 30 min
    "pool_pre_ping": True,   # check connection is alive before use
}
```

Rule of thumb: `pool_size + max_overflow` × number of app workers ≤ `max_connections` × 0.8 (leave headroom for migrations, admin queries).

### 6.3 PgBouncer — the standard external pooler

[PgBouncer](https://www.pgbouncer.org/) sits between your app and PostgreSQL. Hundreds of app connections multiplex onto a small number of real database connections. Critical for serverless.

```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
blog = host=127.0.0.1 port=5432 dbname=blog

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
```

**Transaction mode vs session mode**:

| Mode | How it works | Tradeoff |
|------|--------------|----------|
| **Session** | One client = one server connection for the session's lifetime | No multiplexing, but supports session features (temp tables, SET, LISTEN) |
| **Transaction** | Server connection returns to pool at transaction end | High multiplexing; no session state, no prepared statements (until PG 17+) |
| **Statement** | Returns after each statement | Only safe for autocommit; rarely used |

**Transaction mode** is what you want for serverless. The catch: SQLAlchemy needs `pool_recycle` configured to play nicely, and you must disable prepared statement caching:

```python
app.config["SQLALCHEMY_ENGINE_OPTIONS"] = {
    "poolclass": "sqlalchemy.pool.NullPool",  # let PgBouncer do the pooling
    "connect_args": {"prepare_threshold": 0},  # disable prepared statements (PG 17+ changes this)
}
```

### 6.4 pgcat — the modern alternative

[pgcat](https://github.com/postgresml/pgcat) is a newer Rust-based pooler with multi-tenant sharding built in. Worth a look if you need to shard PostgreSQL across multiple instances.

### 6.5 Connection pooling in serverless — essential

In AWS Lambda / Google Cloud Functions / Cloudflare Workers, each concurrent invocation is a separate process. Without PgBouncer, 200 concurrent Lambda calls = 200 PostgreSQL connections = 💀.

Standard pattern:

```
Lambda → RDS Proxy (AWS managed PgBouncer) → RDS PostgreSQL
```

Or self-hosted:

```
Lambda → PgBouncer (on a small EC2 instance) → RDS PostgreSQL
```

### 6.6 Mermaid — connection pooling architecture

```mermaid
flowchart TB
    subgraph Clients["Flask app (20 workers)"]
        C1["Worker 1<br/>pool=10"]
        C2["Worker 2<br/>pool=10"]
        C3["..."]
        C4["Worker 20<br/>pool=10"]
    end

    subgraph Pooler["PgBouncer (transaction mode)"]
        P1["Client slots: 1000"]
        P2["Server pool: 25"]
    end

    subgraph DB["PostgreSQL"]
        D1["max_connections = 100"]
        D2["Actual usage: ~25 connections"]
    end

    C1 --> P1
    C2 --> P1
    C3 --> P1
    C4 --> P1
    P1 --> P2
    P2 --> D2

    style Clients fill:#e0f2fe,stroke:#0284c7
    style Pooler fill:#fef3c7,stroke:#d97706
    style DB fill:#dcfce7,stroke:#16a34a
```

#pgbouncer #connection-pooling #serverless #performance

---

## 7. Modern Database Migrations (2026)

### 7.1 Alembic — the standard

[Alembic](https://alembic.sqlalchemy.org/) is SQLAlchemy's migration tool. Flask-Migrate wraps it with `flask db` commands:

```bash
flask db init          # create migrations/ folder
flask db migrate -m "add posts table"   # autogenerate migration
flask db upgrade       # apply pending migrations
flask db downgrade     # undo last migration
flask db current       # show current revision
flask db history       # show migration history
```

### 7.2 A modern migration file

```python
"""add posts table

Revision ID: a1b2c3d4e5f6
Revises: 9z8y7x6w5v4u
Create Date: 2026-01-15 10:30:00.000000
"""
from alembic import op
import sqlalchemy as sa

revision = "a1b2c3d4e5f6"
down_revision = "9z8y7x6w5v4u"
branch_labels = None
depends_on = None

def upgrade():
    op.create_table(
        "posts",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("title", sa.String(200), nullable=False),
        sa.Column("body", sa.Text(), nullable=True),
        sa.Column("author_id", sa.Integer(), sa.ForeignKey("users.id"), nullable=False),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )
    op.create_index("ix_posts_author_id", "posts", ["author_id"])

def downgrade():
    op.drop_index("ix_posts_author_id", table_name="posts")
    op.drop_table("posts")
```

### 7.3 Best practices

| Rule | Why |
|------|-----|
| One change per migration | Easier to roll back; cleaner history |
| Always provide `downgrade()` | So you can roll back |
| Test both up *and* down in CI | Migrations that can't roll back will bite you in prod |
| Use `op.batch_alter_table` for SQLite | SQLite can't ALTER most things; batch mode recreates the table |
| Don't autogenerate without reviewing | Alembic misses constraints, server defaults, partial indexes |
| Separate data migrations from schema migrations | Schema changes should be instant; data migrations may need background work |
| Run migrations forward-only in prod | Downgrades are for dev; in prod, write a *new* migration that reverses |

### 7.4 Batch mode for SQLite

SQLite can't drop columns, alter types, or add constraints to existing tables. Batch mode handles this by recreating the table:

```python
def upgrade():
    with op.batch_alter_table("posts") as batch_op:
        batch_op.alter_column("title", existing_type=sa.String(200), nullable=False)
        batch_op.add_column(sa.Column("slug", sa.String(200)))
        batch_op.drop_column("legacy_field")
```

### 7.5 Expand-and-contract — zero-downtime migrations

The big idea: never make a change that breaks the running app. Split it across multiple deploys.

```mermaid
sequenceDiagram
    participant App as App (v1)
    participant DB as Database
    participant App2 as App (v2)

    Note over App,DB: Phase 1: EXPAND (additive only)
    App->>DB: Migration 1: ADD COLUMN email_new (nullable)
    App->>DB: App v1 still writes to old column

    Note over App,DB: Phase 2: MIGRATE (backfill + dual-write)
    App2->>DB: Deploy app v2: dual-writes to old + new
    App2->>DB: Backfill: UPDATE users SET email_new = email
    App2->>DB: App v2 reads from new column

    Note over App,DB: Phase 3: CONTRACT (remove old)
    App2->>DB: Migration 2: DROP COLUMN email
    Note over App2: App v2 only uses email_new
```

This pattern lets you change schema with **zero downtime** — no maintenance windows, no read-only mode, no scary "DROP COLUMN" that locks the table for an hour.

### 7.6 When to write raw SQL migrations

Alembic's `op` API is great for schema, but for some things raw SQL is cleaner:

- Complex data backfills (use `op.execute()`)
- PostgreSQL-specific features (extensions, partial indexes with WHERE)
- Performance-sensitive operations (e.g., `CREATE INDEX CONCURRENTLY`)

```python
def upgrade():
    op.execute("CREATE EXTENSION IF NOT EXISTS vector")
    op.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    op.execute("CREATE INDEX CONCURRENTLY ON posts USING gin(title gin_trgm_ops)")
    # CONCURRENTLY doesn't work inside a transaction — set:
    # with op.get_context().autocommit_block():
```

### 7.7 CI testing of migrations

Run this in your GitHub Actions:

```bash
# Apply all migrations to a fresh DB, then downgrade all the way back
flask db upgrade
flask db downgrade base
flask db upgrade
```

If any step fails, the migration is broken. Catch it before deploy, not during.

#alembic #migrations #flask-migrate #zero-downtime

---

## 8. Modern Full-Text Search (2026)

### 8.1 The options

| Tool | Type | Best when |
|------|------|-----------|
| **PostgreSQL FTS** | Built-in | < 1M docs, simple queries, want one DB |
| **Elasticsearch / OpenSearch** | Dedicated | Millions of docs, complex analytics, log search |
| **Meilisearch** | Self-hosted, lightweight | Up to ~10M docs, great DX, sub-50ms search |
| **Typesense** | Self-hosted, fast | Like Meilisearch but tuning-focused |
| **Whoosh** | Pure Python, embedded | Small datasets, no extra server |
| **Algolia** | SaaS | Best DX, expensive at scale |
| **Tantivy** (via `tantivy-py`) | Rust-based embedded | High-perf embedded, niche |

### 8.2 Decision matrix

```mermaid
flowchart TD
    Q["Need full-text search?"]
    Q -->|"No, just LIKE queries"| None["Use Postgres ILIKE<br/>or trigram index"]
    Q -->|"Yes"| Size{"How many<br/>documents?"}

    Size -->|"< 100K"| PG["PostgreSQL FTS<br/>(tsvector + GIN)"]
    Size -->|"100K–10M"| Meili["Meilisearch<br/>or Typesense"]
    Size -->|"> 10M or complex analytics"| ES["Elasticsearch /<br/>OpenSearch"]

    Meili --> Budget{"Budget for SaaS?"}
    Budget -->|"Yes"| Algolia["Algolia<br/>(best DX, expensive)"]
    Budget -->|"No"| Meili2["Self-host Meilisearch"]

    PG --> Note["Start here.<br/>Migrate later if needed."]
    style Note fill:#fef3c7,stroke:#d97706
    style PG fill:#dcfce7,stroke:#16a34a
```

### 8.3 Meilisearch — the modern default

If Postgres FTS isn't enough, Meilisearch is what most teams reach for. Single binary, instant search, typo tolerance out of the box:

```bash
brew install meilisearch
meilisearch --master-key=your-key
```

```python
import meilisearch

client = meilisearch.Client("http://localhost:7700", "your-key")

# Index documents
client.index("posts").add_documents([
    {"id": 1, "title": "Flask Tutorial", "body": "Learn Flask step by step"},
    {"id": 2, "title": "Async Flask", "body": "Using async views in Flask 3.x"},
])

# Search
results = client.index("posts").search("flask", {"limit": 10})
```

### 8.4 PostgreSQL FTS — start here

For most Flask apps, PostgreSQL FTS is enough. The setup is in §2.3 above. The 80/20:

- Add a `tsvector` generated column
- GIN index it
- Query with `@@` and `plainto_tsquery`
- Done — no new infrastructure

> [!teach] Teaching Tip — The "start with Postgres FTS" lecture
> Students always want to add Elasticsearch on day one. Show them the math: a 50K-row blog posts table with a GIN index on a generated `tsvector` column searches in <5ms. That's faster than the network roundtrip to an Elasticsearch cluster would be. **Start with Postgres FTS. Add a search engine when you have evidence it's slow.** Same lesson as caching: don't pre-optimize.

#full-text-search #meilisearch #elasticsearch #postgresql-fts

---

## 9. Vector Databases for AI (2026 — hot topic!)

### 9.1 What is a vector database?

Large language models (and other embedding models) turn text into **vectors** — lists of ~1536 floats that capture semantic meaning. Two pieces of text with similar meaning have *similar vectors*. A vector database stores these and answers: "given this query vector, give me the K most similar stored vectors."

This powers:
- **Semantic search** — "find docs about the same topic as this query"
- **RAG (Retrieval-Augmented Generation)** — feed relevant context to an LLM
- **Recommendation systems** — "users like this also liked..."
- **Deduplication** — "is this image basically the same as one we already have?"

### 9.2 The vector DB landscape

| Tool | Type | Best when |
|------|------|-----------|
| **pgvector** | PostgreSQL extension | You already have Postgres, < 10M vectors |
| **Pinecone** | SaaS | Don't want to operate infra, pay per query |
| **Weaviate** | Open-source, self-host or cloud | Hybrid search (vectors + filters + GraphQL) |
| **Chroma** | Open-source, embedded | Local dev, prototypes, small datasets |
| **Qdrant** | Rust-based, fast | High performance, payload filtering |
| **Milvus** | Distributed, scale | Billions of vectors |
| **LanceDB** | Embedded, columnar | Local-first AI apps |

### 9.3 Why pgvector is usually the right answer

If you already have PostgreSQL, **start with pgvector**. Reasons:

- No new infrastructure
- Your relational data and vectors live together (one JOIN, one transaction)
- Backup, security, replication — already solved
- Up to ~10M vectors with HNSW indexes, perfectly performant

Move to a dedicated vector DB when you hit ~10M+ vectors or need features pgvector lacks (e.g., hybrid search with built-in BM25).

### 9.4 Flask + SQLAlchemy + pgvector — full example

Setup:

```bash
pip install pgvector sqlalchemy psycopg2-binary openai
# In Postgres:
# CREATE EXTENSION vector;
```

Model:

```python
from pgvector.sqlalchemy import Vector
from sqlalchemy import String, Text
from sqlalchemy.orm import Mapped, mapped_column

class Document(db.Model):
    __tablename__ = "documents"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text)
    embedding: Mapped[list[float]] = mapped_column(Vector(1536))  # OpenAI dim

    def __repr__(self):
        return f"<Document {self.title}>"
```

Index (HNSW for approximate nearest neighbour, much faster than exact):

```python
from sqlalchemy import Index

# After model is defined:
Index(
    "idx_documents_embedding",
    Document.embedding,
    postgresql_using="hnsw",
    postgresql_with={"m": 16, "ef_construction": 64},
    postgresql_ops={"embedding": "vector_cosine_ops"},
).create(db.engine)
```

Ingest:

```python
from openai import OpenAI
client = OpenAI()

def embed(text: str) -> list[float]:
    """Get embedding vector from OpenAI."""
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
    )
    return response.data[0].embedding

def ingest_document(title: str, content: str):
    doc = Document(
        title=title,
        content=content,
        embedding=embed(content[:8000]),  # truncate to model max
    )
    db.session.add(doc)
    db.session.commit()
```

Semantic search:

```python
def semantic_search(query: str, limit: int = 5) -> list[Document]:
    query_embedding = embed(query)
    stmt = (
        select(Document)
        .order_by(Document.embedding.cosine_distance(query_embedding))
        .limit(limit)
    )
    return list(db.session.scalars(stmt))

# In a Flask view:
@app.route("/search")
def search():
    q = request.args.get("q", "")
    if not q:
        return render_template("search.html", results=[], q="")
    results = semantic_search(q)
    return render_template("search.html", results=results, q=q)
```

With similarity scores:

```python
from sqlalchemy import select, literal

def semantic_search_with_scores(query: str, limit: int = 5):
    query_embedding = embed(query)
    similarity = 1 - Document.embedding.cosine_distance(literal(query_embedding))
    stmt = (
        select(Document, similarity.label("similarity"))
        .order_by(Document.embedding.cosine_distance(literal(query_embedding)))
        .limit(limit)
    )
    return db.session.execute(stmt).all()
```

### 9.5 RAG pattern — putting it together

```python
@app.route("/ask", methods=["POST"])
def ask():
    question = request.form["question"]

    # 1. Find relevant documents via vector search
    relevant = semantic_search(question, limit=3)
    context = "\n\n".join(d.content for d in relevant)

    # 2. Build a prompt with the retrieved context
    prompt = f"""Answer the question based on this context:

{context}

Question: {question}
Answer:"""

    # 3. Send to LLM
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
    )
    answer = response.choices[0].message.content

    # 4. Show user the answer + sources
    return render_template("answer.html", answer=answer, sources=relevant)
```

### 9.6 Mermaid — Vector search flow

```mermaid
flowchart LR
    U["User query<br/>'how do I deploy Flask?'"] --> E1["Embedding model<br/>(OpenAI / local)"]
    E1 --> V1["Query vector<br/>[0.12, -0.45, ...]"]

    subgraph DB["PostgreSQL + pgvector"]
        D1["Doc 1: 'Deploying Flask with Gunicorn'<br/>[0.11, -0.43, ...]"]
        D2["Doc 2: 'Flask tutorials'<br/>[0.09, -0.31, ...]"]
        D3["Doc 3: 'Docker setup'<br/>[0.41, 0.22, ...]"]
        I["HNSW index"]
    end

    V1 --> I
    I --> R["cosine_distance(query, doc)<br/>ORDER BY distance"]
    R --> Top["Top-K most similar<br/>Doc 1, Doc 2"]
    Top --> C["Context for LLM"]
    C --> LLM["LLM generates answer"]
    LLM --> Out["Answer + sources"]

    style DB fill:#e0f2fe,stroke:#0284c7
    style LLM fill:#fef3c7,stroke:#d97706
    style Out fill:#dcfce7,stroke:#16a34a
```

#pgvector #vector-databases #ai #rag #embeddings #llm

---

## 10. Time-Series Data (2026)

### 10.1 When you need time-series

A "time-series database" (TSDB) is optimised for workloads where:
- Every row has a timestamp
- You append far more than you update
- You query "average over the last hour" not "give me row 42"
- Data volume grows linearly with time (so retention matters)

Examples: IoT sensors, application metrics, financial ticks, server monitoring, user activity events.

### 10.2 TimescaleDB — Postgres extension

[TimescaleDB](https://www.timescale.com/) extends PostgreSQL with **hypertables** — tables that look normal but are auto-partitioned by time. You get Postgres features (SQL, JOINs, transactions) plus TSDB performance.

```sql
CREATE EXTENSION timescaledb;

CREATE TABLE sensor_readings (
  time        TIMESTAMPTZ NOT NULL,
  sensor_id   TEXT NOT NULL,
  temperature DOUBLE PRECISION,
  humidity    DOUBLE PRECISION
);

-- Convert to hypertable (auto-partitions by time)
SELECT create_hypertable('sensor_readings', 'time', chunk_time_interval => INTERVAL '1 day');

-- Insert like a normal table
INSERT INTO sensor_readings VALUES ('2026-01-15 10:00:00+00', 'sensor-1', 22.5, 0.45);

-- Query with continuous aggregates (auto-refreshed materialized views)
CREATE MATERIALIZED VIEW sensor_hourly
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 hour', time) AS bucket,
  sensor_id,
  AVG(temperature) AS avg_temp,
  MAX(temperature) AS max_temp
FROM sensor_readings
GROUP BY bucket, sensor_id;

-- Auto-drop old data
SELECT add_retention_policy('sensor_readings', INTERVAL '90 days');
```

### 10.3 InfluxDB — purpose-built

[InfluxDB](https://www.influxdata.com/) is a dedicated TSDB with its own query language (InfluxQL or Flux). Faster ingestion than TimescaleDB at the extreme, but you lose SQL and JOINs. Best for pure monitoring/telemetry pipelines.

### 10.4 Flask + TimescaleDB example

```python
class SensorReading(db.Model):
    __tablename__ = "sensor_readings"

    time: Mapped[datetime] = mapped_column(DateTime(timezone=True), primary_key=True)
    sensor_id: Mapped[str] = mapped_column(String(50), primary_key=True)
    temperature: Mapped[Optional[float]] = mapped_column(Float)
    humidity: Mapped[Optional[float]] = mapped_column(Float)

@app.route("/api/sensors/<sensor_id>/latest")
def latest_reading(sensor_id: str):
    stmt = (
        select(SensorReading)
        .where(SensorReading.sensor_id == sensor_id)
        .order_by(SensorReading.time.desc())
        .limit(1)
    )
    reading = db.session.scalars(stmt).first()
    if reading is None:
        abort(404)
    return jsonify({
        "time": reading.time.isoformat(),
        "temperature": reading.temperature,
        "humidity": reading.humidity,
    })

@app.route("/api/sensors/<sensor_id>/hourly")
def hourly_avg(sensor_id: str):
    # Uses the continuous aggregate
    stmt = (
        text("""
            SELECT bucket, avg_temp, max_temp
            FROM sensor_hourly
            WHERE sensor_id = :sid
              AND bucket > NOW() - INTERVAL '24 hours'
            ORDER BY bucket
        """)
    ).bindparams(sid=sensor_id)
    rows = db.session.execute(stmt).all()
    return jsonify([{"time": r.bucket.isoformat(), "avg": r.avg_temp, "max": r.max_temp} for r in rows])
```

> [!teach] Teaching Tip — Don't teach TSDBs first
> Time-series databases are a *specialty tool*. 95% of student projects don't need them. Teach PostgreSQL first, show TimescaleDB as "the upgrade path when your metrics table gets slow at 100M rows." The right time to reach for a TSDB is *after* you have evidence Postgres can't keep up — usually 6+ months into a real project.

#timescaledb #influxdb #time-series #iot #metrics

---

## 11. Putting It Together — A 2026 Flask Data Stack

Here's what a *modern* Flask app's data layer looks like in 2026:

```
┌─────────────────────────────────────────────────────────────┐
│                     Flask app (3.x)                          │
│  - SQLAlchemy 2.x with Mapped[T] typing                     │
│  - Async views for I/O-bound endpoints                      │
│  - Pydantic v2 for input validation                         │
└─────────────────────────────────────────────────────────────┘
              │                       │
              ▼                       ▼
     ┌─────────────────┐    ┌─────────────────┐
     │  PgBouncer      │    │  Redis / Valkey │
     │  (transaction   │    │  - cache        │
     │   mode)         │    │  - sessions     │
     └─────────────────┘    │  - rate limit   │
              │             │  - Celery broker│
              ▼             └─────────────────┘
     ┌─────────────────────────────┐
     │  PostgreSQL 17              │
     │  + pgvector (semantic search)│
     │  + pg_trgm (fuzzy text)     │
     │  + TimescaleDB (metrics)    │
     │  + PostGIS (if geo)         │
     └─────────────────────────────┘
              │
              ▼
     ┌─────────────────┐
     │  Read replica   │  ← analytics queries
     └─────────────────┘
```

If this feels like a lot, **it is** — and most apps don't need all of it. The right move:

1. **Start with SQLite** + Litestream. Ship the app.
2. **Move to PostgreSQL** when you outgrow single-writer or need full-text / JSON / vector.
3. **Add Redis** when you need caching, sessions across workers, or a queue.
4. **Add PgBouncer** when you move to serverless or hit connection limits.
5. **Add pgvector** when you build an AI feature.
6. **Add TimescaleDB** when your metrics table gets slow.
7. **Add a read replica** when analytics queries slow down the primary.

Each step is a *response to evidence*, not a thing you do upfront.

---

## 12. Teaching Tips — A 4-Week Databases Unit

### Week 1 — Foundations
- Day 1: SQLAlchemy 2.x `select()` API. Compare to old `Query` API. Show migration is mechanical.
- Day 2: `Mapped[T]` typing. Why types matter (mypy, IDE, autocompletion).
- Day 3: Relationships in 2.x. `back_populates`, eager vs lazy loading.
- Day 4: Alembic migrations. Autogenerate, review, edit. Always write `downgrade()`.
- Day 5: SQLite + Litestream. **Show students SQLite is real.**

### Week 2 — Going Production
- Day 1: Switch to PostgreSQL. JSONB columns.
- Day 2: Indexes. B-tree, GIN, EXPLAIN ANALYZE. Show what's slow and why.
- Day 3: Connection pooling. PgBouncer transaction mode. Why serverless needs it.
- Day 4: Redis as cache, session store, rate limiter. Three uses in one lesson.
- Day 5: Backups and recovery. `pg_dump`, PITR, restore drills.

### Week 3 — Search & AI
- Day 1: PostgreSQL FTS. `tsvector`, `tsquery`, GIN.
- Day 2: pgvector basics. Embeddings, similarity search.
- Day 3: Build a RAG endpoint in Flask. Search → context → LLM.
- Day 4: Meilisearch comparison. When to upgrade from Postgres FTS.
- Day 5: Vector DB landscape. Why pgvector is usually enough.

### Week 4 — Scale & Operations
- Day 1: Read replicas. SQLAlchemy `bind` routing.
- Day 2: TimescaleDB. Hypertables, continuous aggregates.
- Day 3: Zero-downtime migrations (expand-and-contract).
- Day 4: Migration testing in CI. Up, down, up.
- Day 5: Capstone — pick a real-world scenario (multi-tenant SaaS? AI chat? IoT dashboard?) and design the data stack.

### Anti-patterns to flag in class

- ❌ "Let's add Elasticsearch on day one" — start with Postgres FTS
- ❌ "Let's use MongoDB because it's NoSQL" — Postgres JSONB does that
- ❌ "Let's hand-roll connection pooling" — use PgBouncer
- ❌ "Let's write our own rate limiter" — Redis INCR + EXPIRE, or Flask-Limiter
- ❌ "Let's skip the downgrade" — you will regret it in prod
- ❌ "Let's store vectors in S3" — pgvector exists, use it

> [!quote] Closing thought
> "Every few years, the database world redisCOVERs that the boring option (PostgreSQL + SQLite + Redis) does 95% of what people pay specialized tools to do. The specialized tools are real and valuable — but they belong in your toolkit *after* you understand the boring one deeply. Learn PostgreSQL until it's boring. Then add the fancy stuff."
>
> — A teacher who has been doing this long enough to be suspicious of hype cycles

---

## 13. Companion Documents

- [[Modern-Flask-And-Python]] (R1) — Flask 3.x, Python 3.12/3.13, modern packaging
- [[Modern-Deployment-And-DevOps]] (R2) — deployment, CI/CD, observability, security (includes a Redis caching overview that complements this deep-dive)
- [[Modern-Frontend-Integration]] (R3) — HTMX, Alpine, Tailwind, SPA patterns
- [[Lesson-12-Database-Setup]] — your first SQLAlchemy setup
- [[Lesson-13-CRUD-And-Querying]] — the `Query` API (note: teaches 1.x style — pair with this doc's §1 for the 2.x upgrade)
- [[Lesson-15-Flask-Migrate]] — Alembic basics (this doc's §7 is the modern upgrade)
- [[Lesson-22-Email-And-Tasks]] — Celery + Redis (this doc's §4 expands Redis beyond just a broker)

---

> [!info] About this document
> This is a research reference for the Flask Teaching Course, written for instructors and curious students who want to know what the database ecosystem looks like *right now*. It's not a tutorial — it's a map. For step-by-step lessons, start at Lesson 12 and come back here when you want context on a specific topic.
