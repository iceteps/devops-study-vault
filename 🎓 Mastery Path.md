---
tags: [devops, mastery, learning-path, dashboard]
aliases: [Mastery Path, Learning Path, Become a Master, Skill Tree]
---

# 🎓 Mastery Path — from student to master, topic by topic

> [!abstract] What this is
> Your **skill tree** for the whole course. Every topic has the same four-rank ladder, and every rank has **concrete, checkable criteria** wired to things that already exist: this vault's drills and boss challenges, [Shell Quest](https://github.com/iceteps/shell-quest) missions, the quiz, and the REAL graded assignments. No vibes — you either meet the bar or you don't yet. Tick the boxes **in this note**; it's your dashboard.

---

## 🪜 The ladder (same 4 ranks everywhere)

| Rank | The bar | How it feels |
|---|---|---|
| 🥉 **Apprentice** | You've *understood* — read the note, cards in rotation | "I can follow along" |
| 🥈 **Practitioner** | You've *done it once* — walkthrough + game mission beaten | "I can do it with the note open" |
| 🥇 **Expert** | You've *done it under pressure* — all drills, boss cleared, quiz ≥80% | "I can do it from memory" |
| 🏆 **Master** | You've *proven it in the real world* — real assignment / artifact + you can teach it | "I can debug it AND explain it" |

**House rules:** ranks are per-topic and in order — no skipping. The game's `demo` mode is a legit Apprentice→Practitioner bridge: watch it solved once, then beat it yourself (demoed objectives pay no XP, so the ladder stays honest). "Teach it" = say the answer OUT LOUD in 3–5 sentences without looking; if you stumble, you're not done.

## 🔄 The loop (how a study session actually goes)

**Daily 15 minutes:** open Obsidian → `Ctrl+P → Spaced Repetition: Review flashcards` (5 min) → one Shell Quest mission or one drill block from the topic you're ranking up (10 min). Done.

