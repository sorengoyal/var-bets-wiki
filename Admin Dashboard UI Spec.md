# Admin Dashboard UI Spec

## Decisions

| Decision | Detail |
|---|---|
| Pool structure | One pool = one VAR incident. Multiple pools per match. |
| Polymarket | One market per match: **"Who will win?"** (standard match winner odds). |
| Hedging | Fully automated. System bets against Polymarket match-winner odds, exploiting shifts caused by VAR decisions. |
| Admin scope | Single platform operator — sees all matches and pools globally. |

---

## Dashboard Layout

```
┌──────────────────────────────────────────────────────────────┐
│  VAR Bets Admin                              [wallet] [⚙️]   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─ LIVE: Argentina vs Egypt ──────────────────────────────┐ │
│  │  ⏱️ 62:34 · FIFA World Cup                              │ │
│  │                                                        │ │
│  │  ┌─ Polymarket: "Who will win? ARG vs EGY?" ───────────┐ │ │
│  │  │                                                    │ │ │
│  │  │  Argentina 38% · Draw 28% · Egypt 34%             │ │ │
│  │  │                                                    │ │ │
│  │  │  Argentina win %                                  │ │ │
│  │  │  45% ┤╮                                           │ │ │
│  │  │  40% ┤╰──╮                 ╭─                     │ │ │
│  │  │  35% ┤   ╰──╮           ╭──╯                     │ │ │
│  │  │  30% ┤      ╰───────────╯                        │ │ │
│  │  │      ├──────┬──────┬──────┬──────┬──────          │ │ │
│  │  │      0     20     40     60     80     90         │ │ │
│  │  │      ↑                    ↑                        │ │ │
│  │  │   kickoff          VAR: Egypt                      │ │ │
│  │  │                    goal disallowed (38')           │ │ │
│  │  └────────────────────────────────────────────────────┘ │ │
│  │                                                        │ │
│  │  ┌─ Pool #1: Penalty VAR (32') ──────────────────────┐ │ │
│  │  │  Status: ● SETTLED · Outcome: CONFIRMED           │ │ │
│  │  │  Volume: 420 USDC · Payouts: 380 USDC · Net: +19 USDC│ │ │
│  │  └────────────────────────────────────────────────────┘ │ │
│  │                                                        │ │
│  │  ┌─ Pool #2: Offside VAR (67') ──────────────────────┐ │ │
│  │  │  Status: ● OPEN · Bets: 23 · Volume: 315 USDC      │ │ │
│  │  │  ┌ Bet Feed ────────────────────────────────────┐ │ │ │
│  │  │  │ 12:34:05  B2x...A7   Overturned   5.0 USDC   │ │ │ │
│  │  │  │ 12:33:51  9kL...F2   Confirmed    2.0 USDC   │ │ │ │
│  │  │  └──────────────────────────────────────────────┘ │ │ │
│  │  │  ┌ Hedge ───────────────────────────────────────┐ │ │ │
│  │  │  │ 12:34:10  BUY Egypt       15 USDC  Poly #45321│ │ │ │
│  │  │  │ 12:33:48  SELL Argentina  8 USDC   Poly #45321│ │ │ │
│  │  │  │ Reason: Hedging Pool #2 VAR Overturn risk    │ │ │ │
│  │  │  └──────────────────────────────────────────────┘ │ │ │
│  │  └────────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─ Upcoming ───────────────────────────────────────────────┐│
│  │  Brazil vs France      Kickoff in 02:15:30               ││
│  │  Germany vs Spain      Kickoff in 05:42:10               ││
│  └──────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌─ Completed ──────────────────────────────────────────────┐│
│  │  Argentina vs Egypt │ 2 pools │ 735 USDC │ Net: +38 USDC  ││
│  └──────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

### Hierarchy

```
Dashboard
 ├── Match Card (LIVE)          ← one at a time
 │    ├── Match Timer
 │    ├── Polymarket Odds Graph  ← shared across all pools in this match
 │    └── Pool Cards[]           ← one per VAR incident
 │         ├── Pool Status
 │         ├── Bet Feed          ← bets on this specific VAR incident
 │         └── Hedge Log         ← hedges against Polymarket for this pool
 ├── Upcoming Matches           ← created, not yet kicked off
 └── Completed Matches          ← all pools settled, match over
      └── Match Summary
           └── Pool Cards[]     ← final state, payouts visible
