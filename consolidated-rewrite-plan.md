# Mixpanel Implementation Skill — Consolidated Rewrite Plan

This plan consolidates recommendations from three independent AI reviews (Claude, Cursor, OpenAI) into a single rewrite specification. Where the three plans agreed, the consensus is adopted. Where they diverged, a decision has been made based on the principles below.

---

## Governing Principles

1. **No one-way doors.** Every mandatory question in the fast path exists because skipping it creates rework or data corruption that can't be undone. Everything else gets deferred.
2. **User chooses the mode upfront.** Don't assume. Ask once, route cleanly.
3. **Time to first event is the north star for Quick Start.** Every step that doesn't directly contribute to events appearing in Live View is either trimmed or deferred.
4. **The full flow is preserved, not degraded.** Quick Start is an addition, not a replacement.
5. **Keep the journey simple and reversible.** The user can always go deeper later. They should never have to undo something the skill pushed them into.

---

## Decision Log (Where the Plans Diverged)

| Question | Plan A (Brief) | Plan B (Recommendations) | Plan C (Rewrite Recs) | Decision |
|---|---|---|---|---|
| How many mandatory questions? | ~3 + research | 6 non-skippable | 3-5, some deferred | **Minimum possible with no one-way doors** (see below) |
| Identity: gate or inline? | Inline basics, defer rest | Conditional gate with escalation | Similar to B, less explicit | **Inline with escalation flag** (see Step 5) |
| Fast path = default? | 4th mode with selection guidance | Default for all new users | In between | **Ask the user which mode** — no assumption |
| Mini tracking plan? | Quick thumbs-up | Mini plan with trigger/properties/duplication | Agrees with B | **Yes — mini tracking plan (B/C approach)** |
| One project OK for session one? | Defers verification | Not addressed | Yes, one project OK | **One project fine for Quick Start; dev/prod for Full Rollout** |

---

## Structural Change: Mode Selection at Entry

### Current behavior
The skill defaults all users to the full 8-phase flow.

### New behavior
The skill asks one routing question at the start:

> "What brings you here today?"
> 1. **Quick Start** — Get your first events into Mixpanel in one session
> 2. **Full Implementation** — Build a complete, production-ready analytics setup from scratch
> 3. **Add Tracking** — Extend an existing Mixpanel implementation with new events
> 4. **Audit** — Review and diagnose an existing implementation

The agent states the selected mode explicitly and offers to switch at any point.

### Mode mapping to current skill structure

| Mode | Maps to | What changes |
|---|---|---|
| **Quick Start** | New compressed flow (see below) | Trims phases 0-4, inlines identity, defers governance |
| **Full Implementation** | Current Full Greenfield Rollout (phases 0-7) | No changes — preserved as-is |
| **Add Tracking** | Current Focused Remediation | Minor reframe: starts with "what do you want to track?" not "what's broken?" |
| **Audit** | Current Implementation Audit | No changes |

### Mode switching rules (preserved from current skill)
- If Quick Start surfaces high identity complexity, consent risk, or CDP/warehouse usage → offer to escalate to Full Implementation
- If Full Implementation user says "can we just get something working first?" → offer to switch to Quick Start
- Escalation is always an offer, never automatic. The user decides.

---

## Quick Start Flow — Detailed Specification

This is the only mode that requires significant rewriting. The other three modes are preserved or lightly reframed.

### Success criteria
A Quick Start session is successful when:
- Two events (`sign_up_completed` + Value Moment) are defined with a mini tracking plan
- Tracking code is written and placed
- At least one event is confirmed in Live View
- Basic identity (identify on login, reset on logout) is wired in
- Any hard blockers (consent, CDP routing) have been surfaced

### Step 1 — Mandatory Questions (No One-Way Doors Only)

Ask only the questions where a wrong assumption creates irreversible rework:

**Question 1: "What platform are you building on?"**
(web, iOS, Android, React Native, Flutter, server-side, combination)
- **Why mandatory:** Determines SDK selection. Wrong SDK = rewrite.
- **Can be inferred from Pre-Flight:** Yes — skip if codebase scan already answered this.

**Question 2: "Are you sending data through a CDP or warehouse tool already?" (Segment, Rudderstack, mParticle, Snowflake, BigQuery)**
- **Why mandatory:** If yes, the entire implementation path changes. SDK installation gets skipped; routing goes through the integration. Building direct SDK when a CDP exists = duplication and architectural mismatch.
- **Can be inferred from Pre-Flight:** Sometimes (package.json may reveal Segment/Rudderstack).

