---
tags: [devops, helm, kubernetes, class-06]
aliases: [Helm, Helm Charts, Kubernetes Package Manager, Class 6]
class: 06
difficulty: intermediate
---

# ⎈ Class 06 — Helm

> [!info]- 🔢 Which class is this, really?
> The number in this note's title follows the **course repo folder** `class6/` — **not** the calendar session. The numbering is genuinely chaotic — **three systems disagree**: the repo folders, the teacher's drive folders, and his deck titles (which reflect the real calendar). Case in point: the Git material lives in repo folder `class3/` and drive folder `class 3`, but its own deck title says **Class 6** — the actual session count. Session 1 was the DevOps intro (see [[DevOps Foundations]]). Bottom line: navigate by **topic** via [[DevOps Experts — MOC]], never by number.


> [!abstract] TL;DR
> **Helm** is the **package manager for Kubernetes** — think `apt`/`npm`/`brew`, but for k8s apps. Instead of hand-writing and `kubectl apply`-ing a pile of YAML, you bundle it into a **[[Terminology#Chart|Chart]]**: reusable templates (`templates/`) full of placeholders like `{{ .Values.replicaCount }}`, filled in from a **[[Terminology#Values|values.yaml]]** file. `helm install` renders the templates + values into real manifests and ships them to the cluster as a versioned **[[Terminology#Release|Release]]**. Swap the values file per environment (`-f values-dev.yaml`) and the *same chart* deploys to dev, staging, or prod. `helm upgrade` rolls forward, `helm rollback` rolls back. **One chart, many deploys, zero copy-paste.**

## 🎯 Learning goals
- [ ] Explain what a Helm **chart**, **release**, and **values** file are — and how they relate
- [ ] Read a chart file tree: `Chart.yaml`, `values.yaml`, `templates/`
- [ ] Write & read templates using `{{ .Values.x }}` and `{{ .Release.Name }}`
- [ ] Run the core loop: `helm create` → `template` → `install` → `upgrade` → `list` → `uninstall`
- [ ] Override values **per environment** with `-f values-dev.yaml` and `--set`
- [ ] Say **why** Helm beats raw `kubectl apply` for repeatable deploys

---

## 🧩 The big idea

> [!example] Analogy: Helm chart = an IKEA flat-pack with a values sheet 🪑
> A raw Kubernetes YAML file is like a **fully-built, glued-together bookshelf** — great, but if you want a slightly taller one you rebuild it from scratch.
>
> A **Helm chart** is the **flat-pack kit**: the `templates/` are the pre-cut planks with holes drilled where the screws go (`{{ .Values.replicaCount }}`), and `values.yaml` is the **little instruction sheet** that says "use 4 shelves, paint it nginx-colour, ship version latest." Want a *different* shelf for the dev room? Don't reprint the planks — just hand over a **different sheet** (`values-dev.yaml`). Same kit, different result.

Another way to see it: **templates + values = mail-merge for YAML**. The template is the letter with `Dear {{ name }}`, the values file is the spreadsheet of names, and `helm template`/`install` is the "merge" button that spits out finished documents.

> [!tip] Why this matters
> Real apps aren't one YAML file — they're a Deployment + Service + ConfigMap + Ingress + Secret + HPA… all with the same app name, image, and port repeated everywhere. Change the image tag by hand across 6 files and you *will* typo one. Helm makes the app **one parameterised unit** you version, share, install, and roll back atomically.

---

## 🧠 Core concepts

**Helm** → the CLI + templating engine that packages, installs, and manages Kubernetes apps. It's a client-side tool (Helm 3 needs no server-side "Tiller"). See [[Terminology#Helm]].

**Chart** → a folder of files that describes a related set of [[Terminology#Kubernetes|Kubernetes]] resources. The unit Helm installs. See [[Terminology#Chart]].

**Values** → the configuration inputs for a chart, living in `values.yaml` (defaults) and overridable at install time. Accessed in templates as `.Values.*`. See [[Terminology#Values]].

**Release** → a specific *installed instance* of a chart in a cluster, with a name and a version history. Install the same chart twice with different names → two releases. See [[Terminology#Release]].

**Template** → a YAML file in `templates/` containing Go-template placeholders (`{{ ... }}`) that Helm renders into a real manifest. See [[Terminology#Template]].

### 📂 The chart file tree (from class6 `my-service/`)

```text
my-service/
├── Chart.yaml            # metadata: chart name, version, appVersion
├── values.yaml           # DEFAULT config values (the "instruction sheet")
└── templates/            # the parameterised manifests (the "planks")
    ├── deployment.yaml   # renders a Deployment from values
    └── service.yaml      # renders a Service from values
```

**`Chart.yaml`** — the chart's ID card. From class6:
```yaml
apiVersion: v2                 # v2 = Helm 3 charts
name: my-microservice          # the CHART name (≠ release name!)
description: A simple web service deployment
version: 0.1.0                 # version of THE CHART itself (bump on chart edits)
appVersion: "1.0.0"            # version of the APP inside (informational)
```

**`values.yaml`** — the defaults every template reads from:
```yaml
replicaCount: 4
image:
  repository: nginx
  tag: "latest"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
```

**`templates/deployment.yaml`** — note the placeholders. This is the whole point:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deploy        # ← the RELEASE name, chosen at install
spec:
  replicas: {{ .Values.replicaCount }}    # ← pulled from values.yaml
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: microservice
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

> [!tip] The two built-in objects you'll use constantly
> - **`.Values.*`** → whatever's in `values.yaml` (or overridden with `-f`/`--set`). `.Values.image.tag` walks the YAML tree.
> - **`.Release.Name`** → the name you pass to `helm install <name>`. Prefixing resources with it (e.g. `{{ .Release.Name }}-deploy`) lets you install the same chart many times without name collisions.

---

## 🛠️ Guided walkthrough — the core loop

A hands-on mini-lab. Run it against your kind/minikube/dev cluster.

1. **Scaffold a chart**
   ```bash
   helm create my-app
   ls my-app        # → Chart.yaml  charts/  templates/  values.yaml
   ```

2. **Edit `values.yaml`** — set replicas and image:
   ```yaml
   replicaCount: 2
   image:
     repository: nginx
     tag: "1.25"
   ```

3. **Preview the render (no cluster needed!)**
   ```bash
   helm template my-release ./my-app
   ```
   *Expected:* fully-rendered YAML on stdout — you should see `replicas: 2`, the image `nginx:1.25`, and resource names prefixed with `my-release`. **No cluster is contacted.**

4. **Install for real**
   ```bash
   helm install my-release ./my-app
   ```
   *Expected:*
   ```text
   NAME: my-release
   LAST DEPLOYED: ...
   NAMESPACE: default
   STATUS: deployed
   REVISION: 1
   ```

5. **Look at what got created**
   ```bash
   kubectl get all
   ```
   *Expected:* a `deployment.apps/my-release-...`, a `service/my-release-...`, and 2 running pods.

6. **Change a value and upgrade** — scale to 3:
   ```bash
   helm upgrade my-release ./my-app --set replicaCount=3
   kubectl get deployment
   ```
   *Expected:* the deployment now shows `READY 3/3`, and `helm list` shows **REVISION 2**.

7. **Clean up**
   ```bash
   helm uninstall my-release
   helm list          # → empty
   kubectl get all    # → the release's resources are gone
   ```

> [!warning] `helm template` vs `helm install`
> `helm template` renders **locally** and prints YAML — it **never** talks to the cluster and doesn't create a release. It's for previewing/debugging. `helm install`/`upgrade` actually apply to the cluster and record a revision. Don't confuse "it rendered fine" with "it deployed."

---

## 🔬 Drills (earn XP)

> 🗡️ **Warm up in [Shell Quest](https://github.com/iceteps/shell-quest):** mission 11 "Package It ⎈" — template → install → upgrade --set → rollback → history, risk-free. Type `demo` to watch it solved first.

> [!note] The dev override file the drills use — `class6/values-dev.yaml` (real contents)
> ```yaml
> replicaCount: 4
> image:
>   repository: nginx
>   tag: "latest"
> service:
>   type: ClusterIP
>   port: 80
> cron:
>   schedule: "* * * * *"    # the CronJob template reads this
> daemonset:
>   resources:
>     requests: { cpu: 200m, memory: 128Mi }
>     limits:   { cpu: 200m, memory: 256Mi }
> ```
> So vs the chart default: **replicaCount 4**, plus the `cron.*` and `daemonset.*` keys the extra templates consume — these are the "differing fields" Drill 3 asks you to spot.

- [ ] **(10 XP)** Run `helm create drill-app`, then `ls drill-app/templates`. List the files Helm scaffolded. **Done when:** you can name at least 3 files under `templates/` and say what `_helpers.tpl` is for (named template partials).
- [ ] **(15 XP)** In `my-service`, run `helm template rel ./my-service` and find the line that becomes the Deployment's `name`. **Done when:** you can explain why it reads `rel-deploy` and not `my-microservice-deploy` (release name vs chart name).
- [ ] **(15 XP)** Render with the dev override: `helm template rel ./my-service -f values-dev.yaml`. **Done when:** you can point to the field that differs from the default `values.yaml` render (hint: `cron.schedule` / `daemonset...cpu`).
- [ ] **(20 XP)** Install `my-service` into a fresh namespace: `helm upgrade --install rel ./my-service -n dev --create-namespace -f values-dev.yaml`. **Done when:** `helm list -n dev` shows STATUS `deployed` and `kubectl get all -n dev` shows the pods.
- [ ] **(20 XP)** Bump the image tag with an inline override: `helm upgrade rel ./my-service -n dev --set image.tag=1.25`. **Done when:** `helm history rel -n dev` shows REVISION 2 and `kubectl describe deploy -n dev` shows `nginx:1.25`.
- [ ] **(25 XP)** Break something on purpose: set `replicaCount: two` (a string) in `values.yaml`, run `helm template`. **Done when:** you can read the render error and explain that `replicas` expects an int.
- [ ] **(25 XP)** Roll back: after the image bump, run `helm rollback rel 1 -n dev`. **Done when:** the deployment's image reverts and `helm history` shows a new REVISION pointing back to the old spec.

---

## 🧗 Extra credit — beyond class *(addition)*

> [!example] 🆕 These drills are an **addition** — not covered in the class materials
> They're the next skill up from what class taught, chosen because you'll meet them in real work (and in the [[SkyWatch Capstone]]). Higher XP, higher payoff.

- [ ] **helm rollback (25 XP)** — class ended at upgrade. Do `helm upgrade` with a broken value (bad image tag), see pods fail, then `helm rollback <release> 1`. **Done when:** `helm history <release>` tells the whole story.
- [ ] **--set vs -f (15 XP)** — override one value three ways: edit values.yaml, `-f values-dev.yaml`, and `--set replicaCount=4`. Figure out the precedence order. **Done when:** you can rank them without looking.
- [ ] **Grow the chart (30 XP)** — add a `templates/configmap.yaml` fed from a new `config:` block in values.yaml, mount it into the deployment, and `helm upgrade`. **Done when:** `kubectl get configmap` shows your rendered values.

## 🔎 Learn to fish — find it yourself (don't just copy)

> [!tip] Beat "monkey-see-monkey-do"
> The real skill isn't memorizing commands — it's finding the right one **fast**. Pros do this all day. Build the reflex:
> 1. **Ask the tool first:** `<tool> --help`, `<tool> <subcommand> --help`, `man <tool>`.
> 2. **Official docs = source of truth** (below) — not random blogs or old Stack Overflow answers.
> 3. **Search smart:** `<what you want> site:<official-docs-domain>`; add the tool's version if behaviour changed between releases.
> 4. **Read the WHOLE error message** — it almost always names the missing flag or the fix.
> 5. `tldr <command>` gives real-world examples (install a `tldr` client — it's the friendly `man`).

**📚 Docs & references**
- [Helm docs](https://helm.sh/docs/)
- [Chart template guide](https://helm.sh/docs/chart_template_guide/)
- [Artifact Hub (find charts)](https://artifacthub.io/)

**⚡ Built-in help — try these BEFORE searching**
```bash
helm --help              # every subcommand
helm <cmd> --help        # e.g. helm upgrade --help
helm show values <chart> # see every value you can override before installing
```

> [!example] Power move for this class
> Before installing any chart, run `helm show values <chart>` to see the full knobs list — that's how you know what to put in your `values.yaml`.

## 🧪 Self-check quiz

> [!question]- What's the difference between the **chart name** and the **release name**?
> The **chart name** (`name:` in `Chart.yaml`, e.g. `my-microservice`) identifies the *package*. The **release name** is what you pass to `helm install <name> ...` — the *installed instance*. In templates, `{{ .Release.Name }}` gives the release name, which is why the class6 deployment is `<release>-deploy`, not `my-microservice-deploy`. You can install one chart under many release names.

> [!question]- Does `helm template` deploy anything to the cluster?
> **No.** It renders templates + values into final YAML **locally** and prints it. It contacts no cluster and creates no release. It's your primary tool for previewing and debugging what Helm *would* apply. Only `helm install`/`helm upgrade` touch the cluster.

> [!question]- You have `values.yaml` with `replicaCount: 4` and also pass `-f values-dev.yaml` with `replicaCount: 2`. What wins, and what if you add `--set replicaCount: 5`?
> Later sources override earlier ones. Precedence (low → high): chart's `values.yaml` → `-f` files (in order) → `--set`. So `-f values-dev.yaml` overrides the default to **2**, and `--set replicaCount=5` beats them all → **5**.

> [!question]- Why is `helm upgrade --install` preferred over plain `helm install` in CI?
> It's **idempotent**: it installs the release if it doesn't exist and upgrades it if it does. CI can run the exact same command on every commit with no "does this release already exist?" branching — no errors from re-installing an existing release.

> [!question]- What does `{{ .Values.image.repository }}:{{ .Values.image.tag }}` render to, given the class6 defaults?
> `nginx:latest` — `.Values.image.repository` = `nginx`, `.Values.image.tag` = `latest`. Helm walks the nested YAML in `values.yaml` via dotted paths.

> [!question]- How do you undo a bad upgrade without editing YAML?
> `helm rollback <release> <revision>`. Helm records every `install`/`upgrade` as a numbered revision (`helm history <release>`), so you roll back to a known-good one atomically.

---

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- Helm
> The package manager for Kubernetes — a CLI + templating engine that bundles k8s manifests into charts and installs/upgrades/rolls them back as releases.

> [!question]- Chart
> A folder (`Chart.yaml` + `values.yaml` + `templates/`) describing a related set of Kubernetes resources; the unit Helm installs.

> [!question]- values.yaml
> The default configuration inputs for a chart; overridable at install time and read in templates via `.Values.*`.

> [!question]- Release
> A named, versioned installed instance of a chart in a cluster (has a revision history).

> [!question]- Template
> A file in `templates/` with Go-template placeholders (`{{ ... }}`) that Helm renders into a real manifest.

> [!question]- Chart.yaml `version` vs `appVersion`
> `version` = version of the chart package itself; `appVersion` = version of the app inside (informational).

> [!question]- helm template
> Renders chart + values to YAML **locally**; does NOT contact the cluster or create a release.

> [!question]- helm install
> Renders and applies a chart, recording it as revision 1 of a new release.

> [!question]- helm upgrade
> Applies changes to an existing release, creating a new revision.

> [!question]- helm uninstall
> Removes all resources of a release in one operation.

> [!question]- helm rollback
> Reverts a release to a previous revision number.

> [!question]- Values precedence
> `values.yaml` < `-f file` < `--set` (later wins).

> [!question]- chart name vs release name
> Chart name identifies the package; release name is the installed instance (`.Release.Name`).

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> Helm::The package manager for Kubernetes — a CLI + templating engine that bundles k8s manifests into charts and installs/upgrades/rolls them back as releases.
> Chart::A folder (`Chart.yaml` + `values.yaml` + `templates/`) describing a related set of Kubernetes resources; the unit Helm installs.
> values.yaml::The default configuration inputs for a chart; overridable at install time and read in templates via `.Values.*`.
> Release::A named, versioned installed instance of a chart in a cluster (has a revision history).
> Template::A file in `templates/` with Go-template placeholders (`{{ ... }}`) that Helm renders into a real manifest.
> Chart.yaml `version` vs `appVersion`::`version` = version of the chart package itself; `appVersion` = version of the app inside (informational).
> helm template::Renders chart + values to YAML **locally**; does NOT contact the cluster or create a release.
> helm install::Renders and applies a chart, recording it as revision 1 of a new release.
> helm upgrade::Applies changes to an existing release, creating a new revision.
> helm uninstall::Removes all resources of a release in one operation.
> helm rollback::Reverts a release to a previous revision number.
> Values precedence::`values.yaml` < `-f file` < `--set` (later wins).
> chart name vs release name::Chart name identifies the package; release name is the installed instance (`.Release.Name`).

## ⚠️ Gotchas

> [!warning] Traps that bite everyone
> - **`helm template` renders without a cluster.** "It rendered" ≠ "it deployed." A perfect render can still fail on `install` (e.g. image doesn't exist, RBAC denies it).
> - **Templates are indentation-sensitive YAML.** A misplaced `{{ ... }}` or wrong indent produces invalid YAML. Render first; `helm template` (or `helm install --dry-run`) shows the exact broken output.
> - **Release name ≠ chart name.** Resources are named from `{{ .Release.Name }}`, not `Chart.yaml`'s `name`. Two installs = two releases; reusing a release name **upgrades**, it doesn't create a second copy.
> - **Types matter.** `replicaCount: "2"` (string) where an int is expected can render `replicas: "2"` and get rejected by the API. Quote strings, leave numbers unquoted.
> - **`--set` beats `-f` beats `values.yaml`.** If an override "isn't taking," a later, higher-precedence source is winning.
> - **Bump `Chart.yaml` `version` when you change the chart**, not just `appVersion`. `version` is the *chart* package version; downstream tooling and repos rely on it.
> - **Namespaces aren't auto-created.** Use `-n <ns> --create-namespace` (as in class6) or the install fails on a missing namespace.

---

## 🏆 Boss challenge — Chart Chef final

**Package an app as a chart and upgrade it to a new image tag — the full lifecycle, no hand-editing manifests.**

1. Start from `class6/my-service` (or `helm create` your own). Confirm `Chart.yaml`, `values.yaml`, and `templates/deployment.yaml` + `service.yaml` exist.
2. **Preview** the dev render: `helm template boss ./my-service -f values-dev.yaml` — verify the placeholders resolved (release-prefixed names, correct replicas, `nginx:latest`).
3. **Deploy** to a fresh namespace:
   ```bash
   helm upgrade --install boss ./my-service -n dev --create-namespace -f values-dev.yaml
   ```
   Verify with `helm list -n dev` (STATUS `deployed`) and `kubectl get all -n dev`.
4. **Upgrade to a new image tag** without touching any YAML:
   ```bash
   helm upgrade boss ./my-service -n dev -f values-dev.yaml --set image.tag=1.25
   ```
   Confirm with `helm history boss -n dev` (REVISION 2) and `kubectl describe deploy -n dev | grep Image` → `nginx:1.25`.
5. **Roll back** to revision 1 and confirm the image reverts: `helm rollback boss 1 -n dev`.
6. **Clean up:** `helm uninstall boss -n dev`.

**Win condition:** you moved through create → template → install → upgrade → rollback → uninstall using **only** values and CLI overrides — never editing a rendered manifest by hand.

> [!success] Class 06 cleared when you can…
> …explain chart / release / values in one breath, read `{{ .Values.x }}` and `{{ .Release.Name }}` in a template, run the full `helm create → template → install → upgrade → rollback → uninstall` loop, and deploy the *same chart* to two environments with `-f values-dev.yaml`. **Do that and you've earned it.**
>
> 🎖️ **Badge: Chart Chef** — you cook one chart and plate it for every environment.

---

> [!cheatsheet]- 🔒 Command cheat-sheet — try the drills FIRST (expanding = peeking 👀)
> ⚠️ **Wait!** Expanding this is basically giving up. Did you really attempt the drill from memory? If not, collapse me and try again — the struggle is what makes it stick. Still stuck? Fine, the commands are below. 👇
>
> ```bash
> # --- environment sanity ---
> kubectl get nodes                 # cluster reachable?
> helm version                      # Helm installed? (need Helm 3+)
>
> # --- scaffold a brand-new chart with best-practice boilerplate ---
> helm create my-app                # generates Chart.yaml, values.yaml, templates/, etc.
>
> # --- RENDER locally, DON'T touch the cluster (your #1 debugging tool) ---
> helm template my-release ./my-app # prints the final YAML Helm *would* apply
>
> # --- install: render + apply + record a release ---
> helm install my-release ./my-app
> helm list                         # show installed releases (add -A for all namespaces)
> kubectl get all                   # see the pods/deploy/svc Helm created
>
> # --- upgrade: push a change (new value) into an existing release ---
> helm upgrade my-release ./my-app --set replicaCount=3   # inline override
> helm history my-release           # every revision of this release
> helm rollback my-release 1        # go back to revision 1
>
> # --- environment overrides via a values FILE (the clean way) ---
> helm upgrade --install yariv-release ./my-service/ -n dev \
>   --create-namespace -f values-dev.yaml         # install-or-upgrade in one shot
>
> # --- multiple --set overrides at once ---
> helm upgrade --install yariv-release ./my-service/ -n dev --create-namespace \
>   --set cron.schedule="* * * * *" \
>   --set replicaCount=2 \
>   --set daemonset.resources.requests.cpu=200m
>
> # --- tear it down ---
> helm uninstall my-release         # removes ALL resources of the release in one go
> ```
>
> > [!tip] `helm upgrade --install` is the GitOps-friendly hero
> > It **installs** if the release doesn't exist yet, and **upgrades** if it does. Idempotent — safe to run in CI on every commit without branching logic. That's the exact form used in class6.
>
> ---

## 🔗 Related
- [[Class 05 - Kubernetes]] — the resources (Deployments, Services) Helm packages
- [[Class 08 - GitOps and CI-CD]] — where `helm upgrade --install` runs automatically
- [[Terminology]] — [[Terminology#Helm]] · [[Terminology#Chart]] · [[Terminology#Values]] · [[Terminology#Release]] · [[Terminology#Template]] · [[Terminology#Kubernetes]]
- [[DevOps Experts — MOC]] — course map
