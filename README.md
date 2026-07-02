# pipeline-frontend

Agent-facing skill collection for **design-driven frontend development**. Consumers are LLM/agents,
not humans — read [CONTRACT.md](CONTRACT.md).

**What:** a forge-agnostic, machine-agnostic, LLM-agnostic frontend dev pipeline as 3 thin
command-skills over a git+md state bus. Human-relayed (no scheduler); each command prints a
handoff the operator copies to the next bot. The only durable asset is the orchestration contract;
the skill behind each command is a swappable `roles.yaml` slot.

Design-first: the pipeline freezes a design system (`DESIGN.md` + `tokens.css` + reference
screenshots) before any code, then gates implementation against it — a deterministic git diff
(design un-tampered) plus a visual review (render matches the design).

## Files

- `CONTRACT.md` — frozen protocol every command follows: shim loop · state machine · design-freeze
  gate · visual gate · handoff · forge adapter.
- `DESIGN.md` — design contract: core principle · commands · borrowed/rejected · constraints ·
  open items.
- `roles.yaml` — per-target-repo slot→skill bindings (copy into the target repo's `.pipeline/`).
- `skills/fp-*/SKILL.md` — the 3 command shims.

| command | slot → skill | in → out |
|---|---|---|
| fp-design | design-intelligence skill | product brief → `DESIGN.md` + `tokens.css` + `references/*` frozen (`design-rev`) |
| fp-impl | `<autonomous-coding-skill>` | frozen design → working frontend code + PR (zero design-paths edits) |
| fp-review | `design`/`ui` skill | diff + live render → visual review + merge (only stage that merges) |

## Onboard a target project (paste into its `AGENTS.md` / `CLAUDE.md`)

> **This project's frontend is developed via the `pipeline-frontend` toolchain — a forge-agnostic,
> machine-agnostic, LLM-agnostic agent dev pipeline whose only durable asset is a git+markdown
> state bus under `.pipeline/`. Any capable agent runs its commands — the pipeline is not bound to
> any tool, framework, agent or LLM.**
>
> **How it works.** Work flows through `fp-design → fp-impl → fp-review`. Each command is a
> ~20-line shim: `git pull --rebase` → read `.pipeline/current.json` + the feature's `journal.md`
> → resolve the stage's skill via `.pipeline/roles.yaml` → invoke that skill → write only its
> stage's write-set → append one entry to `.pipeline/<feature>/journal.md` → commit once → git push
> → print a self-contained handoff for the next (cold, possibly different-LLM) node. No shared
> memory, no scheduler, no DB: a human relays the handoff, any agent rebuilds state from
> `git pull` alone.
>
> **Design-first.** `fp-design` freezes the design system (`DESIGN.md` + `tokens.css` +
> `references/*.png`) to trunk as `design-rev` BEFORE any code. `fp-impl` produces code without
> touching `design-paths`. `fp-review` enforces two gates: `git diff <design-rev>..review-tip --
> <design-paths>` non-empty ⇒ reject (design tampered), AND a visual review comparing the live
> render against the frozen references.
>
> **Source of truth is `journal.md`** (append-only; last entry = live position). State machine:
> `todo → in-progress → review → done`. **Hard invariants:** only `fp-review` merges, only after
> explicit human confirmation; never edit frozen `design-paths` (re-route to `fp-design` to
> re-freeze); never force-push trunk; metadata on trunk, reviewed code on `feat/<feature>` via PR.
>
> **To act:** read `CONTRACT.md` in [`jackypanster/pipeline-frontend`](https://github.com/jackypanster/pipeline-frontend)
> first, then this repo's `.pipeline/<feature>/DESIGN.md` + `tokens.css` + `references/` + the
> journal tail. Do not hand-edit work out of band — run the stages.

## Install (agent: execute this)

You are an agent installing this collection on your runtime. Do it once, then verify dependencies.

```bash
# 1. READ-ONLY consumer clone (never edit skills here — see CONTRACT §Self-improvement).
git clone https://github.com/jackypanster/pipeline-frontend.git ~/workspace/pipeline-frontend
#    refresh: git -C ~/workspace/pipeline-frontend fetch && git reset --hard origin/main

# 2. Install the command shims into the runtime that will RUN them.
#    Runtime loading skills from a directory (e.g. ~/.claude/skills):
cp -r ~/workspace/pipeline-frontend/skills/fp-* ~/.claude/skills/
#    Or add "~/workspace/pipeline-frontend/skills" as an external_dirs list item in your config.

# 3. Per target project, point the slots at your chosen skills:
mkdir -p <target-repo>/.pipeline && cp ~/workspace/pipeline-frontend/roles.yaml <target-repo>/.pipeline/roles.yaml
```

### Verify + supplement dependencies (BEFORE running any command)

The commands delegate to the skills named in `roles.yaml`. Each must RESOLVE on the runtime that
runs its command. **Check every one; install any missing from its source:**

| delegated skill | used by | source |
|---|---|---|
| design-intelligence skill (e.g. `ui-ux-pro-max`) | design | `github.com/nextlevelbuilder/ui-ux-pro-max-skill` — `npx ui-ux-pro-max-cli init --ai claude --global`. Generates a complete design system from a product brief, locally, zero API key. Any skill that produces a frozen `DESIGN.md` + `tokens.css` is a valid slot. |
| `design`/`ui` (screenshot-iteration visual review) | review | your runtime's taste-engine skill — compares a live render against references and names concrete visual deltas |
| `<autonomous-coding-skill>` (impl slot) | impl | your runtime's autonomous think→code→check skill |

**Check procedure:** for each skill, confirm it loads on the runtime. Missing ⇒ install from its
source into that runtime's skill dir ⇒ re-check. **Cross-runtime trap:** a skill installed for one
runtime is NOT resolvable from another — install it where the command runs. Each command verifies
its OWN slot on init and STOPs if missing. Set each slot to the skill's real installed name, never a
bare/abstract token.

**Brand names are install examples only** — not part of the contract. Never copy a specific tool
name into the onboarding snippet or `roles.yaml`.

## State

Contract + 3 command shims (design/impl/review). Rationale: [DESIGN.md](DESIGN.md).
