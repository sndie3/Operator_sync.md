# Operator Sync API — Studio System Sync Flow

Base sa "Studio - System Sync Flow" PDF. Tanan endpoints nangayo og token sa header (`Authorization: Bearer <token>`, parehas sa ubang API). Ang operator grant kay resolved per round's game_code: `OPERATOR:3SMANIA COMBO` / `OPERATOR:HARI-TARI COMBO` / `OPERATOR:REGNUM COMBO`.

Base URL: `/api`

## State Machine

```
NULL → OPENED → TIMER_ON → G2G → CLOSE
                         ↘ CANCELLED  (Cancel G2G, refund all bets)
```

Ang `sync_state` kay separate column sa `ma_operator_rounds` — dili naapektuhan ang existing `status` lifecycle (READY → OPEN → CLOSED → SETTLED).

---

## 1. Open Round Replay

**PDF payload #1**: `{ Round ID, Round Status: "Opened" }`

```
POST /api/operator/rounds/{round_id}/sync/open
```

Transitions round READY → OPEN (commits fair-RNG seed), sets `sync_state = 'OPENED'`.

**Response:**
```json
{
  "round_id": 123,
  "round_status": "Opened"
}
```

**Errors:**
- `404` — Round not found
- `409` — Round is not READY / another round already open

---

## 2. Round Timer Status

**PDF payload #2**: `{ Round ID, Round Status: "Timer On" | "Good2Go" | "Cancelled" }`

```
GET /api/operator/rounds/{round_id}/sync/timer-status
```

Polling endpoint. Auto-transitions based on timer:
- Timer running → `sync_state = 'TIMER_ON'` → returns `"Timer On"`
- Timer expired → status OPEN→CLOSED, `sync_state = 'G2G'` → returns `"Good2Go"`
- Already cancelled → returns `"Cancelled"`
- Already closed → returns `"Close"`

**Response:**
```json
{
  "round_id": 123,
  "round_status": "Timer On"
}
```

**Errors:**
- `404` — Round not found
- `409` — Round is not OPEN

---

## 3. Cancel G2G

**PDF: "Cancel G2G"** (appears on both pages)

```
POST /api/operator/rounds/{round_id}/sync/cancel-g2g
```

Cancels a round in Good2Go state. **Refunds ALL bets** at full stake (no rake, no settlement). Uses `COMBO_REFUND` ref_type. Distinct from the existing `/cancel` endpoint (which only works on rounds with NO bets).

**Response:**
```json
{
  "round_id": 123,
  "round_status": "Cancelled"
}
```

**Errors:**
- `404` — Round not found
- `409` — Round is not in Good2Go state
- `503` — Refund float short

---

## 4. Compute & Reply Odds

**PDF: "Compute & Reply Odds"**

```
GET /api/operator/rounds/{round_id}/sync/odds
```

Returns odds for the round:
- **hari-tari** (parimutuel): dynamic odds = `net_pool / side_total` per side
- **3smania / regnum** (fixed multiplier): published combo multiplier

**Response (hari-tari):**
```json
{
  "round_id": 123,
  "odds": {
    "HARI": 1.87,
    "TARI": 2.15
  }
}
```

**Response (3smania):**
```json
{
  "round_id": 123,
  "odds": {
    "0": 3.54,
    "1": 3.54,
    "2": 3.54
  }
}
```

**Response (regnum):**
```json
{
  "round_id": 123,
  "odds": {
    "A": 8.87,
    "2": 8.87,
    "3": 8.87
  }
}
```

**Errors:**
- `404` — Round not found

---

## 5. Send Round Result

**PDF payload #3**: `{ RoundID, Round Result }`

```
POST /api/operator/rounds/{round_id}/sync/result
```

Stages the operator's result WITHOUT closing/settling. Player lobby sees it immediately via current-round poll. Settlement happens at `/sync/close`.

**Request body:**
```json
{
  "round_result": "HARI"
}
```

Result format per game:
- **hari-tari**: `"HARI"`, `"TARI"`, `"DRAW"`, or `"CANCEL"`
- **3smania**: single number `"0"`–`"9"`
- **regnum**: 3 cards dash-separated `"A-2-3"`

**Response:**
```json
{
  "round_id": 123,
  "round_result": "HARI"
}
```

**Errors:**
- `400` — Invalid result
- `404` — Round not found
- `409` — Round is not OPEN or CLOSED

---

## 6. Close Round Reply

**PDF payload #4**: `{ Round ID, Round Status: "Close", Round Result }`

```
POST /api/operator/rounds/{round_id}/sync/close
```

Closes the round with the declared result. Stops betting (if still OPEN), runs full settlement (payouts/refunds/jackpot), sets `sync_state = 'CLOSE'`.

**Request body:**
```json
{
  "round_result": "HARI"
}
```

**Response:**
```json
{
  "round_id": 123,
  "round_status": "Close",
  "round_result": "HARI"
}
```

**Errors:**
- `404` — Round not found
- `409` — Round is not OPEN or CLOSED / was cancelled
- `500` — Settlement failed

---

## Auth (Paano maka-access sa API)

Ang API kay dili public — kinahanglan mag-pass og **token** sa header. Parehas ra na sa uban nga API nga gigamit sa team. Dili kinahanglan maghimo og login screen — ang token kay i-pass lang sa header sa tanan API calls.

### Unsa ang 3 ka check sa auth?

Kung ang operator mag-call sa bisan unsa nga sync endpoint, mo-check ang backend sa 3 ka butang:

