## Output format

Emit markdown to stdout. Structure:

```
## Summary

<2-4 sentences: what this PR does and the overall shape of findings.>

## Findings

### [severity] <one-line title> — `path/to/file.ts:LINE`

<Multi-paragraph explanation: what's wrong, why it matters, suggested fix.
Code excerpts welcome in fenced blocks. Use **bold** for the actionable fix.>

---

### [severity] <next finding> — `path/to/file.ts:LINE`
...
```

Severity tags: `[high]`, `[medium]`, `[low]`, `[nit]`. Order findings
severity-desc, then file path. Use `[low-confidence]` as an additional tag
when you're inferring rather than seeing the bug directly.

Severity guidance:

- **high**: correctness bug, security gap, data loss, irreversible migration,
  cross-tenant leak. Should block merge.
- **medium**: regression a user will notice, operational hazard, swallowed
  error, race condition with realistic trigger. Worth fixing this PR.
- **low**: maintainability, drift risk, brittle pattern, missing index.
  Fix if cheap; otherwise file follow-up.
- **nit**: pure style/consistency. Use sparingly — most nits should be left
  to the linter.

If you find nothing worth reporting, write a short `## Summary` explaining
what you reviewed and why nothing surfaced, then an empty `## Findings`
section. Don't pad with weak findings to look productive.

End with a `## Skipped` section if you deliberately didn't review some area
(binary fixtures, vendored deps, generated files), with a one-line reason.