```

---

## Pool States (per VAR incident)

```
           CREATED ──→ OPEN ──→ CLOSED ──→ SETTLED
              ↑                    ↑           ↑
         pool created        VAR check      payouts
         (match kicks off)   resolves       executed
```

A pool is `CREATED` when the VAR check is announced by the admin. It goes `OPEN` when the first bet is allowed. It goes `CLOSED` when the VAR decision is known. It goes `SETTLED` when payouts are processed on-chain.

**Multiple pools can be OPEN simultaneously** within the same match.

---

## Component Specs

### Match Card (LIVE)

```
┌──────────────────────────────────────────────┐
│  🏟  Argentina vs Egypt                      │
│      FIFA World Cup                          │
│                                              │
│  ⏱️  62:34  ·  Second Half                   │
│                                              │
│  Active Pools: 1 OPEN · 1 SETTLED            │
│  Total Volume: 735 USDC                       │
│  Polymarket: ARG 38% / Draw 28% / EGY 34%   │
└──────────────────────────────────────────────┘
```

Collapsible. Click to expand and show Polymarket graph + all pool cards.

### Polymarket Odds Graph (per match)

Shows "Who will win?" match-winner odds over time. One graph per match, shared across all pools.

- **Lines**: Toggle between Home Win % / Draw % / Away Win % (or show all three)
- **X-axis**: Match time (0' to 90'+)
- **Y-axis**: Polymarket price as percentage
- **Annotations**: Dashed vertical lines at each VAR incident, labeled with type + minute
- **Color coding**: Segment during each VAR pool's OPEN period in a distinct color
- **Current marker**: Vertical dashed line at current match time
- **Hover**: Exact percentages, timestamp, Polymarket volume

### Pool Card (per VAR incident)

**When OPEN:**

```
┌─ Pool #2: Offside VAR ──────────────────────┐
│  VAR Check: Offside · Minute: 67'           │
│  Status: ● OPEN                             │
│                                              │
│  ┌──────────────────────────────────────────┐│
│  │  Market           Bets   Volume   Odds   ││
│  │  ──────────────── ────── ──────── ────── ││
│  │  Overturned       18     240 USDC  0.42   ││
│  │  Confirmed        5      75 USDC   0.58   ││
│  └──────────────────────────────────────────┘│
│                                              │
│  ▼ Bet Feed (23 bets)                        │
│  ┌──────────────────────────────────────────┐│
│  │ 12:34:05  B2x...A7   Overturned  5.0 USDC││
│  │ 12:33:51  9kL...F2   Confirmed   2.0 USDC││
│  │ 12:33:22  C7m...k9M   Overturned  10 USDC││
│  └──────────────────────────────────────────┘│
│                                              │
│  ▼ Hedge Activity                            │
│  ┌──────────────────────────────────────────┐│
│  │ 12:34:10  BUY Egypt        15 USDC       ││
│  │ 12:33:48  SELL Argentina   8 USDC        ││
│  │ Net: +15 USDC EGY / -8 USDC ARG           ││
│  └──────────────────────────────────────────┘│
└──────────────────────────────────────────────┘
```

**When SETTLED:**

```
┌─ Pool #1: Penalty VAR ──────────────────────┐
│  VAR Check: Penalty · Minute: 32'           │
│  Status: ✓ SETTLED · Outcome: CONFIRMED     │
│                                              │
│  ┌──────────────────────────────────────────┐│
│  │  Market         Bets    Pool    Payout   ││
│  │  ────────────── ──────  ──────  ──────── ││
│  │  Overturned     15      250     0 USDC    ││
│  │  Confirmed ✓    10      170     380 USDC  ││
│  │  ─────────────────────────────────────── ││
│  │  Fee: 21 USDC · Hedge P&L: -8 USDC        ││
│  │  Net Profit: +19 USDC                     ││
│  └──────────────────────────────────────────┘│
│  Tx: 4xK9...m2P                             │
└──────────────────────────────────────────────┘
```

### Match-Level Summary (Completed)

```
┌─ Argentina vs Egypt ────────────────────────────────────────┐
│  Final Score: 2-1 · 2 VAR incidents · All settled          │
│                                                             │
│  ┌ Pools ──────────────────────────────────────────────────┐│
│  │  #    VAR Type    Minute  Outcome     P&L               ││
│  │  ───  ──────────  ──────  ──────────  ─────────         ││
│  │  1    Penalty     32'     Confirmed   +19 USDC           ││
│  │  2    Offside     67'     Overturned  +19 USDC           ││
│  │  ────────────────────────────────────────────           ││
│  │  Match Total:                           +38 USDC         ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

