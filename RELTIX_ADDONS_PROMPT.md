# Add-ons prompt: "Moments" edges (reltix challenge)

> Use this **only after the foundation (`RELTIX_BUILD_PROMPT.md`) works end to end.**
> Paste one add-on at a time into the same coding agent chat, in priority order. Each add-on must plug into the existing event bus, manifests, agent and loop closer **without rewriting the foundation**, and must also work in `DEMO_MODE=cached` (re-run `record_cache.py` after each one).

---

## Shared instructions (paste these first)

You are continuing the "Moments" project. The foundation is working: mock centrix, manifest-driven connector with hot reload, Moment Engine (rules and intents), conversation agent, mock partners, loop closer, three-panel frontend and cached demo mode. Add the following feature cleanly as a new module. Keep all existing acceptance criteria passing. Write complete, runnable code with no placeholders. After finishing, give the exact commands and what I should see, and update `scripts/demo_script.md`.

---

## PRIORITY 1: AI connector builder (the showstopper)

In the property manager UI, "**+ Add partner**" lets you upload a partner's PDF (price list, service description, API docs) or paste a URL or text.
1. The LLM extracts: name, category, type, coverage, services and prices (which become `ask` options), required data (mapped to the context field dictionary), triggers (it suggests events from the service type, e.g. cleaning → contract.terminated), SLA, and commission if mentioned.
2. It generates a manifest, validates it against the schema (and fixes it automatically, up to 2 attempts), then shows a **review screen**: highlighted fields, any data flagged as sensitive, and the suggested triggers in plain language ("Offer to tenants 30 days before move-out").
3. The property manager clicks **Activate**. The manifest is saved to `partners/`, hot-reloaded and live.
4. A mock partner portal and callback endpoint are created for it automatically.

Ship `demo_assets/sparkleclean_pricelist.pdf`: a realistic German move-out cleaning company price list (Endreinigung, prices per m², window cleaning add-on, service area NRW).
**Demo change:** replace "Connect cleaning partner" with uploading this PDF live; about 60 seconds later the partner is live, and the termination flow then works with real prices.
API: `POST /partners/draft` (file, URL or text → manifest draft), `POST /partners/activate`.

## PRIORITY 2: Building-level pattern detection

If ≥ 3 units in one building report the same issue type within 48 hours, create **one building opportunity**, link the individual cases to it, book **one** contractor, and broadcast an update to **all** tenants in the building, each in their own language.
Seed: 3 messages from different Lindenstraße 12 tenants in the last 36 hours about cold radiators / no hot water (one in Turkish).
Control bar: "3 tenants report cold radiators".
Acceptance: exactly one building case, one contractor job, and a broadcast to every tenant.

## PRIORITY 3: German cost allocation and WEG owner approval

