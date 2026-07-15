---
tags: [devops, kubernetes, k8s, class-05]
aliases: [K8s, Kubernetes Core Objects, Class 5, Pods and Deployments]
class: 05
difficulty: intermediate
---

# ☸️ Class 05 — Kubernetes

> [!info]- 🔢 Which class is this, really?
> The number in this note's title follows the **course repo folder** `class5/` — **not** the calendar session. The numbering is genuinely chaotic — **three systems disagree**: the repo folders, the teacher's drive folders, and his deck titles (which reflect the real calendar). Case in point: the Git material lives in repo folder `class3/` and drive folder `class 3`, but its own deck title says **Class 6** — the actual session count. Session 1 was the DevOps intro (see [[DevOps Foundations]]). Bottom line: navigate by **topic** via [[DevOps Experts — MOC]], never by number.


> [!abstract] TL;DR
> **Kubernetes (K8s)** is a container orchestrator: you hand it a **desired state** written in YAML, and it works nonstop to make reality match. You almost never run raw containers — you declare **Pods** (via **Deployments**), expose them with **Services**, route external traffic through an **Ingress**, feed them with **ConfigMaps** + **Secrets**, lock them down with **RBAC**, and give them disk with a **PVC**. The `kubectl apply -f file.yaml` command is the doorway to all of it. If a Pod dies, K8s **self-heals** by spinning up a replacement. Learn the objects + how they wire together and you own the platform.

---

## 🎯 Learning goals

- [ ] Explain the **declarative desired-state** model (you say *what*, K8s figures out *how*).
- [ ] Name the core objects and what each is *for*: Pod, Deployment/ReplicaSet, Service, Ingress, ConfigMap, Secret, RBAC, PVC, Job/CronJob, DaemonSet.
- [ ] Wire the mental model: **Deployment → ReplicaSet → Pods ← Service ← Ingress**.
- [ ] Run the core `kubectl` workflow: `apply`, `get`, `describe`, `scale`, `logs`, `delete`.
- [ ] Watch **self-healing** live by killing a Pod and seeing it respawn.
- [ ] Deploy the whole stack from a single multi-doc manifest (`full-k8s-deploy.yaml`).

---

## 🧩 The big idea

Think of Kubernetes as the **conductor of an orchestra** 🎻. You don't tell each violinist when to bow — you hand the conductor the *score* (your YAML = the **desired state**) and say "make it sound like this." The conductor keeps every section in sync, and if a violinist walks off stage, the conductor instantly waves in an understudy. That "keep reality matching the score, forever" loop is the **reconciliation loop** — the beating heart of K8s.

Another framing: a **shipping port** 🚢. Containers arrive; the port (control plane) decides which dock (node) each goes to, keeps the right number afloat, and reroutes when one sinks. You just publish the manifest.

**How the core objects fit together:**

```
                            🌍 Internet
                                 │
                          ┌──────▼───────┐
                          │   Ingress    │  host/path routing (example.com/)
                          └──────┬───────┘
                                 │
                          ┌──────▼───────┐
                          │   Service    │  stable IP + DNS, load-balances
                          │  (ClusterIP) │  selector: app=demo
                          └──────┬───────┘
                                 │  (matches label app=demo)
             ┌───────────────────┼───────────────────┐
             ▼                   ▼                    ▼
        ┌─────────┐         ┌─────────┐          ┌─────────┐
        │  Pod 🟢 │         │  Pod 🟢 │          │  Pod 🟢 │   ← replicas: 3
        └─────────┘         └─────────┘          └─────────┘
             ▲                   ▲                    ▲
             └───────────────────┴────────────────────┘
                                 │  creates & heals
                          ┌──────┴───────┐
                          │  ReplicaSet  │  keeps count = 3
                          └──────┬───────┘
                                 │  owned & rolled by
                          ┌──────┴───────┐
                          │  Deployment  │  the thing YOU write
                          └──────────────┘

   Side feeds:  ConfigMap 📄 + Secret 🔐 → env/vars into Pods
                PVC 💾 → persistent disk for Pods
                ServiceAccount + Role + RoleBinding 🛡️ → who-can-do-what
```

