# Spec: Financial World Generator

Status: DRAFT for review. Generalizes the single-scenario `TARGET_EXAMPLE.md`, the kind of output this system should produce. `TARGET_EXAMPLE.md` is written as a detailed world description that doubles as the main prompt handed to the authoring agents. It is also expressed as `examples/saas_startup` (one instance of this system).

## Assumptions (correct me before planning starts)

1. Greenfield: no implementation exists in this repo yet.
2. Python 3.12. The core is **non-deterministic by design**: an LLM decides every event an entity produces, inside the simulation loop, so noise and behavior are steered by prompts attached on the dashboard.
3. For now the output is **JSON tables only**: AccountingBench-style `source_data` tables (bank, card, processor, payroll) emitted as JSON and streamed to the dashboard, which displays them in real time as they are generated.
4. The world manifest is the single contract between "authoring" and "simulation": a human or an agent can write it, the simulator only reads it.

## Objective

Create near-real financial scenarios and situations: many varied, realistic synthetic financial worlds. A world is generated top-down in *intent* but simulated bottom-up in *money movement*:

```
world generator -> entity + relationship generator -> story generator -> noise (at every level)
        \_______________ authoring agents write ______________/
                              |
                        World Manifest ("dashboard": entities, relationships, stories, one-offs, prompts, time range)
                              |
                  compiler -> discrete-event engine -> per-institution renderers
                              |                              |
                   event log          source_data tables (Penrose-like)
```

How a run works: each entity's agent, acting on its own relationships, stories and prompts, decides its events within the time range without seeing the rest of the world. All events go into one shared stream. Each institution (Mercury, Stripe, Ramp, Rippling) takes the events that involve it and writes them as rows in its own JSON tables, and cross-cutting behavior such as settlement delays and payout batching happens there. The dashboard is interactive: edit any prompt, rerun, and the data is recreated.

Users: (a) authors who want a new world by prompt, (b) reviewers who inspect and edit the manifest in a dashboard and rerun the simulation.

The product is the **dashboard**. Code builds the skeleton (entity shells, relationship map, time range) and every entity, relationship, story and noise item carries an associated prompt. These prompts are where one-offs live (a late payment, a personal charge, a disputed invoice) and all of them are editable from the dashboard; data is generated from them.

The first version generated from the main prompt is plain: it creates what was asked for and nothing more, with no one-offs added on its own. Reviewers then add and edit one-offs in the per-entity prompts and regenerate.

Success looks like: a reviewer types a one-paragraph prompt, reviews a generated manifest, runs the simulator, and gets institution tables with realistic friction (settlement lags, batched payouts, messy descriptors, personal commingling) plus an event log that explains where every row came from.

## Tech Stack

- Python 3.12, `pydantic` v2 (manifest + event models), `PyYAML`, `typer` (CLI); LLM calls (authoring agents and every run-time decision) go through the `Llm` interface, whose only implementation for now is `ClaudeCodeLlm` (one file in `worldgen/llm/`), which calls the locally installed Claude Code in headless mode (`claude -p`) and so runs on the user's Claude subscription. No API key and no `anthropic` SDK dependency yet
- JSON tables as the only output format; a streaming channel (e.g. server-sent events or websocket) pushes rows to the dashboard as they are generated
- `pytest`, `hypothesis` (property tests), `ruff`, `mypy --strict`
- Money is `decimal.Decimal` everywhere; never `float`. Dates are `datetime.date`; engine time is integer day index from `start_date`.

## Commands

```
Install:   uv sync
Test:      uv run pytest -q --cov=worldgen --cov-fail-under=90
Lint:      uv run ruff check . && uv run ruff format --check .
Types:     uv run mypy --strict worldgen
Author:    uv run worldgen author --prompt "B2B SaaS, 18 months, 5 staff" --out worlds/demo/manifest.yaml
Validate:  uv run worldgen validate worlds/demo/manifest.yaml
Simulate:  uv run worldgen simulate worlds/demo/manifest.yaml     # new run in worlds/demo/runs/<run_id>/: <institution>/<table>.json + log.jsonl
Runs:      uv run worldgen runs list|show|tail worlds/demo   # run history; `tail` follows a live run's log
Verify:    uv run worldgen verify worlds/demo/runs/<run_id>        # ledger invariants + every row traces to an event
Dashboard: uv run worldgen dashboard worlds/demo/manifest.yaml   # edit prompts, run/rerun the simulation, live JSON table view
```

## Project Structure

Convention: one file per agent, tool, template, story op, institution mechanic, renderer, booking rule and validator.

