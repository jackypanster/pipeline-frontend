# CLAUDE.md — working ON pipeline-frontend

Conventions for an agent editing THIS repo (the pipeline itself). To RUN the pipeline against a
target project, read `CONTRACT.md` instead. Artifacts here are agent-first: encode decisions as
bans / checklists / gates, never human narrative.

## How changes land
- This is the SOURCE repo. Changes reach `main` via a gated meta-PR: branch → review → **explicit
  human confirm → squash-merge**. Never self-edit a live/installed skill in place, never force-push
  trunk (`CONTRACT.md §Self-improvement`, §State machine).

## Design decisions of record
- **Target scope (the default surface).** This pipeline builds web frontends for backend services
  (REST / APIs / CLIs) — mostly dashboards, admin panels, dev/ops consoles: data-dense,
  **desktop-primary but responsive** down to tablet/phone, cross-project, cross-platform (Mac/Linux).
  It's a default assumption, not a hard wall — a feature may narrow it in its own DESIGN.md. See
  `DESIGN.md §Constraints`.
- **The design skill is a swappable slot, not a bound tool.** `roles.yaml` `design:` is the abstract
  `<design-system-skill>` placeholder + a per-surface heuristic — never a brand name
  (`DESIGN.md §Choosing the design slot`). Rationale: a public 5-skill community benchmark
  (frontend-design / web-design-guidelines / ui-ux-pro-max / taste-skill / emil-design-eng) showed no
  single generator wins all four quality axes (visual character / anti-slop / a11y / motion), and the
  weakest one had been the canonical default. Given the scope above, the **common** surface is
  data-dense/back-office, so the **working default is a11y-first / low-variance** — taste-skill-class
  generators self-de-scope dashboards/data-tables and fit only the rare landing/marketing surface.
- **Three of the four axes are baked into the frozen DESIGN.md template, not the slot.** `fp-design`
  authors normative `§Anti-slop` (bans), `§Accessibility` (checklist), `§Motion discipline` — so they
  hold regardless of which generator fills the slot. Only the aesthetic direction is a per-feature
  slot choice. Bake bans/checklists, never positive how-to (positive templates limit the generator).
- **Contrast is a hard gate via `@google/design.md` lint — pinned to the tool's OBSERVED behavior.**
  v0.3.0 reports a failing WCAG-AA pair as `severity:"warning"` and **exits 0**, so the gate parses
  the `below WCAG AA` message (never the exit code / error-severity) and first asserts the DESIGN.md
  declares component fg/bg pairs (a pairless design emits no finding and is uncheckable).
- **Spec asserts behavior via the public surface; `fp-review` src-lints source properties.** Motion
  source-discipline (transform/opacity-only, `prefers-reduced-motion`) and contrast live in the
  review src-lint against the built artifact — never in the frozen spec, which must not assume a
  source shape impl owns.

## Reusable lesson
When a gate depends on an external CLI, **run the tool and pin the gate to observed output** (severity
strings, exit codes, field names) — never to assumed or from-docs behavior. This repo's contrast gate
shipped as a silent no-op in first draft (assumed "error severity / non-zero exit") and only became
real after `@google/design.md@0.3.0` was actually run. Three review rounds + running it caught it.
