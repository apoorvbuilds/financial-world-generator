# Tasks: Financial World Generator

Plan: `tasks/plan.md` (decisions D1-D5, architecture, risks). Spec: `SPEC.md`.

**Rules for every task**

- **Gate.** All three must pass before a task is checked off:
  ```
  uv run pytest -q --cov=worldgen --cov-fail-under=90
  uv run ruff check . && uv run ruff format --check .
  uv run mypy --strict worldgen
  ```
- Every source file listed gets a mirrored test under `tests/`. Test files are not listed separately.
- Tests use `FakeLlm` or `AsAgreedLlm`. Live tests live in `tests/live/` and run only with `WORLDGEN_LIVE=1`.
- Commit after each task. Commit messages carry no Claude attribution lines.

---

## Phase 0: Decisions

## Task 0: Confirm D1-D5 and sync SPEC.md

**Description:** Get the user's answers to D1-D5 in `tasks/plan.md` and write them into SPEC.md as current design, without mentioning removed alternatives. This covers the LLM backend (replacing `ClaudeCodeLlm` and `claude -p`), dropping call caps and pause/resume (SC9, OQ1, run statuses, log kinds), the milestone window, the D4 data-model gap-fills (accounts, event fields, story steps, prompt history file, institution packages, `world.config`) and the narrator's status.

**Acceptance criteria:**
- [ ] D1-D5 each have a recorded answer in `tasks/plan.md`
- [ ] SPEC.md Tech Stack, Project Structure, Core Concepts (Event), Simulation Model, Run history, SC9 and Open Questions match those answers
- [ ] SPEC.md describes only the current design

**Verification:**
- [ ] Manual: the user reads the SPEC.md diff and approves it

**Dependencies:** None
**Files likely touched:** `SPEC.md`, `tasks/plan.md`
**Estimated scope:** S

---

## Phase 1: Foundation and tracer bullet

## Task 1: Project scaffold and CLI entry point

**Description:** Set up a uv project for Python 3.12.
- Runtime dependencies: `pydantic` v2, `PyYAML`, `typer`.
- Dev dependencies: `pytest`, `pytest-cov`, `hypothesis`, `ruff`, `mypy`, `types-PyYAML`, `jsonschema`.
- A `worldgen` console script backed by an empty typer app.
- ruff and `mypy --strict` configured in `pyproject.toml`.
- `.gitignore` ignores `worlds/*/runs/` in place of the stale `worlds/*/out/`.

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

## Task 2: Manifest core models and loader

**Description:** Define pydantic v2 models, all frozen:
- `World`: name, start, end, currency, `schema_version`, optional seed, `config`.
- `Entity`: id, kind, prompt, descriptors, institution.
- `Relationship`: id, parties, kind, prompt, optional terms, channel.
- `Manifest`: the root, loaded from YAML.

Terms are a discriminated union on `type`. This task adds `one_time`, `monthly_fixed` and `accounts` (on `banks_with`, per D4). Money fields parse from 2dp strings to `Decimal` and reject floats. Also write `examples/tracer/manifest.yaml`: an investor, `co_self`, a landlord and Mercury, with an equity wire (one-time), $7,000 rent (monthly), and Mercury X0666 declared under `banks_with`.

**Acceptance criteria:**
- [ ] The tracer manifest loads and survives a YAML round trip
- [ ] A float amount, an unknown field or a wrong type fails with an error naming the YAML path
- [ ] mypy strict passes with no `Any` leaking from the YAML layer

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/manifest`
- [ ] Gate passes

**Dependencies:** Task 1
**Files likely touched:** `worldgen/manifest/{world,entity,relationship,manifest}.py`, `worldgen/manifest/terms/{one_time,monthly_fixed,accounts}.py`, `examples/tracer/manifest.yaml`
**Estimated scope:** M (small model files)

## Task 3: Stories, noise and `worldgen validate`

**Description:** Add two models:
- `Story`: id, plus dated steps (op, day, actor, args).
- `Noise`: a scope (world, entity, relationship, story or step) plus a prompt.

Add two validators:
- Dangling references: relationship parties, account institutions, story actors, noise scopes.
- Time range: story steps and term dates outside the world range.

`worldgen validate <manifest>` prints one error per problem, each pointing at its location (e.g. `relationships[3].parties[1]: unknown entity 'cust_contos'`), and exits 1 if there are any. Unknown op types are added to `validate` in Task 11.

**Acceptance criteria:**
- [ ] One fixture manifest per error class fails with the exact pointer in the message
- [ ] The tracer manifest validates with exit code 0
- [ ] `worldgen/validators/` has 100% line coverage

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/validators tests/test_cli.py`
- [ ] Gate passes
- [ ] Manual: `uv run worldgen validate examples/tracer/manifest.yaml`

**Dependencies:** Task 2
**Files likely touched:** `worldgen/manifest/{story,noise}.py`, `worldgen/validators/{manifest_refs,time_range}.py`, `worldgen/cli.py`, `tests/fixtures/manifests/`
**Estimated scope:** M

## Task 4: Clock, Rng and money

**Description:**
- `Clock` maps dates to integer day indices from the world start. It answers business-day questions on the Federal Reserve holiday calendar, computed in code with no added dependency. On that calendar a Saturday holiday does not close the preceding Friday.
- `Rng` wraps `random.Random`, with an optional best-effort seed, and is the only source of randomness for coded mechanics.
- The `money` helpers quantize `Decimal` to cents, parse and format 2dp strings, and raise on float input.

