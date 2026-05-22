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

### 7. Dead config & unwired parameters

- **Optional function parameter that no call site sets**: the behavior
  gated by the parameter is unreachable. Either make it required and
  update callers, or remove it.
- **Build args / env vars referenced in CI but unused in the consuming
  Dockerfile / app**: dead config, easy to mistake for a real secret
  channel later.
- **Feature flags / settings that have no read site** after a refactor.

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
