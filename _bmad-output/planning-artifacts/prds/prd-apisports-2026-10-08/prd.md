---
title: "PRD: API-Sports Ingestion Container"
status: draft
created: 2026-10-08
updated: 2026-10-20
---

# PRD: API-Sports Ingestion Container
*Working title — confirm.*

## 0. Document Purpose

This PRD states what the API-Sports Ingestion Container must do, and why. It is written for three readers:

- **The owner**, who makes and records every product and architecture decision.
- **The downstream BMAD workflows.** These are architecture, epics and stories, and the unattended bmad-loop development sessions that build from those stories. They extract requirements from this document, so every functional requirement carries testable consequences.
- **AI agents, or the owner after months away.** They need the reasoning behind each requirement, not only the requirement.

It builds on the product brief in `_bmad-output/planning-artifacts/briefs/brief-apisports-2026-10-05/`, which holds the brief, its addendum and its decision log. This PRD does not repeat the brief's reference detail. Depth that belongs to architecture goes into this PRD's `addendum.md`, and the decision trail lives in `.memlog.md` beside it.

The document must be shareable at any time, both as a reference BMAD artifact and alongside public analysis built on this data. It therefore contains no secrets, and every simplification is labeled as a choice.

**How it is organized.** The Glossary fixes the vocabulary, and every other section uses its terms verbatim. Each feature gives a behavioral description followed by its functional requirements. These are numbered globally (FR-1 to FR-n), so IDs stay stable if features move. Non-functional requirements that span features have their own section.

**Inline tags.**

- `[ASSUMPTION: …]` marks an inference the owner has not yet confirmed. Every assumption is listed in the Assumptions Index.
- `[SHORTCUT: what is skipped; the professional version; when to revisit]` marks a deliberate simplification, chosen because the full version does not yet earn its complexity.
- `[SOLO EXCLUSION: the practice; why it does not fit a one-person project]` marks a team practice that is deliberately left out. Every exclusion is collected under Non-Goals.
- `[NOTE FOR PM]` marks an unresolved tension or a deferred decision for the owner, who acts as PM.

## 1. Vision

The API-Sports Ingestion Container keeps a complete, faithful copy of API-Football (v3) and API-AFL (v1) data in a self-hosted SQL Server 2025 database, where it feeds personal sports analytics. Airflow runs it on a schedule. Everything runs on owned hardware, so the API subscriptions are the only recurring cost. The pipelines it replaces ran on cloud credits and stopped when the credits ended.

The container treats each kind of data according to what a gap would cost. Fixtures, results, statistics, lineups and predictions can be fetched again at any time, so they are refreshed in place, and a finished game is queryable within minutes of the final whistle. Pre-match odds cannot be fetched again once they leave the provider's seven-day history. Odds are therefore never overwritten, and collection starts with every game's closing line.

The container is also the first instance of a reusable ingestion pattern. A shared core provides one container interface, a raw landing zone that is queryable in SQL, and shared logging. Adapters beneath the core handle each retrieval method, starting with HTTP. Adding an endpoint means adding its dataset definition. Adding a source means adding an adapter only when its retrieval method is new. Every architectural decision is recorded as an ADR, and the system is documented well enough that an AI agent, or the owner returning after months away, can run and extend it from the repository alone.

**Thesis.** The PRD rests on two bets:

1. **Build the machinery once.** Quota protection, raw landing and a predictable container interface are built once, as a documented pattern. Each later source adds definitions, not machinery.
2. **Irrecoverable data sets the reliability bar.** Pre-match odds are the only data that cannot be fetched again, so they get the strictest guarantees. Everything else can tolerate a gap, because a gap can be backfilled.

## 2. Target User

*Pending.*

## 3. Glossary

*Downstream workflows and readers use these terms exactly. The raw landing zone keeps the provider's own names; this PRD and the queryable model use the terms below.*

- **Audit columns** — the two UTC times on every row of every table: `TimeCreatedUtc`, when the row was first stored, and `LastModifiedUtc`, when its content last changed (FR-12).
- **Backfill window** — a date range, kept by hand in configuration and widened by a margin, during which the backfill ranks above refreshes and season statistics (FR-29). The first windows are the international breaks of the FIFA International Match Calendar and one tournament period.
- **Closing line** — a game's pre-match odds as they stood at kickoff: the provider's last pre-match update before the game started. It is pulled 5 to 10 minutes before kickoff, and it stays retrievable after the game from the provider's 7-day history (FR-26).
- **Competition** — a league or a cup. The provider calls every competition a *league*, with a type of `league` or `cup`. This PRD and the queryable model say competition. The raw landing zone keeps the provider's name, `league`.
- **Coverage map** — the provider's statement, for each competition and season, of which datasets it holds. API-Football publishes it in the `leagues` response, and the queryable model of `leagues` exposes it. API-AFL publishes none, and its single competition is treated as fully covered.
- **Coverage phase** — a stage of historical collection, defined by which seasons it collects (§8.2).
- **Daily quota prediction** — the estimate, for each day, of the calls each part of the priority order will need, and of what is left for the backfill (FR-31).
- **Dataset** — the pattern's unit of ingestion. For an API, a dataset is one endpoint.
- **Dataset definition** — the two parts that describe a dataset: the provider's specification of it (for API-Sports, the endpoint in its OpenAPI specification), and the queryable model of its data, shaped the way the provider models it.
- **Day schedule** — every game on one date, in every competition and every status, retrieved in one call: `fixtures?date=` for API-Football and `games?date=` for API-AFL (FR-15). It is kept in the request cache, and it shows which games exist and which have changed (FR-21).
- **Derivable endpoint** — an endpoint whose data can be computed from data the container collects, so it is not collected (FR-23).
- **Endpoint** — one path in a provider's API specification, such as `/fixtures`. API-Football v3 has 38 endpoints and API-AFL v1 has 16.
- **Endpoint register** — the list, derived by a test from the API specifications and the dataset definitions, showing which endpoints have a dataset definition (FR-1).
- **Explained absence** — data that is missing for a recorded reason, such as the coverage map marking its dataset unavailable, or the provider returning an empty response.
- **Fetch log** — the shared record of every retrieval: source, dataset, parameters, outcome, timing and quota headers (§4.9).
- **Game** — one scheduled match between two teams, in one competition and season. API-Football calls it a *fixture* and API-AFL calls it a *game*. This PRD and the queryable model always say game. The raw landing zone keeps the provider's endpoint names, such as `fixtures`.
- **Game processing** — the one workflow that collects a single game's data, whatever the game's stage (FR-11).
- **Game record** — an API-Football game as `fixtures?ids=` returns it, with its events, lineups, statistics and players, stored once per game in the per-record cache (FR-21).
- **Game stage** — a game's place in its lifecycle, derived from the provider's status (FR-6):
  - *pre-game*: scheduled and not yet started;
  - *live*: in progress;
  - *final*: finished.

  No stage follows final. Whether a final game can still change is worked out from the time since it became final and from whether its season has closed.
- **Implementation tier** — a group of endpoints built together. Tiers set the order of development (§8.1).
- **In-play detail** — a live game's team statistics, player statistics and events, collected while the game is in play (FR-19).
- **Kickoff capture** — the retrieval of a game's predictions at its scheduled kickoff. It stores the game's pre-match prediction (FR-17).
- **Live feed** — the provider's list of every game in play, retrieved in one call: `fixtures?live=all` for API-Football, updated every 15 seconds. It carries each game's status and score, and its events where the competition covers them. The container uses it only to know which games are in play, and does not store its game data (FR-21). API-AFL has no live feed (FR-15).
- **Multi-game retrieval** — a retrieval that returns data for many games in one call, such as a day schedule, the live feed, injuries by competition and season, or the game detail of up to 20 games through `fixtures?ids=` (FR-20).
- **Per-record cache** — the cache of records split out of responses to batches of ids, whose composition varies: game records from `fixtures?ids=`, and player and coach records from `sidelined` and `trophies` (FR-21).
- **Queryable model** — the SQL views or tables over the raw landing zone that expose each dataset the way the provider models it, using this Glossary's names for shared concepts such as game and competition.
- **Raw landing zone** — the database area where retrieved data is stored exactly as the provider returned it, under the provider's endpoint names. It holds the request cache and the per-record cache (FR-21).
- **Re-processing mark** — the knowledge that a game, or a competition and season, must be processed again, with the reason and the time it arose. It is kept until processing succeeds, so a needed re-processing is never lost (FR-10). How it is kept is an architecture decision.
- **Request cache** — the cache of responses to requests that repeat with the same parameters, such as each competition's season schedule, the day schedule and injuries by competition and season. It is the brief's cache (FR-21).
- **Restatement** — a change the provider makes to a game's data after the game is final.
- **Restatement window** — the fixed period after a game becomes final during which game processing checks the game again: 15 minutes and 1 hour after it was first seen final, then once a day (FR-8). It lasts 7 days from the time the game was first seen final, and it does not change as the season progresses. A game first seen final more than 3 months after it was played, as the backfill finds most games, is already past its window. It is not a stage.
- **Retrieval** — one attempt to obtain data for one request: either a call to the provider or a cache hit that stands in for one.
- **Season** — one edition of a competition, identified by the provider's season key (a four-digit year). A season is *closed* once every one of its games is final and past its restatement window, or once its last scheduled game is more than 3 months in the past (owner, 2026-10-20; configurable). The second condition also closes a season with games that were never played, such as a season cut short. While a season is open, its games can still be restated. [ASSUMPTION: a season closes as soon as either condition holds.]
- **Top-tier competition** — a competition the owner designates as top tier, by hand, in configuration (FR-3). Its teams get more frequent refreshes of transfers and sidelined data (FR-22).
- **Workflow** — a named unit of work the container runs, such as the reference refresh or game processing (§4.2).

