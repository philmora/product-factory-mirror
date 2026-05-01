# product-factory-mirror

**Public airlock for The Product Factory — sanitized cross-project craft recipes.**

This is the public-facing mirror of a private wiki at `~/BaseCamp/factory/`. Everything in this repo is hand-sanitized and explicitly marked `public: true` before publishing. Nothing here references employer-specific systems, customer data, or proprietary internals.

## What's in here

Portable recipes for shipping software products — the practice, not the projects.

- Framer + design-system techniques
- Claude Code workflow patterns
- PM/build craft
- Cross-project design decisions
- Anti-patterns

Each recipe is one markdown file in `recipes/`. The catalog lives in `recipes.json` (parallel to `wiki.json` in `philmora/work-brain-mirror`).

## How it surfaces

This repo is the source for `https://philmora.com/factory` — a Framer page that fetches `recipes.json` at runtime and renders each recipe via the same prose components used at `/work-brain`.

```
~/BaseCamp/factory/                        ← private wiki (encrypted sparsebundle)
        │
        │  airlock (manual sanitization, per-recipe)
        ▼
philmora/product-factory-mirror            ← THIS REPO (sanitized subset)
        │
        ▼ at runtime
philmora.com/factory                       ← rendered Framer page
```

## Writing discipline

A file in this repo is **not a filtered copy of a private file**. It's a hand-authored, generalized companion — same idea, same structure, but:

- No employer-specific systems, codebases, customer names, or roadmap details
- No personal names without explicit permission
- No tokens, keys, internal URLs, or auth-required references
- No project-specific knowledge that wouldn't apply to a brand-new product

If a recipe can't be generalized without gutting it, it doesn't belong here yet.

## Frontmatter

Every recipe starts with:

```yaml
---
title: <short title>
slug: <kebab-case>
public: true
tags: [...]
applies-to: [framer, generic, ...]
created: YYYY-MM-DD
updated: YYYY-MM-DD
status: draft | shipped | deprecated
---
```

## Sister surfaces

- **The Product Factory** on claude.ai — the conversation side; this wiki is the durable side
- `~/BaseCamp/factory/` — the private wiki (this is its public subset)
- `philmora/work-brain-mirror` — the sister mirror for `/work-brain` (concepts, not recipes)
- `philmora.com/factory` — the rendered surface (TBD)

## License

Recipes here are CC BY 4.0 — fork, copy, riff. Attribution: link back to philmora.com.
