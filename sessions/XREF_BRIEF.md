# Brief: replace cross-references with in-place explanations

The student doesn't want bare pointers like "(Concept 57)", "see Concept 47", "from Session 03", "(Session 10)", "recap (Session 07)". They force the reader to remember what that concept was. Replace each one with a SHORT plain-language reminder of the idea itself, right there (one line at most), or simply delete the pointer if the sentence is already clear without it.

Example:
- Before: `**A stable DNS name**: the Service's name, such as `web` (Concept 57).`
- After: `**A stable DNS name**: the Service's name, such as `web`. Kubernetes runs its own DNS inside the cluster, which turns that name into the Service's IP, so other Pods can just call `http://web`.`

Other examples:
- "the kubelet working locally (Kubernetes I)" → "the kubelet (the agent on each node that starts and restarts containers) working on its own"
- "(the networking session)" → just remove it, or give a 3–6 word reminder ("the loopback address, meaning 'this machine'")

RULES
- KEEP section headings and titles that number things ("## Concept 54 · Services", "# Session 07 · …", "**Concepts:** 54 → 57"). Only change inline references inside sentences, bullets and table cells.
- NEVER change question or option lines (lines starting with **W, **Q, **L, **C, **M, **<digit>, or - A) to - D)). If a question itself contains a reference, leave it.
- Use the Edit tool for each change (small, targeted edits). Do NOT rewrite whole files.
- Don't touch _ANSWERS.md files.
- After finishing each file, verify with: `grep -nE "Concept [0-9]+|Session [0-9]+" <file>` and check that every remaining match is a heading, title, question line or the header line.
