# Implementation Plan - VAR Bets

## Story 1: Infrastructure & Data Foundation
**Goal**: Establish the environment and the data "heartbeat" of the app.
**Acceptance Criteria**:
- [ ] Turbo repository initialized with `pnpm`, Node 24, and TypeScript 6. Git should also be initialized.
- [ ] PostgreSQL instance running via Docker.
- [ ] TxLine Mock Service operational:
    - Loads `fixtures.json` and `scores-<id>.json`. (File contents will be added manually)
    - Virtual clock correctly increments based on `.env` `START_TIME`.
    - `/fixtures` and `/scores` endpoints return data filtered by virtual time.
- [ ] Database schema implemented in the backend including all tables from the Obsidian spec (Numerical IDs, UUIDv7 for pool identifiers).

## Story 2: Backend Engine (The Logic)
**Goal**: Build the automation that creates and resolves markets.
**Acceptance Criteria**:
- [ ] Full set of CRUD APIs for Fixtures, Pools, Bets, and Payouts operational.
- [ ] Fixture Poller cron job syncs data from Mock Service every 5 minutes.
- [ ] Score Listener cron job (1 min) successfully:
    - Detects `"var"` $\rightarrow$ creates a new pool record.
    - Detects `"var_end"` $\rightarrow$ closes pool and initiates payout flow.
- [ ] Payout logic correctly calculates winners based on the `option` and `Outcome` and generates `payouts` records.

## Story 3: Connectivity & Real-time
**Goal**: Enable the "live" communication layer and user identity.
**Acceptance Criteria**:
- [ ] Backend accepts and validates Solana wallet addresses as user identity.
- [ ] WebSocket Gateway implemented and broadcasting:
    - `poolUpdated` when a bet is placed or pool state changes.
    - `payoutExecuted` when a pool is resolved.
- [ ] End-to-end flow verified: Mock Event $\rightarrow$ Backend Cron $\rightarrow$ DB Update $\rightarrow$ WS Broadcast.

## Story 4: Frontend Experience
**Goal**: Create the user-facing betting interface.
**Acceptance Criteria**:
- [ ] Next.js application setup with Solana Wallet Adapter for Phantom.
- [ ] Home Page displays active and resolved pools in real-time.
- [ ] Betting Modal implemented:
    - Allows selection of "Confirmed" vs "Overturned".
    - Quick-select buttons for $1, $10, $100.
    - Successful `POST /bets` call updates the UI.
- [ ] User Dashboard (sidebar) correctly displays "My Active Bets" and "My Payouts" for the connected wallet.
- [ ] UI updates instantly via WebSocket listeners.

## Story 5: Polishing & QA
**Goal**: Ensure stability and edge-case handling.
**Acceptance Criteria**:
- [ ] Full E2E test scenario completed (VAR start $\rightarrow$ Bet $\rightarrow$ VAR end $\rightarrow$ Payout).
- [ ] Input validation implemented (e.g., prevents betting on closed pools or negative amounts).
- [ ] UI/UX refinements completed for responsiveness and accessibility.
