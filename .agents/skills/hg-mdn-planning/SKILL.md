---
name: hg-mdn-planning
description: Plan a feature or edit before coding it: requirements, draft, spec, and work-log files in `.agents/plans`, with clarifying questions, bottom-up phases, acceptance criteria, and a validation plan (build, tests, review, UI click-through).  Use before any non-trivial feature or change, and when asked for a plan, spec, design, or work log.
---

# Planning Features and Edits

## Overview

A plan is cheap and a wrong turn is not: settle the objective, the design, and the order of work **before** touching code, and keep a short log of what was done.  Plans live in `.agents/plans` as four files that share one name and move from loose to precise:

| File | Holds | Audience |
|---|---|---|
| `*.req.md` | the raw business and technical requirements | the team |
| `*.draft.md` | the high-level approach, researched against the code and authoritative sources, with options | the team deciding |
| `*.spec.md` | the detailed, hand-off-ready design: files, snippets, phases, tests, open decisions | the implementers |
| `*.log.md` | the phase checklist with acceptance criteria and implementation notes | whoever picks the work up |

## When to Use

- A feature or change that touches more than one layer (data, backend, front-end), or more than one slice
- The user asks for a plan, spec, design, estimate, or "how would we..."
- A request is ambiguous, or has more than one reasonable design

Skip it for a small, local fix: say what you will change and do it.

## Instructions

<rules>
- Name files `{yyyy-MM-dd}-{action}-{description}.{req|draft|spec|log}.md` in `./.agents/plans` (create it if missing; `2026-06-18-add-notebook-sharing.spec.md`); keep `{action}-{description}` identical across the four files
- Read any plan the user already provided first; **if there is none, guide the user to one** (questions below) before writing code
- **Ask clarifying questions** when: something is ambiguous or unstated; analysis of the code reveals a gap in the design; there are several ways to do it and someone must choose; the design contradicts itself; the design removes existing files, classes, or behavior; a decision carries risk (outdated approach, deprecated library, performance, non-standard practice)
- **Present options with a recommended default** for each decision: favor simplicity, reuse of what exists, a small blast radius, performance, and security
- **Research authoritative sources** (first-party docs, the library's repository) to confirm assumptions, and **keep the citations and deep links** in the plan for the next reader
- Ground every draft and spec in the real code: open the files, name them, and use their actual shapes
- Challenge inconsistencies and one-way doors now, while it is cheap; tell the user about risks they have not seen
- Stick to the plan; ask before deviating; record deviations in the log
- Check the plan against the codebase again before starting: it may have drifted
- Order work **bottom-up** and slice it so each phase leaves the build green
</rules>

## Process

### 1. Requirements (`.req.md`)

If the user has no plan, work through what to consider, and write down the answers:

- What is the objective, and who is it for?
- Which capabilities does it need to meet the objective?
- Explicit design objectives: how it should behave, performance and size constraints, security
- Edge cases and failure modes
- Technologies or libraries to research
- Names of the key entities and operations
- The order of updates across layers (data, backend, front-end)
- What can safely be done in parallel (for example with sub-agents)
- How it will be validated at the end

### 2. Draft (`.draft.md`)

Turn requirements into an approach:

- Address each accepted functional and non-functional requirement with a high-level technical approach
- Ground it in the code and in authoritative external sources; expand the referenced files from the real sources
- Prose and bullets; explore alternatives and recommend a default that fits the existing codebase
- Include the high-level verification schedule: how correctness will be checked

### 3. Spec (`.spec.md`)

Make the draft implementable with no ambiguity:

- **Specific**: name the files and the lines or members to change; show annotated snippets of the key changes
- **Phased, bottom-up**, each phase writing its tests with its code (handler test first): models, entities, and migrations; services and stores; handlers; endpoints and the OpenAPI contract; regenerate the client; front-end stores; pages and components; UI verification
- **A test plan**: the scenarios to cover, by level (handler, store, store integration, front-end store, component, page); say where a human must coordinate (a signed-in session, a device)
- **The flow**: the logical path of data and calls through the feature
- **Telemetry**: logs, spans, and metrics to add (`hg-mdn-backend`, `references/telemetry-logging.md`)
- **Open Decisions**: every unresolved choice in one section, each with options and a recommendation
- **Consistent**: check that names, types, and flows agree across sections; call out any inconsistency
- Written for a technical audience that has not seen the discussion

### 4. Work log (`.log.md`)

A short, ordered checklist distilled from the spec, grouped into a handful of slices (not one per file), each leaving the build green.  The order may differ from the spec's phases when that makes better slices.

```md
# (Descriptive title of the feature work)

**Date**: yyyy-MM-dd
**Plan**: path/to/the-working.spec.md
**Draft**: path/to/the-working.draft.md
**Status**: (last checkpoint)

---

## Phase Checklist

### 1. Description of this phase

(Short summary of the objective and work.)

- [ ] (Short, clear task)
- [ ] (Short, clear task)

#### Phase 1 Acceptance Criteria

- `mise run check` passes
- Code review pass on the changed (non-generated) code
- (Criterion)

#### Phase 1 Implementation Notes

(Deviations, surprises, special callouts, and the path the work actually took.)

---

### 2. Description of this phase
```

Update the checklist and notes as each slice lands, so anyone can pick the work up.

## Good Plan Criteria

- Clear, explicit steps that an implementer can execute without asking
- States the **objective**, the **reasoning**, and the technical and design constraints
- Names the key files to change, each with a one-line description of the change
- Links the relevant project docs and skills (`hg-mdn-backend`, `hg-mdn-frontend`) and external references
- Shows short annotated snippets where they clarify scope and intent

## Validation Plan

Every plan ends with how the work will be checked:

- `mise run check` passes: builds with no new warnings, front-end type check, lint, format check, all tests (the same list as AGENTS.md "Working agreements")
- New tests for the new behavior; refactored tests where behavior moved (`hg-mdn-backend`, `references/testing.md`)
- A code review pass on the changed code, excluding generated files
- UI click-through with the Playwright MCP: exercise the feature, the empty, loading, and error states, light and dark, narrow and wide (`hg-mdn-frontend`, `references/testing.md`); ask the user for a signed-in session if the app needs one
- Documentation updated where the project keeps it

## Overall Flow

1. Ensure there is a plan (requirements, then draft, then spec, then log); ask questions until it is unambiguous.
2. Review the plan against the codebase.
3. Implement the log's slices in order; run the validation for each.
4. Update the log as you go; note deviations and why.
5. At the end, run the full validation plan and report what was verified and what was not.
