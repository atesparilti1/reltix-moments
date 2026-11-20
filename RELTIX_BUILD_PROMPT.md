# Build prompt: "Moments" foundation, the service layer for centrix (reltix challenge)

> Paste everything below into ChatGPT / Codex / your OpenAI coding agent.
> This is the **foundation only**. When it works end to end, continue with `RELTIX_ADDONS_PROMPT.md`.

---

## ROLE

You are a senior full-stack engineer at a hackathon. Build fast, keep it simple, ship a demo that cannot break on stage. Write complete, runnable code: no placeholders, no "TODO: implement". Everything runs locally with one command per service. Isolate anything uncertain (SDK method names, external APIs) in one module and make it configurable. Design the code so features can be added later (an event bus, generic handlers) without rewrites.

## CONTEXT

**reltix** is a German AI-native property management company (Düsseldorf, Essen, Frankfurt; NRW, Hessen, Hamburg). It manages residential properties itself on its in-house platform **centrix**, and is building an **open platform that connects service businesses** (banking, energy, construction…) to real estate operations. This challenge is a prototype of that platform.

**What centrix knows:** properties and units, people, contracts and cases, communication. **There is no dataset or API, so we build our own.**

**What a winning solution does:** (1) detect the moment, (2) hand over context, (3) hold the conversation, (4) close the loop.
Channels: owner/tenant portal, phone, email or message, property manager. At least one must reach the resident.
**Deliverables:** **A.** a working demo (event → resident → partner → result) and **B.** a connector spec (the next partner plugs in the same way).

## THE CONCEPT

**"Moments"** turns life events in a building into services that run themselves: detected from centrix, pre-filled, agreed with the resident in one tap, delivered by a partner, and written back to the property file. **Every partner plugs in through one manifest file, with no per-partner code.**

## FOUNDATION SCOPE (what this prompt builds)

1. Mock centrix (data, API, event bus)
2. The connector spec: partner manifests, schema, hot reload, written spec and OpenAPI
3. Moment Engine: rule triggers, intent detection, emergency handling, deduplication
4. Conversation agent in a resident portal chat
5. Mock partner portal and contractor job page, with callbacks
6. Loop closer: property file, timeline, notifications
7. A three-panel demo frontend with a control bar
8. Consent and audit log, the "pre-filled vs asked" counter
9. A cached demo mode for an offline-safe demo

## PRINCIPLES

- **Consent first and data minimization (GDPR).** Explicit consent before any personal data reaches a partner. Partners get **only the fields declared in their manifest**. Every handover goes into an audit log.
- **Never ask what centrix already knows.** Show "9 fields pre-filled · 2 asked" on every flow.
- **One tap beats typing:** quick replies, slot pickers, approve/decline buttons.
- **No invented facts:** prices, dates and availability come only from manifests and partner responses.
- **A human in the loop:** jobs over a configurable cost threshold (default €500) need property-manager approval with one click.
- **Bilingual:** German and English, matching the resident's language.
- **Demo-proof:** `DEMO_MODE=cached` replays recorded LLM responses if the Wi-Fi or API fails.

## ARCHITECTURE

```
centrix mock (DB + event bus)
  │ events: contract.signed, contract.terminated, message.received, partner.status
  ▼
MOMENT ENGINE
  ├─ rule triggers (from manifests)
  └─ intent detector (LLM: intent, urgency, summary, language)
  ▼
OPPORTUNITY ──► CONVERSATION AGENT (OpenAI tools) ◄──► resident (portal chat), PM (approval)
  ▼
CONNECTOR (manifest-driven, generic) ──► partner (mock portal / webhook)
  ▲                                         │ callbacks (HMAC-signed)
  └─────────────────────────────────────────┘
  ▼
LOOP CLOSER → property file (case, contract, documents, timeline) + notifications
```

## TECH STACK

- **Backend:** Python 3.11, FastAPI, Pydantic, SQLModel + SQLite, `openai` SDK (tool calling, structured outputs), `simpleeval`, `watchfiles`, `jsonschema`
- **Frontend:** React + Vite + Tailwind + Framer Motion, WebSocket live updates
- **Config:** `.env` with `OPENAI_API_KEY`, `OPENAI_MODEL` (default: the latest GPT model available), `DEMO_MODE=live|cached`, `APPROVAL_THRESHOLD_EUR=500`

## 1. MOCK CENTRIX (`app/centrix/`)

A realistic German seed dataset (`seed.py`):
- **3 properties:** Lindenstraße 12 Düsseldorf (12 units, central gas boiler), Kaiserstraße 45 Frankfurt (10 units), Rüttenscheider Str. 88 Essen (8 units). Each has a building access note, a heating type and a boiler model.
- **Units:** floor, m², rooms, appliances, meter IDs, fiber availability.
- **People:** owners, tenants (language de or en, preferred channel, phone, email, consent flags, preferences such as "not before 9:00"), contractors and partner companies.
- **Contracts:** one rental **signed with move-in in 14 days**, one **terminated with move-out in 30 days**, and existing service contracts.
- **Cases and communication:** realistic history, plus a few open threads.
- **Documents:** past invoices stored in the property file.

