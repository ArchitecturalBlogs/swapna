---
layout: post
title: "Validate AI-Generated Database Schemas — Structural, Domain, and Hallucination Checks in One Tool"
date: 2026-04-14 09:00:00 +0000
author: Swapna
categories: [ai, database, python, open-source]
tags: [python, ai, database, llm, data-modeling, open-source, flask, ollama, schema-validation, hallucination]
excerpt: "I asked Claude to design a 15-entity real estate data model. It looked right. Then I built a validator to check if it actually was — and found three production bugs hiding in plain sight."
image: /assets/images/ai-schema-validator-banner.png
github: https://github.com/swapna15/ai-data-model-validator-agent
reading_time: 14
---

You ask an AI assistant to design a data model for your real estate platform. It comes back in thirty seconds with a clean, well-formatted 15-entity schema — Company, Agent, Property, Lease Agreement, Sale Agreement, Transaction, Customer, Lead, Inspection, Support Ticket, Document, Payment. Every entity has a primary key. Foreign keys reference the right tables. The relationships look reasonable.

It *looks* right.

But is it actually right?

Does `Property.price` store the price in USD? Euros? Bitcoin? There is no `currency` field. If you ever expand to another market, every financial query in your codebase is broken by design.

Is `Lease_Agreement.property_id` really necessary, or is the property already reachable through `transaction_id → Transaction.property_id`? Is that a deliberate performance shortcut or an accidental data model smell the AI introduced without flagging it?

`Support_Ticket.related_to_id` ends in `_id`. Is it a foreign key? It is not marked `is_fk=true`. There is no declared reference target. Your ORM will not enforce referential integrity, and you will not find out until production.

These are not theoretical problems. They are the exact issues I found after building a data model with Claude — a data model I initially thought was production-ready.

This post is the story of what I built to catch them, why I built it the way I did, and what I learned along the way. The tool validates any AI-generated schema — not just real estate — across four independent layers: structural, relational, domain knowledge, and hallucination scoring. It runs entirely on your machine, costs nothing after setup, and completes in under a second.