## 4. Features

*Each feature gets a behavioral description, then functional requirements numbered globally, each with testable consequences. FR numbers follow drafting order for now and are renumbered in document order when the PRD is finalized.*

### 4.1 Container interface

**Description.** One container interface serves every source (brief): how a run is started and given its parameters, how secrets are supplied, what exit codes mean and how re-runs behave. It is documented well enough for the integrator to write an Airflow DAG without reading the source, and for an agent new to the project to say how to run each workflow (§4.10).

The container has no continuous mode (owner, 2026-10-20). Every workflow runs as scheduled runs that do their work and exit. Work on a clock finer than a minute, the live feed every 15 seconds (FR-15) with its in-play cycle (FR-19), runs as consecutive runs, never two at once: each starts as soon as the previous one ends, or 15 seconds after the previous one started, whichever is later. Airflow can enforce one run at a time and start the next as soon as the previous ends; addendum §15 has the details and the caveats. The owner starts some runs by hand: backfill runs (FR-28), re-processing requests (FR-13) and the timing validation (FR-25).

**Functional Requirements:**

#### FR-32: Runs, parameters, secrets and configuration

**Consequences (testable):**

- Two runs of the same workflow with the same parameters never overlap, because they would race to write the same data. The live-feed workflow is the main case. Runs with different parameters may overlap, such as backfill runs for different seasons (owner, 2026-10-20). All overlapping runs coordinate through the database, so that no game is processed twice at the same time (FR-11) and every run respects the same reserve (FR-30).
- The live-feed workflow starts a new run as soon as the previous one ends, or 15 seconds after the previous one started, whichever is later. The actual spacing is recorded (FR-19), so the cadence Airflow sustains is measured, not assumed.
- A run takes its parameters when it starts: the workflow, and for runs started by hand, the competitions, seasons or games concerned.
- Secrets, such as the API keys, come from environment variables or Docker secrets, never from the image, the repository, a document or a log line.
- Configuration kept by hand lives in a file in the repository, mounted into the container (OQ-5). That covers competition ranks, the top-tier list, per-competition rules, backfill windows and their margin, and cadences. A change is versioned and reviewable, and needs no image rebuild.
- Exit codes distinguish at least success, a retryable failure (such as quota or network) and a non-retryable failure (such as bad configuration), as the brief requires. The exact codes, how gaps and errors are tracked, and how needed one-off runs are started are decided in architecture and implementation (owner, 2026-10-20).

### 4.2 Workflows

Workflows are composed from shared per-dataset steps. The set: the stage watch (FR-15); game processing (FR-11), which handles one game in any stage; the reference refresh, which also collects season-level data (FR-22); the odds snapshot (FR-26, FR-27); and the backfill (FR-28). The stage watch and the other workflows trigger game processing, and so does the owner.

**What game processing collects at each stage.** Pre-match odds follow their own workflow, the odds snapshot (FR-26, FR-27).

| Stage | Collected | When |
|---|---|---|
| Pre-game | Lineups (FR-16), predictions (FR-17), injuries (FR-18) and pre-match odds (FR-26, FR-27) | Lineups from 40 minutes before kickoff. Predictions daily, after related games finish, and at kickoff. Injuries every 2 hours from 48 hours before kickoff. Odds by date: every hour on the day and the day before, every 2 hours 2 days ahead, every 6 hours 3 and 4 days ahead; and 5 to 10 minutes before kickoff |
| Live | In-play detail through `fixtures?ids=`, triggered by each live-feed retrieval (FR-19) | Every 60 seconds for each live game while the backfill runs, then every 15 seconds |
| Final | Every per-game dataset, with injuries taken from a multi-game retrieval (FR-18) | As soon as the game leaves the live feed (FR-7, FR-15) |
| Final, during the restatement window | Every per-game dataset except predictions (FR-8) | 15 minutes and 1 hour after final, then once a day for 7 days |

**Functional Requirements:**

#### FR-11: One game-processing workflow

One workflow processes a single game, whatever its stage. It collects every dataset that the game's stage and its competition's coverage call for, or a narrower set when its trigger asks for one, such as predictions only (FR-14). Games due at the same time can be processed together, so that they share multi-game retrievals (FR-20). All of these run through it:

- pre-game, live and final processing;
- restatement checks (FR-8);
- re-processing marks (FR-10);
- owner requests (FR-13);
- the prediction cascade (FR-14).

**Consequences (testable):**

- Every retrieval made for a single game goes through game processing, and through no other path. Where a multi-game retrieval already holds a game's data, game processing uses it instead of calling again (FR-20).
- Processing a game again when the provider's data has not changed leaves the stored data unchanged, including `LastModifiedUtc` (FR-12).
- When game processing is triggered for a game that is already being processed, the game's stored data stays consistent, and the game is not processed twice at the same time.

#### FR-15: Stage watch

The stage watch follows every game's stage through two multi-game retrievals, and it starts game processing when a game's stage changes:

- **The live feed, every 15 seconds,** the provider's update rate. A game that leaves the live feed has stopped play. Each live-feed retrieval is followed by one `fixtures?ids=` retrieval, in calls of up to 20 games, for the games that just left the feed and the live games due for in-play detail (FR-19). The feed's own game data is not stored (FR-21).
- **The day schedule, every 5 minutes.** It catches what the live feed cannot show: a game that finishes without ever appearing in the live feed, and changes to games not yet started, such as a new kickoff time, a postponement or a cancellation close to kickoff. A game never seen live is therefore seen final within about 5 minutes, inside the 15-minute limit (§5, item 4). One call covers every game of the day, so this costs about 290 calls a day. The day schedule is kept in the request cache. A change it shows to a game that is not live triggers a retrieval of the game record (FR-21).

**Consequences (testable):**