**API:** `/centrix/properties|units|people|contracts|cases|messages|documents|timeline`, plus an event bus (an in-process async queue and `POST /centrix/events`).

## 2. CONNECTOR SPEC (deliverable B)

Each partner is one **manifest** (`partners/*.yaml`), validated against `spec/partner-manifest.schema.json`:

```yaml
id: fastnet-internet
name: FastNet Glasfaser
category: internet            # internet | energy | heating | plumbing | cleaning | moving | ...
type: sell                    # sell | deliver
coverage: { cities: [Düsseldorf, Essen, Köln] }
triggers:
  - event: contract.signed
    when: "days_until(contract.start_date) <= 21 and unit.has_fiber"
  - intent: internet_problem
context:                      # fields from the context dictionary; the engine pre-fills them
  required: [resident.full_name, resident.email, unit.address, unit.floor, contract.start_date]
  optional: [resident.phone, resident.language]
consent:
  required: true
  purpose: "Set up your internet connection"
ask:                          # only what the resident can answer
  - id: plan
    question: "Which speed do you want?"
    options: ["100 Mbit – €29.99", "500 Mbit – €39.99", "1 Gbit – €49.99"]
  - id: activation_date
    type: date
    default: "{{contract.start_date}}"
actions:
  submit: { method: POST, url: "{{PARTNER_BASE_URL}}/fastnet/orders" }
callbacks: { statuses: [received, accepted, scheduled, completed, cancelled, failed] }
result_mapping:
  on_completed:
    - create: contract
      fields: { provider: "{{partner.name}}", start: "{{result.activation_date}}" }
    - timeline: "Internet activated by {{partner.name}} on {{result.activation_date}}"
  notify:
    resident: "Your internet will be active on {{result.activation_date}} 🎉"
    property_manager: "Internet set up for {{resident.full_name}} ({{unit.label}})"
commercials: { model: referral, amount_eur: 80 }
sla: { accept_within_minutes: 60, complete_within_hours: 48 }
```

The messages in the manifest are written once in English. The agent translates them into the resident's language at runtime.

**Write these:**
- `spec/CONNECTOR_SPEC.md`: event types (a CloudEvents envelope), the **context field dictionary** (every centrix field with its sensitivity level: public, personal or sensitive, and whether it needs consent), the handover payload, the callback API, auth (HMAC-signed webhooks plus per-partner API keys), idempotency keys, retries with backoff, error states, versioning, and "Onboard a partner in 10 minutes".
- `spec/openapi.yaml`: OpenAPI 3 for the partner-facing API.
- **Hot reload:** the engine watches `partners/`. A new or changed manifest is validated and activated live, with a UI toast: "New partner connected: SparkleClean". Invalid manifests show a clear error and are not activated.

**Partners to ship:**
- `internet.yaml` (sell, triggered by contract.signed)
- `heating-repair.yaml` (deliver, triggered by heating_issue / hot_water_issue intents)
- `plumbing.yaml` (deliver, triggered by leak / plumbing intents)
- `cleaning.yaml` (sell, move-out cleaning, triggered by contract.terminated). **Store it in `partners_available/`, not `partners/`**, so the demo can plug it in live by moving it (a "Connect" button in the UI does this) and prove that the next partner plugs in with zero code.

## 3. MOMENT ENGINE (`app/engine/`)

- **Rule triggers:** evaluate manifest `when` expressions on events (`simpleeval` with helpers such as `days_until` and `age_years`).
- **Intent detector:** for every resident message, the LLM returns `{intent, urgency (emergency|high|normal|low), summary, language, entities}` using structured outputs. Matching manifests are found through their `intent` triggers.
- **Emergencies** (water leak, gas smell, no heating in freezing weather) skip the normal flow: they show emergency instructions and the emergency number immediately and alert the property manager.
- **Deduplication and preferences:** never offer the same thing to the same person twice. Respect opt-outs and contact-time preferences.

Output: `Opportunity {id, partner_id, trigger, people[], unit|building, prefilled, missing, value_eur, status}`.

## 4. CONVERSATION AGENT (`app/agent/`)

OpenAI tool calling. The system prompt is generated from the manifest plus the centrix context, so there's **no per-partner code**.
Tools: `get_context`, `ask(question, options)`, `record_answer`, `request_consent`, `propose_slots`, `submit_to_partner`, `request_approval(summary, amount)`, `escalate`, `notify`.
Rules: warm, short and efficient, in the resident's language. Confirm pre-filled data in one line, ask only for missing fields, use quick replies, get consent before sharing, never invent facts. The conversation state is stored, so it can resume later.

**Resident channel: portal chat.** A phone-frame UI with quick replies, a slot picker, notifications and a DE/EN switch.

