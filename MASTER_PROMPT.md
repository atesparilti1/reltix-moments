# MASTER PROMPT: centrix · Heating repair (reltix challenge)

> Paste this into the coding agent (Codex / Claude Code) **inside the `benthiew/reltix` repo**.
> It merges every team input into one build plan: the official reltix brief, `docs/IDEA.md`, the proposal comparison, Benedict's coordination vision, Jan Felix's repair coordination prompt, the Moments connector ideas, and the implementation guardrails.
> Where this prompt and the repo disagree, the precedence is: `docs/DECISIONS.md` → `docs/IDEA.md` → `AGENTS.md` → this prompt.

---

## 0. Before you write any code

1. Read `AGENTS.md`, `docs/IDEA.md`, `docs/PROGRESS.md`, `docs/DECISIONS.md` and `docs/brainstorm/TRACK.md`.
2. Read the existing repair slice: `src/data/repair.ts`, `src/lib/repair/types.ts`, `src/lib/repair/draft.ts`, `src/app/api/repair/route.ts`, `src/components/repair/RepairWorkspace.tsx`.
3. **Keep the stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4, OpenAI through `src/lib/openai.ts` (`ask`, `askJSON`), model IDs only in `src/lib/models.ts`, no database (seed data in `src/data/*.ts` plus the in-memory session that already exists in the repair route).
4. Read the relevant guide in `node_modules/next/dist/docs/` before using any Next.js API you are not sure about.
5. Work in vertical slices. After each slice: `npm run typecheck`, `npm run lint`, try it in the running app, update `docs/PROGRESS.md`, and commit.

---

## 1. The challenge (official brief, summarized)

**"Plug a Partner into centrix."** Pick one ecosystem partner and turn one moment in a property's life into a finished outcome **with zero forms**.

The solution must:
1. **Detect the moment:** when and for whom the service is relevant
2. **Hand over context:** pre-fill what the partner needs from centrix
3. **Hold the conversation:** as easy as possible for everyone, in a sensible channel
4. **Close the loop:** keep everyone updated, then link the result (appointment, invoice) back to the property

**Deliverables:**
- **A working demo of one flow:** a centrix event fires → the resident is reached → the partner receives the job and reports back → the result appears on the unit
- **A partner connector spec:** services or offers, eligibility rules, availability, how to place a job, status updates. The next partner must plug in the same way.

**Judging:** demo quality, end to end (30%) · user experience, minimal effort for resident, partner and property manager (25%) · feasibility and realism: data, consent, privacy, partner side (20%) · connector spec (15%) · partner choice (10%).

---

## 2. What we're building (team decision)

**One heating repair with one simulated, pre-onboarded contractor.** This is the brief's own "broken heater → craftsman" example: the right craftsman picked, a slot booked with the tenant, the repair done, and the invoice linked to the case.

**Cast (all fictional, labeled as demo data):**
- **Lena Hoffmann:** resident, Unit 4B, Musterstraße 12, 52062 Aachen. One radiator stays cold.
- **Mara Klein:** property manager at reltix. Approves scope and the price cap.
- **Maria Becker:** owner of Unit 4B. Informed of the approved job and the outcome.
- **Warmwerk Aachen:** a fictional heating contractor, pre-onboarded by reltix, covering Aachen and servicing this heating type.

