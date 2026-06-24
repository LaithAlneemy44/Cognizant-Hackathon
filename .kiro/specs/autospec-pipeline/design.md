# Design Document

## AutoSpec Pipeline

---

## Overview

AutoSpec Pipeline is a multi-agent Python system that accepts a plain-English product brief and autonomously produces a structured requirements document, working source code, and a passing pytest test suite. A central `Pipeline` orchestrator sequences four Python-class-based agents (`SpecAgent`, `BuildAgent`, `TestAgent`, and an optional `ReviewAgent`). All LLM calls are routed through a provider-agnostic `LLMWrapper` that supports AWS Bedrock (boto3/Claude) and the OpenAI SDK, selected at configuration time. A configurable retry loop between `BuildAgent` and `TestAgent` handles transient or fixable failures, and a Mermaid flowchart is generated at the end of every run to document the actual data flow.

---

## Architecture

### High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Pipeline                                                                    │
│                                                                              │
│  ┌──────────────┐   spec   ┌──────────────┐  code  ┌──────────────┐        │
│  │  SpecAgent   │─────────▶│  BuildAgent  │───────▶│  TestAgent   │        │
│  └──────────────┘          └──────────────┘        └──────────────┘        │
│         │                        ▲                        │                  │
│         │                        │   failure_context      │                  │
│         │                        └────────────────────────┘                  │
│         │                          (retry loop, 1–10 times)                  │
│         │                                                                    │
│         │ (optional, after passed=True)                                      │
│         ▼                                                                    │
│  ┌──────────────┐                                                            │
│  │ ReviewAgent  │                                                            │
│  └──────────────┘                                                            │
│                                                                              │
│  All agents share one LLMWrapper instance                                   │
└─────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
   ┌──────────────┐
   │  LLMWrapper  │
   │  ┌─────────┐ │
   │  │ Bedrock │ │
   │  └─────────┘ │
   │  ┌─────────┐ │
   │  │ OpenAI  │ │
   │  └─────────┘ │
   └──────────────┘
```

### Module Layout

```
autospec_pipeline/
├── pipeline.py          # Pipeline orchestrator
├── agents/
│   ├── __init__.py
│   ├── spec_agent.py    # SpecAgent
│   ├── build_agent.py   # BuildAgent
│   ├── test_agent.py    # TestAgent
│   └── review_agent.py  # ReviewAgent (optional)
├── llm/
│   ├── __init__.py
│   └── llm_wrapper.py   # LLMWrapper + ProviderConfig routing
├── errors.py            # Named error types
├── diagram.py           # HandoffDiagram generation
└── __init__.py
```

---

## Components and Interfaces

### 1. Error Types (`errors.py`)

All named error types inherit from a common `AutoSpecError` base for easy catch-all handling.

```python
class AutoSpecError(Exception):
    """Base class for all AutoSpec Pipeline errors."""

class SpecGenerationError(AutoSpecError):
    """Raised by SpecAgent when the LLM response is empty or unparseable."""

class CodeGenerationError(AutoSpecError):
    """Raised by BuildAgent when no fenced code block is found in the LLM response."""

class LLMProviderError(AutoSpecError):
    """Raised by LLMWrapper when the underlying provider SDK raises an exception."""

class ReviewGenerationError(AutoSpecError):
    """Raised by ReviewAgent when the LLM response is empty or unparseable."""

class ConfigurationError(AutoSpecError):
    """Raised by Pipeline or LLMWrapper when configuration is invalid at construction time."""
```

---

### 2. LLMWrapper (`llm/llm_wrapper.py`)

`LLMWrapper` is a provider-agnostic completion interface. It validates the supplied `ProviderConfig` dictionary at construction time and routes every `complete()` call to the appropriate SDK.

#### Interface

```python
class LLMWrapper:
    def __init__(self, provider_config: dict) -> None: ...
    def complete(self, prompt: str) -> str: ...
