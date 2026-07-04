---
name: fp-impl
description: "Frontend pipeline stage 2 — implement the frozen design as working frontend code: make the frozen behavioral spec green, consume the frozen tokens, de-hardcode, wire the real backend, open a PR. Writes ZERO design-paths/spec-paths. Use after fp-design. Args: repo, branch."
---

# fp-impl

Stage 2. Follow the **shim loop in CONTRACT.md** with slot = `impl`.

**Skill:** the `impl` slot runs an autonomous think→code→check loop (your runtime's autonomous
coding skill — Claude Code, Codex, etc.). The pipeline is runtime-agnostic: bind whichever
autonomous-coding skill your runtime provides in `roles.yaml`. It writes **`src` + build config +
white-box tests**; it must NOT create or edit anything under `design-paths` or `spec-paths`.
Constrain it accordingly.

## Steps

1. `git pull --rebase`. Read `current.json` (must have `design-rev` — else STOP, run `fp-design`).
2. Resolve `impl` slot; verify installed (else STOP). **Create or reconcile the feature branch**
   **`feat/<feature>`** (per CONTRACT §State authority — one branch per feature):
   - **New branch:** cut it from trunk (`main`) — it inherits the frozen design.
   - **Existing branch (design re-frozen mid-impl):** if trunk's `design-rev` advanced since the
     branch was cut, **rebase it onto trunk and force-push**
     (`git fetch origin && git rebase origin/main && git push --force-with-lease`) so it carries the
     current frozen design. This force-push is the **sanctioned exception** (your own in-flight
     branch, never trunk).
   Then flip `current.json.stage: in-progress` and commit it to **`main`** (trunk-authoritative
   metadata — a cold node must read the live state from trunk).
3. **implement**: read the frozen design (`DESIGN.md` + `tokens.css` + `references/`) AND the
   frozen spec (`spec/`) FIRST. Then code inside `src/**` + build config on `feat/<feature>` until:
   - the feature builds and **the frozen spec runs green by explicit path** (e.g.
     `npx vitest run .pipeline/<feature>/spec/` — wire the runner if the repo has none; review will
     run it the same way and never trusts the default test glob or your self-report);
   - `tokens.css` is **actually consumed**: import the frozen file (relative path or build alias),
     or keep a copy that stays **byte-identical** (review checks `git diff --no-index`);
   - src styles use `var(--*)` — **no raw color literals** (hex/rgb/hsl/oklch); review lints this;
   - the rendered UI matches the frozen design.
   - **honor DESIGN.md's normative §Anti-slop / §Accessibility / §Motion discipline sections** — the
     spec's a11y + motion-behavior subset (reduced-motion block, transform/opacity-only transitions,
     `aria-sort` / skip-link / `data-*` hooks) is part of "spec green"; the rest the visual + contrast
     gates check.
   De-hardcode placeholders, wire the real backend/API (base URL from `.env` per CONTRACT step 2).
   Loop think→code→check within the turn budget. Only code lives on the branch; **never touch
   `design-paths` or `spec-paths`** (DESIGN.md / tokens.css / references/ / spec/). A wrong or
   untestable spec is NOT yours to fix — re-route to `fp-design` naming the problem.
4. **Done** ⇒ push `feat/<feature>`, open/update a PR via the forge adapter, then on `main` flip
   `current.json.stage` to `impl` and **append your handoff to `journal.md`** — one commit on `main`.
   The handoff MUST name (review runs these verbatim and false-rejects a correct impl without them):
   - the **literal spec-run command** as you wired it (e.g. `npx vitest run .pipeline/<feature>/spec/`);
   - the **token consumption method + location** (the import path of the frozen `tokens.css`, or
     the path of the byte-identical copy).
   Opening the PR needs the repo's forge token (loaded per CONTRACT step 2). If the token is absent,
   **do NOT fail** — push the branch + make that same `main` commit anyway, and say in the handoff
   that the PR must be opened manually (branch + base named). Hand off to **fp-review**.
5. **Fail / budget exhausted** ⇒ on `main`: `attempts++`, decide by retry budget BEFORE committing:
   - **`attempts < 3`** ⇒ `stage` back to `design` (the pre-impl state), journal `status=failed`,
     next = **fp-impl** (informed retry — the `## Attempt N` note carries the failure context).
   - **`attempts >= 3`** ⇒ `stage: blocked` (terminal), journal `status=blocked`, STOP and surface
     to the human (no `fp-hunt` stage in this MVP).
   Commit the `current.json` + journal update to `main` in ONE commit, then print the handoff.

## Hard rules

- Never touch `design-paths` (`DESIGN.md` / `tokens.css` / `references/`) or `spec-paths`
  (`spec/`) — the freeze gate. Never merge. Only the feature's `src` + build config.
- Done requires the frozen spec green BY PATH (incl. its a11y + motion-behavior subset) + tokens
  consumed (import or byte-identical copy) + zero raw color literals in src styles + DESIGN.md's
  normative sections honored — review re-checks these deterministically (incl. design.md contrast lint).
- Code lives on `feat/<feature>`; `current.json` stage flips commit to `main` (trunk authority —
  never leave state stranded on the branch). If the frozen design or spec is wrong, do NOT edit it —
  re-route to `fp-design` to re-freeze (the handoff MUST name what to change).