> [!tip] The golden rule
> **You manage the top of the chain; K8s manages the bottom.** You edit the **Deployment**; it owns the ReplicaSet; the ReplicaSet owns the Pods. Never hand-create Pods for a real app — always go through a Deployment so self-healing and rollouts work.

---

## 🧠 Core concepts

See [[Terminology#Kubernetes]] and [[Terminology#kubectl]] for the glossary anchors.

### 🏗️ Workloads (things that run your code)

- **Pod** → the smallest deployable unit: one (or a few tightly-coupled) containers sharing a network + storage. Ephemeral — treated as cattle, not pets. → [[Terminology#Pod]]
- **Deployment** → the object *you* actually write for stateless apps. Declares `replicas: N` and a Pod `template`, then manages rollouts and rollbacks. → [[Terminology#Deployment]]
- **ReplicaSet** → the middle-manager the Deployment creates automatically; its only job is "keep exactly N matching Pods alive." You rarely touch it directly.
- **Job** → run a task **to completion** once, then stop (e.g. compute π, run a migration). → [[Terminology#Job]]
- **CronJob** → a Job on a **schedule** (cron syntax, e.g. `* * * * *` = every minute). → [[Terminology#CronJob]]
- **DaemonSet** → runs **one Pod per node** — perfect for node-level agents like log shippers or monitors. → [[Terminology#DaemonSet]]

### 🌐 Networking (how traffic reaches Pods)

- **Service** → a stable IP + DNS name in front of a shifting set of Pods, chosen by **label selector**. Pods come and go; the Service endpoint stays put. → [[Terminology#Service]]
  - **ClusterIP** (default) → reachable **only inside** the cluster. Great for internal APIs.
  - **NodePort** → opens a port on **every node** in the range **30000–32767**, so external clients can hit `NodeIP:nodePort`. → [[Terminology#NodePort]]
  - **LoadBalancer** → asks the cloud provider for an external load balancer with a public IP.
- **Ingress** → HTTP(S) router at the edge: maps `host`/`path` (e.g. `example.com/`) to a backend Service. One entry point for many Services. Needs an Ingress **controller** installed to actually work. → [[Terminology#Ingress]]

### ⚙️ Config (externalize settings from images)

- **ConfigMap** → **non-secret** key/value config (`ENV: dev`, `LOG_LEVEL: debug`) injected as env vars or files. Change config without rebuilding the image. → [[Terminology#ConfigMap]]
- **Secret** → same idea for sensitive data (passwords, tokens). Stored **base64-encoded** — that is encoding, **NOT encryption**. → [[Terminology#Secret]]

### 🛡️ Security (RBAC — who can do what)

Three-piece combo → [[Terminology#RBAC]]:
- **ServiceAccount** → an identity **for Pods/apps** (not humans), e.g. `app-sa`.
- **Role** → a namespaced list of allowed **verbs on resources** (e.g. `get`, `list` on `pods`, `configmaps`).
- **RoleBinding** → the glue: grants a Role **to** a subject (our ServiceAccount). No binding = no permissions.

### 💾 Storage

- **PersistentVolumeClaim (PVC)** → a Pod's **request** for durable storage ("give me 1Gi, `ReadWriteOnce`"). Survives Pod restarts, unlike a Pod's ephemeral disk. → [[Terminology#PersistentVolumeClaim]]

### 🗂️ Namespace (the folder that scopes it all)

- **Namespace** → a virtual cluster-within-a-cluster (e.g. `dev`) for isolating and grouping resources. Names only need to be unique *within* a namespace. → [[Terminology#Namespace]]

> [!warning] Namespace scoping bites everyone once
> Almost every object above lives **inside a namespace**. If `kubectl get pods` shows nothing, you probably forgot `-n dev`. Set a default context namespace or you'll keep chasing "missing" resources.

---

## 🛠️ Guided walkthrough — mini-lab

> [!example] Prereqs
> A local cluster: **minikube**, **kind**, **k3d**, or **Docker Desktop → Enable Kubernetes**. Verify with `kubectl get nodes` (you want at least one `Ready` node). All files below are in `devops-course/class5/`.

**1. Create the namespace.**
```bash
kubectl apply -f namespace.yaml
# Expected: namespace/dev created
```

**2. Deploy the app (3 replicas) + expose it.**
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
# Expected: deployment.apps/app-deployment created
#           service/app-service created
```

**3. Watch the Pods appear.**
```bash
kubectl get pods -n dev
# Expected (names vary):
# NAME                              READY   STATUS    RESTARTS   AGE
# app-deployment-7c9f8d6b5-abcde    1/1     Running   0          20s
# app-deployment-7c9f8d6b5-fghij    1/1     Running   0          20s
# app-deployment-7c9f8d6b5-klmno    1/1     Running   0          20s
```

**4. Scale up to 5.**
```bash
kubectl scale deployment app-deployment --replicas=5 -n dev
kubectl get pods -n dev        # now 5 Pods Running
```

**5. Break it on purpose — witness self-healing. 🩹**
```bash
kubectl delete pod <one-pod-name> -n dev
kubectl get pods -n dev        # briefly 4/5, then a NEW pod appears -> back to 5
# The ReplicaSet noticed count < desired and reconciled. Magic (it's a loop).
```

**6. Deploy the whole stack in one shot.**
```bash
kubectl apply -f full-k8s-deploy.yaml
# Creates namespace, ServiceAccount, ConfigMap, Secret, PVC, Pod,
# Deployment, Service, Ingress, Role, RoleBinding — all at once.
```

**7. Verify everything landed.**
```bash
kubectl get all -n dev
kubectl get pvc -n dev
kubectl get configmap,secret -n dev
kubectl auth can-i get pods --as=system:serviceaccount:dev:app-sa -n dev
# Expected last line: yes   (because the RoleBinding grants pod-reader)
```

**8. Clean up (optional).**
```bash
kubectl delete namespace dev   # nukes everything scoped inside it
```

---

## 🔬 Drills (earn XP)

- [ ] **Drill 1 — Namespace + Deploy (10 XP).** Apply `namespace.yaml` then `deployment.yaml`.
      **Done when:** `kubectl get pods -n dev` shows 3 Pods `Running`.
- [ ] **Drill 2 — Scale dance (10 XP).** Scale to 5, then back to 2.
      **Done when:** pod count changes both times without editing YAML.
- [ ] **Drill 3 — Self-heal (15 XP).** Delete one Pod and observe recovery.
      **Done when:** desired replica count is restored automatically.
- [ ] **Drill 4 — Config + Secret (15 XP).** Apply `configmap.yaml` and `secret.yaml`, then decode the secret: `kubectl get secret app-secret -n dev -o jsonpath='{.data.password}' | base64 -d`.
      **Done when:** you print the plaintext and understand base64 ≠ encryption.
- [ ] **Drill 5 — Service types (15 XP).** Read `service-types.yaml`; explain ClusterIP vs NodePort vs LoadBalancer aloud.
      **Done when:** you can say when to use each and the NodePort range 30000–32767.
- [ ] **Drill 6 — RBAC proof (20 XP).** Apply serviceaccount/role/rolebinding, then run the `auth can-i` check.
      **Done when:** the check returns `yes`; delete the RoleBinding and confirm it flips to `no`.
- [ ] **Drill 7 — Full stack (25 XP).** Apply `full-k8s-deploy.yaml` and verify with `kubectl get all -n dev`.
      **Done when:** Namespace, Deployment, Service, Ingress, PVC, ConfigMap, Secret, and RBAC all exist in `dev`.

---

## 🧗 Extra credit — beyond class *(addition)*

> [!example] 🆕 These drills are an **addition** — not covered in the class materials
> They're the next skill up from what class taught, chosen because you'll meet them in real work (and in the [[SkyWatch Capstone]]). Higher XP, higher payoff.

- [ ] **Probes — the self-healing trigger (30 XP)** — class relied on default health. Add a `livenessProbe` (httpGet or tcpSocket) to the deployment, then break it on purpose (wrong port) and watch K8s restart the pod in a loop (`kubectl get pods -w`). **Done when:** you can explain liveness vs readiness in one sentence each.
- [ ] **Rollout + rollback (30 XP)** — `kubectl set image deployment/app-deployment app=nginx:1.27 -n dev`, watch `kubectl rollout status`, then `kubectl rollout undo`. **Done when:** `kubectl rollout history` shows both revisions.
- [ ] **requests vs limits (20 XP)** — add `resources.requests` and `resources.limits` to a pod; then set a limit lower than the app needs and watch it get OOMKilled (`kubectl describe pod`). **Done when:** you know which one the scheduler uses.

## 🖥️ From the class deck *(addition — mined from `class4-kubernetes.pptx`)*

> [!info] Four things the slides stress that this note's core-concepts section doesn't spell out
> 1. **Everything talks through the API Server — nothing talks to etcd, the Scheduler, or Kubelet directly.** The Scheduler *watches* the API Server for unassigned Pods, the Controller Manager *fetches* desired state from it and *writes back* changes, and Kubelet *polls* it for Pod specs then reports status back. etcd is the API Server's private backend database — no other component touches it directly. One hub, everyone else spokes.
> 2. **Master components vs Node components — the deck's own split.** *Master* (control plane): API Server, etcd, Scheduler, Controller Manager — make cluster-wide decisions, can run on any machine. *Node*: kubelet, kube-proxy, container runtime — run on **every** node, actually execute the workloads. This maps directly onto this note's [[#🏗️ Workloads (things that run your code)|Workloads]] vs the control plane you never touch.
> 3. **Kubernetes isn't the only orchestrator** — the deck opens with **Docker Swarm** (native Docker clustering) and **Mesosphere Marathon** (long-running-app framework on Apache Mesos) as siblings, then explains why K8s won: environmental consistency (laptop == cloud), cloud/OS portability, and application-centric management over raw VM abstraction.
> 4. **`kubectl` has two working styles**, and the deck names both explicitly: **Declarative** (write YAML, `kubectl apply -f`) — everything this note's walkthrough uses — vs **Ad Hoc** (one-off imperative commands like `kubectl run`, `kubectl create deployment ... --image=...`). The deck's own demo: `kubectl create deployment hello-node --image=...` → `kubectl scale --replicas=3` → `kubectl expose --type=ClusterIP` is the Ad Hoc equivalent of this note's `deployment.yaml` + `service.yaml`.
>
> **Pod networking, one layer deeper than this note goes:** every Pod gets its own cluster-routable IP. Same-node Pods talk over a veth pair; cross-node Pods route through the CNI's virtual overlay network — either way the Pod IP stays reachable and location-transparent. DNS-based service discovery (Pods resolve Services **by name**, not IP) sits on top of this.
>
> 📄 Full deck: [class4-kubernetes.pptx](uploads/class-04-kubernetes/class4-kubernetes.pptx) *(in this repo's `uploads/`)*
>
> 🔢 **Numbering, again:** the file is titled "Class 4" but its own opening slide says **"Class 3"** (and credits a different instructor, Eduard Usatchev, than the deck's Drive owner). A third disagreement to add to the pile — see the callout at the top of this note. Navigate by topic.

## 📬 The REAL assignment (from Yariv's drive)

> [!important] 🧪 Kubernetes Basics — CLI Assignment (YAML Provided)
> The actual graded homework — **CLI only, no Helm, and you may not edit the provided YAML files**:
> 1. **Install** `kubectl`, `minikube`, and Docker/Podman. Verify: `kubectl version --client`, `minikube version`.
> 2. **Start & inspect** the cluster: `minikube start`, then `kubectl cluster-info`, `kubectl get nodes`, `kubectl get namespaces`, `kubectl get pods -n kube-system`.
> 3. **Explore Services** cluster-wide: `kubectl get services -A` — and be ready to explain what a Kubernetes Service *is*.
> 4. **Deploy** the provided four-file `k8s/` folder (`backend-deployment.yaml` + `backend-service.yaml` → `nginxdemos/hello` on **ClusterIP**; `frontend-deployment.yaml` + `frontend-service.yaml` → `nginx:alpine` on **NodePort**) with a single `kubectl apply -f .` from inside `k8s/`.
> 5. **Verify**: `kubectl get deployments`, `get pods`, `get services` all show both apps running.
> 6. **Reach the frontend from a browser** — Minikube has a one-command way to open a Service directly (the assignment deliberately doesn't name it: `minikube service <name>` is the tool to go find yourself).
> 7. **Inspect**: `kubectl describe deployment frontend`, `describe pod <name>`, `logs <name>`.
> 8. **Tear down**: `kubectl delete -f .`, then confirm `get pods`/`get deployments`/`get services` are all empty.
> 9. **Bonus:** scale the backend, and explain *why the frontend is reachable but the backend isn't* — that's this note's own [[#🌐 Networking (how traffic reaches Pods)|ClusterIP vs NodePort]] distinction, graded.
>
> 📄 Full assignment: [kubernetes-basics-cli-assignment.docx](uploads/class-04-kubernetes/kubernetes-basics-cli-assignment.docx) *(in this repo's `uploads/`)*

## 🔎 Learn to fish — find it yourself (don't just copy)

> [!tip] Beat "monkey-see-monkey-do"
> The real skill isn't memorizing commands — it's finding the right one **fast**. Pros do this all day. Build the reflex:
> 1. **Ask the tool first:** `<tool> --help`, `<tool> <subcommand> --help`, `man <tool>`.
> 2. **Official docs = source of truth** (below) — not random blogs or old Stack Overflow answers.
> 3. **Search smart:** `<what you want> site:<official-docs-domain>`; add the tool's version if behaviour changed between releases.
> 4. **Read the WHOLE error message** — it almost always names the missing flag or the fix.
> 5. `tldr <command>` gives real-world examples (install a `tldr` client — it's the friendly `man`).

**📚 Docs & references**
- [Kubernetes docs](https://kubernetes.io/docs/home/)
- [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [API / object reference](https://kubernetes.io/docs/reference/kubernetes-api/)

**⚡ Built-in help — try these BEFORE searching**
```bash
kubectl --help           # every subcommand
kubectl <cmd> --help     # e.g. kubectl create --help
kubectl explain pod.spec.containers   # describe ANY field of ANY resource
```

> [!example] Power move for this class
> `kubectl explain` is a superpower: `kubectl explain deployment.spec.strategy` documents any field offline — no need to hunt through YAML examples.

## 🧪 Self-check quiz

> [!question]- What is the difference between the *desired state* and the *actual state*, and who closes the gap?
> **Desired state** = what your YAML declares (e.g. `replicas: 3`). **Actual state** = what's really running right now. The **control plane's reconciliation loop** continuously compares them and takes action (create/delete Pods) until actual == desired. That loop is why K8s self-heals.

> [!question]- You delete a Pod created by a Deployment. What happens, and why?
> A new Pod is created almost immediately. The Deployment owns a **ReplicaSet** whose sole job is keeping the count at the declared number. Deleting a Pod drops the count below desired, so the ReplicaSet reconciles by creating a replacement. (Delete the *Deployment* instead and the Pods stay gone.)

> [!question]- Why can't a client on the internet reach a `ClusterIP` Service directly?
> `ClusterIP` is only routable **inside** the cluster. To expose externally you use **NodePort** (opens a port 30000–32767 on each node), **LoadBalancer** (cloud external IP), or an **Ingress** in front of the Service.

> [!question]- Are Kubernetes Secrets encrypted?
> No — by default they are only **base64-encoded**, which is trivially reversible (`base64 -d`). Anyone with read access to the Secret can decode it. Real protection needs encryption-at-rest (etcd), RBAC restrictions, and/or an external secrets manager.

> [!question]- What three objects make up an RBAC grant for an app, and what does each do?
> **ServiceAccount** (the app's identity, e.g. `app-sa`), **Role** (a namespaced set of allowed verbs on resources, e.g. `get`/`list` on `pods`), and **RoleBinding** (attaches the Role to the ServiceAccount). Without the RoleBinding, the ServiceAccount has no permissions.

> [!question]- When would you choose a DaemonSet over a Deployment?
> When you need **exactly one Pod per node** — node-level agents like log collectors, monitoring, or CNI plugins. A Deployment gives you N replicas placed anywhere; a DaemonSet tracks the node set and runs one Pod on each.

---

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- Kubernetes
> Open-source container orchestrator that keeps your declared desired state true via a continuous reconciliation loop.

> [!question]- kubectl
> The CLI you use to talk to the Kubernetes API server (`apply`, `get`, `describe`, `scale`, `logs`, `delete`).

> [!question]- Pod
> Smallest deployable unit — one or more containers sharing network + storage; ephemeral.

> [!question]- Deployment
> Declares replicas + a Pod template; manages ReplicaSets, rollouts, and rollbacks for stateless apps.

> [!question]- ReplicaSet
> Ensures exactly N matching Pods are running; created and owned by a Deployment.

> [!question]- Self-healing
> K8s automatically replaces failed/deleted Pods to match the desired replica count.

> [!question]- Service
> Stable IP + DNS name that load-balances across Pods selected by label.

> [!question]- ClusterIP
> Default Service type — reachable only from inside the cluster.

> [!question]- NodePort
> Service type exposing a port (30000–32767) on every node for external access.

> [!question]- LoadBalancer
> Service type that provisions a cloud external load balancer with a public IP.

> [!question]- Ingress
> HTTP(S) edge router mapping host/path to backend Services; needs an Ingress controller.

> [!question]- Namespace
> Virtual cluster for scoping/isolating resources (e.g. `dev`); names unique within it.

> [!question]- ConfigMap
> Non-secret key/value config injected as env vars or files.

> [!question]- Secret
> Sensitive key/value data stored base64-encoded (encoding, not encryption).

> [!question]- ServiceAccount
> Identity for apps/Pods (not humans) used by RBAC.

> [!question]- Role
> Namespaced set of allowed verbs on resources.

> [!question]- RoleBinding
> Grants a Role to a subject such as a ServiceAccount.

> [!question]- RBAC
> Role-Based Access Control — ServiceAccount + Role + RoleBinding define who can do what.

> [!question]- Job
> Runs a task to completion once, then stops.

> [!question]- CronJob
> Runs a Job on a cron schedule (e.g. `* * * * *` = every minute).

> [!question]- DaemonSet
> Runs exactly one Pod per node — ideal for node-level agents.

> [!question]- PersistentVolumeClaim
> A Pod's request for durable storage (size + access mode) that survives restarts.

> [!question]- kubectl apply -f
> Declarative command that creates or updates resources to match a manifest.

> [!question]- Desired state
> The end state you declare in YAML; the control plane works to make reality match it.

> [!question]- Master components (control plane)
> API Server, etcd, Scheduler, Controller Manager — make cluster-wide decisions; can run on any machine.

> [!question]- Node components
> kubelet, kube-proxy, container runtime — run on every node, actually execute the workloads.

> [!question]- Docker Swarm
> Docker's own native clustering tool — an alternative container orchestrator to Kubernetes.

> [!question]- Ad Hoc (kubectl)
> Imperative one-off commands (`kubectl run`, `kubectl create deployment ...`) as opposed to declarative YAML + `apply`.

> [!question]- Minikube
> Runs a single-node Kubernetes cluster locally for development/learning; `minikube start` / `minikube service <name>`.

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> Kubernetes::Open-source container orchestrator that keeps your declared desired state true via a continuous reconciliation loop.
> kubectl::The CLI you use to talk to the Kubernetes API server (`apply`, `get`, `describe`, `scale`, `logs`, `delete`).
> Pod::Smallest deployable unit — one or more containers sharing network + storage; ephemeral.
> Deployment::Declares replicas + a Pod template; manages ReplicaSets, rollouts, and rollbacks for stateless apps.
> ReplicaSet::Ensures exactly N matching Pods are running; created and owned by a Deployment.
> Self-healing::K8s automatically replaces failed/deleted Pods to match the desired replica count.
> Service::Stable IP + DNS name that load-balances across Pods selected by label.
> ClusterIP::Default Service type — reachable only from inside the cluster.
> NodePort::Service type exposing a port (30000–32767) on every node for external access.
> LoadBalancer::Service type that provisions a cloud external load balancer with a public IP.
> Ingress::HTTP(S) edge router mapping host/path to backend Services; needs an Ingress controller.
> Namespace::Virtual cluster for scoping/isolating resources (e.g. `dev`); names unique within it.
> ConfigMap::Non-secret key/value config injected as env vars or files.
> Secret::Sensitive key/value data stored base64-encoded (encoding, not encryption).
> ServiceAccount::Identity for apps/Pods (not humans) used by RBAC.
> Role::Namespaced set of allowed verbs on resources.
> RoleBinding::Grants a Role to a subject such as a ServiceAccount.
> RBAC::Role-Based Access Control — ServiceAccount + Role + RoleBinding define who can do what.
> Job::Runs a task to completion once, then stops.
> CronJob::Runs a Job on a cron schedule (e.g. `* * * * *` = every minute).
> DaemonSet::Runs exactly one Pod per node — ideal for node-level agents.
> PersistentVolumeClaim::A Pod's request for durable storage (size + access mode) that survives restarts.
> kubectl apply -f::Declarative command that creates or updates resources to match a manifest.
> Desired state::The end state you declare in YAML; the control plane works to make reality match it.
> Master components::API Server, etcd, Scheduler, Controller Manager — cluster-wide decisions, run on any machine.
> Node components::kubelet, kube-proxy, container runtime — run on every node, execute the workloads.
> Docker Swarm::Docker's own native clustering tool — an alternative container orchestrator to Kubernetes.
> Ad Hoc (kubectl)::Imperative one-off commands (kubectl run, kubectl create deployment) vs declarative YAML + apply.
> Minikube::Runs a single-node Kubernetes cluster locally for development/learning.

## ⚠️ Gotchas

> [!warning] Traps that catch everyone
> - **Secrets are base64, NOT encrypted.** `echo -n 'pass' | base64` → `cGFzcw==`, and `base64 -d` reverses it instantly. Treat encoded ≠ safe.
> - **NodePort range is 30000–32767.** Pick outside it and the API rejects the Service.
> - **Everything is namespace-scoped.** Forgot `-n dev`? Your resources look "missing." Namespaces don't share names.
> - **Deleting a Deployment ≠ deleting a Pod.** Delete a *Pod* → it respawns. Delete the *Deployment* → all its Pods vanish for good.
> - **Ingress needs a controller.** The Ingress object alone does nothing until an Ingress controller (nginx, Traefik, etc.) is installed.
> - **Service selector must match Pod labels.** `selector: app=demo` only finds Pods labeled `app: demo`. A typo = a Service with zero endpoints and silent 503s.
> - **`kubectl apply` is declarative, not additive.** It reconciles the cluster to the file. Remove a field from the file and re-apply → that field is removed from the cluster too.
> - **Pods are cattle, not pets.** Never rely on a specific Pod name or its ephemeral disk surviving — use Deployments + PVCs.

---

## 🏆 Boss challenge

> [!example] Ship a real app end-to-end
> 1. Create namespace `dev`.
> 2. Deploy an app via a **Deployment** (start with the `nginx` image, `replicas: 3`).
> 3. Feed it a **ConfigMap** (`ENV: dev`, `LOG_LEVEL: debug`) and a **Secret** (`password`), and confirm both exist with `kubectl get configmap,secret -n dev`.
> 4. Expose it with a **Service** (try `NodePort` so you can actually curl it from your host).
> 5. **Reach it:** `curl` the NodePort (or `minikube service` / port-forward) and get the nginx welcome page.
> 6. **Scale** to 5 replicas, delete one Pod, and confirm it self-heals back to 5.
> 7. **Bonus:** add the ServiceAccount + Role + RoleBinding and prove access with `kubectl auth can-i get pods --as=system:serviceaccount:dev:app-sa -n dev` returning `yes`.
>
> **Cleared when:** the app is reachable, config + secret are wired, and it heals + scales on demand.

---

> [!success] Class 05 cleared when you can…
> …explain desired-state reconciliation, name every core object and how it wires into **Deployment → ReplicaSet → Pods ← Service ← Ingress**, run the full `kubectl` workflow, watch a Pod self-heal, and deploy the entire stack from `full-k8s-deploy.yaml` — all from memory.
>
> 🎖️ **Badge: Pod Whisperer**

---

> [!cheatsheet]- 🔒 Command cheat-sheet — try the drills FIRST (expanding = peeking 👀)
> ⚠️ **Wait!** Expanding this is basically giving up. Did you really attempt the drill from memory? If not, collapse me and try again — the struggle is what makes it stick. Still stuck? Fine, the commands are below. 👇
>
> ```bash
> # ── Encode a value for a Secret (base64, NOT encryption!) ──────────
> echo -n 'mypassword' | base64          # -n avoids a trailing newline
>
> # ── Apply manifests (declarative: create OR update to match file) ──
> kubectl apply -f namespace.yaml        # create the 'dev' namespace first
> kubectl apply -f deployment.yaml       # roll out the app-deployment
> kubectl apply -f full-k8s-deploy.yaml  # apply the ENTIRE stack at once
>
> # ── Inspect ────────────────────────────────────────────────────────
> kubectl get pods -n dev                # list Pods in the dev namespace
> kubectl get all -n dev                 # Pods, Deployments, Services, etc.
> kubectl get pvc -n dev                 # persistent volume claims
> kubectl get configmap,secret -n dev    # config + secrets
> kubectl describe pod <pod-name> -n dev # deep detail + events (debug gold)
> kubectl logs <pod-name> -n dev         # container stdout/stderr
>
> # ── Scale (change desired replica count live) ─────────────────────
> kubectl scale deployment app-deployment --replicas=5 -n dev
>
> # ── Self-healing: delete a Pod, watch it come back ────────────────
> kubectl delete pod <pod-name> -n dev   # ReplicaSet recreates it instantly
>
> # ── RBAC check: can this ServiceAccount do the action? ────────────
> kubectl auth can-i get pods \
>   --as=system:serviceaccount:dev:app-sa -n dev   # -> "yes" or "no"
> ```
>
> > [!tip] `describe` is your debugger
> > When a Pod is stuck (`Pending`, `CrashLoopBackOff`, `ImagePullBackOff`), run `kubectl describe pod <name> -n dev` and read the **Events** at the bottom — they almost always tell you exactly what's wrong.
>
> ---

## 🔗 Related

- [[Class 06 - Helm]] — package and template all this YAML into reusable charts.
- [[Class 08 - GitOps and CI-CD]] — apply manifests automatically from Git.
- [[Terminology]] — full glossary.
- [[DevOps Experts — MOC]] — course map.
