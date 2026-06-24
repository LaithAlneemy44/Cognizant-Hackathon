# AutoSpec Pipeline — Agent Handoff Diagram

This diagram documents the data flow and handoff contracts between agents in the AutoSpec Pipeline.

To view on mermaid.live: copy the content of any code block below (starting from `flowchart`) and paste it into the editor at https://mermaid.live

---

## Full Pipeline Flow

```mermaid
flowchart TD
    INPUT["Pipeline Inputs"]
    SPEC["SpecAgent"]
    BUILD["BuildAgent"]
    TEST["TestAgent"]
    REVIEW["ReviewAgent (Optional)"]
    OUTPUT["Pipeline Result"]
    LLM["LLMWrapper"]

    INPUT -->|"brief + hints"| SPEC
    SPEC -->|"spec dict"| BUILD
    BUILD -->|"code"| TEST
    TEST -->|"passed=True"| REVIEW
    TEST -->|"passed=False + failure_context"| BUILD
    REVIEW -->|"alignment report"| OUTPUT
    TEST -->|"passed=True, no review"| OUTPUT
    SPEC -.->|"LLM call"| LLM
    BUILD -.->|"LLM call"| LLM
    TEST -.->|"LLM call"| LLM
    REVIEW -.->|"LLM call"| LLM
```

---

## Retry Loop

```mermaid
flowchart LR
    TA["TestAgent"]
    BA["BuildAgent"]
    DONE["Exit Loop"]

    TA -->|"passed=False + stdout/stderr"| BA
    BA -->|"revised code"| TA
    TA -->|"passed=True or retries exhausted"| DONE
```

---

## Error Handling Paths

```mermaid
flowchart TD
    SA["SpecAgent"] -->|"SpecGenerationError"| ERR["Pipeline: set error, halt"]
    BA["BuildAgent"] -->|"CodeGenerationError or LLMProviderError"| ERR
    TA["TestAgent"] -->|"TimeoutExpired or exception"| ERR
    RA["ReviewAgent"] -->|"ReviewGenerationError"| ERR
    LLM["LLMWrapper"] -->|"ConfigurationError at construction"| CONF["Pipeline raises before agents run"]
```

---

## LLMWrapper Provider Routing

```mermaid
flowchart LR
    AGENT["Any Agent"]
    WRAPPER["LLMWrapper"]
    BEDROCK["AWS Bedrock - boto3/Claude"]
    OPENAI["OpenAI API - openai SDK"]

    AGENT -->|"complete(prompt)"| WRAPPER
    WRAPPER -->|"provider=bedrock"| BEDROCK
    WRAPPER -->|"provider=openai"| OPENAI
    BEDROCK -->|"response text"| WRAPPER
    OPENAI -->|"response text"| WRAPPER
    WRAPPER -->|"str"| AGENT
```

---

## Agent Handoff Contracts

| From | To | Payload | Condition |
|------|----|---------|-----------|
| Pipeline Inputs | SpecAgent | `brief`, `acceptance_criteria_hints` | Always |
| SpecAgent | BuildAgent | `spec` dict: title, introduction, glossary, acceptance_criteria | On success |
| BuildAgent | TestAgent | `code` (str) | On success |
| TestAgent | BuildAgent | `failure_context` = stdout + stderr | When `passed=False` and retries remain |
| TestAgent | ReviewAgent | `code` + `spec` | When `passed=True` and review enabled |
| TestAgent | Pipeline Output | `test_result` dict | When `passed=True` or retries exhausted |
| ReviewAgent | Pipeline Output | `review` list of criterion/covered/notes | When ReviewAgent runs |

---

## Module Structure

```
autospec_pipeline/
├── pipeline.py          — Orchestrator: sequences agents, manages retry loop
├── errors.py            — AutoSpecError and all named subclasses
├── diagram.py           — build_handoff_diagram() Mermaid string builder
├── agents/
│   ├── spec_agent.py    — Brief → structured spec dict
│   ├── build_agent.py   — Spec dict → source code string
│   ├── test_agent.py    — Code → pytest execution → pass/fail + coverage
│   └── review_agent.py  — Spec + code → alignment report (optional)
└── llm/
    └── llm_wrapper.py   — Routes to Bedrock or OpenAI based on config
main.py                  — CLI entry-point
```
