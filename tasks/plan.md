# Implementation Plan: Financial World Generator (`worldgen`)

Source of truth: `SPEC.md`. Target output: `TARGET_EXAMPLE.md`. Tasks and their status live in `tasks/todo.md`.

## Overview

Build `worldgen` as `SPEC.md` describes it. A manifest of entities, relationships, stories and noise prompts drives a day-by-day simulation. An LLM agent for each entity decides every event that entity produces. Coded institution mechanics (clearing delays, Stripe payouts, Ramp statements, Rippling payroll, Mercury statements) turn settled money into AccountingBench-style JSON tables. Every run is kept as a self-contained snapshot with a live log, and a dashboard lets reviewers edit prompts and rerun.

Build order, riskiest first:

1. A thin tracer bullet with a fake LLM. The tracer world is an investor, a company, a landlord and Mercury: an equity wire plus rent.
2. A live LLM on the same tracer. Whether the agent loop gives valid, plausible answers is the main unknown.
3. Each institution added as its own vertical slice.
4. The **first milestone**: a live 15-day run of the TARGET_EXAMPLE world.
5. Prompt revisions and quirks, then the dashboard, then authoring, then the 18-month run.

## Decisions to confirm (Task 0)

These are either missing from SPEC.md or contradicted by later direction from the user. Each one shows a proposed default. Task 0 records the answers and writes them into SPEC.md as current design.

| # | Decision | Proposed default | Blocks |
|---|---|---|---|
| D1 | **LLM backend.** On 2026-10-30 the user said to assume unlimited inference and drop `claude -p`. | `AnthropicLlm` over the Anthropic Python SDK, async, with the model set in world config and the API key read from env. Per-run call caps, `paused` status and `run_paused`/`run_resumed` are dropped (SPEC SC9 and OQ1 get rewritten). The backend retries transient and rate-limit errors with backoff. Adding the SDK dependency and an API-key `Llm` are both "ask first" items in SPEC. | Task 16 |
| D2 | **Milestone window** (15 days) | 2021-03-22..2021-04-05, with the company starting fresh on 2021-03-22 with the $1.5M equity wire. The window covers payroll on the 26th, a month end (Ramp cutoff, Mercury statement and interest, an in-transit Stripe payout) and rent on April 1, so phenomena 1, 2 and 5 can appear. Phenomenon 3 (Contoso prepay from 2021-02-01) and phenomenon 4 (NYC trip, March 11-14) fall outside the window and are tested in the 18-month run. | Task 27 |
| D3 | **Dashboard stack** | Starlette + uvicorn, server-sent events, and a static HTML/JS frontend with no build step. These are new runtime dependencies. | Task 33 |
| D4 | **Data-model gap-fills** (SPEC is silent on these) | (a) Accounts are declared in the terms of `banks_with` relationships (id, name, kind, opening balance, credit limit for cards). (b) Every event is a double-entry transfer between two accounts. Money in flight sits in one clearing account per channel; money from outside the world comes from `world_boundary`. The spec's Event therefore gains `source_account`, `dest_account` and `decision_id`. (c) Story steps compile into dated wake-ups for their actor and are answered like obligations, so the LLM still decides every event. (d) The manifest holds the current prompt text, and `worlds/<world>/prompt_history.json` holds every revision. A revision id is a content hash, so ids exist before the history file does. (e) Each institution is a package `worldgen/institutions/<name>/` containing `renderer.py`, one file per mechanic, and its row models. The engine loads it by name. (f) `world.config` holds tunables (model, drift rates, interest rates, payout delay, cohort split size), and each one is added by the task that first needs it. | Task 2 |
| D5 | **Hierarchical narrator** (user idea from 2026-10-29, still open). Does the narrator act only at authoring time, or throughout the run? Does anything flow back up the tree? | Not built in this plan. The brief is composed from ordered sections, so a "from your parent" section can be added later without changing the engine. It needs an answer before Phase 7 (authoring). | Phase 7 |

