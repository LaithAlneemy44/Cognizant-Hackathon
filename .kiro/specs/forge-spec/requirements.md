# Requirements Document

## Introduction

ForgeSpec is a demo-first MVP web application built for a three-hour AWS Kiro hackathon showcase.
It accepts a short plain-English product brief and routes it through a visually animated pipeline of five specialised software-delivery agents — Blueprint, Atlas, Forge, Sentinel, and Veritas — producing a complete set of runnable software artifacts (spec, architecture, generated code skeleton, test results, and a review report) within a single Streamlit session.

The product proves the tagline **"From idea to tested software — autonomously."** and must be screenshot-quality impressive while remaining completely reliable inside a live demonstration.

---

## Glossary

- **App**: The Streamlit single-page application (`app.py`) that hosts the entire UI.
- **Orchestrator**: The module `agents/orchestrator.py` that coordinates all agents in sequence and writes artifacts to disk.
- **Blueprint_Agent**: `agents/spec_agent.py` — extracts structured acceptance criteria from the product brief.
- **Atlas_Agent**: `agents/architecture_agent.py` — produces the architecture document and API contract.
- **Forge_Agent**: `agents/build_agent.py` — generates the project file structure and code skeletons.
- **Sentinel_Agent**: `agents/test_agent.py` — executes the generated test suite via pytest and captures results.
- **Veritas_Agent**: `agents/review_agent.py` — verifies requirement traceability and produces a review report.
- **Run**: A single execution session identified by a UUID stored under `runs/<run-id>/`.
- **Artifact**: Any file produced during a Run and written to the Run directory.
- **Live_Run**: An execution mode in which the Orchestrator processes the user-supplied brief in real time.
- **Recorded_Demo**: An execution mode that replays a pre-verified run of the incident-management showcase.
- **Pipeline**: The ordered sequence Blueprint → Atlas → Forge → Sentinel → Veritas.
- **Agent_Status**: One of five values — `Waiting`, `Working`, `Complete`, `Failed`, `Cancelled`.
- **Coverage_Target**: A user-supplied integer percentage (1–100) representing the desired test-coverage threshold.
- **Stack**: Either `Python` or `Node`, selected by the user to indicate the target technology.
- **Readiness_Score**: A composite integer 0–100 computed by the Orchestrator from tests passed, coverage, and criteria satisfied.
- **Failure_Context**: A string passed to the Forge_Agent on a retry attempt, containing the concatenated stdout and stderr output from the previous Sentinel_Agent pytest execution.
- **Retry_Count**: An integer tracking how many Forge → Sentinel retry cycles have occurred within a single Run; the initial attempt counts as 0.
- **Max_Retries**: A configurable integer (default 1) limiting the number of Forge → Sentinel retry cycles per Run.

---

## Requirements

### Requirement 1: Product Brief Input

**User Story:** As a hackathon evaluator, I want to enter a short product description, so that I can trigger the agent pipeline with my own idea.

#### Acceptance Criteria

1. THE App SHALL render a text-area labelled **"Describe your product in plain English"** with `max_chars=300` enforced by Streamlit.
2. WHILE the user types, THE App SHALL display a live character counter in the format `<current> / 300` (e.g. `142 / 300`) updated on every keystroke.
3. IF the product brief is empty or contains only whitespace characters (spaces, tabs, newlines), THEN THE App SHALL disable the "Forge Product" button and display the validation message `"Brief cannot be empty."`.
4. IF the product brief exceeds 300 characters, THEN THE App SHALL disable the "Forge Product" button and display the validation message `"Brief must be 300 characters or fewer."`.
5. THE App SHALL render an optional free-text field labelled **"Acceptance-criteria hints"** beneath the product brief, accepting up to 500 characters.
6. THE App SHALL render a selector labelled **"Stack"** with exactly the options `Python` and `Node`, with `Python` selected by default.
7. THE App SHALL render a numeric input labelled **"Coverage target (%)"** accepting integer values between 1 and 100 inclusive; IF a non-integer or out-of-range value is entered, THEN THE App SHALL display a validation message and disable the "Forge Product" button.
8. THE App SHALL render a selector labelled **"Mode"** with exactly the options `Live Run` and `Recorded Demo`, with `Live Run` selected by default.
9. THE App SHALL render a button labelled **"Forge Product"** that is enabled only when the product brief is non-empty, does not exceed 300 characters, and the coverage target is a valid integer between 1 and 100.

