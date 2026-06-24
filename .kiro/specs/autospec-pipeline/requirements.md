# Requirements Document

## Introduction

AutoSpec Pipeline is a multi-agent system triggered by a plain-English product brief that autonomously produces a structured spec, working code, and a passing test suite — end to end, without human intervention mid-run. The pipeline consists of four Python-class-based agents (SpecAgent, BuildAgent, TestAgent, and an optional ReviewAgent) coordinated through a central Pipeline orchestrator. All LLM calls are routed through a provider-agnostic wrapper that supports both AWS Bedrock (boto3/Claude) and OpenAI SDK, selectable at configuration time. Test execution uses temporary files and pytest subprocesses, with an automatic retry loop between BuildAgent and TestAgent up to a configurable maximum.

## Glossary

- **Pipeline**: The top-level orchestrator that sequences agent execution and manages shared state.
- **SpecAgent**: The agent responsible for converting a plain-English brief and acceptance criteria hints into a structured requirements document.
- **BuildAgent**: The agent responsible for generating source code from a structured spec.
- **TestAgent**: The agent responsible for writing test files targeting the appropriate framework for the specified tech stack (pytest for Python, Jest for Node), executing them via subprocess, and reporting pass/fail results.
- **ReviewAgent**: The optional agent that checks alignment between the spec and the generated code.
- **LLMWrapper**: The provider-agnostic abstraction layer that routes LLM calls to either AWS Bedrock or the OpenAI SDK based on configuration.
- **Brief**: A plain-English product description (2–3 sentences) provided as Pipeline input.
- **AcceptanceCriteriaHints**: A list of plain-English hints describing expected behaviours, provided alongside the Brief.
- **TechStackPreference**: A Pipeline input parameter specifying the target language for generated code (Python or Node).
- **MaxRetries**: A configurable integer limiting the number of BuildAgent + TestAgent retry cycles.
- **HandoffDiagram**: A textual or structured diagram documenting the data flow between agents.
- **ProviderConfig**: A configuration object specifying the LLM provider (bedrock or openai) and associated credentials/model identifiers.

---

## Requirements

### Requirement 1: Pipeline Orchestration

**User Story:** As a developer, I want to trigger the full pipeline with a single call so that I receive a spec, code, and passing tests without manual intervention.

#### Acceptance Criteria

1. THE Pipeline SHALL accept a `brief` (str, non-empty), `acceptance_criteria_hints` (list of str), and `tech_stack` (str, one of "python" or "node") as inputs to its `run` method.
2. WHEN the Pipeline `run` method is invoked with valid inputs, THE Pipeline SHALL execute SpecAgent, then BuildAgent, then TestAgent in that sequential order before returning.
3. WHEN all agents complete successfully and TestAgent reports `passed` equal to `True`, THE Pipeline SHALL return a result dictionary containing the keys: `spec` (dict), `code` (str), `test_code` (str), `test_result` (dict), `handoff_diagram` (str), `retry_count` (int), `retry_exhausted` (bool), `error` (None), and `review` (list of dict or None).
4. IF any agent raises an unhandled exception during execution, THEN THE Pipeline SHALL catch the exception, set the `error` key in the result dictionary to a string representation of the exception, set remaining unproduced output keys to `None`, and return without invoking subsequent agents.
5. THE Pipeline SHALL be implemented as a Python class with a public `run` method whose signature accepts the three inputs defined in criterion 1 and returns a dictionary conforming to the structure defined in criterion 3.

---

### Requirement 2: SpecAgent

**User Story:** As a developer, I want the SpecAgent to convert a plain-English brief into a structured requirements document so that downstream agents have an unambiguous specification to work from.

#### Acceptance Criteria

1. WHEN the SpecAgent `run` method is called with a non-empty `brief` (str) and `acceptance_criteria_hints` (list of str), THE SpecAgent SHALL produce a structured requirements document as a Python dictionary.
2. THE SpecAgent SHALL send a single LLM prompt via LLMWrapper to generate the requirements document.
3. WHEN the LLMWrapper returns a response, THE SpecAgent SHALL parse the response into a Python dictionary with exactly the keys: `title` (str), `introduction` (str), `glossary` (dict mapping str to str), and `acceptance_criteria` (list of str).
4. IF the LLMWrapper returns an empty or unparseable response (i.e. a response that is zero-length, null, or cannot be parsed into the required dictionary structure with all required keys), THEN THE SpecAgent SHALL raise a `SpecGenerationError` with a message that identifies the failure reason (empty response or missing/invalid key).
5. THE SpecAgent SHALL be implemented as a Python class with a public `run` method.

