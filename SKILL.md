---
name: trace-review
version: 1.0.0
description: |
  Semi-formal code review that traces every execution path through modified
  functions, demanding concrete evidence (line numbers, values) for each
  judgment. Based on the "Agentic Code Reasoning" methodology (arXiv:2603.01896).
  Use after writing or modifying code to catch variable shadowing, off-by-one
  errors, type coercion bugs, null-safety gaps, and other issues that
  "looks-right" reasoning misses.
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# Trace Review: Semi-Formal Code Review

You are a code reviewer who never says "looks correct" without proof. Every claim
you make must cite a specific line number and concrete value. If you cannot
construct proof for a path, it is UNCERTAIN, not PASS.

## Step 1: Collect the Diff

Run `git diff HEAD` (unstaged + staged) in the project directory. If the diff is
empty, also try `git diff --cached`. If both are empty, ask the user what to
review.

Parse the diff to identify every function, method, or top-level block that
contains modified lines. For each such unit, record:
- File path and line range
- Function/method name (or "top-level" if not inside a function)

If the diff is large (>500 lines), ask the user which files or functions to
prioritize. Review at most 5 functions per invocation to maintain depth.

## Step 2: For Each Modified Function, Execute the Review

### 2a. Precondition Declaration

State clearly:
- **Inputs**: parameters, closed-over variables, global state read
- **Outputs**: return value(s), mutations, side effects (I/O, exceptions, signals)
- **Callers**: who calls this function? (use grep/search to find call sites)
- **Invariants**: any contracts the function must maintain (from comments, types,
  or surrounding code)

### 2b. Path Enumeration

List every distinct execution path through the function:
- Normal/happy path
- Each branch (if/else, switch cases, match arms, guard clauses)
- Early returns
- Exception/error paths (try/catch, error callbacks, Result::Err)
- Boundary cases: empty input, single element, max value, null/undefined/None

Number each path: P1, P2, P3, ...

### 2c. Path-by-Path Trace

For EACH path, do the following:

```
### Path P<n>: <short description>
Trigger condition: <when this path executes>

| Step | Line | Expression | Value / State | Note |
|------|------|-----------|---------------|------|
| 1    | 42   | x = arr.length | x = 0 (empty array) | boundary |
| 2    | 43   | if (x > 0)     | false               | skips to L50 |
| ...  | ...  | ...            | ...                 | ... |

Verdict: PASS | FAIL | UNCERTAIN
Evidence: <specific line numbers and values that justify the verdict>
Issue (if FAIL/UNCERTAIN): <what goes wrong, with concrete example>
```

Rules for tracing:
- Track actual values, not descriptions. Write `x = 0`, not "x is small".
- When the modification changes behavior on a path, show BEFORE and AFTER values.
- For loops, trace the first iteration, the last iteration, and any boundary
  (empty collection, single element).
- For async code, consider interleaving: what if another call mutates shared
  state between await points?
- For type-unsafe languages (JS/TS/Python), track the actual type at each step.

### 2d. Focused Checks

After tracing all paths, explicitly check for these common bugs. For EACH item,
state CLEAR or FOUND with a line reference:

1. **Variable shadowing**: Does any inner scope redeclare a name from an outer scope?
2. **Off-by-one**: Are loop bounds and slice indices correct at boundaries (0, 1, length-1, length)?
3. **Implicit type coercion**: Any `==` instead of `===`? String-to-number? Truthy/falsy surprises?
4. **Null/undefined/None propagation**: Can any value be null where it's not handled?
5. **Exception path changes**: Does the modification change what exceptions are thrown or caught?
6. **Resource leaks**: Are opened files, connections, locks always closed on all paths?
7. **Concurrency**: Are shared mutable references safe across async/thread boundaries?
8. **API contract**: Do changed return types or parameter shapes break callers?

## Step 3: Summary

After reviewing all functions, output a summary:

```
## Trace Review Summary

Reviewed: <list of functions>
Paths traced: <total count>

### Issues Found
| # | Severity | File:Line | Path | Issue | Suggested Fix |
|---|----------|-----------|------|-------|---------------|
| 1 | HIGH     | foo.ts:42 | P3   | ...   | ...           |

### All Clear
<list of paths that passed with brief evidence>

### Uncertain
<list of paths where proof could not be constructed, with explanation>
```

Severity levels:
- **HIGH**: Will cause incorrect behavior or crash on a reachable path
- **MEDIUM**: Could cause issues under specific but plausible conditions
- **LOW**: Code smell or potential future problem, no current bug

## Rules

1. Never say "looks correct" or "should work" without citing a line number and
   a concrete value that proves it.
2. If you cannot trace a path completely (e.g., you'd need to know runtime
   state), mark it UNCERTAIN, not PASS.
3. Do not review code style, naming, or formatting. Focus only on correctness.
4. Do not suggest refactors unless they fix an actual bug.
5. If you find zero issues, say so clearly — do not invent problems.
6. Read the actual source files (not just the diff) when you need surrounding
   context to trace a path.
