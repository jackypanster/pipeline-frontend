---
name: fp-design
description: "Frontend pipeline stage 1 — lock the aesthetic and freeze the design system BEFORE any code. Generates DESIGN.md + tokens.css + references (preview.html + screenshots) + a behavioral red test (spec/) from a product brief, freezes them to trunk as design-rev. Use to START a pipeline-frontend feature. Args: repo, branch, the product brief."
---

# fp-design

Stage 1 of the frontend pipeline. Follow the **shim loop in CONTRACT.md** with slot = `design`.

**Skill:** the `design` slot resolves to a design-intelligence skill that generates a complete
design system (palette, typography, spacing, components, motion) from a product brief. It REASONS
and GENERATES; the shim owns the freeze commits, journal, and handoff.

## Steps

1. `git pull --rebase`. Read/seed `.pipeline/current.json`; this command may CREATE it (set
   `feature` from the brief's slug, `stage: design`).
2. Resolve `design` slot from `.pipeline/roles.yaml`; verify the skill is installed (else STOP).
3. **Survey the target repo before generating** (skip only if the brief is a greenfield project):
   - Read existing source (if any), package.json / framework config, any existing design tokens,
     brand assets. The design must fit the stack.
   - Read the product brief. Identify the product category (SaaS, e-commerce, dashboard, portfolio,
     ...), the target audience, and the aesthetic direction.
   - Then **invoke the design skill**: generate the complete design system from the brief. It should
     produce palette (with hex/OKLCH + CSS variables), typography (font pairings + Google Fonts
     import), spacing scale, component guidance, motion, and anti-patterns to avoid. Target a
     *specific* direction, not a generic "clean modern" default.
4. **Write the frozen design system** to `.pipeline/<feature>/`:
   - `DESIGN.md` — the human-readable design system: visual thesis, palette, typography, spacing,
     components, layout, depth, do/don't, responsive breakpoints, motion. (The `design`/`ui` skill's
     9-section DESIGN.md scaffold is the canonical shape.)
   - `tokens.css` — CSS variables (`--color-primary`, `--spacing-*`, `--font-*`, ...). This is the
     **machine-diffable half** of the freeze — it MUST be valid CSS (**verify by parsing** —
     stylelint or any CSS parser, never by eyeball) and internally consistent with DESIGN.md.
   - `references/preview.html` — a static, self-contained page rendering the frozen system (tokens
     applied, each component from DESIGN.md, light+dark if the design specifies both). Regenerable
     and diffable — this is where the reference shots come from.
   - `references/*.png` — screenshots **captured from preview.html** (never hand-picked mockups —
     an implementation can't pixel-match an unachievable mockup and review deadlocks). At least one
     desktop + one 375px-mobile capture. These are the **visual oracle** `fp-review` compares the
     live render against, pre-ship.
   - `spec/*` — a minimal headless **behavioral red test** (CONTRACT §What this freezes): stable
     `data-*` hooks per screen/component, key interactions, basic a11y. YOU (the fp-design agent)
     author it — the design skill only does the design system; spec author ≠ implementer. It is
     RED now (the feature doesn't exist); `fp-impl` makes it green. Assert behavior ONLY — never
     pixels, colors, spacing or layout.
5. **Freeze in two ordered commits** (CONTRACT §Freeze gate):
   - **Freeze commit** — `git add DESIGN.md tokens.css references/ spec/`, commit. Record its hash
     as **`design-rev`**.
   - **Record commit** — set `current.json` `{stage: design, design-rev: <hash>}`, append your
     handoff to `journal.md`, `git add current.json journal.md`, commit once.
   Both commits are straight to trunk (metadata + the frozen design live on trunk).
6. **Print the handoff** to **fp-impl** (already journaled in step 5): repo, branch, artifact paths,
   `design-rev`, `do: read DESIGN.md + tokens.css + spec/, implement the frontend — make the frozen
   spec green, consume tokens.css (import or byte-identical copy), no raw color literals, never
   touch design-paths or spec-paths`.

## Hard rules

- Ask the human for the aesthetic direction; wait for answers; do not guess the vibe.
- One feature at a time. Write only `design-paths` (`DESIGN.md` + `tokens.css` + `references/*`) +
  `spec-paths` (`spec/*`) + `current.json` metadata. No src, no build config, no implementation.
- `tokens.css` MUST parse as valid CSS (verified, not eyeballed) and be consistent with `DESIGN.md`.
  `references/` MUST exist and its `*.png` MUST be captured from the frozen `preview.html` (the
  visual gate has no honest oracle otherwise).
- `spec/` MUST exist and be red-at-freeze; it asserts behavior (`data-*` hooks, interactions, a11y)
  and NEVER pixels/colors/spacing — aesthetics belongs to the visual gate, not the spec.
