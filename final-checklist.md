# ForgeSpec — Final Presentation Checklist

> Complete every item before entering the judging room.
> Check in order. Do not skip sections.

---

## 1. Repository State

- [ ] All source files committed and pushed to `main`
- [ ] `docs/demo-script.md` present and current
- [ ] `docs/pitch.md` present and current
- [ ] `docs/backup-demo-plan.md` present and current
- [ ] `docs/judge-questions.md` present and current
- [ ] `docs/final-checklist.md` present and current
- [ ] `docs/recorded-run/` directory contains all five backup artefacts:
  - [ ] `00-brief.txt`
  - [ ] `01-spec.md`
  - [ ] `02-implementation.py`
  - [ ] `03-test-output.txt`
  - [ ] `04-traceability-matrix.md`
- [ ] README.md updated with project description and run instructions
- [ ] No uncommitted changes (`git status` is clean)

---

## 2. Live Demo Environment

- [ ] Kiro IDE installed and licensed on the demo machine
- [ ] ForgeSpec workspace open in Kiro
- [ ] Agent pipeline verified to run end-to-end on the demo brief
  - [ ] Spec Agent produces `specs/` file
  - [ ] Code Agent produces `src/` file
  - [ ] Test Agent runs pytest and output is visible in terminal
  - [ ] Validation Agent produces traceability output
- [ ] Font size set to 18 pt or larger
- [ ] Display resolution confirmed readable on the room's screen
- [ ] Chat panel empty and ready for input
- [ ] File explorer collapsed (tree reveal is part of the demo)
- [ ] Terminal panel hidden until the test-execution moment

---

## 3. Backup Demo Environment

- [ ] All `docs/recorded-run/` files openable offline (no network required)
- [ ] Files openable in the IDE without internet access
- [ ] Presenter knows the keyboard shortcut to open each file quickly
- [ ] Backup slide deck exists (screenshots of all five artefacts)
- [ ] Slide deck openable without internet

---

## 4. Machine Health

- [ ] Laptop charged to 100% or plugged in
- [ ] Power adapter packed
- [ ] Screen sleep set to "never" during presentation
- [ ] Notifications / Do Not Disturb enabled
- [ ] VPN disconnected (can interfere with Kiro agent calls)
- [ ] Antivirus real-time scan paused if it causes IDE lag
- [ ] No pending OS updates that could trigger a restart

---

## 5. Network

- [ ] Demo machine tested on the venue's Wi-Fi
- [ ] Mobile hotspot available as fallback
- [ ] Kiro API key / credentials confirmed working on this machine
- [ ] Backup demo confirmed to work fully offline

---

## 6. Pitch Rehearsal

- [ ] Full 3-minute pitch rehearsed at least three times
- [ ] 90-second live demo rehearsed at least five times
- [ ] Backup demo path rehearsed at least twice
- [ ] All ten judge Q&A answers rehearsed out loud
- [ ] Each Q&A answer timed — all under 25 seconds
- [ ] Opening sentence memorised: "Most AI coding tools generate code. ForgeSpec generates *proof*."
- [ ] Closing sentence memorised: "Spec generated. Code generated. Tests executed. Traceability confirmed. ForgeSpec: Ready to Ship."

---

## 7. Team Roles (confirm assignments)

- [ ] Primary presenter assigned
- [ ] Demo operator assigned (may be same person or a second team member)
- [ ] Backup-demo trigger decision: who calls the switch and when
- [ ] Judge Q&A: who answers technical questions vs. business questions

---

## 8. Day-Of Sequence

| Time | Action |
|------|--------|
| T – 60 min | Machine set up in presentation room |
| T – 45 min | Full live demo run end-to-end, verify output |
| T – 30 min | Backup demo path run once |
| T – 15 min | Checklist items 1–5 reviewed |
| T – 5 min | Font size, panels, chat cleared |
| T – 0 | Judges seated, begin pitch |

---

## 9. Post-Presentation

- [ ] Thank judges by name if known
- [ ] Offer to share the GitHub repository link
- [ ] Have the README URL ready to paste into chat or show on screen
- [ ] Note any questions you could not answer — address them in follow-up
