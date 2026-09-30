---
name: cls-audit
description: Audit a web app or native app for layout-shift and accidental-misclick hazards, where async content moves an interactive target just before the user clicks it. Combines real Chrome performance traces (lab + CrUX field CLS) with a source audit of the hazard patterns the metric cannot see, then produces a ranked report with per-finding repro steps. Use when asked to test a site or app for layout shift, CLS, content jumping, misclicks, or "the button moved before I tapped it".
---

# Layout shift and misclick audit

The phenomenon is **layout instability**, measured on the web as **Cumulative Layout Shift (CLS)**, one of Google's Core Web Vitals. Good is 0.1 or less. Native apps have no equivalent metric, which is the structural reason they get less scrutiny.

**The central insight that shapes this whole audit: CLS measures page load of the main document. The worst instances of this bug class usually live where the metric cannot see them.** A trace that comes back 0.00 is weak evidence of safety. Expect to find the real hazards in source, not in the numbers.

Run both halves. The measurement tells you the metric is or is not clean; the source audit tells you whether users are actually safe.

## Phase 0: scope

Establish before measuring:

1. **Surfaces.** Which URLs, which app screens. Ask for the repo path if it was not given.
2. **Platform split.** Web, native, or both. They have different hazards and different immunities; never assume a web finding applies to native or vice versa.
3. **Consequence ceiling.** What is the worst thing a wrong click can do here? Navigation, or moving money? This sets the severity scale for everything after, so get it explicitly.
4. **Which flows matter.** Prioritize the paths carrying the consequence ceiling, not the most-visited paths.

## Phase 1: measure (web only)

Use Chrome DevTools, or the chrome-devtools MCP if available.

1. Navigate to the target URL **first**, then start the trace with reload enabled. Tracing before navigating attributes shifts to the wrong navigation.
2. Record **lab CLS** and, critically, the **field CLS from CrUX** shown alongside it. Field is p75 across real users over 28 days and is the number that matters. Lab on a fast unthrottled machine is a best case.
3. Open the **layout shift culprits** insight. For each shift record:
   - **Score** (how much movement, weighted by how much of the viewport moved)
   - **Start time**, which matters more than score for misclick risk. A shift at 200ms is harmless because nothing is readable yet. A shift at 2s lands after the user has targeted something.
   - **Impacted elements**. If the impacted element is a *container* rather than the thing that grew, something above it resolved late and displaced everything below.
4. Trace **several pages, not one.** Expect wide variance: a tuned primary flow can score 0.00 while a list page on the same domain scores 0.09. The variance is the finding.
5. Record **INP** and **LCP** while you are there. INP above 200ms produces the same felt complaint ("my click didn't register") through a different mechanism, and a slow LCP widens the window during which things can still move.
6. Note where CrUX has **no field data**. Those pages are measured only by your single lab run, and if the cause is a late request, real users see worse.

Report the numbers in a table. Do not stop here.

## Phase 2: map the blind spots

Before reading code, write down what the measurement structurally could not see. This list is where to aim Phase 3:

- **Anything inside a modal, sheet, or dropdown the user opens.** Not part of page load, so no field data and no synthetic trace.
- **Anything after an interaction.** Shifts following user input are discounted or excluded by design.
- **Anything requiring auth or a funded account.** Portfolio, balances, history.
- **Any second-visit behaviour**, where a warm cache changes which data is instant and which is late.
- **All native app surfaces.** No CLS exists at all.
- **Any list whose contents depend on the specific user.** A demo account with three items will not reproduce a fifty-item reorder.

## Phase 3: source audit

Work the hazard taxonomy in `references/hazard-patterns.md`. The highest-yield patterns, in order:

1. **Lists that re-sort because the sort key arrives asynchronously.** The single most productive search. Look for sorts whose inputs include query data, and for cache-then-network fetch policies feeding a sorted list.
2. **Sections or rows spliced into fixed positional slots**, where a pending section contributes zero height and then materializes in place.
3. **Loading gates that are satisfied by partial data**, so the skeleton retires while later data is still arriving.
4. **A second query whose loading state is never folded into the flag the UI gates on.** Symptom: every row's sort key is undefined at first paint, all comparisons tie, then the whole list reorders in one commit.
5. **Primary actions positioned below content that can appear.** Warnings, fee rows, banners. Check whether the action is *disabled* in the same commit that the content appears; if it is, the hazard is largely defused.
6. **Animated height changes.** Worse than an instant jump, because the user tracks the moving target and clicks mid-transit.
7. **Unstable row identity.** Index-based keys, or the same key pointing at a different destination after data resolves.
8. **Feature flags and gated queries that default false while pending**, so content renders and is then *removed*, pulling rows up.

Search for the **mitigations** too, with equal effort. What a codebase already does right tells you which fix to propose (there is usually an in-repo precedent) and keeps the report from reading as an indiscriminate attack. See `references/mitigation-patterns.md`.

## Phase 4: verify before reporting

**Read every load-bearing claim in the source yourself**, especially if subagents did the sweeping. Quote the actual line. Two failure modes to catch:

- A claimed mechanism that the surrounding code already defuses (a guard a few lines away, a disabled state in the same commit).
- A structural immunity that invalidates a whole category. Example: if a native modal pins its footer to the bottom and pads content by the footer height, then inserted content cannot push the CTA down, and every "CTA moves" finding is void on that platform. Finding one of these is worth more than finding five hazards, so look for them deliberately.

Also flag anything that **cannot be settled statically** and say what would settle it, rather than guessing.

## Phase 5: rank

Rank by **likelihood x consequence**, never by shift magnitude. Consequence tiers:

1. **Irreversible value transfer.** Wrong recipient, wrong asset, wrong chain, wrong permission granted, wrong destructive action confirmed.
2. **Selection that feeds an irreversible action.** Picking the wrong token or account that a later screen then acts on. Ask whether a review step or confirmation actually catches it, and check **both directions**: shifting onto an unfamiliar option often trips a warning while shifting onto a familiar one does not.
3. **Wrong navigation or filter.** Annoying, cheap to recover.

A small shift on a payment button outranks a large shift on a filter chip every time.

## Phase 6: write it up

Use `templates/audit-note.md`. Non-negotiable parts:

- **A reproduction recipe for the measurements**, so anyone can re-run them without this skill.
- **Per-finding: file:line, the mechanism in the code's own terms, numbered repro steps, and a "what to look out for" line.** The last one matters most and is most often skipped. A reader watching for the wrong signal concludes there is no bug. Tell them the actual tell: "watch the section headers, not the rows", "watch the bottom of the list, not the top", "the button drifts rather than jumps".
- **Throttling in the repro steps.** Slow 4G widens the window; it does not create the bug. Without it most of these are unreproducible on a developer machine, which is exactly why they ship.
- **What the codebase does well**, as its own section.
- **The blind-spot section from Phase 2.** If the team is using CLS as their guard, they need to know what it cannot see.
- **A single priority-ordered list** at the end, spanning platforms, that someone can turn into tickets.

## Standing rules

- **Never change code during an audit.** Report only, unless fixing was requested separately.
- **Read-only browsing.** Do not sign in, submit forms, approve transactions, or click anything with side effects on a production site. Reproduce destructive-path hazards by reading code, not by triggering them.
- **Do not trust a clean metric.** "CLS 0.00" belongs in the report next to what that number cannot cover.
- **Prefer the smallest fix with in-repo precedent.** It is the one that ships.
- Report adjacent bugs found in passing (a non-deterministic comparator, a missing resize handler) in their own clearly-labelled section, so they are not confused with layout-shift findings.
