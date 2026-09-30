# Hazard patterns

Each entry: the mechanism, what to grep for, and the wrong-click it produces. Ordered roughly by yield.

## 1. Async sort key

A list renders before the value it sorts by has arrived, then re-sorts.

**Grep:** `.sort(`, `sortBy`, `orderBy`, `useMemo` sorts whose dependency array contains query results, `fetchPolicy: 'cache-and-network'`, `staleTime: 0` feeding a sorted list.

**Wrong click:** the row under the cursor becomes a different row. Anchoring cannot help, because a permutation has no stable anchor.

### 1a. Bucket migration, the severe variant

The sort partitions into buckets and concatenates, for example value-ranked items first and zero-value items alphabetically last. An item whose value is missing from the cached payload sits at the very bottom, then jumps into the ranked section when fresh data lands. Not a nudge: a full-list traversal.

**Tell:** two `filter` calls with complementary predicates, then two sorts concatenated.

## 2. Positional section splicing

A list is assembled by spreading N sections in fixed order, each with a `?? []` fallback. A section whose query is pending contributes zero height, then materializes at full height *in its slot*, displacing everything below.

**Grep:** array literals spreading several `?? []` sections; section-builder hooks.

**Aggravators:** one section sourced from local state (instant) while the rest are network, guaranteeing a two-stage paint. Sections containing headers or horizontal tile rows, which are taller than list rows and therefore displace more.

## 3. Loading gate satisfied by partial data

```
const isLoading = (!sections || !sections.length) && loading
```

Once *any* section exists the skeleton retires, and the remainder stream in live.

**Grep:** loading expressions combining a data-presence check with a loading flag via `&&`.

## 4. Orphaned loading state

The UI gates on query A's loading flag, but sorts or renders using query B's data. Before B resolves every key is undefined, comparisons tie, the unsorted order paints, and B's arrival reorders everything at once.

**Grep:** a component with two or more data hooks where only one `loading`/`isLoading` reaches the render gate. Check the comparator for `return 0` on undefined.

## 5. Action below insertable content

A primary button sits after content that can appear: warnings, fee rows, banners, promos.

**Check:** is the button *disabled* in the same commit the content appears? If yes, mostly defused. If the button stays live while moving, this is a real hazard at the top consequence tier.

**Inverse worth recommending:** render the action *above* its warnings. Then nothing appearing can move it.

## 6. Animated height change

A container with a height transition grows when async content lands, gliding everything below it for the duration.

**Grep:** `transition: height`, `transition: all`, `will-change: height`, animated layout height, `height: unset` or `auto` on a container whose content is async-gated.

**Why it is worse than a jump:** the target is in motion for the whole transition, the user tracks it, and the click lands mid-flight.

## 7. Unstable row identity

- **Index-based keys** (`keyExtractor={(_, i) => 'row' + i}`) make a framework reuse the wrong node when membership changes.
- **Same key, different destination.** A row whose label, icon and position are identical but whose target changed after an async read. No remount, no visual cue, and no way for the user to notice.

**Grep:** `keyExtractor` using an index; ternaries on async state that pick a navigation target while the key stays constant.

## 8. Flags and gates that default false while pending

A feature gate or compliance check returns false or null while loading, so the UI renders one shape and then changes when the verdict lands. Both directions hurt: content **inserted** pushes rows down, content **removed** pulls them up.

**Grep:** feature-flag hooks, gated queries, entitlement checks. Confirm what they return *before* resolution, and whether a `null` return means "no" or "not yet".

## 9. Discovery-populated pickers

Lists that append as items are discovered: Bluetooth or WiFi lists, cast and AirPlay targets, wallet or session pickers, network and chain selectors, account switchers.

Worst class in consumer software generally, because the wrong tap connects to or sends to the wrong target immediately. Doubly bad when the picker also sorts by a live value.

## 10. Virtualization anchoring, especially by default

Virtualized lists may offer scroll anchoring (`maintainVisibleContentPosition`, `minIndexForVisible`, `overflow-anchor`).

Two failure modes:
- **Anchoring off or disabled**, often deliberately (sorting moves keyed nodes and anchoring then chases a moved row to an arbitrary position). The side effect is that scroll position stays pinned and different rows sit under the cursor.
- **Anchoring on by library default** while the wrapper types it as opt-in and no caller sets it. The protection is real but invisible and fragile: a version bump or a changed wrapper default silently exposes every list. Pin it explicitly.

**Grep:** the prop name across the repo. If only the wrapper mentions it and no feature code does, check the library's default value.

## 11. Stale press geometry

Some press implementations measure the press rectangle once, on touch start, then test only the finger against that stale rectangle on move. A row sliding out from under a held finger never cancels and fires *its own* handler on release: correct data, wrong position, no visual cue.

An app-wide amplifier, and the thing that converts a cosmetic shift into a wrong tap. Hit-slop widens the window further.

**Grep:** press or pressability compat layers, `pressRect`, `measure(` inside a grant or touch-start handler.

## 12. Hit-testing resolves at dispatch, not at intent

The platform-level root cause, worth stating in any report. Hit-testing runs against the layout current at event dispatch, not the layout the user saw when deciding. A shift between pointer-down and pointer-up delivers the click to whatever now occupies the pixel. Nothing in the event model records the frame the user actually aimed at.

## Adjacent bug classes found in passing

Not layout shift, but they surface constantly during this audit and belong in their own report section:

- **Non-deterministic comparators.** `sort((a) => cond ? -1 : 1)` is single-argument and not antisymmetric, so ordering is implementation-defined. Any comparator ignoring its second argument, or returning a boolean, is broken.
- **Missing resize handlers on gated confirmations.** A scroll-to-bottom confirm gate measured once, with no content-size-change handler, can stay satisfied when a late warning grows the content, letting a user confirm something they never saw.
- **INP above 200ms**, which produces the same user complaint through a different mechanism and is worth reporting alongside.
