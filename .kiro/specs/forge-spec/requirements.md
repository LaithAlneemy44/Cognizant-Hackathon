# Requirements Document

## Introduction

**ForgeSpec** is a four-agent autonomous software-delivery pipeline built for the AWS Kiro hackathon.

A user supplies a plain-English product brief (up to 300 characters) and optional acceptance-criteria hints. A single button press launches the full pipeline — Spec Agent → Build Agent → Test Agent → Review Agent — which produces a structured specification, working Python source code, a real pytest test suite with captured results, a traceability report, and a Mermaid handoff diagram, all written to a self-contained run directory.

Tagline: **"From idea to tested software — autonomously."**

No human intervention is allowed once the pipeline begins.

---

## Glossary

- **Pipeline**: The sequential execution of SpecAgent → BuildAgent → TestAgent → ReviewAgent, coordinated by the Orchestrator.
- **Orchestrator**: The module (`agents/orchestrator.py`) that drives agent execution, manages shared state, and writes artifacts.
- **SpecAgent**: Converts the product brief and hints into `spec.md` and `acceptance.json`.
- **BuildAgent**: Reads `spec.md` and generates working Python source files under `source/`.
- **TestAgent**: Generates and executes a pytest suite against the generated source; writes real results.
- **ReviewAgent**: Compares requirements, code, and test results; produces `review.md` and `final-report.md`.
- **Run**: A single pipeline execution, identified by a UUID, with all artifacts written to `runs/<run-id>/`.
- **Artifact**: Any file produced during a Run and written to the Run directory.
- **Failure_Context**: The concatenated stdout and stderr from a failed TestAgent execution, passed to BuildAgent on a repair retry.
- **Retry_Count**: Integer counting Forge → Sentinel repair cycles; the initial attempt counts as 0.
- **Coverage_Target**: A user-supplied integer (1–100) representing the minimum acceptable test-coverage percentage.

---

## Requirements

### Requirement 1: Input Validation

**User Story:** As a user, I want the interface to validate my inputs before the pipeline starts, so that the pipeline never receives bad data.

#### Acceptance Criteria

1. THE App SHALL render a text area labelled **"Product Brief"** with a maximum of 300 characters.
2. WHILE the user types, THE App SHALL display a live character counter in the format `<current> / 300`.
3. IF the product brief is empty or contains only whitespace, THEN THE App SHALL disable the "Generate Product" button and display the message `"Brief cannot be empty."`.
4. THE App SHALL render an optional text field labelled **"Acceptance-criteria hints"**.
5. THE App SHALL render a selector labelled **"Tech Stack"** with the options `Python` (default) and `Node`; for the hackathon MVP only the Python path is implemented and tested.
6. THE App SHALL render a numeric input labelled **"Coverage target (%)"** accepting integers 1–100 inclusive; IF the value is out of range, THE App SHALL disable the "Generate Product" button.
7. THE App SHALL render a button labelled **"Generate Product"** that is enabled only when the brief is non-empty and the coverage target is valid.
8. IF the Orchestrator receives a brief that is `None`, empty, whitespace-only, or exceeds 300 characters, THEN THE Orchestrator SHALL raise a `ValueError` with a message naming the violated rule and SHALL NOT begin pipeline execution.

---

### Requirement 2: Pipeline Orchestration

**User Story:** As a user, I want the pipeline to run all four agents in sequence without any manual steps, so that I receive a complete result from a single action.

#### Acceptance Criteria

1. WHEN "Generate Product" is clicked, THE Orchestrator SHALL execute the agents in this fixed order: SpecAgent → BuildAgent → TestAgent → ReviewAgent.
2. WHEN a Run begins, THE Orchestrator SHALL generate a UUID run identifier and create the directory `runs/<run-id>/` before invoking any agent.
3. WHILE each agent is executing, THE App SHALL display that agent's status as `Working`; completed agents SHALL show `Complete`; not-yet-started agents SHALL show `Waiting`.
4. IF any agent raises an unhandled exception, THEN THE Orchestrator SHALL catch it, mark that agent as `Failed`, mark all remaining agents as `Cancelled`, and stop pipeline execution without invoking subsequent agents.
5. THE App SHALL display each agent failure message to the user without showing a Python traceback.