**Per topic (the rank climb):** Read the note top-to-bottom **once** → flashcards into rotation → walkthrough with the note open → game mission (`demo` first if it's all new) → drills from memory → boss challenge → `python quiz/quiz.py --topic <t>` → real assignment / proof-artifact → teach-it test.

**Weekly (after each class):** new teacher files → `uploads/` → note updated → new flashcards → check whether Shell Quest/quiz grew a mission for it → schedule the new topic's climb.

---

## 🧭 Foundations

- [ ] 🥉 Read [[DevOps Foundations]]; its 20 flashcards in rotation; explain CI vs CD in one sentence each.
- [ ] 🥈 Draw the course toolchain map from memory (git → CI → registry → K8s → monitoring) and place every course tool on it.
- [ ] 🥇 `python quiz/quiz.py --topic foundations` ≥80%; name the three pillars + all four DORA metrics cold.
- [ ] 🏆 Teach it: explain to a rubber duck why "you build it, you run it" changes *incentives*, not just process; give a canary-vs-blue-green recommendation for SkyWatch and defend it.

## 🐳 Docker — [[Class 01 - Docker Basics]] + [[Class 02 - Docker Networking and Images]]

- [x] 🥉 Both notes read; flashcards (12+18) in rotation. *(long done)*
- [x] 🥈 Shell Quest missions 1–4 beaten — incl. "Ship It ⚓" (the real Assignment 1 rehearsal).
- [ ] 🥇 All drills in both notes + both boss challenges; `quiz.py --topic docker` ≥80%; explain layer caching and `COPY requirements.txt` ordering without notes.
- [x] 🏆 **Real assignment SUBMITTED 2026-07-15:** [hub.docker.com/r/iceteps/hello-docker-app](https://hub.docker.com/r/iceteps/hello-docker-app) (public, `1.0` + `latest`). Remaining Master check: ⬜ teach-it — why does `push` need login when `pull` doesn't? What exactly is a layer?

## 🌿 Git — [[Class 03 - Git]]

- [x] 🥉 Note read; 18 flashcards in rotation.
- [x] 🥈 Shell Quest missions 5–7 beaten (conflict finale).
- [ ] 🥇 All drills + boss; `quiz.py --topic git` ≥80%; run the full stash → rebase → tag → revert tour in a scratch repo from memory.
- [x] 🏆 **Real assignment SUBMITTED 2026-07-15:** [github.com/iceteps/git-python-practice](https://github.com/iceteps/git-python-practice) — core Parts 1–9 **plus the full 11-task bonus** (stash, rebase, tag v1.0.0, .gitignore, detached HEAD, both recovery scenarios). Remaining Master check: ⬜ teach-it — merge vs rebase, and when `reset --hard` is safe.

## ☸️ Kubernetes — [[Class 05 - Kubernetes]]  ← **📍 you are here**

- [ ] 🥉 Note read end-to-end (incl. BOTH deck sections + both REAL-assignment callouts); all 29+ flashcards in rotation.
- [ ] 🥈 Shell Quest missions **8, 9, 10** beaten (8 = the CLI assignment, 10 = the RBAC homework; `demo` first is fair).
- [ ] 🥇 All 7 drills + the 3 extra-credit drills + boss challenge on a REAL minikube; `quiz.py --topic k8s` ≥80%; the `-n` namespace reflex is automatic.
- [ ] 🏆 **Real assignment #1 (CLI):** execute + submit — scaffold is ready at `C:\Users\icete\github\Devops\kubernetes-basics-assignment\`. **Real assignment #2 (Core Resources & RBAC):** all 9 parts + written answers. Teach-it: the yes→no RBAC flip, and why deleting a Deployment-owned pod does nothing.

## ⎈ Helm — [[Class 06 - Helm]]

- [ ] 🥉 Note read; 13 flashcards in rotation; chart/release/values in one breath.
- [ ] 🥈 Shell Quest mission **11** beaten (template → install → upgrade --set → rollback → history).
- [ ] 🥇 All drills (incl. the break-it-on-purpose render error) + boss; `quiz.py --topic helm` ≥80%.
- [ ] 🏆 Proof-artifact: package YOUR flask app (the Docker assignment one) as a chart and install it twice (`dev` + `prod` values) on minikube. Teach-it: what exactly does `helm rollback` restore, and what does it NOT touch?

## 🔁 GitOps / CI-CD — [[Class 08 - GitOps and CI-CD]]

- [ ] 🥉 Note read; 16 flashcards in rotation; pull-vs-push table reproduced from memory.
- [ ] 🥈 Shell Quest mission **12** beaten (push → robot pipeline → OutOfSync → sync).
- [ ] 🥇 All drills + boss; `quiz.py --topic gitops` ≥80%; explain `[skip ci]` and the two yq's without notes.
- [ ] 🏆 Proof-artifact: a real GitHub Actions workflow on one of YOUR repos that builds + pushes an image tagged `${GITHUB_SHA::7}` on push (you already have `devops-course/.github/workflows/ci.yaml` to evolve). Teach-it: why does CI never hold kube credentials in GitOps?

## 📜 Ansible — [[Class 11 - Ansible]] + [[Class 14 - Ansible Lab]]

- [ ] 🥉 Both notes read; flashcards (12+13) in rotation.
- [ ] 🥈 Shell Quest mission **13** beaten (the changed=0 run is the point).
- [ ] 🥇 All drills + bosses in both notes (the class-14 Docker lab counts as your real sandbox); `quiz.py --topic ansible` ≥80%.
- [ ] 🏆 Proof-artifact: one playbook that configures the class-14 lab nodes with nginx + templated index.html, **twice, second run changed=0**, screenshot of both recaps. Teach-it: handlers — when do they run, how many times, and why.

## 🏗️ Terraform — [[Class 12 - Terraform]]

- [ ] 🥉 Note read; 15 flashcards in rotation; the variable-precedence chain recited.
- [ ] 🥈 Shell Quest mission **14** beaten (full lifecycle incl. the literal `yes`).
- [ ] 🥇 All drills + boss; `quiz.py --topic terraform` ≥80%; explain state (and why losing it hurts) without notes.
- [ ] 🏆 Proof-artifact: `terraform apply` the class-12 config against real AWS free-tier (S3 backend per `class12/commands`), then **destroy it the same session** and show the empty `state list`. Teach-it: plan-vs-apply, and what `.tfstate` maps to what.

## 📨 RabbitMQ — [[Class 13 - RabbitMQ Messaging]]

- [ ] 🥉 Note read; 19 flashcards in rotation; queue-vs-exchange table cold.
- [ ] 🥈 Shell Quest mission **15** beaten (produce with nobody listening → inspect depth → drain).
- [ ] 🥇 All drills + boss with the real class-13 compose lab; `quiz.py --topic rabbitmq` ≥80%; run TWO consumers and watch round-robin with your own eyes.
- [ ] 🏆 Proof-artifact: modify the class producer/consumer pair to survive a broker restart (durable queue + persistent messages — the extra-credit drill) and demonstrate zero message loss. Teach-it: work-queue vs fanout in two sentences.

## 🌦️ FINAL BOSS — [[SkyWatch Capstone]]

Unlocks when every topic above is 🥇+. This is all nine topics in one build:

- [ ] Terraform the 3-node cluster → Ansible installs K3s → GHCR images via CI → Helm chart → ArgoCD auto-sync → RabbitMQ RPC pipeline → Prometheus + Grafana.
- [ ] The capstone's own "cleared when": the full runbook **from memory, fresh AWS account, under an hour** (`assignment.md → Session Quick-Start` is the reference).
- [ ] 🏆🏆 **Course Master:** every topic at 🏆 + capstone cleared + you can present the architecture diagram and answer "why" for every arrow on it.

---

## 📊 Dashboard

| Topic | 🥉 | 🥈 | 🥇 | 🏆 |
|---|---|---|---|---|
| Foundations | ⬜ | ⬜ | ⬜ | ⬜ |
| Docker | ✅ | ✅ | ⬜ | ✅* |
| Git | ✅ | ✅ | ⬜ | ✅* |
| Kubernetes 📍 | ⬜ | ⬜ | ⬜ | ⬜ |
| Helm | ⬜ | ⬜ | ⬜ | ⬜ |
| GitOps / CI-CD | ⬜ | ⬜ | ⬜ | ⬜ |
| Ansible | ⬜ | ⬜ | ⬜ | ⬜ |
| Terraform | ⬜ | ⬜ | ⬜ | ⬜ |
| RabbitMQ | ⬜ | ⬜ | ⬜ | ⬜ |
| **SkyWatch capstone** | — | — | — | ⬜ |

\* assignments submitted; the 🥇 drills row and the teach-it check are honest gaps — close them and flip the star to a clean ✅. *(Yes, you can hold 🏆-the-artifact while 🥇-the-fluency still has boxes open — the star marks exactly that state.)*

> [!tip] Keep this note pinned
> This is the vault's **home page for progress**. Open it at the start of every study session, pick the lowest open box on the topic you're climbing, and go. The [[DevOps Experts — MOC]] stays the map of *content*; this note is the map of *you*.
