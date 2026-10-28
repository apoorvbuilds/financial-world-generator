# Target World: AccountingBench-Style SaaS Startup

**Reference benchmark:** [AccountingBench (Penrose)](https://accounting.penrose.com/)

## 1. Purpose: this document is the prompt for the world generator

This file describes, in as much detail as possible, the kind of world the generator in `SPEC.md` should be able to create. It is written to be handed to the authoring agents as the main prompt: from it they produce a manifest (entities, relationships, stories, noise, prompts, time range), which the simulator then runs into the `source_data` tables in section 5. It describes the world and the output, not an implementation. How the simulator works is in `SPEC.md`.

**What AccountingBench-style data is.** The raw exports a small company would have: bank statements, a corporate card, a payment processor, and payroll journal entries. The data is only useful if it has the friction real exports have, so the world must be built from the money's side, not the accounting side:

1. **Causal grounding.** Every row traces to something that happened in the world: a customer renewing, a vendor billing, a payroll run, a person on a trip. Nobody first invents balanced journal entries and then reverse-engineers clean rows.
2. **Isolated actors.** Each customer, vendor, person and institution knows only what it would know. Stripe does not know what is in Mercury. Mercury does not know which customer invoices make up a `Deposit - STRIPE`. Each institution's tables hold only what that institution would record.
3. **Time-lagged money.** Cash moves through ACH, wire, card and payout channels that take days to settle and skip weekends and holidays, so tables disagree on dates at every month end.
4. **Human arbitrariness.** Discretionary spending follows what real people did that week (a client trip, a team dinner, a personal charge by mistake), not uniform random numbers.
5. **No hardcoded friction.** Settlement lags, batching and cutoff differences must come out of how the channels work (section 6), not special-case rules.

## 2. The company and the time frame

- **Business:** a B2B SaaS startup, mixing a long tail of self-serve subscribers with a small number of enterprise accounts on annual contracts.
- **Time range:** 2021-01-01 to 2022-06-30 (18 months), currency USD.
- **Opening position:** a `$1,500,000` equity wire into `Mercury Checking X0666` on day one. That account is the operating account everything else settles into.
- **Headcount:** 5 core staff at the start (section 3.3), growing with a few hires; paid twice a month.
- **Systems the company uses (the only sources of data):**
  - **Mercury** bank with three accounts: `Mercury Checking X0666` (operating), `Mercury Checking X3354`, and `Mercury Treasury` (interest-bearing).
  - **Stripe** to collect card payments from self-serve customers and enterprise customers who pay by card.
  - **Ramp** corporate cards for staff and subscriptions, with a monthly statement paid by autopay from Mercury X0666.
  - **Rippling** for payroll, exporting journal entries on the 10th and 26th of each month.

## 3. The cast: entities to create

Create each as an entity with its own prompt. Give every entity an id, kind and a short prompt describing how it behaves; the behavior comes from the prompt, not from many fixed fields. Entities should be mostly shells.

### 3.1 Customers

- **Self-serve subscribers (hundreds over the period):** monthly plans at `$25`, `$37`, `$39`, `$80`, `$139` or `$270`, paid by card through Stripe. Each has a signup date, a renewal day, and a small chance of churn, refund or dispute. Create them as a cohort that grows over the period rather than as hundreds of hand-written prompts.
- **Enterprise buyers (a handful):** annual prepaid contracts paid by wire or ACH straight into Mercury, not Stripe. The anchor example is `Contoso`: `$85,363.20` per year, contract `2021-02-01` to `2021-11-30`, paid up front, with an expansion after the CEO's New York trip in March 2021. Enterprise buyers approve slowly and may pay late or short-pay.
- **Refunds and disputes:** a small share of charges are refunded or disputed days or weeks after payment, as full or partial refunds through Stripe.

### 3.2 Vendors

- **Cloud and compute:** `VERCEL INC - Monthly Pro Plan` (`$20.00`/month), `AMAZON WEB SERVICES` (variable usage), `RUNPOD.IO` (irregular GPU bursts).
- **Marketing and ads:** `Facebook Ads` (billed when a `$400.00` threshold is reached), `LinkedIn` (campaigns around `$281.46`), `GOOGLE ADS`.
- **SaaS tools:** `FIGMA`, `NOTION LABS`, `SLACK TECHNOLOGIES`, `LINEAR`, `GITHUB`, `GWORKSPACE`, mostly billed to a Ramp card monthly or annually.
- **Facilities, legal and operations:** a commercial landlord (`$7,000.00` rent per month by ACH or wire from Mercury), corporate counsel (quarterly retainer or Net-30 invoices), and `Amazon` (`AMZN MKTP US`) for equipment and supplies.
- Vendors bill one of three ways: a stored Ramp card, an authorized ACH debit from Mercury, or an invoice paid up to 30 days later.

### 3.3 People

Five core people, each with a role, a Ramp card (some with Mercury access) and habits. Their prompts carry the personality.

| Person | Role and access | Typical business activity | Typical personal or commingled activity |
| :--- | :--- | :--- | :--- |
| **Alex Rivera** | Co-founder and CEO (Ramp card, Mercury wire/ACH) | Investor pitches in SF/NYC, board dinners, executive SaaS tools, conference flights | Uses the company Ramp card for Equinox, Peloton, weekend family hotels and groceries (`WHOLE FOODS`); wires personal estimated taxes or a car lease from Mercury |
| **Sam Chen** | Co-founder and CTO (Ramp card, cloud admin) | Scales AWS/Vercel, buys test GPUs (`RUNPOD.IO`), replaces laptops for new hires | Late-night DoorDash during outages; sometimes buys home-automation gear on Amazon with the company card and notes "Personal - will reimburse" |
| **Jordan Taylor** | VP of Sales (Ramp card) | Takes enterprise prospects to steakhouses, books last-minute flights and Ubers, buys client renewal gifts | Extends a 3-day sales trip in Miami over the weekend and charges personal beach-club dining to Ramp without a receipt |
| **Casey Patel** | Senior Engineer (virtual Ramp card) | Renews GitHub/Linear seats, buys monitor cables, orders team lunch at sprint planning | Clean card use; sometimes forgets receipts or writes terse memos ("cables") |
| **Morgan Lee** | Ops and Office Manager (Ramp card, vendor billing) | Pays office rent, orders Costco snacks, books Friday happy hours, handles annual insurance prepayments | Accidentally uses a Ramp card stored in a personal Uber Eats profile on a Sunday and adds a memo flagging the mistake |

Memos on Ramp swipes range from clear ("Client dinner with Acme VP"), to terse, to blank, to personal admissions ("Accidentally used company card for personal Uber Eats, please deduct").

### 3.4 Institutions

These are entities too, and they own the tables in section 5 (each writes only its own).

- **Stripe:** invoices, card charges, refunds and payouts. Fee is `2.9% + $0.30` per charge. Charges accumulate and are swept into payout batches. If refunds exceed charges, Stripe pulls the difference by ACH from Mercury.
- **Ramp:** card transactions and monthly statements. Charges settle 1-3 days after the swipe. Statements close at a month-end cutoff and are paid by autopay from Mercury X0666.
- **Rippling:** payroll journal entries on the 10th and 26th, numbered in sequence (for example `JE #18` on `2021-03-10` and `JE #19` on `2021-03-26`), with matching ACH debits from Mercury for net pay and taxes.
- **Mercury:** settled bank lines for its three accounts, monthly statements, month-end interest (for example `Interest Accrual` of `$0.82` on checking, larger on Treasury) and service fees. It sees only cash that has settled.

## 4. Relationships and money flows

The relationships are the map of the world. Create these edges, with a prompt and terms where they apply:

| Relationship | Money flow and timing |
| :--- | :--- |
| Self-serve customer -> company | Monthly card charge through Stripe on the renewal day; Stripe fee deducted; charges swept into a payout batch; payout lands in Mercury X0666 1-2 business days after it is initiated as `Deposit - STRIPE` |
| Enterprise customer -> company | Annual prepay by wire or ACH directly into Mercury, once per contract, for 10-12 months of service |
| Refunding customer -> Stripe | Refund days or weeks after the charge; net against the next payout, or an ACH pull (`Stripe - Expense - STRIPE`, for example `-$35.39` or `-$282.70`) when refunds exceed charges |
| Company -> vendor (card) | Ramp swipe on the vendor's schedule; settles 1-3 days later; included in the monthly Ramp statement |
| Company -> vendor (ACH or invoice) | ACH debit from Mercury, or a Net-30 invoice paid when due |
| Company -> landlord | `$7,000.00` rent each month by ACH or wire |
| Employee -> company | Salary on the 10th and 26th through Rippling; ACH debits for net pay and taxes |
| Person -> Ramp card | Business and personal swipes with memos of varying quality |
| Executive -> Mercury | Occasional off-cycle wires or ACH, business or personal |
| Ramp -> Mercury | Monthly autopay pull after the statement closes |
| Company -> Mercury Treasury | Internal transfers between the three accounts; monthly interest |

Channels differ in speed: card charges settle in 1-3 days, ACH in 1-3 business days, wires usually same or next business day, and nothing posts on weekends or federal bank holidays unless the channel allows it.

### 4.1 Example story: CEO trip to New York, March 2021

Stories are the cause of discretionary spending. This one shows the shape; the generator should write similar ones for other people and months. Narrative:

> CEO Alex Rivera flew to New York (March 11-14) to close the Contoso enterprise expansion. Hosted Contoso executives at Mastro's Steakhouse on March 12. Stayed two extra personal nights at the hotel with a spouse (March 13-14), bought personal AirPods Max at the Fifth Ave Apple Store on Saturday March 13 with the company Ramp card, and on March 15 initiated an ACH payment from Mercury Checking X0666 to cover a personal vehicle lease.

Money movements it causes:

- Ramp, on Alex's card:
  - `2021-03-11` `DELTA AIR LINES` `$642.20`, memo "NYC flight for Contoso meeting"
  - `2021-03-12` `MASTRO'S STEAKHOUSE NYC` `$784.50`, memo "Dinner w/ Contoso VP Eng & CFO"
  - `2021-03-13` `APPLE STORE #R001 NYC` `$598.81`, memo blank (Saturday, consumer retail)
  - `2021-03-14` `THE STANDARD HIGH LINE` `$1,800.00`, memo "4 nights NYC - 2 nights Contoso biz, 2 nights personal weekend"
- Mercury X0666: `2021-03-16` `ACH Debit - PORSCHE FINANCIAL` `($1,150.00)`

**Default behavior:** on the first generation, create the world plainly: customers, vendors, people, relationships and routine flows. Personal charges, late or short payments, disputes and trips like the one above are one-offs that reviewers add through the per-entity prompts afterward; the generator does not add them on its own unless the prompt asks.

---

## 5. JSON `source_data` Schema Specification

Each financial institution writes strictly to its own tables ([AccountingBench](https://accounting.penrose.com/)). Output is JSON only: one file per table, `<institution>/<table>.json`, holding an array of row objects that validates against the schemas below (JSON Schema 2020-12). Conventions:

- **Money** is a string with exactly two decimal places (`"-642.20"`), never a JSON number, so it parses to `Decimal` without float error.
- **Dates** are ISO `YYYY-MM-DD`; **months** are `YYYY-MM`; **ids** are UUIDs.
- **Nullable columns** are written as `null`; columns not listed in `required` may also be omitted.
- **Foreign keys** cannot be expressed in JSON Schema, so they are marked with the custom keyword `x-references: "<table>.<column>"`.

**`common.schema.json`** (shared definitions)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "common.schema.json",
  "$defs": {
    "uuid": {"type": "string", "format": "uuid"},
    "date": {"type": "string", "format": "date"},
    "month": {"type": "string", "pattern": "^\\d{4}-(0[1-9]|1[0-2])$", "description": "e.g. 2021-03"},
    "money": {
      "type": "string",
      "pattern": "^-?\\d+\\.\\d{2}$",
      "description": "Decimal amount as a string with exactly 2 places; never a JSON number"
    }
  }
}
```

**`mercury.schema.json`**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "mercury.schema.json",
  "description": "Written only by the Mercury Bank agent. One <table>.json file per entry in properties.",
  "type": "object",
  "properties": {
    "mercury_transactions": {
      "title": "mercury_transactions",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {"$ref": "common.schema.json#/$defs/uuid"},
          "account_name": {"type": "string", "enum": ["Mercury Checking X0666", "Mercury Checking X3354", "Mercury Treasury"]},
          "transaction_date": {"$ref": "common.schema.json#/$defs/date"},
          "posted_date": {"$ref": "common.schema.json#/$defs/date"},
          "description": {"type": "string"},
          "amount": {
            "$ref": "common.schema.json#/$defs/money",
            "description": "Positive for deposits, negative for withdrawals"
          },
          "transaction_type": {
            "type": "string",
            "enum": ["ACH Credit", "ACH Debit", "Wire", "Internal Transfer", "Interest", "Fee"]
          },
          "counterparty_name": {"oneOf": [{"type": "string"}, {"type": "null"}]}
        },
        "required": ["id", "account_name", "transaction_date", "posted_date", "description", "amount", "transaction_type"],
        "additionalProperties": false
      }
    },
    "mercury_statement_line_items": {
      "title": "mercury_statement_line_items",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {"$ref": "common.schema.json#/$defs/uuid"},
          "statement_month": {"$ref": "common.schema.json#/$defs/month"},
          "account_name": {"type": "string"},
          "date": {"$ref": "common.schema.json#/$defs/date"},
          "description": {"type": "string"},
          "amount": {"$ref": "common.schema.json#/$defs/money"},
          "running_balance": {"$ref": "common.schema.json#/$defs/money"}
        },
        "required": ["id", "statement_month", "account_name", "date", "description", "amount", "running_balance"],
        "additionalProperties": false
      }
    },
    "mercury_treasury_records": {
      "title": "mercury_treasury_records",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {"$ref": "common.schema.json#/$defs/uuid"},
          "statement_month": {"$ref": "common.schema.json#/$defs/month"},
          "beginning_balance": {"$ref": "common.schema.json#/$defs/money"},
          "deposits": {"$ref": "common.schema.json#/$defs/money"},
          "withdrawals": {"$ref": "common.schema.json#/$defs/money"},
          "interest_earned": {"$ref": "common.schema.json#/$defs/money"},
          "ending_balance": {"$ref": "common.schema.json#/$defs/money"}
        },
        "required": [
          "id",
          "statement_month",
          "beginning_balance",
          "deposits",
          "withdrawals",
          "interest_earned",
          "ending_balance"
        ],
        "additionalProperties": false
      }
    }
  }
}
```

**`ramp.schema.json`**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "ramp.schema.json",
  "description": "Written only by the Ramp card agent. One <table>.json file per entry in properties.",
  "type": "object",
  "properties": {
    "ramp_card_transactions": {
      "title": "ramp_card_transactions",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {"$ref": "common.schema.json#/$defs/uuid"},
          "cardholder_name": {"type": "string"},
          "cardholder_role": {"type": "string"},
          "swipe_date": {"$ref": "common.schema.json#/$defs/date"},
          "transaction_date": {"$ref": "common.schema.json#/$defs/date", "description": "Settlement date on Ramp"},
          "merchant_name": {"type": "string"},
          "amount": {
            "$ref": "common.schema.json#/$defs/money",
            "description": "Positive for spend, negative for merchant refunds"
          },
          "ramp_category": {"oneOf": [{"type": "string"}, {"type": "null"}], "description": "Noisy platform category hint"},
          "memo": {
            "oneOf": [{"type": "string"}, {"type": "null"}],
            "description": "Human-entered memo (may be null, helpful, or personal admission)"
          },
          "statement_id": {
            "oneOf": [{"$ref": "common.schema.json#/$defs/uuid"}, {"type": "null"}],
            "x-references": "ramp_statements.id"
          }
        },
        "required": [
          "id",
          "cardholder_name",
          "cardholder_role",
          "swipe_date",
          "transaction_date",
          "merchant_name",
          "amount"
        ],
        "additionalProperties": false
      }
    },
    "ramp_statements": {
      "title": "ramp_statements",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {"$ref": "common.schema.json#/$defs/uuid"},
          "statement_month": {"$ref": "common.schema.json#/$defs/month"},
          "period_start": {"$ref": "common.schema.json#/$defs/date"},
          "period_end": {"$ref": "common.schema.json#/$defs/date"},
          "opening_balance": {"$ref": "common.schema.json#/$defs/money"},
          "new_charges": {"$ref": "common.schema.json#/$defs/money"},
          "payments_and_credits": {"$ref": "common.schema.json#/$defs/money"},
          "ending_balance": {"$ref": "common.schema.json#/$defs/money"},
          "autopay_date": {"oneOf": [{"$ref": "common.schema.json#/$defs/date"}, {"type": "null"}]}
        },
        "required": [
          "id",
          "statement_month",
          "period_start",
          "period_end",
          "opening_balance",
          "new_charges",
          "payments_and_credits",
          "ending_balance"
        ],
        "additionalProperties": false
      }
    }
  }
}
```

**`stripe.schema.json`**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "stripe.schema.json",
  "description": "Written only by the Stripe processor agent. One <table>.json file per entry in properties.",
  "type": "object",
  "properties": {
    "stripe_invoices": {
      "title": "stripe_invoices",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {"$ref": "common.schema.json#/$defs/uuid"},
          "invoice_number": {"type": "string"},
          "customer_email": {"type": "string"},
          "customer_name": {"oneOf": [{"type": "string"}, {"type": "null"}]},
          "invoice_date": {"$ref": "common.schema.json#/$defs/date"},
          "service_period_start": {"$ref": "common.schema.json#/$defs/date"},
          "service_period_end": {"$ref": "common.schema.json#/$defs/date"},
          "amount_due": {"$ref": "common.schema.json#/$defs/money"},
          "status": {"type": "string", "enum": ["paid", "open", "void"]}
        },
        "required": [
          "id",
          "invoice_number",
          "customer_email",
          "invoice_date",
          "service_period_start",
          "service_period_end",
          "amount_due",
          "status"
        ],
        "additionalProperties": false
      }
    },
    "stripe_charges": {
      "title": "stripe_charges",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {"$ref": "common.schema.json#/$defs/uuid"},
          "invoice_id": {
            "oneOf": [{"$ref": "common.schema.json#/$defs/uuid"}, {"type": "null"}],
            "x-references": "stripe_invoices.id"
          },
          "customer_email": {"type": "string"},
          "charge_date": {"$ref": "common.schema.json#/$defs/date"},
          "amount": {"$ref": "common.schema.json#/$defs/money"},
          "fee_amount": {"$ref": "common.schema.json#/$defs/money"},
          "net_amount": {"$ref": "common.schema.json#/$defs/money"},
          "payout_id": {
            "oneOf": [{"$ref": "common.schema.json#/$defs/uuid"}, {"type": "null"}],
            "x-references": "stripe_payouts.id",
            "description": "Set once swept into a payout batch"
          }
        },
        "required": ["id", "customer_email", "charge_date", "amount", "fee_amount", "net_amount"],
        "additionalProperties": false
      }
    },
    "stripe_refunds": {
      "title": "stripe_refunds",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {"$ref": "common.schema.json#/$defs/uuid"},
          "charge_id": {
            "oneOf": [{"$ref": "common.schema.json#/$defs/uuid"}, {"type": "null"}],
            "x-references": "stripe_charges.id"
          },
          "refund_date": {"$ref": "common.schema.json#/$defs/date"},
          "amount": {"$ref": "common.schema.json#/$defs/money"},
          "reason": {"oneOf": [{"type": "string"}, {"type": "null"}]},
          "payout_id": {
            "oneOf": [{"$ref": "common.schema.json#/$defs/uuid"}, {"type": "null"}],
            "x-references": "stripe_payouts.id"
          }
        },
        "required": ["id", "refund_date", "amount"],
        "additionalProperties": false
      }
    },
    "stripe_payouts": {
      "title": "stripe_payouts",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {"$ref": "common.schema.json#/$defs/uuid"},
          "payout_date": {"$ref": "common.schema.json#/$defs/date", "description": "Date initiated by Stripe"},
          "arrival_date": {"$ref": "common.schema.json#/$defs/date", "description": "Date settled in Mercury Checking X0666"},
          "gross_charges": {"$ref": "common.schema.json#/$defs/money"},
          "fees_deducted": {"$ref": "common.schema.json#/$defs/money"},
          "refunds_deducted": {"$ref": "common.schema.json#/$defs/money"},
          "net_payout_amount": {
            "$ref": "common.schema.json#/$defs/money",
            "description": "Positive for deposit, negative for expense pull"
          },
          "status": {"type": "string", "enum": ["paid", "in_transit"]}
        },
        "required": [
          "id",
          "payout_date",
          "arrival_date",
          "gross_charges",
          "fees_deducted",
          "refunds_deducted",
          "net_payout_amount",
          "status"
        ],
        "additionalProperties": false
      }
    }
  }
}
```

