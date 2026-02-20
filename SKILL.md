---
name: mixpanel-first-implementation
description: Guides a coding agent through helping a new Mixpanel customer implement analytics correctly from day one. Covers discovery, analytics strategy, project setup, data model, tracking plan design, SDK implementation, identity management, and data governance. Use when a user wants to implement Mixpanel, set up Mixpanel, add Mixpanel tracking, configure a new Mixpanel project, or is a new Mixpanel customer starting their first implementation.
---

# Mixpanel First Implementation

Default to all 8 phases **in order**. Each phase gates the next — rushing past discovery leads to wasted implementation and data that is expensive to fix. Use Focused Remediation or Implementation Audit modes only when the customer explicitly asks for a narrow scope. Ask questions conversationally (1–2 at a time), acknowledge answers, then proceed.

Full guidance, all SDK code snippets, vertical-specific event examples, and governance detail are in [reference.md](reference.md). Read specific sections on demand as you work through each phase.

---

## Execution Modes

Pick one mode before Phase 0 and state it explicitly:

| Mode | Use when | Required guardrails |
|---|---|---|
| **Full Greenfield Rollout** | New implementation or major rebuild | Run all phases 0-7 in order |
| **Focused Remediation** | One urgent issue (identity bug, missing events, naming drift) | Run only relevant phase(s), but still enforce all upstream prerequisites that impact correctness |
| **Implementation Audit** | Existing setup needs quality/risk review | Diagnose current state, produce prioritized fixes, then execute fixes via Focused or Full mode |

**Mode switching rules:**
- If you discover missing prerequisites (for example, no signed-off tracking plan), pause and backfill the required earlier phase before proceeding.
- If risk is high (identity merge, consent, or production governance), escalate to Full mode even if the customer started in Focused mode.
- At the end of each mode, summarize what was completed, what remains, and which phase gate is next.

---

## Compliance and Privacy Guardrails

This skill is implementation guidance, not legal advice. Use customer policy and counsel as source of truth when there is conflict.

| Scenario | Default behavior |
|---|---|
| Region includes EU/EEA/UK/CH or CA users | Treat consent as required before non-essential tracking; apply consent gate pattern before SDK initialization |
| Region is unknown | Ask once; if still unknown, use conservative consent-gated behavior until clarified |
| Server-side geolocation enrichment | Only forward IP when customer policy permits; if restricted, omit IP and document reduced geo resolution |
| Identity/profile enrichment | Track minimum required attributes only; avoid sensitive categories unless explicitly approved in policy |

**Fail-safe:** if consent status is unknown in a regulated context, delay tracking initialization and collect clarification first.

---

## Pre-Flight — Codebase Scan

**Run this before Phase 0 if you have access to the codebase. Do not ask the customer anything yet.**

Read the codebase silently and build a working picture to carry into all downstream phases. This replaces most discovery questions and produces a grounded draft tracking plan before the first conversation turn.

| What to read | What to extract |
|---|---|
| Route/page files, controllers, API endpoints | Candidate events — every meaningful user-initiated action (`POST /projects`, `PUT /subscriptions/upgrade`, checkout handler, etc.) |
| Database models or schema files | Candidate properties and their types; User Profile fields; Group entity fields if B2B |
| Auth / session files (login, signup, logout handlers) | Where to place `.identify()`, `.people.set()`, and `.reset()` in Phase 6; whether anonymous browsing exists |
| Existing analytics, logging, or third-party tracking calls (GA4, Amplitude, Segment, `console.log`) | First-draft event names; naming inconsistencies to fix; properties already being collected |
| Package files (`package.json`, `requirements.txt`, `build.gradle`, `Package.swift`, `pubspec.yaml`) | Exact tech stack and framework → SDK selection for Phase 5; confirms platform |
| Environment config files (`.env`, `config/`, `settings.py`) | Where tokens should be injected; whether a dev/prod split already exists |

