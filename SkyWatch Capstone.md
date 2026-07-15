---
tags: [devops, capstone, skywatch, project]
aliases: [SkyWatch, Final Boss, Capstone Project, Weather Pipeline]
difficulty: advanced
---

# 🛰️ SkyWatch — The Capstone (Final Boss)

> [!abstract] TL;DR
> **SkyWatch** is a distributed weather-query pipeline: a user types a city into a **Flask** frontend, the request is published to **RabbitMQ**, a **Python worker** consumes it and calls the free **Open-Meteo API**, and the answer comes back — any city, no API key.
> It's the **graduation project** because it *stitches every class together into one running system*: **Terraform** builds the AWS cluster, **Ansible** installs K3s, **Docker + GitHub Actions** ship the images, **Helm + ArgoCD** deploy via GitOps, **RabbitMQ** decouples the pipeline, and **Prometheus + Grafana** watch it all. If you can build SkyWatch from an empty AWS account to a weather answer in the browser — and then `terraform destroy` it — you've proven the whole course.

---

## 🗺️ The whole picture

**Data flow (request → answer):**

```
Browser
  │  submits a city name
  ▼
Flask Frontend  (port 5000, NodePort 30080)
  │  publishes a job
  ▼
RabbitMQ  (AMQP 5672 · Mgmt 15672 · Prometheus 15692)
  │  worker consumes the job
  ▼
Python Worker  (×2 replicas)
  │  calls
  ▼
Open-Meteo API  (geocoding + forecast)  →  result flows back to the user
```

*(How does the answer travel **back**? RPC over the queue: the frontend publishes each request
with a `reply_to` queue + `correlation_id`, the worker publishes the forecast to that reply
queue, and the waiting frontend matches it by id. Same broker, both directions.)*

**Node layout (3× t3.micro, eu-west-1):**

| Node | Hostname | Runs |
|---|---|---|
| Master | `skywatch-master` | K3s control plane only (tainted `NoSchedule`) |
| Worker 1 | `skywatch-worker` | ArgoCD (7 pods) |
| Worker 2 | `skywatch-worker2` | SkyWatch app + RabbitMQ + Prometheus + Grafana |

**Which class teaches which layer** — this is the map from the syllabus to the project:

| Layer | Tech | Job in SkyWatch | Class note |
|---|---|---|---|
| Infrastructure | Terraform | VPC, security group, 3× EC2, Ansible inventory | [[Class 12 - Terraform]] |
| Configuration | Ansible | Install K3s, kubeconfig, swap, kubectl alias | [[Class 11 - Ansible]] |
| Kubernetes | K3s v1.29.5 | Lightweight cluster that runs everything | [[Class 05 - Kubernetes]] |
| Containerisation | Docker → GHCR | Multi-stage images: frontend + worker | [[Class 02 - Docker Networking and Images]] |
| CI/CD | GitHub Actions | Lint → Build → Push → tag-bump | [[Class 08 - GitOps and CI-CD]] |
| Packaging | Helm 3 | `skywatch` chart (frontend, worker, RabbitMQ) | [[Class 06 - Helm]] |
| GitOps deploy | ArgoCD | Auto-sync from Git, prune + self-heal | [[Class 08 - GitOps and CI-CD]] |
| Messaging | RabbitMQ 3.13 | Decouples frontend from workers | [[Class 13 - RabbitMQ Messaging]] |
| Observability | kube-prometheus-stack | Prometheus scrape + Grafana dashboards | see Part 4 below |

> [!tip] Read the whole architecture as a sentence
> *"Terraform builds the ground, Ansible wires the cluster, Docker packs the app, GitHub Actions ships it, Helm shapes it, ArgoCD deploys it, RabbitMQ decouples it, Prometheus watches it."* Memorise that sentence and you've memorised the project. See [[Terminology]] for any word you trip on.

---

## 🧱 Part 1 — Infrastructure (Terraform + Ansible)

