# CONTRACT.md — the frozen frontend pipeline protocol

Single source of truth for every `fp-*` skill. Commands carry **no logic of their own** —
they follow this. See [DESIGN.md](DESIGN.md) for the design contract (core principle · commands ·
borrowed/rejected · constraints · open items).

## What this freezes — the design system AND a behavioral spec

Frontend correctness has two natures. **Aesthetics** is visual and partly subjective — no test
deterministically judges "does this render match the intended aesthetic," so there is no aesthetic
red test. **Behavior** (screens mount, `data-*` hooks exist, key interactions work) IS
deterministically testable and stays a frozen red test, exactly like the backend sibling's
spec-paths. So `fp-design` freezes BOTH:

- **`design-paths`** = `DESIGN.md` + `tokens.css` + `references/*` — the aesthetic contract.
  `references/` contains `preview.html` (a static, regenerable page rendering the frozen tokens +
  components) and the `*.png` reference screenshots **captured from it**.
- **`spec-paths`** = `spec/*` — a minimal headless behavioral red test (stable `data-*` hooks, key
  interactions, basic a11y). **RED at freeze** (the feature doesn't exist yet; on a greenfield repo
  it may not even run) — `fp-impl` makes it green and never edits it. It asserts behavior ONLY —
  never pixels, colors, spacing or layout (aesthetics has no oracle; that is the visual gate's job).

`fp-review` then runs four checks in order — 1–3 machine-judged, 4 skill+human:

1. **Staleness (routing, NOT a reject):** `git merge-base --is-ancestor <design-rev> <review-tip>`
   fails ⇒ the branch predates the current freeze ⇒ route to `fp-impl` to rebase onto trunk
   (`attempts` unchanged — nobody failed).
2. **Tamper:** `git diff $(git merge-base <trunk> <review-tip>) <review-tip> -- <design-paths> <spec-paths>`
   non-empty ⇒ the branch ITSELF edited frozen paths ⇒ reject.
3. **Behavioral + token adherence:** the frozen spec, run **by explicit path**, must be green; the
   frozen `tokens.css` must be actually consumed (imported, or a byte-identical copy); src styles
   must carry no raw color literals (see §Behavioral & token gate).
4. **Visual (skill + human):** the rendered UI matches the design — screenshot review against the
   frozen references, then a **human confirms** before the only merge.

All must pass. The deterministic checks protect both frozen contracts from silent edits and catch
behavioral/token drift the eye misses; the visual review + human confirm judge adherence. None
alone is sufficient.

## The shim loop (every command runs exactly this)

1. **`git pull --rebase`** — always your first act (no shared memory; rebuild from git).
2. **Load the repo's local config if present.** Most projects keep config/secrets in a
   dotenv-style file (`.env`, `.env.local`/`.envrc`). The delegated skill, the build, and any
   forge CLI may need it. If such a file exists, load its vars into the environment before step 5.
   Do NOT assume a fixed name or casing. **Never print secret values.**
3. **Read `.pipeline/current.json`** → `{repo, branch, pr?, feature, stage}`. Missing + you are
   `fp-design` ⇒ create it. Missing + any other command ⇒ STOP, ask the operator.
4. **Resolve your skill**: read `.pipeline/roles.yaml`, look up your slot. Verify the named skill
   is installed on this runtime. Not installed ⇒ STOP and report (no silent fallback).
5. **Invoke that skill** — it does the REASONING/generation; the shim owns all I/O (writing files,
   staging, committing). The **exception** is `fp-design`: the design-generation skill may author
   `DESIGN.md` / `tokens.css` inline as its core output — but staging/commit/journal/write-set
   enforcement stays the shim's.
6. **Write only within your stage's declared write-set** (see *State authority & write-sets*
   below), **append your composed handoff block as an entry to `.pipeline/<feature>/journal.md`**,
   `git add` those paths **+ `journal.md`**, commit **once** (the journal entry rides this same
   commit — never a separate/orphan commit, never an amend), **then `git push`** to the remote.
   The push is load-bearing: the next node is a COLD session that rebuilds all state from
   `git pull` and shares no memory with you. (Metadata commits straight to trunk as a fast-forward
   append; never force-push trunk.)
7. **Print the handoff block** (already persisted to the journal in step 6) and stop. The human
   relays it to the next bot.

## State machine (frozen — do not change)

