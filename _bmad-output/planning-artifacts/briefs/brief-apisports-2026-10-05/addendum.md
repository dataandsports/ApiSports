---
title: "Addendum: API-Sports Ingestion Container"
status: final
created: 2026-10-05
updated: 2026-10-06
---

# Addendum: API-Sports Ingestion Container

This addendum holds reference detail that the PRD and architecture workflows need but that does not belong in the brief. Decisions are marked with the date they were made. The full decision trail is in `.memlog.md`, in this folder.

## 1. Source API Facts

From `docs/api-sports/` and web research on 2026-10-05.

| | Football v3 | AFL v1 |
|---|---|---|
| Base URL | `https://v3.football.api-sports.io` | (see spec) |
| Spec version | 3.9.3 | 1.2.4 |
| Endpoints | 37 | 16 |
| Auth | `x-apisports-key` header | `x-apisports-key` header |

- **Errors arrive as HTTP 200.** Failures return HTTP 200 with a populated `errors` object, and this includes daily quota exhaustion ("You have reached the request limit for the day"). Cache and raw-layer logic must check the response envelope, not only the HTTP status.
- **Plan limits.** These come from third-party sources, because the official site blocked automated fetches.
  - Free: 100 requests a day *per API*, resetting at 00:00 UTC. Season and endpoint restrictions are unverified.
  - Pro: 7,500 a day ($19/mo).
  - Ultra: 75,000 a day ($29/mo).
  - Mega: 150,000 a day ($39/mo).
  - Per-minute limits: unverified for every plan.
- **Batching and embedding (Football):**
  - `fixtures?id=` embeds events, lineups, statistics and players in one response.
  - `fixtures?ids=` accepts up to 20 fixture IDs. Whether it embeds the same detail is unverified.
  - `injuries` also accepts `ids`, and `sidelined` accepts up to 20 players or coaches.
  - `/predictions` accepts only a single `fixture`, so it costs at least one call per fixture.
- **Call-budget illustration.** Fetching per-fixture detail for one league season of about 380 fixtures costs:
  - about 1,520 calls endpoint by endpoint (4 detail endpoints)
  - about 380 calls with `fixtures?id=`
  - about 19 calls with `ids`, if `ids` embeds the detail
- **Recommended call frequencies** are published per endpoint. Examples: "1 call per day" for reference data, and "1 call per minute (in-progress) or 1 call per day" for fixtures.

### Point-in-time behavior (Football v3 spec)

| Endpoint | Can an earlier state be queried later? | What overwriting loses |
|---|---|---|
| `teams/statistics` | Yes: the `date` parameter returns stats up to a date | Nothing |
| `injuries` | Keyed to fixtures. Unverified whether a pre-match "Questionable" status survives the match | Possibly the pre-match picture |
| `standings` | No date parameter | The table before each match, unless rebuilt from results |
| `predictions` | No date parameter, but past fixtures return the pre-match prediction. User-verified on 2026-10-05: Arsenal's first game of the season shows no games played; the second shows one. | Nothing |
| `odds` (pre-match) | No. Odds are available 1–14 days before a fixture (AFL: 1–7) and keep 7 days of history. They update every 3 hours (AFL: 4 times a day), 10 fixtures per page. | Everything outside the window. **Downtime means permanent loss** |
| `odds/live` | No history. Fixtures appear 5–15 minutes before kickoff and are removed 5–20 minutes after the final whistle. Updates every 5–60 seconds. | Everything not captured live |

## 2. Environment and Constraints

- **Hosting.** A self-owned server runs SQL Server and Airflow/Docker. Infrastructure cost is no longer a constraint.
- **Database.** SQL Server 2025 Standard Developer (64-bit). There is no database size cap, so disk space is the storage limit. It hosts the raw landing zone (cache tables and the queryable model) and, downstream, the cleansed data.
- **Subscriptions.** API-Football is on Ultra: 75,000 requests a day, on an annual subscription (upgraded 2026-10-06). API-AFL is on Pro: 7,500 requests a day.
- **Repository.** Private.
- **Backups.** Scheduled database-level backups, stored off the server. These are out of scope for this project, and the location is still to be decided.
- **Future sources.** The next ingestion project also calls an API. Later sources may be free or open-source sports data libraries, so the pattern must not assume HTTP.

## 3. Landing Zone Design (input for architecture)

### Datasets and the cache