**After scanning, carry forward:**
- Confirmed tech stack (eliminates the platform question in Phase 0)
- A draft list of candidate events with proposed snake_case names
- Candidate properties sourced from model fields and existing logging
- The exact files and line locations where Mixpanel initialization and tracking calls will be written
- The auth file locations and login/logout/re-open patterns (for Phase 6)

Present assumptions to the customer rather than asking from scratch. Only ask what the codebase cannot answer.

---

## Context Block

After each phase, update a structured context block in your working notes. Reference it at the start of each phase rather than relying on conversational memory.

- **Company name:**
- **Business model:** (SaaS subscription / usage-based / transactional / freemium / marketplace / ad-supported)
- **Growth model:** (product-led / sales-led / marketing-led)
- **Customer type:** (B2B / B2C / B2B2C — if B2B: who is the buyer vs. the user?)
- **Stage:** (pre-PMF / growth / scale)
- **Commercial priority:** (acquisition / activation / monetization / retention / expansion)
- **Product type:**
- **Platform(s):**
- **CDP in use:** (Segment / Rudderstack / mParticle / Snowflake / BigQuery / none)
- **Group Analytics:** yes / no
- **EU or CA users:** yes / no
- **Value Moment:**
- **KPIs (2–3):**
- **Dev project token:**
- **Prod project token:**
- **Tracking method:** server-side / client-side / CDP integration
- **Event 1:** `sign_up_completed` — properties: [list]
- **Event 2:** [Value Moment event name] — properties: [list]

Update this block at the end of every phase. Never start a phase without referencing it first.

---

## Phase 0 — Discovery

**If Pre-Flight was run:** Skip the platform and product type questions — these are already confirmed from the codebase scan. Lead with your assumptions summary and ask only the three remaining questions (CDP, Group Analytics, business questions). Do not ask what the codebase already answered.

**Step 1 — Collect company name and URL (always, before anything else).**

Ask:
> "Before we dive in — what's your company name, and do you have a website or product URL I can look at?"

Then run deep research using both the URL and company name. Do not ask the customer anything else until the research is complete.

**Step 2 — Deep research protocol.**

Research across all available sources. The goal is to build a business model picture and understand what the company is commercially trying to drive — not just what the product does.

**Time-box and stop conditions (required):**
- Time-box initial research to 10 minutes or 6 meaningful sources, whichever comes first.
- Stop early once you can confidently fill business model, growth model, customer type, stage, commercial priority, and candidate Value Moment.
- If sources are sparse, contradictory, private, or pre-launch: stop external research and switch to a short clarification set with the customer.

**Low-signal fallback (ask only these):**
1. "How do you make money today, and how do you expect that to evolve in the next 6-12 months?"
2. "Who is the buyer vs. daily user?"
3. "What user action most strongly predicts retention or expansion?"
4. "What compliance or consent constraints should we respect before any tracking starts?"

| Source | What to extract |
|---|---|
| Marketing site (homepage, product pages, pricing) | Core value proposition, target customer (B2B vs B2C, industry, company size), pricing model (subscription, usage-based, freemium, transactional), platform (web, iOS, Android) |
| Pricing page specifically | Plan tiers → infer Mixpanel plan eligibility; free vs paid conversion funnel structure; whether Group Analytics is plausible |
| About / Team / Careers pages | Company stage, team size, open roles (reveal growth priorities and tech stack), founding story |
| Blog / Changelog / Product announcements | Recent feature launches → candidate events; what the team is investing in; what they care about measuring |
| App Store / Play Store listings and reviews | User language for the value moment; what users love and what they complain about; platform confirmation |
| G2, Capterra, ProductHunt, Trustpilot | Third-party user language for the value moment and pain points; competitive context |
| Crunchbase / LinkedIn / TechCrunch | Funding stage and amount → informs growth focus (acquisition vs activation vs retention); investor-implied growth model; team size trajectory |
| Job listings (LinkedIn, Greenhouse, Lever, their careers page) | Tech stack clues (engineering job descriptions list languages and frameworks); data/analytics maturity (do they have a data team?); growth-stage priorities |
| Source code hints (`<script>` tags, JS bundle names, framework meta tags) | Existing analytics tools (GA4, Amplitude, Segment); tech stack confirmation; whether Mixpanel is already partially implemented |

