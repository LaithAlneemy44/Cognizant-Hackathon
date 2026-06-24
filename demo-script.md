# ForgeSpec — Live Demo Script (90 Seconds)

> **Presenter notes:** rehearse until the timing is muscle memory.
> Each section shows elapsed time, exact words, and exact actions.

---

## Pre-Demo Setup (done before judges sit down)

| Action | Detail |
|--------|--------|
| Browser tab open | Kiro IDE with ForgeSpec workspace loaded |
| Chat panel visible | Agent chat ready, empty input field |
| File explorer collapsed | So the tree reveal lands with impact |
| Terminal panel hidden | Reveal it only at test-execution moment |
| Font size | 18 pt minimum — readable from the back of the room |

---

## 0:00 — Opening Sentence

> **Say exactly:**
> "Most AI coding tools generate code. ForgeSpec generates *proof*."

Pause one beat. Let it land.

---

## 0:05 — Product Brief Paste

Click the Kiro chat input field.

**Paste exactly this text:**

```
Build a REST API endpoint POST /invoices that validates required fields
(client_id, amount, due_date), persists to a SQLite database, returns
201 with the created invoice object, and returns 422 with field-level
errors on invalid input. Include unit tests that cover both the happy
path and every validation failure case.
```

> **Say while pasting:**
> "One plain-English sentence. No tickets, no Jira, no architecture meeting."

Hit **Enter / Send**.

---

## 0:15 — Spec Agent Executes

> **Say while the Spec Agent runs:**
> "The Spec Agent is decomposing that brief into a structured requirements
> document — acceptance criteria, edge cases, data contracts — before a
> single line of code is written."

**Action:** Watch the file explorer. When `specs/invoices-spec.md` appears,
click it open briefly so the audience sees the formatted spec.

> **Say:**
> "Structured spec. Traceable. Reviewable. Not just a comment in a file."

---

## 0:30 — Code Agent Executes

> **Say:**
> "The Code Agent reads only that spec. It does not hallucinate requirements
> it was never given."

**Action:** When `src/invoices.py` (or equivalent) appears in the explorer,
click it open. Scroll slowly — let them see the route handler, the
validation logic, the database layer.

> **Say:**
> "Full implementation. Typed. Documented. Generated against the spec,
> not against a vague prompt."

---

## 0:50 — Test Agent Executes

> **Action:** Open the terminal panel (Ctrl+\` or click Terminal).

> **Say:**
> "The Test Agent now writes and *executes* the test suite. Watch the
> terminal — this is a real pytest run, not a screenshot."

**Action:** Watch for pytest output. When green lines appear, point to the screen.

> **Say while tests pass:**
> "Happy path — pass. Missing client ID — pass. Invalid amount — pass.
> Malformed date — pass."

---

## 1:10 — Highlight Metrics

Point to the terminal output.

> **Say:**
> "100% of generated tests passing. Coverage report attached. Every test
> maps back to a numbered acceptance criterion in the spec."

**Action:** Quickly flip to the `reports/traceability-matrix.md` file
(or `specs/invoices-spec.md` if that is where coverage is embedded).

> **Say:**
> "This is the traceability matrix. Requirement to test. Test to result.
> End to end."

---

## 1:25 — Final Sentence

Close all panels. Face the judges.

> **Say exactly:**
> "Spec generated. Code generated. Tests executed. Traceability confirmed.
> ForgeSpec says: *Ready to Ship*."

---

## Timing Summary

| Mark | Event |
|------|-------|
| 0:00 | Opening line |
| 0:05 | Paste brief, hit send |
| 0:15 | Spec file opens |
| 0:30 | Source file opens |
| 0:50 | Terminal opens, tests run |
| 1:10 | Metrics called out |
| 1:25 | Closing line |
| 1:30 | Done |

---

## Things to Never Say During the Demo

- "Hopefully this works…"
- "It usually does this faster…"
- "Let me try that again…"
- "The backend is a bit slow today…"

If anything stalls, move immediately to the **Backup Demo Plan**.
