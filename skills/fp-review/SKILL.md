---
name: fp-review
description: "Frontend pipeline stage 3 — review the implementation against the frozen design and merge. Runs staleness routing, the tamper gate (design/spec-paths untouched on the branch), the behavioral+token gate (frozen spec green, tokens consumed, no raw colors, WCAG contrast lint) AND the visual gate (render matches design), then merges after human confirm and rebaselines references. The ONLY stage that merges. Use after fp-impl. Args: repo, branch, pr."
---

# fp-review

Stage 3. Follow the **shim loop in CONTRACT.md** with slot = `review`.

**Skill:** the `review` slot resolves to a taste-engine skill (`design`/`ui`) that runs
screenshot-iteration visual review — comparing a live render against references and naming concrete
visual deltas. It REASONS; the shim owns the gate, merge, and handoff.

## Steps

1. `git pull --rebase`. Read `current.json` (must have `design-rev` and `stage: impl` — else STOP).
2. Resolve `review` slot; verify installed (else STOP).
3. **GATE 1 — staleness check** (routing, NOT a reject; CONTRACT §Freeze gate):
   - `git fetch origin`, then `git merge-base --is-ancestor <design-rev> <review-tip>`
     (review-tip = the PR head / `feat/<feature>` tip).
   - **Not an ancestor ⇒ the branch predates the current freeze** (trunk re-froze under it). Route
     to `fp-impl` to rebase onto trunk + force-push. `attempts` UNCHANGED — nobody failed; journal
     the routing. Never call this "tampered."
4. **GATE 2 — tamper gate** (deterministic; CONTRACT §Freeze gate):
   - `git diff $(git merge-base origin/<trunk> <review-tip>) <review-tip> -- <design-paths> <spec-paths>`
     — the branch against its OWN fork point, so only edits made on the branch count. Use the
     fetched `origin/<trunk>` (never a stale local trunk) and resolve the placeholders to FULL
     `.pipeline/<feature>/…` paths (bare filenames match nothing ⇒ silent false-pass).
   - **Non-empty diff ⇒ REJECT**: frozen paths were edited during impl. `attempts++`, flip the
     feature back (journal `status=failed`, route to `fp-impl`; or STOP+human at `attempts >= 3`).
     Write `reviews/review-NN.md` naming exactly what changed and that it must be reverted /
     re-frozen via `fp-design` if the change is intentional.
5. **GATE 3 — behavioral + token gate** (deterministic; CONTRACT §Behavioral & token gate). Run
   EVERY check below yourself — never trust impl's self-report (the list mirrors CONTRACT's four
   checks; a count would go stale, so run all of them, not a number):
   - **Spec green:** run the frozen spec **by explicit path, with the literal command the impl
     handoff names** — the runner is impl's build-config choice; guessing one false-rejects a
     correct impl. Never the repo's default test glob (impl owns build config and could exclude
     it). Red ⇒ REJECT. "Not runnable" ⇒ REJECT only after running impl's declared command; if the
     handoff names none, reconstruct from the journal/build config and note the gap.
   - **Tokens consumed:** src imports the frozen `tokens.css`, or carries a byte-identical copy
     (`git diff --no-index .pipeline/<feature>/tokens.css <copy>` empty) — the impl handoff names
     the method + location; verify there. Drifted copy ⇒ REJECT.
   - **Token adherence + motion discipline (src lint):** src styles reference `var(--*)` — no raw
     color literals (hex/rgb/hsl/oklch); prefer stylelint, a grep fallback matching declaration
     VALUES only (ID selectors like `#app`, SVG data, vendored files are not violations —
     sanity-check matches first). The SAME src-lint pass enforces DESIGN.md §Motion discipline:
     transitions/animations target `transform`/`opacity` only (no `top`/`left`/`width`/`height`) and a
     `prefers-reduced-motion` block is present. Both are source properties, checked HERE (the built
     source exists), not in the frozen spec. Violations ⇒ REJECT.
   - **Design-system lint (contrast — hard):** run `npx @google/design.md lint --format=json
     .pipeline/<feature>/DESIGN.md`. Contrast failures come back as `severity:"warning"` with exit 0,
     so parse `findings[]` and **REJECT on any `below WCAG AA` message** or `summary.errors > 0`
     (defends against a design frozen before this gate or a linter-version gap). **A pairless design
     emits no contrast finding and exits 0** — so also confirm the DESIGN.md declares component fg/bg
     pairs (`components.<name>` with both `backgroundColor` and `textColor`); none on a
     text-on-surface design ⇒ REJECT (the gate is vacuous otherwise). **Only if a previously shipped
     feature exists**, run `npx @google/design.md diff .pipeline/<shipped-feature>/DESIGN.md
     .pipeline/<feature>/DESIGN.md` and feed `regression: true` INTO the visual review as triage (a
     signal, never a verdict; skip on feature 1 — a missing baseline exits ENOENT). Linter
     unresolvable ⇒ STOP+human (never skip a hard gate).
   Any REJECT here: `attempts++`, journal `status=failed`, route `fp-impl` (or STOP+human at ≥3),
   findings in `reviews/review-NN.md`.
