# Layout shift and misclick audit: <TARGET>

Run <DATE> against <URLS / APP> and <REPO>. Two methods: Chrome performance traces for the web surfaces, plus a source audit of the patterns the metric cannot see.

The phenomenon is layout instability, measured on the web as Cumulative Layout Shift (CLS). Good is 0.1 or less. Native apps have no equivalent metric.

## Verdict

<One paragraph: is the primary flow safe, and where is the real hazard. State plainly if the worst finding is invisible to CLS.>

| Surface | Result |
| --- | --- |
| <primary flow> | <CLS lab / field, one-line assessment> |
| <list page> | <...> |
| <modal or picker> | <worst finding, one line> |
| <native app> | <...> |

## How to reproduce the measurements

1. Open Chrome DevTools, Performance panel.
2. Navigate to the target URL first, then record with reload enabled. Recording before navigating attributes shifts to the wrong navigation.
3. Read CLS in the metrics summary, then open the layout shift culprits insight.
4. For each shift note the score, the **start time**, and the impacted elements. Start time drives misclick risk: a shift at 200ms is harmless, a shift at 2s is not.
5. Compare lab CLS against the field (CrUX) value in the same panel. Field is p75 across real users over 28 days.

URLs used: <list>

### Results as measured

| Page | Lab CLS | Field CLS (CrUX p75) | Notes |
| --- | --- | --- | --- |
| <url> | <n> | <n or "no field data"> | <worst shift score and start time; INP and LCP if notable> |

<Note any page where CrUX has no field data: that page is measured only by a single unthrottled lab run, and if the cause is a late request, real users see worse.>

## Finding N: <one-line mechanism>

**Severity:** <highest / high / medium / low>. <One line on why, in likelihood x consequence terms.>

**Where:** `<path:line>`, plus `<path:line>`

**Mechanism.**

<Prose, with the actual code quoted. Explain why the code does what it does before saying it is wrong. If a comment shows the team already fixed an adjacent symptom, say so: it tells you the fix is welcome, not novel.>

```
<the actual lines>
```

### Repro

1. <Set up the data state needed: throttle to Slow 4G, cold cache, specific account shape.>
2. <Navigate.>
3. <Observe without clicking, to confirm the shift exists.>
4. <Repeat and click at the moment of the shift.>
5. <Read back what was actually selected.>

**What to look out for:** <The specific tell. Not "watch for movement" but "watch the section headers, not the rows" / "watch the bottom of the list, not the top" / "the button drifts rather than jumps". Also say how the failure presents, especially if it is silent.>

## Separate bugs, not layout shift

<Adjacent defects found in passing: non-deterministic comparators, missing resize handlers, INP. Clearly labelled so they are not confused with the findings above.>

## What the code does well

<Its own section, with file references. This is where the recommended fixes come from, and it keeps the report honest. Name the best structural decision explicitly.>

## The metric blind spot

<What the measurement structurally could not see: modals opened by the user, post-interaction shifts, auth-gated surfaces, user-specific list sizes, and all native surfaces. State whether anything relying on CLS as a guard will keep reporting green.>

## Open questions needing a device or runtime check

<Anything that cannot be settled statically, and what would settle it.>

## Priority order

1. <Smallest fix with the highest consequence, with in-repo precedent named.>
2. <...>
