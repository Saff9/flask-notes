---
title: "Quick Reference — One-Page Flask Lookup"
module: 14-Reference
type: reference
audience: "All students — glance here, then go back to coding"
tags:
  - flask
  - quick-reference
  - reference
  - lookup
  - cheatsheet
related_lessons:
  - "[[Lesson-01-Hello-World]]"
  - "[[Lesson-02-Project-Structure]]"
  - "[[Lesson-04-Routes]]"
  - "[[Lesson-05-URL-Converters]]"
  - "[[Lesson-06-HTTP-Methods]]"
  - "[[Lesson-07-Jinja2-Basics]]"
  - "[[Lesson-12-SQLAlchemy-Setup]]"
  - "[[Lesson-13-CRUD-Operations]]"
  - "[[Lesson-15-Flask-Migrate]]"
  - "[[Lesson-23-Deployment]]"
---

# Quick Reference — One-Page Flask Lookup

> [!info] This is the file you keep open in a side tab.
> Glance at it when you can't remember an import, a CLI command, or an HTTP status code. Then go back to coding. For deeper context, click the lesson links — for copy-paste patterns, see [[Flask-Cheat-Sheet]] — for fixing what's broken, see [[Common-Errors]].

#quick-reference #lookup #reference

---

## 1. Common Imports

```python
# Core Flask
from flask import Flask, render_template, redirect, url_for, abort
from flask import request, session, g, current_app, jsonify, flash
from flask import make_response, send_file, send_from_directory, Response
from flask import Blueprint

# Extensions
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_wtf import FlaskForm
from flask_wtf.file import FileField, FileAllowed, FileRequired
from flask_login import (LoginManager, UserMixin, login_user, logout_user,
                         login_required, current_user)
from flask_mail import Mail, Message

# Werkzeug
from werkzeug.security import generate_password_hash, check_password_hash
from werkzeug.utils import secure_filename
from werkzeug.urls import url_parse

# WTForms fields & validators
from wtforms import (StringField, PasswordField, BooleanField, SubmitField,
                     TextAreaField, IntegerField, FloatField, SelectField,
                     RadioField, DateField, DateTimeField, HiddenField)
from wtforms.validators import (DataRequired, Email, Length, EqualTo,
                                NumberRange, URL, Optional, Regexp,
                                ValidationError, AnyOf)

# SQLAlchemy
from sqlalchemy import or_, and_, not_, func, desc, asc
from sqlalchemy.orm import joinedload, selectinload

# Standard library you'll use a lot
import os
from datetime import datetime, timedelta
from dotenv import load_dotenv
from threading import Thread
```

---

## 2. Common Route Patterns

| Pattern | Code |
|---|---|
| Basic | `@app.route('/')` |
| Dynamic int | `@app.route('/post/<int:id>')` |
| Dynamic string | `@app.route('/user/<username>')` |
| Multiple | `@app.route('/u/<username>/p/<int:post_id>')` |
| Methods | `@app.route('/login', methods=['GET', 'POST'])` |
| Trailing slash | `@app.route('/projects/')` |
| Blueprint | `bp = Blueprint('blog', __name__, url_prefix='/blog')` |
| Reverse | `url_for('blog.view_post', id=42)` |
| External | `url_for('blog.view_post', id=42, _external=True)` |
| Static | `url_for('static', filename='css/style.css')` |
| Abort | `abort(404)` / `abort(403)` / `abort(500)` |
| Redirect | `redirect(url_for('index'))` |

→ Full patterns: [[Flask-Cheat-Sheet]] §2 — Lesson: [[Lesson-04-Routes]]

---

## 3. Common Template Snippets

```jinja2
{# Inheritance #}
{% extends 'base.html' %}
{% block content %} ... {% endblock %}

{# Variables #}
{{ user.username }}
{{ user.email | default('—') }}
{{ items | length }}

{# Conditionals #}
{% if user.is_authenticated %} ... {% endif %}
{% if x %} ... {% elif y %} ... {% else %} ... {% endif %}

{# Loops #}
{% for item in items %}
  {{ loop.index }} {{ item }}
{% else %}
  No items.
{% endfor %}

{# Includes #}
{% include 'partials/_nav.html' %}

{# Set #}
{% set greeting = 'Hi, ' + user.username %}

{# URL generation #}
<a href="{{ url_for('blog.view_post', id=post.id) }}">Read</a>
<img src="{{ url_for('static', filename='img/logo.png') }}">

{# Flash messages #}
{% with msgs = get_flashed_messages(with_categories=true) %}
  {% for cat, msg in msgs %}
    <div class="alert alert-{{ cat }}">{{ msg }}</div>
  {% endfor %}
{% endwith %}

{# CSRF token (every POST form) #}
<form method="post">
  {{ form.hidden_tag() }}
  ...
</form>

{# Render a WTForms field #}
{{ form.username.label }} {{ form.username(size=32) }}
{% for err in form.username.errors %}
  <span class="error">{{ err }}</span>
{% endfor %}

{# Current user state #}
{% if current_user.is_authenticated %}
  Hi, {{ current_user.username }}
{% else %}
  <a href="{{ url_for('auth.login') }}">Sign in</a>
{% endif %}
```