---

### Requirement 3: Spec Agent

**User Story:** As a user, I want the Spec Agent to turn my brief into a structured specification, so that downstream agents have clear acceptance criteria.

#### Acceptance Criteria

1. WHEN the SpecAgent `run` method is called with a non-empty brief (≤ 300 characters) and a list of hints, THE SpecAgent SHALL produce `spec.md` containing a title, an introduction, and numbered acceptance criteria.
2. THE SpecAgent SHALL also write `acceptance.json` as a JSON array where each object has the keys `id` (format `"AC-<N>"`), `description` (matching the criterion text in `spec.md`), and `status` (initial value `"pending"`).
3. BOTH files SHALL be written to the Run directory supplied as a parameter.
4. IF a file write fails, THE SpecAgent SHALL raise an `IOError` identifying the affected path.
5. IF the brief is empty, whitespace-only, or exceeds 300 characters, THE SpecAgent SHALL raise a `ValueError` without writing any files.

---

### Requirement 4: Build Agent

**User Story:** As a user, I want the Build Agent to produce working Python code from the spec, so that the Test Agent has real software to run against.

#### Acceptance Criteria

1. WHEN the BuildAgent `run` method is called with a readable `spec.md`, THE BuildAgent SHALL produce at least one Python source file that is syntactically valid and importable, written under `runs/<run-id>/source/`.
2. THE generated code SHALL be a working implementation, not a stub or skeleton with `pass`-only bodies.
3. THE BuildAgent SHALL write a `generated-files.json` manifest to the Run directory listing the relative paths of every file it wrote.
4. IF `spec.md` does not exist or cannot be read, THE BuildAgent SHALL raise a `ValueError` without writing any files.
5. WHEN called with a non-`None` `failure_context` string (repair retry path), THE BuildAgent SHALL incorporate the failure context into its prompt and produce a revised implementation.

---

### Requirement 5: Test Agent

**User Story:** As a user, I want the Test Agent to generate and execute real tests, so that I see honest pass/fail and coverage numbers.

#### Acceptance Criteria

1. WHEN the TestAgent `run` method is called, THE TestAgent SHALL generate a pytest test file from `acceptance.json` and write it under `runs/<run-id>/tests/`.
2. THE TestAgent SHALL execute the test suite using `pytest` with `pytest-cov` via `subprocess.run` with a 60-second timeout.
3. WHEN pytest completes, THE TestAgent SHALL write `test-output.txt` containing the raw pytest console output.
4. THE TestAgent SHALL write `test-results.json` containing exactly: `tests_run` (int), `tests_passed` (int), `tests_failed` (int), `coverage_pct` (int), `exit_code` (int), `stdout` (str), `stderr` (str), `retry_count` (int), and `retry_exhausted` (bool).
5. THE TestAgent SHALL NEVER fabricate test results; all values in `test-results.json` SHALL be derived from the actual pytest subprocess output.
6. IF `coverage_pct` is below the Coverage_Target, THE TestAgent SHALL set `coverage_below_target` to `true` in `test-results.json`.
7. IF the pytest subprocess itself crashes (exit code non-zero for reasons other than test failures), THE TestAgent SHALL write `test-results.json` with an `"error"` key describing the crash and set `tests_passed` to `0`.

---

### Requirement 6: Repair Retry

**User Story:** As a user, I want the pipeline to attempt one automatic repair if the first test run fails, so that transient or fixable code errors are resolved without manual intervention.

#### Acceptance Criteria

1. IF the first TestAgent result has `tests_failed > 0` or `exit_code != 0`, THEN THE Orchestrator SHALL invoke BuildAgent once more, passing the original `spec.md` and a `failure_context` string containing the concatenated `stdout` and `stderr` from the failed test run.
2. WHEN BuildAgent returns repaired code, THE Orchestrator SHALL immediately invoke TestAgent again with the new files.
3. After this one retry, THE Orchestrator SHALL proceed to ReviewAgent regardless of whether the second test run passed or failed.
4. THE `retry_count` field in `test-results.json` SHALL be set to `1` after the retry; it SHALL be `0` if no retry occurred.
5. THE `retry_exhausted` field SHALL be `true` if the second test run still fails; it SHALL be `false` if tests pass or no retry was needed.
6. IF BuildAgent raises an exception during the repair retry, THE Orchestrator SHALL treat this as a pipeline-level failure (per Requirement 2, criterion 4) and mark the run as failed.