SPEC Open Question 3 gets defaults rather than a blocking decision. An agent is woken on each day it has items due and answers only those items. A cohort's bulk answer is split above 500 members, set in config.

## Architecture Decisions

- **Async engine.** Each simulated day consults every triggered agent concurrently with `asyncio.gather`, and the day closes when all have answered.
- **Fixed order within a day:**
  1. Settle the clearinghouse.
  2. Run institution mechanics.
  3. Consult triggered agents in parallel.
  4. Validate and execute their actions.
  5. Render settled rows.
  6. Log `day_closed`.

  Events created on day d become visible to other agents from day d+1.
- **Conservation by construction.** Every event debits one account and credits another. `worldgen verify` re-checks conservation on every run, and property tests cover it.
- **Structured agent answers.** Answers are JSON validated by pydantic (`AgentAnswer`). Each item carries a `cause` and an `as_agreed` flag; the dashboard's "deviations only" filter shows items where `as_agreed` is false. A parse error gets one re-ask. Missing obligations get one re-ask for just the missing items, then `obligation_unanswered` is logged.
- **Pluggable institutions.** Each institution exposes a mechanics hook and a renderer, resolved from the manifest by name (`institution: mercury` loads `worldgen.institutions.mercury`). The engine core never imports a specific institution. This is what makes SPEC SC8 (add an institution with one file) achievable.
- **Updatable rows.** Rows are keyed by id, and rendering the same id again replaces the row. This covers backfilling `payout_id` and `statement_id` and changing payout status from `in_transit` to `paid`. Every render logs `row_rendered`.
- **Row schemas.** Each output table has a runtime pydantic row model. Tests also check output against the TARGET_EXAMPLE JSON Schemas, copied to `worldgen/institutions/schemas/`, using `jsonschema` as a dev-only dependency. This avoids adding a runtime dependency.
- **Test LLMs.** Tests use `FakeLlm` (scripted, or replaying recorded fixtures) or `AsAgreedLlm`, which answers every item as agreed and gives an offline baseline run. Live runs record their calls so they can be replayed in tests. Tests assert invariants, not golden outputs.
- **Out of scope for this plan:** `booking/` (derived books), tax jurisdictions, reactive terms, answer keys, and the hierarchical narrator (D5).

## Task List (index; status is tracked in `tasks/todo.md`)

**Phase 0: Decisions**
- Task 0: Confirm D1-D5 and sync SPEC.md

**Phase 1: Foundation and tracer bullet (fake LLM)**
- Task 1: Project scaffold and CLI entry point
- Task 2: Manifest core models and loader
- Task 3: Stories, noise and `worldgen validate`
- Task 4: Clock, Rng and money
- Task 5: Accounts, events and ledger
- Task 6: Run directory, `run.json` and `log.jsonl`
- Task 7: Llm interface, fakes and the agent answer schema
- Task 8: Compiler: obligations from terms
- Task 9: Clearinghouse and channels
- Task 10: Agent brief and obligation answering
- Task 11: Op menu, action validation and transfer ops
- Task 12: Day loop with parallel consultation
- Task 13: Institution plugins and Mercury transactions
- Task 14: `worldgen simulate`
- Task 15: `worldgen verify`
- **Checkpoint 1:** the tracer world simulates and verifies offline

**Phase 2: Live LLM loop (highest risk)**
- Task 16: Real Llm backend, recorder and live tracer smoke test
- Task 17: `worldgen runs list|show` with a loop-quality summary
- **Checkpoint 2:** a live tracer run reviewed with the user

**Phase 3: Institutions and behaviors (vertical slices)**
- Task 18: Self-initiated triggers and agent memory
- Task 19: Ramp card swipes, merchant catalog and descriptors
- Task 20: Ramp statements and autopay
- **Checkpoint 3a:** Ramp
- Task 21: Stripe invoices, charges and fees
- Task 22: Stripe payout batching into Mercury
- Task 23: Stripe refunds, disputes and expense pulls
- **Checkpoint 3b:** Stripe
- Task 24: Rippling payroll
- Task 25: Mercury statements, treasury, interest and fees
- Task 26: Cohorts
- **Checkpoint 3c:** all four institutions

