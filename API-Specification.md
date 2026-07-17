# VAR Bets - API Specification

## Overview
The backend is built with **NestJS** and provides a RESTful API for the NextJS frontend and integration with external services (TxLine). 

- **Base URL**: `/api`
- **Format**: JSON
- **Auth**: Public (Hackathon Version).
- **Identity**: User authenticity is verified via their connected Solana wallet address passed in requests.

---

## 1. Pools API
Manages the prediction markets for VAR reviews.

| Endpoint | Method | Description | Key Parameters |
| :--- | :--- | :--- | :--- |
| `/pools` | `GET` | List all active and resolved pools. | `filter=active\|resolved` |
| `/pools/:id` | `GET` | Get detailed state of a specific pool. | `id` (Numeric) |
| `/pools` | `POST` | Create a pool (Internal/Cron). | `{ fixtureId }` |
| `/pools/:id` | `PATCH`| Update pool state (Debug/Manual). | `{ acceptingBets, paidOut }` |
| `/pools/:id` | `DELETE`| Remove pool record. | `id` (Numeric) |

## 2. Fixtures API
Manages football match metadata.

| Endpoint | Method | Description | Key Parameters |
| :--- | :--- | :--- | :--- |
| `/fixtures` | `GET` | List all tracked fixtures. | `active=true\|false` |
| `/fixtures/:id` | `GET` | Get detailed fixture info. | `id` (Numeric) |
| `/fixtures` | `POST` | Manual fixture creation. | `{ Competition, Participant1, ... }` |
| `/fixtures/:id` | `PATCH`| Update fixture metadata. | `{ active: boolean }` |
| `/fixtures/:id` | `DELETE`| Remove fixture. | `id` (Numeric) |

## 3. Bets API
Tracks user wagers. Since there is no smart contract, bets are logged directly in the DB.

| Endpoint | Method | Description | Key Parameters |
| :--- | :--- | :--- | :--- |
| `/bets` | `GET` | List bets (filterable by wallet). | `wallet_address` |
| `/bets/:id` | `GET` | Get specific bet details. | `id` (Numeric) |
| `/bets` | `POST` | Place a bet. | `{ poolId, wallet_address, amount, option }` |

## 4. Payouts API
Tracks distribution of rewards.

| Endpoint | Method | Description | Key Parameters |
| :--- | :--- | :--- | :--- |
| `/payouts` | `GET` | List all payouts (filterable by wallet). | `wallet_address` |
| `/payouts/:id` | `GET` | Get specific payout record. | `id` (Numeric) |

## 5. Scores API
Tracks VAR and match events.

| Endpoint | Method | Description | Key Parameters |
| :--- | :--- | :--- | :--- |
| `/scores` | `GET` | List score events for a fixture. | `fixtureId` |
| `/scores` | `POST` | Internal log of an event. | `{ fixtureId, Action, Data }` |

## 6. Webhooks API
General system webhooks configuration.

| Endpoint | Method | Description | Key Parameters |
| :--- | :--- | :--- | :--- |
| `/webhooks` | `GET` | Get webhook config. | - |
| `/webhooks` | `POST` | Update webhook config. | `{ config: JSON }` |

---

## 7. Real-time Updates

### WebSocket Gateway
`ws://<host>[:port]/socket`
- **Description**: Real-time updates for the frontend.
- **Events Emitted**:
    - `poolUpdated`: Broadcasts latest `amount` and `acceptingBets` status when a new bet is placed or a pool is closed.
    - `payoutExecuted`: Notifies users when a pool is resolved and payouts are assigned.

---

## 8. Background Jobs (Internal)

### Fixture Poller
- **Interval**: Every 5 minutes.
- **Action**: Fetch latest fixtures from TxLine $\rightarrow$ UPSERT into `fixtures` table.

### Score Listener
- **Interval**: Every 1 minute (per active fixture).
- **Action**: Fetch scores from TxLine $\rightarrow$ Check for `"var"` or `"var_end"`.
    - **If "var"**: Create pool record in `pools` table $\rightarrow$ Set `acceptingBets = true`.
    - **If "var_end"**: Mark pool `acceptingBets = false` → Process winner logic → Create `payouts` records → Mark `paidOut = true`.

---

## 9. Simulation Control API
Debug endpoints for resetting and fast-forwarding the mock service clock. See [Simulate Feature](simulate-feature.md) for full design.

| Endpoint | Method | Description |
| :--- | :--- | :--- |
| `/simulate/reset` | `POST` | Wipes DB (bets, payouts, pools, scores) + resets mock clock. Wrapped in a DB transaction. Broadcasts `simulationReset` via WebSocket. |
| `/simulate/fast-forward` | `POST` | Advances mock clock by `{ minutes: N }`. Returns new virtual time. |
