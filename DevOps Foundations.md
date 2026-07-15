---
tags: [devops, foundations, concepts, methodology]
aliases: [DevOps 101]
difficulty: beginner
---

# 🧭 DevOps Foundations (the "why" behind everything)

> [!abstract] TL;DR
> **DevOps** is a culture (plus a toolchain) where the people who **build** software and the people who **run** it work as one team, shipping small changes fast and getting quick feedback from real systems. The goal: reduce the time between "we had an idea" and "it's live and healthy in production" — safely, repeatably, and automatically. Everything else in this course (Docker, Kubernetes, CI/CD, Terraform, RabbitMQ, Prometheus) is just tooling that serves that one idea.

---

## 🎯 Learning goals

- [ ] Explain what DevOps *is* to a non-technical friend in 30 seconds
- [ ] Describe the **feedback loop**: build → ship → observe → learn → repeat
- [ ] Tell **Agile/Scrum** (how we work) apart from **CI/CD** (how we ship)
- [ ] Untangle **monolith vs monorepo vs microservices** without blushing
- [ ] Map every course tool to the *problem* it solves and the *class* that teaches it
- [ ] Pick the right **deployment strategy** (rolling / blue-green / canary) for a situation
- [ ] Know *why* each tool exists — not just how to type its commands

---

## 🧩 The big idea (analogy)

Picture a **Formula 1 pit crew** 🏎️. The car (your app) comes screaming in. In under 3 seconds a coordinated team swaps tyres, refuels, checks telemetry, and launches it back onto the track. Nobody says "that's not my job." Everyone sees the same dashboard. If something's wrong, they know **immediately** and fix it before the next lap.

That's DevOps.

The old way ("throw it over the wall") looked like: developers build for months, hand a CD-ROM to the operations team, and disappear. Ops struggles to run software they didn't write; Devs never learn what breaks in production. Slow, blame-heavy, fragile.

DevOps tears down that wall. **Dev + Ops = one crew** with shared ownership, automated pipelines, and constant telemetry.