```

#### ProviderConfig Schemas

| `provider` value | Required keys                     |
|------------------|-----------------------------------|
| `"bedrock"`      | `model_id` (str), `region` (str)  |
| `"openai"`       | `model_id` (str), `api_key` (str) |

#### Construction-time Validation

1. Check `provider` key exists, is non-empty, and is one of `"bedrock"` or `"openai"`. Raise `ConfigurationError` otherwise.
2. Check all provider-specific required keys exist and are non-empty strings. Raise `ConfigurationError` identifying the missing key otherwise.

#### `complete(prompt)` Behaviour

1. Raise `ValueError` if `prompt` is empty (length 0) or exceeds 1,000,000 characters.
2. Delegate to the appropriate backend method.
3. Wrap any SDK exception in `LLMProviderError(provider_name, original_message)`.

#### Bedrock Backend

```python
def _complete_bedrock(self, prompt: str) -> str:
    client = boto3.client("bedrock-runtime", region_name=self._region)
    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "messages": [{"role": "user", "content": prompt}],
        "max_tokens": 8192,
    })
    response = client.invoke_model(modelId=self._model_id, body=body)
    return json.loads(response["body"].read())["content"][0]["text"]
```

#### OpenAI Backend

```python
def _complete_openai(self, prompt: str) -> str:
    client = openai.OpenAI(api_key=self._api_key)
    response = client.chat.completions.create(
        model=self._model_id,
        messages=[{"role": "user", "content": prompt}],
    )
    return response.choices[0].message.content
```

---

### 3. SpecAgent (`agents/spec_agent.py`)

Converts a plain-English brief and acceptance-criteria hints into a structured requirements dictionary.

#### Interface

```python
class SpecAgent:
    def __init__(self, llm: LLMWrapper) -> None: ...
    def run(self, brief: str, acceptance_criteria_hints: list[str]) -> dict: ...
```

#### Output Shape

```python
{
    "title": str,
    "introduction": str,
    "glossary": dict[str, str],
    "acceptance_criteria": list[str],
}
```

#### Behaviour

1. Build a prompt that embeds `brief` and `acceptance_criteria_hints`.
2. Call `llm.complete(prompt)` exactly once.
3. Attempt to parse the response as JSON (or extract a JSON block from a markdown-fenced response).
4. Validate all four required keys are present with correct types.
5. Raise `SpecGenerationError` if the response is empty, null-equivalent, or missing/malformed keys.

#### Prompt Template (simplified)

```
You are a requirements engineer. Given the following brief and hints, produce a
structured requirements document as a JSON object with exactly these keys:
  "title", "introduction", "glossary", "acceptance_criteria".

Brief:
{brief}

Acceptance Criteria Hints:
{hints_as_bullets}

Return ONLY valid JSON. No markdown fences, no commentary.
```

---

### 4. BuildAgent (`agents/build_agent.py`)

Generates source code from a structured spec dictionary.

#### Interface

```python
class BuildAgent:
    def __init__(self, llm: LLMWrapper) -> None: ...
    def run(
        self,
        spec: dict,
        tech_stack: str,          # "python" or "node"
        failure_context: str | None = None,
    ) -> str: ...
```

#### Behaviour

1. Build a prompt embedding the `spec`, `tech_stack`, and (if present) `failure_context` from a prior failed test run. The prompt instructs the LLM to use a **`python` fenced code block** when `tech_stack == "python"` and a **`javascript` fenced code block** when `tech_stack == "node"`.
2. Call `llm.complete(prompt)` exactly once.
3. Extract the content of the **first** fenced markdown code block using the regex ```` ```[\w]*\n(.*?)``` ```` (DOTALL).
4. Raise `CodeGenerationError` if no code block is found.
5. Let any `LLMProviderError` (or other SDK exception) propagate unchanged.

#### Prompt Template — Python stack

```
You are a software engineer. Using the requirements document below, write a
complete Python implementation.

Return the implementation inside a fenced python code block:
  ```python
  # your code here
  ```

Requirements:
{spec_as_json}
```

#### Prompt Template — Node stack

```
You are a software engineer. Using the requirements document below, write a
complete Node.js / JavaScript implementation.

Return the implementation inside a fenced javascript code block:
  ```javascript
  // your code here
  ```

Requirements:
{spec_as_json}
```

#### Failure Context Prompt Addendum

```
The previous test run FAILED. Below is the test output:
--- stdout ---
{stdout}
--- stderr ---
{stderr}

Please revise the code to fix the failures above.
```

---

### 5. TestAgent (`agents/test_agent.py`)

Generates a test file via LLM targeting the appropriate framework for the specified tech stack (`pytest` for Python, `Jest` for Node) and executes it against the generated code in a subprocess.

#### Interface

```python
class TestAgent:
    def __init__(self, llm: LLMWrapper, quality_threshold: float) -> None: ...
    def run(self, code: str, spec: dict, tech_stack: str) -> dict: ...