**Acceptance criteria:**
- [ ] hypothesis: `add_business_days(d, n)` never lands on a weekend or holiday and is at least d+n calendar days
- [ ] Known dates hold: 2021-07-05 is closed, 2021-12-24 is open, 2022-06-20 is closed
- [ ] `format(Decimal("-642.2")) == "-642.20"`; float input raises `TypeError`

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_clock.py tests/engine/test_rng.py tests/test_money.py`
- [ ] Gate passes

**Dependencies:** Task 1
**Files likely touched:** `worldgen/engine/clock.py`, `worldgen/engine/rng.py`, `worldgen/money.py`
**Estimated scope:** S

## Task 5: Accounts, events and ledger

**Description:**
- `AccountRef` kinds: an institution account (from `banks_with` terms), an external account per entity, a clearing account per channel, and `world_boundary`.
- `Event` is frozen, with fields id, day, op, source_entity, dest_entity, source_account, dest_account, amount (positive `Decimal`), cause, `decision_id` (or mechanic name) and related event ids.
- `EventLog` appends to `events.jsonl` and flushes after every write.
- `Ledger` derives each account's balance from events.

**Acceptance criteria:**
- [ ] hypothesis: for any sequence of events, the balances of all accounts (including `world_boundary`) sum to 0
- [ ] `events.jsonl` round-trips exactly, with amounts as 2dp strings
- [ ] An event with a non-positive or float amount is rejected

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/events`
- [ ] Gate passes

**Dependencies:** Task 4
**Files likely touched:** `worldgen/events/{account,event,event_log,ledger}.py`
**Estimated scope:** S

## Task 6: Run directory, `run.json` and `log.jsonl`

**Description:**
- `run_id`: a sortable UTC timestamp plus a 3-character suffix.
- `RunMeta` (`run.json`): status (running, done, failed or cancelled), start and end times, pid, counts, manifest sha256 and `parent_run`.
- `RunLog`: appends `{seq, ts, day, kind, ...}` to `log.jsonl` and flushes after every entry.
- `RunStore`: creates the run directory with a copy of `manifest.yaml` and a `prompts.json` snapshot, and lists and opens runs.

A run context sets `done` on success and `failed` on any exception. Each log entry kind is its own file under `runs/entries/`, added by the task that first emits it. This task adds `run_started` and `run_finished`.

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

## Task 7: Llm interface, fakes and the agent answer schema

**Description:**
- `Llm` protocol: async `complete(LlmRequest) -> LlmResponse`. `LlmRequest` carries system text, prompt and structured metadata (agent id, day, due item ids, default actions) so that fakes can answer without parsing prose.
- `FakeLlm`: either scripted by a callable or replaying a recorded fixture file. It records every call and raises on a fixture miss.
- `AsAgreedLlm`: answers every due item as agreed, using the metadata.
- `AgentAnswer` contains:
  - per-item answers: item id, decision (pay, pay_late, short_pay, skip, dispute, cancel or change), `as_agreed`, actions and `cause`
  - free actions: op, args, cause
  - timers
  - `memory_note`

  The parser pulls the JSON out of the response text and raises a typed error whose message can be fed back to the LLM.

**Acceptance criteria:**
- [ ] An answer with a missing `cause` or an unknown decision fails with a clear message
- [ ] Fixture replay returns the recorded response for a matching request and raises on a miss
- [ ] No module outside the CLI wiring imports a concrete `Llm` (import test)

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/llm tests/engine/agents/test_answer.py`
- [ ] Gate passes

**Dependencies:** Task 1
**Files likely touched:** `worldgen/llm/{base,fake,as_agreed}.py`, `worldgen/engine/agents/answer.py`
**Estimated scope:** S

## Task 8: Compiler: obligations from terms

**Description:** `Compiler` turns a `Manifest` into a `Schedule` with three parts:
- Obligations by day, expanded from relationship terms. Each one names the agent that answers it: the payer, or the biller for billed terms.
- Story steps by day, as wake-ups for the step's actor (D4c).
- The account registry, from `banks_with` terms.

Each terms expander is its own file. This task adds `one_time` and `monthly_fixed`, and later tasks add more. Obligations are clipped to the world range.

**Acceptance criteria:**
- [ ] Rent `monthly_fixed` on day 1 over 2021-03-22..2021-04-05 yields exactly one obligation, on 2021-04-01
- [ ] The equity `one_time` term on the start day yields one obligation, answered by the investor
- [ ] A story step compiles into a wake-up for its actor on its day

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/compiler`
- [ ] Gate passes

**Dependencies:** Tasks 2, 3, 5
**Files likely touched:** `worldgen/compiler/{compiler,obligation,schedule}.py`, `worldgen/compiler/terms/{one_time,monthly_fixed}.py`
**Estimated scope:** M

## Task 9: Clearinghouse and channels

**Description:** Channel rules are data:

| Channel | Settles |
|---|---|
| card | 1-3 calendar days |
| ach | 1-3 business days |
| wire | same or next business day |
| book | same day |

The clearinghouse handles money in flight:
- `enqueue` moves funds from the source into the channel's clearing account (an initiation event) and picks a settle day with `Rng`, within the channel's bounds.
- `settle(day)` emits clearing-to-destination events for everything due that day.
- `in_flight(day)` reports what is still pending.

**Acceptance criteria:**
- [ ] hypothesis: nothing settles before initiation + min delay or after the max, and nothing settles on a weekend or holiday unless the channel allows it
- [ ] Clearing accounts net to zero once everything has settled
- [ ] 100% line coverage on `clearinghouse.py`

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_clearinghouse.py`
- [ ] Gate passes

**Dependencies:** Tasks 4, 5
**Files likely touched:** `worldgen/engine/{channel,clearinghouse}.py`
**Estimated scope:** S

## Task 10: Agent brief and obligation answering

**Description:** `AgentState` holds an agent's prompt, its balances as institutions report them (settled), its pending outflows, open items, recent events and a rolling memory note.

`Brief` builds the decision prompt from ordered sections:
1. The entity prompt.
2. Noise prompts in scope, with revision ids.
3. The agent's own relationships.
4. Balances.
5. Due obligations and story steps.
6. The op menu.
7. The answer schema.

The sections are ordered so a narrator section can be added later (D5).

`Obligations` collects the answers. Invalid JSON gets one re-ask that includes the parse error. Items left unanswered get one re-ask listing only those items; after that, `obligation_unanswered` is logged and nothing is rendered for them.

**Acceptance criteria:**
- [ ] Isolation: no brief contains another entity's prompt, balances or noise (checked across the tracer world)
- [ ] A missing item gets exactly one re-ask for just that item, then an `obligation_unanswered` entry naming the obligation and the agent
- [ ] The brief exposes the revision id (content hash) of every prompt it includes

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/agents tests/engine/test_obligations.py`
- [ ] Gate passes

