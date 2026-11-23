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

## Example findings

Each finding follows the same shape: **problem** (the structural smell) → **forecast** (what breaks if it's left alone) → **suggestion** (the concrete fix). A few examples per dimension:

<details>
<summary><strong>Cohesion</strong> — unrelated logic sharing one file</summary>

- **Problem:** `utils.ts` accumulates unrelated logic — date formatting, API error parsing, drag-and-drop math — because it was the path of least resistance.
- **Forecast:** Every future dev adds "just one more function" here; the file becomes an untestable junk drawer, and PR reviews get noisy since unrelated changes collide in one file.
- **Suggestion:** Extract by responsibility — `dates.ts`, `api-errors.ts`, `dnd.ts` — and leave `utils.ts` for genuinely generic, stateless helpers only.

</details>

<details>
<summary><strong>Coupling & dependency direction</strong> — imports pointing the wrong way</summary>

- **Problem:** A low-level module (e.g. `lib/store.ts`) starts importing from a high-level one (e.g. a React component) to "just grab a helper."
- **Forecast:** Creates a circular dependency; changes to UI code now risk breaking data-layer logic, and the module can no longer be tested or reused independently.
- **Suggestion:** Move the shared helper down into a module both sides can import (e.g. `lib/types.ts`), and enforce the one-way flow.

</details>

<details>
<summary><strong>File & function size</strong> — a component absorbing every new feature</summary>

- **Problem:** One component keeps absorbing new state (search, filters, editing, drag state) because it's already "the stateful one."
- **Forecast:** Crosses from "the one intentional stateful component" into a God component — every new feature adds to it, re-renders balloon, and onboarding requires reading 500+ lines to make a 5-line change.
- **Suggestion:** Split by concern once it crosses a size/responsibility threshold — e.g. lift filter state into a `useIssueFilters` hook, keep drag-and-drop logic where it is.

</details>

<details>
<summary><strong>Duplication vs. reuse</strong> — the same literal typed in four places</summary>

- **Problem:** A constant (`200`, `"in_progress"`) is defined once in a shared module but re-typed as a literal in three other files.
- **Forecast:** Silent drift — someone changes the constant in one place, the others quietly fall out of sync, and nothing catches it because there's no compiler link between them.
- **Suggestion:** Import the constant instead of retyping it; type-guarded duplication (e.g. a `Record<Status, X>` tsc forces to stay exhaustive) is lower priority than untyped literal duplication.

</details>

<details>
<summary><strong>Global / shared mutable state</strong> — a second source of truth appears</summary>

- **Problem:** A new cache, counter, or config object gets attached directly to `globalThis` or a module-level `let` outside the existing singleton pattern.
- **Forecast:** Multiple uncoordinated sources of truth; behaves differently in dev (hot-reload) vs. prod, and race conditions appear under concurrent requests.
- **Suggestion:** Route it through the existing singleton, or scope it to local component state if it doesn't need to be global at all.

</details>

<details>
<summary><strong>Control-flow complexity</strong> — one handler, every branch inline</summary>

- **Problem:** A handler grows nested conditionals/callbacks (e.g. drag-end logic branching on column, position, and network state all inline).
- **Forecast:** Becomes the function nobody wants to touch — bug fixes get bolted on as more branches rather than restructuring, and coverage gaps grow every extra path.
- **Suggestion:** Flatten by extracting named sub-steps (`computeTargetOrder`, `persistReorder`) so the top-level function reads as a short sequence of calls.

</details>

<details>
<summary><strong>Layering / boundary enforcement</strong> — a client reaching past the API layer</summary>

- **Problem:** A client component starts calling a server-only module directly instead of going through the API layer.
- **Forecast:** Breaks the server/client split silently — works in dev, then fails or leaks server code into the bundle in prod; the next person copies the pattern since "it worked."
- **Suggestion:** Force all client access through the existing `fetch`/`api()` wrapper; treat any direct cross-boundary import as a hard stop, not a style nit.

</details>

<details>
<summary><strong>Naming & discoverability</strong> — vague names hiding what's inside</summary>

- **Problem:** Generic names creep in — `handler.ts`, `manager.ts`, `doStuff()` — usually because the author wasn't sure where the logic belonged.
- **Forecast:** New contributors can't guess what's inside without opening the file; grep-ability drops, and duplicate logic gets re-implemented because the original wasn't discoverable.
- **Suggestion:** Rename to reflect the actual responsibility (`validateCreateInput`, `reorderColumn`) — a vague name is usually a symptom the code itself needs to be split, not just renamed.

</details>

<details>
<summary><strong>Testability as proxy</strong> — business logic trapped inside a handler</summary>

- **Problem:** Validation or ordering logic is inlined directly inside a route handler or component instead of being a pure, importable function.
- **Forecast:** Even once a test runner exists, this logic can't be unit tested without mocking the whole request/render cycle — so it never gets covered.
- **Suggestion:** Extract to a pure function in its own module so it's trivially testable in isolation.

</details>

<details>
<summary><strong>Churn/hotspot correlation</strong> — the file every unrelated PR touches</summary>

- **Problem:** The same file shows up in commit after commit alongside unrelated features — a sign it's become a bottleneck everyone has to touch.
- **Forecast:** Merge conflicts cluster there, and it's a leading indicator of where the next big refactor will be forced, usually under time pressure.
- **Suggestion:** Treat repeated co-change as a prompt to split responsibilities before the next feature forces it, not after.

</details>

## Install

```
claude plugin marketplace add samandar-zarif/structure-reviewer-plugin
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
