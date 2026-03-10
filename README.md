# Mixpanel Implementation Skill

Structured guidance for helping Mixpanel customers implement analytics correctly — from a 15-minute Quick Start to a full production-ready rollout. Supports four modes: Quick Start, Full Implementation, Add Tracking, and Audit.

**This file is for human maintainers.** The agent reads `SKILL.md` at runtime — not this file.

---

## Architecture

The skill is split across two files intentionally:

| File | Role | When the agent reads it |
|---|---|---|
| `SKILL.md` | Mode selection, Quick Start flow, full phase flow (0–7), decision rules, guardrails, and Context Block schemas | Always, at the start of every session |
| `reference.md` | Quick Start SDK snippets, deep playbooks, all SDK code, vertical-specific examples, QA checklists | On demand, when a mode or phase requires it |

This split keeps `SKILL.md` fast to load and easy to reason over. Pushing all SDK snippets and vertical-specific content into `reference.md` means the agent isn't carrying 1,800+ lines of code examples into every conversation — it fetches what it needs, when it needs it.

**Consequence for maintenance:** When something changes in Mixpanel behavior or SDK APIs, update `reference.md` first. Only touch `SKILL.md` if the mode structure, phase flow, decision logic, guardrails, or Context Block schemas change.

---

## File Map

- `SKILL.md` — Mode selection (4 modes), Quick Start flow (7 steps), Full Implementation phases (0–7), Add Tracking mode, Audit mode, compliance guardrails, Pre-Flight codebase scan, Context Block schemas (minimal + full), phase exit checklists, communication habits, and critical rules
- `reference.md` — Quick Start Reference (minimal SDK snippets per platform), full RAE Framework, all SDK code (JS, Python, Node.js, React Native, iOS Swift, Android Kotlin, Flutter, HTTP API), vertical event examples, tracking plan templates, CDP/warehouse integration notes, identity flow walkthroughs, governance pitfalls, and the ID Management QA checklist
- `README.md` — This file; for maintainers only

---

## Modes

The skill asks the customer which mode fits their goal before doing anything else:

| Mode | What it covers | Success criteria |
|---|---|---|
| **Quick Start** | 7-step compressed flow: mandatory questions → context → mini tracking plan → project setup → implementation + identity → Live View verification → wrap-up | Two events live in Mixpanel with basic identity wired in |
| **Full Implementation** | All 8 phases (0–7) in order: Discovery → Analytics Strategy → Project Setup → Data Model → Tracking Plan → Implementation → Identity Management → Data Governance | Complete production-ready analytics setup with governance |
| **Add Tracking** | Starts with "what do you want to track?" → checks existing schema → designs new events → implements and verifies | New events live with correct naming and identity linkage |
| **Audit** | Diagnoses current state → produces prioritized fixes → executes fixes via Add Tracking or Full Implementation | Prioritized fix list with severity ranking |

Mode switching is always an offer, never automatic. Quick Start can escalate to Full Implementation if complexity surfaces. Full Implementation can downshift to Quick Start if the customer wants momentum first.

---

## Full Implementation Phase Summary

These phases apply to Full Implementation mode only. Quick Start uses Live View verification as its primary gate.

| Phase | Goal | Hard gate |
|---|---|---|
| 0 — Discovery | Business model, platform, CDP status, business questions | Must complete before Phase 1 |
| 1 — Analytics Strategy | Named Value Moment + 2–3 KPIs passing the 5M filter | Must complete before Phase 4 |
| 2 — Project Setup | Simplified ID Merge verified, dev + prod projects created, tokens stored | Must complete before Phase 3 |
| 3 — Data Model | Customer aligned on events, properties, profiles, super properties | Must complete before Phase 4 |
| 4 — Tracking Plan | Signed-off event schema for sign_up_completed + Value Moment | Must complete before Phase 5 |
| 5 — Implementation | All tracking code written; at least one event confirmed in dev Live View | Must complete before Phase 6 |
| 6 — Identity Management | identify/reset calls placed correctly; ID QA checklist passed in dev | Must complete before Phase 7 |
| 7 — Data Governance | Lexicon populated, Data Standards enabled, Event Approval enabled | Implementation complete |

---

## Known Limitations

- **No live Mixpanel API access.** The skill cannot query the customer's Mixpanel account, verify project settings, or inspect actual ingested data. It relies on the customer describing their setup.
- **Account plan is unverifiable at runtime.** Feature availability (Group Analytics, Data Standards, Event Approval, warehouse connectors) varies by plan. The skill instructs the agent to verify these against current docs and the customer's account before asserting availability.
- **SDK code may lag behind releases.** Mixpanel ships SDK updates independently of this skill. Always check the current SDK changelog when writing production initialization code.
- **Not legal advice.** The compliance and privacy guardrails in `SKILL.md` are implementation defaults, not legal guidance. Customer policy and counsel are the authoritative source for consent and data residency requirements.
- **No enforcement mechanism.** The skill guides the agent to gate phases and reject shortcuts, but a customer who overrides the agent can bypass any guardrail. The skill documents the risk, not the enforcement.

---

## Maintenance Guide

### When to update `reference.md`

- Mixpanel ships a breaking SDK change (initialization API, `identify`/`reset` signature, ingestion endpoint)
- A new SDK platform becomes commonly requested (e.g., a new Flutter version, a new React Native architecture)
- The Simplified ID Merge behavior or the `$device_id` / `$user_id` contract changes
- A CDP or warehouse integration path changes (Segment schema, Rudderstack SDK, BigQuery connector)
- Tracking plan templates or vertical event examples become stale relative to common customer patterns
- Quick Start Reference snippets need to match updated full SDK sections

### When to update `SKILL.md`

- The mode selection, routing logic, or mode switching rules change
- The Quick Start flow steps need reordering or a step needs to be added/removed
- The phase sequence for Full Implementation needs reordering or a phase needs to be added/removed
- A new critical rule emerges from common implementation failures
- The Context Block schemas (Quick Start or Full) need new fields
- A compliance or privacy default changes (new regulation, new Mixpanel data residency option)
- The Pre-Flight codebase scan logic needs to extract different signals
- Fast-path rules need adjustment (what Quick Start requires vs. defers)

### Staleness signals to watch for

- Customer feedback that a code snippet doesn't match the current SDK
- A Mixpanel changelog entry that changes a behavior this skill relies on (ID merge, Lexicon API, Data Standards enforcement)
- A new Mixpanel plan tier that changes which governance features are available
- Agent-generated identity code that produces ID fragmentation in testing

### How to verify the skill is current

1. Check the [Mixpanel JS SDK changelog](https://github.com/mixpanel/mixpanel-js/releases) and compare against `reference.md § JavaScript (Browser)`.
2. Check the [Mixpanel Python SDK](https://github.com/mixpanel/mixpanel-python) and Node.js SDK release notes.
3. Review Mixpanel's "What's New" blog and changelog for any product-level changes to Identity Management, Lexicon, or Data Standards.
4. Run the full skill on a test scenario and verify the Phase 5 code compiles and produces events in a dev Mixpanel project.
