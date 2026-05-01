---
name: progress-tracker
description: "Structured deliverable validation and spec update stage. Checks implementation against the guide using `docs/testing_rules.md` quality gates, returns a structured result to `dev-helper`, and never handles user interaction directly."
triggers:
  - "validate completion"
  - "check deliverables"
  - "update spec"
  - "mark done"
  - "检查进度"
  - "存档"
---

# progress-tracker

Focused validation and spec update stage. Receives implementation outputs, verifies them against guide requirements using project testing rules, and returns a structured verdict to `dev-helper`.

## Purpose

Provide deterministic validation of task completion:
1. Verify required deliverables exist (files, tests, coverage)
2. Check quality gates per `docs/testing_rules.md`
3. Detect blocking deviations vs acceptable warnings
4. Reconcile residual work items in `dev_spec.md` §9.4, including current-phase carry-forward obligations
5. Generate structured result for `dev-helper`
6. Execute spec updates only when explicitly authorized

**Output**: Structured validation result + spec update payload (if authorized)

**Boundary**: No user interaction. No flow control. No template rendering.

---

## Input Contract

| Parameter | Source | Required | Description |
|-----------|--------|----------|-------------|
| `task_id` | dev-helper | Yes | Task identifier (e.g., P1-1) |
| `guide_path` | dev-helper | Yes | Path to `instructions/{task_id}_guide.md` |
| `dev_spec_path` | dev-helper | Yes | Path to `dev_spec.md` |
| `delivered_files` | coder output | Yes | List of `{path, status: NEW/MODIFY, summary}` |
| `test_files` | coder output | Yes | List of `{path, status: NEW/MODIFY, scenario_count}` |
| `ac_mapping` | coder output | Yes | List of `{ac_id, evidence}` |
| `test_result` | testing output | Yes | Structured testing payload from Stage 4 |
| `auto_update` | dev-helper | No | Boolean - allow immediate spec update if clean |

---

## Validation Rules (Per docs/testing_rules.md)

### Level 1: Deliverable Existence (Always Checked)

**Rule 1.1**: All files marked NEW in guide §3.1 must exist and be non-empty.
- Fail: File missing or size == 0
- Severity: BLOCKING

**Rule 1.2**: All files marked MODIFY in guide §3.1 must exist.
- Fail: Target file missing
- Severity: BLOCKING

**Rule 1.3**: Required test files must exist per docs/testing_rules.md §6.
- Check: `tests/unit/test_{module}.py` for unit tasks
- Severity: BLOCKING if missing

### Level 2: Quality Gates (Per docs/testing_rules.md)

**Rule 2.1**: Unit tests must pass (if unit layer required).
- Source: `test_result.passed / test_result.failed`
- Threshold: 100% of task-scoped tests passing
- Severity: BLOCKING if failed > 0

**Rule 2.2**: Coverage threshold per docs/testing_rules.md §6.
- Core logic: >= 80% for unit tests
- Source: `test_result.coverage`
- Severity: WARNING if below (or BLOCKING if guide specifies strict)

**Rule 2.3**: AC mapping must be declared.
- Check: Guide §1 AC list exists AND coder output includes AC mapping
- Format: List of `{ac_id, evidence}` pairs
- Severity: BLOCKING if AC exists but no mapping provided

### Level 3: Residual Backlog Reconciliation

**Rule 3.1**: Relevant `OPEN` residual items from `dev_spec.md` §9.4 must be reconciled.
- Check: each relevant residual is either resolved by delivered evidence or explicitly carried forward as still open
- Severity: WARNING if unresolved residual remains; BLOCKING if guide claimed closure but evidence is missing

**Rule 3.2**: Unfinished obligations discovered in the selected task or its current phase must be added to residual tracker payload.
- Check: deviations discovered during validation are normalized into residual entries even if the main task can otherwise be accepted
- Severity: BLOCKING only if the missing behavior invalidates claimed task completion; otherwise WARNING + carry forward

**Rule 3.3**: Residuals may be marked `DONE` only with closure evidence.
- Check: each closed residual includes evidence from delivered files/tests and the resolving task id
- Severity: BLOCKING if a residual is marked closed without evidence

### Level 4: Interface Contract (Minimal Check)