**Dependencies:** Tasks 3, 7, 8
**Files likely touched:** `worldgen/engine/agents/{agent_state,brief}.py`, `worldgen/engine/agents/templates/decide.md`, `worldgen/engine/obligations.py`, `worldgen/runs/entries/obligation_unanswered.py`
**Estimated scope:** M

## Task 11: Op menu, action validation and transfer ops

**Description:** Each op is one file under `engine/ops/`, declaring its name, its pydantic args and the transfers it makes. `OpMenu` discovers the ops in the package, so adding an op needs no registry edit.

`ActionValidator` checks four things:
- the op exists
- its args parse
- the actor owns the source account
- funds are available, meaning settled balance minus pending outflows, or available credit for a card account

A rejection logs `action_rejected` with its reason and emits nothing. The first ops are `ach_transfer` and `wire_transfer`. `worldgen validate` now also rejects story steps with unknown op types.

**Acceptance criteria:**
- [ ] An overdraft attempt produces `action_rejected` with no event and no row
- [ ] An agent cannot move money out of an account it doesn't own
- [ ] An unknown op in a story step fails `validate` with a pointer, completing SC1

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/ops tests/engine/agents/test_action_validator.py`
- [ ] Gate passes

**Dependencies:** Tasks 3, 7, 9
**Files likely touched:** `worldgen/engine/ops/{op_menu,ach_transfer,wire_transfer}.py`, `worldgen/engine/agents/action_validator.py`, `worldgen/runs/entries/action_rejected.py`
**Estimated scope:** M

## Task 12: Day loop with parallel consultation

**Description:** `Engine.run` drives each day in the fixed order from the plan:
1. Settle the clearinghouse.
2. Run institution mechanics.
3. Find triggered agents, from two triggers: an item due today, or an incoming event from yesterday.
4. Consult them concurrently.
5. Validate and execute their actions.
6. Render.
7. Log `day_closed`.

Every LLM call logs `llm_decision`, with the full prompt, the response, prompt revision ids, causes, `as_agreed` flags and `latency_ms`. Every event logs an `event` entry linked to its decision.

**Acceptance criteria:**
- [ ] Two agents due on the same day are in flight at the same time (`FakeLlm` barrier test)
- [ ] An event created on day d reaches its recipient's brief no earlier than d+1
- [ ] Every due item has an `llm_decision` answering it or an `obligation_unanswered` entry, and every event has a `decision_id` or a mechanic name (SC12)

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_engine.py tests/engine/agents/triggers`
- [ ] Gate passes

**Dependencies:** Tasks 6, 10, 11
**Files likely touched:** `worldgen/engine/engine.py`, `worldgen/engine/agents/triggers/{item_due,incoming_event}.py`, `worldgen/runs/entries/{llm_decision,event,day_closed}.py`
**Estimated scope:** M

## Task 13: Institution plugins and Mercury transactions

**Description:** The `Institution` interface has two parts: a mechanics hook that runs at its step in the day, and a renderer for settled events on the institution's own accounts. Institutions are loaded by name from the manifest, and the engine imports none of them directly.

Mercury renders `mercury_transactions`:
- `transaction_date` is the initiation date and `posted_date` is the settlement date.
- The amount is signed.
- `transaction_type` comes from the channel, and `counterparty_name` is filled in.

The `JsonFiles` sink keeps each table keyed by row id (re-rendering an id replaces the row) and writes `<institution>/<table>.json`. Every render logs `row_rendered` with its linked event ids.

**Acceptance criteria:**
- [ ] The tracer run renders the equity wire and the April 1 rent ACH, each posted on a business day no earlier than its initiation
- [ ] Every row has a `row_rendered` entry linking at least one event
- [ ] `worldgen/engine` imports nothing from `worldgen.institutions.<name>` (import test)

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions tests/sinks`
- [ ] Gate passes

**Dependencies:** Task 12
**Files likely touched:** `worldgen/institutions/{institution,institution_loader}.py`, `worldgen/institutions/mercury/{renderer,rows}.py`, `worldgen/sinks/json_files.py`, `worldgen/runs/entries/row_rendered.py`
**Estimated scope:** M

## Task 14: `worldgen simulate`

**Description:** `Simulation` connects the pieces inside a `RunStore` run context: manifest, compiler, engine, institutions and sinks. `worldgen simulate <manifest> --llm as-agreed|fake:<fixture>` creates `worlds/<world>/runs/<run_id>/` (or a directory set by `--world-dir`) and prints the run directory.

**Acceptance criteria:**
- [ ] `uv run worldgen simulate examples/tracer/manifest.yaml --llm as-agreed` produces a complete run directory: `run.json`, `manifest.yaml`, `prompts.json`, `log.jsonl`, `events.jsonl`, `mercury/mercury_transactions.json`
- [ ] The run ends with `status: done` and a `run_finished` entry with counts
- [ ] Running it twice gives two separate run directories

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/runs/test_simulation.py`
- [ ] Gate passes
- [ ] Manual: run it and read `log.jsonl`

**Dependencies:** Task 13
**Files likely touched:** `worldgen/runs/simulation.py`, `worldgen/llm/llm_factory.py`, `worldgen/cli.py`
**Estimated scope:** S

## Task 15: `worldgen verify`

**Description:** Validators, one file each:
- **conservation:** for each institution account, opening balance + sum of rows = closing balance; across the world, events sum to zero outside `world_boundary`.
- **event_links:** every row links at least one event, with 0 orphans.
- **log_consistency:** every `row_rendered` entry matches a row in the tables and vice versa, and every `event` entry matches `events.jsonl` and vice versa.
- **row_schemas:** every row validates against its row model. Tests also check the output against the TARGET_EXAMPLE JSON Schemas with `jsonschema`.

`worldgen verify <run_dir>` prints a report and exits 1 on any failure.

