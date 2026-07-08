---
tags: [devops, git, version-control, class-03]
aliases: [Class 3 Git, Git Core Workflow, Git Basics, Commit Captain]
class: 03
difficulty: beginner
---

# 🌿 Class 03 — Git

> [!info]- 🔢 Which class is this, really?
> The number in this note's title follows the **course repo folder** `class3/` — **not** the calendar session. The real session numbering is shifted: calendar-session 1 was the DevOps intro (see [[DevOps Foundations]]), so e.g. Docker basics was *taught* in session 2 even though its repo folder is `class1`. When in doubt, navigate by **topic** via [[DevOps Experts — MOC]], not by number.


> [!abstract] TL;DR
> **Git is your project's time machine + save-point system.** You edit files in your **Working Directory**, hand-pick changes into the **Staging area** with `git add`, snapshot them into your **Local Repo** with `git commit -m`, and beam them to the **Remote** (GitHub/origin) with `git push`. **Branches** let you build in a parallel universe without breaking `main`. The whole class is one muscle: **`add → commit → push`**. Bonus concept: **monolith vs monorepo** (how you organize code, not how you version it).

---

## 🎯 Learning goals

- [ ] Explain the 4 states of a change: Working Directory → Staging → Local Repo → Remote.
- [ ] `clone` a repo and read `git status` without fear.
- [ ] Stage with `git add`, snapshot with `git commit -m`, publish with `git push`.
- [ ] Read a `git diff` and a `git log`.
- [ ] Create a branch, `checkout`/`switch` to it, and push it with `-u origin`.
- [ ] Say the difference between a **monolith** and a **monorepo** out loud.

---

## 🧩 The big idea

🎮 **Think of a video game.** Your Working Directory is *the level you're playing right now* — messy, in-progress, one wrong move from disaster. A **commit** is a **save point**: a permanent snapshot you can always return to. **Staging** is the moment *just before* you save where you choose *what* goes into the save file.

📸 **Or think of a group photo.** Everyone's milling around (Working Directory). You call out "you, you, and you — get in frame" (`git add`). *Click* — the photo is taken (`git commit`). Then you post it to the family group chat so everyone sees it (`git push` to the **remote**).

> [!tip] The mental model that unlocks everything
> **A file isn't "in Git" just because you saved it in your editor.** Git only tracks what you explicitly `add` (stage), and only *remembers forever* what you `commit`. Saving in VS Code ≠ committing. Committing ≠ pushing.

### The pipeline (memorize this diagram)

```text
┌─────────────────────────────────────┐
│         WORKING DIRECTORY           │  ✏️  your edits — untracked
│   (the level you're playing now)    │      + modified files
└─────────────────────────────────────┘
              │ git add <file>
              ▼                    ▲ git restore --staged  (un-stage)
┌─────────────────────────────────────┐
│            STAGING AREA             │  📸  the photo you framed
│      (chosen for the next save)     │      (aka "the index")
└─────────────────────────────────────┘
              │ git commit -m "msg"
              ▼
┌─────────────────────────────────────┐
│            LOCAL REPO               │  💾  save points — your
│      (.git — full history)          │      commit history
└─────────────────────────────────────┘
              │ git push
              ▼                    ▲ git pull  (bring team's work down)
┌─────────────────────────────────────┐
│              REMOTE                 │  ☁️  origin — GitHub,
│      (what the team sees)           │      shared by everyone
└─────────────────────────────────────┘
```

**One trip down = publish** (`add → commit → push`) · **coming back up:** `git pull` (from remote), `git restore --staged` (un-stage), plain editing (working dir never locks).

> [!example] Read it as one sentence
> "I **edit** in my Working Directory, I **`add`** the good bits to Staging, I **`commit`** them to my Local Repo, and I **`push`** them to the Remote so my team can see them."

---

## 🧠 Core concepts