- Make Lindenstraße 12 a WEG (owners' association), with owners and a **Miteigentumsanteil** (ownership share) per unit.
- On job completion, classify each cost as `owner` (repair, Instandsetzung), `tenant_service_charge` (umlagefähige Betriebskosten such as the annual boiler service, per BetrKV categories) or `weg_shared` (common property, split by Miteigentumsanteile). Use an LLM classification plus a manifest default (`cost_allocation.default`), with a property-manager override.
- **Owner approval card:** "Repair €640, WEG shared. Your share: €53.33. Approve / Decline", in an owner view.
- Acceptance: the shares add up exactly to the total.

## PRIORITY 4: Self-healing flows and partner router

- Add `heating-repair-backup.yaml` (a second heating partner).
- **Router:** when several partners match, rank them by SLA compliance, average rating, price, coverage and current load, plus warranty holder (a warranty repair goes to the original installer). Show the explanation: "FixIt Heizung chosen: 4.8★, 94% on-time, warranty holder."
- **SLA watchdog:** if a partner doesn't accept within `sla.accept_within_minutes`, re-route to the next-best partner and tell the resident. Add a "Fast-forward 2 h" control.
- **No-show / cancellation:** propose new slots automatically.
- **Warranty repeat failure:** seed an earlier boiler repair from 4 months ago. If it fails again within the warranty window, flag the job and send it to the original contractor at no cost, and inform the owner of the savings.

## PRIORITY 5: Moment calendar (predictive moments)

A daily job lists upcoming moments for the next 60 days: move-ins and outs, contract ends, boiler service due (last service over 11 months ago, before the heating season), appliances past their expected life, warranties expiring soon. Each carries `expected_value_eur` (commission or avoided cost). Show it as a timeline in the property manager view, with "Start now" buttons. Add `boiler-service.yaml` (deliver, predictive; cost allocation `tenant_service_charge`).

## PRIORITY 6: Impact dashboard and pitch numbers

- Dashboard tab: zero-touch rate, average time to resolution vs a baseline, fields pre-filled vs asked, calls avoided by pattern detection, referral revenue, property-manager hours saved, and AI cost per conversation (from the token logs in `llm.py`).
- A projection for 1,000 / 10,000 units from `business.yaml` (configurable assumptions).
- `docs/pitch_notes.md`: who pays (partner referral or lead fees; property-manager time and owner savings), why it makes reltix the operating system and marketplace for building services, unit economics per 1,000 units, risks (GDPR, liability, partner quality) and how we handle them, and a rollout plan.

## PRIORITY 7: Autonomy dial

In settings, for each action type (offer, share data, book, spend up to €X, notify owners) and each partner, choose `auto`, `approve` or `manual`. A badge on every timeline step shows whether it ran automatically or was approved by a person. Manifests get `autonomy_default`.

## PRIORITY 8: Phone agent (ElevenLabs)

An ElevenLabs conversational agent with a German voice. It recognizes the caller from centrix by phone number and uses the same agent tools through server-tool webhooks (`POST /voice/tools/{tool}`). The conversation state is shared with chat, so a call can continue a chat conversation. Behind a feature flag (`VOICE_ENABLED`). Demo: "Hallo, meine Heizung ist kalt" → creates the case and books a slot. Control bar: "Simulate phone call".

## PRIORITY 9: More languages

Add Turkish, Arabic, Polish and Ukrainian residents to the seed data (common in NRW). The agent and all notifications reply in the resident's language (right-to-left layout for Arabic in the chat).

## PRIORITY 10: Conformance test kit

`python -m moments.conformance --partner partners/x.yaml --url https://partner/api` validates the manifest and simulates a full order lifecycle against the partner's endpoint (submit, signed callbacks, idempotency, error states), printing a pass/fail report. It must pass for all mock partners.

## STRETCH

- **Email channel via n8n:** outbound emails and inbound replies → `POST /agent/inbound`; render emails in the UI. Plus an n8n workflow acting as an external partner that auto-accepts after 5 seconds.
- **MCP server (`moments-mcp`):** tools such as `list_opportunities`, `get_handover`, `update_status` and `attach_document`, scoped by partner API key and limited to manifest-declared fields, so a partner's own AI agent can plug into centrix through the same standard.
- **Live tracking** for residents: scheduled → technician en route → done.

---

## Full demo script once the add-ons are in ("A day at Lindenstraße 12", 3 minutes)

1. **Sign new contract:** Ayşe gets a welcome message in Turkish with the internet offer (9 pre-filled, 1 asked). One tap, consent, order sent, centrix updated.
2. **3 tenants report cold radiators:** one building case. The router picks the warranty holder, owner approval shows the WEG split, and all 12 tenants get the update.
3. **The contractor doesn't accept:** Fast-forward, and the job is re-routed to the backup partner automatically.
4. **Job done:** the invoice is attached, the cost is allocated (owner / service charge / WEG shares), and the dashboard updates.
5. **Showstopper:** upload SparkleClean's PDF. About 60 seconds later the partner is live. Terminate a contract and the move-out cleaning offer arrives with real prices. Zero code.
6. **Close** on the impact dashboard and the 10,000-unit projection.
