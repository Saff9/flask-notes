---
title: "Lesson 23 — Deployment: Gunicorn, Nginx, Docker, and Going Live"
module: 11-Deployment
lesson_number: 23
estimated_time: "180 minutes"
difficulty: advanced
prerequisites:
  - "Completed every previous lesson — this is the final lesson and ties everything together"
  - "Specifically [[Lesson-22-Email-And-Tasks]] — environment variables, secrets, and the structure of a complete app"
  - "Specifically [[Lesson-15-Flask-Migrate]] — you'll run migrations in production"
  - "Specifically [[Lesson-16-User-Registration]] — production needs strong SECRET_KEY for sessions"
  - "Comfortable on the Linux command line — you'll install packages, edit config files, and run docker"
tags:
  - flask
  - deployment
  - gunicorn
  - nginx
  - docker
  - wsgi
  - production
  - env-vars
  - lesson
related_lessons:
  - "[[Lesson-22-Email-And-Tasks]]"
  - "[[Lesson-15-Flask-Migrate]]"
  - "[[Lesson-16-User-Registration]]"
  - "[[Lesson-02-Project-Structure]]"
---

# Lesson 23 — Deployment: Gunicorn, Nginx, Docker, and Going Live

> [!success] Final lesson of the course — Module 11: Deployment.
> You've spent 22 lessons building Flask apps that run on `http://127.0.0.1:5000`. That's the *development* server — a single Python process that you started by hand, with `debug=True`, on your own laptop. It's perfect for writing code and terrible for serving real users. Today you'll learn what changes when you go to production: a different server (Gunicorn), a reverse proxy in front (Nginx), environment variables for secrets, Docker for reproducible packaging, and a checklist of things you must verify before launch. By the end, you'll be able to take any app from this course and put it on the public internet — properly.

## 1. Learning Objectives

By the end of this lesson you will be able to:

1. **Explain why `app.run(debug=True)` is forbidden in production** — single-threaded, debug exposes code, no process management, no scale.
2. **Describe the WSGI contract** — what Gunicorn gives Flask, and what Flask gives Gunicorn.
3. **Run Gunicorn** with multiple workers and explain the `2 × CPU + 1` rule.
4. **Use environment variables and `.env` files** for secrets, loaded with `python-dotenv`.
5. **Configure Nginx as a reverse proxy** in front of Gunicorn and explain *why* (TLS, static files, buffering, load balancing).
6. **Write a `Dockerfile`** that packages your Flask app + Gunicorn into a reproducible image.
7. **Write a `docker-compose.yml`** that runs Flask + Postgres + Redis together.
8. **Run database migrations** in a containerized deployment.
9. **Apply the production checklist** — debug off, strong SECRET_KEY, Postgres not SQLite, HTTPS, logging, monitoring, backups.
10. **Choose a deployment platform** (Heroku, Render, Railway, DigitalOcean, AWS) based on your needs.

#deployment #gunicorn #nginx #docker #wsgi #production

## 2. Prerequisites

Before this lesson you should be comfortable with:

- **Virtual environments and `requirements.txt`** — [[Lesson-02-Project-Structure]].
- **Configuration** — [[Lesson-22-Email-And-Tasks]] showed you environment variables for secrets.
- **Database migrations** — [[Lesson-15-Flask-Migrate]]. In production you run them before the new code starts.
- **The structure of a complete app** — routes, templates, models, blueprints.

This lesson assumes you can SSH into a Linux server, edit a file with `nano` or `vim`, and run `sudo apt install`. If you've never touched a Linux shell, the Docker path (Section 6) is friendlier — it abstracts most of the OS away.

## 3. The Problem

### 3.1 "It works on my laptop — why won't it work in production?"

You wrote `app.run(debug=True)`, opened `http://127.0.0.1:5000`, and everything works. Then you put it on a server and:

- **It's slow.** One request at a time. Two users hit it simultaneously — one waits.
- **It's insecure.** `debug=True` means anyone who triggers an error sees a live Python debugger in their browser. That debugger can execute arbitrary code. There are bots scanning the internet for `/console` right now.
- **It crashes silently.** The dev server doesn't restart on crash. A single unhandled exception kills the process and your site is down.
- **It can't do HTTPS.** The dev server has no TLS support. Browsers will flag your site as "not secure."
- **It serves static files badly.** The dev server sends CSS/JS through Python — fine for one developer, deadly for 1000 users.
- **There's no process manager.** You SSH in, run `python app.py`, and the moment your SSH session closes, the server dies.

```mermaid
flowchart TD
    DEV[Development<br/>app.run debug=True]
    DEV --> D1[Single thread]
    DEV --> D2[Debug exposes code]
    DEV --> D3[No TLS]
    DEV --> D4[Dies when SSH closes]
    DEV --> D5[Python serves static files]
    PROD[Production]
    PROD --> P1[Gunicorn: 4+ workers]
    PROD --> P2[debug=False]
    PROD --> P3[Nginx terminates TLS]
    PROD --> P4[systemd or Docker: auto-restart]
    PROD --> P5[Nginx serves static files directly]
    style DEV fill:#fdd,stroke:#900
    style PROD fill:#dfd,stroke:#070
```

### 3.2 The production architecture

A real Flask deployment has four layers:

```mermaid
flowchart LR
    U[User's browser] -->|HTTPS:443| N[Nginx<br/>reverse proxy]
    N -->|HTTP:8000| G[Gunicorn<br/>WSGI server]
    G -->|4 worker processes| F[Flask app<br/>app:app]
    F --> DB[(PostgreSQL)]
    F --> R[(Redis<br/>for Celery)]
    N -->|static files| S[static/ folder]
    N -->|TLS cert| LE[Let's Encrypt]
```

1. **Nginx** (port 443) — accepts the user's HTTPS request, terminates TLS, serves static files directly, and proxies everything else to Gunicorn.
2. **Gunicorn** (port 8000, localhost only) — a WSGI server. It forks N worker processes and load-balances incoming requests among them.
3. **Flask app** (inside Gunicorn) — your code. Doesn't know it's running under Gunicorn; it just answers WSGI calls.
4. **PostgreSQL** and **Redis** — backing services, often on separate machines.

The key insight: **Flask doesn't serve the internet. Gunicorn does. Nginx shields Gunicorn from the internet.** Your Flask code is the innermost layer.

### 3.3 What is WSGI?

**WSGI** (Web Server Gateway Interface) is a Python standard (PEP 3333). It's a *contract*: a Python callable that takes `(environ, start_response)` and returns an iterable of bytes. Flask's `app` object is a WSGI callable. Gunicorn is a WSGI server — it calls your `app`.

```python
# A minimal WSGI app — this is essentially what Flask's `app` is:
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return [b'Hello, world!\n']
```

You don't write this yourself — Flask does. But knowing it exists tells you why `gunicorn app:app` works: Gunicorn imports the `app` variable from your `app.py` module and calls it. That's the whole interface.

> [!info] ASGI is the newer standard
> WSGI is synchronous. For WebSockets and async, there's **ASGI** (used by FastAPI, Starlette). Flask supports ASGI via the `asgiref` shim, but the classic, battle-tested deployment is WSGI + Gunicorn. We'll use that here.

## 4. The Code

### 4.1 The production-ready app

Create a folder `myapp/` with this structure:

```
myapp/
├── app.py
├── requirements.txt
├── .env.example
├── .gitignore
├── Dockerfile
├── docker-compose.yml
├── nginx.conf
└── templates/
    └── index.html
```

`app.py`:

```python
import os
from flask import Flask, render_template

def create_app():
    app = Flask(__name__)

    # All config from environment variables — no hardcoded secrets.
    app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY', 'dev-only-key')
    app.config['SQLALCHEMY_DATABASE_URI'] = os.environ.get(
        'DATABASE_URL', 'sqlite:///dev.db'
    )
    app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
    app.config['DEBUG'] = False  # Never True in production.

    @app.route('/')
    def index():
        return render_template('index.html', env=os.environ.get('APP_ENV', 'dev'))

    @app.route('/health')
    def health():
        return {'status': 'ok'}, 200

    return app

app = create_app()

if __name__ == '__main__':
    # Only used for local development.
    app.run(host='127.0.0.1', port=5000, debug=True)
```

`templates/index.html`:

```html
<!DOCTYPE html>
<html>
<head><title>MyApp</title></head>
<body>
  <h1>Hello from MyApp</h1>
  <p>Running in <strong>{{ env }}</strong> environment.</p>
</body>
</html>
```