**What to build:** a single `main.tf` declaring the default VPC, one security group (SSH, K3s API 6443, NodePort range, intra-cluster self-rule), and one EC2 per role. The Ubuntu 22.04 AMI is **looked up dynamically** (no hard-coded AMI ID). After `terraform apply`, the Ansible inventory is rendered from a template. Then two **idempotent** Ansible roles (`k3s_master`, `k3s_worker`) install and join K3s. → [[Class 12 - Terraform]], [[Class 11 - Ansible]]

> [!warning] The five gotchas that eat a whole session
> - **Private IP for `K3S_URL`.** In AWS, VPC→public-IP traffic egresses via the Internet Gateway and *bypasses the SG self-rule* — so port 6443 looks closed to the worker. Join using the master's **private IP**.
> - **`--tls-san <public_ip>`.** The K3s server cert only includes the private IP by default; add the public IP as a SAN at install time or `kubectl` from your laptop throws a TLS error.
> - **`K3S_TOKEN` must be an env var**, not `--token`. The installer pipe reads `K3S_TOKEN=... sh -s - agent` from the environment; passed as a CLI flag it's silently dropped and never written to the agent's env file.
> - **Pin K3s to `v1.29.5+k3s1`.** v1.35's API-server + etcd footprint OOMs on 1 GB RAM. Pinning is not optional on t3.micro.
> - **Idempotent uninstall check.** Guard the uninstall task with `systemctl is-active k3s` (or `k3s-agent`) *first* — otherwise re-running the playbook fires the uninstall script (which exists on a healthy node) and wipes a live cluster.

> [!example] The self-rule trap, concretely
> Your SG allows `6443` from *itself*. You test with the master's **public** IP → connection refused. You swap to the **private** IP → instant join. Same port, same rule; the only difference is which network path the packet takes. This is the single most confusing bug in the whole project.

---

## 🐳 Part 2 — Containerisation & CI/CD

**What to build:** two Dockerfiles on `python:3.11-slim` — one for the Flask **frontend**, one for the Python **worker** — pushed to **GHCR** (`ghcr.io/<owner>/skywatch-frontend`, `-worker`). RabbitMQ uses the public `rabbitmq:3.13-management` image (no auth). → [[Class 02 - Docker Networking and Images]]

**The GitHub Actions pipeline** (`.github/workflows/ci.yml`, on push to `main` + PRs): → [[Class 08 - GitOps and CI-CD]]

| Step | What it does |
|---|---|
| 1. Lint | `flake8` on `frontend/app.py` + `worker/worker.py` (max-line 120) |
| 2–3. Build & push | `docker/build-push-action` → `:<sha7>` **and** `:latest` for both images |
| 4. Update values | **mikefarah/yq** bumps `frontend.image.tag` + `worker.image.tag` in `values.yaml` |
| 5. Commit bump | `git commit` with `[skip ci]` to avoid an infinite trigger loop |

Each image is tagged with the **7-char commit SHA** (`${GITHUB_SHA::7}`) so every deploy is traceable to a commit. ArgoCD sees the `values.yaml` change and rolls out in ~3 min.

> [!warning] The `yq` gotcha (two different tools, same name)
> `pip install yq` installs a **Python wrapper** with *incompatible syntax* — `yq -i` in-place editing won't behave. Download the **mikefarah/yq** Go binary directly via `wget` in CI. This bites everyone once.

---

## ⎈ Part 3 — GitOps deploy (Helm + ArgoCD)

**Chart layout** (`helm/skywatch/`, no external dependencies): → [[Class 06 - Helm]], [[Class 05 - Kubernetes]]

```
templates/
├── frontend-deployment.yaml
├── frontend-service.yaml    # NodePort 30080
├── worker-deployment.yaml
├── rabbitmq-secret.yaml     # RABBITMQ_DEFAULT_* + rabbitmq-{username,password}
└── rabbitmq.yaml            # StatefulSet (1 replica) + ClusterIP Service
```