```
worldgen/
  manifest/        -> pydantic models: World, Entity, Relationship, Story, Noise, TimeRange
  authoring/
    agents/        -> world_agent.py, entity_agent.py, relationship_agent.py, story_agent.py, noise_agent.py
    tools/         -> one file per tool the agents may call
    templates/     -> prompt templates (one per agent)
  compiler/        -> manifest -> obligation schedule (resolves refs, expands relationship terms into dated obligations)
  engine/
    clock.py       -> day tick, business-day calendar, holidays
    rng.py         -> Rng: the injected source of randomness for coded mechanics
    agents/        -> agent state, wake-up triggers, action validation, one file per trigger
    obligations.py -> open obligations per agent; asks again, then logs, any left unanswered
    cohorts.py     -> expands a cohort's bulk answer into one event per member
    revisions.py   -> prompt revision history
    clearinghouse.py -> in-flight ACH/wire/card queue with settlement delays
    ops/           -> one file per story operation (invoice, refund, payroll_run, card_swipe, ...)
  institutions/    -> one renderer per institution: mercury.py, stripe.py, ramp.py, rippling.py; descriptors.py (merchant descriptor per row)
  events/          -> event schema + event log writer
  booking/         -> one file per booking rule (derived books, post-v1 policy variants)
  validators/      -> one file per invariant (conservation, isolation, schema)
  llm/             -> base.py (Llm interface), claude_code.py (ClaudeCodeLlm, headless `claude -p`), fake.py (test double)
  sinks/           -> json_files.py, live_stream.py, run_log.py (append-only log.jsonl writer; live_stream tails it)
  runs/            -> run_id.py, run_meta.py, run_store.py (list/open), one file per log entry kind
  cli.py
examples/saas_startup/   -> TARGET_EXAMPLE expressed as a manifest
tests/                    -> mirrors worldgen/ layout
docs/                     -> manifest reference, adding-an-institution guide
```

## Core Concepts

| Concept | Definition |
|---|---|
| **Entity** | Isolated actor with its own state and clock: customer, vendor, person, institution, (later) tax jurisdiction. Authoring generates entities as mostly empty shells (id, kind, a few fields) that carry a `prompt` by default; the prompt, not coded fields, is what drives the data later generated for that entity. Writes only to its own tables. |
| **Relationship** | An edge between entities (customer-of, employed-by, banks-with, ...). Carries a `prompt` (e.g. "annual prepay, disputes invoices") and optional terms: amount, schedule, payment channel. This is where recurring obligations such as subscriptions, prepays and rent live. The compiler turns terms into dated obligations, and the agent that owes or bills each one decides what actually happens. Terms may be fixed or left to the prompt, so amounts and timing can be non-deterministic. The set of relationships is the map of the world, and the main structure the code owns. |
| **Obligation** | A dated item an entity owes or bills under a relationship's terms (rent due on the 1st, a renewal on the 14th, payroll on the 10th). It wakes the agent that owes or bills it, and that agent's answer produces the events (see Simulation Model). |
| **Story** | Time-ordered causal chain of ops (e.g. "CEO NYC trip closes Contoso expansion") that triggers money movement through relationships and channels. |
| **Noise / one-off** | A free-text prompt attached at any level (world, entity, relationship, story, single op) and added to the brief of every agent in its scope, which acts on it when deciding events: blank or terse memos, duplicate charges, refunds, late or short payments, personal commingling. Institution-side messiness comes from institution mechanics and situations: each merchant's statement name is rendered from its `descriptors` (variants written by the authoring LLM), and posting lags come from the clearinghouse. |
| **Op** | Atomic engine action emitting zero or more events, e.g. `card_swipe`, `stripe_charge`, `payout_batch`, `ach_pull`. |
| **Event** | Immutable record: `id, day, op, source_entity, dest_entity, amount, cause` (`cause` is a short plain-text summary of why it happened, e.g. "Contoso annual prepay due under r_contoso"). Every rendered row traces back to events. |

## Simulation Model

Decisions are made while the clock runs, not planned up front.

