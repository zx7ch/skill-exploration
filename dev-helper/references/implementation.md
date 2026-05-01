---
name: implementation
description: "Architecture analysis skill that acts as a senior architect. Analyzes technical architecture, module design, and test requirements from `dev_spec.md` §1 and §6-§9 plus `docs/testing_strategy.md` §12 for the current task, then outputs a comprehensive development guide to `instructions/{task_id}_guide.md`. Triggered by dev-helper Stage 2 or explicit 'analyze architecture' and '设计开发方案' commands."
triggers:
  - "analyze architecture"
  - "design implementation"
  - "设计开发方案"
  - "输出开发指南"
  - "create dev guide"
---

# implementation

Senior Architect skill that transforms specification into executable development guide.

## Purpose

Read technical architecture (`dev_spec.md` §1), task implementation details (§9.3), and test matrix (`docs/testing_strategy.md` §12), then synthesize a comprehensive development guide that enables the coder stage to implement without ambiguity.

**Output**: `instructions/{task_id}_guide.md`

`dev-helper` selects the task before this stage starts. Do not search for the next task inside this reference. Consume the task id and task context provided by `dev-helper`, then produce the guide for that task only.

---

## Input Sources

### Required Inputs From `dev-helper`

- `task_id` - the selected task to guide
- `task_name` - optional but recommended for guide readability
- `task_context` - dependency state, acceptance criteria, and any already-known blockers
- `guide_path` - normally `instructions/{task_id}_guide.md`

Must read these sections from `dev_spec.md` and `docs/testing_strategy.md`:

| Document | Section | Content | Purpose |
|----------|---------|---------|---------|
| dev_spec.md | §1.5 Module Design | Module responsibilities, interfaces, dependencies | Understand component boundaries |
| dev_spec.md | §1.5.x Technical Architecture | API Router, Workflow Engine, LLM Client, RAG, Session Manager | Tech stack & design decisions |
| dev_spec.md | §6 Strategy Agent | Data quality driven decision tree, Query Expansion | Strategy implementation logic |
| dev_spec.md | §5 Generation Agent | Proposal pool, parallel generation, similarity check | Generation implementation logic |
| dev_spec.md | §7.4.1 SQLite Schema | Database tables, indexes, SQL operations | Data persistence design |
| dev_spec.md | §8.4 Error Handling | Error codes, retry strategies, failure recovery | Error handling patterns |
| dev_spec.md | §9.2 Progress Tracker | Task status, dependencies | Confirm selected task status and dependencies |
| dev_spec.md | §9.3 Task Details | Task scope, acceptance criteria, file paths | Implementation boundaries |
| dev_spec.md | §9.4 Residual Tracker | Open unresolved implementation gaps by owning task and contract | Carry forward unfinished obligations |
| docs/testing_strategy.md | §12 Roadmap Test Matrix | Per-task test files, scenarios, targets | Test-driven design |
| docs/testing_strategy.md | §2 Test Pyramid | Unit/Integration/E2E/Acceptance layering | Test strategy |
| docs/testing_strategy.md | §3 Dependency Policy | Mock vs real dependencies per layer | Test isolation rules |

---

## Analysis Process

### Phase 1: Context Extraction (10%)
- Use the `task_id` selected by `dev-helper`
- Read dev_spec.md §9.2 only to confirm dependency and status context for that task
- Locate the selected task details in dev_spec.md §9.3 and expand them into an executable contract: task goal, in-scope work, out-of-scope work, required deliverables, acceptance criteria, test expectations, and residual-handling requirements.
- Read dev_spec.md §9.4 and collect all `OPEN` residual items relevant to the selected task. Relevance includes: same owner task, same-phase carry-forward items, dependency owner task, overlapping modified files, shared contracts/interfaces, or `Carry Into` targets that mention the selected task or its phase.
- Read docs/testing_strategy.md §12 Roadmap Test Matrix for corresponding task ID (e.g., <a id="ts-p1-1"></a>)
- Note dependencies on other modules (check if deps are ✅ C or 🔄 IP)
- If §9.3 is vague on a critical path, derive the missing precision from the authoritative architecture/contracts sections and record the synthesized detail explicitly in the guide instead of using placeholders.

### Phase 2: Architecture Mapping (30%)
- Read dev_spec.md §1.5 Module Design for task-related modules
- Map task into system architecture (which layer? which domain?)
- Identify interfaces to implement or consume (refer to §1.5.x Technical Architecture subsections)
- Define data models/structures (refer to §3 Data Models, §7.4.1 SQLite Schema)
- Note cross-module communication patterns (§1.5 Module Design dependency graph)

