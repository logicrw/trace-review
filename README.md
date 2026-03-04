# trace-review

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for semi-formal code review. Instead of "looks correct", it traces every execution path through your modified functions, demanding concrete evidence (line numbers, values) for each judgment.

Based on the methodology described in [Agentic Code Reasoning](https://arxiv.org/abs/2603.01896) (arXiv:2603.01896).

## What it does

When invoked, the skill:

1. Reads `git diff` to identify modified functions
2. For each function, declares inputs, outputs, callers, and invariants
3. Enumerates every execution path (happy path, branches, error paths, boundary cases)
4. Traces each path step-by-step with concrete values at each line
5. Gives a verdict (PASS / FAIL / UNCERTAIN) backed by evidence
6. Runs focused checks for common bugs: variable shadowing, off-by-one, type coercion, null propagation, resource leaks, concurrency issues, API contract breaks

## What it catches

- Variable shadowing across scopes
- Off-by-one errors at boundaries
- Implicit type coercion (`==` vs `===`, truthy/falsy)
- Null/undefined/None propagation
- Exception path changes
- Resource leaks (files, connections, locks)
- Concurrency issues with shared state
- API contract breaks affecting callers

## Install

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/logicrw/trace-review.git ~/.claude/skills/trace-review
```

Or copy `SKILL.md` into `~/.claude/skills/trace-review/`.

## Usage

In Claude Code:

```
/trace-review
```

Run it after making code changes. The skill reads your `git diff` and reviews modified functions.

## License

MIT
