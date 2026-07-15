---
tags: [devops, moc, index, home]
aliases: [Home, Index, MOC, Map of Content, DevOps Experts]
---

# 🗺️ DevOps Experts — Map of Content

The home base for the whole course. Every note connects back here and through [[Terminology]].
New? Do the 5-minute [[00 - Start Here (Setup and Plugins)]] first so flashcards & callouts render.

> [!abstract] What this vault is
> A self-learning DevOps course: one note per class, each with a plain-English idea, a
> hands-on walkthrough, **XP drills**, a **collapsible self-quiz**, **spaced-repetition
> flashcards**, gotchas, and a boss challenge. Work through it, tick the boxes, collect the badges. 🎖️

---

## 🧭 Suggested learning path

Follow the arrows — each builds on the last. Concepts first, then the capstone ties it all together.

```
[[DevOps Foundations]]  (the "why")
        │
        ▼
🐳 Docker      → [[Class 01 - Docker Basics]] → [[Class 02 - Docker Networking and Images]]
🌿 Git         → [[Class 03 - Git]]
☸️ Kubernetes  → [[Class 05 - Kubernetes]] → [[Class 06 - Helm]]
🔁 Delivery    → [[Class 08 - GitOps and CI-CD]]
📜 Automation  → [[Class 11 - Ansible]] → [[Class 14 - Ansible Lab]]
🏗️ Infra       → [[Class 12 - Terraform]]
📨 Messaging   → [[Class 13 - RabbitMQ Messaging]]
        │
        ▼
🛰️ FINAL BOSS → [[SkyWatch Capstone]]  (everything combined)
```

---

## 📚 All notes

| # | Note | Topic | Badge |
|---|---|---|---|
| — | [[00 - Start Here (Setup and Plugins)]] | Obsidian setup, plugins, how to study | 🚀 |
| — | [[DevOps Foundations]] | What DevOps is: Agile, CI/CD, strategies | 🎖️ DevOps Initiate |
| — | [[Terminology]] | Master glossary (the hub) | 📖 |
| 01 | [[Class 01 - Docker Basics]] | Images vs containers, run/exec, lifecycle | 🎖️ Container Wrangler |
| 02 | [[Class 02 - Docker Networking and Images]] | Networks, name resolution, building images | 🎖️ Image Smith |
| 03 | [[Class 03 - Git]] | add/commit/push, branches, monorepo | 🎖️ Commit Captain |
| 05 | [[Class 05 - Kubernetes]] | Pods, Deployments, Services, RBAC | 🎖️ Pod Whisperer |
| 06 | [[Class 06 - Helm]] | Charts, values, templates, releases | 🎖️ Chart Chef |
| 08 | [[Class 08 - GitOps and CI-CD]] | GitHub Actions + ArgoCD loop | 🎖️ Pipeline Pilot |
| 11 | [[Class 11 - Ansible]] | Playbooks, roles, handlers, idempotency | 🎖️ Playbook Pro |
| 12 | [[Class 12 - Terraform]] | IaC, providers, state, plan/apply | 🎖️ Infra Architect |
| 13 | [[Class 13 - RabbitMQ Messaging]] | Queues, producer/consumer, AMQP | 🎖️ Queue Conductor |
| 14 | [[Class 14 - Ansible Lab]] | Dockerized control+nodes lab | 🎖️ Automation Alchemist |
| 🏆 | [[SkyWatch Capstone]] | The full end-to-end project | 🎖️ SkyWatch Commander 🛰️ |

