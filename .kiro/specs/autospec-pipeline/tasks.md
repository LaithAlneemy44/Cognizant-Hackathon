# Implementation Plan: AutoSpec Pipeline

## Overview

Implement AutoSpec Pipeline as a Python package (`autospec_pipeline/`) consisting of a provider-agnostic `LLMWrapper`, four agent classes (`SpecAgent`, `BuildAgent`, `TestAgent`, `ReviewAgent`), a `HandoffDiagram` builder, a central `Pipeline` orchestrator, a CLI entry-point, and a comprehensive test suite covering all 21 correctness properties using Hypothesis plus unit/example-based tests.

---

## Tasks

- [ ] 1. Project scaffolding — directory structure, dependencies, and `pyproject.toml`
  - Create the `autospec_pipeline/` package tree:
    ```
    autospec_pipeline/
    ├── __init__.py
    ├── pipeline.py
    ├── errors.py
    ├── diagram.py
    ├── agents/
    │   ├── __init__.py
    │   ├── spec_agent.py
    │   ├── build_agent.py
    │   ├── test_agent.py
    │   └── review_agent.py
    └── llm/
        ├── __init__.py
        └── llm_wrapper.py
    tests/
    ├── __init__.py
    ├── test_llm_wrapper.py
    ├── test_spec_agent.py
    ├── test_build_agent.py
    ├── test_test_agent.py
    ├── test_review_agent.py
    ├── test_diagram.py
    ├── test_pipeline.py
    └── test_properties.py
    main.py
    ```
  - Create `pyproject.toml` (or `setup.cfg`) declaring:
    - `boto3`, `openai`, `pytest`, `pytest-cov`, `hypothesis` as dependencies
    - Package name `autospec-pipeline`, entry-point `autospec = main:main`
  - Create stub `__init__.py` files so all sub-packages are importable
  - _Requirements: 9.1_

- [ ] 2. Implement `errors.py` — all named error types
  - [ ] 2.1 Define the `AutoSpecError` base class and all five named subclasses
    - `AutoSpecError(Exception)` — base class
    - `SpecGenerationError(AutoSpecError)`
    - `CodeGenerationError(AutoSpecError)`
    - `LLMProviderError(AutoSpecError)` — accepts `provider` and `original_message` args
    - `ReviewGenerationError(AutoSpecError)`
    - `ConfigurationError(AutoSpecError)`
    - _Requirements: 2.4, 3.4, 6.4, 6.5, 6.6, 7.3, 9.2_

