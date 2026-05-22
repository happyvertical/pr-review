# pr-review

Pre-PR self-review prompt generator. Harness-agnostic: it emits a prompt + diff
bundle to stdout, and you pipe it to whichever LLM tool you prefer (codex-cli,
claude-cli, gemini, web chat, …).

Built to compress review iterations by front-loading the kind of analysis a
careful reviewer (or an AI reviewer like GitHub copilot-cli) does *after* you
open the PR — so the author can fix issues now instead of in a back-and-forth.

## Install

```bash
# From your repos root:
export PATH="$PWD/pr-review/bin:$PATH"

# Or symlink into your $PATH:
ln -s "$PWD/pr-review/bin/pr-review" ~/bin/pr-review
```

## Quick start

From inside any git repo:

```bash
# codex-cli (uses your ~/.codex/config.toml defaults: gpt-5.5 + xhigh)
pr-review | codex exec -

# Claude Code (one-shot)
claude -p "$(pr-review)"

# Gemini CLI
pr-review | gemini -

# Paste into a web chat
pr-review --pretty | pbcopy   # macOS
pr-review --pretty | xclip    # linux
```

Default base branch is auto-detected from `origin/HEAD`, falling back to
`origin/dev`, `origin/develop`, `origin/main`, `origin/master` in that order.
Override with `--base origin/main`.

## How it works

```
            ┌──────────────────────┐
            │ pr-review/prompts/   │  shared core checklist (10 themes
            │   checklist.md       │  mined from real PR review history)
            │   output-json.md     │
            │   output-pretty.md   │
            └──────────┬───────────┘
                       │
            ┌──────────▼───────────┐
            │ <repo>/.pr-review/   │  per-repo extensions:
            │   extensions.md      │  framework conventions, infra
            │                      │  patterns, tenant model, paths
            └──────────┬───────────┘
                       │
            ┌──────────▼───────────┐
            │ pr-review (script)   │  bundles checklist + extensions +
            │                      │  diff + repo metadata into a prompt
            └──────────┬───────────┘
                       │ stdout
                       ▼
              your harness of choice
```

The script *only* emits a prompt — it never calls an LLM directly. That keeps
it harness-agnostic and easy to maintain as CLIs evolve.

## Options

```
--base BRANCH         Base branch to diff against (default: auto-detect)
--repo DIR            Repo directory (default: $PWD)
--pretty              Emit markdown findings instead of JSON
--include-new-files   Embed full contents of newly-added files (<50KB)
--core-dir DIR        Path to shared core (default: discovered)
-h, --help            Show help
```

## Per-repo extensions

Each repo can add its own patterns in `<repo>/.pr-review/extensions.md`.
These are appended to the core checklist when reviewing that repo.

Good extensions cover:

- **Framework conventions** (e.g. SMRT's `0` vs `0.0`, Svelte 5 `$state`
  proxy equality, when to use `transformJSON` vs `toJSON`).
- **Project-specific paths** (`apps/dashboard/migrations/` vs the repo
  root `migrations/`, kustomize overlay layout).
- **Tenant model specifics** (which columns are tenant-scoped, which
  routes must check access level).
- **Known footguns** captured from past review comments.

See `anytown.ai/.pr-review/extensions.md` for a worked example.

## Output

Default is JSON (machine-readable, easy to post as PR comments later):

```json
{
  "summary": "Refactor extracting PortalChatTool from PortalAssistantSidebar...",
  "findings": [
    {
      "severity": "medium",
      "category": "refactor-regression",
      "file": "packages/site-portal/src/lib/components/PortalChatTool.svelte",
      "line": 1131,
      "title": "Dropped .primary CSS rule during component extraction",
      "body": "The previous component had `.primary, .composer button { ... }` but here only `.composer button` keeps the primary styling. The Approve button (line 771-782) now falls through to the generic button rule and looks identical to Reject. **Fix:** add the `.primary` rule mirroring the dropped declaration.",
      "confidence": "high"
    }
  ],
  "skipped": []
}
```

Use `--pretty` for a human-readable markdown report instead.

## Workflow recipes

### Run before every push

```bash
# .git/hooks/pre-push (or via husky)
pr-review --pretty | tee /tmp/pr-review.md
read -p "Review and continue? [y/N] " ans
[[ "$ans" == "y" ]] || exit 1
```

### Run in a Claude Code slash command

Add to `~/.claude/skills/pr-review/skill.md` or a project-level skill — it
runs the script and pipes through claude-cli's planning loop, surfacing findings
in your terminal.

### Iterate to convergence

```bash
# Round 1: get findings
pr-review --pretty | codex exec - > /tmp/round1.md

# Round 2: ask the same harness to apply the fixes you accept
codex exec "Apply the high+medium severity findings from /tmp/round1.md to the working tree"

# Round 3: re-run pr-review to confirm convergence
pr-review --pretty | codex exec -
```

When round-N produces no high/medium findings, you're done. Push.

### Ensemble review (multiple LLMs in parallel)

For high-stakes PRs, run the same `pr-review` prompt through several LLMs
in parallel and merge findings. Each model has different blind spots —
codex-cli catches things claude-cli misses and vice versa. copilot-cli adds a third
viewpoint that's tuned to GitHub's review style.

This is a recipe, not a script — pr-review stays emit-only on purpose so
you compose with whatever ensemble glue suits your harness.

