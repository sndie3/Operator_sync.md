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

---

## Complete Code Example (JavaScript / Frontend)

Ari ang complete code unsaon pagkonek gikan sa frontend hangtod sa close sa round. Copy-paste lang ni sa imong project.

### Setup

```javascript
// I-configure kini sa imong .env o config file
const API_BASE = "https://demo.ept.ph/api";

// I-store ang token sa localStorage o state management (Redux, Context, etc.)
// Kuhaon gikan sa login response
let token = localStorage.getItem("operator_token") || null;
let currentRoundId = null;  // kini ang round_id nga atong gamiton sa tibuok flow
```

### 1. Login (kuhaon ang token)

```javascript
async function login(phone, password) {
  const res = await fetch(`${API_BASE}/auth/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      phone: phone,           // e.g. "09123456789"
      password: password,
      accept_terms: true,
      mac_address: "00:00:00:00:00:00"
    })
  });

  if (!res.ok) {
    throw new Error(`Login failed: ${res.status}`);
  }

  const data = await res.json();
  token = data.token;  // KINI ang token — gamiton sa tanan sunod nga API calls

  // I-store sa localStorage para dili mag-login balik kung mo-refresh ang page
  localStorage.setItem("operator_token", token);

  console.log("Login successful. Token saved.");
  return data;
}
```

### 2. Create Round (kuhaon ang round_id)

```javascript
async function createRound(gameCode, roundNumber, timerSeconds, sessionId = 1) {
  const res = await fetch(`${API_BASE}/operator/${gameCode}/rounds`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${token}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      round_number: roundNumber,    // e.g. 5
      timer_seconds: timerSeconds,  // 5 to 600
      session_id: sessionId         // optional, default 1
    })
  });

  if (!res.ok) {
    const err = await res.json();
    throw new Error(`Create round failed: ${err.detail || res.status}`);
  }

  const data = await res.json();
  currentRoundId = data.round_id;  // KINI ang round_id — gamiton sa tibuok sync flow

  console.log(`Round created. round_id = ${currentRoundId}`);
  return data;
}
```

### 3. Open Round (sugdan ang round)

```javascript
async function openRound() {
  if (!currentRoundId) throw new Error("Wala pay round_id. Mag-create og round una.");

  const res = await fetch(`${API_BASE}/operator/rounds/${currentRoundId}/sync/open`, {
    method: "POST",
    headers: { "Authorization": `Bearer ${token}` }
  });

  if (!res.ok) {
    const err = await res.json();
    throw new Error(`Open round failed: ${err.detail || res.status}`);
  }

  const data = await res.json();
  console.log(`Round ${data.round_id} status: ${data.round_status}`);  // "Opened"
  return data;
}
```

### 4. Poll Timer Status (every second)

```javascript
let timerInterval = null;

async function pollTimerStatus() {
  const res = await fetch(`${API_BASE}/operator/rounds/${currentRoundId}/sync/timer-status`, {
    headers: { "Authorization": `Bearer ${token}` }
  });

  if (!res.ok) {
    const err = await res.json();
    throw new Error(`Timer status failed: ${err.detail || res.status}`);
  }

  const data = await res.json();
  console.log(`Round ${data.round_id} status: ${data.round_status}`);

  if (data.round_status === "Good2Go") {
    // Stop na ang betting. Kuhaon ang odds ug mo-post og result.
    clearInterval(timerInterval);
    await getOdds();
  } else if (data.round_status === "Cancelled" || data.round_status === "Close") {
    clearInterval(timerInterval);
  }

  return data;
}

