# Eye rectangle gate simulator

**Live: https://viorem.github.io/eye-rect-calibration-simulator/**

Single-file interactive simulator for the face-calibration width-axis gates, built to
share the centring-vs-closeness argument with the team.

Everything lives in [`index.html`](index.html) — no build step, no dependencies, no
network calls. Open it directly in a browser, or deploy it as a static site.

## Tabs

| Tab | What it shows |
|---|---|
| **A — Current rule** | The width-axis rule as first described. Includes a **Left bound** switch: `× r` as described vs `× (1 − r)`, the likely intended symmetric form. |
| **B — Proposed** | Centring separated from closeness: independent `too close` / `too far` / `off centre` / `out of frame` checks. |
| **C — Production code** | Faithful port of the shipped Kotlin. Gates **both axes** and reports directional `WarningLocation` values. |
| **D — Compare** | Allowed centring window vs `eRect_width` for every variant, with a value table. |

Tab C is the authoritative one — it is verified against a direct transcription of the
Kotlin. Note its X rule differs from tab A: it is symmetric about 540 with a tolerance
of `ratio × 540` that does **not** vary with `eRect_width`.

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

**C — production (the shipped Kotlin)**

```
expectedX = 1080 / 2.0 = 540        expectedY = 1920 / 2.5 = 768
xError = (expectedX − eRect_x − eRect_width  / 2) / expectedX
yError = (expectedY − eRect_y − eRect_height / 2) / expectedY

buffer       = shouldBeRelaxed ? relaxationRatio       : 0
centreBuffer = shouldBeRelaxed ? centreRelaxationRatio : 0

minOuter = minOuterDistance × (1 − buffer)
maxOuter = maxOuterDistance × (1 + buffer)
rLR, rTop, rBottom = ratio × (1 + centreBuffer)      // corrected, see below

ERROR if avgOuterDistance <= minOuter   (too far)
ERROR if avgOuterDistance >= maxOuter   (too close)
xError < −rLR      → RIGHT     xError > rLR     → LEFT
yError < −rBottom  → BOTTOM    yError > rTop    → TOP     // corrected, see below
```

**The tab differs from the shipped Kotlin in exactly two places**, both deliberate
corrections rather than porting errors:

1. Offsets scale by `(1 + centreBuffer)`; the shipped code has `(1 − buffer)`. A
   symmetric ± tolerance must scale *up* to widen. The distance band was already
   correct because its two ends move in opposite directions. Before the fix,
   `shouldBeRelaxed = true` loosened distance while tightening centring by 30%.
2. The Y branches are swapped. `yError` is positive when `cy` is small — the face is
   *high* — so that case is `TOP` and takes `rTop`. The shipped code fires `TOP` on a
   *low* face, contradicting X, where a large `cx` correctly gives `RIGHT`.

Tolerance is `ratio × 540` horizontally and `ratio × 768` vertically. `expectedY` is
`1920/2.5`, not the vertical centre, because eyes belong above the midline.

Centring has its own `centreRelaxationRatio`, separate from the distance
`relaxationRatio` but gated by the same `shouldBeRelaxed` flag, so the two axes can be
tuned independently. The **relax edge** preset sits at `cx = 665`: it warns unrelaxed
and passes once relaxed, which is relaxation behaving the way it should.

The stage reads both axes off the edges rather than labelling them inside the frame:
allowed `cx` on the strip **below** the frame, allowed `cy` on the strip to its
**left**, and `minOuterDistance` / `maxOuterDistance` as caliper lines under the eye
rect, which track it as it moves.

An indicative head outline is drawn around the eye rect on every rule tab — **1.35×
the eye-rect width, 2.3× tall**, eye line 46% down from the crown. It is decoration
for reading the geometry, not part of any check; the frame clips it, so "too close"
shows as a head overflowing the view.

Those ratios are **perspective-corrected, not flat anatomical**. Flat anatomy gives
width `150/90 = 1.67×` and height `230/90 = 2.56×` the outer-eye distance. But a
selfie camera is only ~300–450 mm away and the widest part of the skull sits ~95 mm
further from the lens than the eye corners, so head width is projected from greater
depth and compressed by `D/(D + 95)` — about 0.81 at 400 mm, giving ≈1.35. Crown and
chin are only ~70 mm and ~30 mm back, so height loses less: ≈2.3. The eye line barely
moves (45.6% vs 46%) because the compression is near-symmetric about it. Ears are not
drawn: at this range they sit at or behind the silhouette tangent and are largely
self-occluded, so including them overstates the width.

Deriving the distance live from `eRect_width` was tried and rejected. At a 60°
horizontal FOV, `eRect_width = 750 px` (`maxOuterDistance`) implies `D ≈ 112 mm` and a
head *narrower* than the eye corners, which is impossible — so the eye rect cannot be
90 mm of real outer-canthal distance in this coordinate space. Fixed ratios stay sane
across the whole range instead.

Note the checks here are per-frame, so the aggregate `faceErrorThresholdRatio` /
no-face-percentage gate is deliberately not modelled.

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