---

### Requirement 2: Orchestrator Validation

**User Story:** As a developer integrating the orchestrator, I want the orchestrator to validate its inputs before processing, so that the pipeline never crashes due to bad data.

#### Acceptance Criteria

1. IF the Orchestrator receives a product brief that is `None`, empty, or whitespace-only, THEN THE Orchestrator SHALL raise a `ValueError` whose message names the violated rule (e.g. `"Product brief must not be empty."`) and SHALL NOT begin pipeline execution.
2. IF the Orchestrator receives a product brief exceeding 300 characters, THEN THE Orchestrator SHALL raise a `ValueError` with the message `"Product brief must not exceed 300 characters."` and SHALL NOT begin pipeline execution.
3. WHEN the Orchestrator raises a `ValueError`, THE App SHALL catch it, display a styled error message that indicates a validation error (no Python traceback visible to the user), and halt all further processing.

---

### Requirement 3: Agent Pipeline Execution

**User Story:** As a user, I want to watch each agent work through my brief step by step, so that I can see the autonomous delivery process unfold.

#### Acceptance Criteria

1. WHEN the "Forge Product" button is clicked in `Live Run` mode, THE Orchestrator SHALL execute the Pipeline agents in the fixed order: Blueprint_Agent, Atlas_Agent, Forge_Agent, Sentinel_Agent, Veritas_Agent.
2. WHILE each agent is executing, THE App SHALL display that agent's Agent_Status as `Working`, all already-completed agents as `Complete`, and all not-yet-started agents as `Waiting`.
3. WHEN an agent completes successfully, THE App SHALL transition that agent's Agent_Status to `Complete`.
4. IF an agent raises an unhandled exception, THEN THE App SHALL transition that agent's Agent_Status to `Failed` and display the error message (without a Python traceback) in that agent's status area in the Pipeline view.
5. WHEN an agent's Agent_Status transitions to `Failed`, THE Orchestrator SHALL halt further Pipeline execution and transition all remaining `Waiting` agents to `Cancelled`.
6. WHEN a Run is initiated, THE Orchestrator SHALL assign a UUID-based Run identifier before executing the first agent.
7. WHEN a Run is initiated, THE Orchestrator SHALL create the directory `runs/<run-id>/` before writing any Artifacts.
8. WHILE the Sentinel_Agent result has `tests_passed` equal to `0` or `tests_failed` greater than `0`, AND the current Retry_Count is strictly less than Max_Retries, THE Orchestrator SHALL invoke Forge_Agent again with the original `architecture.md` and a Failure_Context string containing the concatenated stdout and stderr from the previous Sentinel_Agent pytest execution, then immediately invoke Sentinel_Agent with the newly generated files.
9. WHEN the Retry_Count equals Max_Retries and the most recent Sentinel_Agent result still has failing tests, THE Orchestrator SHALL exit the retry loop, set `retry_exhausted` to `true` in the Run's `input.json` (or a dedicated `run-summary.json`), and continue to Veritas_Agent. THE Orchestrator SHALL record the final Retry_Count in the same summary file under the key `retry_count`.

---

### Requirement 4: Blueprint Agent — Spec Generation

**User Story:** As a user, I want the Blueprint Agent to produce a structured specification, so that all downstream agents have clear acceptance criteria to work from.

#### Acceptance Criteria

