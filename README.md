# Operator Sync API — Studio System Sync Flow

Base sa "Studio - System Sync Flow" PDF. Tanan endpoints operator-authed (require `OPERATOR:3SMANIA COMBO` / `OPERATOR:HARI-TARI COMBO` / `OPERATOR:REGNUM COMBO` grant, resolved per round's game_code).

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

## Auth

Tanan endpoints require operator JWT (same as existing operator API). Grant resolved per round's `game_code`:
- `3smania` → `OPERATOR:3SMANIA COMBO`
- `hari-tari` → `OPERATOR:HARI-TARI COMBO`
- `regnum` → `OPERATOR:REGNUM COMBO`

Header: `Authorization: Bearer <token>`

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