- **[[Terminology#Git|Git]]** 🧬 — a distributed **version-control system**. "Distributed" = *everyone has the full history on their machine*, not just a central server. You can commit, branch, and view history completely offline.
- **[[Terminology#Repository|Repository (repo)]]** 📦 — a project folder that Git tracks. The magic lives in the hidden `.git/` subfolder, which stores every commit, branch, and config. Created by `git init` (new) or `git clone` (copy an existing one).
- **[[Terminology#Staging area|Staging area]]** 🎯 — a.k.a. the **index**. The waiting room between your edits and a commit. This is what makes Git feel surgical: you can commit *some* of your changes and leave the rest for later.
- **[[Terminology#Commit|Commit]]** 💾 — an immutable snapshot of the staged changes + a message + author + timestamp + a unique SHA hash (like `a1b2c3d`). Commits are the save points. **A commit needs a message** (`-m "..."`).
- **[[Terminology#Branch|Branch]]** 🌿 — a movable pointer to a commit. `main` is just the default branch. Make a new branch to work on a feature without touching `main`. Branches are cheap — make them freely.
- **[[Terminology#Merge|Merge]]** 🔀 — combines the history of one branch into another (e.g. bringing `branch1` back into `main` when the feature is done).
- **[[Terminology#Remote|Remote]] / origin** 🌐 — a copy of the repo hosted elsewhere (GitHub, GitLab). **`origin`** is the conventional name for "the remote you cloned from." `push` sends commits up; `pull` brings commits down.
- **HEAD** 📍 — a pointer to "where you are right now" (usually the tip of your current branch).

### 🏗️ Monolith vs Monorepo (don't confuse them!)

These sound alike but answer **completely different questions**. (Student's rough note said *"definitely mono is better"* — let's make that opinion defensible 😄.)

| | **Monolith** | **Monorepo** |
|---|---|---|
| **Question it answers** | How is the app *deployed/built*? (architecture) | How is the code *stored*? (repository layout) |
| **Meaning** | One big application, one deployable unit | One repo holding **many** projects/services |
| **Opposite of** | Microservices | Multi-repo (one repo per service) |
| **Example** | A single Rails/Django app that does everything | Google's giant repo; a repo with `frontend/`, `backend/`, `infra/` |
| **Pain** | Hard to scale teams; one bug can sink the ship | Big clone size; needs good tooling (Nx, Turborepo, Bazel) |
| **Win** | Simple to build/deploy/reason about early on | Shared code, atomic cross-project commits, one PR spans everything |

- **[[Terminology#Monolith|Monolith]]** = *deployment* shape (one unit) — vs microservices.
- **[[Terminology#Monorepo|Monorepo]]** = *storage* shape (one repo, many projects) — vs many small repos.

> [!warning] The trap
> You can have a **monorepo full of microservices**, or a **monolith split across many repos**. They're orthogonal. "Mono is better" isn't a law — a monorepo shines for coordinated teams with good tooling; a monolith shines when you want to move fast and stay simple early. Pick per context.

---

## 🛠️ Guided walkthrough — mini-lab

Follow along in a real terminal. This mirrors Yariv's class session.

1. **Get a repo.** Clone the class repo (or `mkdir myrepo && cd myrepo && git init` for a fresh one):
   ```bash
   git clone https://github.com/yfreifeld/gitclass.git
   cd gitclass
   ```
2. **Create/edit a file.**
   ```bash
   vi Dockerfile        # type: FROM ubuntu:22.04  → save & quit (Esc, then :wq)
   ```
3. **Check status.** `git status`
   > Expected: `Dockerfile` under **"Untracked files"** (red).
4. **Stage it.** `git add Dockerfile` → then `git status`
   > Expected: `Dockerfile` under **"Changes to be committed"** (green).
5. **Commit it.**
   ```bash
   git commit -m "adding new file Dockerfile"
   ```
   > Expected: `[main a1b2c3d] adding new file Dockerfile · 1 file changed, 1 insertion(+)`
6. **Push it.** `git push`
   > Expected: `... main -> main` upload lines.
7. **Branch off + switch.**
   ```bash
   git branch branch1        # create
   git checkout branch1      # switch (or: git switch branch1)
   git branch                # the * should be on branch1
   ```
8. **Edit again, then diff.**
   ```bash
   vi Dockerfile             # add: WORKDIR /app  and  COPY . .
   git diff
   ```
   > Expected: green `+WORKDIR /app` / `+COPY . .` lines with `@@` hunk headers.
9. **Commit + log.**
   ```bash
   git add Dockerfile
   git commit -m "adding WORKDIR and COPY commands"
   git log --oneline
   ```
   > Expected: two lines, newest on top, each `<sha> <message>`.
10. **Publish the new branch.** `git push -u origin branch1`
    > Expected: `Branch 'branch1' set up to track 'origin/branch1'.`

---

## 🔬 Drills (earn XP)

- [ ] **(10 XP) First save point.** In a fresh `git init` repo, create `hello.txt`, then `add` + `commit -m`. **Done when:** `git log` shows exactly one commit.
- [ ] **(10 XP) Status radar.** Edit a tracked file but DON'T stage it. **Done when:** you can point to the word "modified" in `git status` and explain it's still in the Working Directory.
- [ ] **(15 XP) Surgical staging.** Change two files, stage only one. **Done when:** `git status` shows one file green (staged) and one red (unstaged) at the same time.
- [ ] **(15 XP) Diff reader.** Run `git diff` before staging, then `git add`, then `git diff` again. **Done when:** you can explain why the second `git diff` is empty (hint: `git diff --staged`).
- [ ] **(20 XP) Branch hop.** Create `feature-x`, switch to it with `git switch`, confirm with `git branch`. **Done when:** the `*` sits on `feature-x`.
- [ ] **(20 XP) Upstream unlock.** Push `feature-x` with `-u`, then make one more commit and push with a bare `git push`. **Done when:** the second push works with no `origin`/branch arguments.
- [ ] **(25 XP) Time traveler.** Use `git log --oneline` to find an old commit SHA and `git show <sha>` to inspect it. **Done when:** you can read what changed in that specific commit.

---

## 🖥️ From the class deck *(addition — mined from `Git.pptx`)*

> [!info] What the slides add beyond the live session
> 1. **First-time setup — do this ONCE before any commit** (the deck's exercise starts with it):
>    ```bash
>    git config --global user.name  "Your Name"
>    git config --global user.email "you@example.com"
>    ```
>    Without it, your commits are attributed to nobody (and GitHub won't link them to you).
> 2. **Why Git exists:** born in 2005 when the Linux kernel's BitKeeper deal collapsed — goals: speed, simple design, fully **distributed** (vs centralized VCS like SVN).
> 3. **[[Terminology#SHA (commit hash)|SHA]] & [[Terminology#HEAD|HEAD]]:** every commit is a node with a unique SHA; HEAD points at where you are. `git show <sha>` inspects any node; `git log --graph --oneline --all` draws the whole tree — the exact command the assignment asks you to submit.
> 4. **The deck's table is monorepo vs MULTI-repo** (not monolith!): monorepo = atomic cross-project changes + unified CI/CD but needs tooling to scale; multi-repo = team autonomy + independent releases but hard cross-repo coordination.
>
> 😄 Fun fact for the numbering saga: this deck's own title slide says **"Class 6"**. Navigate by topic.
>
> 📄 Full deck: [class3-git.pptx](uploads/class-03-git/class3-git.pptx) *(in this repo's `uploads/`)*

## 📬 The REAL assignment (from Yariv's drive)

> [!important] 🧪 Git Fundamentals — Branching, Merging & Conflicts
> The actual graded homework, step by step:
> 1. **Create** a public GitHub repo `git-python-practice` (no README) and **clone** it.
> 2. **Commit** an `app.py` with a `greet(name)` function → stage → commit → push to `main`.
> 3. **Branch:** create `feature/add-time`, switch to it, make the greeting include the current time (`datetime`), commit, push the branch.
> 4. **Merge without conflict:** back on `main`, merge the feature in, push.
> 5. **Manufacture a conflict:** change `greet()` *differently* on `main` AND on the feature branch (two commits).
> 6. **Trigger + resolve:** merge → conflict in `app.py` → open it, combine both ideas, **remove every `<<<<<<< ======= >>>>>>>` marker**, stage, commit, push.
> 7. **Submit:** repo URL + output of `git log --oneline --graph` + a working final `app.py`.
>
> **Bonus part:** history spelunking (`git log --oneline`, `--graph --all`, `git show <hash>`) and diff practice (`git diff`, `--staged`, `git restore`, `git restore --staged`) — running `git status` before and after every step.
>
> 🗡️ **Practice it risk-free first:** missions 5–7 in [Shell Quest](https://github.com/iceteps/shell-quest) are this exact assignment — mission 7 is the conflict finale.

## 🧗 Extra credit — beyond class *(addition)*

> [!example] 🆕 These drills are an **addition** — not covered in the class materials
> They're the next skill up from what class taught, chosen because you'll meet them in real work (and in the [[SkyWatch Capstone]]). Higher XP, higher payoff.

- [ ] **Cause (and survive) a merge conflict (30 XP)** — class stopped at branching. Make two branches edit the SAME line of the same file, merge one into the other, and resolve the `<<<<<<<` conflict markers by hand. **Done when:** `git log --oneline --graph` shows the merge and you weren't scared.
- [ ] **`git stash` — the panic button (20 XP)** — start editing, then pretend an urgent fix is needed: `git stash`, confirm the working dir is clean, fix-commit something else, `git stash pop`. **Done when:** you know when stash beats commit.
- [ ] **Time travel, safely (25 XP)** — learn the difference the hard way: make 3 throwaway commits, `git revert` the middle one (history preserved), then `git reset --hard HEAD~1` the last (history rewritten). **Done when:** you can say which one is safe on a shared branch and why.

## 🔎 Learn to fish — find it yourself (don't just copy)

> [!tip] Beat "monkey-see-monkey-do"
> The real skill isn't memorizing commands — it's finding the right one **fast**. Pros do this all day. Build the reflex:
> 1. **Ask the tool first:** `<tool> --help`, `<tool> <subcommand> --help`, `man <tool>`.
> 2. **Official docs = source of truth** (below) — not random blogs or old Stack Overflow answers.
> 3. **Search smart:** `<what you want> site:<official-docs-domain>`; add the tool's version if behaviour changed between releases.
> 4. **Read the WHOLE error message** — it almost always names the missing flag or the fix.
> 5. `tldr <command>` gives real-world examples (install a `tldr` client — it's the friendly `man`).

**📚 Docs & references**
- [Git reference (all commands)](https://git-scm.com/docs)
- [Pro Git book (free)](https://git-scm.com/book/en/v2)
- [GitHub docs](https://docs.github.com/en/get-started)

**⚡ Built-in help — try these BEFORE searching**
```bash
git help <command>       # opens the full manual, e.g. git help commit
git <command> -h         # quick one-screen flag summary, e.g. git branch -h
git status               # when lost, this always tells you what to do next
```

> [!example] Power move for this class
> `git status` is your compass — it literally suggests the next command (how to unstage, how to publish a branch). Read it before Googling.

## 🧪 Self-check quiz

> [!question]- 1. You saved a file in VS Code. Is it in Git's history yet?
> **No.** Saving in your editor only touches the **Working Directory**. Git records nothing until you `git add` (stage) and then `git commit` (snapshot). Saving ≠ committing.

> [!question]- 2. What's the difference between `git add` and `git commit`?
> `git add` **stages** — it copies a change into the Staging area (index), choosing *what* goes in the next snapshot. `git commit` **records** — it turns everything staged into a permanent, hashed snapshot in the Local Repo. Add = pick; Commit = save.

> [!question]- 3. `git commit` opened a weird text editor asking for a message. What happened, and how do you avoid it?
> You committed **without `-m`**, so Git launched an editor (often vi) for the commit message. Type the message, save & quit (`Esc :wq`), or next time use `git commit -m "your message"` to supply it inline.

> [!question]- 4. First push of a brand-new branch fails with "no upstream branch". Fix?
> Push with `git push -u origin <branch>`. The `-u` (`--set-upstream`) links your local branch to the remote one. After that, plain `git push`/`git pull` work on that branch.

> [!question]- 5. Monolith vs monorepo — one-liner each.
> **Monolith** = one big *deployable application* (opposite: microservices). **Monorepo** = one *repository storing many projects* (opposite: multi-repo). Different questions: architecture vs code layout. They're orthogonal.

> [!question]- 6. What does `origin` actually mean?
> It's the default **name** (alias) for the **remote** you cloned from. `git push` = `git push origin <current-branch>` under the hood. You could rename it or add more remotes, but `origin` is the convention.

---

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- Working Directory
> Your project folder as it is right now — edited, untracked, and modified files that Git has not yet recorded.

> [!question]- Staging area (index)
> The waiting room between edits and a commit; `git add` puts changes here to choose exactly what the next commit will contain.

> [!question]- git add
> Stages a change — moves it from the Working Directory into the Staging area for the next commit.

> [!question]- git commit -m
> Records all staged changes as a permanent, hashed snapshot in the local repo, with an inline message.

> [!question]- git push
> Uploads local commits to the remote (origin) so teammates can see them.

> [!question]- git pull
> Downloads and integrates commits from the remote into your local branch.

> [!question]- git status
> Shows which files are untracked, modified, or staged — the "where am I" command you run constantly.

> [!question]- git diff
> Shows line-by-line changes in the Working Directory that are not yet staged (use `--staged` for staged changes).

> [!question]- git log
> Displays commit history, newest first (`--oneline` for a compact view); press q to quit the pager.

> [!question]- git clone
> Copies an existing remote repository to your machine and sets up `origin` automatically.

> [!question]- git branch branch1
> Creates a new branch pointer named branch1 but does NOT switch to it.

> [!question]- git checkout / git switch
> Switches your working tree to another branch (switch is the modern, safer alias).

> [!question]- git push -u origin branch1
> Pushes a new branch and sets its upstream, so later bare `git push`/`git pull` know where to go.

> [!question]- origin
> The conventional name for the remote you cloned from.

> [!question]- HEAD
> A pointer to where you currently are — usually the tip of your current branch.

> [!question]- Monolith
> One large deployable application (opposite of microservices) — an architecture concept.

> [!question]- Monorepo
> A single repository containing many projects/services (opposite of multi-repo) — a code-storage concept.

> [!question]- Commit SHA
> The unique hash id (e.g. a1b2c3d) that permanently identifies a commit.

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> Working Directory::Your project folder as it is right now — edited, untracked, and modified files that Git has not yet recorded.
> Staging area (index)::The waiting room between edits and a commit; `git add` puts changes here to choose exactly what the next commit will contain.
> git add::Stages a change — moves it from the Working Directory into the Staging area for the next commit.
> git commit -m::Records all staged changes as a permanent, hashed snapshot in the local repo, with an inline message.
> git push::Uploads local commits to the remote (origin) so teammates can see them.
> git pull::Downloads and integrates commits from the remote into your local branch.
> git status::Shows which files are untracked, modified, or staged — the "where am I" command you run constantly.
> git diff::Shows line-by-line changes in the Working Directory that are not yet staged (use `--staged` for staged changes).
> git log::Displays commit history, newest first (`--oneline` for a compact view); press q to quit the pager.
> git clone::Copies an existing remote repository to your machine and sets up `origin` automatically.
> git branch branch1::Creates a new branch pointer named branch1 but does NOT switch to it.
> git checkout / git switch::Switches your working tree to another branch (switch is the modern, safer alias).
> git push -u origin branch1::Pushes a new branch and sets its upstream, so later bare `git push`/`git pull` know where to go.
> origin::The conventional name for the remote you cloned from.
> HEAD::A pointer to where you currently are — usually the tip of your current branch.
> Monolith::One large deployable application (opposite of microservices) — an architecture concept.
> Monorepo::A single repository containing many projects/services (opposite of multi-repo) — a code-storage concept.
> Commit SHA::The unique hash id (e.g. a1b2c3d) that permanently identifies a commit.

## ⚠️ Gotchas

> [!warning] The classic beginner traps
> - **"I committed but nothing changed."** You forgot to `git add` first. Only *staged* changes get committed. Empty stage → nothing to commit.
> - **`git commit` hijacked my terminal with vi.** No `-m` → Git opened an editor. Escape vi with `Esc` then `:wq` (write & quit) or `:q!` (quit without saving). Use `-m "msg"` to avoid it.
> - **First push rejected — "no upstream branch."** New branch needs `git push -u origin <branch>` once.
> - **`git branch branch1` didn't move me.** Creating a branch ≠ switching to it. You still need `git checkout branch1` / `git switch branch1`.
> - **"You are in 'detached HEAD' state."** You checked out a specific commit SHA instead of a branch. Commits made here can be lost. Get back safely: `git switch -` or create a branch with `git switch -c rescue`.
> - **add vs commit confusion.** `add` = *pick* what goes in the photo. `commit` = *take* the photo. Two separate steps on purpose.
> - **Pushing ≠ done for the team.** `commit` is local; teammates see nothing until you `push`.

---

## 🏆 Boss challenge

**The full branch round-trip.** Do it start to finish without notes:

1. On `main`, create and switch: `git switch -c feature-boss`.
2. Edit a file, `git add`, `git commit -m "boss: add a feature"`.
3. `git push -u origin feature-boss`.
4. Switch back: `git switch main`.
5. Merge your work in: `git merge feature-boss`.
6. `git log --oneline` — confirm the feature commit now sits on `main`.
7. `git push` to publish the merged `main`.

> [!success] Beat the boss when…
> `git log --oneline` on `main` shows your `feature-boss` commit, your working tree is clean, and both branches are pushed to `origin`. You've done a real feature workflow. 🐉⚔️

---

> [!success] Class 03 cleared when you can…
> - [ ] Narrate a change through **Working Directory → Staging → Local Repo → Remote** without looking.
> - [ ] Run `clone/status/add/commit -m/push/diff/log` from memory.
> - [ ] Create a branch, switch to it, and push it with `-u origin`.
> - [ ] Explain **monolith vs monorepo** as two different questions.
>
> 🎖️ **Badge earned: Commit Captain** 🧢

---

> [!cheatsheet]- 🔒 Command cheat-sheet — try the drills FIRST (expanding = peeking 👀)
> ⚠️ **Wait!** Expanding this is basically giving up. Did you really attempt the drill from memory? If not, collapse me and try again — the struggle is what makes it stick. Still stuck? Fine, the commands are below. 👇
>
> Exactly the commands from **class3/commands** (Yariv's useful commands), annotated:
>
> ```bash
> git clone https://github.com/yfreifeld/gitclass.git  # copy a remote repo to your machine (sets up 'origin')
> cd gitclass/                                          # move into the newly cloned repo folder
> ls                                                    # list files so you can see what you cloned
>
> vi Dockerfile                                         # create/edit a file (vi is the terminal editor)
> git status                                            # show what's changed / staged / untracked — run this A LOT
> git add Dockerfile                                    # stage Dockerfile: move it Working Dir -> Staging
> git status                                            # confirm it's now "Changes to be committed" (green)
>
> git commit Dockerfile                                 # commit — but with NO -m it opens an editor for the message
> git commit Dockerfile -m "adding new file Dockerfile" # commit WITH a message inline (the normal way)
> git status                                            # confirm "nothing to commit, working tree clean"
> git push                                              # send local commits up to the remote (origin)
>
> git branch branch1                                    # create a new branch called branch1 (doesn't switch to it!)
> git branch                                            # list LOCAL branches; the * marks the one you're on
> git branch -a                                         # list ALL branches, including remote-tracking ones
>
> git checkout branch1                                  # SWITCH to branch1 (modern alias: git switch branch1)
> git branch                                            # verify the * is now on branch1
> ls                                                    # look around on the new branch
>
> vi Dockerfile                                         # edit the file again, now on branch1
> git status                                            # see it's modified
> git diff                                              # show the exact line-by-line changes (before staging)
> git commit Dockerfile -m "adding WORKDIR and COPY commands"  # snapshot the change on branch1
> git status                                            # clean again
> git log                                               # scroll the commit history (newest first). press q to quit
> git status
>
> git push -u origin branch1                            # push branch1 AND set upstream so future 'git push' just works
> ```
>
> > [!tip] `-u` (a.k.a. `--set-upstream`) — the first-push magic word
> > The **first** time you push a *new* branch, plain `git push` errors ("no upstream"). `git push -u origin branch1` links your local `branch1` to `origin/branch1`. After that, a bare `git push` / `git pull` knows where to go. You only need `-u` once per branch.
>
> ---

## 🔗 Related

- [[Class 08 - GitOps and CI-CD]] — where this workflow gets automated in pipelines.
- [[Terminology]] — full glossary ([[Terminology#Git|Git]], [[Terminology#Branch|Branch]], [[Terminology#Merge|Merge]], [[Terminology#Remote|Remote]]).
- [[DevOps Experts — MOC]] — course map of contents.