**Phase 4: First milestone (15 days)**
- Task 27: `examples/saas_startup` manifest and as-agreed baseline
- Task 28: Live 15-day milestone run
- **Checkpoint 4:** milestone review with the user

**Phase 5: Revisions, quirks, run history**
- Task 29: Prompt history and revision links
- Task 30: Agent quirks after a run
- Task 31: Runs tail, rerun from a run, cancel
- Task 32: Institution situations
- **Checkpoint 5**

**Phase 6: Dashboard**
- Task 33: Dashboard server and world view
- Task 34: Edit prompts and run from the dashboard
- Task 35: Live log panel and live tables
- **Checkpoint 6a:** edit, run and watch rows stream
- Task 36: Calendar and decision detail
- Task 37: Run history, replay and compare
- Task 38: Noise traceability
- **Checkpoint 6b**

**Phase 7: Authoring**
- Task 39: Authoring tools and the world and entity agents
- Task 40: Relationship, story and noise agents with a repair loop
- Task 41: `worldgen author` and the 20-prompt fixture suite
- **Checkpoint 7**

**Phase 8: Completion**
- Task 42: Isolation property, toy `plaid` institution and docs
- Task 43: 18-month run and the full phenomena suite
- **Checkpoint: Complete**

## SPEC success criteria to task map

| SC | Tasks |
|---|---|
| 1 validate errors | 3, 11 |
| 2 phenomena | 22, 19, 28 (1, 2, 5); 43 (all five) |
| 3-5 variation, conservation, event links | 5, 9, 15, 28, 43 |
| 6 noise traceable | 10, 12, 38 |
| 7 author + dashboard | 33-35, 39-41 |
| 8 new institution with one file | 13, 42 |
| 9 streaming (call caps per D1) | 35 |
| 10 actions validated | 11 |
| 11 run history | 6, 14, 15, 31, 37 |
| 12 obligations answered | 10, 12 |

## Parallelization

- **Must be sequential:** the engine core, Tasks 5, 8, 9, 10, 11, 12, 13, 14, 15.
- **After Checkpoint 2, three independent lanes:** Ramp (19-20), Stripe (21-23), and Rippling plus Mercury (24-25). Each lane works in its own institution package. Ops are separate files that the op menu discovers, so the lanes don't conflict.
- **Authoring (39-41)** needs only the manifest models, `validate` and the op menu (Tasks 2, 3, 11). It can run alongside Phases 3-6 once D5 is answered.
- **Dashboard (33-38)** needs the run store, the log and revisions (6, 14, 29, 31).

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| LLM answers are invalid, implausible or too uniform | High | Strict answer schema with one re-ask; the loop-quality summary from `runs show`; a live run in Phase 2 before the slices; a milestone review before the dashboard |
| Money leaks through clearing, fees, payouts or refunds | High | Double-entry events, hypothesis conservation tests, and `verify` on every run in CI |
| Wall-clock time for 18 months (hundreds of agents per day) | Medium | Sparse triggers, cohorts with bulk answers, and per-day parallelism; timings come from `runs show` |
| SPEC.md disagrees with recent direction (`claude -p`, call caps) | Medium | Task 0 syncs the spec before code depends on it |
| The hierarchical narrator reshapes how briefs are built | Medium | Section-based brief composer; D5 must be answered before authoring |
| Non-determinism makes tests flaky | Medium | Injected `Rng` and `Llm`; fakes and recorded fixtures; invariants instead of goldens |
| A brief leaks another entity's information | Medium | Isolation test on briefs (Task 10) and import-isolation test on renderers (Task 42) |
| The dashboard grows unbounded | Medium | No-build frontend; one view per task; read from run files only |

## Open Questions

- D1-D5 above. D1, D2 and D4 are needed early; D3 by Phase 6; D5 by Phase 7.
