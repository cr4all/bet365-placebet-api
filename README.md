<div align="center">

# bet365-placebet-api

**Bet365 bet placement as an HTTP service.**
No browser. No chromedriver. No UI selectors. JSON in, bets out.

[![Docs](https://img.shields.io/badge/API%20Console-live-0A66C2?style=for-the-badge&logo=swagger&logoColor=white)](https://linegate.com/docs)
[![Telegram](https://img.shields.io/badge/Telegram-@cr4all__dev-2CA5E0?style=for-the-badge&logo=telegram&logoColor=white)](https://t.me/cr4all_dev)
[![REST](https://img.shields.io/badge/API-REST%20v1-111111?style=for-the-badge)](https://linegate.com/docs)

**[Quickstart](#quickstart) · [How it works](#how-it-works) · [Endpoint map](#endpoint-map) · [QR auth](#qr-authentication) · [Errors](#errors--idempotency) · [FAQ](#faq) · [Access](#access--pricing)**

</div>

---

> **TL;DR** — `POST /v1/bet365/sessions` once to log an account in (QR handled server-side), then `POST /v1/bet365/sessions/{id}/bets` for every bet slip you want placed. Keepalive, balance, open bets, history, cashout — all over HTTP.

---

## Quickstart

| | |
|---|---|
| **Base URL** | `https://linegate.com` |
| **Docs / console** | [linegate.com/docs](https://linegate.com/docs) |
| **Auth** | `Authorization: Bearer b365_…` |
| **Content type** | `application/json` |

Three calls and you're live.

### 1 · Log an account in

Creates the upstream session and logs in — QR verification, if bet365 asks for it, is solved for you. Concurrency is capped by your plan (`maxConcurrentSessions`).

```bash
curl -sX POST https://linegate.com/v1/bet365/sessions \
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

Keep the `id` — it's your handle for everything that follows.

### 2 · Place a bet

`Idempotency-Key` is **required**. Retry with the same key and you can never double-place.

```bash
curl -sX POST https://linegate.com/v1/bet365/sessions/6f1e2c3a-4b5d-4e6f-8a9b-0c1d2e3f4a5b/bets \
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

`type` accepts `singles`, `multiples`, and `betbuilder`; `selections[]` carries `sport_id`, `fi`, `id`, `odd`, plus optional `line`, `stake`, `eachway`.

### 3 · Watch the account

```bash
# Balance (free / bonus)
curl -s https://linegate.com/v1/bet365/sessions/$SESSION_ID/balance -H "Authorization: Bearer b365_xxxxxxxxxxxx"

# Everything still running
curl -s https://linegate.com/v1/bet365/sessions/$SESSION_ID/openbets -H "Authorization: Bearer b365_xxxxxxxxxxxx"

# Full placement history
curl -s "https://linegate.com/v1/bet365/bets?page=1&page_size=50" -H "Authorization: Bearer b365_xxxxxxxxxxxx"
```

<details>
<summary><b>Full Python example (login → place → verify)</b></summary>

```python
import uuid
import requests

BASE = "https://linegate.com"
AUTH = {"Authorization": "Bearer b365_xxxxxxxxxxxx"}

# 1 · log in (QR handled automatically)
session = requests.post(f"{BASE}/v1/bet365/sessions", headers=AUTH, json={
    "domain": "bet365.com",
    "username": "account_user",
    "password": "secret",
    "proxy": "http://user:pass@1.2.3.4:8080",
    "country_code": "GB",
    "device": "android",
    "keepalive": True,
}).json()

session_id = session["id"]

# 2 · place a bet
bet = requests.post(
    f"{BASE}/v1/bet365/sessions/{session_id}/bets",
    headers={**AUTH, "Idempotency-Key": str(uuid.uuid4())},
    json={
        "type": "singles",
        "stake": 10.00,
        "selections": [
            {"sport_id": 1, "fi": 123456789, "id": 987654321, "odd": "1.85"}
        ],
    },
).json()

print("bet placed:", bet["bet"]["upstreamBetId"], "| replayed:", bet["replayed"])

# 3 · confirm
assert requests.get(f"{BASE}/v1/bet365/sessions/{session_id}/active", headers=AUTH).json()["data"]["active"]
```

</details>

Everything else — every field, every response, a live "Try it out" console — lives at **[linegate.com/docs](https://linegate.com/docs)**.

---

## How it works

```mermaid
sequenceDiagram
    autonumber
    participant BOT as Your software
    participant LG as Linegate API
    participant B365 as bet365

    BOT->>LG: POST /v1/bet365/sessions
    LG->>B365: Login (QR solved if required)
    B365-->>LG: Session active
    LG-->>BOT: sessionId
    BOT->>LG: POST /sessions/{id}/bets + Idempotency-Key
    LG->>B365: Place bet
    B365-->>LG: Bet accepted
    LG-->>BOT: upstreamBetId
    Note over LG,B365: keepalive keeps the session warm between bets
```

**Design principles**

- **Server-side by default** — cookies, device fingerprints, proxies and QR solving live on our side. You never run a browser.
- **One session, one account** — every session pins its own proxy, device profile and geo. No cross-account bleed.
- **Idempotent writes** — placements take an `Idempotency-Key`; a replay returns the original result with `"replayed": true`.
- **Sessions that stay warm** — `keepalive` plus explicit `…/keepalive` pings keep logins alive between bets.
- **Predictable envelopes** — reads return `data` + `requestId`; failures return a typed error object (below).

### Browser automation → two HTTP calls

```diff
- driver = webdriver.Chrome()
- driver.get("https://bet365.com")
- WebDriverWait(driver, 30).until(EC.element_to_be_clickable(...))
- driver.find_element(By.CSS_SELECTOR, "#betslip-stake").send_keys("10")
- driver.find_element(By.CSS_SELECTOR, "#place-bet").click()
- # ...repeat per market, per account, pray the DOM survives the night
+ requests.post("https://linegate.com/v1/bet365/sessions", headers=AUTH, json=account)
+ requests.post(f"{BASE}/v1/bet365/sessions/{session_id}/bets", headers=AUTH, json=slip)
```

---

## Endpoint map

```text
https://linegate.com
│
├─ POST   /v1/bet365/sessions                                Create session + log in          ← start here
├─ GET    /v1/bet365/sessions                                List sessions
├─ GET    /v1/bet365/sessions/{id}                           Get session state
├─ PATCH  /v1/bet365/sessions/{id}                           Rewire proxy / device / geo
├─ DELETE /v1/bet365/sessions/{id}                           Tear down session
├─ POST   /v1/bet365/sessions/{id}/login                     Login / re-login
├─ POST   /v1/bet365/sessions/{id}/logout                    Logout
├─ POST   /v1/bet365/sessions/{id}/keepalive                 Keep session warm
├─ GET    /v1/bet365/sessions/{id}/balance                   Balance (cash / bonus)
├─ GET    /v1/bet365/sessions/{id}/status                    Upstream status
├─ GET    /v1/bet365/sessions/{id}/active                    Alive upstream?
├─ GET    /v1/bet365/sessions/{id}/openbets                  Open bets
├─ GET    /v1/bet365/sessions/{id}/settledbets               Settled bets
├─ POST   /v1/bet365/sessions/{id}/geocheck                  Geocheck
├─ POST   /v1/bet365/sessions/{id}/authenticator/verify      Auth-app / 2FA challenge
│
├─ POST   /v1/bet365/sessions/{sid}/bets                     Place a bet                     ← Idempotency-Key required
├─ POST   /v1/bet365/sessions/{sid}/bets/{betId}/approval    Referral approval check
├─ POST   /v1/bet365/sessions/{sid}/bets/{betId}/receipt     Italian receipt payload
├─ GET    /v1/bet365/bets                                    Betting history (paginated, filterable)
├─ POST   /v1/bet365/sessions/{sid}/cashout                  Cash out an open bet
│
├─ POST   /v1/bet365/guest/sessions                          Guest / data session (no account needed)
└─ GET    /v1/bet365/prematch                                Prematch payload
```

---

## QR authentication

Some accounts and regions get a QR challenge at login. You never see it — it's part of the session lifecycle:

1. `POST /v1/bet365/sessions` (or `…/{id}/login`) starts the upstream login.
2. If bet365 raises a QR challenge, the backend solves it through its QR infrastructure.
3. You get `"status": "ACTIVE"` back; `GET …/{id}/active` double-checks it upstream.
4. Every later bet reuses the live session — no re-auth, no manual steps, ever.

No screenshots, no scan-this-code moments, no device farms to babysit.

---

## Errors & idempotency

| Status | Meaning |
|--------|---------|
| `400` | Invalid body — or missing `Idempotency-Key` on a placement |
| `401` | Missing or invalid API key |
| `403` | Subscription inactive, or plan's concurrency / quota exceeded |
| `404` | Unknown session / bet id |

Every failure comes back in one shape:

```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "API key is required",
    "safe_retry": false
  }
}
```

- `safe_retry: true` → retry (jittered backoff is your friend). `false` → fix the request first.
- Placement is **idempotent**: reuse the same `Idempotency-Key` and you get the original bet back with `"replayed": true` — never a duplicate.

---

## FAQ

**Do I need a browser, a GUI VPS, or chromedriver?**
No. Everything — login, QR, placement, cashout — happens server-side behind this API.

**Will running accounts through this get them flagged?**
Each session is isolated with its own proxy, device profile and geo, and keepalive traffic models a real client. No headless-browser artifacts, no shared fingerprints between accounts.

**My placement request timed out. Did the bet land?**
Re-POST with the same `Idempotency-Key`. If the original went through you'll get `"replayed": true` and the original bet — not a second one.

**Multiples, bet builders, each-way?**
Yes — `type: "multiples" | "betbuilder"`, plus `eachway` and per-selection stake/line fields.

**Cashout?**
`POST /v1/bet365/sessions/{sid}/cashout` with the upstream `betid` and target `total_returns`.

**Which regions are supported?**
Most bet365 regions, including QR-required ones. Ping [@cr4all_dev](https://t.me/cr4all_dev) with your country and we'll confirm.

**How many accounts can run at once?**
One session per account; how many sessions run in parallel follows the slots on your plan.

---

## Access & pricing

Priced per **session slot** — one slot = one bet365 account live at the same time.

> 1 account online = 1 slot · 25 accounts in parallel = 25 slots · 100+ = bulk pricing.

To get a key, pick your slots, or ask anything technical:

<div align="center">

**👉 Telegram — [@cr4all_dev](https://t.me/cr4all_dev)**

</div>

---

<details>
<summary><b>Responsible use</b></summary>

This service is intended only for users operating accounts they own or are explicitly authorized to manage. Clients are responsible for ensuring their use complies with all applicable laws, regulations, platform rules, and local gambling requirements. We do not guarantee profits, betting outcomes, account status, or sportsbook decisions.

</details>

<div align="center">

**bet365-placebet-api** — placement · QR · keepalive · cashout · history

`https://linegate.com` · [docs](https://linegate.com/docs) · [@cr4all_dev](https://t.me/cr4all_dev)

</div>
