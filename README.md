# railway-iot-infra

Docker Compose and infrastructure configuration for **railway-iot-platform**.

---

## Structure

```
railway-iot-infra/
├── docker-compose.yml    ← Full stack definition (8 services + simulator profile)
├── .env                  ← Local secrets (not committed)
├── .env.example          ← Template for new environments
└── nginx/
    └── nginx.conf        ← Reverse proxy + rate limiting + security headers
```

---

## Quick Start

```bash
cp .env.example .env
# Edit .env — set strong values for SECRET_KEY, DB_PASSWORD, REDIS_PASSWORD, ML_API_KEY

docker compose up -d

# Check all services are healthy
docker compose ps

# View logs
docker compose logs -f api

# Start simulator (separate profile)
docker compose --profile simulator up -d simulator
```

---

## Services

| Service | Image / Build | Internal Port | Description |
|---|---|---|---|
| `nginx` | `nginx:1.27-alpine` | 8003 (host) | Reverse proxy, rate limiting, security headers |
| `api` | `../railway-iot-django-api` | 8000 | Django 5 ASGI (Daphne) — REST + WebSocket |
| `ml` | `../railway-iot-ml-engine` | 8004 | FastAPI anomaly detection engine |
| `dashboard` | `../railway-iot-dashboard` | 3100 | Next.js 14 dashboard |
| `db` | `timescale/timescaledb:2.14.2-pg16` | 5432 | TimescaleDB (PostgreSQL 16) |
| `redis` | `redis:7.2-alpine` | 6379 | Celery broker + Django Channels layer |
| `mqtt` | `../railway-iot-mqtt-broker` | 1883 | Mosquitto MQTT 5.0 broker |
| `celery` | `../railway-iot-celery-worker` | — | Async task workers |
| `simulator` | `../railway-iot-sensor-simulator` | — | Sensor simulator (profile: `simulator`) |

---

## Environment Variables

Copy `.env.example` to `.env` and configure all values before starting.

### Database

| Variable | Default | Description |
|---|---|---|
| `DB_NAME` | `railway_db` | PostgreSQL database name |
| `DB_USER` | `railway` | Database user |
| `DB_PASSWORD` | — | **Required.** Set a strong password |
| `DB_HOST` | `db` | Hostname (Docker service name) |
| `DB_PORT` | `5432` | PostgreSQL port |

### Redis

| Variable | Description |
|---|---|
| `REDIS_PASSWORD` | Redis AUTH password |
| `REDIS_URL` | Full Redis URL (used by Django + Celery) |

### Django

| Variable | Description |
|---|---|
| `SECRET_KEY` | **Required.** Django secret key (min 50 chars). No default — server fails to start if unset |
| `DEBUG` | `True` for local dev, `False` for production |
| `ALLOWED_HOSTS` | Comma-separated list of allowed hostnames |
| `CORS_ALLOWED_ORIGINS` | Comma-separated list of allowed CORS origins |

### MQTT

| Variable | Default | Description |
|---|---|---|
| `MQTT_HOST` | `mqtt` | Mosquitto hostname |
| `MQTT_PORT` | `1883` | MQTT port |
| `MQTT_USERNAME` | — | Broker username (leave blank for dev) |
| `MQTT_PASSWORD` | — | Broker password (leave blank for dev) |
| `MQTT_TOPIC` | `rail/#` | Subscription topic pattern |

### ML Engine

| Variable | Description |
|---|---|
| `ML_API_KEY` | API key for ML Engine. Sent as `X-Api-Key` header from Django |

### Dashboard (build-time)

| Variable | Description |
|---|---|
| `NEXT_PUBLIC_API_URL` | REST API base URL baked into the Next.js build |
| `NEXT_PUBLIC_WS_URL` | WebSocket base URL baked into the Next.js build |

> **Note:** `NEXT_PUBLIC_*` variables are baked at build time by `npm run build`. Changing them requires rebuilding the dashboard image: `docker compose build --no-cache dashboard`.

---

## Nginx

Nginx listens on port **8003** and proxies:

| Path | Upstream | Notes |
|---|---|---|
| `/api/` | `api:8000` | REST API |
| `/ws/` | `api:8000` | WebSocket upgrade |
| `/admin/` | `api:8000` | Django Admin |
| `/ml/` | `ml:8004` | ML Engine (internal) |
| `/` | `dashboard:3000` | Next.js frontend |

### Security Headers

All responses include:
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `X-XSS-Protection: 1; mode=block`
- `Content-Security-Policy` (restricts sources)
- `Referrer-Policy: strict-origin-when-cross-origin`
- `server_tokens off` (hides Nginx version)

### Rate Limiting

| Zone | Limit | Applies To |
|---|---|---|
| `auth` | 10 req/min per IP | `/api/v1/auth/` |
| `api` | 60 req/min per IP | `/api/v1/` |
| `ws` | 20 conn/min per IP | `/ws/` |

---

## Useful Commands

```bash
# Rebuild a single service
docker compose build --no-cache api

# Restart without rebuilding
docker compose restart api

# Flush Redis (clear Celery queues)
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" FLUSHALL

# Django management commands
docker compose exec api python manage.py createsuperuser
docker compose exec api python manage.py migrate
docker compose exec api python manage.py shell

# Follow Celery worker logs
docker compose logs -f celery

# View MQTT messages
docker compose exec mqtt mosquitto_sub -h localhost -t "rail/#" -v
```

---

## Production Notes

- Set `DEBUG=False` and use a unique, strong `SECRET_KEY`
- Place a real TLS certificate on Nginx (e.g., Let's Encrypt)
- Enable MQTT authentication (`MQTT_USERNAME` / `MQTT_PASSWORD`) and TLS on port 8883
- Use Docker secrets or a secrets manager (Vault, AWS Secrets Manager) instead of `.env` files
- Set `ALLOWED_HOSTS` and `CORS_ALLOWED_ORIGINS` to your actual domain names
