---
title: "Product Brief: API-Sports Ingestion Container"
status: final
created: 2026-10-05
updated: 2026-10-06
---

# Product Brief: API-Sports Ingestion Container

## Executive Summary

This project builds a Docker container that copies football and AFL data from API-Sports into a self-hosted SQL Server 2025 instance, to feed personal sports analytics. Airflow runs it on a schedule. It replaces earlier pipelines that ran on cloud credits and stopped when the credits ended. This one runs on owned hardware, and the API subscriptions are its only recurring cost. The first goal is the current and previous season of every competition, major competitions first, with finished games processed within minutes of the final whistle.

Data lands exactly as the provider returns it. It is exposed through a queryable model that mirrors the provider's own data model. Game data can be fetched again whenever needed, so it is refreshed in place. Betting odds cannot: the provider keeps only seven days of history. Odds are therefore never overwritten. Collection starts with each game's closing line and expands from there. A request cache avoids redundant calls, a shared fetch log records every retrieval, and a failed workflow can simply be run again.

The container is also the first instance of a reusable ingestion pattern. Every future source shares a common core, including one container interface, a faithful raw landing zone and shared logging. Retrieval-specific adapters, starting with HTTP, sit beneath them. Decisions are recorded as architecture decision records (ADRs). Everything is documented well enough that AI agents, or the owner returning after months away, can operate and extend the system from the repository alone.

## The Problem

There is no running source of sports data for the personal analytics this project exists to feed. Earlier pipelines ran on about $300 a month of Azure credits and stopped when the credits ended. Their data was deliberately not kept, so this project starts from zero. Cost now has a structural fix: SQL Server 2025 and Airflow run on a self-owned server, and this project must fit that environment.

Some data can be recovered later and some cannot. Fixtures, results, statistics, lineups and predictions can be fetched again at any time, so a gap in collection is an inconvenience. Pre-match betting odds are different. API-Sports publishes them only in a window before each game (1–14 days for football, 1–7 for AFL) and keeps seven days of history. For the odds feed, downtime means permanent data loss.

More sources will follow: another API next, and possibly open-source sports data libraries after that. Without a shared approach, each would rebuild the same machinery:
- avoiding redundant calls against a quota
- landing raw data so it can be reprocessed later
- giving the scheduler a predictable container to run

This project builds that machinery once.

## Who This Serves

One person fills all three roles, but each role needs something different:

- **The analyst** needs football and AFL data that is correct as of any point in time, plus a complete odds timeline. All of it must be queryable in SQL Server for cleaning, merging with other sources and modeling.
- **The integrator** connects the container to Airflow. The integrator needs the container's interface documented well enough to write a DAG without reading the source.
- **The next builder** applies the pattern to a new source. The next builder needs to reuse the pattern without working out the decisions behind it again.

Two conditions apply to all three:
- **AI-assisted work.** All work is done with AI agents under the BMAD method, so documentation is written for agents as much as for people. When it is unclear whether to document something, it gets documented.
- **Long gaps.** The repository will go untouched for months at a time. Everything needed to debug and maintain it must be recoverable from the repository alone.

The project is also a deliberate exercise in building a pipeline end to end and owning every architectural decision. This is a lens on how the roles are served, not a fourth audience. In practice, every decision is made explicitly and recorded as an ADR. When a decision changes, a new ADR supersedes the old one; accepted ADRs are never edited.

## The Solution

The project delivers a Docker container that collects data from API-Football (v3) and API-AFL (v1) and lands it in SQL Server 2025. It also delivers documentation for running the container from Airflow. Ingestion keeps an exact copy of what the provider returns (see The Pattern). Reshaping the data into a model of the sport happens downstream.

The container runs a few named workflows rather than one command per endpoint:

| Workflow | Cadence |
|---|---|
| Reference refresh | Weekly or less |
| Fixtures refresh (live fixtures and results) | Ideally every minute; never less often than every 15 minutes |
| Odds snapshot | At first, one closing-line snapshot per game; later, every 1–3 hours, calibrated against the quota |
| Backfill | On demand |

Each workflow knows what to fetch and in what order. For example, per-fixture endpoints need fixture IDs from `/fixtures` first. Each workflow also stays within rate limits and skips anything the cache already holds fresh. A run that fails partway through can simply be run again: the cache lets it resume instead of starting over.

Workflows share one daily quota per API. While the current and previous seasons are being filled in, the backfill takes priority and odds collection stays lean. Days with fewer games give the backfill more room.

Most of the retrieval logic lives in this project, where it is versioned, tested and documented. The Airflow DAG starts workflows on a schedule and then runs downstream cleansing. The analyst gets football and AFL data accumulating in SQL Server, including every odds snapshot. The data is queryable without going through the container.

## The Pattern

The project's lasting output is a pattern. It separates what every later ingestion project reuses from what depends on how a source is accessed. The pattern's unit is the **dataset**: for an API, one endpoint. A **dataset definition** has two parts:
- the provider's specification of the dataset (for API-Sports, the endpoint in its OpenAPI spec)
- a queryable data model that represents the provider's data the way the provider models it

**Core: used by every project**

- **One container interface.** Every source runs behind the same Airflow-facing interface. It covers how a run is parameterized, how secrets are supplied, what exit codes mean and how re-runs behave.
- **A faithful raw landing zone, queryable in SQL.** Raw data lands in SQL Server exactly as the provider returned it. A queryable model on top mirrors the provider's own model. It is built from views, or from stored procedures that create persisted tables where performance requires it. Errors in the data are kept. Later quality checks may flag them but never change them.
- **Composable workflows.** Workflows combine shared, per-dataset processing steps with different arguments. The fixtures refresh and the backfill run the same code over different competitions and date ranges.
- **A retention mode for each dataset.** A dataset either overwrites its stored version on refresh or appends every retrieval. For API-Sports, odds append and everything else overwrites.
- **Shared logging.** All logging lives in the core, whatever the retrieval method. This includes the fetch log, which records every retrieval with a categorized outcome. Together they show what ran, what failed and why, across all datasets.
- **Documentation as a deliverable.** Run instructions, ADRs and maintenance notes are written for AI agents and for someone returning after months away.