1. WHEN the Blueprint_Agent receives a product brief that is non-empty and does not exceed 300 characters, THE Blueprint_Agent SHALL produce a Markdown specification file (`spec.md`) containing a title, an introduction, and numbered acceptance criteria.
2. IF the Blueprint_Agent receives an invalid product brief (empty, whitespace-only, or exceeding 300 characters), THEN THE Blueprint_Agent SHALL raise a `ValueError` without writing any files.
3. WHEN the Blueprint_Agent produces acceptance criteria, THE Blueprint_Agent SHALL also write a machine-readable `acceptance.json` file as an array of objects, each with the keys `id` (string, format `"AC-<N>"`), `description` (string matching the corresponding spec.md criterion text), and `status` (string, initial value `"pending"`).
4. WHEN the Blueprint_Agent writes `spec.md` and `acceptance.json`, it SHALL write them to the Run directory path supplied to it as a parameter.
5. IF a file write operation fails (e.g. permission error, disk full), THEN THE Blueprint_Agent SHALL raise an `IOError` with a message identifying the affected file path.

---

### Requirement 5: Atlas Agent — Architecture Design

**User Story:** As a developer, I want the Atlas Agent to produce an architecture document and API contract, so that the Forge Agent has a concrete structure to implement.

#### Acceptance Criteria

1. WHEN the Atlas_Agent receives a readable `spec.md` artifact, THE Atlas_Agent SHALL produce `architecture.md` containing: a system overview section, a component list with at least two named components each described by role, and an API contract with at least one endpoint definition specifying HTTP method, path, request schema, and response schema.
2. IF the Atlas_Agent receives a `spec.md` path that does not exist or cannot be read, THEN THE Atlas_Agent SHALL raise a `ValueError` identifying the missing/malformed input without writing any files.
3. THE Atlas_Agent SHALL write `architecture.md` to the Run directory path supplied to it as a parameter.

---

### Requirement 6: Forge Agent — Code Generation

**User Story:** As a user, I want the Forge Agent to produce concrete project files, so that the generated software can actually be executed by the test agent.

#### Acceptance Criteria

1. WHEN the Forge_Agent receives a readable `architecture.md`, THE Forge_Agent SHALL produce at least one Python source file that is syntactically valid and importable, and at least one pytest test file that contains at least one test function referencing the source module.
2. IF the Forge_Agent receives an `architecture.md` path that does not exist or cannot be parsed, THEN THE Forge_Agent SHALL raise a `ValueError` without writing any files.
3. THE Forge_Agent SHALL write a `generated-files.json` manifest to the Run directory containing a JSON array of relative file paths for every file written by the Forge_Agent.
4. THE Forge_Agent SHALL write all generated source and test files to the Run directory (or its subdirectories).

---

### Requirement 7: Sentinel Agent — Test Execution

**User Story:** As a user, I want the Sentinel Agent to run the generated tests, so that I see real pass/fail coverage numbers.

#### Acceptance Criteria

1. WHEN the Sentinel_Agent receives a non-empty list of generated test files, THE Sentinel_Agent SHALL execute them using `pytest` with coverage measurement enabled via `pytest-cov`.
2. IF the Sentinel_Agent receives an empty list of test files, THEN THE Sentinel_Agent SHALL write zeroed results (`tests_run: 0`, `tests_passed: 0`, `tests_failed: 0`, `coverage_pct: 0`) without invoking pytest.
3. WHEN the test suite completes (pass or fail), THE Sentinel_Agent SHALL write `test-output.txt` containing the raw pytest console output.
4. WHEN the test suite completes (pass or fail), THE Sentinel_Agent SHALL write `test-results.json` containing at minimum: `tests_run` (integer), `tests_passed` (integer), `tests_failed` (integer), and `coverage_pct` (integer).
5. IF the pytest process itself crashes (non-zero exit for reasons other than test failures), THEN THE Sentinel_Agent SHALL write `test-results.json` with a `"error"` key describing the crash and set `tests_passed` to `0`.
6. IF the coverage measured is below the Coverage_Target, THEN THE Sentinel_Agent SHALL set `coverage_below_target` to `true` in `test-results.json`.

---

### Requirement 8: Veritas Agent — Requirement Traceability

**User Story:** As an evaluator, I want the Veritas Agent to confirm that every acceptance criterion is addressed, so that I can trust the output is complete.

#### Acceptance Criteria