**Synthesize into a business model summary before asking any questions.** Carry this forward into all downstream phases:

- **Business model:** How they make money (SaaS subscription / usage-based / marketplace take rate / transactional / ad-supported / freemium-to-paid)
- **Growth model:** How they acquire and retain users (product-led growth / sales-led / marketing-led / community-led)
- **Customer type:** B2B (who is the buyer vs the user?) / B2C / B2B2C
- **Stage:** Pre-PMF / growth / scale (inferred from funding, team size, product maturity)
- **Commercial priority:** What the company is trying to drive right now — acquisition, activation, monetization, retention, or expansion
- **Candidate Value Moment:** The specific action that signals a user got value, inferred from product descriptions, user reviews, and pricing structure
- **Candidate KPIs:** What a company at this stage and in this vertical would logically measure

**Step 3 — Present assumptions, then ask only what research couldn't answer.**

Lead with the business model summary rather than raw product facts:

> "Based on what I found, here's how I'm thinking about your business: you're a B2B SaaS project management tool targeting mid-market engineering teams, subscription-based, currently in growth stage with Series B funding. Your likely Value Moment is a project reaching a milestone or a report being generated. Does that framing sound right?"

Then ask only the questions research couldn't answer:

| Remaining question | Decision It Drives |
|---|---|
| Do you use a CDP or data warehouse? (Segment, Rudderstack, mParticle, Snowflake, BigQuery) | If yes → skip SDK installation; route through integration in Phase 5 |
| Do you have the Group Analytics add-on? (if not inferable from pricing page) | If yes → surface Group Analytics in Phase 3 |
| What are the 2–3 most important business questions you want Mixpanel to answer? | Drives event selection, KPI design, and tracking plan in Phases 1–4 |

**If neither a codebase nor a URL is available**, ask all five questions conversationally:

| Question | Decision It Drives |
|---|---|
| What type of product? (SaaS, e-commerce, media, fintech, mobile game, marketplace, internal tool) | Vertical-specific event examples in Phase 4 |
| What platform(s)? (web, iOS, Android, React Native, Flutter, server-side only, combo) | SDK selection in Phase 5 |
| Do you use a CDP or data warehouse? (Segment, Rudderstack, mParticle, Snowflake, BigQuery) | If yes → skip SDK installation; route through integration in Phase 5 |
| Do you have the Group Analytics add-on? (availability varies by plan) | If yes → surface Group Analytics in Phase 3 |
| What are the 2–3 most important business questions you want Mixpanel to answer? | Drives event selection, KPI design, and tracking plan in Phases 1–4 |

Store all confirmed answers in the Context Block. They gate which content you surface in later phases.

**Output of this phase:** Business model summary (how they make money, growth model, customer type, stage, commercial priority) confirmed with customer. Product type, platform(s), CDP status, Group Analytics flag, and top 2–3 business questions captured in Context Block. Required before Phase 1.

---

## Phase 1 — Analytics Strategy

**Before presenting any framework, ask:**
1. "What does success look like in the next 90 days — acquisition, activation, engagement, or retention?"
2. "What is the single most important action a user can take that signals they're getting real value?"

**Then:**
- If the customer already has defined KPIs and a named value metric: skip the RAE framework introduction. Validate their existing KPIs against the 5M filter and confirm or refine their Value Moment name. Proceed once you have a confirmed Value Moment and 2–3 KPIs.
- For customers new to product analytics: select the **RAE Framework** (Reach / Activation / Engagement) → see `reference.md § Phase 1` for full framework
- Name the customer's **Value Moment** explicitly: `[Core Action] at [Natural Frequency]`
  - e.g. "Your Value Moment is `report_generated` — weekly. This is one of the first two events we'll track."