**Rule 4.1**: Public classes declared in guide §3.2 must exist in delivered code.
- Check: Class name present in module AST
- Severity: BLOCKING if missing

**Rule 4.2**: Public methods declared in guide §3.2 must exist.
- Check: Method name present in class
- Severity: BLOCKING if missing

**Rule 4.3**: Signature details (params, types, async) are NOT validated by default.
- Note: Only checked if guide provides machine-readable schema
- Rationale: Avoid high false-positive rate from parsing informal specs

---

## Output Schema

```json
{
  "task_id": "P1-1",
  "status": "CLEAN | BLOCKING | WARNING",
  "can_auto_update": true,
  "checks": {
    "deliverables": {
      "status": "PASS | FAIL",
      "blocking": ["MISSING_FILE:app/services/xhs_spider.py"],
      "warnings": []
    },
    "quality_gates": {
      "status": "PASS | FAIL",
      "blocking": ["TEST_FAILED:tests/unit/test_spider.py::test_retry"],
      "warnings": ["COVERAGE_LOW:75% < 80%"]
    },
    "ac_mapping": {
      "status": "PASS | FAIL",
      "mapped": ["AC1->test_first_success", "AC2->test_retry_logic"],
      "unmapped": ["AC3", "AC4"]
    },
    "residuals": {
      "status": "PASS | WARNING | FAIL",
      "closed": [
        {
          "id": "RES-P4-3-003",
          "resolved_by": "P5-1",
          "evidence": ["tests/unit/test_router.py::test_invalid_last_event_id"]
        }
      ],
      "still_open": [
        {
          "id": "RES-P1-5-001",
          "carry_into": "P5-2",
          "reason": "Lifecycle log events still missing in live SSE path"
        }
      ],
      "newly_opened": [
        {
          "id": "RES-P5-1-001",
          "owner_task": "P5-1",
          "carry_into": "P5-2",
          "summary": "Discovered unfinished purge/job cancellation contract"
        }
      ],
      "notes": []
    },
    "interface": {
      "status": "PASS | PARTIAL",
      "missing_classes": [],
      "missing_methods": ["XHSSpiderClient._classify_error"]
    }
  },
  "spec_update_payload": {
    "section_9_2": {
      "task_id": "P1-1",
      "new_status": "✅ D",
      "completed_date": "2026-03-14",
      "notes": "-"
    },
    "section_9_3": {
      "audit_trail": "...markdown...",
      "deviations": []
    },
    "section_9_4": {
      "closed": ["RES-P4-3-003"],
      "still_open": ["RES-P1-5-001"],
      "newly_opened": ["RES-P5-1-001"]
    },
    "downstream_updates": ["P1-2", "P2-2"]
  },
  "next_action": {
    "action": "proceed | return_to_coder | escalate_to_dev_helper",
    "reason": "Missing required deliverables"
  }
}
```

---

## Spec Update Execution

### When Updates Are Applied

**Auto-update allowed only if**:
- `auto_update: true` passed by dev-helper
- `status` is `CLEAN`
- All quality gates passed
- AC mapping complete
- No relevant residual item remains `OPEN` after reconciliation
- No newly discovered unfinished current-task/current-phase obligation needs to be carried forward

**Otherwise**:
- Generate `spec_update_payload` but do NOT write to `dev_spec.md`
- Return payload to `dev-helper` for routing
- `dev-helper` decides when or whether to apply it

### Update Format

**§9.2 Progress Tracker Row**:
```
| {task_id} | {task_name} | ✅ D | {deps} | {date} | {notes} |
```

**§9.3 Implementation Notes** (appended to task section):
```markdown
#### {task_id} Completion Record
- Status: ✅ D (Done)
- Date: {timestamp}
- Deliverables: {file_count} files, {test_count} tests
- Quality: {coverage}% coverage, {pass_rate}% pass rate
- Deviations: {deviation_count} (if any)
```

**§9.4 Residual Tracker**:
```markdown
| RES-... | {owner_task} | {phase} | OPEN/DONE | {date} | {summary} | {affected_files_or_contracts} | {carry_into_task_or_phase} | {closure_criteria} | {closure_evidence_or_-} | {resolved_by_or_-} |
```