→ Lesson: [[Lesson-07-Jinja2-Basics]] + [[Lesson-08-Template-Logic]]

---

## 4. Common Query Patterns

```python
# Read
User.query.get(id)                 # by PK or None
User.query.get_or_404(id)          # by PK or 404
User.query.all()                   # all rows
User.query.first()                 # first row or None
User.query.first_or_404()          # first row or 404
User.query.count()                 # count

# Filter
User.query.filter_by(username='alice').first()    # kwargs (AND only)
User.query.filter(User.email.endswith('@x.com')).all()  # expressions
User.query.filter(or_(User.a == 1, User.b == 2))  # OR
User.query.filter(and_(...)).filter(...)          # AND

# Order, limit, offset
Post.query.order_by(Post.created_at.desc()).limit(10).all()
Post.query.order_by(Post.created_at.desc()).offset(20).limit(10).all()

# Pagination (Flask-SQLAlchemy 3.x)
pagination = Post.query.paginate(page=1, per_page=10, error_out=False)
posts = pagination.items
pagination.total, pagination.pages, pagination.has_next, pagination.has_prev

# Create
user = User(username='alice')
db.session.add(user)
db.session.commit()

# Update
user.bio = 'Hi!'
db.session.commit()

# Delete
db.session.delete(user)
db.session.commit()

# Aggregates
from sqlalchemy import func
db.session.query(func.count(User.id)).scalar()
db.session.query(User.country, func.count(User.id)).group_by(User.country).all()

# Relationships
user.posts.all()                  # one-to-many
post.tags.all()                   # many-to-many
post.comments.order_by(Comment.created_at.asc()).all()

# Avoid N+1
Post.query.options(joinedload(Post.author), selectinload(Post.tags)).all()
```

→ Lesson: [[Lesson-13-CRUD-Operations]] + [[Lesson-14-Relationships]]

---

## 5. Common CLI Commands

### 5.1 Flask CLI

```bash
flask run                      # start dev server
flask run --port 5001          # different port
flask run --host 0.0.0.0       # visible on network
flask run --debug              # debug mode (auto-reload + debugger)
flask shell                    # interactive REPL with app context
flask routes                   # list all registered routes
flask --app run.py run         # specify app module explicitly
```

### 5.2 Flask-Migrate (Alembic) CLI

```bash
flask db init                  # one-time setup, creates migrations/
flask db migrate -m "msg"      # generate migration from model changes
flask db upgrade               # apply pending migrations
flask db downgrade             # undo last migration
flask db current               # show current revision
flask db history               # show all revisions
flask db stamp head            # mark DB as up-to-date without running migrations
flask db merge -m "merge"      # merge multiple heads
```

→ Lesson: [[Lesson-15-Flask-Migrate]]

### 5.3 Production

```bash
# Gunicorn (Linux/macOS)
gunicorn -w 4 -b 0.0.0.0:8000 'run:app'
gunicorn -w $(($(nproc) * 2 + 1)) -b 0.0.0.0:8000 'run:app'

# Waitress (Windows)
waitress-serve --port=8000 run:app

# Docker
docker build -t myblog .
docker run -p 5000:5000 --env-file .env myblog
docker-compose up --build
docker-compose exec web flask db upgrade
docker-compose exec db psql -U bloguser blog
```

→ Lesson: [[Lesson-23-Deployment]]

### 5.4 Database inspection

```bash
# SQLite
sqlite3 app.db ".tables"
sqlite3 app.db ".schema users"
sqlite3 app.db "SELECT * FROM users;"

# Postgres
psql -h localhost -U bloguser -d blog
\dt                            # list tables
\d users                       # describe table
\q                             # quit
```

### 5.5 Testing

```bash
pytest -v                      # verbose
pytest tests/test_models.py    # one file
pytest -k "login"              # filter by name
pytest --maxfail=1             # stop on first failure
pytest --pdb                   # drop into debugger on failure
pytest --cov=app               # coverage report
```

---

