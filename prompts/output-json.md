## Output format

Emit a single JSON object to stdout. No prose before or after. No markdown
code fence — just the JSON. Schema:

```ts
type Severity = "high" | "medium" | "low" | "nit";
type Confidence = "high" | "medium" | "low";

type Finding = {
  severity: Severity;
  category: string;     // one of the checklist section titles, lowercased + kebab
                        // e.g. "refactor-regression", "tenant-isolation",
                        // "race-condition", "silent-error", "hardcoded-value",
                        // "doc-drift", "dead-config", "infra-hazard",
                        // "type-tightening", "sql-correctness", "mechanical"
  file: string;         // relative path from repo root
  line?: number;        // 1-based, the most relevant line if pinpointable
  title: string;        // one-line summary, under 100 chars
  body: string;         // multi-paragraph: what, why it matters, suggested fix.
                        // markdown OK. include code excerpts when helpful.
  confidence: Confidence;
};

type Output = {
  summary: string;      // 2-4 sentences: what this PR does and the overall
                        // shape of findings (e.g. "Mostly refactor regressions
                        // from the component extraction; one tenant isolation
                        // gap worth blocking on.")
  findings: Finding[];  // ordered by severity desc, then file path
  skipped: string[];    // areas you deliberately didn't review and why
                        // (e.g. "binary fixture files", "vendored deps")
};
```

Severity guidance:

- **high**: correctness bug, security gap, data loss, irreversible migration,
  cross-tenant leak. Should block merge.
- **medium**: regression a user will notice, operational hazard, swallowed
  error, race condition with realistic trigger. Worth fixing this PR.
- **low**: maintainability, drift risk, brittle pattern, missing index.
  Fix if cheap; otherwise file follow-up.
- **nit**: pure style/consistency that wouldn't change behavior. Use
  sparingly — most nits should be left to the linter.

If you find nothing worth reporting, emit `{"summary": "...", "findings":
[], "skipped": []}` with a one-line summary explaining what you reviewed
and why nothing surfaced. Don't pad with weak findings to look productive.