`requirements.txt`:

```
Flask==3.0.3
gunicorn==22.0.0
psycopg2-binary==2.9.9
python-dotenv==1.0.1
Flask-SQLAlchemy==3.1.1
```

`.env.example`:

```
# Copy this file to .env and fill in real values.
# NEVER commit .env to git.

APP_ENV=production
SECRET_KEY=replace-with-a-long-random-string
DATABASE_URL=postgresql://flask:flaskpass@localhost/myapp
```

`.gitignore`:

```
.env
__pycache__/
*.pyc
instance/
dev.db
uploads/
```

### 4.2 Running with Gunicorn

Install dependencies and Gunicorn:

```bash
pip install -r requirements.txt
pip install gunicorn
```

Set environment variables (load from `.env` later):

```bash
export APP_ENV=production
export SECRET_KEY=$(python -c "import secrets; print(secrets.token_hex(32))")
export DATABASE_URL=postgresql://flask:flaskpass@localhost/myapp
```

Start Gunicorn:

```bash
gunicorn -w 4 -b 0.0.0.0:8000 app:app
```

You'll see:

```
[2024-08-05 14:00:00 +0000] [1234] [INFO] Starting gunicorn 22.0.0
[2024-08-05 14:00:00 +0000] [1234] [INFO] Listening at: http://0.0.0.0:8000 (1234)
[2024-08-05 14:00:00 +0000] [1234] [INFO] Using worker: sync
[2024-08-05 14:00:00 +0000] [1235] [INFO] Booting worker with pid: 1235
[2024-08-05 14:00:00 +0000] [1236] [INFO] Booting worker with pid: 1236
[2024-08-05 14:00:00 +0000] [1237] [INFO] Booting worker with pid: 1237
[2024-08-05 14:00:00 +0000] [1238] [INFO] Booting worker with pid: 1238
```

Test it:

```bash
curl http://localhost:8000/
# <html>...Hello from MyApp...Running in production environment...</html>

curl http://localhost:8000/health
# {"status":"ok"}
```

### 4.3 Loading `.env` with python-dotenv

Add to the top of `app.py` (before `create_app()`):

```python
from dotenv import load_dotenv
load_dotenv()  # reads .env if present, ignores otherwise
```