- **Agents:** every entity runs as an agent. It holds its prompt and its own state: balances, open obligations, recent events and a short rolling memory summary. It sees only its own information, never the rest of the world.
- **Wake-up triggers:** an agent is consulted by the LLM only when triggered, to keep calls sparse. Triggers are an incoming event (an invoice arrives), an obligation falling due, a timer the agent set for itself, a low-probability random drift tick (mostly for people), and a balance crossing a threshold.
- **LLM decides every event:** the LLM decides every event an entity produces; paying rent on time is a decision just like paying it late. Most answers are "as agreed", so a world stays mostly ordinary, but recurring flows drift the way real ones do (rent paid on the 3rd one month, a vendor raising its price, payroll changing after a hire).
- **Obligations:** when obligations fall due, the agent is woken with a brief listing each open one and must answer every item (pay, pay late, short-pay, skip, dispute, cancel, change), each with a `cause`. If an answer leaves an item out, the engine asks once more for just the missing items, then logs `obligation_unanswered` and renders nothing for it.
- **Cohorts:** a large group of similar entities (the self-serve subscribers) runs as one agent that answers in bulk, e.g. "all 212 renew except these 7: 3 churn, 2 cards fail, 1 downgrades, 1 disputes". The LLM decides; the engine expands the answer into one event per member.
- **Institution mechanics are coded:** the LLM decides what entities do; code decides how institutions record it. Fees, payout batching, settlement timing, statements, autopay pulls and interest are coded institution mechanics. An institution's agent can still decide situations on top of them (hold a payout for review, return a wire), validated like any other action.
- **Actions:** the LLM chooses from a fixed menu of ops with arguments. The engine validates every action (the op exists, funds are available, money is conserved) and executes it. The LLM never writes rows. A catalog of realistic merchants and price ranges keeps invented activity plausible.
- **Reacting to its own finances:** institutions report balances back to the account's owner. Low cash lets an agent move money between its own accounts, delay or cut spending, chase receivables, or raise cash. A payment attempted without funds surfaces as a rejected payment or fee. Customers react the same way (pay late, short-pay, cancel). These feedback loops are intended, and each decision is logged with its `cause`.
- **Parallelism:** agents cannot see each other, so all agents active on a simulated day are consulted in parallel. The day closes when all have answered, and the events they produced become visible to others from the next day. Simulated time runs faster than wall-clock time.
- **Quirks and prompt revisions:** after a run an agent may propose a quirk, which is appended to its prompt as a new revision, visible and editable on the dashboard. Each revision records its text, its author (a user or an agent), the run and time it came from, and can be reverted. Later runs start from the revised prompt, so worlds can grow more eccentric over repeated runs.
- **Logging:** runs are not reproducible, so every LLM prompt and response is recorded in the event log, along with the prompt revision it was made under. Each run also writes a human-readable run log (see Run history) that is shown live and kept for every run.

### Run history (log)

Every run is kept, never overwritten, and is a self-contained snapshot of that simulated world. Each run gets its own directory `<world_dir>/runs/<run_id>/` (`run_id` is a sortable timestamp plus short suffix) holding everything that went in and everything that came out:

```
worlds/demo/runs/20261028T142201Z-a3f/
  run.json          -> metadata: start/end time, status (running|paused|done|failed|cancelled), counts, manifest sha256, parent_run (if rerun from another run)
  manifest.yaml     -> exact copy of the manifest as run (entities, relationships, stories, noise, time range)
  prompts.json      -> every prompt in force at start, keyed by owner (main, entity, relationship, story, noise), each with its revision id, text and author
  prompt_revisions.json -> revisions added during the run (agent quirks), with the day and decision that produced them
  log.jsonl         -> append-only, one JSON object per line, in the order things happened (includes full LLM prompts and responses)
  events.jsonl      -> events for this run
  mercury/*.json  stripe/*.json  ramp/*.json  rippling/*.json
```

- **Snapshot rule:** a run directory is never edited after it finishes and never refers to files outside itself (no pointers to the live manifest or current prompts). Editing the world afterward changes nothing in past runs. Opening a run shows the prompts, manifest, decisions and tables exactly as they were.
- **Rerun:** rerunning with the same initial prompts creates a new sibling run directory with its own manifest copy, prompts, log and tables. Because the core is non-deterministic, the two snapshots may differ, and the dashboard can compare them. Rerunning from an old run ("restore and run") loads that run's `manifest.yaml` and `prompts.json` as the starting point, and records it as `parent_run`.

- **Log entries:** `{seq, ts, day, kind, ...}` where `kind` is `run_started`, `day_closed`, `event`, `llm_decision` (prompt, response, prompt revision, `cause`), `action_rejected`, `obligation_unanswered` (obligation id, agent), `quirk_added`, `row_rendered` (institution, table, row id, linked event ids), `run_paused` / `run_resumed` (reason: rate limit or call cap) or `run_finished`. The log is the run's narrative; the event log (`events.jsonl`) remains the source of truth for money, and `log.jsonl` references it by event id rather than duplicating amounts.
- **Written as it happens:** the engine appends each entry and flushes it as soon as it occurs (including on crash, so a failed run keeps a partial log and `status: failed`). The live stream sends exactly the entries being written, so the file and the dashboard can never disagree.
- **Displayed live:** the dashboard has a run log panel that tails the active run, newest at the bottom, with the simulated day on each line. It filters by kind, entity and institution, and "deviations only". Clicking an entry opens the same detail as the calendar (prompt and response, linked rows).
- **History:** a run list shows every past run (time, status, prompt revisions, row and deviation counts). Opening one replays its log in the same panel at any speed, and two runs can be opened side by side to compare the days where they diverge. Deleting a run removes its directory.
- **Retention:** no pruning for now. Runs are never deleted automatically and there is no prune command; `log.jsonl` is plain text and may grow large, which is accepted.

