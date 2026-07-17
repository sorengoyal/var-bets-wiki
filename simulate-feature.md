# VAR Bets - Simulate & Fast-Forward Feature

## Overview
A debug toolbar visible at all times on the frontend that allows resetting the entire simulation state or fast-forwarding the mock clock. Designed for hackathon demo scenarios where you need to repeatedly walk through the full VAR lifecycle: fixture sync → VAR event → bet → VAR resolution → payout.

## Architecture

```
┌──────────────────┐     ┌──────────────┐     ┌─────────────────┐
│   NextJS App     │────▶│  NestJS API   │────▶│ Mock Service     │
│  [Simulate btn]  │     │  /simulate/*   │     │ /simulate/*     │
│  [Fast-Forward]  │     │  (orchestrates)│     │ (clock only)    │
└──────────────────┘     └──────┬─────────┘     └─────────────────┘
                                │
                                ▼
                         ┌──────────────┐
                         │  PostgreSQL  │
                         │  (truncated) │
                         └──────────────┘
```

The NestJS backend is the orchestrator: it resets the mock clock *and* clears its own database in a single transactional operation, then broadcasts the reset to all connected frontends via WebSocket.

---

## 1. Mock Service API (New Endpoints)

| Endpoint | Method | Description |
| :--- | :--- | :--- |
| `/simulate/reset` | `POST` | Resets the virtual clock to the `START_TIME` from `.env`. |
| `/simulate/fast-forward` | `POST` | Advances the virtual clock by `{ minutes: N }`. |

**`POST /simulate/reset`**
- No body required.
- Response: `{ "currentTime": "<new iso time>" }`

**`POST /simulate/fast-forward`**
- Body: `{ "minutes": 5 }`
- Response: `{ "currentTime": "<new iso time>" }`

---

## 2. Backend Orchestrator (New Endpoint)

**`POST /api/simulate/reset`**

Wrapped in a single DB transaction:

1. Call Mock Service `POST /simulate/reset`.
2. `TRUNCATE` tables: `bets`, `payouts`, `pools`, `scores`.
3. Set all `fixtures_metadata.active = false`.
4. Commit transaction.
5. Broadcast `simulationReset` via WebSocket to all connected clients.

**`POST /api/simulate/fast-forward`**

1. Call Mock Service `POST /simulate/fast-forward` with `{ minutes }`.
2. Return `{ "currentTime" }`.
3. The existing cron jobs (fixtures poller, score listener) will naturally pick up newly-available events on their next tick.

---

## 3. Frontend UI

### Placement
A fixed debug toolbar in the **top-right header**, always visible. Styled subtly to distinguish it from user-facing controls (e.g., slightly muted background, monospace font for the clock display).

### Components
- **Current virtual time display** — e.g. `⏱ 12:03:00`
- **Fast-Forward button** — `+1m` and `+5m` buttons (or a dropdown). Each click calls `POST /api/simulate/fast-forward`.
- **🔄 Restart Simulation button** — calls `POST /api/simulate/reset`.

### Behavior
- On click, call the backend endpoint.
- On success (for reset), the page auto-refreshes (or clears local state and re-fetches).
- WebSocket `simulationReset` event triggers an auto-refresh on *all* connected browser tabs.

---

## 4. WebSocket Gateway (New Event)

| Event | Direction | Payload | Description |
| :--- | :--- | :--- | :--- |
| `simulationReset` | Server → Client | `{}` | Broadcast to all clients after a reset completes. Frontend should clear local state and re-fetch pools/fixtures. |

---

## 5. Reset Transaction (DB Truncation Order)

Since tables have foreign key relationships, the truncation order matters:
1. `bets` (depends on `pools`)
2. `payouts` (depends on `pools`)
3. `pools` (depends on `fixtures`)
4. `scores` (depends on `fixtures`)
5. Then `UPDATE fixtures_metadata SET active = false`

Or equivalently: `TRUNCATE bets, payouts, pools, scores CASCADE` + the metadata update.

---

## 6. Fast-Forward User Flow

1. User clicks `+5m`.
2. Backend advances mock clock by 5 minutes.
3. On next cron tick (≤1 minute), score listener sees new events → creates pool if `"var"` is now in the past.
4. Frontend receives `poolUpdated` via WebSocket → new pool appears.

This means fast-forward is not instant from the UI perspective — there's up to a 1-minute delay (the score listener interval). For the demo, you could reduce the cron interval, or add an optional `?trigger=true` param to fast-forward that immediately invokes the cron logic synchronously.
