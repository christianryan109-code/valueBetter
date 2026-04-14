# OddsEdge — Project Brief for Claude Code

## What This Project Is

OddsEdge is a sports betting odds aggregator and positive expected value (+EV) detector.
It scrapes live odds from multiple bookmakers simultaneously, calculates a consensus "true"
probability for each event using the mean of the implied probability distribution (after
removing the bookmaker margin/vig), and then highlights any bookmaker offering odds that
imply a better-than-true price — i.e. a +EV opportunity.

The end goal is a public-facing website where users can see live odds comparisons and
flagged +EV bets across multiple sports and markets.

---

## Tech Stack

- **Language:** Python 3.11+
- **Scraping:** `Playwright` (async) for JS-heavy bookmaker sites; `httpx` for direct API endpoints
- **Async orchestration:** `asyncio` — all scrapers run in parallel, not sequentially
- **Database:** `SQLite` for local dev; swap to `PostgreSQL` for production
- **ORM:** `SQLAlchemy` (async)
- **Backend API:** `FastAPI`
- **Task scheduler:** `APScheduler` — re-run scrapers every 3 minutes
- **Frontend:** Plain HTML + CSS + vanilla JS (no framework needed; keep it simple)
- **Deployment target:** Linux VPS (Ubuntu), served via `uvicorn` + `nginx`

Do NOT use Django. Do NOT use React. Keep dependencies minimal.

---

## Project Structure

```
oddsedge/
├── CLAUDE.md                  ← you are here
├── README.md
├── requirements.txt
├── .env.example               ← env vars template (never commit .env)
├── main.py                    ← entry point: starts FastAPI + scheduler
│
├── scrapers/
│   ├── __init__.py
│   ├── base.py                ← abstract BaseScraper class all scrapers inherit from
│   ├── bet365.py
│   ├── paddypower.py
│   ├── boylesports.py
│   ├── betfair.py
│   └── ...                   ← one file per bookmaker
│
├── engine/
│   ├── __init__.py
│   ├── probability.py         ← vig removal, true prob calculation, EV detection
│   └── models.py              ← OddsSnapshot, Event, EVOpportunity dataclasses
│
├── db/
│   ├── __init__.py
│   ├── database.py            ← SQLAlchemy setup, session factory
│   └── crud.py                ← DB read/write functions
│
├── api/
│   ├── __init__.py
│   └── routes.py              ← FastAPI routes: /events, /odds, /ev
│
└── frontend/
    ├── index.html
    ├── style.css
    └── app.js
```

---

## Scraper Architecture

### BaseScraper (scrapers/base.py)

Every bookmaker scraper must inherit from `BaseScraper` and implement:

```python
class BaseScraper:
    name: str              # e.g. "Bet365"
    async def scrape(self) -> list[OddsSnapshot]:
        raise NotImplementedError
```

Each scraper returns a list of `OddsSnapshot` objects. Never return raw dicts.

### Scraper Implementation Notes

- Use `Playwright` with `async_playwright` in headless mode
- Set realistic user-agent strings on every request
- Add random delays between 1.5–3.5 seconds between page actions (use `asyncio.sleep`)
- Most modern bookmakers load odds via internal XHR/fetch calls — inspect Network tab
  first; if an internal JSON API exists, use `httpx` to hit it directly instead of
  parsing HTML. This is far more reliable.
- Each scraper should catch its own exceptions and return an empty list on failure,
  logging the error — never let one broken scraper crash the whole run
- Log every scrape attempt with timestamp, bookmaker name, and count of events returned

### Running Scrapers in Parallel

In main orchestrator, run all scrapers concurrently:

```python
results = await asyncio.gather(*[s.scrape() for s in scrapers], return_exceptions=True)
```

---

## Probability Engine (engine/probability.py)

This is the core maths. Implement these functions:

### 1. `decimal_to_implied_prob(odds: float) -> float`
Convert decimal odds to raw implied probability.
```
implied_prob = 1 / odds
```

### 2. `remove_vig(implied_probs: list[float]) -> list[float]`
Remove the bookmaker's overround so probabilities sum to 1.
```
total = sum(implied_probs)
fair_probs = [p / total for p in implied_probs]
```

### 3. `consensus_true_prob(fair_probs_per_book: list[list[float]]) -> list[float]`
Take the mean across bookmakers for each outcome.
```
true_prob[outcome] = mean([book[outcome] for book in fair_probs_per_book])
```
This is the "true" probability — the mean of the distribution across all books.