### Phase 3: Implementation Strategy (40%)
- **Module Structure**: File organization based on §9.3 Task Details "修改文件"
- **Residual Closure Plan**: Map each relevant `OPEN` residual item to either (a) work included in this guide, or (b) an explicit blocker/deferral rationale if it truly cannot be addressed in the selected task
- **Current-Phase Carry-Forward Plan**: Identify unfinished obligations from the selected task/current phase that must stay visible in later work if not fully closed in this turn
- **Interface Design**: Public APIs, method signatures (refer to §1.5.x module specs)
- **Data Flow**: Input validation → processing → output (refer to §2 Sequence Diagram, §6/§5 Agent workflows)
- **Error Handling**: Exception hierarchy, retry logic (refer to §8.4 Error Handling, §8.4.1 Spider retry spec)
- **Dependencies**: What to import, what to mock in tests (refer to docs/testing_strategy.md §3 Dependency Policy)

### Phase 4: Test Strategy Design (20%)
- Read docs/testing_strategy.md §12 Roadmap Test Matrix for specific task test requirements
- Unit test coverage targets (refer to §7 Unit Test Design + §12.x test scenarios)
- Integration points requiring mocks (refer to §3 Dependency Policy)
- Test data fixtures needed (refer to §4 Fixtures and Test Utilities)
- Critical path scenarios from test matrix "测试场景" and "测试目标"

---

## Output Specification

Save to: `instructions/{task_id}_guide.md`

Return a structured result to `dev-helper` after writing the guide:

```json
{
  "task_id": "P1-1",
  "status": "READY | BLOCKED",
  "guide_path": "instructions/P1-1_guide.md",
  "blockers": [],
  "assumptions": [],
  "notes": []
}
```

### Template Structure
````markdown
# Development Guide: {task_id} - {task_name}

> Generated: {timestamp}
> Architect: implementation skill
> Status: Draft → Ready for development
> Source: dev_spec.md §9.3, docs/testing_strategy.md §12.x

## 1. Task Context

### Scope Boundary
- **Task ID**: {task_id}
- **Task Name**: {task_name}
- **Phase**: {phase_name}
- **Dependencies**:
  - {dependency_status_summary}
- **Task Goal**:
  - {goal_summary}

### In Scope
- {in_scope_1}
- {in_scope_2}

### Out Of Scope
- {out_of_scope_1}
- {out_of_scope_2}

### Required Deliverables
- Production: {prod_deliverable_1}
- Tests: {test_deliverable_1}
- Spec/Docs: {spec_doc_deliverable_1}

### Acceptance Criteria (from dev_spec.md §9.3 + relevant residuals)
- [ ] AC1 {ac_1}
- [ ] AC2 {ac_2}
- [ ] AC3 {additional_acceptance_criteria}

### Residual Obligations (from dev_spec.md §9.4)
- **Relevant OPEN Residuals**:
  - {residual_id_1}: {summary_1} -> closure required in this guide unless blocked
  - {residual_id_2}: {summary_2}
- **Current-Phase Carry-Forward Items To Re-check**:
  - {carry_forward_item_1}
  - {carry_forward_item_2}
- **Resolved By This Task**:
  - {residuals_expected_to_close}
- **Deferred / Blocked**:
  - {residuals_not_addressable_with_reason}

### Contract Inventory
- Upstream contracts: {upstream_contracts}
- Downstream contracts: {downstream_contracts}
- Files/interfaces with compatibility risk: {compatibility_risks}

### Test Requirements (from docs/testing_strategy.md §12.x)
- **Test File**: `{primary_test_file}`
- **Test Scenarios**:
  1. {scenario_1}
  2. {scenario_2}
  3. {additional_scenarios}
- **Test Target**: {test_target_summary}

---

## 2. Architecture Context

### System Position (from dev_spec.md §1.5 Module Design)
``` 
{system_position_diagram_or_summary}
```

### Tech Stack
- Language/runtime: {runtime}
- Primary libraries/services: {libraries_and_dependencies}
- Execution pattern: {sync_async_or_worker_pattern}
- Key behavioral constraints: {behavioral_constraints}

### Constraints
- {constraint_1}
- {constraint_2}
- {constraint_3}

---

## 3. Technical Design

### 3.1 Module Structure (from dev_spec.md §9.3)

**Files to Create/Modify:**
```
{project_tree_snippet_for_this_task}
```

**Per-file Change Intent**:
| Path | NEW/MODIFY | Required Change | Linked AC / Residual |
|------|------------|-----------------|----------------------|
| `{file_1}` | `{status_1}` | {change_intent_1} | {ac_or_residual_1} |
| `{file_2}` | `{status_2}` | {change_intent_2} | {ac_or_residual_2} |