`status: todo → in-progress → review → done`; `blocked` = terminal.
`attempts` starts at 0; on failure or review rejection `attempts++`; `attempts >= 3 ⇒ blocked`.
**Only `fp-review` merges**, and only after an explicit human confirm.
**Never force-push trunk or any shared/published ref; never delete anything beyond a feature's
own branch.** (An in-flight `feat/<feature>` branch is the coder's own — `fp-impl` MAY rebase it
onto trunk + force-push to absorb an advanced design re-freeze. That is standard PR hygiene on
your own branch, NOT a trunk force-push.)

One feature in flight at a time (the human serializes; `current.json` is a single pointer).

## Layout (`.pipeline/` lives in the TARGET repo)

```text
.pipeline/
  current.json            {repo, branch, pr?, feature, stage}  # fast pointer (cache — journal tail is authoritative)
  <feature>/
    journal.md            append-only run log — one entry per completed stage
    DESIGN.md             the frozen design system: palette / typography / spacing / components / motion
    tokens.css            CSS variables (the machine-diffable half of the freeze)
    references/           preview.html (regenerable render of the frozen system) + *.png captured from it — the visual oracle
    spec/                 frozen behavioral red test (data-* hooks / interactions / a11y) — red at freeze, impl makes it green
    arch.md  CONTEXT.md   optional component boundaries / domain glossary
    impl-notes.md         optional: what fp-impl changed, gotchas
    reviews/review-NN.md  fp-review findings
```

## State authority & write-sets

**Trunk is the single state authority.** All `.pipeline/` metadata (`current.json`, `DESIGN.md`,
`tokens.css`, `references/`, reviews) AND the frozen design system live on trunk (`main`/`master`),
committed straight there. `current.json` MUST be on trunk (the cold-node bootstrap pointer).

**A feature branch carries ONLY the reviewable code diff.** Name it `feat/<feature>`. `fp-impl`
cuts it from trunk, writes `src` there, opens the PR. `fp-review` squash-merges it (the only
merge).

**Reconcile an in-flight branch with advanced trunk design.** If trunk's frozen design advances
after the branch was cut — a **re-freeze** (new `design-rev`) — the existing `feat/<feature>`
carries the STALE design. `fp-impl` must **rebase `feat/<feature>` onto trunk + force-push** before
continuing, so the branch carries the current frozen design. An un-rebased branch is caught by
review's **staleness check** (design-rev not an ancestor of the tip) and routed back to `fp-impl`
to rebase — a routing outcome, `attempts` unchanged, never a false "tampered" reject.

| stage | write-set (may create/modify) | must NOT touch |
|---|---|---|
| design | `DESIGN.md`, `tokens.css`, `references/*`, `spec/*`, `CONTEXT.md` (optional) | src, build config |
| impl | `src/**`, impl-paths, build config, the feature's `status` field | **design-paths + spec-paths** (the freeze gate) |
| review | `reviews/*`, `references/*.png` (post-merge rebaseline ONLY), card `status`→done | any product code (it merges, never authors) |

**`design-paths`** = `DESIGN.md` + `tokens.css` + `references/*` (the frozen design system).
**`spec-paths`** = `spec/*` (the frozen behavioral red test). Together they are the freeze gate.
`(design-paths ∪ spec-paths) ∩ impl-paths = ∅` (`fp-design` asserts; `fp-review` re-checks).

Every stage additionally **appends one entry to `journal.md`** (append-only metadata, same class
as `current.json`).

## The freeze gate — design-paths + spec-paths, frozen on trunk

`fp-design` freezes the design system AND the behavioral spec in **two ordered commits**:

1. **Freeze commit** — write the complete `DESIGN.md` + `tokens.css` + `references/*` (incl.
   `preview.html` and the `*.png` captured from it) + `spec/*`, touching **only
   `design-paths` + `spec-paths`**, in **ONE** commit. Its hash = the feature's single
   **`design-rev`**, recorded in `current.json`. `tokens.css` must be valid CSS — **verify by
   parsing it** (stylelint or any CSS parser), never by eyeball; `DESIGN.md` must be internally
   consistent. The behavioral spec is authored by the **fp-design agent itself** (the delegated
   design skill only does the design system) — spec author ≠ implementer holds: design authors the
   red test, impl makes it green.
2. **Record commit** — advance `current.json.stage` to `design` and record `design-rev`. This
   commit touches **metadata only (current.json)**, never frozen paths — so the freeze stays
   intact.

Invariant: `(design-paths ∪ spec-paths) ∩ impl-paths = ∅`. `fp-impl` produces the frontend code via
`src` + `impl-paths` only, and must NOT create/modify/delete anything under `design-paths` or
`spec-paths`. `fp-review` runs the deterministic checks **FIRST**, in this order (review-tip = PR
head / `feat/<feature>` tip, after `git fetch`):