---

### Requirement 3: BuildAgent

**User Story:** As a developer, I want the BuildAgent to generate source code from the structured spec so that the pipeline produces executable output.

#### Acceptance Criteria

1. WHEN the BuildAgent `run` method is called with a structured requirements document (dict) and a `tech_stack` value (str, "python" or "node"), THE BuildAgent SHALL produce source code targeting the specified tech stack as a non-empty string.
2. THE BuildAgent SHALL send a single LLM prompt via LLMWrapper that includes the requirements document and the `tech_stack` value so that the LLM generates code in the correct language.
3. WHEN `tech_stack` is `"python"`, THE BuildAgent prompt SHALL instruct the LLM to return the implementation in a fenced `python` code block (i.e. ` ```python ... ``` `).
4. WHEN `tech_stack` is `"node"`, THE BuildAgent prompt SHALL instruct the LLM to return the implementation in a fenced `javascript` code block (i.e. ` ```javascript ... ``` `).
5. WHEN the LLMWrapper returns a response, THE BuildAgent SHALL extract the first fenced markdown code block (content between the opening ` ``` ` and closing ` ``` ` delimiters, regardless of the language tag) and return its content as a string.
6. IF the LLMWrapper returns a response containing no fenced markdown code block, THEN THE BuildAgent SHALL raise a `CodeGenerationError` with a message that states no code block was found in the response.
7. THE BuildAgent SHALL be implemented as a Python class with a public `run` method.
8. IF the LLMWrapper raises an exception during the `complete` call, THEN THE BuildAgent SHALL allow the exception to propagate to the caller without wrapping it.
9. THE BuildAgent `run` method SHALL accept an optional third parameter `failure_context` (str or None, default None) that, when provided, is appended to the LLM prompt as additional context describing a prior test failure.

---

### Requirement 4: TestAgent

**User Story:** As a developer, I want the TestAgent to write and execute tests against generated code so that I know whether the code satisfies the acceptance criteria for both Python and Node tech stacks.

#### Acceptance Criteria

1. WHEN the TestAgent `run` method is called with a non-empty `code` (str), a `spec` (dict) containing at least an `acceptance_criteria` key, and a `tech_stack` (str, "python" or "node"), THE TestAgent SHALL generate a test file as a non-empty string using a single LLM prompt via LLMWrapper targeting the correct test framework for the specified tech stack.
2. WHEN `tech_stack` is `"python"`, THE TestAgent SHALL write the `code` to one temporary `.py` file and the generated test content to a separate temporary `test_*.py` file before executing tests; both files SHALL be created in the system's default temp directory.
3. WHEN `tech_stack` is `"node"`, THE TestAgent SHALL write the `code` to one temporary `.js` file and the generated test content to a separate temporary `*.test.js` file; both files SHALL be created inside a single newly-created temporary directory, and THE TestAgent SHALL also write a minimal `package.json` to that directory declaring Jest as a dev dependency before executing tests.
4. WHEN `tech_stack` is `"python"` and both temporary files are written, THE TestAgent SHALL execute `pytest` against the test file via `subprocess.run` with `--import-mode=importlib`, `--cov` coverage flag, `capture_output=True` (or equivalent stdout/stderr capture), and a timeout of 60 seconds.
5. WHEN `tech_stack` is `"node"` and all temporary files are written, THE TestAgent SHALL first run `npm install` via `subprocess.run` in the temporary directory with `capture_output=True` and a timeout of 60 seconds, and then execute `npx jest --coverage --coverageReporters=text` via `subprocess.run` with `capture_output=True` and a timeout of 60 seconds.
6. WHEN test execution completes for either tech stack, THE TestAgent SHALL return a result dictionary containing exactly: `passed` (bool), `exit_code` (int matching the test runner process exit code), `stdout` (str), `stderr` (str), and `coverage` (float between 0.0 and 1.0, or `None` if coverage data is unavailable).
7. WHEN `tech_stack` is `"python"`, THE TestAgent SHALL parse `coverage` from the pytest-cov summary line matching the pattern `TOTAL ... N%`, converting `N` to `float(N) / 100`; if no such line is present, `coverage` SHALL be `None`.
8. WHEN `tech_stack` is `"node"`, THE TestAgent SHALL parse `coverage` from the Jest text coverage summary line matching the pattern `All files ... N%`, converting `N` to `float(N) / 100`; if no such line is present, `coverage` SHALL be `None`.
9. IF the `coverage` value is not `None` and is strictly less than the `quality_threshold` supplied at construction time, THEN THE TestAgent SHALL set `passed` to `False` in the returned result dictionary regardless of the test runner exit code.
10. WHEN `tech_stack` is `"python"`, THE TestAgent SHALL delete both temporary files after test execution completes, regardless of whether the tests passed or failed, including when an exception occurs during execution.
11. WHEN `tech_stack` is `"node"`, THE TestAgent SHALL delete the entire temporary directory (including the `node_modules` folder, the code file, the test file, and the `package.json`) after test execution completes, regardless of whether the tests passed or failed, including when an exception occurs during execution.
12. THE TestAgent SHALL be implemented as a Python class with a public `run` method that accepts `code` (str), `spec` (dict), and `tech_stack` (str) and returns the result dictionary defined in criterion 6.

