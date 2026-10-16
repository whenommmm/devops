# Rewrite brief for sessions 07–16 (read fully)

## What went wrong
The last condensing pass squeezed explanations into dense shorthand, e.g.:
> "STATUS shows a **reason** (why a container is in its **state**: `Waiting`, `Running` or `Terminated`), not the phase; `Terminating` (being deleted) isn't a phase either. … Back-off = growing waits between restarts (10 s, 20 s…) …"
The student (a beginner) couldn't understand it: terms were dropped in without being explained. They asked us to **remove unimportant STUFF, not explanations**.

## The model to copy
`/Users/vanshsrivastava/DevOPS/sessions/06_pods-replicasets-deployments.md` was just rewritten by hand to the right balance. **Read it first and match its style exactly:**
- Each important idea: **what it is → why it exists → a simple example or analogy**, in short plain paragraphs or bullets.
- Every new term is explained **in words** where it appears, never just named in parentheses.
- Tables for comparisons; block-style YAML with a comment on each line; practical commands with one short comment each, plus 1–3 key output lines.
- **Unimportant details are DROPPED entirely** instead of being crammed in: version quirks (keep only those that affect the practical, in one line), exact timings and numbers unless exam-relevant, history, rarely-used flags, second and third examples of the same thing, long output dumps, optional deep-dives.

## Sources
- The current file: keep its structure, all questions, the "✍️ Write the command" section, lab-file references (09/10 use `~/DevOPS/sessions/labs/...`), and setup caveats.
- The detailed original before condensing: `/private/tmp/claude-condense/<same filename>.orig`. Recover the **explanations** of important ideas from it.

## Hard rules
- **Every question line and option line stays byte-identical** (warm-up, ✅ checks, 📝 quiz, ✍️ C-tasks, and the mini mock in 16). Before editing, copy the current file to `/private/tmp/claude-rewrite/<file>.cur`. After editing, check with a script that every line starting with `**W`, `**Q`, `**L`, `**M`, `**C`, `**<digit>` or `- A)`…`- D)` from the .cur still exists.
- **Length:** each file **≤ 460 lines** (15 covers two lectures: ≤ 500; in 16 the teaching part ≤ 260 lines, and the mock + last-look sheet stay verbatim).
- No game-dev analogies. Never touch `_ANSWERS.md` files. Validate any YAML you change with `kubectl apply --dry-run=client -f`. Never create resources, never push to GitHub.
