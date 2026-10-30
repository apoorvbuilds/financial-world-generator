# Tasks: Financial World Generator

Plan: `tasks/plan.md` (decisions D1-D6, architecture, risks). Spec: `SPEC.md`.

**Rules for every task**

- **Gate.** All three must pass before a task is checked off:
  ```
  uv run pytest -q --cov=worldgen --cov-fail-under=90
  uv run ruff check . && uv run ruff format --check .
  uv run mypy --strict worldgen
  ```
- Every source file listed gets a mirrored test under `tests/`. Test files are not listed separately.
- Tests use `FakeLlm` (scripted or replaying fixtures) or the `tests/support/as_agreed.py` policy. Live tests live in `tests/live/` and run only with `WORLDGEN_LIVE=1`.
- A task that changes the manifest schema updates `docs/manifest-reference.md` once that file exists (Task 48).
- Commit after each task. Commit messages carry no Claude attribution lines.

---

## Phase 0: Decisions

## Task 0: Confirm D1-D6 and sync SPEC.md and TARGET_EXAMPLE.md

**Description:** Get the user's answers to D1-D6 in `tasks/plan.md` and write them into SPEC.md as current design:
- Project Structure: institution packages and the merchant catalog.
- Core Concepts: the Event fields `args` and `links`, and zero-amount record events.
- Manifest Shape: `nodes[].brief`.
- Commands: `worldgen prompts history|revert` (D5k) and `--overlay` (D5l).
- Run history: `run_started` lists the starting tree (D6), and what a spawned node and the contact rule are, as answered.
- Open Questions: resolved as in D3.

Also bring TARGET_EXAMPLE.md in line with SPEC: the root narrator replaces "the authoring agents", self-serve subscribers become actor nodes instead of a cohort, institutions get staff nodes, and each table is "written only by <institution>'s renderer". Write only the current design, with no mention of removed alternatives.

**Acceptance criteria:**
- [ ] D1-D6 each have a recorded answer in `tasks/plan.md`
- [ ] SPEC.md and TARGET_EXAMPLE.md agree with each other and with the answers
- [ ] Both files describe only the current design

**Verification:**
- [ ] Manual: the user reads both diffs and approves them

**Dependencies:** None
**Files likely touched:** `SPEC.md`, `TARGET_EXAMPLE.md`, `tasks/plan.md`
**Estimated scope:** S

---

## Phase 1: Foundation and tracer bullet

## Task 1: Project scaffold and CLI entry point

**Description:** Set up a uv project for Python 3.12.
- Runtime dependencies: `pydantic` v2, `PyYAML`, `typer`.
- Dev dependencies: `pytest`, `pytest-cov`, `pytest-asyncio`, `hypothesis`, `ruff`, `mypy`, `types-PyYAML`, `jsonschema`.
- A `worldgen` console script backed by an empty typer app.
- ruff and `mypy --strict` configured in `pyproject.toml`.
- In `.gitignore`, replace the stale `worlds/*/out/` with `worlds/*/runs/` and `worlds/*/authoring/`.

**Acceptance criteria:**
- [ ] `uv sync` succeeds from a clean checkout
- [ ] `uv run worldgen --help` exits 0
- [ ] The gate passes, with the 90% coverage threshold active from the start

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/test_cli.py`
- [ ] Gate passes
- [ ] Manual: `uv run worldgen --help`

**Dependencies:** Task 0
**Files likely touched:** `pyproject.toml`, `worldgen/__init__.py`, `worldgen/cli.py`, `.gitignore`
**Estimated scope:** XS

## Task 2: Manifest models: world, config, nodes, entities, accounts, access

**Description:** Define frozen pydantic v2 models:
- `World`: name, start, end, currency, `schema_version`, optional seed.
- `Config`: model, effort, chapter, `max_tool_calls_per_turn`. Later tasks add their own tunables.
- `Node`: id, role (narrator, actor or staff), parent, `stands_for`, prompt, brief.
- `Entity`: id, kind, name, descriptors, institution.
- `Account`: id, holder, `at`, kind, name, opening, limit.
- `Access`: node, account, `can`.
- `Manifest`: the root, loaded from YAML.

Money fields parse from 2dp strings to `Decimal` and reject floats. Unknown fields are rejected.

Write `examples/tracer/manifest.yaml`:
- **Narrators:** `n_root`, the founder's life, and under it `n_company`.
- **Actors:** `n_morgan`, the ops manager with `ach` and `wire` access on X0666, under `n_company`; `n_investor` and `n_landlord` under `n_root`.
- **Staff:** `n_mercury`.
- **Entities:** the company, Morgan, the investor, the landlord and Mercury.
- **Accounts:** Mercury X0666, the investor's bank and the landlord's bank.

**Acceptance criteria:**
- [ ] The tracer manifest loads and survives a YAML round trip
- [ ] A float amount, an unknown field or a wrong type fails with an error naming the YAML path
- [ ] mypy strict passes with no `Any` leaking out of the YAML layer

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/manifest`
- [ ] Gate passes

**Dependencies:** Task 1
**Files likely touched:** `worldgen/manifest/{world,config,node,entity,account,access,manifest}.py`, `examples/tracer/manifest.yaml`
**Estimated scope:** M (small model files)

## Task 3: Manifest models: relationships, terms, stories, noise

