---
title: "Addendum: PRD for the API-Sports Ingestion Container"
created: 2026-10-08
updated: 2026-10-20
---

# Addendum: PRD for the API-Sports Ingestion Container

This addendum holds the detail that architecture and later phases need but that does not belong in the PRD: alternatives considered, design implications and sizing. Each decision is marked with the date it was made. The full decision trail is in `.memlog.md`, in this folder. Source-API facts, the landing-zone design inputs and the original quota sizing remain in the brief's addendum (`_bmad-output/planning-artifacts/briefs/brief-apisports-2026-10-05/addendum.md`).

## 1. Change tracking and restatements

### Decisions

- **2026-10-08.** A restatement overwrites the stored data. No earlier version is kept, and recreating a past state is not a goal.
- **2026-10-08.** `LastModifiedUtc` changes only when a row's content changes, never when a retrieval or check finds it unchanged (PRD FR-12).
- **2026-10-10.** The audit columns `TimeCreatedUtc` and `LastModifiedUtc` go on every table: raw landing zone, queryable model and operational tables. More audit columns can be added if a need appears.
- **2026-10-10.** Times are stored in UTC wherever possible (PRD NFR-1).
- **Odds are outside these decisions.** The phase-1 rule from the brief stands: every closing-line call is kept. Keeping every odds snapshot is evaluated in a later phase.

### Options considered

