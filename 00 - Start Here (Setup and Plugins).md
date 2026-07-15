---
tags: [devops, setup, meta, start-here]
aliases: [Start Here, Setup, Prerequisites, Plugins]
---

# 🚀 Start Here — Setup & Plugins

Welcome! This vault is a self-learning DevOps course. 5 minutes of setup makes everything
(flashcards, collapsible answers, checkboxes, the knowledge graph) work properly.
New here? Read this, then go to [[DevOps Experts — MOC]].

> [!tip] The one that matters most
> If your **flashcards look like plain text** (`Term::Definition`) instead of review cards,
> you're just missing the **Spaced Repetition** plugin. Fix it in step 2 below. 👇

---

## 1. Turn on Community Plugins (once)

Obsidian ships with "Restricted Mode" on, which blocks community plugins.

1. **Settings** (⚙️ bottom-left) → **Community plugins**
2. Click **Turn on community plugins** (accept the prompt — these plugins are open-source and safe)

That's it — now you can install the ones below.

---

## 2. 🃏 Spaced Repetition — makes the flashcards work *(required for cards)*

Every class note has a `## 🃏 Flashcards` section with cards written as `Question::Answer`.
Without this plugin they're just text. With it, you get real spaced-repetition review.

**Install:**
1. Settings → **Community plugins** → **Browse**
2. Search **"Spaced Repetition"** (by Stephen Mwangi) → **Install** → **Enable**

> [!tip] Two ways to study the flashcards (you have both)
> Each note's **🃏 Flashcards** section now gives you a choice:
> 1. **👆 Flip-cards (in the note):** click any `❓` card to reveal its answer — no plugin needed, works in Reading/Live Preview. Great for a quick scan.
> 2. **🔁 Spaced-repetition review (pop-up):** the real memorisation tool — the plugin schedules cards so you review them right before you'd forget.

**To run a spaced-repetition review:**
- Click the **🃏 flashcard icon in the left ribbon**, OR press `Ctrl+P` → **"Spaced Repetition: Review flashcards in all notes"**.
- Pick the **devops** deck → a card shows the **front** → think → **Show Answer** → rate **Hard / Good / Easy**.
- Just installed it and see 0 cards? **Reload once** with `Ctrl+R`.

> [!info] About the collapsed "🔁 Raw review deck" at the bottom of each Flashcards section
> Those `Term::Definition` lines are what the plugin actually reads. They're tucked in a
> collapsed box on purpose — you never need to open it. If a review ever shows **0 cards**,
> it means the plugin isn't reading cards inside callouts on your version — tell me and it's a 10-second fix.
- Our cards are tagged `#flashcards/devops`, so they show up as a **`devops`** deck (a sub-deck of `flashcards`).
- Rate each card **Hard / Good / Easy** — the plugin schedules the next review automatically.

> [!info] If a deck doesn't appear
> Settings → **Spaced Repetition** → **Flashcard tags**: make sure it includes `#flashcards`
> (the default). Our `#flashcards/devops` sub-tag is picked up automatically. The card
> separator is `::` for one-liners (already the default) — no change needed.

> [!example] Card format used in these notes
> ```
> Image::A read-only template for a container.
> Container::A running, isolated instance of an image.
> ```
> Reading side = before `::`, answer side = after.

---

## 2.5 🎨 Install the CSS snippet (styles the cheat-sheets & decks)

The vault ships a stylesheet that makes the `[!cheatsheet]` boxes flash when you peek
and styles the collapsed review decks. One-time setup:

1. Copy `setup/cheatsheet-flash.css` (in this folder) into `<your-vault>/.obsidian/snippets/`.
2. Obsidian → **Settings → Appearance → CSS snippets** → refresh → toggle `cheatsheet-flash` **on**.

Skip it and everything still works — just plainer.

## 3. 📖 Callouts & collapsible answers — *built in, no plugin*

The colored boxes (`> [!tip]`, `> [!warning]`) and the **collapsible quiz answers**
(`> [!question]-` — click to reveal) are native Obsidian features.

> [!warning] Seeing raw `> [!question]-` text instead of a nice box?
> You're in **Source mode**. Switch to **Live Preview** or **Reading view**:
> click the book/pencil icon top-right, or `Ctrl+E` to toggle. Callouts only render outside Source mode.

The `-` after `[!question]` means **collapsed by default** — so you think first, then click to reveal the answer. That's the whole point of the self-quiz. 🧠

---

## 4. ✅ Checkboxes — *built in*

Drills and learning goals use `- [ ]` checkboxes. In Reading/Live Preview, **click the box**
to tick it as you complete each drill. Your progress saves in the note.

---

## 5. Recommended (optional) plugins

| Plugin | Why you'd want it |
|---|---|
| **Dataview** | Auto-build progress dashboards / query notes by tag or `class:` field. |
| **Graph view** *(built in)* | `Ctrl+G` — see how every note links through [[Terminology]]. Genuinely motivating. |
| **Excalidraw** | Draw your own architecture diagrams (the repo's `architecture.excalidraw` opens here). |

---

## 6. How to actually study (the loop)

> [!tip] Suggested rhythm per class note
> 1. Read **TL;DR → Big idea → Core concepts**.
> 2. Do the **Guided walkthrough** on your machine.
> 3. Beat the topic's [Shell Quest](https://github.com/iceteps/shell-quest) mission — brand-new topic? type `demo` inside the mission to *watch* it solved once, then play it for real.
> 4. Attempt the **Drills** (earn the XP) — *don't* open the cheat-sheet yet.
> 5. Take the **Self-check quiz** (reveal answers only after guessing).
> 6. Review the **Flashcards** (and again in a few days — that's the spaced repetition).
> 7. Stuck on a command? *Now* open the collapsed **💻 Cheat-sheet** at the bottom.
> 8. Tick your progress in [[🎓 Mastery Path]] — it's the dashboard that decides what's next.

---

## 🔗 Where to go next
- 🎓 [[🎓 Mastery Path]] — **your daily home page**: rank ladders per topic + what to do next
- 🗺️ [[DevOps Experts — MOC]] — the map of every note + suggested learning path
- 📖 [[Terminology]] — the master glossary (everything links here)
- 🧭 [[DevOps Foundations]] — start here conceptually if DevOps is new to you
