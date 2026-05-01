---
name: dev-helper
description: "Development workflow orchestrator for 'continue develop', 'next task', '开发', '下一阶段', and '继续开发'. It reads `dev_spec.md`, selects exactly one actionable task by checking `IP` before `Pending`, dispatches implementation/coder/testing/progress-tracker stages via local reference documents, coordinates exceptions, and reports the final result to the user."
---

# dev-helper

Development workflow orchestrator for this project.

`dev-helper` owns stage sequencing, blocker handling, retry decisions, and final user communication. It does not replace the stage references. It reads them, routes work through them, and coordinates the handoff between them.

## Core Responsibilities

1. Read `dev_spec.md` and determine the single next task to process.
2. Prefer the current `IP` task; if none exists, choose the first dependency-ready `Pending` task.
3. Generate a development guide through the implementation stage before any coding begins.
4. Route the task through coding, testing, and progress tracking.
5. Coordinate exceptions, retries, and stop conditions.
6. Stop after one task and wait for explicit user confirmation.

## Source Of Truth

Use these local reference documents as the authority for Stage 2-5:

- Stage 2 `implementation`: `references/implementation.md`
- Stage 3 `coder`: `references/coder.md`
- Stage 4 `testing`: `references/testing.md`
- Stage 5 `progress-tracker`: `references/progress_tracker.md`

Use these project documents as the authority for task discovery and delivery requirements:

- Task spec: `dev_spec.md`
- Test policy: `docs/testing_rules.md`
- Test strategy: `docs/testing_strategy.md`
- Generated task guide: `instructions/{task_id}_guide.md`
- Residual tracker: `dev_spec.md` §9.4
- Task-detail contract: `dev_spec.md` §9.3

If `instructions/` does not exist yet, create it before writing the generated guide.

## Orchestration Workflow

### Stage 1: Select The Task

1. Read `dev_spec.md`.
2. Find the current `IP` task first.
3. If there is no `IP` task, choose the first dependency-ready `Pending` task.
4. If dependencies are not ready, the task boundary is unclear, or the spec contains blockers, stop and report them to the user.

### Stage 2: Implementation Planning

1. Follow `references/implementation.md`.
2. Pass `task_id`, `task_name`, `task_context`, and `guide_path` into the implementation stage.
3. Read the required sections from `dev_spec.md`, including `§9.3 任务详情`, `§9.4 遗留开发任务跟踪`, the selected task row in `§9.2`, and `docs/testing_strategy.md`.
4. Merge explicit task scope, cross-section contracts, and all relevant `OPEN` residual items into the guide. Relevant residuals include items owned by the selected task, same-phase carry-forward items, dependency-owner items, any overlapping files/contracts touched by the task, and any residual whose `Carry Into` target names the selected task or its phase.
5. Require the guide to make task details executable: explicit in-scope/out-of-scope, required deliverables, acceptance checklist, residual closure plan, test obligations, and spec-sync expectations. If `§9.3` is not precise enough, the implementation stage must enrich the guide from authoritative architecture/contracts instead of leaving placeholders.
6. Re-open the generated guide and verify it covers every required task-detail field plus all relevant `OPEN` residual items. If any critical item is missing, reject the guide as incomplete and stop or re-run Stage 2.
7. Produce `instructions/{task_id}_guide.md`.
8. Return the guide result to `dev-helper`.
9. Treat that guide as the source of truth for downstream stages.

### Stage 3: Coding

1. Follow `references/coder.md`.
2. Implement code and required tests strictly from `instructions/{task_id}_guide.md`.
3. Return a structured implementation result to `dev-helper`.

### Stage 4: Testing

1. Follow `references/testing.md`.
2. Validate the coder deliverables against `instructions/{task_id}_guide.md` and `docs/testing_rules.md`.
3. Return a structured pass/fail report to `dev-helper`.
4. If failures are fixable, `dev-helper` decides whether to route back to Stage 3.
5. If the failure is caused by ambiguity, environment, or unavailable prerequisites, stop and escalate through `dev-helper`.

### Stage 5: Progress Tracking

1. Follow `references/progress_tracker.md`.
2. Validate the completion package and prepare the structured progress report.
3. Reconcile `dev_spec.md` §9.4 residual items: add every newly discovered unfinished obligation from the selected task and current phase, keep still-open items open, and mark completed items `DONE` with closure evidence, resolved-by task, and updated carry-forward state.
4. A task can be `CLEAN` only when all guide-scoped obligations and relevant residual obligations are completed in this turn. If any unfinished item still needs later work, it must remain or become `OPEN` in `§9.4`, and the result must downgrade to at least `WARNING`.
5. If authorized and `status == CLEAN`, apply spec updates; otherwise return the update payload to `dev-helper`.
6. `dev-helper` presents the final user-facing summary.

## Orchestrator Rules

- Process exactly one task per invocation.
- Never skip Stage 2 guide generation.
- Never accept a guide that ignores relevant `OPEN` residual items from `dev_spec.md` §9.4.
- Never accept a guide that leaves `§9.3` task details underspecified on the critical path when the missing detail can be derived from local source-of-truth docs.
- Never accept a guide that omits explicit handling for current-phase carry-forward obligations.
- The task is selected only once in Stage 1 by `dev-helper`; downstream stages must not re-select work.
- Never let Stage 3 invent requirements outside the guide.
- Never let Stage 4 or Stage 5 take over orchestration decisions.
- Normal stage outputs must return to `dev-helper` for routing and final handling.
- Only isolated stage-local exceptions that cannot be usefully handed back may surface directly to the user.
- `dev-helper` owns retry count, escalation, and the final stop/go decision.
- Never let Stage 5 report `CLEAN` if an unfinished current-task/current-phase obligation was not written back to `dev_spec.md` §9.4.
- After successful completion, stop and wait for `next task` or `继续开发`.

## Stop Conditions

Stop and report to the user when:

- `dev_spec.md` is missing or contradictory
- the selected task has unsatisfied dependencies
- the generated guide is incomplete on a critical path
- the generated guide omits relevant `OPEN` residual items, current-phase carry-forward obligations, or cross-section contracts needed for the selected task
- the generated guide does not make the selected task's scope, deliverables, acceptance criteria, or residual handling precise enough for coding/testing
- testing reports `environment_issue` or `spec_ambiguity`
- repeated fix cycles exceed the allowed retry limit

## Final User Output

Return a concise completion summary that includes:

- selected task
- generated guide path
- implementation result
- testing result
- progress-tracker result
- next recommended task

Then stop and wait for explicit confirmation before continuing.
