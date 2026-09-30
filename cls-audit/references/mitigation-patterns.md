# Mitigation patterns

Search for these with the same effort you spend on hazards. Two reasons: a hazard with a guard a few lines away is not a finding, and an existing in-repo pattern is the fix you should propose, because it is the one that will actually ship.

## Hold the pane

Keep the skeleton until **every enabled section** has resolved, rather than retiring it when the first one arrives. Costs a little perceived latency, removes the entire insertion class.

The usual objection is that it feels slower. The answer is that a list which rearranges under the user is not usable earlier, it only *looks* ready earlier.

## Reserve the space

- Explicit `width` and `height`, or `aspect-ratio`, on anything async: images, logos, media, charts.
- `min-height` on containers that grow, with a comment saying it is load-bearing so nobody "cleans it up".
- Skeletons whose height **equals the real content's height**. A skeleton of the wrong height is a layout shift with extra steps.
- Export the sizing helper so skeleton and content cannot drift apart. If content computes its width from data, the reservation cannot depend on that data.

## Put the action above its warnings

Structural and free. If the primary button renders before any conditional content, nothing appearing can move it.

## Banner and disable in the same commit

When content appears that the user must read, flip the action to disabled in the same render. The pane can grow freely because the target is not live.

**Fail closed while pending.** A safety check that returns "not blocked" while still loading is the same bug as a flag defaulting false: fold the loading flags into the disabled condition.

## Freeze while submitting

Once an action is in flight, stop accepting new derived data into the view. Name the escape hatch something discouraging so the next person has to think before using it.

## Reserve with opacity, not unmounting

`opacity: 0` plus `pointer-events: none` holds the space and makes the target unclickable. Correct on both axes. Unmounting gives up the space; `visibility: hidden` alone leaves a clickable target in some setups.

## Stage refresh behind an explicit action

The "show new posts" pill. New content is fetched and held, and the user decides when it lands. The correct pattern for any prepending feed, and the one thing consumer feeds get right.

## Sort server-side

If the backend returns the list already ordered, there is no client re-sort to shift. Sidesteps hazard 1 entirely.

## Stable keys, stable order

Keys derived from identity (an id, an address), never from position. Once an item is rendered in a bucket, keep it there for the session even if its value arrives later.

## Hold the previous value

Debounce, persist the last good value, and dim rather than unmount while refetching. Prevents the empty-then-full oscillation that causes repeated shifts.

## Pin anchoring explicitly

If a virtualized list depends on scroll anchoring, set it in the call site rather than inheriting a library default. A default is not a decision, and it disappears silently on upgrade.

## Animate continuously if you must move something

If a change is unavoidable, a view-transition style animation at least lets a human track the target instead of re-acquiring it. Much weaker than not moving it, and never a substitute for reserving space.

## What does not work

- **Confirmation dialogs on everything.** Trades one frustration for another and trains people to dismiss.
- **Suppressing clicks for N ms after a re-render.** Taxes every correct click to rescue the rare wrong one, and there is no shipped browser mechanism for it.
- **Guessing intent.** You cannot distinguish "meant the element that was there" from "meant the element that is there now".