- [ ] 3. Implement `LLMWrapper` (`llm/llm_wrapper.py`)
  - [ ] 3.1 Implement construction-time ProviderConfig validation
    - Accept `provider_config: dict` as sole constructor argument
    - Validate `provider` key exists, is non-empty, and is one of `"bedrock"` or `"openai"`; raise `ConfigurationError` otherwise
    - Validate all provider-specific required keys (`model_id` + `region` for Bedrock; `model_id` + `api_key` for OpenAI) exist and are non-empty strings; raise `ConfigurationError` naming the missing key otherwise
    - Store validated config values as private instance attributes
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.8_

  - [ ] 3.2 Implement `complete(prompt: str) -> str` with prompt-length validation
    - Raise `ValueError` if `prompt` is empty (length 0) or exceeds 1,000,000 characters before making any API call
    - Dispatch to `_complete_bedrock` or `_complete_openai` based on stored provider
    - Wrap any SDK exception in `LLMProviderError(provider_name, original_message)`
    - _Requirements: 6.1, 6.6, 6.7_

  - [ ] 3.3 Implement `_complete_bedrock` backend
    - Use `boto3.client("bedrock-runtime", region_name=self._region)`
    - Build Anthropic Messages API body with `anthropic_version`, `messages`, `max_tokens=8192`
    - Invoke model and extract `response["body"].read()["content"][0]["text"]`
    - _Requirements: 6.2_

  - [ ] 3.4 Implement `_complete_openai` backend
    - Use `openai.OpenAI(api_key=self._api_key)`
    - Call `chat.completions.create(model=self._model_id, messages=[...])`
    - Return `response.choices[0].message.content`
    - _Requirements: 6.3_

  - [ ]* 3.5 Write property test — Property 13: LLMWrapper rejects invalid provider at construction
    - **Property 13: LLMWrapper Rejects Invalid Provider at Construction**
    - Use Hypothesis `st.one_of(st.none(), st.just(""), st.text().filter(lambda s: s not in ("bedrock","openai")))` for provider values
    - Assert `ConfigurationError` is raised before any API call
    - **Validates: Requirements 6.4**

  - [ ]* 3.6 Write property test — Property 14: LLMWrapper rejects missing required keys at construction
    - **Property 14: LLMWrapper Rejects Missing Required Keys at Construction**
    - Hypothesis generates partial `ProviderConfig` dicts with one or more required keys absent or empty
    - Assert `ConfigurationError` naming the missing key
    - **Validates: Requirements 6.5**

  - [ ]* 3.7 Write property test — Property 15: SDK exceptions wrapped in LLMProviderError
    - **Property 15: SDK Exceptions Wrapped in LLMProviderError**
    - Hypothesis generates arbitrary exception types; mock SDK raises each; assert `LLMProviderError` wraps provider name and original message
    - **Validates: Requirements 6.6**

  - [ ]* 3.8 Write property test — Property 16: Prompt length validation in LLMWrapper
    - **Property 16: Prompt Length Validation in LLMWrapper**
    - Hypothesis generates empty strings (`st.just("")`) and strings longer than 1,000,000 chars; assert `ValueError` raised without any API call
    - **Validates: Requirements 6.7**

- [ ] 4. Implement `SpecAgent` (`agents/spec_agent.py`)
  - [ ] 4.1 Implement `SpecAgent.__init__` and `run` method
    - Constructor accepts `llm: LLMWrapper` and stores it
    - `run(brief: str, acceptance_criteria_hints: list[str]) -> dict`
    - Build the requirements-engineer prompt embedding `brief` and `hints_as_bullets`
    - Call `llm.complete(prompt)` exactly once
    - Parse response as JSON (handle markdown-fenced JSON blocks)
    - Validate all four keys: `title` (str), `introduction` (str), `glossary` (dict), `acceptance_criteria` (list[str])
    - Raise `SpecGenerationError` if response is empty, null-equivalent, or missing/malformed keys
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [ ]* 4.2 Write property test — Property 3: SpecAgent output conforms to required schema
    - **Property 3: SpecAgent Output Conforms to Required Schema**
    - Hypothesis generates non-empty `brief` strings and lists of hint strings; mock LLMWrapper returns valid JSON; assert returned dict has exactly the four required keys with correct types
    - **Validates: Requirements 2.1, 2.3**

  - [ ]* 4.3 Write property test — Property 4: SpecGenerationError on empty or unparseable response
    - **Property 4: SpecGenerationError on Empty or Unparseable Response**
    - Hypothesis generates empty strings, null-equivalent strings, and non-JSON garbage; assert `SpecGenerationError` raised with a non-empty message identifying the failure reason
    - **Validates: Requirements 2.4**

  - [ ]* 4.4 Write unit tests for SpecAgent happy-path and edge cases
    - Test exact prompt structure includes `brief` and formatted hints
    - Test that `llm.complete` is called exactly once per `run` invocation
    - Test JSON embedded inside markdown fences is successfully parsed
    - _Requirements: 2.2, 2.3_