```
CHECK 1: Token valid ba?         → Kung dili, 401 (walay token, wrong token, o expired na)
CHECK 2: Account activated ba?   → Kung dili, 403 (naay account pero dili pa activated)
CHECK 3: Naay operator grant ba? → Kung wala, 403 (activated ka pero walay permissyon sa game)
```

Kung tanan 3 ka check OK → padayon ang request. Kung bisag usa lang ang fail → mo-error.

### Unsa ang buhaton sa team? Token lang ang kinahanglan

Ang token kay **i-provide lang sa admin** — dili na kinahanglan mangita ang team. I-butang lang sa header sa tanan sync API calls:

```
Authorization: Bearer <token>
```

**Example (cURL):**
```bash
curl -X POST http://localhost:8000/api/operator/rounds/123/sync/open \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json"
```

**Example (JavaScript/frontend):**
```javascript
const response = await fetch("/api/operator/rounds/123/sync/open", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${token}`,   // token nga gi-provide sa admin
    "Content-Type": "application/json"
  }
})
```

Kung walay header o wrong ang token → **401 Unauthorized**.

### Operator Grant (kinsa ka ba sa unsa nga game)

Bisan naay token ug activated ang account, kinahanglan pa naay **operator grant** para sa game nga round nga gi-call. Ang grant kay resolved **per round's game_code** — buot pasabot, ang backend mo-check unsa nga game ang round, dayon mo-check kung ang operator naay permissyon para sa game.

| Game | Grant Key (permissyon nga kinahanglan) |
|------|----------------------------------------|
| 3smania | `OPERATOR:3SMANIA COMBO` |
| hari-tari | `OPERATOR:HARI-TARI COMBO` |
| regnum | `OPERATOR:REGNUM COMBO` |

Ang grant kay pwede ma-assign sa duha ka paagi:
1. **Per-user** — directly sa `ma_user_permissions` table (specific user ra)
2. **Per-role** — sa `ma_user_roles` → `ma_role_permissions` (tanang users sa role)

Kung ang operator walay grant para sa game sa round → **403 Insufficient permissions**.

### Auth Flow (simplified)

```
┌─────────────────────────────────────────────────────────────┐
│  OPERATOR (frontend)                                        │
│                                                             │
│  1. Token nga gi-provide sa admin                           │
│                                                             │
│  2. Tanan sync API calls:                                   │
│     Authorization: Bearer <token>                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  BACKEND (3 ka check)                                       │
│                                                             │
│  CHECK 1: Token valid ba?                                   │
│     → Kung dili: 401 "Invalid or expired token"             │
│                                                             │
│  CHECK 2: Account activated ba?                             │
│     → Kung dili: 403 "Account not activated"                │
│                                                             │
│  CHECK 3: Naay operator grant para sa game?                 │
│     → Kung wala: 403 "Insufficient permissions"             │
│                                                             │
│  ✓ Tanan OK → padayon ang request                           │
└─────────────────────────────────────────────────────────────┘
```

### Common Auth Errors

| Error | Meaning | Unsa ang buhaton |
|-------|---------|------------------|
| `401` Invalid or expired token | Walay token, wrong token, o expired na | Mangayo og bag-ong token sa admin |
| `403` Account not activated | Naay account pero dili pa activated | I-activate ang account sa admin |
| `403` Insufficient permissions | Activated ka pero walay operator grant | Kuha og operator grant sa admin (`OPERATOR:3SMANIA COMBO` etc.) |

---

## Flow (end-to-end)

```
1. POST /sync/open          → { round_status: "Opened" }
2. GET  /sync/timer-status   → { round_status: "Timer On" }   (poll until timer expires)
3. GET  /sync/timer-status   → { round_status: "Good2Go" }    (timer expired)
4. GET  /sync/odds           → { odds: {...} }                 (compute odds)
5. POST /sync/result         → { round_result: "HARI" }        (post result)
6. POST /sync/close          → { round_status: "Close", round_result: "HARI" }  (settle)

   ↳ Alternative at step 3: POST /sync/cancel-g2g → { round_status: "Cancelled" }  (refund all)
```

---

## Unsa ang buhaton sa API (para sa dili programmers)

Ang API kay **backend code** nga nagdagan sa server. Ang iyang trabaho:

1. **Mo-dawat sa requests gikan sa operator console** (frontend)
2. **Mo-check kung pwede ang operator** (auth — kinsa ka ba, pwede ka ba)
3. **Mo-process ang round lifecycle** base sa PDF flow:
   - Open round → start timer → wait for timer → Good2Go
   - Post result → close round → settle (compute winners, payouts, refunds)
4. **Mo-return og JSON response** nga i-display sa frontend

Ang frontend (operator console) kay **wala nag-compute og bisan unsa** — tanan logic naa sa backend. Ang frontend mo-call lang sa endpoints ug i-display ang response.

### Unsa ang buhaton sa team (frontend)?

1. **Token** — i-provide sa admin, i-pass sa `Authorization: Bearer <token>` header
2. **Open button** — mo-call sa `POST /sync/open`
3. **Timer display** — mag-poll sa `GET /sync/timer-status` every second, i-display ang `round_status`
4. **G2G screen** — kung `Good2Go` na, i-display ang odds gikan sa `GET /sync/odds`
5. **Result input** — operator mo-type og result (e.g. "HARI"), mo-call sa `POST /sync/result`
6. **Close button** — mo-call sa `POST /sync/close` gamit ang same result
7. **Cancel button** — kung dili mo-post og result, mo-call sa `POST /sync/cancel-g2g`

Tanan computation (settlement, payouts, refunds, jackpot, odds) — **backend ang mo-ana**. Frontend mo-display lang.