- Help them apply the **5M filter** (Meaningful, Measurable, Manageable, Movable, Time-bound) to candidate KPIs
- Warn against vanity metrics (downloads, page views) and lagging-only metrics (churn, revenue)

**Output of this phase:** Named Value Moment + 2–3 KPIs. Required before Phase 4.

---

## Phase 2 — Mixpanel Project Setup

**Ask:**
1. "Have you already created a Mixpanel account and project, or starting fresh?"
2. "Do you have a separate dev/staging environment?"
3. "Do you have users in the EU or California?" — If yes, flag for a consent gate in Phase 5 before any initialization code is written.

**Steps in order:**

**A. Verify Simplified ID Merge** (non-negotiable first step)
- Project Settings → Identity Management → confirm "Simplified API"
- If it shows "Original API" and no data has been tracked yet: switch it before proceeding
- If data has already been tracked under Original API: do not switch without reading the migration guide

**B. Determine project structure** (before creating anything)

| Scenario | Recommendation |
|---|---|
| Web + mobile, same product, same users | Single project |
| Completely separate products / user bases | Separate projects |
| Same product, different feature sets per platform | Single project + `platform` super property |

This determines how many projects to create in the next step.

**C. Create dev and production projects** (always at minimum two)
- Name clearly: `[Product] - Production` and `[Product] - Development`
- Set timezone to match primary business location (cannot change retroactively without affecting historical data)
- If step B determined multiple production projects are needed, create each with the same naming pattern
- Use environment-based config to switch tokens automatically → see `reference.md § Phase 2` for JS example

**D. Collect project tokens — ask the customer to provide them now**

Once dev and production projects exist, ask:
> "Can you copy the project token for each project? You'll find them at mixpanel.com → your project → Project Settings → Project Token. Paste both here and I'll inject them directly into the initialization code — no manual search-and-replace needed."

| What to collect | Where the customer finds it |
|---|---|
| Production project token | mixpanel.com → Production project → Settings → Project Token |
| Dev/staging project token | mixpanel.com → Development project → Settings → Project Token |

**Store both tokens in the Context Block.** They are injected verbatim into every initialization code snippet produced in Phase 5. Do not use `'YOUR_PROJECT_TOKEN'` placeholders — if the tokens are in hand, use them.

If the customer cannot provide tokens yet (e.g., someone else owns the Mixpanel account): proceed with placeholder values and flag that tokens must be substituted before any events are sent.

**E.** Assign minimum-necessary roles: Owner, Admin, Analyst, Consumer.

**Output of this phase:** Simplified ID Merge verified, project structure decided, dev and production projects created, both tokens stored in the Context Block, EU/CA flag noted, roles assigned. Required before Phase 3.

---

## Phase 3 — Data Model

**Ask:** "Have you worked with an event-based analytics tool before, or is this your first time?"
- Yes → brief orientation
- No → full walkthrough from `reference.md § Phase 3`

**Core concepts to convey:**

| Concept | Key fact |
|---|---|
| **Events** | Immutable, timestamped actions. Required fields: event name, distinct_id, timestamp. |
| **Event Properties** | Point-in-time; never change after ingestion. Send numerics without quotes or they become strings. |
| **User Profiles** | Mutable, current state. Join retroactively to events via distinct_id. Only create for identified users. |
| **Super Properties** | Auto-attached to every event. Use for: `app_version`, `platform`, `plan_type`, `experiment_group`. |

**Property type reminder:** If you send `price = "29.99"` (quoted), Mixpanel treats it as String — cannot aggregate. Always send numeric values unquoted.

**User Profiles join to the latest state.** If you need "plan at time of event," track `plan_type` as an event property too.

**If Group Analytics confirmed in Phase 0:** Surface Group Analytics section from `reference.md § Phase 3 — Group Analytics`. Key call: `mixpanel.set_group("company_id", "acme-corp")` + set Group Profiles.