---

### Requirement 7: Review Agent and Handoff Diagram

**User Story:** As an evaluator, I want a traceability report and a handoff diagram, so that I can verify completeness and understand the agent flow.

#### Acceptance Criteria

1. WHEN the ReviewAgent `run` method is called with `acceptance.json` and `test-results.json`, THE ReviewAgent SHALL set each criterion's `status` to `"satisfied"` if `tests_passed > 0` and `tests_failed == 0`, or `"not_satisfied"` otherwise.
2. THE ReviewAgent SHALL write `review.md` to the Run directory containing a traceability table with columns: Criterion ID, Description, and Status.
3. THE ReviewAgent SHALL write `final-report.md` summarising: total criteria, criteria satisfied, tests passed, tests failed, and coverage percentage.
4. THE Orchestrator SHALL generate a `handoff-diagram.mmd` file containing a valid Mermaid `flowchart LR` with nodes for: Product Brief, SpecAgent, BuildAgent, TestAgent, ReviewAgent, and Final Result, connected by directed edges in pipeline order.
5. IF `retry_count` is greater than `0`, THE handoff diagram SHALL include an additional directed edge from TestAgent back to BuildAgent labelled `"repair retry"`.

---

### Requirement 8: Artifacts, Streamlit Interface, and Developer Setup

**User Story:** As a developer, I want a self-contained run directory, a minimal Streamlit UI, and simple setup scripts, so that the project runs in one command on any machine and every run is inspectable.

#### Acceptance Criteria

1. WHEN a Run completes successfully, the Run directory SHALL contain: `input.json`, `spec.md`, `acceptance.json`, `generated-files.json`, `test-output.txt`, `test-results.json`, `review.md`, `handoff-diagram.mmd`, `final-report.md`, plus all source files under `source/` and test files under `tests/`.
2. WHEN a Run begins, THE Orchestrator SHALL write `input.json` containing: `run_id`, `brief`, `hints`, `stack`, `coverage_target`, and `timestamp` (ISO-8601 UTC).
3. Generated source and test files SHALL NOT be deleted after test execution.
4. THE App SHALL display, after pipeline completion, direct links or expandable panels for: spec, generated source, test output, and final report.
5. THE App SHALL apply a clean dark theme; visual polish is secondary to a working pipeline.
6. THE project SHALL provide `requirements.txt` with exact version pins (`==`) for all Python dependencies.
7. THE project SHALL provide `run.sh` (Linux/macOS) and `run.bat` (Windows) that install dependencies and launch `streamlit run app.py`.
8. THE project SHALL provide a `.gitignore` excluding `runs/`, `__pycache__/`, `*.pyc`, and `.pytest_cache/`.
9. THE project SHALL include a `README.md` with: project overview, prerequisites (Python 3.9+), launch instructions for Windows and Linux/macOS, and a description of the pipeline.
10. THE Kiro spec SHALL be maintained at `.kiro/specs/forgespec/` containing `requirements.md`, `design.md`, and `tasks.md`.

---

## Sample Run

The following brief is used for development validation and demonstration:

> "Build a task API where users can create tasks, list tasks and mark a task as complete. Task titles cannot be empty, and completing an unknown task must return a not-found response."

The sample run SHALL produce working Python source code and a real passing pytest suite.

---

## Out of Scope (MVP)

The following are explicitly excluded from the hackathon MVP:

- Multi-provider LLM abstraction (LLMWrapper, OpenAI support)
- Node.js test execution
- Recorded Demo / replay mode
- Configurable retry count (retry count is fixed at 1)
- Readiness score formula
- Incident-management-specific bundled artifacts
- Production authentication, databases, or deployment
- Elaborate UI animations, glass-card styling, or a six-tab results dashboard
- Temporary file deletion after test execution
- "READY TO SHIP" banner or startup-style marketing dashboard