The code is open source: [github.com/swapna15/ai-data-model-validator-agent](https://github.com/swapna15/ai-data-model-validator-agent)

---

## The origin: a real estate data model

It started as a specific problem. I was designing a real estate platform and asked Claude to generate a comprehensive data model — one that would give a 360° view of everything happening in a property deal. Not just listings. The full lifecycle: customer acquisition, lead management, viewings, offers, negotiations, contract signing, payments, inspections, and post-sale support.

Claude produced a 15-entity schema spanning four domains:

**Core platform** — `Company`, `Agent`, `Property`, `Property_Media`. The foundational layer. A company employs agents, agents manage properties, properties have media galleries and feature lists.

**Customers and leads** — `Customer`, `Lead`, `Appointment`. The CRM layer. A customer generates a lead against a specific property with a specific agent. A lead schedules appointments — viewings, meetings, signing events. Customer types cover the full spectrum: buyer, seller, landlord, tenant, investor.

**Transactions** — `Transaction`, `Lease_Agreement`, `Sale_Agreement`, `Payment`. The financial layer. A `Transaction` is the parent record for every deal. It spawns either a `Lease_Agreement` or a `Sale_Agreement`. Both agreement entities track the full signing workflow — draft, pending review, signed, active or closed. `Payment` covers every monetary movement: rent cycles, security deposits, earnest money, agent commissions.

**Operations** — `Inspection`, `Support_Ticket`, `Document`. The support layer. `Inspection` records pre-sale and routine property checks. `Support_Ticket` links polymorphically to any entity. `Document` is a polymorphic store: any entity in the system can attach documents via an `entity_type` + `entity_id` pair.

The design has several genuinely good qualities. `Transaction` as the central hub means you can reconstruct any deal's full history with a single join chain. The polymorphic document store avoids a separate documents table for every entity type. The `Customer.type` enum covering five roles means one table serves the entire CRM.

But it also had problems. And because the schema looked polished, those problems were invisible to the eye.

That is the core issue with AI-generated schemas: they are optimised to look right, not to be right.

---

## Three kinds of problems in AI schemas

After studying the generated schema carefully, I found the issues fell cleanly into three categories.

**Structural errors** — things that are factually wrong. Missing primary keys. Nullable primary keys. Foreign key fields (`_id` suffix) with no declared reference target. Fields using a data type that cannot store the data they represent — a price stored as a string, or a timestamp stored as an integer. These are objective failures. There is no interpretation involved.

**Domain gaps** — things that are missing by industry standards. A real estate platform without `address_line1` on `Property` cannot do geo-coding or comply with MLS requirements. An agent without `license_state` has an ambiguous license number — the same number means different things across US states. A payment record without `currency` will break the moment you process a transaction in a non-default currency. These are not bugs in the traditional sense, but they cause expensive rework six months into a project.

**Hallucinations** — things that are wrong in a subtler sense. A field placed on the wrong entity. A relationship with the wrong cardinality. A field name that implies one thing but has a type that implies another. An enum containing values that do not exist in industry practice. These require contextual reasoning, not just rule matching.

The manual review process that found these issues took hours and still missed things. I needed something faster, repeatable, and thorough.

---

## Architecture: four validation layers

The validator runs four independent layers against every schema. Each is a pure function — it takes a `Schema` object and returns a list of typed issues. They share nothing at runtime, which means each can be tested in isolation, run in any order, and streamed to the UI independently.

### The schema representation

Everything flows through a lightweight dataclass:

```python
@dataclass
class Schema:
    domain:        str
    description:   str
    entities:      List[EntityDef]
    relationships: List[RelationshipDef]

@dataclass
class FieldDef:
    name:       str        # "address_line1"
    data_type:  str        # "decimal", "enum(a,b,c)", "uuid"
    is_pk:      bool
    is_fk:      bool
    nullable:   bool
    references: Optional[str]  # "OtherEntity.field"
```

It was deliberately kept thin. No validation logic lives inside these dataclasses — validators consume them, the types themselves do not validate. This means the parser can load partially-malformed schemas without refusing to continue, which is important because a schema with a broken field is exactly the kind of schema you most want to validate.

### Layer 1 — Entity validator

The entity validator runs deterministic structural checks on every entity and every field. Sub-millisecond, no network calls, no external dependencies.

Checks include: does every entity have exactly one primary key? Is the primary key non-nullable? Does every entity have a `created_at` audit field? Do all field names follow snake_case convention? Are all data types known? Do foreign key fields end with the `_id` suffix convention? Do boolean fields start with `is_`, `has_`, or `can_`? Are there duplicate field names?

These sound basic. They are. But in practice, every AI-generated schema I have tested fails at least three of them on the first run.

### Layer 2 — Relationship validator

The relationship validator resolves every foreign key reference against the actual entity map — if `Agent.company_id` references `Company.id`, it verifies that a `Company` entity exists and has a field called `id`. It checks that every declared cardinality is one of `1:1`, `1:N`, or `M:N`. It flags `1:1` relationships with no declared FK. It detects orphan entities — entities with no relationships at all — which almost always means the schema is incomplete.

### Layer 3 — Domain validator

This is where the project evolved most significantly. The first version hardcoded roughly 350 lines of Python rules specific to real estate. Adding a new domain would have required editing the source file.

The second version replaced that with a rule pack system. Eight domain packs are now built in:

| Domain | Entities covered |
|---|---|
| Real Estate | Property, Agent, Lease, Sale, Payment, Inspection, Document |
| E-Commerce | Product, Order, OrderItem, User, Payment, Category |
| Hospital Management | Patient, Doctor, Appointment, Prescription, Invoice |
| CRM | Contact, Deal, Activity |
| SaaS | Organisation, User, Subscription |
| Fintech | Account, Transaction, User |
| Logistics | Shipment, Warehouse, InventoryItem |
| HR | Employee, LeaveRequest |

When the validator runs, it reads the schema's `domain` field and fuzzy-matches it against the available packs. Unknown domains still get universal rules — `updated_at` on every entity, soft-delete flag considerations, enum nullability warnings.

Running the validator against the original real estate schema finds **35 warnings** and **52 info-level gaps**. The warnings include: `Property` missing `address_line1`, `latitude`, `longitude`, `mls_number`, and `currency`. `Agent` missing `license_state` and `is_active`. `Payment` missing `currency` and `invoice_no`. `Inspection` missing `inspector_license_no`. `Document` missing `mime_type`.

Every one of these would have caused a real production problem.

### Layer 4 — Hallucination scorer

The hallucination scorer runs ten independent checks and produces a score from 0 to 100 and a letter grade from A+ to F.

**Type rules.** Forty pattern-based rules map field name patterns to expected types. Fields matching `_(at)$` must be `timestamp`. Fields matching `price|cost|amount|fee|deposit|rent` must be `decimal` or `float`. Fields ending in `_id` must be `uuid`, `int`, or `bigint` — never `decimal` or `bool`. This catches the most common hallucination type: a financial field stored as a string, which passes JSON serialisation but breaks every arithmetic query.

**Enum validator.** Scans enum values against a dictionary of known-bad values. A `status` field containing `"quantum"` or `"undefined"`. A `role` field containing `"wizard"` or `"rockstar"`. A `currency` field containing `"tokens"` or `"karma"` instead of ISO 4217 codes. These are real values I have seen AI assistants generate.

**Gibberish detector.** Flags placeholder and test field names: `foo`, `bar`, `test123`, `quantum_score`, `blockchain_hash`. These appear when an AI is filling in a schema it does not fully understand.

**Context mismatch rules.** Known-wrong (field, entity) combinations. `monthly_rent` on `Property` — rent is a lease term, not a property attribute. `commission_rate` on `Property` — commissions belong on `Transaction` or `Agent`. `unit_price` on `Order` — unit price belongs on `OrderItem`.

**Denormalisation smell.** Flags string fields that store names which should be FK references. `inspector_name` as a string on `Inspection` — this should be an `inspector_id` FK to an `Inspector` entity. Every denormalised name field is a consistency bug waiting to happen.

**Currency orphan.** Any entity with a monetary field (`price`, `amount`, `fee`, `deposit`, `rent`) but no companion `currency` field gets flagged. This caught ten findings in the real estate schema — across Property, Lead, Transaction, Lease_Agreement, Sale_Agreement, and Payment.

**Unmarked FK detector.** Any field ending in `_id` that is not flagged `is_fk=true` gets flagged. `Support_Ticket.related_to_id` and `Document.entity_id` both triggered this.

**Polymorphic validator.** Detects the `entity_type` + `entity_id` pattern and checks that `entity_id` is not mistakenly marked `is_fk=true`. Polymorphic references cannot have real DB-level FK constraints.

**Redundant FK detector.** Flags foreign keys reachable via an existing path. `Lease_Agreement.property_id` is already reachable via `transaction_id → Transaction.property_id`. May be intentional for performance, but should be explicitly documented.

**Impossible relationship rules.** `Order → Product` as `1:1` — an order contains many products. `Patient → Doctor` as `1:1` — a patient sees multiple doctors. `Permission → User` as `1:N` — typically M:N via a join table.

Running all ten checks against the real estate schema produces **14 findings** and a score of **25/100, grade B**. The score is not meant to shame — it is a concrete target. The path from B to A+ is exactly the list of findings in the report.

---

## The journey: what changed during development

Building this project in the open meant building it messily. Here are the turning points worth knowing about.

**From real estate-specific to generic.** The first version had the real estate schema hardcoded as Python dataclasses. Refactoring to a JSON-first generic architecture meant rebuilding the schema representation, the parser, all four validators, and the CLI. The payoff was a tool that works for any domain without any code changes.

**The $20 credit bill.** The original domain validator and hallucination scorer both used the Anthropic API — `claude-sonnet-4-6` with `max_tokens=4096` and no turn limit on the agentic loop. This burned through $20 of API credits in about a day. The fix had four parts: switch to `claude-haiku-4-5` (roughly 15× cheaper), cap `max_tokens` at 1024, hard cap the loop at 6 turns, and keep only the last turn in message history. Cost per run dropped from approximately $0.50 to approximately $0.01.

The deeper lesson: agentic loops that resend growing message histories every turn have quadratic token cost. The sixth turn sends six times the tokens of the first. If the loop is unbounded, the cost is unbounded.

**Going fully local.** The final architecture replaced both API-based validators with local engines: a pattern-based hallucination scorer (ten checks, instant, zero cost) and a rule pack domain validator (eight packs, also instant, also free). An optional Ollama layer adds LLM semantic reasoning on top — local model, no API key, no cost.

**The `showView()` bug.** The UI used `element.style.display = ''` to "remove" the inline style and let CSS take over. The problem is that the CSS had `display: none` on the panels — clearing the inline style does not show the element, it just lets `display: none` snap back. The fix is to always set an explicit value: `'block'` or `'flex'` when showing, `'none'` when hiding.

**The Python 3.11 f-string error.** One line used escaped quotes (`\"`) inside an f-string expression block `{}`. Python 3.12 relaxed this restriction. Python 3.11 enforces it strictly. The code worked fine in my Python 3.12 container and failed immediately on the user's 3.11 machine. The fix was to extract the ternary into a helper function before the f-string.

**The `KeyError: 'severity'`.** Haiku sometimes omits fields not marked `required` in a tool schema, even when Sonnet reliably includes them. Using `inp["severity"]` instead of `inp.get("severity", "warning")` crashed the validator. The fix is to use `.get()` with a sensible default for every field on every tool call response.

---

## Cost transparency: from $20 to $0

The cost evolution drove every architectural decision in the second half of the project.

| Implementation | Cost per run | Notes |
|---|---|---|
| Sonnet + unbounded loop | ~$0.50 | Original — burned $20 in one day |
| Haiku + guardrails | ~$0.01 | 4 fixes applied |
| Local pattern engine | $0.00 | Pure Python, no network |
| Local + Ollama | $0.00 | One-time model download |

The cost table by layer in the final version:

| Layer | Cost per run | Requires |
|---|---|---|
| Entity validator | $0.00 | Nothing |
| Relationship validator | $0.00 | Nothing |
| Domain validator (local) | $0.00 | Nothing |
| Hallucination scorer (local) | $0.00 | Nothing |
| Hallucination scorer (Ollama) | $0.00 | Ollama + model pulled |
| Hallucination scorer (Anthropic API) | ~$0.01 | API key + credits |

For most developers, the local-only setup is sufficient. The Ollama layer adds semantic analysis — catching field names that are contextually wrong, missing enum values, relationships that contradict business logic — without any ongoing cost.

---

## The web UI: live streaming results

The interface is a split panel. The left side is a JSON schema editor with example schemas you can load in one click — Real Estate, E-Commerce, Hospital Management — or paste your own. The right side streams validation results as they arrive.

Progress appears in real time using Server-Sent Events. Each validator emits two events: a `progress` event when it starts and a `step_done` event when it finishes with the full issue list attached. The browser renders each step as it lands.

Results land across five tabs: **Summary** gives the full issue count by category and grade. **Hallucination** shows the findings table with confidence percentages. **Entity**, **Relationship**, and **Domain** each show their issues with severity badges (ERR / WRN / INFO) and full messages.

Getting SSE streaming right in Flask required solving three separate problems:

1. Werkzeug buffers response chunks by default — yield a `: ping\n\n` comment frame at the start of the generator to force header flushing
2. A stray `new EventSource(...)` line was firing a GET request to a POST-only endpoint, returning 405 on every run
3. Flask's development reloader forks the process and interferes with streaming — `use_reloader=False` resolves this

Each of these bugs produced the same symptom: the right panel showed nothing after clicking Validate.

---

## Using it

**Web UI:**

```bash
git clone https://github.com/swapna15/ai-data-model-validator-agent
cd ai-data-model-validator-agent
pip install -r requirements.txt
python app.py
# open http://localhost:5000
```

Click an example to load a schema, click Validate Schema, watch results stream in.

**Paste your own schema** in this format:

```json
{
  "domain": "Your Domain Name",
  "entities": [
    {
      "name": "YourEntity",
      "domain": "subdomain",
      "description": "What this entity represents",
      "fields": [
        {"name": "id",         "data_type": "uuid",      "is_pk": true,  "nullable": false},
        {"name": "name",       "data_type": "string",    "nullable": false},
        {"name": "created_at", "data_type": "timestamp", "nullable": false}
      ]
    }
  ],
  "relationships": [
    {
      "from_entity": "ParentEntity",
      "to_entity":   "ChildEntity",
      "cardinality": "1:N",
      "label":       "has",
      "from_fk":     "ChildEntity.parent_id"
    }
  ]
}
```

**CLI:**

```bash
python main.py --schema examples/real_estate.json
python main.py --schema examples/ecommerce.json
python main.py --schema my_schema.json
```

**Add Ollama for semantic analysis:**

```bash
brew install ollama           # macOS
ollama pull mistral           # 4.1 GB — recommended sweet spot
ollama serve                  # keep this terminal open
```

Once running, the UI badge switches to `🟢 Ollama: mistral:latest` and every validation automatically adds the LLM layer on top of the pattern engine.

---

## What comes next

**More domain packs.** The eight built-in packs cover common domains, but the pack format is declarative JSON — adding a new industry requires no Python. A legal case management pack, a gaming platform pack, an IoT telemetry pack would all be valuable contributions.

**Multi-format input.** The current format is custom JSON. Real projects use SQL DDL, Prisma schema files, OpenAPI specs, and Django model definitions. A parser layer that accepts any of these would dramatically expand the tool's reach.

**The eval harness.** Each test case provides a schema and an expected set of findings. The harness computes precision, recall, and F1 per category. This is the right way to measure whether a rule change improves detection quality without introducing false positives.

**Consensus mode.** When multiple LLM backends are available, running both in parallel and keeping only findings they agree on cuts false positives sharply.

**A fix engine.** The validator tells you what is wrong. A fixer would take the same findings and produce a corrected schema, closing the validate → fix → re-validate loop.

---

## Conclusion

AI assistants generate database schemas confidently. They produce clean formatting, consistent naming, and plausible relationships. But confidence is not correctness.

`Property.price` without `currency` is a multi-market bug. `Lease_Agreement.property_id` being reachable via the transaction chain is a redundant denormalisation. `Support_Ticket.related_to_id` being unmarked as a FK is a missing constraint. None of these cause an immediate error in development. All of them surface as expensive production problems.

The four-layer architecture exists because no single technique catches everything. The structural validator catches objective errors. The domain validator catches omissions wrong for your specific industry. The hallucination scorer catches fields and relationships wrong in context. The Ollama layer adds semantic reasoning without API cost, network dependency, or wait time.

The schema you validated this week is not the same as the schema you will ship. But knowing where it stands — a grade, a score, a list of specific findings with concrete suggestions — is the difference between an AI-generated schema that ships and one that gets caught before it does.

---

*Full source code, documentation, and example schemas at [github.com/swapna15/ai-data-model-validator-agent](https://github.com/swapna15/ai-data-model-validator-agent). Contributions, domain pack additions, and issue reports are welcome.*
