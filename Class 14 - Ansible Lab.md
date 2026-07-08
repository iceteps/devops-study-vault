---
tags: [devops, ansible, docker, lab, class-14]
aliases: [Ansible Lab, Dockerized Ansible, Class 14, Semaphore Lab]
class: 14
difficulty: intermediate
---

# 🧪 Class 14 — Ansible Lab (Dockerized)

> [!info]- 🔢 Which class is this, really?
> The number in this note's title follows the **course repo folder** `class14/` — **not** the calendar session. The numbering is genuinely chaotic — **three systems disagree**: the repo folders, the teacher's drive folders, and his deck titles (which reflect the real calendar). Case in point: the Git material lives in repo folder `class3/` and drive folder `class 3`, but its own deck title says **Class 6** — the actual session count. Session 1 was the DevOps intro (see [[DevOps Foundations]]). Bottom line: navigate by **topic** via [[DevOps Experts — MOC]], never by number.


> [!abstract] TL;DR
> You spin up a **whole mini-datacenter on your laptop** with one `docker compose up -d`: a **control node** (`ansible-control`) that bosses around **two web nodes** (`node1`, `node2`) over SSH — all containers on one Compose network. An `entrypoint.sh` script auto-generates an SSH keypair and pushes it to the nodes, so `ansible all -m ping` just works. Then you run two real playbooks — **`install_nginx.yml`** and **`deploy_website.yml`** (the latter uses a **Jinja2** template) — and watch a live website appear at `http://localhost:8081` and `:8082`. Bonus: a **Semaphore** web UI at `:3000` gives Ansible a GUI. This is Class 11 theory made *hands-on*. 🚀

## 🎯 Learning goals

