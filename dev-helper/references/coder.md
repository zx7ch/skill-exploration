---
name: coder
description: Implements code and unit tests from `instructions/{task_id}_guide.md`. Use when a task already has an approved implementation guide and the next step is to write production code strictly to that guide, or to fix implementation defects against the same guide.
triggers:
  - "implement code"
  - "write code"
  - "code implementation"
  - "写代码"
  - "开发代码"
  - "开始实现"
---

# coder

Implement production code from an existing task guide.

## Use This Skill When

- The task already has `instructions/{task_id}_guide.md`.
- The guide is the source of truth for code structure and test scope.
- The next action is implementation, not product design.

Do not use this skill to invent architecture. If the guide is missing, incomplete, or contradictory on a critical path, return the blocker to `dev-helper`. Only isolated stage-local exceptions that cannot be usefully handed back may be surfaced directly to the user.

## Inputs

Read only what is needed:

1. `instructions/{task_id}_guide.md` — required
2. Existing files referenced by the guide — if the guide says `MODIFY`
3. `dev_spec.md` — only if the guide explicitly relies on exact error codes or contracts
4. `docs/testing_rules.md` — default testing rules source
5. `docs/testing_strategy.md` — only if the guide explicitly requires deeper test design not covered by `docs/testing_rules.md`

## Required Checks

Before editing code, verify:

- The target task id is clear.
- Required dependencies marked in the guide already exist.
- The guide defines enough detail to implement signatures, behavior, and failure paths.

Return a blocker to `dev-helper` if any of these are true:

- Placeholder content such as `TODO`, `TBD`, or missing algorithm steps appears in the critical path.
- The guide conflicts with existing public interfaces and no migration rule is given.
- The guide references exact codes or schemas that cannot be resolved locally.

Only in a narrow single-stage exception, where no meaningful handoff back to `dev-helper` is possible, may this stage ask the user directly for clarification.

## Execution Flow

1. Read the task context and implementation checklist in the guide.
2. Identify target files and whether each is `NEW` or `MODIFY`.
3. Implement code in the guide's prescribed order.
4. Add or update unit tests for every required scenario.
5. Verify acceptance criteria against code and tests.
6. Return a structured implementation result to `dev-helper`.

## Implementation Rules

### 1 Follow the Guide, Not Preference

- Keep public class names, method names, signatures, return types, and exceptions exactly as specified.
- Do not add new public APIs unless the guide explicitly requires them.
- Private helpers are allowed when they do not change the external contract.

### 2 Respect Existing Code

When the guide says `MODIFY`:

- Preserve backward-compatible public interfaces unless the guide explicitly changes them.
- Change only the scope required by the guide.
- Do not refactor unrelated modules.

### 3 Implement Behavior Exactly

Translate the guide's algorithm directly:

- Preserve condition order.
- Preserve retry counts, delays, thresholds, and state transitions.
- Preserve named error branches and status mappings.

If the guide specifies an exact constant or sequence, use that exact value.

### 4 Error Handling Is Part of the Contract

Implement all error branches described in the guide.

If exact error codes are required:

- Use the exact code names from the local source of truth.
- Do not invent near matches.

### 5 Keep Code Aligned With Project Conventions

- Match the surrounding module style.
- Use existing project layout and import conventions.
- Add types, models, and exceptions in the locations already used by the project.

## Test Rules

Write tests during implementation, not after.

This skill is project-specific. `docs/testing_rules.md` is the default testing rules source for this project.

### Required References

For normal implementation tasks:

- Read `docs/testing_rules.md`
- Follow it as the default source for layer boundaries, dependency policy, fixtures, naming, markers, and shift-left expectations

Read `docs/testing_strategy.md` only when:

- the guide explicitly references it
- the task needs roadmap matrix mapping
- the task requires higher-layer testing details not covered by `docs/testing_rules.md`

### Required Test Constraints

- Implement every guide-listed scenario, and map each acceptance criterion to at least one test or parameterized case.
- Apply the layer boundary, dependency policy, fixture usage, naming, marker, and execution rules from `docs/testing_rules.md`.
- Cover success paths, boundary paths, and failure paths. When the task adds a new state, error code, threshold, event, or persisted field, add tests that lock that behavior.
- Assert externally visible behavior and contract results. Prefer outputs, state transitions, raised exceptions, error codes, and persisted effects over internal implementation details.
- Use exact exception types, error codes, and status mappings where the guide or local source of truth defines them.
- Do not create integration, e2e, or acceptance tests from this skill unless the guide explicitly requires them.

### Practical Rule

If the guide lists 6 unit-test scenarios, the finished unit test file should cover those 6 scenarios unless the project already uses a clearer equivalent such as parametrization or shared scenario helpers.

## Minimal Working Pattern

Use this as a constraint reference, not as a template to copy blindly:

```python
class Service:
    def __init__(self, dependency: Dependency) -> None:
        self._dependency = dependency

    async def execute(self, request: Request) -> Result:
        # Follow the guide's algorithm and failure handling exactly.
        ...
```

```python
class TestService:
    @pytest.mark.asyncio
    async def test_execute_success(...):
        ...

    @pytest.mark.asyncio
    async def test_execute_raises_expected_error(...):
        ...
```

## Output Format

For normal stage completion, return a structured result to `dev-helper`:

```json
{
  "task_id": "P1-1",
  "status": "COMPLETED | BLOCKED",
  "files": [
    {"path": "app/services/example.py", "status": "NEW", "summary": "Core service implementation"}
  ],
  "test_files": [
    {"path": "tests/unit/test_example.py", "status": "NEW", "scenario_count": 6}
  ],
  "ac_mapping": [
    {"ac_id": "AC1", "evidence": "tests/unit/test_example.py::test_success"}
  ],
  "blockers": [],
  "assumptions": [],
  "notes": []
}
```

If a narrow stage-local exception requires direct user clarification, keep it concise and explain why it cannot be routed meaningfully through `dev-helper`.

## Boundaries

- Do not proceed to integration, end-to-end validation, or progress tracking unless another skill owns that stage.
- Do not claim tests passed unless they were actually run.
- Do not fill missing requirements with guesses.
- Normal outputs go back to `dev-helper`, not directly to the user.

## Fix Mode

If invoked to fix defects after test failure:

1. Read the failure report.
2. Re-read only the relevant parts of the guide.
3. Fix the smallest scope necessary to restore guide compliance.
4. Update or extend unit tests if coverage was missing.
5. Stop after implementation and report what changed.
