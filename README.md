# HTTP Monitor

A small uptime-monitoring service written in Go. Register a URL, and the
service probes it every few seconds with a burst of HTTP requests, keeps the
recent status codes, and flags the URL as **in alert** when too many of those
requests fail.

- **REST API** with user accounts and JWT authentication
- **Scheduler + workers** decoupled by a NATS JetStream queue (run as many workers as you need)
- **SQLite** storage, no external database to set up
- **Configurable** through environment variables

---

## Table of contents

1. [How it works](#how-it-works)
2. [Requirements](#requirements)
3. [Quick start](#quick-start)
4. [Configuration](#configuration)
5. [API reference](#api-reference)
6. [Alert rules](#alert-rules)
7. [Project layout](#project-layout)
8. [Development](#development)
9. [Troubleshooting](#troubleshooting)
10. [Known limitations](#known-limitations)

---

## How it works

```
                 ┌──────────────────────── server process ───────────────────────┐
  client ──────▶ │  REST API  ──(channel)──▶  scheduler ──▶ publishes every URL   │
                 └─────┬────────────────────────────────────────┬─────────────────┘
                       │ read / write                           │ every `period` seconds
                       ▼                                        ▼
                    SQLite  ◀────── save results ────── worker(s) ◀── NATS JetStream
                                                           │
                                                           └──▶ N concurrent GET requests to the URL
```

1. A user registers or logs in and receives a **JWT**.
2. The user submits a URL and a **threshold** to `POST /request`.
3. The API stores the URL and hands it to the **scheduler**.
4. Every `common.period` seconds the scheduler publishes each monitored URL to **NATS**.
5. A **worker** picks up the message, sends `common.requests` concurrent GET requests
   (10 by default), appends the status codes to the URL's history and saves it.
6. The user reads results from `/result` or `/result/url`, and checks `/alerts`.

On startup the scheduler reloads every URL from the database, so restarting the
server does not stop monitoring.

The project builds to a single binary with two sub-commands: `server` and `worker`.

## Requirements

| Requirement | Why |
|---|---|
| Go 1.16+ | build the project |
| A C compiler (gcc/clang) | the SQLite driver uses cgo |
| A NATS server **with JetStream** | message queue between scheduler and worker |

Install NATS from <https://docs.nats.io/running-a-nats-service/introduction/installation>
or run it with Docker:

```bash
docker run -p 4222:4222 nats:latest -js
```

## Quick start

```bash
# 1. get the code and build
git clone https://github.com/shirinebadi/http-monitor.git
cd http-monitor/cmd/http-monitor
go build

# 2. start NATS with JetStream enabled (separate terminal)
nats-server -js

# 3. set a secret for signing tokens (required, use a long random value)
export HTTP_MONITOR_JWT_SECRET="$(openssl rand -hex 32)"

# 4. start the server (separate terminal)
./http-monitor server

# 5. start a worker (separate terminal, SAME directory and SAME secret/env)
./http-monitor worker
```

> The server and the worker share one SQLite file (`HttpMDB.db`, relative to
> the directory you start them from). Start both from the same directory, or set
> `HTTP_MONITOR_DATABASE_PATH` to an absolute path for both.

Try it:

```bash
# create an account (returns a token)
curl -s -X POST localhost:21345/register \
  -H 'Content-Type: application/json' \
  -d '{"name":"ali","password":"a-good-password"}'

export TOKEN="<paste the token here>"

# monitor a site, alert after 3 failures
curl -s -X POST localhost:21345/request \
  -H 'Content-Type: application/json' -H "Token: $TOKEN" \
  -d '{"url":"https://example.com","threshold":3}'

# wait a few seconds, then look at the results
curl -s localhost:21345/result -H "Token: $TOKEN"
curl -s localhost:21345/alerts -H "Token: $TOKEN"
```

## Configuration

Defaults are defined in `internal/app/http-monitor/config/default.go`. Every key
can be overridden with an environment variable: prefix `HTTP_MONITOR_`, uppercase,
dots replaced by underscores.

| Key | Environment variable | Default | Description |
|---|---|---|---|
| `server.address` | `HTTP_MONITOR_SERVER_ADDRESS` | `:21345` | HTTP listen address |
| `jwt.secret` | `HTTP_MONITOR_JWT_SECRET` | *(none, required)* | Secret used to sign tokens. The server refuses to start without it |
| `jwt.expiration` | `HTTP_MONITOR_JWT_EXPIRATION` | `60` | Token lifetime, in minutes |
| `nats.host` | `HTTP_MONITOR_NATS_HOST` | `nats://localhost:4222` | NATS server URL |
| `nats.topic` | `HTTP_MONITOR_NATS_TOPIC` | `url` | NATS subject used for jobs |
| `nats.name` | `HTTP_MONITOR_NATS_NAME` | `httpMonitor` | JetStream stream name |
| `database.path` | `HTTP_MONITOR_DATABASE_PATH` | `HttpMDB.db` | SQLite file shared by server and worker |
| `common.period` | `HTTP_MONITOR_COMMON_PERIOD` | `3` | Seconds between two monitoring rounds |
| `common.requests` | `HTTP_MONITOR_COMMON_REQUESTS` | `10` | HTTP requests per round, per URL |
| `common.timeout` | `HTTP_MONITOR_COMMON_TIMEOUT` | `10` | Seconds before a single request is abandoned |
| `common.window` | `HTTP_MONITOR_COMMON_WINDOW` | `100` | Number of most recent results kept and counted (must be ≥ `requests`) |
| `common.workers` | `HTTP_MONITOR_COMMON_WORKERS` | `20` | URLs one worker process checks at the same time |

The server and worker should run with the same configuration.

## API reference

All endpoints speak JSON. Errors have the form `{"error": "message"}`.

Endpoints marked 🔒 need a token, sent as either:

```
Token: <token>
Authorization: Bearer <token>
```

### `POST /register`

Creates an account and returns a token.

```json
{ "name": "ali", "password": "a-good-password" }
```

- `name`: required, up to 64 characters
- `password`: 8 to 72 characters

| Status | Meaning | Body |
|---|---|---|
| 201 | created | `{"token": "..."}` |
| 400 | invalid input | `{"error": "..."}` |
| 409 | name already taken | `{"error": "username already taken"}` |

### `POST /login`

Same body as `/register`.

| Status | Meaning | Body |
|---|---|---|
| 200 | success | `{"token": "..."}` |
| 401 | wrong name or password | `{"error": "wrong name or password"}` |

### `POST /request` 🔒

Starts monitoring a URL.

```json
{ "url": "https://example.com", "threshold": 3 }
```

- `url`: required, must start with `http://` or `https://`
- `threshold`: optional number, default `5`, between 1 and `common.window`

| Status | Meaning | Body |
|---|---|---|
| 201 | monitoring started | `{"url": "https://example.com", "threshold": 3}` |
| 400 | invalid URL or threshold | `{"error": "..."}` |
| 401 | missing or invalid token | `{"error": "..."}` |
| 409 | you already monitor this URL | `{"error": "..."}` |

### `GET /result` 🔒

Returns the stored status codes of **all** your URLs, oldest first.

```json
{
  "https://example.com": [200, 200, 200, 503, 200, 0]
}
```

A code of `0` means the request itself failed (timeout, DNS error, connection refused).

### `POST /result/url` 🔒

Returns the codes of one URL.

```json
{ "url": "https://example.com" }
```

| Status | Meaning |
|---|---|
| 200 | `{"https://example.com": [200, 200, ...]}` |
| 404 | you are not monitoring that URL |

### `GET /alerts` 🔒

Returns the URLs that are currently in alert. URLs that are healthy are not listed.

```json
{ "https://example.com": true }
```

## Alert rules

- A response is **healthy only if its status code is 2xx**. Redirects are followed
  automatically, so a `301` that ends in `200` counts as healthy. Anything else, including
  `3xx`/`4xx`/`5xx` that remain after redirects and failed requests (`0`), counts as a failure.
- Only the newest `common.window` results are kept (100 by default, about 10 rounds).
- A URL is **in alert** when `failures in the window >= threshold` (and there is at least one failure).
- Because old results age out, a URL that recovers leaves the alert state by itself
  once enough healthy rounds have pushed the failures out of the window.

## Project layout

```
cmd/http-monitor/            program entry point
internal/app/http-monitor/
├── cmd/                     cobra commands: root, server, worker
│   ├── server/              API + scheduler wiring
│   └── worker/              URL checker
├── config/                  defaults, env overrides, validation
├── data/
│   ├── db/                  GORM + SQLite access
│   └── nats/                JetStream publish / subscribe
├── handler/                 HTTP handlers, JWT, auth middleware
├── model/                   User, Url, Status and their interfaces
├── request/                 request bodies and input validation
└── scheduler/               periodic publishing of monitored URLs
design.png                   original architecture sketch
```

## Development

```bash
go mod tidy        # first time, to sync dependencies
go vet ./...
go test ./...
```

Unit tests cover the status/history logic, input validation, the scheduler and
configuration loading. The API and the worker are best tested end to end with
the curl commands from the [quick start](#quick-start).

To watch the queue while developing, install the `nats` CLI and run:

```bash
nats stream info httpMonitor
nats consumer info httpMonitor monitor
```

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `jwt.secret is not set` | Export `HTTP_MONITOR_JWT_SECRET` before starting the server |
| `connect to NATS ... connection refused` | NATS is not running, or `nats.host` is wrong |
| `enable JetStream (is nats-server started with -js?)` | Restart NATS with the `-js` flag |
| `Binary was compiled with 'CGO_ENABLED=0'` / sqlite errors at build | Install gcc/clang and build with `CGO_ENABLED=1` |
| Results stay empty | The worker is not running, or it uses a different database file or NATS settings than the server |
| `database is locked` | Server and worker should share one local file; do not put it on a network drive |
| Every login fails after upgrading | Older databases store plain-text passwords; delete `HttpMDB.db` and register again |

## Known limitations

- There is no endpoint to delete or pause a monitored URL yet.
- URLs are not restricted, so users can make the worker request internal
  addresses (SSRF). Run it only for trusted users or inside a restricted network.
- If several workers run at once, two of them can occasionally check the same URL
  in the same round (the overlap guard is per worker process).
- SQLite is meant for small deployments. For heavy use, switch to PostgreSQL.
- Alerts are pull-based (`/alerts`); there are no email/webhook notifications.