## 6. Common HTTP Status Codes

| Code | Meaning | When to use |
|---|---|---|
| 200 | OK | Successful GET, POST that returns content |
| 201 | Created | Successful POST that creates a resource |
| 204 | No Content | Successful DELETE, PUT with no body |
| 301 | Moved Permanently | Old URL forever redirected |
| 302 | Found | Temporary redirect (default for `redirect()`) |
| 304 | Not Modified | Cache hit |
| 400 | Bad Request | Malformed input, missing field |
| 401 | Unauthorized | Not logged in |
| 403 | Forbidden | Logged in but not allowed |
| 404 | Not Found | Resource doesn't exist |
| 405 | Method Not Allowed | POST to a GET-only route |
| 409 | Conflict | Duplicate resource |
| 422 | Unprocessable Entity | Validation failed |
| 429 | Too Many Requests | Rate limit hit |
| 500 | Internal Server Error | Your code crashed |
| 502 | Bad Gateway | Upstream service down |
| 503 | Service Unavailable | Maintenance mode |

```python
# Returning specific codes
return render_template('404.html'), 404
return jsonify({'error': 'gone'}), 410
return '', 204
abort(403)
```

→ Lesson: [[Lesson-06-HTTP-Methods]]

---

## 7. Common Environment Variables

| Variable | Purpose |
|---|---|
| `FLASK_APP` | Module containing the app (e.g. `run.py`) |
| `FLASK_ENV` | `development` or `production` |
| `FLASK_DEBUG` | `1` to enable debug mode |
| `FLASK_CONFIG` | Which config class to load (e.g. `production`) |
| `SECRET_KEY` | Signs sessions, CSRF tokens, flash messages |
| `DATABASE_URL` | SQLAlchemy URI (e.g. `postgresql://user:pass@host/db`) |
| `MAIL_SERVER` | SMTP host |
| `MAIL_PORT` | SMTP port (587 for TLS, 465 for SSL) |
| `MAIL_USERNAME` | SMTP user |
| `MAIL_PASSWORD` | SMTP password |
| `MAIL_DEFAULT_SENDER` | From: address |

```python
# Loading env vars
import os
from dotenv import load_dotenv
load_dotenv()  # reads .env into os.environ

secret = os.environ.get('SECRET_KEY') or 'dev-fallback'
```

→ Lesson: [[Lesson-22-Email-And-Tasks]]

---

## 8. Common Configuration Keys

```python
app.config['SECRET_KEY']                       # str, signs sessions
app.config['SQLALCHEMY_DATABASE_URI']          # str, DB connection
app.config['SQLALCHEMY_TRACK_MODIFICATIONS']   # bool, False recommended
app.config['SQLALCHEMY_ECHO']                  # bool, print SQL queries
app.config['WTF_CSRF_ENABLED']                 # bool, disable for tests
app.config['WTF_CSRF_TIME_LIMIT']              # int, seconds (default 3600)
app.config['PERMANENT_SESSION_LIFETIME']       # timedelta, default 31 days
app.config['MAX_CONTENT_LENGTH']               # int, bytes — request size cap
app.config['UPLOAD_FOLDER']                    # str, path for uploads
app.config['PREFERRED_URL_SCHEME']             # 'http' or 'https'
app.config['DEBUG']                            # bool
app.config['TESTING']                          # bool
app.config['SERVER_NAME']                      # str, for url_for external
```

---

## 9. Common Jinja Filters

| Filter | Effect | Example |
|---|---|---|
| `default('x')` | fallback if undefined | `{{ name \| default('Anon') }}` |
| `upper` / `lower` | case | `{{ name \| upper }}` |
| `capitalize` | first letter cap | `{{ name \| capitalize }}` |
| `title` | title case | `{{ title \| title }}` |
| `length` | count | `{{ items \| length }}` |
| `truncate(n)` | truncate + ellipsis | `{{ body \| truncate(100) }}` |
| `round(n)` | decimal places | `{{ price \| round(2) }}` |
| `join(sep)` | list to string | `{{ tags \| join(', ') }}` |
| `replace(a, b)` | substring replace | `{{ s \| replace('a', 'b') }}` |
| `escape` / `e` | HTML escape | `{{ input \| escape }}` |
| `safe` | skip auto-escaping | `{{ html \| safe }}` ⚠️ |
| `striptags` | remove HTML tags | `{{ html \| striptags }}` |
| `trim` | strip whitespace | `{{ s \| trim }}` |
| `first` / `last` | list endpoints | `{{ items \| first }}` |
| `sort` | sort ascending | `{{ items \| sort }}` |
| `reverse` | reverse list | `{{ items \| reverse \| list }}` |
| `tojson` | JSON-encode | `{{ data \| tojson }}` |

