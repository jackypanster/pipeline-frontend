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
     9-section DESIGN.md scaffold is the canonical shape.) **Regardless of which skill filled the
     slot, the DESIGN.md YOU freeze MUST additionally carry three NORMATIVE sections — bans and
     checklists only, never positive how-to (positive templates limit the generator; bans do not):**
     - **§Anti-slop** — banned defaults: Inter (use Geist/Outfit/Cabinet Grotesk/Satoshi or a
       brief-specific pairing), purple/indigo gradients, centered-hero + three-equal-column cards,
       section-number eyebrows, em-dash, decorative version/status pills and fake product UI, lazy
       `divide-y` long lists.
     - **§Accessibility** — required: skip-link, `:focus-visible` + focus-trap on modals, `aria-sort`
       on sortable tables, `tabular-nums` on numeric columns, `prefers-reduced-motion` honored,
       semantic landmarks + labels.
     - **§Motion discipline** — transitions animate **transform/opacity ONLY** (never
       top/left/width/height); a `prefers-reduced-motion` media query is mandatory; every named
       interaction motion (e.g. invalid→shake, interruptible toggle, fast-out/slow-in toast) carries
       a `data-*` hook.
     Declare foreground/background color PAIRINGS in the design.md component-entry form so
     `design.md lint` can compute contrast (no declared pairs ⇒ zero contrast findings ⇒ the hard
     gate silently passes — see step 5).
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
     `data-*` hooks per screen/component, key interactions, and the deterministic a11y + motion
     subset (behavior/structure ONLY, never pixels): a `@media (prefers-reduced-motion)` block
     exists; no src transition/animation targets top/left/width/height (parse the static CSS); each
     named interaction's `data-*` hook toggles (e.g. bad input ⇒ `[data-invalid]`); a skip-link
     anchor exists; sortable tables carry `aria-sort`; numeric cells carry the tabular-nums hook.
     YOU (the fp-design agent)
     author it — the design skill only does the design system; spec author ≠ implementer. It is
     RED now (the feature doesn't exist); `fp-impl` makes it green. Assert behavior ONLY — never
     pixels, colors, spacing or layout. Assert through the app's **public surface** only: render
     the root / visit a route and query `data-*` hooks. NEVER deep-import internal component
     modules — module layout belongs to impl, and a spec pinned to guessed import paths forces impl
     to contort `src` around your guesses or loop back here (the likeliest self-inflicted
     deadlock).
5. **Freeze in two ordered commits** (CONTRACT §Freeze gate):
   - **Contrast hard gate (before freezing)** — run `npx @google/design.md lint
     .pipeline/<feature>/DESIGN.md`. Any WCAG-AA `contrast-ratio` error / `broken-ref` /
     `orphaned-tokens` ⇒ **STOP, fix the palette/tokens, do NOT freeze**. Also assert the lint output
     actually CONTAINS contrast checks (non-zero) — zero means the fg/bg pairings weren't declared
     and the gate is fake; fix the DESIGN.md pairings and re-run. Linter unresolvable ⇒ STOP and ask
     the operator (never freeze past an unrun hard gate).
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
- `DESIGN.md` MUST carry the three normative sections (§Anti-slop / §Accessibility / §Motion
  discipline — bans and checklists only, no positive how-to) and MUST pass `design.md lint` (WCAG
  contrast — hard) with a non-zero contrast-check count BEFORE freezing (step 5).
- `spec/` MUST exist and MUST NOT pass at freeze: where a runner exists, run it and confirm red;
  on a greenfield repo with no runner yet, unrunnable is acceptable and expected. A spec that
  PASSES with no implementation is broken — fix it before freezing. It asserts behavior (`data-*`
  hooks, interactions, a11y) via the app's public surface, and NEVER pixels/colors/spacing —
  aesthetics belongs to the visual gate, not the spec.
