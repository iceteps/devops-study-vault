---
tags: [devops, ansible, configuration-management, class-11]
aliases: [Ansible, Configuration Management, Playbooks, Ad-Hoc Commands, Idempotency]
class: 11
difficulty: intermediate
---

# 📜 Class 11 — Ansible

> [!info]- 🔢 Which class is this, really?
> The number in this note's title follows the **course repo folder** `class11/` — **not** the calendar session. The numbering is genuinely chaotic — **three systems disagree**: the repo folders, the teacher's drive folders, and his deck titles (which reflect the real calendar). Case in point: the Git material lives in repo folder `class3/` and drive folder `class 3`, but its own deck title says **Class 6** — the actual session count. Session 1 was the DevOps intro (see [[DevOps Foundations]]). Bottom line: navigate by **topic** via [[DevOps Experts — MOC]], never by number.


> [!abstract] TL;DR
> **Ansible** is an **agentless** [[Terminology#Idempotency|idempotent]] configuration-management tool. You describe the *desired state* of many machines in YAML, and Ansible connects over **plain SSH** (no agent to install!) to make reality match your description.
> - **[[Terminology#Inventory|Inventory]]** (`/etc/ansible/hosts`) = the list of machines, grouped (e.g. `[servers]`).
> - **Ad-hoc commands** = one-off tasks: `ansible servers -m ping`.
> - **[[Terminology#Playbook|Playbooks]]** = YAML files of ordered [[Terminology#Task|tasks]] using [[Terminology#Module|modules]] (`apt`, `user`, `copy`, `template`, `debug`).
> - **[[Terminology#Handler|Handlers]]** run *only when notified*, once, at the end of the play.
> - **[[Terminology#Role|Roles]]** = reusable, structured folders (`tasks/`, `handlers/`, `templates/`, `files/`).
> - **[[Terminology#Template (Jinja2)|Jinja2]]** templates (`.j2`) render config files per-host with `{{ variables }}`.
> - **Idempotency** = run it 100 times, the box ends up the same; only *drift* gets fixed. 🟢 `ok` vs 🟡 `changed`.

## 🎯 Learning goals

- [ ] Explain why Ansible being **agentless over SSH** is a big deal.
- [ ] Build an [[Terminology#Inventory|inventory]] and group hosts under `[servers]`.
- [ ] Fire **ad-hoc commands** with `-m` (module) and `-a` (args).
- [ ] Write a [[Terminology#Playbook|playbook]] with tasks, variables, `--tags`, `when`, and `register`.
- [ ] Wire a [[Terminology#Handler|handler]] that fires on `notify`.
- [ ] Refactor a playbook into a [[Terminology#Role|role]] (`roles/common/...`).
- [ ] Render a per-host config with a [[Terminology#Template (Jinja2)|Jinja2]] template.
- [ ] Predict `ok` vs `changed` and articulate **[[Terminology#Idempotency|idempotency]]**.

## 🧩 The big idea

Imagine you're a head chef 👨‍🍳 with **50 identical kitchens**. You don't fly to each kitchen and cook by hand. You write **one recipe** (a playbook) and hand it to every kitchen at once over the phone lines (SSH). Each line reads *"ensure the oven is at 200°C"* — not *"turn the dial three clicks."* If the oven is already at 200°C, nobody touches it. That "ensure the end state, don't blindly repeat steps" property is **[[Terminology#Idempotency|idempotency]]**, and it's the whole soul of Ansible.

> [!tip] Agentless = the killer feature
> Tools like Puppet/Chef need an **agent daemon** running on every node. Ansible needs **nothing** on the managed nodes except SSH + Python (already there on any Linux box). The "control node" pushes changes *out*. Less to install, less to break, less to secure. 🎉

**Declarative, not imperative:** you say *what* the world should look like (`nginx state=present`), not *how* to get there. Ansible figures out the diff and only does what's missing.

## 🧠 Core concepts

The hierarchy flows: **[[Terminology#Inventory|Inventory]] → [[Terminology#Playbook|Playbook]] → [[Terminology#Task|Tasks]] → [[Terminology#Module|Modules]]**, and playbooks can be packaged into **[[Terminology#Role|Roles]]**.

- **Control node** — the machine you run `ansible`/`ansible-playbook` from. It holds your playbooks and SSH keys. Nothing special is needed on the *managed* nodes.
- **[[Terminology#Inventory|Inventory]]** — the address book at `/etc/ansible/hosts`. Hosts get grouped in brackets. Our demo:
  ```ini
  [servers]
  node1
  node2
  ```
  Groups can also carry variables (see the lab inventory `[managed_nodes:vars]` with `ansible_user=root`).
- **Ad-hoc command** — a single [[Terminology#Module|module]] run against a group, no file needed. `ansible <group> -m <module> -a "<args>"`.
- **[[Terminology#Playbook|Playbook]]** — a YAML file mapping a **host group** to an ordered list of [[Terminology#Task|tasks]]. Runs top to bottom.
- **[[Terminology#Task|Task]]** — one named unit of work that calls exactly one **module**. `name:` is the human label you see in output.
- **[[Terminology#Module|Module]]** — the actual doer: `apt` (packages), `user` (accounts), `copy` (files), `template` (Jinja2), `lineinfile`, `debug` (print), `shell`/`command` (raw). Most core modules are **idempotent**; `shell`/`command` are **not**.
- **Variables** — defined inline (`vars:`), in a `vars_files:` include, or in `group_vars/`. Referenced with `{{ var }}`. You can also `register:` a task's output into a variable and branch on it with `when:`.
- **Tags** — labels on tasks so you can run a subset: `--tags=tag1`.
- **[[Terminology#Handler|Handler]]** — a special task that runs **only if notified** by a changed task, **once**, at the **end** of the play. Classic use: "restart nginx *only if* the config actually changed."
- **[[Terminology#Role|Role]]** — a standardized folder layout that Ansible auto-loads: `roles/<name>/tasks/main.yml`, `handlers/main.yml`, `templates/`, `files/`, `vars/`, `defaults/`. Makes config reusable and shareable (Ansible Galaxy).
- **[[Terminology#Template (Jinja2)|Jinja2 template]]** (`.j2`) — a text file with `{{ placeholders }}` rendered per-host by the `template` module, so each machine gets its own customized config.
- **[[Terminology#Idempotency|Idempotency]]** — re-running converges to the same state. Output tells the story: `ok=` (already correct) vs `changed=` (had to fix drift).
- **`become: true`** — privilege escalation (sudo). Installing packages/adding users needs root; set it per-play or per-task. The demo playbooks rely on it — without it, `apt` fails with permission errors.
- **Facts** — per-host system info Ansible gathers automatically at play start (`gather_facts: true`): `ansible_facts['os_family']`, IPs, memory. Branch on them with `when: ansible_facts['os_family'] == "Debian"` — that's how one playbook serves mixed fleets.

## 🛠️ Guided walkthrough — dockerized `ansible-demo` mini-lab

A tiny lab with one **control** container and two managed nodes (`node1`, `node2`), all over SSH.

> [!example] Step-by-step
> 1. **Clone & boot the lab** (on your host):
>    ```bash
>    git clone https://github.com/avielb/ansible-demo
>    cd ansible-demo
>    docker compose up -d          # brings up control + node1 + node2
>    ```
> 2. **Exec into the control node** and grab the playbooks:
>    ```bash
>    docker exec -it ansible-demo-ansible-1 bash
>    git clone https://github.com/avielb/ansible-demo
>    cd ansible-demo
>    ```
> 3. **Set up passwordless SSH** to the nodes (password is `screencast`):
>    ```bash
>    ssh-keygen
>    ssh-copy-id node1
>    ssh-copy-id node2
>    ```
> 4. **Install the inventory:**
>    ```bash
>    mkdir /etc/ansible
>    cp hosts /etc/ansible/hosts
>    ```
> 5. **Ping the fleet** — the "hello world" of Ansible:
>    ```bash
>    ansible servers -m ping
>    ```
>    Expected:
>    ```
>    node1 | SUCCESS => { "ping": "pong" }
>    node2 | SUCCESS => { "ping": "pong" }
>    ```
> 6. **Run the demo playbook** (adds a user, registers `ls -l` output, conditionally adds another user, fires a handler):
>    ```bash
>    ansible-playbook demo.yml
>    ```
>    Expected first run — note `changed` on the user task and the handler firing at the end:
>    ```
>    TASK [Add User avielb] ****************  changed: [node1]  changed: [node2]
>    TASK [Register statement] ************  changed: [node1]  changed: [node2]
>    RUNNING HANDLER [Handler_No1] ********  ok: [node1] => { "msg": "Message from Handler Number 1" }
>    PLAY RECAP ***  node1: ok=4 changed=2 ...   node2: ok=4 changed=2 ...
>    ```
> 7. **Run it AGAIN** and watch idempotency:
>    ```bash
>    ansible-playbook demo.yml
>    ```
>    Now the user already exists → `ok`, not `changed`. Because nothing *changed*, the **handler does NOT fire this time**. That's the whole point. 🟢

> [!tip] Class 14 has a cleaner lab
> This demo uses hostname-based nodes and manual `ssh-copy-id`. The [[Class 14 - Ansible Lab]] uses a tidier dockerized inventory (`hosts.ini` with IPs `172.20.0.11/12`, a mounted `id_rsa`, and `StrictHostKeyChecking=no`) — go there for a smoother re-run.

## 🔬 Drills (earn XP)

> 🗡️ **Warm up in [Shell Quest](https://github.com/iceteps/shell-quest):** mission 13 "Agentless Army 📜" — ping, playbook, the changed=0 idempotency proof, and a handler firing on drift.

- [ ] **(10 XP)** Run `ansible servers -m ping` and get `pong` from both nodes. **Done when:** both report `SUCCESS`.
- [ ] **(15 XP)** Install `vim` ad-hoc via the `apt` module, then run the *same* command again. **Done when:** first run is `changed`, second is `ok` — you've *seen* idempotency.
- [ ] **(15 XP)** Run `ansible-playbook vars.yml --tags=tag1` then `--tags=tag2`. **Done when:** `tag1` prints the vars/file vars and `tag2` loops over `listVar` (`Item4..Item1`).
- [ ] **(20 XP)** Run `demo.yml`, then `ssh node1 userdel avielb`, then run `demo.yml` again. **Done when:** the re-run shows `changed` for `Add User avielb` **and** the handler fires again (drift was healed).
- [ ] **(20 XP)** Run `ansible-playbook common.yml` (the role). **Done when:** nginx + vim are present on both nodes and `aviel.txt` is copied to `/var/tmp`.
- [ ] **(25 XP)** Run `ansible-playbook template.yml`, then `ssh node1 cat /root/hostname.conf`. **Done when:** each node's file contains *its own* hostname (proof Jinja2 rendered per-host).
- [ ] **(25 XP)** Add a `when:` guard to a task using a `register`ed result (mirror `demo.yml`'s `when: result.rc == 1`). **Done when:** the task skips or runs based on the registered return code.

## 🧗 Extra credit — beyond class *(addition)*

> [!example] 🆕 These drills are an **addition** — not covered in the class materials
> They're the next skill up from what class taught, chosen because you'll meet them in real work (and in the [[SkyWatch Capstone]]). Higher XP, higher payoff.

- [ ] **ansible-vault (30 XP)** — class stored vars in plain YAML. Encrypt a secret: `ansible-vault create secret.yml`, reference it in a playbook, run with `--ask-vault-pass`. **Done when:** the file is unreadable in git but usable in the play.
- [ ] **Dry-run everything (15 XP)** — re-run a playbook with `--check` (and `--diff`). **Done when:** you can say what check mode can and cannot predict (hint: shell tasks).
- [ ] **Loops + conditionals (25 XP)** — write a task that creates 3 users with `loop:`, but only `when:` the OS family is Debian (`ansible_facts`). **Done when:** the same playbook is a no-op on the second run (idempotent!).

## 🔎 Learn to fish — find it yourself (don't just copy)

> [!tip] Beat "monkey-see-monkey-do"
> The real skill isn't memorizing commands — it's finding the right one **fast**. Pros do this all day. Build the reflex:
> 1. **Ask the tool first:** `<tool> --help`, `<tool> <subcommand> --help`, `man <tool>`.
> 2. **Official docs = source of truth** (below) — not random blogs or old Stack Overflow answers.
> 3. **Search smart:** `<what you want> site:<official-docs-domain>`; add the tool's version if behaviour changed between releases.
> 4. **Read the WHOLE error message** — it almost always names the missing flag or the fix.
> 5. `tldr <command>` gives real-world examples (install a `tldr` client — it's the friendly `man`).

**📚 Docs & references**
- [Ansible docs](https://docs.ansible.com/ansible/latest/)
- [Module index (all modules)](https://docs.ansible.com/ansible/latest/collections/index_module.html)
- [Playbook guide](https://docs.ansible.com/ansible/latest/playbook_guide/index.html)

**⚡ Built-in help — try these BEFORE searching**
```bash
ansible-doc -l           # list every module
ansible-doc apt          # full docs + examples for a module, offline
ansible --help / ansible-playbook --help
```

> [!example] Power move for this class
> `ansible-doc <module>` is gold: `ansible-doc copy` shows every parameter AND copy-paste examples, right in your terminal. Use it instead of guessing YAML keys.

## 🧪 Self-check quiz

> [!question]- Why is "agentless" such a selling point for Ansible?
> No daemon/agent needs to be installed, running, or upgraded on managed nodes. Ansible connects over standard **SSH** (+ Python, already present). Fewer moving parts, smaller attack surface, nothing to maintain on the fleet.

> [!question]- What's the difference between an ad-hoc command and a playbook?
> An **ad-hoc command** runs a single module once from the CLI (`ansible servers -m ping`). A **playbook** is a reusable YAML file of many ordered tasks (`ansible-playbook demo.yml`). Use ad-hoc for quick checks, playbooks for repeatable configuration.

> [!question]- When exactly does a handler run?
> Only when a task **notifies** it *and that task reported `changed`*. It runs **once**, **at the very end** of the play (after all tasks), even if notified many times. If nothing changed, the handler is silent.

> [!question]- In the `PLAY RECAP`, what's the difference between `ok` and `changed`?
> `ok` = the task ran and the system was **already** in the desired state (nothing done). `changed` = Ansible **had to modify** something to reach the desired state. A clean re-run of an idempotent playbook shows `changed=0`.

> [!question]- Why are `shell` / `command` modules the enemy of idempotency?
> They run raw commands blindly every time and can't tell if the work was already done, so they always report `changed`. Prefer purpose-built modules (`apt`, `user`, `copy`, `lineinfile`) that check current state first. If you must use `shell`, guard it with `creates:`/`when:`.

> [!question]- How does a Jinja2 template give each host a different config?
> The `template` module renders the `.j2` file **per host**, substituting `{{ variables }}` (like the host's own `ansible_hostname`) at run time. So `template.yml` writes a `hostname.conf` containing node1's name on node1 and node2's on node2.

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- Ansible
> An agentless, idempotent configuration-management tool that manages nodes over SSH using declarative YAML.

> [!question]- Agentless
> Managed nodes need no installed agent/daemon — Ansible connects over plain SSH + Python.

> [!question]- Inventory
> The list of managed hosts (default `/etc/ansible/hosts`), organized into groups like `[servers]`.

> [!question]- Ad-hoc command
> A one-off single-module run from the CLI, e.g. `ansible servers -m ping`.

> [!question]- Playbook
> A YAML file mapping host groups to an ordered list of tasks.

> [!question]- Task
> One named unit of work in a playbook that invokes exactly one module.

> [!question]- Module
> The component that performs an action (e.g. `apt`, `user`, `copy`, `template`, `debug`).

> [!question]- Handler
> A task that runs only when notified by a changed task, once, at the end of the play.

> [!question]- Role
> A standardized folder layout (`tasks/`, `handlers/`, `templates/`, `files/`) for reusable configuration.

> [!question]- Idempotency
> Running the same playbook repeatedly converges to the same state; only drift is corrected.

> [!question]- Tags
> Labels on tasks so you can run a subset via `--tags=`.

> [!question]- Jinja2 template (`.j2`)
> A text file with `{{ placeholders }}` rendered per host by the `template` module.

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> Ansible::An agentless, idempotent configuration-management tool that manages nodes over SSH using declarative YAML.
> Agentless::Managed nodes need no installed agent/daemon — Ansible connects over plain SSH + Python.
> Inventory::The list of managed hosts (default `/etc/ansible/hosts`), organized into groups like `[servers]`.
> Ad-hoc command::A one-off single-module run from the CLI, e.g. `ansible servers -m ping`.
> Playbook::A YAML file mapping host groups to an ordered list of tasks.
> Task::One named unit of work in a playbook that invokes exactly one module.
> Module::The component that performs an action (e.g. `apt`, `user`, `copy`, `template`, `debug`).
> Handler::A task that runs only when notified by a changed task, once, at the end of the play.
> Role::A standardized folder layout (`tasks/`, `handlers/`, `templates/`, `files/`) for reusable configuration.
> Idempotency::Running the same playbook repeatedly converges to the same state; only drift is corrected.
> Tags::Labels on tasks so you can run a subset via `--tags=`.
> Jinja2 template (`.j2`)::A text file with `{{ placeholders }}` rendered per host by the `template` module.

## ⚠️ Gotchas

> [!warning] Set up SSH keys FIRST
> Ansible has no magic transport — if `ssh node1` doesn't work, `ansible` won't either. Run `ssh-keygen` + `ssh-copy-id node1/node2` (demo password: `screencast`) before anything else. On the Class 14 lab a private key is mounted and `StrictHostKeyChecking=no` sidesteps the host-key prompt.

> [!warning] Handlers are conditional AND deferred
> A handler fires only if a notifying task reported `changed`, and only at the **end** of the play. First run: changed → handler fires. Idempotent second run: `ok` → **handler stays silent**. This is a feature (don't restart nginx if config didn't change), not a bug.

> [!warning] YAML indentation will bite you
> Ansible is YAML — **spaces only, never tabs**, and indentation defines structure. A misaligned `notify:` or `tasks:` block silently changes meaning or errors out. Keep 2-space indents consistent.

> [!warning] `shell`/`command` break idempotency
> These run raw shell every time and always report `changed`. They can't detect prior work. Reach for a real module (`apt`, `user`, `lineinfile`, `copy`) whenever one exists; guard unavoidable shell with `creates:` or `when:`.

> [!warning] Group name mismatch = no hosts
> Your playbook's `hosts: servers` must match a group that actually exists in the inventory. If the inventory group is `[managed_nodes]` but the play says `hosts: servers`, Ansible matches **zero** hosts and does nothing.

## 🏆 Boss challenge

Build a **role** named `webserver` that installs and configures nginx from a Jinja2 template, restarting only on change.

> [!example] Target structure & files
> ```
> roles/webserver/
> ├── tasks/main.yml
> ├── handlers/main.yml
> └── templates/index.html.j2
> ```
> `roles/webserver/tasks/main.yml`:
> ```yaml
> ---
> - name: Install nginx
>   apt: name=nginx state=present
>
> - name: Deploy homepage from template
>   template:
>     src: index.html.j2
>     dest: /var/www/html/index.html
>   notify: Restart nginx          # only fires if the file actually changes
>
> - name: Ensure nginx is running
>   service:
>     name: nginx
>     state: started
>     enabled: yes
> ```
> `roles/webserver/handlers/main.yml`:
> ```yaml
> ---
> - name: Restart nginx
>   service:
>     name: nginx
>     state: restarted
> ```
> `roles/webserver/templates/index.html.j2`:
> ```jinja2
> <h1>Hello from {{ ansible_hostname }}</h1>
> ```
> Site playbook `site.yml`:
> ```yaml
> ---
> - hosts: servers
>   roles:
>     - webserver
> ```
> **Win condition:** `ansible-playbook site.yml` shows `changed` + handler on the first run; a second run is all `ok` with the handler silent; and `curl node1` / `curl node2` each return that node's own hostname in the `<h1>`. 🥇

> [!success] Class 11 cleared when you can…
> …explain agentless SSH, build an inventory, fire ad-hoc commands, write a playbook with variables/tags/`register`/`when`, wire a notify→handler, package it into a role, render a per-host Jinja2 template — and correctly predict `ok` vs `changed` on a re-run because you *get* [[Terminology#Idempotency|idempotency]].
>
> 🎖️ **Badge: Playbook Pro**

> [!cheatsheet]- 🔒 Command cheat-sheet — try the drills FIRST (expanding = peeking 👀)
> ⚠️ **Wait!** Expanding this is basically giving up. Did you really attempt the drill from memory? If not, collapse me and try again — the struggle is what makes it stick. Still stuck? Fine, the commands are below. 👇
>
> ```bash
> # ── One-time SSH key setup (control node → managed nodes) ──
> ssh-keygen                       # generate a key pair on the control node
> ssh-copy-id node1                # push public key to node1 (password: "screencast")
> ssh-copy-id node2                # ...and node2. Now Ansible logs in key-only.
>
> # ── Inventory ──
> mkdir /etc/ansible
> cp hosts /etc/ansible/hosts      # install our [servers] inventory
> cat /etc/ansible/hosts           # confirm node1 / node2 are grouped
>
> # ── Ad-hoc commands (single module, no playbook) ──
> ansible servers -m ping                                # connectivity test -> "pong"
> ansible servers -a "echo Hello World!"                 # -a with no -m = 'command' module
> ansible servers -m apt -a "name=nginx state=present"   # install nginx idempotently
>
> # ── Playbooks ──
> ansible-playbook vars.yml --tags=tag1     # run only tasks tagged tag1 (vars from file)
> ansible-playbook vars.yml --tags=tag2     # run only the with_items loop task
> ansible-playbook demo.yml                 # handlers + register + when + user module
> ansible-playbook common.yml               # runs the 'common' ROLE
> ansible-playbook template.yml             # render a Jinja2 template per host
>
> # ── Verify the template landed on each node ──
> ssh node1 cat /root/hostname.conf
> ssh node2 cat /root/hostname.conf
>
> # ── Roles from Ansible Galaxy (community roles) ──
> ansible-galaxy install geerlingguy.java
> ansible-playbook -i hosts -u root galaxy-role.yml
>
> # ── Secrets with Ansible Vault ──
> ansible-vault create secrets.yml          # encrypted var file
> ansible-playbook secret-play.yml --ask-vault-pass
> ```
>
> > [!tip] `-m` vs `-a`
> > `-m` picks the **module**; `-a` passes its **arguments** as `key=value` pairs. With no `-m`, Ansible defaults to the `command` module — that's why `ansible servers -a "echo ..."` just works.

## 🔗 Related

- [[Class 14 - Ansible Lab]] — cleaner dockerized inventory & re-runnable lab
- [[Class 12 - Terraform]] — declarative *provisioning* (Ansible configures what Terraform creates)
- [[Terminology]] — glossary of all terms above
- [[DevOps Experts — MOC]] — course map