- [ ] Understand the **control node → target nodes** model over SSH
- [ ] See how **`entrypoint.sh` distributes SSH keys** so Ansible can connect password-free
- [ ] Read what **`ansible.cfg`** and **`inventory.ini`** configure and how they wire together
- [ ] Run `ansible all -m ping` and interpret **green vs red** output
- [ ] Execute `install_nginx.yml` and `deploy_website.yml` playbooks end-to-end
- [ ] Prove **[[Terminology#Idempotency|idempotency]]** by re-running a playbook (`ok` vs `changed`)
- [ ] Edit a **[[Terminology#Template (Jinja2)|Jinja2 template]]** and redeploy
- [ ] Open the deployed site in a **browser** on the mapped ports
- [ ] Know that **Semaphore** puts a web UI on top of Ansible

## 🧩 The big idea

Imagine you're a **shift manager at a datacenter** 🏢. You have one desk (the **control node**) with a phone, and two server racks (**node1**, **node2**) across the room. You don't walk over and configure each rack by hand — you *call* them over a secure line (SSH) and read them a checklist (a **playbook**). If a task is already done, you skip it; if not, you do it. That's Ansible.

The magic of this lab: that entire datacenter is **just containers** on your machine. No cloud bill, no VMs to babysit. `docker compose up -d` builds the desk, the racks, and the secure phone line — and even a **Semaphore** dashboard so you can push the button from a web page instead of the terminal.

> [!tip] Why this matters
> In the real world you'll manage 5, 50, or 5000 servers. You can't SSH into each one by hand. Ansible + an inventory scales the *same playbook* from 2 nodes to 2000 with zero code change. This lab is that pattern in miniature — and it's **agentless**: the nodes need nothing but SSH + Python, which they already have.

## 🧠 Core concepts

**Control node** — the `ansible-control` container. It holds the playbooks, the `inventory.ini`, the `ansible.cfg`, and the private SSH key. It never runs a web server itself; it *orchestrates* the others. See [[Terminology#Ansible]].

**Target / managed nodes** — `node1` and `node2` containers. Each runs an SSH server (`sshd`) and has a `ubuntu` user with passwordless sudo (baked in `Dockerfile`). They're the racks getting configured. They expose port 80 to the host as **8081** and **8082**.

**Shared SSH keys (the secret sauce)** — `entrypoint.sh` on the control node:
1. Generates a keypair at `/root/.ssh/id_rsa` (only if missing — idempotent!).
2. Loops over `node1 node2` and uses `sshpass` + `ssh-copy-id` (password `ubuntu`) to install the **public key** on each node, retrying every 2s until the node's SSH is up.
This is why `ansible all -m ping` works with **no password prompt** — the trust is pre-wired. The key lives in a shared Docker **volume** (`ssh_keys`) so **Semaphore** can read it too.

**[[Terminology#Inventory|Inventory]]** (`inventory.ini`) — the address book. A `[web]` group lists `node1` and `node2`. `[web:vars]` sets `ansible_user=ubuntu` and points at the private key. Crucially, the hostnames must match the **Compose service/hostnames** so Docker's DNS resolves them.

```ini
[web]
node1
node2

[web:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/root/.ssh/id_rsa
```

**`ansible.cfg`** — global defaults so you don't repeat flags:
- `inventory = inventory.ini` → no need for `-i` on every command.
- `host_key_checking = False` → don't choke on unknown SSH host keys (fine in a lab).
- `retry_files_enabled = False` → don't litter `.retry` files on failure.

**The two [[Terminology#Playbook|playbooks]]:**
- **`install_nginx.yml`** — targets `hosts: web`, `become: true` (sudo), one task: `apt` install nginx. Minimal, focused.
- **`deploy_website.yml`** — installs nginx *and* uses `copy` to drop an `index.html` (`<h1>Hello from Ansible</h1>`) then `service: started`. The **[[Terminology#Template (Jinja2)|Jinja2 template]]** `nginx.conf.j2` shows the templating pattern with a `{{ nginx_port }}` variable that Ansible renders per-host.

**[[Terminology#Docker Compose|Docker Compose]]** ties it all together: one `ansible-net` bridge network, `depends_on` ordering, port maps, and the shared `ssh_keys` volume.

## 🛠️ Guided walkthrough — mini-lab

Follow `class14/commands` end to end. ⏱️ ~10 minutes.

1. **Go to the lab folder** (on the host):
   ```bash
   cd ansible_lab_files
   ```
2. **Launch the datacenter:**
   ```bash
   docker compose up -d
   ```
   First run builds the images (Ubuntu 22.04 + Ansible / sshd). Give it a minute. The control node's entrypoint prints `Waiting for nodes to be ready and distributing SSH keys...` then `SSH key distributed to node1 / node2`.
   > [!warning] Let the keys propagate!
   > If you exec in *instantly* and `ping` fails, the entrypoint may still be copying keys. Check with `docker compose logs ansible-control` — wait for **"Ansible control node is ready."** before you ping.

3. **Enter the control node:**
   ```bash
   docker compose exec ansible-control bash
   ```
   Your prompt is now inside the container, `WORKDIR /ansible` (your files live here via the `.:/ansible` bind mount).

4. **Ping all nodes:**
   ```bash
   ansible all -m ping
   ```
   Expected (green):
   ```
   node1 | SUCCESS => { "ansible_facts": {...}, "changed": false, "ping": "pong" }
   node2 | SUCCESS => { ... "ping": "pong" }
   ```
   `pong` = SSH + Python working on that node. 🎉

5. **Install nginx:**
   ```bash
   ansible-playbook playbooks/install_nginx.yml
   ```
   Recap shows `ok=2 changed=1` per host (the install task **changed** something the first time).

6. **Deploy the website:**
   ```bash
   ansible-playbook playbooks/deploy_website.yml
   ```
   This installs nginx (already present → `ok`), writes `index.html`, and starts nginx.

7. **Open it in a browser** (back on the host):
   - 👉 http://localhost:8081 → served by **node1**
   - 👉 http://localhost:8082 → served by **node2**
   You should see **"Hello from Ansible"**. That HTML was written to `/var/www/html/index.html` *inside the container* by the playbook. Magic. ✨

8. **(Optional) Meet Semaphore** — open http://localhost:3000, log in with `admin` / `admin123`. It's a web GUI for running these same playbooks. Same inventory, same keys (shared volume), pretty buttons.

## 🔬 Drills (earn XP)

- [ ] **Drill 1 (10 XP) — Prove idempotency.** Re-run `ansible-playbook playbooks/install_nginx.yml`. **Done when:** the recap shows `changed=0` on the second run (nginx already installed → Ansible does nothing). See [[Terminology#Idempotency]].
- [ ] **Drill 2 (15 XP) — One-liner recon.** Run `ansible all -m ping -o` and `ansible web --list-hosts`. **Done when:** you can name every host Ansible is managing and see compact `pong` output.
- [ ] **Drill 3 (20 XP) — Ad-hoc command.** Run `ansible web -m shell -a "hostname && nginx -v" -b`. **Done when:** each node reports its own hostname and the installed nginx version.
- [ ] **Drill 4 (25 XP) — Edit the Jinja2 template.** Open `templates/nginx.conf.j2`, change the return string to `'Hello from <your name>'`. Wire it into a playbook using the `template` module (dest `/etc/nginx/sites-available/default`) with `vars: { nginx_port: 80 }`, add a **[[Terminology#Handler|handler]]** to restart nginx, and re-run. **Done when:** `curl localhost:8081` shows your new text. Re-run again → **no changes** (idempotent).
- [ ] **Drill 5 (25 XP) — Add a 3rd node.** In `docker-compose.yml` add a `node3` (copy `node2`, map `8083:80`), and add `node3` to `inventory.ini`. Re-run `docker compose up -d` then re-run `deploy_website.yml`. **Done when:** http://localhost:8083 also serves the site *and* the control node distributed a key to node3 (add it to `entrypoint.sh`'s loop!).
- [ ] **Drill 6 (30 XP) — Dry run vs real.** Run `deploy_website.yml --check` first, note it reports would-be changes without applying, then run for real. **Done when:** you can explain the difference between `--check` and a live run.
- [ ] **Drill 7 (30 XP) — Semaphore run.** In the Semaphore UI (`:3000`), create a project pointing at the inventory + a playbook and run a task from the browser. **Done when:** a playbook completes via the web UI, not the terminal.

## 🧗 Extra credit — beyond class *(addition)*

> [!example] 🆕 These drills are an **addition** — not covered in the class materials
> They're the next skill up from what class taught, chosen because you'll meet them in real work (and in the [[SkyWatch Capstone]]). Higher XP, higher payoff.

- [ ] **Third node (25 XP)** — add a `node3` service to docker-compose.yml (port 8083) and to `inventory.ini`, re-run the playbooks. **Done when:** all three nodes serve the site — and note how little you had to change (that's the point of inventory).
- [ ] **Per-host content (30 XP)** — use `host_vars/` (or inventory vars) to give each node a different site title through the same `nginx.conf.j2`/site template. **Done when:** 8081 and 8082 show different titles from ONE template.
- [ ] **Rolling change (25 XP)** — add `serial: 1` to the play and re-run while refreshing the browser. **Done when:** you saw nodes update one-by-one instead of all at once (this is how prod avoids downtime).

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
- [ansible-doc module index](https://docs.ansible.com/ansible/latest/collections/index_module.html)
- [Docker Compose reference](https://docs.docker.com/compose/)

**⚡ Built-in help — try these BEFORE searching**
```bash
ansible all -m ping      # verify connectivity before debugging playbooks
ansible-doc <module>     # e.g. ansible-doc template
docker compose --help    # up / exec / logs / down
```

> [!example] Power move for this class
> When a playbook fails, run the failing module ad-hoc first: `ansible all -m <module> -a "..."`. Faster feedback than editing the playbook each time. And `ansible-doc template` explains the Jinja2 module.

## 🧪 Self-check quiz

> [!question]- Why does `ansible all -m ping` succeed with no password prompt?
> Because `entrypoint.sh` on the control node generated an SSH keypair and used `sshpass` + `ssh-copy-id` to install the **public key** on each node's `ubuntu` user *before* you ever ran a command. Ansible then authenticates with the matching **private key** (`/root/.ssh/id_rsa`, set in the inventory). No password needed.

> [!question]- What does `ansible.cfg`'s `host_key_checking = False` do, and why is it OK here?
> It skips SSH's "are you sure you want to connect to this unknown host?" prompt. In a throwaway lab where containers are recreated constantly, host keys change and prompting would break automation. In **production** you'd usually leave host key checking **on** for security.

> [!question]- On the *second* run of `install_nginx.yml`, why is `changed=0`?
> **Idempotency.** The `apt` module checks current state: nginx is already `present`, so there's nothing to do → it reports `ok` (not `changed`). Ansible describes *desired state*, not step-by-step commands, so re-running is safe.

> [!question]- Where do you run the playbooks — the host or the control container?
> **Inside the `ansible-control` container** (`docker compose exec ansible-control bash` first). The host doesn't have Ansible, the inventory, or the SSH keys. The control node is the one wired to reach the nodes over `ansible-net`.

> [!question]- Why must inventory hostnames be `node1` / `node2` exactly?
> Because Docker Compose creates DNS entries for each service using its **service/hostname**. `node1` resolves to that container's IP *on the `ansible-net` network*. If the inventory said `webserver1`, Ansible couldn't resolve it and SSH would fail.

> [!question]- What extra value does the `deploy_website.yml` playbook add over `install_nginx.yml`?
> It doesn't just install nginx — it **deploys content**: uses the `copy` module to write `index.html` and the `service` module to ensure nginx is **started**. It's a fuller "configure + deploy" workflow. The `nginx.conf.j2` template shows the next step: rendering config per-host with variables like `{{ nginx_port }}`.

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- Control node
> The machine (here the `ansible-control` container) that stores playbooks/inventory and runs Ansible, connecting *out* to managed nodes over SSH. Agentless — nothing special installed on the targets.

> [!question]- Managed / target node
> A host Ansible configures (here `node1`, `node2`). Needs only SSH + Python. Exposes port 80 to the host as 8081/8082.

> [!question]- entrypoint.sh (this lab)
> Startup script on the control node that generates an SSH keypair and uses `sshpass` + `ssh-copy-id` to distribute the public key to each node, retrying until SSH is up.

> [!question]- inventory.ini
> Ansible's address book — a `[web]` group of `node1`/`node2` plus `[web:vars]` setting `ansible_user=ubuntu` and the private-key path.

> [!question]- ansible.cfg
> Config file setting defaults: `inventory=inventory.ini`, `host_key_checking=False`, `retry_files_enabled=False`.

> [!question]- ansible all -m ping
> Ad-hoc connectivity test; `-m ping` runs the ping module. Success returns `"ping": "pong"` — proves SSH + Python work.

> [!question]- ansible-playbook
> Command that executes a YAML playbook against inventory hosts, running its tasks in order.

> [!question]- become: true
> Playbook directive to run tasks with privilege escalation (sudo) on the target.

> [!question]- Idempotency
> Running the same playbook repeatedly yields the same end state; unchanged tasks report `ok`, not `changed`. See [[Terminology#Idempotency]].

> [!question]- Jinja2 template
> A `.j2` file (e.g. `nginx.conf.j2`) with `{{ variables }}` that Ansible renders per-host via the `template` module. See [[Terminology#Template (Jinja2)]].

> [!question]- Handler
> A task triggered only when `notify`-ed by a changed task (e.g. restart nginx after config change). See [[Terminology#Handler]].

> [!question]- Semaphore
> A web UI on top of Ansible (here at `localhost:3000`, admin/admin123) for running playbooks with a GUI instead of the CLI.

> [!question]- ssh_keys volume
> Shared Docker volume where the control node writes the keypair so Semaphore can read it (`:ro`).

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> Control node::The machine (here the `ansible-control` container) that stores playbooks/inventory and runs Ansible, connecting *out* to managed nodes over SSH. Agentless — nothing special installed on the targets.
> Managed / target node::A host Ansible configures (here `node1`, `node2`). Needs only SSH + Python. Exposes port 80 to the host as 8081/8082.
> entrypoint.sh (this lab)::Startup script on the control node that generates an SSH keypair and uses `sshpass` + `ssh-copy-id` to distribute the public key to each node, retrying until SSH is up.
> inventory.ini::Ansible's address book — a `[web]` group of `node1`/`node2` plus `[web:vars]` setting `ansible_user=ubuntu` and the private-key path.
> ansible.cfg::Config file setting defaults: `inventory=inventory.ini`, `host_key_checking=False`, `retry_files_enabled=False`.
> ansible all -m ping::Ad-hoc connectivity test; `-m ping` runs the ping module. Success returns `"ping": "pong"` — proves SSH + Python work.
> ansible-playbook::Command that executes a YAML playbook against inventory hosts, running its tasks in order.
> become: true::Playbook directive to run tasks with privilege escalation (sudo) on the target.
> Idempotency::Running the same playbook repeatedly yields the same end state; unchanged tasks report `ok`, not `changed`. See [[Terminology#Idempotency]].
> Jinja2 template::A `.j2` file (e.g. `nginx.conf.j2`) with `{{ variables }}` that Ansible renders per-host via the `template` module. See [[Terminology#Template (Jinja2)]].
> Handler::A task triggered only when `notify`-ed by a changed task (e.g. restart nginx after config change). See [[Terminology#Handler]].
> Semaphore::A web UI on top of Ansible (here at `localhost:3000`, admin/admin123) for running playbooks with a GUI instead of the CLI.
> ssh_keys volume::Shared Docker volume where the control node writes the keypair so Semaphore can read it (`:ro`).

## ⚠️ Gotchas

> [!warning] Common traps that eat 20 minutes
> - **SSH keys must propagate before ping works.** `entrypoint.sh` handles this, but it *retries* until nodes' SSH is up. If you exec in too fast, `ping` fails. Check `docker compose logs ansible-control` for **"Ansible control node is ready."** first.
> - **Run playbooks from the control *container*, not the host.** The host has no Ansible, no inventory, no keys. Always `docker compose exec ansible-control bash` first.
> - **Inventory hostnames must match Compose service/hostnames.** `node1`/`node2` resolve via Docker DNS on `ansible-net`. Rename one without updating both files and SSH can't find it.
> - **Add new nodes in THREE places.** A 3rd node needs entries in `docker-compose.yml` (service + port), `inventory.ini` (`[web]`), *and* `entrypoint.sh`'s key-distribution loop — otherwise it has no key and ping fails.
> - **Port already in use?** 8081/8082/3000 must be free on the host. `docker compose down` old stacks first.
> - **`become: true` needs passwordless sudo** — provided by the `Dockerfile` (`ubuntu ALL=(ALL) NOPASSWD:ALL`). On real hosts you'd supply `--ask-become-pass`.

## 🏆 Boss challenge

**Change the live website through Ansible and prove it in the browser.**

1. Edit `deploy_website.yml` — change the `content:` block to your own HTML, e.g. `<h1>Deployed by <your name> 🚀</h1>`.
2. (Harder) Instead of inline `content`, add a `templates/index.html.j2` with `<h1>{{ site_owner }} runs {{ inventory_hostname }}</h1>`, switch the task to the `template` module, and pass `vars: { site_owner: "You" }`. Now **each node shows its own hostname** — proof the template renders per-host.
3. Re-run `ansible-playbook playbooks/deploy_website.yml`.
4. Refresh http://localhost:8081 **and** http://localhost:8082.

**Win condition:** Both pages show your new content, node1 and node2 display *different* hostnames, and a second playbook run reports `changed=0` for the unchanged tasks. 🏅

> [!success] Class 14 cleared when you can…
> …stand up the dockerized lab, ping both nodes, run both playbooks, see "Hello from Ansible" in the browser on 8081/8082, explain how `entrypoint.sh` + `ansible.cfg` + `inventory.ini` cooperate, and prove idempotency by re-running a playbook to `changed=0`.
>
> 🎖️ **Badge: Automation Alchemist** — you turned containers into an obedient datacenter with a single YAML incantation.

> [!cheatsheet]- 🔒 Command cheat-sheet — try the drills FIRST (expanding = peeking 👀)
> ⚠️ **Wait!** Expanding this is basically giving up. Did you really attempt the drill from memory? If not, collapse me and try again — the struggle is what makes it stick. Still stuck? Fine, the commands are below. 👇
>
> ```bash
> # --- On the HOST (your laptop) ---
> cd ansible_lab_files
>
> # Build + start all containers in the background (control, node1, node2, semaphore)
> docker compose up -d
>
> # Drop into a shell on the control node
> docker compose exec ansible-control bash
>
> # --- INSIDE the ansible-control container ---
> # Test connectivity to every host in the inventory (expect green SUCCESS/pong)
> ansible all -m ping
>
> # Install nginx on both web nodes
> ansible-playbook playbooks/install_nginx.yml
>
> # Deploy the website (installs nginx + drops index.html + starts service)
> ansible-playbook playbooks/deploy_website.yml
>
> # --- Handy extras (not in the base script but great to know) ---
> ansible all -m ping -o                 # one-line-per-host output
> ansible web --list-hosts               # who's in the 'web' group?
> ansible-playbook playbooks/deploy_website.yml --check   # dry-run, change nothing
> ansible web -m shell -a "nginx -v" -b  # ad-hoc command across all nodes
> ```

## 🔗 Related

- [[Class 11 - Ansible]] — the theory this lab makes hands-on
- [[Class 02 - Docker Networking and Images]] — the Compose network + images under the hood
- [[Terminology]] — [[Terminology#Ansible]] · [[Terminology#Playbook]] · [[Terminology#Inventory]] · [[Terminology#Template (Jinja2)]] · [[Terminology#Handler]] · [[Terminology#Idempotency]] · [[Terminology#Docker Compose]]
- [[DevOps Experts — MOC]] — course map
