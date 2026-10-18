# Brief: 10-minute FAST versions of the sessions (exam is tomorrow, Mon 19 Oct 2026)

The student (3rd-year college student, zero DevOps background before this course, Mac user) has ONE day left and said they will only read a session if it takes **~10 minutes** and ends with **one 5-question quiz**. Your job: turn one full session file into a fast version. The full session is the source of truth for facts; the official syllabus line for the lecture (given in your task) decides what must be included.

## Output files
1. `sessions/fast/NN_fast.md`: the 10-minute read + the 5-question quiz (questions and options only).
2. `sessions/fast/NN_fast_ANSWERS.md`: the answer key.

## Size (strict)
- Reading part: **900–1,300 words** (about 90–130 lines). Count with `wc -w`. If it is longer, cut whole TOPICS of lesser exam value, never cram.
- No practicals, no warm-up, no checks, no "write the command" tasks, no clean-up steps.

## Shape of NN_fast.md
```
# Session NN · <Title> · FAST (10-min read)
*One-line "what this lecture is about" in plain words.*

## 1 · <concept title>
2–3 plain sentences: what it is and why it exists (problem → idea). Define every technical word where it first appears, and give the full form of every acronym (e.g. "HPA (Horizontal Pod Autoscaler)").
- 3–6 bullets of key facts. Each bullet is ONE idea written as a full, plain sentence (no shorthand like "STATUS shows reason (…state: Waiting…)").
- A tiny code/YAML/command snippet ONLY if it is directly examinable, max ~8 lines, followed by one plain line per key part.
⚠️ **Exam trap:** 1–2 commonly confused pairs, each in one sentence ("X does…, whereas Y…").

## 2 · … (one section per concept of the full session; 3–5 sections)

## ⚡ Must-remember (one table, max 10 rows: command/flag/status word/term → meaning)

## 📝 Quiz (5 MCQs; answer in chat as "1 B, 2 D, …")
```

## Writing rules (the student has complained about each of these)
- Explain, don't list: keep "what it is → why → tiny example" for every important idea; cut unimportant ideas entirely instead of compressing them into fragments.
- Every acronym gets its full form on first use; every term gets a plain meaning.
- At most one short everyday analogy per concept (restaurant, post office, hotel…). **No game-development analogies.**
- No bare cross-references ("Concept 57", "see Session 06", "way (b)", "the worked example above"): say the actual idea in place.
- Block-style YAML only (no `{a: 1}` flow style).
- Ports on this Mac: 8080/8443 are taken by another cluster; if a port-forward appears, use 9080/9443. (Mostly irrelevant here since there are no practicals.)

## Quiz rules (strict; the student has complained about MCQs many times)
- Exactly **5 MCQs**, each with options A–D. Hard, full-marks level: scenario, debugging ("an engineer runs X and sees Y; what's the cause?"), predict-the-output, "which is NOT". No pure one-word recall.
- Cover at least 4 of the concept sections.
- All four options roughly the same length (within ~10 characters). If one option gives a reason, all do.
- The correct option is **not the longest** in at least 4 of the 5 questions.
- Distractors must be genuinely tempting: true of a neighbouring concept, half right, off by one detail, a common misconception, a near-miss flag. Never silly, never the exact opposite.
- No giveaway words (don't put "password" in the right option only, etc.).
- Correct letters: use at least 3 different letters, no letter more than twice, and don't put two identical letters in a row.
- Every question must be answerable from the fast file alone.

## Answer key format (exact; the student insisted)
```
**1.** <full question, including any code>
- A) …
- B) …
- C) …
- D) …

**Correct: X.** <why it's right, plain words>. <why each tempting wrong option is wrong>.

---
```

## Safety
Only create/edit the two files named in your task. Don't edit the full session file. Don't run git, kubectl, docker, helm, terraform, or anything that creates resources (reading files and `wc` are fine).

When done, report: word count of the reading part, the 5 correct letters, and for each question which option is longest.