---

### Requirement 5: Retry Loop

**User Story:** As a developer, I want the pipeline to automatically retry code generation and testing on failure so that transient or fixable errors are resolved without human intervention.

#### Acceptance Criteria

1. WHILE the TestAgent result has `passed` equal to `False` and the current retry count is strictly less than `max_retries`, THE Pipeline SHALL invoke BuildAgent again with the original requirements document, the original `tech_stack`, and a `failure_context` string containing the previous TestAgent `stdout` and `stderr` concatenated.
2. WHEN BuildAgent produces new code during a retry, THE Pipeline SHALL immediately invoke TestAgent with the new code and the original requirements document.
3. WHEN the retry count equals `max_retries` and the most recent TestAgent result still has `passed` equal to `False`, THE Pipeline SHALL exit the retry loop, set `retry_exhausted` to `True` in the pipeline result dictionary, and return without further agent invocations.
4. THE Pipeline SHALL record the total number of retry attempts as an integer in the pipeline result dictionary under the key `retry_count`; the initial attempt SHALL count as 0 retries.
5. IF BuildAgent raises an exception during a retry attempt, THEN THE Pipeline SHALL treat the exception as a pipeline-level error (per Requirement 1 criterion 4), set `retry_exhausted` to `False`, and halt the retry loop.
6. THE retry loop SHALL NOT execute if the initial TestAgent result has `passed` equal to `True`; in that case `retry_count` SHALL be 0 and `retry_exhausted` SHALL be `False`.

---

### Requirement 6: LLMWrapper

**User Story:** As a developer, I want a provider-agnostic LLM wrapper so that I can switch between AWS Bedrock and OpenAI without changing agent code.

#### Acceptance Criteria

1. THE LLMWrapper SHALL expose a `complete(prompt: str) -> str` method as its sole public interface for generating completions; the method SHALL return a non-empty string on success.
2. WHEN a ProviderConfig specifying `provider: "bedrock"` is supplied at construction time, THE LLMWrapper SHALL require the keys `model_id` (str) and `region` (str) in ProviderConfig and route all `complete` calls to the AWS Bedrock Runtime API using the boto3 SDK.
3. WHEN a ProviderConfig specifying `provider: "openai"` is supplied at construction time, THE LLMWrapper SHALL require the keys `model_id` (str) and `api_key` (str) in ProviderConfig and route all `complete` calls to the OpenAI Chat Completions API using the OpenAI Python SDK.
4. IF the `provider` value in ProviderConfig is absent, empty, or not one of `"bedrock"` or `"openai"`, THEN THE LLMWrapper SHALL raise a `ConfigurationError` at construction time with a message identifying the invalid provider value.
5. IF a required ProviderConfig key for the selected provider is missing or empty, THEN THE LLMWrapper SHALL raise a `ConfigurationError` at construction time identifying the missing key.
6. IF the underlying provider SDK raises any exception during a `complete` call (including network errors, authentication failures, and throttling), THEN THE LLMWrapper SHALL raise an `LLMProviderError` containing the provider name and the original exception message.
7. IF `complete` is called with a prompt that is empty (zero-length) or exceeds 1,000,000 characters, THEN THE LLMWrapper SHALL raise a `ValueError` before making any API call.
8. THE LLMWrapper SHALL be implemented as a Python class that accepts a ProviderConfig dictionary as its sole constructor argument.