### Dashboard tracking

- **Calendar:** each simulated day shows its events, LLM decisions (with `cause`), quirks added or triggered, and the rows that landed in each institution's table. Opening a decision shows the prompt and response.
- **Zoom levels and filters:** a month grid shaded by activity, a day view with the full list, filters by entity or institution, and a "deviations only" filter showing days where an LLM changed something.
- **Prompt history:** every prompt has a revision list. Decisions link to the revision they were made under, so a deviation can be traced to a user edit or an agent-added quirk.

## Manifest Shape (excerpt)

```yaml
world: {name: demo, start: 2021-01-01, end: 2022-06-30, currency: USD}   # seed: optional, best-effort
entities:
  - {id: cust_contoso, kind: customer, prompt: "enterprise buyer, slow approvals, pays by wire"}
  - {id: cohort_selfserve, kind: cohort, member_kind: customer, prompt: "self-serve monthly plans ($25-$270) paid by card through Stripe; grows steadily; a few churn, fail or dispute each month"}
  - {id: v_amazon, kind: vendor, prompt: "equipment and supplies, bought on Ramp cards", descriptors: ["AMZN Mktp US*{ref}", "Amazon.com*{ref}", "AMZN MKTP US"]}   # written by the authoring LLM; {ref} becomes a random reference code
  - {id: inst_mercury, kind: institution, institution: mercury}
relationships:
  - id: r_contoso
    parties: [cust_contoso, co_self]
    kind: customer_of
    prompt: "annual prepay, expands mid-term after the NYC trip"
    terms: {type: prepaid_annual, amount: "85363.20", start: 2021-02-01, end: 2021-11-30}   # optional; compiled into dated obligations that cust_contoso's agent answers
    channel: wire
stories:
  - id: s_nyc_trip
    ops:
      - {op: card_swipe, day: 2021-03-11, actor: p_ceo, merchant: DELTA AIR LINES, amount: "642.20"}
noise:
  - {scope: {story: s_nyc_trip}, prompt: "leaves memos blank on the personal weekend charges"}
  - {scope: {entity: cust_contoso}, prompt: "pays late and short-pays once, then sends a corrective wire"}
```

## Code Style

```python
# worldgen/institutions/descriptors.py
import string

from worldgen.engine.rng import Rng
from worldgen.manifest.entity import Entity


class Descriptors:
    """Render how a merchant's name appears on one statement row."""

    ref_chars = string.ascii_uppercase + string.digits

    def render(self, merchant: Entity, rng: Rng) -> str:
        pattern = rng.choice(merchant.descriptors or [merchant.name])
        ref = "".join(rng.choice(self.ref_chars) for _ in range(9))
        return pattern.replace("{ref}", ref)
```

Conventions: `snake_case` files named for the single thing they hold; one public class per file; docstring is one line; randomness through the injected `Rng`, and LLM calls through an injected `Llm` client, never imported directly, so tests can substitute fakes; no module-level state.

## Testing Strategy