**RabbitMQ — the Bitnami removal story** → [[Class 13 - RabbitMQ Messaging]]
The original design vendored the `bitnami/rabbitmq` sub-chart. Mid-project, **Bitnami removed all their images from Docker Hub** — old and new tags 404, and the new OCI registry needs auth for anonymous pulls. Fix: **replace the sub-chart with inline templates** on the official `rabbitmq:3.13-management` image. Key decisions:

- **Single Secret** `skywatch-rabbitmq` holds *both* the `RABBITMQ_DEFAULT_*` keys (consumed by the RabbitMQ container via `envFrom`) and the `rabbitmq-{username,password}` keys (referenced by frontend/worker via `secretKeyRef`).
- **TCP socket probes on 5672**, not `rabbitmq-diagnostics ping` — the exec probe times out under memory pressure (the Erlang VM slows when the memory-watermark alarm fires); a TCP check is instant and load-independent.
- **`ignoreDifferences` for `volumeClaimTemplates`** — the API server injects default fields (`storageClass`, `volumeMode`) into the VolumeClaimTemplate, causing perpetual **OutOfSync**.

**ArgoCD config** (v2.11.2, namespace `argocd`, NodePort 30081):

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: StatefulSet
      name: skywatch-rabbitmq
      jsonPointers: [ /spec/volumeClaimTemplates ]
  syncPolicy:
    automated:   { prune: true, selfHeal: true }
    syncOptions: [ CreateNamespace=true, ServerSideApply=true ]
```

All 7 ArgoCD pods are pinned to `skywatch-worker` via `nodeSelector`; the app lands on `skywatch-worker2`; the master is tainted so nothing schedules there. 2 GB swap on master + worker2 keeps 913 MB nodes alive.

> [!tip] `prune: true` + `selfHeal: true` = real GitOps
> Delete a pod by hand and self-heal recreates it. Delete a manifest from Git and prune removes it from the cluster. **Git is the single source of truth** — you stop `kubectl apply`-ing and start `git push`-ing.

---

## 📊 Part 4 — Observability (Prometheus + Grafana)

**Install `kube-prometheus-stack`** — but its large CRDs **time out** on the t3.micro API server via `helm install`. The two-step fix:

```bash
# 1. Apply CRDs server-side, one at a time (avoids timeouts)
kubectl apply --server-side --request-timeout=3m -f <each crd>.yaml   # sleep 10 between

# 2. Install the chart, skipping the CRDs you just applied
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace --skip-crds -f helm/monitoring/values.yaml
```

**Slim values for t3.micro** — `alertmanager.enabled: false` (~100 MB saved), Prometheus `retention: 24h` + `limits {cpu 500m, mem 512Mi}`, Grafana `NodePort 30030` / admin `skywatch-grafana` / `limits {cpu 200m, mem 128Mi}`, both pinned to `skywatch-worker2`.

**RabbitMQ ServiceMonitor** — the `3.13-management` image auto-starts a Prometheus endpoint on **port 15692** (the `rabbitmq_prometheus` plugin is on by default). A ServiceMonitor tells Prometheus to scrape it:

```yaml
kind: ServiceMonitor
spec:
  selector: { matchLabels: { app.kubernetes.io/name: skywatch-rabbitmq } }
  endpoints: [ { port: prometheus, interval: 30s } ]