1. WHEN the Veritas_Agent receives `acceptance.json` and `test-results.json`, THE Veritas_Agent SHALL update the `status` field of each criterion to `"satisfied"` if `tests_passed > 0` and `tests_failed == 0`, or `"not_satisfied"` otherwise; criteria with no mapped tests SHALL be set to `"not_satisfied"`.
2. WHEN the Veritas_Agent has computed satisfaction statuses, THE Veritas_Agent SHALL write `review.md` to the Run directory containing a traceability table with columns: Criterion ID, Description, and Status.
3. WHEN the Veritas_Agent has written `review.md`, THE Veritas_Agent SHALL write `final-report.md` to the Run directory summarising: Readiness_Score, total criteria, criteria satisfied, tests passed, and coverage percentage.
4. THE Veritas_Agent SHALL compute the Readiness_Score as: `int(round(((criteria_satisfied / max(total_criteria, 1)) * 0.5 + (tests_passed / max(tests_run, 1)) * 0.3 + min(coverage_pct / 100, 1.0) * 0.2) * 100))`, expressed as an integer 0–100.

---

### Requirement 9: Artifact Persistence

**User Story:** As a developer, I want every run to produce a complete, self-contained artifact set, so that I can inspect or replay any run.

#### Acceptance Criteria

1. WHEN a Run begins, THE Orchestrator SHALL write `input.json` to the Run directory containing: `run_id` (string UUID), `brief` (string), `hints` (string or null), `stack` (string), `coverage_target` (integer), `mode` (string), and `timestamp` (ISO-8601 UTC string).
2. WHEN a Run completes successfully, the Run directory SHALL contain at minimum the following top-level artifacts, plus any generated source and test files listed in `generated-files.json`: `input.json`, `spec.md`, `acceptance.json`, `architecture.md`, `generated-files.json`, `test-output.txt`, `test-results.json`, `review.md`, `final-report.md`.
3. WHEN a Run ends with a `Failed` agent, the Run directory SHALL contain `input.json` and all Artifacts written by agents that completed before the failure.
4. WHEN the Orchestrator writes any Artifact file, THE Orchestrator SHALL use a context manager (or equivalent) to ensure the file handle is closed before the next agent begins execution.

---

### Requirement 10: Recorded Demo Mode

**User Story:** As a presenter, I want a Recorded Demo mode that replays a pre-verified run, so that the showcase never fails due to environment issues.

#### Acceptance Criteria

1. WHEN `Recorded Demo` mode is selected and "Forge Product" is clicked, THE App SHALL load artifacts from a bundled pre-verified run directory (the incident-management showcase) rather than executing the Orchestrator live.
2. WHEN `Recorded Demo` mode is active, THE App SHALL display a non-dismissible `"Recorded Demo Run"` label in both the pipeline status area and the results display, persisting from the start of artifact loading until the results section is closed or reset.
3. WHEN `Recorded Demo` mode is active, THE App SHALL animate each Pipeline agent status transition (Waiting → Working → Complete) with a 1–3 second per-stage delay to match the visual rhythm of a Live_Run.
4. WHEN `Recorded Demo` mode is active, THE App SHALL NOT display any present- or future-tense claims about test execution; all log entries SHALL use past-tense descriptions (e.g. "Tests executed", not "Running tests" or "Tests will run").
5. IF the bundled pre-verified run directory is absent or any required artifact file is missing, THEN THE App SHALL display a styled error message identifying the missing file(s) and SHALL NOT fall back silently to Live_Run execution.

---

### Requirement 11: Premium Dark Dashboard UI

**User Story:** As a hackathon judge, I want the interface to look like a premium AI developer command centre, so that the demo makes a strong visual impression.

#### Acceptance Criteria

1. THE App SHALL apply a near-black background with purple, blue, and cyan glow accent colours throughout the interface.
2. THE App SHALL render agent pipeline cards using a glass-style card visual treatment with strong spacing and clear typography.
3. THE App SHALL display a terminal-style scrollable log panel that appends execution log lines during Pipeline execution.
4. WHEN all Pipeline agents reach `Complete` status, THE App SHALL display a **"READY TO SHIP"** banner with highlighted visual treatment.
5. THE App SHALL render a Final Dashboard containing: Requirements Generated, Files Created, Tests Passed, Coverage %, Acceptance Criteria Satisfied, and Readiness Score metrics.
6. THE App SHALL render a tabbed results section with the following tabs: Requirements, Architecture, API Endpoints, Test Results, Traceability, Execution Logs.
7. WHEN a Run completes, THE App SHALL display a Mermaid `flowchart LR` agent handoff diagram within the results section, with nodes for each of the five Pipeline agents and directed edges representing the execution path, including a labelled back-edge from Sentinel to Forge when `retry_count` is greater than 0 (labelled `"retry (N)"` where N is the retry count).

