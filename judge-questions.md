# ForgeSpec — Judge Q&A Preparation

> **Rule:** every answer must be deliverable in under 25 seconds.
> Never claim something works unless it exists in the repository.
> If something is partially built or aspirational, say so plainly.

---

## Q1: Is the code actually generated?

Yes. The Code Agent produces source files during the pipeline run.
The files exist in the repository — you can open them, read them,
and run them. They are not templates with variables swapped in;
they are generated from the spec that the Spec Agent produced.

---

## Q2: Are the tests actually executed?

Yes. The Test Agent calls pytest as a subprocess and captures the
stdout. The terminal output you see in the demo is a live process,
not a screenshot. The saved output in `docs/recorded-run/` is the
captured result of a real prior execution.

---

## Q3: Why use multiple agents instead of one?

Single-agent systems have no internal checkpoints. If the model drifts,
there is nothing to catch it. Specialised agents create explicit
handoff points — the Code Agent cannot proceed without a valid spec,
and the Validation Agent cannot report success without test results.
Each boundary is an auditable gate.

---

## Q4: What happens when tests fail?

The pipeline halts. The Validation Agent reports which acceptance
criteria have no passing test. The system does not produce a
"Ready to Ship" result. The failure is logged with the specific
test name and the criterion it maps to, so the developer knows
exactly what to fix.

---

## Q5: What is Kiro doing here?

Kiro is the IDE and agent orchestration layer. It provides the
chat interface where the brief is entered, manages the agent
execution loop, gives each agent access to the file system,
and surfaces the artefacts in the editor as they are created.
ForgeSpec is built on top of Kiro's agent infrastructure.

---

## Q6: What is mocked or simulated?

The SQLite database is a real local database — no mock.
The pytest run is a real process execution — no mock.
The only thing that could be considered a simplification is the
scope of the demo brief: it targets a single endpoint to keep
the demo within 90 seconds. The pipeline handles larger briefs
by the same mechanism.

---

## Q7: How would this scale to a real project?

The brief input scales naturally — a larger brief produces a larger
spec, which produces more code and more tests. For enterprise scale,
the agents would run in a CI pipeline rather than an IDE. The
traceability matrix becomes the audit artefact for each PR.
The architecture does not change; only the execution environment does.

---

## Q8: How do you prevent hallucinated success?

The Validation Agent reads the actual pytest exit code, not the
test file content. A zero exit code with all tests collected and
passing is required before "Ready to Ship" is issued. The agent
cannot self-report success — it must parse real process output.

---

## Q9: Why is this useful for consulting teams specifically?

Consulting teams bill on delivery. Rework caused by requirement
drift is unrecoverable time. ForgeSpec produces a spec before code,
so requirement disagreements surface in minutes rather than in
sprint review. The traceability matrix is also a client deliverable —
it demonstrates due diligence without additional documentation effort.

---

## Q10: What would you build next?

Three things: first, a requirement-conflict detector in the Spec Agent
that flags contradictions before code generation starts. Second,
integration with an existing ticket system so the spec auto-populates
Jira or Linear. Third, a regression mode where the pipeline re-runs
against a changed brief and highlights which existing tests are now
invalidated.
