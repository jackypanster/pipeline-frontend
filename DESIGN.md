# pipeline-frontend — design contract

A thin skill-aggregation pipeline for **frontend** development: design-first, code-second. The
only durable asset is this contract (command sequence + handoff format + git+md state convention +
the design-freeze gate). Each command is a ~20-line shim delegating to a swappable skill.
Forge-agnostic, machine-agnostic, human-relayed, no scheduler.

## The gate — design system freeze PLUS a behavioral red test

Correctness for a frontend surface splits in two. **Aesthetics** is visual and partly subjective:
no test deterministically answers "does this render match the intended aesthetic" — so no aesthetic
red test exists. **Behavior** (screens mount, `data-*` hooks exist, interactions work) IS
deterministically testable — the general pipeline's ADR-005 practice ("视觉不可冻 → 用稳定 data-*
钩子锁行为，视觉留 review 眼审") carries over unchanged. So `fp-design` freezes the **design
system** (`design-paths`: `DESIGN.md` + `tokens.css` + `references/*` incl. the regenerable
`preview.html` the reference shots are captured from) AND a minimal **behavioral red test**
(`spec-paths`: `spec/*` — red at freeze, impl makes it green, asserts behavior never pixels), and
the gate has four mandatory layers:

| layer | what | judged by | gate |
|---|---|---|---|
| staleness | branch carries the current freeze | machine | `design-rev` not ancestor of review-tip ⇒ route rebase (NOT a reject, attempts unchanged) |
| tamper | frozen paths untouched ON the branch | machine | `git diff $(git merge-base trunk tip) tip -- design-paths spec-paths` non-empty ⇒ reject |
| behavioral + token | spec green (run by path) · tokens.css consumed (import or byte-identical copy) · no raw color literals in src | machine | any failure ⇒ reject |
| visual | rendered UI matches the design | skill + human | `design`/`ui` skill screenshot-iteration + human confirm; post-rebaseline pixel diff as triage signal |

All must pass. The tamper half is a NEGATIVE check — untouched frozen files prove nothing about the
code (fake-green trap); the behavioral/token layer is the deterministic POSITIVE check; the visual
layer + human confirm judge adherence, which no machine can.

## Core principle: skills reason/generate, the shim does I/O

The design skill (`fp-design`) may author `DESIGN.md` / `tokens.css` inline — that IS its core
output, the one sanctioned exception. Every other skill reasons; the shim owns all I/O (writing,
staging, commit, journal, handoff, write-set enforcement).

Each command body (~20 lines): `git pull --rebase → read current.json + md → resolve skill via
roles.yaml → invoke skill → write your stage's write-set + append handoff to journal.md → commit
(one) → push → print handoff`.

## Core principle: stages are decoupled — each runs cold, the handoff is the only bridge

Any stage may be run by a **different agent on a different LLM in a fresh conversation with zero
shared memory**. The pipeline is explicitly built for this: there is no shared session, no DB, no
scheduler. The **only** thing that crosses a stage boundary is the **handoff block** — persisted to
`journal.md` (so it survives chat death) and printed for the operator to relay. Therefore a stage's
output must be a **complete, self-contained briefing** for a cold node: it has only `git pull` +
`CONTRACT.md` + your handoff. Point at artifact paths (git is the bus — never paste bodies), give
concrete numbered steps, name feature-specific gotchas, and err toward MORE next-step detail, not
less — a thin handoff makes a cold frontier bot guess wrong. `journal.md` (append-only) is the
source of truth: its tail entry IS the live position; `current.json` is only a fast cache. The
resume protocol for any cold start is `git pull --rebase` → read the journal tail → its handoff is
your briefing.

## Commands (MVP — 3 stages)

| command | delegates to (skill) | in → out |
|---|---|---|
| fp-design | design intelligence skill (generates a complete design system from a product brief) | brief → `DESIGN.md` + `tokens.css` + `references/*` (preview.html + shots) + `spec/*` (red) frozen on trunk (`design-rev`) |
| fp-impl | `<autonomous-coding-skill>` (think → code → check loop) | frozen design → working frontend code on `feat/<feature>` (spec green; zero design/spec-paths edits) |
| fp-review | `design`/`ui` skill (screenshot-iteration visual review) | diff + live render → review (freeze + behavioral/token + visual gates) + merge (only stage that merges) |

```yaml
# .pipeline/roles.yaml  (one per target repo; any line independently swappable to a best-of-breed skill)
design:  ui-ux-pro-max             # generates a complete design system (palette/type/spacing/components) from a product brief
impl:    <autonomous-coding-skill> # your runtime's real installed think→code→check skill
review:  design                    # the local design/ui skill's screenshot-iteration mode (visual review vs frozen refs)
```

`roles.yaml` names the skill only; which agent/bot you paste into selects the runtime. On init,
each command verifies its OWN slot resolves to an installed skill (hard gate — no silent mid-run
failure). **Brand names are install examples only** — never copy a specific tool name into the
onboarding snippet or the contract.

## Borrowed / rejected

**Borrowed:** the ~20-line shim shape; `journal.md` as authoritative tail; the handoff block; the
state machine; the forge adapter; propose-only self-improvement; the `DESIGN.md` scaffold standard
(open-sourced by Google Stitch, from `getdesign.md`); three-layer design-token architecture
(primitive→semantic→component); the general pipeline's ADR-005 hybrid — freeze behavior as a
headless red test (`data-*` hooks), leave aesthetics to skill+human review.
**Rejected:** an AESTHETIC red-test gate (wrong domain — aesthetics has no deterministic oracle;
the BEHAVIORAL red test is kept — behavior does have one); binding a
specific design-generation tool into the contract (Stitch / OpenDesign / Gemini AI Studio / Claude
Code's `design` skill are interchangeable slots, the contract names none); pixel diff as an
autonomous pass/fail oracle (adopted only as a post-rebaseline triage SIGNAL feeding the visual
review — DiffSpot, arxiv 2605.29615, shows VLM-only review misses fine-grained drift, so the
signal is warranted; auto-verdicts on pixels stay rejected as brittle); a mandatory online
design-tool stage (fails closed offline — the local `design_system` script generates
deterministically with zero network, so the MVP runs air-gapped).

## Constraints

No cron, no scheduler (human relays) · not coupled to any machine · LLM-agnostic (design/review
want a frontier model; impl tolerates a capable local LLM) · commands are extensible — a new verb
is a new ~20-line shim + one `roles.yaml` line + the prior command's handoff naming it.

## Open items (deferred — add when a real signal demands it)

- **`fp-task` (decompose into atomic cards)** — MVP freezes one design unit per feature. Add when a
  feature needs multiple independent impl cards with card-scoped review.
- **`fp-hunt` (root-cause a blocked card)** — MVP has no `blocked` path exercised. Add `fp-hunt`
  when a real impl/review deadlock appears; until then `attempts >= 3` stops and surfaces to the
  human.
- **Visual regression baseline (Playwright)** — RESOLVED, no longer open. The ratchet trigger
  ("a real miss signal") arrived as published evidence instead of a local failure: DiffSpot
  (arxiv 2605.29615) shows VLM review systematically misses fine-grained spacing/color/opacity
  drift. Adopted as a post-rebaseline triage signal feeding the visual review — never an
  auto pass/fail (CONTRACT §Visual gate).
- **`fp-freeze` (promote the first feature's design to a repo-level product.md)** — a one-off
  consolidation after feature 1 ships, so features 2+ inherit a stable house style. Not a pipeline
  stage; an opportunistic skill run when the design language is settled.