> [!warning] 🔢 Numbering decoder — read once, save confusion forever
> **Three numbering systems exist and they disagree**, so note numbers (which follow the **course repo folders**) tell you nothing about the calendar:
> - **Repo folders** (`class1/` = Docker basics, `class3/` = Git …) — how the files are organized. Our notes follow these.
> - **The teacher's drive folders** (`class 2` = Docker, `class 3` = Git) — shifted differently.
> - **His deck titles = the real calendar**: the "class 3" Git deck's own title slide says **Class 6**, and that's the actual session count (session 1 was the intro → [[DevOps Foundations]]; several sessions per topic happen).
> - Repo folders 04, 07, 09, 10 have no hands-on materials, so they have no notes.
> - **Fresh example (2026-07-15):** the teacher's Drive folder for "lesson 4" contains `Class 4 - kubernetes.pptx` — whose own opening slide says **"Class 3"** — but the actual hands-on Kubernetes material lives in **repo folder `class5/`**, mined into [[Class 05 - Kubernetes]]. Four different numbers (drive "4", deck-internal "3", repo "5", and whatever the real calendar session is) for one topic. This is why the rule exists.
>
> **Rule of thumb: navigate by *topic*, never by number.** Every class note repeats this in a small collapsed callout under its title. When in doubt about "which class are we on," check the latest deck's title slide.

---

## 📤 Uploads & homework

- **`uploads/class-NN-<topic>/`** — the teacher's materials (pptx decks, assignment docs), added after each class. What's been mined from them already lives inside the class notes as **📬 The REAL assignment** and **🖥️ From the class deck** sections.
- **`homework/class-NN-<topic>/`** — your solution work per assignment (see the sharing-etiquette note in `homework/README.md`).
- **Official course repo (Yariv's):** https://github.com/yfreifeld/devops-course — the upstream source of the `classN/` lab files.

> The weekly ritual after each class is documented in `uploads/README.md`: drop the new files in, distill them into the class note, commit + push.

---

## 🏆 Badge tracker

Tick a badge when you clear that note's `> [!success]` conditions:

- [ ] 🎖️ DevOps Initiate — [[DevOps Foundations]]
- [ ] 🎖️ Container Wrangler — [[Class 01 - Docker Basics]]
- [ ] 🎖️ Image Smith — [[Class 02 - Docker Networking and Images]]
- [ ] 🎖️ Commit Captain — [[Class 03 - Git]]
- [ ] 🎖️ Pod Whisperer — [[Class 05 - Kubernetes]]
- [ ] 🎖️ Chart Chef — [[Class 06 - Helm]]
- [ ] 🎖️ Pipeline Pilot — [[Class 08 - GitOps and CI-CD]]
- [ ] 🎖️ Playbook Pro — [[Class 11 - Ansible]]
- [ ] 🎖️ Infra Architect — [[Class 12 - Terraform]]
- [ ] 🎖️ Queue Conductor — [[Class 13 - RabbitMQ Messaging]]
- [ ] 🎖️ Automation Alchemist — [[Class 14 - Ansible Lab]]
- [ ] 🎖️ SkyWatch Commander 🛰️ — [[SkyWatch Capstone]]

---

## 🧩 How every note is built

> [!info] Consistent structure across all class notes
> TL;DR → Learning goals → **Big idea (analogy)** → Core concepts → Guided walkthrough →
> **🔬 Drills (XP)** → **🧪 Self-quiz (collapsible)** → **🃏 Flashcards** → Gotchas →
> **🏆 Boss challenge** → *💻 Cheat-sheet (collapsed — peek only if stuck)* → Related.
>
> The cheat-sheet sits at the bottom **on purpose**: try the drills from your head first,
> reveal the commands only when you're truly stuck.

Everything cross-links through [[Terminology]]. Open **Graph view** (`Ctrl+G`) to see the web. 🕸️

---

## 🎮 Extras

- **🗡️ [Shell Quest](https://github.com/iceteps/shell-quest)** — learn by *typing the real commands* in a
  simulated terminal. 7 missions with XP and levels; missions 4 and 7 mirror the course's
  **actual graded assignments** (Docker Hub push · merge-conflict resolution).
  ```bash
  git clone https://github.com/iceteps/shell-quest && cd shell-quest
  python quest.py
  ```
- **⚡ Quick Quiz** — a fast self-test across every topic, now living **inside the
  Shell Quest repo** (one clone = both games):
  ```bash
  python quiz/quiz.py                 # 12 random questions
  python quiz/quiz.py --topic git     # one topic
  ```
- **🔎 "Learn to fish"** — every class note has a section with official-docs links and how to
  *discover* commands yourself (`--help`, `kubectl explain`, `ansible-doc`, …) so studying
  never becomes monkey-see-monkey-do.
