# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository. (No Claude? It works as a CONTRIBUTING guide too.)

## What this is

A shared **Obsidian study vault** for a DevOps course — one self-learning note per class plus a glossary hub, MOC index, and spaces for the teacher's uploads and student homework. It is *content*, not code: edits are Markdown, and the quality bar is "polished enough that the whole class studies from it".

## Non-negotiable rules

1. **Ground truth only.** Every command, number, and assignment requirement must be traceable to real course materials (the teacher's repo `yfreifeld/devops-course`, his uploaded decks/docs in `uploads/`, or the live sessions). Never invent content; if material is missing, say so.
2. **Answers stay hidden behind interaction.** Self-quiz answers live in collapsed `> [!question]-` callouts; the cheat-sheet is LAST in every note and collapsed (`> [!cheatsheet]-`); flashcards are click-to-flip. Never restructure a note so answers are visible before the student has to think.
3. **Navigate by topic, not number.** Note titles use the course repo's folder numbers, which differ from calendar sessions (session 1 was the intro; the Git deck literally says "Class 6"). Every class note carries a collapsed numbering-decoder callout under its title — keep it.

## Note anatomy (fixed section order — do not reorder)

TL;DR → 🎯 Learning goals → 🧩 Big idea (analogy + vertical diagram, code lines ≤95 chars) → 🧠 Core concepts → 🛠️ Guided walkthrough → 🔬 Drills (XP + "Done when:") → 🧗 Extra credit *(marked "(addition)")* → 🖥️ From the class deck → 📬 The REAL assignment → 🔎 Learn to fish → 🧪 Self-check quiz → 🃏 Flashcards → ⚠️ Gotchas → 🏆 Boss challenge → 💻 collapsed cheat-sheet → 🔗 Related.

- **Flashcards are dual-mode:** visible `> [!question]- Term` flip-cards PLUS a collapsed `> [!srdeck]-` callout holding raw `Term::Definition` lines tagged `#flashcards/devops` (read by the Spaced Repetition plugin). Keep both in sync when editing cards.
- **Glossary is the hub:** new technical terms get a `### Term` heading in `Terminology.md` and `[[Terminology#Term]]` links from the notes. Wikilinks everywhere; the MOC (`DevOps Experts — MOC.md`) indexes every note and tracks badges.
- **Bulk edits across many notes:** use small idempotent Python scripts (guard with a marker-string check so re-running never duplicates sections) rather than hand-editing 10+ files.

## The weekly ritual (after each class)

1. `uploads/class-NN-<topic>/` ← the teacher's new files, renamed clean (lowercase-dashes, no emoji/spaces). NN = **calendar** session number; the topic suffix disambiguates vs note numbering.
2. Mine them (pptx/docx = zip + XML regex, stdlib only) into the class note's `📬 The REAL assignment` and `🖥️ From the class deck` sections.
3. Student solution work goes under `homework/class-NN-<topic>/` — mind the sharing-etiquette note there (public repo; no solutions before deadlines).
4. Commit + push (this repo has no CI; the review is human).

## Ecosystem

- Companion game repo: https://github.com/iceteps/shell-quest (missions reference notes here via their `vault_note` field — renaming a note breaks that link; update both).
- `TEACHER-PROMPT.md` is the reusable prompt to regenerate this whole toolkit for another course — keep it updated if conventions here change.
- `setup/cheatsheet-flash.css` must stay in sync with the snippet users install (it styles `[!cheatsheet]` and `[!srdeck]` callouts).
- Owner: `iceteps`; the teacher (`yfreifeld`) may become a collaborator. `.obsidian/`, `Spaced Repetition/`, `.trash/` are gitignored (personal state) — never commit them.