---

### Requirement 7: ReviewAgent (Optional)

**User Story:** As a developer, I want an optional ReviewAgent to verify spec-to-code alignment so that I can catch gaps between requirements and implementation.

#### Acceptance Criteria

1. WHERE ReviewAgent is enabled in the Pipeline configuration, THE Pipeline SHALL invoke ReviewAgent after TestAgent reports `passed` equal to `True`.
2. WHEN the ReviewAgent `run` method is called with the structured requirements document and the generated source code, THE ReviewAgent SHALL produce an alignment report containing one entry per acceptance criterion, where each entry includes a `covered` field of type `bool` and a `notes` field of type `str` with a maximum length of 500 characters.
3. IF the LLMWrapper returns a response that is empty (zero-length or null) or cannot be deserialized into the expected alignment report structure, THEN THE ReviewAgent SHALL raise a `ReviewGenerationError` with a message that identifies which input caused the failure and states that the response was empty or unparseable.
4. THE ReviewAgent SHALL be implemented as a Python class with a public `run` method.
5. WHERE ReviewAgent is enabled, WHEN ReviewAgent produces an alignment report in which one or more acceptance criteria have `covered` equal to `False`, THE Pipeline SHALL record the alignment report as a pipeline artifact and continue without failing the pipeline run.

---

### Requirement 8: HandoffDiagram

**User Story:** As a developer, I want the pipeline to produce a handoff diagram so that I can visualise the data flow between agents.

#### Acceptance Criteria

1. WHEN the Pipeline completes a run (whether successful or halted by error), THE Pipeline SHALL generate a HandoffDiagram as a valid Mermaid `flowchart LR` string that includes a node for each of: Pipeline Inputs, SpecAgent, BuildAgent, TestAgent, and (where `enable_review_agent` is `True`) ReviewAgent, with directed edges representing the actual data flow that occurred during the run.
2. WHEN at least one retry occurred during the run (i.e. `retry_count` is greater than 0), THE HandoffDiagram SHALL include exactly one additional directed edge from TestAgent back to BuildAgent labelled `"retry (N)"` where N is the value of `retry_count`.
3. THE Pipeline SHALL include the HandoffDiagram string in the pipeline result dictionary under the key `handoff_diagram`.
4. IF the Pipeline halted early due to an agent exception (per Requirement 1 criterion 4), THEN THE HandoffDiagram SHALL include only the nodes and edges corresponding to agents that were invoked before the error occurred.

---

### Requirement 9: Configuration

**User Story:** As a developer, I want all pipeline and LLM settings to be supplied via a configuration object so that the pipeline can be adapted to different environments without code changes.

#### Acceptance Criteria

1. THE Pipeline SHALL accept a configuration dictionary at construction time containing: `provider` (str, non-empty), `model_id` (str, non-empty), `max_retries` (int, 1–10 inclusive), `quality_threshold` (float, 0.0–1.0 inclusive), `enable_review_agent` (bool), and provider-specific credential fields (`region` for Bedrock; `api_key` for OpenAI).
2. IF a required configuration key (`provider`, `model_id`, `max_retries`, `quality_threshold`, `enable_review_agent`) is absent from the configuration dictionary, THEN THE Pipeline SHALL raise a `ConfigurationError` at construction time with a message identifying the missing key by name.
3. IF any configuration value has the wrong type or is out of the allowed range (e.g. `max_retries` is not an int or is outside 1–10, `quality_threshold` is not a float or is outside 0.0–1.0), THEN THE Pipeline SHALL raise a `ConfigurationError` at construction time identifying the offending key and its invalid value.
4. THE Pipeline SHALL pass exactly the keys `provider`, `model_id`, and all provider-specific credential fields to LLMWrapper at construction time, and SHALL NOT pass `max_retries`, `quality_threshold`, or `enable_review_agent` to LLMWrapper.