### 3.2 Class & Interface Design

**Primary Class Or Entry Point**: `{primary_interface}`

```python
class {PrimaryInterfaceName}:
    def __init__(self, ...):
        ...

    def or async def {main_method}(self, ...) -> {return_type}:
        """
        {behavior_contract}

        Raises:
            {expected_error_1}
            {expected_error_2}
        """
        ...
```

**Data Structures**:
```python
@dataclass
class {DataStructureName}:
    {field_name}: {field_type}
    ...
```

### 3.3 Algorithm & Logic Flow

**Core Flow**:
```
{step_1}
  -> {step_2}
  -> {step_3}
  -> {failure_handling_and_state_transitions}
```

### 3.4 Implementation Checklist
- [ ] Implement/adjust `{module_or_interface_1}` for {reason_1}
- [ ] Cover `{error_or_boundary_case_1}`
- [ ] Close or deliberately carry forward `{residual_id_or_gap}`
- [ ] Add/update tests for every AC and residual expectation

**Error Classification Rules**:
- {retryable_error_rules}
- {non_retryable_error_rules}

### 3.5 Error Handling Strategy

**Exception Hierarchy Or Failure Mapping**:
```
{base_exception_or_result_type}
├── {failure_type_1}
├── {failure_type_2}
└── {failure_type_3}
```

**State / Persistence Notes**:
- {state_rule_1}
- {state_rule_2}

---

## 4. Testing Strategy (from docs/testing_strategy.md §12.x, §2, §3)

### 4.1 Test Pyramid Mapping

| Level | File | Count | Focus | Mock Strategy |
|-------|------|-------|-------|---------------|
| Unit | `{unit_test_file}` | {unit_case_count} | {unit_focus} | {unit_mock_strategy} |
| Integration | `{integration_test_file_or_na}` | {integration_case_count} | {integration_focus} | {integration_mock_strategy} |
| E2E | `{e2e_file_or_na}` | {e2e_case_count} | {e2e_focus} | {e2e_mock_strategy} |

### 4.2 Critical Test Scenarios (from docs/testing_strategy.md §12.x)

**Must Implement** (per test matrix):
1. `{test_case_1}`
2. `{test_case_2}`
3. `{additional_test_cases}`

**Mock Requirements**:
- {mock_requirement_1}
- {mock_requirement_2}
- {mock_requirement_3}

### 4.3 Test Data Fixtures (from docs/testing_strategy.md §4)

```python
# tests/conftest.py fixtures needed:
@pytest.fixture
def {fixture_name_1}():
    ...

@pytest.fixture
def {fixture_name_2}():
    ...
```

### 4.4 Shift-left Cadence (from docs/testing_strategy.md §5.1)

Per shift-left requirement:
- Unit tests must be written **immediately after** code implementation
- Must pass before marking task complete
- Coverage target: {coverage_target}

---

## 5. Implementation Checklist

### Coding Sequence (Order Matters)
1. [ ] {coding_step_1}
2. [ ] {coding_step_2}
3. [ ] {coding_step_3}
4. [ ] {coding_step_4}
5. [ ] {coding_step_5}

### Dependencies to Install/Verify
```
{dependency_or_setup_checks}
```

### Configuration Required (from dev_spec.md §8.4.1, §1.5.8)
```yaml
{configuration_block_if_any}
```

---

## 6. Risk & Notes (from dev_spec.md §8.4.1, §1.5.8)

**Technical Debt Warning**:
- {technical_debt_note_1}
- {technical_debt_note_2}

**Architecture Decision**:
- {architecture_decision_1}
- {architecture_decision_2}

**Spec Alignment**:
- {spec_alignment_note_1}
- {spec_alignment_note_2}

**Cross-task Dependencies**:
- {cross_task_dependency_1}
- {cross_task_dependency_2}

## 7. Spec Sync Expectations

- If any guide-scoped obligation is left unfinished after coding/testing, it must be written back into `dev_spec.md` §9.4 as an `OPEN` residual item.
- If an existing relevant residual is fully closed by this task, progress-tracker should mark it `DONE` with closure evidence and `Resolved By = {task_id}`.
- `DONE` residuals are audit-only and must not be reintroduced into future guides unless a new regression is discovered.
````

For a long concrete example, see `examples/implementation-guide-example.md`. Treat that file as an example only, never as the default content for unrelated tasks.

### Guide Quality Gate

Before returning `READY`, verify the guide includes all of the following:

- explicit in-scope and out-of-scope boundaries
- file-level change intent for every planned edit
- acceptance criteria that are precise enough to test
- all relevant `OPEN` residual items plus current-phase carry-forward checks
- spec-sync expectations for unresolved or newly discovered gaps