**Question 3: "Do you have users in the EU or California?"**
- **Why mandatory:** If yes, consent must gate SDK initialization. Shipping events before consent = compliance violation that requires data deletion.
- **Can be inferred from Pre-Flight:** No — this is a business/legal fact, not a code fact.

**Question 4: "What's the most important action a user takes in your product?"**
- **Why mandatory:** This names the Value Moment event. Without it, we don't know what to track.
- **Can be inferred from Pre-Flight:** Partially — the agent can propose candidates from route/controller analysis, but the user confirms.

That's it. Four questions maximum (fewer if Pre-Flight answers some).

**What about Group Analytics and identity complexity?** These are important but not one-way doors in the Quick Start context:

- **Group Analytics:** Can be added later without rework. The events tracked in Quick Start don't become invalid if Group Analytics is added afterward. Defer to "what's next" recommendations.
- **Identity complexity:** Basic identify/reset is correct for both simple and complex cases. The risk is that complex cases need *more* identity work — but the basic work isn't *wrong*. Surface it as a flag, not a gate (see identity section below).

### Step 2 — Context Gathering (Research or Pre-Flight)

The agent uses whatever input is available, in priority order:

| Available input | What the agent does | Time budget |
|---|---|---|
| **Codebase access** | Pre-Flight scan (unchanged from current skill). Extracts tech stack, candidate events, auth flow, existing analytics. | No time limit — this is the highest-value accelerator |
| **Company URL** | Light Research: homepage + pricing page + login/signup page. Extract product type, B2B/B2C signal, candidate Value Moment, sign_up_method values. Cap at 3 pages, under 2 minutes. | 2 minutes max |
| **Neither** | Skip research entirely. Use the 4 mandatory questions above. | 0 minutes |

**Rules for Light Research:**
- Stop as soon as the agent can confidently fill: product type, platform confirmation, B2B vs B2C, candidate Value Moment
- Do NOT research: Crunchbase, job listings, G2/Capterra, blog, LinkedIn, TechCrunch
- Do NOT require company name or URL before proceeding — if the user doesn't offer one, skip research and ask the questions
- If the homepage and pricing page answer everything, stop there

**After context gathering, present assumptions:**
> "Based on [what I found / what you told me], here's what I'm working with: [platform], [tracking method], [Value Moment candidate]. Sound right?"

One confirmation, then move on.

### Step 3 — Mini Tracking Plan (2 Events)

For each of the two events, capture:

**Event 1: `sign_up_completed`**
```
Event name:    sign_up_completed
Trigger:       User completes account creation (after DB write, after identify)
Where it fires: [signup handler / endpoint identified in Pre-Flight or asked]
Required properties:
  - sign_up_method (string): "email", "google", "apple", "sso"
  - platform (string): "web", "ios", "android"
Optional properties:
  - referral_source (string): UTM source or referral code if available
Duplication notes: Do not fire on social auth redirect — only on final account creation
```

**Event 2: [Value Moment event]**
```
Event name:    [inferred from Step 1, e.g. report_generated]
Trigger:       [specific user action]
Where it fires: [handler / endpoint]
Required properties:
  - [2-3 properties inferred from codebase or vertical defaults]
Optional properties:
  - [1-2 additional if obvious]
Duplication notes: [any edge cases]
```

Present both to the user for confirmation. This is a lightweight review, not a formal sign-off — but it's structured enough that the implementation has clear specs.

### Step 4 — Project Setup (Minimal)

For Quick Start:
- Confirm the user has one Mixpanel project with a token
- If they can't find it: direct them to mixpanel.com → Project Settings → Project Token
- Store the token in the Context Block
- Move on