> [!tip] The three pillars to remember
> 1. **Culture** — shared ownership, blameless, fast feedback.
> 2. **Automation** — if a human does it twice, a machine should do it forever.
> 3. **Measurement** — you can't improve what you can't see ([[Terminology#Observability|observability]]). The industry's four standard numbers (**DORA metrics**): deployment frequency, lead time for changes, change-failure rate, time to restore. Fast *and* safe, measured.

> [!info] Idempotency — a word you'll hear a lot
> An operation is **idempotent** if running it once or a hundred times gives the same end state. "Make sure this package is installed" is idempotent; "install this package" (which errors on the second run) is not. DevOps tooling ([[Class 11 - Ansible|Ansible]], [[Class 12 - Terraform|Terraform]], [[Class 05 - Kubernetes|Kubernetes]]) loves idempotency because it makes automation safe to re-run.

---

## 🌀 Ways of working — Agile, Scrum, sprints, Jira

DevOps ships *fast*, but fast toward *what*? That's where **[[Terminology#Agile|Agile]]** comes in — a philosophy of building software in small increments, getting feedback, and adapting, instead of planning everything up front (the old "waterfall" way).

**[[Terminology#Scrum|Scrum]]** is the most popular Agile *framework*. Its rhythm:

- **Sprint** — a fixed timebox (often 2 weeks) in which the team commits to a chunk of work.
- **Backlog** — the prioritized list of everything left to do.
- **Daily standup** — a short sync: what I did, what I'll do, what's blocking me.
- **Sprint review + retrospective** — show what shipped, then reflect on how to work better.
- **Roles** — Product Owner (what/why), Scrum Master (removes blockers), the Dev team (how).

**[[Terminology#Jira|Jira]]** is the tool most teams use to track all of this — tickets, sprints, boards, burndown charts. (You may also hear it lovingly misspelled as "gira" 😄.)

> [!example] How it fits DevOps
> Agile decides *what small thing* to build next. CI/CD (below) is the machinery that gets that small thing safely to production. Small batches = small risk = fast feedback. Agile and DevOps are two halves of the same "go small, go fast" mindset.

---

## 🏛️ Architecture styles — monolith vs monorepo vs microservices

The single most common beginner confusion: **"mono" appears in two totally different words that mean different things.** Let's kill that confusion.

- **[[Terminology#Monolith|Monolith]]** = an **architecture**. One big deployable app; all features live in one running process.
- **[[Terminology#Monorepo|Monorepo]]** = a **repository layout**. One Git repo holding many projects/services. Says *nothing* about how they run.
- **Microservices** = an **architecture**. Many small independent services, each deployable on its own, talking over the network.

Key insight: these are **orthogonal**. You can have a monolith in a monorepo, microservices in a monorepo, or microservices spread across many repos ("polyrepo"). "Mono is better" isn't a universal truth — it depends on the axis you mean!

| Aspect | Monolith 🗿 | Monorepo 📦 | Microservices 🕸️ |
|---|---|---|---|
| What it is | Runtime architecture | Code storage layout | Runtime architecture |
| Unit of deploy | The whole app at once | (N/A — it's about code, not deploy) | Each service independently |
| Best when | Small team, early product | You want shared code + atomic commits | Large teams, independent scaling |
| Upside | Simple to build, test, deploy | One place to search; easy refactors | Scale/deploy parts independently |
| Downside | Scales as one blob; risky deploys | Tooling/CI can get heavy | Network complexity, ops overhead |
| Course link | — | [[Class 03 - Git]] | [[Class 05 - Kubernetes]], [[Class 13 - RabbitMQ Messaging]] |

> [!tip] Rule of thumb
> Start with a **monolith** (simple!). Split into **microservices** only when team size, scaling needs, or deploy pain actually forces it. Whether you keep it all in a **monorepo** is a separate decision about developer ergonomics.

---

## 🔧 The DevOps toolchain map

Every tool in this course exists to solve one specific *concern*. Memorize the mapping — concern first, tool second.

| Concern | Tool | Class that teaches it |
|---|---|---|
| Version control | Git | [[Class 03 - Git]] |
| Containers (package the app + its deps) | Docker | [[Class 01 - Docker Basics]] |
| Orchestration (run containers at scale) | Kubernetes | [[Class 05 - Kubernetes]] |
| Packaging K8s apps (templated manifests) | Helm | [[Class 06 - Helm]] |
| Configuration management (set up existing machines) | Ansible | [[Class 11 - Ansible]] |
| Provisioning (create the infrastructure) | Terraform | [[Class 12 - Terraform]] |
| CI/CD & GitOps (automated build → test → deploy) | GitHub Actions / ArgoCD | [[Class 08 - GitOps and CI-CD]] |
| Messaging / decoupling services | RabbitMQ | [[Class 13 - RabbitMQ Messaging]] |
| Observability (metrics, dashboards, alerts) | Prometheus / Grafana | [[SkyWatch Capstone]] |

> [!info] Config management vs provisioning — the classic exam trap
> **[[Class 12 - Terraform|Terraform]] provisions** — it *creates* infrastructure that doesn't exist yet (VMs, networks, load balancers). Think "build the house."
> **[[Class 11 - Ansible|Ansible]] configures** — it takes existing machines and *sets them up* (install packages, edit config, start services). Think "furnish the house."
> Terraform is **declarative** ("here's the desired end-state, make it so"). Ansible leans **procedural/idempotent** ("run these steps to reach the state"). In practice teams use both: Terraform to make the servers, Ansible to configure them.

> [!info] Containers vs VMs at a glance 🚢
> A **VM** virtualizes *hardware* — each VM ships a whole guest OS (heavy, slow to boot, GBs). A **container** virtualizes the *OS* — it shares the host kernel and packages just your app + its libraries (light, boots in ms, MBs). Containers give you "works on my machine → works everywhere" without the bulk of a full VM. That's why [[Class 01 - Docker Basics|Docker]] became the unit of shipping, and why we need [[Class 05 - Kubernetes|Kubernetes]] to herd thousands of them.

> [!question] Why does Kubernetes even exist?
> One Docker container on your laptop is easy. Now run **500** of them across **20 machines**, restart the ones that crash, scale up when traffic spikes, roll out a new version with zero downtime, and reconnect networking automatically. Doing that by hand is impossible. **Kubernetes** (aka **[[Terminology#Kubernetes|"k8s"]]** — the student's note calls it "cognights" 😉, which is roughly how "k-eights" sounds) is the **orchestrator** that does all of this for you: scheduling, self-healing, scaling, and rolling updates.

---

## 🚦 Deployment strategies

Once your pipeline can ship, *how* do you release a new version without breaking users? Three classic strategies:

| Strategy | How it works | Risk | Rollback | Use when |
|---|---|---|---|---|
| **[[Terminology#Rolling update\|Rolling update]]** 🔄 | Replace instances a few at a time until all run the new version | Medium — bad version reaches some users gradually | Roll forward/back instance by instance | Default in [[Class 05 - Kubernetes\|Kubernetes]]; stateless apps |
| **[[Terminology#Blue-green deployment\|Blue-green]]** 🔵🟢 | Run two full environments; flip all traffic from old (blue) to new (green) at once | Low — instant cutover, instant switch-back | Flip the router back to blue | You need instant rollback & can afford double infra |
| **[[Terminology#Canary deployment\|Canary]]** 🐤 | Send a small % of traffic to the new version, watch metrics, then ramp up | Lowest — only a few users see a bad release | Stop the rollout, drain the canary | You want to validate with real traffic before full release |

> [!tip] Mnemonic
> **Rolling** = gradual swap. **Blue-green** = two worlds, flip a switch. **Canary** = test on a few brave birds first. 🐤 (The name comes from "canary in a coal mine.")

> [!info] What about Jenkins?
> **Jenkins** (the student's "jenkies") is the classic, veteran **CI server** — for years it was *the* way to automate build/test/deploy pipelines via "Jenkinsfiles" and a huge plugin ecosystem. Modern teams increasingly use **GitHub Actions** (config lives with your code) and **ArgoCD** (GitOps, covered in [[Class 08 - GitOps and CI-CD]]), but Jenkins is still everywhere in industry — know that it exists and what it does.

---

## 🔬 Drills (earn XP)

- [ ] **(15 XP)** Draw the **CI/CD loop** from memory: commit → build → test → package → deploy → observe → back to commit. **Done when:** you can sketch it with no reference and name a course tool at each stage.
- [ ] **(10 XP)** Explain **idempotency** to a friend using a real example. **Done when:** they can give you a *new* idempotent vs non-idempotent example back.
- [ ] **(20 XP)** Fill a blank table mapping each course tool → the real-world problem it solves. **Done when:** all 9 rows of the toolchain map are correct from memory.
- [ ] **(15 XP)** Write one paragraph untangling **monolith vs monorepo vs microservices**, explicitly stating which axis (runtime vs code layout) each belongs to. **Done when:** a beginner reads it and stops confusing "mono" words.
- [ ] **(10 XP)** For each deployment strategy, name **one scenario** where it's the best choice. **Done when:** you justify the pick in one sentence each.
- [ ] **(15 XP)** Explain the **VM vs container** difference and why it makes Kubernetes necessary. **Done when:** you connect "lightweight containers" → "need for an orchestrator."
- [ ] **(15 XP)** Describe the **Scrum sprint cycle** and where DevOps automation plugs in. **Done when:** you place standup, sprint, review, retro, and CI/CD in the right order.

**Total available: 100 XP** 🎯

---

## 🧪 Self-check quiz

> [!question]- What single sentence captures what DevOps *is*?
> A culture + toolchain that unites the people who build software with those who run it, to ship small changes fast, safely, and repeatably — with constant feedback from production.

> [!question]- How is Agile/Scrum different from CI/CD?
> **Agile/Scrum** is *how we organize work* (sprints, backlog, standups — deciding what small thing to build next). **CI/CD** is *the automated machinery* that builds, tests, and ships that work to production. Different layers of the same "go small, go fast" idea.

> [!question]- A friend says "monorepo is a type of architecture." Correct them.
> No — a **monorepo** is a *code storage layout* (one Git repo, many projects). Architecture is about how software *runs*: **monolith** (one deployable) vs **microservices** (many independent services). You can put either architecture in a monorepo — they're orthogonal.

> [!question]- Terraform vs Ansible — which provisions and which configures?
> **Terraform provisions** — creates infrastructure that doesn't exist yet (declarative desired-state). **Ansible configures** — sets up existing machines (install/config/start, idempotently). Build the house vs furnish the house.

> [!question]- Why can't we just run raw Docker containers in production without Kubernetes?
> Because at scale you need scheduling across many machines, self-healing of crashed containers, autoscaling, service networking, and zero-downtime rollouts — all automatically. Kubernetes is the orchestrator that provides these; doing it by hand is infeasible.

> [!question]- Which deployment strategy gives the safest real-traffic validation, and how?
> **Canary** 🐤 — route a small percentage of real users to the new version, watch its metrics/observability, then gradually ramp up (or abort) based on what you see.

---

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- DevOps
> A culture + toolchain uniting Dev and Ops to ship small changes fast, safely, and repeatably with constant production feedback.

> [!question]- Feedback loop
> build → ship → observe → learn → repeat, as fast and tight as possible.

> [!question]- Idempotency
> An operation you can run many times with the same end result — the backbone of safe automation.

> [!question]- Agile
> A philosophy of building software in small, adaptive increments instead of one big up-front plan.

> [!question]- Scrum
> An Agile framework built around fixed-length sprints, a backlog, standups, reviews, and retrospectives.

> [!question]- Sprint
> A fixed timebox (often 2 weeks) in which a team commits to and delivers a chunk of work.

> [!question]- Jira
> The popular tool for tracking Agile work — tickets, sprints, boards, burndowns.

> [!question]- Monolith
> A runtime architecture where the whole app is one deployable unit.

> [!question]- Monorepo
> A code-storage layout: one Git repository holding many projects/services.

> [!question]- Microservices
> A runtime architecture of many small, independently deployable services talking over the network.

> [!question]- Container vs VM
> A container shares the host kernel and packages just the app (light); a VM ships a whole guest OS (heavy).

> [!question]- Kubernetes
> The orchestrator that schedules, scales, self-heals, and rolls out containers across many machines.

> [!question]- Provisioning (Terraform)
> Declaratively creating infrastructure that doesn't exist yet.

> [!question]- Configuration management (Ansible)
> Idempotently setting up existing machines (install/config/start services).

> [!question]- Message queue
> A buffer that decouples services — producers drop messages, consumers pick them up when ready (RabbitMQ).

> [!question]- Observability
> The ability to understand a system's internal state from its outputs — metrics, logs, traces (Prometheus/Grafana).

> [!question]- Rolling update
> Replace instances a few at a time until all run the new version.

> [!question]- Blue-green deployment
> Two full environments; flip all traffic from old (blue) to new (green) instantly.

> [!question]- Canary deployment
> Route a small % of traffic to the new version, watch metrics, then ramp up.

> [!question]- Jenkins
> The classic CI server for automating build/test/deploy pipelines via Jenkinsfiles and plugins.

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> DevOps::A culture + toolchain uniting Dev and Ops to ship small changes fast, safely, and repeatably with constant production feedback.
> Feedback loop::build → ship → observe → learn → repeat, as fast and tight as possible.
> Idempotency::An operation you can run many times with the same end result — the backbone of safe automation.
> Agile::A philosophy of building software in small, adaptive increments instead of one big up-front plan.
> Scrum::An Agile framework built around fixed-length sprints, a backlog, standups, reviews, and retrospectives.
> Sprint::A fixed timebox (often 2 weeks) in which a team commits to and delivers a chunk of work.
> Jira::The popular tool for tracking Agile work — tickets, sprints, boards, burndowns.
> Monolith::A runtime architecture where the whole app is one deployable unit.
> Monorepo::A code-storage layout: one Git repository holding many projects/services.
> Microservices::A runtime architecture of many small, independently deployable services talking over the network.
> Container vs VM::A container shares the host kernel and packages just the app (light); a VM ships a whole guest OS (heavy).
> Kubernetes::The orchestrator that schedules, scales, self-heals, and rolls out containers across many machines.
> Provisioning (Terraform)::Declaratively creating infrastructure that doesn't exist yet.
> Configuration management (Ansible)::Idempotently setting up existing machines (install/config/start services).
> Message queue::A buffer that decouples services — producers drop messages, consumers pick them up when ready (RabbitMQ).
> Observability::The ability to understand a system's internal state from its outputs — metrics, logs, traces (Prometheus/Grafana).
> Rolling update::Replace instances a few at a time until all run the new version.
> Blue-green deployment::Two full environments; flip all traffic from old (blue) to new (green) instantly.
> Canary deployment::Route a small % of traffic to the new version, watch metrics, then ramp up.
> Jenkins::The classic CI server for automating build/test/deploy pipelines via Jenkinsfiles and plugins.

## 🏆 Boss challenge

> [!example] Explain the **whole [[SkyWatch Capstone|SkyWatch]] pipeline end-to-end** using ONLY the concepts on this page.
> Out loud (or in writing), narrate the full journey of a code change using this page's vocabulary — no hand-waving:
> 1. A ticket in **[[Terminology#Jira|Jira]]** defines the small increment for this **sprint**.
> 2. You commit code to **[[Class 03 - Git|Git]]**.
> 3. **[[Class 08 - GitOps and CI-CD|CI/CD]]** kicks off: build a **[[Class 01 - Docker Basics|Docker]]** image, run tests, push the image.
> 4. **GitOps (ArgoCD)** sees the new manifest and tells **[[Class 05 - Kubernetes|Kubernetes]]** to deploy it — packaged with **[[Class 06 - Helm|Helm]]**.
> 5. Infra underneath was **provisioned** by **[[Class 12 - Terraform|Terraform]]** and configured by **[[Class 11 - Ansible|Ansible]]**.
> 6. Services stay decoupled via **[[Class 13 - RabbitMQ Messaging|RabbitMQ]]** message queues.
> 7. The rollout uses a **rolling / canary** strategy for zero-downtime release.
> 8. **[[SkyWatch Capstone|Prometheus/Grafana]]** observe the result — metrics close the **feedback loop**, and the next Jira ticket begins.
>
> **Cleared when** you can do all 8 steps smoothly, naming the concern *and* the tool at each stage.

---

## 🔗 Related

- [[Class 01 - Docker Basics]]
- [[Class 03 - Git]]
- [[Class 05 - Kubernetes]]
- [[Class 06 - Helm]]
- [[Class 08 - GitOps and CI-CD]]
- [[Class 11 - Ansible]]
- [[Class 12 - Terraform]]
- [[Class 13 - RabbitMQ Messaging]]
- [[SkyWatch Capstone]]
- [[Terminology]]
- [[DevOps Experts — MOC]]

---

> [!success] Foundations cleared when you can…
> …explain what DevOps is in one breath, map every course tool to the problem it solves, untangle the three "mono/micro" architecture terms, pick the right deployment strategy for a situation, and narrate the full SkyWatch pipeline end-to-end using only these concepts.
>
> 🎖️ **Badge: DevOps Initiate**