**Description:**
- `Relationship`: id, parties, kind, `written_by`, handlers, prompt, optional terms, channel.
- Terms are a discriminated union on `type`. This task adds:
  - `one_time`: one amount on one day.
  - `recurring_fixed`: one amount every month, quarter or year, on a given day (rent, a retainer, an annual plan).
  - `prepaid_annual`: one up-front payment for a service period with a start and an end (Contoso's contract).
- `Story`: id, `written_by`, steps. Each step has a day, an actor, an intent, and optionally a merchant and an amount.
- `Noise`: a scope (world, node, relationship, story or step) plus a prompt.

The tracer manifest gains `r_equity`, a $1.5M wire due on the start day, and `r_lease`, a monthly `recurring_fixed` term of $7,000 by ACH on the 1st, handled by `n_morgan`.

**Acceptance criteria:**
- [ ] The extended tracer manifest loads and round-trips
- [ ] An unknown terms `type` fails with a pointer
- [ ] A handler entry for an entity that isn't a party to the relationship fails with a pointer

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/manifest`
- [ ] Gate passes

**Dependencies:** Task 2
**Files likely touched:** `worldgen/manifest/{relationship,story,noise}.py`, `worldgen/manifest/terms/{one_time,recurring_fixed,prepaid_annual}.py`, `examples/tracer/manifest.yaml`
**Estimated scope:** M (small model files)

## Task 4: `worldgen validate`: references, tree, time range

**Description:** Three validators, one file each:
- **manifest_refs** checks references from:
  - node parents and `stands_for`
  - account holders and `at`
  - access nodes and accounts
  - relationship parties, handlers and `written_by`
  - story actors and `written_by`
  - noise scopes
- **tree** checks that:
  - there is exactly one root, and it is a narrator
  - there are no cycles
  - every parent is a narrator and every actor is a leaf
  - staff nodes have no parent and stand for an institution
- **time_range** checks that story steps and term dates fall inside the world range.

`worldgen validate <manifest>` prints one error per problem, each with a pointer (for example `relationships[3].handlers.co_self: unknown node 'n_jordn'`), and exits 1 if there are any.

**Acceptance criteria:**
- [ ] One fixture manifest per error class fails with the exact pointer in its message
- [ ] The tracer manifest validates with exit code 0
- [ ] `worldgen/validators/` has 100% line coverage

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/validators tests/test_cli.py`
- [ ] Gate passes
- [ ] Manual: `uv run worldgen validate examples/tracer/manifest.yaml`

**Dependencies:** Task 3
**Files likely touched:** `worldgen/validators/{manifest_refs,tree,time_range}.py`, `worldgen/cli.py`, `tests/fixtures/manifests/`
**Estimated scope:** M

## Task 5: Clock, Rng and money

**Description:**
- `Clock` maps dates to day indices from the world start. It answers business-day questions on the Federal Reserve holiday calendar, computed in code with no added dependency. On that calendar a Saturday holiday does not close the preceding Friday. It also answers chapter questions: is this the last day of a chapter, and does this day open one?
- `Rng` wraps `random.Random` with an optional best-effort seed. It is the only source of randomness for coded mechanics, and it also generates UUIDs.
- The `money` helpers quantize `Decimal` to cents, parse and format 2dp strings, and raise on float input.

**Acceptance criteria:**
- [ ] hypothesis: `add_business_days(d, n)` never lands on a weekend or holiday and is at least d+n calendar days
- [ ] Known dates hold: 2021-07-05 is closed, 2021-12-24 is open, 2022-06-20 is closed; 2021-03-31 is the last day of a monthly chapter
- [ ] `format(Decimal("-642.2")) == "-642.20"`; float input raises `TypeError`

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_clock.py tests/engine/test_rng.py tests/test_money.py`
- [ ] Gate passes

**Dependencies:** Task 1
**Files likely touched:** `worldgen/engine/clock.py`, `worldgen/engine/rng.py`, `worldgen/money.py`
**Estimated scope:** S

## Task 6: Accounts, events and ledger

**Description:**
- The account registry is built from the manifest's accounts, plus an implicit `external` account for each entity that lacks one (D5a), one clearing account per channel, and `world_boundary`. Card accounts are liabilities with a limit.
- `Event` is frozen, with fields id, day, op, `source_account`, `dest_account`, amount, cause, turn (or mechanic name), `args` and `links` (D5c). The amount is a non-negative `Decimal`, and it is zero only for record-only ops.
- `EventLog` appends to `events.jsonl` and flushes after every write.
- `Ledger` derives each account's balance from events.

**Acceptance criteria:**
- [ ] hypothesis: for any sequence of events, the balances of all accounts (including `world_boundary`) sum to 0
- [ ] `events.jsonl` round-trips exactly, with amounts as 2dp strings
- [ ] An event with a negative or float amount is rejected

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/events tests/engine/test_accounts.py`
- [ ] Gate passes

**Dependencies:** Tasks 2, 5
**Files likely touched:** `worldgen/engine/accounts.py`, `worldgen/events/{event,event_log,ledger}.py`
**Estimated scope:** S

## Task 7: Run directory, `run.json` and `log.jsonl`

**Description:**
- `run_id`: a sortable UTC timestamp plus a 3-character suffix.
- `RunMeta` (`run.json`): status (running, done, failed or cancelled), start and end times, pid, counts, manifest sha256 and `parent_run`.
- `RunLog`: appends `{seq, ts, day, kind, ...}` to `log.jsonl` and flushes after every entry.
- `RunStore`: creates the run directory with a copy of `manifest.yaml` and a `prompts.json` snapshot (every prompt keyed by owner, with its content-hash revision id), and lists and opens runs.

A run context sets `done` on success and `failed` on any exception. This task adds the `run_started` entry, which lists every node of the starting tree (id, role, parent, `stands_for`) so a followed log opens with the seed agents (D6), and the `run_finished` entry.

**Acceptance criteria:**
- [ ] An exception inside a run leaves a readable partial `log.jsonl` and `status: failed`
- [ ] Two runs of one manifest create two sibling directories, and editing the source manifest afterward leaves both byte-identical (hash test)
- [ ] `seq` is strictly increasing, and the file is valid JSONL after every append

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/runs tests/sinks`
- [ ] Gate passes

**Dependencies:** Task 2
**Files likely touched:** `worldgen/runs/{run_id,run_meta,run_store}.py`, `worldgen/sinks/run_log.py`, `worldgen/runs/entries/{run_started,run_finished}.py`
**Estimated scope:** M

## Task 8: Llm interface and `FakeLlm`

**Description:**
- The `Llm` protocol: async `create(LlmRequest) -> LlmResponse`. The types mirror the Messages API with tool use:
  - **Request:** system, tools (name, description, input schema), and messages made of text, `tool_use` and `tool_result` blocks.
  - **Response:** content blocks, stop reason, and usage including cache read and write tokens.
- Each request carries a `TurnMeta` (node id, role, day, turn id, call index), so fakes can answer without parsing prose.
- `FakeLlm` either runs a scripted callable or replays recorded whole-turn fixtures keyed by `TurnMeta`. It records every call and raises on a fixture miss.

**Acceptance criteria:**
- [ ] Replay returns the recorded response for a matching request and raises on a miss
- [ ] A scripted fake can return several `tool_use` blocks in one response
- [ ] No module outside the CLI and simulation wiring imports a concrete `Llm` (import test)

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/llm`
- [ ] Gate passes

**Dependencies:** Task 1
**Files likely touched:** `worldgen/llm/{base,fake}.py`
**Estimated scope:** S

## Task 9: Tool base class and `AgentLoop`

**Description:**
- **`Tool`:** name, description, a strict pydantic input model exported as JSON schema, and `run(input, view) -> result`, which returns what the model sees. Invalid input comes back to the model as an error result; it does not raise.
- **`AgentLoop.run_turn(node, role, brief)`:**
  1. Send the role's system prompt and tools, then the brief.
  2. Execute every `tool_use` block in the response and send all the results back together, in order, in one message.
  3. Repeat until a response has no tool call, or the turn reaches `max_tool_calls_per_turn`.

  An optional follow-up hook can continue the turn once with an extra message; Task 14 uses it. Each turn is a fresh conversation.
- **Logging:** `turn_started`, `turn_finished`, `llm_call` (request, response, prompt revision ids) and `tool_call` (tool, input, result).

**Acceptance criteria:**
- [ ] Parallel tool calls produce one result message; the turn stops at exactly `max_tool_calls_per_turn` (hypothesis over scripted fakes)
- [ ] The system prompt and tools are byte-identical across two turns of the same role, and they come before the brief
- [ ] `agents/loop.py` has 100% line coverage

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/agents tests/tools`
- [ ] Gate passes

**Dependencies:** Tasks 7, 8
**Files likely touched:** `worldgen/agents/loop.py`, `worldgen/tools/tool.py`, `worldgen/runs/entries/{turn_started,turn_finished,llm_call,tool_call}.py`
**Estimated scope:** M

## Task 10: Compiler: obligations and story steps

**Description:** `Compiler` turns a `Manifest` into a `Schedule` with two parts:
- **Obligations by day,** expanded from relationship terms. Each obligation has an id, its relationship, the handler node, payer, payee, amount, due day, channel and paying op. The handler comes from the relationship's `handlers`, or defaults to the node that stands for the party. The paying op comes from the terms type and the channel; for `ach`, `wire` and `book` it is `pay` (Task 31b adds `card`).
- **Story steps by day,** grouped by actor.

Each terms expander is its own file; this task adds `one_time`, `recurring_fixed` and `prepaid_annual`. Obligations are clipped to the world range. The compiler can also add a single relationship to an existing schedule (needed in Task 24).

**Acceptance criteria:**
- [ ] Monthly `recurring_fixed` rent due on the 1st over 2021-03-22..2021-04-05 yields exactly one obligation, on 2021-04-01, handled by `n_morgan`
- [ ] The equity `one_time` term yields one obligation on the start day, handled by `n_investor`, and a `prepaid_annual` term yields one obligation on its start date
- [ ] A story step lands on its actor's day

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/compiler`
- [ ] Gate passes

**Dependencies:** Tasks 3, 5
**Files likely touched:** `worldgen/compiler/{compiler,obligation,schedule}.py`, `worldgen/compiler/terms/{one_time,recurring_fixed,prepaid_annual}.py`
**Estimated scope:** M

## Task 11: Clearinghouse and channels

**Description:** Channel rules are data:

| Channel | Settles |
|---|---|
| card | 1-3 calendar days |
| ach | 1-3 business days |
| wire | same or next business day |
| book | same day |

The clearinghouse handles money in flight:
- `enqueue` emits the initiation event from the source account to the channel's clearing account, and picks a settle day with `Rng` within the channel's bounds.
- `settle(day)` emits a linked settlement event, from the clearing account to the destination, for everything due that day.
- `in_flight(day)` reports what is still pending.
- A held item (used in Tasks 30 and 34) settles only after it is released.

**Acceptance criteria:**
- [ ] hypothesis: nothing settles before initiation + min delay or after the max, and nothing settles on a weekend or holiday unless the channel allows it
- [ ] Clearing accounts net to zero once everything has settled
- [ ] `engine/clearinghouse.py` has 100% line coverage

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_clearinghouse.py tests/engine/test_channel.py`
- [ ] Gate passes

**Dependencies:** Tasks 5, 6
**Files likely touched:** `worldgen/engine/{channel,clearinghouse}.py`
**Estimated scope:** S

## Task 12: Ops as act tools: validation, reservation, `pay`, `transfer_own`

**Description:** An `Op` is an act tool. It declares its name, its args (which always include `cause`), the access it needs (a `can` value) and the transfers it makes. `OpMenu` discovers the ops in `engine/ops/`, so adding an op needs no registry edit.

Every call is checked on the spot:
- The op exists and its args parse.
- The actor holds the source account through its entity, or has access to it with the right `can`.
- Funds are available: the posted balance minus pending outflows, or the remaining credit on a card account.

A call that passes reserves its funds and is enqueued in the clearinghouse within one synchronous step (D5d). A call that fails returns its reason as the tool result, logs `action_rejected`, and emits nothing.

The first ops are `pay` (to an entity, over `ach`, `wire` or `book`) and `transfer_own`. `validate` now also rejects unknown `can` values and channels, which completes SC1.

**Acceptance criteria:**
- [ ] An overdraft attempt returns its reason within the same turn, logs `action_rejected`, and leaves no event
- [ ] Two concurrent turns spending from one account cannot overdraw it (`asyncio.gather` test)
- [ ] An unknown access verb fails `validate` with a pointer

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/ops tests/validators`
- [ ] Gate passes

**Dependencies:** Tasks 4, 9, 11
**Files likely touched:** `worldgen/engine/ops/{op,op_menu,pay,transfer_own}.py`, `worldgen/runs/entries/action_rejected.py`, `worldgen/validators/manifest_refs.py`
**Estimated scope:** M

## Task 13: Node tree, actor briefs and read tools

**Description:**
- `tree/nodes.py` is the live tree: parent, children, role, the entity each node stands for, and subtree queries.
- `NodeView` is the only path from a brief or a read tool to world state. It covers the node's entity's accounts, the accounts it has access to, its obligations and its own history.
- `briefs.py` builds an actor's brief from ordered sections: the prompt (with its revision id), the chapter brief, noise in scope (empty until Task 29), and today's items (obligations due today and overdue, story steps, and balance changes since the last turn). `turn_started` logs the brief along with its revision ids.
- The actor role (system template plus tools) gets two read tools:
  - `my_balances` shows posted and available balances.
  - `my_obligations` shows due, overdue, and upcoming items within `obligation_horizon_days` (D3).

**Acceptance criteria:**
- [ ] `n_morgan`'s brief on 2021-04-01 lists the rent obligation, and `n_landlord`'s brief that day doesn't
- [ ] `my_balances` shows the equity wire only after it has settled
- [ ] A read tool cannot reach an account the node has no access to (test through `NodeView`)

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tree tests/agents tests/tools/read`
- [ ] Gate passes

**Dependencies:** Tasks 9, 10
**Files likely touched:** `worldgen/tree/{nodes,briefs}.py`, `worldgen/agents/{node_view,roles/actor}.py`, `worldgen/agents/templates/actor.md`, `worldgen/tools/read/{my_balances,my_obligations}.py`
**Estimated scope:** M (small tool files)

## Task 14: Answering obligations

**Description:** The `answer_obligation` act tool takes a decision (`pay`, `pay_late` with a new day, `short_pay` with an amount, `skip`, `dispute`, `cancel` or `change`) and a `cause`.
- A paying answer runs the obligation's paying op from an account the handler names, validated like any act tool.
- `pay_late` moves the obligation to the new day, where it returns to the handler's brief.
- The tool call records whether the answer is a deviation (D5i).

`engine/obligations.py` tracks open obligations per handler. If a turn ends with a due obligation unanswered, the loop continues once with a message listing only the missing items. If they are still unanswered, the engine logs `obligation_unanswered` and renders nothing for them.

Also add `tests/support/as_agreed.py`, a scripted policy that pays every obligation as agreed. It is the offline driver for whole worlds.

**Acceptance criteria:**
- [ ] A missing item gets exactly one follow-up listing only that item, then an `obligation_unanswered` entry naming the obligation and the handler
- [ ] `pay_late` by 3 days brings the item back to the brief 3 days later, and it is paid then
- [ ] No event pays an obligation without an `answer_obligation` call (unit form of SC12)

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_obligations.py tests/engine/ops`
- [ ] Gate passes

**Dependencies:** Tasks 12, 13
**Files likely touched:** `worldgen/engine/obligations.py`, `worldgen/engine/ops/answer_obligation.py`, `worldgen/runs/entries/obligation_unanswered.py`, `tests/support/as_agreed.py`
**Estimated scope:** M

## Task 15: Day loop

**Description:** `engine/day.py` runs one simulated day in the order set in the plan: build briefs, run every actor's turn in parallel (a `TaskGroup`), then close the day. Day close settles the clearinghouse, calls the institution mechanics and render hooks (empty until Task 16), and moves tomorrow's items into place. `engine/engine.py` loops over the world range. Staff nodes start taking turns in Task 30. `idle_turns` (D3) is read from config.

This task adds two log entries: `event`, one per event, by id; and `day_closed`.

**Acceptance criteria:**
- [ ] Two actors' turns on the same day are in flight at the same time (barrier test with `FakeLlm`)
- [ ] An event created on day d is visible to another node no earlier than d+1
- [ ] Every event names its turn or a mechanic

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_day.py tests/engine/test_engine.py`
- [ ] Gate passes

**Dependencies:** Tasks 7, 14
**Files likely touched:** `worldgen/engine/{day,engine}.py`, `worldgen/runs/entries/{event,day_closed}.py`
**Estimated scope:** M

## Task 16: Institution plugins and Mercury transactions

**Description:** The `Institution` interface has two parts: mechanics that run at day close, and a renderer for settled events on the institution's own accounts. Institutions are loaded by name from the manifest (D5g), and the engine imports none of them directly.

Mercury renders `mercury_transactions`:
- `transaction_date` is the initiation date and `posted_date` is the settlement date.
- The amount is signed.
- `transaction_type` comes from the channel.
- `description` and `counterparty_name` come from the counterparty's descriptors or name.

`institutions/descriptors.py` (the class in SPEC's Code Style) renders a counterparty's statement name from its `descriptors`, filling `{ref}` from `Rng`. Every institution uses it.

The `JsonFiles` sink keeps each table keyed by row id (re-rendering an id replaces the row) and writes `<institution>/<table>.json` at each day close. Every render logs `row_rendered` with its linked event ids.

**Acceptance criteria:**
- [ ] The tracer run renders the equity wire and the 04-01 rent ACH, each posted on a business day no earlier than its initiation, with descriptions rendered from the counterparty's descriptors
- [ ] Every row has a `row_rendered` entry linking at least one event
- [ ] `worldgen/engine` imports nothing from `worldgen.institutions.<name>` (import test)

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions tests/sinks`
- [ ] Gate passes

**Dependencies:** Task 15
**Files likely touched:** `worldgen/institutions/{institution,loader,descriptors}.py`, `worldgen/institutions/mercury/{renderer,rows}.py`, `worldgen/sinks/json_files.py`, `worldgen/runs/entries/row_rendered.py`
**Estimated scope:** M

## Task 17: `worldgen simulate`

**Description:** `Simulation` connects the manifest, compiler, engine, institutions and sinks inside a `RunStore` run context. `worldgen simulate <manifest> --llm fake:<fixture_dir>` creates `worlds/<world>/runs/<run_id>/` (or a directory set by `--world-dir`) and prints the run directory. The Anthropic backend becomes the default in Task 19.

**Acceptance criteria:**
- [ ] Simulating the tracer with the `as_agreed` policy produces a complete run directory: `run.json`, `manifest.yaml`, `prompts.json`, `log.jsonl`, `events.jsonl` and `mercury/mercury_transactions.json`
- [ ] The run ends with `status: done` and a `run_finished` entry with counts
- [ ] Running it twice gives two separate run directories

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/runs/test_simulation.py`
- [ ] Gate passes
- [ ] Manual: run it and read `log.jsonl`

**Dependencies:** Task 16
**Files likely touched:** `worldgen/runs/simulation.py`, `worldgen/llm/factory.py`, `worldgen/cli.py`
**Estimated scope:** S

## Task 18: `worldgen verify`

**Description:** Validators, one file each:
- **conservation:** for every account, opening balance + sum of events = closing balance, and the rendered rows give the same closing balance; all events sum to zero, counting `world_boundary`.
- **event_links:** every row links at least one event, with 0 orphans.
- **log_consistency:** every `row_rendered` entry matches a row and vice versa, and every `event` entry matches `events.jsonl` and vice versa.
- **row_schemas:** every row validates against its row model. Tests also check output against the TARGET_EXAMPLE JSON Schemas with `jsonschema`.
- **obligations_answered:** every obligation due within the run has an `answer_obligation` call or an `obligation_unanswered` entry, and no event pays an obligation without such a call (SC12).

`worldgen verify <run_dir>` prints a report and exits 1 on any failure.

**Acceptance criteria:**
- [ ] The tracer run verifies clean
- [ ] Each validator catches its corrupted-run fixture: a deleted row, an extra row, a tampered amount, an orphan event, a paid obligation with no answer
- [ ] `worldgen/validators/` has 100% line coverage

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/validators`
- [ ] Gate passes
- [ ] Manual: `uv run worldgen verify <tracer run dir>`

**Dependencies:** Task 17
**Files likely touched:** `worldgen/validators/{conservation,event_links,log_consistency,row_schemas,obligations_answered}.py`, `worldgen/institutions/schemas/*.schema.json`, `worldgen/cli.py`
**Estimated scope:** M

## Checkpoint 1: Tracer bullet (after Tasks 1-18)
- [ ] Gate passes
- [ ] `simulate` then `verify` on the tracer world is clean offline
- [ ] The user has read one run directory and agrees with the log, event and brief shapes

---

## Phase 2: Live loop

## Task 19: `AnthropicLlm`, recorder and live tracer smoke test

**Description:** Load the `claude-api` skill before writing this. `AnthropicLlm` uses the async Anthropic client:
- The model and effort come from `config`.
- The system prompt and tools are marked for prompt caching.
- Transient errors (rate limits, overload, 5xx, connection errors) are retried with capped exponential backoff and no cap on attempts, so they never fail a run on their own. Non-transient errors, such as a bad request or failed auth, fail the run.
- Credentials come only from the environment.

`RecordingLlm` wraps any `Llm` and writes whole-turn fixtures for `FakeLlm`. The live smoke test runs the tracer over its full range. This task adds the `anthropic` dependency.

**Acceptance criteria:**
- [ ] `WORLDGEN_LIVE=1 uv run pytest -q tests/live` runs the tracer live, with a clean `verify` and 0 `obligation_unanswered`
- [ ] The recorded fixtures replay the run offline through `FakeLlm`, and the replayed run also verifies clean
- [ ] Simulated 529 and connection errors are retried until the turn completes, and the API key never appears in `log.jsonl`, `run.json` or fixtures

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/llm`, plus the live test above
- [ ] Gate passes

**Dependencies:** Tasks 0 (D4), 18
**Files likely touched:** `pyproject.toml`, `worldgen/llm/{anthropic_llm,recording,factory}.py`, `tests/live/test_tracer_live.py`
**Estimated scope:** M

## Task 20: `worldgen runs list|show` with a loop-quality summary

**Description:** `runs list <world_dir>` prints every run: time, status, prompt revisions (from `prompts.json` and `prompt_revisions.json`), row counts and deviation counts (D5i). `runs show <run_dir>` opens with the run's tree (D6): every node at the start, with its role, parent, `stands_for` and prompt revision, then the nodes spawned or retired during the run and the narrator that did it. It then summarises the run:
- turns and LLM calls
- tool calls per turn (mean and max), and turns that hit the cap
- rejected actions by reason
- follow-ups and unanswered obligations
- deviations, messages and spawns
- report and brief lengths
- rows per table
- tokens (input, output, cache read, cache write) and latency
- wall-clock time per simulated day

These numbers are how the loop gets judged.

**Acceptance criteria:**
- [ ] The tree and the summary are computed from run files alone and work on a copied run directory
- [ ] A run marked `running` whose pid is gone is listed as `failed`
- [ ] The summary numbers match hand-counted values on a fixture run

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/runs/test_run_summary.py`
- [ ] Gate passes

**Dependencies:** Task 17
**Files likely touched:** `worldgen/runs/{run_summary,run_store}.py`, `worldgen/cli.py`
**Estimated scope:** S

## Checkpoint 2: Live loop (after Tasks 19-20)
- [ ] Gate passes
- [ ] A live tracer run reviewed with the user, using `runs show` and `log.jsonl`
- [ ] Decide whether the actor template or tool schemas need changes before building on them

---

## Phase 3: Narrator tree

## Task 21: Chapters, part 1: reports up

**Description:**
- On the last day of a chapter, an actor's brief asks for a report. The actor gives it with `report_up`.
- On the first day of the next chapter, narrators take turns level by level, starting from the deepest level, with narrators on the same level running in parallel. Each reads `children_reports` and gives its own `report_up`.
- A narrator sees its own prompt and brief, and its children's prompts and reports.
- Add the narrator role (system template plus tools).

Log entries: `chapter_started` and `report_written`. A child that didn't report shows as "no report" to its parent.

**Acceptance criteria:**
- [ ] Across a tracer run spanning 2021-03-31/04-01, every actor is asked for a report on 03-31, and `n_company` reports before `n_root`
- [ ] Narrators on the same level are in flight at the same time (barrier test)
- [ ] `children_reports` returns only the narrator's own children's reports

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tree/test_chapters.py tests/tools`
- [ ] Gate passes

**Dependencies:** Task 15
**Files likely touched:** `worldgen/tree/chapters.py`, `worldgen/tools/tree/report_up.py`, `worldgen/tools/read/children_reports.py`, `worldgen/agents/{roles/narrator.py,templates/narrator.md}`, `worldgen/runs/entries/{chapter_started,report_written}.py`
**Estimated scope:** M (small tool files)

## Task 22: Chapters, part 2: briefs down

**Description:** After the reports go up, narrators take turns from the root down, level by level, and write each child's next chapter brief with `brief_child`. A child that gets no new brief keeps its old one. `my_chapters` returns a narrator's own past chapters (the reports it read, the reports it wrote, and the briefs it wrote). A new chapter brief replaces the old one in an actor's brief from the next turn. The `brief_written` entry is added.

**Acceptance criteria:**
- [ ] After the chapter step on 04-01, each actor's brief carries the brief its parent wrote that day
- [ ] The root writes its briefs before `n_company` writes any
- [ ] `brief_child` to a node that isn't the caller's child is rejected with a reason

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tree tests/tools`
- [ ] Gate passes

**Dependencies:** Task 21
**Files likely touched:** `worldgen/tree/chapters.py`, `worldgen/tools/tree/brief_child.py`, `worldgen/tools/read/my_chapters.py`, `worldgen/runs/entries/brief_written.py`
**Estimated scope:** S

## Task 23: Growing the tree

**Description:** Narrator tools, validated on the spot:
- `spawn_child`: role, prompt, brief, and optionally `stands_for`.
- `retire_child`: rejected while the node still handles active relationships.
- `create_entity`: includes descriptors.
- `open_account`: the holder must be an entity that a node in the caller's subtree stands for.
- `grant_access`: the grantee must be a node in the caller's subtree.

`tree/spawn.py` applies these changes to the live tree, the account registry and the `NodeView`s. A spawned node takes its first turn the next day; a retired node takes no more turns. Log entries: `node_spawned` and `node_retired`, each naming the narrator.

**Acceptance criteria:**
- [ ] A hire spawned on day d takes its first turn on d+1 with the brief it was given, and can use the card account it was granted
- [ ] Retiring a node that still handles an active relationship is rejected, and the reason lists those relationships
- [ ] Every node added during a run has a `node_spawned` entry naming the narrator

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tree tests/tools/tree`
- [ ] Gate passes

**Dependencies:** Task 22
**Files likely touched:** `worldgen/tree/spawn.py`, `worldgen/tools/tree/{spawn_child,retire_child,create_entity,open_account,grant_access}.py`, `worldgen/runs/entries/{node_spawned,node_retired}.py`
**Estimated scope:** M (small tool files)

## Task 24: Relationships and stories written during a run

**Description:**
- **`write_relationship`** creates, amends (terms, handlers, prompt) or ends a relationship. Every party that isn't an institution must have a node in the writer's subtree. A new or amended relationship is compiled into obligations from the next day.
- **`write_story`** writes steps for actors in the writer's subtree, dated from the next day and within the world range.

The tracer's `n_company` can now take on a vendor during a run.

**Acceptance criteria:**
- [ ] A relationship written on day d produces obligations from d+1, never on d
- [ ] A relationship or story that names a node outside the writer's subtree is rejected with a reason
- [ ] Ending a relationship drops its future obligations, and the retire rule from Task 23 then allows retiring the node

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tools/tree tests/compiler`
- [ ] Gate passes

**Dependencies:** Task 23
**Files likely touched:** `worldgen/tools/tree/{write_relationship,write_story}.py`, `worldgen/compiler/compiler.py`
**Estimated scope:** S

## Task 25: Tree-rule and isolation validators

**Description:** Two validators that `worldgen verify` runs on every run:
- **tree_rules** (SC14): every node except the root has exactly one parent; every node added during a run has a `node_spawned` entry; and every relationship and story, whether from day 0 or written during the run, was written by a narrator whose subtree holds the nodes it names.
- **isolation** (SC13): for each node's `llm_call` requests, no other node's private text appears. Private text means prompt revisions, notes and chapter briefs. The exceptions are messages sent to the node and, for a narrator, its children's prompts and reports.

The isolation test plants a unique canary in every node's prompt, notes and brief, then runs the tracer through the `as_agreed` policy.

**Acceptance criteria:**
- [ ] The canary tracer run passes isolation, and a planted leak (one node's brief containing another node's prompt) fails it
- [ ] Each tree rule catches its corrupted-run fixture
- [ ] Both validators have 100% line coverage

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/validators`
- [ ] Gate passes

**Dependencies:** Tasks 18, 24
**Files likely touched:** `worldgen/validators/{tree_rules,isolation}.py`, `tests/support/canaries.py`
**Estimated scope:** M

## Checkpoint 3: Narrator tree (after Tasks 21-25)
- [ ] Gate passes
- [ ] A live tracer run across 03-31/04-01 is reviewed with the user: the reports, the briefs, and any spawns or relationships written
- [ ] Decide on changes to the narrator template before actor life and institutions build on it

---

## Phase 4: Actor life and staff

## Task 26: Messages and inbox

**Description:** `send_message` (to a node, with text and an optional reference to a relationship or obligation) follows the contact rule: an actor can message the handlers on the other side of the relationships it handles, and anyone who has messaged it. Messages are delivered to the recipient's inbox the next day and appear among its items. `my_inbox` reads the inbox. A `message_sent` entry is logged. The isolation validator learns that delivered messages are an allowed channel.

**Acceptance criteria:**
- [ ] A message sent on day d appears in the recipient's brief on d+1 and not on d
- [ ] A message to a node outside the contact rule is rejected with a reason, and a reply to a node that has messaged the sender is allowed
- [ ] Isolation still passes on a run where nodes exchange messages that quote their own prompts

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_inbox.py tests/tools tests/validators`
- [ ] Gate passes

**Dependencies:** Task 25
**Files likely touched:** `worldgen/engine/inbox.py`, `worldgen/tools/talk/send_message.py`, `worldgen/tools/read/my_inbox.py`, `worldgen/runs/entries/message_sent.py`, `worldgen/validators/isolation.py`
**Estimated scope:** M

## Task 27: The `invoice` op

**Description:** The biller's handler sends an invoice (relationship, amount, due day, memo). The invoice goes to the payer's handler as a message, opens an obligation for that handler on the due day, and emits a zero-amount record event (D5c). This supports Net-30 vendors such as corporate counsel.

**Acceptance criteria:**
- [ ] An invoice sent on day d reaches the payer as a message on d+1 and as an obligation on its due day
- [ ] Paying it with `answer_obligation` closes it; leaving it unpaid follows the Task 14 rules
- [ ] An invoice on a relationship the sender doesn't handle is rejected

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/ops tests/engine/test_obligations.py`
- [ ] Gate passes

**Dependencies:** Tasks 14, 26
**Files likely touched:** `worldgen/engine/ops/invoice.py`, `worldgen/engine/obligations.py`, `worldgen/compiler/obligation.py`
**Estimated scope:** S

## Task 28: Timers, notes, history search and statements

**Description:**
- `set_timer` (day, note) adds an item to the node's brief on that day.
- `note_to_self` appends to the node's notes.
- `search_my_history` searches the node's own past turns, tool calls, events, messages and notes, by text or date range.
- `my_statement` lists the events on an account the node can use, over a date range.

What a node remembers between turns is reached only through these tools.

**Acceptance criteria:**
- [ ] A timer set on day d for d+k fires on exactly d+k
- [ ] A note written on day d is found by `search_my_history` on a later day
- [ ] Neither `search_my_history` nor `my_statement` ever returns another node's entries or an account without access

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tools tests/engine/test_timers.py`
- [ ] Gate passes

**Dependencies:** Task 15
**Files likely touched:** `worldgen/engine/timers.py`, `worldgen/tools/self/{set_timer,note_to_self}.py`, `worldgen/tools/read/{search_my_history,my_statement}.py`
**Estimated scope:** M (small tool files)

## Task 29: Noise in briefs

**Description:** Noise prompts are added to the brief of every agent in their scope:

| Scope | Whose briefs |
|---|---|
| world | everyone |
| node | that node's subtree |
| relationship | the relationship's handlers |
| story | briefs that carry a step of the story |
| step | the brief that carries that step |

Every `turn_started` entry lists the revision ids of the noise in its brief (SC6, first part).

**Acceptance criteria:**
- [ ] Noise scoped to `n_company` appears in the brief of every node under it and of no other node
- [ ] Step-scoped noise appears only on that step's day, in that actor's brief
- [ ] Every `turn_started` entry lists its noise revision ids

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tree/test_briefs.py`
- [ ] Gate passes

**Dependencies:** Task 13
**Files likely touched:** `worldgen/tree/{briefs,noise_scope}.py`
**Estimated scope:** S

## Task 30: Staff nodes

**Description:**
- **Turns:** staff nodes take a turn every day. Their brief lists items that their institution's mechanics flag, such as a wire over a config threshold.
- **Read tool:** `institution_accounts` shows balances and recent activity on the accounts their institution holds.
- **Situation tools:** `return_wire` sends an incoming wire back, and Mercury charges a fee. `flag_review` holds an outgoing wire for N business days through the clearinghouse hold.
- Staff can also use `send_message`.

Situations are validated like any other act tool.

**Acceptance criteria:**
- [ ] A flagged wire settles N business days late, and conservation holds
- [ ] A returned wire sends the funds back, with a `Fee` row in `mercury_transactions`
- [ ] A staff tool on an account at another institution is rejected

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tools/staff tests/agents`
- [ ] Gate passes

**Dependencies:** Tasks 16, 26
**Files likely touched:** `worldgen/agents/{roles/staff.py,templates/staff.md}`, `worldgen/tools/staff/{institution_accounts,return_wire,flag_review}.py`
**Estimated scope:** M

## Checkpoint 4
- [ ] Gate passes; the tracer, extended with a Net-30 vendor and a large wire, runs offline and verifies clean, including tree rules and isolation

---

## Phase 5: Institutions

## Task 31: Merchant catalog, `card_swipe` and Ramp transactions

**Description:**
- **Catalog:** `catalog/merchants.yaml` (D5h) lists realistic merchants with a category, a price range and descriptor variants. `merchant_catalog` searches it by name or category.
- **`card_swipe` op:** takes a card account the actor has `card` access to, a merchant (catalog or vendor entity), an amount and an optional memo, and runs over the card channel. A swipe over the card's limit is rejected, as is a swipe at an unknown merchant (the rejection points to `merchant_catalog`).
- **Ramp renderer:** writes `ramp_card_transactions`:
  - `swipe_date` is the initiation date and `transaction_date` is the settlement date.
  - The cardholder's name and role come from the node with card access.
  - `merchant_name` comes from the descriptors.
  - `ramp_category` is a noisy hint drawn from the catalog category.
  - `memo` is whatever the actor wrote, or null.

**Acceptance criteria:**
- [ ] A swipe on 2021-03-30 that settles on 2021-04-01 renders `swipe_date` 2021-03-30 and `transaction_date` 2021-04-01 (phenomenon 5 at row level, with `Rng` stubbed)
- [ ] An unknown merchant or an over-limit swipe is rejected within the turn
- [ ] Descriptor variants with `{ref}` produce different statement names across rows

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/ramp tests/engine/ops/test_card_swipe.py tests/catalog`
- [ ] Gate passes

**Dependencies:** Tasks 12, 16
**Files likely touched:** `worldgen/catalog/{merchant_catalog.py,merchants.yaml}`, `worldgen/tools/read/merchant_catalog.py`, `worldgen/engine/ops/card_swipe.py`, `worldgen/institutions/ramp/{renderer,rows}.py`
**Estimated scope:** M

## Task 31b: Vendors that bill a card

**Description:** Card-on-file billing for SaaS tools, insurance and other plans billed to a Ramp card (TARGET_EXAMPLE section 4, "Company -> vendor (card)"). An obligation on a `channel: card` relationship goes to the company's handler, and a paying answer runs `card_swipe` at the vendor on a card the handler names and has `card` access to. The swipe is validated and rendered like any other.

**Acceptance criteria:**
- [ ] A monthly `recurring_fixed` Figma relationship on `channel: card` gives its handler one obligation a month, and paying it renders a `ramp_card_transactions` row with Figma's descriptor on the named card
- [ ] Paying from a card the handler can't use, or beyond the card's limit, is rejected within the turn
- [ ] A yearly plan yields one obligation a year, and skipping it produces no swipe and counts as a deviation

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/compiler tests/engine/ops tests/institutions/ramp`
- [ ] Gate passes

**Dependencies:** Task 31
**Files likely touched:** `worldgen/compiler/obligation.py`, `worldgen/engine/ops/answer_obligation.py`
**Estimated scope:** S

## Task 32: Ramp statements and autopay

**Description:** At month end, Ramp closes a statement over transactions by settlement date. It renders `ramp_statements` and backfills `statement_id` on those transactions. Autopay starts an ACH pull of the ending balance from Mercury X0666 a configured number of days after close. The pull appears as a Mercury `ACH Debit` and in `payments_and_credits` on the next Ramp statement.

**Acceptance criteria:**
- [ ] For every statement, `ending_balance = opening_balance + new_charges - payments_and_credits`
- [ ] A swipe that settles after the cutoff lands on the next statement
- [ ] The autopay row in Mercury links to that statement's events

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/ramp`
- [ ] Gate passes

**Dependencies:** Task 31
**Files likely touched:** `worldgen/institutions/ramp/{statements,autopay,renderer,rows}.py`
**Estimated scope:** S

## Checkpoint 5a: Ramp
- [ ] Gate passes; the tracer plus a cardholder and a vendor billing a card runs offline, and `verify` is clean

## Task 33: Stripe subscriptions, charges, fees and invoices

**Description:**
- **Terms:** `monthly_card` terms (plan price, renewal day) go on `customer_of` relationships. On each renewal, Stripe issues an invoice as a record event, and the customer's handler answers the obligation: pay, skip (the card fails), cancel or change plan.
- **Op:** paying runs `stripe_charge`, from the customer's card account into the company's Stripe balance account.
- **Fees:** a Stripe mechanic takes 2.9% + $0.30, in cents, rounded half up.
- **Renderer:** writes `stripe_invoices` (status `paid`, `open` or `void`) and `stripe_charges` (amount, fee, net, invoice id).

Cancelling ends the relationship and stops later renewals.

**Acceptance criteria:**
- [ ] hypothesis over amounts: `net = amount - fee` holds exactly, and money is conserved
- [ ] A skipped renewal leaves an `open` invoice and no charge row
- [ ] Every charge links its invoice, and a cancelled subscription has no later invoices

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/stripe tests/compiler/terms`
- [ ] Gate passes

**Dependencies:** Tasks 14, 16
**Files likely touched:** `worldgen/compiler/terms/monthly_card.py`, `worldgen/engine/ops/stripe_charge.py`, `worldgen/institutions/stripe/{fees,renderer,rows}.py`
**Estimated scope:** M

## Task 34: Stripe payouts into Mercury, with holds

**Description:** The payouts mechanic runs each business day:
1. Sweep the available balance (charges older than the availability delay in config) into one payout.
2. Send it by ACH to Mercury X0666, arriving 1-2 business days later.
3. Render `stripe_payouts`, with status `in_transit` until arrival and `paid` after.
4. Backfill `payout_id` on the swept charges.

Mercury renders the arrival from Stripe's descriptors as `Deposit - STRIPE`. Stripe staff get `hold_payout` and `release_payout`.

**Acceptance criteria:**
- [ ] Phenomenon 2: N charges produce one payout row and one Mercury row, and the sums agree
- [ ] Phenomenon 1: charges on March 29-31 give `payout_date` 2021-03-30 and `arrival_date` 2021-04-01, absent from Mercury's March rows
- [ ] A held payout arrives N business days late, a payout still in flight at run end stays `in_transit`, and money is conserved

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/stripe tests/tools/staff`
- [ ] Gate passes

**Dependencies:** Tasks 30, 33
**Files likely touched:** `worldgen/institutions/stripe/{payouts,renderer,rows}.py`, `worldgen/tools/staff/{hold_payout,release_payout}.py`
**Estimated scope:** M

## Task 35: Stripe refunds, disputes and expense pulls

**Description:**
- **`refund` op:** the company's handler refunds a charge, full or partial, with a reason. This usually follows a customer's message.
- **`dispute_charge` op:** a customer disputes a past charge. Stripe reverses the amount and adds a dispute fee.

Refunds and reversals are netted against the next payout. When they exceed the available balance, Stripe pulls the difference from Mercury by ACH, rendered as `Stripe - Expense - STRIPE`.

**Acceptance criteria:**
- [ ] `refunds_deducted` equals the sum of the refunds swept into that payout, and every refund row carries its `charge_id`
- [ ] A negative net produces a negative `net_payout_amount` and a Mercury `ACH Debit` row
- [ ] Conservation holds across a refund-heavy fixture

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/stripe tests/engine/ops`
- [ ] Gate passes

**Dependencies:** Task 34
**Files likely touched:** `worldgen/engine/ops/{refund,dispute_charge}.py`, `worldgen/institutions/stripe/{payouts,renderer,rows}.py`
**Estimated scope:** M

## Checkpoint 5b: Stripe
- [ ] Gate passes; a fixture world with card customers and refunds verifies clean, and the phenomenon 1 and 2 tests pass

## Task 36: Rippling payroll

**Description:** `semi_monthly` terms (the 10th and 26th, gross pay per employee) go on `employed_by` relationships. The company's handler, who needs payroll-admin access, answers each payroll obligation: run as agreed, change after a hire, or skip.

Paying runs the `payroll_run` op, and the Rippling mechanics then:
- compute withholding and employer taxes from a coded flat-rate table
- render `rippling_journal_entries`, numbered in sequence, with balanced debit and credit lines
- start ACH debits from Mercury X0666 for net pay (to the employees' external accounts) and for taxes (to `world_boundary`)

**Acceptance criteria:**
- [ ] Journal numbers rise by 1 per payroll run from the number set in config, and each entry balances
- [ ] The Mercury debits equal the net-pay and tax credit lines and link to the payroll events
- [ ] A skipped payroll produces no journal entry and no debit, and its `answer_obligation` call carries a cause

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/rippling tests/compiler/terms`
- [ ] Gate passes

**Dependencies:** Tasks 14, 16
**Files likely touched:** `worldgen/compiler/terms/semi_monthly.py`, `worldgen/engine/ops/payroll_run.py`, `worldgen/institutions/rippling/{payroll,renderer,rows}.py`
**Estimated scope:** M

## Task 37: Mercury statements, treasury, interest, fees and returns

**Description:** Mercury's month-end mechanics:
- `mercury_statement_line_items` per account, with a running balance.
- `mercury_treasury_records`: beginning balance, deposits, withdrawals, interest and ending balance.
- Interest credited at month end on checking and Treasury, at annual rates set in config.
- Service fees.
- Returns: a debit started by an institution mechanic (autopay, a Stripe pull, payroll) against insufficient funds is returned, followed by a fee row.

`transfer_own` between Mercury accounts renders as `Internal Transfer`.

**Acceptance criteria:**
- [ ] `running_balance` carries over continuously between months and equals the ledger balance at each line
- [ ] The March statement excludes items posted on April 1
- [ ] Treasury `ending = beginning + deposits - withdrawals + interest`, and a returned debit is followed by its fee row

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/mercury`
- [ ] Gate passes

**Dependencies:** Task 16
**Files likely touched:** `worldgen/institutions/mercury/{statements,treasury,interest,returns,rows}.py`
**Estimated scope:** M

## Checkpoint 5c: All institutions
- [ ] Gate passes; all four institutions render, and fixture worlds verify clean
- [ ] Review the state of the system with the user before the milestone

---

## Phase 6: First milestone

## Task 38: `examples/saas_startup` manifest and offline baseline

**Description:** Hand-write the TARGET_EXAMPLE world over the D1 window (D2):
- **Tree:** `n_root` holds Alex's life. Under it are `n_alex` and `n_company`, and under `n_company` are the departments (engineering, sales, ops and self-serve), each with its actors. There is one staff node per institution.
- **Cast:** the five people with Ramp cards and Mercury access as in TARGET_EXAMPLE 3.3; Contoso and the other enterprise customers; a few self-serve subscribers as actors (the self-serve narrator spawns more); and vendors with descriptor variants.
- **Accounts:** Mercury (three accounts), Stripe, Ramp and Rippling.
- **Relationships:** with the terms from TARGET_EXAMPLE section 4.
- **Day-0 briefs** for every node.

The first version is plain: no stories and no one-off noise. The NYC trip story and the Contoso noise go in `examples/saas_startup/extras/` as overlays. `validate` and `simulate` accept `--overlay <file>`, which merges lists by id.

**Acceptance criteria:**
- [ ] `validate` exits 0 with and without each overlay
- [ ] A run with the `as_agreed` policy, followed by `verify` (including tree rules and isolation), is clean
- [ ] A test confirms every entity and relationship from TARGET_EXAMPLE sections 3 and 4 is present, by id

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/examples`
- [ ] Gate passes

**Dependencies:** Tasks 0 (D1, D2), 25, 27, 28, 29, 31b, 32, 35, 36, 37
**Files likely touched:** `examples/saas_startup/manifest.yaml`, `examples/saas_startup/extras/{nyc_trip,contoso_noise}.yaml`, `worldgen/manifest/overlay.py`, `worldgen/cli.py`
**Estimated scope:** M

## Task 39: Live 15-day milestone run

**Description:** Run `saas_startup` live over the D1 window through `RecordingLlm`, once with `idle_turns: always` and once with `when_items` (D3). Commit the recorded fixtures; they are synthetic, with no real PII. Add replay tests that query the output tables for phenomena 1, 2 and 5. Write a review note in `tasks/milestone-15d.md` covering:
- both `runs show` summaries, with wall-clock time per day
- sample deviations
- the chapter step on 04-01
- anything that looked unrealistic

**Acceptance criteria:**
- [ ] The live runs verify clean (conservation, event links, obligations, tree rules, isolation), with 0 unanswered items or each one explained in the note
- [ ] Replay tests for phenomena 1, 2 and 5 pass offline
- [ ] The review note is written

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/examples/test_phenomena_15d.py`
- [ ] Gate passes
- [ ] Manual: the user reads the note and browses the tables

**Dependencies:** Tasks 19, 20, 38
**Files likely touched:** `tests/fixtures/llm/saas_15d/`, `tests/examples/test_phenomena_15d.py`, `tasks/milestone-15d.md`
**Estimated scope:** S

## Checkpoint 6: Milestone (after Tasks 38-39)
- [ ] Gate passes
- [ ] The user reviews the live 15-day run and decides on prompt, tool or template changes
- [ ] Settle the `idle_turns` default from the measurements, and write it into SPEC.md

---

## Phase 7: Authoring

## Task 40: Day-0 tree run and manifest writer

**Description:** `tree/day_zero.py` runs the tree to day 0:
1. The root narrator gets the main prompt and the day-0 section of the narrator template, which asks for a plain first generation: only what the prompt asks for.
2. It uses the tree tools from Tasks 23-24 to spawn children; create entities, accounts, access grants, relationships and stories; and brief each child.
3. Each new narrator then takes its own day-0 turn, top-down and level by level, with narrators on the same level in parallel. Actors take no day-0 turns.

`manifest/writer.py` serializes the resulting tree state, including `nodes[].brief`, to `manifest.yaml`. The authoring log goes to `<world_dir>/authoring/<id>/log.jsonl` (D5j).

**Acceptance criteria:**
- [ ] With `FakeLlm` fixtures, a TARGET_EXAMPLE-like prompt yields a manifest that loads, and the written manifest round-trips
- [ ] A prompt with no one-offs produces no stories and no noise
- [ ] Levels run in order, and narrators on the same level are in flight at the same time

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tree/test_day_zero.py tests/manifest`
- [ ] Gate passes

**Dependencies:** Tasks 0 (D5j), 24
**Files likely touched:** `worldgen/tree/day_zero.py`, `worldgen/manifest/writer.py`, `worldgen/agents/templates/narrator.md`
**Estimated scope:** M

## Task 41: Validate-and-repair loop

**Description:** After the day-0 tree run, `validate` runs on the draft. Each error is sent to the narrator that wrote the offending item, found through `written_by` or through the authoring log's tool calls, as a follow-up turn listing only its errors. This repeats for up to `authoring_repair_rounds` (config). A draft still invalid after that ends with a report and a nonzero exit.

**Acceptance criteria:**
- [ ] A fixture whose first draft has a dangling reference is repaired within the configured number of rounds
- [ ] Each error goes to the narrator that wrote the item, and no error goes to any other node
- [ ] A draft still invalid after the last round ends with a clear report

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tree/test_repair.py`
- [ ] Gate passes

**Dependencies:** Task 40
**Files likely touched:** `worldgen/tree/repair.py`, `worldgen/tree/day_zero.py`
**Estimated scope:** S

## Task 42: `worldgen author` and the 20-prompt fixture suite

**Description:** `worldgen author --prompt ... --out <manifest>` runs Tasks 40-41 and writes the manifest. 20 fixture prompts with recorded responses are replayed in tests. As a live check, TARGET_EXAMPLE.md is used as the main prompt, and the result is compared by hand against `examples/saas_startup`.

**Acceptance criteria:**
- [ ] At least 18 of the 20 fixture prompts produce manifests that pass `validate` (SC7, first part)
- [ ] An authored manifest runs with the `as_agreed` policy and verifies clean
- [ ] A live authoring run from TARGET_EXAMPLE.md is reviewed with the user

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tree tests/test_cli.py`
- [ ] Gate passes
- [ ] Manual: author, then simulate, then `runs show`

**Dependencies:** Task 41
**Files likely touched:** `worldgen/cli.py`, `tests/fixtures/authoring/`, `tests/tree/test_author_suite.py`
**Estimated scope:** S

## Checkpoint 7
- [ ] Gate passes; prompt, manifest and run work end to end

---

## Phase 8: Revisions, quirks, run history

## Task 43: Prompt history and revision links

**Description:** `<world_dir>/prompt_history.json` (D5f) holds an ordered list of revisions for every prompt owner (node, relationship, story, noise). Each revision has an id (content hash), text, author (user or agent), and the run, day and turn it came from, if any. The manifest holds the head text.

Users edit prompts directly in the manifest (D5k). When `simulate` finds manifest text that differs from the head, it appends a revision authored by `user`. `worldgen prompts history <manifest> <owner>` lists an owner's revisions, and `worldgen prompts revert <manifest> <owner> <revision_id>` appends a new revision with the old text and writes it to the manifest. `prompts.json` in each run snapshot is written from this history.

**Acceptance criteria:**
- [ ] A prompt edited in the manifest becomes a `user` revision at the next `simulate`, and that run's `prompts.json` carries its id
- [ ] Revert restores the old text as a new revision, writing both the manifest and the history in one atomic step
- [ ] Turns link to the revision ids they ran under, so a deviation can be traced to a user edit, a narrator's brief or an agent's quirk

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_revisions.py tests/test_cli.py`
- [ ] Gate passes

**Dependencies:** Tasks 7, 13
**Files likely touched:** `worldgen/engine/revisions.py`, `worldgen/runs/prompts_snapshot.py`, `worldgen/cli.py`
**Estimated scope:** S

## Task 44: `add_quirk`

**Description:** The actor tool `add_quirk` appends text to the actor's own prompt as a new revision, authored by the agent, with the run, day and turn. The revision applies from the actor's next turn. It is written to the run's `prompt_revisions.json`, logged as `quirk_added`, and appended to the live manifest and history in one atomic write, so later runs start from the revised prompt.

**Acceptance criteria:**
- [ ] A quirk added on day d is in that actor's brief from d+1, and not in the rest of d's turn
- [ ] The quirk appears in the live manifest, in the history, in `prompt_revisions.json` and as a `quirk_added` entry
- [ ] The next run's `prompts.json` includes it, and the earlier run's directory is unchanged

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/tools/self tests/engine/test_revisions.py`
- [ ] Gate passes

**Dependencies:** Task 43
**Files likely touched:** `worldgen/tools/self/add_quirk.py`, `worldgen/engine/revisions.py`, `worldgen/runs/entries/quirk_added.py`
**Estimated scope:** S

## Task 45: `runs tail` with filters, and cancel

**Description:**
- `runs tail <run_dir>` prints a live run's log as it is written, newest at the bottom, with the simulated day on each line. On a finished run it replays the log from the files.
- Filters: `--kind`, `--node`, `--entity`, `--institution` and `--deviations-only` (D5i).
- `runs cancel <run_dir>`, or SIGINT or SIGTERM, cancels in-flight turns, marks the run `cancelled` and closes the log cleanly.

**Acceptance criteria:**
- [ ] `tail` prints each entry within a second of its append, and on a finished run its output equals the filtered file content
- [ ] Each filter keeps exactly the matching entries on a fixture run, and `--deviations-only` keeps only deviations
- [ ] `runs cancel` mid-run leaves `status: cancelled` and a readable log

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/runs`
- [ ] Gate passes
- [ ] Manual: `runs tail --node n_morgan` on a live tracer run, then `runs cancel`

**Dependencies:** Tasks 17, 20
**Files likely touched:** `worldgen/runs/{run_store,log_filter,cancel}.py`, `worldgen/cli.py`
**Estimated scope:** M

## Task 46: Rerun from a run

**Description:** `simulate --from-run <run_dir>` starts a sibling run from that run's `manifest.yaml` and `prompts.json` and records `parent_run`. `runs list` shows each run's parent.

**Acceptance criteria:**
- [ ] A `--from-run` child's `manifest.yaml` and `prompts.json` are byte-identical to its parent's
- [ ] Two runs from the same prompts give two separate snapshot directories (SC11)
- [ ] A past run reopens and replays from its own directory alone (tested by copying it elsewhere)

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/runs`
- [ ] Gate passes

**Dependencies:** Task 45
**Files likely touched:** `worldgen/runs/{simulation,run_store}.py`, `worldgen/cli.py`
**Estimated scope:** S

## Task 47: Noise traceability

**Description:** `runs show <run_dir> --noise <id>` lists the rows a noise prompt touched. A row counts if it came from an event made in a turn whose brief included that noise revision. The list is computed from the run's files alone.

**Acceptance criteria:**
- [ ] On a fixture run with one scoped noise prompt, the listed rows are exactly those reached through turns that included it
- [ ] Noise outside every brief's scope lists no rows (SC6)

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/runs/test_noise_trace.py`
- [ ] Gate passes

**Dependencies:** Tasks 20, 29
**Files likely touched:** `worldgen/runs/noise_trace.py`, `worldgen/cli.py`
**Estimated scope:** S

## Checkpoint 8
- [ ] Gate passes; a rerun after a quirk shows the revision in its `prompts.json` and in `runs show`

---

## Phase 9: Completion

## Task 48: Toy `plaid` institution, renderer isolation and docs

**Description:**
- An import test checks that no institution package imports another's tables or renderer.
- `docs/adding-an-institution.md` walks through a toy `plaid` institution added with one renderer file (`institutions/plaid/renderer.py`) and one test file, with no engine edits. A test runs the guide's code.
- `docs/manifest-reference.md` covers every manifest field; a test fails if any model field is undocumented.

**Acceptance criteria:**
- [ ] The import test passes, and fails on a planted cross-import
- [ ] Adding the guide's `plaid` files makes it render without touching `worldgen/engine` (SC8)
- [ ] The manifest reference covers every field

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/test_institution_imports.py tests/docs`
- [ ] Gate passes

**Dependencies:** Tasks 16, 18
**Files likely touched:** `tests/test_institution_imports.py`, `docs/{adding-an-institution,manifest-reference}.md`, `tests/docs/`
**Estimated scope:** S

## Task 49: 18-month run and the full phenomena suite

**Description:** Extend `saas_startup` to TARGET_EXAMPLE's full range (2021-01-01..2022-06-30), with the NYC trip and Contoso noise overlays. Run it live with recording, and add replay tests for all five phenomena.

**Acceptance criteria:**
- [ ] Replay tests for phenomena 1-5 query the output tables and pass (SC2)
- [ ] `verify` is clean (SC3-5, SC12-14)
- [ ] The `runs show` timings and a review note go in `tasks/milestone-18m.md`

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/examples`
- [ ] Gate passes
- [ ] Manual: the user reviews the run with `runs show` and `runs tail`

**Dependencies:** Tasks 39, 44, 46
**Files likely touched:** `examples/saas_startup/manifest.yaml`, `tests/fixtures/llm/saas_18m/`, `tests/examples/test_phenomena_18m.py`, `tasks/milestone-18m.md`
**Estimated scope:** M

## Checkpoint: Complete
- [ ] Every SPEC success criterion is met (map in `tasks/plan.md`)
- [ ] The gate passes, with 100% coverage on `validators/`, `engine/clearinghouse.py` and `agents/loop.py`
- [ ] Ready for review