→ Lesson: [[Lesson-08-Template-Logic]]

---

## 10. Common Model Field Types

| WTForms Field | HTML Input | Python Type |
|---|---|---|
| `StringField` | `<input type="text">` | str |
| `PasswordField` | `<input type="password">` | str |
| `TextAreaField` | `<textarea>` | str |
| `IntegerField` | `<input type="number">` | int |
| `FloatField` / `DecimalField` | `<input type="number">` | float / Decimal |
| `BooleanField` | `<input type="checkbox">` | bool |
| `DateField` / `DateTimeField` | `<input type="date">` | date / datetime |
| `SelectField` | `<select>` | str |
| `SelectMultipleField` | `<select multiple>` | list |
| `RadioField` | `<input type="radio">` | str |
| `FileField` | `<input type="file">` | FileStorage |
| `SubmitField` | `<input type="submit">` | — |
| `HiddenField` | `<input type="hidden">` | str |
| `EmailField` | `<input type="email">` | str |
| `URLField` | `<input type="url">` | str |

→ Lesson: [[Lesson-11-Flask-WTF]]

---

## 11. Common SQLAlchemy Column Types

| Column | SQL Type |
|---|---|
| `db.Integer` | INTEGER |
| `db.String(n)` | VARCHAR(n) |
| `db.Text` | TEXT |
| `db.Boolean` | BOOLEAN |
| `db.DateTime` | DATETIME |
| `db.Date` | DATE |
| `db.Time` | TIME |
| `db.Float` | FLOAT |
| `db.Numeric(p, s)` | NUMERIC |
| `db.LargeBinary` | BLOB |
| `db.JSON` | JSON |
| `db.Enum('a', 'b')` | ENUM |
| `db.PickleType` | BLOB (Python pickle) |
| `db.UUID` | UUID |

→ Lesson: [[Lesson-12-SQLAlchemy-Setup]]

---

## 12. Common Validators

| Validator | Purpose |
|---|---|
| `DataRequired()` | not empty and not None |
| `InputRequired()` | not empty (accepts whitespace) |
| `Email()` | valid email format |
| `Length(min=, max=)` | string length range |
| `NumberRange(min=, max=)` | numeric range |
| `URL()` | valid URL |
| `EqualTo('other')` | match another field |
| `Optional()` | skip if empty |
| `Regexp(r'...')` | regex match |
| `AnyOf(['a', 'b'])` | must be in list |
| `NoneOf(['x', 'y'])` | must NOT be in list |
| `IPAddress()` | valid IP |
| `UUID()` | valid UUID |

→ Lesson: [[Lesson-11-Flask-WTF]]

---

## 13. Common HTTP Methods

| Method | CRUD | Idempotent | Safe | Use |
|---|---|---|---|---|
| GET | Read | Yes | Yes | Fetch a resource |
| POST | Create | No | No | Create a new resource |
| PUT | Update/Replace | Yes | No | Replace a resource |
| PATCH | Update | No | No | Partial update |
| DELETE | Delete | Yes | No | Remove a resource |
| HEAD | Read | Yes | Yes | Headers only (no body) |
| OPTIONS | — | Yes | Yes | What methods are allowed? |

→ Lesson: [[Lesson-06-HTTP-Methods]]

---

## 14. Common Decorators

```python
@app.route('/path')                 # register URL
@app.route('/path', methods=['POST'])
@login_required                     # require authenticated user
@csrf.exempt                        # disable CSRF (use carefully)
@app.before_request                 # run before every request
@app.after_request                  # run after every request
@app.teardown_request               # cleanup, even on error
@app.errorhandler(404)              # custom error page
@app.template_filter('name')        # register Jinja filter
@app.context_processor              # inject template variables
```

```python
# Custom context processor — make var available in every template
@app.context_processor
def inject_user():
    return dict(current_year=datetime.utcnow().year)
# Now {{ current_year }} works in any template.
```

---

## 15. Common File Locations