- [ ] 5. Implement `BuildAgent` (`agents/build_agent.py`)
  - [ ] 5.1 Implement `BuildAgent.__init__` and `run` method
    - Constructor accepts `llm: LLMWrapper` and stores it
    - `run(spec: dict, tech_stack: str, failure_context: str | None = None) -> str`
    - Build prompt embedding `spec`, `tech_stack`, and optional failure-context addendum
    - Call `llm.complete(prompt)` exactly once
    - Extract first fenced markdown code block using regex `` ```[\w]*\n(.*?)``` `` (DOTALL)
    - Raise `CodeGenerationError` if no code block found
    - Let `LLMProviderError` and other exceptions propagate unchanged
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6_

  - [ ]* 5.2 Write property test — Property 5: BuildAgent code block extraction
    - **Property 5: BuildAgent Code Block Extraction**
    - Hypothesis generates strings with one or more fenced code blocks at various positions; assert `run` returns content of the **first** block regardless of how many are present
    - **Validates: Requirements 3.1, 3.3**

  - [ ]* 5.3 Write property test — Property 6: CodeGenerationError on missing code block
    - **Property 6: CodeGenerationError on Missing Code Block**
    - Hypothesis generates strings containing no triple-backtick delimiters; assert `CodeGenerationError` raised stating no code block was found
    - **Validates: Requirements 3.4**

  - [ ]* 5.4 Write property test — Property 7: LLMWrapper exception propagation from BuildAgent
    - **Property 7: LLMWrapper Exception Propagation from BuildAgent**
    - Hypothesis generates arbitrary exception subclasses; mock `llm.complete` raises each; assert the same exception type propagates unchanged from `BuildAgent.run`
    - **Validates: Requirements 3.6**

  - [ ]* 5.5 Write unit tests for BuildAgent
    - Test failure-context addendum appears in prompt when `failure_context` is provided
    - Test `llm.complete` called exactly once per `run` invocation
    - _Requirements: 3.2_

- [ ] 6. Implement `TestAgent` (`agents/test_agent.py`)
  - [ ] 6.1 Implement `TestAgent.__init__` and `run` method — temp files and subprocess
    - Constructor accepts `llm: LLMWrapper` and `quality_threshold: float`
    - `run(code: str, spec: dict) -> dict`
    - Call `llm.complete(prompt)` once to generate pytest test content
    - Write `code` to `NamedTemporaryFile(suffix=".py", delete=False)`
    - Write test content to `NamedTemporaryFile(prefix="test_", suffix=".py", delete=False)`
    - Execute `subprocess.run(["pytest", test_file.name, "--import-mode=importlib", f"--rootdir={tempdir}", "-v"], capture_output=True, timeout=60, text=True)`
    - _Requirements: 4.1, 4.2, 4.3_

  - [ ] 6.2 Implement result dict construction, coverage parsing, and `passed` override
    - Parse `coverage` from `TOTAL    ...    <N>%` line in stdout; set `None` if absent
    - Set `passed = (result.returncode == 0)`
    - Override `passed = False` when `coverage is not None and coverage < self.quality_threshold`
    - Return dict with keys: `passed`, `exit_code`, `stdout`, `stderr`, `coverage`
    - Delete both temp files in a `finally` block
    - _Requirements: 4.4, 4.5, 4.6, 4.7_

  - [ ]* 6.3 Write property test — Property 8: TestAgent result shape invariant
    - **Property 8: TestAgent Result Shape Invariant**
    - Hypothesis generates arbitrary pytest exit codes and stdout/stderr strings; mock subprocess; assert returned dict has exactly the five required keys with correct types
    - **Validates: Requirements 4.4**

  - [ ]* 6.4 Write property test — Property 9: Coverage below threshold forces passed = False
    - **Property 9: Coverage Below Threshold Forces Passed = False**
    - Hypothesis generates `(exit_code, coverage, threshold)` triples where `coverage < threshold`; assert `passed == False` regardless of exit code
    - **Validates: Requirements 4.5**

  - [ ]* 6.5 Write property test — Property 10: Temp files always deleted
    - **Property 10: Temp Files Always Deleted**
    - Run `TestAgent.run` under both passing, failing, and exception-raising subprocess scenarios; after each call assert `os.path.exists(code_path) == False` and `os.path.exists(test_path) == False`
    - **Validates: Requirements 4.6**

  - [ ]* 6.6 Write unit tests for TestAgent subprocess parameters
    - Verify `subprocess.run` is called with `capture_output=True`, `timeout=60`, `text=True`
    - Verify `--import-mode=importlib` flag is present in the command list
    - _Requirements: 4.3_