- **Dataset.** A dataset is the pattern's unit. For an API, a dataset is one endpoint.
- **Dataset definition** (user's term). A dataset definition consists of two parts:
  - the provider's API specification (for API-Sports, its OpenAPI spec)
  - the queryable data model that represents the provider's data the way the provider models it
- **Cache.** A custom HTTP cache persisted in database tables is required, so that calls do not burn through API limits. The cache doubles as the raw layer, and its design is meant to be replicated as the raw layer for other ingestion projects.

### Making raw data queryable

Raw JSON is stored as a column in SQL Server 2025. Architecture chooses among the following options, based on performance:
1. views over the cached JSON
2. stored procedures that build persisted tables
3. stored procedures run on top of the views

Indexing for analysis (by team ID and so on) belongs to the downstream cleansed model, which follows the sport's real data model. The ingestion tables do not need it (decided 2026-10-06).

### Retention (decided 2026-10-05)

| Data | Retention mode | Rationale |
|---|---|---|
| Odds (pre-match) | **Append**: keep every call | Tracks line movement; odds cannot be recovered after the 7-day history |
| Standings | Overwrite | Rebuilt per league from fixture results |
| Predictions | Overwrite | Re-fetchable for past games and point-in-time safe (user-verified) |
| Everything else | Overwrite when a refresh is wanted | Faster, and needs less storage |

Overwriting is acceptable as long as "what was known when" can be reconstructed. Point-in-time training sets exclude any data from after a game starts.

### Table layout: one table per endpoint (preferred direction)

**For:**
- Retention mode becomes a property of the table.
- The odds volume, which only ever grows, stays isolated from everything else: indexes, maintenance and query performance.
- Keys and indexes can suit each endpoint.
- Queryable views map one-to-one onto tables.

**Against:**
- Every new endpoint needs DDL: 37 + 16 tables if all endpoints are in scope. Generating the DDL from one documented template reduces this to ceremony, which is cheap when AI agents do the typing.
- Questions that span endpoints (what ran, what failed, how much quota was spent) need a home. The shared fetch log provides one under any layout.

The user proposed typed, indexed parameter columns. These only work cleanly per endpoint, because each endpoint has its own parameters. What they buy depends on the query:

| Query | Do typed parameter columns help? |
|---|---|
| Cache lookup ("is there a fresh response for this exact request?") | Not much. A unique key on a hash of the normalized request is just as fast under either layout. |
| Operational ("everything fetched for league 39, season 2025"; "what is stale") | **Yes.** Typed columns support index seeks; a generic parameter bag does not. |
| Analytical ("every match Arsenal played") | Rarely. These depend on payload contents: a fixture fetched by `date` has no `team` parameter. Entity keys must come from the payload, through computed columns or persisted tables. |
| Downstream incremental ("what changed since the last run") | No. This needs an index on fetch or update time, under either layout. |

**Rule:** parameter columns serve operations; entity columns come from the payload.

### Request identity and normalization

- **Request identity is not entity identity.** One fixture can arrive through many requests: by league and season, by date, by `id`, or batched by `ids`. "Overwrite" therefore applies per request. The queryable model must resolve the latest version of each entity across overlapping requests.
- **Normalization questions for architecture:**
  - Treat `ids=1-2-3` and `ids=3-2-1` as the same request.
  - Decide how multi-ID list parameters are stored.
  - Include `timezone` in request identity, or pin it to UTC, because it changes the response.

### Logging and fetch-log outcomes

All logging, including the fetch log, lives in the shared core (decided 2026-10-06). The fetch log records every retrieval: source, dataset, parameters, outcome, timing and quota headers.

Candidate outcome categories:
- success with data
- **success but empty** (`results: 0`). This needs its own refresh rule. Caching it as final would hide data that arrives later, such as odds before a fixture's window opens, or lineups before they are published.
- API error in the body of a 200 response, such as quota exhausted, plan restriction or invalid parameters
- HTTP error (4xx or 5xx)
- network failure or timeout
- cache hit, with no call made

## 4. Workflows (starting set, agreed 2026-10-06)

| Workflow | Fetches | Cadence |
|---|---|---|
| Reference refresh | Countries, leagues, seasons, teams, venues | Weekly or less |
| Fixtures refresh | Live fixtures and results; full detail for finished fixtures (events, lineups, statistics, players, predictions, injuries); upcoming fixtures | Ideally every minute. **15 minutes is the absolute maximum** before a finished game starts processing. Data is stored and queryable within a further 5 minutes |
| Odds snapshot | Pre-match odds, appending every snapshot | See section 5 |
| Backfill | Past seasons for competitions chosen at run time | On demand; resumes through the cache |

- **Modular design is required.** Workflows combine shared, per-endpoint processing steps with different arguments. For example, the fixtures refresh and the backfill run the same endpoint processing.
- **Scheduling every minute** means 1,440 runs a day. Architecture should compare per-minute container runs with fewer, longer runs that poll every minute internally.

## 5. Odds

- **Phase 1 (decided 2026-10-06).** Collect the closing line only, keeping every call. Regular snapshots come later.
- **Later cadence.** Every 1–3 hours, calibrated against quota; AFL is calibrated the same way. The user wants a cadence faster than the provider's 3-hour cycle. Polling faster than the provider's cycle captures no new values, but it limits staleness: polling every 3 hours can leave the freshest copy nearly 6 hours old.
- **Direction: "never miss a line move."** Store only snapshots that differ from the previous one. The user's stated trigger for this change was a subscription upgrade, and that happened on 2026-10-06. Even so, phase 1 still keeps every call.
- **Calibration experiment (after phase 1).** Record a hash of each odds payload. The fetch log then shows how often consecutive snapshots actually differ. That evidence guides the choice of rate within the 1–3 hour range, and the move to storing only changes.
- **Closing-line capture (to verify):**
  1. After a game ends, fetch `/odds?fixture=<id>` and compare the odds' `update` timestamp with kickoff.
  2. If the odds are frozen at kickoff and still available, collect them within 3 hours after the game goes final. The fixtures refresh that picks up the finished game can fetch them, so no separate workflow is needed.
  3. If the odds update during play, snapshot them within the hour before kickoff. This needs a frequent pre-kickoff sweep (for example every 15–30 minutes) over fixtures kicking off within the next hour.
- **Storage volume.** Volume = fixtures in scope × snapshots per day × payload size. Each payload carries every bookmaker and every bet type, so measure one fixture's payload before raising the cadence. With Ultra, storage (not quota) is the binding constraint on cadence.
- **In-play odds.** Out of scope; a nice-to-have, pending a feasibility study. They keep no history and update every 5–60 seconds, which does not suit batch scheduling.

## 6. Quota and Sizing (all inputs unverified)

- **Current estimates (football on Ultra):**
  - Phase 1 covers every competition for 2 seasons: about 1,100 × 2 × 150, or roughly 330k fixtures. At 1–2 calls per fixture, that is 330k–660k calls, or roughly 5–9 days.
  - Phase 2 covers every season since 2022-01-01: 0.75M–1.5M calls, or roughly 10–20 days.
  - Major competitions alone: about 50 competitions × 2 seasons × 300 fixtures, or roughly 30k fixtures.
  - On Pro, phase 1 was estimated at 2–3 months. That estimate drove the upgrade.
- **Cost drivers:**
  - **Odds.** About (snapshots per day) × (fixtures with open odds ÷ 10) calls a day. To count fixtures with open odds, read `paging.total` from `/odds?date=<day>` for each day in the 14-day window.
  - **Per-fixture endpoints** dominate the backfill. `/predictions` costs one call per fixture. Fixture detail costs one call per fixture with `id`, or one call per 20 fixtures if `ids` embeds the detail.
- **Priority rule (decided 2026-10-06):**
  - While the current and previous seasons fill in, the backfill wins. Odds stay lean, and the odds cadence slows when the backfill needs the quota.
  - Days with fewer games give the backfill more quota.
- **Levers:**
  - Confirm whether `ids` embeds the detail.
  - Process major competitions first.
  - Restrict predictions to selected competitions.
  - Upgrade the plan. The subscription is annual, so any upgrade is a 12-month commitment.
- **Per-minute limit.** Unverified for Ultra. Using 75k calls a day means about 52 calls a minute on average.

## 7. Decision Records (adopted 2026-10-06)

- **Format.** Architectural decisions are recorded as ADRs: Nygard's format plus a MADR-style "options considered" section with pros and cons.
- **Changes.** An accepted ADR is superseded by a new one, never edited.
- **To decide at architecture.** ADRs live either as separate numbered files (for example `docs/adr/NNNN-title.md`) or as entries in the architecture document.