**Contractor job page:** a one-tap, mobile-friendly job link with the full handover and buttons: accept, propose slot, done (with invoice amount).

## 5. MOCK PARTNERS (`app/partners_mock/`)

A small portal per partner (in the frontend) showing incoming orders and jobs with the pre-filled handover. Its buttons (Accept, Propose slot, Mark completed with an invoice amount, Cancel) call our callback webhooks exactly as described in the spec, as a real partner would. Mock partners are generated automatically for every active manifest.

## 6. LOOP CLOSER (`app/loop/`)

On each callback: update the opportunity status, apply `result_mapping` (create or update centrix contracts and cases, timeline entries, documents such as the invoice), and send notifications to the resident and property manager per the manifest. Jobs over the approval threshold wait for property-manager approval before booking.

## FRONTEND (the stage)

**Main demo screen, split into three panels plus a control bar:**
1. **Property manager / centrix:** a live event feed, an opportunities pipeline (detected → contacted → consented → sent → scheduled → done) with animated transitions, case detail with timeline and property file, the audit and consent log, approvals, the connected partners list, and "pre-filled vs asked" counters.
2. **Resident phone:** chat with quick replies, a slot picker and notifications.
3. **Partner / contractor:** the incoming handover with **pre-filled fields highlighted in green and asked fields in blue**, plus action buttons.

**Control bar:** Sign new contract · Tenant reports heater issue · Terminate contract · Connect cleaning partner · Reset demo.

**Design:** warm and clean, in reltix colors (deep brown or burgundy accent, cream background), big readable type for the projector, smooth transitions, and status pills.

## DEMO SCRIPT (about 2 minutes; write it to `scripts/demo_script.md`)

1. **Sign new contract:** the tenant gets a welcome message with the internet offer (9 pre-filled, 1 asked). One tap, consent, order sent. The partner accepts and completes, centrix updates, and the resident and property manager are notified. *(Detect → hand over → converse → close the loop: sell.)*
2. **Tenant reports heater issue:** the agent asks urgency, access and time slots. The contractor gets a pre-filled job and proposes a slot, the tenant confirms with one tap, and the job is marked done with an invoice that is attached to the property file. *(Deliver.)*
3. **Connect cleaning partner:** a new manifest is hot-loaded live. Click "Terminate contract" and the tenant immediately gets a move-out cleaning offer. **Zero code changed.** *(Deliverable B.)*

## API

- `POST /centrix/events`, `GET /centrix/...`
- `GET /opportunities[/{id}]`
- `POST /agent/inbound {person_id, channel, text}`
- `GET /partners`, `POST /partners/connect/{id}`, `POST /partners/{id}/callback` (HMAC)
- `GET /jobs/{token}` (contractor job page), `POST /approvals/{id}`
- `GET /audit`, `POST /demo/reset`
- WebSocket `/ws`

## PROJECT STRUCTURE

```
app/ main.py, models.py, db.py, events.py, llm.py (all OpenAI calls + cache + token logging)
     centrix/   engine/ (rules, intents)   agent/ (agent, tools, prompts/)
     connector/ (manifest, schema, handover, callbacks, hot_reload)
     loop/ (closer, notify)   partners_mock/   demo/
partners/            partners_available/ (cleaning.yaml)
spec/ (CONNECTOR_SPEC.md, partner-manifest.schema.json, openapi.yaml)
web/
scripts/ demo_script.md, demo_reset.py, record_cache.py
README.md (5-minute setup)
```

## BUILD ORDER

1. Seed data, centrix API, event bus, `llm.py` with caching
2. Manifest schema, loader, validator, hot reload, `internet.yaml`
3. Rule trigger → opportunity → agent (terminal) → handover → mock partner → callback → loop closer
4. Frontend with three panels, WebSocket, control bar → **move-in flow end to end**
5. Intent detection, emergencies, repair flow, slot booking, contractor job page, approvals
6. `cleaning.yaml` live connect and the termination flow
7. Consent/audit log, pre-filled counters, polish
8. CONNECTOR_SPEC.md and openapi.yaml
9. `record_cache.py`: run the demo once in live mode and save every LLM response, so `DEMO_MODE=cached` replays it offline

After each step, give the exact commands and what I should see.

## ACCEPTANCE CRITERIA

- The 3-step demo script runs end to end in about 2 minutes, in both `live` and `cached` mode, and `Reset` restores the starting state.
- Connecting `cleaning.yaml` while running adds a working flow with **no code changes**; an invalid manifest is rejected with a clear error.
- No partner ever receives an undeclared field, or any field before consent; everything appears in the audit log.
- The resident is never asked for data centrix already has.
- Jobs over the threshold wait for approval.
- Emergency messages show emergency instructions immediately.
- Works in German and English.
- The code is modular, so the add-ons (`RELTIX_ADDONS_PROMPT.md`) can be added without rewrites.

Start now with steps 1–4, then continue step by step.