**Do NOT require for Quick Start:**
- Dev/prod project split (recommend as follow-up)
- Simplified ID Merge verification (it's the default since April 2024)
- Role assignment
- Timezone verification
- Project structure decisions

**One project is acceptable for session one.** Dev/prod split is surfaced in "what's next."

### Step 5 — Implementation + Identity

This is the core of Quick Start. The agent writes real code, placed in specific files if Pre-Flight was run.

**Implementation covers:**
1. SDK initialization (with real token from Step 4)
2. Consent gate if EU/CA users flagged in Step 1
3. `sign_up_completed` event call
4. Value Moment event call
5. Basic identity: `identify()` on login/signup, `reset()` on logout

**Identity approach — inline with escalation flag:**

For Quick Start, identity is NOT a separate phase. The agent wires in three calls as part of implementation:

```
On signup:  create user in DB → identify(user.id) → people.set() → track('sign_up_completed')
On login:   identify(user.id)
On logout:  reset()
```

This is correct for all complexity levels. It doesn't become *wrong* if complexity is high — it just becomes *incomplete*.

**The escalation flag:** After wiring basic identity, the agent checks for complexity signals (from Pre-Flight or conversation):

| Signal | What it means |
|---|---|
| Anonymous browsing exists before login | Anonymous-to-authenticated bridging needed |
| Multi-device or multi-platform usage | Cross-device identity testing needed |
| Shared devices or account switching | Reset logic needs careful placement |
| SSO with multiple identity providers | Identity source needs to be stable |

If any signals are present, the agent says:
> "Your basic identity is wired and will work correctly. But I noticed [signal] — that means there are edge cases we should test before production. Want to do a full identity QA pass now, or come back to it?"

This is an offer, not a gate. The user decides.

**Why this approach:** Basic identify/reset is never *wrong* — it's just sometimes *insufficient*. That makes it safe to ship and revisit, unlike consent (where shipping without it = compliance violation) or CDP routing (where shipping SDK when CDP exists = duplication). The inline approach preserves momentum for simple cases while still surfacing risk for complex ones — without adding a classification question ("is your identity setup simple or complex?") that most first-time users can't answer well.

### Step 6 — Verify in Live View

- Deploy to dev environment (or local if no dev exists)
- Open Mixpanel Live View
- Trigger both events
- Confirm they appear with correct properties
- Confirm identity is linking events to the user

**Do not proceed to "what's next" until at least one event is confirmed in Live View.**

### Step 7 — Quick Start Wrap-Up

Summarize what was shipped:
> "You now have two events live in Mixpanel — `sign_up_completed` and `[Value Moment]` — with basic identity wired in."

Present prioritized next steps:

1. **Add more events** — Expand tracking plan from the 2-event foundation
2. **Full identity QA** — Test anonymous bridging, multi-device, edge cases (especially if complexity flags were raised)
3. **Dev/prod project split** — Create a separate dev project before sending production traffic
4. **Analytics strategy** — Define KPIs and measurement framework (RAE framework available)
5. **Data governance** — Set up Lexicon, Data Standards, and Event Approval as you scale

Each next step maps to content that already exists in the current skill (phases 1, 2, 4, 6, 7). The user can come back for any of them.

---

## Changes to SKILL.md — Summary

### What changes

1. **Add mode selection question at entry** — Replace "default to all 8 phases" with the 4-mode routing question
2. **Add Quick Start flow** — The 7-step flow described above, as a new section
3. **Add "Add Tracking" mode** — Light reframe of Focused Remediation for the "extend existing" use case
4. **Update Context Block** — Add a minimal version for Quick Start (platform, tracking method, EU/CA flag, Value Moment, 2 events, one token)
5. **Add fast-path rules section** — Explicit list of what Quick Start skips and what it preserves
6. **Update mode switching rules** — Add escalation paths from Quick Start to Full Implementation

### What stays unchanged

- Full Greenfield Rollout (phases 0-7) — preserved as-is
- Implementation Audit mode — preserved as-is
- Pre-Flight codebase scan — preserved as-is
- Compliance and Privacy Guardrails — preserved as-is
- Critical Rules section — preserved as-is, applies to all modes
- Phase Exit Checklists — preserved for Full Implementation, not enforced in Quick Start
- Communication Habits — preserved as-is
- Current-Docs Verification — preserved as-is

---

## Changes to reference.md — Summary

### What changes

1. **Add a Quick Start Reference section at the top** — Minimal SDK snippets for each platform: init + track + identify/reset. Three code blocks per SDK, not the full lifecycle walkthrough. This lets the agent reach implementation code without navigating the full SDK guide.

### What stays unchanged

- All existing SDK sections (full lifecycle)
- Vertical-specific event examples
- Identity flows (client-side and server-side)
- Governance content
- Tracking plan templates
- Everything else

---

## Fast-Path Rules (New Section for SKILL.md)

```markdown
## Fast-Path Rules

In Quick Start mode:

### Do not require before implementation:
- Company name or URL research
- Deep external research (Crunchbase, job listings, G2, etc.)
- Business model synthesis
- RAE framework, 5M filter, or formal KPI design
- Broad event taxonomy or tracking plan beyond the first 2 events
- Cross-functional sign-off on tracking plan
- Dev/prod project split
- Simplified ID Merge verification (it's the default)
- Role assignment or governance setup
- Full data model education

### Do require before implementation:
- Platform confirmation (one-way door: wrong SDK = rewrite)
- CDP/warehouse status (one-way door: SDK when CDP exists = duplication)
- EU/CA consent status (one-way door: events before consent = compliance violation)
- Value Moment identification (can't track without knowing what to track)
- Mini tracking plan for 2 events (structured enough for clean implementation)
- One valid project token

### Do include during implementation:
- Consent gate if EU/CA users (before SDK init)
- Basic identity (identify on login/signup, reset on logout)
- Live View verification

### Surface after implementation (as next steps, not gates):
- Expanded tracking plan
- Full identity QA (especially if complexity flags raised)
- Dev/prod project split
- Analytics strategy and KPI framework
- Data governance (Lexicon, Data Standards, Event Approval)
- Group Analytics setup (if B2B)
```

---

## Minimal Context Block (New, for Quick Start)

```markdown
## Quick Start Context Block

- **Platform(s):**
- **Tracking method:** client-side / server-side / CDP
- **CDP in use:** none / [name]
- **EU or CA users:** yes / no
- **Value Moment:**
- **Event 1:** `sign_up_completed` — properties: [list]
- **Event 2:** [Value Moment event] — properties: [list]
- **Project token:**
- **Identity complexity flags:** [none / anonymous browsing / multi-device / shared devices / account switching]
```

The full Context Block (with business model, growth model, KPIs, etc.) is used only in Full Implementation mode.

---

## Should This Be Split Into Multiple Skills?

### Recommendation: Not yet, but structure it for eventual splitting.

**The case for splitting:**
- Quick Start and Full Implementation are genuinely different user journeys with different pacing, different required context, and different success criteria
- A single skill with mode-switching logic adds complexity to the prompt and increases context window pressure
- Separate skills would have cleaner intent matching (user says "set up Mixpanel" → Quick Start skill; user says "full analytics implementation" → Full Implementation skill)

**The case against splitting (for now):**
- The reference.md content is shared — both modes use the same SDK snippets, identity flows, and compliance guardrails
- Mode escalation (Quick Start → Full Implementation) is smoother within one skill than as a handoff between two
- The "Add Tracking" and "Audit" modes are thin enough that they don't warrant their own skills
- Maintaining two skills with shared reference content introduces sync risk

**Recommended structure for this rewrite:**
- Keep it as one skill, but organize SKILL.md with clear section boundaries between modes
- Put the mode selection at the very top
- Put Quick Start as the first detailed section (it's the most common path)
- Put Full Implementation as the second section (unchanged from current)
- Put Add Tracking and Audit as shorter sections at the end
- This structure makes it easy to split later if context window pressure becomes a problem

**If splitting later:** The natural split would be:
1. **mixpanel-quick-start** — Steps 1-7 from above + Quick Start Reference from reference.md
2. **mixpanel-full-implementation** — Current phases 0-7 unchanged
3. **mixpanel-remediation** — Add Tracking + Audit modes (could share a skill since both start with diagnosis)

Each would carry its own copy of the Compliance and Privacy Guardrails and Critical Rules sections.

---

## Implementation Priority

If making changes incrementally, do them in this order:

### Change 1: Add mode selection at entry
Replace "default to all 8 phases" with the 4-mode routing question. This is the single highest-impact change — it stops routing every user through the full flow.

### Change 2: Add the Quick Start flow
Write the 7-step Quick Start section into SKILL.md. This gives the most common user type a clean, fast path.

### Change 3: Add Quick Start Reference to reference.md
Minimal SDK snippets (init + track + identify/reset) for each platform at the top of reference.md. This lets the agent reach code fast without navigating the full SDK guide.

### Change 4: Add fast-path rules and minimal Context Block
Makes the lightweight behavior explicit so the agent doesn't fall back to full-flow habits.

### Change 5: Reframe Focused Remediation as "Add Tracking"
Light rename and reframe — starts with "what do you want to track?" instead of "what's broken?"

---

## What This Plan Preserves

- The full 8-phase flow, unchanged
- All compliance and privacy guardrails
- All Critical Rules
- Pre-Flight codebase scan
- The full reference.md content
- Phase Exit Checklists (for Full Implementation)
- Current-Docs Verification protocol
- Implementation Audit mode

## What This Plan Adds

- Mode selection at entry (4 modes)
- Quick Start flow (7 steps, ~15 minutes to first event)
- Fast-path rules section
- Minimal Context Block for Quick Start
- Quick Start Reference section in reference.md
- Identity complexity escalation flag

## What This Plan Removes from the Critical Path

- Mandatory company/URL research before implementation
- Deep business model synthesis
- RAE framework / 5M filter / formal KPI design
- Broad event taxonomy design
- Cross-functional tracking plan sign-off
- Dev/prod project split requirement
- Role assignment and governance setup
- Full data model education walkthrough

All of these remain available in Full Implementation mode and as recommended next steps after Quick Start.
