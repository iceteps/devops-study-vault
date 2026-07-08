---
tags: [devops, cicd, gitops, argocd, github-actions, class-08]
aliases: [GitOps, CI/CD, ArgoCD, GitHub Actions, Pipeline Pilot, Class 08]
class: 08
difficulty: intermediate
---

# 🔁 Class 08 — GitOps & CI/CD

> [!info]- 🔢 Which class is this, really?
> The number in this note's title follows the **course repo folder** `class8/` — **not** the calendar session. The real session numbering is shifted: calendar-session 1 was the DevOps intro (see [[DevOps Foundations]]), so e.g. Docker basics was *taught* in session 2 even though its repo folder is `class1`. When in doubt, navigate by **topic** via [[DevOps Experts — MOC]], not by number.


> [!abstract] TL;DR
> **CI/CD** automates the boring, error-prone path from *code on your laptop* to *running in a cluster*.
> - **CI (Continuous Integration)** = every `git push` triggers a pipeline that **lints, builds, and pushes a Docker image** tagged with the commit SHA. 🏗️
> - **CD the GitOps way** = CI then **bumps the image tag inside a Git repo's Helm values**, and **[[Terminology#ArgoCD|ArgoCD]]** — sitting *inside* the cluster — notices the git change and **pulls** it in, running `prune` + `selfHeal` to make reality match Git. 🧲
> - **Git is the single source of truth.** You never `kubectl apply` by hand. You change a file, commit, and the cluster follows. 🎯
> The whole loop: **push code → CI builds+pushes image `myapp:abc1234` → CI edits `values.yaml` tag → ArgoCD syncs cluster.**

---

## 🎯 Learning goals

- [ ] Explain the full loop: **push → CI build/push image → tag bump in GitOps repo → ArgoCD sync**.
- [ ] Read and write a GitHub Actions workflow: `on: push`, `steps`, `uses`, `secrets`.
- [ ] Understand why we tag images with `${GITHUB_SHA::7}` and never `:latest`.
- [ ] Store credentials as **GitHub Secrets** (`DOCKERHUB_USERNAME`/`DOCKERHUB_TOKEN`), never hard-code them.
- [ ] Read an ArgoCD `Application` manifest: `project`, `destination`, `source`, `syncPolicy`.
- [ ] Contrast **pull-based GitOps** vs **push-based deploys** and say why pull wins.
- [ ] Know the classic gotchas: `[skip ci]` loops, the wrong `yq`, `project: default`.

---

## 🧩 The big idea

> [!tip] Analogy — GitOps is a thermostat 🌡️
> You don't manually blow warm air around the room. You set a **setpoint** (say 22°C) and the thermostat *continuously* nudges reality toward it. If someone opens a window (drift!), the heater kicks in — no human required.
>
> In GitOps, **Git is the setpoint** ("the cluster should run image `abc1234`"). **ArgoCD is the thermostat** — it watches Git and watches the live cluster, and whenever they disagree it **syncs** to close the gap. Someone `kubectl delete`s a pod by hand? ArgoCD's **selfHeal** puts it back. That's the whole magic. ✨

**The full loop (text diagram):**

```
   👩‍💻 dev
    │  git push  (branch: dev / main)
    ▼
┌───────────────────────────────────────────────┐
│  GitHub Actions — CI  (push-based part)        │
│  1. checkout                                   │
│  2. SHA_TAG = first 7 chars of commit SHA      │
│  3. docker login (uses SECRETS)                │
│  4. docker build + push  myapp:abc1234  ──────►│──► 🐳 DockerHub (image registry)
│  5. clone GitOps repo                          │
│  6. sed -i 'tag: "abc1234"' values-dev.yaml    │
│  7. git commit + push  ────────────────────────│──► 📦 GitOps repo (Helm values)
└───────────────────────────────────────────────┘                    │
                                                                      │ git change
                                       ┌──────────────────────────────┘
                                       ▼
                        ┌──────────────────────────────────┐
                        │  ArgoCD  (pull-based part)        │
                        │  • watches GitOps repo            │
                        │  • detects new tag → SYNC         │
                        │  • prune + selfHeal               │
                        └───────────────┬──────────────────┘
                                        │ pulls & applies
                                        ▼
                                 ☸️ Kubernetes cluster
                                 (now running myapp:abc1234)
```

