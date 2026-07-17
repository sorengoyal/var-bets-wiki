# VAR Bets - Frontend Specification

## Overview
The frontend is a modern web application built with **Next.js**. It provides a real-time interface for users to track VAR reviews, place bets using their Solana wallet, and monitor their payouts.

- **Framework**: Next.js
- **Language**: TypeScript
- **Wallet Integration**: Phantom Wallet (via Solana Wallet Adapter)
- **State Management**: React Hooks + WebSocket for real-time pool updates.
- **Styling**: Tailwind CSS (implied for modern Next.js projects).

---

## 1. Core Pages & Components

### A. Home Page
The central hub showing all current and historical prediction markets.

- **Pools List**: 
    - **Active Pools**: Displays a list of pools currently `acceptingBets = true`.
        - **Details**: Pool size (total amount), time since launch, and fixture details (Teams, Competition).
    - **Resolved Pools**: Displays pools that have been closed.
        - **Details**: Final result (Confirmation/Overturned), resolution status, and payout confirmation.
- **Wallet Connection**: 
    - A "Connect Wallet" button in the header.
    - Once connected, displays the shortened wallet address.

### B. Betting Modal (Pop-up)
Triggered when a user clicks on an active pool.

- **Betting Interface**:
    - **Option Selection**: Toggle between the two bet outcomes (e.g., "Confirmed" vs "Overturned").
    - **Amount Selection**: 
        - Quick-select buttons: `$1`, `$10`, `$100`.
        - Custom amount input field.
- **Action**: "Place Bet" button.
- **Process**:
    1. User clicks "Place Bet".
    2. Frontend sends a request to `POST /api/bets` with `poolId`, `wallet_address`, `amount`, and `option`.
    3. Upon success, the modal closes, and the Home Page triggers a refresh/socket update.

### C. User Dashboard (Sidebar/Section)
Provides personalized data for the connected wallet.

- **My Active Bets**: A list of bets the current wallet has placed in open pools.
- **My Payouts**: A history of payouts received by the wallet, showing the pool ID and the amount credited.

---

## 2. Real-time Integration

### WebSocket Implementation
To ensure a "live" feel during fast-paced VAR reviews, the frontend maintains a WebSocket connection to the NestJS gateway.

- **Listener**: `poolUpdated`
    - **Action**: When received, the frontend updates the specific pool's `amount` and `acceptingBets` status in the UI without a full page reload.
- **Listener**: `payoutExecuted`
    - **Action**: Updates the "My Payouts" section and marks the pool as `paidOut = true` on the Home Page.

---

## 3. User Flow

1.  **Discovery**: User arrives at Home Page $\rightarrow$ Sees a live VAR pool for "Team A vs Team B".
2.  **Authentication**: User clicks "Connect Wallet" $\rightarrow$ Approves connection via Phantom.
3.  **Engagement**: User clicks the pool $\rightarrow$ Selects "Overturned" $\rightarrow$ Selects "$10" $\rightarrow$ Clicks "Place Bet".
4.  **Verification**: Backend updates DB $\rightarrow$ Socket broadcasts `poolUpdated` $\rightarrow$ User sees the pool size increase instantly.
5.  **Resolution**: VAR ends $\rightarrow$ Backend triggers payout $\rightarrow$ Socket broadcasts `payoutExecuted` $\rightarrow$ User sees the payout in their sidebar.

---

## 4. Technical Requirements
- **Runtime**: Node 24, TypeScript 6.
- **Package Manager**: pnpm.
- **Build System**: Turbo.
- **API Client**: Axios or Fetch for REST calls; `socket.io-client` or native WebSockets for real-time data.

## 5. Debug Toolbar (Simulation Controls)
A fixed toolbar in the top-right header, always visible, for hackathon demo purposes.

- **Virtual Clock Display**: Shows current mock service time (e.g., `⏱ 12:03:00`).
- **Fast-Forward Buttons**: `+1m` and `+5m` buttons that call `POST /api/simulate/fast-forward`.
- **Restart Simulation Button**: `🔄 Restart` calls `POST /api/simulate/reset`.
    - On success, the page auto-refreshes (or clears state and re-fetches).
    - WebSocket listener for `simulationReset` triggers refresh on all connected tabs.

See [Simulate Feature](simulate-feature.md) for full architecture details.
