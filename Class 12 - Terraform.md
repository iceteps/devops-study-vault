---
tags: [devops, terraform, iac, aws, class-12]
aliases: [Terraform, IaC, Infrastructure as Code, Class 12, Remote State]
class: 12
difficulty: intermediate
---

# 🏗️ Class 12 — Terraform

> [!info]- 🔢 Which class is this, really?
> The number in this note's title follows the **course repo folder** `class12/` — **not** the calendar session. The real session numbering is shifted: calendar-session 1 was the DevOps intro (see [[DevOps Foundations]]), so e.g. Docker basics was *taught* in session 2 even though its repo folder is `class1`. When in doubt, navigate by **topic** via [[DevOps Experts — MOC]], not by number.


> [!abstract] TL;DR
> [[Terminology#Terraform|Terraform]] is [[Terminology#Infrastructure as Code|Infrastructure as Code]]: you **declare the desired end-state** of your cloud (in `.tf` files written in HCL), and Terraform figures out the exact API calls to reach it. It talks to a cloud through a [[Terminology#Provider|provider]] (here: `aws`), tracks reality in a **[[Terminology#State|state]] file** (a ledger of what it built), and moves you there with the loop `init → plan → apply → destroy`. Split files by concern (`provider / networking / instances / variables / outputs`) and keep state **remote in an [[Terminology#Backend|S3 backend]]** so the whole team shares one source of truth. Golden rule: **`plan` before `apply`, `destroy` when done, and NEVER commit `tfstate` / `tfvars` / `*.pem`.**

## 🎯 Learning goals
- [ ] Explain **declarative desired-state** vs. writing imperative scripts.
- [ ] Understand what the **state file** is and why it exists.
- [ ] Write a [[Terminology#Provider|provider]], a [[Terminology#Resource|resource]], a [[Terminology#Variable|variable]], and an **output**.
- [ ] Run the full lifecycle: `init → plan → apply → output → destroy`.
- [ ] Configure a **remote S3 backend** and know why it beats local state.
- [ ] Split config across `.tf` files **by concern** and know why.
- [ ] Provision a real **[[Terminology#EC2|EC2]] instance + [[Terminology#Security group|security group]]** in a [[Terminology#VPC|VPC]], read its IP, then tear it all down.

## 🧩 The big idea

Imagine you're renovating a house. You **don't** phone the foreman every morning barking one instruction at a time ("lay a brick... now another..."). Instead you hand him a **blueprint** of the finished house. He walks the site, compares blueprint to reality, and does **only the missing work** — the *diff*. Change the blueprint later ("add a window"), hand it back, and he cuts exactly one window. Nothing else moves.

- 📐 **The blueprint** = your `.tf` files (desired state, declarative).
- 👷 **The foreman** = Terraform's engine (computes the diff, calls the cloud APIs).
- 📒 **The foreman's ledger** = the **[[Terminology#State|state file]]** (`terraform.tfstate`) — his written record of *exactly what he has already built and its real IDs*. Without the ledger he'd have to re-survey the whole house every time, and might rebuild things that already exist.

That's the whole mental model: **you describe the *what*, Terraform owns the *how*, and state is its memory.**

> [!tip] Declarative, not imperative
> A bash script says *"run these steps in this order."* Terraform says *"here's the world I want — make it so."* Run `apply` five times on an unchanged config and nothing happens after the first: it's **idempotent** because Terraform compares desired-state to the ledger and sees zero diff.

## 🧠 Core concepts

**[[Terminology#Terraform|Terraform]]** — a tool by HashiCorp that provisions infrastructure from declarative config written in **HCL** (HashiCorp Configuration Language).

**[[Terminology#Infrastructure as Code|Infrastructure as Code (IaC)]]** — managing servers, networks, and services through version-controlled text files instead of clicking in a web console. Reviewable, repeatable, diff-able, deletable.

**[[Terminology#Provider|Provider]]** — the plugin that teaches Terraform how to talk to a specific platform's API. Ours is `aws`. It's the *only* line that says "we're building on [[Terminology#AWS|AWS]]":
```hcl
# provider.tf
provider "aws" {
  region = var.aws_region   # note: value comes from a variable, not hardcoded
}
```

**[[Terminology#Resource|Resource]]** — one real thing Terraform manages. Syntax: `resource "<TYPE>" "<LOCAL_NAME>" { ... }`. You reference one resource from another by `TYPE.LOCAL_NAME.ATTRIBUTE`, and Terraform uses those references to build a dependency graph and order the work:
```hcl
# networking.tf
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id   # <-- implicit dependency: subnet waits for the VPC
  cidr_block = "10.0.1.0/24"
}
```

**Data source** — read-only lookup of something Terraform did *not* create. In `instances.tf` we ask AWS for the newest Amazon Linux 2023 AMI instead of hardcoding a fragile image id:
```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023*-kernel-6.1-x86_64"]
  }
}
```

**[[Terminology#Variable|Variable]]** — a named input, so configs are reusable and not full of magic strings:
```hcl
# variables.tf
variable "aws_region" {
  default = "eu-west-1"
}
```

**Output** — a value Terraform prints (and stores) after `apply` — the useful facts you actually need, like the server's IP:
```hcl
# outputs.tf
output "public_ip" {
  value = aws_instance.web.public_ip
}
```

**[[Terminology#State|State]]** — `terraform.tfstate`, the JSON ledger mapping your config to the real resources (IDs, IPs, attributes). It's how Terraform knows the diff. **It can contain secrets** → never commit it.

**[[Terminology#Backend|Backend]]** — *where* state lives. Default is a local file; for teams you push it to a **remote S3 backend** so everyone shares one ledger (and it can be **locked** to stop two people applying at once):
```hcl
# backend.tf  (commented out until your bucket exists)
# terraform {
#   backend "s3" {
#     bucket = "terraform-state-student-yfreifeld"
#     key    = "dev/terraform.tfstate"
#     region = "eu-west-1"
#   }
# }
```

### Why split the `.tf` files by concern?
Terraform reads **all** `*.tf` in a folder and merges them — order and filenames don't matter to the engine, they matter to **humans**. Splitting by concern keeps each file small and reviewable:

| File | Owns |
|---|---|
| `provider.tf` | which cloud + region ([[Terminology#Provider|provider]] config) |
| `networking.tf` | [[Terminology#VPC|VPC]], subnet, gateway, routes, [[Terminology#Security group|security group]] |
| `instances.tf` | the [[Terminology#EC2|EC2]] server, its key pair, the AMI lookup |
| `variables.tf` | inputs (region, sizes...) |
| `outputs.tf` | facts to print after apply (IPs...) |

## 🛠️ Guided walkthrough

> [!example] From zero to a running server and back to zero
> 1. **Configure AWS creds & verify.** Make sure the AWS CLI is authenticated, then confirm the identity:
>    ```bash
>    aws sts get-caller-identity
>    ```
>    Expected shape — an ARN and account number:
>    ```json
>    { "UserId": "AIDA...", "Account": "123456789012", "Arn": "arn:aws:iam::123456789012:user/you" }
>    ```
> 2. **Create the remote-state bucket** (one time, name must be globally unique — swap in your name):
>    ```bash
>    aws s3api create-bucket --bucket devops-course-aias --region eu-west-1 \
>      --create-bucket-configuration LocationConstraint=eu-west-1
>    ```
>    Then uncomment the `backend "s3"` block in `backend.tf` and point `bucket` at the name you just made.
> 3. **`terraform init`** — pulls the `aws` provider (and `tls`/`local` used by the key pair) and connects the backend. Expected: `Terraform has been successfully initialized!`
> 4. **`terraform plan`** — READ THE DIFF. It should say something like `Plan: 12 to add, 0 to change, 0 to destroy.` Every line prefixed with `+` is a resource that *will be created*. Nothing is built yet — this is the safe preview.
> 5. **`terraform apply`** — type `yes` at the prompt. Terraform builds the [[Terminology#VPC|VPC]] → subnet → gateway/routes → [[Terminology#Security group|SG]] → key pair → [[Terminology#EC2|EC2]], in dependency order. Ends with `Apply complete! Resources: 12 added...` and prints outputs.
> 6. **Inspect outputs:**
>    ```bash
>    terraform output
>    # public_ip = "34.240.xx.xx"
>    ```
> 7. **`terraform destroy`** — type `yes`. Frees everything so you stay in the free tier: `Destroy complete! Resources: 12 destroyed.`
>
> ⚠️ **`terraform.tfstate` and `*.tfvars` are gitignored on purpose** — state and variable files can hold IPs, keys, and secrets. The generated `terraform-key.pem` (from the `tls_private_key` resource) is a **private SSH key** — treat it like a password and never commit it.

## 🔬 Drills (earn XP)

- [ ] **(10 XP)** Run `aws sts get-caller-identity` and confirm you're in the right account. **Done when:** you see your ARN + account id.
- [ ] **(15 XP)** `terraform init` the `class12/terraform` folder. **Done when:** you see `successfully initialized` and a `.terraform/` dir appears.
- [ ] **(20 XP)** Run `terraform plan` **without** applying. **Done when:** you can read the `Plan: N to add` line and name 3 resources it will create.
- [ ] **(20 XP)** Add a new variable `instance_type` (default `"t3.micro"`) in `variables.tf` and use it in `instances.tf` via `var.instance_type`. **Done when:** `plan` still succeeds with no diff to the instance type.
- [ ] **(20 XP)** Add an output `vpc_id` that returns `aws_vpc.main.id`. **Done when:** it appears in `terraform plan` under "Changes to Outputs".
- [ ] **(25 XP)** Create your S3 state bucket, uncomment `backend.tf`, re-run `terraform init` and migrate state. **Done when:** init reports the S3 backend and your `.tfstate` is now in the bucket, not local.
- [ ] **(30 XP)** Full cycle: `apply`, grab `public_ip` from `terraform output`, then `destroy`. **Done when:** the IP resolves during apply and `destroy` reports everything removed.

## 🧗 Extra credit — beyond class *(addition)*

> [!example] 🆕 These drills are an **addition** — not covered in the class materials
> They're the next skill up from what class taught, chosen because you'll meet them in real work (and in the [[SkyWatch Capstone]]). Higher XP, higher payoff.

- [ ] **Data sources (30 XP)** — the class hard-codes knowledge; pros look it up. Use `data "aws_ami"` with a name filter to fetch the latest Ubuntu 22.04 AMI dynamically instead of a fixed AMI id. **Done when:** `terraform plan` resolves the AMI without you pasting one.
- [ ] **count / for_each (30 XP)** — turn one `aws_instance` into three with `count = 3` (names via `${count.index}`), plan it, then refactor to `for_each` over a map. **Done when:** you can say when for_each beats count (hint: deleting the middle one).
- [ ] **Read the state (20 XP)** — `terraform state list`, then `terraform state show <resource>`. **Done when:** you found one attribute in state that isn't in your .tf files (proof state ≠ code).

## 🔎 Learn to fish — find it yourself (don't just copy)

> [!tip] Beat "monkey-see-monkey-do"
> The real skill isn't memorizing commands — it's finding the right one **fast**. Pros do this all day. Build the reflex:
> 1. **Ask the tool first:** `<tool> --help`, `<tool> <subcommand> --help`, `man <tool>`.
> 2. **Official docs = source of truth** (below) — not random blogs or old Stack Overflow answers.
> 3. **Search smart:** `<what you want> site:<official-docs-domain>`; add the tool's version if behaviour changed between releases.
> 4. **Read the WHOLE error message** — it almost always names the missing flag or the fix.
> 5. `tldr <command>` gives real-world examples (install a `tldr` client — it's the friendly `man`).

**📚 Docs & references**
- [Terraform docs](https://developer.hashicorp.com/terraform/docs)
- [AWS provider registry (resource docs)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform CLI reference](https://developer.hashicorp.com/terraform/cli)

**⚡ Built-in help — try these BEFORE searching**
```bash
terraform -help          # top-level commands
terraform <cmd> -help    # e.g. terraform plan -help
terraform validate       # catch config errors before applying
```

> [!example] Power move for this class
> Every resource (like `aws_instance`) has a Registry page listing every argument + a working example. Search `terraform aws_instance` → the registry result is the source of truth, not blogs.

## 🧪 Self-check quiz

> [!question]- What does "declarative desired-state" mean, and how is it different from a bash script?
> You describe the **final world you want**, not the steps to get there. Terraform diffs desired-state against the state ledger and does only what's missing — making it **idempotent** (re-running an unchanged config is a no-op). A bash script is imperative: it runs every step every time, whether or not it's already done.

> [!question]- Why does the state file exist, and why must you never commit it?
> State is Terraform's **memory / ledger** — it maps your config to real resource IDs so it can compute the diff and know what to change or delete. It's not committed because it can contain **secrets and sensitive attributes**, and a shared committed copy would cause conflicts and drift.

> [!question]- What is a provider, and which file declares it here?
> A **provider** is the plugin that speaks a platform's API. Here it's `aws`, declared in `provider.tf` with `region = var.aws_region`. It's the piece that binds the whole config to [[Terminology#AWS|AWS]].

> [!question]- Why put remote state in an S3 backend instead of a local file?
> So a **team shares one source of truth**, it's durable/backed-up, and it can be **locked** to prevent two people applying simultaneously and corrupting state. Local state lives on one laptop and doesn't scale to teams.

> [!question]- In `aws_subnet.public`, what does `vpc_id = aws_vpc.main.id` create?
> An **implicit dependency**. Referencing another resource's attribute tells Terraform to build the VPC *first*, then the subnet. This is how Terraform derives its dependency graph and ordering — you don't write the order yourself.

> [!question]- Why is `cidr_blocks = ["0.0.0.0/0"]` on the SSH ingress rule risky?
> It opens port 22 to the **entire internet** — anyone can attempt to SSH to your box. Safer is to scope it to your own IP (e.g. `["203.0.113.4/32"]`). Wide-open SSH is a classic attack surface.

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- Terraform
> HashiCorp tool that provisions infrastructure from declarative HCL config, computing the diff between your desired state and the real world.

> [!question]- Infrastructure as Code (IaC)
> Managing infra through version-controlled text files instead of manual console clicks — repeatable, reviewable, diff-able.

> [!question]- Provider
> Plugin that teaches Terraform to talk to a specific platform's API (e.g. `aws`); declared in `provider.tf`.

> [!question]- Resource
> A single real managed object, written `resource "TYPE" "NAME" {}` (e.g. `aws_instance.web`).

> [!question]- Data source
> Read-only lookup of something Terraform did NOT create (e.g. `data "aws_ami"` to find the latest AMI).

> [!question]- Variable
> A named input value (`variable "aws_region"`), referenced as `var.aws_region`.

> [!question]- Output
> A value Terraform prints/stores after apply (e.g. `output "public_ip"`).

> [!question]- State (tfstate)
> JSON ledger mapping config to real resource IDs; Terraform's memory. Never commit — may hold secrets.

> [!question]- Backend
> Where state is stored; local file by default, or **remote S3** for shared, lockable team state.

> [!question]- terraform init
> Downloads providers and wires up the backend.

> [!question]- terraform plan
> Dry-run showing the diff (+add / ~change / -destroy) — changes nothing.

> [!question]- terraform apply
> Executes the plan, builds real infra, updates state.

> [!question]- terraform destroy
> Tears down all resources tracked in state.

> [!question]- Idempotent
> Re-running an unchanged config produces no changes, because desired-state already matches the ledger.

> [!question]- Security group
> Virtual firewall on an AWS resource controlling ingress/egress by port, protocol, and CIDR.

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> Terraform::HashiCorp tool that provisions infrastructure from declarative HCL config, computing the diff between your desired state and the real world.
> Infrastructure as Code (IaC)::Managing infra through version-controlled text files instead of manual console clicks — repeatable, reviewable, diff-able.
> Provider::Plugin that teaches Terraform to talk to a specific platform's API (e.g. `aws`); declared in `provider.tf`.
> Resource::A single real managed object, written `resource "TYPE" "NAME" {}` (e.g. `aws_instance.web`).
> Data source::Read-only lookup of something Terraform did NOT create (e.g. `data "aws_ami"` to find the latest AMI).
> Variable::A named input value (`variable "aws_region"`), referenced as `var.aws_region`.
> Output::A value Terraform prints/stores after apply (e.g. `output "public_ip"`).
> State (tfstate)::JSON ledger mapping config to real resource IDs; Terraform's memory. Never commit — may hold secrets.
> Backend::Where state is stored; local file by default, or **remote S3** for shared, lockable team state.
> terraform init::Downloads providers and wires up the backend.
> terraform plan::Dry-run showing the diff (+add / ~change / -destroy) — changes nothing.
> terraform apply::Executes the plan, builds real infra, updates state.
> terraform destroy::Tears down all resources tracked in state.
> Idempotent::Re-running an unchanged config produces no changes, because desired-state already matches the ledger.
> Security group::Virtual firewall on an AWS resource controlling ingress/egress by port, protocol, and CIDR.

## ⚠️ Gotchas

> [!warning] Read this before you touch the cloud
> - 🚫 **NEVER commit `terraform.tfstate`, `*.tfvars`, or `*.pem`.** State and vars can hold secrets; the `.pem` is your private SSH key. Keep them in `.gitignore`.
> - 👀 **Always `plan` before `apply`.** The plan is your safety preview — a `-` (destroy) you didn't expect is your warning to stop.
> - 🌀 **State drift:** if someone changes infra by hand in the console, reality no longer matches state. Next `plan` will try to "fix" the drift. Change infra *through Terraform*, not clicks.
> - 🔒 **Remote state locking:** an S3 backend (classically paired with a DynamoDB lock, or S3-native locking) stops two people applying at once and corrupting the ledger.
> - 💸 **Free-tier discipline:** a running `t3.micro` + resources cost money over time. **`terraform destroy` when you're done** to stay free.
> - 🔥 **`cidr_blocks = ["0.0.0.0/0"]` on port 22** exposes SSH to the whole internet — scope it to your IP in anything real.
> - 📛 **S3 bucket names are globally unique** — `devops-course-yourname`, not a name someone already took.

## 🏆 Boss challenge

> [!example] Provision → observe → obliterate
> Using the `class12/terraform` config as your base:
> 1. Bring up the full stack with `terraform apply`: a [[Terminology#VPC|VPC]] + public subnet + internet gateway + route table, a [[Terminology#Security group|security group]] allowing SSH, and a **`t3.micro` [[Terminology#EC2|EC2]]** with an auto-generated key pair.
> 2. Run `terraform output` and capture the instance's **public IP**.
> 3. Bonus: `ssh -i terraform-key.pem ec2-user@<public_ip>` to prove it's live (AL2023 default user is `ec2-user`).
> 4. **`terraform destroy`** — verify every one of the ~12 resources is gone and your AWS bill stays flat.
>
> 🥇 You win when apply → output → destroy runs clean and you can explain each resource's role from memory.

> [!cheatsheet]- 🔒 Command cheat-sheet — try the drills FIRST (expanding = peeking 👀)
> ⚠️ **Wait!** Expanding this is basically giving up. Did you really attempt the drill from memory? If not, collapse me and try again — the struggle is what makes it stick. Still stuck? Fine, the commands are below. 👇
>
> ```bash
> # --- Prereq: confirm which AWS account/identity you're pointed at ---
> aws sts get-caller-identity          # who am I? which account? (sanity check before you build)
>
> # --- One-time: create the S3 bucket that will hold remote state ---
> aws s3api create-bucket \
>   --bucket devops-course-[NAME_OF_USER] \
>   --region eu-west-1 \
>   --create-bucket-configuration LocationConstraint=eu-west-1   # eu-west-1 needs this constraint
>
> # --- The core Terraform lifecycle (run inside the terraform/ folder) ---
> terraform init        # download the aws provider + wire up the backend
> terraform plan        # DRY RUN: show the diff (+ create / ~ change / - destroy). Change nothing.
> terraform apply       # execute the plan (prompts yes/no). Builds real infra + writes state.
> terraform output      # print the declared outputs, e.g. public_ip
> terraform destroy     # tear down everything in state (stay in free-tier!). Prompts yes/no.
>
> # handy extras
> terraform fmt         # auto-format .tf files
> terraform validate    # check syntax without touching the cloud
> terraform apply -auto-approve   # skip the yes prompt (careful in real life)
> ```

## 🔗 Related
- [[Class 11 - Ansible]] — config management (mutating existing servers) vs. Terraform's provisioning (creating infra).
- [[SkyWatch Capstone]] — where you'll provision the real project infrastructure with IaC.
- [[Terminology]] — glossary hub.
- [[DevOps Experts — MOC]] — course map.

---

> [!success] Class 12 cleared when you can…
> …explain declarative desired-state and why the state ledger exists, write a provider + resource + variable + output, run `init → plan → apply → output → destroy`, move state into an S3 backend, and stand up an EC2 + security group, print its IP, and destroy it — all without committing a single `tfstate` or `.pem`.

🎖️ **Badge: Infra Architect**
