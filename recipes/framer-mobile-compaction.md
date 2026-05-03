---
title: "Framer Mobile Compaction"
slug: framer-mobile-compaction
dek: "A repeatable technique for cutting mobile scroll length 15-42% on Framer sites without removing content. Section padding compression, hide-vs-compact decisions, 2-up grid breakpoints."
public: true
tags: [framer, mobile, design-system, breakpoints, compaction]
applies-to: [framer, generic]
created: 2026-05-01
updated: 2026-05-01
status: shipped
order: 1
category: recipes
---

# Framer Mobile Compaction

A repeatable technique for taking a Framer site that "renders fine but scrolls forever" on mobile and cutting total scroll length 15-42% without removing content. The first move is *measurement*; the second move is *density-not-restructure*; the third move is *2-up grids on tablet, 1-up on phone*.

## When to apply

- Total mobile scroll exceeds ~10 viewports on iPhone 14 (390×844)
- Hero section feels tall even after viewing it once
- Section padding looks "right" on desktop but creates dead space on mobile
- Content is genuinely there (essay teasers, testimonials, stat blocks) — not a layout-broken case
- Two-column or three-column desktop grids that collapse to single column on mobile

Not for: layouts that are actively broken on mobile (nav not rendering, content overflowing horizontally). Fix structural bugs first.

## Steps

### 1. Measure first — establish the baseline

Use Playwright at the three test viewports (320, 390, 600) and capture:

- `document.documentElement.scrollHeight` (total page height)
- `document.documentElement.scrollWidth` (catch horizontal overflow)
- `window.innerHeight` and `innerWidth`
- Per-section heights (`section[].getBoundingClientRect().height`)
- Computed total scroll in **viewports** (`scrollHeight / vh`)

Record before/after numbers in the recipe applied, not just "looks better." If you can't quantify the improvement, you don't know whether you helped.

### 2. Identify the offenders

Rank sections by height. The top-3 offenders typically account for 50%+ of the page height. They're where the recipe applies.

Common offender patterns:
- **Hero with `min-height: 100vh`** still applied on mobile (eats ~840px on iPhone 14 even when content is shorter)
- **Multi-card sections** that stack 1-up on mobile (4 cards × ~350px each = 1400px)
- **Stats/data sections** with desktop-grid spacing carried into mobile
- **Long lists** (essays, positions) with generous per-row vertical padding

### 3. Apply the density toolkit

In priority order — biggest single-knob wins first:

#### Section vertical padding (biggest)

Desktop generous, mobile dense. Typical move: `120px` → `48px` top, `48px` → `24px` bottom. Per-section savings: 100-200px each. Across 6+ sections, this alone can save 800-1500px.

```css
@media (max-width: 720px) {
  .pm-root .sec-head { padding: 48px 0 24px; }
  .pm-root .now-grid { padding-bottom: 32px; }
  /* etc. */
}
```

#### Hero `min-height: 100vh` removal

Drop it on mobile. The content tells the hero how tall to be.

```css
@media (max-width: 720px) {
  .pm-root .hero { min-height: auto; padding-top: 64px; padding-bottom: 28px; }
}
```

#### Hide-vs-compact for secondary content

Some sections have a "primary + secondary" structure (e.g. a stats grid + a build log card). On mobile, the secondary card is often noise. Hide it entirely.

```css
@media (max-width: 720px) {
  .pm-root .now-log { display: none; }
}
```

This is more aggressive than tightening padding, but ~360-600px savings per hidden card.

#### 2-up grids at tablet width (480-720)

The single biggest visual win, when applicable. A 4-card section that's 1-up on phone becomes 2-up on small tablet. Width range: 480px to your mobile breakpoint (typically 720). Below 480, fall back to 1-up.

```css
@media (min-width: 481px) and (max-width: 720px) {
  .pm-root .builder-grid { grid-template-columns: 1fr 1fr; }
  .pm-root .essays { display: grid; grid-template-columns: 1fr 1fr; gap: 0 16px; }
  .pm-root .card-defs { display: grid; grid-template-columns: 1fr 1fr; gap: 6px 16px; }
}
```

At 600px viewport, this alone can drop total scroll from ~10 viewports to ~7.7 (the Butchsonic baseline).

#### Line-clamp deks on mobile

If a list has `<title> + <dek>` repeating rows (essays, articles), clamp the dek to 2 lines on phone, 3 lines on tablet.

```css
@media (max-width: 720px) {
  .essay-dek {
    font-size: 15px;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
  }
}
```

#### Card padding compression

Internal padding inside cards: `28px 24px` → `18px 14px`. Per-card savings: ~30-40px.

#### Stat block compression

`stat-n` font clamps `(40px, 5vw, 72px)` → `(36px, 9vw, 52px)` on mobile. Drops the lower bound and steepens the ramp so the number compresses with the viewport.

### 4. Don't restructure — only compact

Tempting moves to AVOID:
- Removing entire sections "because mobile users don't need them"
- Reflowing display titles to mid-case ("read on for...")
- Replacing the hero portrait with a smaller circle

These are content changes disguised as compaction. Resist. The recipe is layout/typography only.

### 5. Tablet sweet spot

If the site has substantial 2-up-grid content (multiple cards), the 480-720 breakpoint is where you can hit Butchsonic-grade compaction (~7.7 viewports). Below 480, you're 1-up no matter what — so the 320 (iPhone SE) experience will always be ~50% taller than the 600 (small tablet) experience for the same content.

That's an acceptable trade-off. iPhone SE users tolerate scroll; what kills them is awkward wrap, overflow, and broken nav.

## Gotchas

- **Soft-hyphens in display titles wrap weird at 320.** A `&shy;` in `unrecogniz&shy;able` will split the word with a visible hyphen mid-display title. See `rules/anti-patterns.md` #2.
- **`min-height: 100vh` on the hero is silent until measured.** Even when mobile rules collapse the grid, the height stays. Always check explicitly and override with `min-height: auto`.
- **Tables in markdown-rendered prose can horizontally overflow at 320.** Vertical scroll measurement won't catch it. Always measure `docW > vw` separately. See anti-pattern #3.
- **2-up grids below 480 are too narrow.** Don't try; 1-up is the floor. Each card needs ~200px minimum to stay readable.
- **Don't change the desktop layout while compacting mobile.** If desktop is shipped and approved, mobile-only media queries are the contract.

## Validation

Quantitative — record before and after:

- Total scroll height in pixels at 320, 390, 600
- Total scroll in viewports (`scrollHeight / vh`)
- Per-section heights
- `docW == vw` (no horizontal overflow)

Qualitative — visual diff:

- Screenshot at 320, 390, 600 before AND after; compare side-by-side
- Hero feels "compressed but breathing," not "cramped"
- 2-up tablet grids look intentional, not accidental

## Validated on

- `butchsonic.com` · 2026-04-27 · 11,279px → 6,538px @ 390 (13.2 → 7.7 viewports, **42% reduction**)
- `philmora.com` · 2026-04-30 · 11,594px → 9,693px @ 390 (13.7 → 11.5 viewports, **16% reduction**); 22.8 → 18.7 @ 320 (**18%**); ~10 → 7.7 @ 600 (**22%, hits Butchsonic baseline**)

## References

- philmora `framer-snapshot-2026-04-30-mobile.md` (commit `cb66717`): https://github.com/philmora/philmora-site/commit/cb66717
- Butchsonic `framer-build.md` (the original technique source, before extraction to this recipe)

## Updates

<!-- Append-only. Date each addition. -->
