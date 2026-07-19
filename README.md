# structure-reviewer

A Claude Code plugin that assesses code structure and architecture health, forecasting spaghetti-code risk.

Unlike a line-level code review, `structure-reviewer` doesn't look for bugs. It judges organization, coupling, and maintainability trajectory — then produces a single **Spaghetti Risk Score from 1 to 10** with per-dimension evidence.

## What it does

The agent evaluates 10 structural dimensions (coupling & dependency direction, cohesion, file/function size, duplication, global mutable state, control-flow complexity, naming, layering, testability, and churn/hotspot correlation), weighting recently-touched code more heavily. It works from concrete evidence — `file:line` references, import graphs, size counts — not vibes, and is strictly read-only (`Read`, `Grep`, `Glob`, `Bash`).

Output includes:

- An overall risk score with a one-line verdict
- A per-dimension score/evidence table
- Ranked top risks
- What's genuinely healthy (it won't manufacture problems)
- The single highest-leverage fix to do first

## Install

```
claude plugin marketplace add odilboyev/structure-reviewer-plugin
claude plugin install structure-reviewer@structure-reviewer
```

Or, to try it locally without installing:

```
claude --plugin-dir ./structure-reviewer-plugin
```

## Usage

Invoke the agent directly:

```
@structure-reviewer:structure-reviewer review this PR
```

```
@structure-reviewer:structure-reviewer how spaghetti is src/api/?
```

```
@structure-reviewer:structure-reviewer do a full architecture review of this repo
```

Claude Code will also route to it automatically after a significant refactor or before merging a new module, whenever you ask for a "structure review," "architecture review," or "how spaghetti is this."

Use `code-review` instead for line-level correctness bugs — this agent only judges structure, not correctness.
