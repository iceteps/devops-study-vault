---
tags: [devops, glossary, reference, moc]
aliases: [Glossary, DevOps Terms, Terminology]
---

# 📖 Terminology — DevOps Experts master glossary

The hub for the whole course. Every class note links back here. Skim it before class,
search it during class. Related: [[DevOps Experts — MOC]] · [[DevOps Foundations]].

> [!tip] How to use in Obsidian
> Other notes link to a term like `[[Terminology#Container]]`. Click any term below to
> jump; use `Ctrl+O` to hop between notes and the graph view to see how it all connects.

---

## 🐳 Docker & containers

### Docker
A platform that packages an app plus everything it needs into a **container** so it runs
the same on any machine. Solves "works on my machine". See [[Class 01 - Docker Basics]].

### Image
A read-only **template** for a container — app code + dependencies + OS libs, frozen. Built
from a [[Terminology#Dockerfile]]. You *run* an image to get a container. Think: class (image) vs object (container).

### Container
A running, isolated instance of an [[Terminology#Image]]. Lightweight (shares the host
kernel), disposable, and starts in milliseconds. Deleting one loses anything not in a [[Terminology#Volume]].

### Dockerfile
A text recipe listing the steps to build an [[Terminology#Image]] (`FROM`, `WORKDIR`, `COPY`,
`RUN`, `EXPOSE`, `CMD`). Each instruction becomes a [[Terminology#Layer]].

### Layer
One cached step of an image build. Docker reuses unchanged layers, so ordering matters:
copy `requirements.txt` and install deps *before* copying app code, so code edits don't bust the dependency cache.

### Registry
A store for images. **Docker Hub** (public), **GHCR** (GitHub Container Registry), and cloud
registries. `docker push`/`docker pull` move images to/from a registry.

### Volume
Persistent storage that outlives a container. Use it for data you can't lose (databases).
A **bind mount** maps a host folder into the container instead.

### Docker network
A virtual network letting containers talk to each other. On a **user-defined** network,
containers reach each other **by name** (e.g. `ping rabbitmq`). See [[Class 02 - Docker Networking and Images]].

### Hypervisor
The software/firmware layer that **virtual machines** run on top of; it carves the host's
RAM/CPU into full VMs. Containers skip it — they share the host **kernel**, which is why
they're lighter and faster to start than VMs.

### Nginx
A wildly popular open-source **web server / reverse proxy / load balancer / HTTP cache**.
The course's favorite demo container (`nginx:alpine`) and what Ansible deploys in [[Class 14 - Ansible Lab]].

### Port mapping
`-p 8080:5000` exposes container port 5000 on host port 8080 so your browser can reach it.
Left = host, right = container.

### Docker Compose
Defines a multi-container app in one `docker-compose.yml` and runs it with `docker compose up`.
Handles networks, dependencies, and healthchecks for you.

---

## 🌿 Git & version control

### Git
A distributed **version control** system: tracks every change, lets you branch, and sync with others. See [[Class 03 - Git]].

### Repository
A project tracked by Git — your files plus the full `.git` history. Local + optional [[Terminology#Remote]] copy.

### Commit
A saved snapshot of staged changes with a message. The unit of history. `git commit -m "msg"`.

### Staging area
The "on-deck" list of changes that will go into the next commit. `git add <file>` stages;
`git status` shows what's staged vs not.

### Branch
A movable pointer to a line of work. `git branch feature` creates, `git checkout feature`
(or `git switch`) moves onto it. Lets you work without touching `main`.

### Merge
Combines one branch's changes into another. Conflicts happen when the same lines changed in both.

### Remote
A hosted copy of the repo (e.g. GitHub) named `origin`. `git push` sends commits up, `git pull` brings them down.

### Clone
`git clone <url>` — download a full copy of a remote repo, history and all.

### Monolith
One large codebase/app where everything is deployed together. Simple at first, hard to scale/change independently.

### Monorepo
**One repository** holding many projects/services (still deployable separately). Easier shared
tooling and atomic cross-project changes. (Note from your notes: *mono ≠ monolith* — a monorepo can hold microservices.)

### Multi-repo
The opposite of a [[Terminology#Monorepo]]: **one repo per project/service**. Wins team autonomy
and independent releases; costs cross-repo coordination. (Yariv's deck compares them head-to-head.)

### SHA (commit hash)
The unique fingerprint identifying every commit (e.g. `a1b2c3d`). Git commands accept it to
point at any node in history (`git show <sha>`, `git checkout <sha>`).

### HEAD
Git's pointer to **where you are right now** — normally the tip of your current branch.
"Detached HEAD" = you checked out a commit directly instead of a branch.

---

## ☸️ Kubernetes (K8s)

### Kubernetes
An orchestrator that runs, scales, heals, and networks containers across many machines. See [[Class 05 - Kubernetes]].

### Pod
The smallest deployable unit — one (or a few tightly-coupled) containers sharing network/storage. Pods are disposable.

### Deployment
Declares a desired state ("run 3 replicas of this image") and keeps it true — self-heals, rolls out updates, scales.

### ReplicaSet
The controller a Deployment uses under the hood to keep N identical Pods running.

### Service
A stable network address + load balancer for a set of Pods (which come and go). Types below.

### NodePort
A [[Terminology#Service]] type that opens a port on every node (30000–32767) so you can reach the app from outside the cluster.

### Namespace
A virtual cluster-within-a-cluster for isolating resources (e.g. `dev`, `monitoring`).

### ConfigMap
Non-secret configuration (env vars, files) injected into Pods, separate from the image.

### Secret
Like a ConfigMap but for sensitive values (base64-encoded, e.g. passwords). Referenced via `secretKeyRef`.

### Ingress
HTTP(S) routing rules mapping hostnames/paths to Services — one entry point for many apps.

### RBAC
Role-Based Access Control — who can do what. A **Role** grants permissions; a **RoleBinding**
attaches it to a user/[[Terminology#ServiceAccount]].

### ServiceAccount
An identity for Pods/processes (not humans) to authenticate to the K8s API.

### StatefulSet
Like a Deployment but for stateful apps needing stable names + storage (databases, RabbitMQ).

### DaemonSet
Runs exactly one Pod **per node** — used for agents (logging, monitoring).

### Job
Runs a Pod to completion once (batch task). **CronJob** runs a Job on a schedule.

### PersistentVolumeClaim
A Pod's request for durable storage (a [[Terminology#Volume]] that survives Pod restarts).

### Node
A worker machine (VM/physical) in the cluster that runs Pods.

### kubectl
The K8s CLI. `kubectl apply -f x.yaml`, `get`, `describe`, `logs`, `scale`, `delete`.

### Taint
A mark on a Node that repels Pods unless they tolerate it (e.g. keep app pods off the control-plane node).

### K3s
A lightweight, single-binary Kubernetes distribution — great for small/edge nodes (used in the [[SkyWatch Capstone]]).

---

## ⎈ Helm

### Helm
The "package manager for Kubernetes" — bundles many manifests into one installable **chart**. See [[Class 06 - Helm]].

### Chart
A Helm package: templated manifests + metadata (`Chart.yaml`) + defaults (`values.yaml`).

### Values
`values.yaml` — the knobs (image tag, replicas, resources) injected into templates. Override per environment.

### Release
One installed instance of a chart in a cluster (`helm install <release> ./chart`).

### Template
A manifest with `{{ .Values.x }}` placeholders that Helm renders into real YAML. `helm template` previews it.

---

## 📜 Ansible

### Ansible
Agentless **configuration management** — describe a server's desired state in YAML, apply over SSH. See [[Class 11 - Ansible]].

### Playbook
A YAML file of plays/tasks describing what to configure on which hosts.

### Role
A reusable, structured bundle of tasks/handlers/templates/vars (e.g. an `nginx` role).

### Inventory
The list of hosts (and groups) Ansible manages — `hosts.ini` / `inventory.ini`.

### Task
A single action calling a [[Terminology#Module]] (install a package, copy a file).

### Handler
A task that only runs when **notified** by a change (e.g. "restart nginx" after its config changed).

### Module
A reusable unit Ansible calls to do work (`apt`, `copy`, `template`, `service`, `ping`).

### Idempotency
Running the same playbook twice yields the same result — it only changes what's not already correct. Core Ansible principle.

### Template (Jinja2)
A `.j2` file with `{{ variables }}` that Ansible renders per host (e.g. a custom `nginx.conf`).

---

## 🏗️ Terraform & IaC

### Infrastructure as Code
Managing servers/networks/cloud resources with version-controlled code instead of clicking consoles.

### Terraform
Declarative IaC tool: you describe the desired cloud infra; it figures out the changes to reach it. See [[Class 12 - Terraform]].

### Provider
A plugin for a platform (AWS, Azure, GCP) that Terraform uses to create resources.

### Resource
One infrastructure object declared in `.tf` (an EC2 instance, a security group, a VPC).

### State
Terraform's record (`terraform.tfstate`) of what it has created, mapping code to real resources. Often stored remotely in a [[Terminology#Backend]].

### Plan
`terraform plan` — a dry run showing what will change before you apply.

### Apply
`terraform apply` — actually creates/updates/destroys resources to match the code.

### Backend
Where state is stored (local file, or shared **S3 bucket** for teams). See [[Class 12 - Terraform]].

---

## 🔁 CI/CD & GitOps

### CI/CD
**Continuous Integration** (auto build+test on every push) + **Continuous Delivery/Deployment** (auto ship). See [[Class 08 - GitOps and CI-CD]].

### Pipeline
The automated sequence of steps (lint → build → push → deploy) triggered by a commit.

### GitHub Actions
GitHub's built-in CI/CD. Workflows live in `.github/workflows/*.yml`, triggered `on: push`.

### Jenkins
A veteran self-hosted CI/CD server, configured via pipelines/Jenkinsfiles. (From your notes.)

### Artifact
A build output (a Docker image, a zip) produced by CI and consumed later in the pipeline.

### GitOps
Git is the single source of truth for infra/deploys. You change Git; an agent syncs the cluster to match.

### ArgoCD
A GitOps controller for Kubernetes: watches a repo and auto-applies changes to the cluster. See [[Class 08 - GitOps and CI-CD]].

### Sync
ArgoCD applying the Git-declared state to the cluster. Can be manual or `automated`.

### Drift
When the live cluster diverges from Git (someone `kubectl`'d by hand). ArgoCD flags it **OutOfSync**.

### Self-heal
ArgoCD auto-reverting manual drift back to what Git says.

---

## 📨 Messaging

### Message queue
A buffer that decouples producers from consumers — senders drop messages, workers pick them up when ready. See [[Class 13 - RabbitMQ Messaging]].

### RabbitMQ
A popular message broker implementing [[Terminology#AMQP]]. Ships a management UI on port 15672.

### Producer
The app that **sends** messages to a queue.

### Consumer
The app/worker that **receives** and processes messages from a queue.

### Exchange
RabbitMQ's router — receives messages from producers and routes them to queues by rules.

### Routing key
A label on a message the [[Terminology#Exchange]] uses to decide which queue(s) it goes to.

### AMQP
Advanced Message Queuing Protocol — the wire protocol RabbitMQ speaks (port 5672).

---

## 📊 Observability

### Observability
Being able to understand a system's internal state from its outputs: metrics, logs, traces.

### Prometheus
A metrics database that **scrapes** `/metrics` endpoints and stores time-series data. See [[SkyWatch Capstone]].

### Grafana
Dashboards and graphs on top of Prometheus (and other) data sources.

### ServiceMonitor
A K8s custom resource telling Prometheus which Service to scrape and how often.

### Metrics
Numeric measurements over time (request count, queue depth, CPU) that power dashboards and alerts.

---

## 🚦 Deployment strategies (from your notes)

### Rolling update
Replace old Pods with new ones gradually — the default K8s update. Zero downtime if healthchecks are right.

### Blue-green deployment
Run two identical environments (blue=old, green=new); switch all traffic at once. Instant rollback by switching back.

### Canary deployment
Release to a small % of users first, watch metrics, then ramp up. Limits blast radius of a bad release.

---

## 🌀 Agile & ways of working (from your notes)

### Agile
Iterative delivery in small increments with fast feedback, over big up-front plans.

### Scrum
An Agile framework: fixed-length **sprints**, daily standups, roles (PO, Scrum Master, team).

### Sprint
A short (1–2 week) timebox in which the team commits to and completes a set of work.

### Jira
Atlassian's issue/project tracker for planning and tracking Agile work (tickets, boards, sprints).

---

## ☁️ Cloud & AWS

### AWS
Amazon Web Services — the cloud provider used in the capstone.

### EC2
Elastic Compute Cloud — rentable virtual servers. The capstone uses `t3.micro` (free-tier).

### VPC
Virtual Private Cloud — your isolated network in AWS where resources live.

### Security group
A virtual firewall on AWS resources — rules for allowed inbound/outbound traffic (SSH, ports).

### YAML
The indentation-based config format used everywhere here (K8s, Ansible, Compose, CI). Whitespace matters; no tabs.

---

> [!info] Missing a term?
> Add it here as a `### Term` heading and link to it from anywhere with `[[Terminology#Term]]`.
> This file is meant to grow with you across the whole course.