**`rippling.schema.json`**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "rippling.schema.json",
  "description": "Written only by the Rippling payroll agent. One <table>.json file per entry in properties.",
  "type": "object",
  "properties": {
    "rippling_journal_entries": {
      "title": "rippling_journal_entries",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {"$ref": "common.schema.json#/$defs/uuid"},
          "entry_date": {"$ref": "common.schema.json#/$defs/date"},
          "type": {"const": "Payroll Journal Entry"},
          "num": {"type": "integer", "description": "Journal number, e.g. 18, 19"},
          "description": {"type": "string"},
          "account": {"type": "string", "description": "Rippling's local export account label"},
          "debit": {"oneOf": [{"$ref": "common.schema.json#/$defs/money"}, {"type": "null"}]},
          "credit": {"oneOf": [{"$ref": "common.schema.json#/$defs/money"}, {"type": "null"}]}
        },
        "required": ["id", "entry_date", "type", "num", "description", "account"],
        "additionalProperties": false
      }
    }
  }
}
```

---

## 6. How real-world friction should emerge

Because each entity acts on its own schedule through channels with real delays, discrepancies appear across tables without any special-case rules. A correct world shows all of the following:

| Phenomenon | Causal chain across independent entities |
| :--- | :--- |
| **1. In-transit month-end payouts** | Customers pay on March 29-31 -> Stripe batches a payout on March 30 (`payout_date = 2021-03-30`, `arrival_date = 2021-04-01`) -> the ACH is held until April 1 -> the Mercury March statement excludes it |
| **2. Many-to-one payout batching** | Dozens of customer charges, processing fees and refunds in `stripe_*` are swept into a single net transfer that appears in `mercury_statement_line_items` simply as `Deposit - STRIPE` |
| **3. Enterprise prepayments vs. service delivery** | Contoso signs an annual contract (`2021-02-01` to `2021-11-30`) and wires cash up front, one cash event in Mercury for 10-12 months of ongoing service |
| **4. Executive and employee personal commingling** | A weekend trip produces swipes at `APPLE STORE` and `EQUINOX` on Ramp and a personal car lease wired from Mercury X0666, mixed in with normal weekday business travel |
| **5. Ramp swipe-to-settlement cutoff lag** | A card swipe on March 30 (`swipe_date = 2021-03-30`) settles on April 1 (`transaction_date = 2021-04-01`), so it lands on the next month's Ramp statement |

Further messiness the generator should allow through noise prompts: inconsistent vendor descriptors across months, duplicate or late-posted rows, blank or misleading memos, and customers who pay late, short-pay or send a corrective wire.

---

## 7. What the generator should produce from this prompt

1. **World:** name, the time range in section 2, currency, and a global `escalation_p` that is small.
2. **Entities:** the cast in section 3, as shells with prompts. Use a cohort for the self-serve tail and individual entities for enterprise customers, key vendors, the five people and the four institutions.
3. **Relationships:** the edges in section 4, with terms where they are fixed (Contoso's amount and dates, rent, plan prices, payroll dates) and prompts where they are loose (when a customer pays, how an executive behaves).
4. **Stories:** a small number of time-ordered chains such as the New York trip, kept out by default per section 4.1 and available as examples to reviewers.
5. **Noise:** a baseline of coded perturbations (descriptor variants, blank memos) at modest rates; prompt-based one-offs only when asked.