```

#### Output Shape

```python
{
    "passed": bool,
    "exit_code": int,
    "stdout": str,
    "stderr": str,
    "coverage": float | None,
}
```

#### Common Behaviour (both stacks)

1. Call `llm.complete(prompt)` once to generate a test file targeting `code` and `spec["acceptance_criteria"]`, instructing the LLM to use pytest (Python) or Jest (Node) depending on `tech_stack`.
2. Dispatch to the Python or Node execution path based on `tech_stack`.
3. Set `passed = (result.returncode == 0)`.
4. If `coverage is not None and coverage < self.quality_threshold`, override `passed = False`.
5. Clean up all temp resources in a `finally` block (see per-stack details below).

#### Python Execution Path

1. Write `code` to `tempfile.NamedTemporaryFile(suffix=".py", delete=False)` in the system temp dir → `code_file`.
2. Write test content to `tempfile.NamedTemporaryFile(prefix="test_", suffix=".py", delete=False)` in the system temp dir → `test_file`.
3. Execute:
   ```python
   result = subprocess.run(
       ["pytest", test_file.name, "--import-mode=importlib",
        "--cov", "--cov-report=term-missing", "-v"],
       capture_output=True,
       timeout=60,
       text=True,
   )
   ```
4. Parse `coverage` from the pytest-cov summary line:
   ```
   TOTAL    ...    <N>%
   ```
   Convert as `float(N) / 100`. If no such line exists, `coverage = None`.
5. In `finally`: delete `code_file.name` and `test_file.name` (both `.py` files).

#### Node Execution Path

1. Create a temporary directory: `tmp_dir = tempfile.mkdtemp()`.
2. Write `code` to `os.path.join(tmp_dir, "index.js")`.
3. Write test content to `os.path.join(tmp_dir, "index.test.js")`.
4. Write a minimal `package.json` to `os.path.join(tmp_dir, "package.json")`:
   ```json
   {
     "name": "autospec-test",
     "version": "1.0.0",
     "scripts": { "test": "jest --coverage --coverageReporters=text" },
     "devDependencies": { "jest": "*" }
   }
   ```
5. Install dependencies:
   ```python
   subprocess.run(
       ["npm", "install"],
       cwd=tmp_dir,
       capture_output=True,
       timeout=60,
   )
   ```
6. Execute tests:
   ```python
   result = subprocess.run(
       ["npx", "jest", "--coverage", "--coverageReporters=text"],
       cwd=tmp_dir,
       capture_output=True,
       timeout=60,
       text=True,
   )
   ```
7. Parse `coverage` from the Jest text coverage summary line:
   ```
   All files    ...    <N>%
   ```
   Convert as `float(N) / 100`. If no such line exists, `coverage = None`.
8. In `finally`: delete the entire temp directory recursively using `shutil.rmtree(tmp_dir)` (removes `index.js`, `index.test.js`, `package.json`, and `node_modules`).

---

### 6. ReviewAgent (`agents/review_agent.py`)

Optional agent that checks spec-to-code alignment after a passing test run.

#### Interface

```python
class ReviewAgent:
    def __init__(self, llm: LLMWrapper) -> None: ...
    def run(self, spec: dict, code: str) -> list[dict]: ...
```

#### Output Shape

A list with one entry per acceptance criterion:
```python
[
    {"criterion": str, "covered": bool, "notes": str},  # notes <= 500 chars
    ...
]
```

#### Behaviour

1. Build a prompt asking the LLM to evaluate each criterion in `spec["acceptance_criteria"]` against the `code`.
2. Call `llm.complete(prompt)` once.
3. Parse the response as a JSON list.
4. Validate each entry has `covered` (bool) and `notes` (str, max 500 chars).
5. Raise `ReviewGenerationError` if the response is empty, null-equivalent, or cannot be deserialized.

---

### 7. HandoffDiagram (`diagram.py`)

Generates a Mermaid `flowchart LR` string documenting the actual data flow of a pipeline run.

#### Interface

```python
def build_handoff_diagram(
    agents_invoked: list[str],
    retry_count: int,
    enable_review_agent: bool,
) -> str: ...
```

#### Diagram Rules

| Condition | Nodes / Edges included |
|---|---|
| Always | `PipelineInputs → SpecAgent → BuildAgent → TestAgent` |
| `retry_count > 0` | `TestAgent →\|"retry (N)"\| BuildAgent` |
| `enable_review_agent=True` and ReviewAgent ran | `TestAgent → ReviewAgent` |
| Early halt after SpecAgent error | Only `PipelineInputs → SpecAgent` |

#### Example Output (successful run with 2 retries, no review)

```
flowchart LR
    PipelineInputs --> SpecAgent
    SpecAgent --> BuildAgent
    BuildAgent --> TestAgent
    TestAgent -->|"retry (2)"| BuildAgent