**Group Analytics late-discovery:** If the customer indicates during this phase that they need account-level analysis (e.g., "we sell to companies and need to see usage by account") and Group Analytics was not confirmed in Phase 0: ask directly whether they have the Group Analytics add-on. If yes, surface the Group Analytics section now before moving to Phase 4, and update the Group Analytics flag in the Context Block.

**CDP late-discovery:** If the customer mentions they use a CDP (Segment, Rudderstack, mParticle) at any point after Phase 0: note it in the Context Block. When you reach Phase 5, route through the integration path rather than SDK installation. The tracking plan design in Phase 4 remains valid — only the implementation method changes.

**Output of this phase:** Customer aligned on the Mixpanel data model (events, properties, profiles, super properties). Group Analytics scope confirmed. Required before Phase 4.

---

## Phase 4 — Tracking Plan

**If a codebase scan was run (Pre-Flight):** Do not start with open-ended questions. Instead, present the draft event list derived from routes, controllers, and models. Show proposed snake_case names, candidate properties sourced from model fields, and flag any naming inconsistencies found in existing logging code. Then ask the customer to:
1. Confirm which events actually matter for their KPIs (priority, not exhaustiveness)
2. Fill in business intent the code can't reveal ("what does a user completing this action tell you?")
3. Validate or correct property values and enumerations not visible in the schema

**If no codebase is available**, ask:
1. "Do you have existing screen flows, wireframes, or user journey maps? Sharing them helps translate them into events."
2. "What are the top 3 user actions where you'd say they're getting real value?"

Design the full tracking plan using the 7-step sequence below, then implement in two-event increments — ship `sign_up_completed` and the Value Moment first, validate they are working, and add remaining events from the signed-off plan progressively.

**Start with exactly two events:**
- **Event 1:** `sign_up_completed` (or equivalent) — with properties: `sign_up_method`, `referral_source`, `platform`
- **Event 2:** The Value Moment named in Phase 1

**Tracking plan sequence (do not skip steps):**
```
1. Define KPIs          → Phase 1 output
2. Map KPIs to flows    → user journeys that drive each KPI
3. Flows → events       → discrete actions within each journey
4. Events → properties  → context needed to analyze each action
5. Identify globals     → properties on almost every event → super properties
6. Identify profiles    → attributes describing the user → user properties
7. Document             → write into tracking plan template before writing any code
```

**Naming — enforce from day one (Mixpanel is case-sensitive):**
- Event names: `object_verb` in `snake_case` → `checkout_completed`, `video_played`, `report_generated`
- Property names: `snake_case`, descriptive, no abbreviations → `payment_method`, `plan_type`
- Property values: lowercase strings, consistent → `"free"` not `"Free"` or `"FREE"`
- Never use `$` or `mp_` prefixes on custom properties

**Granularity test:** One event + its properties should answer your business question without needing a dozen filter conditions.

**Dynamic names:** Never construct event or property names at runtime — creates thousands of unique names and can quickly exhaust practical event-name limits (verify current limits in docs/account plan).

**Null values:** Omit properties that don't apply; never send `null`, `""`, or `"N/A"`.

**Tracking plan templates by vertical** (from `reference.md § Phase 4`):
- SaaS, E-Commerce, Media/Content, Fintech, Blank

**Vertical-specific event examples** are in `reference.md § Phase 4 — Vertical-Specific Event Examples`.

**The tracking plan must be reviewed and signed off by product, engineering, and analytics before implementation begins.**

**Output of this phase:** Signed-off tracking plan with at minimum `sign_up_completed` and the Value Moment fully specified (name, trigger, properties). Required before Phase 5.

---

## Phase 5 — Implementation