Now you can keep `.env` in the project folder and `python-dotenv` will load it. **`.env` stays out of git** (it's in `.gitignore`).

### 4.4 The Nginx reverse proxy

On Ubuntu/Debian:

```bash
sudo apt install nginx
sudo nano /etc/nginx/sites-available/myapp
```

Paste this config:

```nginx
server {
    listen 80;
    server_name myapp.example.com;

    # Redirect everything to HTTPS (after you set up TLS)
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.example.com;

    # TLS certificates (use certbot to generate — see Section 8)
    ssl_certificate     /etc/letsencrypt/live/myapp.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.example.com/privkey.pem;

    # Serve static files directly from Nginx — never let Python do this.
    location /static/ {
        alias /home/deploy/myapp/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Everything else goes to Gunicorn.
    location / {
        proxy_pass         http://127.0.0.1:8000;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_redirect     off;
    }
}
```

Enable it:

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t       # test config
sudo systemctl reload nginx
```

### 4.5 Keeping Gunicorn alive with systemd

`/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=MyApp Gunicorn service
After=network.target

[Service]
User=deploy
Group=www-data
WorkingDirectory=/home/deploy/myapp
EnvironmentFile=/home/deploy/myapp/.env
ExecStart=/home/deploy/myapp/venv/bin/gunicorn \
          -w 4 -b 127.0.0.1:8000 \
          --access-logfile - --error-logfile - \
          app:app
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl enable myapp
sudo systemctl start myapp
sudo systemctl status myapp
```

Now Gunicorn starts on boot, restarts on crash, and logs to journald. SSH disconnects no longer kill it.

### 4.6 The Dockerfile

`Dockerfile`:

```dockerfile
# Stage 1: build dependencies in a slim Python base image.
FROM python:3.11-slim AS base

# Don't write .pyc files; buffers go straight to stdout for logs.
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

# Install OS-level deps (psycopg2 needs libpq).
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first so Docker caches this layer.
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the app.
COPY . .

# Run as a non-root user for safety.
RUN useradd -m flaskuser
USER flaskuser

# Gunicorn is the entrypoint.
EXPOSE 8000
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8000", "--access-logfile", "-", "--error-logfile", "-", "app:app"]
```

Build and run:

```bash
docker build -t myapp .
docker run --rm -p 8000:8000 \
    -e APP_ENV=production \
    -e SECRET_KEY=$(python -c "import secrets; print(secrets.token_hex(32))") \
    -e DATABASE_URL=postgresql://flask:flaskpass@host.docker.internal/myapp \
    myapp
```

You'll see Gunicorn boot inside the container and start listening on `0.0.0.0:8000`. From the host:

```bash
curl http://localhost:8000/health
# {"status":"ok"}
```

### 4.7 The `docker-compose.yml`

`docker-compose.yml`:

```yaml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: flask
      POSTGRES_PASSWORD: flaskpass
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U flask -d myapp"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - APP_ENV=production
      - SECRET_KEY=${SECRET_KEY:-dev-insecure-key}
      - DATABASE_URL=postgresql://flask:flaskpass@db:5432/myapp
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

volumes:
  pgdata:
```

Start everything:

```bash
export SECRET_KEY=$(python -c "import secrets; print(secrets.token_hex(32))")
docker compose up --build
```

You'll see Postgres, Redis, and Gunicorn all boot, in the right order:

```
[+] Running 4/4
 ✔ Network myapp_default        Created
 ✔ Volume "myapp_pgdata"        Created
 ✔ Container myapp-db-1         Healthy
 ✔ Container myapp-redis-1      Started
 ✔ Container myapp-web-1        Started
Attaching to db-1, redis-1, web-1
db-1     | The files belonging to this database system belong to user 'postgres'...
redis-1  | Ready to accept connections tcp
web-1    | [2024-08-05 14:00:00 +0000] [1] [INFO] Starting gunicorn 22.0.0
web-1    | [2024-08-05 14:00:00 +0000] [1] [INFO] Listening at: http://0.0.0.0:8000
web-1    | [2024-08-05 14:00:00 +0000] [1] [INFO] Using worker: sync
web-1    | [2024-08-05 14:00:00 +0000] [7] [INFO] Booting worker with pid: 7
...
```

### 4.8 Running migrations in Docker

If you have Flask-Migrate, run migrations in a one-shot container:

```bash
docker compose exec web flask --app app db upgrade
```

Or, better, run it as an entrypoint script that runs before Gunicorn starts:

```dockerfile
# ... (rest of Dockerfile)
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8000", "app:app"]
```

`entrypoint.sh`:

```bash
#!/bin/sh
set -e
# Run migrations before starting the server.
flask --app app db upgrade
exec "$@"
```

Now every container start runs migrations automatically.

## 5. Line-by-Line Explanation

### 5.1 The `create_app()` factory pattern

```python
def create_app():
    app = Flask(__name__)
    app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY', 'dev-only-key')
    # ...
    return app

app = create_app()
```

- Wrapping setup in `create_app()` is the **application factory** pattern. It lets you create multiple instances (useful for testing) and delay configuration until you know the environment.
- `os.environ.get('SECRET_KEY', 'dev-only-key')` — read from env, fall back to `'dev-only-key'`. The fallback is for local dev only; production must set the real value.
- `app = create_app()` at module level — Gunicorn needs `app` to exist when it imports `app.py`. (You can also expose it via `wsgi.py` if you prefer.)

### 5.2 Why `app.config['DEBUG'] = False`

```python
app.config['DEBUG'] = False
```

- In debug mode, Flask shows the Werkzeug debugger on any unhandled exception. That debugger has a built-in Python REPL accessible from the browser. If `PIN protection` is on (it is by default), the PIN is printed to *your* console — but on a server, attackers can read logs.
- Debug mode also re-starts the server on file changes (great for dev, deadly for prod — every deploy would cause uncontrolled restarts).
- **Always** set `DEBUG=False` in production. Rely on `FLASK_ENV` or environment-based config if you want debug behavior locally.

### 5.3 The `/health` endpoint

```python
@app.route('/health')
def health():
    return {'status': 'ok'}, 200
```

- A `/health` (or `/healthz`) endpoint is what load balancers and orchestrators (Kubernetes, AWS ALB, Docker Swarm) probe to know if your app is alive.
- Keep it cheap — no database queries. Return 200 if the process is up; let separate `/ready` endpoints check dependencies (DB, Redis).
- Many platforms require this. Render, Railway, Fly.io all default to checking `/health`.

### 5.4 Gunicorn flags explained

```bash
gunicorn -w 4 -b 0.0.0.0:8000 app:app
```

- `-w 4` — **4 worker processes**. The rule of thumb is `2 × CPU_cores + 1`. On a 2-core machine, 5 workers. On a 4-core machine, 9.
- `-b 0.0.0.0:8000` — bind to all interfaces, port 8000. Behind Nginx, you'd bind to `127.0.0.1:8000` instead so only Nginx can reach Gunicorn.
- `app:app` — `module:variable`. Import the `app` module (your `app.py`), get the `app` attribute (the Flask instance).

Other useful flags:

| Flag | What it does |
|---|---|
| `--workers 4` (same as `-w 4`) | Worker count |
| `--bind 0.0.0.0:8000` (same as `-b`) | Bind address |
| `--access-logfile -` | Log access to stdout (good for Docker/journald) |
| `--error-logfile -` | Log errors to stdout |
| `--timeout 30` | Kill workers stuck longer than 30 s |
| `--graceful-timeout 30` | Time given to in-flight requests during reload |
| `--preload` | Load app once before forking — saves memory |
| `--max-requests 1000 --max-requests-jitter 100` | Restart workers periodically to fight memory leaks |

### 5.5 The worker model

Gunicorn's default "sync" worker is a **pre-fork** model:

1. The master process loads your Flask app.
2. It forks N times, creating N worker processes.
3. Each worker handles one request at a time.
4. The master load-balances by accepting connections and handing them to workers.

With 4 workers, you handle 4 simultaneous requests. If one crashes, the master restarts it. This is why a single Python exception doesn't take down your whole site.

```mermaid
flowchart TB
    M[Gunicorn Master<br/>pid 1] --> W1[Worker pid 7]
    M --> W2[Worker pid 8]
    M --> W3[Worker pid 9]
    M --> W4[Worker pid 10]
    W1 --> F1[Flask app:app<br/>imported in master, forked]
    W2 --> F2[Flask app:app]
    W3 --> F3[Flask app:app]
    W4 --> F4[Flask app:app]
    N[Nginx] -->|one request each| M
    style M fill:#dfd,stroke:#070
    style W1 fill:#ddf,stroke:#008
    style W2 fill:#ddf,stroke:#008
    style W3 fill:#ddf,stroke:#008
    style W4 fill:#ddf,stroke:#008
```

### 5.6 Environment variables vs. `.env`

```python
from dotenv import load_dotenv
load_dotenv()
app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY', 'dev-only-key')
```

- `load_dotenv()` — opens `.env` if it exists and sets the variables into `os.environ`. If `.env` doesn't exist, it does nothing (great for production platforms where env vars come from the orchestrator).
- `.env` — a file in your project root. Lines like `KEY=value`. **Never committed to git.**
- In production, you typically *don't* use `.env` at all — you set env vars in the platform (Render dashboard, AWS task definition, systemd `EnvironmentFile=`). `load_dotenv()` just doesn't find a file and returns.

> [!warning] Don't confuse `.env` with config
> `.env` is for **secrets and environment-specific settings** — `SECRET_KEY`, `DATABASE_URL`, API keys. It is NOT a place for app behavior flags. Behavior flags go in `app.config` and are committed to git.

### 5.7 Nginx configuration breakdown

```nginx
location /static/ {
    alias /home/deploy/myapp/static/;
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

- `location /static/` — Nginx intercepts any URL starting with `/static/`.
- `alias ...` — Nginx serves files from disk directly. **Python never sees these requests.** This is dramatically faster than `send_from_directory`.
- `expires 30d` — tells the browser to cache for 30 days. Repeat visitors don't re-download your CSS.

```nginx
location / {
    proxy_pass         http://127.0.0.1:8000;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
}
```

- `proxy_pass http://127.0.0.1:8000` — forward to Gunicorn on localhost.
- `proxy_set_header Host $host` — preserve the original Host header so Flask sees `myapp.example.com`, not `127.0.0.1:8000`.
- `X-Forwarded-For` and `X-Real-IP` — pass the user's real IP. Without these, Flask sees every request as coming from `127.0.0.1` (Nginx).
- `X-Forwarded-Proto $scheme` — tells Flask whether the original request was HTTPS. Flask uses this for `request.is_secure` and to generate correct `url_for` URLs.

> [!tip] Use `ProxyFix` middleware
> For Flask to actually trust those headers, wrap your app:
> ```python
> from werkzeug.middleware.proxy_fix import ProxyFix
> app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1, x_host=1)
> ```
> Without this, `request.remote_addr` returns Nginx's IP, not the user's.

### 5.8 The Dockerfile, line by line

```dockerfile
FROM python:3.11-slim AS base
```
- `python:3.11-slim` — official Python image, slim variant (~150 MB vs ~900 MB for full). Smaller images are faster to pull and have fewer security vulnerabilities.

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
```
- `PYTHONDONTWRITEBYTECODE=1` — don't write `.pyc` files. Saves space.
- `PYTHONUNBUFFERED=1` — don't buffer stdout/stderr. Lets you see logs in `docker logs` immediately.

```dockerfile
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```
- `WORKDIR /app` — like `cd /app`, creates it if missing.
- **The two-step COPY is critical.** Copying `requirements.txt` first and running `pip install` before copying the rest of the code means Docker **caches the pip layer**. Change a Python file? Only the final `COPY . .` re-runs. Rebuilds go from 60 seconds to 3 seconds.
- `--no-cache-dir` — don't cache pip downloads inside the image.

```dockerfile
RUN useradd -m flaskuser
USER flaskuser
```
- Containers run as **root by default**. If an attacker breaks your app, they're root inside the container. Creating and switching to a non-root user is a basic safety measure.

```dockerfile
EXPOSE 8000
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8000", "app:app"]
```
- `EXPOSE 8000` — documentation only. Doesn't actually publish the port; `docker run -p 8000:8000` does.
- `CMD ["gunicorn", ...]` — the exec form (a list). Preferred over the shell form (`CMD gunicorn ...`) because it makes Gunicorn PID 1, which handles signals correctly (SIGTERM for clean shutdown).

### 5.9 docker-compose healthcheck

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U flask -d myapp"]
  interval: 5s
  timeout: 5s
  retries: 5
```
- Compose polls the database with `pg_isready` every 5 seconds. The `web` service's `depends_on: condition: service_healthy` makes the web container wait until Postgres is *actually ready to accept connections*, not just started.
- Without this, Flask starts before Postgres is up, the first DB query fails, and your boot crashes. This is the #1 docker-compose bug.

### 5.10 The `entrypoint.sh` pattern

```bash
#!/bin/sh
set -e
flask --app app db upgrade
exec "$@"
```
- `set -e` — exit on any error. If migrations fail, the container exits instead of starting with a broken DB.
- `flask --app app db upgrade` — runs all pending Alembic migrations.
- `exec "$@"` — replace the shell with the CMD (Gunicorn). Without `exec`, Gunicorn runs as a child of the shell, which breaks signal handling.

## 6. Running the Code

### 6.1 Local production test with Docker

```bash
cd myapp
docker compose up --build
```

In another terminal:

```bash
curl http://localhost:8000/health
# {"status":"ok"}

curl http://localhost:8000/
# <!DOCTYPE html>...Running in production environment...

docker compose exec db psql -U flask -d myapp -c '\dt'
# List of relations (empty for now — no models)
```

Stop everything:

```bash
docker compose down         # stop containers, keep data
docker compose down -v      # stop containers, delete the Postgres volume too
```

### 6.2 Deploying to a real VPS (DigitalOcean / Linode / Hetzner)

1. **Spin up** an Ubuntu 22.04 droplet. SSH in.
2. **Install deps**: `sudo apt update && sudo apt install -y nginx postgresql python3-venv certbot python3-certbot-nginx`.
3. **Create a deploy user**: `sudo adduser deploy && sudo usermod -aG sudo deploy`.
4. **Clone your repo** into `/home/deploy/myapp`.
5. **Create venv**: `python3 -m venv venv && source venv/bin/activate && pip install -r requirements.txt gunicorn`.
6. **Copy `.env.example` to `.env`** and fill in real values. Generate `SECRET_KEY` with `python -c "import secrets; print(secrets.token_hex(32))"`.
7. **Create the database** in Postgres: `sudo -u postgres createuser flask -P && sudo -u postgres createdb myapp -O flask`.
8. **Run migrations**: `flask --app app db upgrade`.
9. **Install the systemd unit** (Section 4.5). `sudo systemctl enable --now myapp`.
10. **Install the Nginx config** (Section 4.4). `sudo ln -s ... && sudo nginx -t && sudo systemctl reload nginx`.
11. **Get TLS**: `sudo certbot --nginx -d myapp.example.com`. Certbot edits your Nginx config to add HTTPS and sets up auto-renewal.
12. **Visit** `https://myapp.example.com`.

### 6.3 Deploying to Render

1. Push your repo to GitHub.
2. On Render: **New → Web Service**. Connect the repo.
3. Set:
   - **Build command**: `pip install -r requirements.txt`
   - **Start command**: `gunicorn -w 4 -b 0.0.0.0:$PORT app:app`
   - **Environment variables**: `SECRET_KEY`, `DATABASE_URL` (Render provides this when you add a Postgres database).
4. Render builds, deploys, gives you a URL like `https://myapp.onrender.com`. Free tier sleeps after inactivity; paid tier starts at $7/month.

### 6.4 Deploying to Heroku

1. Install the [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli).
2. `heroku create my-app-name`.
3. Add a `Procfile`:

   ```
   web: gunicorn -w 4 -b 0.0.0.0:$PORT app:app
   ```
4. `heroku addons:create heroku-postgresql:hobby-dev`.
5. `git push heroku main`.
6. `heroku run flask --app app db upgrade` (or add it to a `release` phase in `Procfile`).
7. Visit `https://my-app-name.herokuapp.com`.

## 7. Common Mistakes

### 7.1 Running with `app.run(debug=True)` in production

```bash
python app.py  # NO — this is the dev server
```

Symptoms: slow, dies on SSH disconnect, exposes your code on errors.

**Fix:** always Gunicorn. Use `app.run()` only inside `if __name__ == '__main__':` for local dev.

### 7.2 Forgetting to set `SECRET_KEY`

```python
app.config['SECRET_KEY'] = 'dev'  # commit this and you've leaked sessions
```

Anyone reading your repo can forge session cookies — sign in as any user, including admin.

**Fix:** `os.environ.get('SECRET_KEY')`. Generate with `secrets.token_hex(32)`. Rotate if it leaks.

### 7.3 Using SQLite in production

```python
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///prod.db'
```

SQLite is a *file*. Concurrent writes lock the file. Two users sign up at the same time → one gets "database is locked." It also doesn't survive a server restart if the file is in a non-persistent location (common on Heroku/Render dynos).

**Fix:** PostgreSQL or MySQL in production. Both handle concurrent writes properly.

### 7.4 Binding Gunicorn to 0.0.0.0 without Nginx

```bash
gunicorn -b 0.0.0.0:8000 app:app  # exposed directly to the internet
```

You can — but you're now serving HTTPS terminations, static files, and DoS protection from Python. Gunicorn isn't designed for that.

**Fix:** bind to `127.0.0.1:8000`, put Nginx in front. Nginx handles TLS, static files, slow clients, and rate limiting.

### 7.5 Not running migrations

You deploy new code with a new model. Flask-SQLAlchemy `db.create_all()` doesn't update existing tables. Your site crashes on first query.

**Fix:** use Flask-Migrate. Run `flask --app app db upgrade` as part of every deploy.

### 7.6 Committing `.env`

```bash
git add .env && git commit -m "config"
git push   # now on GitHub forever
```

Even after `git rm .env`, it's still in history.

**Fix:** add `.env` to `.gitignore` *first*. If you've already committed it, use `git filter-repo` or BFG to scrub history, then rotate every secret.

### 7.7 No `/health` endpoint

Load balancers can't tell if your app is alive. They keep sending traffic to a dead container.

**Fix:** always have a cheap `/health` returning 200.

### 7.8 Running as root in Docker

An attacker who finds an RCE in your app is root in the container. From there, container escape is sometimes possible.

**Fix:** add a non-root user and `USER` directive (Section 5.8).

### 7.9 No logging in production

`print()` statements disappear. When something breaks, you have nothing.

**Fix:** use Python's `logging` module, configure it to send to stdout, and let Docker / journald capture it. Set up structured logging (JSON) if you'll feed logs to Datadog, Loki, or CloudWatch.

### 7.10 No error monitoring

A user hits a 500. They don't tell you. You don't know.

**Fix:** install Sentry (`pip install sentry-sdk[flask]`). Errors stream to a dashboard with full stack traces, request context, and user info. There's a free tier.

```python
import sentry_sdk
from sentry_sdk.integrations.flask import FlaskIntegration

sentry_sdk.init(
    dsn=os.environ['SENTRY_DSN'],
    integrations=[FlaskIntegration()],
    traces_sample_rate=0.1,
)
```

### 7.11 Picking the wrong worker count

Too few → underutilized CPU. Too many → context-switching overhead and memory pressure.

**Fix:** start with `(2 × CPU_cores + 1)`. Load-test with `wrk` or `ab`. Adjust based on real numbers.

### 7.12 Not setting `--timeout`

A slow request hangs a worker forever. With 4 workers and 4 slow requests, your site is down.

**Fix:** `--timeout 30` (or 60 for slow endpoints). Workers are killed and restarted if they exceed it.

## 8. The Production Checklist

Before you launch, verify each item:

- [x] **`DEBUG=False`** in production config.
- [x] **Strong `SECRET_KEY`** — 32+ random bytes, from env var.
- [x] **Production database** — Postgres, not SQLite.
- [x] **Environment variables for secrets** — no credentials in code or git.
- [x] **`.env` in `.gitignore`** — never commit it.
- [x] **Gunicorn with multiple workers** — `2 × CPU + 1`.
- [x] **Nginx (or platform proxy) in front** — for TLS, static, buffering.
- [x] **HTTPS via Let's Encrypt** — `certbot --nginx -d myapp.example.com`.
- [x] **HTTP → HTTPS redirect** — no plaintext traffic.
- [x] **`/health` endpoint** — for load balancer checks.
- [x] **Migrations run on deploy** — `flask db upgrade` in entrypoint or release phase.
- [x] **Logging to stdout** — captured by Docker/journald.
- [x] **Error monitoring** — Sentry or equivalent.
- [x] **Backups** — automated daily DB dumps, tested restore.
- [x] **Restart on crash** — systemd `Restart=always` or Docker `restart: unless-stopped`.
- [x] **Resource limits** — Docker memory/CPU limits, or platform plan sized to traffic.
- [x] **Security headers** — `Content-Security-Policy`, `X-Content-Type-Options`, `X-Frame-Options` (via `flask-talisman`).
- [x] **Rate limiting** — `flask-limiter` on login/reset endpoints.
- [x] **CSRF protection** — `flask-wtf` on every form.
- [x] **`allowed_hosts`** equivalent — Nginx `server_name` restricts which Host headers reach Flask.

## 9. Platform Comparison

Where to deploy? It depends on your time, budget, and learning goals.

| Platform | Pros | Cons | Best for |
|---|---|---|---|
| **Heroku** | Easiest. One-click Postgres, Redis, add-ons. Great CLI. | Expensive at scale ($7/dyno minimum, $50 for Postgres). US/EU regions only. | Beginners, prototypes, small apps. |
| **Render** | Heroku-like simplicity, cheaper ($7/web), free tier (sleeps), Docker-native. | Smaller add-on ecosystem than Heroku. | Same audience as Heroku, but cheaper. |
| **Railway** | Beautiful UI, git-push deploys, generous free trial, supports Docker. | Newer, smaller community. | Hobby projects, solo developers. |
| **Fly.io** | Deploy close to users (multi-region), Docker-native, generous free tier. | More setup than Render; requires `fly.toml`. | Apps needing low latency worldwide. |
| **DigitalOcean Droplet / Hetzner / Linode** | Cheapest at scale ($5–10/mo VPS), full control, learn real Linux. | You do everything: Nginx, TLS, updates, security. | Students, self-hosters, learning deployment. |
| **AWS (EC2 / ECS / Lambda)** | Infinite scale, every service you'll ever need. | Steep learning curve, surprise bills, 10-page IAM policies. | Enterprise, scalable SaaS, resume-building. |
| **Google Cloud Run** | Serverless containers, scale to zero, pay per request. | Cold starts, vendor lock-in, complex networking. | Sporadic traffic, containerized apps. |

> [!tip] My recommendation for this course
> Deploy once to **Render** (free tier, zero setup) to see your app on the internet. Then deploy to a **DigitalOcean droplet** to learn the Nginx + systemd + certbot dance. That combination gives you both the "easy path" and the "I actually understand what's happening" path.

```mermaid
flowchart TD
    Q[What should I deploy to?]
    Q --> B{Budget?}
    B -->|Free / learning| R[Render or Fly.io free tier]
    B -->|5-10 / month, want to learn Linux| DO[DigitalOcean / Hetzner VPS]
    B -->|5-10 / month, want zero ops| RR[Railway or Render paid]
    B -->|Enterprise / SaaS at scale| AWS[AWS or GCP]
    Q --> T{Traffic pattern?}
    T -->|Always-on, steady| VPS[VPS or always-on PaaS]
    T -->|Bursty, occasional| CR[Cloud Run / Lambda — scale to zero]
    T -->|Global, latency-sensitive| FY[Fly.io multi-region]
```

## 10. Modern WSGI Servers — Granian, uvicorn, and the 2026 landscape

Gunicorn is the default, but it's not the only option. Newer servers bring measurable improvements for specific workloads.

### 10.1 Granian — a Rust-based WSGI/ASGI server

[Granian](https://github.com/emmett-sh/granian) (first released 2023, popular by 2026) is a WSGI/ASGI server written in Rust. It calls your Python app from Rust via PyO3, which gives it two advantages over Gunicorn:

- **Rust HTTP parser** — about 2-3× faster than Gunicorn's pure-Python parser for the request handling itself.
- **Async runtime (tokio)** — supports both WSGI (sync Flask) and ASGI (async) on the same binary, with `--interface asgi|wsgi|rsGI`.

```bash
pip install granian
granian --interface wsgi --workers 4 app:app
```

Granian is a drop-in replacement for Gunicorn in most cases. The flags are different (`--workers` instead of `-w`, `--interface` instead of worker class), but the model is the same: a master process forks N workers, each worker handles requests.

| Server | Best for | Notes |
|---|---|---|
| **Gunicorn** | Most Flask apps. Battle-tested. | Default choice; sync worker is simple. |
| **Granian** | High-throughput Flask on modern hardware. | Rust HTTP parser; faster for IO-bound workloads. |
| **uWSGI** | Legacy / very custom setups. | Powerful but config is famously arcane. |
| **uvicorn** | ASGI apps (FastAPI, Starlette, async Flask). | Pair with `--workers N` for production. |
| **Hypercorn** | ASGI with HTTP/2 or HTTP/3. | Used by Quart. |

> [!question] Student asks: "Should I switch from Gunicorn to Granian?"
> Measure first. For most Flask apps, the bottleneck is the database, the network, or your view code — not the WSGI server. Granian shines when you have **very high request rates with minimal Python work per request** (e.g. a thin API that proxies to a microservice). For a typical CRUD app where each request hits the database, the difference between Gunicorn and Granian is in the noise. If you're curious, run `wrk -t4 -c100 -d30s http://localhost:8000/health` against both and compare.

### 10.2 Worker types — sync vs gevent vs gthread

Gunicorn ships with several worker types, each suited to different workloads:

```bash
gunicorn -w 4 -k sync app:app          # default: one request per worker at a time
gunicorn -w 4 -k gthread --threads 4 app:app   # threads per worker
gunicorn -w 4 -k gevent app:app        # greenlets: 1000s of concurrent connections
```

- **`sync`** — one request at a time per worker. Simple, predictable. Best for CPU-bound or standard request/response apps.
- **`gthread`** — each worker runs N threads. Good when your app uses third-party libraries that block (sync HTTP calls, slow DB drivers) and you want some concurrency without changing your code.
- **`gevent`** — uses greenlets (cooperative coroutines) to handle hundreds of concurrent connections per worker via monkey-patching. Great for thousands of long-polling or WebSocket-style connections. Requires that all your libraries play nice with gevent (most do; some don't).

```mermaid
flowchart TD
    Choose["Choosing a Gunicorn worker type"] --> Q1{"CPU-bound or simple I/O?"}
    Q1 -->|"Yes"| Sync["sync (default)"]
    Q1 -->|"Mostly I/O"| Q2{"Many slow/long-poll connections?"}
    Q2 -->|"Yes, 100s-1000s"| Gevent["gevent<br/>(greenlets)"]
    Q2 -->|"Moderate concurrency"| Gthread["gthread<br/>(threaded)"]
    Q2 -->|"Need real OS threads / async libs"| Uvicorn["uvicorn worker<br/>(ASGI)"]
```

> [!warning] gevent monkey-patches the standard library
> `gevent` replaces `socket`, `threading`, `time.sleep`, and more with greenlet-friendly versions. If your app uses a library that holds a real OS thread (some database drivers, native code), gevent can deadlock it. Test thoroughly before deploying gevent in production.

## 11. Modern Docker — uv, distroless, multi-stage builds

### 11.1 `uv` for 10-100× faster pip installs

[uv](https://github.com/astral-sh/uv) (released 2024 by Astral, the same people behind Ruff) is a Rust-based replacement for `pip` and `virtualenv`. For a `requirements.txt` with 50 packages, `uv pip install` finishes in seconds where `pip install` takes minutes.

```dockerfile
# Install uv in your image:
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Then use uv instead of pip:
RUN uv pip install --system -r requirements.txt
```

For a typical Flask app, this drops the Docker build from ~90 seconds to ~5 seconds for the dependency layer — and that's not a one-time saving; it's **every rebuild** during development.

```dockerfile
# Modern Dockerfile using uv + multi-stage build
FROM python:3.12-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev     # install only production deps into .venv

FROM python:3.12-slim AS runtime
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8000", "app:app"]
```

> [!info] Why a "lock file"?
> `requirements.txt` lists packages by name and version range. `uv.lock` (like `poetry.lock` or `package-lock.json`) records the **exact** resolved versions of every transitive dependency. Reproducible builds, no "works on my machine" drift. For any non-trivial app, a lock file is mandatory.

### 11.2 Distroless images for security

A standard `python:3.12-slim` image includes Debian, apt, shell, common Linux utilities, and ~80 MB of stuff an attacker could use if they broke in. **Distroless** images from Google strip all of that — no shell, no package manager, no utilities, just Python and your code.

```dockerfile
FROM python:3.12-slim AS builder
# ... build venv with uv ...
COPY . /app

FROM gcr.io/distroless/python3-debian12:nonroot
COPY --from=builder /app /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /site-packages
ENV PYTHONPATH=/site-packages
WORKDIR /app
USER nonroot
CMD ["-m", "gunicorn", "-w", "4", "-b", "0.0.0.0:8000", "app:app"]
```

If an attacker finds an RCE in your Flask app, they land in a container with no shell, no `apt`, no `curl`, no `wget`, no `python -c`. They can still exfiltrate the database by hitting your endpoints, but they can't easily escalate to lateral movement. Defense in depth.

> [!question] Student asks: "If there's no shell, how do I debug?"
> `kubectl exec -it pod -- /bin/sh` won't work — there's no sh. Use `kubectl debug` (which adds a sidecar container with a shell) or `kubectl logs`. Trade-off: you lose debugging convenience for security. Most teams run a normal slim image in dev and a distroless image in production.

### 11.3 `.dockerignore` — keep your image lean

Without `.dockerignore`, `COPY . .` ships your `.git/` (often hundreds of MB), your venv, your tests, your `.env` (containing real secrets!), and any random file in the project root into the image.

```gitignore
# .dockerignore
.git
.gitignore
venv/
.venv/
__pycache__/
*.pyc
.pytest_cache/
.env
.env.*
*.md
tests/
.github/
Dockerfile*
docker-compose*.yml
node_modules/
*.sqlite3
*.db
```

> [!danger] The .env-in-Docker-image footgun
> If `.env` is not in `.dockerignore`, your secrets are baked into the image. Anyone who pulls the image — including from a public registry by mistake — has your `SECRET_KEY`, database password, and SMTP credentials. Always add `.env` to `.dockerignore` *first*, then write your Dockerfile.

## 12. Caddy — auto-HTTPS without certbot

Nginx + certbot works, but it's two pieces to configure and maintain. [Caddy](https://caddyserver.com/) is a modern web server (written in Go) that **automatically** obtains and renews Let's Encrypt certificates. You point it at a domain and HTTPS just works.

```bash
sudo apt install caddy
```

`/etc/caddy/Caddyfile`:

```
myapp.example.com {
    encode gzip
    handle /static/* {
        root * /home/deploy/myapp/static
        file_server
    }
    reverse_proxy 127.0.0.1:8000
}
```

```bash
sudo systemctl reload caddy
```

That's it. Caddy starts, sees `myapp.example.com`, reaches out to Let's Encrypt, gets a cert, serves HTTPS on 443, and renews it automatically 30 days before expiry. No `certbot --nginx`, no `cron`, no certificate files to manage.

```mermaid
flowchart LR
    Browser -->|HTTPS| Caddy["Caddy<br/>:443"]
    Caddy -->|"first boot"| LE["Let's Encrypt<br/>ACME API"]
    LE -->|cert| Caddy
    Caddy -->|HTTP| Gunicorn["Gunicorn<br/>:8000"]
    Caddy -->|static files| Disk["/home/deploy/myapp/static"]
    style Caddy fill:#dfd,stroke:#070
```

> [!tip] When to pick Caddy vs Nginx
> **Caddy**: new project, you value simplicity, you want auto-HTTPS, you don't need esoteric Nginx modules. **Nginx**: existing infrastructure, you need specific Nginx modules (lua, fancy redirects), you have Nginx expertise on the team. Both are excellent; Caddy is the modern default for new deployments.

## 13. Database in Production — PostgreSQL and PgBouncer

### 13.1 Why Postgres, not SQLite

SQLite is a single file. That's fine for development, but in production:

- **One writer at a time.** SQLite locks the whole file on write. Two users registering simultaneously → "database is locked".
- **No network access.** SQLite is in-process. You can't have a separate database server.
- **No replication.** SQLite doesn't natively support read replicas or failover.
- **No connection pooling.** Each connection is a file handle.

PostgreSQL is a real database server: it accepts TCP connections, supports hundreds of concurrent writes, has streaming replication, full-text search, JSON columns, and 30 years of optimization.

### 13.2 Connection pooling with PgBouncer

Postgres forks a process per connection. Each connection costs ~10 MB of RAM on the server. A Flask app with 4 Gunicorn workers, each keeping 5 connections open, is 20 connections — fine. But add Celery workers, a separate API service, monitoring tools, and you're at 100+ connections, eating 1 GB of RAM just for idle connections.

**PgBouncer** sits between your app and Postgres. It maintains a small pool of real Postgres connections (say, 20) and multiplexes hundreds of client connections through them. Your app thinks it's connected to Postgres; PgBouncer routes queries to the actual database.

```mermaid
flowchart LR
    subgraph App["Your app"]
        W1["Gunicorn worker 1<br/>5 connections"]
        W2["Gunicorn worker 2<br/>5 connections"]
        W3["Celery worker<br/>10 connections"]
    end
    subgraph Pool["PgBouncer<br/>(transaction mode)"]
        P1["conn 1"]
        P2["conn 2"]
        P3["..."]
        P10["conn 10"]
    end
    App -->|"40 client conns"| Pool
    Pool -->|"10 real conns"| PG[("PostgreSQL<br/>10 server processes")]
    style Pool fill:#dfd,stroke:#070
```

```yaml
# docker-compose.yml with PgBouncer
services:
  pgbouncer:
    image: edoburu/pgbouncer:latest
    environment:
      DB_HOST: db
      DB_USER: flask
      DB_PASSWORD: flaskpass
      DB_NAME: myapp
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 200
      DEFAULT_POOL_SIZE: 10
    ports:
      - "6432:5432"
    depends_on:
      - db

  web:
    environment:
      - DATABASE_URL=postgresql://flask:flaskpass@pgbouncer:5432/myapp
    depends_on:
      - pgbouncer
```

> [!info] Transaction mode vs session mode
> `POOL_MODE=transaction` (default for web apps) checks out a real connection per transaction. Your app's idle time between requests doesn't hold a Postgres connection. Trade-off: session-level features like `LISTEN/NOTIFY`, server-side cursors, and temporary tables don't survive across transactions. For most Flask apps, transaction mode is what you want.

## 14. Static Files in Production — CDN, Cloudflare, S3

Flask's dev server serves `/static/` from Python. In production that's 100× too slow. The standard pattern is:

1. **Cache headers** — `Cache-Control: public, max-age=31536000, immutable` for fingerprinted assets (CSS/JS with content-hash filenames), so the browser caches them forever.
2. **CDN** — put a CDN (Cloudflare, CloudFront, Fastly) in front of your origin. Users fetch static files from a server near them.
3. **Origin** — either Nginx serves them from disk (simple), or you upload them to S3/R2 and the CDN pulls from there (better for multi-region).

```mermaid
flowchart LR
    User["User in Tokyo"] -->|fetch /static/app.abc123.css| CDN["CDN edge<br/>(Tokyo)"]
    CDN -->|cache hit| User
    CDN -->|cache miss| Origin["Origin<br/>(us-east-1)"]
    Origin -->|Nginx| Disk["/var/www/static"]
    Origin -->|S3| S3[("S3 bucket")]
    style CDN fill:#dfd,stroke:#070
```

For Flask specifically:

- Use [Flask-Assets](https://flask-assets.readthedocs.io/) or a build tool (Vite, esbuild) to produce hashed filenames like `app.abc123.css`.
- Set far-future cache headers.
- Generate the URL with `url_for('static', filename='app.abc123.css')` so each deploy's new hash busts the cache automatically.

> [!tip] The "Cloudflare in front of everything" shortcut
> Cloudflare's free tier sits in front of your origin (DigitalOcean droplet, Render, Heroku) and caches static files at the edge. Zero code changes — just point your domain's DNS at Cloudflare. For most small/medium apps, this is the highest-leverage free optimization you can do.

## 15. Logging and Observability

### 15.1 Structured logging with structlog

`print()` and basic `logging` produce lines like `2026-01-15 12:34:56 INFO user 42 logged in`. A human can read them; a log aggregator (Datadog, Loki, CloudWatch) struggles.

**Structured logging** outputs JSON: `{"timestamp": "...", "level": "info", "event": "login", "user_id": 42, "ip": "1.2.3.4"}`. Every log is a searchable document.

```bash
pip install structlog
```

```python
# app/logging_config.py
import structlog
import logging

def configure_logging(app):
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt='iso'),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        cache_logger_on_first_use=True,
    )
```

```python
# In a route:
import structlog
logger = structlog.get_logger()

@app.route('/login', methods=['POST'])
def login():
    logger.info('login_attempt', username=form.username.data, ip=request.remote_addr)
    # ...
    logger.info('login_success', user_id=user.id)
```

Each log line is a JSON object. In production you ship these to Loki, Datadog, or CloudWatch and query them like SQL.

### 15.2 Sentry — automatic error capture

```bash
pip install sentry-sdk[flask]
```

```python
import sentry_sdk
from sentry_sdk.integrations.flask import FlaskIntegration

sentry_sdk.init(
    dsn=os.environ['SENTRY_DSN'],
    integrations=[FlaskIntegration()],
    traces_sample_rate=0.1,        # 10% of requests get performance traces
    environment=os.environ.get('APP_ENV', 'development'),
    send_default_pii=False,        # don't send IP addresses
)
```

Once initialized, any unhandled exception in a Flask view is captured by Sentry with:

- The full stack trace.
- The request URL, method, headers, and form data (filtered for secrets).
- The user (if logged in — `sentry_sdk.set_user({'id': user.id})`).
- breadcrumbs — the 30 log messages and HTTP calls leading up to the error.

You get an email/Discord/Slack ping minutes after the user sees the 500. This is the difference between "we have no idea why users complain" and "we have a stack trace and a fix deployed in 20 minutes."

### 15.3 OpenTelemetry — distributed tracing

When your app is a single Flask process, a stack trace tells you where time went. When your app is Flask → Postgres → Redis → Celery worker → external API → back, a stack trace per service isn't enough. You need a **trace** — a single view of "this one user request touched these 5 services, spent 80 ms in Postgres, 1.2 s in the external API."

[OpenTelemetry](https://opentelemetry.io/) is the standard for instrumenting this. You add it to Flask, SQLAlchemy, Redis, requests — every library has an integration. Each request gets a trace ID; child spans inherit it. You ship the spans to a collector (Jaeger, Tempo, Honeycomb, Datadog) and visualize the whole request as a waterfall.

```bash
pip install opentelemetry-distro opentelemetry-instrumentation-flask \
            opentelemetry-instrumentation-sqlalchemy opentelemetry-instrumentation-redis
opentelemetry-bootstrap -a install
```

```python
# app/__init__.py
from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint=os.environ['OTEL_ENDPOINT']))
)

def create_app():
    app = Flask(__name__)
    FlaskInstrumentor().instrument_app(app)
    SQLAlchemyInstrumentor().instrument(engine=db.engine)
    # ...
```

```mermaid
flowchart TB
    Browser -->|HTTP| Flask["Flask: 1.8s total"]
    Flask --> DB["Postgres SELECT: 80ms"]
    Flask --> Redis["Redis GET: 5ms"]
    Flask --> API["External API: 1.4s ← bottleneck"]
    Flask --> Worker["Enqueue Celery task: 10ms"]
    style API fill:#fdd,stroke:#900
```

With tracing in place, you open Jaeger, find the slow request, and immediately see "1.4 s in the external API." Without tracing, you'd be guessing.

## 16. Zero-Downtime Deployment

Users hate maintenance pages. Three patterns to deploy without dropping requests:

### 16.1 Blue-green deployment

Run two identical environments — "blue" (current) and "green" (new). Route traffic to blue. Deploy new code to green. Test green. Flip the router. If green breaks, flip back to blue.

```mermaid
flowchart LR
    LB[Load balancer] -->|100% traffic| Blue[Blue — v1.2]
    LB -.->|0% traffic| Green[Green — v1.3 staging]
    Note["After testing green, swap:"]
    LB -->|100% traffic| Green
    LB -.->|0% traffic| Blue
```

- Pros: instant rollback (flip the router).
- Cons: 2× infrastructure cost (you're paying for two environments).

### 16.2 Canary deployment

Roll out the new version to 5% of users, watch metrics, ramp to 25%, 50%, 100%. If error rate spikes at any step, stop and roll back.

```mermaid
flowchart LR
    LB[Load balancer] -->|"95%"| Stable[Stable v1.2]
    LB -->|"5%"| Canary[Canary v1.3]
    Canary --> Metrics{Error rate OK?}
    Metrics -->|Yes| Ramp[Ramp to 25%, 50%, 100%]
    Metrics -->|No| Rollback[Roll back to 0%]
```

- Pros: catches issues that affect only some users (specific geography, browser, data shape).
- Cons: needs sophisticated routing (weighted load balancing) and good metrics.

### 16.3 Rolling deployment

Replace instances one at a time. Kubernetes does this by default with `maxUnavailable: 1` and `maxSurge: 1`. Render, Fly.io, and AWS ECS support it natively.

For Gunicorn specifically, there's a built-in trick: send `SIGHUP` (or `kill -HUP <master_pid>`) and Gunicorn reloads its workers gracefully — old workers finish in-flight requests before being replaced.

```bash
kill -HUP $(cat /var/run/gunicorn.pid)
```

```mermaid
sequenceDiagram
    participant O as Operator
    participant M as Gunicorn master
    participant W1 as Worker 1 (old)
    participant W1n as Worker 1 (new)
    O->>M: SIGHUP
    M->>W1n: Fork new worker
    M->>W1: SIGQUIT (graceful)
    Note over W1: Finish in-flight requests
    W1-->>M: exited cleanly
    M->>M: All workers replaced
```

> [!question] Student asks: "Do I really need zero-downtime?"
> For a hobby project, no — a 30-second maintenance window at 3 AM is fine. For any app with users in multiple time zones, an e-commerce site, or anything paying customers rely on, yes. The cost of zero-downtime infra is real (more complexity, often more cost); the cost of a 2-minute outage during peak hours is often higher (lost revenue, support tickets, churned users).

## 17. CI/CD with GitHub Actions

Manual deploys are error-prone. The fix: every push to `main` triggers a workflow that runs tests, builds the image, pushes it to a registry, and deploys.

`.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']

    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv pip install --system -r requirements.txt pytest
      - run: pytest -v
        env:
          DATABASE_URL: postgresql://postgres:test@localhost/myapp_test
          REDIS_URL: redis://localhost:6379/0
          SECRET_KEY: ci-test-key-not-real

  build-and-deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Log in to registry
        run: echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
      - name: Push image
        run: |
          docker tag myapp:${{ github.sha }} ghcr.io/youruser/myapp:latest
          docker tag myapp:${{ github.sha }} ghcr.io/youruser/myapp:${{ github.sha }}
          docker push ghcr.io/youruser/myapp:latest
          docker push ghcr.io/youruser/myapp:${{ github.sha }}
      - name: Deploy to Fly.io
        run: |
          curl -L https://fly.io/install.sh | sh
          /home/runner/.fly/bin/fly deploy --image ghcr.io/youruser/myapp:${{ github.sha }} --app myapp
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

```mermaid
flowchart TB
    Push["git push main"] --> Trigger["GitHub Actions triggers"]
    Trigger --> Test["Job: test<br/>pytest + Postgres + Redis"]
    Test -->|"all green"| Build["Job: build<br/>docker build"]
    Build --> Push["Push image to registry<br/>with sha tag + latest"]
    Push --> Deploy["Job: deploy<br/>fly deploy --image ..."]
    Deploy --> Live["New version live<br/>with rolling restart"]
    Test -->|"failed"| Stop["Stop. Don't deploy."]
    style Live fill:#dfd,stroke:#070
    style Stop fill:#fdd,stroke:#900
```

> [!tip] The deploy-on-green rule
> Never deploy from a branch whose tests didn't pass. The `if: github.ref == 'refs/heads/main'` guard in the build job ensures only main commits deploy, and the `needs: test` ensures the build doesn't run if tests fail. This single rule eliminates 80% of "I deployed a broken version" incidents.

## 18. Expanded Production Checklist

Before you launch, verify each item:

### Configuration & Secrets
- [x] **`DEBUG=False`** in production config.
- [x] **Strong `SECRET_KEY`** — 32+ random bytes, from env var, rotated periodically.
- [x] **Environment variables for secrets** — no credentials in code or git.
- [x] **`.env` in `.gitignore`** — never commit it.
- [x] **`SENTRY_DSN`, `DATABASE_URL`, SMTP creds** all from env vars.
- [x] **Different secrets in dev/staging/prod** — dev keys must not work in prod.

### Database
- [x] **Production database** — Postgres, not SQLite.
- [x] **PgBouncer** in front of Postgres if you have many connections.
- [x] **Automated daily backups** with tested restore.
- [x] **Point-in-time recovery** enabled (Postgres WAL archiving or managed-DB feature).
- [x] **Migrations run on every deploy** — `flask db upgrade` in entrypoint or release phase.
- [x] **Migrations are reversible** — every `upgrade()` has a `downgrade()`.

### Web Server
- [x] **Gunicorn (or Granian) with multiple workers** — `2 × CPU + 1`.
- [x] **`--timeout 30`** (or 60) — slow workers get killed and restarted.
- [x] **`--max-requests 1000 --max-requests-jitter 100`** — restart workers periodically (mitigates memory leaks).
- [x] **Nginx or Caddy in front** — for TLS, static files, buffering.
- [x] **Gunicorn bound to `127.0.0.1:8000`**, not `0.0.0.0` (only Nginx can reach it).
- [x] **systemd `Restart=always`** or Docker `restart: unless-stopped`.

### HTTPS & Networking
- [x] **HTTPS via Let's Encrypt / Caddy** — auto-renewal enabled.
- [x] **HTTP → HTTPS redirect** — no plaintext traffic.
- [x] **HSTS header** — `Strict-Transport-Security: max-age=31536000; includeSubDomains`.
- [x] **`ProxyFix` middleware** so Flask sees the user's real IP and scheme.
- [x] **Firewall** (ufw / security groups) — only 22, 80, 443 open.

### Application
- [x] **`/health` and `/ready` endpoints** — for load balancer checks.
- [x] **CSRF protection** on every form (`flask-wtf`).
- [x] **Rate limiting** on login/reset endpoints (`flask-limiter`).
- [x] **Security headers** — `Content-Security-Policy`, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy` (use `flask-talisman`).
- [x] **`SESSION_COOKIE_SECURE=True`** in production (only over HTTPS).
- [x] **`SESSION_COOKIE_HTTPONLY=True`** (default, but verify).
- [x] **`SESSION_COOKIE_SAMESITE='Lax'`** — CSRF protection for top-level navigations.
- [x] **Password hashing** with bcrypt or argon2 — never SHA-256 or MD5.

### Files & Uploads
- [x] **`MAX_CONTENT_LENGTH`** set — DoS protection.
- [x] **`secure_filename`** on every user-provided filename.
- [x] **File type validation** — extension + MIME + magic bytes.
- [x] **Uploads stored outside `static/`** or in S3/R2.

### Observability
- [x] **Logging to stdout** — captured by Docker/journald.
- [x] **Structured logging** (JSON via structlog) if using a log aggregator.
- [x] **Error monitoring** — Sentry or equivalent.
- [x] **Uptime monitoring** — UptimeRobot, Pingdom, or platform-native.
- [x] **Metrics** — request rate, error rate, p95 latency (Prometheus + Grafana, or platform-native).
- [x] **Tracing** — OpenTelemetry if multi-service.

### Operations
- [x] **CI/CD** — automated test → build → deploy on every push to main.
- [x] **Rollback plan** — know how to roll back before you need to.
- [x] **Resource limits** — Docker memory/CPU limits, or platform plan sized to traffic.
- [x] **Out-of-disk alerts** — disk full is the #1 cause of "mysterious" outages.
- [x] **On-call rotation** — someone gets woken up when Sentry fires at 3 AM.

> [!quote] A note on the checklist
> This list looks intimidating, but most items take 5 minutes once. After your first production launch, you'll set up these patterns in a single afternoon for any new app. The point is **not** to do all 35 items before launch; it's to *know which ones you're skipping* and accept the risk consciously.

## 19. Exercises

> [!todo] These will solidify your deployment muscle

### Exercise 1 — Run Gunicorn locally
Take any app from the course (e.g. the gallery from Lesson 21). Install Gunicorn. Run `gunicorn -w 4 -b 127.0.0.1:8000 app:app`. Visit `http://localhost:8000`. Confirm it works without `flask run`.

### Exercise 2 — Add `/health` and `/ready`
Add `/health` (always 200) and `/ready` (200 only if the database is reachable). Test by stopping Postgres and watching `/ready` return 503.

### Exercise 3 — Dockerize the Lesson 22 email app
Write a `Dockerfile` and `docker-compose.yml` for the email + Celery app. You need three services: `web` (Flask), `worker` (Celery), `redis` (broker). The worker uses the same image but a different `command:`.

### Exercise 4 — Deploy to Render (free tier)
Push your app to GitHub, create a Render web service, add a Render Postgres database, set the env vars, and visit the live URL. Take a screenshot — this is your first deployed app.

### Exercise 5 — Add HTTPS on a VPS
Rent a $5 DigitalOcean droplet, point a domain at it (use a free `.dev` or your own), install Nginx + certbot, and serve your Flask app over HTTPS. Confirm with `curl -v https://yourdomain.com/health` — you should see a valid certificate chain.

### Exercise 6 — Add Sentry
Sign up for Sentry's free tier. `pip install sentry-sdk[flask]`. Initialize it with your DSN. Trigger an error (e.g. `raise Exception('test')` in a route). See it appear in the Sentry dashboard with a full stack trace.

### Exercise 7 — Set up systemd
On a Linux machine (a VPS or a local VM), install the systemd unit from Section 4.5. Start the service, kill Gunicorn with `kill -9`, and watch systemd restart it within 3 seconds. Then reboot the VM and confirm the service comes up automatically.

### Exercise 8 — Load test with `wrk`
Install `wrk` (`apt install wrk` or `brew install wrk`). Run `wrk -t4 -c100 -d30s http://localhost:8000/health`. Compare throughput with `-w 1` vs `-w 4` vs `-w 9` Gunicorn workers. Find the sweet spot for your CPU.

## 20. What We Learned

This was a long lesson — let's summarize the architecture-level lessons:

1. **`app.run(debug=True)` is for development only.** It's single-threaded, exposes code via the debugger, and dies when SSH closes.
2. **Production uses a WSGI server** (Gunicorn). Gunicorn forks N workers, load-balances, restarts crashed workers, and handles signals properly.
3. **The worker count rule of thumb is `2 × CPU + 1`.** Start there, load-test, adjust.
4. **Nginx sits in front of Gunicorn.** It terminates TLS, serves static files directly, sets security headers, and buffers slow clients.
5. **All secrets come from environment variables.** Use `python-dotenv` for local dev. Never commit `.env`.
6. **PostgreSQL in production, not SQLite.** Concurrent writes need a real database.
7. **Migrations run on every deploy.** Use Flask-Migrate and `flask db upgrade` in your entrypoint or release phase.
8. **Docker packages your app + dependencies reproducibly.** The two-step `COPY requirements.txt + pip install` is critical for layer caching.
9. **docker-compose orchestrates multi-container apps** — Flask + Postgres + Redis + Celery worker all defined in one YAML file.
10. **systemd (or Docker `restart: unless-stopped`) keeps Gunicorn alive** — auto-starts on boot, restarts on crash.
11. **Let's Encrypt gives free TLS** via `certbot`. Set up auto-renewal.
12. **Production checklist:** debug off, strong SECRET_KEY, Postgres, env vars, HTTPS, Gunicorn with workers, Nginx, logging, Sentry, backups, restart-on-crash.
13. **Platform choice depends on budget and learning goals.** Render for easy, DigitalOcean for learning, AWS for scale, Cloud Run for bursty traffic.

```mermaid
mindmap
  root((Deployment))
    Why not app.run
      single-threaded
      debug exposes code
      no process management
      no TLS
    Gunicorn
      WSGI server
      N workers 2*CPU+1
      master + forks
      systemd to keep alive
    Nginx
      reverse proxy
      terminates TLS
      serves static files
      proxy_set_header for X-Forwarded
    Configuration
      env vars for secrets
      .env + python-dotenv
      .gitignore .env
      DEBUG=False
      strong SECRET_KEY
    Docker
      Dockerfile
      slim base image
      requirements.txt layer caching
      non-root user
      CMD exec form
    docker-compose
      web + db + redis
      healthcheck
      depends_on condition
      volumes for db persistence
    Production checklist
      Postgres not SQLite
      HTTPS via Let's Encrypt
      /health endpoint
      migrations on deploy
      Sentry for errors
      backups
    Platforms
      Heroku easy expensive
      Render easy cheap
      Railway beautiful UI
      Fly.io multi-region
      DigitalOcean learn Linux
      AWS scale complex
```

## 21. Next Lesson

There is no next lesson — **this is the end of the course.** 🎉

You've gone from `app = Flask(__name__)` and a single "Hello, World" route all the way to a containerized, TLS-secured, multi-worker Flask application running on the public internet with a real database, background workers, and error monitoring.

What's left now is to **build something**. Pick a project — a blog, a task tracker, a habit app, a URL shortener, an API for something you care about — and ship it. Every lesson in this course exists because at some point, building real things, you needed it.

A few directions worth exploring next, now that you have the foundations:

- **Async Flask** with `Quart` or ASGI — for WebSockets and streaming responses.
- **FastAPI** — a sister framework optimized for APIs, with automatic OpenAPI docs.
- **HTMX + Flask** — server-rendered HTML with reactive frontend features, no React.
- **Kubernetes** — when Docker Compose isn't enough.
- **CI/CD** with GitHub Actions — automate tests and deploys on every push.
- **Observability** — structured logging, OpenTelemetry, distributed tracing.

But those are bonus topics. The fundamentals you've learned in these 23 lessons are enough to build and ship almost anything. Go make something.

> [!success] You finished the Flask Teaching Course.
> You can now: build a Flask app from scratch, structure it with blueprints, model data with SQLAlchemy, authenticate users, expose a REST API, accept file uploads, send email, run background tasks, and deploy the whole thing to the internet with Docker, Gunicorn, and Nginx. That's the full stack.

Thank you for following along. 🚀
