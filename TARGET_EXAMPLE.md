# Multi-Entity Event-Driven Financial Data Simulator Specification

**Reference Benchmark:** [AccountingBench (Penrose)](https://accounting.penrose.com/)

## 1. Executive Summary & Core Philosophy

Traditional synthetic financial datasets fail because they are generated **top-down from accounting records**: a script invents balanced journal entries first and then reverse-engineers clean rows in bank, credit card, and payment processor tables. That approach eliminates the real-world friction of money movement—asynchronous settlement lags, many-to-one payout batching, inconsistent vendor descriptors, and arbitrary human behavior.

This simulator reverses that paradigm by modeling a **bottom-up, multi-agent economic world** with zero accounting assumptions baked into the generation process:

1. **Causal Grounding:** Every dollar movement is triggered by an underlying real-world event—an exchange of goods or services, a contract renewal, a payroll cycle, or a human persona's weekly work and personal activities.
2. **Strict Entity Isolation ("Blind Entities"):** Each external participant and financial institution (Customers, Vendors, Employees, Stripe, Ramp, Rippling, Mercury Bank) operates as an isolated agent with its own local state and clock. When an entity acts, it **completely ignores** entities not directly involved in that step and writes exclusively to its own tables.
3. **Narrative-Driven Human Arbitrariness:** Rather than generating uncorrelated random numbers for discretionary spending, a two-stage **Human Persona & Lifestyle Agent** first writes a weekly/monthly narrative diary of what executives and staff did in their corporate and personal lives, then translates those narrative beats into realistic clusters of transactions (including executives spending business funds on personal items).

---

## 2. System Architecture & Simulation Layers

The simulator runs across a configurable multi-month timeline governed by a daily tick clock and three decoupled layers:

```text
+-----------------------------------------------------------------------------------+
| LAYER 1: WORLD CLOCK & ASYNCHRONOUS CLEARINGHOUSE ENGINE                          |
| - Steps simulation day-by-day (00:00 -> 23:59 UTC)                                |
| - Enforces weekends, federal bank holidays, and T+1 / T+2 / T+3 ACH settlement    |
| - Holds in-flight card authorizations, ACH pulls, and wires until maturity        |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| LAYER 2: ECONOMIC COUNTERPARTY & HUMAN NARRATIVE AGENTS (Trigger Events)          |
|                                                                                   |
|  +------------------------+  +------------------------+  +---------------------+  |
|  | Customer Cohort Agent  |  | Vendor/Supplier Agent  |  | Workforce & Human   |  |
|  | - Self-serve SaaS      |  | - Cloud (Vercel/AWS)   |  | Persona Diary Agent |  |
|  | - Enterprise contracts |  | - SaaS tools / Ads     |  | - Exec personal use |  |
|  | - Refunds & disputes   |  | - Rent, Legal, Amazon  |  | - Travel & meals    |  |
|  +-----------+------------+  +-----------+------------+  +----------+----------+  |
+--------------|---------------------------|--------------------------|-------------+
               | (Card Charges / Wires)    | (Card Swipes / ACH)      | (Swipes / Payroll / Wires)
               v                           v                          v
+-----------------------------------------------------------------------------------+
| LAYER 3: FINANCIAL INSTITUTION AGENTS (Isolated State -> Write to `source_data`)  |
|                                                                                   |
|  +-----------------+   +-----------------+   +-----------------+   +-----------+  |
|  | Stripe Agent    |   | Ramp Card Agent |   | Rippling Agent  |   | Mercury   |  |
|  | -> stripe_*     |   | -> ramp_*       |   | -> rippling_*   |   | Bank      |  |
|  | (T+2/wk payout) |   | (1-3d settle)   |   | (10th & 26th)   |   | -> merc_* |  |
|  +--------+--------+   +--------+--------+   +--------+--------+   +-----+-----+  |
|           |                     |                     |                  ^        |
|           +---------------------+---------------------+------------------+        |
|                         (Settled ACH / Wire Events via Clearinghouse)             |
+-----------------------------------------------------------------------------------+
```

---

## 3. Detailed Specification of Entity Agents & Tools

Every agent is restricted to its own toolset and local view of the world. No agent may query or mutate another entity's internal tables.

### 3.1 Layer 1: World Clock & Clearinghouse Engine (`sim_engine`)
Acts as the physics engine for time and money movement between institutions.
* **What It Knows:** Current simulation timestamp, business-day calendar, pending card authorizations, and in-flight ACH/wire transfers between external institutions and Mercury Bank.
* **What It Ignores:** Customer identities, internal company operations, and platform-specific data.
* **Tools:**
  * `advance_day_tick(current_date: date) -> dict`: Advances the simulation clock by 1 day, firing scheduled calendar triggers (payroll dates, subscription renewals, statement cutoffs) and maturing in-flight clearinghouse transfers.
  * `enqueue_clearinghouse_transfer(origin_entity: str, dest_bank_account: str, amount: Decimal, initiated_date: date, settlement_delay_days: int, raw_memo: str, metadata: dict) -> UUID`: Places an ACH credit/debit or wire transfer into the interbank queue to settle after `settlement_delay_days` business days.
  * `deliver_matured_transfers(current_date: date) -> list[UUID]`: Delivers all transfers whose `effective_settlement_date <= current_date` to the target bank agent (`Mercury Bank Agent`).

### 3.2 Layer 2A: Customer Cohort Agent (`counterparty.customers`)
Simulates hundreds of self-serve SaaS subscribers and a roster of enterprise B2B buyers.
* **What It Knows:** Only its own customer profile—subscription tier ($25, $37, $39, $80, $139, or $270/month; or annual enterprise prepay contracts like `Contoso` at $85,363.20/year), renewal date, payment method (Stripe card vs. direct Mercury wire/ACH), and churn/refund probability ([AccountingBench](https://accounting.penrose.com/)).
* **What It Ignores:** Startup bank balances, Stripe's payout batching, and other customers.
* **Tools:**
  * `sign_subscription_contract(customer_id: UUID, plan_type: str, billing_cycle: str, amount: Decimal, start_date: date, end_date: date, payment_channel: str) -> UUID`: Creates a service agreement for monthly or multi-month/annual SaaS access.
  * `trigger_customer_billing_event(customer_id: UUID, contract_id: UUID, billing_date: date) -> dict`: Emits an invoice/charge request to the `Stripe Processor Agent` (for card customers) or initiates a wire/ACH via the `Clearinghouse` (for enterprise invoice customers).
  * `request_customer_refund_or_dispute(customer_id: UUID, charge_id: UUID, request_date: date, amount: Decimal, reason: str) -> UUID`: Triggers a full or partial refund or chargeback event in Stripe days or weeks after the original payment.

### 3.3 Layer 2B: Vendor & Supplier Agent (`counterparty.vendors`)
Simulates external businesses providing cloud infrastructure, SaaS tools, advertising, office space, and professional services.
* **What It Knows:** Its own service delivery schedule, pricing model (fixed monthly tier, usage-based compute metering, or annual prepaid contract), and how it bills the startup (charging a stored Ramp card, pulling an authorized ACH debit from Mercury, or sending a Net-30 invoice).
* **What It Ignores:** Startup bank balances, other vendors, and internal company operations.
* **Vendor Roster Categories:**
  * **Cloud & Compute Providers:** `VERCEL INC - Monthly Pro Plan` ($20.00/mo), `AMAZON WEB SERVICES` (variable usage), `RUNPOD.IO` (GPU bursts) ([AccountingBench](https://accounting.penrose.com/)).
  * **Marketing & Ad Networks:** `Facebook Ads` ($400.00 threshold billing), `LinkedIn` ($281.46 campaigns), `GOOGLE ADS` ([AccountingBench](https://accounting.penrose.com/)).
  * **SaaS Tools:** `FIGMA`, `NOTION LABS`, `SLACK TECHNOLOGIES`, `LINEAR`, `GITHUB`, `GWORKSPACE`.
  * **Facilities, Legal & Operations:** Commercial landlord (monthly wire/ACH rent `$7,000.00`), corporate counsel (quarterly retainer/Net-30 invoice), `Amazon` (`AMZN MKTP US` equipment/supplies) ([AccountingBench](https://accounting.penrose.com/)).
* **Tools:**
  * `deliver_vendor_service(vendor_id: UUID, service_period_start: date, service_period_end: date, amount: Decimal, description: str) -> UUID`: Logs the actual delivery of goods/services.
  * `charge_startup_ramp_card(vendor_id: UUID, card_id: UUID, txn_date: date, amount: Decimal, merchant_descriptor: str) -> UUID`: Sends a card authorization request to the `Ramp Card Agent`.
  * `initiate_vendor_ach_or_wire_request(vendor_id: UUID, bank_account_num: str, txn_date: date, amount: Decimal, ach_descriptor: str) -> UUID`: Submits a direct ACH debit or wire payment instruction to the `Clearinghouse`.
  * `issue_net30_vendor_invoice(vendor_id: UUID, invoice_date: date, due_date: date, amount: Decimal, service_period: tuple[date, date]) -> UUID`: Delivers an invoice to be paid up to 30 days later.

### 3.4 Layer 2C: Workforce & Human Persona Diary Agent (`counterparty.humans`)
Simulates both structured employment (salaried work periods) and unstructured, narrative-driven human life and corporate behavior.
* **What It Knows:** Each human's role (e.g., CEO/Co-founder, CTO/Co-founder, VP of Sales, Senior Engineers, Office Manager), base salary, hire/termination dates, assigned Ramp card, bank access permissions, and personal lifestyle habits.
* **What It Ignores:** Payment processor settlement logic and other financial institutions' internal tables.
* **Tools (Two-Stage Narrative-to-Transaction Pipeline):**
  * `generate_persona_period_diary(persona_id: UUID, period_start: date, period_end: date, company_context: dict) -> dict`: Generates a weekly or monthly narrative summary of what the human did across both corporate duties (client travel, conferences, team dinners, emergency hardware purchases) and personal life (family vacations, personal shopping, gym memberships, personal tax payments, accidental card mix-ups).
  * `emit_diary_ramp_swipes(persona_id: UUID, diary_id: UUID, swipe_events: list[dict]) -> list[UUID]`: Translates narrative events into correlated `Ramp Card Agent` authorizations with realistic dates, city codes, merchant strings, tips, and tax amounts.
  * `submit_ramp_employee_memo(persona_id: UUID, ramp_txn_id: UUID, memo_text: str, receipt_uploaded: bool) -> bool`: Simulates the human attaching (or neglecting to attach) a note to their Ramp swipe—ranging from clear business notes (*"Client dinner with Acme VP"*) to personal admissions (*"Accidentally used company card for personal Uber Eats, please deduct"*) or blank memos.
  * `initiate_executive_bank_draw(persona_id: UUID, mercury_account: str, txn_date: date, amount: Decimal, counterparty_memo: str, is_personal_draw: bool) -> UUID`: Allows an executive with direct bank credentials to wire or ACH money out of `Mercury Checking X0666` for either off-cycle business needs or personal executive spending ([AccountingBench](https://accounting.penrose.com/)).
  * `log_employee_pay_period(persona_id: UUID, pay_period_start: date, pay_period_end: date, salary_earned: Decimal, bonus: Decimal, reimbursements: Decimal) -> UUID`: Registers completed work for the `Rippling Payroll Agent` to process on the 10th and 26th ([AccountingBench](https://accounting.penrose.com/)).

### 3.5 Layer 3A: Stripe Processor Agent (`institution.stripe`)
Operates as an isolated payment processor between customers and the startup's bank.
* **What It Knows:** Customer invoices, card charges, refunds/disputes, Stripe processing fees (`2.9% + $0.30`), and its own unsettled balance.
* **What It Ignores:** Mercury Bank balances, Ramp cards, and Rippling payroll.
* **Tools:**
  * `create_stripe_invoice_and_charge(customer_email: str, invoice_num: str, txn_date: date, gross_amount: Decimal, metadata: dict) -> UUID`: Writes an invoice and charge record into `source_data.stripe_invoices` and `source_data.stripe_charges` with its calculated processing fee ([AccountingBench](https://accounting.penrose.com/)).
  * `record_stripe_refund(charge_id: UUID, refund_date: date, refund_amount: Decimal, is_negative_balance_pull: bool) -> UUID`: Writes a refund to `source_data.stripe_refunds`; if daily refunds exceed charges, initiates an ACH debit (`Stripe - Expense - STRIPE`, e.g., `-$35.39` or `-$282.70`) from Mercury Checking via the Clearinghouse ([AccountingBench](https://accounting.penrose.com/)).
  * `execute_stripe_payout_batch(payout_date: date) -> UUID`: Sweeps accumulated net charges minus fees into a single payout record in `source_data.stripe_payouts` and enqueues an ACH credit (`Deposit - STRIPE`) in the Clearinghouse that settles in `Mercury Checking X0666` 1–2 business days later ([AccountingBench](https://accounting.penrose.com/)).

### 3.6 Layer 3B: Ramp Corporate Card Agent (`institution.ramp`)
Operates as an isolated corporate charge card platform.
* **What It Knows:** Cardholder authorizations, settlement delays (1–3 days after swipe), merchant category hints, employee memos, and monthly statement cycles.
* **What It Ignores:** What sits in Mercury, Stripe, or Rippling.
* **Tools:**
  * `authorize_and_settle_card_txn(card_id: UUID, swipe_date: date, settle_date: date, merchant_name: str, amount: Decimal, ramp_category_hint: str, employee_memo: str) -> UUID`: Writes settled card transactions to `source_data.ramp_card_transactions` ([AccountingBench](https://accounting.penrose.com/)). Swipes near month-end (e.g., March 30) that settle in early April create natural cutoff differences.
  * `generate_ramp_monthly_statement(statement_month: str, cutoff_date: date) -> UUID`: Compiles all settled transactions up to `cutoff_date` into `source_data.ramp_statements` and schedules an autopay ACH pull from `Mercury Checking X0666` via the Clearinghouse ([AccountingBench](https://accounting.penrose.com/)).

### 3.7 Layer 3C: Rippling Payroll Agent (`institution.rippling`)
Operates as an isolated payroll and HR tax engine.
* **What It Knows:** Employee salaries, tax withholding rates, benefit deductions, and bi-monthly pay dates (e.g., the 10th and 26th of each month) ([AccountingBench](https://accounting.penrose.com/)).
* **What It Ignores:** All non-payroll company activity.
* **Tools:**
  * `run_bimonthly_payroll_export(payroll_date: date, je_num: int) -> list[UUID]`: Aggregates active employees' gross pay, employer taxes, benefits, and net pay splits, writing multi-line rows into `source_data.rippling_journal_entries` (e.g., `JE #18` on `2021-03-10` and `JE #19` on `2021-03-26`) ([AccountingBench](https://accounting.penrose.com/)).
  * `initiate_payroll_ach_debits(payroll_date: date, net_pay_total: Decimal, tax_impound_total: Decimal) -> list[UUID]`: Enqueues the corresponding cash debits against `Mercury Checking X0666` via the Clearinghouse.

### 3.8 Layer 3D: Mercury Bank & Treasury Agent (`institution.mercury`)
Operates as the commercial bank housing `Mercury Checking X0666`, `Mercury Checking X3354`, and `Mercury Treasury` ([AccountingBench](https://accounting.penrose.com/)).
* **What It Knows:** Only settled cash movements delivered by the Clearinghouse, internal transfers between its three accounts, monthly treasury yield rates, and bank service fees.
* **What It Ignores:** Which individual customer invoices make up a `Deposit - STRIPE` batch, or unposted checks/card swipes that haven't settled.
* **Tools:**
  * `post_mercury_settled_transaction(account_code: str, settled_date: date, description: str, amount: Decimal, txn_type: str) -> UUID`: Appends a settled line to `source_data.mercury_transactions` (e.g., `Deposit - STRIPE`, `Stripe - Expense - STRIPE`, `ACH Debit - RAMP`, `ACH Debit - RIPPLING`) ([AccountingBench](https://accounting.penrose.com/)).
  * `accrue_and_post_bank_interest_and_fees(month_end_date: date) -> list[UUID]`: Posts month-end interest earned (e.g., `Interest Accrual` `$0.82` on checking or larger yields in `source_data.mercury_treasury_records`) and monthly service fees ([AccountingBench](https://accounting.penrose.com/)).
  * `publish_mercury_monthly_statements(month_end_date: date) -> list[UUID]`: Generates official statement headers and line items in `source_data.mercury_statement_line_items` for `X0666`, `X3354`, and `Mercury Treasury` ([AccountingBench](https://accounting.penrose.com/)).

---

## 4. Deep Dive: The Two-Stage Human Narrative & Lifestyle Engine

To ensure human-triggered transactions are realistic rather than uniform noise, the **Workforce & Human Persona Diary Agent** simulates a fixed cast of company insiders each week:

| Persona | Role & Access | Typical Corporate Activity | Typical Personal / Commingled Activity |
| :--- | :--- | :--- | :--- |
| **Alex Rivera** | Co-Founder & CEO *(Full Ramp + Mercury Wire/ACH access)* | Investor pitches in SF/NYC, board dinners, SaaS executive tools, conference flights. | Uses corporate Ramp card for personal Equinox gym, Peloton, weekend family hotels, personal groceries (`WHOLE FOODS`), and wires personal estimated taxes or car lease payments from Mercury. |
| **Sam Chen** | Co-Founder & CTO *(Ramp Card + AWS/Cloud admin)* | Scales AWS/Vercel infrastructure, buys test GPUs (`RUNPOD.IO`), replaces MacBook Pros for new hires. | Orders late-night DoorDash during outages; occasionally buys personal home-automation gear on Amazon (`AMZN MKTP US`) with corporate card and notes *"Personal - will reimburse"*. |
| **Jordan Taylor** | VP of Sales *(Ramp Card)* | Takes enterprise prospects to steakhouses, books last-minute flights/Ubers, buys client renewal gifts. | Extends a 3-day sales trip in Miami over the weekend and charges personal beach club dining to Ramp without a receipt. |
| **Casey Patel** | Senior Engineer *(Virtual Ramp Card)* | Renews GitHub/Linear seats, buys monitor cables, orders team lunch during sprint planning. | Clean card usage; occasionally forgets to upload a receipt or writes terse memos (`"cables"`). |
| **Morgan Lee** | Ops & Office Manager *(Ramp Card + Vendor Billing)* | Pays office rent, orders Costco snacks, books Friday team happy hours, manages annual insurance prepayments. | Accidentally uses Ramp card stored in personal Uber Eats profile on a Sunday evening and immediately adds a Ramp memo flagging the mistake. |

### Example Narrative-to-Transaction Trace (Week 2 of March 2021)
1. **Stage 1 (`generate_persona_period_diary`):**
   > *"CEO Alex Rivera flew to New York (March 11–14) to close the Contoso enterprise expansion. Hosted Contoso executives at Mastro's Steakhouse on March 12. Stayed two extra personal nights at the hotel with spouse (March 13–14), bought personal AirPods Max at the Fifth Ave Apple Store on Saturday March 13 using the company Ramp card, and initiated an ACH payment from Mercury Checking X0666 on March 15 to cover a personal vehicle lease."*
2. **Stage 2 (Triggered Entity Events):**
   * `Ramp Card Agent` receives 4 swipes on Alex's card:
     * `2021-03-11`: `DELTA AIR LINES` — `$642.20` (Memo: *"NYC flight for Contoso meeting"*)
     * `2021-03-12`: `MASTRO'S STEAKHOUSE NYC` — `$784.50` (Memo: *"Dinner w/ Contoso VP Eng & CFO"*)
     * `2021-03-13`: `APPLE STORE #R001 NYC` — `$598.81` (Memo: *""* — left blank, Saturday timestamp, consumer retail)
     * `2021-03-14`: `THE STANDARD HIGH LINE` — `$1,800.00` (Memo: *"4 nights NYC - 2 nights Contoso biz, 2 nights personal weekend"*)
   * `Clearinghouse` -> `Mercury Bank Agent` (`X0666`) receives 1 ACH Debit:
     * `2021-03-16`: `ACH Debit - PORSCHE FINANCIAL` — `($1,150.00)`

---

## 5. PostgreSQL `source_data` Schema Specification

Each financial institution agent writes strictly to its own tables inside the `source_data` schema ([AccountingBench](https://accounting.penrose.com/)).

```sql
CREATE SCHEMA IF NOT EXISTS source_data;

-- 1. Mercury Bank Transactions & Statements (Written only by Mercury Bank Agent)
CREATE TABLE source_data.mercury_transactions (
    id UUID PRIMARY KEY,
    account_name TEXT NOT NULL, -- 'Mercury Checking X0666', 'Mercury Checking X3354', 'Mercury Treasury'
    transaction_date DATE NOT NULL,
    posted_date DATE NOT NULL,
    description TEXT NOT NULL,
    amount NUMERIC(14, 2) NOT NULL, -- Positive for deposits, negative for withdrawals
    transaction_type TEXT NOT NULL, -- 'ACH Credit', 'ACH Debit', 'Wire', 'Internal Transfer', 'Interest', 'Fee'
    counterparty_name TEXT
);

CREATE TABLE source_data.mercury_statement_line_items (
    id UUID PRIMARY KEY,
    statement_month VARCHAR(7) NOT NULL, -- e.g. '2021-03'
    account_name TEXT NOT NULL,
    date DATE NOT NULL,
    description TEXT NOT NULL,
    amount NUMERIC(14, 2) NOT NULL,
    running_balance NUMERIC(14, 2) NOT NULL
);

CREATE TABLE source_data.mercury_treasury_records (
    id UUID PRIMARY KEY,
    statement_month VARCHAR(7) NOT NULL,
    beginning_balance NUMERIC(14, 2) NOT NULL,
    deposits NUMERIC(14, 2) NOT NULL,
    withdrawals NUMERIC(14, 2) NOT NULL,
    interest_earned NUMERIC(14, 2) NOT NULL,
    ending_balance NUMERIC(14, 2) NOT NULL
);

-- 2. Ramp Corporate Credit Card (Written only by Ramp Card Agent)
CREATE TABLE source_data.ramp_card_transactions (
    id UUID PRIMARY KEY,
    cardholder_name TEXT NOT NULL,
    cardholder_role TEXT NOT NULL,
    swipe_date DATE NOT NULL,
    transaction_date DATE NOT NULL, -- Settlement date on Ramp
    merchant_name TEXT NOT NULL,
    amount NUMERIC(14, 2) NOT NULL, -- Positive for spend, negative for merchant refunds
    ramp_category TEXT,             -- Noisy platform category hint
    memo TEXT,                      -- Human-entered memo (may be NULL, helpful, or personal admission)
    statement_id UUID
);

CREATE TABLE source_data.ramp_statements (
    id UUID PRIMARY KEY,
    statement_month VARCHAR(7) NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    opening_balance NUMERIC(14, 2) NOT NULL,
    new_charges NUMERIC(14, 2) NOT NULL,
    payments_and_credits NUMERIC(14, 2) NOT NULL,
    ending_balance NUMERIC(14, 2) NOT NULL,
    autopay_date DATE
);

-- 3. Stripe Payment Processor (Written only by Stripe Processor Agent)
CREATE TABLE source_data.stripe_invoices (
    id UUID PRIMARY KEY,
    invoice_number TEXT NOT NULL,
    customer_email TEXT NOT NULL,
    customer_name TEXT,
    invoice_date DATE NOT NULL,
    service_period_start DATE NOT NULL,
    service_period_end DATE NOT NULL,
    amount_due NUMERIC(14, 2) NOT NULL,
    status TEXT NOT NULL -- 'paid', 'open', 'void'
);

CREATE TABLE source_data.stripe_charges (
    id UUID PRIMARY KEY,
    invoice_id UUID REFERENCES source_data.stripe_invoices(id),
    customer_email TEXT NOT NULL,
    charge_date DATE NOT NULL,
    amount NUMERIC(14, 2) NOT NULL,
    fee_amount NUMERIC(14, 2) NOT NULL,
    net_amount NUMERIC(14, 2) NOT NULL,
    payout_id UUID -- Links to stripe_payouts once swept into a batch
);

CREATE TABLE source_data.stripe_refunds (
    id UUID PRIMARY KEY,
    charge_id UUID REFERENCES source_data.stripe_charges(id),
    refund_date DATE NOT NULL,
    amount NUMERIC(14, 2) NOT NULL,
    reason TEXT,
    payout_id UUID
);

CREATE TABLE source_data.stripe_payouts (
    id UUID PRIMARY KEY,
    payout_date DATE NOT NULL,         -- Date initiated by Stripe
    arrival_date DATE NOT NULL,        -- Date settled in Mercury Checking X0666
    gross_charges NUMERIC(14, 2) NOT NULL,
    fees_deducted NUMERIC(14, 2) NOT NULL,
    refunds_deducted NUMERIC(14, 2) NOT NULL,
    net_payout_amount NUMERIC(14, 2) NOT NULL, -- Positive for deposit, negative for expense pull
    status TEXT NOT NULL               -- 'paid', 'in_transit'
);

-- 4. Rippling Payroll (Written only by Rippling Payroll Agent)
CREATE TABLE source_data.rippling_journal_entries (
    id UUID PRIMARY KEY,
    entry_date DATE NOT NULL,
    type TEXT NOT NULL,        -- 'Payroll Journal Entry'
    num INTEGER NOT NULL,      -- e.g., 18, 19
    description TEXT NOT NULL,
    account TEXT NOT NULL,     -- Rippling's local export account label
    debit NUMERIC(14, 2),
    credit NUMERIC(14, 2)
);
```

---

## 6. How Real-World Friction Emerges Naturally from Isolated Entities

Because each entity runs independently on its own schedule, real-world discrepancies appear across tables without hardcoding any special edge-case rules:

| Phenomenon | Causal Chain Across Independent Entities |
| :--- | :--- |
| **1. In-Transit Month-End Payouts** | Customers pay invoices on March 29–31 (`Stripe Agent`) -> Stripe batches a payout on March 30 (`payout_date = 2021-03-30`, `arrival_date = 2021-04-01`) -> `Clearinghouse` holds the ACH until April 1 -> `Mercury Bank` March statement excludes it ([AccountingBench](https://accounting.penrose.com/)). |
| **2. Many-to-One Payout Batching** | Dozens of individual customer charges, processing fees, and refunds inside `source_data.stripe_*` are swept into a single net ACH transfer that appears on `source_data.mercury_statement_line_items` simply as `Deposit - STRIPE` ([AccountingBench](https://accounting.penrose.com/)). |
| **3. Enterprise Prepayments vs. Service Delivery** | Enterprise customer `Contoso` signs an annual contract (`2021-02-01` to `2021-11-30`) and wires cash upfront, creating a single cash event in Mercury for 10–12 months of ongoing service delivery ([AccountingBench](https://accounting.penrose.com/)). |
| **4. Executive & Employee Personal Commingling** | `Human Persona Agent` generates a weekend diary for an executive -> swipes Ramp at `APPLE STORE` and `EQUINOX`, and wires a personal car lease from `Mercury Checking X0666` alongside normal weekday business travel. |
| **5. Ramp Swipe-to-Settlement Cutoff Lag** | Employee swipes a Ramp card on March 30 (`swipe_date = 2021-03-30`), but `Ramp Agent` settles the charge on April 1 (`transaction_date = 2021-04-01`), pushing it onto the next month's Ramp statement. |

---

## 7. Execution Workflow & Configuration Parameters

### 7.1 Step-by-Step Generation Pipeline
1. **Initialize World & Entities (`init_simulation.py`):**
   * Instantiate the PostgreSQL `source_data` schema and internal clearinghouse queue.
   * Seed the starting equity wire (`$1,500,000` into `Mercury Checking X0666`), initial employee roster (5 core personas + scaling hires), vendor contracts, and initial customer cohort.
2. **Run Daily Simulation Loop (`run_world_clock.py`):**
   * At the start of each simulated week, invoke the **Human Persona Diary Agent** (`generate_persona_period_diary`) for each active staff member and schedule their narrative-driven card swipes, memos, and executive bank draws across the days of that week.
   * For each day `t`:
     1. Run `Customer Cohort Agent` renewals, signups, and refunds for day `t`.
     2. Run `Vendor Agent` service deliveries, card charges, and ACH pulls for day `t`.
     3. Run `Stripe Agent` invoice/charge captures and scheduled payout batches.
     4. Run `Ramp Agent` card authorizations and T+1/T+2 settlements.
     5. If `t` is the 10th or 26th, run `Rippling Agent` bi-monthly payroll export and ACH pulls.
     6. Run `Clearinghouse` to deliver matured ACH/wire transfers to `Mercury Bank Agent`.
     7. If `t` is month-end, run `Mercury Bank Agent` interest/fees and statement generation, plus `Ramp Agent` monthly statement closing.

### 7.2 Tunable Simulator Knobs (`config.yaml`)
* `customer_growth_rate`: Monthly net new self-serve and enterprise SaaS signups.
* `executive_commingling_frequency`: Probability per week that a founder uses a corporate card or bank account for a personal expense (`0.0` to `1.0`).
* `employee_memo_completeness`: Distribution of detailed vs. vague vs. blank Ramp memos (`e.g., 60% detailed, 25% terse, 15% blank`).
* `clearinghouse_ach_delay_days`: Min/max business days for ACH settlement (`1` to `3` days).