function startPollingTimer() {
  // I-poll every 1 second
  timerInterval = setInterval(pollTimerStatus, 1000);
}
```

### 5. Get Odds (kung Good2Go na)

```javascript
async function getOdds() {
  const res = await fetch(`${API_BASE}/operator/rounds/${currentRoundId}/sync/odds`, {
    headers: { "Authorization": `Bearer ${token}` }
  });

  if (!res.ok) {
    const err = await res.json();
    throw new Error(`Get odds failed: ${err.detail || res.status}`);
  }

  const data = await res.json();
  console.log("Odds:", data.odds);
  // e.g. { "HARI": 1.87, "TARI": 2.15 }

  // I-display ang odds sa screen diri...
  return data;
}
```

### 6. Send Result (i-post ang result)

```javascript
async function sendResult(roundResult) {
  // roundResult format per game:
  //   hari-tari: "HARI", "TARI", "DRAW", "CANCEL"
  //   3smania:   "0" to "9"
  //   regnum:    "A-2-3" (3 cards, dash-separated)

  const res = await fetch(`${API_BASE}/operator/rounds/${currentRoundId}/sync/result`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${token}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({ round_result: roundResult })
  });

  if (!res.ok) {
    const err = await res.json();
    throw new Error(`Send result failed: ${err.detail || res.status}`);
  }

  const data = await res.json();
  console.log(`Result posted: ${data.round_result}`);
  return data;
}
```

### 7. Close Round (settle ang round)

```javascript
async function closeRound(roundResult) {
  const res = await fetch(`${API_BASE}/operator/rounds/${currentRoundId}/sync/close`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${token}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({ round_result: roundResult })  // same result sa Step 6
  });

  if (!res.ok) {
    const err = await res.json();
    throw new Error(`Close round failed: ${err.detail || res.status}`);
  }

  const data = await res.json();
  console.log(`Round ${data.round_id} closed. Result: ${data.round_result}`);
  currentRoundId = null;  // reset para sa sunod nga round
  return data;
}
```

### 8. Cancel G2G (backup — kung dili mo-post og result)

```javascript
async function cancelG2G() {
  const res = await fetch(`${API_BASE}/operator/rounds/${currentRoundId}/sync/cancel-g2g`, {
    method: "POST",
    headers: { "Authorization": `Bearer ${token}` }
  });

  if (!res.ok) {
    const err = await res.json();
    throw new Error(`Cancel G2G failed: ${err.detail || res.status}`);
  }

  const data = await res.json();
  console.log(`Round ${data.round_id} status: ${data.round_status}`);  // "Cancelled"
  currentRoundId = null;
  return data;
}
```

---

## Complete Flow (gamiton nimo tanan nga functions)

```javascript
// === MAIN FLOW ===

async function runRoundFlow() {
  try {
    // Step 1: Login (kung wala pay token)
    if (!token) {
      await login("09123456789", "yourpassword");
    }

    // Step 2: Create Round
    // gameCode: "hari-tari", "3smania", o "regnum"
    await createRound("hari-tari", 5, 60);

    // Step 3: Open Round
    await openRound();

    // Step 4: Start polling timer (every 1 second)
    // Kung Good2Go na, auto-stop ang polling ug mo-call sa getOdds()
    startPollingTimer();

    // Step 5 & 6: Kung Good2Go na, operator mo-type og result
    // e.g. button click sa UI
    // await sendResult("HARI");

    // Step 7: Close round (same result sa Step 6)
    // await closeRound("HARI");

  } catch (err) {
    console.error("Error:", err.message);
  }
}

// I-run
runRoundFlow();
```

---

## Common Errors ug unsaon

| Status | Meaning | Unsa ang buhaton |
|--------|---------|-------------------|
| `401` | Walay token, wrong token, o expired | Mag-login balik para kuha og bag-ong token |
| `403` | Account not activated o walay operator grant | I-check sa admin ang imong account |
| `404` | Round not found | I-check kung sakto ang `round_id` |
| `409` | Round is not READY / OPEN / Good2Go | I-check ang state sa round sa wala pa mag-call |
| `400` | Invalid result | I-check ang result format per game |
| `500` | Settlement failed | I-contact ang backend team |
| `503` | Refund float short | I-contact ang admin (kulang ang settlement pool) |

---

## Mga importanteng reminders

1. **Token** — i-store sa localStorage, dili i-hardcode sa code
2. **round_id** — dili same sa round_number. Ang `round_id` kay gikan sa backend response.
3. **Timer polling** — every 1 second lang, ayaw sobra para dili mo-overload ang server
4. **Result format** — sunod sa game type:
   - hari-tari: `"HARI"`, `"TARI"`, `"DRAW"`, `"CANCEL"`
   - 3smania: `"0"` ngadto sa `"9"`
   - regnum: `"A-2-3"` (3 cards, dash-separated)
5. **Close result** — same sa result nga imong gi-send sa Step 6
6. **Sunod nga round** — balik sa Create Round, same token pa. Dili na kinahanglan mag-login balik.