6. **GATE 4 — visual gate** (CONTRACT §Visual gate):
   - Build/run `feat/<feature>`. Capture real screenshots at **desktop** (primary — the aesthetic
     judgment) **and** each responsive breakpoint `DESIGN.md` declares (default phone ~375px, plus
     tablet ~768px if declared). Small-screen shots judge **adaptive correctness** (no overflow /
     clipping, content + actions reachable), not pixel-parity or aesthetics. Responsive is the norm
     (desktop-primary, adapts down); only a DESIGN.md that explicitly declares desktop-only is exempt.
   - Invoke the **`design`/`ui` skill** in screenshot-iteration mode: compare the live render
     against the frozen `references/*.png` + the rules in `DESIGN.md`/`tokens.css`. Name concrete
     visual deltas (color drift, spacing, typography, hierarchy, responsive breakage) — never vague
     "make it modern." Preserve the human's negative label when diagnostic.
   - **Pre-ship, references are design-intent, not pixel oracles** (they were captured from
     `preview.html`, not from an implementation): judge adherence to the DESIGN.md/tokens rules —
     never demand pixel equality against a pre-code render (that deadlocks the feature).
   - **Cross-feature pixel signal:** if the branch renders surfaces covered by a PREVIOUSLY
     SHIPPED feature's rebaselined references (`.pipeline/<shipped-feature>/references/*.png` —
     accepted-implementation shots, not preview-derived), additionally run a deterministic
     screenshot diff (Playwright snapshots or any pixel differ) vs those baselines and feed the
     boxed differences INTO your review as triage — regression protection for shipped surfaces; a
     signal, never an auto pass/fail. (A feature's own review never sees its own rebaseline — that
     lands at `done`.)
   - Write findings to `reviews/review-NN.md`. If deviations exceed acceptable polish ⇒ REJECT
     (`attempts++`, route `fp-impl`) with the specific deltas named. If it matches the frozen design
     ⇒ pass.
7. **Human confirm + merge** (the ONLY merge). Only after ALL gates pass AND the human explicitly
   confirms: squash-merge `feat/<feature>` via the forge adapter (`gh pr merge` / `gitee-cli` /
   `git merge`). Never merge without human confirm; never author product code (review only merges).
8. **Post-merge rebaseline + close** (CONTRACT §Visual gate). On `main`, in ONE commit: replace
   `references/*.png` with the ACCEPTED render's screenshots (the shipped implementation — not the
   mockup — is the durable baseline for regressions and follow-up features; keep `preview.html`),
   set `current.json.stage: done`, append the journal entry noting the rebaseline.
9. **Print the handoff**: feature shipped, next action (new feature via `fp-design`, or stop).

## Hard rules

- All gates mandatory, in order: staleness (route rebase, not a reject) → tamper diff over
  `design-paths` + `spec-paths` (non-empty ⇒ reject) → behavioral+token (spec green by path incl. the
  DOM-observable a11y/interaction subset, tokens consumed, no raw colors, motion source-discipline
  src lint, design.md lint clean / WCAG contrast — run every one yourself) → visual review
  (deviations ⇒ reject). None alone is sufficient. Motion is judged by its interaction hooks (spec)
  AND its source discipline (src lint: transform/opacity-only, `prefers-reduced-motion`); only motion
  AESTHETICS stays with the static-screenshot visual gate (scripted-interaction review deferred).
- Only `fp-review` merges, only after explicit human confirm. Never force-push trunk. Review writes
  only `reviews/*` + `references/*.png` (post-merge rebaseline ONLY) + `current.json`/`journal.md`
  metadata — never product code.
- A behavioral/token or visual rejection routes to `fp-impl`. A design-contract rejection (the
  frozen design or spec itself is wrong) re-routes to `fp-design` to re-freeze — name what changed
  in the handoff.
