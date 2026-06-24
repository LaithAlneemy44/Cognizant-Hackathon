# ForgeSpec — Judge Q&A Preparation

Prepared answers for likely questions from technical AWS judges.

---

## Architecture & Design

**Q: Why use multiple agents instead of a single large model?**

A single model asked to go from brief to code in one pass loses the structured thinking that makes software traceable. Each stage of software development — requirements analysis, architecture, implementation, testing, review — is a distinct discipline with distinct inputs and outputs. Separating them into specialised agents means each stage can be evaluated independently, failures are localised rather than buried, and every intermediate artifact is inspectable. It also mirrors how competent engineering teams actually work.

---

**Q: How does the pipeline handle ambiguous input?**

Blueprint is specifically designed to surface ambiguity rather than paper over it. When the input brief contains underspecified requirements, Blueprint flags them explicitly in the requirements document. Downstream agents — Atlas, Forge, Sentinel — work from that document, so any ambiguity identified early is resolved before code is written rather than silently assumed.

---

**Q: What is the data flow between agents?**

Each agent produces a structured artifact that becomes the input for the next stage:

- Brief → **Blueprint** → Requirements document
- Requirements document → **Atlas** → Architecture document
- Architecture document → **Forge** → Implementation
- Requirements document + Implementation → **Sentinel** → Test suite and results
- All prior artifacts → **Veritas** → Audit report

No agent receives raw prose except Blueprint. Every subsequent agent works from structured, machine-readable outputs.

---

**Q: How does ForgeSpec use AWS services?**

ForgeSpec is designed to run on AWS infrastructure. The agent pipeline is orchestrated as a serverless workflow, making it cost-effective for variable workloads. Model inference runs through Amazon Bedrock, giving access to foundation models without infrastructure management. Artifacts are stored in S3 with versioning enabled, providing the persistence layer for the audit trail. The architecture is stateless by design — each pipeline run is isolated and reproducible.

---

## Traceability & Quality

**Q: What does "traceable" actually mean in practice?**

Every requirement in the Blueprint output carries an identifier. Atlas references those identifiers when describing architectural decisions. Forge annotates implementation against requirement IDs. Sentinel maps each test to the requirement it validates. Veritas checks that every requirement has a corresponding implementation and a passing test. The audit report shows the full chain: from the original brief sentence to the line of code that satisfies it.

---

**Q: How do you validate that the code actually meets requirements — not just that tests pass?**

Test passage is a necessary condition, not a sufficient one. Veritas performs a semantic review: it reads the requirements document and the implementation side by side, checking for functional completeness. A feature that was specified but never implemented would pass a coverage check silently. Veritas catches that gap because it works from requirements outward, not from code inward.

---

**Q: What happens when a test fails?**

A test failure is surfaced in the Sentinel output with the failing test, the targeted requirement, and the observed versus expected behaviour. Veritas notes any failures in the audit report and marks the associated requirements as unverified. The pipeline does not suppress failures — it makes them explicit and traceable.

---

## Product & Market

**Q: Who is the target user?**

Two primary profiles:

1. **Individual developers and small teams** who need to move quickly from idea to working software without the overhead of a formal specification process.
2. **Engineering teams inside organisations** that need auditability — regulated industries, government contractors, teams with strict change-management requirements.

Both benefit from the same core output: structured, traceable software. The first values speed; the second values the audit trail.

---

**Q: How is this different from GitHub Copilot or Cursor?**

Copilot and Cursor accelerate code writing. They work at the line and function level, inside a file the developer has already decided to create.

ForgeSpec operates at a different abstraction layer. It starts before the first file exists — at the point where requirements are still ambiguous — and produces a complete, tested software artifact with a documented specification. It is not a coding assistant. It is a software factory.

---

**Q: What is the monetisation model?**

ForgeSpec is positioned as a usage-based SaaS product. Customers pay per pipeline run, with pricing tiers based on project complexity and output size. Enterprise customers can negotiate reserved capacity with SLA guarantees. The audit trail output has particular value in regulated industries, which supports a premium tier.

---

**Q: What is the biggest technical risk?**

The honest answer is hallucination in the requirements stage. If Blueprint misinterprets a brief, every downstream agent works from a flawed foundation. The mitigation is explicit ambiguity flagging — Blueprint is prompted to surface uncertainty rather than resolve it silently — combined with Veritas closing the loop against the original brief text. A human review of the Blueprint output before advancing the pipeline is recommended for high-stakes projects.

---

**Q: What does the roadmap look like?**

Near term: support for iterative refinement — the ability to revise requirements and re-run the pipeline from any stage without starting over. Medium term: integration with existing repositories so ForgeSpec can extend a codebase rather than only generate new ones. Longer term: feedback loops where Veritas findings are used to improve Blueprint prompting over time.

---

## Demo & Live Questions

**Q: Can you walk us through a live example?**

Yes. We submit a brief — for example: *"Build a REST API that manages a to-do list with user authentication."* Blueprint produces a requirements document with functional requirements and acceptance criteria. Atlas designs the API structure, data model, and auth flow. Forge implements the endpoints. Sentinel runs tests against the auth and CRUD behaviour. Veritas confirms every requirement was addressed. The full pipeline completes in under two minutes and produces a complete, reviewable artifact set.

---

**Q: What would you build differently if you started over?**

We would invest earlier in the schema design for inter-agent artifacts. The structured handoff format between agents is the most critical piece of the architecture — it is what makes the pipeline composable and auditable. Early versions used less structured formats, which made Veritas's job harder. Tightening those schemas up front would have saved significant iteration time.
