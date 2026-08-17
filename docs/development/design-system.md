---
icon: lucide/palette
---

# Design System

This page documents the visual conventions for the ELifeRPG brand, derived
from the project logo — shared across the web portal, this docs site, and
the in-game mod. It's the reference to check before adding a color, font,
or icon that isn't already covered here. The Implementation section below
covers only this docs site's build; the portal and mod apply the same
conventions in their own codebases.

## Logo

<span style="display:inline-flex; gap:24px; align-items:flex-start;">
<span style="display:inline-block; background:#ffffff; padding:12px; border-radius:8px; text-align:center;">
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 80.261627 77.909518" width="120" height="120"><path fill="#296bd8" d="M 80.22456,22.53807 68.076458,0 H 21.055044 L 0,38.99231 21.090616,77.909513 68.246559,77.637314 80.261611,55.358674 69.051306,49.433216 60.658955,64.962325 28.642068,65.231072 17.867946,45.241355 60.488286,45.236755 60.487784,32.65233 H 17.839737 l 10.766187,-19.978392 31.881334,-0.0012 8.56485,15.88821 z"/></svg>
<br><sub style="color:#4a4640;">light mode</sub>
</span>
<span style="display:inline-block; background:#14161d; padding:12px; border-radius:8px; text-align:center;">
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 80.261627 77.909518" width="120" height="120"><path fill="#ddd9d1" d="M 80.22456,22.53807 68.076458,0 H 21.055044 L 0,38.99231 21.090616,77.909513 68.246559,77.637314 80.261611,55.358674 69.051306,49.433216 60.658955,64.962325 28.642068,65.231072 17.867946,45.241355 60.488286,45.236755 60.487784,32.65233 H 17.839737 l 10.766187,-19.978392 31.881334,-0.0012 8.56485,15.88821 z"/></svg>
<br><sub style="color:#c6c1b8;">dark mode</sub>
</span>
</span>

The logo is a bold, geometric "E" mark, stored as a vector.

- **Two colors, chosen by scheme.** Light mode uses `#296bd8`; dark mode
  uses a light warm grey (`#ddd9d1`), not blue — the dark canvas already
  carries a lighter accent blue on its links, and a second, different-toned
  blue on the logo right next to it reads as a mismatch rather than a
  deliberate choice. Grey removes that clash and leaves the link color as
  the only blue you see at night.
- **Keep clearspace.** Leave padding around the mark equal to at least the
  width of one letter-stroke on every side — don't crop or crowd it.
- **Don't recolor or distort it beyond the two defined scheme colors.**
  Don't place it on busy backgrounds or apply effects (drop shadows,
  gradients, outlines). Any new variant (a third color, a new crop) should
  be a deliberate addition to this page, not an ad hoc one-off.
- **Minimum size.** Don't render it below ~24px — the letterform's angles
  stop reading clearly below that.

## Color palette

The palette starts from the logo's own blue, then shifts to a deeper, more
saturated "royal blue" for the canonical brand color — richer and more
distinct than a stock Material blue.

| Token | Hex | Role |
|---|---|---|
| Primary | `#1c4fc0` | Brand blue. Header/nav chrome, `.md-button--primary` fill. |
| Primary — light | `#86a8ef` | Lighter tint, used where the theme needs a lighter primary variant. |
| Primary — dark | `#123a91` | Darker shade; also doubles as the light-mode link color (see below). |
| Link (light mode) | `#123a91` | Resting link color on white — independently ≥4.5:1 (AA). |
| Link (dark mode) | `#5482e8` | Resting link color on the dark canvas — independently ≥4.5:1 (AA). |
| Accent (hover/focus) | `#1c4fc0` light · `#86a8ef` dark | Link hover/focus and the active nav-item highlight — not the resting link color (see Implementation). |
| Dark-mode background | `#14161d` | A darker, faintly blue-tinted canvas — paired with the deep-blue primary rather than a neutral grey-black. |
| Logo (light mode) | `#296bd8` | The logo's own blue — close to, but not the same token as, Primary. |
| Logo (dark mode) | `#ddd9d1` | Light warm grey — see Logo section above for why dark mode doesn't reuse blue. |

Semantic colors (admonition types, diffs, syntax highlighting) aren't part
of the brand palette — the docs site uses its theme's defaults for these.

## Typography

- **Text:** [Inter](https://fonts.google.com/specimen/Inter) — a
  geometric, high-legibility sans that pairs naturally with the logo's
  blocky letterform.
- **Code:** [JetBrains Mono](https://fonts.google.com/specimen/JetBrains+Mono).

## Iconography

Icons use the [Lucide](https://lucide.dev/) set. Stick to Lucide for any
new icon rather than mixing in other sets — consistency over any single
icon being a slightly better fit.

## Implementation

Palette, font, and logo/favicon config live in `zensical.yml`
(`theme:` block); color overrides and the logo's dark-mode swap live in
`docs/stylesheets/extra.css`. One gotcha worth knowing: the resting link
color comes from `--md-typeset-a-color`, not `--md-accent-fg-color` (that
one only drives hover/focus and the active nav-item highlight) — needs an
explicit value per color scheme to hit AA contrast on both surfaces.