- When a game leaves the live feed, game processing retrieves the game in the `fixtures?ids=` retrieval that follows. It processes the game as final only if the game's own status says it finished. A game that left for another reason, such as an abandonment, takes the stage its status maps to (FR-6). [ASSUMPTION: the game's own status confirms that it finished before final processing starts, because abandoned and cancelled games also leave the live feed.]
- A game that the day schedule shows final, but that was never seen in the live feed, is processed as final in the same way. That it was never seen live is recorded.
- A game postponed or cancelled before kickoff is seen within about 5 minutes. Its pending pre-game work stops, and the lineups and kickoff capture it will never have are recorded as explained absences, with the provider's status as the reason.
- A failed retrieval of the live feed is never read as "no games in play". That includes an error reported inside a successful HTTP response.
- A game that reappears in the live feed after it was processed as final is live again, and it is processed as final again when it leaves.
- Both cadences can be changed without changing code.
- [ASSUMPTION: API-AFL has no live feed, because its `games` endpoint has no `live` parameter, and its data updates every 30 seconds. Its day schedule, retrieved every 30 seconds while any game is due to be in play, takes the live feed's role.]

The live feed runs as consecutive scheduled runs, one at a time, with no continuous mode (owner, 2026-10-20; §4.1, FR-32). Airflow is not built for sub-minute work, so architecture confirms the spacing it can sustain. One option is for each run to perform several 15-second cycles before it exits, to spread the start-up cost (addendum §15).

#### FR-16: Lineups before kickoff

For each game whose competition covers lineups, checks for lineups start 40 minutes before kickoff, the earliest the provider publishes them, and they grow more frequent as kickoff nears. They stop once both teams' lineups are stored, or when the game goes live. Final processing collects the lineups again (FR-7), so a change made after the last check is still stored.

**Consequences (testable):**

- No lineup check is made more than 40 minutes before the game's latest known kickoff time.
- The check schedule can be changed without changing code.
- A game that goes live without stored lineups is recorded, with the result of each check.

#### FR-17: Predictions before kickoff

For each game whose competition covers predictions, predictions are stored before kickoff and kept current until then. The provider updates predictions every hour, and each finished game can change the predictions of games still to come. Three triggers refresh a game's predictions:

- **Daily,** for every game due to kick off within the next 7 days. A daily refresh of every future game in every season would need more calls than the quota allows (addendum §9).
- **After a game in the same competition and season becomes final,** for every game due to kick off within the next 48 hours, once the provider has had time to absorb the result. The provider updates predictions hourly, at times it does not publish, so the wait is set from measurement rather than guessed.
- **At scheduled kickoff:** the kickoff capture, which stores the game's pre-match prediction.

**Consequences (testable):**

- Every game whose competition covers predictions has predictions stored before kickoff. A game without them is recorded, with the reason.
- The kickoff capture is taken at the scheduled kickoff time, and its retrieval time is recorded.
- Apart from the kickoff capture, a game's predictions are retrieved at most once an hour, whatever triggers the retrieval. [ASSUMPTION]
- The stored data shows how long after a game becomes final the predictions of related games change, and when in the hour the provider updates predictions. The wait, and the timing of refreshes, can be set from that evidence, so that each refresh lands just after a provider update.
- The horizon, the window and the wait can each be changed without changing code.
- Once a game has kicked off, its predictions are never retrieved again, except to fill a missing kickoff capture. A kickoff capture is never overwritten.
- Final processing retrieves predictions only for a game that has no kickoff capture. Past games' predictions show the pre-match state (owner-verified 2026-10-05), so this fallback stores the same prediction. [ASSUMPTION]

#### FR-18: Injuries by competition and season

Injuries are needed before kickoff, because they may inform pre-game analytics. They are retrieved by competition and season, so one call returns the injuries of every game in the season. Injuries are their own endpoint, never part of a game record: each competition-and-season response is kept in the request cache (FR-21). Before kickoff, the injuries of every competition with a game due within the next 48 hours, the same window as predictions (FR-17), are refreshed every 2 hours. The provider updates injuries every 4 hours, so a 2-hour cadence halves how long the stored injuries can lag an update the provider makes just after a retrieval. Final processing and restatement checks keep them current after the game. The owner's tests show that a game's injury records stay retrievable after the game, including a pre-match "Questionable" status, and that a whole season needs no pagination (OQ-6, addendum §9). Injuries can therefore be fetched again like the rest of the game data, and they stay in tier 4.

**Consequences (testable):**

- Every game whose competition covers injuries has its injuries stored before kickoff, from a retrieval made within the 2 hours before kickoff. A game without them is recorded, with the reason.
- No injuries call is made for a single game when a competition-and-season retrieval covers it.
- Each injury record is linked to its game through the game id in the provider's response.
- Final processing and restatement checks take a game's injuries from a competition-and-season retrieval. One retrieval serves every game of that competition and season processed while it is fresh (§4.5).
- A change to a game's injury records after the game, such as an edited status, is detected like any other restatement (FR-8, FR-9).

#### FR-19: In-play detail

While games are live, each live-feed retrieval triggers a `fixtures?ids=` retrieval, in calls of up to 20 games, for the games that just left the feed (FR-15) and the live games due for in-play detail. The functional target is every live game in every cycle, so that its detail is refreshed every 15 seconds. While the backfill runs, until phase 2 is complete, the rate starts at 60 seconds instead, freeing calls for the backfill (owner, 2026-10-20). Otherwise the rate may be lowered for technical reasons only, such as a cycle taking longer than 15 seconds to process, or more simultaneous calls than the provider's rate limits allow. Because every in-play retrieval follows a live-feed retrieval in one cycle, no two schedules compete to refresh the same game. In-play detail has no fixed share of the quota; its place in the priority order decides when it gives way (FR-29). Final processing collects everything again, so in-play detail is never the only copy.

**Consequences (testable):**

- Each live game whose competition covers any in-play dataset is included in a `fixtures?ids=` retrieval at the current in-play rate, taken from the latest live feed.
- Each cycle's duration, and the calls it makes, are recorded, so that a cycle running past 15 seconds is visible. Cycles never overlap: a cycle that runs long delays the next one.
- A game that just left the live feed is included in the next `fixtures?ids=` retrieval, whenever it was last refreshed.
- When calls run short, in-play detail gives way only to the work above it in the priority order (FR-29).
- When calls run short, higher-ranked competitions keep their in-play rate longest (FR-3). [ASSUMPTION]
- The rate can be changed without changing code. A lowered rate is recorded with its reason; after the backfill, the target is 15 seconds.

In-play detail serves whatever reads the data during play. Each refresh overwrites the one before (§4.7), so nothing of it remains once the game is final. It costs about 11,000 to 17,000 calls a day at 15 seconds, about 15 to 22% of the quota, and about 3,000 to 4,000 at 60 seconds (addendum §9). The provider updates statistics and player statistics once a minute, so most refreshes return the same statistics; the events and the score are what a 15-second rate keeps fresher.

#### FR-22: Season-level and reference data

Data that is not per-game follows the rules below, which the owner set on 2026-10-17. Each rule can be changed without changing code. Teams in top-tier competitions (§3) get some refreshes more often. API-AFL follows the same principles.

| Endpoint | Rule | Why |
|---|---|---|
| `standings` | When a game in the competition turns final. [ASSUMPTION: again after a restatement in the competition.] | Most standings could be calculated from results, but competitions have their own rules, and some change them between seasons. The Belgian league, for example, split its season in two for many years, halving points for the second phase and running play-off groups with different rules. Keeping such rules current for every competition is not sustainable. The provider offers no live standings; any live standings are calculated outside this project (FR-23) |
| `teams/statistics`, `players` (season statistics) | After each game ends, for both teams, by competition, season and team, when the quota allows (§4.6) | Low priority. Calculations from game data can be checked against these values. Some team statistics cannot be built by adding up player statistics |
| `teams` | Daily for each competition and season; configurable per competition | A team can move between competitions during a season, for example from Champions League qualifying to the Europa League |
| `players/squads` | Weekly for each team. Also when `transfers` shows a new transfer for the team, and when a game's lineup holds a player missing from the team's stored squad | |
| `transfers` | Daily for each team in a top-tier competition, weekly for every other team. Also when a game's lineup holds a player missing from the team's stored squad; a triggered call is made by team, like the scheduled ones | One call per team. Each result decides whether the team's squad must be retrieved again |
| `sidelined` | Daily for the players and coaches of top-tier teams, weekly for the rest, matching `transfers` | Up to 20 players or coaches per call, named by id and batched across teams. Each entry names its player or coach (owner-tested), so the records go to the per-record cache (FR-21) |
| `trophies` | *Open.* Likely once at the end of each season. [ASSUMPTION: when a competition's season closes (§3), for every player and coach who appeared in it.] Not part of the backfill: collected in one pass over every stored player and coach once the backfill is complete (§8.2). A season that is already closed when the backfill loads it does not trigger this | Among the lowest priorities. A season-end rule fits best if the endpoint holds only team honours, not individual awards such as best player or the Ballon d'Or; a test call shows which |
| `coachs` | One call per team about 2 days before each of its games. Also when a game's lineup names a coach who is not the team's current coach in the stored coaches data | To know each team's coach going into each game. Lineup coach data is not reliable enough: a lineup can give a coach the id 0 (owner's example: fixture 1490541, Philadelphia Union), while `coachs` gives the real id and the dates the coach was with the team |
| `players/profiles` | Once a season | Name, birth date and nationality never change. Height and weight can change between seasons, so dated versions are kept (FR-24). Position and shirt number come with each game |
| `venues` | Once a season, by country | One call returns every venue in a country, with its capacity; `teams` returns only part of a venue's data. Dated versions are kept (FR-24) |
| `leagues`, `countries`, `teams/countries`, `timezone` | Daily | One call each. `teams/countries` lists different countries from `countries` |
| `odds/bookmakers`, `odds/bets` | Daily | Reference lists for odds; one call each |
| Season schedules (`fixtures` by competition and season) | [ASSUMPTION: daily for each competition in season.] | The day schedule covers the current day every 5 minutes (FR-15) |

The timing of the pulls made after a game ends is validated by FR-25 before it is fixed.

**Consequences (testable):**

- A new transfer involving a team triggers a retrieval of that team's squad.
- A player in a game's lineup who is missing from the team's stored squad triggers retrievals of that team's squad and transfers.
- The `teams` rule can be set per competition, and the top-tier designation changed by hand, without changing code.
- Every retrieval under these rules is recorded in the fetch log with the rule or trigger that caused it.
- When the quota budget leaves no room for season statistics, they are skipped and the skip is recorded.

#### FR-25: Validating the provider's update timing

The provider's stated update rates, such as "every hour" for standings, are read as guidance on how often to check, not as the moment its data changes (owner, 2026-10-17). The timing of the pulls made after a game ends is therefore validated before it is fixed: for each endpoint concerned, the container measures how long after a game's final the provider's data reflects the game. Until the results are in, the rules in FR-22 apply. Predictions are measured the same way (FR-17).

**Consequences (testable):**

- For a sample of games across competitions, standings, team statistics and player statistics are retrieved again at set intervals after each game's final, and each retrieval records whether the content changed (FR-12).
- The results give, for each endpoint, the delay after which its data reflects a finished game.
- The validation can be run again if the provider's behaviour seems to change, and its results are kept with the fetch log.
- The pull timing for each endpoint can be set from the results without changing code.

#### FR-28: Backfill

The backfill collects past games and seasons, in the order of the coverage phases (§8.2), with the same processing steps as live collection. Phase 1 runs in two passes: first every phase-1 competition's games and statistics, then their predictions, each in priority order (owner, 2026-10-20). Predictions are about two-thirds of phase 1's calls (addendum §13).

**Consequences (testable):**

- The owner can start a backfill run by hand, naming competitions and seasons, as the brief describes (owner, 2026-10-20). [ASSUMPTION: a run started without names takes the next work in phase order.]
- A run that stops partway, for example when the quota runs out, resumes where it left off: data still fresh in the caches is not retrieved again (FR-21).
- Progress is recorded per competition and season, and per pass, so it is clear what each phase still lacks.
- A game first seen final more than 3 months after it was played gets no restatement checks (FR-8) and is not measured against the arrival targets (FR-7). A season already closed when it is loaded does not trigger trophies (FR-22).
- Not backfilled: pre-match odds beyond the provider's 7-day history; a team's registered squad on a past date, because `players/squads` returns only the current squad (squads at game level come with each game's lineups); and the dated versions of profiles and venues, which start when collection starts. Trophies come after the backfill (§8.2).

[NOTE FOR PM: whether the backfill also starts itself, working through the phases whenever the quota budget leaves room, is deferred (OQ-11). The owner generally prefers automation.]

### 4.3 Coverage

**Description.** The container collects all the data both APIs offer: every endpoint of API-Football v3 (38) and API-AFL v1 (16), for every competition and season the provider offers. The exceptions are derivable endpoints, whose data can be computed from data already collected (FR-23). In-play odds (`odds/live`, `odds/live/bets`) come last, after a feasibility study (§8.1).

Three things bound the work at any moment:

- The provider's **coverage map** says which datasets exist for each competition and season. The container collects only those, and records the rest as explained absences.
- The **implementation tiers** (§8.1) set the order in which endpoints are built.
- The **coverage phases** (§8.2) set the order in which seasons are collected. Within each phase, competitions are taken in priority order.

When the coverage map changes, for example when a past season gains player statistics, the competition and season are marked for re-processing (§4.12).

**Functional Requirements:**

#### FR-1: Derived endpoint register

A test derives the endpoint register from the two API specifications and the dataset definitions in the repository. Nothing about endpoint status is maintained by hand. The register is the endpoint list that the brief's "queryable model" success check uses. The coverage map in the `leagues` model cannot play this role, because it describes the provider's data, not what this system has built. The options considered are in the addendum, §7.

**Consequences (testable):**

- The test lists every endpoint in either specification and shows, for each one, whether a dataset definition covers it, either directly or by carrying it.
- A dataset definition that carries another endpoint's data declares that endpoint, and the carried endpoint counts as covered.
- The test fails when a dataset definition names an endpoint that is in neither specification.
- A derivable endpoint (FR-23) is listed as derivable, with the data it is derived from. It is not reported as missing.
- Endpoints without a definition are reported, not failed, so that tiers still to be built do not break the build. Implementation tiers stay in §8.1.
- After a specification file is updated, any new endpoint appears in the report as having no definition.

#### FR-2: Every endpoint's data is landed

For every implemented endpoint, the endpoint's data is landed in the raw landing zone and exposed in the queryable model. An endpoint whose data arrives in full inside another endpoint's response is covered by that response and is not called separately. For example, the game detail embedded in `fixtures?id=` and `fixtures?ids=` would cover `fixtures/events`, `fixtures/lineups`, `fixtures/statistics` and `fixtures/players`, if OQ-7 confirms that it holds everything they return.

**Consequences (testable):**

- Each implemented endpoint's data can be queried in the queryable model, whether the endpoint was called directly or its data was carried by another response.
- An endpoint is marked as carried only after a recorded comparison shows that, for a sample of games from several competitions, the carrying response holds everything a direct call returns.

#### FR-23: Derivable endpoints

The owner can mark an endpoint as derivable: its data can be computed from data the container already collects, so it is not collected. This narrows the 2026-10-08 decision that every endpoint is collected (§4.3). Marked derivable on 2026-10-17:

- `fixtures/rounds`: the distinct rounds (`league.round`) in the schedules and game records;
- `leagues/seasons`: the seasons listed under each competition in `leagues`;
- `players/topscorers`, `players/topassists`, `players/topyellowcards` and `players/topredcards`: from player statistics;
- `fixtures/headtohead`: every game between two teams, which the stored games hold once the seasons are backfilled;
- `players/seasons` and `teams/seasons`: lists of seasons, available from `leagues` and the collected data;
- `players/teams` (player careers): career data covers only what the provider covers, with nothing before 2008 and nothing from uncovered competitions, so it adds nothing to the collected player statistics and squads.

Team and player season statistics stay collected (FR-22), so that calculations from game data can be checked against the provider's values. Computing derivable data, including live standings, happens outside this ingestion project (owner, 2026-10-17; §7.1). Ingestion supplies only the data it is computed from.

**Consequences (testable):**

- The endpoint register (FR-1) lists each derivable endpoint as derivable, with the data it is derived from.
- No call is made to a derivable endpoint, except to check a derivation against the provider.

#### FR-3: Every competition and season, in priority order

The container collects every competition the provider offers, for every season in the active coverage phase (§8.2). Within a phase, competitions are collected in priority order, major competitions first.

**Consequences (testable):**

- Every competition has a priority rank, and collection within a phase follows rank order.
- The owner can change ranks between runs without changing code. Ranks live in the configuration file in the repository (FR-32, OQ-5).
- A competition the provider adds is picked up by the next reference refresh. It ranks after every ranked competition until the owner ranks it.
- The owner designates the top-tier competitions (§3) by hand, in configuration that changes without changing code.

#### FR-4: Collect only what the provider covers

The coverage map is exposed through the queryable model of the `leagues` endpoint. Workflows read it there, and collect only the datasets it marks as available.

**Consequences (testable):**

- No call is made for a dataset that the coverage map marks unavailable for that competition and season.
- Each dataset skipped this way is recorded as an explained absence, queryable by competition and season.
- A dataset the coverage map says nothing about, such as `transfers`, is collected. An empty response is recorded as an explained absence.
- API-AFL's single competition is treated as fully covered, so every API-AFL dataset is collected. Coverage tracking for API-AFL is added only if a need appears.

#### FR-5: Coverage changes trigger re-processing

When the coverage map changes for a competition and season, the competition and season get a re-processing mark (FR-10). Re-processing collects every dataset the coverage map now marks available.

**Consequences (testable):**

- A change to a competition and season's coverage moves the `LastModifiedUtc` of its coverage record (FR-12), and the mark is set by the run that stored the change.
- A dataset that becomes available for a past season is collected for that season when the mark is processed.
- When the provider withdraws coverage, data already stored is kept. The change is visible through the coverage record's `LastModifiedUtc`.

### 4.4 Dataset definitions and adapters

*Pending.* What a dataset definition contains. Adding an endpoint changes only its definition; a new source adds an adapter only when its retrieval method is new.

### 4.5 Retrieval and request cache

*Pending.* Request identity, freshness rules, error detection from the response envelope, pagination, rate limits.

**Functional Requirements:**

#### FR-20: Multi-game retrievals first

Where an endpoint can return many games' data in one call, the container retrieves the data that way instead of game by game. For API-Football, the multi-game retrievals are:

- the day schedule;
- the live feed, which shows which games are in play (it also carries events where covered, owner-tested 2026-10-11, but the container takes events from `fixtures?ids=`);
- each competition's season schedule;
- injuries by competition and season, a whole season in one call (owner-tested 2026-10-11);
- `fixtures?ids=`, which carries the full game detail of up to 20 games (owner-tested 2026-10-11; OQ-2).

The game-detail endpoints and `predictions` take one game per call (owner-tested 2026-10-10). API-AFL's game-detail endpoints accept a date or up to 10 game ids.

**Consequences (testable):**

- No single-game call is made for data that a multi-game retrieval returned within its freshness rule.
- When several games need their detail at the same time, such as live games each minute or games that leave the live feed in the same retrieval, the detail is retrieved with `fixtures?ids=`, up to 20 games per call.
- Data from a multi-game retrieval can be queried per game, like data from a single-game call.

#### FR-21: Two caches: by request and by record

The container keeps two caches, and whether a request repeats decides which one it uses. Both are part of the raw landing zone. Both store data exactly as the provider returned it, and overwrite it only when its content changes (FR-12).

- **The request cache** holds the response to each request that repeats with the same parameters. It covers each competition's season schedule (`fixtures` by competition and season), the day schedule, injuries by competition and season (FR-18), and every other endpoint. This is the brief's cache. Freshness is judged per request.
- **The per-record cache** holds records split out of responses to batches of ids. Such batches are made up of whichever ids are due, so the same request rarely repeats, and a cache keyed by request would almost never hit. `fixtures?ids=` responses are split by game, and each game has one game record. `sidelined` and `trophies` responses, which name up to 20 players or coaches by id, are split by person in the same way (FR-22). Freshness is judged per record, from the latest retrieval that covered it.

A game gets its game record the first time something needs its detail: a lineup check (FR-16), the live cycle (FR-19), final processing (FR-7), a restatement check (FR-8) or the backfill. Each entry in a new schedule response is compared with the latest stored schedule entry for the same game, from either schedule, so both sides have the same shape. When an entry has changed, and the game has a game record and is not live, the record is retrieved again through `fixtures?ids=`. How the queryable model combines the cached schedules and the game records is left to the model's design (§10, OQ-10).

The live feed is in neither cache. It shows which games are in play, and each live-feed retrieval triggers the `fixtures?ids=` retrieval for the live games due for in-play detail and the games that just left the feed (FR-15, FR-19). Injuries are their own endpoint and are never part of a game record. API-AFL keeps its separation by endpoint, and the same rule decides which cache each of its requests uses. The per-record cache adds to the brief's request cache; it does not replace it (addendum §10).

[NOTE FOR PM: the handling of games missing from a response, and the game list in the fetch log, are the assistant's additions, pending confirmation.]

**Consequences (testable):**

- An API-Football game has at most one game record.
- Which other games share a batch, and their order, never changes a game's record.
- Retrieving the same content again, through either cache, leaves the stored data unchanged, including `LastModifiedUtc`.
- When a schedule response shows a change to a game that has a game record and is not live, compared with the latest stored schedule entry for that game, the game gets a re-processing mark (FR-10), so its record is retrieved again. A game postponed after its lineups were retrieved, for example, never keeps its old status.
- A game record from an older retrieval never overwrites one from a newer retrieval (§5, item 5).
- The fetch log records each retrieval's envelope and the games it returned, so every cached response and game record can be traced to the retrieval that produced it (§5, item 6).
- A game requested in a batch but missing from the response is recorded as missing. It is not treated as fresh.
- A run repeated after a failure skips every request and every game whose cached data is still fresh.

### 4.6 Quota budget

**Description.** Each API has one daily quota, shared by every workflow: 75,000 calls for API-Football and 7,500 for API-AFL. Collection takes about 42,000 to 63,000 football calls a day before the backfill, depending on the in-play rate, and the backfill takes what is left (addendum §14). When calls run short, work gives way in a fixed order. The order follows what a cut would lose: only the odds snapshots are lost for good, because the provider cannot return an earlier state of a game's odds. Everything else is only delayed or less fresh, and the backfill can wait longest. API-AFL follows the same order.

**Functional Requirements:**

#### FR-29: Priority order

When calls run short, work gives way in this order, the last first (owner, 2026-10-20):

1. Odds: the final pull before kickoff, the pulls of yesterday's games and `odds/mapping` (FR-26), and the schedule by date (FR-27).
2. The stage watch: live feed and day schedule (FR-15).
3. Final processing and the early checks (FR-7, FR-8).
4. Pre-game captures: lineup checks, prediction kickoff captures and pre-game injuries (FR-16 to FR-18).
5. In-play detail (FR-19).
6. Refreshes: predictions (FR-14, FR-17), the reference refresh with season-level data (FR-22), and daily restatement checks (FR-8).
7. Season statistics (FR-22).
8. The backfill (FR-28).

This replaces the brief's rule that the backfill wins while phase 1 fills in: odds snapshots cannot wait, and the backfill can. During a backfill window (§3), the backfill rises above refreshes and season statistics, staying below in-play detail (owner, 2026-10-20). Top-tier club competitions pause for international breaks, so collection needs far fewer calls, and national-team games take relatively few. The windows are date ranges kept by hand in the configuration file (FR-32). They start as the international breaks of the FIFA International Match Calendar, plus one tournament period, all dates inclusive:

- 2026-11-09 to 2026-11-17;
- 2027-03-22 to 2027-03-30;
- 2027-06-07 to 2027-06-15;
- 2027-06-19 to 2027-07-11: not a FIFA window, but the CONCACAF Gold Cup and the 2027 Africa Cup of Nations heavily reduce CONCACAF and CAF club games while UEFA's major competitions are on their summer break;
- 2027-09-20 to 2027-10-05.

A margin widens every window by 1 day on each side (owner, 2026-10-20). The reason: the owner expects the day or two around a break to have fewer club games as well. If the daily quota prediction or actual use shows no such drop on margin days (FR-31), the margin is reduced or removed.

**Consequences (testable):**

- When calls run short, no work stops while work below it in the order is still running.
- Each stop is recorded with its reason, and the work resumes once calls are available again.
- On a date inside a backfill window or its margin, the backfill ranks above refreshes and season statistics.
- The order, the backfill's raised position, the windows and the margin can be changed by hand without changing code.

#### FR-30: A reserve for the rest of the day

Before lower-priority work starts, the quota left for the day is compared with what the work above it still needs until the daily reset, as the daily quota prediction estimates it (FR-31). Lower work starts only when the quota left exceeds that reserve (owner, 2026-10-20). This keeps the order meaningful through the day: without a reserve, an evening's closing-line pulls could find nothing left.

**Consequences (testable):**

- Lower work stops when the quota left meets the reserve, and the stop is recorded.
- The daily reset time and the per-minute limit are read from the provider's quota headers, which the fetch log records, not assumed.
- When the provider reports the quota exhausted, inside a successful HTTP response, all work stops until the reset, and the stop is recorded. The container interface (§4.1) reports it as a retryable failure.

#### FR-31: Daily quota prediction

Each day, the container predicts how many calls each part of the priority order (FR-29) will need, and so how many the backfill can use (owner, 2026-10-20). The prediction draws on the day's schedule (which games, in which competitions, with what coverage, at what times), the odds schedule by date, the refresh cadences and recent actual use. It feeds the reserve (FR-30). It also looks ahead over the next 7 days, whose schedules are known, so the backfill's progress can be forecast (owner, 2026-10-20).

**Consequences (testable):**

- A prediction is made before each day starts, and it can be recomputed for the rest of the day at any moment, as the reserve requires.
- The backfill's allowance for a day is the quota minus the predicted need of all the work above it.
- Each day's prediction is stored next to the actual use of each part of the order, so the prediction's accuracy can be measured and improved.
- The stored predictions and actual use show whether backfill windows and their margins reduce demand as expected (FR-29).

### 4.7 Raw landing zone and queryable model

*In progress.* Faithful storage, retention modes, a model shaped like the provider's, latest-version resolution.

**Retention so far.** Pre-match odds are appended, and every snapshot is kept (§4.8). Player profiles and venues keep dated versions (FR-24). Every other record is overwritten when its content changes, and only then. The audit columns show that, and when, it changed. They do not show what it was before. [SHORTCUT: superseded versions are not kept, only the fact and time of each change; the professional version keeps every version of a game's data once the game is final, which limits stored versions to the restatement period; revisit when a public number changes and its earlier value must be shown.] Keeping every version at all times was rejected, because frequent polling, such as the live feed every 15 seconds (FR-15), would create far too many records.

**Functional Requirements:**

#### FR-12: Audit columns on every table

Every table carries two audit columns, `TimeCreatedUtc` and `LastModifiedUtc`. That covers the raw landing zone, the queryable model and any operational table. `LastModifiedUtc` changes only when a row's content changes. It never changes when a retrieval or a check finds the content unchanged. Views in the queryable model expose the audit columns of the rows they read. More audit columns can be added where a need appears.

**Consequences (testable):**

- Storing a record for the first time sets both columns to the same UTC time.
- Retrieving identical content again leaves `LastModifiedUtc` unchanged. The retrieval appears only in the fetch log.
- Retrieving changed content overwrites the record and sets `LastModifiedUtc` to the time of that retrieval. `TimeCreatedUtc` never changes.
- Pre-match odds are never overwritten (§4.8). Each odds snapshot is a new record.

#### FR-24: Dated versions of player profiles and venues

Player profiles and venues keep dated versions of their data, so that changes over time can be followed, such as a player's height and weight, or a venue's details. This is a deliberate exception to overwriting (§4.7). Both are retrieved about once a season (FR-22), so the versions stay few. A new version is stored only when the content has changed; the fetch log shows each date on which it was checked and found unchanged.

**Consequences (testable):**

- Each version records the retrieval it came from, and that retrieval's time.
- A retrieval that finds the content unchanged adds no version.
- The latest version can be queried as the current value.

### 4.8 Odds capture

**Description.** Pre-match odds are the only data that cannot be fetched again once they leave the provider's 7-day history, so they get the strictest guarantees (§1). Every odds retrieval is kept: odds are appended, never overwritten (FR-12), with every bookmaker and every bet type. Odds feed the analysis leading up to each game, so they are pulled before kickoff as well as for the closing line. Storage is not a concern for now (owner, 2026-10-17). [SHORTCUT: every odds snapshot is stored, even when unchanged; the professional version stores only snapshots that differ from the previous one, as the brief's "never miss a line move" direction intends; revisit when odds storage grows enough to matter.]

Pre-match odds stop updating before kickoff and stay retrievable after the game (OQ-1, resolved 2026-10-17). In the owner's sample, the game kicked off at 19:00 UTC, the odds were last updated at 18:09:16 UTC, and they were still available the next day. So a closing line missed before kickoff can still be retrieved for days afterwards.

**Functional Requirements:**

#### FR-26: The closing line of every game with odds

Every game with pre-match odds has its closing line stored. The final pull is made 5 to 10 minutes before kickoff, so the closing line is available for analysis before the game (owner, 2026-10-17; kept 2026-10-18 alongside the schedule by date). The schedule's pulls of yesterday's games, every 6 hours (FR-27), store the closing line as the provider holds it after the game. They catch a provider update in the last minutes before kickoff, and a pre-kickoff pull that failed.

**Consequences (testable):**

- Each game with odds gets a pull 5 to 10 minutes before its kickoff, one call per game, or one call per competition when several of its games kick off together.
- The closing line is the latest stored snapshot whose `update` time is at or before kickoff.
- A game whose closing line is still missing is retried daily until its odds leave the 7-day history. It is then recorded as a gap, with the reason.
- `odds/mapping` is retrieved daily, so the games that should have a closing line are known. A game listed there without a closing line is a recorded gap, never a silent one (§5, item 2).
- Each stored closing line keeps each bookmaker's `update` time. Data verification (§4.11) flags a closing line updated after kickoff.
- When tier 3 goes live, the closing lines of the previous 7 days' games are retrieved too, because the provider still holds them.
- API-AFL's `odds` endpoint takes only a game, so its closing lines are retrieved one game at a time, in the same way.

#### FR-27: Odds snapshots by date

Odds are pulled by date, 10 games per call, on a cadence that tightens as the date approaches (owner, proposed 2026-10-17, adopted 2026-10-18):

| Date | Cadence |
|---|---|
| Yesterday | Every 6 hours |
| Today | Every hour |
| Tomorrow | Every hour |
| In 2 days | Every 2 hours |
| In 3 and in 4 days | Every 6 hours |

With the final pull before kickoff (FR-26), each game gets about 70 snapshots, most of them before kickoff, so odds can feed the analysis leading up to each game. The schedule replaces the earlier plan of pulls at set points before kickoff (about 48, 24, 12, 6 and 3 hours), each of which it covers within about an hour. It also replaces the brief's rule of closing lines only in phase 1. It costs about 4,000 calls a day (addendum §12). After the backfill, the window and the cadence are revisited toward the brief's regular snapshots across the whole 14-day window (OQ-3).

**Consequences (testable):**

- Each date in the window is retrieved on its cadence, and every retrieval is recorded in the fetch log.
- A game that has odds but is missing from a retrieval of its date is recorded as a gap.
- Snapshots are appended, never overwritten (FR-12), and stored per game (FR-21).
- The dates covered and each date's cadence can be changed without changing code.
- Odds pulls rank first in the priority order (FR-29). They never give way to the backfill, and when calls run short they are the last work to stop.

### 4.9 Fetch log and run tracking

*Pending.* Categorized outcomes, run ID, coverage and progress reporting, hand-off to cleansing.

### 4.10 Documentation and decision records

*Pending.* Airflow guide, ADRs, runbooks, shareable architecture overview, cold-start test.

### 4.11 Data verification

*Pending.* Automated checks against the Data Correctness Standard (§5), run with each workflow, with results recorded.

### 4.12 Game lifecycle and re-processing

**Description.** Each game moves through three stages: pre-game, live and final. The stage decides what game processing (FR-11) collects for the game, and how often (§4.2). The stage watch reads the live feed every 15 seconds, so a finished game normally starts processing within seconds of the final whistle (FR-15).

The provider can change a game's data after the game is final, sometimes months later, while the season is still open. Such a change is a restatement. Restatements are found in two ways:

- **Restatement checks.** During its restatement window, each final game is processed again 15 minutes and 1 hour after it was first seen final, then once a day. The early checks catch the provider's last updates to a game that has just finished. The daily checks are slower than pre-game, live and final processing, because a later restatement need not be found at once.
- **Noticed in passing.** Whenever any processing retrieves data for a final game and that data differs from what is stored, the game gets a re-processing mark. For example, a season schedule might show a changed score.

Any issue found while processing a game also marks it, and so does a coverage change (FR-5). The owner can request re-processing by hand, for one game or for a whole competition and season. A restatement overwrites the stored data, and the audit columns record that it changed, and when (FR-12). Because later predictions may draw on the restated game's statistics, a restatement also re-fetches the predictions of every game that kicked off at the same time or later (FR-14).

**Functional Requirements:**

#### FR-6: Game stages

The container derives each game's stage from the provider's status. The stage decides what game processing collects for the game, and how often.

**Consequences (testable):**

- Each game has exactly one current stage: pre-game, live or final.
- Every provider status maps to exactly one stage, including postponed, suspended, interrupted, abandoned, cancelled, awarded and walkover games.
- A status with no mapping is recorded as a processing issue, and the game is marked (FR-10).
- The time each game was first seen final is stored. The restatement window is computed from it.

#### FR-7: Final processing within minutes

When a game becomes final, game processing collects all of the game's per-game datasets and lands them within the arrival targets (§5, item 4). The stage watch sees most games leave the live feed within about 15 seconds of the provider marking them final (FR-15).

**Consequences (testable):**

- Processing starts within 2 minutes of the provider marking the game final as a target, and never later than 15 minutes. The data is queryable within a further 5 minutes. In normal operation, processing is expected to start within about 15 seconds, the live feed's update rate. The 2-minute target leaves room for irregular timing in the provider's updates.
- The provider does not publish the moment it marks a game final. Delays are therefore measured from the first sign that the game ended: the first live-feed retrieval in which it no longer appears, or, for a game never seen live, the first retrieval that shows it final. For every game, the container records that time, the time processing started, and the time the data became queryable.
- A game that misses the 15-minute or the 5-minute limit is recorded as late, with the delay.
- The arrival targets apply to games whose end the stage watch sees (FR-15). Games loaded by the backfill are not measured against them.

#### FR-8: Restatement checks

During its restatement window, each final game is processed again 15 minutes and 1 hour after it was first seen final, then once a day. The two early checks catch the provider's last updates to a game that has just finished, which detail retrieved at the whistle can miss. If the provider's data differs from the stored data, that difference is a restatement: the changed data overwrites the stored data and moves `LastModifiedUtc` (FR-12). The checks skip predictions. Instead, a restatement starts the prediction cascade (FR-14).

**Consequences (testable):**

- The window lasts 7 days from the time the game was first seen final, and it does not change as the season progresses.
- A game first seen final more than 3 months after it was played, typically by the backfill, is past its window from the start: it gets no early or daily checks (owner, 2026-10-18). A game first seen final sooner gets the full window, counted from when it was first seen final. The 3-month threshold can be changed without changing code.
- The early checks run 15 minutes and 1 hour after the game was first seen final. After that, checks run once a day, to conserve calls. The timings, the daily cadence and the window length can be changed without changing code.
- A check covers all of the game's data except pre-match odds and predictions.
- Every check is recorded with the game, the time and the result: unchanged, restated or failed.
- A check that cannot run when due, because of quota or downtime, is recorded as missed, and it runs at the next opportunity.

#### FR-9: Restatements noticed in passing

Whenever any processing retrieves data for a final game, the container compares that data with the stored data. A difference in any field gives the game a re-processing mark (FR-10).

**Consequences (testable):**

- A season schedule retrieval that shows a changed score, status or kickoff time for a final game marks that game.
- Detection covers every field the retrieval returns, not only the score.
- Detection applies to every final game, whether or not its restatement window is still open.

#### FR-10: Re-processing marks

A game, or a competition and season, gets a re-processing mark when any of these happens:

- a restatement is noticed in passing (FR-9);
- processing hits an issue;
- coverage changes (FR-5);
- the owner requests it (FR-13).

The system must know that the re-processing is needed, and it must be able to start it. That knowledge survives failures and restarts, and it is kept until processing succeeds. A needed re-processing is therefore never lost, even when it cannot run at once. [NOTE FOR PM: how marks are kept and acted on is an architecture decision. The options include a tracking table, a message queue, and deriving marks from data already stored (addendum, §2).]

**Consequences (testable):**

- Each mark identifies what it covers (a game, or a competition and season), the reason, and the time it arose.
- A mark is cleared only when processing succeeds, and the run that cleared it is recorded. A failed attempt leaves the mark in place and records the failure.
- A competition-and-season mark is cleared once every game and every season-level dataset in it has been processed again.
- Every mark that existed before a restart still exists after it.

#### FR-13: Manual re-processing

The owner can request re-processing of one game, or of one competition and season. The second covers every game and every season-level dataset in it.

**Consequences (testable):**

- Each request sets a re-processing mark with the reason "owner request" (FR-10).
- The container interface documents how to make each kind of request (§4.1).

#### FR-14: Prediction cascade after a restatement

When a game is restated, the container fetches again the predictions of every game in the same competition and season that has not yet kicked off and whose predictions are already stored: the games due within the next 7 days (FR-17). Such a game's prediction may draw on the restated game's statistics, so it may have changed. The daily refresh would pick up the change within a day; the cascade picks it up sooner. Games that have already kicked off keep their kickoff capture (FR-17), which is the prediction the provider gave before the game. Re-fetching it would replace it with one computed after kickoff, from corrected data, against the brief's rule that point-in-time training sets exclude data from after a game starts (option B, chosen 2026-10-11; addendum §3).

**Consequences (testable):**

- A restatement of any of a game's data except its predictions starts the cascade.
- The cascade never re-fetches the predictions of a game that has kicked off, the restated game included.
- A re-fetched prediction that differs from the stored one overwrites it and moves `LastModifiedUtc` (FR-12).
- A prediction change found by the cascade does not start a further cascade.
- The cascade runs through game processing (FR-11), limited to predictions.
- A game due within 48 hours has its predictions re-fetched at most once an hour, the provider's update rate, as after a final (FR-17). Close to kickoff, each completed or restated game can change them, so they are kept as current as the provider allows. A game due later has them re-fetched at most once a day, however many restatements affect it.

## 5. Data Correctness Standard

The data, the analysis built on it and this system's architecture may be shared publicly at any time. Ingested data must therefore never be wrong. It may be incomplete, provided the gap is recorded.

> **Missing and recorded is acceptable. Wrong is not.**

Ingestion vouches for its own copy, not for the provider. Errors in the provider's data are stored exactly as returned. Downstream quality checks may flag them but never change them. The ingested copy is **off** when any of the following holds:

1. **Unfaithful.** What is stored differs from what the provider returned.
2. **Missing without a record.** Something in scope is absent, and nothing records why. Examples: a skipped page, a finished game without detail, a missed closing line. An absence the provider declares, such as a dataset its coverage map marks unavailable for a competition and season, is an explained absence, not a gap (FR-4).
3. **Stale without a flag.** Data is older than its freshness rule allows, and nothing flags it.
4. **Late without a record.** A game misses a timing target, before kickoff or after the final whistle, and nothing records the miss. The owner sets these targets, and architecture may revise them when it shows a technical constraint.
   - **Before kickoff.** Every game whose competition covers predictions has predictions stored before kickoff, and a kickoff capture at kickoff (FR-17). Lineups are checked for from 40 minutes before kickoff until they are stored or the game goes live (FR-16). Injuries are stored before kickoff, from a retrieval made within the 2 hours before it (FR-18). Odds are pulled 5 to 10 minutes before kickoff (FR-26), and on the schedule by date before it (FR-27).
   - **Arrival.** Processing of a finished game starts within 2 minutes of the provider marking it final, and never later than 15 minutes. The data is queryable within a further 5 minutes (FR-7). [ASSUMPTION: the brief's 15-minute limit still applies alongside the new 2-minute target.]
   - **Restatement checks.** Every final game is checked for restatements throughout its restatement window (FR-8).
5. **Misrepresented.** The queryable model disagrees with the raw data. Rows are duplicated or dropped, or an older version of an entity is shown instead of a newer one when requests overlap.
6. **Untraceable or misdated.** A value cannot be traced to the retrieval that produced it, or to the time of that retrieval. This also covers wrong audit columns: a `LastModifiedUtc` that moved although nothing changed, or a change that did not move it (FR-12).

Data verification (§4.11) checks this standard automatically wherever a check is possible. Nobody is on call (§7.2), so the standard is met through detection: every missed target is recorded, and the system catches up when it can.

## 6. Cross-Cutting Non-Functional Requirements

*In progress.*

#### NFR-1: Times in UTC

Times are stored in UTC wherever possible.

- Every time the container writes is in UTC: audit columns, fetch-log entries, marks and delay measurements.
- [ASSUMPTION: every request to an endpoint that accepts a `timezone` parameter sets it to UTC, so the provider's times also arrive in UTC. This settles the brief's question of whether `timezone` belongs in request identity: it is always UTC.]
- Raw payloads are stored exactly as returned, even when they contain times that are not in UTC. Faithfulness comes first (§5, item 1).
- [ASSUMPTION: the name of every column that holds a time ends in `Utc`, as the audit columns' names do.]

## 7. Non-Goals

### 7.1 Product non-goals

- **Airflow DAGs and scheduling.** The project delivers the container and instructions for running it from Airflow, not the DAGs.
- **Downstream modeling.** Cleansing, any model of the sport itself, and merging with other sources all happen downstream. So does any calculation of derivable data, such as live standings or season statistics built from game data (FR-23).
- **Correcting provider data.** Quality checks flag provider errors. Nothing in ingestion changes them.
- **Recreating past states.** Data other than odds, player profiles and venues (FR-24) is overwritten when it changes. The audit columns show that a change happened, and when, but not the earlier value (§4.7).
- **Database backups.** Backups must be stored off the server. That work is outside this project, and the location is still to be decided.
- **Other data sources.** The pattern is built to accept them, but this project builds none.
- **Publishing data.** Public analysis happens outside this project. [ASSUMPTION: so does compliance with the provider's terms for republishing its data. Ingestion supports it only by keeping every value traceable (§5, item 6).]

### 7.2 Solo exclusions

These are team practices that do not fit a one-person project. Each one is a deliberate choice, not an oversight.

- **Time-based sprints.** One developer gains nothing from time boxes, velocity tracking or sprint ceremonies. Work still follows epics and stories, and direction can change at any point. BMAD's sprint-status file serves as an ordered story queue, with no dates or time boxes.
- **Formal sign-off gates.** The owner is the only approver. Approvals are recorded in the decision log or in an ADR.
- **On-call and paging.** Nobody is on call. Freshness targets are design targets for normal operation, not response-time commitments. When a target is missed, the miss is detected and recorded (§5).
- **A second human code reviewer.** AI review takes that role, through bmad-code-review and the review sessions in bmad-loop.

## 8. MVP Scope

### 8.1 Implementation tiers

Every endpoint is in scope (§4.3). The tiers set the order in which endpoints are built, so development stays focused on one layer at a time. Tiers are built in order.

1. **Tier 1: core reference data.** The data that drives all processing: competitions (`leagues`, including the coverage map), seasons (listed in `leagues`; the `leagues/seasons` endpoint is derivable, FR-23) and teams, for both APIs. [ASSUMPTION: `countries`, `teams/countries`, `venues` and `timezone` join tier 1 as small reference datasets.]
2. **Tier 2: games.** Schedules and results: `fixtures` for API-Football and `games` for API-AFL, with game stages (FR-6) and game processing (FR-11). Each API-Football game record comes from `fixtures?ids=` and already holds the game's detail (FR-21); tier 4 exposes that detail in the queryable model.
3. **Tier 3: pre-match odds.** The closing line for every game with odds on offer: `odds` for both APIs. [ASSUMPTION: the odds reference endpoints come with tier 3: `odds/bookmakers` and `odds/bets` for both APIs, and `odds/mapping` for API-Football.]
4. **Tier 4: game statistics and player data.** Game detail and player information. For API-Football, game detail means events, lineups, statistics and player statistics. For API-AFL, it means quarters, events, and team and player statistics. [ASSUMPTION: every remaining endpoint also falls in tier 4, and its order within the tier is set when epics are planned. That covers predictions, injuries, standings, team and player season statistics, squads, coaches, transfers, trophies, sidelined and player profiles (FR-22).] Leaderboards, rounds, head-to-head, season lists and player careers are derivable and not collected (FR-23). Injuries stay in tier 4, because a game's injury records, including a pre-match "Questionable" status, remain retrievable after the game (§10, OQ-6).
5. **Later: in-play odds.** `odds/live` and `odds/live/bets`, after a feasibility study, once pre-match odds and game and player data are in. They are not a priority while closing lines are captured.

Pre-match odds cannot be backfilled beyond the provider's seven-day history. Odds coverage therefore starts when tier 3 goes live, with the closing lines of the previous 7 days' games (FR-26).

### 8.2 Coverage phases

The phases set the order in which seasons are collected, as agreed in the brief:

1. **Phase 1.** The current season and the one before, for every competition, as soon as possible, in two passes (owner, 2026-10-20): first the games and statistics of every phase-1 competition (standings, teams, team and player statistics, injuries), in priority order; then their predictions, in priority order. This replaces the brief's rule that each competition is completed before the next starts. During phase 1, odds follow FR-26 and FR-27.
2. **Phase 2.** Every competition, for every season in progress on 1 January 2022 or starting after it. Major competitions come first.
3. **Phase 3.** As many competitions and seasons as possible.

Trophies are not part of the phases. They are collected in one pass over every stored player and coach once the backfill is complete (FR-22). [ASSUMPTION: "complete" means after phase 2, because phase 3 has no end.]

## 9. Success Metrics

*Pending.*

## 10. Open Questions

*To be revisited with the owner.*

1. **OQ-1. Closing-line timing.** Are pre-match odds frozen at kickoff, and still retrievable after the game? The answer decides when the closing line is taken (§4.8). *(From the brief.)* *Resolved 2026-10-17:* yes, on the owner's sample. The game kicked off at 19:00 UTC, its odds were last updated at 18:09:16 UTC, and they were still available the next day. The closing line is pulled 5 to 10 minutes before kickoff and stays retrievable afterwards (FR-26), and every closing line's `update` time keeps being checked.
2. **OQ-2. Batch detail.** *Resolved 2026-10-11:* yes. The owner's test shows that `fixtures?ids=`, with up to 20 dash-delimited game ids, returns the full game detail, as `fixtures?id=` does. Game detail, in-play detail included, therefore costs one call per 20 games (FR-19, FR-20). The game-detail endpoints and `predictions` still take one game per call. *(From the brief.)*
3. **OQ-3. Odds cadence.** Which rate within the 1–3 hour range best balances measured odds changes against the quota? *(From the brief.)* It sets the cadence of the regular snapshots after the backfill (FR-27).
4. **OQ-4. Hand-off to cleansing.** How does downstream cleansing learn what a run changed? The brief proposes a run ID stamped on the fetch log and on every row written. `LastModifiedUtc` (FR-12) now also supports "changed since" queries. *(From the brief.)*
5. **OQ-5. Competition configuration.** Do competition priority ranks (FR-3), the top-tier designation (§3), per-competition rules such as the `teams` cadence (FR-22), and the backfill windows (FR-29) live in a repository file, or in a database control table that can change without rebuilding the image? *(From the brief, reframed now that every competition is in scope.)* *Resolved 2026-10-20:* a file in the repository, mounted into the container, so changes are versioned and reviewable, and need no image rebuild (FR-32).
6. **OQ-6. Injury status.** *Resolved 2026-10-11:* yes. Whole-season retrievals for MLS 2026 and the Premier League 2025 still hold "Questionable" records for past games, for example on 21 February 2026 and 20 September 2025. The provider keeps the status after the game, so injuries can be fetched again later, and they stay in tier 4 (FR-18). It is not proven that every pre-match record survives unchanged; restatement checks would detect a later change. *(From the brief.)*
7. **OQ-7. Carried endpoints.** Do the four game-detail endpoints return anything that `fixtures?id=` does not, such as the half-time split of statistics? The answer decides whether they are called separately (FR-2). With OQ-2 resolved, this is the last check before `fixtures?ids=` counts as carrying all four.
8. **OQ-8. Restatement window length.** *Resolved 2026-10-10:* the window is a fixed 7 days, and it does not grow as the season progresses. Checks run daily and skip predictions. Instead, a restatement re-fetches the predictions of every game that kicked off at the same time or later (FR-14). No season closes before its games' windows expire (§3, Season).
9. **OQ-9. Restated versions.** *Resolved 2026-10-08:* data is overwritten and no earlier version is kept, and the audit columns record each change (FR-12, §4.7).
10. **OQ-10. Games in the queryable model.** How does the queryable model combine the cached schedules and the game records (FR-21)? *Deferred to the model's design (owner, 2026-10-11).* The likely direction is to take a game's summary fields from either table, depending on the game's stage, for example the full schedule table LEFT JOIN the per-game table. Each table may also be used on its own, depending on what a use case needs.
11. **OQ-11. Self-starting backfill.** Should the backfill also start itself, working through the phases whenever the quota budget leaves room (FR-28)? *Deferred (owner, 2026-10-20).* A manual start must stay possible, and automation is generally preferred.

## 11. Assumptions Index

- §3, Season: a season closes as soon as either condition holds.
- FR-15: a game that leaves the live feed starts final processing only after its own status confirms that it finished.
- FR-15: API-AFL's day schedule, retrieved every 30 seconds while a game is due to be in play, takes the live feed's role.
- FR-17: apart from the kickoff capture, a game's predictions are retrieved at most once an hour.
- FR-17: final processing retrieves predictions only for a game without a kickoff capture.
- FR-19: when calls run short, higher-ranked competitions keep their in-play rate longest.
- FR-22: standings are retrieved again after a restatement in the competition.
- FR-22: trophies are retrieved when a competition's season closes, for every player and coach who appeared in it.
- §8.2: trophies are collected once the backfill's phase 2 is complete, because phase 3 has no end.
- FR-22: season schedules are retrieved daily for each competition in season.
- FR-28: a backfill run started without named competitions and seasons takes the next work in phase order.
- NFR-1: requests set `timezone` to UTC wherever the endpoint accepts it.
- NFR-1: every column that holds a time has a name ending in `Utc`.
- §5, item 4: the brief's 15-minute limit still applies alongside the 2-minute target.
- §7.1: compliance with the provider's terms for republishing data is a downstream concern.
- §8.1, tier 1: `countries`, `venues` and `timezone` join tier 1.
- §8.1, tier 3: the odds reference endpoints come with tier 3.
- §8.1, tier 4: every remaining endpoint falls in tier 4, ordered when epics are planned.
