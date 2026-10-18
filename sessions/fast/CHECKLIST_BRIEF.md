# Brief: add a full "🎯 Exam checklist" table to a FAST session

The student (beginner, exam tomorrow) asked: "before the questions, include a table of what is important for exams: everything from the session that could potentially be asked as an MCQ or as writing commands."

## What to do (edit only `sessions/fast/NN_fast.md`)
REPLACE the existing `## ⚡ Must-remember` section (heading + its table) with:

```
## 🎯 Exam checklist (everything that could be asked)
*MCQ = know the fact · CMD = be able to write it from memory.*

| Asked as | Item | What you must know / the exact command |
|---|---|---|
| … rows … |
```

Place it exactly where Must-remember was (just before `## 📝 Quiz`). Don't change anything else in the file (the reading sections and the quiz stay exactly as they are). Don't touch the answer key.

## Rows
- **25–35 rows**, grouped in the same order as the session's concepts (you may put a bold group label in the Item column of the first row of each group, e.g. `**Probes** · liveness`).
- Sources, in this order: (1) the fast file itself, (2) the FULL session file `sessions/NN_*.md` (not `_ANSWERS`), especially its "➕ Small but important" table, its ⚠️ traps, and its "✍️ Write the command" tasks, (3) the official syllabus line for the lecture (in `/Users/vanshsrivastava/DevOPS/SYLLABUS.md`). Include every syllabus subtopic at least once.
- **CMD rows** (aim for 8–14): the exact command or YAML key path a question could ask the student to write, in backticks, with realistic names, e.g. `kubectl create secret generic db --from-literal=PASS=x`. Add a few plain words saying what it does. Prefer the commands from the full session's ✍️ tasks and practicals.
- **MCQ rows**: one fact per row, written so it can be understood on its own (plain words, no shorthand fragments, no undefined terms). Include: definitions, "X vs Y" differences (write as "X does…, whereas Y…"), status words/exit codes/error messages and what they mean, defaults, full forms of acronyms (e.g. "HPA = Horizontal Pod Autoscaler"), and common traps.
- Every fact must be correct and must agree with the full session. No bare cross-references ("Concept 57"). No game-dev analogies.
- Keep each cell short (ideally under ~25 words), but complete.

## Safety
Edit only the one file named in your task. Don't run git, kubectl, docker, helm, terraform, or anything that creates resources.

When done, report: number of rows (MCQ vs CMD), and which syllabus subtopics you mapped to which rows.