---

### Requirement 10: Demo Replay Mode

**User Story:** As a presenter, I want the pipeline to operate in a replay mode using saved LLM responses so that I can demonstrate the system reliably without depending on live network access or LLM availability.

#### Acceptance Criteria

1. THE Pipeline configuration SHALL accept an optional `replay_dir` key (str, path to a directory) that, when present, activates replay mode.
2. WHEN `replay_dir` is present and non-empty in the configuration, THE LLMWrapper SHALL load pre-recorded responses from JSON files in that directory instead of making live API calls; each file SHALL correspond to one agent call and be named by invocation order (e.g. `001_spec.json`, `002_build.json`, `003_test.json`, `004_review.json`).
3. WHEN a Pipeline run completes successfully with `replay_dir` absent (i.e. live mode), THE Pipeline SHALL optionally save all LLM responses to a `recordings/` subdirectory within the working directory if the configuration key `record` is set to `True`, using the naming convention defined in criterion 2.
4. WHEN `replay_dir` is active and a required replay file is missing for a given agent invocation, THE LLMWrapper SHALL raise an `LLMProviderError` with a message stating which replay file was expected but not found.
5. THE Pipeline SHALL produce identical output structure (all result dictionary keys) regardless of whether it is running in live mode or replay mode.

---

### Requirement 11: Artifact Preservation

**User Story:** As a judge or reviewer, I want the pipeline to persist all generated artifacts from a successful run so that I can inspect the complete output without re-running the pipeline.

#### Acceptance Criteria

1. WHEN a Pipeline run completes with `passed` equal to `True`, THE Pipeline SHALL write all generated artifacts to a `sample-run/` directory in the working directory, including: `spec.json` (the structured spec), the generated code file (named appropriately for the tech stack), the generated test file, `test_output.txt` (combined stdout and stderr from test execution), `result.json` (the full pipeline result dictionary serialized as JSON), and the handoff diagram as a `.mmd` file.
2. IF the `sample-run/` directory already exists, THEN THE Pipeline SHALL overwrite its contents with the latest successful run artifacts.
3. THE repository SHALL include a pre-generated `sample-run/` directory containing a complete successful pipeline execution result that passes all tests, checked into version control as a reference demo artifact.

---

### Requirement 12: Demo Interface

**User Story:** As a presenter or evaluator, I want a minimal interactive interface so that I can demonstrate the pipeline end-to-end without writing code.

#### Acceptance Criteria

1. THE repository SHALL include a `demo.py` CLI script (or equivalent) that provides a minimal interactive interface for running the pipeline.
2. WHEN invoked, THE demo script SHALL accept a brief, acceptance criteria hints, and a tech stack selection as inputs (via CLI prompts or arguments).
3. WHEN a pipeline run completes, THE demo script SHALL display the generated spec, generated code, test output, and handoff diagram in a human-readable format.
4. THE demo script MAY be implemented as a simple CLI script that prompts for input and prints formatted output, or as a lightweight web interface (e.g. Streamlit).

---

### Requirement 13: Kiro Orchestration Artifacts

**User Story:** As a hackathon judge, I want to see documentation of how Kiro was used to design and orchestrate the multi-agent system so that I can evaluate the depth of Kiro integration.

#### Acceptance Criteria

1. THE repository SHALL contain a `docs/orchestration-design.md` file that maps each runtime agent (SpecAgent, BuildAgent, TestAgent, ReviewAgent) to the Kiro spec workflow.
2. THE `docs/orchestration-design.md` SHALL explain how Kiro was used to design and orchestrate the multi-agent system, including references to the Kiro spec files located in `.kiro/specs/autospec-pipeline/`.
3. THE `docs/orchestration-design.md` SHALL describe the agent-to-spec-phase mapping, showing which Kiro spec phase (requirements, design, tasks) corresponds to which agent's responsibilities.