```

---

### 8. Pipeline (`pipeline.py`)

The top-level orchestrator.

#### Interface

```python
class Pipeline:
    def __init__(self, config: dict) -> None: ...
    def run(
        self,
        brief: str,
        acceptance_criteria_hints: list[str],
        tech_stack: str,
        quality_threshold: float,
        max_retries: int,
    ) -> dict: ...
```

#### Result Dictionary

```python
{
    "spec": dict | None,
    "code": str | None,
    "test_result": dict | None,
    "handoff_diagram": str,        # always present
    "retry_count": int,            # 0 on first-run pass or early error
    "retry_exhausted": bool,       # True only when max_retries reached without passing
    "error": str | None,           # exception string on failure, else None
    "review": list[dict] | None,   # ReviewAgent output, or None
}
```

#### Construction-time Validation

Required keys checked at `__init__` time:

| Key | Type | Constraint |
|-----|------|-----------|
| `provider` | str | non-empty |
| `model_id` | str | non-empty |
| `max_retries` | int | 1–10 inclusive |
| `quality_threshold` | float | 0.0–1.0 inclusive |
| `enable_review_agent` | bool | — |
| `region` *(Bedrock only)* | str | non-empty |
| `api_key` *(OpenAI only)* | str | non-empty |

Raise `ConfigurationError` naming the offending key on any violation.

Keys passed to `LLMWrapper`: `provider`, `model_id`, plus provider-specific credential key(s).

#### `run` Method — Execution Flow

```
validate run-time inputs
│
├─ SpecAgent.run(brief, hints)      → spec
│     [on exception → return error result]
│
├─ BuildAgent.run(spec, tech_stack) → code
│     [on exception → return error result]
│
├─ TestAgent.run(code, spec, tech_stack)        → test_result
│     [on exception → return error result]
│
├─ RETRY LOOP (while not passed and retry_count < max_retries):
│     retry_count += 1
│     failure_context = test_result["stdout"] + test_result["stderr"]
│     BuildAgent.run(spec, tech_stack, failure_context) → code
│         [on exception → return error result, retry_exhausted=False]
│     TestAgent.run(code, spec, tech_stack)     → test_result
│         [on exception → return error result]
│
├─ (if retry_count == max_retries and not passed) → retry_exhausted = True
│
├─ (if enable_review_agent and passed):
│     ReviewAgent.run(spec, code)   → review
│
└─ build_handoff_diagram(...)       → handoff_diagram
   return result dict
```

#### Run-time Input Validation

| Parameter | Constraint |
|-----------|-----------|
| `brief` | non-empty str |
| `acceptance_criteria_hints` | list[str] |
| `tech_stack` | one of `"python"`, `"node"` |
| `quality_threshold` | float, 0.0–1.0 |
| `max_retries` | int, 1–10 |

Raise `ValueError` (or `ConfigurationError`) for invalid run-time inputs before invoking any agent.

---

## Data Models

### ProviderConfig

```python
# Bedrock
{
    "provider": "bedrock",
    "model_id": "anthropic.claude-3-5-sonnet-20241022-v2:0",
    "region": "us-east-1",
}

