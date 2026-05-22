## How to review

Before you write any findings, do this:

1. **Read the commit messages and the diff once end-to-end** to form a mental
   model of what this PR is trying to accomplish.
2. **Identify the *kind* of change**: refactor, new feature, bug fix, infra
   change, dependency bump, doc update. Different kinds have different
   failure modes (refactors lose behavior; new features have unreachable
   code paths; infra changes break ops).
3. **For each modified file, ask: "what existed here before, and what
   silently went away?"** Most regressions are deletions, not additions.
4. **Run the checklist below**, but only emit findings you'd actually defend
   in a review. False positives waste the author's time and erode trust.

Confidence calibration: a finding is **high-confidence** when you can point
at the exact lines and explain the failure path concretely. It is
**low-confidence** when you're inferring possible behavior from context. Mark
low-confidence findings as such — don't drop them, but make the uncertainty
explicit.

**Findings are evidence, not orders.** The author will verify each finding
against the code before fixing. Your job is to give them accurate, citable
evidence — exact file paths, line numbers, and the concrete failure path.
Vague claims ("this could fail") get dismissed and erode trust in the next
round. Concrete claims ("when both sides are undefined, `!==` returns false,
the gate body doesn't run, request proceeds") get acted on.

## Checklist

### 1. Refactor regressions (the highest-frequency miss)

When the diff extracts code into a new component / file / function, or
splits one module into several, **diff the old against the new** and look
for things that silently disappeared:

- **Dropped CSS rules / style selectors** during component extraction.
  Example: a component extracted into a new file loses a `.primary` rule
  that styled an Approve button — the button now looks identical to its
  sibling Reject button.
- **Dropped event listeners or `onMount` calls** during a split.
- **Dropped error-handling branches**: an `else` after `if (response.ok)`,
  a `.catch()` clause, a fallback message, a `console.warn`.
- **Dropped fallbacks or default props** — a child that used to receive
  `currentPath` now doesn't.
- **Dropped behavior the user could observe**: confirmations, redirects,
  toast messages, validation steps.

For each such drop, ask: was it intentional (then it should be in the
commit message) or accidental (then it's a regression).

### 2. Tenant isolation & multi-tenant correctness

Any change touching a model with a `tenant_id` / `tenantId` column, or
calling a function that crosses tenant boundaries:

- **Equality checks against possibly-undefined tenant fields**: `if
  (content.tenantId !== context.tenantId) throw 403` is meant as an auth
  gate, but when both sides are `undefined` the comparison evaluates
  `false`, the gate body doesn't run, and the request is silently
  authorized for cross-tenant access. Restate as an affirmative
  assertion that requires both sides to be defined *before* comparing
  (e.g. `if (!content.tenantId || !context.tenantId ||
  content.tenantId !== context.tenantId) throw 403`).
- **Cross-tenant data leaks via missing filters**: an API listing call
  that post-filters in-memory but doesn't pass `tenantId` to the upstream
  query. Records lacking the filter field slip through.
- **Trust-on-first-use claiming**: code that writes the caller's tenant
  onto any row whose tenant column is null. First caller wins; later
  callers of the other tenant see "their" data attributed elsewhere.
- **Fallbacks to non-application namespaces**: e.g. reading a tenant id
  from a service-internal field when the application-namespace field is
  absent.
- **Tenant-scoped routes that only check session, not access level**:
  compare against sibling routes in the same PR — if one rejects
  `viewer`, the rest probably should too.

### 3. Concurrency, races, and effect ordering

- **Concurrent writes to the same row**: `Promise.all` over jobs that may
  target the same DB row. Last-write-wins on non-monotonic columns.
- **Mount-time races** in Svelte 5 / React: an `onMount` reconcile that
  competes with a `$useEffect` / `$effect` flush touching the same state.
  Initial state can clobber hydrated state depending on flush order.
- **`for…of` mutating the iterated array**: prepended entries aren't
  visited; later dedup/slice operates on the mutated array and may drop
  freshly-added items.
- **Effects whose dependencies trigger expensive side effects**: rebuilding
  a context object every render and passing identity-fresh values into a
  setter that re-triggers an HTTP fetch on each call.
- **Resource leaks on early return / throw**: a `ReadableStream` reader
  that throws without `reader.cancel()` leaves the underlying socket open.

### 4. Silent error swallowing

- **`fetch` responses gated only by `if (response.ok)` with no `else`**:
  failures show the user nothing. At minimum log; ideally surface to UI.
- **`|| true` / `catch {}` blocks that hide actionable errors**: e.g.
  `kubectl delete … || true` masks RBAC failures, not just NotFound.
  Match the specific error case you intend to ignore.
- **`try { … } catch { return null }` patterns where `null` means
  "missing"**: a 500 from upstream gets treated identically to a 404, so
  outage triggers a fallback path that may double-create resources.
- **Streams whose buffered output isn't surfaced when a command fails**.

### 5. Hardcoded values that shouldn't be hardcoded

- **Author/machine-specific absolute paths**: `/Users/<name>/...`,
  `/home/<name>/...`, `C:\Users\...`. These break for every other
  contributor and CI runner.
- **Production hostnames in test/dev fixtures**: a test runner whose base
  URL is fully configurable can be accidentally pointed at prod.
- **Hard-coded image registries / repo names** in deploy workflows: any
  rename downstream silently breaks.
- **Hard-coded ports** in dev wrappers that override environment overrides
  (`--port 47291` wins over `process.env.VITE_DEV_PORT`).
- **Hard-coded fallback dates** like `2024-01-01` that surface as
  plausible but wrong "Effective from" dates in the UI.
- **Hard-coded keys/secrets/test fixtures with weak crypto** (e.g.
  512-bit RSA keys that some CI configurations reject).

### 6. Documentation–code drift

For any PR that touches code referenced by docs *or* updates docs:

- **Doc claims a default that the code doesn't honor**: "Defaults to
  `true` for static builds" while `this.x = config.x ?? !config.y` makes
  it `true` for everyone.
- **Doc describes a workflow guarantee the code can't enforce**: e.g.
  "always waits for human review" when the merging token may have
  bypass-allowance.
- **Doc step is grammatically broken or ambiguous** after a recent edit.
- **Examples in docs that point at paths or commands the PR moved**.

### 7. Config hazards: dead, surprising, or over-active

Config that's missing a consumer is wasteful; config that has *invisible
side effects on consumers* is the inverse problem and is just as
common in shared-config / monorepo / base-config setups.

- **Optional function parameter that no call site sets**: the behavior
  gated by the parameter is unreachable. Either make it required and
  update callers, or remove it.
- **Build args / env vars referenced in CI but unused in the consuming
  Dockerfile / app**: dead config, easy to mistake for a real secret
  channel later.
- **Feature flags / settings that have no read site** after a refactor.
- **Shared config with invisible side effects on consumers**: a config
  consumed by other repos or packages (tsconfig, eslint, prettier,
  vite, biome, package.json `scripts`) that turns on a behavior with
  consumer-visible side effects without documenting it. Example:
  `incremental: true` in a shared `tsconfig.base.json` silently emits
  `.tsbuildinfo` files in every consumer's tree. If the behavior would
  surprise the consumer, it belongs in the consumer's own config (let
  them opt in), not the shared base.
- **Shared config too narrow for the documented use cases**: shipping
  a tsconfig that documents itself as appropriate for "SvelteKit apps"
  but sets `lib: ["ES2022"]` only (no DOM types) — consumers hit
  confusing type errors. Either narrow the documented use case, add a
  per-environment variant (browser/node), or require an explicit
  override and call it out.
- **README install instructions that contradict package.json**:
  README tells consumers to `pnpm add -D foo bar baz` but `foo` and
  `bar` are already `optionalDependencies` (installed transitively).
  Causes duplicate installation and version skew. Only document
  installs the consumer must do themselves.
- **Engine/version constraints looser than what the lockfile actually
  needs**: `engines.node: ">=20"` looks reasonable, but if the lockfile
  pulls in deps requiring `^20.19.0`, consumers on Node 20.0-20.18
  hit install/runtime failures. The declared constraint must satisfy
  every transitive requirement. Same trap on `actions/setup-node`'s
  `node-version: '20'` — it picks whatever 20.x is already in the
  runner's tool cache by semver match, which can lag the current
  latest 20.x; pass `check-latest: true` to force a fresh lookup, or
  pin to the minimum required minor (`'20.19'`) so the cache can't
  resolve to something too old. Same family of issues with the
  `packageManager` field in package.json, `.nvmrc`, Docker base image
  tags, and CI tool-version pins. Pin to the strictest minimum the
  dependency tree requires, or bump to the org-standard runtime
  version.

### 8. Infrastructure & deploy hazards

For changes under `manifests/`, `.github/workflows/`, `infra/`, `iac/`,
`Dockerfile`, kustomize overlays:

- **Privilege escalation in RBAC**: namespace-scoped `verbs: ["create"]`
  on `batch/jobs` lets the service account create pods that mount any
  Secret/ConfigMap in the namespace. Scope or isolate.
- **Pod/log read access broader than required** — narrow to the specific
  workload by label or run in a dedicated namespace.
- **Bootstrap slices missing prerequisite resources**: a "first slice"
  overlay that doesn't include the namespace it deploys into.
- **Duplicate patches across overlays** that will silently drift apart.
- **`workflow_dispatch` + `on: push` to the same branch**: bootstrapping
  can trigger both, double-deploying.
- **`pulls.merge` with a SHA that may have drifted** between validation
  and merge — capture and re-check the SHA at merge time.
- **No error handling on merge API calls**: 405/409 responses leave PRs
  stuck with no retry.
- **`shift 2` in shell option parsers without arity checks**: a missing
  value crashes with a confusing `shift count out of range` instead of a
  clear usage error.
- **Interpolated shell variables into `psql -c` / `sed` / `perl`
  substitutions** without escaping — fine today, time-bomb tomorrow.
- **Shell escape sequences and regex patterns that visually differ
  from their parsed meaning**: invisible trailing whitespace is the
  classic foot-gun — a regex `^Merge\  ` (escaped space *plus* a
  trailing literal space) matches `Merge` followed by **two** spaces,
  but on screen it looks identical to the intended `^Merge\ `
  (escaped space alone) which matches `Merge` followed by one space.
  Git writes one space after `Merge` in subjects, so the buggy form
  never matches. Same trap with `'\n'` (literal backslash-n) vs
  `$'\n'` (actual newline) in bash, `\d` semantics differing across
  BRE vs ERE vs PCRE, `[abc]` (class) vs `\[abc\]` (literals). When
  writing an allowlist regex or shell substitution, validate against
  a real sample of the input you're trying to match — don't trust
  the on-screen rendering.
- **Third-party GitHub Actions pinned to moving tags instead of
  commit SHAs**: `actions/checkout@v5` is a mutable tag — the action
  repo (or anyone who compromises the maintainer's account) can
  rewrite it at any time, with no signal to consuming workflows. Per
  [GitHub's supply-chain guidance](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#using-third-party-actions),
  pin to the full SHA with a tag comment for readability:
  `actions/checkout@<sha> # v5`. Fetch the current SHA with
  `gh api repos/actions/checkout/git/refs/tags/v5 -q .object.sha`.
  Renovate/Dependabot keep pinned SHAs current. First-party
  `actions/*` is lower risk but the discipline is uniform; vendored
  agent workflows in the org already follow this pattern.
- **GitHub Actions workflow-command injection from user-controlled
  content**: printing a commit subject, PR title, branch name, or any
  other event-payload string inside a `::error::` / `::warning::` /
  `::notice::` / `::set-output::` line without escaping lets an
  attacker inject arbitrary workflow commands via `%` / `\r` / `\n` in
  the source string. Per [GitHub's workflow-commands docs](https://docs.github.com/en/actions/learn-github-actions/workflow-commands-for-github-actions),
  apply these replacements before printing user input: `%` → `%25`,
  CR → `%0D`, LF → `%0A`. A safe bash helper:
  ```bash
  escape_workflow_msg() {
    local s="$1"
    s="${s//\%/%25}"
    s="${s//$'\r'/%0D}"
    s="${s//$'\n'/%0A}"
    printf '%s' "$s"
  }
  ```
  (Even in non-attack cases, a commit subject containing `%` will get
  URL-decoded in the log and confuse you.)
- **`echo "$user_input"` treats leading `-n`/`-e`/`-E` as flags**: when
  a commit subject (or any user-controlled string) starts with one of
  those, bash builtins and some `/bin/echo` implementations swallow the
  argument as an option instead of printing it. Downstream `grep` /
  pipelines silently get empty input and a wrong result. Use
  `printf '%s\n' "$user_input"` — always safe regardless of content.
- **GitHub Actions permissions broader than the workflow needs**: each
  `permissions:` entry should map to a real API call the workflow
  makes. `pull-requests: read` on a workflow that only reads git log
  and event payload is dead scope. Drop it (least privilege).

### 9. Type-system tightening (where it matters)

- **Generic string-fallback overloads** that defeat the purpose of typed
  event/RPC overloads: callers can still pass wrong payloads under the
  fallback signature. Use conditional generics to exclude known keys.
- **`Record<string, T>` as a generic constraint** when callers pass
  interfaces without an index signature — won't compile downstream.
- **Sentinel values** like `'unknown'` emitted onto a typed event bus
  consumers don't recognize.

### 10. SQL & query correctness

- **Positional parameter binding order mismatched with the SQL text**:
  spreading params from a JOIN clause before the WHERE-clause params when
  the JOIN appears later in the string places the wrong values in each
  slot. The query silently returns wrong rows.
- **Missing indexes** on columns used as the natural reverse-lookup key
  in a join or `WHERE`.
- **Down-migration that doesn't undo all of the up**: leaves the schema
  irreversible.
- **`LIMIT N ORDER BY created_at DESC` with no status filter**: completed
  history pushes active rows off the bottom of the page.

### 11. Mechanical (only if your linter doesn't catch them)

Skip these if the project has Biome / Prettier / ESLint that would flag
them. Otherwise:

- Duplicate type-only imports of `./$types`.
- Numeric literals without separators in a codebase that uses `_`.
- SSR-unsafe `$app/navigation` calls outside a `browser` guard.
- **Shebang interpreter doesn't match the file's actual runtime
  requirements**: `#!/usr/bin/env node` on a `.ts` file *can* work on
  modern Node because Node strips erasable type syntax natively —
  `interface`, `as` casts, generic parameters, parameter type
  annotations are all fine. Default-enabled in **Node 22.18+** and
  **Node 23.6+ / 24**; available via `--experimental-strip-types`
  in Node 22.6 through 22.17. But Node *cannot* run non-erasable
  TypeScript: enum values, namespaces with runtime code, parameter
  properties (`constructor(public x: number)`), TSX/JSX, or
  decorators that need transformation. For those, you need `tsx`,
  `ts-node`, or a build step.

  The shebang should reflect the file's actual runtime: drop it if
  the file is only invoked via package.json scripts, use
  `#!/usr/bin/env -S tsx` for scripts that need tsx, or
  `#!/usr/bin/env node` *only when* the file's syntax stays within
  what Node's stripper supports and your project's pinned Node is
  ≥ 22.18 or ≥ 23.6 (or ≥ 22.6 with the flag). Same trap class for
  `#!/usr/bin/env python` running 3.10+ match-statement syntax in
  an env where `python` resolves to 3.9, or `#!/bin/sh` running
  bashisms like `[[ ]]` / `$'...'` — the interpreter the shebang
  names must actually parse the file as written.
