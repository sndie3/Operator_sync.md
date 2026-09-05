# Operator Sync API

Base sa "Studio - System Sync Flow" PDF. 6 ka endpoints para sa round sync flow.

**Server address:** `https://demo.ept.ph`
**Port:** `443` (default HTTPS — dili na kinahanglan i-butang sa URL)

Ang tanan endpoints kay nagsugod sa `/api`. So ang full URL kay ang server address + `/api` + ang endpoint.

Example: ang Open Round endpoint kay:
```
https://demo.ept.ph/api/operator/rounds/{round_id}/sync/open
```

Ang `{round_id}` kay ilisan sa actual nga round ID (e.g. `123`).

**Auth:** I-pass ang token sa header (i-provide sa admin):
```
Authorization: Bearer <token>
```

---

## Complete Setup (Login → Create Round → Sync Flow)

### Step 0: Login (kuhaon ang token)

**Request:**
```
POST https://demo.ept.ph/api/auth/login
```

**Body:**
```json
{
  "phone": "09123456789",
  "password": "yourpassword",
  "accept_terms": true,
  "mac_address": "00:00:00:00:00:00"
}
```

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "uid": "OP001",
  "first_name": "Juan",
  "last_name": "Dela Cruz",
  "position": "OPERATOR",
  "permissions": ["OPERATOR:3SMANIA COMBO", ...],
  "feature_grants": ["OPERATOR:3SMANIA COMBO", ...]
}
```

**Kuhaon ang `token` gikan sa response — kana ang gamiton sa `Authorization: Bearer <token>` header sa tanan sunod nga API calls.**

---

### Step 1: Create Round (kuhaon ang round_id)

**Request:**
```
POST https://demo.ept.ph/api/operator/{game_code}/rounds
Authorization: Bearer <token>
```

**Body:**
```json
{
  "round_number": 5,
  "timer_seconds": 60,
  "session_id": 1
}
```

- `game_code` — `3smania`, `hari-tari`, o `regnum`
- `round_number` — number sa round (>= 1)
- `timer_seconds` — 5 to 600 seconds
- `session_id` — optional, default 1

**Response:**
```json
{ "round_id": 123, "status": "READY", "round_number": 5 }
```

**Kuhaon ang `round_id` gikan sa response — kana ang gamiton sa tibuok sync flow.**

---

### Step 2: Open Round

```
POST /api/operator/rounds/{round_id}/sync/open
```
**Response:** `{ "round_id": 123, "round_status": "Opened" }`

Operator mo-click "Start" → mo-call ni. Maghimo ang backend og RNG seed (provably fair).

---

### Step 3: Poll Timer Status (every second)

```
GET /api/operator/rounds/{round_id}/sync/timer-status
```
**Response:** `{ "round_id": 123, "round_status": "Timer On" }`

I-poll every second. I-display ang timer sa screen. Kung timer expired → `"Good2Go"`.

---

### Step 4: Good2Go (timer expired)

```
GET /api/operator/rounds/{round_id}/sync/timer-status
```
**Response:** `{ "round_id": 123, "round_status": "Good2Go" }`

Stop na ang betting. Operator mo-display sa odds ug mo-post og result.

---

### Step 5: Get Odds

```
GET /api/operator/rounds/{round_id}/sync/odds
```
**Response:** `{ "round_id": 123, "odds": { "HARI": 1.87, "TARI": 2.15 } }`

I-display ang odds sa screen.

---

### Step 6: Send Result

```
POST /api/operator/rounds/{round_id}/sync/result
```
**Body:** `{ "round_result": "HARI" }`
**Response:** `{ "round_id": 123, "round_result": "HARI" }`

Operator mo-type og result. Result format per game:
- **hari-tari:** `"HARI"`, `"TARI"`, `"DRAW"`, `"CANCEL"`
- **3smania:** single number `"0"`–`"9"`
- **regnum:** 3 cards dash-separated `"A-2-3"`

---

### Step 7: Close Round

```
POST /api/operator/rounds/{round_id}/sync/close
```
**Body:** `{ "round_result": "HARI" }`
**Response:** `{ "round_id": 123, "round_status": "Close", "round_result": "HARI" }`

Operator mo-click "Close". Backend mo-settle (payout winners, refund losers, jackpot, commission).

---

### Alternative: Cancel G2G (kung dili mo-post og result)

```
POST /api/operator/rounds/{round_id}/sync/cancel-g2g
```
**Response:** `{ "round_id": 123, "round_status": "Cancelled" }`

Refund tanan bets. Gamiton lang kung ang round kay sa Good2Go state ug dili mo-post og result.

---

### Sunod nga round

Kung bag-ong round na, balik sa **Step 1** — mag-create og round, kuhaon ang bag-ong `round_id`, dayon Steps 2-7. Dili na kinahanglan mag-login balik (gamiton pa ang same token).

---

## Endpoints

### 1. Open Round

```
POST /api/operator/rounds/{round_id}/sync/open
```

**Response:**
```json
{ "round_id": 123, "round_status": "Opened" }
```

**Errors:** `404` Round not found | `409` Round is not READY

---

### 2. Timer Status

```
GET /api/operator/rounds/{round_id}/sync/timer-status
```

I-poll every second. Auto-transition base sa timer:
- Timer running → `"Timer On"`
- Timer expired → `"Good2Go"` (stop betting)
- Cancelled → `"Cancelled"`
- Closed → `"Close"`

**Response:**
```json
{ "round_id": 123, "round_status": "Timer On" }
```

**Errors:** `404` Round not found | `409` Round is not OPEN

---

### 3. Cancel G2G

```
POST /api/operator/rounds/{round_id}/sync/cancel-g2g
```

I-cancel ang round sa Good2Go state. Refund tanan bets.

**Response:**
```json
{ "round_id": 123, "round_status": "Cancelled" }
```

**Errors:** `404` Round not found | `409` Not in Good2Go | `503` Refund float short

---

### 4. Odds

```
GET /api/operator/rounds/{round_id}/sync/odds
```

**Response (hari-tari):**
```json
{ "round_id": 123, "odds": { "HARI": 1.87, "TARI": 2.15 } }
```

**Response (3smania):**
```json
{ "round_id": 123, "odds": { "0": 3.54, "1": 3.54, "2": 3.54 } }
```

**Response (regnum):**
```json
{ "round_id": 123, "odds": { "A": 8.87, "2": 8.87, "3": 8.87 } }
```

**Errors:** `404` Round not found

---

### 5. Send Result

```
POST /api/operator/rounds/{round_id}/sync/result
```

**Request body:**
```json
{ "round_result": "HARI" }
```

Result format per game:
- **hari-tari:** `"HARI"`, `"TARI"`, `"DRAW"`, `"CANCEL"`
- **3smania:** single number `"0"`–`"9"`
- **regnum:** 3 cards dash-separated `"A-2-3"`

**Response:**
```json
{ "round_id": 123, "round_result": "HARI" }
```

**Errors:** `400` Invalid result | `404` Round not found | `409` Not OPEN or CLOSED

---

### 6. Close Round

```
POST /api/operator/rounds/{round_id}/sync/close
```

**Request body:**
```json
{ "round_result": "HARI" }
```

**Response:**
```json
{ "round_id": 123, "round_status": "Close", "round_result": "HARI" }
```

**Errors:** `404` Round not found | `409` Not OPEN or CLOSED / cancelled | `500` Settlement failed

---

## Auth

I-provide sa admin ang token. I-pass sa header sa tanan API calls:
```
Authorization: Bearer <token>
```

**Example (cURL):**
```bash
curl -X POST https://demo.ept.ph/api/operator/rounds/123/sync/open \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json"
```

**Example (JavaScript):**
```javascript
fetch("/api/operator/rounds/123/sync/open", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${token}`,
    "Content-Type": "application/json"
  }
})
```

**Errors:**
| Error | Meaning |
|-------|---------|
| `401` | Walay token, wrong token, o expired |
| `403` | Account not activated o walay operator grant |

---

## Unsa ang buhaton sa team

1. **Token** — i-provide sa admin, i-pass sa header
2. **Open** — `POST /sync/open`
3. **Timer** — `GET /sync/timer-status` every second
4. **Odds** — `GET /sync/odds` kung Good2Go na
5. **Result** — `POST /sync/result` gamit ang result
6. **Close** — `POST /sync/close` gamit ang same result
7. **Cancel** — `POST /sync/cancel-g2g` kung dili mo-post og result

Ang backend ang mo-handle sa tanan — settlement, payouts, refunds, jackpot, odds. Ang frontend mo-display lang sa response.