# OpenAI
{
    "provider": "openai",
    "model_id": "gpt-4o",
    "api_key": "sk-...",
}
```

### Spec Dictionary

```python
{
    "title": "Task Manager API",
    "introduction": "A RESTful service for managing tasks...",
    "glossary": {
        "Task": "A unit of work with a title, status, and optional due date.",
    },
    "acceptance_criteria": [
        "GIVEN a valid task payload WHEN POST /tasks is called THEN a task is created and returned with a 201 status.",
        "WHEN GET /tasks is called THEN all existing tasks are returned as a JSON array.",
    ],
}
```

### Pipeline Result Dictionary

```python
{
    "spec": {...},
    "code": "def create_task(...):\n    ...",
    "test_result": {
        "passed": True,
        "exit_code": 0,
        "stdout": "...",
        "stderr": "",
        "coverage": 0.87,
    },
    "handoff_diagram": "flowchart LR\n    PipelineInputs --> SpecAgent\n    ...",
    "retry_count": 1,
    "retry_exhausted": False,
    "error": None,
    "review": None,
}
```

---

## Error Handling

| Scenario | Raised By | Caught By | Effect |
|---|---|---|---|
| Invalid provider in ProviderConfig | `LLMWrapper.__init__` | `Pipeline.__init__` or caller | `ConfigurationError` |
| Missing ProviderConfig key | `LLMWrapper.__init__` | `Pipeline.__init__` or caller | `ConfigurationError` |
| Empty / invalid prompt | `LLMWrapper.complete` | Agent | `ValueError` propagates |
| SDK network / auth error | `LLMWrapper.complete` | Agent | `LLMProviderError` propagates to Pipeline |
| Empty/unparseable LLM response (spec) | `SpecAgent.run` | `Pipeline.run` | `error` set, remaining keys `None` |
| No code block in response | `BuildAgent.run` | `Pipeline.run` | `error` set, remaining keys `None` |
| Empty/unparseable LLM response (review) | `ReviewAgent.run` | `Pipeline.run` | `error` set |
| pytest / Jest timeout | `subprocess.run` | `TestAgent.run` | `TimeoutExpired` → `TestAgent` propagates |
| Missing Pipeline config key | `Pipeline.__init__` | Caller | `ConfigurationError` |
| Invalid config value/type | `Pipeline.__init__` | Caller | `ConfigurationError` |
| BuildAgent exception during retry | `BuildAgent.run` | `Pipeline.run` | `error` set, `retry_exhausted=False` |

All agents clean up temp resources in `finally` blocks regardless of exception path. For Python stack: both `.py` temp files are deleted. For Node stack: the entire temp directory is removed recursively via `shutil.rmtree`.

---

## Sequence Diagrams

### Happy Path (no retries, no review)

```
Caller → Pipeline.run(brief, hints, tech_stack, threshold, max_retries)
  Pipeline → SpecAgent.run(brief, hints)
    SpecAgent → LLMWrapper.complete(prompt)
    LLMWrapper → Bedrock/OpenAI API
    ← response
    SpecAgent parses JSON → spec dict
  ← spec
  Pipeline → BuildAgent.run(spec, tech_stack)
    BuildAgent → LLMWrapper.complete(prompt)
    ← response with code block
    BuildAgent extracts code
  ← code
  Pipeline → TestAgent.run(code, spec, tech_stack)
    TestAgent → LLMWrapper.complete(prompt) → test content
    TestAgent writes temp files
    TestAgent → subprocess pytest / npx jest
    TestAgent parses result, deletes temp files
  ← test_result {passed: True}
  Pipeline → build_handoff_diagram(...)
← result dict
```

### Retry Path (1 retry, success on second attempt)

```
Pipeline → TestAgent → {passed: False}
  retry_count = 1
  Pipeline → BuildAgent.run(spec, tech_stack, failure_context)
  ← new_code
  Pipeline → TestAgent.run(new_code, spec, tech_stack)
  ← test_result {passed: True}
  retry_exhausted = False
Pipeline → build_handoff_diagram(retry_count=1)
```

### Retry Exhausted Path

```
# max_retries = 3, all fail
retry_count reaches 3
retry_exhausted = True
Pipeline → build_handoff_diagram(retry_count=3)
← result dict with retry_exhausted=True, passed=False
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

---

### Property 1: Pipeline Result Keys Are Always Present

*For any* pipeline run with valid inputs (successful, failed, or error-halted), the returned dictionary SHALL contain all required keys: `spec`, `code`, `test_result`, `handoff_diagram`, `retry_count`, `retry_exhausted`, and `error`.

**Validates: Requirements 1.3, 1.4**

---

### Property 2: Exception Capture Sets Error and Nulls Subsequent Keys