---

### Requirement 12: Incident-Management Showcase Content

**User Story:** As a presenter using Recorded Demo mode, I want the bundled showcase to demonstrate a realistic incident-management API, so that evaluators see a credible generated output.

#### Acceptance Criteria

1. THE Recorded_Demo Artifacts SHALL include an API specification document covering the following eight scenarios as distinct operations: creating an incident, validating that incident title is required, validating that severity is one of `low`, `medium`, `high`, or `critical`, listing all incidents, prioritising critical incidents first in list results, assigning an engineer to an incident, resolving an incident, and returning an error response indicating an unknown incident ID when a non-existent incident ID is referenced.
2. THE Recorded_Demo test suite SHALL contain at least eight test functions, where each test function exercises exactly one of the eight scenarios defined in criterion 1 and is named to identify the scenario it covers.
3. THE Recorded_Demo `test-results.json` SHALL report all eight test functions as passed and SHALL report a statement-coverage percentage of 85 or above.
4. THE Recorded_Demo `final-report.md` SHALL report a Readiness_Score on a 0–100 scale of 90 or above.

---

### Requirement 13: Reliability and Developer Experience

**User Story:** As a developer setting up the project, I want standard reliability files so that the app launches in one command on any machine.

#### Acceptance Criteria

1. THE project SHALL provide a `requirements.txt` listing all Python dependencies with exact version pins using the `==` operator (e.g. `streamlit==1.35.0`).
2. IF a virtual environment does not already exist, THEN THE `run.sh` script SHALL create one, install dependencies from `requirements.txt`, and launch `streamlit run app.py`; the script SHALL have executable permissions (`chmod +x` equivalent).
3. IF a virtual environment does not already exist, THEN THE `run.bat` script SHALL install dependencies from `requirements.txt` and launch `streamlit run app.py` on Windows.
4. THE project SHALL provide a `.gitignore` file excluding `runs/`, `__pycache__/`, `*.pyc`, and `.pytest_cache/`.
5. THE project SHALL provide a `README.md` containing at minimum: a project overview, prerequisites (Python 3.9+), step-by-step launch instructions for Windows and Linux/macOS, and a description of the two demo modes.
6. THE Orchestrator entry point SHALL be `streamlit run app.py`.

---

### Requirement 14: Provider Abstraction (P2 — Future Extension)

**User Story:** As a developer deploying ForgeSpec in different environments, I want LLM calls routed through a provider-agnostic wrapper so that I can switch between AWS Bedrock and OpenAI without changing agent code.

> **Scope note:** This requirement is out of scope for the three-hour hackathon MVP. It is captured here as a P2 extension to prevent architectural decisions in the MVP from blocking a clean future implementation.

#### Acceptance Criteria

1. IN A FUTURE RELEASE, a `LLMWrapper` class SHALL expose a `complete(prompt: str) -> str` method as its sole public interface and route calls to either AWS Bedrock (boto3/Claude) or the OpenAI SDK based on a `ProviderConfig` supplied at construction time.
2. IN A FUTURE RELEASE, IF the `provider` value in ProviderConfig is absent or not one of `"bedrock"` or `"openai"`, THEN THE LLMWrapper SHALL raise a `ConfigurationError` at construction time identifying the invalid value.
3. IN A FUTURE RELEASE, IF the underlying provider SDK raises any exception during a `complete` call, THEN THE LLMWrapper SHALL raise an `LLMProviderError` containing the provider name and the original exception message.
4. FOR THE MVP, all agents SHALL invoke the AWS Bedrock Claude API directly (without the LLMWrapper abstraction layer); the agent interfaces SHALL be designed so that introducing LLMWrapper in a future release requires no changes to agent public method signatures.