> [!warning] Notice the handoff
> **CI stops at "edited a file in Git."** It does **not** touch the cluster. The credentials to deploy live *inside the cluster*, not in your CI. That separation is the security superpower of GitOps.

---

## 🧠 Core concepts

**Continuous Integration (CI)** → automatically **build + test** every change so integration problems surface in minutes, not at release. Here it also builds and pushes the Docker **artifact**. See [[Terminology#CI/CD]] and [[Terminology#Pipeline]].

**Continuous Delivery/Deployment (CD)** → automatically get that tested artifact *ready to ship* (Delivery) or *actually shipped* (Deployment). In GitOps, "shipped" = a commit that changes the desired state.

**[[Terminology#GitHub Actions|GitHub Actions]]** → GitHub's built-in CI/CD engine. A **workflow** (YAML in `.github/workflows/`) has **triggers** (`on:`), **jobs**, and **steps**. Steps either `run:` a shell command or `uses:` a prebuilt **action** (e.g. `actions/checkout@v4`, `docker/login-action@v3`).

**Artifact** → the immutable build output you ship. Here it's the **Docker image** `user/myapp:abc1234`. Tagging by **commit SHA** makes every deploy traceable and reproducible. See [[Terminology#Artifact]].

**[[Terminology#GitOps|GitOps]]** → operate infrastructure by **committing to Git**. Desired state lives in a repo; an agent reconciles the live system to match. Benefits: full audit log, easy rollback (`git revert`), no snowflake clusters.

**[[Terminology#ArgoCD|ArgoCD]]** → the reconciling agent for Kubernetes. It runs *in* the cluster, watches a Git repo, and keeps the cluster equal to Git. Configured via an **`Application`** custom resource.

**[[Terminology#Sync|Sync]]** → the act of making the cluster match Git. Can be manual (button/CLI) or **automated**.

**[[Terminology#Drift|Drift]]** → when live state ≠ Git state (someone hand-edited the cluster). ArgoCD flags it as **OutOfSync**.

**[[Terminology#Self-heal|Self-heal]]** (`selfHeal: true`) → ArgoCD automatically reverts drift back to Git. Your `kubectl edit` gets undone. 😅

**Prune** (`prune: true`) → if you *delete* a resource from Git, ArgoCD deletes it from the cluster. Without prune, removed things linger as orphans.

### Pull vs Push — the key distinction 🔑

| | **Push-based deploy** | **Pull-based GitOps** |
|---|---|---|
| Who applies changes? | CI runs `kubectl apply` / `helm upgrade` | ArgoCD *inside* the cluster pulls |
| Where are cluster creds? | In CI secrets (broad access ⚠️) | Only inside the cluster ✅ |
| Drift correction? | None — CI only acts on push | Continuous self-heal ✅ |
| Source of truth | Wherever CI last ran | Git, always ✅ |
| Rollback | Re-run pipeline | `git revert` a commit ✅ |

> [!tip] Rule of thumb
> **Push** = CI reaches *into* the cluster. **Pull** = the cluster reaches *out* to Git. GitOps is pull-based, and that's why your CI in this class **never gets kube credentials** — it just edits a YAML file.

---

## 🛠️ Guided walkthrough

Goal: wire a **minimal** push → image → tag-bump → ArgoCD sync loop.

1. **Create the CI workflow.** In your app repo, add `.github/workflows/ci.yaml`:
   ```yaml
   name: CI
   on:
     push:
       branches: [dev]
   jobs:
     build:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - name: Short SHA
           id: vars
           run: echo "SHA_TAG=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT
         - uses: docker/login-action@v3
           with:
             username: ${{ secrets.DOCKERHUB_USERNAME }}
             password: ${{ secrets.DOCKERHUB_TOKEN }}
         - uses: docker/build-push-action@v5
           with:
             push: true
             tags: ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ steps.vars.outputs.SHA_TAG }}
   ```

2. **Add the secrets.** In GitHub → repo → *Settings → Secrets and variables → Actions*, add `DOCKERHUB_USERNAME` and a `DOCKERHUB_TOKEN` (a DockerHub **access token**, not your password).

3. **Add the tag-bump step** (see the `gitops.ci.yaml` snippet above): clone the GitOps repo with a `GITOPS_REPO_TOKEN`, `sed` the tag into `values-dev.yaml`, commit, push.

4. **Create the ArgoCD Application** — apply `dev.yaml` once:
   ```bash
   kubectl apply -f dev.yaml
   ```
   Because `syncPolicy.automated` is set, you never sync by hand again.

5. **Push a code change to `dev`** and *watch it happen*:
   - GitHub → **Actions** tab: your workflow runs, builds `myapp:abc1234`, pushes it, commits the tag bump.
   - ArgoCD UI: the app flips **OutOfSync → Syncing → Synced/Healthy** within its polling window (~3 min, or instantly with a webhook).
   - `kubectl get pods -n dev`: a new pod rolls out running the new image. 🎉

> [!example] What "success" looks like
> `argocd app get myapp-dev` shows **Sync Status: Synced** and **Health: Healthy**, and the running image is `.../myapp:abc1234` matching your latest commit's short SHA.

---

## 🔬 Drills (earn XP)

- [ ] **Drill 1 — Trigger literacy (10 XP).** Write an `on:` block that runs only on pushes to `main` *and* `dev`. **Done when:** it matches `class8/gitops.ci.yaml` exactly.
- [ ] **Drill 2 — Short SHA (15 XP).** Add a step that exposes `${GITHUB_SHA::7}` as a step output named `SHA_TAG`. **Done when:** a later step references it as `${{ steps.vars.outputs.SHA_TAG }}`.
- [ ] **Drill 3 — Secretsproof (15 XP).** Convert a workflow that has a hard-coded DockerHub password into one using `${{ secrets.DOCKERHUB_TOKEN }}`. **Done when:** grepping the file finds zero literal credentials.
- [ ] **Drill 4 — Tag bump (20 XP).** Write the `sed` one-liner that rewrites `tag: "..."` to the new short SHA in a values file. **Done when:** running it locally on a sample `values-dev.yaml` flips the tag and nothing else.
- [ ] **Drill 5 — Read the Application (20 XP).** From `dev.yaml`, name the four `spec` blocks and what each controls. **Done when:** you can recite project / destination / source / syncPolicy from memory.
- [ ] **Drill 6 — Drift demo (25 XP).** With selfHeal on, `kubectl scale` a deployment to 0 by hand. **Done when:** ArgoCD scales it back and you can explain *why*.
- [ ] **Drill 7 — Rollback (25 XP).** `git revert` the tag-bump commit in the GitOps repo. **Done when:** ArgoCD auto-syncs the cluster back to the previous image with no `kubectl`.

---

## 🧗 Extra credit — beyond class *(addition)*

> [!example] 🆕 These drills are an **addition** — not covered in the class materials
> They're the next skill up from what class taught, chosen because you'll meet them in real work (and in the [[SkyWatch Capstone]]). Higher XP, higher payoff.

- [ ] **Gate the build (25 XP)** — class pipeline builds on every push. Add a separate `lint` job and make the build job `needs: lint` so a flake8 failure blocks the image build. **Done when:** a deliberate lint error stops the pipeline at the right stage.
- [ ] **Manual trigger (15 XP)** — add `workflow_dispatch:` to the triggers and run the workflow from the Actions tab with a button. **Done when:** you know when humans-in-the-loop beats every-push.
- [ ] **Matrix build (30 XP)** — build/test on Python 3.11 AND 3.12 with `strategy: matrix:` instead of the two copy-pasted setup steps the class workflow used. **Done when:** one job definition produces two runs.

## 🔎 Learn to fish — find it yourself (don't just copy)

> [!tip] Beat "monkey-see-monkey-do"
> The real skill isn't memorizing commands — it's finding the right one **fast**. Pros do this all day. Build the reflex:
> 1. **Ask the tool first:** `<tool> --help`, `<tool> <subcommand> --help`, `man <tool>`.
> 2. **Official docs = source of truth** (below) — not random blogs or old Stack Overflow answers.
> 3. **Search smart:** `<what you want> site:<official-docs-domain>`; add the tool's version if behaviour changed between releases.
> 4. **Read the WHOLE error message** — it almost always names the missing flag or the fix.
> 5. `tldr <command>` gives real-world examples (install a `tldr` client — it's the friendly `man`).

**📚 Docs & references**
- [GitHub Actions docs](https://docs.github.com/en/actions)
- [Workflow syntax reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Argo CD docs](https://argo-cd.readthedocs.io/en/stable/)

**⚡ Built-in help — try these BEFORE searching**
```bash
gh --help                # GitHub CLI (gh run list, gh workflow view)
argocd app --help        # inspect/sync apps from the CLI
# In the Actions tab, open a failed run and read the red step's log top-to-bottom
```

> [!example] Power move for this class
> Actions failing? Open the run in the GitHub **Actions** tab and read the failing step's log — the exact line and fix are almost always in there. Search the error text + `site:docs.github.com`.

## 🧪 Self-check quiz

> [!question]- Where do the credentials that deploy to the cluster live in a pull-based GitOps setup?
> **Inside the cluster**, owned by ArgoCD. CI never receives kube credentials — it only edits a YAML file in Git and pushes. This shrinks the blast radius: a compromised CI runner can't reach your cluster directly.

> [!question]- Why tag images with the commit SHA (`${GITHUB_SHA::7}`) instead of `:latest`?
> `:latest` is a mutable, ambiguous pointer — you can't tell *which* code is running, and rolling back is guesswork. A SHA tag is **immutable and traceable**: `myapp:abc1234` always means exactly that commit, so deploys and rollbacks are deterministic.

> [!question]- What do `prune: true` and `selfHeal: true` each do?
> **prune** deletes cluster resources that were *removed from Git* (no orphans). **selfHeal** reverts *manual drift* — if someone hand-edits or deletes a live resource, ArgoCD restores it to match Git. Together they enforce "Git is the only truth."

> [!question]- The pipeline commits a tag bump back to the same repo it watches. What can go wrong, and how do you fix it?
> An **infinite CI loop** — the bot's commit triggers the workflow again, which commits again... Fix: put `[skip ci]` in the bot's commit message (or push the bump to a *separate* GitOps repo, which is what this class does). Path filters or `paths-ignore` also help.

> [!question]- What breaks if you omit `project: default` from the ArgoCD Application in v2.x?
> The Application is invalid — `spec.project` is **required** in ArgoCD 2.x. Every Application must belong to an AppProject; `default` is the built-in catch-all. Omit it and ArgoCD rejects the manifest.

> [!question]- Contrast pull-based and push-based deployment in one sentence each.
> **Push:** CI reaches *into* the cluster and applies changes (creds in CI, no drift correction). **Pull:** an in-cluster agent (ArgoCD) reaches *out* to Git and reconciles continuously (creds stay in-cluster, self-heals drift).

---

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- - CI (Continuous Integration)
> Automatically build + test every push so integration bugs surface early; here it also builds/pushes the Docker image.

> [!question]- - CD (Continuous Delivery/Deployment)
> Automatically make the tested artifact deployable (delivery) or deploy it (deployment); in GitOps a deploy = a Git commit.

> [!question]- - GitOps
> Operating infrastructure by committing desired state to Git and letting an agent reconcile the live system to match.

> [!question]- - ArgoCD
> In-cluster Kubernetes controller that watches a Git repo and keeps the cluster equal to it, via the `Application` CR.

> [!question]- - Artifact
> Immutable build output you ship — here the Docker image `user/myapp:<sha>`.

> [!question]- - GitHub Actions trigger `on: push`
> Runs the workflow when commits are pushed to the listed branches.

> [!question]- - `${GITHUB_SHA
> 7}`:: The first 7 characters of the commit SHA, used as an immutable, traceable image tag.

> [!question]- - GitHub Secrets
> Encrypted values (e.g. `DOCKERHUB_TOKEN`) injected as `${{ secrets.NAME }}`; never hard-coded in YAML.

> [!question]- - `syncPolicy.automated`
> Tells ArgoCD to sync without a human clicking; enables prune + selfHeal.

> [!question]- - prune (ArgoCD)
> Deletes cluster resources that were removed from Git.

> [!question]- - selfHeal (ArgoCD)
> Reverts manual drift in the live cluster back to the Git-defined state.

> [!question]- - Drift
> When live cluster state differs from the Git-defined desired state (ArgoCD marks it OutOfSync).

> [!question]- - Pull-based deploy
> Cluster agent pulls desired state from Git; creds stay in-cluster.

> [!question]- - Push-based deploy
> CI pushes changes into the cluster with `kubectl`/`helm`; creds live in CI.

> [!question]- - `project: default`
> The AppProject an Application belongs to — required in ArgoCD 2.x.

> [!question]- - `targetRevision`
> The Git branch/tag/commit ArgoCD tracks for an Application's source.

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> - CI (Continuous Integration)::Automatically build + test every push so integration bugs surface early; here it also builds/pushes the Docker image.
> - CD (Continuous Delivery/Deployment)::Automatically make the tested artifact deployable (delivery) or deploy it (deployment); in GitOps a deploy = a Git commit.
> - GitOps::Operating infrastructure by committing desired state to Git and letting an agent reconcile the live system to match.
> - ArgoCD::In-cluster Kubernetes controller that watches a Git repo and keeps the cluster equal to it, via the `Application` CR.
> - Artifact::Immutable build output you ship — here the Docker image `user/myapp:<sha>`.
> - GitHub Actions trigger `on: push`::Runs the workflow when commits are pushed to the listed branches.
> - `${GITHUB_SHA::7}`:: The first 7 characters of the commit SHA, used as an immutable, traceable image tag.
> - GitHub Secrets::Encrypted values (e.g. `DOCKERHUB_TOKEN`) injected as `${{ secrets.NAME }}`; never hard-coded in YAML.
> - `syncPolicy.automated`::Tells ArgoCD to sync without a human clicking; enables prune + selfHeal.
> - prune (ArgoCD)::Deletes cluster resources that were removed from Git.
> - selfHeal (ArgoCD)::Reverts manual drift in the live cluster back to the Git-defined state.
> - Drift::When live cluster state differs from the Git-defined desired state (ArgoCD marks it OutOfSync).
> - Pull-based deploy::Cluster agent pulls desired state from Git; creds stay in-cluster.
> - Push-based deploy::CI pushes changes into the cluster with `kubectl`/`helm`; creds live in CI.
> - `project: default`::The AppProject an Application belongs to — required in ArgoCD 2.x.
> - `targetRevision`::The Git branch/tag/commit ArgoCD tracks for an Application's source.

## ⚠️ Gotchas

> [!warning] The classics that bite everyone
> - **CI commit loops.** When the pipeline commits a tag bump into a watched repo, it can re-trigger itself forever. Add **`[skip ci]`** to the bot commit message, or push the bump to a *separate* GitOps repo (what this class does).
> - **The wrong `yq`.** `pip install yq` installs a *different* tool (a `jq` wrapper) with incompatible syntax. Use the **mikefarah `yq` binary** (Go) for `yq e -i '.image.tag = "abc1234"' file`. In the class file we sidestep this entirely with `sed`.
> - **Never hard-code secrets.** DockerHub creds go in **GitHub Secrets** (`DOCKERHUB_USERNAME`/`DOCKERHUB_TOKEN`), referenced as `${{ secrets.* }}`. A token in plain YAML is a leaked token.
> - **`project: default` is required** in ArgoCD 2.x — omitting `spec.project` makes the Application invalid.
> - **`:latest` is a trap.** Mutable tags make "what's running?" unanswerable and rollbacks unreliable. Always tag by SHA.
> - **Give ArgoCD write? No.** ArgoCD only *reads* Git and *writes* the cluster. **CI** writes Git. Keep those lanes separate.
> - **`allowEmpty: false`** guards against a bad commit wiping everything — ArgoCD refuses to sync to an empty desired state.
> - **Polling lag.** Default ArgoCD reconciliation is ~3 min. Impatient? Wire a **Git webhook** to ArgoCD for near-instant sync.
> - **`sed` is greedy.** `s/tag:.*/.../` will rewrite *every* line starting with `tag:`. Make sure your values file has exactly one image tag, or scope with `yq`.

---

## 🏆 Boss challenge

> [!example] Pilot the whole pipeline 🛩️
> **Mission:** wire `push → image → tag-bump → ArgoCD sync` end-to-end (for real, or narrate it flawlessly).
> 1. Push a trivial change to `dev`. Predict the short SHA tag *before* the run finishes.
> 2. Open the **Actions** tab — confirm the image `myapp:<sha>` lands on DockerHub.
> 3. Confirm the bot commit in the GitOps repo changed exactly one line: `tag: "<sha>"`.
> 4. Watch ArgoCD go **OutOfSync → Synced/Healthy** and the new pod roll out.
> 5. **Break it on purpose:** `kubectl scale deploy/myapp -n dev --replicas=0`. Watch **selfHeal** undo you.
> 6. **Roll back:** `git revert` the bump commit and prove the cluster follows with zero `kubectl apply`.
>
> **You win when** you can point to the running image's tag and trace it back to a specific commit — and back again.

> [!success] Class 08 cleared when you can…
> - [ ] Draw the full loop from memory: push → CI build/push image → tag bump in Git → ArgoCD pull/sync.
> - [ ] Write a `on: push` workflow that tags an image with `${GITHUB_SHA::7}` using secrets.
> - [ ] Read every block of an ArgoCD `Application` and explain `prune` + `selfHeal`.
> - [ ] Explain pull vs push and why GitOps keeps cluster creds *out* of CI.
> - [ ] Name three gotchas without looking: `[skip ci]`, the wrong `yq`, `project: default`.
>
> 🎖️ **Badge: Pipeline Pilot**

---

> [!cheatsheet]- 🔒 Command cheat-sheet — try the drills FIRST (expanding = peeking 👀)
> ⚠️ **Wait!** Expanding this is basically giving up. Did you really attempt the drill from memory? If not, collapse me and try again — the struggle is what makes it stick. Still stuck? Fine, the commands are below. 👇
>
> ### The real CI workflow (`.github/workflows/ci.yaml`) — annotated
>
> ```yaml
> name: CI
> on:
>   push:
>     branches:            # trigger ONLY on pushes to these branches
>       - main
>       - dev
> jobs:
>   ci:
>     runs-on: ubuntu-latest        # fresh VM for every run
>     steps:
>       - name: Checkout repository
>         uses: actions/checkout@v4 # pull your code onto the runner
>
>       - name: Set up Python 3.11
>         if: github.ref == 'refs/heads/main'   # conditional step: only on main
>         uses: actions/setup-python@v5
>         with: { python-version: "3.11" }
>
>       # (lint/test steps go here — commented out in the class file)
>       # - run: pip install -r requirements.txt
>       # - run: pytest
>
>       - name: Login to Docker Hub
>         uses: docker/login-action@v3
>         with:
>           username: ${{ secrets.DOCKERHUB_USERNAME }}  # from repo Secrets
>           password: ${{ secrets.DOCKERHUB_TOKEN }}     # NEVER hard-code
>
>       - name: Build and push Docker image
>         uses: docker/build-push-action@v5
>         with:
>           push: true
>           tags: yfreifeld/dockertests:${{ github.sha }}  # tag = commit SHA
> ```
>
> ### The GitOps-updating workflow (`class8/gitops.ci.yaml`) — the tag-bump magic
>
> ```yaml
> - name: Set IMAGE TAG
>   id: vars
>   run: echo "SHA_TAG=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT   # 7-char short SHA → step output
>
> - name: Build and Push Image
>   run: |
>     docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ steps.vars.outputs.SHA_TAG }} .
>     docker push   ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ steps.vars.outputs.SHA_TAG }}
>
> - name: Update Helm values file
>   run: |
>     cd helm-gitops-repo
>     # pick the values file based on which branch pushed
>     if [ "${{ github.ref_name }}" = "dev" ]; then
>       VALUES_FILE="charts/myapp/values-dev.yaml"
>     else
>       VALUES_FILE="charts/myapp/values-prod.yaml"
>     fi
>     sed -i "s/tag:.*/tag: \"${{ steps.vars.outputs.SHA_TAG }}\"/" $VALUES_FILE  # bump the tag
>     git config user.name  "github-actions"
>     git config user.email "github-actions@github.com"
>     git add $VALUES_FILE
>     git commit -m "Update image tag to ${{ steps.vars.outputs.SHA_TAG }}"
>     git push          # <-- this commit is what ArgoCD will react to
> ```
>
> ### The ArgoCD `Application` (`class8/dev.yaml`) — annotated
>
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: myapp-dev
>   namespace: argocd            # ArgoCD lives in its own namespace
> spec:
>   project: default             # REQUIRED in ArgoCD 2.x — omit it and it breaks
>   destination:                 # WHERE to deploy
>     namespace: dev
>     server: https://kubernetes.default.svc   # this same cluster
>   source:                      # WHAT to deploy (the desired state)
>     repoURL: https://github.com/<your-org>/helm-gitops-repo.git
>     targetRevision: dev        # which git branch/tag to track
>     path: charts/myapp         # folder holding the Helm chart
>     helm:
>       valueFiles:
>         - values-dev.yaml      # the file CI edits with the new tag
>   syncPolicy:
>     automated:
>       prune: true              # delete cluster resources removed from Git
>       selfHeal: true           # revert manual drift back to Git
>       allowEmpty: false        # refuse to sync to an empty desired state (safety)
>     syncOptions:
>       - CreateNamespace=true   # auto-create `dev` namespace if missing
> ```
>
> ### Handy ArgoCD CLI
>
> ```bash
> argocd app list                 # list Applications
> argocd app get myapp-dev        # health + sync status
> argocd app sync myapp-dev       # force a manual sync now
> argocd app history myapp-dev    # see past syncs (for rollback)
> argocd app rollback myapp-dev 3 # roll back to revision #3
> ```
>
> ---

## 🔗 Related

- [[Class 03 - Git]] — commits, branches, and revert (the rollback mechanism)
- [[Class 06 - Helm]] — the charts and `values.yaml` that CI bumps and ArgoCD renders
- [[SkyWatch Capstone]] — where you wire this pipeline for real
- [[Terminology]] — [[Terminology#CI/CD]], [[Terminology#Pipeline]], [[Terminology#GitHub Actions]], [[Terminology#GitOps]], [[Terminology#ArgoCD]], [[Terminology#Sync]], [[Terminology#Drift]], [[Terminology#Self-heal]], [[Terminology#Artifact]]
- [[DevOps Experts — MOC]] — course map