```

> [!warning] Server-side first, then `--skip-crds`
> If you skip step 1, Helm bundles the CRDs into the release and the install hangs on the tiny API server. If you skip `--skip-crds` in step 2, you get a CRD ownership conflict. You need **both halves** of the trick.

---

## 🧭 Build roadmap (drills, earn XP)

The real homework. Do these **in order** — each part depends on the one before. Tick a box only when its **Done when** is true.

### 🧱 Part 1 — Infrastructure (Terraform + Ansible)
- [ ] **+10 XP** Write `main.tf`: default VPC, SG, 3× t3.micro, dynamic Ubuntu 22.04 AMI. **Done when:** `terraform apply` returns 3 running instances.
- [ ] **+10 XP** Render the Ansible inventory from a Terraform template. **Done when:** `inventory.ini` lists master + 2 workers with correct IPs.
- [ ] **+15 XP** `k3s_master` role: pin `v1.29.5+k3s1`, add `--tls-san <public_ip>`, disable Traefik, expose the join token as a fact. **Done when:** `kubectl get nodes` works *from your laptop*.
- [ ] **+15 XP** `k3s_worker` role: join via `K3S_URL` (**private IP**) + `K3S_TOKEN` **env var**. **Done when:** `kubectl get nodes` shows all 3 `Ready`.
- [ ] **+10 XP** Guard uninstall with `systemctl is-active`. **Done when:** re-running the playbook on a live cluster changes nothing (fully idempotent).

### 🐳 Part 2 — Containerise & CI/CD
- [ ] **+10 XP** Dockerfiles for frontend + worker on `python:3.11-slim`. **Done when:** both build and run locally.
- [ ] **+15 XP** `ci.yml`: flake8 → build → push `:sha7` + `:latest` to GHCR. **Done when:** a push produces two tagged images in GHCR.
- [ ] **+15 XP** Add the mikefarah/yq tag-bump step with `[skip ci]`. **Done when:** CI commits a `values.yaml` update *without* re-triggering itself.

### ⎈ Part 3 — GitOps deploy
- [ ] **+10 XP** Author the `skywatch` Helm chart (frontend, worker, RabbitMQ inline). **Done when:** `helm template` renders with no errors.
- [ ] **+15 XP** Install ArgoCD, pin 7 pods to worker1, create the Application. **Done when:** ArgoCD shows the app **Synced + Healthy**.
- [ ] **+15 XP** Add `ignoreDifferences` for `volumeClaimTemplates` + TCP probes on 5672. **Done when:** the app stays **Synced** (no perpetual OutOfSync).
- [ ] **+10 XP** Taint master `NoSchedule`, add 2 GB swap to worker2. **Done when:** app pods only land on worker2 and nothing OOMs.

### 📊 Part 4 — Observability
- [ ] **+15 XP** Apply CRDs `--server-side`, then `helm install --skip-crds`. **Done when:** all monitoring pods are `Running`.
- [ ] **+10 XP** Create the RabbitMQ ServiceMonitor on port 15692. **Done when:** RabbitMQ appears as an **UP** target in Prometheus.
- [ ] **+10 XP** Open Grafana, confirm dashboards render. **Done when:** you see live cluster + RabbitMQ metrics.

> [!success] Capstone cleared when you can…
> …run the [[#🏆 Final boss — the end-to-end challenge|end-to-end challenge]] top to bottom — `terraform apply` → `ansible-playbook` → ArgoCD sync → weather in the browser → metrics in Grafana → `terraform destroy` — **from memory, on a fresh AWS account, in under an hour, with no cluster wipes.** (Command-by-command runbook: the course repo's `assignment.md` → *Session Quick-Start*.)
>
> 🎖️ **Badge: SkyWatch Commander 🛰️**

---

## 🧪 Self-check quiz

> [!question]- Why must the worker join the master using the **private** IP, not the public one?
> In AWS, traffic sent to an instance's **public** IP egresses through the **Internet Gateway**, which bypasses the security-group **self-rule** — so port 6443 appears closed. Intra-VPC traffic must use **private** IPs to stay inside the SG.

> [!question]- What does `--tls-san <public_ip>` fix, and why at install time?
> It adds the public IP as a **Subject Alternative Name** in the K3s server's TLS cert. Without it, `kubectl` from a laptop (hitting the public IP) fails cert validation. The cert is generated **at install**, so the SAN must be present then — you can't add it after the fact without regenerating.

> [!question]- Why is `K3S_TOKEN` passed as an environment variable instead of `--token`?
> The K3s install pipe (`... sh -s - agent`) reads `K3S_TOKEN` from the **environment**. Passed as a `--token` CLI flag it's silently dropped and never written to the agent's env file, so the join fails.

> [!question]- Why did `ansible-playbook --limit worker` break the cluster?
> `--limit worker` **skips the master play**, so the `k3s_token` fact is never set — the worker play has no token to join with. Always run the **full** playbook so the master fact is gathered first.

> [!question]- Why did ArgoCD report the RabbitMQ StatefulSet as perpetually OutOfSync?
> The Kubernetes API server injects **default fields** (`storageClass`, `volumeMode`) into `volumeClaimTemplates` that aren't in the manifest. Live ≠ desired forever → OutOfSync. Fix: an `ignoreDifferences` stanza for `/spec/volumeClaimTemplates`.

> [!question]- Why TCP socket probes instead of `rabbitmq-diagnostics ping`?
> Under memory pressure the Erlang VM slows (the memory-watermark alarm fires), so the **exec** probe times out and Kubernetes kills a healthy pod. A **TCP check on 5672** is instant and unaffected by VM load.

> [!question]- Why apply Prometheus CRDs `--server-side` before `helm install --skip-crds`?
> The `kube-prometheus-stack` CRDs are huge and **time out** on the tiny t3.micro API server when bundled into a Helm release. Applying them **server-side** first (one at a time) succeeds; `--skip-crds` then stops Helm from re-installing them and causing an ownership conflict.

---

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- K3s pin on t3.micro
> Pin `v1.29.5+k3s1` — v1.35's API-server + etcd footprint OOMs on 1 GB RAM.

> [!question]- K3S_URL IP choice
> Use the master's **private** IP; public IPs egress via the IGW and bypass the SG self-rule (6443 looks closed).

> [!question]- K3S_TOKEN delivery
> Must be an **environment variable** on the install pipe; the installer ignores it as a `--token` CLI flag.

> [!question]- Bitnami RabbitMQ
> Bitnami removed all Docker Hub images (404s); replace the sub-chart with inline templates on `rabbitmq:3.13-management`.

> [!question]- ArgoCD OutOfSync (StatefulSet)
> API server adds default fields to `volumeClaimTemplates`; fix with `ignoreDifferences` in the Application.

> [!question]- Non-idempotent uninstall
> Uninstall tasks with a `removes:` guard still fire on a healthy node; add a `systemctl is-active` check first.

> [!question]- Prometheus CRD timeout
> Apply CRDs with `kubectl apply --server-side` first, then `helm install --skip-crds`.

> [!question]- RabbitMQ Prometheus port
> **15692** — the `rabbitmq_prometheus` plugin is enabled by default in the `3.13-management` image.

> [!question]- RabbitMQ probe type
> **TCP socket on 5672**, not exec ping — exec times out when the Erlang memory-watermark alarm slows the VM.

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> K3s pin on t3.micro::Pin `v1.29.5+k3s1` — v1.35's API-server + etcd footprint OOMs on 1 GB RAM.
> K3S_URL IP choice::Use the master's **private** IP; public IPs egress via the IGW and bypass the SG self-rule (6443 looks closed).
> K3S_TOKEN delivery::Must be an **environment variable** on the install pipe; the installer ignores it as a `--token` CLI flag.
> Bitnami RabbitMQ::Bitnami removed all Docker Hub images (404s); replace the sub-chart with inline templates on `rabbitmq:3.13-management`.
> ArgoCD OutOfSync (StatefulSet)::API server adds default fields to `volumeClaimTemplates`; fix with `ignoreDifferences` in the Application.
> Non-idempotent uninstall::Uninstall tasks with a `removes:` guard still fire on a healthy node; add a `systemctl is-active` check first.
> Prometheus CRD timeout::Apply CRDs with `kubectl apply --server-side` first, then `helm install --skip-crds`.
> RabbitMQ Prometheus port::**15692** — the `rabbitmq_prometheus` plugin is enabled by default in the `3.13-management` image.
> RabbitMQ probe type::**TCP socket on 5672**, not exec ping — exec times out when the Erlang memory-watermark alarm slows the VM.

## 🧠 Lessons learned

> [!quote] Lesson 1 — Pin K3s on constrained nodes
> K3s v1.35 OOMed on t3.micro. **Always pin K3s to a known-good version on resource-constrained nodes** — `v1.29.5+k3s1` is stable on 1 GB.

> [!quote] Lesson 2 — Intra-VPC traffic uses private IPs
> The worker couldn't join via the master's public IP. **In AWS, intra-VPC traffic must use private IPs** — public IPs egress via the IGW and bypass SG self-rules.

> [!quote] Lesson 3 — `--tls-san` needs the public IP
> `kubectl` from a laptop threw a TLS cert error. **`--tls-san` must include the public IP at install time**, not just the hostname.

> [!quote] Lesson 4 — K3s reads the token from the environment
> The token wasn't stored in the agent env file. **The K3s installer reads `K3S_TOKEN` from the environment**, not from a `--token` CLI flag.

> [!quote] Lesson 5 — Don't `--limit` past the master
> `ansible-playbook --limit worker` broke the cluster. **`--limit` skips the master play so the `k3s_token` fact is never set** — always run the full playbook.

> [!quote] Lesson 6 — Vendor registries can vanish
> Bitnami removed its images from Docker Hub. **Use official Docker Hub images or mirror to your own registry** — a vendor registry can disappear mid-project.

> [!quote] Lesson 7 — Two tools named `yq`
> `pip install yq` ≠ mikefarah yq. **Two incompatible tools share the same name** — always download the mikefarah binary explicitly in CI.

> [!quote] Lesson 8 — Teach ArgoCD to ignore server defaults
> ArgoCD showed perpetual OutOfSync for the StatefulSet. **The API server adds default fields to `volumeClaimTemplates`** — use `ignoreDifferences` in the ArgoCD Application.

> [!quote] Lesson 9 — "removes:" is not idempotent
> Re-running Ansible wiped a live cluster. **Shell tasks with a `removes:` guard aren't idempotent when the uninstall script exists on a healthy node** — add an active-service check first.

> [!quote] Lesson 10 — Big CRDs need server-side apply
> Prometheus CRD install timed out via Helm. **Apply large CRDs with `kubectl apply --server-side` first, then `helm install --skip-crds`.**

---

## 🏆 Final boss — the end-to-end challenge

Do this on a **fresh AWS account**, timed, no notes:

1. **Provision** — `cd terraform && terraform apply -auto-approve` (3 nodes, ~2 min).
2. **Configure** — `cd ../ansible && ansible-playbook -i inventory.ini playbook.yml` (K3s + join, ~3 min). Point `kubectl` at the new `kubeconfig`.
3. **Deploy via GitOps** — install ArgoCD (pin to worker1), `kubectl apply -f argocd/application.yaml`, taint the master. ArgoCD auto-syncs SkyWatch from Git.
4. **See weather in the browser** — open `http://<worker2-ip>:30080`, type a city, get a forecast. The Flask → RabbitMQ → worker → Open-Meteo pipeline is live.
5. **Watch metrics in Grafana** — open `http://<worker2-ip>:30030` (`admin` / `skywatch-grafana`), confirm cluster + RabbitMQ dashboards. RabbitMQ is an **UP** target in Prometheus.
6. **Tear down** — `cd terraform && terraform destroy -auto-approve` to stay in the free tier.

> [!success] You beat the final boss when…
> …all six steps pass on the first try, with **no private-IP mistakes, no token errors, no cluster wipes, no OutOfSync**. That's the whole course in one run.
>
> 🎖️ **Badge: SkyWatch Commander 🛰️**

---

## 🔗 Related

- [[Class 12 - Terraform]] — infrastructure as code (VPC, SG, EC2)
- [[Class 11 - Ansible]] — config management (K3s install, idempotency)
- [[Class 05 - Kubernetes]] — the K3s cluster everything runs on
- [[Class 02 - Docker Networking and Images]] — multi-stage images → GHCR
- [[Class 08 - GitOps and CI-CD]] — GitHub Actions + ArgoCD
- [[Class 06 - Helm]] — the `skywatch` chart
- [[Class 13 - RabbitMQ Messaging]] — the message queue that decouples the pipeline
- [[Terminology]] — every term you trip on
- [[DevOps Experts — MOC]] — course map / table of contents
