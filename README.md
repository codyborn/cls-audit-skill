# cls-audit (Claude Code skill)

A reusable Claude Code skill for auditing a web app or native app for **layout-shift and accidental-misclick hazards**: the failure where async content moves an interactive target just before you click it, and you hit the wrong thing.

It pairs real Chrome performance traces (lab plus CrUX field CLS) with a source audit of the hazard patterns the metric cannot see, then produces a ranked report where every finding carries file:line, numbered repro steps, and a "what to look out for" line.

## Why the source audit is the point

CLS is collected on page load, from the main document. The worst instances of this bug class tend to live where it cannot look: inside a modal the user opens, after an interaction, behind auth, or in a native app where no such metric exists at all.

A trace returning 0.00 is weak evidence of safety. The skill treats a clean metric as a fact to report *next to* a list of what that number cannot cover.

## Install

Copy (or symlink) the skill directory into your Claude Code skills folder:

```bash
git clone git@github.com:codyborn/cls-audit-skill.git
ln -s "$(pwd)/cls-audit-skill/cls-audit" ~/.claude/skills/cls-audit
```

Claude Code picks it up automatically. It triggers on requests to test a site or app for layout shift, CLS, content jumping, misclicks, or "the button moved before I tapped it".

Works with the `chrome-devtools` MCP server if you have it, and falls back to plain DevTools instructions if not.

## Usage

```
Audit https://example.com for layout shift. Source is at ~/repos/example.
```

Give it the URLs, the repo path, and ideally the worst thing a wrong click can do on that product. That last one sets the severity scale for the whole report.

## Layout

```
cls-audit/
  SKILL.md                        # the audit, phases 0-6 (3b covers no-source targets)
  references/
    hazard-patterns.md            # 12 hazard patterns: mechanism, what to grep, resulting wrong-click
    mitigation-patterns.md        # what good looks like, and what does not work
    black-box-techniques.md       # auditing a site you cannot read the source of
  templates/
    audit-note.md                 # report skeleton
```

## Core principles (if you read nothing else)

1. **Do not trust a clean metric.** Measure, then write down what the measurement structurally could not see, then go look there.
2. **Rank by likelihood times consequence, never by shift magnitude.** A small shift on a payment button outranks a large shift on a filter chip.
3. **Hunt for structural immunities as hard as for hazards.** Finding that a platform pins its action buttons to the bottom voids an entire category of finding, and is worth more than five individual hazards.
4. **Verify every load-bearing claim in the source yourself.** Quote the line. Check whether a guard a few lines away already defuses it.
5. **The "what to look out for" line is the most valuable part of a finding.** A reader watching the wrong signal concludes there is no bug. Tell them the actual tell.
6. **Throttle in every repro.** Slow 4G widens the window, it does not create the bug. Without it these are unreproducible on a developer machine, which is exactly why they ship.
7. **Search for the mitigations too.** The fix with in-repo precedent is the one that ships.
8. **Report negative results.** What you tested and found clean is coverage information; without it a reader cannot tell an untested surface from a safe one.

## License

MIT
