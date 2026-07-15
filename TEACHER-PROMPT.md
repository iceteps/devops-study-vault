# 🎁 For the teacher: build this study toolkit for ANY course

This vault was generated with AI (Claude Code) from real course materials. This file
contains the **reusable master prompt** that recreates the whole toolkit — the study
vault, and optionally the practice game — for **any subject, any year**. It was built
by a student; feel free to adapt it.

## How to use it

1. Install [Claude Code](https://claude.com/claude-code) and open a folder containing
   your course materials (slide decks, assignment docs, lab files, command lists —
   whatever exists; a folder per class works best).
2. Copy the entire prompt below, fill in the three `<<...>>` blanks at the top, paste it in.
3. Review what it builds, then iterate: *"class 4's note is missing X"*, *"add a mission
   for topic Y"*. It's a conversation, not a one-shot.
4. **Next year:** re-open the project, point it at the updated materials, and say what
   changed — *"the Docker assignment is now about compose; update the note, the
   assignment section, and the game mission."* Structure survives; content refreshes.

---

## THE PROMPT (copy everything below this line)

```
You are building a complete self-learning study toolkit for my course.

COURSE: <<course name, e.g. "DevOps Experts — 2027 cohort">>
MATERIALS: <<path to the folder with my per-class materials (decks/assignments/labs)>>
OUTPUT VAULT: <<path where the Obsidian vault folder should be created>>

Read ALL my materials first (pptx/docx are zip files — extract their text with
Python's zipfile + XML regex; lab folders often contain a `commands` runbook file
that is the ground truth for what each class actually ran). Everything you write
must be grounded in these real materials — never invent commands, numbers, or
assignment requirements. If materials for a class are missing, say so and skip it
rather than fabricating content.

DELIVERABLE 1 — an Obsidian study vault, one polished note per class, plus:
- `Terminology.md`: a master glossary — every technical term as a `### Term`
  heading with a 1–3 line plain-English definition; ALL notes link into it with
  [[Terminology#Term]] wikilinks. This is the hub of the vault.
- A "MOC" (map of content) home note: suggested learning path, a table of all
  notes, a badge tracker, and — if the class numbering in folders differs from
  the calendar sessions — a "numbering decoder" so students navigate by TOPIC.
- `00 - Start Here`: plugin setup (Spaced Repetition for flashcards), how
  callouts/checkboxes work, and the study loop.
- A "Foundations" note for the course's conceptual layer (methodology, mental
  models) and, if there's a final project, a "Capstone" note mapping each of its
  parts to the class that teaches it.
- `uploads/class-NN-<topic>/` folders holding my original files (renamed clean:
  lowercase, dashes, no spaces/emoji), and `homework/class-NN-<topic>/` folders
  for student work — each with a README documenting the weekly ritual: new class
  → drop files in uploads → distill them into the class note → commit + push.

EVERY class note follows this exact section order (pedagogy is the point):
1.  Title + a small collapsed callout clarifying the class number if numbering is confusing
2.  [!abstract] TL;DR — 2-3 sentences + why a practitioner cares
3.  🎯 Learning goals — a checkbox list
4.  🧩 The big idea — ONE memorable analogy + a text diagram if the topic has a
    flow (diagrams VERTICAL and under 95 chars wide so they never wrap)
5.  🧠 Core concepts — bold term → short explanation → glossary wikilink
6.  🛠️ Guided walkthrough — numbered do-it-now mini-lab from my real commands,
    with expected output
7.  🔬 Drills (earn XP) — 5-7 graduated checkbox tasks, each with an XP value
    and a "Done when:" criterion
8.  🧗 Extra credit — 2-3 harder real-world drills clearly marked "(addition —
    not covered in class)"
9.  🖥️ From the class deck — anything my slides teach that the note didn't cover
10. 📬 The REAL assignment — my actual graded homework distilled step-by-step
11. 🔎 Learn to fish — official docs links + how to DISCOVER commands
    (--help, built-in doc tools, how to search) so this never becomes
    monkey-see-monkey-do
12. 🧪 Self-check quiz — 4-6 collapsible `> [!question]-` callouts (student must
    think before revealing the answer)
13. 🃏 Flashcards — two modes: visible click-to-flip `> [!question]-` cards, PLUS
    a collapsed callout holding raw `Term::Definition` lines tagged
    #flashcards/<course> for the Spaced Repetition plugin
14. ⚠️ Gotchas — the mistakes students actually make, as [!warning] callouts
15. 🏆 Boss challenge — one integrative task
16. 💻 Command cheat-sheet — LAST and COLLAPSED (`> [!cheatsheet]-` custom
    callout with a "try the drills first" warning line). Never put the
    cheat-sheet before the drills — it defeats the learning.
17. 🔗 Related — wikilinks to neighboring notes and the MOC
Finish each note with a [!success] "cleared when you can…" checklist and a fun
badge name. Ship a CSS snippet (in `setup/`) that styles the cheatsheet callout
and flashes it when expanded.

DELIVERABLE 2 (optional but the students' favorite) — a terminal practice game:
a pure-stdlib Python "simulated shell" where students type the REAL commands of
my course against a fake world that responds like the real tools (state: the
domain objects of my subject). Missions = stories with objectives that check
WORLD STATE, not keystrokes; hints cost XP; personal progress in a gitignored
JSON file; and crucially a `--selftest` mode where every mission carries its own
solution script proving it's completable. Make missions that mirror my REAL
graded assignments so the game is a risk-free rehearsal.

QUALITY BARS:
- Ground truth only: every command/example traceable to my materials.
- Self-learning first: answers hidden behind interaction (collapse, flip, hint).
- Fun is a feature: analogies, XP, badges, streaks — but never at the cost of accuracy.
- Idempotent maintenance: bulk edits across notes via small guarded scripts, so
  re-running never duplicates sections.
- Before publishing anything to GitHub: scan for personal data/secrets, add a
  .gitignore for personal state (Obsidian workspace, review history, progress
  files), write a README with clone-into-vault instructions, then create the
  repo(s) with `gh repo create --public --source . --push`.

Start by listing my classes and what materials you found for each, propose the
note list, then build after I confirm.
```

---

**2026-07-15 additions — fold these into the prompt when regenerating:**

- **Mastery Path note**: a per-topic 4-rank ladder (Apprentice → Practitioner → Expert →
  Master) whose criteria point at the vault's own drills, the game's missions, the quiz, and
  the REAL assignments; plus a dashboard table and a daily/weekly loop. It becomes the
  learner's home page; add a fork-reset callout so shared copies start clean.
- **Game demo mode**: every mission's solution script doubles as a step-by-step replay
  (`demo` → Enter advances → `takeover`); watching pays no XP.
- **Teach lines**: one micro-lesson per objective, printed on completion + recapped at
  mission end; a lint pass inside the selftest enforces parity.
- **Vault↔game sync**: the game renders a live progress note into the learner's vault
  (opt-in via a gitignored config file) — progress visible where they study.
- **Full-course game coverage**: one mission per topic minimum + a final campaign mission
  chaining every tool end-to-end as the capstone dress rehearsal.

*Reference implementations of everything the prompt describes:*
*[devops-study-vault](https://github.com/iceteps/devops-study-vault) (this vault) ·
[shell-quest](https://github.com/iceteps/shell-quest) (the game).*