- [ ] 7. Implement `ReviewAgent` (`agents/review_agent.py`)
  - [ ] 7.1 Implement `ReviewAgent.__init__` and `run` method
    - Constructor accepts `llm: LLMWrapper` and stores it
    - `run(spec: dict, code: str) -> list[dict]`
    - Build prompt asking LLM to evaluate each criterion in `spec["acceptance_criteria"]` against `code`
    - Call `llm.complete(prompt)` exactly once
    - Parse response as JSON list; validate each entry has `covered` (bool) and `notes` (str, len ≤ 500)
    - Raise `ReviewGenerationError` if response is empty, null-equivalent, or cannot be deserialized
    - _Requirements: 7.1, 7.2, 7.3, 7.4_

  - [ ]* 7.2 Write property test — Property 17: ReviewAgent alignment report shape
    - **Property 17: ReviewAgent Alignment Report Shape**
    - Hypothesis generates spec dicts with N acceptance criteria (N from 1 to 20) and non-empty code strings; mock LLMWrapper returns valid JSON list of N entries; assert each entry has `covered: bool` and `notes: str` with `len(notes) <= 500`
    - **Validates: Requirements 7.2**

  - [ ]* 7.3 Write unit tests for ReviewAgent error handling
    - Test `ReviewGenerationError` raised when response is empty string
    - Test `ReviewGenerationError` raised when response is non-JSON
    - Test that the error message identifies which input caused the failure
    - _Requirements: 7.3_

- [ ] 8. Implement `HandoffDiagram` (`diagram.py`)
  - [ ] 8.1 Implement `build_handoff_diagram` function with all node/edge rules
    - Signature: `build_handoff_diagram(agents_invoked: list[str], retry_count: int, enable_review_agent: bool) -> str`
    - Always emit `flowchart LR` header
    - Always include `PipelineInputs --> SpecAgent --> BuildAgent --> TestAgent` when all ran
    - If `retry_count > 0`: add `TestAgent -->|"retry (N)"| BuildAgent` with correct N
    - If `enable_review_agent=True` and `"ReviewAgent"` in `agents_invoked`: add `TestAgent --> ReviewAgent`
    - For early-halt runs (e.g. SpecAgent error), include only nodes/edges for invoked agents
    - _Requirements: 8.1, 8.2, 8.3, 8.4_

  - [ ]* 8.2 Write property test — Property 19: HandoffDiagram contains correct nodes for actual run
    - **Property 19: HandoffDiagram Contains Correct Nodes for Actual Run**
    - Hypothesis generates all subsets of the agent list and corresponding `agents_invoked` lists; assert node appears ↔ agent was invoked; assert no extra nodes present
    - **Validates: Requirements 8.1, 8.4**

  - [ ]* 8.3 Write property test — Property 20: HandoffDiagram retry edge
    - **Property 20: HandoffDiagram Retry Edge**
    - Hypothesis generates `retry_count` values in [1, 10]; assert exactly one edge `TestAgent -->|"retry (N)"| BuildAgent` appears and N matches `retry_count`
    - **Validates: Requirements 8.2**

  - [ ]* 8.4 Write unit tests for HandoffDiagram edge cases
    - Test output is valid Mermaid `flowchart LR` string (starts with `flowchart LR`)
    - Test early-halt with only `PipelineInputs` and `SpecAgent` invoked
    - Test retry edge absent when `retry_count == 0`
    - _Requirements: 8.1, 8.4_