**Product philosophy (from Benedict's vision): NO BS · SAVE TIME · SAVE MONEY.**
- Every unnecessary click, question or call is a design failure.
- Ask only for what centrix doesn't already know.
- Clearly separate **confirmed facts** (from the property record), **missing information** (still to be asked) and **AI-generated drafts**.
- Not another dashboard, not another chatbot: a system that gets the job done and only involves people when a decision is truly theirs.

**Why this partner and this moment (for the 10% partner-choice score; put it in the UI and pitch):**
- Heating failures are frequent, urgent and coordination-heavy: the brief notes a broken heater "can take a dozen calls and emails."
- centrix already holds what a contractor needs: address, unit, floor, heating type, equipment, service history and access details. That's maximum reuse of context.
- The outcome is concrete and verifiable: an appointment, a completed repair and an invoice linked to the case.

---

## 3. Roles, channel and the end-to-end flow

**Channel:** the resident portal (a web page; the brief says this is enough). The property manager and contractor each get their own view of **the same case**, switchable through a role switcher in the local demo.

**The flow (each step is a state in the case lifecycle):**

| # | Step | Who acts | What happens |
|---|------|----------|--------------|
| 1 | **Report** (already built) | Lena | One free-text message in the portal ("My radiator stays cold even with the dial all the way up"). It raises a `case.reported` event on her known unit. |
| 2 | **Understand** (already built) | AI + Lena | OpenAI extracts the symptom and room. If the room is missing, the portal asks **one** follow-up with quick-reply buttons. The brief shows confirmed facts from the record (heating type, equipment, last service: "Radiator valves serviced 12 Sep 2025"). |
| 3 | **Match** | System | The connector checks the partner manifests: Warmwerk covers 52062, services "central hot-water heating / panel radiators", has capacity. Show **why it qualifies** in one line. |
| 4 | **Consent** | Lena | Lena sees **exactly which fields** will be shared, with whom (Warmwerk Aachen) and why. She taps **Allow** or **Don't share**. Declining leaves the job unsent and tells Mara. |
| 5 | **Approve** | Mara | The property manager view shows the AI-drafted brief, the matched contractor with its reason, and a **suggested next step** ("Approve Warmwerk, routine radiator visit, cap €180"). Mara approves with one click. Higher scope or price needs a new approval. Maria (owner) gets an info update. |
| 6 | **Dispatch** | Connector | The server builds the job payload **only from the allowlisted fields**, after consent and approval, and submits it to the simulated contractor. The contractor view receives the job. |
| 7 | **Schedule** | Warmwerk → Lena | The contractor proposes available slots (fixed seed slots). Lena confirms one with one tap in the portal. All three views show the appointment. |
| 8 | **Complete** | Warmwerk | The contractor marks the job done, with a short result note ("Radiator vented, valve pin freed, heating normally") and the sample invoice (€148.75 = €125.00 + 19% VAT €23.75). The cost stays under the €180 cap. |
| 9 | **Close the loop** | System | The unit record links the case, appointment, outcome and invoice. Lena gets a completion update, and Mara and Maria get a short summary. The case is closed. |

**Effort targets (show them on screen, derived from recorded data, not hard-coded):**
- **Lena:** 1 message, at most 1 follow-up, 1 consent tap, 1 slot tap.
- **Mara:** 1 approval click.
- **Warmwerk:** receives a complete job, then 1 slot proposal and 1 completion.
- A **"Reused from centrix vs asked"** counter, for example "8 fields reused · 1 asked". It counts only values actually supplied to the flow and questions actually asked.

---

## 4. Case lifecycle (define once; use in the types, the API and the UI)

```
reported → needs_information → brief_ready → awaiting_consent → awaiting_approval
→ sent_to_partner → slot_offered → scheduled → completed → closed
side states: consent_declined, approval_declined, cancelled, failed
```

- Extend `RepairPhase` / `RepairState` in `src/lib/repair/types.ts`, and the command union in `src/app/api/repair/route.ts`, with new actions: `give-consent`, `decline-consent`, `approve`, `decline-approval`, `partner-offer-slots`, `confirm-slot`, `partner-complete`, `reset`.
- The server **validates every transition**. Invalid actions return 409 (the existing pattern). Keep the existing `revision` check, so duplicate or concurrent clicks are safe.
- Each case keeps an **append-only timeline** of every event: who acted, when, which fields were shared, and the decision made.

---

## 5. centrix stand-in data (`src/data/repair.ts`, extend it)

Base it on the brief's example payloads, and keep it internally consistent:
- **Property snapshot:** `unit_4b`, Musterstraße 12, 52062 Aachen, built 1972, gas central heating (central hot-water heating), energy certificate class E (valid until 2029-03-31), floor 2, 68 m², 3 rooms, electricity meter `1ESY1160123456`, access note ("Ring Hoffmann, 2nd floor left; caretaker key box code given on confirmed visits only").
- **Equipment:** panel radiators in each room; the gas boiler in the basement (model only as a plain label, no invented specs).
- **Service history:** 1 entry: radiator valves serviced 12 Sep 2025 by Warmwerk Aachen.
- **People:** owner Maria Becker (`c_101`); tenant Lena Hoffmann (`c_202`, lena@example.com, +49 151 00000000, moved in 2026-12-01; preferred language English, also reads German); property manager Mara Klein.
- **Lease:** €780 cold rent, from 2026-12-01.
- **Contractor profile (Warmwerk Aachen):** trade, service area postal codes 520xx, services, onboarded by reltix (date), rating, response SLA.
- **Availability:** 3 fixed slots in the next days.
- **Sample invoice:** €148.75 gross.
- Label everything in the UI as **demo data**.

---

## 6. The partner connector (deliverable B, built into the code)

Make the contractor integration **generic**, so the demo partner is just the first one plugged in.

**`src/lib/connector/`**
- `types.ts`: a `PartnerManifest` type:
  - `id`, `name`, `category` (`heating_repair` | `internet` | `energy` | …), `kind` (`deliver` | `sell`)
  - `services[]` (code, label, `price_cap_suggestion_eur` / price info)
  - `eligibility` (postal code prefixes, heating types or property attributes, events or intents handled)
  - `data_fields` allowlist: each field has its centrix source path, purpose and sensitivity (`basic` | `personal` | `access`); some fields are released only after scheduling (for example the access code)
  - `availability` (how slots are requested and returned)
  - `job_submission` (endpoint and payload shape)
  - `status_updates` (allowed statuses and the transitions they trigger)
  - `result_mapping` (how appointment, outcome and invoice map onto the unit record)
- `partners/warmwerk.ts`: the manifest for the demo partner.
- `connector.ts`: generic functions with no partner-specific code: `matchPartners(caseContext)` (returns partners plus a human-readable reason), `buildHandover(manifest, context, consent)` (serializes **only the allowlisted fields**, server-side, and refuses without a valid consent), `submitJob`, `applyStatusUpdate` (validates the transition), `mapResultToUnit`.
- **The simulated partner** lives behind the same interface: API routes such as `src/app/api/partner/[partnerId]/jobs/route.ts`, plus status callback routes. The contractor view calls these routes exactly as a real partner system would. In the UI, label it "Simulated partner (Warmwerk Aachen)".

**`docs/CONNECTOR_SPEC.md`** (the hand-in document):
1. Purpose and the flow diagram: event → match → consent → approval → job → status → result
2. What a partner must provide: services or offers, eligibility rules, coverage, availability, pricing or price caps, job or order submission, status updates, result documents
3. The manifest format (a JSON Schema), using the Warmwerk manifest as the example
4. Data sharing: the field dictionary with sensitivity levels, the purpose limit, consent records (use the brief's consent shape: `data_shared`, `given_at`, `channel`) and staged release (access details only after scheduling)
5. The API: the job request, status update and result payloads, with **example JSON that exactly matches what the demo exchanges**. Auth: per-partner API key plus HMAC-signed callbacks, idempotency keys, retries and error states.
6. The status lifecycle and which transitions each party may trigger
7. **"The next partner":** a second, filled-in manifest for **Example Fibre GmbH (internet at move-in)**, plus a short energy-contract example, to show the same spec works **without redesign**. These are spec examples only, not built flows.
8. Assumptions: what is simulated, what would be real in production, and what we didn't build

---

## 7. Where AI is used (and where it is not)

OpenAI is the main sponsor, so make the AI **visible**, but keep humans in control.

**AI does:**
- **Intake** (built): extracts the symptom and room from free text, using the property context; flags missing information.
- **Repair brief:** a concise contractor-ready brief combining the report with confirmed record facts, clearly separating "reported", "from the record" and "missing".
- **Property manager assistant:** suggests the next step and a scope/price cap with a one-line rationale. **Mara approves; the AI never sends anything.**
- **Plain-language updates:** short status messages for Lena, Mara and Maria at each lifecycle step.
- **Completion summary:** turns the contractor's result note and invoice into the unit record entry.

**AI never:** diagnoses a fault, shares data, approves spending, books, or advances a job. The server enforces consent, the allowlist, approvals and transitions.

**Mock mode (exists, keep it):** every AI step has a deterministic, labeled mock result, so the full demo runs without an API key or Wi-Fi. A live AI failure falls back visibly to mock and never blocks the flow.

**Emergencies:** gas smell, fire, smoke or CO are already refused by intake. Show a fixed, property-configured emergency message (from seed data, never generated by the LLM) and an escalation note to Mara.

---

## 8. UI (judges see the UI first)

Follow `AGENTS.md`'s quality bar, plus `$frontend-design` and `$ui-polish`.

- **The role switcher:** Resident (Lena) · Property manager (Mara) · Contractor (Warmwerk). An optional side-by-side "stage view" shows all three at once for the demo.
- **Resident portal:** a conversation-style case view with the report, the one follow-up, a consent card (the exact fields, the recipient, the purpose, Allow / Don't share), slot quick-replies, live status and the completion message.
- **Property manager view:** the AI brief with the "reported / record / missing" separation, the matched partner with its reason, the suggested next step with **Approve (cap €180) / Decline**, the unit record and timeline, and the consent and audit log.
- **Contractor view:** mobile-first and phone-friendly, built for tradespeople who dislike software. The job card shows only the shared fields; buttons: Offer slots, Mark completed (result note plus sample invoice).
- **The event trace panel** (on every view or in the stage view): trigger → match (reason) → consent (fields) → approval → job sent → slot → completion → unit record updated. This makes the platform behavior visible to judges.
- **Counters:** "8 fields reused · 1 asked", plus a per-role effort tally (Lena: 4 taps).
- **Guided "Run demo" mode** with pause/continue (manual controls stay available), a **Reset** button with confirmation, and a visible **"Demo data · simulated partner"** badge.
- Loading, empty and error states on every AI action. No dead links, no lorem ipsum, responsive down to phone width.

---

## 9. Guardrails (security, privacy, correctness)

- API keys only in `.env.local`, read only in `src/lib/*` and `src/app/api/*`. Never in client components, logs, fixtures or commits.
- **The allowlist is enforced server-side** at serialization time; filtering in the UI alone is not enough. Validate consent scope, recipient, purpose and timestamp before any handover, and reject missing or withdrawn consent.
- An append-only audit trail for consent, disclosures, approvals, partner requests, status updates and overrides. Store field **names** and decisions, not sensitive values.
- Validate partner status transitions and amounts (the invoice must stay within the approved cap, otherwise it goes back to approval). Duplicate updates are safe, and closed cases can't be reopened.
- Distinguish **contact consent** (we may message Lena in the portal) from **sharing consent** (we may share named fields with Warmwerk). Include the decline path and a withdraw-consent path.
- Reset restores the identical starting state (IDs, slots, timeline), with deterministic seed data.
- Never present simulated actions as real bookings or real spending.

---

## 10. Demo script (3 minutes; update `docs/DEMO_SCRIPT.md`)

| Time | Say | Do |
|------|-----|----|
| 0:00 | "A broken heater takes a dozen calls and emails. Lena reports it once." | The stage view. |
| 0:15 | One message. AI understands it and asks only which room. | Resident: send the report, tap "Bedroom". |
| 0:40 | "centrix already knows the heating type and last service, so nothing is re-asked." | Show the brief: reported / record / missing, and "8 reused · 1 asked". |
| 1:00 | "Lena sees exactly what's shared, and with whom." | Consent card → Allow. |
| 1:15 | "Mara gets one decision, with the AI's suggested next step." | Approve, cap €180. |
| 1:30 | "Warmwerk receives a complete job, no phone calls." | Contractor: job card → offer slots. |
| 1:50 | "Lena books in one tap." | Resident: confirm the slot. |
| 2:05 | "Done, and invoiced, inside the cap." | Contractor: complete with the invoice. |
| 2:20 | "The loop closes on the unit record." | Event trace and unit record. |
| 2:35 | "And the next partner plugs in the same way." | `CONNECTOR_SPEC.md`: the Warmwerk manifest next to the Example Fibre manifest. |
| 2:50 | Business: partners get demand without acquisition costs, residents get zero forms, and reltix saves manager hours. | A closing slide. |

**Backup:** mock mode, a recorded backup video, and a hotspot.

---

## 11. Build order (from the current state)

The intake slice (report → draft → room follow-up → review) is **done**. Continue:

1. **Seed data and lifecycle:** extend `src/data/repair.ts` and the types and state machine; add the timeline.
2. **Connector core:** `PartnerManifest`, the Warmwerk manifest, `matchPartners` with its reason, and `buildHandover` with server-side allowlist enforcement.
3. **Consent and approval:** the consent card (resident), the approval with cap and AI-suggested next step (property manager), and the decline paths.
4. **Simulated partner:** partner API routes, the contractor view (job card, offer slots), slot confirmation in the portal.
5. **Completion and loop closing:** the result note, the invoice (checked against the cap), the unit record links, notifications in all views.
6. **Event trace, counters, stage view, guided demo mode, reset.**
7. **`docs/CONNECTOR_SPEC.md`** (with the Example Fibre and energy manifests), aligned with the real payloads.
8. **Polish and rehearsal:** `$ui-polish`, `$web-design-guidelines`, `$ship-check`, the demo script, a backup video.

---

## 12. Acceptance checks

- The full flow runs end to end in under 3 minutes, in both mock and live mode, and Reset restores the identical start.
- The contractor receives a job **only after** consent **and** approval. Declining either leaves the job unsent, and the resident and property manager see why.
- The job payload contains only manifest-allowlisted fields. Access details are released only after scheduling.
- An invoice over the approved cap is blocked and sent back for approval.
- Duplicate clicks and repeated status updates don't corrupt the case.
- Emergency reports show the fixed emergency message and are never dispatched as routine.
- The counters are derived from recorded data.
- The example JSON in `CONNECTOR_SPEC.md` matches what the demo actually exchanges.
- Typecheck, lint and `npm run build` pass. `docs/PROGRESS.md` is up to date.

---

## 13. Stretch (only after section 12 passes, and only with team agreement, because `docs/IDEA.md` currently excludes these)

1. **Shared context across channels (a brief bonus):** an ElevenLabs voice agent (`src/components/VoiceAgent.tsx` already exists) where Lena reports by phone and the case continues in the portal, using the same case and timeline.
2. **Owner approval above the cap:** Maria approves larger jobs from her own view.
3. **Contractor rating and provider network** (from Jan Felix's proposal): Lena rates the visit; ratings feed the contractor profile and reltix's provider review.
4. **German / English toggle** for the resident portal.

**Out of scope:** real dispatch or messaging, payments, real provider onboarding, marketplaces, multi-property dashboards, free-form chat between parties, calendars, and deployment, unless the team decides otherwise in `docs/DECISIONS.md`.

---

**Start with step 1 of section 11. Before coding, write a short 3–6 step plan for the slice, then build it, verify it, update `docs/PROGRESS.md` and commit.**