- **Unit** (`tests/<mirror path>`): every op, institution mechanic, renderer, validator. LLM and `Rng` faked so tests stay stable even though the core is not deterministic.
- **Property** (`hypothesis`): conservation (sum of events per account = rendered balance), isolation (an institution renderer never imports another's tables), settlement (no row posted before `initiated + min_delay`, none on weekends/holidays unless the channel allows).
- **Invariants over goldens**: no checksum goldens. `examples/saas_startup` is run and checked for the TARGET_EXAMPLE phenomena and the validators, not for exact rows.
- **LLM calls** (authoring and run time): mocked with recorded fixtures; tests assert manifests validate, refs resolve and invariants hold, never exact prose. One opt-in live smoke test behind `WORLDGEN_LIVE=1`.
- **Coverage**: >= 90% lines on `worldgen/`, 100% on `validators/` and `engine/clearinghouse.py`.

## Boundaries

- **Always:** use `Decimal` for money; route randomness through `Rng` and LLM calls through `Llm`; log every LLM decision to the event log; validate every agent action against the op menu and available funds; run tests + ruff + mypy before commit; add a validator when adding an invariant; keep one-thing-per-file layout; update `docs/` when the manifest schema changes.
- **Ask first:** adding runtime dependencies; changing the manifest schema in a breaking way (bump `schema_version`); changing output table schemas; choosing an accounting policy for derived books; any real network call beyond Claude Code's own; adding an API-key-based `Llm` implementation.
- **Never:** let an LLM write rendered rows without a matching event (money must still be conserved); let one institution renderer read another's tables; use `float` for amounts; hardcode special-case friction (it must emerge from op delays and clearinghouse rules); commit API keys or generated worlds containing real PII.

## Success Criteria

1. `worldgen validate` rejects manifests with dangling refs, ops outside the time range, or unknown op types, each with a pointing error message.
2. `examples/saas_startup` reproduces every phenomenon in TARGET_EXAMPLE section 6 (in-transit month-end Stripe payout, many-to-one payout batching, enterprise prepay vs delivery, exec commingling, Ramp swipe-to-settle cutoff lag), asserted by tests that query the output tables.
3. Variation without breakage: two runs of the same manifest may differ, but every run passes conservation and event-link checks (criteria 4 and 5).
4. Conservation: for every institution account, opening balance + sum(rendered rows) = closing balance; sum of cash events across the world = 0 (outside the world boundary account).
5. Event links: 100% of rendered rows link to at least one event; `worldgen verify` reports 0 orphans.
6. Noise is traceable: every `llm_decision` records the noise prompts in its brief (by revision id), and the dashboard lists, for any noise prompt, the rows it touched. Noise is checked by the invariants.
7. `worldgen author` turns a one-paragraph prompt into a manifest that passes `validate` on >= 90% of 20 fixture prompts (mocked-LLM replay), producing an interactive dashboard that lists entities, relationships, stories, noise, prompts and time range. Editing any prompt and pressing run regenerates the JSON tables, shown live as they are produced.
8. A new institution can be added by creating one renderer file and one test file, with no edits to engine core (demonstrated by a toy `plaid` renderer in the docs guide).
9. Rows start streaming to the dashboard as soon as a run starts, and a run stays within its configured LLM limits (`max_llm_calls` and `max_concurrent_llm_calls` per run, defaults set in the world config). Since nothing happens without the LLM, hitting a limit pauses the run instead of failing it: on the subscription rate limit it logs `run_paused`, sets `status: paused` and resumes when the limit resets; on `max_llm_calls` it stays paused until the cap is raised on the dashboard or the run is cancelled. The simulated clock does not advance while paused.
10. Every agent action is validated: an action outside the op menu or beyond available funds is rejected and logged, never rendered.
11. Run history: every run creates `runs/<run_id>/` with `run.json` and an append-only `log.jsonl`; rerunning never overwrites an earlier run. Every `row_rendered` entry matches a row in that run's tables and every `event` entry matches an event, and vice versa (tested). Entries reach the dashboard as they are appended, a killed run leaves a readable partial log with `status: failed`, and a past run can be reopened and replayed from its files alone. Each run directory contains its own `manifest.yaml` and `prompts.json`; editing the live manifest or prompts afterward leaves earlier run directories byte-identical (tested), and two runs started from the same initial prompts produce two separate snapshot directories.
12. Every obligation is answered: each obligation due within a run has either an `llm_decision` answering it or an `obligation_unanswered` entry, and no event pays an obligation without such a decision (tested).

## Open Questions

1. **LLM provider and budget:** resolved for now: Claude only, run on the Claude subscription through headless Claude Code (`ClaudeCodeLlm`), so there is no per-token cost, but usage is bounded by subscription rate limits. Controlled by per-run `max_llm_calls` and `max_concurrent_llm_calls` (see Success Criteria 9). Remaining questions: sensible default values for those caps, and whether headless Claude Code on a subscription is allowed and fast enough for many parallel calls per simulated day. Check this first: every event needs an LLM call, so nothing moves without it. If not, add an API-key `Llm` implementation behind the same interface.
2. **LLM or coded:** resolved: the LLM decides every entity event; institution mechanics are coded (see Simulation Model).
3. **Obligation briefs:** how far ahead an agent's brief looks (whether a vendor that bills monthly is woken once per bill or answers several months at once), and how large a cohort's bulk answer can get before it must be split.
4. **Run history retention:** resolved as keep every run as a full self-contained snapshot, with LLM prompt/response bodies stored in the run's `log.jsonl` (see Run history). No pruning for now.