- [ ] 9. Checkpoint — wire up imports and verify module structure
  - Ensure all `__init__.py` files export the public classes and functions
  - Run `python -c "from autospec_pipeline.pipeline import Pipeline"` to confirm imports resolve
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Implement `Pipeline` orchestrator (`pipeline.py`)
  - [ ] 10.1 Implement `Pipeline.__init__` with construction-time config validation
    - Accept `config: dict` as sole constructor argument
    - Validate all required keys: `provider`, `model_id`, `max_retries`, `quality_threshold`, `enable_review_agent`
    - Validate types and ranges: `max_retries` int ∈ [1,10], `quality_threshold` float ∈ [0.0,1.0], `enable_review_agent` bool
    - Raise `ConfigurationError` naming the offending key on any violation
    - Construct `LLMWrapper` with only `provider`, `model_id`, plus provider-specific credential key(s)
    - Construct `SpecAgent`, `BuildAgent`, `TestAgent` (and optionally `ReviewAgent`) sharing the same `LLMWrapper`
    - _Requirements: 9.1, 9.2, 9.3, 9.4_

  - [ ] 10.2 Implement `Pipeline.run` — main execution flow and run-time input validation
    - Validate `brief` (non-empty str), `acceptance_criteria_hints` (list[str]), `tech_stack` (one of "python"/"node"), `quality_threshold` (float 0.0–1.0), `max_retries` (int 1–10)
    - Execute `SpecAgent.run → BuildAgent.run → TestAgent.run` in order
    - Wrap each agent call in try/except; on exception set `error` key and return immediately with remaining output keys as `None`
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

  - [ ] 10.3 Implement the retry loop inside `Pipeline.run`
    - While `test_result["passed"] == False` and `retry_count < max_retries`: increment `retry_count`, build `failure_context = stdout + stderr`, call `BuildAgent.run` then `TestAgent.run`
    - On `BuildAgent` exception during retry: set `error`, set `retry_exhausted = False`, halt
    - After loop: set `retry_exhausted = True` if `retry_count == max_retries and not passed`, else `False`
    - Skip retry loop entirely if initial `passed == True`; set `retry_count = 0`, `retry_exhausted = False`
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6_

  - [ ] 10.4 Implement ReviewAgent integration and HandoffDiagram generation in `Pipeline.run`
    - After successful test pass, if `enable_review_agent == True`: call `ReviewAgent.run(spec, code)`; store result under `review` key; do NOT fail pipeline on partial coverage
    - Call `build_handoff_diagram(agents_invoked, retry_count, enable_review_agent)` and store under `handoff_diagram`
    - Always include `handoff_diagram` in returned result, even on early error
    - _Requirements: 7.1, 7.5, 8.1, 8.3, 8.4_

  - [ ]* 10.5 Write property test — Property 1: Pipeline result keys always present
    - **Property 1: Pipeline Result Keys Are Always Present**
    - Hypothesis generates valid input combinations; mock all agents; assert returned dict always contains all eight required keys: `spec`, `code`, `test_result`, `handoff_diagram`, `retry_count`, `retry_exhausted`, `error`, `review`
    - **Validates: Requirements 1.3, 1.4**

  - [ ]* 10.6 Write property test — Property 2: Exception capture sets error and nulls subsequent keys
    - **Property 2: Exception Capture Sets Error and Nulls Subsequent Keys**
    - Hypothesis generates which agent (SpecAgent, BuildAgent, TestAgent) raises an exception; assert `error` is non-empty string and all keys for agents not yet invoked are `None`
    - **Validates: Requirements 1.4**

  - [ ]* 10.7 Write property test — Property 11: Retry count accurately reflects loop iterations
    - **Property 11: Retry Count Accurately Reflects Loop Iterations**
    - Hypothesis generates pass/fail sequences and `max_retries` values; mock TestAgent with side-effect list; assert `retry_count` equals exact number of retry-loop body executions
    - **Validates: Requirements 5.4, 5.6**

  - [ ]* 10.8 Write property test — Property 12: Retry exhaustion sets retry_exhausted = True
    - **Property 12: Retry Exhaustion Sets retry_exhausted = True**
    - Hypothesis generates `max_retries` ∈ [1,10]; mock TestAgent to always return `passed=False`; assert `retry_exhausted == True` and `retry_count == max_retries`
    - **Validates: Requirements 5.3**

  - [ ]* 10.9 Write property test — Property 18: Partial coverage does not fail pipeline
    - **Property 18: Partial Coverage Does Not Fail Pipeline**
    - Hypothesis generates alignment reports with at least one `covered=False` entry; mock ReviewAgent; assert `result["error"] is None`
    - **Validates: Requirements 7.5**

  - [ ]* 10.10 Write property test — Property 21: Pipeline rejects invalid configuration at construction
    - **Property 21: Pipeline Rejects Invalid Configuration at Construction**
    - Hypothesis generates config dicts with one required key removed or set to an out-of-range/wrong-type value; assert `ConfigurationError` names the offending key
    - **Validates: Requirements 9.2, 9.3**

  - [ ]* 10.11 Write unit tests for Pipeline agent ordering and key forwarding
    - Verify SpecAgent → BuildAgent → TestAgent are called in that order (use `unittest.mock.call_args_list` or call recording)
    - Verify `LLMWrapper` receives only `provider`, `model_id`, and provider-specific credential key(s) — not `max_retries`, `quality_threshold`, or `enable_review_agent`
    - _Requirements: 1.2, 9.4_

