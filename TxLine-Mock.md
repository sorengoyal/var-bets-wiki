# VAR Bets - TxLine Mock Service

## Overview
The **TxLine Mock Service** is a lightweight TypeScript-based HTTP server designed to simulate the TxLine API. It allows the VAR Bets backend to be developed and tested locally without depending on a live external sports data provider.

## Core Logic
Instead of a real-time feed, the service simulates the passage of time using an internal virtual clock.

- **Virtual Clock**: The service reads a `START_TIME` from the `.env` file. Upon startup, it initializes a clock that increments in real-time.
- **State Mapping**: When an API request is received, the service calculates the current "virtual time" and matches it against the timestamps in the provided JSON data files to determine which event to return.

## Data Architecture
The service is file-driven and expects a specific directory structure:

### 1. Fixtures
- **File**: `fixtures.json`
- **Role**: Contains the master list of all matches, competitions, and IDs.
- **API Mapping**: Used to populate the `/fixtures` endpoint.

### 2. Scores & Events
- **Files**: `scores-<fixtureId>.json` (one file per fixture)
- **Role**: A chronological list of events (e.g., goals, VAR reviews) for a specific match.
- **Content Structure**:
    ```json
    [
      { "timestamp": "2026-07-16T12:00:00Z", "Action": "goal", "Data": { ... } },
      { "timestamp": "2026-07-16T12:01:00Z", "Action": "var", "Data": { "Type": "Goal" } },
      { "timestamp": "2026-07-16T12:03:00Z", "Action": "var_end", "Data": { "Outcome": "Overturned" } }
    ]
    ```
- **API Mapping**: Used to populate the `/scores` endpoint.

## API Schema (Mirrors TxLine)

| Endpoint | Method | Description | Logic |
| :--- | :--- | :--- | :--- |
| `/fixtures` | `GET` | Fetch all match data. | Returns the contents of `fixtures.json`. |
| `/scores?fixtureId=<id>` | `GET` | Fetch events for a match. | Returns events from `scores-<id>.json` where `timestamp` $\le$ `virtual_current_time`. |

## Configuration (`.env`)
| Variable | Description | Example |
| :--- | :--- | :--- |
| `PORT` | Port for the mock server. | `4000` |
| `START_TIME` | The ISO timestamp the virtual clock starts at. | `2026-07-16T10:00:00Z` |
| `DATA_DIR` | Path to the folder containing JSON files. | `./data` |

## Workflow for Testing
1.  Set `START_TIME` in `.env` to a few minutes before a "VAR" event in your JSON files.
2.  Start the Mock Service.
3.  Start the VAR Bets Backend.
4.  The Backend's cron job will poll the Mock Service → The Mock Service will see the virtual time has reached the "VAR" event → Backend triggers the pool creation.

## Simulation Control API
Endpoints to reset and fast-forward the virtual clock.

| Endpoint | Method | Description |
| :--- | :--- | :--- |
| `/simulate/reset` | `POST` | Resets virtual clock to `START_TIME` from `.env`. Returns `{ currentTime }`. |
| `/simulate/fast-forward` | `POST` | Advances virtual clock by `{ minutes: N }`. Returns `{ currentTime }`. |

See [Simulate Feature](simulate-feature.md) for full architecture details.