```bash
PR_BASE=origin/dev
OUT=/tmp/pr-review-$(date +%s)
mkdir -p "$OUT"

# Capture each tool's findings to separate files
pr-review --base "$PR_BASE"            | codex exec -        > "$OUT/codex.json" 2> "$OUT/codex.err" &
pr-review --base "$PR_BASE"            | claude -p           > "$OUT/claude.json" 2> "$OUT/claude.err" &
pr-review --base "$PR_BASE" --no-diff  | codex review --base "$PR_BASE" - > "$OUT/codex-review.txt" 2> "$OUT/codex-review.err" &
gh copilot suggest -t shell "$(pr-review --base "$PR_BASE" --pretty)" > "$OUT/copilot.txt" 2> "$OUT/copilot.err" &
wait

# Merge — ask one LLM to deduplicate and rank
cat "$OUT"/*.json "$OUT"/*.txt | claude -p "Merge these review findings into one prioritized checklist. Dedupe near-duplicates. Order by severity desc, then by file. Drop findings that fewer than 2 reviewers flagged unless severity is high." > "$OUT/merged.md"

less "$OUT/merged.md"
```

**Operational notes:**

- **Allow at least 15 minutes per reviewer.** Some tools time out at 2-5
  minutes by default — bump it. Slow reviews are usually thorough
  reviews; fast ones often skim.
- **Capture stdout and stderr separately.** When a tool returns no
  findings or a malformed result, the cause is almost always in stderr.
  Saving them apart makes triage trivial.
- **Use `--no-diff` for `codex review`.** It fetches its own diff from
  `--base`. Sending the diff inside the prompt too just confuses the
  model about which diff to review.
- **Treat each tool's output as evidence, not as orders.** Verify
  before fixing. Tools hallucinate file paths and line numbers more
  often when temperature is high — concrete file:line references with
  matching context are reliable, vague claims are not.
- **Don't make a downstream PR draft solely because** one of the
  reviewers raised a low-confidence concern. Address the concrete
  findings, document the dismissed ones in the PR body, ship.

For a real-world example of this pattern wired into a shipping pipeline,
see the `/review-cycle` and `/ship` slash commands at
https://github.com/willgriffin/personal-ship-command (or whatever
ensemble harness you prefer).

## The calibration loop

The checklist will rot if you don't close the feedback loop. `pr-review-tune`
compares pre-PR findings against what reviewers actually said on the merged
PR, and proposes specific edits to `checklist.md` / `extensions.md` as a
unified diff.

### Setup: capture pre-PR findings

```bash
# Run as part of your pre-push workflow
pr-review | codex exec - | pr-review-capture | tee /dev/tty
#                          ^^^^^^^^^^^^^^^^^
#                          saves to .pr-review/history/<sha>.json
```

`.pr-review/history/` is gitignored by default — findings stay local. (If
you want shared visibility across collaborators, you can post them as a PR
comment via `gh pr comment` in a second step; not built in by design.)

### Tune: after some PRs have merged

```bash
# Tune against the last 10 merged PRs
pr-review-tune --last 10 | codex exec - > tune-proposal.md

# Or a specific PR
pr-review-tune --pr 422 | claude -p

# Or everything since a date
pr-review-tune --since 2026-04-01 | codex exec - > tune-proposal.md
```

The tune prompt enforces three rules to prevent prompt rot:

1. **N-occurrence threshold** (default 3): a missed pattern must appear in
   ≥N PRs in the batch before it can be proposed as a new checklist item.
   One-off quirks won't rewrite the prompt.
2. **Consolidation preferred**: existing sections get strengthened rather
   than the checklist growing unboundedly.
3. **Removal proposals**: items that fire repeatedly with zero reviewer
   agreement get flagged for weakening or removal.
4. **Token budget** (default ~8000 tokens for the checklist): forces
   aggressive consolidation when near the cap.

The output is a unified diff. Review it, apply with `patch -p0`, commit:

```bash
codex exec - < tune-proposal.md > diff.patch    # or however your harness emits
patch -p0 < diff.patch                          # apply the proposed edits
git diff prompts/ .pr-review/extensions.md      # see what changed
# decide what to keep, commit if good
```

### Cadence

- **Per-PR calibration** is noisy; don't run after every PR.
- **Every ~10 PRs** is a good cadence for `--last 10`. Apply edits
  manually for the first 2-3 rounds so you can see whether the tune
  prompt is drifting. Automate later if it isn't.
- **Monthly coverage health check**: run `--since` over a longer window
  and ask the harness specifically about *categories of bugs* reviewers
  still find that the checklist doesn't address. That surfaces blind
  spots that the per-PR loop won't.

### What the tune loop can't fix

The loop converges toward "what reviewers caught that pr-review didn't."
It does not catch what *neither* caught:

- Business-logic intent (does the PR do the right thing?)
- Cross-PR design questions
- Bugs both pr-review and copilot-cli share blind spots on

Keep a human in the loop periodically. The tune doesn't replace review;
it shortens iterations on the review you'd do anyway.

## What this catches

Based on analysis of 110 review comments across 36 anytown.ai PRs:

| Theme | % of comments | Catchable pre-PR? |
|---|---|---|
| Refactor regressions (dropped CSS/handlers/fallbacks) | ~20% | Yes — diff-aware |
| Tenant isolation gaps | ~15% | Yes — pattern-based |
| Svelte 5 effects/onMount races | ~10% | Yes — checklist |
| Silent error swallowing | ~10% | Yes — grep-able |
| Hardcoded values | ~10% | Yes — grep + checklist |
| Doc-code drift | ~10% | Partial — depends on doc proximity |
| Infra duplication / RBAC scope | ~8% | Yes — checklist |
| Dead config / unwired params | ~6% | Partial — needs call-site analysis |
| Concurrency on shared rows | ~5% | Yes — checklist |
| Pure mechanical (separators, dupe imports) | ~6% | Leave to linter |

Realistic compression target: 50–70% of round-2 review cycles eliminated.
A/B test it: count the findings copilot-cli adds *on top of* what pr-review
already caught. If that delta drops below half of baseline, it's working.

## License

MIT.