### 4. `expected_value(odds: float, true_prob: float) -> float`
Calculate EV for a given bet:
```
ev = (true_prob * (odds - 1)) - (1 - true_prob)
```
Positive EV → flag it. Threshold: ev > 0.03 (i.e. >3% edge) to filter noise.

### 5. `find_ev_opportunities(snapshots: list[OddsSnapshot], true_probs: dict) -> list[EVOpportunity]`
Loop through all bookmaker odds for each event/outcome and flag any where EV > threshold.

---

## Data Models (engine/models.py)

```python
@dataclass
class OddsSnapshot:
    bookmaker: str
    event_id: str          # consistent ID across bookmakers e.g. "Man Utd vs Arsenal 2026-04-14"
    sport: str
    home_team: str
    away_team: str
    kickoff: datetime
    home_odds: float
    draw_odds: float | None   # None for sports with no draw
    away_odds: float
    scraped_at: datetime

@dataclass
class EVOpportunity:
    event_id: str
    bookmaker: str
    outcome: str           # "home" | "draw" | "away"
    bookmaker_odds: float
    true_prob: float
    ev_percentage: float
    detected_at: datetime
```

---

## Database (db/)

### Tables
- `events` — unique events (event_id, sport, teams, kickoff)
- `odds_snapshots` — every scrape result (FK to events)
- `ev_opportunities` — flagged +EV bets (FK to events)

### Rules
- Use async SQLAlchemy sessions
- Upsert odds snapshots — don't create duplicates for same bookmaker/event/scrape window
- Keep last 24 hours of odds history, purge older rows on each scheduler run

---

## API Routes (api/routes.py)

| Route | Method | Returns |
|---|---|---|
| `/api/events` | GET | All active events with consensus true probs |
| `/api/odds/{event_id}` | GET | All bookmaker odds for a specific event |
| `/api/ev` | GET | All current +EV opportunities |
| `/api/sports` | GET | List of available sports |

All responses return JSON. Include `scraped_at` timestamps in every response.

---

## Frontend (frontend/)

Single-page dashboard. No frameworks. Keep it clean.

**Layout:**
- Header: "OddsEdge" logo + last updated timestamp
- Filter bar: sport selector (football, tennis, etc.)
- Main table: one row per event showing:
  - Teams + kickoff time
  - Consensus true odds (converted from true_prob back to decimal)
  - Each bookmaker's current odds (colour-coded: green = above true odds, red = below)
  - EV% badge if >3% edge exists anywhere for that event
- Clicking an event expands it to show full breakdown

**Polling:** `app.js` fetches `/api/ev` every 60 seconds and updates the table without
full page reload.

**Styling:** Dark theme. Use CSS variables for colours. Mobile responsive.

---

## Environment Variables (.env)

```
DATABASE_URL=sqlite+aiosqlite:///./oddsedge.db
SCRAPE_INTERVAL_SECONDS=180
EV_THRESHOLD=0.03
LOG_LEVEL=INFO
```

---

## Coding Conventions

- All async functions — no blocking I/O anywhere
- Type hints on every function signature
- Docstrings on every class and public method
- Use `loguru` for logging (not print statements)
- Use `pydantic` for any data validation at API boundaries
- No hardcoded strings — put bookmaker names, URLs, selectors in config dicts at top of each scraper file
- Every scraper file must have a `if __name__ == "__main__"` block that runs the scraper
  standalone so it can be tested in isolation

---

## Build Order — Do This In This Exact Order

1. **Set up project structure** — create all folders and empty `__init__.py` files
2. **Define data models** — `engine/models.py` first, everything else depends on this
3. **Set up database** — `db/database.py` and `db/crud.py`
4. **Build probability engine** — `engine/probability.py` with unit tests
5. **Build one scraper** — start with `scrapers/boylesports.py` (simpler site)
6. **Build FastAPI skeleton** — `api/routes.py` returning dummy data
7. **Wire up scheduler** — `main.py` connecting scraper → DB → API
8. **Build frontend** — `frontend/index.html` consuming the live API
9. **Add remaining scrapers** one at a time, testing each in isolation

---

## What NOT To Do

- Do NOT build all scrapers at once — build and verify one at a time
- Do NOT use `requests` (it's synchronous) — use `httpx` with async
- Do NOT store odds as strings — always floats
- Do NOT expose raw DB errors to the API — catch and return proper HTTP errors
- Do NOT skip error handling in scrapers — one broken scraper must not stop the others
- Do NOT commit `.env` files

---

## First Task for Claude Code

When starting a new session, begin by saying:
> "I've read CLAUDE.md. Ready to build OddsEdge."

Then ask which step of the Build Order to work on, or proceed with Step 1 if starting fresh.