```
myapp/
├── app/                       # application package
│   ├── __init__.py            # create_app() factory
│   ├── models.py              # SQLAlchemy models
│   ├── extensions.py          # db, migrate, login, etc.
│   ├── auth/                  # auth blueprint
│   │   ├── __init__.py        # bp = Blueprint(...)
│   │   ├── forms.py
│   │   └── routes.py
│   ├── blog/                  # blog blueprint
│   ├── templates/             # Jinja templates
│   │   ├── base.html
│   │   ├── auth/
│   │   └── blog/
│   └── static/                # CSS, JS, images, uploads
│       ├── css/style.css
│       ├── js/app.js
│       └── uploads/
├── migrations/                # Alembic migrations
│   ├── env.py
│   └── versions/              # generated migration scripts
├── tests/
│   ├── conftest.py            # pytest fixtures
│   ├── test_models.py
│   └── test_routes.py
├── config.py                  # config classes
├── requirements.txt
├── .env                       # secrets, NOT in git
├── .flaskenv                  # FLASK_APP, FLASK_ENV (in git)
├── .gitignore
├── Dockerfile
├── docker-compose.yml
└── run.py                     # entry point: app = create_app()
```

→ Lesson: [[Lesson-02-Project-Structure]] + [[Lesson-18-Blueprints]]

---

## 16. Command Reference Mindmap

```mermaid
mindmap
  root((Flask CLI))
    Run
      flask run
      flask run --debug
      flask run --port 5001
      flask shell
    Inspect
      flask routes
      flask --version
    Database
      flask db init
      flask db migrate
      flask db upgrade
      flask db downgrade
      flask db current
      flask db history
    Deploy
      gunicorn run:app
      docker build
      docker-compose up
      psql
      sqlite3
    Test
      pytest -v
      pytest -k filter
      pytest --pdb
      pytest --cov
```

---

## 17. Common "How Do I..." Quick Answers

| Question | Answer | Lesson |
|---|---|---|
| How do I get a query param? | `request.args.get('q', '')` | [[Lesson-05-URL-Converters]] |
| How do I get POST data? | `request.form['field']` | [[Lesson-06-HTTP-Methods]] |
| How do I get JSON from a request? | `request.get_json()` | [[Lesson-19-Building-JSON-API]] |
| How do I redirect? | `return redirect(url_for('index'))` | [[Lesson-04-Routes]] |
| How do I render a template? | `render_template('page.html', x=1)` | [[Lesson-07-Jinja2-Basics]] |
| How do I show a flash message? | `flash('Saved!', 'success')` | [[Lesson-10-HTML-Forms]] |
| How do I hash a password? | `generate_password_hash(pw)` | [[Lesson-16-User-Model-AndPasswords]] |
| How do I log a user in? | `login_user(user)` | [[Lesson-17-Flask-Login]] |
| How do I require login? | `@login_required` | [[Lesson-17-Flask-Login]] |
| How do I get the current user? | `current_user` | [[Lesson-17-Flask-Login]] |
| How do I add a column? | `flask db migrate` + `flask db upgrade` | [[Lesson-15-Flask-Migrate]] |
| How do I upload a file? | `request.files['field']` + `secure_filename` | [[Lesson-21-File-Uploads]] |
| How do I return JSON? | `return jsonify({...})` | [[Lesson-19-Building-JSON-API]] |
| How do I send an email? | `mail.send(Message(...))` | [[Lesson-22-Email-And-Tasks]] |
| How do I run in production? | `gunicorn run:app` | [[Lesson-23-Deployment]] |
| How do I create a blueprint? | `bp = Blueprint('x', __name__)` | [[Lesson-18-Blueprints]] |
| How do I raise a 404? | `abort(404)` or `Model.query.get_or_404(id)` | [[Lesson-13-CRUD-Operations]] |
| How do I add a static file link? | `url_for('static', filename='...')` | [[Lesson-09-Static-Files]] |
| How do I add a CSRF token? | `{{ form.hidden_tag() }}` in template | [[Lesson-11-Flask-WTF]] |
| How do I paginate? | `query.paginate(page=1, per_page=10)` | [[Lesson-13-CRUD-Operations]] |

---

## 18. Where to Go Next

- **[[Flask-Cheat-Sheet]]** — copy-paste patterns for every situation.
- **[[Common-Errors]]** — what to do when this sheet's patterns break.
- **[[Map-of-Content]]** — the master index of every lesson.
- **[[Lesson-24-Build-A-Blog]]** — the capstone that uses everything here.
- **Official Flask docs** — <https://flask.palletsprojects.com/>
- **Flask-SQLAlchemy docs** — <https://flask-sqlalchemy.palletsprojects.com/>
- **Flask-WTF docs** — <https://flask-wtf.readthedocs.io/>
- **Awesome Flask** — <https://github.com/humiaozuzu/awesome-flask>

> [!tip] Bookmark this page.
> When you can't remember a status code, when you forget whether it's `filter_by` or `filter`, when you can't recall the validators list — come back here. Five seconds of lookup beats ten minutes of guessing.