*For any* agent that raises an exception at any stage of the pipeline, the returned result SHALL have `error` set to a non-empty string and all keys corresponding to outputs of agents that were not yet invoked SHALL be `None`.

**Validates: Requirements 1.4**

---

### Property 3: SpecAgent Output Conforms to Required Schema

*For any* non-empty `brief` and any list of `acceptance_criteria_hints`, when the LLMWrapper returns a valid parseable response, `SpecAgent.run` SHALL return a dictionary containing exactly the keys `title`, `introduction`, `glossary`, and `acceptance_criteria` with the correct types.

**Validates: Requirements 2.1, 2.3**

---

### Property 4: SpecGenerationError on Empty or Unparseable Response

*For any* LLMWrapper response that is empty (zero-length or null-equivalent) or is not parseable into the required schema, `SpecAgent.run` SHALL raise a `SpecGenerationError` with a message identifying the failure reason.

**Validates: Requirements 2.4**

---

### Property 5: BuildAgent Code Block Extraction

*For any* LLM response string containing one or more fenced markdown code blocks, `BuildAgent.run` SHALL return the content of the **first** code block as a non-empty string, regardless of how many blocks are present or where they appear.

**Validates: Requirements 3.1, 3.3**

---

### Property 6: CodeGenerationError on Missing Code Block

*For any* LLM response string that contains no fenced markdown code block delimiters, `BuildAgent.run` SHALL raise a `CodeGenerationError` stating that no code block was found.

**Validates: Requirements 3.4**

---

### Property 7: LLMWrapper Exception Propagation from BuildAgent

*For any* exception type raised by `LLMWrapper.complete` during a `BuildAgent.run` call, the exception SHALL propagate to the caller without wrapping or suppression.

**Validates: Requirements 3.6**

---

### Property 8: TestAgent Result Shape Invariant

*For any* tech stack (`"python"` or `"node"`) and any test execution outcome (exit code 0, 1, 2, or other), `TestAgent.run` SHALL return a dictionary containing exactly the keys `passed` (bool), `exit_code` (int), `stdout` (str), `stderr` (str), and `coverage` (float or `None`).

**Validates: Requirements 4.6**

---

### Property 9: TestAgent Uses Correct Test Runner per Tech Stack

*For any* `tech_stack` value, `TestAgent.run` SHALL invoke the correct test runner: `pytest` (with `--import-mode=importlib` and `--cov`) when `tech_stack` is `"python"`, and `npx jest` (with `--coverage --coverageReporters=text`) when `tech_stack` is `"node"`. No other test runner SHALL be invoked for either stack.

**Validates: Requirements 4.4, 4.5**

---

### Property 10: Coverage Below Threshold Forces Passed = False

*For any* test runner exit code and any `coverage` value that is not `None` and is strictly less than the `quality_threshold` supplied at `TestAgent` construction time, the returned result dictionary SHALL have `passed` set to `False`.

**Validates: Requirements 4.9**

---

### Property 11: Temp Files Always Deleted

*For any* execution outcome of `TestAgent.run` — including successful test runs, failed test runs, and runs that raise an exception:

- When `tech_stack` is `"python"`: both temporary `.py` files (the code file and the `test_*.py` file) SHALL be deleted from the filesystem.
- When `tech_stack` is `"node"`: the entire temporary directory (including `index.js`, `index.test.js`, `package.json`, and the `node_modules` folder) SHALL be deleted recursively via `shutil.rmtree`.

Cleanup SHALL occur before `run` returns or propagates an exception.

**Validates: Requirements 4.10, 4.11**

---

### Property 12: Retry Count Accurately Reflects Loop Iterations

*For any* sequence of BuildAgent/TestAgent outcomes and any `max_retries` value in [1, 10], the `retry_count` in the pipeline result SHALL equal the exact number of times the retry loop body executed (0 on a first-run pass, up to `max_retries` on full exhaustion).

**Validates: Requirements 5.4, 5.6**

---

### Property 13: Retry Exhaustion Sets retry_exhausted = True

*For any* `max_retries` value where all BuildAgent/TestAgent retry cycles produce `passed = False`, the pipeline result SHALL have `retry_exhausted = True` and `retry_count = max_retries`.

**Validates: Requirements 5.3**

---

### Property 14: LLMWrapper Rejects Invalid Provider at Construction