- [ ] 11. Implement CLI entry-point (`main.py`)
  - [ ] 11.1 Implement `main()` function and `argparse` CLI
    - Accept CLI arguments: `--brief` (str, required), `--hints` (str, repeatable), `--tech-stack` (str, default `"python"`), `--quality-threshold` (float, default `0.8`), `--max-retries` (int, default `3`), `--provider` (str, required), `--model-id` (str, required), `--region` (str), `--api-key` (str), `--enable-review-agent` (bool flag)
    - Build config dict and call `Pipeline(config).run(...)` 
    - Print the result dict as formatted JSON to stdout
    - Exit with code 0 on success, 1 when `result["error"] is not None`
    - _Requirements: 1.1, 1.5, 9.1_

  - [ ]* 11.2 Write unit tests for CLI argument parsing and exit codes
    - Test `--brief` is required and raises SystemExit when absent
    - Test that `result["error"] is not None` causes exit code 1
    - Test formatted JSON is written to stdout on success
    - _Requirements: 1.5_

- [ ] 12. Final checkpoint — full test suite
  - Run `pytest tests/ --cov=autospec_pipeline --cov-report=term-missing -v`
  - Ensure all tests pass, ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- All 21 correctness properties from the design are covered as Hypothesis property tests in tasks 3, 4, 5, 6, 7, 8, and 10
- Checkpoints (tasks 9 and 12) are integration validation gates
- The `LLMWrapper` is shared across all agents — construct it once in `Pipeline.__init__`
- Temp file cleanup (Property 10) must be verified in both success and exception paths
- The `handoff_diagram` key must always be present in the result dict, even on early error halt

---

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1"] },
    { "id": 1, "tasks": ["3.1"] },
    { "id": 2, "tasks": ["3.2", "3.3", "3.4"] },
    { "id": 3, "tasks": ["3.5", "3.6", "3.7", "3.8", "4.1", "5.1", "6.1", "7.1", "8.1"] },
    { "id": 4, "tasks": ["4.2", "4.3", "4.4", "5.2", "5.3", "5.4", "5.5", "6.2", "7.2", "7.3", "8.2", "8.3", "8.4"] },
    { "id": 5, "tasks": ["6.3", "6.4", "6.5", "6.6", "10.1"] },
    { "id": 6, "tasks": ["10.2", "10.3"] },
    { "id": 7, "tasks": ["10.4", "11.1"] },
    { "id": 8, "tasks": ["10.5", "10.6", "10.7", "10.8", "10.9", "10.10", "10.11", "11.2"] }
  ]
}
```