---

## WebSocket Events (Backend → Dashboard)

| Event | Payload | Effect |
|---|---|---|
| `match_created` | `{ matchId, teams, kickoffTime }` | New "Upcoming" card |
| `match_live` | `{ matchId }` | Card moves to LIVE zone, timer starts |
| `pool_created` | `{ matchId, poolId, varType, minute }` | New pool card in match |
| `pool_opened` | `{ matchId, poolId }` | Pool card shows OPEN, bets accepted |
| `bet_placed` | `{ matchId, poolId, market, wallet, amount }` | Row in bet feed |
| `odds_update` | `{ matchId, homePct, drawPct, awayPct, timestamp }` | Point on Polymarket graph |
| `hedge_executed` | `{ matchId, poolId, direction, outcome, amount, txHash }` | Row in hedge log |
| `pool_closed` | `{ matchId, poolId, outcome }` | Pool card → SETTLED with result |
| `pool_settled` | `{ matchId, poolId, payouts, profit, txHash }` | Payout data populated |
| `match_completed` | `{ matchId }` | Match card → Completed zone |

---

## Data Models (TypeScript)

```typescript
interface PoolCard {
  id: string;
  matchId: string;
  varType: "penalty" | "offside" | "red_card" | "goal" | string;
  minute: number;
  state: "CREATED" | "OPEN" | "CLOSED" | "SETTLED";
  markets: {
    overturned: { betCount: number; volumeUsdc: number; odds: number };
    confirmed: { betCount: number; volumeUsdc: number; odds: number };
  };
  totalVolumeUsdc: number;
  uniqueBettors: number;
  settlement?: {
    outcome: "overturned" | "confirmed";
    payouts: { overturned: number; confirmed: number };
    platformFeeUsdc: number;
    hedgePnlUsdc: number;
    netProfitUsdc: number;
    settlementTx: string;
  };
}

interface MatchCard {
  id: string;
  homeTeam: string;
  awayTeam: string;
  competition: string;
  kickoffTime: Date;
  state: "UPCOMING" | "LIVE" | "COMPLETED";
  polymarket: {
    marketId: string;
    currentOdds: { homeWin: number; draw: number; awayWin: number };
    priceHistory: { timestamp: Date; homeWin: number; draw: number; awayWin: number }[];
  };
  pools: PoolCard[];
  totalVolumeUsdc: number;
  netProfitUsdc?: number; // set when COMPLETED
}

interface BetEvent {
  id: number;
  matchId: string;
  poolId: string;
  timestamp: Date;
  walletAddress: string;
  marketName: string;
  amountUsdc: number;
}

interface HedgeEvent {
  id: number;
  matchId: string;
  poolId: string;
  timestamp: Date;
  direction: "BUY" | "SELL";
  outcome: string; // e.g. "Egypt Win", "Argentina Win", "Draw"
  amountUsdc: number;
  polymarketId: string;
  txHash: string;
}
```

---

## Edge Cases

| Case | Handling |
|---|---|
| Two VAR checks close together | Both pools show as OPEN. Polymarket graph annotates both. Independent bet feeds. |
| Pool created before match is LIVE | Show pool in "Upcoming" match card, greyed out until kickoff |
| Polymarket API down | Graph shows last data + "Delayed" badge. Hedges pause/queue. System alerts admin. |
| Zero bets on a pool | Pool shows "No bets placed" — no payout calculation needed |
| Hedge fails | Hedge log shows FAILED row in red. Alert badge on pool. Retry mechanism indicator. |
| Match abandoned/postponed | All open pools → CANCELLED state. Bets refunded. Dashboard shows refund tx. |
| No active match | Empty state: "No matches currently live. Next match starts in 02:15:30." |
| Very long bet feed | Virtual scrolling, last 200 rows rendered |
