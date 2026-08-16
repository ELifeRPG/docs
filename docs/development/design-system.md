---
icon: lucide/palette
---

# Design System

This page documents the visual conventions for the docs site, derived from
the project logo. It's the reference to check before adding a color, font,
or icon that isn't already covered here.

## Logo

<span style="display:inline-flex; gap:24px; align-items:flex-end;">
<span>
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 80.261627 77.909518" width="120" height="120"><path fill="#1c4fc0" d="M 80.22456,22.53807 68.076458,0 H 21.055044 L 0,38.99231 21.090616,77.909513 68.246559,77.637314 80.261611,55.358674 69.051306,49.433216 60.658955,64.962325 28.642068,65.231072 17.867946,45.241355 61.511427,45.236755 61.476348,32.65233 H 17.839737 l 10.766187,-19.978392 31.881334,-0.0012 8.56485,15.88821 z"/></svg>
<br><sub>light mode</sub>
</span>
<span style="background:#14161d; padding:12px; border-radius:8px;">
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 80.261627 77.909518" width="120" height="120"><path fill="#ddd9d1" d="M 80.22456,22.53807 68.076458,0 H 21.055044 L 0,38.99231 21.090616,77.909513 68.246559,77.637314 80.261611,55.358674 69.051306,49.433216 60.658955,64.962325 28.642068,65.231072 17.867946,45.241355 61.511427,45.236755 61.476348,32.65233 H 17.839737 l 10.766187,-19.978392 31.881334,-0.0012 8.56485,15.88821 z"/></svg>
<br><sub style="color:#c6c1b8;">dark mode</sub>
</span>
</span>

The logo is a bold, geometric "E" mark — hand-designed in Inkscape (not
auto-traced), stored as a **vector SVG**. The base file has a flat black
fill; every colored appearance on the site comes from CSS, not from
separate image files.

- **One file, colored by CSS, not two images.** The site no longer ships
  separate light/dark PNGs. The theme inlines `overrides/.icons/eliferpg/logo.svg`
  directly into the page (see Implementation below), and `extra.css` sets
  its `fill` per color scheme — light mode blue, dark mode grey. The
  preview above is the same path data, just with each variant's `fill`
  hardcoded for illustration.
- **Two colors, chosen by scheme.** Light mode uses the primary blue
  (`#1c4fc0`); dark mode uses a light warm grey (`#ddd9d1`), not the
  primary blue — the dark canvas already carries a lighter accent blue on
  its links, and a second, different-toned blue on the logo right next to
  it reads as a mismatch rather than a deliberate choice. Grey removes
  that clash and leaves the link color as the only blue you see at night.
- **Keep clearspace.** Leave padding around the mark equal to at least the
  width of one letter-stroke on every side — don't crop or crowd it.
- **Don't recolor or distort it beyond the two defined scheme colors.**
  Don't place it on busy backgrounds or apply effects (drop shadows,
  gradients, outlines). Being a flat vector, further recolors are just a
  new `fill` value — but any new variant should be a deliberate addition
  to this page, not an ad hoc one-off.
- **Minimum size.** Don't render it below ~24px — the letterform's angles
  stop reading clearly below that.
- **Design source:** `design/` holds the editable originals — `logo.svg`
  (the Inkscape file, with the brand blue fill and full editor metadata)
  plus `logo-icon.png` / `logo-icon-bg.png` (512×512 renders, transparent
  and on the dark canvas). Not used by the build directly; treat it as
  where the next edit starts.
- **Shipped copies:** `docs/assets/logo.svg` (served statically, used for
  previews like the one above) and `overrides/.icons/eliferpg/logo.svg`
  (the copy the theme actually inlines — kept identical by hand, since
  it's one small file) are both the same path data, stripped of editor
  metadata and recolored to flat black. `docs/assets/logo.png` is
  `design/logo-icon.png` copied as-is, used only for the favicon —
  browser tabs don't reliably respect page-level dark mode, so it stays a
  fixed blue bitmap.

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
| Logo (dark mode) | `#ddd9d1` | Light warm grey — see Logo section above for why dark mode doesn't reuse primary blue. |

Semantic colors (admonition types, diffs, syntax highlighting) are left as
the theme's defaults and aren't overridden.

## Typography

- **Text:** [Inter](https://fonts.google.com/specimen/Inter) — a
  geometric, high-legibility sans that pairs naturally with the logo's
  blocky letterform.
- **Code:** [JetBrains Mono](https://fonts.google.com/specimen/JetBrains+Mono).

Both are configured via `theme.font` in `zensical.yml`; no further type
scale customization is applied beyond the theme's defaults.

## Iconography

Page and navigation icons use the [Lucide](https://lucide.dev/) set,
referenced by name in each page's frontmatter, e.g.:

```yaml
---
icon: lucide/gamepad-2
---
```

Stick to Lucide for any new page or nav icon rather than mixing in other
icon sets (Material, FontAwesome, Octicons are also bundled but should be
avoided for consistency).

## Implementation

| What | Where |
|---|---|
| Palette / font / icon-logo / favicon config | `zensical.yml` (`theme:` block) |
| Color overrides (CSS custom properties, logo `fill`) | `docs/stylesheets/extra.css` |
| Logo, inlined by the theme | `overrides/.icons/eliferpg/logo.svg` |
| Logo, for doc-page previews | `docs/assets/logo.svg` (same content, kept in sync by hand) |
| Favicon source (raster, fixed blue) | `docs/assets/logo.png` |

The logo is set via `theme.icon.logo: eliferpg/logo` with `theme.logo`
left unset — Zensical's logo partial checks `theme.logo` first and falls
back to `theme.icon.logo` only if it's absent, so setting both would
silently keep the old `<img>` behavior and ignore the icon. `custom_dir:
overrides` has to be enabled for `eliferpg/logo` to resolve at all — it
adds `overrides/.icons/` to the icon search path ahead of the theme's
bundled sets (`lucide/…`, `material/…`, etc.).

Three CSS custom properties are in play, and only one of them controls
what most people expect:

| Token | Actually drives | Notes |
|---|---|---|
| `--md-primary-fg-color` | Header logo tint context, `.md-button--primary` fill | Used sparingly in Zensical's default theme variant — it does **not** fill the header background. |
| `--md-typeset-a-color` | **Resting link color** | Defaults to primary if left unset. Needs an explicit value per `[data-md-color-scheme]` to hit AA on both surfaces — a single shared value cannot satisfy both. |
| `--md-accent-fg-color` | Link hover/focus, active nav-item highlight | Not the resting link color, despite the name suggesting otherwise. |

The logo's dark-mode color swap is two `fill` rules scoped to
`[data-md-color-scheme="default"] .md-logo svg` / `[...="slate"] .md-logo
svg` — this only works because the SVG is inlined into the DOM rather
than loaded via `<img src>`; an externally-referenced image can't be
restyled by page CSS at all, colored or not.