| Option | For | Against | Status |
|---|---|---|---|
| Keep every version of every record | Full history; any past state can be rebuilt | Frequent polling would create far too many records: each competition's schedule every one or two minutes, as expected on 2026-10-08, and the live feed every 15 seconds since 2026-10-10 | Rejected |
| Keep versions only once a game is final, during the restatement period (the owner's alternative) | Shows what a public number was before it changed; the number of versions stays small | More storage and more complex queries; current analytics do not need earlier values | Recorded as the professional version behind the `[SHORTCUT]` in PRD §4.7. Revisit when a public number changes and its earlier value must be shown |
| Overwrite on change, with audit columns | Simple; storage stays small; `LastModifiedUtc` shows that a row changed, and when | Shows that a change happened but not what it was | **Chosen** |

### Implications for architecture

- **Per-game change detection is already required.** In the brief's design, a raw landing record holds one provider response, and a schedule response can cover about 380 games. PRD FR-9 requires knowing *which* game in such a response changed. That means comparing each game's data on its own. Since 2026-10-11, multi-game responses are split by game before they are stored (PRD FR-21, §10), so that comparison happens per game as data lands. Once that comparison exists, per-row audit columns in the queryable model follow almost for free. A plain view over the stored JSON cannot do either, so this pushes the choice between views and persisted tables toward persisted tables, maintained by a merge with change detection. The brief left that choice to architecture on performance grounds alone. Views that remain expose the audit columns of the rows they read.
- **Comparing content, not bytes.** Change detection must ignore differences that carry no data, so that unchanged content never moves `LastModifiedUtc`. Architecture decides how JSON is normalized before comparison, for example key order, array order, and any fields that vary between calls.
- **Downstream hand-off.** `LastModifiedUtc` lets cleansing ask what changed since a given time. It complements the run ID proposed in the brief (PRD OQ-4).

## 2. Game processing as one workflow (decided 2026-10-08)

The owner's direction: a game is processed the same way in every stage, through one workflow, and other workflows trigger it when needed (PRD FR-11).

Triggers known so far:

- **Stage watch.** A game that leaves the live feed, or that the day schedule shows final although it never appeared live. Processing starts within 2 minutes as a target (PRD FR-7, FR-15).
- **Pre-game timers.** Lineup checks from 40 minutes before kickoff, prediction refreshes, and the kickoff capture (PRD FR-16, FR-17).
- **In-play detail.** Live games, within the quota share for in-play detail (PRD FR-19).
- **Restatement checks.** Each final game, 15 minutes and 1 hour after it first becomes final, then once a day for 7 days (PRD FR-8).
- **Re-processing marks.** These are set when a restatement is noticed in passing, when processing hits an issue, when coverage changes, or when the owner requests it (PRD FR-9, FR-10, FR-13).
- **Backfill.** Every game of a competition and season.

On 2026-10-08 the owner expected to poll each competition's schedule every one or two minutes. Since 2026-10-10, the stage watch instead polls the live feed every 15 seconds and the day schedule, about once an hour at first and every 5 minutes since 2026-10-11 (PRD FR-15).

### Options for keeping and acting on re-processing marks

The owner treats a mark as a need, not a mechanism (2026-10-10). The system must know that something needs re-processing, and it must be able to start it. The mechanism is open:

| Option | How it works | For | Against |
|---|---|---|---|
| A. Tracking table | Rows in SQL Server record each pending re-processing; workflows read them and clear them | Durable, and written in the same transaction as the data; easy to see what is pending, even after months away; no new infrastructure | Workflows must poll it; needs care so two runs do not take the same row |
| B. Message queue | Each mark is a message that starts processing | Event-driven, so processing starts without polling | A new service to run, secure and back up on the one server; harder to see what is pending; messages must be made durable |
| C. Derived from stored data | Pending work is computed by queries over data already required. Examples: a game whose schedule row changed after its last full processing, or whose last attempt failed in the fetch log | No extra state that can drift; uses the audit columns and the fetch log | Every trigger must be expressible as a query. An owner request leaves no trace in the data, so it would run at once as a parameterized run instead of being queued |

Options can be combined, for example C for restatements and failures plus A for owner requests. Option B brings infrastructure that a single server rarely needs. Options A and C fit the database that already exists.

Other questions for architecture:

- Does game processing run inside the triggering workflow's container run, or as runs of its own?
- How do two triggers for the same game avoid processing it twice at the same time?
- How does the stage watch keep a 15-second cadence under Airflow: a run that stays up and polls, or a run every minute that polls four times (PRD FR-15)?

## 3. Restatement-check sizing (rough; every input unverified)

Inputs come from the brief's estimates:

- about 1,100 football competitions
- about 150 games per competition season, so about 165,000 games per season year
- seasons running about 300 days, so about 550 games becoming final per day on average, with weekend peaks well above that

Since 2026-10-11, each final game is checked 15 minutes and 1 hour after it was first seen final, then once a day for 7 days (PRD FR-8). A check re-fetches two things and skips predictions:

- **Game detail.** One call per 20 games with `fixtures?ids=`, which carries the full detail (owner-tested 2026-10-11, PRD OQ-2), instead of one call per game with `fixtures?id=`.
- **Injuries.** Since 2026-10-10, one retrieval per competition and season, shared by every game of that competition and season checked the same day (PRD FR-18). That is at most about 900 calls a day, one per in-season competition, however many games are checked.

That makes about 0.05 calls per game, plus the injuries retrievals.

### Prediction cascade (decided 2026-10-10, PRD FR-14; re-sized 2026-10-11)

As decided on 2026-10-10, a restatement re-fetched the predictions of every game that kicked off at the same time as the restated game or later, because those predictions may draw on the restated game's statistics. Since 2026-10-11 (option B, below), it reaches only games not yet kicked off. Predictions cost one call per game and do not batch. The scope is the same competition and season, limited to games whose predictions are already stored (confirmed 2026-10-11).

**Reach.** Predictions are stored only for games already played and for games due within 7 days (PRD FR-17). A restatement therefore reaches the games played since the restated game kicked off, plus the next 7 days of games: about 5 to 20 games, depending on how busy the competition is. The 2026-10-10 estimate of about 75 games assumed predictions stored for the rest of the season.

**Cap: once an hour or once a day per game (owner's question, 2026-10-11).** With a daily cap, each game in reach is re-fetched at most once a day, however many restatements touch its competition. With an hourly cap, it is re-fetched once for each hour in which a restatement touches its competition. That count depends on when restatements are found: the early checks run at each game's 15-minute and 1-hour marks, and the daily checks at each game's daily time.

| Restatements | Daily cap | Hourly cap |
|---|---|---|
| Rare: 1% of final games (about 5 a day) | about 50 (0.1%) | about 50: rarely more than one restatement per competition per day |
| Common: 30% of final games (about 165 a day, over about 100 competitions) | about 1,000 (1.3%) | about 2,000 to 3,000 (3 to 4%) |
| The early checks find changes in most games (about 440 a day, over about 300 competitions) | about 3,000 (4%) | about 9,000 to 18,000 (12 to 24%) |

Inputs, all unmeasured: about 10 games reached per restatement; 2 to 3 distinct hours with restatements per competition per day in the common case, and 3 to 6 in the early-check case.

**The early checks matter more than the cap.** A just-finished game's statistics and player ratings may change in the first hour, so the 15-minute and 1-hour checks could record a change for most games. Counted as restatements, those changes would start a cascade for nearly every game.

**Conflict with the kickoff capture (raised 2026-10-11).** For a game that has already kicked off, the cascade overwrites its kickoff capture with a prediction the provider computed after kickoff, from data corrected after kickoff. The kickoff capture is meant to be the pre-match prediction (PRD FR-17), and the brief's retention rule says point-in-time training sets exclude data from after a game starts. If the provider freezes past predictions, the cascade's calls for played games change nothing. If it recomputes them, they replace the pre-match view.

| Option | What a restatement re-fetches | Calls a day | Against |
|---|---|---|---|
| A. As decided on 2026-10-10 | Games played since the restated game, and games due within 7 days | The table above | Overwrites kickoff captures |
| B. Kickoff captures are final (**chosen** 2026-10-11) | Games due within 7 days | About half of A | Played games keep the prediction the provider gave at kickoff, before the correction |
| C. A restatement counts as a final for predictions (assistant's recommendation) | Games in the same competition due within 48 hours, through the refresh after a final (PRD FR-17), under its hourly cap | At most about 1,000; usually a few hundred | Games 2 to 7 days out wait for the next daily refresh: at most a day |

Under C, the cap question disappears, because the FR-17 hourly cap already applies. Changes found by the early checks add little, because the refresh after a final waits for the provider to absorb the result anyway.

**Chosen: B (2026-10-11).** Under B, a restatement reaches about half as many games as under A, about 5, so the cap table above roughly halves:

- rare restatements: about 25 calls a day with either cap;
- common restatements: about 500 with the daily cap, or 1,000 to 1,500 with the hourly cap;
- early checks finding changes in most games: about 1,500 with the daily cap, or 4,500 to 9,000 with the hourly cap.

**Cap chosen (2026-10-11): hourly within 48 hours of kickoff, daily beyond.** The owner expects predictions to change as each other game completes, and wants them as current as possible close to kickoff. About 1.5 of the 5 games in reach are due within 48 hours, so the cascade costs:

- rare restatements: about 25 calls a day;
- common restatements: about 650 to 800;
- the early checks finding changes in most games: at most about 2,400 to 3,750.

It costs less in practice, because the refresh after each final (PRD FR-17) re-fetches the same games, and the hourly cap merges the two.

No one knows yet how often restatements happen. Every check records its result as unchanged, restated or failed (PRD FR-8), so the first weeks of operation will measure it.

| Checks | Games checked per day | Calls per day, one game per call | Calls per day, with `ids` batching (applies since 2026-10-11) |
|---|---|---|---|
| Early: 15 minutes and 1 hour after final (**chosen** 2026-10-11) | about 550, twice | about 1,100 | about 100 to 1,100, depending on how many games finish together |
| 7 days, daily (**chosen**) | about 3,850 | about 4,750 (6% of 75,000) | about 1,100 |
| 7 days, every 6 hours (a possible later increase) | about 3,850, four times | about 19,000 (25%) | about 4,400 |

The daily rows include the injuries retrievals. Before injuries moved to competition-and-season retrievals on 2026-10-10, the same rows were about 7,700 and 30,800 calls without batching. Since 2026-10-11, injuries are also refreshed before kickoff (PRD FR-18), so a competition with games coming up is usually fresh already. The injuries share of these rows is therefore an upper bound.

A window that grew until the season closed was considered and rejected on 2026-10-10: the window is fixed. Mid-season, it would have meant checking about 82,500 games a day, or about 165,000 calls a day without batching.

Restatement checks share the quota with live processing, odds and the backfill (PRD §4.6). API-AFL is negligible: one competition of about 200 games a season, on a quota of 7,500 calls a day, and its detail endpoints batch up to 10 games per call.

## 4. Coverage map and API-AFL coverage (decided 2026-10-08)

- The coverage map is not tracked as a separate structure. It is part of the queryable model built from the `leagues` endpoint, and workflows read it there (PRD FR-4).
- A coverage change shows up as a moved `LastModifiedUtc` on the competition and season's coverage row, and it marks the competition and season for re-processing (PRD FR-5).
- API-AFL publishes no coverage map. It currently has one competition, which is treated as fully covered. Coverage tracking for API-AFL is added only if a need appears.

## 5. Implementation tiers: rationale (decided 2026-10-08)

The owner's principle is to build the core data that drives all processing first, then games, then pre-match odds, then game statistics and player data. In-play odds wait until pre-match odds and game and player data are in (PRD §8.1).

Observations:

- **Every day before tier 3 is lost odds history.** Pre-match odds cannot be backfilled beyond the provider's seven-day history.
- **Predictions and injuries now come later.** The brief's fixtures refresh included them in phase 1; under the tiers they fall in tier 4.
  - Predictions are safe to collect late, because past-game predictions were verified to show the pre-match state.
  - Injuries are safe to collect late too. Their pre-match "Questionable" status survives the game (PRD OQ-6, resolved 2026-10-11), so they stay in tier 4.
- **Pre-game collection starts with tier 4.** Lineups and injuries before kickoff, and the kickoff capture (PRD FR-16 to FR-18), begin when tier 4 builds those endpoints. Nothing is lost before then, because all three can be fetched after the game.
- **Detail arrives with tier 2.** Game records come from `fixtures?ids=` (PRD FR-21), so from tier 2 on, every game that final processing has handled already holds its events, lineups, statistics and players. Tier 4 exposes that detail in the queryable model, with no new retrievals for those games. In-play refreshes and lineup checks before kickoff still wait for tier 4.
- **Tier 4 is large.** It holds about 30 endpoints, and its order is set when epics are planned.

## 6. Specification facts noted during the PRD (2026-10-08)

- The football specification lists 38 endpoint paths. The brief and its addendum say 37.
- `/leagues` returns a coverage map for each competition and season (football).
- Football `/teams` includes venue data.
- Lineups are available 20 to 40 minutes before kickoff.
- The API-AFL detail endpoints (`games/quarters`, `games/events`, `games/statistics/teams`, `games/statistics/players`) accept `ids`, up to 10 games, and `date`.
- The API-AFL specification lists 22 football endpoints that API-AFL does not have.

## 7. Endpoint register: options (PRD FR-1; option B chosen 2026-10-10)

The question FR-1 answers: which of the 54 endpoints are built, which are still to come, and does any endpoint in the provider's specifications have no counterpart here? The coverage map cannot answer this, because it describes the provider's data, not this system.

| | A. Maintained register | B. Derived register | C. No register |
|---|---|---|---|
| What it is | A file in the repository lists every endpoint with its tier, its status and any carrying endpoint. A test checks the file against the specifications | No list to keep. A test compares the specifications' endpoints with the dataset definitions in the repository and reports every endpoint that has none. A carrying definition declares which endpoints it carries. Tiers stay in PRD §8.1 | PRD §8.1 tiers, plus BMAD epics, stories and sprint status |
| Notices a new provider endpoint | Yes | Yes | No |
| Can drift from reality | Yes: status is typed by hand | No: status is computed | Partly: stories track work items, not endpoints |
| Answers "what is built?" after months away | Read one file | Run one test | Read the stories |
| Upkeep | Highest | Low, after setup | None |
| Needs an operational table | No | No: it runs as a test | No |

**Chosen: B, the derived register (2026-10-10).** It keeps the check that catches drift between the specifications and the code, and it answers "what is built?", without a list to maintain. Endpoints without a definition are reported but do not fail the test, so tiers still to be built do not break the build. Only a definition that names an endpoint missing from both specifications fails it.

## 8. Operational state the requirements imply (2026-10-10)

The owner is not yet convinced that operational tables are needed. This inventory lists the state the current requirements call for, and where each piece could live. Architecture decides.

| State | Required by | Could live in |
|---|---|---|
| Every retrieval and its outcome | Brief; PRD §4.9 | The fetch log, the one operational table already decided |
| The time each game was first seen final | PRD FR-6, FR-7, FR-8 | A column on the game's model row, or derived from the fetch log |
| Processing start and queryable times per game | PRD FR-7 | The fetch log, or run tracking (PRD §4.9) |
| Pending re-processing (marks) | PRD FR-10, FR-13 | A tracking table, a message queue, or derived from data (§2) |
| Restatement-check results and missed checks | PRD FR-8 | The fetch log |
| The coverage map and its changes | PRD FR-4, FR-5 | The `leagues` queryable model, through its audit columns |
| Competition ranks | PRD FR-3 | A repository file or a control table (PRD OQ-5) |
| Which endpoints are built | PRD FR-1 | A test over the repository (§7, decided); no table needed |
| The time each game was first seen live, and the time it left the live feed | PRD FR-7, FR-15 | Columns on the game's model row, or the fetch log's game lists for live-feed retrievals (the feed's game data is not stored) |
| Pre-game work that is due: lineup checks, prediction refreshes, kickoff captures | PRD FR-16, FR-17 | Derived from kickoff times and stored data, or a tracking table (§2) |
| Quota spent on in-play detail | PRD FR-19 | The fetch log, read by the quota budget (PRD §4.6) |
| Daily quota predictions, and actual use per priority group | PRD FR-31 | A small operational table, or a daily summary derived from the fetch log |
| When each game's record was last retrieved | PRD FR-21 | A column on each stored row, or the fetch log's list of the games each retrieval returned (§10) |

The fetch log is the only operational table decided so far. Depending on the choices above, everything else can live in the fetch log, in model rows or in the repository, plus at most one tracking table.

## 9. Stage watch, pre-game and in-play collection (2026-10-10, updated 2026-10-11)

The owner's answers to the 4.2 stage questions, the provider facts behind them, and the sizing behind PRD FR-15 to FR-21.

### Decisions

- **Lineups** are needed before kickoff. Checks start 40 minutes before kickoff and grow more frequent as kickoff nears (PRD FR-16).
- **Predictions** are needed before kickoff. Future games are refreshed daily. Each time another game finishes, games due within 24 (possibly 48) hours are refreshed, after a wait for the provider to absorb the result. A capture is taken at scheduled kickoff. All of it respects the provider's hourly update rate (PRD FR-17). Settled on 2026-10-11: a 7-day horizon for the daily refresh, a 48-hour window for refreshes after a final in the same competition, and a wait set from measurement.
- **Injuries** are retrieved by competition and season, not per game (PRD FR-18). Since 2026-10-11, they are also needed before kickoff, for pre-game analytics, and are refreshed every 2 hours within 48 hours of kickoff.
- **Live play.** The live feed is polled every 15 seconds and the day schedule about once an hour, changed on 2026-10-11 to every 5 minutes. The faster schedule also catches postponements and cancellations close to kickoff. A game that leaves the live feed starts final processing (PRD FR-15).
- **Final timing (2026-10-11).** FR-7 keeps its 2-minute target. About 15 seconds is expected, and the target leaves room for irregular provider updates. Each final game is checked again 15 minutes and 1 hour after final (PRD FR-8).
- **Two caches (2026-10-11).** Requests that repeat (season schedules, the day schedule, injuries, every other endpoint) use the request cache. `fixtures?ids=` batches use the per-record cache (named the per-game cache until 2026-10-17), with one game record per game. How the queryable model combines the schedules and the game records is deferred (PRD OQ-10), and the live feed is not stored (PRD FR-21, §10). Injuries are their own endpoint, never part of a game record. API-AFL keeps its separation by endpoint.
- **In-play rate (2026-10-11).** Every live game in every 15-second cycle is the functional target. The rate may be lowered only for technical reasons, such as processing time or rate limits (PRD FR-19).
- **Prediction cascade (2026-10-11).** Option B: kickoff captures are final, and the cascade reaches only games not yet kicked off. Its cap is hourly for games due within 48 hours and daily beyond (§3).
- **In-play detail** is wanted every minute, but quota limits it. The owner's priority was team statistics, then player statistics, then events (PRD FR-19).
- **In-play detail through `fixtures?ids=` (2026-10-11).** The live feed supplies the ids of the live games, and `fixtures?ids=` retrieves their detail in batches of up to 20. One call returns all three datasets, so the priority above decides nothing for in-play detail. It would apply only to single-game detail calls, if PRD OQ-7 shows any are still needed (assistant's reading, confirmed by the owner 2026-10-11). Each live-feed retrieval triggers the `fixtures?ids=` retrieval, for the live games and for the games that just left the feed, in one cycle (2026-10-11).

### Provider facts

From the football specification unless noted. "Owner-tested" means the owner confirmed it against the live API on the date given.

| Endpoint | Provider update rate | Multi-game form |
|---|---|---|
| `fixtures` | 15 seconds | `live=all`, which carries events where covered but no other detail (owner-tested 2026-10-11); `date`; `league` with `season`; `ids`, up to 20, which carries the full game detail (owner-tested 2026-10-11) |
| `fixtures/events` | 15 seconds | None: one `fixture` per call |
| `fixtures/statistics`, `fixtures/players` | 1 minute | None: one `fixture` per call (owner-tested 2026-10-10 for statistics) |
| `fixtures/lineups` | 15 minutes; published 20 to 40 minutes before kickoff | None: one `fixture` per call (owner-tested 2026-10-10) |
| `predictions` | 1 hour | None: one `fixture` per call (owner-tested 2026-10-10) |
| `injuries` | 4 hours; data from April 2021; one call a day recommended | `league` with `season`: a whole season in one call, without pagination (owner-tested 2026-10-11); optionally `date`; `ids` (up to 20) |
| API-AFL `games` | 30 seconds; no `live` parameter | `date`, `league` with `season` |
| API-AFL game detail | 30 seconds | `date`, or `ids` (up to 10) |

### The injuries tests

**Sample (2026-10-10).** The owner's request was `injuries?league=39&season=2025&date=2025-12-26`, made after the game. It returned 14 records on one page, all for game 1379145 (Manchester United v Newcastle, 26 December 2025, 20:00 UTC): seven per team, every one of type "Missing Fixture". The reasons range from injuries to international duty. The sample showed that records survive after the game, that a retrieval by competition and season narrowed to a date works, and that every record carries its game id, team, competition and season. With no "Questionable" record in it, it could not settle PRD OQ-6.

**Whole seasons (2026-10-11).** The owner retrieved every injury of a season with `league` and `season` alone, for MLS 2026 and the Premier League 2025:

- Each season came back in one response, without pagination, in the sample's format. The response is large: too large to attach to the PRD discussion.
- Both seasons hold "Questionable" records for past games, for example MLS on 21 February 2026 and the Premier League on 20 September 2025. The provider keeps the status after the game, which resolves PRD OQ-6.
- Not proven: that every pre-match record survives unchanged. A before-and-after comparison of one game would show it. Restatement checks detect a later change in any case (PRD FR-18).

**Backfill effect.** For past seasons, injuries cost one call per competition and season, instead of one call per game or one per 20 games with `ids`. For phase 1, that is about 2,200 calls instead of about 330,000 or 16,500.

### Design implications for architecture

- **Leaving the live feed is not the same as finishing.** Abandoned, cancelled and otherwise stopped games leave it too. Confirming the status costs no extra call: final processing starts by retrieving the game's detail, which carries its status. Games that leave the live feed in the same retrieval share one `fixtures?ids=` call per 20 games.
- **Some finished games never appear in the live feed.** Examples are competitions whose results arrive only after the game, and any game played while the stage watch was down. Only the day schedule catches these, so its cadence bounds their delay: about 5 minutes since 2026-10-11 (PRD FR-15).
- **A failed poll must not look like an empty feed.** If a provider error inside an HTTP 200 response were read as "no games in play", every live game would go to final processing at once.
- **A status can go back.** A game marked final can reappear as live when the provider corrects its status. The stage follows the provider (PRD FR-6).
- **Detail at the whistle may not be settled.** Statistics and player statistics update every minute, so detail fetched seconds after the whistle can miss the provider's last update. Player ratings may also be finalized later. Decided 2026-10-11: checks at 15 minutes and 1 hour after final (PRD FR-8). Through `fixtures?ids=`, both together cost about 100 to 1,100 calls a day, depending on how many games finish together.
- **Lineups update every 15 minutes at the provider.** Checks closer together than that shorten the wait between the provider's update and its capture. They cannot find data the provider has not yet updated. Games with nearby kickoffs can share each check through `fixtures?ids=`, if it carries lineups before kickoff as it does after the game (untested).
- **The wait before a predictions refresh.** The provider updates predictions hourly, at times it does not publish. A refresh a few minutes after a final may return the old prediction, and the hourly cap then blocks another try for an hour. The wait can be measured: compare when a refreshed prediction last changed (`LastModifiedUtc`) with when the triggering game was first seen final. Decided 2026-10-11: the wait is configurable and set from these measurements.
- **The predictions' hourly cycle.** The provider updates predictions once an hour, at a time it does not publish. Under an hourly cap, a refresh made just before the provider's update can leave a prediction almost an hour old. Measuring when in the hour predictions change (their `LastModifiedUtc`) lets refreshes be timed just after the update (PRD FR-17).
- **Comparing like with like.** Each schedule entry is compared with the latest stored schedule entry for the same game, from either schedule (PRD FR-21), so both sides have the same shape. Comparing schedule entries with game records instead would have needed care: if the two differed in any field, a naive comparison would see a change on every poll and retrieve the game every 5 minutes.
- **The 15-second cycle is a processing budget.** On a busy afternoon, about 150 live games mean about 8 `fixtures?ids=` responses per cycle, each carrying 20 games' lineups, statistics and player statistics. Retrieving, splitting, comparing and storing them must fit in 15 seconds, or the next cycle starts late. Architecture should plan for that, for example with content hashes for change detection and bulk writes. PRD FR-19 records each cycle's duration, so overruns are visible.
- **Kickoff times for the coming week** can come from seven day schedules, one call each, instead of one season schedule per competition (about 900 calls).
- **Pre-game work is timed per game.** Lineup checks, prediction refreshes and kickoff captures fall due at times set by each game's kickoff, so something must know what is due and when. The options in §2 apply here too: a tracking table, or due work derived from stored data.
- **The per-minute rate limit is unverified.** Batching keeps in-play detail small: about 20 calls a minute for 100 live games at the 15-second rate. Bursts now come mostly from predictions, which do not batch. Kickoff captures for 100 games that start at the same time, for example, are 100 calls in one minute. Spreading the captures over the last minutes before kickoff would smooth that. Injuries refreshes every 2 hours for several hundred competitions should likewise be spread across the 2 hours, not fired together. The quota headers in the fetch log will show the real limit.
- **Events come by one route.** The live feed carries events too, but since 2026-10-11 its game data is not stored, so events come only from `fixtures?ids=` (PRD FR-21). They are as fresh as the in-play cadence: about a minute.
- **Whole-season injuries are large.** Retrieving one whole season per in-season competition each day moves and compares a large response each time. A retrieval narrowed to one date is smaller, but it costs one call per competition per date. PRD FR-18 uses whole seasons because calls are scarcer than bandwidth. The pre-game refresh every 2 hours (2026-10-11) multiplies the volume: about twelve whole-season responses a day for each competition with a game coming up. Narrowing the pre-game refreshes to the dates of the coming games keeps responses small, for at most about twice the calls. Architecture decides.
- **Overlap with the prediction cascade (PRD FR-14).** For games not yet played, the daily refresh already picks up a restatement's effect within a day. For games already played, the cascade conflicts with the kickoff capture. Its reach, its cap and the options are in §3.

### Sizing (rough; every input unverified)

Inputs:

- the brief's estimate of about 550 games a day on average, with weekends well above that;
- about 900 competitions in season on a given day (about 1,100 competitions, with seasons running about 300 days a year);
- about 100 calls per game for each dataset polled once a minute during play (about 95 minutes of play plus stoppage time, with half-time skipped).

Updated 2026-10-11 for `fixtures?ids=` batching (PRD OQ-2), whole-season injuries (PRD OQ-6), and the owner's decisions of that day.

| Collection | Rule | Calls a day | Share of 75,000 |
|---|---|---|---|
| Live feed | Every 15 seconds, all day | Up to 5,760 | 7.7% |
| Day schedule | Every 5 minutes (decided 2026-10-11) | About 288 | 0.4% |
| Lineups | From 40 minutes before kickoff until stored, about 2 to 4 checks; games with nearby kickoffs share `fixtures?ids=` calls | About 200 to 600 | Under 1% |
| Predictions, daily refresh | 7-day horizon | About 3,850 | 5% |
| | Every future game of every season, mid-season | About 82,500 | 110% |
| Predictions after a final | Games in the same competition due within 48 hours (decided 2026-10-11); about 3 to 6 refreshes a game | About 1,500 to 3,500 | 2 to 5% |
| | Every game within 24 hours, hourly cap (not chosen) | About 13,200 | 18% |
| | Every game within 48 hours, hourly cap (not chosen) | About 26,400 | 35% |
| Kickoff capture | One per game | About 550 | 0.7% |
| Final processing | Games that leave the live feed join the next `fixtures?ids=` call, often one already made for live games | Up to 550 | Under 1% |
| Game records refreshed after a schedule change | `fixtures?ids=` for each change to a game that has a record and is not live | Tens | Under 0.1% |
| Early checks | 15 minutes and 1 hour after final, through `fixtures?ids=` | About 100 to 1,100 | Under 1.5% |
| Injuries | Before kickoff, every 2 hours for each competition with a game due within 48 hours (decided 2026-10-11); after the game, shared with final processing and restatement checks | About 5,500 to 8,500 | 7 to 11% |
| Restatement checks | Game detail through `fixtures?ids=`; injuries are in the row above | About 200 | 0.3% |
| In-play detail | Every live game in every 15-second cycle (decided 2026-10-11) | About 11,000 to 17,000 | 15 to 22% |
| | Once a minute, if lowered for technical reasons | About 3,000 to 4,000 | 4 to 5% |
| | For comparison: one game per call, team statistics alone | About 55,000 | 73% |

Without in-play detail, the total is about 18,000 to 25,000 calls a day (24 to 33%), before odds, the backfill and the prediction cascade (§3). In-play detail every 15 seconds adds about 11,000 to 17,000, for about 29,000 to 42,000 (39 to 56%), leaving about 33,000 to 46,000 for odds and the backfill. In-play detail is then the largest group, followed by predictions (about 5,900 to 7,900 calls a day) and injuries (about 5,500 to 8,500). The in-play range depends on how many games are live at once: off-peak, fewer games share each call. Weekend peaks raise every per-game row.

**Backfill effect.** For phase 1 (about 330,000 games), game detail through `fixtures?ids=` costs about 16,500 calls and injuries about 2,200. Predictions, at one call per game (about 330,000), are now nearly the whole backfill.

API-AFL is small beside this: one competition of about 9 games a round, on 7,500 calls a day. Its day schedule every 30 seconds costs about 120 calls an hour while games are due in play, and each of its detail endpoints covers a whole day's games in one call by `date`.

## 10. Caching by request and by game (2026-10-11)

### How the design evolved

All on 2026-10-11:

1. **Per-game caching (owner).** Batches built from the live feed, or from whatever games are due, rarely repeat. A call for games A-B-C is likely followed by A-B-D or A-B-C-D, so caching by request would almost never hit. The owner proposed splitting each response by game and caching per game, with a record of which games each response contained, and never reading live data from the cache.
2. **Per-dataset parts (assistant).** Store each dataset of each game separately, so that the live feed (the game and its events) and `fixtures?ids=` (everything) each update only what they carry. Withdrawn: the model would have to choose the newest part from several sources, and the two retrievals can finish out of order.
3. **One live cycle and two formats (owner).** Each live-feed retrieval triggers a `fixtures?ids=` retrieval for the live games and for the games that just left the feed, and the live feed's game data is not stored. A game would have a summary format from the schedules and a detail format from `fixtures?ids=`, with the model using the detail when it exists.
4. **One record per game (owner).** With game updates driven by triggers, only one version of a game needs storing. The assistant's condition: the schedules must also act only as triggers.
5. **Two caches (owner).** Injuries are not part of the fixtures feed, so they stay their own endpoint. The season schedule, by competition and season, is a request that repeats, so a cache keyed by request suits it; only the `fixtures?ids=` batches need the per-game cache. The assistant generalized the rule: any request that repeats uses the request cache, which also covers the day schedule, injuries and every other endpoint. How the queryable model combines the two caches was then deferred to the model's design (PRD OQ-10).

| Option | For | Against | Status |
|---|---|---|---|
| Per-dataset parts | Each retrieval updates only what it carries; the live feed's 15-second events could be stored | The model must choose the newest part from several sources; overlapping retrievals can finish out of order | Withdrawn |
| Two formats per game | The schedules' data is stored without extra calls | Two versions of each game; the model needs a rule for which to use, and a detail record goes stale when the schedule changes | Superseded |
| One record per game, schedules not stored | One version of each game | Schedule entries must be compared with game records of a different shape; nothing records what a schedule said; a re-run fetches the schedules again | Superseded |
| Two caches: a request cache for requests that repeat, a per-game cache for `fixtures?ids=` (renamed the per-record cache on 2026-10-17, when `sidelined` and `trophies` joined it) | Schedule changes are found by comparing like with like; the brief's request cache stays for every other request; re-runs resume from both caches | Two cache mechanisms to build | **Chosen** |

How the queryable model combines the two caches is deferred to the model's design (owner, 2026-10-11; PRD OQ-10). The likely direction is the full schedule table LEFT JOIN the per-game table, taking a game's summary fields from either by game stage, with each table also usable on its own. The assistant had suggested reading games only from game records. That needed an `ids` call for every game seen for the first time, about 8,000 a season-year. With the model deferred, that trigger was dropped: a game record is now created the first time something needs the game's detail.

### Relation to the brief's cache

The brief's request cache stays the default for every request that repeats. The per-record cache is added for batches of ids, whose composition varies: `fixtures?ids=`, and since 2026-10-17 `sidelined` and `trophies` (PRD FR-22). The brief proposed normalizing `ids`, so that `1-2-3` and `3-2-1` count as one request (brief addendum, §3). That is no longer needed, because `ids` responses are never looked up by request. The brief's rule that the queryable model resolves the latest version of each entity across overlapping requests still applies to the request cache. Injuries, for example, reach the model through competition-and-season responses.

### Assistant's additions (pending owner confirmation)

- **Newer wins.** When retrievals of the same game overlap, an older one must never overwrite a game record written by a newer one (PRD §5, item 5).
- **Freshness of game records.** `LastModifiedUtc` moves only when content changes (PRD FR-12), so per-game freshness needs the time each game record was last retrieved. It can be kept as a "last retrieved" time on each record, or derived from the fetch log's list of the games each retrieval returned, which is the owner's record of "which games each response contained". Architecture chooses. The request cache keeps the brief's per-request freshness.
- **Missing games.** A game requested in a batch but absent from the response is recorded as missing. It is not treated as fresh.
- **Envelopes go to the fetch log.** An `ids` response is split by game, so its envelope (parameters, errors, result count, paging) is recorded with its retrieval, and splitting loses nothing the provider returned (PRD §5, item 1). The request cache stores whole responses, envelope included.

## 11. Season-level and reference data (raised 2026-10-11; owner's rules 2026-10-17)

Everything that is not per-game data: PRD §4.2, items 3 and 4. Facts come from the football specification. The sizing is rough, and every input is unverified.

### Provider facts

| Group | Endpoint | Provider update rate | Unit of one call |
|---|---|---|---|
| Changes when games are played | `standings` | Every hour; some competitions have several rankings (group phases, opening and closing rankings) | A competition and season |
| | `teams/statistics` | Twice a day; an optional `date` returns the statistics up to that date | One team in a competition and season |
| | `players` (player statistics) | Several times a week | 20 players per page, by competition and season or by team and season |
| | `players/topscorers`, `topassists`, `topyellowcards`, `topredcards` | Several times a week | A competition and season (top 20) |
| Changes at any time | `players/squads` | Several times a week (one call a week recommended) | One team |
| | `transfers` | Several times a week | One team or player |
| | `coachs` | Every day | One team or coach |
| | `sidelined`, `trophies` | Several times a week | Up to 20 players or coaches, named by id |
| | `players/profiles` | Several times a week (one call a week recommended) | 250 players per page, or one player |
| | `players/teams` (careers) | Several times a week | One player |
| Schedules and reference | `fixtures` by competition and season (season schedule) | Every 15 seconds | A competition and season |
| | `fixtures/rounds` | Every day | A competition and season |
| | `leagues` (with the coverage map) | Several times a day (one call an hour recommended) | Every competition |
| | `leagues/seasons`, `countries`, `timezone`, `teams/countries` | Rarely | Everything |
| | `teams` (each team's home venue included) | Several times a week | A competition and season, a country, or a team |
| | `venues` | Several times a week | One venue, or every venue in a country |
| | `teams/seasons` | Several times a week | One team |
| Possibly carried | `fixtures/headtohead` | Every 15 seconds | Two teams' games against each other, which the stored schedules already hold (PRD FR-2) |

### The owner's rules (2026-10-17)

The rules are in PRD FR-22, and the derivable endpoints in PRD FR-23. They were set in two rounds on 2026-10-17. Where the second round changed the first, only the result is shown.

- **Standings** are retrieved when a game in the competition turns final. Calculating them from results works for most competitions, but not for those with their own rules. The owner's example is the Belgian league, which for many years split its season in two, halving points for the second phase and running play-off groups with different point rules. Rules also change between seasons, so keeping them current for every competition is not sustainable. The provider offers no live standings (owner-confirmed).
- **Team and player season statistics** are retrieved after each game ends, for both teams, by competition, season and team, when the quota allows. They stay collected so that calculations from game data can be checked against them. Some team statistics cannot be built by adding up player statistics.
- **Calculations** of derivable data, including live standings, happen outside this ingestion project.
- **Derivable, not collected:** the four leaderboards, `fixtures/rounds`, `leagues/seasons`, `fixtures/headtohead` (every game between two teams, which the stored games hold once the seasons are backfilled), `players/seasons`, `teams/seasons`, and `players/teams` (player careers, which cover only what the provider covers: nothing before 2008 and nothing from uncovered competitions).
- **Teams** daily per competition, configurable per competition. **Squads** weekly, plus triggers from transfers and from lineups.
- **Transfers** daily for top-tier teams and weekly for the rest, plus a call when a lineup holds a player missing from the team's stored squad. The owner had first set every 12 hours, thinking transfers was one call for everything; it needs a team or a player.
- **Sidelined** daily for top-tier teams and weekly for the rest, like transfers, with multi-id responses confirmed to name each entry's player or coach, so they can be cached per record.
- **Trophies** need a better way to make the calls: likely once at the end of each season, especially if the endpoint holds only team honours rather than individual awards. They are addressed only after the backfill is complete.
- **Top tier** is designated by hand, in configuration, not by a rank cut-off.
- **Coaches** one call per team about 2 days before each of its games.
- **Player profiles** once a season, and **venues** once a season by country, both keeping dated versions only when content changes (PRD FR-24). Venues carry capacity, not attendance, and `teams` returns only part of a venue's data.
- **Leagues, countries, teams/countries and timezone** daily. `teams/countries` lists different countries from `countries`.
- **API-AFL** follows the same principles.
- **Provider update timing** is validated, not assumed (PRD FR-25). The owner reads the stated rates as a guide to how often to check, not as the moment data changes.

### Provider data quality: coach ids in lineups

A lineup's coach node can carry the id 0. The owner's example: in fixture 1490541, the Philadelphia Union (team 1599) coach Ryan Richter appears with id 0, while `coachs?team=1599` returns him as coach 26109, with career dates (Philadelphia Union from 2026-08-01; Philadelphia Union II from 2026-01-01 to 2026-07-01). The lineup is stored as returned, because ingestion never corrects provider data (PRD §5). Data verification (PRD §4.11) can flag zero ids, and downstream can resolve the game's coach from the `coachs` career dates. Coach ids for `sidelined` and `trophies` come from `coachs`, not from lineups.

### Analysis and open points

- **Validating the provider's update timing (decided 2026-10-17, PRD FR-25).** The specification says standings update hourly, team statistics twice a day and player statistics several times a week. If that held literally, a retrieval made as a game turns final could miss the game, and for the last game of a round nothing would refresh the numbers until the competition played again. The owner reads the rates as guidance on how often to check, and wants the real timing measured. A design sketch:
  - take about 100 games from about 20 competitions;
  - retrieve standings, both teams' statistics and both teams' player statistics at final, then 15 minutes, 1 hour, 3 hours, 12 hours, 1 day and 3 days later;
  - record, for each endpoint, the first retrieval whose content changed after the final;
  - cost: about 5,000 calls, once.

  If the data lags, existing checks can carry the later pulls: standings again at the game's 1-hour check (PRD FR-8), season statistics at its first daily check.
- **Top tier.** Transfers and sidelined data refresh more often for top-tier teams. The owner designates top-tier competitions by hand, in configuration (where it lives is PRD OQ-5). The sizing assumes about 50 competitions, about 1,000 teams.
- **Trophies.** Matching sidelined would have cost about 4,100 calls a day for one of the lowest priorities. Trophies change mostly at the end of seasons and tournaments, so the owner wants a better way to make the calls. A proposal:
  - when a competition's season closes (PRD §3), retrieve trophies for every player and coach who appeared in it, 20 per call: about 33,000 calls a year, about 90 a day on average;
  - once the backfill is complete, one pass over every stored player and coach captures their trophy histories;
  - narrower still: only the players and coaches of the teams that placed.

  A season-end rule suits the endpoint if it holds only team honours, such as winner or runner-up of a competition. One call for a well-known player shows whether individual awards appear too.
- **The backfill grows.** Each past competition and season adds about 62 season-level calls: standings, teams, statistics for about 20 teams, and player statistics at about 2 pages per team. For phase 1's 2,200 competition-seasons, that is about 136,000 calls, which takes phase 1 to about 490,000 calls.

### Sizing (rough)

Inputs: about 900 competitions in season on a given day, about 300 of them with games; about 550 games a day; about 15,000 teams, 15,000 coaches and 375,000 players; about 1,000 top-tier teams (assumed, unverified).

| Endpoint | Rule | Calls a day |
|---|---|---|
| `standings` | When a game turns final | About 600 |
| `teams/statistics`, `players` | After each game, both teams, when the quota allows | About 3,300 |
| `teams` | Daily per competition in season | About 900 |
| `players/squads` | Weekly, plus triggers | About 2,400 |
| `transfers` | Daily for top-tier teams, weekly for the rest, plus triggers | About 3,000 |
| `sidelined` | Daily for top-tier teams, weekly for the rest, batched across teams | About 4,100 |
| `trophies` | When each competition's season closes (proposal) | About 90 on average |
| `coachs` | Per team about 2 days before each game | About 1,100 |
| `players/profiles`, `venues` | Once a season | Under 100 |
| `leagues`, `countries`, `teams/countries`, `timezone` | Daily | 4 |
| Season schedules | Daily per competition in season | About 900 |

Season-level data takes about 16,500 calls a day (22%). Added to game-level collection (about 29,000 to 42,000, §9), the total is about 45,500 to 58,500 (61 to 78%), leaving about 16,500 to 29,500 for odds and the backfill. At that rate, phase 1 of the backfill takes about 2.5 to 4 weeks, before odds take their share. Before the owner's 2026-10-17 trophies decision, with trophies matching sidelined, the season-level figure was about 20,500.

## 12. Pre-match odds (raised 2026-10-17; schedule by date adopted 2026-10-18)

PRD §4.2, item 5, and PRD §4.8. Facts come from the two specifications. The sizing is rough, and every input is unverified.

### Provider facts

| Endpoint | Availability and updates | Unit of one call |
|---|---|---|
| `odds` (API-Football) | 1 to 14 days before a game, with 7 days of history; updated every 3 hours | One game, or 10 games per page by date or by competition and season; filters by bookmaker and bet type |
| `odds/mapping` | Every game that has odds; updated daily | 100 games per page |
| `odds/bookmakers`, `odds/bets` | Reference lists; several times a week | Everything |
| `odds` (API-AFL) | 1 to 7 days before a game, with 7 days of history; updated four times a day; 14 bookmakers, 39 bet types | One game |
| `odds/live`, `odds/live/bets` | In play only, no history | Later tier (PRD §8.1) |

### Carried over from the brief (brief addendum, §5)

- **Phase 1:** the closing line only, keeping every call. Regular snapshots come later.
- **Later cadence:** every 1 to 3 hours, calibrated against the quota, after a calibration experiment that hashes each payload to see how often consecutive snapshots differ.
- **Direction:** "never miss a line move", eventually storing only snapshots that differ from the previous one.
- **Closing-line test, not yet run:** after a game ends, fetch `odds?fixture=<id>` and compare the odds' `update` time with kickoff. If the odds are frozen at kickoff and still available, the closing line can be collected after the game. If they change during play, it must be captured before kickoff.
- **Storage** is the binding constraint on cadence, not quota: every snapshot is kept, and each payload carries every bookmaker and bet type.

### The owner's answers (2026-10-17)

- **The test, first half:** a game's pre-match odds were still available the day after it. The closing line can therefore be taken after the game, from the 7-day history (PRD FR-26). Still to check: whether each bookmaker's `update` time is at or before kickoff, which shows the odds froze when the game started.
- **Storage** is not a concern for now; reducing it comes later. Every bookmaker, every bet type and every snapshot is kept. PRD §4.8 labels this as a shortcut against the brief's "never miss a line move" direction, which stores only snapshots that change.
- **Defaults accepted:** `odds/mapping` daily for gap detection, odds stored per game, and API-AFL closing lines per game.
- **The test, second half (resolves PRD OQ-1):** the sample game kicked off at 2026-10-16T19:00:00Z, and its odds were last updated at 18:09:16Z, 51 minutes before kickoff. The odds stop updating before kickoff and stay available afterwards.
- **The plan until the backfill is complete:** closing lines first, plus earlier odds, because odds are wanted in the analysis leading up to each game. The final pull is 5 to 10 minutes before kickoff, with further pulls about 3, 6, 12, 24 and 48 hours before. Calls are batched, and simplicity beats precision: each pull may be 1 to 4 hours off its point, tighter closer to kickoff. This replaces the brief's phase-1 rule of closing lines only (PRD FR-26, FR-27).

### What the closing-line test decides

| Test result | When the closing line is taken | Calls a day |
|---|---|---|
| Frozen at kickoff, still available after the game | After the game, by date, 10 games per page | About 55 |
| Changed during play, or removed after kickoff | At kickoff, one call per game, like the predictions' kickoff capture (PRD FR-17) | About 550 |

**The closing line (PRD FR-26).** The final pull 5 to 10 minutes before kickoff costs one call per game, about 550 a day, or less where several games of a competition kick off together. The assistant's refinement, pending confirmation: keep the next-day retrieval by date (about 55 a day) as a safety net. It catches a provider update in the last minutes before kickoff and a failed pre-kickoff pull, and a game still missing is retried daily until its odds leave the 7-day history. With `odds/mapping` (about 77 calls a day for 7,700 games) and the two reference lists, closing-line work costs about 700 calls a day.

### Snapshots before kickoff (superseded on 2026-10-18 by the schedule by date)

Two ways to make the pulls at about 48, 24, 12, 6 and 3 hours before kickoff:

| Approach | How | Calls a day | Timing |
|---|---|---|---|
| A sweep by date (assistant's recommendation) | Every 3 hours, retrieve by date every game due within the next 48 hours: about 3 dates × 55 pages per sweep | About 1,320 | Every game gets a snapshot within about 1.5 hours of each point, plus the snapshots in between |
| The same sweep every 2 hours | As above | About 2,000 | Within about 1 hour of each point, which meets the tightest tolerance |
| A pull per game per point | One call per game at each of the five points, grouped where games share a competition and a kickoff | Up to about 2,750 | Close to each point; needs per-game timers |

The sweep is the simpler design: a fixed schedule, no per-game timers, and fewer calls, because 10 games share each call. Its extra snapshots cost storage, which is not a concern for now.

### A schedule by date (owner's proposal 2026-10-17; adopted 2026-10-18, PRD FR-27)

The owner noticed that how odds are pulled had never been set, and proposed a cadence for each date, retrieved by date at 10 games per call:

| Date | Cadence | Pulls a day | Calls a day, at about 55 pages a date |
|---|---|---|---|
| Yesterday | Every 6 hours | 4 | About 220 |
| Today | Every hour | 24 | About 1,320 |
| Tomorrow | Every hour | 24 | About 1,320 |
| In 2 days | Every 2 hours | 12 | About 660 |
| In 3 days | Every 6 hours | 4 | About 220 |
| In 4 days | Every 6 hours | 4 | About 220 |
| **All** | | | **About 3,960** |

- **Against the previous plan.** About 4,000 calls a day, against about 2,000 for the 3-hourly sweep with a final pull per game and a next-day safety net. The schedule covers every earlier point: 48 hours out falls on the 2-hourly day, within about an hour, and 24, 12, 6 and 3 hours out fall on hourly days, within about 30 minutes. It also adds days 3 and 4.
- **Yesterday every 6 hours** takes over the next-day safety net, with four tries a day instead of one. A game still missing after that is retried per game until its 7-day history ends (PRD FR-26).
- **The final pull 5 to 10 minutes before kickoff** is not part of the schedule. With hourly sweeps, the latest snapshot before kickoff is at most an hour old; the owner's sample game was last updated 51 minutes before kickoff. The owner kept the final pull per game (2026-10-18), at about 550 calls a day.
- **Totals.** With game-level (§9) and season-level (§11) collection, the schedule brings the total to about 49,500 to 62,500 calls a day (66 to 83%), or about 50,000 to 63,000 with the final pull. That leaves about 12,000 to 25,500 for the backfill, so phase 1 takes about 3 to 6 weeks.
- **Pages per date are an upper bound.** 55 pages assumes every game of an average day has odds. One call, `odds?date=<a busy Saturday>`, gives `paging.total` and calibrates the figure. Weekend dates hold two to three times the games, so on days whose window covers a weekend, odds calls can reach about 8,000 to 12,000.
- **Storage.** Each game gets about 72 snapshots: about 55 before kickoff, and the rest after it, when the odds no longer change. Every snapshot carries every bookmaker and bet type. Measuring one game's payload, as the brief planned, turns this into a storage figure. Storing only snapshots that changed would remove the frozen copies first.

### Regular snapshots, after the backfill

Retrieved by date, a full sweep of the 14-day window is about 14 dates × 55 pages, or about 770 calls. Every 3 hours that is about 6,200 calls a day (8%); every hour, about 18,500 (25%). By game instead, the same sweep would take about 7,700 calls.

Running totals, with game-level (§9) and season-level (§11) collection:

| Odds | Total calls a day | Left for the backfill |
|---|---|---|
| Closing lines only | About 46,200 to 59,200 (62 to 79%) | About 15,800 to 28,800 |
| Closing lines plus a 3-hourly sweep of the next 48 hours (superseded) | About 47,500 to 60,500 (63 to 81%) | About 14,500 to 27,500 |
| **Chosen (2026-10-18):** the schedule by date plus the final pull per game | About 50,000 to 63,000 (67 to 84%) | About 12,000 to 25,000 |
| Plus snapshots every 3 hours | About 52,400 to 65,400 (70 to 87%) | About 9,600 to 22,600 |
| Plus snapshots every hour | About 64,700 to 77,700 (86 to 104%) | Little or none |

Phase 1 of the backfill is about 490,000 calls (§11), so it takes about 2.5 to 4.5 weeks with closing lines only, about 2.5 to 5 weeks under the plan until the backfill is complete, and about 3 to 7 weeks with snapshots of the whole 14-day window every 3 hours. The brief's rule that the odds cadence slows when the backfill needs the quota shortens the second range.

### Design implications

- **Per-record storage.** Pages by date hold whichever 10 games have odds at that moment, so a page's content shifts between calls. Splitting each response by game, as for `fixtures?ids=` (PRD FR-21), keeps each game's snapshots together.
- **Knowing which games should have a closing line.** `odds/mapping` lists every game that has odds. Checking it against the closing lines captured turns a missed closing line into a recorded gap rather than a silent one (PRD §5, item 2).
- **Pages by date drift.** A page holds whichever 10 games have odds at that moment, so a game can move between pages while a sweep is fetching them, and be missed or fetched twice. Fetching twice is harmless. A miss shows up as a missing snapshot, recorded as a gap, and the closing line has its own pulls (PRD FR-26).
- **The 3-hour rate is guidance too.** As with standings (§11), the stated rate may not be when odds actually change. The calibration experiment measures it.

## 13. The backfill (raised 2026-10-18; decided 2026-10-20, except self-start)

PRD §4.2, item 6, and PRD §8.2. The sizing is rough, and every input is unverified.

### From the brief

- The backfill is a workflow for "past seasons for competitions chosen at run time", run on demand, resuming through the cache (brief addendum, §4).
- The coverage phases set its order: phase 1, the current and previous season of every competition, major competitions first; phase 2, every season in progress on 1 January 2022 or starting after it; phase 3, as many as possible (PRD §8.2).
- "While the current and previous seasons fill in, the backfill wins. Odds stay lean, and the odds cadence slows when the backfill needs the quota. Days with fewer games give the backfill more quota."
- A cold-start test asks how to backfill a competition, so the procedure must be documented.

### What a competition and season costs

| Data | Calls for a season of about 150 games and 20 teams |
|---|---|
| Season schedule | 1 |
| Game records, 20 games per call | About 8 |
| Predictions, one call per game (no kickoff capture exists for a past game, PRD FR-17) | About 150 |
| Injuries, whole season (seasons from April 2021) | 1 |
| Standings and teams | 2 |
| Team statistics, one call per team | About 20 |
| Player statistics, about 2 pages per team | About 40 |
| **All** | **About 220** |

Not backfilled: pre-match odds older than the 7-day history, and the dated versions of profiles and venues, which start when collection starts. Squads need care. The `players/squads` endpoint takes only a team or a player, with no season or date, so it returns today's squad, and a past registered squad cannot be retrieved from it. Squads at game level are backfilled, though: each game record's lineup holds the matchday squad (starters and substitutes), and player statistics by team and season list who played for each team. Only the full registered squad on a past date, including players who never made a matchday squad, is lost. Transfers, sidelined data and coaches need no backfill, because their current pulls return each team's or person's whole history. Trophies come after the backfill (PRD §8.2).

- **Phase 1:** about 2,200 competition-seasons, about 490,000 calls. Predictions are about 330,000 of them, and everything else about 160,000.
- **Phase 2:** about 3 to 4 more seasons per competition, roughly 750,000 to 1,000,000 calls.
- **Time:** at the 12,000 to 25,000 calls a day left after everything else (§12), phase 1 takes about 3 to 6 weeks, and phase 2 about 1 to 3 months more.

### A gap in the restatement rules

PRD FR-8 counts the restatement window from the time a game was first seen final. A game loaded by the backfill is first seen final when the backfill reaches it, possibly years after it was played. As written, every backfilled game would get the 15-minute and 1-hour checks and seven daily checks: about 150,000 extra calls for phase 1, for games long settled. Two other rules have the same problem:

- PRD FR-7's arrival targets would flag no backfilled game as late, but they would measure delays that mean nothing for a game played long ago.
- PRD FR-22's season-close trophies trigger would fire for every backfilled season, which the owner wants handled after the backfill instead.

The assistant proposed treating a game first seen final more than 7 days after its kickoff as past its window.

**The owner's rule (2026-10-18), in PRD §3 and FR-8:**

- A competition-season is complete once its last scheduled game is more than 6 months in the past; the owner would consider 3 months. The PRD starts at 3, configurable.
- A game first seen final more than 3 months after it was played is past its restatement window from the start.
- A game first seen final within 3 months of being played, such as a recent game loaded by the backfill, gets the full 7-day window from when it is first seen. For games of roughly the last 3 months, about 50,000 at 550 a day, that costs about 22,500 calls for game records over the backfill's run, instead of about 150,000 for every backfilled game.
- A season the backfill finds already complete does not trigger the season-close trophies pull; the pass after the backfill covers it.
- The assistant added that the arrival targets (PRD FR-7) apply only to games whose end the stage watch sees.

### Ordering options for phase 1

| Order | Effect | Status |
|---|---|---|
| Competition by competition, all data (the brief's "completed in priority order") | Top competitions are complete within days; minor competitions' games and statistics wait until the end, 3 to 6 weeks out | Superseded |
| Every competition's games and statistics first, predictions after | All of phase 1 except predictions lands in about 1 to 2 weeks; predictions follow over 2 to 4 weeks | **Chosen** (2026-10-20) |

### Decisions (2026-10-20)

- **3 months** for both rules: a season is complete once its last scheduled game is more than 3 months past, and a game first seen final more than 3 months after it was played is past its restatement window.
- **Phase 1 in two passes:** games and statistics first, then predictions, each in priority order (PRD §8.2, FR-28).
- **Manual start:** the owner must be able to start a backfill run by hand. Whether it also starts itself is deferred; automation is generally preferred (PRD OQ-11).
- **Arrival targets** apply only to games whose end the stage watch sees (PRD FR-7).

## 14. Quota budget (raised and decided 2026-10-20)

PRD §4.6. API-Football's Ultra plan allows 75,000 calls a day. API-AFL has its own 7,500, and its demand is small. Every figure below is rough and unverified.

### From the brief

- Workflows share one daily quota per API.
- While the current and previous seasons fill in, the backfill takes priority and odds stay lean; days with fewer games give the backfill more room. That rule dates from when phase-1 odds meant closing lines only.
- Quota exhaustion arrives as an HTTP 200 response with an error inside it, and exit codes must separate retryable failures, such as quota, from non-retryable ones.
- The per-minute limit and the time of the daily reset are unverified. The quota headers in the fetch log will show both.

### Demand, and what a cut would lose

| Group | Calls a day | If cut on a given day |
|---|---|---|
| Odds: closing-line pulls (5 to 10 minutes before kickoff, yesterday's pulls, `odds/mapping`) | About 850 | A closing line can be retried for 7 days; only its timeliness for pre-game analysis is lost |
| Odds: the schedule by date, other days | About 3,750 | **Lost for good.** The endpoint has no time parameter, so an earlier state of a game's odds can never be requested |
| Stage watch: live feed and day schedule | About 6,050 | Delay; the day schedule finds finished games later |
| Final processing and early checks | About 650 to 1,650 | Delay |
| Pre-game captures: lineup checks, prediction kickoff captures, pre-game injuries | About 6,250 to 9,650 | Timeliness for pre-game analysis; the data stays retrievable after the game |
| In-play detail | About 11,000 to 17,000 | The live view; final processing stores the end state anyway |
| Refreshes: predictions (daily, after finals, cascade), the reference refresh with season-level data, daily restatement checks | About 18,750 to 20,750 | Freshness |
| Season statistics after each game | About 3,300 | Freshness; the owner set them "when the quota allows" |
| Backfill | What is left: about 12,000 to 25,000 | Delay |

Only the schedule's odds snapshots are lost for good when cut. Everything else is delayed or less fresh.

### A proposed order (highest first)

1. Odds: closing-line pulls and the schedule by date.
2. Stage watch.
3. Final processing and early checks.
4. Pre-game captures.
5. In-play detail.
6. Refreshes.
7. Season statistics.
8. Backfill.

This puts the backfill last, which supersedes the brief's phase-1 rule: odds snapshots cannot wait, and the backfill can.

### Running short during a day

One way to keep the order meaningful: each time work starts, compare the quota left for the day with what the work above it still needs before the reset, estimated from the day's schedule. Lower work stops first when the two meet, and each stop is recorded. Without such a reserve, work runs first come, first served until the quota is gone, and an evening's closing lines can find nothing left.

### In-play detail's share

PRD FR-19 gave in-play detail a share of the quota. Under an order, the share can simply be its place: in-play runs at its full rate unless the work above it needs the calls.

### Decisions (2026-10-20)

- **The order** is approved as proposed (PRD FR-29). It replaces the brief's phase-1 rule, and the odds pulls never give way to the backfill.
- **The reserve** is approved (PRD FR-30). The owner added that the backfill's priority rises when top-tier competitions are not active, especially during international breaks. The assistant's reading, pending confirmation: such a day has no game in any top-tier competition, and on it the backfill moves above refreshes and season statistics.
- **In-play detail** has no fixed share; its place in the order decides. While the backfill runs, its rate starts at 60 seconds instead of 15 (PRD FR-19).

### Totals with in-play at 60 seconds during the backfill

In-play detail at 60 seconds costs about 3,000 to 4,000 calls a day instead of 11,000 to 17,000, which frees about 8,000 to 13,000 for the backfill. Total collection is then about 42,000 to 50,000 calls a day (56 to 67%), leaving about 25,000 to 33,000 for the backfill. Phase 1, about 490,000 calls, takes about 15 to 20 days; phase 2, about 750,000 to 1,000,000 calls, about 3 to 6 weeks more. Days without top-tier games add to that.

### Backfill windows (decided 2026-10-20)

The assistant first read the owner's "times when top-tier leagues are not active" as days with no game in any top-tier competition. That would have failed during international breaks if national-team competitions were on the top-tier list. The owner chose explicit windows instead: the international breaks of the FIFA International Match Calendar, kept by hand in a configuration file or a curated table (PRD FR-29, OQ-5). During a window, the backfill ranks above refreshes and season statistics, below in-play.

| Window (dates inclusive) | Note |
|---|---|
| 2026-11-09 to 2026-11-17 | International break |
| 2027-03-22 to 2027-03-30 | International break |
| 2027-06-07 to 2027-06-15 | International break |
| 2027-09-20 to 2027-10-05 | International break |
| 2027-06-19 to 2027-07-11 | Not a FIFA window. The CONCACAF Gold Cup and the 2027 Africa Cup of Nations heavily reduce CONCACAF and CAF club games while UEFA's major competitions are on their summer break (owner, 2026-10-20) |

**Margin (decided 2026-10-20).** Every window is widened by 1 day on each side, because the owner expects the day or two around a break to have fewer club games too. The reason is recorded so the margin can be adjusted: if the daily quota prediction or actual use shows no drop on margin days (PRD FR-31), the margin is reduced or removed.

**Daily quota prediction (owner, 2026-10-20; PRD FR-31).** Each day, the container predicts each priority group's calls from the day's schedule, the odds schedule by date, the refresh cadences and recent use, and the backfill gets what is left. The prediction feeds the reserve (PRD FR-30). Stored next to actual use, it measures its own accuracy and tests the windows and margins. Looking 7 days ahead, which the schedules allow, would also forecast the backfill's progress (pending confirmation). During a break, club collection shrinks sharply, so the backfill can take most of the quota that in-play detail, pre-game captures and refreshes would otherwise use.

## 15. Container interface (raised and decided 2026-10-20)

PRD §4.1.

### From the brief

- One container interface for every source: how a run is parameterized, how secrets are supplied, what exit codes mean and how re-runs behave.
- Documented well enough that the integrator can write an Airflow DAG without reading the source, and a cold-start agent can answer questions such as how to backfill a competition.
- Exit codes distinguish success, a retryable failure (quota, network) and a non-retryable failure (such as bad configuration).
- Airflow starts workflows on a schedule. Checking every minute would mean 1,440 runs a day; architecture was to compare that with fewer, longer runs that poll internally.

### What the PRD has added since

- **Work that runs on a clock finer than any schedule:** the live feed every 15 seconds (PRD FR-15), the in-play cycle (FR-19), lineup checks from 40 minutes before kickoff (FR-16), prediction kickoff captures (FR-17), odds pulls 5 to 10 minutes before kickoff and hourly by date (FR-26, FR-27).
- **Work started by hand:** backfill runs (FR-28), re-processing requests (FR-13) and the timing validation (FR-25).
- **A quota reserve** that weighs all work together (FR-30), so every piece of work must see the same quota state.
- FR-15 left two run models to architecture: a run that stays up and polls, or a run every minute that polls four times.

### Run model options

| Model | How | For | Against | Status |
|---|---|---|---|---|
| Two modes | A continuous mode stays up and runs everything on a clock; one-off runs handle work started by hand, then exit with a code | Meets the 15-second cadence and minute-precise timers directly; one place sees the whole day's quota | Exit codes say little about a process that never exits; something must restart it if it stops | Not chosen |
| Scheduled runs only | Airflow starts every run; each does what is due and exits | Every run ends with an exit code, as the brief assumed | Many container starts a day; timers land only to the minute | **Chosen** (2026-10-20) |

### Other points

- **Runs at the same time.** In either model, a run started by hand can overlap clock-driven work. Coordination through the database keeps a game from being processed twice (FR-11) and keeps every run inside the same reserve (FR-30).
- **Gaps and exit codes.** A run can finish with gaps it recorded: explained absences, or work skipped under the quota. Either such a run exits as a success and the gaps live in the fetch log and run tracking (§4.9), or it gets a fourth exit code, "completed with gaps", which a DAG can flag.
- **Configuration** (PRD OQ-5: ranks, the top-tier list, per-competition rules, backfill windows, cadences). A file in the repository is versioned, reviewable and readable by a cold-start agent; mounted into the container, it changes without an image rebuild. A control table changes with a SQL statement, outside version control unless exported.
- **Secrets.** API keys come from environment variables or Docker secrets, never from the image, the repository, a document or a log line.

### Decisions (2026-10-20)

- **No continuous mode.** Every workflow runs as scheduled runs that exit (PRD §4.1, FR-32).
- **The live feed runs one run at a time.** Each run starts as soon as the previous one ends, or 15 seconds after the previous one started, whichever is later.
- **Overlap is ruled out only for the same parameters.** Two runs of one workflow with the same parameters would race to write the same data, so they never overlap. Runs with different parameters, such as backfill runs for different seasons, may (owner's correction, 2026-10-20).
- **Exit codes, gap and error tracking, and starting needed one-off runs** are decided in architecture and implementation. The brief's three kinds of exit code stay the minimum.
- **Configuration** lives in a file in the repository, mounted into the container (PRD OQ-5, resolved).

### Running the live feed under Airflow (input for architecture)

The owner asked whether Airflow can keep a single run going and start the next straight away, or 15 seconds after the last one started. It can, with caveats:

- `max_active_runs=1` keeps a DAG to one run at a time.
- The `@continuous` schedule, available from Airflow 2.6, starts a new run as soon as the previous one ends; it requires `max_active_runs=1`. The server's Airflow version needs checking.
- To keep runs at least 15 seconds apart, a run that finishes early waits out the rest of its 15 seconds before it exits.
- Airflow is built for batch work. Scheduler latency and container start-up add seconds to every run, so the real spacing has to be measured (PRD FR-19 records it).
- One way to spread the start-up cost: each run performs several 15-second cycles, for example four, about a minute, before it exits. That cuts container starts from about 5,760 a day to about 1,440.
- The other clock-driven work (lineup checks from 40 minutes before kickoff, the odds pull 5 to 10 minutes before, hourly odds, refreshes, the backfill) fits ordinary schedules in separate DAGs. Their timing tolerances are minutes, not seconds.
