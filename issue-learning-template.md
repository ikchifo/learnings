# <Tech Area> Learnings (from <Project issue #...>)

## Date
- YYYY-MM-DD

## Context
- Issue:
- Symptom:
- Environment (GPU/CPU, framework versions, OS):

## Problem Statement
- What was expected:
- What actually happened:

## Core Learning
- One-paragraph summary of the key lesson.

### Expected Shape/Flow
```text
<ASCII diagram of expected graph/data flow>
```

### Actual Shape/Flow
```text
<ASCII diagram of actual graph/data flow>
```

## Terminology (Quick)
- **<term>**: <plain-English definition>
- **<term>**: <plain-English definition>
- **<term>**: <plain-English definition>

## Root Cause
- Primary cause:
- Contributing factors:

## Fix Summary
- Change made:
- Why it works:
- Safety/compatibility notes:

## Verification
- Commands run:
```bash
<command 1>
<command 2>
```
- Results:
- Remaining gaps or blocked checks:

## Why This Matters
- Impact on reliability/performance/maintainability.

## Reusable Debugging Workflow
1. Reproduce and capture logs.
2. Dump intermediate artifacts (graphs/traces/profiles).
3. Compare expected vs actual structure.
4. Identify strict assumptions in matching/optimization logic.
5. Normalize/canonicalize or relax safely.
6. Add regression test for the failing shape.

## References
### Official Docs
- <link>
- <link>

### Related Issues/PRs
- <link>
- <link>

## Next Time Checklist
- [ ] Reproduction command recorded.
- [ ] Environment versions recorded.
- [ ] Before/after artifacts linked.
- [ ] Root cause clearly separated from symptom.
- [ ] Fix includes regression test.