Rules:
- Close residuals only when evidence exists in delivered files/tests
- If a residual is partially addressed but not fully closed, keep it `OPEN`, update carry-forward target if needed, and refresh summary/notes
- Any newly discovered unfinished contract gap must be appended as a new residual row

**Dependency Propagation**:
- Scan §9.2 for tasks depending on `{task_id}`
- Update their dependency lists to show `✅ D`
- If all deps now `✅ D`, mark task as `⏳ R` (Ready)

---

## Error Handling

| Scenario | Response |
|----------|----------|
| Guide file not found | Return `BLOCKING`, error "Guide not found" |
| dev_spec.md not found | Return `BLOCKING`, error "Spec not found" |
| Coder output malformed | Return `BLOCKING`, validate deliverables list |
| Test result missing | Return `BLOCKING`, require testing skill output |
| AST parsing fails | Log warning, skip interface checks (don't block) |

---

## Execution Rules

1. **No Side Effects Without Authorization**: Never write to `dev_spec.md` unless `auto_update: true` and validation clean.

2. **Deterministic Output**: Same inputs always produce same output. No random IDs, no timestamps in logic (only in audit records).

3. **Fail Fast on Missing Inputs**: If required parameter missing, return BLOCKING immediately with clear error.

4. **Use docs/testing_rules.md as Default**: Read `docs/testing_rules.md` for layer definitions, coverage thresholds, and quality gates. Only read `docs/testing_strategy.md` if the guide explicitly references it.

5. **Minimal AST Checking**: Only verify public API surface exists. Don't validate implementation details, parameter types, or internal logic.

6. **Structured Only**: Output only JSON-structured result. No markdown templates, no ASCII diagrams, no user-facing text blocks.

7. **dev-helper Owns Flow**: This stage validates and prepares updates. `dev-helper` decides to apply, retry, escalate, or stop.

8. **Clean/Warn/Block Only**: Three validation statuses. No intermediate states. BLOCKING = must fix before done. WARNING = acceptable with note. CLEAN = no issues.

9. **Residual Ledger Is Mandatory**: If unfinished work still exists after validation, it must appear in `section_9_4.newly_opened` or `section_9_4.still_open`; never drop it from the report.

---

## Usage by dev-helper

### Example: Clean Completion

```python
# Dev-helper calls progress-tracker
result = progress_tracker.validate(
    task_id="P1-1",
    guide_path="instructions/P1-1_guide.md",
    delivered_files=coder_output.files,
    test_files=coder_output.test_files,
    ac_mapping=coder_output.ac_mapping,
    test_result=testing_output,
    auto_update=True  # Skip user confirm if clean
)

# Result
{
  "status": "CLEAN",
  "can_auto_update": true,
  "spec_update_payload": {...}
}

# Dev-helper action
if result.can_auto_update:
    apply_spec_update(result.spec_update_payload)
    generate_completion_report(result)
    # Task done, wait for "next task"
```

### Example: Blocking Deviation

```python
result = progress_tracker.validate(..., auto_update=False)

# Result
{
  "status": "BLOCKING",
  "can_auto_update": false,
  "checks": {
    "deliverables": {
      "status": "FAIL",
      "blocking": ["MISSING_FILE:app/exceptions/spider_exceptions.py"]
    }
  },
  "next_action": {
    "action": "return_to_coder",
    "reason": "Required deliverable missing: app/exceptions/spider_exceptions.py"
  }
}

# Dev-helper action
route_next_stage(result)
```

---

## Integration Points

**Reads**:
- `instructions/{task_id}_guide.md` (§1 AC, §3 structure, §3.2 interface)
- `docs/testing_rules.md` (quality gates, thresholds)
- `dev_spec.md` (for dependency propagation)

**Receives from**:
- `coder` stage: delivered_files, test_files, and ac_mapping
- `testing` stage: structured test_result
- `dev-helper`: orchestration parameters (auto_update, etc.)

**Produces**:
- Structured validation result (JSON schema above)
- Spec update payload (when authorized)

**Does NOT**:
- Read user input
- Render UI templates
- Control workflow flow
- Make retry decisions
- Update spec without authorization
- Communicate directly with the user
