# VAR Bets - Database Schema

## Overview
The database is a PostgreSQL instance serving as the state management and event log for the VAR Bets platform. It tracks football fixtures, VAR events, on-chain pool states, and betting activity.

## Tables

### 1. Fixtures & Metadata
Stores football match information and their current active status.

| Table               | Column               | Type                      | Description                                     |
| :------------------ | :------------------- | :------------------------ | :---------------------------------------------- |
| `fixtures`          | `id`                 | Number (auto-incremented) | Primary Key                                     |
|                     | `Timestamp`          | DateTime                  | Event timestamp                                 |
|                     | `StartTime`          | DateTime                  | Match start time                                |
|                     | `Competition`        | String                    | Name of the league/competition                  |
|                     | `CompetitionId`      | String                    | External competition identifier                 |
|                     | `FixtureGroupId`     | String                    | Grouping for related fixtures                   |
|                     | `Participant1Id`     | String                    | Team 1 ID                                       |
|                     | `Participant1`       | String                    | Team 1 Name                                     |
|                     | `Participant2Id`     | String                    | Team 2 ID                                       |
|                     | `Participant2`       | String                    | Team 2 Name                                     |
|                     | `FixtureId`          | String                    | External Fixture ID                             |
|                     | `Participant1IsHome` | Boolean                   | Home/Away indicator                             |
| `fixtures_metadata` | `id`                 | Number (auto-incremented) | Primary Key                                     |
|                     | `fixture_id`         | Number                    | FK $\rightarrow$ `fixtures.id`                  |
|                     | `active`             | Boolean                   | Whether this fixture is currently being tracked |

### 2. Pools & Betting
Tracks the lifecycle of a prediction market created for a specific VAR review.

| Table     | Column           | Type                      | Description                                        |
| :-------- | :--------------- | :------------------------ | :------------------------------------------------- |
| `pools`   | `id`             | Number (auto-incremented) | Primary Key                                        |
|           | `fixture_id`     | Number                    | FK $\\rightarrow$ `fixtures.id`                     |
|           | `amount`         | Numeric                   | Total pool size                                    |
|           | `acceptingBets`  | Boolean                   | True if pool is open for betting                   |
|           | `paidOut`        | Boolean                   | True if the payout has been executed               |
| `bets`    | `id`             | Number (auto-incremented) | Primary Key                                        |
|           | `pool_id`        | Number                    | FK $\rightarrow$ `pools.id`                        |
|           | `wallet_address` | String                    | User's Solana wallet address                       |
|           | `amount`         | Numeric                   | Bet amount                                         |
|           | `option`         | String                    | The bet position (e.g., "Confirmed", "Overturned") |
| `payouts` | `id`             | Number (auto-incremented) | Primary Key                                        |
|           | `poolId`         | Number                    | FK $\rightarrow$ `pools.id`                        |
|           | `wallet_address` | String                    | User's Solana wallet address                       |
|           | `amount`         | Numeric                   | Payout amount                                      |

### 3. Events & Scoring
Logs raw events from the sports API.

| Table               | Column      | Type                      | Description                                                             |
| :------------------ | :---------- | :------------------------ | :---------------------------------------------------------------------- |
| `scores`            | `id`        | Number (auto-incremented) | Primary Key                                                             |
|                     | `fixtureId` | Number                    | FK $\rightarrow$ `fixtures.id`                                          |
|                     | `Action`    | String                    | Event type (e.g., `"var"`, `"var_end"`)                                 |
|                     | `Data`      | JSONB                     | Event details (e.g., `{"Type": "Goal"}` or `{"Outcome": "Overturned"}`) |

### 4. Infrastructure
| Table | Column | Type | Description |
| :--- | :--- | :--- | :--- |
| `webhooks` | `id` | UUID (v7) | Primary Key |
| | `config` | JSONB | Internal system webhook configuration |

## Relationships
- `fixtures` $\leftarrow$ `fixtures_metadata` (1:1)
- `fixtures` $\leftarrow$ `scores` (1:N)
- `fixtures` $\leftarrow$ `pools` (1:N)
- `pools` $\leftarrow$ `bets` (1:N)
- `pools` $\leftarrow$ `payouts` (1:N)

## Common Columns
All entities will have a created_at and updated_at timestamp
