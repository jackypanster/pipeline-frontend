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
     components, layout, depth, do/don't, responsive breakpoints (desktop-primary, adaptive DOWN to
     tablet & phone is the DEFAULT — declare the breakpoints and how data-dense components like tables
     / toolbars / nav reflow; a desktop-only design must say so explicitly), motion. (The `design`/`ui` skill's
     9-section DESIGN.md scaffold is the canonical shape.) **Regardless of which skill filled the
     slot, the DESIGN.md YOU freeze MUST additionally carry three NORMATIVE sections — bans and
     checklists only, never positive how-to (positive templates limit the generator; bans do not):**
     - **§Anti-slop** — banned defaults: Inter (use Geist/Outfit/Cabinet Grotesk/Satoshi or a
       brief-specific pairing), purple/indigo gradients, centered-hero + three-equal-column cards,
       section-number eyebrows, em-dash, decorative version/status pills and fake product UI, lazy
       `divide-y` long lists.
     - **§Accessibility** — required: skip-link, `:focus-visible` + focus-trap on modals, `aria-sort`
       on sortable tables, `tabular-nums` on numeric columns, form errors linked via
       `aria-describedby` + `aria-invalid` on invalid fields, `prefers-reduced-motion` honored,
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
     **desktop** capture (the primary target) plus one per responsive breakpoint DESIGN.md declares
     (default a phone width; add tablet if declared). These are the **visual oracle** `fp-review`
     compares the live render against, pre-ship.
   - `spec/*` — a minimal headless **behavioral red test** (CONTRACT §What this freezes): stable
     `data-*` hooks per screen/component, key interactions, and the deterministic a11y + interaction
     subset — **assert through the rendered PUBLIC SURFACE only** (query the DOM; never parse src
     files or assume a style-source shape — impl owns that, and a spec pinned to a guessed source
     shape false-rejects a valid impl): each named interaction's `data-*` hook toggles (e.g. bad
     input ⇒ `[data-invalid]`); a skip-link anchor exists; sortable tables carry `aria-sort`; numeric
     cells carry the tabular-nums hook. (The source-level motion discipline — transform/opacity-only,
     a `prefers-reduced-motion` block — is NOT a spec assertion; it assumes a source shape that
     doesn't exist yet, so `fp-review` lints it against the built artifact instead.)
     YOU (the fp-design agent)
     author it — the design skill only does the design system; spec author ≠ implementer. It is
     RED now (the feature doesn't exist); `fp-impl` makes it green. Assert behavior ONLY — never
     pixels, colors, spacing or layout. Assert through the app's **public surface** only: render
     the root / visit a route and query `data-*` hooks. NEVER deep-import internal component
     modules — module layout belongs to impl, and a spec pinned to guessed import paths forces impl
     to contort `src` around your guesses or loop back here (the likeliest self-inflicted
     deadlock).
5. **Freeze in two ordered commits** (CONTRACT §Freeze gate):
   - **Contrast hard gate (before freezing)** — run `npx @google/design.md lint --format=json
     .pipeline/<feature>/DESIGN.md`. **The tool emits a failing AA pair as `severity:"warning"` and
     exits 0**, so DON'T trust exit code/severity: parse `findings[]` and **STOP if any message
     contains `below WCAG AA`** or `summary.errors > 0` — fix the palette/tokens, do NOT freeze. The
     check is only real if the DESIGN.md declares component fg/bg pairs (`components.<name>` with both
     `backgroundColor` and `textColor`) — the lint output carries NO pair count, so verify the pairs
     by READING the DESIGN.md; a text-on-surface design with none declared is incomplete, fix before
     freezing. Linter unresolvable ⇒ STOP and ask the operator (never freeze past an unrun hard gate).
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
  discipline — bans and checklists only, no positive how-to), MUST declare component fg/bg pairs so
  contrast is actually checked, and MUST pass `design.md lint` (no `below WCAG AA` finding, no
  structural errors — contrast is emitted as a `warning` with exit 0, so gate on the message, never
  the exit code) BEFORE freezing (step 5).
- `spec/` MUST exist and MUST NOT pass at freeze: where a runner exists, run it and confirm red;
  on a greenfield repo with no runner yet, unrunnable is acceptable and expected. A spec that
  PASSES with no implementation is broken — fix it before freezing. It asserts behavior (`data-*`
  hooks, interactions, a11y) via the app's public surface, and NEVER pixels/colors/spacing —
  aesthetics belongs to the visual gate, not the spec.