**Acceptance criteria:**
- [ ] The tracer run verifies clean
- [ ] Each validator catches its corrupted-run fixture: a deleted row, an extra row, a tampered amount, an orphan event
- [ ] `worldgen/validators/` has 100% coverage

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/validators`
- [ ] Gate passes
- [ ] Manual: `uv run worldgen verify <tracer run dir>`

**Dependencies:** Task 14
**Files likely touched:** `worldgen/validators/{conservation,event_links,log_consistency,row_schemas}.py`, `worldgen/institutions/schemas/*.schema.json`, `worldgen/cli.py`
**Estimated scope:** M

## Checkpoint 1: Tracer bullet (after Tasks 1-15)
- [ ] Gate passes
- [ ] `simulate` then `verify` on the tracer world, offline, is clean
- [ ] The user has read one run directory and agrees with the log and event shapes

---

## Phase 2: Live LLM loop

## Task 16: Real Llm backend, recorder and live tracer smoke test

**Description:** Implement the D1 backend behind `Llm`. The proposal is `AnthropicLlm`:
- async client
- model taken from `world.config` or `--model`
- retries with backoff on transient and rate-limit errors
- API key read from the environment only

`RecordingLlm` wraps any `Llm` and writes request/response pairs as `FakeLlm` fixtures. The live smoke test runs the tracer world for its full range.

**Acceptance criteria:**
- [ ] `WORLDGEN_LIVE=1 uv run pytest -q tests/live` runs the tracer live, with a clean `verify` and 0 `obligation_unanswered`
- [ ] The recorded fixtures replay the same run offline through `FakeLlm`, and that run also verifies clean
- [ ] The API key never appears in `log.jsonl`, `run.json` or fixtures (test)

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/llm`, plus the live test above
- [ ] Gate passes

**Dependencies:** Tasks 0 (D1), 15
**Files likely touched:** `worldgen/llm/<backend>.py`, `worldgen/llm/recording.py`, `worldgen/llm/llm_factory.py`, `tests/live/test_tracer_live.py`
**Estimated scope:** M

## Task 17: `worldgen runs list|show` with a loop-quality summary

**Description:** `runs list <world_dir>` prints every run (time, status, counts). `runs show <run_dir>` summarises one run from its files:
- LLM calls
- first-try parse rate
- re-asks
- rejected actions by reason
- unanswered items
- deviations (`as_agreed` false)
- rows per table
- mean latency

This is the measure used to judge the LLM loop.

**Acceptance criteria:**
- [ ] The summary is computed from run files alone and works on a copied run directory
- [ ] A run marked `running` whose pid is gone is listed as `failed`
- [ ] Summary numbers match hand-counted values on a fixture run

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/runs/test_run_summary.py`
- [ ] Gate passes

**Dependencies:** Task 14
**Files likely touched:** `worldgen/runs/{run_summary,run_store}.py`, `worldgen/cli.py`
**Estimated scope:** S

## Checkpoint 2: Live loop (after Tasks 16-17)
- [ ] Gate passes
- [ ] A live tracer run reviewed with the user using `runs show` and `log.jsonl`
- [ ] Decide whether the decision template or answer schema needs changes before scaling out

---

## Phase 3: Institutions and behaviors

## Task 18: Self-initiated triggers and agent memory

**Description:** Three triggers, one file each:
- **timer:** an agent asks to be woken on a given day.
- **drift_tick:** a low-probability daily wake drawn from `Rng`. The rate is set per entity kind in `world.config`, with people highest.
- **balance_threshold:** wakes the owner once when an owned account crosses a floor.

After each answer, the agent's `memory_note` replaces its rolling summary, which has a bounded length.

**Acceptance criteria:**
- [ ] A timer set on day d for d+k wakes the agent on exactly d+k
- [ ] Drift with p=0 never fires and with p=1 fires every day, through the injected `Rng`
- [ ] The balance threshold fires once per downward crossing, not on every day the balance stays below

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/agents/triggers`
- [ ] Gate passes

**Dependencies:** Task 12
**Files likely touched:** `worldgen/engine/agents/triggers/{timer,drift_tick,balance_threshold}.py`, `worldgen/engine/agents/agent_state.py`
**Estimated scope:** S

## Task 19: Ramp card swipes, merchant catalog and descriptors

**Description:** The `card_swipe` op (cardholder, merchant, amount, memo) runs over the card channel. Card accounts are liabilities with a credit limit set in the `banks_with` terms. A merchant is either a vendor entity or an entry in the catalog. The catalog is a data file of realistic merchants, each with a category and a price range. A swipe at an unknown merchant, or outside the catalog price range, is rejected.

`descriptors.py` renders a merchant's statement name from its variants, filling `{ref}` with a random code. The Ramp renderer writes `ramp_card_transactions`:
- `swipe_date` is initiation and `transaction_date` is settlement.
- `cardholder_name` and `cardholder_role` are filled in.
- `merchant_name` comes from the descriptors.
- `ramp_category` is a noisy hint from the catalog.
- `memo` is whatever the agent wrote, or null.

**Acceptance criteria:**
- [ ] A swipe on 2021-03-30 that settles 2021-04-01 renders `swipe_date` 2021-03-30 and `transaction_date` 2021-04-01 (phenomenon 5 at row level, with `Rng` stubbed)
- [ ] A swipe at a merchant in neither the entities nor the catalog produces `action_rejected`
- [ ] Descriptor variants with `{ref}` produce different statement names across rows

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/ramp tests/engine/ops/test_card_swipe.py tests/catalog`
- [ ] Gate passes

**Dependencies:** Tasks 13, 18
**Files likely touched:** `worldgen/engine/ops/card_swipe.py`, `worldgen/catalog/{merchant_catalog.py,merchants.yaml}`, `worldgen/institutions/descriptors.py`, `worldgen/institutions/ramp/{renderer,rows}.py`
**Estimated scope:** M

## Task 20: Ramp statements and autopay

**Description:** Ramp's month-end mechanic closes a statement over transactions by settlement date, renders `ramp_statements`, and backfills `statement_id` on those transactions. Autopay starts an ACH pull of the ending balance from Mercury X0666 a configured number of days after close. The pull shows on both sides: a Mercury `ACH Debit`, and `payments_and_credits` on the next Ramp statement.

**Acceptance criteria:**
- [ ] For every statement, `ending_balance = opening_balance + new_charges - payments_and_credits`
- [ ] A swipe that settles after the cutoff lands on the next statement
- [ ] The autopay row in Mercury links to that statement's events

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/ramp`
- [ ] Gate passes

**Dependencies:** Task 19
**Files likely touched:** `worldgen/institutions/ramp/{statements,autopay,renderer,rows}.py`
**Estimated scope:** S

## Checkpoint 3a: Ramp
- [ ] Gate passes; tracer plus a person with a Ramp card runs `as-agreed`, and `verify` is clean

## Task 21: Stripe invoices, charges and fees

**Description:** `monthly_card` terms (plan price, renewal day) go on `customer_of` relationships, and the customer answers each renewal: pay, card fails, cancel or downgrade. The `stripe_charge` op runs over the card channel into the company's Stripe balance account. A Stripe mechanic takes the fee of 2.9% + $0.30 (in cents, rounded half up). The renderer writes `stripe_invoices` (status paid, open or void) and `stripe_charges` (amount, fee, net, linked invoice).

**Acceptance criteria:**
- [ ] hypothesis over amounts: `net = amount - fee` holds exactly and the money is conserved
- [ ] A failed card leaves an `open` invoice and no charge row
- [ ] Every charge links its invoice

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/stripe tests/compiler/terms`
- [ ] Gate passes

**Dependencies:** Tasks 8, 13
**Files likely touched:** `worldgen/compiler/terms/monthly_card.py`, `worldgen/engine/ops/stripe_charge.py`, `worldgen/institutions/stripe/{fees,renderer,rows}.py`
**Estimated scope:** M

## Task 22: Stripe payout batching into Mercury

**Description:** The payouts mechanic runs each business day. It sweeps the available Stripe balance (charges older than the availability delay set in config) into one payout, sent by ACH to Mercury X0666 and arriving 1-2 business days later. It renders `stripe_payouts`, with status `in_transit` until arrival and `paid` after, and backfills `payout_id` on the swept charges. Mercury renders the arrival as a single `Deposit - STRIPE` ACH Credit.

**Acceptance criteria:**
- [ ] Phenomenon 2: N charges produce one payout row and one Mercury row, and the sums agree
- [ ] Phenomenon 1: charges on March 29-31 give `payout_date` 2021-03-30 and `arrival_date` 2021-04-01, absent from Mercury's March rows
- [ ] A payout still in flight when the run ends stays `in_transit`

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/stripe`
- [ ] Gate passes

**Dependencies:** Task 21
**Files likely touched:** `worldgen/institutions/stripe/{payouts,renderer,rows}.py`, `worldgen/institutions/mercury/renderer.py`
**Estimated scope:** S

## Task 23: Stripe refunds, disputes and expense pulls

**Description:** Two new ops:
- `stripe_refund` (full or partial, with a reason), decided by the company agent in response to a customer's request event.
- `card_dispute`, decided by the customer. Stripe then reverses the amount and adds a dispute fee.

Refunds are netted against the next payout. When refunds exceed the available balance, Stripe pulls the difference from Mercury by ACH, rendered there as `Stripe - Expense - STRIPE`.

**Acceptance criteria:**
- [ ] `refunds_deducted` equals the sum of the refunds swept into that payout, and every refund carries its `charge_id`
- [ ] A negative net produces a negative `net_payout_amount` and a Mercury ACH Debit row
- [ ] Conservation holds across a refund-heavy fixture

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/stripe tests/engine/ops`
- [ ] Gate passes

**Dependencies:** Task 22
**Files likely touched:** `worldgen/engine/ops/{stripe_refund,card_dispute}.py`, `worldgen/institutions/stripe/{payouts,renderer}.py`
**Estimated scope:** M

## Checkpoint 3b: Stripe
- [ ] Gate passes; a fixture world with card customers and refunds verifies clean, and the phenomenon 1 and 2 tests pass

## Task 24: Rippling payroll

**Description:** `semi_monthly` terms (days 10 and 26, gross pay per employee) go on `employed_by` relationships, and the company agent answers each payroll: run as agreed, change after a hire, or skip. The `payroll_run` op triggers Rippling mechanics:
- Compute withholding and employer taxes from a coded flat-rate table.
- Render `rippling_journal_entries`, numbered in sequence with balanced debit and credit lines.
- Start ACH debits from Mercury X0666 for net pay and for taxes.

**Acceptance criteria:**
- [ ] Journal entry numbers rise by 1 per payroll run from the number set in config, and each entry balances
- [ ] The Mercury debits equal the net-pay and tax credit lines and link to the payroll events
- [ ] A skipped payroll produces no journal entry and no debit, and has an `llm_decision` with a cause

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/rippling`
- [ ] Gate passes

**Dependencies:** Tasks 8, 13
**Files likely touched:** `worldgen/compiler/terms/semi_monthly.py`, `worldgen/engine/ops/payroll_run.py`, `worldgen/institutions/rippling/{payroll,renderer,rows}.py`
**Estimated scope:** M

## Task 25: Mercury statements, treasury, interest and fees

**Description:** Mercury month-end mechanics:
- `mercury_statement_line_items` per account, with a running balance.
- `mercury_treasury_records`: beginning balance, deposits, withdrawals, interest, ending balance.
- Interest credited at month end on checking and Treasury, at annual rates set in config.
- Service fees.
- Returns: a debit started by an institution against insufficient funds is returned and a fee row is rendered.

Also adds the `internal_transfer` op between a company's own accounts, over the book channel.

**Acceptance criteria:**
- [ ] `running_balance` carries over continuously between months and equals the ledger balance at each line
- [ ] The March statement excludes items posted April 1
- [ ] Treasury `ending = beginning + deposits - withdrawals + interest`, and a returned debit is followed by its fee row

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/institutions/mercury`
- [ ] Gate passes

**Dependencies:** Task 13
**Files likely touched:** `worldgen/institutions/mercury/{statements,treasury,interest,rows}.py`, `worldgen/engine/ops/internal_transfer.py`
**Estimated scope:** M

## Task 26: Cohorts

**Description:** A cohort entity has a member kind, an initial size and a growth prompt. Its members get names and emails generated from `Rng` and a name catalog. All the cohort's items due on a day are answered in a single bulk call: everyone as agreed, apart from listed exceptions (churn, card fails, downgrade, dispute, refund request), plus any new signups. `cohorts.py` expands the answer into one decision and one op per member. Calls are split above the member count set in config (default 500).

**Acceptance criteria:**
- [ ] 212 due with 7 exceptions gives 205 as-agreed charges, and the 7 exceptions are handled as stated
- [ ] An exception naming an unknown member gets one re-ask, then `obligation_unanswered` for that member
- [ ] A new signup is first charged on its signup day

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_cohorts.py`
- [ ] Gate passes

**Dependencies:** Task 21
**Files likely touched:** `worldgen/manifest/entity.py`, `worldgen/engine/cohorts.py`, `worldgen/engine/agents/cohort_answer.py`, `worldgen/catalog/names.py`, `worldgen/compiler/compiler.py`
**Estimated scope:** M

## Checkpoint 3c: All institutions
- [ ] Gate passes; all four institutions render, and fixture worlds verify clean
- [ ] Review the state of the system with the user before the milestone

---

## Phase 4: First milestone

## Task 27: `examples/saas_startup` manifest and as-agreed baseline

**Description:** Hand-write the TARGET_EXAMPLE world as a manifest over the D2 window:
- the company and its investor
- the five people, each with a Ramp card
- Mercury (with its three accounts), Stripe, Ramp and Rippling
- the self-serve cohort
- the enterprise customers, including Contoso
- vendors, each with descriptor variants
- relationships with terms as in TARGET_EXAMPLE section 4

The first version is plain, as TARGET_EXAMPLE section 4.1 asks: no stories and no one-off noise. The NYC trip story and the Contoso noise go in `examples/saas_startup/extras/` as overlays. `validate` and `simulate` accept `--overlay <file>`, which merges lists by id.

**Acceptance criteria:**
- [ ] `validate` exits 0, with and without each overlay
- [ ] `simulate --llm as-agreed` followed by `verify` is clean
- [ ] A test confirms every entity and relationship from TARGET_EXAMPLE sections 3 and 4 is present, by id

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/examples`
- [ ] Gate passes

**Dependencies:** Tasks 20, 23, 24, 25, 26
**Files likely touched:** `examples/saas_startup/manifest.yaml`, `examples/saas_startup/extras/{nyc_trip,contoso_noise}.yaml`, `worldgen/manifest/manifest.py`, `worldgen/cli.py`
**Estimated scope:** M

## Task 28: Live 15-day milestone run

**Description:** Run `saas_startup` live over the D2 window through `RecordingLlm`. Commit the recorded fixtures; they are synthetic, with no real PII. Add replay tests that query the output tables for phenomena 1, 2 and 5. Write a review note in `tasks/milestone-15d.md` with the `runs show` summary, sample deviations, and any decisions that looked unrealistic.

**Acceptance criteria:**
- [ ] The live run verifies clean, with 0 unanswered items (or each one explained in the note)
- [ ] Replay tests for phenomena 1, 2 and 5 pass offline
- [ ] The review note is written

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/examples/test_phenomena_15d.py`
- [ ] Gate passes
- [ ] Manual: the user reads the note and browses the tables

**Dependencies:** Tasks 16, 27
**Files likely touched:** `tests/fixtures/llm/saas_15d/`, `tests/examples/test_phenomena_15d.py`, `tasks/milestone-15d.md`
**Estimated scope:** S

## Checkpoint 4: Milestone (after Tasks 27-28)
- [ ] Gate passes
- [ ] The user reviews the live 15-day run and decides on any prompt or schema changes
- [ ] D5 (narrator) is answered or explicitly deferred again before Phase 7

---

## Phase 5: Revisions, quirks, run history

## Task 29: Prompt history and revision links

**Description:** `worlds/<world>/prompt_history.json` holds, for every prompt owner (main, entity, relationship, story, noise), an ordered list of revisions: id (content hash), text, author (user or agent), run_id and timestamp. The manifest holds the head text; a validator checks that the head matches the manifest. Reverting appends a new revision with the old text. Run snapshots write `prompts.json` from this history.

**Acceptance criteria:**
- [ ] Editing a prompt adds a revision and updates the manifest in a single atomic write
- [ ] Revert restores the old text as a new revision
- [ ] `validate` fails when the head revision and the manifest text disagree

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_revisions.py tests/validators`
- [ ] Gate passes

**Dependencies:** Tasks 6, 10
**Files likely touched:** `worldgen/engine/revisions.py`, `worldgen/runs/prompts_snapshot.py`, `worldgen/validators/prompt_history.py`
**Estimated scope:** S

## Task 30: Agent quirks after a run

**Description:** Before `run_finished`, each agent that acted gets one call with its memory and decisions, and may propose a quirk. A quirk is appended to the agent's prompt as a revision, with author `agent`, the run, and the day and decision it came from. It is written to the run's `prompt_revisions.json`, logged as `quirk_added`, and added to the live history, so later runs start from it. It is controlled by `--quirks/--no-quirks` (on by default).

**Acceptance criteria:**
- [ ] A `FakeLlm` quirk appears in the live history, in `prompt_revisions.json` and as a `quirk_added` entry
- [ ] The next run's `prompts.json` includes the quirk
- [ ] Nothing in the run directory is written after `run_finished`

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/test_quirks.py`
- [ ] Gate passes

**Dependencies:** Task 29
**Files likely touched:** `worldgen/engine/quirks.py`, `worldgen/runs/entries/quirk_added.py`, `worldgen/engine/revisions.py`, `worldgen/runs/simulation.py`
**Estimated scope:** S

## Task 31: Runs tail, rerun from a run, cancel

**Description:** `runs tail <run_dir>` follows a live `log.jsonl`. `simulate --from-run <run_dir>` starts a sibling run from that run's `manifest.yaml` and `prompts.json` and records `parent_run`. SIGINT or SIGTERM marks the run `cancelled` and closes the log cleanly.

**Acceptance criteria:**
- [ ] A `--from-run` child's `manifest.yaml` and `prompts.json` are byte-equal to its parent's
- [ ] Two runs from the same prompts give two separate snapshot directories (SC11)
- [ ] SIGTERM mid-run leaves `status: cancelled` and a readable log

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/runs`
- [ ] Gate passes
- [ ] Manual: `runs tail` on a live tracer run

**Dependencies:** Task 14
**Files likely touched:** `worldgen/runs/{tail,run_store}.py`, `worldgen/runs/simulation.py`, `worldgen/cli.py`
**Estimated scope:** S

## Task 32: Institution situations

**Description:** Institution entities get agents too. Mechanics wake them for noteworthy items, such as a large or first payout or an inbound wire over a threshold. An institution agent can apply a situation from its own menu, either holding a payout for N days or returning a wire, and these are validated like any other action. Its default answer is to proceed.

**Acceptance criteria:**
- [ ] A held payout arrives N business days late, with money still conserved
- [ ] A returned wire sends the funds back, with a fee row
- [ ] Every situation has an `llm_decision` with a cause

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/engine/ops tests/institutions/stripe`
- [ ] Gate passes

**Dependencies:** Tasks 22, 25
**Files likely touched:** `worldgen/engine/agents/triggers/institution_review.py`, `worldgen/engine/ops/{hold_payout,return_wire}.py`, `worldgen/institutions/stripe/payouts.py`
**Estimated scope:** M

## Checkpoint 5
- [ ] Gate passes; a rerun after a quirk shows the revision in its `prompts.json` and in `runs show`

---

## Phase 6: Dashboard

## Task 33: Dashboard server and world view

**Description:** `worldgen dashboard <manifest>` serves a local app on the D3 stack. Its page lists the world (time range), entities, relationships, stories, noise, and every prompt with its head revision. JSON endpoints are read-only in this task.

**Acceptance criteria:**
- [ ] Every manifest item appears on the page
- [ ] The endpoints are tested with the framework's test client
- [ ] There is no frontend build step

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/dashboard`
- [ ] Gate passes
- [ ] Manual: open the dashboard for `saas_startup` in a browser

**Dependencies:** Tasks 0 (D3), 29
**Files likely touched:** `worldgen/dashboard/app.py`, `worldgen/dashboard/routes/world.py`, `worldgen/dashboard/static/{index.html,app.js}`, `worldgen/cli.py`
**Estimated scope:** M

## Task 34: Edit prompts and run from the dashboard

**Description:** Any prompt can be edited inline, which creates a revision authored by `user`. Each prompt shows its revision list, with revert. A Run button starts a simulation in the background and returns the run id, and run status is read from `run.json`.

**Acceptance criteria:**
- [ ] An edit updates both the manifest and the history, and the new revision is visible
- [ ] The Run button creates a run directory
- [ ] Editing while a run is going doesn't change that run's snapshot

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/dashboard`
- [ ] Gate passes
- [ ] Manual: edit a prompt, run, and see the new run listed

**Dependencies:** Tasks 31, 33
**Files likely touched:** `worldgen/dashboard/routes/{prompts,runs}.py`, `worldgen/dashboard/run_launcher.py`, `worldgen/dashboard/static/app.js`
**Estimated scope:** M

## Task 35: Live log panel and live tables

**Description:** `live_stream.py` tails `log.jsonl` and sends each entry over SSE exactly as written. The run log panel shows the newest entry at the bottom, with the simulated day on every line, and filters by kind, entity, institution and "deviations only". The tables view updates each institution's tables as `row_rendered` entries arrive, fetching row content by id.

**Acceptance criteria:**
- [ ] The streamed content equals the file content (test reads both)
- [ ] Rows appear in the browser as soon as a run starts
- [ ] The deviations filter shows only items where `as_agreed` is false

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/sinks tests/dashboard`
- [ ] Gate passes
- [ ] Manual: start a run and watch the log and tables fill in the browser

**Dependencies:** Task 34
**Files likely touched:** `worldgen/sinks/live_stream.py`, `worldgen/dashboard/routes/stream.py`, `worldgen/dashboard/static/{log_panel,tables}.js`
**Estimated scope:** M

## Checkpoint 6a
- [ ] Gate passes; edit a prompt, run, and watch rows stream end to end in the browser, with the user

## Task 36: Calendar and decision detail

**Description:** A month grid is shaded by activity. Each day view lists events, decisions with their causes, quirks, and rows by institution. A decision's detail shows the prompt, the response, links to its prompt revisions, and its linked rows. The entity, institution and deviation filters are shared with the log panel.

**Acceptance criteria:**
- [ ] The day counts on the grid match `log.jsonl`
- [ ] Links in decision detail resolve to the revision text and to the rows
- [ ] The filters apply to the calendar and the log panel alike

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/dashboard`
- [ ] Gate passes
- [ ] Manual: browse the milestone run in the calendar

**Dependencies:** Task 35
**Files likely touched:** `worldgen/dashboard/routes/calendar.py`, `worldgen/dashboard/static/{calendar,decision_detail}.js`
**Estimated scope:** S

## Task 37: Run history, replay and compare

**Description:**
- **Run list:** time, status, revisions, row counts and deviation counts.
- **Replay:** open a past run and play its log in the log panel at a chosen speed.
- **Compare:** open two runs side by side, aligned by simulated day, with days whose event sets differ highlighted.

**Acceptance criteria:**
- [ ] A past run renders from its own directory alone (tested by copying it elsewhere)
- [ ] On a fixture pair, compare flags exactly the days with different event sets
- [ ] Replay speed changes take effect while it plays

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/runs/test_run_diff.py tests/dashboard`
- [ ] Gate passes
- [ ] Manual: compare two milestone runs

**Dependencies:** Task 35
**Files likely touched:** `worldgen/dashboard/routes/history.py`, `worldgen/runs/run_diff.py`, `worldgen/dashboard/static/history.js`
**Estimated scope:** S

## Task 38: Noise traceability

**Description:** For any noise prompt, list the rows it touched. A row counts if it came from an event made by a decision whose brief included that noise revision. This is shown on the noise item in the world view and by `runs show --noise <id>`.

**Acceptance criteria:**
- [ ] On a fixture run with one scoped noise prompt, the listed rows are exactly those reached through decisions that included it
- [ ] An out-of-scope noise prompt lists no rows (SC6)

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/runs/test_noise_trace.py tests/dashboard`
- [ ] Gate passes

**Dependencies:** Tasks 17, 33
**Files likely touched:** `worldgen/runs/noise_trace.py`, `worldgen/dashboard/routes/world.py`, `worldgen/dashboard/static/app.js`, `worldgen/cli.py`
**Estimated scope:** S

## Checkpoint 6b
- [ ] Gate passes; the user does a full dashboard walkthrough on the milestone world

---

## Phase 7: Authoring (needs D5)

## Task 39: Authoring tools and the world and entity agents

**Description:** Authoring agents call tools, one file each, to build a manifest draft. This task adds the tools `set_world`, `add_entity` and `validate_draft`. Tool calls are JSON in responses through `Llm`, so fakes can replay them.
- `world_agent` reads the main prompt and sets up the world.
- `entity_agent` creates entity shells with prompts and descriptor variants, using a cohort for any long tail.

The templates enforce a plain first generation: only what the prompt asks for.

**Acceptance criteria:**
- [ ] With `FakeLlm` fixtures, a TARGET_EXAMPLE-like prompt gives a world and entities that pass model validation
- [ ] A prompt with no one-offs produces no stories and no noise
- [ ] A tool call with bad arguments returns the error to the agent instead of raising

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/authoring`
- [ ] Gate passes

**Dependencies:** Tasks 0 (D5), 3, 7
**Files likely touched:** `worldgen/authoring/tools/{set_world,add_entity,validate_draft}.py`, `worldgen/authoring/agents/{world_agent,entity_agent}.py`, `worldgen/authoring/templates/`
**Estimated scope:** M

## Task 40: Relationship, story and noise agents with a repair loop

**Description:** Adds the tools `add_relationship`, `add_story` and `add_noise`, and the relationship, story and noise agents. `author.py` runs the agents in order, validates the draft, and sends each error back to the agent responsible, for up to N rounds.

**Acceptance criteria:**
- [ ] A fixture whose first draft has a dangling ref is repaired within N rounds
- [ ] An error routes to the agent that owns that part of the manifest
- [ ] A draft still invalid after N rounds ends with a clear report

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/authoring`
- [ ] Gate passes

**Dependencies:** Task 39
**Files likely touched:** `worldgen/authoring/tools/{add_relationship,add_story,add_noise}.py`, `worldgen/authoring/agents/{relationship_agent,story_agent,noise_agent}.py`, `worldgen/authoring/author.py`
**Estimated scope:** M

## Task 41: `worldgen author` and the 20-prompt fixture suite

**Description:** `worldgen author --prompt ... --out <manifest>` writes the manifest and seeds the prompt history, including the main prompt. 20 fixture prompts with recorded responses are replayed in tests. As a live check, TARGET_EXAMPLE.md is used as the prompt and the result compared by hand against `examples/saas_startup`.

**Acceptance criteria:**
- [ ] At least 18 of the 20 fixture prompts produce manifests that pass `validate` (SC7)
- [ ] The authored manifest opens in the dashboard and runs
- [ ] A live authoring run from TARGET_EXAMPLE.md is reviewed with the user

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/authoring`
- [ ] Gate passes
- [ ] Manual: author, then open the dashboard, then run

**Dependencies:** Task 40
**Files likely touched:** `worldgen/cli.py`, `tests/fixtures/authoring/`, `tests/authoring/test_author_suite.py`
**Estimated scope:** S

## Checkpoint 7
- [ ] Gate passes; prompt to manifest to dashboard to run works end to end

---

## Phase 8: Completion

## Task 42: Isolation property, toy `plaid` institution and docs

**Description:**
- A property test checks that no institution package imports another's tables or renderer.
- `docs/adding-an-institution.md` walks through a toy `plaid` institution added with one renderer file and one test file, with no engine edits. A test checks the guide's code.
- `docs/manifest-reference.md` covers every manifest field; a test fails if a model field is undocumented.

**Acceptance criteria:**
- [ ] The isolation test passes and fails on a planted cross-import
- [ ] Adding the guide's `plaid` files makes it render without touching `worldgen/engine` (SC8)
- [ ] The manifest reference covers every field

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/test_isolation.py tests/docs`
- [ ] Gate passes

**Dependencies:** Tasks 13, 15
**Files likely touched:** `tests/test_isolation.py`, `docs/{adding-an-institution,manifest-reference}.md`, `tests/docs/`
**Estimated scope:** S

## Task 43: 18-month run and the full phenomena suite

**Description:** Extend `saas_startup` to TARGET_EXAMPLE's full range (2021-01-01..2022-06-30), with the NYC trip and Contoso noise overlays. Run it live with recording and add replay tests for all five phenomena.

**Acceptance criteria:**
- [ ] Replay tests for phenomena 1-5 query the output tables and pass (SC2)
- [ ] `verify` is clean (SC3-5), and every due item is answered or logged (SC12)
- [ ] `runs show` timings and the review note are added to `tasks/milestone-18m.md`

**Verification:**
- [ ] Tests pass: `uv run pytest -q tests/examples`
- [ ] Gate passes
- [ ] Manual: the user reviews the run in the dashboard

**Dependencies:** Tasks 28, 32
**Files likely touched:** `examples/saas_startup/manifest.yaml`, `tests/fixtures/llm/saas_18m/`, `tests/examples/test_phenomena_18m.py`, `tasks/milestone-18m.md`
**Estimated scope:** M

## Checkpoint: Complete
- [ ] Every SPEC success criterion is met (map in `tasks/plan.md`)
- [ ] The gate passes, with 100% coverage on `validators/` and `engine/clearinghouse.py`
- [ ] Ready for review
