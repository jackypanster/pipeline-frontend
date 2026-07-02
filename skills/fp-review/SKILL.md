---
name: fp-review
description: "Frontend pipeline stage 3 — review the implementation against the frozen design and merge. Runs the deterministic freeze gate (design-paths un-tampered) AND the visual gate (render matches design), then merges after human confirm. The ONLY stage that merges. Use after fp-impl. Args: repo, branch, pr."
---

# fp-review

Stage 3. Follow the **shim loop in CONTRACT.md** with slot = `review`.

**Skill:** the `review` slot resolves to a taste-engine skill (`design`/`ui`) that runs
screenshot-iteration visual review — comparing a live render against references and naming concrete
visual deltas. It REASONS; the shim owns the gate, merge, and handoff.

## Steps

1. `git pull --rebase`. Read `current.json` (must have `design-rev` and `stage: impl` — else STOP).
2. Resolve `review` slot; verify installed (else STOP).
3. **GATE 1 — deterministic freeze gate** (run FIRST; CONTRACT §Freeze gate):
   - `git fetch origin && git diff <design-rev> <review-tip> -- <design-paths>`
     (review-tip = the PR head / `feat/<feature>` tip).
   - **Non-empty diff ⇒ REJECT**: the frozen design was edited during impl. `attempts++`, flip the
     feature back (journal `status=failed`, route to `fp-impl`; or STOP+human at `attempts >= 3`).
     Write `reviews/review-NN.md` naming exactly what `design-paths` changed and that it must be
     reverted / re-frozen via `fp-design` if the change is intentional.
   - Empty diff ⇒ proceed to Gate 2 (the design contract is intact).
4. **GATE 2 — visual gate** (CONTRACT §Visual gate):
   - Build/run `feat/<feature>`. Capture real screenshots at desktop width **and** 375px mobile (and
     any breakpoint `DESIGN.md` specifies).
   - Invoke the **`design`/`ui` skill** in screenshot-iteration mode: compare the live render
     against the frozen `references/*.png` + the rules in `DESIGN.md`/`tokens.css`. Name concrete
     visual deltas (color drift, spacing, typography, hierarchy, responsive breakage) — never vague
     "make it modern." Preserve the human's negative label when diagnostic.
   - Write findings to `reviews/review-NN.md`. If deviations exceed acceptable polish ⇒ REJECT
     (`attempts++`, route `fp-impl`) with the specific deltas named. If it matches the frozen design
     ⇒ pass.
5. **Human confirm + merge** (the ONLY merge). Only after BOTH gates pass AND the human explicitly
   confirms: squash-merge `feat/<feature>` via the forge adapter (`gh pr merge` / `gitee-cli` /
   `git merge`). Then on `main`: `current.json.stage: done`, append the journal entry (one commit).
   Never merge without human confirm; never author product code (review only merges).
6. **Print the handoff**: feature shipped, next action (new feature via `fp-design`, or stop).

## Hard rules

- Two gates, both mandatory: deterministic `design-paths` diff (non-empty ⇒ reject) THEN visual
  review (deviations ⇒ reject). Neither alone is sufficient.
- Only `fp-review` merges, only after explicit human confirm. Never force-push trunk. Review writes
  only `reviews/*` + `current.json`/`journal.md` metadata — never product code.
- A visual rejection routes to `fp-impl`. A design-contract rejection (the frozen design itself is
  wrong) re-routes to `fp-design` to re-freeze — name what changed in the handoff.
