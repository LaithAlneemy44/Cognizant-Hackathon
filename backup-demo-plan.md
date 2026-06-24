# ForgeSpec — Backup Demo Plan

> **When to use:** network timeout, Kiro agent lag, IDE crash, or any
> condition that would cause the live demo to stall for more than 10 seconds.
>
> **How to trigger:** the presenter decides. Do not wait. Switch immediately.

---

## The Mindset

A backup demo is not a failure state. It is a design choice.
Deterministic replay of a verified run is more rigorous than a
live execution that depends on network latency and API availability.

Frame it that way.

---

## Opening Line for the Handoff

When switching to the backup, say exactly:

> "To keep the demonstration deterministic, I'll replay the last verified
> run. These artifacts and test results were generated and executed by
> the same pipeline you just heard described."

No apology. No explanation of what went wrong. Move directly to the artefacts.

---

## What to Show (in order)

### Step 1 — The Brief

Open `docs/recorded-run/00-brief.txt` (or paste the brief from the demo
script into a plain text editor).

> **Say:**
> "This is the input. One plain-English sentence."

### Step 2 — The Spec

Open `docs/recorded-run/01-spec.md` or `specs/invoices-spec.md`.

Scroll through it slowly. Point out:
- Numbered acceptance criteria
- Data contract / schema section
- Edge cases explicitly listed

> **Say:**
> "The Spec Agent produced this before any code was written. Every
> downstream artefact traces back to a numbered item here."

### Step 3 — The Generated Code

Open `docs/recorded-run/02-implementation.py` or the relevant source file.

Scroll to show:
- Route handler
- Validation logic
- Database interaction

> **Say:**
> "The Code Agent read only the spec. The implementation matches the
> acceptance criteria line for line."

### Step 4 — The Test Output

Open `docs/recorded-run/03-test-output.txt`.

This is the captured pytest terminal output from the verified run.
Read the final summary line aloud:

> **Say:**
> "Seven tests collected. Seven passed. Zero failures. This ran on the
> same pipeline — the only difference is I'm showing you the saved output
> rather than triggering a new run in real time."

### Step 5 — The Traceability Matrix

Open `docs/recorded-run/04-traceability-matrix.md`.

Point to the table. Read two rows.

> **Say:**
> "Every requirement has a test. Every test has a result.
> This is the evidence layer that AI coding tools currently skip."

---

## Closing Line (same as live demo)

> "Spec generated. Code generated. Tests executed. Traceability confirmed.
> ForgeSpec: Ready to Ship."

---

## Preparation Checklist for the Recorded Run

Before the presentation, ensure these files exist and are accurate:

- [ ] `docs/recorded-run/00-brief.txt` — the exact brief used
- [ ] `docs/recorded-run/01-spec.md` — the spec the agent produced
- [ ] `docs/recorded-run/02-implementation.py` — the generated source
- [ ] `docs/recorded-run/03-test-output.txt` — captured pytest stdout
- [ ] `docs/recorded-run/04-traceability-matrix.md` — the traceability table

All five files must be committed to the repo and openable offline.

---

## If Everything Fails (nuclear option)

If the machine itself is unavailable, use the slide deck.
The slide deck must contain screenshots of all five artefacts above,
plus the pytest green output.

> **Say:**
> "I'll walk you through the artefacts on screen — same content,
> static format."

Then narrate each slide using the same script as Step 1–5 above.