1. **Staleness (routing):** `git merge-base --is-ancestor <design-rev> <review-tip>` — if the
   design-rev is NOT an ancestor of the tip, the branch predates the current freeze. This is NOT a
   reject: route to `fp-impl` to rebase onto trunk + force-push, `attempts` unchanged.
2. **Tamper (reject):** `git diff $(git merge-base <trunk> <review-tip>) <review-tip> -- <design-paths> <spec-paths>`
   — diffs the branch against its OWN fork point, so only edits made ON the branch count.
   **Non-empty ⇒ reject** (`attempts++`, route to impl, or blocked at ≥3).

If the design itself is wrong, that is NOT an impl fix — re-route to `fp-design` to re-freeze (a
new single commit, new `design-rev`; the re-route handoff MUST name what changed). Git-only, no CI.

**Re-freeze** updates **only `design-rev`** (and the frozen files in the new freeze commit) and is
NOT initial authoring, so it never resets `attempts`. A re-freeze advances trunk's design under
any in-flight branch, so the handoff to impl must say **rebase `feat/<feature>` onto trunk +
force-push** before continuing (the staleness check enforces this at review time).

## The behavioral & token gate — fp-review's deterministic positive half

The freeze gate is a **negative** check (frozen files untouched). Untouched frozen files prove
nothing about the code — impl can leave every frozen path intact and still ship a UI that ignores
them (the fake-green trap). So after the freeze gate, `fp-review` runs three **positive**
deterministic checks; any failure ⇒ reject (`attempts++`, route impl):

1. **Behavioral spec green.** Run the frozen spec **by explicit path** (e.g.
   `npx vitest run .pipeline/<feature>/spec/`) against the branch — never trust the repo's default
   test glob (impl owns build config and could exclude it), never trust impl's self-report.
   Red or not runnable ⇒ reject.
2. **Tokens consumed.** The frozen `tokens.css` must actually reach the build: src imports the
   frozen file directly (relative path or build alias), OR carries a copy that is **byte-identical**
   (`git diff --no-index .pipeline/<feature>/tokens.css <copy>` empty). A drifted copy ⇒ reject.
3. **Token adherence.** Src styles must reference `var(--*)` — no raw color literals
   (hex/`rgb()`/`hsl()`/`oklch()`) in src stylesheets or inline styles. Check with stylelint
   (e.g. `color-no-hex` + declaration-strict-value) or a grep sweep. Violations ⇒ reject.

These are the only places in a frontend flow where a real machine oracle exists — use them fully so
the visual gate spends human attention on aesthetics only.

## The visual gate — fp-review's subjective half

After the deterministic gates pass, `fp-review` runs the **visual review**:

1. Build/run the feature branch. Capture real screenshots of the implemented UI at desktop width
   **and** 375px mobile (and any breakpoint the design specifies).
2. Invoke the **`design`/`ui` skill** in screenshot-iteration mode: compare the live render against
   the frozen `references/*.png` + the rules in `DESIGN.md`/`tokens.css`. Name concrete visual
   deltas (color drift, spacing, typography, hierarchy, responsive breakage) — never vague "make it
   modern."
3. Write findings to `reviews/review-NN.md`. If the implementation deviates from the frozen design
   beyond acceptable polish, **reject** (`attempts++`, route impl) with the specific deltas named.
4. Only when the visual review passes AND the human explicitly confirms does `fp-review` merge.

**Reference provenance — pre-ship vs post-ship.** Before the feature first ships, `references/*.png`
are captured from the frozen `preview.html` — they are **design-intent references, not pixel
oracles**: a real implementation never pixel-matches a pre-code render (fonts, real data, browser
differences). Judge adherence to the DESIGN.md/tokens rules the references embody, never pixel
equality — demanding pixel equality against a mockup deadlocks the feature into `blocked`.

**Post-merge rebaseline.** Immediately after the human-confirmed merge, capture the ACCEPTED
render's screenshots and commit them to `references/` (replacing the preview-derived shots) in the
post-merge metadata commit; journal the rebaseline. The accepted implementation — not the mockup —
is the durable visual baseline for regressions and follow-up features.

**Secondary pixel signal (post-rebaseline only).** From the first rebaseline onward, additionally
run a deterministic screenshot diff (Playwright snapshots or any pixel differ) of the live render
vs the accepted baselines, and feed the boxed differences INTO the visual review as triage input.
Rationale: VLM-only review systematically misses fine-grained spacing/color/opacity drift
(DiffSpot, arxiv 2605.29615) — the pixel diff catches what the eye and the model miss. It is a
signal, **never** an auto pass/fail: the skill + human still judge. Pre-rebaseline it is
meaningless (diffing against a mockup) — do not run it.

