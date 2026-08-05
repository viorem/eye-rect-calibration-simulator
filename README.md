# Eye rectangle gate simulator

**Live: https://viorem.github.io/eye-rect-calibration-simulator/**

Single-file interactive simulator for the face-calibration width-axis gates, built to
share the centring-vs-closeness argument with the team.

Everything lives in [`index.html`](index.html) — no build step, no dependencies, no
network calls. Open it directly in a browser, or deploy it as a static site.

## Tabs

| Tab | What it shows |
|---|---|
| **A — Current rule** | The shipped algorithm. Includes a **Left bound** switch: `× r` as shipped vs `× (1 − r)`, the likely intended symmetric form. |
| **B — Proposed** | Centring separated from closeness: independent `too close` / `too far` / `off centre` / `out of frame` checks. |
| **C — Compare** | Allowed centring window vs `eRect_width` for all three variants, with a value table. |

Geometry (`eRect_x`, `eRect_width`), `maxOuterDistance` and the relaxation factor are
shared across tabs A and B, so the same face position can be judged by both models.
The `far right` preset is the sharpest demo — tab A fails it, tab B passes it.

## The rules

Frame is 1080 × 1920 (portrait); only the width axis is gated.

**A — as shipped**

```
expectedXOffset   = (1080 − eRect_width) / 2
effectiveMaxOuter = maxOuterDistance × (1 + relaxation)
FAIL if eRect_width > effectiveMaxOuter
FAIL if eRect_x    < expectedXOffset × maxOffsetRatio
FAIL if eRect_x    > expectedXOffset × (1 + maxOffsetRatio)
```

The two bounds scale differently (`r` vs `1 + r`), so the allowed window is
asymmetric — more latitude leftward than rightward at every distance — and it
collapses toward zero as the face approaches the frame width.

**B — proposed**

```
midpoint c      = eRect_x + eRect_width / 2
dx              = c − 540                      (540 = 1080 / 2, the frame centre)
policyTol       = centreTolerance × 1080 × (1 + relaxation)
frameLimit      = (1080 − eRect_width) / 2
effectiveWindow = 540 ± min(policyTol, frameLimit)

FAIL if eRect_width > maxOuterDistance × (1 + relaxation)
FAIL if eRect_width < minOuterDistance × (1 − relaxation)
FAIL if |dx| > policyTol
FAIL if |dx| > frameLimit
```

Relaxation widens `policyTol` but never `frameLimit` — the former is policy, the
latter is geometry — so loosening the tolerance can never permit an out-of-frame
eye rect. Whichever is smaller binds, and the UI names which one.

## Deploying

Hosted on **GitHub Pages** from `main` at the repository root. To publish a change,
just push — Pages rebuilds automatically, usually within a minute:

```bash
git add index.html && git commit -m "Update simulator" && git push
```

Note the repository is **public**, so the page and its parameter defaults are
publicly reachable.

### Firebase Hosting (alternative)

`firebase.json` is configured as a fallback — serves this directory, `index.html`
set to `no-cache` so redeploys appear immediately. It needs your Google account:

```bash
npx firebase-tools login
```

```bash
npx firebase-tools use --add
```

`use --add` picks the project and writes `.firebaserc`. Then `npx firebase-tools
deploy --only hosting`.
