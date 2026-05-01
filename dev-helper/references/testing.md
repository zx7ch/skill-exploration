---
name: testing
description: Executes task-scoped tests, validates results against the implementation guide and `docs/testing_rules.md`, and returns a structured pass/fail report to `dev-helper`. Use for dev-helper Stage 4 or explicit test execution requests.
triggers:
  - "run tests"
  - "execute tests"
  - "test execution"
  - "测试"
  - "执行测试"
  - "验证测试"
---

# testing

Execute task-scoped tests and report results to the orchestrator.

## Use This Skill When

- Implementation for the current task is complete enough to validate
- The task already has `instructions/{task_id}_guide.md`
- `dev-helper` needs a pass/fail decision and actionable failure context

This skill does not orchestrate the workflow. `dev-helper` owns retry count, fix-cycle decisions, and stage transitions.

## Inputs

Read only what is needed:

1. `instructions/{task_id}_guide.md` — required
2. `docs/testing_rules.md` — default testing policy source
3. Relevant test files for the current task
4. `docs/testing_strategy.md` — only if the guide explicitly requires deeper test design, roadmap matrix mapping, or higher-layer policy

## Required Checks

Before executing tests, verify:

- The target task id is clear
- The guide identifies the expected test scope or test files clearly enough to select targets
- The referenced test files exist

Stop and report back to `dev-helper` if:

- The guide is ambiguous about what should be tested
- Required test files are missing
- The test layer required by the guide exceeds what is currently available
- The environment is missing a required local dependency to run the task-scoped tests

## Execution Flow

1. Read the guide's testing section and identify the required test scope
2. Read `docs/testing_rules.md` and apply the project test-layer and quality rules
3. Select the smallest correct test target for the task
4. Execute the tests and collect pass/fail, duration, and coverage data when required
5. Compare results against the guide and `docs/testing_rules.md`
6. Return a structured result to `dev-helper`

## Test Target Selection

- Prefer the exact test files listed in the guide
- If the guide does not list files, run the task's directly corresponding tests for the required layer
- Default to `unit` scope unless the guide explicitly requires `integration`, `e2e`, or `acceptance`
- Do not expand to the full suite unless the guide or failure pattern indicates the task requires broader validation

## Execution Rules

### 1) Use Project Rules First

- Use `docs/testing_rules.md` as the default source for layer boundaries, dependency expectations, fixtures, markers, and minimum quality rules
- Read `docs/testing_strategy.md` only when the task explicitly needs deeper policy not covered by `docs/testing_rules.md`

### 2) Validate the Right Things

- Verify the tests required by the guide are present and executable
- Verify scenario coverage against the guide's required behaviors, not just file presence
- Verify acceptance criteria are covered by tests or by explicit guide-mapped scenarios
- Treat coverage as a gate only when the guide or `docs/testing_rules.md` makes it applicable for the selected scope

### 3) Keep Results Task-Scoped

- Run the smallest correct command for the task
- Prefer a specific file or targeted marker set over the entire test tree
- When a targeted run is insufficient to validate the changed behavior, report that limitation to `dev-helper`

## Failure Analysis

When tests fail, classify the dominant cause as one of:

- `code_defect` — implementation behavior does not meet the guide
- `test_defect` — test setup, assertion, fixture, or mock is wrong
- `contract_violation` — interface or behavior diverges from the guide
- `environment_issue` — local dependency or runtime setup prevents correct execution
- `spec_ambiguity` — the guide is not precise enough to decide expected behavior

Keep analysis short and actionable. Focus on observed failure, expected behavior, likely owner, and minimal next action.

## Output Format

Return a structured result to `dev-helper`:

```json
{
  "task_id": "P1-1",
  "status": "PASSED | FAILED | BLOCKED",
  "test_targets": ["tests/unit/test_example.py"],
  "layer": "unit",
  "passed": 6,
  "failed": 0,
  "duration_seconds": 2.14,
  "coverage": "84%",
  "ac_coverage": ["AC1", "AC2"],
  "scenario_coverage": ["success path", "retry path"],
  "failures": [
    {
      "test_name": "tests/unit/test_example.py::test_retry",
      "expected": "retries and succeeds",
      "observed": "raises PermanentError on first failure",
      "likely_cause": "code_defect",
      "location": "app/services/example.py:48"
    }
  ],
  "next_action": {
    "action": "return_to_coder | escalate_to_dev_helper | stop",
    "reason": "Minimal routing recommendation based on the dominant failure cause"
  },
  "notes": []
}
```

## Boundaries

- Do not hold or mutate retry counters
- Do not decide whether to re-run `coder`; return the failure context to `dev-helper`
- Do not proceed to `progress-tracker`; return success or failure to `dev-helper`
- Do not invent expected behavior beyond the guide and project testing rules

## Practical Defaults

- For normal implementation tasks, run only the task-scoped `unit` tests
- Use markers only when the project already uses them for the selected target
- Include coverage only when it is meaningful for the selected test scope and available from the run