*For any* `ProviderConfig` dictionary where `provider` is absent, empty, or not one of `"bedrock"` or `"openai"`, `LLMWrapper.__init__` SHALL raise a `ConfigurationError` before any API call is made.

**Validates: Requirements 6.4**

---

### Property 15: LLMWrapper Rejects Missing Required Keys at Construction

*For any* `ProviderConfig` where any required key for the selected provider is absent or empty, `LLMWrapper.__init__` SHALL raise a `ConfigurationError` identifying the missing key.

**Validates: Requirements 6.5**

---

### Property 16: SDK Exceptions Wrapped in LLMProviderError

*For any* exception type raised by the underlying provider SDK (boto3 or OpenAI) during a `complete` call, `LLMWrapper` SHALL raise an `LLMProviderError` containing the provider name and the original exception message.

**Validates: Requirements 6.6**

---

### Property 17: Prompt Length Validation in LLMWrapper

*For any* prompt that is empty (length 0) or exceeds 1,000,000 characters, `LLMWrapper.complete` SHALL raise a `ValueError` without making any API call.

**Validates: Requirements 6.7**

---

### Property 18: ReviewAgent Alignment Report Shape

*For any* `spec` dictionary with N acceptance criteria and any non-empty `code` string, when the LLMWrapper returns a parseable response, `ReviewAgent.run` SHALL return a list of exactly N entries each containing a `covered` field of type `bool` and a `notes` field of type `str` with length ≤ 500 characters.

**Validates: Requirements 7.2**

---

### Property 19: Partial Coverage Does Not Fail Pipeline

*For any* alignment report produced by `ReviewAgent` in which one or more entries have `covered = False`, the pipeline run SHALL not set the `error` key and SHALL continue to completion.

**Validates: Requirements 7.5**

---

### Property 20: HandoffDiagram Contains Correct Nodes for Actual Run

*For any* pipeline run, the `handoff_diagram` string SHALL be a valid Mermaid `flowchart LR` and SHALL contain a node for every agent that was invoked during the run, and SHALL NOT contain a node for any agent that was not invoked.

**Validates: Requirements 8.1, 8.4**

---

### Property 21: HandoffDiagram Retry Edge

*For any* pipeline run where `retry_count > 0`, the `handoff_diagram` string SHALL contain exactly one directed edge from `TestAgent` to `BuildAgent` with a label of the form `"retry (N)"` where N equals `retry_count`.

**Validates: Requirements 8.2**

---

### Property 22: Pipeline Rejects Invalid Configuration at Construction

*For any* configuration dictionary missing a required key, or containing a key with the wrong type or an out-of-range value, `Pipeline.__init__` SHALL raise a `ConfigurationError` naming the offending key before any agent or LLMWrapper is constructed.

**Validates: Requirements 9.2, 9.3**

---

## Testing Strategy

### Dual Testing Approach

Both unit/example-based tests and property-based tests are used in combination for comprehensive coverage.

**Unit / example-based tests** cover:
- Specific happy-path examples for each agent
- Agent invocation order (using mock call recording)
- Subprocess call parameter verification
- Integration between Pipeline and LLMWrapper (key forwarding)

**Property-based tests** (using Hypothesis) cover all 22 properties above with a minimum of 100 iterations each. Each property test is tagged:

```
Feature: autospec-pipeline, Property {N}: {property_title}
```

### Key Testing Patterns

- **Mock `LLMWrapper`**: All agent tests inject a `MagicMock` for `LLMWrapper` to avoid real API calls.
- **Temp file cleanup** (Property 10): Verified by checking `os.path.exists(path)` (Python) or `os.path.exists(tmp_dir)` (Node) after `TestAgent.run` in both passing and exception scenarios.
- **Test runner selection** (Property 8a): Verified with `unittest.mock.patch("subprocess.run")` to assert the correct command list (`["pytest", ...]` vs `["npx", "jest", ...]`) is passed for each `tech_stack` value.
- **Retry loop** (Properties 11, 12): Controlled via a side-effect list on the mocked `TestAgent` to produce a predetermined pass/fail sequence.
- **Coverage override** (Property 9): Hypothesis generates `(exit_code, coverage, threshold)` triples; asserts `passed == (exit_code == 0 and (coverage is None or coverage >= threshold))`.
- **Prompt length** (Property 16): Hypothesis generates strings of length 0 and lengths > 1,000,000 to verify `ValueError`.