**Decision gate (from Phase 0 and Phase 2 answers):**
- Customer uses Segment/Rudderstack/mParticle → use CDP integration, no new SDK → see `reference.md § Phase 5 — Integration Pointers`
- Customer uses Snowflake/BigQuery → see `reference.md § Phase 5 — Warehouse Connectors`
- EU or CA users flagged in Phase 2 → surface the consent pattern from `reference.md § Phase 5 — Consent and Opt-In Tracking` before writing any initialization code
- Otherwise, ask: "Do you want to track from the server (backend), browser/app (client), or both?" — or skip this question if the codebase scan already made the answer obvious

**Tracking method recommendation:**

| Method | Use When | Key Tradeoff |
|---|---|---|
| **Server-side** (preferred) | Any event observable on your backend | Reliable; must manage IDs and parse User-Agent manually |
| **Client-side web** | Anonymous behavior pre-login; UI interactions | 15–30% event loss to ad blockers; use a proxy to mitigate |
| **Client-side mobile** | Native iOS/Android/RN/Flutter | Old app versions persist; harder to fix bugs |

**Token injection:** Use the real project tokens collected in Phase 2 — never write `'YOUR_PROJECT_TOKEN'` if the tokens are already in hand. If a dev/prod split exists, emit both initializations with the correct token for each environment.

**Then surface the relevant SDK section(s) from `reference.md § Phase 5 — SDK Implementation Guide`:**

| Platform | Reference Section |
|---|---|
| JavaScript (Browser) | `reference.md § JavaScript (Browser)` |
| Python (server) | `reference.md § Python (Server-Side)` |
| Node.js (server) | `reference.md § Node.js (Server-Side)` |
| React Native | `reference.md § React Native` |
| iOS Swift | `reference.md § iOS (Swift)` |
| Android Kotlin | `reference.md § Android (Kotlin)` |
| Flutter | `reference.md § Flutter` |
| HTTP API | `reference.md § HTTP API (Language-Agnostic)` |

Each SDK section in reference.md covers the full lifecycle: install → init → track event → super properties → user profile → identify → reset.

**If a codebase scan was run (Pre-Flight):** Do not produce generic code snippets. Write implementation code directly into the specific files identified during the scan:
- Initialization code → the app entry point or config file already identified
- Super property registration → immediately after init, or after the login handler
- Event tracking calls → inside the exact controller, handler, or component functions where the action occurs
- Environment token switching → use the existing env config pattern already present in the codebase

**Server-side:** Forward client IP (`ip`) only when policy and consent rules permit geolocation enrichment. Always set `$insert_id` for deduplication. Parse User-Agent manually for `$browser`, `$os`, `$device`.

**QA gate — verify before proceeding to Phase 6:** Ask the customer to deploy their current changes to the dev environment, open Mixpanel Live View (mixpanel.com → dev project → Live View), and confirm at least one event appears. Do not proceed to identity management until basic event ingestion is confirmed working. Debugging initialization and identity at the same time makes root-cause analysis very difficult.

**Output of this phase:** All tracking and initialization code written and placed in the codebase. At least one event confirmed arriving in Mixpanel Live View. Customer ready to wire up identity calls.

---

## Phase 6 — Identity Management

**If a codebase scan was run (Pre-Flight):** The login, signup, logout, and session-restore handlers are already located. Do not ask whether anonymous browsing exists — read the auth flow to determine it directly. Place `.identify()`, `.people.set()`, `.register()`, and `.reset()` calls in the exact locations already identified. Only ask if something in the auth flow is ambiguous (e.g., whether a middleware handles re-authentication on page load).

**If no codebase is available, ask:**
1. "Does your product have anonymous browsing before login, or do users authenticate immediately?"
2. "Do users access your product on multiple devices or platforms?"

If anonymous browsing exists or multi-device usage is likely: cover this phase in full.
If users always authenticate immediately (e.g., SSO-only internal tool): anonymous bridging section can be skipped.

**The three required calls (client-side):**
```
On login or signup     → mixpanel.identify(user.id)
On app re-open         → mixpanel.identify(user.id)  [if already logged in]
On logout              → mixpanel.reset()
```

**How Simplified ID Merge works:** When an event contains both `$device_id` and `$user_id` for the first time, Mixpanel merges all past and future events under the `$user_id` as canonical `distinct_id`.

