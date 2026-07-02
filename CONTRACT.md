# CONTRACT.md — the frozen frontend pipeline protocol

Single source of truth for every `fp-*` skill. Commands carry **no logic of their own** —
they follow this. See [DESIGN.md](DESIGN.md) for the design contract (core principle · commands ·
borrowed/rejected · constraints · open items).

## What this freezes — the design system, not a red test

Frontend correctness is visual and partly subjective: no test deterministically judges "does this
render match the intended aesthetic." So this pipeline freezes the **design system** (`design-paths`:
`DESIGN.md` + `tokens.css` + `references/*.png`) instead of a red test, and splits the gate into two
halves:

1. **Deterministic half (git diff, machine-judged):** the frozen design system must not be tampered
   with during implementation — `fp-review` runs
   `git diff <design-rev> <review-tip> -- <design-paths>`; non-empty ⇒ reject.
2. **Visual half (skill + human-judged):** does the implemented code actually render the design?
   `fp-review` runs a visual review (the `design`/`ui` skill's screenshot-iteration mode) against the
   frozen `references/*.png`, then a **human confirms** before the only merge.

Both halves must pass. The deterministic half protects the design contract from silent edits; the
visual half + human confirm judge adherence. Neither alone is sufficient.

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
    references/           frozen reference screenshots / mockups (*.png) — the visual oracle
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
continuing, so the branch carries the current frozen design. Otherwise review's freeze gate diffs
the new `design-rev` against a stale branch tip and **falsely rejects**.

| stage | write-set (may create/modify) | must NOT touch |
|---|---|---|
| design | `DESIGN.md`, `tokens.css`, `references/*`, `CONTEXT.md` (optional) | src, build config |
| impl | `src/**`, impl-paths, build config, the feature's `status` field | **design-paths** (the freeze gate) |
| review | `reviews/*`, card `status`→done | any product code (it merges, never authors) |

**`design-paths`** = `DESIGN.md` + `tokens.css` + `references/*` (the frozen design system). It is
the freeze gate. `design-paths ∩ impl-paths = ∅` (`fp-design` asserts; `fp-review` re-checks).

Every stage additionally **appends one entry to `journal.md`** (append-only metadata, same class
as `current.json`).

## The freeze gate — design-paths, frozen on trunk

`fp-design` freezes the design system in **two ordered commits**:

1. **Freeze commit** — write the complete `DESIGN.md` + `tokens.css` + `references/*.png`,
   touching **only `design-paths`**, in **ONE** commit. Its hash = the feature's single
   **`design-rev`**, recorded in `current.json`. `tokens.css` must compile (valid CSS); `DESIGN.md`
   must be internally consistent. `references/` carries the accepted screenshots/mockups that
   `fp-review` will visually compare against.
2. **Record commit** — advance `current.json.stage` to `design` and record `design-rev`. This
   commit touches **metadata only (current.json)**, never `design-paths` — so the freeze stays
   intact.

Invariant: `design-paths ∩ impl-paths = ∅`. `fp-impl` produces the frontend code via `src` +
`impl-paths` only, and must NOT create/modify/delete anything under `design-paths`. `fp-review`
runs the deterministic diff
`git diff <design-rev> <review-tip> -- <design-paths>` (review-tip = PR head) **FIRST**;
**non-empty ⇒ reject** (`attempts++`, route to impl, or hunt at ≥3). If the design itself is wrong,
that is NOT an impl fix — re-route to `fp-design` to re-freeze (a new single commit, new
`design-rev`; the re-route handoff MUST name what changed). Git-only, no CI.

**Re-freeze** updates **only `design-rev`** (and the design files in the new freeze commit) and is
NOT initial authoring, so it never resets `attempts`. Both re-freeze advance trunk's design under
any in-flight branch, so the handoff to impl must say **rebase `feat/<feature>` onto trunk +
force-push** before continuing.

## The visual gate — fp-review's subjective half

After the deterministic diff is empty (design un-tampered), `fp-review` runs the **visual review**:

1. Build/run the feature branch. Capture real screenshots of the implemented UI at desktop width
   **and** 375px mobile (and any breakpoint the design specifies).
2. Invoke the **`design`/`ui` skill** in screenshot-iteration mode: compare the live render against
   the frozen `references/*.png` + the rules in `DESIGN.md`/`tokens.css`. Name concrete visual
   deltas (color drift, spacing, typography, hierarchy, responsive breakage) — never vague "make it
   modern."
3. Write findings to `reviews/review-NN.md`. If the implementation deviates from the frozen design
   beyond acceptable polish, **reject** (`attempts++`, route impl) with the specific deltas named.
4. Only when the visual review passes AND the human explicitly confirms does `fp-review` merge.

There is no automated pass/fail for aesthetics. The deterministic gate guarantees the design was
not silently re-authored; the visual gate + human confirm judge adherence. **Both are mandatory.**

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
  - <path/references/>   — frozen reference screenshots — the visual oracle
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
