# pipeline-frontend — design contract

A thin skill-aggregation pipeline for **frontend** development, driven by **design first, code
second**. The only durable asset is this contract (command sequence + handoff format + git+md
state convention + the design-freeze gate). Each command is a ~20-line shim delegating to a
swappable skill. Forge-agnostic, machine-agnostic, human-relayed, no scheduler.

Sibling of [`jackypanster/pipeline`](https://github.com/jackypanster/pipeline) (the backend
test-driven pipeline). Same architecture, different gate.

## Why a separate repo — the gate is fundamentally different

The backend pipeline's anti-cheat gate is a **deterministic git diff over a frozen red test**:
`pipeline-task` freezes a failing test, `pipeline-impl` makes it green without touching
`spec-paths`, `pipeline-review` diffs `spec-rev..review-tip -- spec-paths`. Correctness is a
machine-judged pass/fail. **Frontend correctness is not.** "Does this render match the intended
aesthetic?" is visual and partly subjective — no test answers it deterministically. Forcing both
into one CONTRACT would mean "which gate applies to which feature," a recipe for drift.

So this pipeline freezes the **design system** (`design-paths`: `DESIGN.md` + `tokens.css` +
`references/*.png`) instead of a red test, and splits the gate:

| half | what | judged by | gate |
|---|---|---|---|
| deterministic | design system not tampered during impl | machine | `git diff <design-rev> <review-tip> -- <design-paths>` non-empty ⇒ reject |
| visual | rendered UI matches the design | skill + human | `design`/`ui` skill screenshot-iteration + human confirm |

Shared with the backend (unchanged): the shim loop, `journal.md` as source of truth, the handoff
block, the state machine, the forge adapter, self-improvement (propose-only, review-gated). The
**only** rewrite is the freeze gate's protected object (red test → design system) and the addition
of the visual half.

## Core principle: skills reason/generate, the shim does I/O

The design-generation skill (`fp-design`) may author `DESIGN.md` / `tokens.css` inline — that IS
its core output, the one sanctioned exception. Every other skill reasons; the shim owns all I/O
(writing, staging, commit, journal, handoff, write-set enforcement).

Each command body (~20 lines): `git pull --rebase → read current.json + md → resolve skill via
roles.yaml → invoke skill → write your stage's write-set + append handoff to journal.md → commit
(one) → push → print handoff`.

## Commands (MVP — 3 stages)

| command | delegates to (skill) | in → out |
|---|---|---|
| fp-design | design intelligence skill (generates a complete design system from a product brief) | brief → `DESIGN.md` + `tokens.css` + `references/*` frozen on trunk (`design-rev`) |
| fp-impl | `<autonomous-coding-skill>` (think → code → check loop) | frozen design → working frontend code on `feat/<feature>` (zero design-paths edits) |
| fp-review | `design`/`ui` skill (screenshot-iteration visual review) | diff + live render → review (visual gate + freeze gate) + merge (only stage that merges) |

```yaml
# .pipeline/roles.yaml  (one per target repo; any line independently swappable to a best-of-breed skill)
design:  ui-ux-pro-max    # generates a complete design system (palette/type/spacing/components) from a product brief
impl:    <autonomous-coding-skill>   # set to your runtime's real installed think→code→check skill
review:  design           # the local design/ui skill's screenshot-iteration mode (visual review vs frozen refs)
```

`roles.yaml` names the skill only; which agent/bot you paste into selects the runtime. On init,
each command verifies its OWN slot resolves to an installed skill (hard gate — no silent mid-run
failure). **Brand names are install examples only** — never copy a specific tool name into the
onboarding snippet or the contract; both reach target projects and must stay tool-agnostic.

## Not in MVP (deferred — add when a real signal demands it)

- **`fp-task` (decompose into atomic cards)** — MVP freezes one design unit per feature. Add when a
  feature needs multiple independent impl cards with card-scoped review. The backend's multi-card
  machinery (card-scoped verify, full-suite final gate) would port over, with `verify` becoming a
  per-card visual + build check.
- **`fp-hunt` (root-cause a blocked card)** — MVP has no `blocked` path exercised yet (no impl
  loop). Add `fp-hunt` when a real impl/review deadlock appears; until then `attempts >= 3` stops
  and surfaces to the human.
- **Visual regression baseline (Playwright)** — the visual gate is skill + human today. If review
  repeatedly misses pixel regressions across iterations, add an optional Playwright snapshot
  baseline as a *secondary* deterministic signal (never a replacement for human confirm —
  aesthetics are not pixel-equality). Do not add pre-emptively: it is heavy and brittle, and the
  current skill-driven review has no recorded miss yet (ratchet: every line traces to a real failure).
- **`fp-freeze` (promote the first feature's design to a repo-level product.md)** — a one-off
  consolidation after feature 1 ships, so features 2+ inherit a stable house style. Not a pipeline
  stage; an opportunistic skill run when the operator decides the design language is settled.

## Borrowed / rejected

**Borrowed (from the backend pipeline):** the ~20-line shim shape; `journal.md` as authoritative
tail; the handoff block; the state machine; the forge adapter; the propose-only self-improvement.
**Borrowed (from the ecosystem):** the `DESIGN.md` scaffold standard (open-sourced by Google
Stitch, from `getdesign.md`); design-token architecture (three-layer primitive→semantic→component).
**Rejected:** shipping a red-test gate (wrong domain — aesthetics have no deterministic oracle);
binding a specific design-generation tool into the contract (tool-agnostic by design — Stitch /
OpenDesign / Gemini AI Studio / Claude Code's `design` skill are all interchangeable slots, the
contract names none); heavy visual-regression CI pre-emptively (brittle, added only on a real miss
signal); a mandatory online design-tool stage (fails closed offline — the local `design_system`
script generates deterministically with zero network, so the MVP runs air-gapped).

## Constraints

No cron, no scheduler (human relays) · not coupled to any machine · LLM-agnostic (design/review
want a frontier model; impl tolerates a capable local LLM) · commands are extensible — a new verb
is a new ~20-line shim + one `roles.yaml` line + the prior command's handoff naming it.