**Correct signup flow order:**
1. Create user in database
2. Call `.identify(user.id)`
3. Set profile properties via `.people.set()`
4. Update super properties via `.register()`
5. Track `sign_up_completed` event (AFTER identify — so it's attributed correctly)

**Server-side identity:** SDKs do not auto-generate `$device_id`. Store a UUID in a cookie. Pass `$device_id` on every pre-login event. Pass both `$device_id` and `$user_id` on the first post-login event. See `reference.md § Phase 6 — Server-Side Identity Flow` for full Python example.

**Full client-side flow** (signup → logout → re-open) is in `reference.md § Phase 6 — Client-Side Identity Flow`.

**QA before production:** Run the ID Management QA Checklist from `reference.md § Phase 6`.

**Output of this phase:** Identity calls (`identify`, `reset`) placed in the correct locations. ID Management QA checklist passed in dev before any production deployment. Required before Phase 7.

---

## Phase 7 — Data Governance

**Ask:**
1. "Who will be responsible for keeping event names and properties consistent over time — a data engineer, PM, analyst, or all three?"
2. "Do you have a shared internal wiki (Notion, Confluence, Google Drive) for your tracking plan and governance docs?"

**Assign roles before implementation:**

| Role | Responsibility |
|---|---|
| **Data Owner** | Approves new events before they go live |
| **Analyst / PM** | Documents use cases; verifies events match tracking plan |
| **Engineer** | Implements only reviewed and approved events |
| **Data Governor** | Oversees Lexicon; enforces naming standards; runs quarterly reviews |

**Set up Lexicon immediately** (Data Management → Lexicon):
For every event shipped, add: Description (one sentence: what triggers it, what it represents), Tags (domain/team), Example property values.

**Enable Data Standards** (Project Settings → Data Standards):
- Require `snake_case` for all event and property names
- Require descriptions before events appear in reports

**Enable Event Approval** (Project Settings → Event Approval):
- Unreviewed event names go to a pending queue until a Data Owner approves them
- Prevents test events, typos, and undocumented tracking from polluting production

**Hiding vs. Dropping:**
- **Hide** → still stored, just removed from UI dropdowns. Use for deprecated events you may still need historically.
- **Drop** → stops ingesting new data. Cannot be undone. Hide first, observe one quarter, then drop.

**Merging divergent events:** Lexicon → select both events → Merge → choose canonical name → update future tracking.

**Quarterly review:** Audit zero-volume events, check for missing Lexicon descriptions, validate naming conventions on new events.

See `reference.md § Phase 7` for: governance pitfalls table, tracking plan column schema, naming change management process.

**Close:** After Phase 7, summarize the full implementation plan back to the customer:
1. Their Value Moment and top 2–3 KPIs
2. Their two starting events and properties
3. Tracking method (server-side / client-side / CDP)
4. Remind: add Lexicon descriptions for every event they ship
5. Point to the tracking plan template for their vertical
6. Offer to write the first event call for their specific stack

**Output of this phase:** Lexicon populated, Data Standards enabled, Event Approval enabled, governance roles named and documented. Implementation complete.

---

## Phase Exit Checklists (Gate Review)

Run the relevant checklist before moving to the next phase.

**Phase 0 exit**
- Business model summary confirmed with customer.
- CDP/warehouse status, Group Analytics flag, and top business questions captured in Context Block.
- Platform and product type captured from codebase or confirmed via questions.

**Phase 1 exit**
- One named Value Moment confirmed.
- 2-3 KPIs pass the 5M filter.
- KPI-to-business-question linkage is explicit.

**Phase 2 exit**
- Simplified ID Merge setting verified.
- Dev and production projects exist with correct timezone.
- EU/CA (or stricter) consent flag documented.

**Phase 3 exit**
- Customer can distinguish events, event properties, user profiles, and super properties.
- Group Analytics scope confirmed or explicitly out of scope.

**Phase 4 exit**
- `sign_up_completed` and Value Moment event fully specified (trigger + properties).
- Naming conventions validated (`snake_case`, stable values).
- Tracking plan reviewed and approved by product, engineering, and analytics.

**Phase 5 exit**
- Initialization and event calls implemented in codebase.
- At least one event observed in dev Live View.
- Tracking path (SDK/CDP/warehouse) matches discovery decisions.

**Phase 6 exit**
- `identify`, `reset`, and profile/super-property ordering validated.
- ID Management QA checklist passed in dev.
- Multi-device and anonymous-to-auth flows tested where applicable.

**Phase 7 exit**
- Lexicon entries populated for shipped events.
- Data Standards and Event Approval enabled.
- Governance roles named and quarterly review owner assigned.

---

## Current-Docs Verification (Before Hard Assertions)

Before stating hard limits, plan entitlements, or irreversible settings, verify against current Mixpanel docs and the customer's account plan.

Quick verification checklist:
1. Confirm feature availability (Group Analytics, governance features, connectors) for the active plan.
2. Confirm any numeric limits (event names, property constraints, rate limits) from current docs.
3. Confirm irreversible settings (identity mode, timezone implications) before implementation.
4. Record what was verified and source links in working notes when decisions depend on it.

---

## Critical Rules — Highest-Stakes Implementation Decisions

Get these wrong and the data is permanently corrupted or very expensive to fix.

**Project setup:**
- Never track to production before creating and verifying a separate dev/staging project
- Verify Simplified ID Merge is enabled BEFORE sending a single event — cannot safely change after data exists
- Set project timezone correctly at creation — cannot change retroactively without affecting historical data

**Identity management:**
- Always call `.identify(user.id)` on EVERY login AND every app re-open while the user is already logged in
- Always call `.reset()` on logout — failing to do so merges the next user's session with the previous user
- Never use email as `$user_id` — emails change; use your database primary key
- Never call `.identify()` before creating the user in your database
- Never call `.people.set()` before `.identify()` — profiles set before identify may not merge correctly
- Track the `sign_up_completed` event AFTER `.identify()`, not before
- Never merge two `$user_id` values — not supported in Simplified API; use one stable ID from the start
- Do not create User Profiles for anonymous users

**Data model:**
- Never send numeric values as quoted strings — they become non-aggregatable strings
- Never construct event or property names dynamically at runtime — creates thousands of unique names
- Never use `$` or `mp_` prefixes on custom event or property names
- Omit properties entirely when they have no applicable value — do not send `null` or `""`
- Mixpanel is case-sensitive: `checkout_completed` ≠ `Checkout_Completed` — enforce snake_case from day one

**Compliance and privacy:**
- If consent is required and status is unknown, do not initialize non-essential tracking
- Do not forward IP or sensitive attributes when customer policy disallows them
- Prefer data minimization: collect only properties needed to answer agreed business questions

**Governance:**
- Do not begin implementation without a reviewed and signed-off tracking plan
- Hide events before dropping them — dropping is irreversible and stops new data ingestion immediately
- Never drop data without a quarter of observation after hiding it

---

## Reference

All detailed guidance is in [reference.md](reference.md), organized by phase heading.

Key sections:
- **Phase 0** — Discovery questions and gate logic
- **Phase 1** — Full RAE Framework, Value Moment formula, 5M filter, KPI tables
- **Phase 2** — Project setup steps, token-switching code, role permissions
- **Phase 3** — Full data model, property types, Group Analytics code
- **Phase 4** — Tracking plan methodology, vertical-specific events, template links
- **Phase 5** — All SDK code (JS, Python, Node.js, React Native, iOS Swift, Android, HTTP API), CDP/warehouse integration
- **Phase 6** — Full identity flows (client-side and server-side), QA checklist
- **Phase 7** — Governance framework, pitfalls table, tracking plan column schema
- **Reference table** — All key Mixpanel documentation URLs
