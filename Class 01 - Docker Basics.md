---
tags: [devops, docker, class-01]
aliases: [Docker Basics, Class 1 Docker, Containers 101]
class: 01
difficulty: beginner
---

# 🐳 Class 01 — Docker Basics

> [!info]- 🔢 Which class is this, really?
> The number in this note's title follows the **course repo folder** `class1/` — **not** the calendar session. The real session numbering is shifted: calendar-session 1 was the DevOps intro (see [[DevOps Foundations]]), so e.g. Docker basics was *taught* in session 2 even though its repo folder is `class1`. When in doubt, navigate by **topic** via [[DevOps Experts — MOC]], not by number.


> [!abstract] TL;DR
> A Docker **image** is a frozen, read-only template; a **container** is a running (or stopped) instance of that image — like starting a program from an installer. You `pull` images, `run` containers, `exec` into them to poke around, and throw them away when you're done. A real engineer cares because containers make "works on my machine" die: the *exact* same environment ships from laptop to CI to production, and spinning up a clean one takes seconds.

## 🎯 Learning goals
- [ ] Explain the difference between an [[Terminology#Image]] and a [[Terminology#Container]] without hesitating
- [ ] Pull an image from a [[Terminology#Registry]] and run a container from it
- [ ] Get an interactive shell **inside** a running container with `docker exec`
- [ ] Create files inside a container and understand why they vanish
- [ ] List, inspect, and read logs of containers (`ps`, `logs`, `images`)
- [ ] Clean up stopped containers and dangling images without fear

## 🧩 The big idea

Think of an **image as a recipe** 📜 and a **container as the cooked meal** 🍲.

- The recipe (image) is written once, never changes, and can be copied a million times.
- Each meal (container) is cooked *from* that recipe. You can cook ten meals from one recipe, eat them, and toss the plates — the recipe is untouched.
- If you scribble on a plate mid-meal (create a file inside a container), that scribble dies with the plate. The recipe never learns about it.

That last point is the whole personality of Docker: **containers are disposable**. You don't nurse them back to health — you delete them and cook a fresh one from the same recipe.

## 🧠 Core concepts

- **[[Terminology#Docker]]** — a platform that packages software plus everything it needs to run into a single portable unit, isolated from the host.
- **[[Terminology#Image]]** — a read-only, layered snapshot of a filesystem + metadata (what to run, env vars, etc.). Immutable. `ubuntu:latest` is an image. The `:latest` part is the **tag** (a version label).
- **[[Terminology#Container]]** — a running or stopped *instance* of an image. It gets its own writable layer on top of the image's read-only layers. Delete it and that writable layer is gone.
- **[[Terminology#Registry]]** — a warehouse of images you `pull` from and `push` to. Docker Hub is the default public one; `docker pull ubuntu` fetches from there.
- **[[Terminology#Dockerfile]]** — a plain-text script of instructions to *build* your own image. (You'll write these next class — Class 01 just uses pre-built images.)
- **[[Terminology#Volume]]** — the fix for disposability: a persistent storage area that outlives the container, so data you *want* to keep survives. (Preview — deeper dive later.)

> [!tip] The mental model that unlocks everything
> **Image = class, Container = object.** One image → many containers, exactly like one class → many instances in code. If you've ever written `x = new Thing()`, you already understand `docker run`.

## 🛠️ Guided walkthrough — do this on your machine now

Open a terminal and run these in order. Expected output shown so you know it worked.

1. **Pull the image**
   ```bash
   docker pull ubuntu:latest
   ```
   Expected: a few `Pull complete` lines, ending with `Status: Downloaded newer image for ubuntu:latest` (or `Image is up to date` if you already have it).

2. **Run a detached container**
   ```bash
   docker run -dit --name devops2912 ubuntu:latest bash
   ```
   Expected: one long hex string (the container ID). No shell yet — it's running in the background.

3. **Confirm it's alive**
   ```bash
   docker ps
   ```
   Expected: one row with `NAME` = `devops2912`, `STATUS` = `Up ... seconds`.

4. **Jump inside**
   ```bash
   docker exec -it devops2912 bash
   ```
   Expected: your prompt changes to something like `root@<id>:/#`. You are now *inside* the container.

5. **Make some files (inside the container)**
   ```bash
   cd /root
   touch file1.txt
   mkdir temp
   cp file1.txt file2.txt
   mv file1.txt temp/
   ls
   ls temp
   ```
   Expected: `ls` shows `file2.txt  temp`, and `ls temp` shows `file1.txt`.

6. **Prove disposability** — leave, kill, recreate:
   ```bash
   exit                                   # back to your host shell
   docker rm -f devops2912                # force-remove the container
   docker run -dit --name devops2912 ubuntu:latest bash
   docker exec -it devops2912 ls /root
   ```
   Expected: `/root` is **empty**. Your `file2.txt` and `temp/` are gone. Same recipe, brand-new plate. 🍽️

> [!example] What "it works on my machine" looks like solved
> The image you pulled is byte-for-byte the one a teammate in another country pulls. Step 5 produces the same result for both of you. No "but I have Python 3.9 and you have 3.11" — the environment travels *with* the image.

## 🔬 Drills (earn XP)

- [ ] **(10 XP)** Pull `ubuntu:latest` and list your local images. **Done when:** `docker images` shows an `ubuntu` row with tag `latest`.
- [ ] **(15 XP)** Run a container named `drill1` detached, then confirm it's up. **Done when:** `docker ps` shows `drill1` with `STATUS = Up`.
- [ ] **(15 XP)** `exec` into `drill1`, run `env` and `pwd`, then `exit`. **Done when:** you saw env vars and a working directory path, and you're back on your host shell.
- [ ] **(20 XP)** Inside `drill1`, recreate the file dance: `touch`, `mkdir temp`, `cp`, `mv` into `temp/`. **Done when:** `ls temp` shows your moved file.
- [ ] **(20 XP)** Read the container's logs from the host. **Done when:** `docker logs drill1` runs without error (may be empty — that's fine, you proved you can reach them).
- [ ] **(15 XP)** Destroy `drill1` and prove it's gone. **Done when:** `docker ps -a` no longer lists `drill1`.
- [ ] **(25 XP)** Full cleanup: remove any leftover stopped containers and dangling images. **Done when:** `docker ps -a` is empty of drill containers and `docker image prune` reports reclaimed space (or nothing to do).

> 🧮 **Total available: 120 XP.** 100+ = you've earned the badge below.

## 🧗 Extra credit — beyond class *(addition)*

> [!example] 🆕 These drills are an **addition** — not covered in the class materials
> They're the next skill up from what class taught, chosen because you'll meet them in real work (and in the [[SkyWatch Capstone]]). Higher XP, higher payoff.

- [ ] **Data that survives (25 XP)** — class showed containers are disposable. Now beat that: create a named volume (`docker volume create mydata`), run `docker run -dit -v mydata:/root/keep --name v1 ubuntu bash`, create a file in `/root/keep`, remove the container, start a NEW one with the same `-v` — the file is still there. **Done when:** you can explain what belongs in a volume vs the container.
- [ ] **X-ray a container (20 XP)** — `docker inspect v1` dumps everything. Extract just the IP with `docker inspect -f '{{.NetworkSettings.IPAddress}}' v1`. **Done when:** you found one more field worth extracting (try `.State.StartedAt`).
- [ ] **Put it on a diet (15 XP)** — re-run ubuntu with `--memory=100m` and check `docker stats`. **Done when:** you can say why prod containers always get memory limits.

## 🔎 Learn to fish — find it yourself (don't just copy)

> [!tip] Beat "monkey-see-monkey-do"
> The real skill isn't memorizing commands — it's finding the right one **fast**. Pros do this all day. Build the reflex:
> 1. **Ask the tool first:** `<tool> --help`, `<tool> <subcommand> --help`, `man <tool>`.
> 2. **Official docs = source of truth** (below) — not random blogs or old Stack Overflow answers.
> 3. **Search smart:** `<what you want> site:<official-docs-domain>`; add the tool's version if behaviour changed between releases.
> 4. **Read the WHOLE error message** — it almost always names the missing flag or the fix.
> 5. `tldr <command>` gives real-world examples (install a `tldr` client — it's the friendly `man`).

**📚 Docs & references**
- [Docker — Get Started](https://docs.docker.com/get-started/)
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Docker Hub (find images)](https://hub.docker.com/)

**⚡ Built-in help — try these BEFORE searching**
```bash
docker --help            # top-level: every subcommand
docker run --help        # every flag for run (-d, -it, -p, --name, --rm ...)
docker ps --help
```

> [!example] Power move for this class
> Any subcommand takes `--help`: `docker exec --help`, `docker rm --help`. That's how you learn flags without Googling.

## 🧪 Self-check quiz

> [!question]- What is the difference between an image and a container?
> An **image** is a read-only, immutable template (the recipe). A **container** is a runnable instance created from an image (the cooked meal), with its own writable layer. One image → many containers.

> [!question]- Why did your files disappear after `docker rm -f` + re-run?
> Files you created lived in the container's **writable layer**, which is destroyed when the container is removed. The image never changed. To persist data you'd use a [[Terminology#Volume]].

> [!question]- What do the flags in `docker run -dit` mean?
> `-d` detached (background), `-i` interactive (keep stdin open), `-t` allocate a tty (terminal). Together: run in the background but stay ready for an interactive `exec` later.

> [!question]- What's the difference between `docker run` and `docker exec`?
> `docker run` **creates and starts a new container** from an image. `docker exec` **runs a command inside an already-running container**. New meal vs. reaching into a meal you're already cooking.

> [!question]- Where did `docker pull ubuntu:latest` download the image from, and what is `latest`?
> From a [[Terminology#Registry]] — Docker Hub by default. `latest` is the image **tag** (a version label); it's just a name, not a guarantee of freshness.

> [!question]- How do you check which containers are running vs. all containers ever created?
> `docker ps` shows only **running** containers. `docker ps -a` shows **all** — running and stopped.

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- Image
> A read-only, immutable, layered template that a container is created from (the recipe).

> [!question]- Container
> A running or stopped instance of an image, with its own writable layer (the cooked meal).

> [!question]- docker pull
> Downloads an image from a registry (Docker Hub by default) to your machine.

> [!question]- docker run
> Creates and starts a NEW container from an image.

> [!question]- docker exec
> Runs a command inside an ALREADY-running container.

> [!question]- -dit flags
> -d detached, -i interactive (stdin open), -t tty (terminal).

> [!question]- docker ps
> Lists running containers; add -a to include stopped ones.

> [!question]- docker logs
> Prints the stdout/stderr a container has produced.

> [!question]- docker images
> Lists images stored locally on your machine.

> [!question]- Registry
> A store of images you pull from / push to; Docker Hub is the default public one.

> [!question]- Disposable container
> A container you delete and recreate rather than repair; its writable data is lost on removal.

> [!question]- Tag
> A version label on an image, e.g. the `latest` in `ubuntu:latest`.

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> Image::A read-only, immutable, layered template that a container is created from (the recipe).
> Container::A running or stopped instance of an image, with its own writable layer (the cooked meal).
> docker pull::Downloads an image from a registry (Docker Hub by default) to your machine.
> docker run::Creates and starts a NEW container from an image.
> docker exec::Runs a command inside an ALREADY-running container.
> -dit flags::-d detached, -i interactive (stdin open), -t tty (terminal).
> docker ps::Lists running containers; add -a to include stopped ones.
> docker logs::Prints the stdout/stderr a container has produced.
> docker images::Lists images stored locally on your machine.
> Registry::A store of images you pull from / push to; Docker Hub is the default public one.
> Disposable container::A container you delete and recreate rather than repair; its writable data is lost on removal.
> Tag::A version label on an image, e.g. the `latest` in `ubuntu:latest`.

## ⚠️ Gotchas

> [!warning] `docker run` does NOT reuse a container
> Every `docker run` makes a **brand-new** container. If you `run` twice with the same `--name`, the second fails with a name conflict. To get back into an existing container, use `docker exec`, not `docker run`.

> [!warning] Your data is not safe inside a container
> Anything you write inside a container dies when the container is removed. Never store something you care about in a container's filesystem — that's what [[Terminology#Volume]]s are for.

> [!warning] Removing an image while a container uses it
> `docker rmi ubuntu` fails if any container (even a stopped one) was created from it. Remove the containers first (`docker rm`), or use `-f` deliberately.

> [!warning] `exit` inside the shell can stop the container
> If you `exec`'d in with a shell, typing `exit` just leaves the shell — the container keeps running (because it was started detached). But if that shell **was** the container's main process (e.g. you ran without `-d`), exiting stops the whole container. Know which one you're in.

## 🏆 Boss challenge

Do this end-to-end in one sitting, no notes:

1. Pull `ubuntu:latest`.
2. Run a detached container named `boss` from it.
3. `exec` in and create `/root/report.txt` containing the text `container wrangler` (use `echo container wrangler > /root/report.txt`).
4. From the **host**, prove the file exists inside the container without opening an interactive shell (hint: `docker exec boss cat /root/report.txt`).
5. Destroy `boss`, recreate a fresh `boss` from the same image, and prove `/root/report.txt` is **gone**.
6. Clean up: remove the container and run `docker image prune`.

**You win when** you can articulate *out loud* why step 5's file vanished and step 1's image stayed intact.

> [!success] Class 01 cleared when you can…
> - [ ] Say the image-vs-container difference in one breath
> - [ ] Pull, run, and exec without looking anything up
> - [ ] Explain why containers are disposable and where real data should live
> - [ ] List, log, and clean up containers and images confidently
>
> 🎖️ **Badge: Container Wrangler** — unlocked at 100+ XP and a completed boss challenge.

> [!cheatsheet]- 🔒 Command cheat-sheet — try the drills FIRST (expanding = peeking 👀)
> ⚠️ **Wait!** Expanding this is basically giving up. Did you really attempt the drill from memory? If not, collapse me and try again — the struggle is what makes it stick. Still stuck? Fine, the commands are below. 👇
>
> ```bash
> docker pull ubuntu:latest                        # download the ubuntu image from Docker Hub
> docker run -dit --name devops2912 ubuntu:latest bash   # start a container: -d detached, -i interactive, -t tty; named devops2912
> docker exec -it devops2912 bash                  # open an interactive shell INSIDE the running container
>
> # --- these run INSIDE the container ---
> env                                              # show environment variables
> pwd                                              # print working directory
> cd                                               # change directory (no arg = go home, /root)
> ls                                               # list files
> echo                                             # print text / test that the shell works
> cat                                              # print a file's contents
> cp                                               # copy a file
> mv                                               # move / rename a file
>
> cd /root                                         # go to root's home directory
> touch file1.txt                                  # create an empty file
> mkdir temp                                       # make a directory
> cp file1.txt file2.txt                           # copy file1 -> file2
> mv file1.txt temp/                               # move file1 into temp/
> ```
>
> > [!tip] Decode `-dit` once and never forget it
> > `-d` **d**etached (runs in background), `-i` **i**nteractive (keeps stdin open), `-t` allocates a **t**ty (a terminal). `-dit` = "run in the background but stay ready for me to jump in later." That's why we can `exec` into it afterward.

## 🔗 Related
- [[Class 02 - Docker Networking and Images]]
- [[Terminology]]
- [[DevOps Experts — MOC]]
