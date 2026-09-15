---
title: "Lesson 12 — SQLAlchemy Setup: Models, Sessions, and Databases"
module: 05-Database
lesson_number: 12
estimated_time: "120 minutes"
difficulty: intermediate
prerequisites:
  - "Completed [[Lesson-11-Flask-WTF]] — you can build forms and validate input"
  - "Basic SQL familiarity helps but isn't required"
tags:
  - flask
  - sqlalchemy
  - flask-sqlalchemy
  - database
  - orm
  - sqlite
  - postgresql
  - session
  - lesson
related_lessons:
  - "[[Lesson-11-Flask-WTF]]"
  - "[[Lesson-13-CRUD-Operations]]"
  - "[[Lesson-14-Relationships]]"
  - "[[Lesson-15-Flask-Migrate]]"
  - "[[Lesson-16-User-Model-AndPasswords]]"
---

# Lesson 12 — SQLAlchemy Setup: Models, Sessions, and Databases

> [!success] Time to make your data permanent.
> So far, every "database" in our Flask apps has been a Python dict in memory. Restart the server → lose all the data. That's fine for demos; it's a non-starter for real apps. Today you'll learn how to **persist** data — store it in a real database that survives restarts, handles concurrent users, and scales to millions of rows. We'll use **SQLAlchemy** (the most popular Python ORM) via **Flask-SQLAlchemy** (the Flask integration). We'll go deep on what databases *are*, why ORMs exist, how the SQLAlchemy session works, and the modern 2.0 style with `Mapped[T]` type hints. By the end you'll have a real `User` model stored in a real SQLite file.

## 1. Learning Objectives

By the end of this lesson you will be able to:

1. **Explain what a database is** (relational model, tables, rows, columns, primary keys, foreign keys).
2. **Write basic SQL** — `CREATE TABLE`, `INSERT`, `SELECT`, `UPDATE`, `DELETE` — and understand the four CRUD verbs.
3. **Explain what an ORM is** and why we use one instead of raw SQL, including the trade-offs.
4. **Describe SQLAlchemy's architecture** (Core + ORM) and how Flask-SQLAlchemy integrates with Flask.
5. **Configure a database URI** for SQLite, PostgreSQL, and MySQL, and decode each part of the URI.
6. **Choose between SQLite and PostgreSQL** based on use case (concurrency, types, scale, deployment).
7. **Define a model** with `db.Model`, `db.Column`, and column types — both legacy and 2.0 styles.
8. **Use `db.create_all()`** to create tables from models — and understand its limitations.
9. **Explain the session** in depth — `db.session.add()`, `db.session.commit()`, `db.session.rollback()`, the identity map, and the Unit of Work pattern.
10. **Read and write data** with the SQLAlchemy 2.0 `select()` style.
11. **Use modern SQLAlchemy 2.0** with `Mapped[T]` and `mapped_column()` type hints.
12. **Inspect the database** with `sqlite3` CLI and DB Browser for SQLite.
13. **Reason about async SQLAlchemy** and when it actually helps.

## 2. Prerequisites

- **[[Lesson-11-Flask-WTF]] completed.** You can build forms, validate input, and handle POST requests.
- **Python OOP fluency.** Classes, attributes, `__init__`, `__repr__`, inheritance.
- **(Helpful) Basic SQL.** If you've never written a `SELECT` statement, we'll cover the basics — but you may want to skim a SQL tutorial first.

## 3. The Problem

Every app you've built so far stores its data in Python variables:

```python
USERS = {
    'alice': {'password': 'wonderland', 'email': 'alice@example.com'},
}
```

This has **three fatal flaws**:

1. **Data disappears on restart.** Stop the server, start it again — `USERS` is empty. Users can't sign up because their accounts vanish nightly. Imagine if your bank worked this way.
2. **Data is per-process.** Run Flask with 4 Gunicorn workers, and each worker has its own copy of `USERS`. User registers on worker 1 → worker 2 doesn't know about them → user can't log in (their next request might hit worker 2). The workers don't share memory.
3. **No querying.** Want "all users who signed up in the last 7 days"? You'd loop through the dict in Python. For 10 users that's fine. For 10 million, it's catastrophic — the server would freeze for minutes scanning the dict, and you'd have to load all 10M rows into memory.

> [!analogy] The shared filing cabinet
> Think of an in-memory dict as a sticky note on one employee's desk. Other employees can't see it; if the employee goes home (process restarts) the note is gone. A database is the **shared filing cabinet in the back room**: every employee (process) can walk in, look up a folder, write a new file, and the cabinet is still there tomorrow. Yes, the cabinet is slower to walk to than glancing at a sticky note — but it's durable, shared, and indexed.

The fix: a **database** — a separate process (or file) that stores data durably and lets you query it efficiently. Your Flask app talks to the database over a connection.

```mermaid
flowchart TB
    subgraph "Before: in-memory dict"
        W1a["Worker 1<br/>USERS = {alice}"]
        W2a["Worker 2<br/>USERS = {} (empty!)"]
        W1a -.->|"Restart → wiped"| W2a
    end
    subgraph "After: shared database"
        W1b["Worker 1"]
        W2b["Worker 2"]
        W3b["Worker 3"]
        DB[(Database)]
        W1b -->|"INSERT/SELECT"| DB
        W2b -->|"INSERT/SELECT"| DB
        W3b -->|"INSERT/SELECT"| DB
    end
```

## 4. The Code

### 4.1 What is a database?

A database is an organised collection of data with a program (the **DBMS** — Database Management System) that stores, retrieves, and queries it. The most common type for web apps is the **relational database**, which organises data into **tables** (think: spreadsheets) with **rows** (records) and **columns** (fields).

```
users table:
+----+----------+-------------------+--------+---------------------+
| id | username | email             | age    | created_at          |
+----+----------+-------------------+--------+---------------------+
|  1 | alice    | alice@example.com | 30     | 2026-07-10 14:32:11 |
|  2 | bob      | bob@example.com   | 25     | 2026-07-10 14:32:11 |
|  3 | carol    | carol@example.com | NULL   | 2026-07-11 09:01:42 |
+----+----------+-------------------+--------+---------------------+
```

Each **row** is one user. Each **column** is one piece of data about that user. The `id` is a **primary key** — a unique identifier for each row (no two rows can share an `id`). `NULL` is SQL's special "no value" marker — distinct from `0` or `''`. It means "we don't know Carol's age."

> [!analogy] Spreadsheet vs database
> A spreadsheet is a good first mental model: each sheet is a table, each column header is a field, each row is a record. But spreadsheets have no constraints — you can put "banana" in the age column. Relational databases enforce **constraints**: `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK`. They also have **indexes** (instant lookup by id, no scanning), **transactions** (atomic multi-row changes), and a **query language** (SQL) far more powerful than Excel formulas.

### 4.2 The relational model in one diagram

