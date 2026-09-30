# Black-box techniques

For auditing a site whose source you cannot read. Everything here runs from DevTools or a CDP client, and none of it needs the repo.

## 1. The layout-shift observer

Install **before page scripts run** (DevTools: a "run before load" init script, or a CDP `Page.addScriptToEvaluateOnNewDocument`; the chrome-devtools MCP exposes this as `navigate_page`'s `initScript`). `buffered: true` also recovers entries emitted before the observer attached.

```js
window.__shifts = [];
new PerformanceObserver((list) => {
  for (const e of list.getEntries()) {
    window.__shifts.push({
      t: Math.round(e.startTime),
      score: +e.value.toFixed(4),
      recentInput: e.hadRecentInput,
      nodes: (e.sources || []).map(s => {
        const n = s.node;
        if (!n || !n.tagName) return 'detached';
        const testid = n.getAttribute && n.getAttribute('data-testid');
        const cls = typeof n.className === 'string' ? '.' + n.className.split(' ')[0] : '';
        return n.tagName + (testid ? '[' + testid + ']' : '') + cls;
      }).slice(0, 3)
    });
  }
}).observe({ type: 'layout-shift', buffered: true });
```

Then read it out:

```js
() => {
  const s = window.__shifts || [];
  const sum = a => +a.reduce((x, y) => x + y.score, 0).toFixed(4);
  return {
    total: sum(s),
    count: s.length,
    excluded_from_cls: s.filter(x => x.recentInput).length,
    excluded_score: sum(s.filter(x => x.recentInput)),
    late: s.filter(x => x.t > 2500).length,
    biggest: s.slice().sort((a, b) => b.score - a.score).slice(0, 6)
  };
}
```

### Why it beats the trace

- **Per-shift scores and timestamps**, including shifts the CLS session-window accounting drops.
- **`hadRecentInput` is visible**, so you can count what CLS excludes by design instead of inferring it.
- **Live DOM nodes in `sources`.** This is the important one. On a minified site the class names are hashes and tell you nothing; the same node's `data-testid` or component attribute often names the component outright. Recovering `component-shelf` from a node whose class is `UDd6VhjOTDGlgJq8l5Fk` is the difference between "something moved" and "the shelves move".
- **It keeps working after load**, which is where the trace stops looking.

### Limits

- Programmatic scrolling does **not** set `hadRecentInput`. Only discrete input (click, keypress, tap) does. Dispatch real events if you want to exercise that path.
- `sources` can be empty or `detached` for nodes removed before the callback ran.
- An SPA route change does not reset the observer, which is usually what you want, but means you must timestamp your navigations to attribute shifts.

## 2. Drive the blind spots

After installing the observer, read it between each step:

1. Load, wait for settle, read. This is the baseline the trace also sees.
2. **Scroll** through several viewports with pauses, then read. Catches lazily-loaded content inserting above what you passed.
3. **Open every menu, picker, sheet and dropdown** a user would open, one at a time, reading after each. This is the largest blind spot and where the worst findings usually live.
4. **Switch tabs, filters and sort orders**, reading after each. Re-sorts show up here.
5. **Repeat cold.** Clear cache and redo step 1 a few times; the spread between runs is part of the finding.

## 3. Row-identity diffing

When the observer says a container moved but not why, capture ordering directly:

```js
() => [...document.querySelectorAll('<row selector>')]
  .map((el, i) => i + ':' + (el.getAttribute('data-testid') || el.textContent.trim().slice(0, 40)))
```

Run shortly after paint and again a couple of seconds later, then diff:

- **Same length, different order**: a re-sort. The dangerous case, and anchoring cannot fix it.
- **Longer**: an insertion. Check whether it landed above or below the fold.
- **Shorter**: a removal pulling rows up. Often a gate resolving to "no".
- **Same order, different content under the same identity**: the worst case, because nothing visible changed position. Check whether the link targets changed.

## 4. Reading the trace's impacted elements

If the impacted element is a **container** (a scroll node, a main content wrapper, a footer container) rather than the thing that grew, then something *above* it resolved late and displaced everything below. The container is the victim, not the cause. Look upward in the DOM for the element whose height changed, or use the observer to catch the growing node directly.

Several shifts within a few hundred milliseconds naming the **same repeated element type** is the signature of independent sections resolving one at a time.

## 5. What you cannot conclude

Be explicit in the report about the ceiling of a black-box audit:

- You can establish **that** content displaces a target and **when**, to the millisecond.
- You cannot establish **which request was late**, so causes are hypotheses. Label them.
- You cannot see auth-gated surfaces without signing in, and signing in to someone else's production account is out of scope for an audit. Say which surfaces went unexamined; a personalized home feed is usually the one that matters most and the one you cannot reach.
- You cannot propose the in-repo fix, only the general one.

Findings and repro steps are reachable. File:line and "use the pattern you already have in X" are not.
