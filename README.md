# Webhook Relay

[![Python Version](https://img.shields.io/badge/python-3.10%20%7C%203.11%20%7C%203.12-blue.svg)](https://www.python.org/)
[![CI Build](https://img.shields.io/badge/build-passing-brightgreen.svg)](https://github.com/dailyprojectfactory/webhook-relay)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Code Style: Ruff](https://img.shields.io/badge/code%20style-ruff-000000.svg)](https://github.com/astral-sh/ruff)

A developer-centric reverse proxy and local interception utility designed to capture, inspect, validate, and replay incoming HTTP webhooks. Built with FastAPI, Typer, and SQLite, Webhook Relay records raw payloads and headers with microsecond fidelity, verifies cryptographic HMAC signatures, and enables zero-friction re-dispatching against local development servers without triggering upstream third-party webhooks.

---

## Problem Statement & Target Audience

Integrating third-party webhooks (e.g., Stripe, GitHub, Twilio, Shopify) during local development presents recurring engineering bottlenecks:

- **Ephemerality & Tunnel Breakdowns**: Tunnels like ngrok or Cloudflare Tunnels forward requests into ephemeral application states; if the local service crashes, the event payload is lost.
- **Provider Burnout & Rate Limits**: Testing edge-case handling or bugfixes requires triggering upstream actions repeatedly, polluting external sandbox environments or exceeding API rate quotas.
- **Opaque Signatures**: Debugging HMAC signature verification failures (e.g., mismatched raw bodies, header canonicalization mismatches) is difficult without inspecting the verbatim byte stream and raw HTTP headers.

**Target Audience:** Backend engineers, API developers, and QA automation teams building or maintaining services that rely on reliable ingestion of external event notifications.

---

## Key Features

- **Dynamic Ingestion Proxy**: Captures incoming HTTP `POST` and `PUT` webhook payloads across arbitrary path routes (`/capture/{path:path}`) while fully preserving raw byte buffers, query parameters, and downstream headers.
- **HMAC Signature Verification & Schema Validation**: Computes and evaluates cryptographic digests (SHA-256, SHA-512) against configurable provider headers (`X-Hub-Signature-256`, `Stripe-Signature`, etc.) and validates incoming JSON payloads using strict Pydantic schemas.
- **Embedded Persistence Engine**: Records transaction metadata, raw body content, request headers, downstream relay attempts, and response status codes into an embedded, zero-configuration SQLite store.
- **Interactive Terminal Dashboard & Replay Engine**: Provides an interactive terminal interface built with Typer and Rich to live-tail captured requests, inspect formatted JSON/headers, and trigger manual re-dispatches to local endpoints via asynchronous HTTP clients (`httpx`).

---

## Architecture & System Design

Webhook Relay operates as an intermediary capture node sitting between public edge tunnels (e.g., ngrok) and your target internal service.

```
                                  ┌────────────────────────────────────────┐
                                  │             Webhook Relay              │
                                  │                                        │
┌─────────────────┐  POST / PUT   │  ┌──────────────┐   HMAC / Schema Check│
│  External Host  │──────────────▶│  │  FastAPI     │───────────────────┐  │
│ (Stripe, GitHub)│               │  │  Capture API │                   ▼  │
└─────────────────┘               │  └──────┬───────┘          ┌────────┴──────┐
                                  │         │                  │ Pydantic /    │
                                  │         ▼                  │ Cryptography  │
                                  │  ┌──────────────┐          └────────┬──────┘
                                  │  │ Core Engine  │◀──────────────────┘  │
                                  │  │ (src/core.py)│                      │
                                  │  └──────┬───────┘                      │
                                  │         │                              │
                                  │         ├──────────────┐               │
                                  │         ▼              ▼               │
                                  │  ┌──────────────┐ ┌───────────────┐    │
                                  │  │ SQLite Store │ │ Async Dispatch│    │
                                  │  │ (Local DB)   │ │ (httpx)       │    │
                                  │  └──────────────┘ └───────┬───────┘    │
                                  └───────────────────────────┼────────────┘
                                                              │
                                            POST /local/event │
                                                              ▼
                                                   ┌─────────────────────┐
                                                   │ Local Backend App   │
                                                   │ (e.g., port 8000)   │
                                                   └─────────────────────┘
```

### Module Responsibilities

- **`src/models.py`**: Defines domain models representing incoming payload snapshots, database record schemas, replay configurations, and cryptographic signature policies.
- **`src/core.py`**: Houses the persistent SQLite abstraction layer, the HMAC verification pipeline, and the asynchronous `httpx` HTTP forwarding/replay client.
- **`src/api.py`**: Implements the capture gateway routing logic (`/capture/...`) and exposure of the management REST API (`/api/events/...`) for headless automation.
- **`src/main.py`**: Unified entry point orchestrating the Uvicorn web process alongside the Typer CLI subcommands.

---

## Technology Stack

| Layer | Component | Description |
| :--- | :--- | :--- |
| **Runtime** | Python 3.10+ | Core language platform |
| **API Framework** | FastAPI | High-performance asynchronous capture and management gateway |
| **CLI Framework** | Typer & Rich | Command-line dispatch interface, formatted tables, and live logs |
| **Data Validation** | Pydantic v2 | Strict serialization and schema verification models |
| **Persistence** | SQLite | Serverless, zero-maintenance local relational datastore |
| **Networking** | HTTPX | Fully asynchronous HTTP/1.1 and HTTP/2 client for replaying events |
| **Server** | Uvicorn | ASGI web server hosting capture routes |
| **Test Engine** | Pytest | Unit and integration test orchestration |

---

## Installation & Virtual Environment Setup

### Prerequisites

- Python 3.10 or higher
- Git

### Step-by-Step Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/dailyprojectfactory/webhook-relay.git
   cd webhook-relay
   ```

2. **Provision an isolated virtual environment:**
   ```bash
   python -m venv .venv
   source .venv/bin/activate  # On Windows: .venv\Scripts\activate
   ```

3. **Install runtime and development dependencies:**
   ```bash
   pip install --upgrade pip
   pip install -r requirements.txt
   ```

---

## Environment Variables Configuration

Copy the sample environment file to create your local runtime configuration:

```bash
cp .env.example .env
```

### Configuration Schema

| Variable | Default Value | Description |
| :--- | :--- | :--- |
| `RELAY_HOST` | `127.0.0.1` | Network interface to bind the relay ingestion server. |
| `RELAY_PORT` | `8888` | Port where incoming webhooks are captured. |
| `RELAY_DB_PATH` | `webhooks.sqlite3` | File path for the embedded SQLite database. |
| `TARGET_URL` | `http://127.0.0.1:8000` | Default downstream endpoint where captured requests are replayed. |
| `HMAC_SECRET` | `""` | Secret key used to verify cryptographic signatures (optional). |
| `HMAC_HEADER` | `X-Signature` | Header name containing the signature hash to validate against. |

---

## Usage: CLI Commands & API Endpoints

### 1. Starting the Ingestion Gateway

Start the relay server using the integrated Typer CLI:

```bash
python -m src.main serve --port 8888 --target http://localhost:8000
```

### 2. Capturing Webhooks

Point your webhook sources or forwarding tunnels to the capture endpoint. All routes matching `/capture/{path}` are recorded:

```bash
curl -X POST http://127.0.0.1:8888/capture/stripe \
     -H "Content-Type: application/json" \
     -H "X-Signature: 8b0662657e2bb7a87c1" \
     -d '{"id": "evt_test123", "type": "payment_intent.succeeded", "amount": 2500}'
```

### 3. CLI Inspection and Live Tailing

List the most recent events captured in the local database:

```bash
python -m src.main list --limit 10
```

*Terminal Output:*
```
┏━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━┓
┃ Event ID             ┃ Method  ┃ Path             ┃ Status Code    ┃ Captured At         ┃
┡━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━┩
│ 3c49a038-f14d-487b   │ POST    │ /capture/stripe  │ 200 (Stored)   │ 2026-03-31 02:40:11 │
└──────────────────────┴─────────┴──────────────────┴────────────────┴─────────────────────┘
```

Inspect the raw payload, headers, and signature status of an event:

```bash
python -m src.main inspect 3c49a038-f14d-487b
```

### 4. Replaying Payloads

Re-dispatch an event to the configured target server:

```bash
python -m src.main replay 3c49a038-f14d-487b --target http://localhost:8000/api/v1/webhooks
```

### 5. Management REST API

The service also exposes a RESTful interface for programmatic queries and replay automation:

- **List Recorded Webhooks:**
  ```bash
  curl http://127.0.0.1:8888/api/events
  ```
- **Fetch Specific Payload Snapshot:**
  ```bash
  curl http://127.0.0.1:8888/api/events/3c49a038-f14d-487b
  ```
- **Trigger Remote Replay:**
  ```bash
  curl -X POST http://127.0.0.1:8888/api/events/3c49a038-f14d-487b/replay \
       -H "Content-Type: application/json" \
       -d '{"target_url": "http://127.0.0.1:8000/hooks/stripe"}'
  ```

---

## Automated Testing Instructions

The test suite validates database integrity, signature verification routines, and API endpoints using `pytest` and `httpx.AsyncClient`.

Run the complete test suite:

```bash
pytest
```

Run tests with test coverage reporting:

```bash
pytest --cov=src --cov-report=term-missing
```

Run core operational tests specifically:

```bash
pytest tests/test_core.py -v
```

---

## Project Structure

```
webhook-relay/
├── .env.example              # Sample environment definitions
├── .gitignore                # Git ignore specifications
├── requirements.txt          # Python dependencies
├── src/
│   ├── __init__.py           # Package marker
│   ├── api.py                # FastAPI capture routing & management endpoints
│   ├── core.py               # SQLite storage, HMAC validation, httpx dispatcher
│   ├── main.py               # Application entry point & Typer CLI commands
│   └── models.py             # Pydantic schemas and database entity models
└── tests/
    ├── __init__.py           # Test package marker
    ├── test_api.py           # Ingestion and management endpoint tests
    └── test_core.py          # Storage, replay client, and HMAC unit tests
```

---

## Roadmap / Future Enhancements

- [ ] **Dynamic Mutation Engine**: Allow modifying payload fields inline via the CLI prior to replaying.
- [ ] **Pre-configured Provider Presets**: Built-in verification logic for complex multi-header formats (e.g., Stripe timestamp-signature pairs, GitHub SHA-1/SHA-256 fallback).
- [ ] **Automatic Retry Workflows**: Configurable exponential backoff forwarding to target endpoints for testing service recovery under network failure.
- [ ] **Bidirectional Webhook Proxying**: Transparent pass-through proxying that captures both request bodies and downstream application responses in real time.

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for full details.