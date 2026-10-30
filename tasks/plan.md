# Implementation Plan: Financial World Generator (`worldgen`)

Source of truth: `SPEC.md`. Target output: `TARGET_EXAMPLE.md`. Tasks and their status live in `tasks/todo.md`.

## Overview

Build `worldgen` as `SPEC.md` describes it. A world is a tree of agents:

- **Narrators** tell one part of the story each. At their reviews, on dates each narrator sets for itself, they read reports from their subtree and write their children's next briefs, and they can grow or prune their part of the tree. A child's escalation brings its parent's review forward to the next day.
- **Actors** sit at the leaves and take one tool-use turn per simulated day. They move money only through act tools, which are validated on the spot.
- **Staff nodes** decide situations on top of coded institution mechanics.

Ops emit double-entry events. A clearinghouse settles them with channel delays, and per-institution renderers write AccountingBench-style JSON tables. Every run is a self-contained snapshot with a log you can follow live (`runs tail`). Reviewers edit prompts in the manifest and rerun.

Build order, riskiest first:

1. **Offline tracer bullet.** A fake LLM drives a tiny world (an investor, an ops manager, a landlord and Mercury) through the real agent loop: an equity wire, then rent.
2. **The same tracer on Claude.** The first big unknown is whether actors use their tools sensibly.
3. **The narrator tree inside the run:** reviews (reports up, briefs down, each narrator choosing its next date), escalation, and a growing tree. This is the second big unknown.
4. **Actor life and staff:** messages, invoices, memory and noise.
5. **Institutions,** each a vertical slice, in three parallel lanes.
6. **First milestone:** a live 15-day run of the TARGET_EXAMPLE world from a hand-written manifest.
7. **The rest:** authoring (running the tree to day 0), then prompt revisions, quirks and run history, then the 18-month run.

## Decisions to confirm (Task 0)

These are missing from SPEC.md or were chosen during its rewrite without confirmation. Each has a proposed default. Task 0 records the answers and writes them into SPEC.md (and TARGET_EXAMPLE.md) as current design.