```mermaid
flowchart TB
    subgraph "Relational model"
        Schema["Schema<br/>(design)"]
        Schema --> Table1["Table: users"]
        Schema --> Table2["Table: posts"]
        Table1 --> Row1["Row: (1, alice, ...)"]
        Table1 --> Row2["Row: (2, bob, ...)"]
        Table2 --> Row3["Row: (1, 'Hello', user_id=1)"]
        Row3 -.->|"FK: user_id references users.id"| Row1
    end
```

The two big ideas:

- **Tables** hold one type of entity (users, posts, comments).
- **Foreign keys** connect rows across tables. A post row stores `user_id = 1`, pointing at the user who wrote it. This is *how* relationships work in relational databases — by value, not by reference. (We'll go deep on this in [[Lesson-14-Relationships]].)

### 4.3 What is SQL?

**SQL (Structured Query Language)** is the standardised language for talking to relational databases. It's been around since 1974 and is one of the most stable technologies in computing — SQL you wrote in 1995 still works today. Every relational database (PostgreSQL, MySQL, SQLite, Oracle, SQL Server) speaks SQL, with minor dialect differences.

The four fundamental operations (CRUD — Create, Read, Update, Delete):

```sql
-- Create: add a row
INSERT INTO users (username, email, age) VALUES ('alice', 'alice@example.com', 30);

-- Read: query rows
SELECT * FROM users WHERE username = 'alice';

-- Update: change an existing row
UPDATE users SET age = 31 WHERE username = 'alice';

-- Delete: remove a row
DELETE FROM users WHERE username = 'alice';
```

And the schema-definition verbs:

```sql
-- Define a table
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username VARCHAR(80) UNIQUE NOT NULL,
    email VARCHAR(120) UNIQUE NOT NULL,
    age INTEGER,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Inspect it
.schema users

-- Drop it (destroy all data!)
DROP TABLE users;
```

You *can* write raw SQL in Flask (using `db.session.execute(text("SELECT ..."))`), and for complex queries it's sometimes the right choice. But most of the time, we use an **ORM**.

### 4.4 What is an ORM?

An **ORM (Object-Relational Mapper)** is a layer that lets you interact with the database using Python objects instead of SQL strings. You define a Python class; the ORM translates it into a database table. You create an instance; the ORM runs an `INSERT`. You query with a Python method; the ORM runs a `SELECT` and returns Python objects.

```python
# Instead of writing SQL:
# INSERT INTO users (username, email) VALUES ('alice', 'alice@example.com');

# You write Python:
user = User(username='alice', email='alice@example.com')
db.session.add(user)
db.session.commit()
```

```mermaid
flowchart LR
    Python["Python code<br/>user = User(name='alice')<br/>db.session.add(user)<br/>db.session.commit()"]
    Python -->|ORM translates| SQL["SQL<br/>INSERT INTO users (name)<br/>VALUES ('alice')"]
    SQL --> DB[(Database)]
    DB -->|row inserted| Table["users table:<br/>| 1 | alice |"]
```

The ORM is the **translator** between Python's object-oriented world and the database's relational world. You think in objects; the database thinks in tables; the ORM bridges the gap.

> [!analogy] The ORM as a UN translator
> Picture a UN meeting: each delegate speaks their own language. The translator converts English → French in real time. The English delegate doesn't need to learn French; the French delegate doesn't need to learn English. The translator does the work — but the translator can also subtly fail (a joke that doesn't translate, a phrase with no exact equivalent). The ORM is that translator between Python-land and SQL-land. It does 90% of the work flawlessly; the 10% (complex queries, dialect-specific features) you handle yourself.

#### Pros of using an ORM

- **No SQL strings in your code.** No syntax errors, no SQL injection (the ORM escapes values automatically — never concatenate user input into SQL!).
- **Database-agnostic.** Write your models once; switch from SQLite to PostgreSQL by changing one config line.
- **Type safety (in SQLAlchemy 2.0).** IDE autocomplete on `user.email`, mypy can check your queries.
- **Relationships as attributes.** `user.posts` returns the user's blog posts — the ORM runs the JOIN for you.
- **Migrations.** Tools like Alembic (Lesson 15) diff your models and generate schema migrations.

#### Cons of using an ORM

- **Performance overhead.** Each row becomes a Python object, which is slower than raw SQL.
- **"Magic" behaviour.** Lazy loading, cascades, and sessions can surprise you (the famous N+1 problem — Lesson 14).
- **Complex queries are awkward.** Five-table JOINs with subqueries are easier in SQL than in ORM syntax.
- **Another thing to learn.** ORM concepts (sessions, units of work, identity maps) take time.

> [!question] Student asks: "If ORMs are slower than raw SQL, why use them at all?"
> Because developer time is far more expensive than database time, and most apps are not Facebook-scale. A query that takes 12 ms instead of 8 ms is invisible to users. But writing raw SQL for every operation, hand-escaping user input, managing connections, mapping rows to objects — that's weeks of work and a security nightmare. The ORM trades a tiny bit of speed for a huge amount of safety and productivity. When a query *does* need to be fast, you drop down to raw SQL for that one query — Lesson 13 shows how.

The consensus in 2026: **use an ORM for 90% of queries, drop down to raw SQL for the 10% that need it.** That's what we'll do.

### 4.5 SQLAlchemy architecture

SQLAlchemy has two layers:

1. **Core** — the low-level layer. Provides the SQL expression language (`select()`, `insert()`, `update()`, `delete()`), connection pooling, transaction management, and database-agnostic type coercion. Core *is* a database abstraction, but you're still writing SQL-like expressions.
2. **ORM** — the high-level layer. Maps Python classes to tables, manages sessions, lazy-loads relationships. Built on top of Core.

```mermaid
flowchart TB
    YourCode["Your Flask code"]
    YourCode --> ORM["SQLAlchemy ORM<br/>(models, sessions, relationships)"]
    ORM --> Core["SQLAlchemy Core<br/>(SQL expressions, connections, pooling)"]
    Core --> Driver["DBAPI driver<br/>(psycopg2, sqlite3, etc.)"]
    Driver --> DB[(Database<br/>PostgreSQL/SQLite/MySQL)]
    style ORM fill:#cfe,stroke:#369
    style Core fill:#fec,stroke:#963
    style Driver fill:#fce,stroke:#936
```

You typically use the ORM. You drop down to Core for performance-critical code or complex queries. You drop down to raw SQL (via `text()`) for things even Core can't express.

> [!info] Why two layers?
> Not everyone wants a full ORM. Data scientists running analytical queries want fast SQL execution without Python objects. Core gives them that. Application developers who *do* want objects use the ORM. SQLAlchemy is "two libraries in one" — you pick your level of abstraction.

### 4.6 Flask-SQLAlchemy — the integration layer

Flask-SQLAlchemy is a thin wrapper that:

- Creates a single `db = SQLAlchemy(app)` object you use everywhere.
- Handles the **Flask application context** (so sessions are scoped to requests — each request gets its own session, automatically cleaned up).
- Adds Flask-specific helpers like `Model.query` (legacy) and `get_or_404()` (returns a 404 if no row found — perfect for web routes).
- Reads config from `app.config` (`SQLALCHEMY_DATABASE_URI`, etc.).

Without Flask-SQLAlchemy, you'd be writing 30 lines of boilerplate to set up engine/session/declarative_base per app. Flask-SQLAlchemy does it in one line.

### 4.7 Install

```bash
pip install Flask-SQLAlchemy
```

This installs Flask-SQLAlchemy 3.x (which uses SQLAlchemy 2.0+ internally — important for modern features). Verify the version:

```bash
python -c "import flask_sqlalchemy, sqlalchemy; print(flask_sqlalchemy.__version__, sqlalchemy.__version__)"
# 3.1.1 2.0.30
```

### 4.8 `app.py`

```python
from flask import Flask, render_template
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///myapp.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)


class User(db.Model):
    __tablename__ = 'users'

    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    age = db.Column(db.Integer, nullable=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

    def __repr__(self):
        return f'<User {self.username}>'


# Create tables (only runs if they don't exist)
with app.app_context():
    db.create_all()


@app.route('/')
def index():
    users = db.session.execute(db.select(User).order_by(User.username)).scalars().all()
    return render_template('index.html', users=users)


if __name__ == '__main__':
    app.run(debug=True)
```

### 4.9 `templates/index.html`

```html
<!DOCTYPE html>
<html>
<body>
    <h1>Users</h1>
    {% if users %}
        <ul>
            {% for user in users %}
                <li>{{ user.username }} — {{ user.email }} (joined {{ user.created_at }})</li>
            {% endfor %}
        </ul>
    {% else %}
        <p>No users yet. Add some via the Flask shell:</p>
        <pre>flask shell
&gt;&gt;&gt; from app import db, User
&gt;&gt;&gt; db.session.add(User(username='alice', email='alice@example.com'))
&gt;&gt;&gt; db.session.commit()</pre>
    {% endif %}
</body>
</html>
```

## 5. Line-by-Line Explanation

### 5.1 The database URI

```python
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///myapp.db'
```

The URI tells SQLAlchemy which database to connect to. Format: `dialect+driver://user:password@host:port/database`.

| Database | URI example | Notes |
|---|---|---|
| SQLite (relative) | `sqlite:///myapp.db` | File at `./myapp.db` |
| SQLite (absolute) | `sqlite:////tmp/myapp.db` | Note: **4 slashes** for absolute path |
| SQLite (in-memory) | `sqlite:///:memory:` | Disappears when process ends — perfect for tests |
| PostgreSQL | `postgresql://user:pass@localhost/mydb` | Default port 5432 |
| PostgreSQL (async) | `postgresql+asyncpg://user:pass@localhost/mydb` | For async SQLAlchemy |
| MySQL | `mysql+pymysql://user:pass@localhost/mydb` | Need `pymysql` package |
| SQL Server | `mssql+pyodbc://user:pass@host/mydb` | Rare in Flask ecosystem |

Let's decode the parts of a PostgreSQL URI:

```
postgresql://user:pass@localhost:5432/mydb
^^^^^^^^^^   ^^^^ ^^^^ ^^^^^^^^^ ^^^^ ^^^^
dialect      user pass  host     port  db
```

- `postgresql` — the dialect (which SQL flavour to speak)
- `user` — DB username (must exist on the server)
- `pass` — DB password (NEVER commit to git; use env var: `os.environ['DATABASE_URL']`)
- `localhost` — server hostname (or IP)
- `5432` — port (Postgres default; MySQL is 3306)
- `mydb` — database name (must already exist; create with `createdb mydb`)

> [!warning] SQLite's four-slash gotcha
> `sqlite:///myapp.db` (3 slashes) = relative path → file at `./myapp.db`.
> `sqlite:////tmp/myapp.db` (4 slashes) = absolute path → file at `/tmp/myapp.db`.
> The "missing" 4th slash trips up every beginner. The rule: count slashes after `sqlite:` — 3 means relative, 4 means absolute. The first 3 slashes are the `://` separator; the 4th is the start of the absolute path `/tmp/...`.

> [!try] Try this
> Try `sqlite:///./subdir/myapp.db`. Where does the file land? (Answer: `./subdir/myapp.db` — but `subdir/` must already exist; SQLite won't create parent directories.) Then try `sqlite:///:memory:` and observe that data vanishes on restart — perfect for tests.

### 5.2 SQLite vs PostgreSQL — when to use which

| Aspect | SQLite | PostgreSQL |
|---|---|---|
| Setup | Zero-config — just a file | Server process + `createdb` |
| Concurrency | One writer at a time (WAL mode improves this) | Many concurrent writers (MVCC) |
| Scale | Good to ~100k rows / low traffic | Billions of rows, heavy load |
| Types | Limited (no native JSON, array) | Rich (JSONB, arrays, geo, full-text) |
| Production use | Small apps, embedded, testing | Most production Flask apps |
| Backups | Copy the file | `pg_dump` |
| Cost | Free | Free (self-host) or $$ (managed) |

**Rule of thumb**: develop with SQLite (zero-config, fast iteration), deploy to PostgreSQL (production-grade, scalable). The ORM abstracts the differences, so your code stays the same.

```mermaid
flowchart LR
    Start["New Flask project"] --> Q1{"Heavy write<br/>concurrency?"}
    Q1 -->|No| Q2{"Need JSONB, arrays,<br/>full-text search?"}
    Q1 -->|Yes| PG["Use PostgreSQL"]
    Q2 -->|No| Q3{"Just learning<br/>or small app?"}
    Q2 -->|Yes| PG
    Q3 -->|Yes| SQLite["Use SQLite"]
    Q3 -->|No| PG
```

> [!question] Student asks: "Why is SQLite limited to one writer?"
> SQLite stores the whole database in a single file. To write safely, it locks the file. PostgreSQL, by contrast, uses **MVCC (Multi-Version Concurrency Control)**: when a transaction modifies a row, Postgres creates a new version of the row while readers continue reading the old version. Many writers can run in parallel, each in their own transaction. SQLite's WAL (Write-Ahead Logging) mode improves things — readers don't block writers and vice versa — but you still can't have two writers at once. For a personal blog this is irrelevant. For Twitter it's fatal.

### 5.3 `SQLALCHEMY_TRACK_MODIFICATIONS`

```python
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
```

Flask-SQLAlchemy used to track every attribute change on every model object (for the `before_models_committed` signal). This added memory overhead and is unnecessary for most apps. Setting it to `False` disables the tracking. In Flask-SQLAlchemy 3.x, the default is already `False`, but setting it explicitly documents your intent.

> [!info] Why was it ever `True`?
> In early Flask-SQLAlchemy (1.x), the signal `models_committed` fired with a list of every change — useful for cache invalidation or audit logs. Almost nobody used it. In 3.x it's off by default. If you *do* need change tracking, use SQLAlchemy events (`event.listen`) on specific models — much more efficient.

### 5.4 The `db` object

```python
db = SQLAlchemy(app)
```

This single line does a lot behind the scenes:

1. Creates a SQLAlchemy **`Engine`** (connection pool) using the URI from config. The engine doesn't open connections yet — it opens them lazily on first use.
2. Creates a **`sessionmaker`** bound to that engine. The sessionmaker is a factory: call `Session()` to get a new session.
3. Creates a declarative **`Model`** base class (`db.Model`). Classes that inherit from `db.Model` are automatically registered as tables.
4. Hooks into Flask's **request lifecycle**: at the start of each request, Flask-SQLAlchemy creates a fresh **scoped session**; at the end, it commits (or rolls back) and closes the session. You never have to manage this manually.
5. Adds the `db` object to your app for access elsewhere.

You use `db` for everything: `db.Model` (base class), `db.Column` (column definition), `db.session` (the session), `db.select` / `db.insert` / `db.func` (SQL expressions), `db.create_all()` (create tables).

```mermaid
flowchart TB
    db["db = SQLAlchemy(app)"]
    db --> Engine["db.engine<br/>(connection pool)"]
    db --> Session["db.session<br/>(scoped, per-request)"]
    db --> Model["db.Model<br/>(declarative base class)"]
    db --> Helpers["Helpers:<br/>db.Column, db.String,<br/>db.select, db.func"]
    Engine -.->|"lazy open"| Pool["5 + 10 overflow<br/>DB connections"]
    Session -.->|"teardown at<br/>request end"| Commit["commit()<br/>or rollback()"]
```

### 5.5 Model definition

```python
class User(db.Model):
    __tablename__ = 'users'

    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    age = db.Column(db.Integer, nullable=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
```

A **model** is a Python class that inherits from `db.Model`. Each class attribute is a column. The pattern:

```python
column_name = db.Column(column_type, **options)
```

When the class is defined, SQLAlchemy reads the class body, builds a `Table` object in its `MetaData`, and registers it. The class itself becomes a "mapper" between Python attributes and table columns.

#### Common column types

| Type | SQL equivalent | Python type | Notes |
|---|---|---|---|
| `Integer` | `INTEGER` | `int` | Whole numbers |
| `Float` | `REAL` / `FLOAT` | `float` | Floating-point |
| `Numeric(precision, scale)` | `NUMERIC` | `Decimal` | Money — exact precision |
| `String(length)` | `VARCHAR(length)` | `str` | Variable-length string |
| `Text` | `TEXT` | `str` | Unlimited length string |
| `Boolean` | `BOOLEAN` | `bool` | True/False |
| `Date` | `DATE` | `datetime.date` | Calendar date |
| `DateTime` | `DATETIME` | `datetime.datetime` | Date + time |
| `Time` | `TIME` | `datetime.time` | Time of day |
| `Interval` | `INTERVAL` | `datetime.timedelta` | Time duration |
| `LargeBinary` | `BLOB` / `BYTEA` | `bytes` | Binary data |
| `JSON` | `JSON` / `JSONB` | `dict` / `list` | PostgreSQL: use `JSONB` |
| `Enum` | `ENUM` | str (constrained) | Limited set of values |
| `UUID` (PG) | `UUID` | `uuid.UUID` | PostgreSQL native |

> [!question] Student asks: "When do I use `String` vs `Text`?"
> `String(80)` is `VARCHAR(80)` — fixed max length, often indexed efficiently. `Text` is unlimited length, never indexed (in some DBs). Use `String` for short fields with a known max (username, email, title). Use `Text` for long free-form content (post body, comments, descriptions). For PostgreSQL, `VARCHAR(n)` and `TEXT` have the same performance — the length limit is just a constraint, not a storage difference. For SQLite, `TEXT` is the underlying storage for all strings.

#### Common column options

| Option | Effect |
|---|---|
| `primary_key=True` | This column is the primary key (unique, not null) |
| `unique=True` | No two rows can have the same value |
| `nullable=False` | Cannot be NULL (must always have a value) |
| `default=value` | Use this value if not specified (Python-side default) |
| `default=callable` | Call this function to get the default (e.g., `datetime.utcnow`) |
| `server_default=text('NOW()')` | Default set by the database, not Python |
| `index=True` | Create a database index for faster lookups |
| `onupdate=callable` | Auto-update this column on every UPDATE (e.g., `updated_at`) |

> [!question] Student asks: "Why `default=datetime.utcnow` and not `default=datetime.utcnow()`?"
> `default=datetime.utcnow()` *calls* the function once at class-definition time — every user would get the *same* timestamp (the moment the app started). `default=datetime.utcnow` (no parentheses) passes the *function itself* — SQLAlchemy calls it each time a new row is created, giving each row its own timestamp. This is a classic beginner mistake.

> [!try] Try this
> Create two models side by side — one with `default=datetime.utcnow()` (called) and one with `default=datetime.utcnow` (callable). Insert one row into each, wait 2 seconds, insert another. Inspect the `created_at` values. The first model will have *identical* timestamps on both rows; the second will differ by ~2 seconds. That's the bug visualised.

> [!info] `default` vs `server_default`
> `default` runs in Python — the value is set on the Python object *before* INSERT. `server_default` runs in SQL — the column has `DEFAULT NOW()` in the schema, and the DB fills it. Use `default` when the value depends on Python (e.g., `uuid.uuid4`). Use `server_default` when you want the same default even if a row is inserted by raw SQL or another app — typical for `created_at` and `updated_at`.

### 5.6 `__tablename__`

```python
__tablename__ = 'users'
```

This explicitly sets the database table name. Without it, SQLAlchemy uses the lowercase class name (`user`). Why might you want to override?

- **Reserved words.** `User`, `Order`, `Group` are reserved in some databases. Best practice: pluralise (`users`, `orders`, `groups`).
- **Naming consistency.** Plural table names (`users`) + singular model names (`User`) is the most common convention. The model is *one* user; the table holds *many* users.
- **Database migration.** If you ever rename a model, you can keep the table name stable (so existing data isn't lost).

### 5.7 `__repr__`

```python
def __repr__(self):
    return f'<User {self.username}>'
```

`__repr__` is Python's "official string representation" — what you see when you `print(user)` or `repr(user)` in a shell. Without it, you'd see `<User object at 0x7f8a3c2b1d90>` which is useless for debugging. Always define `__repr__` on models to show identifying information.

A good `__repr__` includes:

- The class name (`User`)
- The primary key (`id`)
- One or two human-readable fields (`username`)

Example: `f'<User id={self.id} username={self.username!r}>'` → `<User id=1 username='alice'>`.

### 5.8 `db.create_all()`

```python
with app.app_context():
    db.create_all()
```

`db.create_all()` reads all your model classes and runs `CREATE TABLE IF NOT EXISTS` for each one. It's how your Python model definitions become actual database tables.

Important limitations:

- **It only creates tables that don't exist.** If you add a new column to a model later, `create_all()` will NOT alter the existing table. You need a migration tool (Lesson 15).
- **It must run inside an application context.** Flask-SQLAlchemy needs to know which app's database to use; `app.app_context()` provides that.
- **It runs at import time.** Putting it at the top of `app.py` means it runs whenever the module is imported — including in tests, in the Flask shell, in production. That's fine for development; in production, you typically run migrations instead (Lesson 15).

```mermaid
flowchart TB
    Models["Your Python models<br/>class User(db.Model):<br/>  id = Column(...)<br/>  username = Column(...)<br/>class Post(db.Model):<br/>  ..."]
    Models -->|db.create_all()| Meta["SQLAlchemy metadata<br/>(collects all tables)"]
    Meta -->|"CREATE TABLE IF NOT EXISTS<br/>users (...)"| DB[(Database)]
    Meta -->|"CREATE TABLE IF NOT EXISTS<br/>posts (...)"| DB
    DB --> Tables["Tables exist in DB"]
    Note["Note: does NOT alter<br/>existing tables"] -.-> Tables
```

> [!info] What `create_all()` runs internally
> 1. Walk `db.metadata.tables` — a dict of every `Table` object collected from your `db.Model` subclasses.
> 2. For each table, ask the engine to check if it exists (via `SHOW TABLES` / `information_schema` / `sqlite_master`).
> 3. For each non-existent table, emit `CREATE TABLE` SQL.
> 4. Existing tables are *not* diffed or altered — `create_all()` doesn't know what changed.
>
> That last point is *the* reason Flask-Migrate exists (Lesson 15).

### 5.9 The session — `db.session`

The **session** is SQLAlchemy's Unit of Work — a shopping cart for database changes. You `add()` objects to it, then `commit()` to flush them to the database. If something goes wrong, `rollback()` undoes everything since the last commit.

```python
# Create
user = User(username='alice', email='alice@example.com')
db.session.add(user)         # stage the change
db.session.commit()          # actually write to DB

# Update
user.email = 'new@example.com'   # modify the object
db.session.commit()              # writes UPDATE to DB

# Delete
db.session.delete(user)
db.session.commit()
```

```mermaid
flowchart LR
    Create["user = User(...)"] --> Add["db.session.add(user)"]
    Add -->|"staged (in memory)"| Session["Session<br/>(shopping cart)"]
    Modify["user.email = 'new'"] -->|"staged"| Session
    Delete["db.session.delete(user)"] -->|"staged"| Session
    Session -->|"db.session.commit()"| Flush["Write all staged changes<br/>to DB (INSERT/UPDATE/DELETE)"]
    Session -->|"db.session.rollback()"| Clear["Discard all staged changes<br/>revert to last commit"]
```

> [!question] Student asks: "Why do we need a session at all? Why not write directly to the DB?"
> Three reasons:
> 1. **Atomicity.** If you're creating a user and a profile, you want both to succeed or both to fail — never one without the other. The session batches them into a transaction.
> 2. **Identity.** If you query the same user twice in one request, the session returns the same Python object (the "identity map"). This prevents inconsistency and saves queries.
> 3. **Efficiency.** The session batches changes. Ten `add()` calls + one `commit()` is one round-trip to the DB, not ten.

> [!analogy] The session as a checkout line
> Imagine shopping at a supermarket. You don't pay for each item as you take it off the shelf — you put items in your cart (the session) and pay for everything at checkout (commit). If you decide you don't want the chocolate after all, you can put it back (rollback) before checkout. After you pay, the transaction is final. The session works the same way: stage changes, then commit them all atomically.

#### The identity map — a hidden superpower

When you query a user, the session caches the object by primary key. If you query the same user again in the same session, the session returns the *cached* object without hitting the DB:

```python
user1 = db.session.get(User, 1)   # hits DB
user2 = db.session.get(User, 1)   # returns cached user1
user1 is user2  # True — same Python object!
```

This prevents inconsistency: you can't have two `User` objects in one request representing the same DB row, with one having stale data. It also saves DB round-trips.

#### `add()` vs `add_all()`

```python
# One at a time
db.session.add(user1)
db.session.add(user2)
db.session.add(user3)

# All at once
db.session.add_all([user1, user2, user3])
```

Both are equivalent. `add_all()` is just convenience.

#### `flush()` vs `commit()`

`db.session.flush()` writes pending changes to the DB *within the transaction* — but doesn't commit. You can then read the new IDs, run more queries, and decide whether to commit or rollback. `commit()` flushes *and* commits the transaction (makes changes permanent).

```python
user = User(username='alice')
db.session.add(user)
db.session.flush()       # INSERT runs, user.id is now set
print(user.id)           # 1
db.session.rollback()    # undo — user is NOT in DB
```

You rarely call `flush()` directly. It's useful when you need the auto-generated ID *before* commit.

### 5.10 Reading data — SQLAlchemy 2.0 style

```python
users = db.session.execute(db.select(User).order_by(User.username)).scalars().all()
```

This is the SQLAlchemy 2.0 style. Let's break it down:

- `db.select(User)` — build a `SELECT * FROM users` statement.
- `.order_by(User.username)` — add `ORDER BY username`.
- `db.session.execute(stmt)` — run the query, get a `Result` object.
- `.scalars()` — extract the first column of each row (the `User` object, not a `(User,)` tuple).
- `.all()` — materialise the result as a Python list.

The legacy style (still works, but deprecated in 2.0):

```python
users = User.query.order_by(User.username).all()
```

We'll cover the full range of queries in [[Lesson-13-CRUD-Operations]].

> [!question] Student asks: "Why the long 2.0 style? The legacy `Model.query` was shorter."
> Two reasons. First, **consistency**: in 2.0, *every* query goes through `db.session.execute(stmt)` — whether it returns ORM objects, raw columns, or aggregates. The old API had three different ways to query (`Model.query`, `db.session.query(...)`, raw SQL). Second, **decoupling**: the new API separates *building* a query (`select(...)`) from *running* it (`session.execute(...)`). You can pass a `select` object around, build it dynamically, test it, cache it — without touching the session. The legacy API tightly coupled the two.

### 5.11 Modern SQLAlchemy 2.0 — `Mapped[T]` and `mapped_column()`

SQLAlchemy 2.0 (released 2023, dominant in 2026) introduced a new way to define models with **type annotations**:

```python
from sqlalchemy.orm import Mapped, mapped_column
from typing import Optional

class User(db.Model):
    __tablename__ = 'users'

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(80), unique=True)
    email: Mapped[str] = mapped_column(String(120), unique=True)
    age: Mapped[Optional[int]] = mapped_column(nullable=True)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

Benefits:

- **Type hints.** `id: Mapped[int]` tells your IDE and mypy that `user.id` is an `int`. Autocomplete works. Type checkers catch bugs.
- **`Optional[int]` automatically means `nullable=True`.** No need to repeat yourself.
- **Cleaner imports.** One `mapped_column` instead of `Column` vs `mapped_column` confusion.
- **Relationships are typed.** `posts: Mapped[list['Post']] = relationship(back_populates='author')` — `user.posts` is `list[Post]` to your type checker.

```mermaid
flowchart LR
    Old["Legacy style (1.x)<br/>id = db.Column(db.Integer,<br/>  primary_key=True)"]
    New["2.0 style<br/>id: Mapped[int] = mapped_column(<br/>  primary_key=True)"]
    Old -.->|"Works but deprecated<br/>in SQLAlchemy 2.1"| Both["Same DB schema"]
    New --> Both
    New -->|"Bonus: type hints<br/>for IDE/mypy"| Types["user.id is int<br/>user.email is str<br/>user.age is int | None"]
```

For new projects in 2026, use the 2.0 style. For existing projects, you can mix the two styles — they're compatible. We'll use 2.0 style for new examples in this course.

> [!info] What `Mapped[T]` actually does
> `Mapped[int]` is a *generic type alias* that tells SQLAlchemy two things: (1) this is a column declaration, (2) the Python type is `int`. SQLAlchemy reads the annotation and infers the SQL type — `int` → `Integer`, `str` → `String`, `datetime` → `DateTime`. You can still override with an explicit type (`mapped_column(String(80))`). The magic is just PEP 484 type hints — no metaclass trickery.

### 5.12 Modern: async SQLAlchemy

For apps with high I/O concurrency (e.g., a Slack-like real-time app making many parallel DB calls), SQLAlchemy 2.0 supports async:

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

engine = create_async_engine('postgresql+asyncpg://user:pass@localhost/mydb')
AsyncSessionLocal = async_sessionmaker(engine, class_=AsyncSession)

async def get_user(user_id: int):
    async with AsyncSessionLocal() as session:
        stmt = select(User).where(User.id == user_id)
        result = await session.execute(stmt)
        return result.scalars().first()
```

In a Flask async view:

```python
@app.route('/users/<int:user_id>')
async def show_user(user_id):
    user = await get_user(user_id)
    return render_template('user.html', user=user)
```

Async only helps when *other* parts of your request are also async (HTTP calls to other services, async template rendering). If your view is `async` but only does one DB call, you've added complexity for no gain. Use async when you have **concurrent I/O** — multiple DB queries, HTTP calls, or fan-out work. We cover this in [[Modern-Databases-And-Caching]].

> [!question] Student asks: "Does Flask support async natively?"
> Yes — since Flask 2.0 (2021), you can declare `async def` views. Flask runs them in a threadpool per request. But "async views" alone aren't faster — they only help if you `await` multiple I/O operations concurrently (e.g., `asyncio.gather(query1(), query2())`). For a typical CRUD app, sync is simpler and equally fast. Reach for async when you have fan-out work: real-time dashboards, webhook fanout, microservice aggregations.

### 5.13 Behind the scenes: the SQLAlchemy session lifecycle

Let's trace what happens during a single Flask request that creates a user:

```mermaid
sequenceDiagram
    participant B as Browser
    participant F as Flask
    participant S as Session
    participant DB as Database
    B->>F: POST /register (form data)
    F->>S: Create scoped session (per-request)
    F->>F: Build User object in Python
    F->>S: db.session.add(user)
    Note over S: User is staged in session<br/>(NOT in DB yet)
    F->>F: Run validation, business logic
    F->>S: db.session.commit()
    S->>DB: BEGIN TRANSACTION
    S->>DB: INSERT INTO users (...) VALUES (...)
    DB-->>S: OK (returns new id)
    S->>DB: COMMIT
    S-->>F: Done
    F-->>B: 302 redirect
    F->>S: Session closed (auto at request end)
```

Key insights:

- The session is **per-request**. Flask-SQLAlchemy creates a fresh session at the start of each request and closes it at the end. Two concurrent users get two separate sessions.
- The session is **lazy**. Adding an object doesn't write to the DB — it stages the change. Only `commit()` triggers actual SQL.
- The session is **transactional**. Either all staged changes commit, or none do. If `commit()` fails (e.g., a `unique` constraint violation), the session is left in a broken state and you must `rollback()` before using it again.
- The session is an **identity map**. If you query `User` with id=1 twice in the same request, you get the *same* Python object both times.

### 5.14 Connection pooling

Behind the scenes, SQLAlchemy maintains a **connection pool** — a cache of open database connections. Opening a new TCP connection to PostgreSQL takes 10-50ms (TCP handshake + TLS + auth); reusing a pooled connection takes microseconds. The pool defaults to 5 connections, overflow to 10, with idle timeout of 30 seconds.

```mermaid
flowchart LR
    App["Flask app<br/>(multiple workers)"]
    App --> Pool["Connection pool<br/>size=5, overflow=10"]
    Pool --> C1["Conn 1"]
    Pool --> C2["Conn 2"]
    Pool --> C3["Conn 3"]
    Pool --> CN["..."]
    C1 --> DB[(PostgreSQL)]
    C2 --> DB
    C3 --> DB
```

You typically don't tune this in Flask-SQLAlchemy unless you have very high traffic. But it's worth knowing: if you see `QueuePool limit of size X overflow Y reached` errors, your app is opening more concurrent DB connections than the pool allows. Increase `SQLALCHEMY_ENGINE_OPTIONS={'pool_size': 20}` or fix the N+1 query problem (Lesson 14).

> [!try] Try this
> Set `app.config['SQLALCHEMY_ECHO'] = True` and reload. Watch the terminal — SQLAlchemy will log every SQL statement it executes, including bind parameters. This is *invaluable* for understanding what the ORM is doing behind your back. (Turn it off in production!)

## 6. Running the Code

Save `app.py`. Run:

```bash
python app.py
```

You should see:

```
 * Serving Flask app 'app'
 * Running on http://127.0.0.1:5000
```

Open `http://127.0.0.1:5000/`. You'll see "No users yet." — because we haven't added any.

### 6.1 Verify the database file was created

In a new terminal:

```bash
ls -la myapp.db
```

You should see a `myapp.db` file (probably ~8 KB). That's your SQLite database.

### 6.2 Inspect with the `sqlite3` CLI

```bash
sqlite3 myapp.db
```

Inside the `sqlite3` prompt:

```
sqlite> .tables
users

sqlite> .schema users
CREATE TABLE users (
        id INTEGER NOT NULL,
        username VARCHAR(80) NOT NULL,
        email VARCHAR(120) NOT NULL,
        age INTEGER,
        created_at DATETIME,
        PRIMARY KEY (id),
        UNIQUE (username),
        UNIQUE (email)
);

sqlite> SELECT * FROM users;
sqlite> .quit
```

The table exists but is empty. `db.create_all()` made the table; we haven't added any rows.

### 6.3 Add a user via the Flask shell

```bash
flask shell
```

Inside the shell:

```python
>>> from app import db, User
>>> db.session.add(User(username='alice', email='alice@example.com', age=30))
>>> db.session.add(User(username='bob', email='bob@example.com', age=25))
>>> db.session.commit()
>>> User.query.all()
[<User alice>, <User bob>]
>>> exit()
```

Reload `http://127.0.0.1:5000/` in your browser. You should now see:

```
Users
  • alice — alice@example.com (joined 2026-07-10 14:32:11)
  • bob — bob@example.com (joined 2026-07-10 14:32:11)
```

### 6.4 Verify with `sqlite3` again

```bash
sqlite3 myapp.db "SELECT * FROM users;"
1|alice|alice@example.com|30|2026-07-10 14:32:11.123456
2|bob|bob@example.com|25|2026-07-10 14:32:11.123789
```

Real rows in a real database file. Restart the Flask server — the data is still there.

### 6.5 DB Browser for SQLite

If you prefer a GUI, [DB Browser for SQLite](https://sqlitebrowser.org/) is a free, cross-platform tool. Open `myapp.db` and you can browse tables, run queries, and edit data with a spreadsheet-like interface. Great for development; not for production (your prod DB is PostgreSQL, not SQLite).

### 6.6 Real-world: how production databases differ

In production:

- **Connection pooling** is tuned for your workload. Default Flask-SQLAlchemy settings are for development. Production needs higher `pool_size`, longer `pool_recycle` (PostgreSQL drops idle connections after 10 minutes), and a `pool_pre_ping` setting to detect dropped connections.
- **Read replicas** offload SELECT queries to a replica server, leaving the primary for writes. SQLAlchemy supports this via `bind` routing.
- **Backups** are automated. For PostgreSQL: `pg_dump` to S3 nightly + WAL archiving for point-in-time recovery. Test restoring backups regularly — an untested backup is no backup.
- **Monitoring** tracks query times, slow queries, connection count, disk usage. Tools: `pg_stat_statements`, Datadog, Sentry, New Relic.
- **Migrations** are run via CI/CD, not manually. We cover this in Lesson 15.
- **Schema is version-controlled** via Alembic migrations, committed to git alongside your code.
- **Secrets** (DB password) are loaded from environment variables or a secrets manager — never in source code.

```mermaid
flowchart TB
    App["Production Flask app"]
    App --> Pool["Pool<br/>size=20"]
    Pool --> Primary[("Primary DB<br/>(writes)")]
    Pool --> Replica1[("Replica 1<br/>(reads)")]
    Pool --> Replica2[("Replica 2<br/>(reads)")]
    Primary -.->|"streaming<br/>replication"| Replica1
    Primary -.->|streaming| Replica2
    Backup["pg_dump nightly<br/>+ WAL archive"] -.-> Primary
    S3[("S3 backup bucket")] -.-> Backup
```

### 6.7 Historical context

Relational databases predate the web by decades. IBM's System R (1974) introduced SQL. Oracle (1979) commercialised it. PostgreSQL started as a UC Berkeley research project in 1986. MySQL in 1995. SQLite in 2000 — designed to be embedded in other programs (Firefox, Android, iOS apps all use it).

Object-Relational Mappers emerged in the late 1990s as web apps needed to bridge OOP languages and relational DBs. Hibernate (Java, 2001), ActiveRecord (Ruby on Rails, 2004), Django ORM (2005), SQLAlchemy (2005). Each took a slightly different approach — ActiveRecord is "convention over configuration," SQLAlchemy is "explicit and powerful."

SQLAlchemy 1.0 (2010) stabilised the API. SQLAlchemy 2.0 (2023) was a major rewrite unifying Core and ORM onto a single `select()` API, adding type hints and async. Flask-SQLAlchemy 3.x (2022) adopted 2.0 internally. The 2.0 style with `Mapped[T]` is the 2026 standard for new projects.

## 7. Common Mistakes

### 7.1 Forgetting `db.session.commit()`

```python
user = User(username='alice', email='alice@example.com')
db.session.add(user)
# forgot commit!
return redirect(url_for('index'))
```

Without `commit()`, the user is staged in the session but never written to the DB. On the next request, they're gone. Always commit after `add()`, `delete()`, or modifying attributes.

> [!warning] Flask-SQLAlchemy auto-commit gotcha
> Flask-SQLAlchemy 3.x *used* to auto-commit at the end of each request. That behaviour was removed in 3.1 — you must commit explicitly. Don't rely on auto-commit. Always `db.session.commit()` after writes.

### 7.2 Forgetting the application context

```python
# Outside a Flask route, e.g., in a script:
from app import db, User
db.create_all()   # RuntimeError: working outside of application context
```

Wrap context-dependent code:

```python
from app import app, db
with app.app_context():
    db.create_all()
```

### 7.3 `db.create_all()` doesn't update existing tables

```python
class User(db.Model):
    # ... existing columns ...
    bio = db.Column(db.Text)   # NEW column added later
```

You add a `bio` column, run `db.create_all()`, but the existing `users` table doesn't get the new column. `create_all()` only creates tables that don't exist. Use **Flask-Migrate** (Lesson 15) for schema changes.

### 7.4 `db.Table` vs `db.Model`

```python
# WRONG — db.Table is for association tables, not models
class User(db.Table):
    ...
```

`db.Model` is for full ORM models (with classes, instances, attributes). `db.Table` is for raw SQL tables — used for many-to-many association tables (Lesson 14). They're different.

### 7.5 Reserved table names

```python
class Order(db.Model):
    pass   # __tablename__ defaults to 'order' — a SQL reserved word!
```

`order`, `user`, `group` are reserved in many SQL dialects. Always set `__tablename__` explicitly and use plural names (`users`, `orders`, `groups`).

### 7.6 `nullable=False` without validation

```python
email = db.Column(db.String(120), nullable=False)
```

`nullable=False` means the *database* rejects NULL. But your Flask code can still try to insert NULL — you'll get an `IntegrityError` at runtime. Always validate *before* committing (e.g., with Flask-WTF's `DataRequired()`).

### 7.7 Model import ordering

```python
# app.py
db = SQLAlchemy(app)

class Post(db.Model):
    author_id = db.Column(db.Integer, db.ForeignKey('users.id'))   # references 'users'

class User(db.Model):
    __tablename__ = 'users'
    ...
```

`Post` references `users.id` before `User` is defined. This works because `ForeignKey('users.id')` is just a string reference, not an import. But it's brittle — define models in dependency order, or use `from app import models` to load them all at once.

### 7.8 Mixing legacy `Model.query` and 2.0 `select()` inconsistently

```python
# Don't mix styles within one codebase unless you have a reason
users = User.query.all()                              # legacy
users = db.session.execute(db.select(User)).scalars().all()  # 2.0
```

Both work, but mixing them confuses readers. Pick one (prefer 2.0) and apply it consistently.

### 7.9 Using `datetime.now()` instead of `datetime.utcnow()`

```python
created_at = db.Column(db.DateTime, default=datetime.now)  # local time — BAD
```

`datetime.now()` returns server-local time — which varies with timezone. `datetime.utcnow()` returns UTC, which is timezone-agnostic. In production, use timezone-aware `datetime.now(timezone.utc)` (Python 3.11+). For SQLite, store UTC and convert in templates.

## 8. Exercises

> [!tip] The 20-minute rule
> Stuck for 20 minutes? Take a break.

1. **Add a `Post` model.** Define a `Post` model with `id`, `title` (String 200, not null), `body` (Text), `created_at` (DateTime, default utcnow). Run `db.create_all()`. Verify the table appears in `sqlite3 .tables`.

2. **`default` callable vs called.** Define two models — one with `default=datetime.utcnow()`, another with `default=datetime.utcnow`. Add a row to each, wait 2 seconds, add another row. Compare the `created_at` values. What do you notice?

3. **Add a `bio` column.** Add `bio = db.Column(db.Text)` to your `User` model. Restart the server. Does the new column appear in the DB? (It shouldn't — that's the point. We'll fix this with migrations in Lesson 15.)

4. **`__tablename__` for a reserved word.** Define an `Order` model without setting `__tablename__`. What's the default? Why might that be a problem? Add `__tablename__ = 'orders'` to fix it.

5. **Column types exploration.** Define a model with one column of each type: `Integer`, `Float`, `Numeric`, `String`, `Text`, `Boolean`, `Date`, `DateTime`, `JSON`. Insert a row using the Flask shell. Inspect with `sqlite3 .schema` and `SELECT * FROM`. Notice how SQLite stores each type.

6. **`__repr__` practice.** Add a `__repr__` method to your `Post` model that shows the title and ID. Verify it improves debugging output.

7. **Switch to 2.0 style.** Rewrite your `User` model using `Mapped[T]` and `mapped_column()`. Run the app — confirm everything still works. Notice the type hints in your IDE.

8. **In-memory database for tests.** Create a second Flask app config that uses `sqlite:///:memory:`. Write a small script that creates a user, queries it back, and confirms it works — all without touching `myapp.db` on disk.

9. **Connect to PostgreSQL.** If you have PostgreSQL installed locally, change the URI to `postgresql://user:pass@localhost/myapp`. Create the database with `createdb myapp`. Run `db.create_all()`. Connect with `psql myapp` and verify the `users` table exists. (Skip if you don't have Postgres.)

10. **Turn on SQL echo.** Set `app.config['SQLALCHEMY_ECHO'] = True`. Reload the page and look at the terminal output. Identify every SQL statement the ORM runs. Then add a user via the shell with echo on — observe the INSERT statement and the parameter binding. Count how many queries it takes to load the index page.

11. **Inspect the connection pool.** Add `SQLALCHEMY_ENGINE_OPTIONS={'pool_size': 2, 'max_overflow': 0}`. Write a view that opens 5 concurrent DB connections in threads (or just a query in a loop). What happens when the pool is exhausted? (You should see a `TimeoutError`.)

12. **Write a raw SQL query.** Use `db.session.execute(db.text('SELECT username FROM users WHERE age > :age'), {'age': 20}).fetchall()` to run raw SQL. When would you prefer this over the ORM? (Hint: aggregate reports, complex JOINs.)

## 9. What We Learned

- **A database** is an organised collection of data with a DBMS that stores and queries it. **Relational databases** organise data into **tables** (rows + columns). **SQL** is the language for talking to them.
- An **ORM** (Object-Relational Mapper) translates Python objects ↔ database rows. **SQLAlchemy** is the most popular Python ORM; **Flask-SQLAlchemy** integrates it with Flask.
- SQLAlchemy has two layers: **Core** (SQL expressions, connections) and **ORM** (models, sessions, relationships). You typically use the ORM, dropping to Core/raw SQL for complex queries.
- **Database URI**: `sqlite:///myapp.db` (3 slashes = relative), `sqlite:////tmp/myapp.db` (4 slashes = absolute), `postgresql://user:pass@host/db`.
- **SQLite** for development and small apps; **PostgreSQL** for production. ORM makes them interchangeable.
- The **`db` object** (`db = SQLAlchemy(app)`) is your one-stop shop: `db.Model` (base class), `db.Column` (column def), `db.session` (session), `db.select` / `db.func` (SQL expressions).
- A **model** is a Python class inheriting `db.Model`. Each class attribute is a column. Column types include `Integer`, `String`, `Text`, `DateTime`, `Boolean`, `JSON`. Common options: `primary_key`, `unique`, `nullable`, `default`, `index`.
- **`db.create_all()`** creates tables that don't exist — but does NOT alter existing ones. Use **Flask-Migrate** (Lesson 15) for schema changes.
- The **session** (`db.session`) is a Unit of Work — stage changes with `add()`/attribute modifications, flush them with `commit()`, undo with `rollback()`. It's per-request, lazy, transactional, and an identity map.
- **SQLAlchemy 2.0 style** uses `Mapped[T]` type hints + `mapped_column()` for better IDE support and type safety. The new query API is `db.session.execute(db.select(...))`.
- **Async SQLAlchemy** (`AsyncSession`, `asyncpg` driver) supports high-concurrency I/O workloads. Only use it when you have parallel DB calls or other async I/O.
- **Connection pooling** reuses DB connections across requests. Defaults work for development; tune `pool_size` in production.

## 10. Next Lesson

You have a database and a model. Now let's do something with it.

In [[Lesson-13-CRUD-Operations]] you'll learn:

- **CRUD** — Create, Read, Update, Delete — the four operations on data.
- **The Query API** in depth (legacy `Model.query` and 2.0 `select()` style).
- **Filtering**: `filter_by()` (simple) vs `filter()` (expressions) — and operators like `==`, `!=`, `>`, `<`, `like`, `ilike`, `in_`, `is_`, `between`.
- **Ordering, limiting, offset, pagination.**
- **`.first()`, `.all()`, `.one()`, `.one_or_none()`, `.get()`, `.get_or_404()`.**
- **Aggregates**: `count()`, `func.sum()`, `func.avg()`.
- **Relationships in queries (JOINs).**
- **The session lifecycle** — what happens on add, commit, rollback.
- **Transactions and atomicity** — multi-statement transactions.
- A complete **user management app** with search, pagination, and CRUD routes.

> [!quote] A note from your teacher
> Databases are where most beginners hit a wall. The ORM hides SQL, but it can't hide the *concepts* — primary keys, foreign keys, transactions, sessions, the N+1 problem. Don't rush this module. Build the exercises, write the queries by hand, watch the SQL log. Once you internalise the relational model, every other database technology (NoSQL, graph DBs, time-series DBs) becomes a variation on a theme you already know.
