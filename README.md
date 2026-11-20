# 🔥🤖 bet365-placebet-api

> **Bet365 bet placement through a clean, scalable REST API — no browser required.**
> Send a request, place a bet. Automate your sportsbook workflow from any backend.

[![Telegram](https://img.shields.io/badge/Telegram-Contact%20%26%20Sales-2CA5E0?style=for-the-badge&logo=telegram)](https://t.me/cr4all_dev)
[![API](https://img.shields.io/badge/API-REST-green?style=for-the-badge)]()
[![Status](https://img.shields.io/badge/status-stable-brightgreen?style=for-the-badge)]()

---

## 🚀 What Is This?

**bet365-placebet-api** is a server-side service that lets you place bets on **bet365** directly from your own software — bots, trading tools, alert systems, or dashboards — using pure API calls.

No headless browsers. No Selenium, Puppeteer, or UI selectors. No fragile DOM automation that breaks on every layout change. Your application sends the bet details over HTTP, and the backend handles the entire placement flow for your authorized sessions.

The service also handles **QR code authentication**, so accounts that require QR verification can log in and place bets without any manual interaction.

---

## ✨ Key Features

- 🌐 **Browser-free** — pure API integration, zero browser automation
- 📲 **QR authentication support** — QR login flows handled automatically
- ✅ **Clean account handling** — stable sessions, safe request patterns, no account flags
- 🧩 **Simple integration** — a few HTTP requests is all it takes
- ⚡ **Low-latency placement** — built for time-sensitive odds
- 🤖 **Made for automation** — bots, value bet scanners, alert tools, arbitrage workflows
- 📈 **Scales by session slot** — run 1 account or 100+ simultaneously
- 🔒 **Session isolation** — each account runs in its own isolated slot
- 🛠️ **Documented REST API** — predictable JSON in, JSON out

---

## 🧠 Why Use It?

Most automated bet365 setups depend on browser sessions, extensions, or headless browsers. These are expensive to maintain, hard to scale, and break constantly.

This API replaces all of that:

| Browser automation | bet365-placebet-api |
|---|---|
| Fragile UI selectors | Stable REST endpoints |
| Heavy server resources | Lightweight HTTP calls |
| Breaks on site updates | Backend maintained for you |
| Hard to scale | Add session slots |
| Manual QR logins | Automated QR authentication |

---

## ⚙️ How It Works

```mermaid
flowchart LR
    A[Your Bot / Software] --> B[PlaceBet API]
    B --> C[Session Slot]
    C --> D[bet365 Account]
    C --> E[QR Authentication]
    D --> F[Bet Placement Result]
    F --> A
```

1. **Your software sends bet details** to the API.
2. **The backend routes the request** through the account's active session slot.
3. **QR authentication is handled** automatically when required.
4. **The bet is placed** on bet365 and the result is returned as JSON.
5. **Scale up** by adding session slots — one slot per account running simultaneously.

---

## 🔌 Quick Start

**POC base URL:** `https://linegate.com`
**Swagger UI:** [https://linegate.com/docs](https://linegate.com/docs)

Every request is authenticated with your API key (`b365_…`) as a Bearer token:

```
Authorization: Bearer b365_xxxxxxxxxxxx
```

### 1. Create a Session (Login)

Creates an upstream placebet session and logs in. QR authentication is handled automatically. Concurrent sessions are limited by your plan (`maxConcurrentSessions`).

```bash
curl -X POST "https://linegate.com/v1/bet365/sessions" \
  -H "Authorization: Bearer b365_xxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "domain": "bet365.com",
    "username": "account_user",
    "password": "secret",
    "proxy": "http://user:pass@1.2.3.4:8080",
    "country_code": "GB",
    "device": "android",
    "keepalive": true
  }'
```

Response:

```json
{
  "id": "6f1e2c3a-4b5d-4e6f-8a9b-0c1d2e3f4a5b",
  "kind": "ACCOUNT",
  "status": "ACTIVE",
  "domain": "bet365.com",
  "username": "account_user",
  "countryCode": "GB",
  "device": "android",
  "keepalive": true,
  "createdAt": "2026-11-20T12:00:00.000Z"
}
```

### 2. Place a Bet

`Idempotency-Key` is **required** — reuse the same key to retry safely without double-placing.

```bash
curl -X POST "https://linegate.com/v1/bet365/sessions/SESSION_UUID/bets" \
  -H "Authorization: Bearer b365_xxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: 9f2c1a3b-4d5e-4f60-8172-a3b4c5d6e7f8" \
  -d '{
    "type": "singles",
    "stake": 10.00,
    "selections": [
      {
        "sport_id": 1,
        "fi": 123456789,
        "id": 987654321,
        "odd": "1.85"
      }
    ]
  }'
```

Response:

```json
{
  "bet": {
    "id": "0b7c8d9e-1f2a-4b3c-8d4e-5f6a7b8c9d0e",
    "sessionId": "6f1e2c3a-4b5d-4e6f-8a9b-0c1d2e3f4a5b",
    "resultCode": "SUCCESS",
    "upstreamBetId": "88420193",
    "createdAt": "2026-11-20T12:01:00.000Z"
  },
  "payload": { "...": "upstream placebet payload" },
  "replayed": false
}
```

### 3. Check Session & Bet Status

```bash
# Balance
curl "https://linegate.com/v1/bet365/sessions/SESSION_UUID/balance" \
  -H "Authorization: Bearer b365_xxxxxxxxxxxx"

# Open bets
curl "https://linegate.com/v1/bet365/sessions/SESSION_UUID/openbets" \
  -H "Authorization: Bearer b365_xxxxxxxxxxxx"

# Betting history (paginated)
curl "https://linegate.com/v1/bet365/bets?page=1&page_size=50" \
  -H "Authorization: Bearer b365_xxxxxxxxxxxx"
```

### Python Example

```python
import uuid
import requests

BASE = "https://linegate.com"
HEADERS = {"Authorization": "Bearer b365_xxxxxxxxxxxx"}

# 1. Create session (login, QR handled automatically)
session = requests.post(f"{BASE}/v1/bet365/sessions", headers=HEADERS, json={
    "domain": "bet365.com",
    "username": "account_user",
    "password": "secret",
    "proxy": "http://user:pass@1.2.3.4:8080",
    "country_code": "GB",
    "device": "android",
    "keepalive": True,
}).json()

session_id = session["id"]

# 2. Place a bet
bet = requests.post(
    f"{BASE}/v1/bet365/sessions/{session_id}/bets",
    headers={**HEADERS, "Idempotency-Key": str(uuid.uuid4())},
    json={
        "type": "singles",
        "stake": 10.00,
        "selections": [
            {"sport_id": 1, "fi": 123456789, "id": 987654321, "odd": "1.85"}
        ],
    },
).json()

print(bet["bet"]["upstreamBetId"])
```

Full schemas and interactive testing: **[https://linegate.com/docs](https://linegate.com/docs)**

---

## 📲 QR Authentication

Some bet365 regions and account states require **QR code verification** during login. The API handles this natively:

1. Your request triggers a login on the assigned session slot.
2. If QR verification is required, the backend solves it through its QR infrastructure.
3. The session becomes active and is ready for bet placement.
4. All subsequent bets reuse the live session — no repeated authentication.

No screenshots, no manual scanning, no user interaction.

---

## 📚 API Overview

Base URL (POC): `https://linegate.com` — interactive docs: [`/docs`](https://linegate.com/docs)

**Bet365 Sessions** — account session lifecycle and runtime ops

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/bet365/sessions` | Create session & log in (QR handled automatically) |
| `GET` | `/v1/bet365/sessions` | List sessions |
| `GET` | `/v1/bet365/sessions/{id}` | Get a session |
| `PATCH` | `/v1/bet365/sessions/{id}` | Update proxy / device / geo |
| `DELETE` | `/v1/bet365/sessions/{id}` | Delete a session |
| `POST` | `/v1/bet365/sessions/{id}/login` | Login / re-login |
| `POST` | `/v1/bet365/sessions/{id}/logout` | Logout |
| `POST` | `/v1/bet365/sessions/{id}/keepalive` | Keepalive the session |
| `GET` | `/v1/bet365/sessions/{id}/balance` | Session balance |
| `GET` | `/v1/bet365/sessions/{id}/status` | Session status |
| `GET` | `/v1/bet365/sessions/{id}/active` | Active upstream check |
| `GET` | `/v1/bet365/sessions/{id}/openbets` | Open bets |
| `GET` | `/v1/bet365/sessions/{id}/settledbets` | Settled bets |
| `POST` | `/v1/bet365/sessions/{id}/geocheck` | Run geocheck |
| `POST` | `/v1/bet365/sessions/{id}/authenticator/verify` | Verify authenticator / 2FA challenge |

**Bet365 Bets** — placebet, approval, receipt, history

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/bet365/sessions/{sessionId}/bets` | Place a bet (requires `Idempotency-Key` header) |
| `POST` | `/v1/bet365/sessions/{sessionId}/bets/{betId}/approval` | Check bet referral approval |
| `POST` | `/v1/bet365/sessions/{sessionId}/bets/{betId}/receipt` | Fetch Italian receipt payload |
| `GET` | `/v1/bet365/bets` | Betting history (paginated, filterable) |

**Bet365 Cashout**

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/bet365/sessions/{sessionId}/cashout` | Cash out an open bet |

**Bet365 Guest & Prematch**

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/bet365/guest/sessions` | Create guest / data session |
| `GET` | `/v1/bet365/prematch` | Fetch prematch payload |

Full request/response schemas: **[https://linegate.com/docs](https://linegate.com/docs)**

---

## 💳 Pricing

Pricing is based on **session slots** — one slot per bet365 account running at the same time.

| Usage | Required Slots |
|-------|----------------|
| 1 account running simultaneously | 1 slot |
| 10 accounts running simultaneously | 10 slots |
| 50 accounts running simultaneously | 50 slots |
| 100+ accounts running simultaneously | Bulk pricing available |

For pricing, availability, and conditions, contact us on Telegram:

👉 **[@cr4all_dev](https://t.me/cr4all_dev)**

---

## 👨‍💻 Who Is This For?

- Betting bot developers
- Automated betting platforms
- Value bet & arbitrage tool operators
- Trading and alert-based betting systems
- Teams managing multiple authorized bet365 sessions
- Anyone who wants a non-browser bet placement workflow

---

## ✅ Benefits

- Save development and maintenance time
- No browser automation infrastructure to babysit
- Integrate with just a few API requests
- Run multiple accounts by adding session slots
- QR authentication and bet placement in one workflow
- Predictable, stable backend execution
- Clean account handling — sessions are isolated and request patterns are safe

---

## 🛡️ Responsible Use

This service is intended only for users operating accounts they own or are explicitly authorized to manage.

Clients are responsible for ensuring their use complies with all applicable laws, regulations, platform rules, and local gambling requirements.

We do not guarantee profits, betting outcomes, account status, or sportsbook decisions.

---

## 🤝 Get Started

Ready to integrate bet365 bet placement into your software?

1. Contact us on Telegram: **[@cr4all_dev](https://t.me/cr4all_dev)**
2. Get your API key and choose your session slots
3. Connect your software using the API examples above
4. Start placing bets through the backend workflow

---

## 📬 Contact

**Telegram:** [@cr4all_dev](https://t.me/cr4all_dev) — pricing, onboarding, technical questions, and support.

---

## 🔥 bet365-placebet-api

**Bet placement. QR authentication. Backend automation. One API.**