| # | Decision | Proposed default | Blocks |
|---|---|---|---|
| D1 | **Milestone window** (15 days) | 2021-03-22..2021-04-05. The company starts fresh on 03-22 with the $1.5M equity wire. The window covers payroll on 03-26, a scheduled review of one department (03-29), the default review of the whole tree on 04-01 (narrators with no `next_review` get the first day of the next month), month end (Ramp cutoff, Mercury statement and interest, an in-transit Stripe payout) and rent on 04-01, so phenomena 1, 2 and 5 can appear. Phenomenon 3 (the Contoso prepay from 2021-02-01) and phenomenon 4 (the NYC trip, 03-11..03-14) fall outside the window and are tested in the 18-month run. | Task 38 |
| D2 | **Milestone manifest source** | Hand-write `examples/saas_startup` (tree, entities, accounts, access, relationships, day-0 briefs). The milestone then judges the run loop by itself. Authoring comes after it and is compared against the hand-written manifest. | Phases 6-7 order |
| D3 | **SPEC open questions** | **OQ1:** a brief lists obligations due today and overdue ones, and `my_obligations` also shows the next `obligation_horizon_days` (config, default 30). **OQ2:** every actor takes a daily turn. Config `idle_turns: always\|when_items` (default `always`) lets the milestone measure the cost of the other choice. | Tasks 13, 15 |
| D4 | **SPEC choices made during the rewrite** | Default model `claude-opus-5-5`; the Account and Access model; staff nodes outside the tree. | Task 2 |
| D5 | **Gap-fills** (SPEC is silent on these) | **(a)** Every entity gets an implicit `external` account (tracked, not rendered) unless it declares one, so catalog merchants, customers and employees can receive money. `world_boundary` is only for money from outside the world. **(b)** An op emits an initiation event into one clearing account per channel; the clearinghouse later emits the linked settlement event. **(c)** Events gain `args` (the op's validated arguments, such as merchant, memo or invoice id) and `links` (related event ids), so renderers never read the run log. Record-only ops, such as an invoice being issued, emit a zero-amount event so their rows still link to one. **(d)** Available funds are the posted balance minus pending outflows. An act tool reserves funds when it runs; act tools don't await, so check-and-reserve is atomic under asyncio. **(e)** `nodes[].brief` holds each node's day-0 brief. Authoring happens at the start of day 0, and day 0's turns follow it. **(f)** The live prompt history is `<world_dir>/prompt_history.json`. A revision id is a content hash, so ids exist before that file does. **(g)** Each institution is a package `worldgen/institutions/<name>/` holding `renderer.py`, one file per mechanic and its row models; the engine loads it by name. **(h)** The merchant catalog is `worldgen/catalog/merchants.yaml` (name, category, price range, descriptors), and vendor entities in the manifest extend it. A `card_swipe` at a merchant found in neither is rejected. **(i)** A deviation is an `answer_obligation` whose decision isn't `pay`, or that pays a different amount or day from the terms. It drives `runs tail --deviations-only` and the deviation counts. **(j)** `worldgen author` writes its log to `<world_dir>/authoring/<id>/log.jsonl`, in the same format as a run log. **(k)** Users edit prompts in the manifest, and `simulate` records each changed prompt as a `user` revision. `worldgen prompts history\|revert` lists and reverts revisions. **(l)** `validate` and `simulate` take `--overlay <file>`, which merges extra stories and noise into a manifest by id, so the plain `examples/saas_startup` and its one-offs live in separate files. **(m)** `default_chapter` is `week`, `month` or `quarter`: the first day of the next calendar week (a Monday), month or quarter after the review day, or after the start day for a narrator's first review. **(n)** An escalation made while a review that covers the parent is under way, before the parent has read its children's reports, is read in that review and moves no date. Otherwise the parent's next review moves to the next day if it was later. The `escalated` entry records which of the two applied. **(o)** A report turn uses the actor's usual system prompt and tools, so the prompt cache still holds, but its act tools and `send_message` are rejected with a reason: acting happens in the day's own turn. | Tasks 2, 5, 6, 16, 21, 22b, 31, 38, 40, 43 |
| D6 | **Overview and subagents** | The overview is the CLI. `run_started` lists every seed node (role, parent, `stands_for`), so `runs tail` opens with the starting tree, and `runs show` prints the tree with its spawns and retirements. A subagent is a node a narrator spawns with `spawn_child`: a new participant in the world. Actors don't spawn. Who can message whom follows SPEC's contact rule: the handlers on the other side of a relationship, and anyone who has messaged you. | Tasks 7, 20, 23, 26 |

## Architecture Decisions

- **Day order.** The manifest is the world at the start of day 0. Each day d runs four steps in order:
  1. **Reviews due today** (never on day 0). A narrator's next review date comes from its manifest `next_review`, then from its own `schedule_review` calls, and otherwise from `default_chapter` (D5m). An escalation can bring the date forward (D5n). A narrator due on the same day as one of its ancestors is reviewed as part of the ancestor's review, and reviews of separate subtrees run in parallel. A review makes three passes over the reviewing narrator's subtree:
     1. Every actor takes a report turn (D5o).
     2. Narrators report up level by level, starting from the deepest.
     3. The reviewing narrator writes its children's briefs and sets its next review, and then briefs go down level by level.

     Narrators on the same level run in parallel.
  2. **Briefs.** Every actor's and staff node's brief is built from state as of the last close plus today's items.
  3. **Turns.** All actor and staff turns run in parallel. Act tools validate and execute inside the turn.
  4. **Day close.** The clearinghouse settles what is due. Institution mechanics run (fees, payouts, statements, autopay, interest, payroll). Renderers write rows to the table files. Messages and timers go into tomorrow's items. `day_closed` is logged.

  Events from day d become visible to other nodes from d+1.
- **One `AgentLoop` for every role.** A role is a system template plus a tool set. Each turn is a fresh conversation. The system prompt and tools are byte-identical per role and sent first, so they are served from the prompt cache. Tools are classes with a strict pydantic input and a `run` method. Bad input returns an error result to the model instead of raising. Ops are the act tools.
- **Isolation through `NodeView`.** Briefs and read tools see the world only through a `NodeView` scoped to one node. It holds:
  - the node's entity's accounts, plus accounts it has access to
  - its inbox, obligations, notes and history
  - for narrators, their children's prompts and reports

  An isolation validator scans the requests in `log.jsonl` for other nodes' private text.
- **Conservation by construction.** Every event moves money between two accounts, and clearing accounts hold money in flight. `verify` re-checks conservation on every run, and hypothesis tests cover ops and mechanics.
- **Obligations carry their paying op.** Each terms type compiles into dated obligations, and the terms type and channel decide the op that pays them: `pay` for rent, `card_swipe` for a SaaS tool billed to a Ramp card, `stripe_charge` for a subscription, `payroll_run` for payroll. `answer_obligation` executes that op from the account the handler names, with the same validation as any other act tool.
- **Pluggable institutions.** The engine never imports a specific institution. Mechanics hook into day close, and renderers read settled events on their own institution's accounts. Row descriptions come from counterparty descriptors, so a payout shows up as `Deposit - STRIPE` without Mercury knowing anything about Stripe.
- **Updatable rows.** Rows are keyed by id, and rendering an id again replaces the row. This covers backfilling `payout_id` and `statement_id` and moving a payout from `in_transit` to `paid`. Every render logs `row_rendered`.
- **Row schemas.** Each output table has a pydantic row model at runtime. Tests also validate rows against TARGET_EXAMPLE's JSON Schemas, copied to `worldgen/institutions/schemas/`, using `jsonschema` as a dev-only dependency.
- **Test LLMs.** `FakeLlm` either runs a script or replays recorded whole-turn fixtures. `tests/support/as_agreed.py` is a scripted policy that pays every obligation as agreed and writes stock reports and briefs; it drives whole worlds offline. No `Llm` implementation is added to `worldgen/` beyond `AnthropicLlm` and the fake.
- **Logging.** Each log entry kind is its own file under `worldgen/runs/entries/`, added by the task that first emits it. Everything a reviewer inspects (`runs list`, `show`, `tail`, the noise trace) is computed from run files alone.
- **Out of scope for this plan:** `booking/` (derived books), tax jurisdictions, reactive terms, answer keys, and worlds without a company at the center.

## Task List (index; status is tracked in `tasks/todo.md`)

**Phase 0: Decisions**
- Task 0: Confirm D1-D6 and sync SPEC.md and TARGET_EXAMPLE.md

**Phase 1: Foundation and tracer bullet (fake LLM)**
- Task 1: Project scaffold and CLI entry point
- Task 2: Manifest models: world, config, nodes, entities, accounts, access
- Task 3: Manifest models: relationships, terms, stories, noise
- Task 4: `worldgen validate`: references, tree, time range
- Task 5: Clock, Rng and money
- Task 6: Accounts, events and ledger
- Task 7: Run directory, `run.json` and `log.jsonl`
- Task 8: Llm interface and `FakeLlm`
- Task 9: Tool base class and `AgentLoop`
- Task 10: Compiler: obligations and story steps
- Task 11: Clearinghouse and channels
- Task 12: Ops as act tools: validation, reservation, `pay`, `transfer_own`
- Task 13: Node tree, actor briefs and read tools
- Task 14: Answering obligations
- Task 15: Day loop
- Task 16: Institution plugins and Mercury transactions
- Task 17: `worldgen simulate`
- Task 18: `worldgen verify`
- **Checkpoint 1:** the tracer world simulates and verifies offline

**Phase 2: Live loop (highest risk)**
- Task 19: `AnthropicLlm`, recorder and live tracer smoke test
- Task 20: `worldgen runs list|show` with a loop-quality summary
- **Checkpoint 2:** a live tracer run reviewed with the user

**Phase 3: Narrator tree (second-highest risk)**
- Task 21: Reviews, part 1: review dates and reports up
- Task 22: Reviews, part 2: briefs down and `schedule_review`
- Task 22b: Escalation
- Task 23: Growing the tree
- Task 24: Relationships and stories written during a run
- Task 25: Tree-rule and isolation validators
- **Checkpoint 3:** the user reads a live tracer run's reviews: the dates narrators chose, the reports and briefs, and any escalations

**Phase 4: Actor life and staff**
- Task 26: Messages and inbox
- Task 27: The `invoice` op
- Task 28: Timers, notes, history search and statements
- Task 29: Noise in briefs
- Task 30: Staff nodes
- **Checkpoint 4**

**Phase 5: Institutions (three lanes)**
- Task 31: Merchant catalog, `card_swipe` and Ramp transactions
- Task 31b: Vendors that bill a card
- Task 32: Ramp statements and autopay
- **Checkpoint 5a:** Ramp
- Task 33: Stripe subscriptions, charges, fees and invoices
- Task 34: Stripe payouts into Mercury, with holds
- Task 35: Stripe refunds, disputes and expense pulls
- **Checkpoint 5b:** Stripe
- Task 36: Rippling payroll
- Task 37: Mercury statements, treasury, interest, fees and returns
- **Checkpoint 5c:** all four institutions

**Phase 6: First milestone (15 days)**
- Task 38: `examples/saas_startup` manifest and offline baseline
- Task 39: Live 15-day milestone run
- **Checkpoint 6:** milestone review with the user

**Phase 7: Authoring (the tree run to day 0)**
- Task 40: Day-0 tree run and manifest writer
- Task 41: Validate-and-repair loop
- Task 42: `worldgen author` and the 20-prompt fixture suite
- **Checkpoint 7**

**Phase 8: Revisions, quirks, run history**
- Task 43: Prompt history and revision links
- Task 44: `add_quirk`
- Task 45: `runs tail` with filters, and cancel
- Task 46: Rerun from a run
- Task 47: Noise traceability
- **Checkpoint 8**

**Phase 9: Completion**
- Task 48: Toy `plaid` institution, renderer isolation and docs
- Task 49: 18-month run and the full phenomena suite
- **Checkpoint: Complete**

## SPEC success criteria to task map

| SC | Tasks |
|---|---|
| 1 validate errors | 4, 12 |
| 2 phenomena | 31, 34, 39 (phenomena 1, 2, 5); 49 (all five) |
| 3-5 variation, conservation, event links | 6, 11, 18, 39, 49 |
| 6 noise traceable | 29, 47 |
| 7 author; edit and rerun | 40-42, 43 |
| 8 new institution with one file | 16, 48 |
| 9 rows as produced, retries, tool-call cap, cancel | 16, 9, 19, 45 |
| 10 act tools validated | 12 |
| 11 run history | 7, 17, 18, 45, 46 |
| 12 obligations answered | 14, 18 |
| 13 isolation | 13, 25, 26, 39 |
| 14 tree rules | 21, 22, 22b, 23, 24, 25 |

## Parallelization

- **Must be sequential:** the core, Tasks 5-18.
- **After Checkpoint 2:** the narrator lane (21-25, with 22b) and the actor-life lane (26-28) both add sections to `tree/briefs.py`. They can run in parallel if the brief is composed from ordered section functions and each lane adds its own.
- **After Checkpoint 4, three institution lanes:** Ramp (31, 31b, 32), Stripe (33-35), and Rippling plus Mercury (36-37). Each lane works in its own institution package and adds its own op files, so they don't conflict.
- **Authoring (40-42)** needs only the tree tools (21-24) and `validate`, so it can run alongside Phases 4-6.
- **Run history (43-47)** needs only Tasks 13, 17, 20 and 29, so it can run alongside Phases 5-7.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Actors misuse tools, invent implausible activity, or all behave alike | High | Strict tool schemas; rejection reasons returned within the turn; a live tracer before any scale-out; `runs show` metrics; the milestone review |
| Narrator briefs are too vague or too directive, reports bloat, or the tree sprawls | High | Live reviews at Checkpoint 3 before institutions; `runs show` counts spawns and report and brief lengths; review at the milestone |
| Narrators review every day or almost never, or actors escalate everything | High | `schedule_review` rejects dates that aren't after today or that fall past the world end; `runs show` reports reviews by trigger, chapter lengths and escalations; the narrator template and Checkpoint 3 and milestone reviews tune it |
| Wall-clock time: hundreds of actors × a daily turn × 540 days, plus a report turn for every actor in a reviewed subtree | High (18-month run) | Parallel turns; prompt-cached prefixes; per-day wall-clock and report turns per review measured at the milestone; the `idle_turns` switch (D3) |
| Money leaks through clearing, fees, payouts or refunds | High | Double-entry events; hypothesis conservation tests; `verify` on every run |
| Concurrent turns overdraw a shared account | Medium | Synchronous check-and-reserve (D5d); a test with concurrent turns on one account |
| A brief or tool result leaks another node's state | Medium | `NodeView` as the only path to state; a log-scanning isolation validator run on the milestone |
| Recorded fixtures stop replaying after prompt or tool changes | Medium | Fixtures keyed by turn and call index; tests assert invariants, not prose; re-record whenever a template or tool schema changes |
| A quirk written to the live manifest collides with a user editing it | Low | One atomic write per quirk; `simulate` turns any manifest change into a `user` revision, so nothing is lost silently |
| `log.jsonl` grows large (every request and response) | Low | SPEC accepts this; `runs tail` filters keep it readable |
| SPEC.md and TARGET_EXAMPLE.md drift (cohorts, "authoring agents") | Medium | Task 0 syncs both before any code depends on them |

## Open Questions

- D1-D6 above. D3-D5 are needed before Task 2, D6 before Task 7, and D1 and D2 by Phase 6.