There is no automated pass/fail for aesthetics. The deterministic gates guarantee the contracts
were not silently re-authored and the behavior/tokens are real; the visual gate + human confirm
judge adherence. **All are mandatory.**

## Handoff block — a self-contained briefing for a COLD next node

**The next node is a FRESH session — possibly a different agent on a different frontier LLM — with
ZERO prior context.** It has only: this repo (via `git pull`), `CONTRACT.md`, and your handoff. So
the handoff must carry **everything it needs to ACT, not a one-liner.** Point at artifact paths (git
is the bus — never paste bodies), give **concrete numbered steps**, and name **feature-specific
gotchas.** A cold frontier bot with a thin handoff guesses wrong — err toward MORE next-step detail,
not less. Chat-friendly: plain text, short lines, no tables.

```text
>>> NEXT
Run fp-<next> on a FRESH session (assume you know nothing — rebuild from the repo + CONTRACT.md).
repo=<url> branch=<branch> pr=<url|none>
Model: <frontier SOTA required | capable-local OK (impl only)> — operator assigns the bot.
First: git pull --rebase; load repo config (.env if present, per CONTRACT step 2).
Read for context (before acting):
  - <repo>/AGENTS.md|CLAUDE.md (if present) — repo-wide conventions
  - <path/DESIGN.md>     — the frozen design system (palette / type / spacing / components / motion)
  - <path/tokens.css>    — CSS variables (the machine-diffable freeze)
  - <path/references/>   — preview.html + frozen reference screenshots — the visual oracle
  - <path/spec/>         — frozen behavioral red test — impl makes it green, never edits it
Your task (concrete, numbered):
  1. <step>
  2. <step>
Feature gotchas (project-specific traps):
  - <e.g. design uses OKLCH; tokens.css is the freeze — never edit it in impl; mock API base in .env>
Done when: <success criterion>. On success: <status transition>, then run fp-<after>.
On failure: attempts++; >=3 ⇒ blocked ⇒ run fp-hunt (TBD).
<<< END
```

## Run journal — the handoff, persisted (append-only)

**At step 6, append your handoff to `.pipeline/<feature>/journal.md`** as part of your stage's
metadata commit (one atomic commit — never separate/orphan, never amend); step 7 only prints what
is already journaled. This makes the run **resumable** (chat dies ⇒ read the tail), **auditable**
(the append sequence IS the run history), and **orchestratable**.

```text
## seq=N · <ISO8601 UTC> · <from-stage>→<to-stage> · <completed|failed|blocked> · by=<bot/LLM tag|?>
done:   <1–3 lines: what this stage actually produced>
output: <artifact path(s)>        # paths, never bodies — git is the bus
--- handoff ---
>>> NEXT
…(the exact handoff block you print, verbatim)…
<<< END
```

Rules: `seq` = per-feature monotonic integer (read the tail, add 1). **Append-only** — never
rewrite; a correction is a NEW entry. **One commit** (rides the metadata commit). **Tail is
authoritative** — the live position = the last entry's `to-stage` + its handoff; `current.json.stage`
is only a fast cache (on disagreement the journal tail wins). A `failed`/`blocked` stage still
appends an entry. **Resume protocol:** `git pull --rebase` → open `journal.md` → read the LAST entry
→ its handoff IS your briefing.

## Forge adapter (review/merge only)

Keyed on `git config --get remote.origin.url`:
- **github.com** → `gh` (`gh pr diff` / `gh pr merge`)
- **gitee** → `gitee-cli` against the instance's PR API
- **anything else / no forge** → `git fetch && git diff base..branch`

The PR is the **preferred** review surface; where no forge exists, `git diff base..branch` is the
sanctioned equivalent (same gates still hold). Merge is always human-confirmed. The pipeline
performs no destructive forge operations.

## Self-improvement — skills propose, review gates (never self-edit)

A running bot must **NEVER edit a live/installed skill in place** — it mutates the contract
mid-flight, untracked and ungated. A bot's pipeline clone is **read-only**:
`git fetch && git reset --hard origin/main` each run. When a run reveals a skill gap, emit a
proposal — `SKILL-PROPOSAL: <skill> — <what to change + why>` — which reaches `main` ONLY through
a gated PR (`fp-review` in meta-PR mode → human-confirm → merge). The frozen invariants (state
machine · only-reviewer-merges · the freeze gate · never-force-push) are not auto-improvable.