**Adapters: one per retrieval method**

- **HTTP adapter.** Used by API-Sports and the next API project. It provides:
  - a request cache
  - error detection that does not rely on HTTP status alone
  - pagination
  - quota tracking, with a daily budget shared across workflows
  - a common definition for cache tables, with typed, indexed columns for each endpoint's parameters (one table per endpoint is the preferred layout)
- **Other adapters.** A source that does not use HTTP, such as an open-source sports data library, skips the HTTP adapter. It still lands data in the same zone, behind the same interface, with the same logging.

## Success Criteria

Each criterion states how it is checked.

**Analyst**

- **Complete detail.** Every finished fixture in an in-scope competition and season has its detail stored, or a fetch-log entry explaining why not. *Check: coverage query.*
- **Closing line.** Every in-scope game with odds on offer has a closing-line snapshot. *Check: fetch log and odds tables.*
- **Freshness.** A finished game starts processing within 15 minutes of the API marking it final. This is an absolute maximum; the target is to check live fixtures and results every minute. The game's data is stored and queryable within a further 5 minutes. *Check: schedule interval and fetch-log timestamps.*
- **Queryable model.** Every in-scope endpoint has a queryable model that mirrors the provider's data model. *Check: endpoint list.*

**Integrator**

- **Exit codes.** Exit codes distinguish success, a retryable failure (quota, network) and a non-retryable failure (such as bad configuration). *Check: tests and documentation.*
- **Safe re-runs.** Re-running any workflow duplicates nothing and loses nothing. *Check: tests.*

**Next builder**

- **New endpoint.** Adding an API-Sports endpoint requires only its dataset definition. The shared code that fetches, caches, logs and lands data does not change. *Check: add an endpoint and confirm that only dataset-definition files changed.*
- **New project.** The next ingestion project reuses the core unchanged. It adds only an adapter, if its retrieval method is new, and its dataset definitions. A dataset definition is the provider's API specification plus a queryable data model that represents the provider's data the way the provider models it. *Check: when the next project starts.*

**Months away**

- **Cold-start test.** An AI agent with no prior context and only the repository correctly answers a fixed set of maintenance questions. Examples: how to backfill a competition, why last night's odds run failed, how to add an endpoint, and how to rotate the API key. The agent also writes a working Airflow task for the fixtures refresh. *Check: re-run at each release.*

## Scope

**In scope**

- **Sources and plans.** API-Football (v3) on the Ultra plan (75,000 requests a day) and API-AFL (v1) on the Pro plan (7,500 a day).
- **Container and workflows.** The container, its workflows, its Airflow-facing interface, and instructions for running it from Airflow.
- **Pattern components.** The core and the HTTP adapter described in The Pattern.
- **Coverage, in phases:**
  1. Live seasons and the season before, for every competition, as soon as possible. Competitions are completed in priority order, major competitions first. Odds are limited to the closing line.
  2. Every competition, for every season in progress on 1 January 2022 or starting after it. Major competitions come first, chosen when the backfill runs.
  3. Eventually, as many competitions and seasons as possible.

**Out of scope**

- **Airflow DAGs and scheduling.** Only the instructions for using the container are delivered.
- **Downstream modeling.** This covers cleansing, any model of the sport itself, and merging with other sources.
- **Correcting data.** Future quality checks only flag issues.
- **Database backups.** These must be stored off the server; the location is still to be decided.
- **In-play odds.** A nice-to-have, pending a feasibility study.
- **Other data sources.**

## Risks and Open Questions

**Risks**

- **Backfill duration.** Football moved to the Ultra plan, on an annual subscription, so the backfill finishes in weeks rather than months. On rough estimates, phase 1 takes about a week and phase 2 a few weeks. Most of that time goes on `/predictions`, which costs one call per fixture. AFL's Pro plan is ample for one competition. The estimates are in the addendum.
- **One server.** Odds history is the only data that cannot be fetched again. Until off-server backups exist, it lives on one machine.
- **Odds storage growth.** Odds are append-only, and each payload covers every bookmaker and bet type. With only the closing line, growth starts small. Once the cadence rises, disk space becomes the limit, not quota. If the provider really updates odds only every 3 hours, polling more often stores identical snapshots.
- **Scheduling every minute.** Checking every minute means 1,440 container runs a day through Airflow. Architecture should compare that with fewer, longer runs that poll every minute internally.

**Open questions** (details in the addendum)

- **Closing-line timing.** Are pre-match odds frozen at kickoff, and still retrievable after the game? If so, collect them within three hours of the final whistle. If they keep updating during play, take the snapshot within the hour before kickoff.
- **Batch detail.** Does `fixtures?ids=` return the same embedded detail as `fixtures?id=`? If it does, fixture detail costs up to 20 times fewer calls.
- **Odds cadence.** Which rate within the 1–3 hour range best balances measured odds changes against the quota?
- **Hand-off to cleansing.** How does downstream cleansing learn what a run changed? The proposal is a run ID stamped on the fetch log and on every row written.
- **Competition list.** Should the in-scope competition list live in the repository, or in a control table that can change without rebuilding the image?
- **Injury status.** Does `injuries` keep a fixture's pre-match "Questionable" status after the match?
