---
tags: [devops, docker, networking, images, class-02]
aliases: ["Docker Networking", "Docker Images", "Dockerfile Build", "Class 2"]
class: 02
difficulty: beginner
---

# 🕸️ Class 02 — Docker Networking & Images

> [!info]- 🔢 Which class is this, really?
> The number in this note's title follows the **course repo folder** `class2/` — **not** the calendar session. The numbering is genuinely chaotic — **three systems disagree**: the repo folders, the teacher's drive folders, and his deck titles (which reflect the real calendar). Case in point: the Git material lives in repo folder `class3/` and drive folder `class 3`, but its own deck title says **Class 6** — the actual session count. Session 1 was the DevOps intro (see [[DevOps Foundations]]). Bottom line: navigate by **topic** via [[DevOps Experts — MOC]], never by number.


> [!abstract] TL;DR
> - Containers on the **same user-defined network** can reach each other **by name** (`ping nginx2`, `wget nginx2`). The **default `bridge` network does NOT give you name resolution** — you'd have to use raw IPs. So: **always `docker network create` your own network.**
> - A **Dockerfile** is a recipe. `docker build -t myimg .` bakes your app into an **image**; `docker run -p host:container` runs it and wires a host port to the container port.
> - Key instructions: `FROM` (base) → `WORKDIR` (cwd) → `COPY` (files in) → `RUN` (build-time commands) → `EXPOSE` (doc hint) → `CMD` (what runs at start).
> - **Layer caching** = Docker reuses unchanged steps. **COPY dependencies BEFORE code** so editing your app doesn't blow away the slow `pip install` layer.
> - **slim / alpine base images** are much smaller than the full one. Smaller = faster pulls, smaller attack surface.

## 🎯 Learning goals

- [ ] Create a user-defined network and attach containers to it
- [ ] Make two containers talk **by name** (ping / wget), and explain why default bridge can't
- [ ] Read every line of a Dockerfile and say what it does
- [ ] `docker build` your own image and tag it
- [ ] Map a container port to your host with `-p` and hit it with `curl`
- [ ] Explain **layer caching** and order your Dockerfile to exploit it
- [ ] Compare `slim`/`alpine` vs full base image sizes

## 🧩 The big idea

**Networking = a private office phone directory. 📞**
Put a bunch of people (containers) in the **same private office** (a user-defined network). Now anyone can pick up the phone and dial a **colleague by name** — "get me `nginx2`" — and Docker's built-in DNS connects the call. Out in the public lobby (the default `bridge` network) there's no directory: you'd need to memorize everyone's phone number (their IP), which changes every time they restart. So you build your own office and invite people in.

**Building an image = writing down a recipe once, then reheating instantly. 🍲**
The `Dockerfile` is the recipe. The first `docker build` cooks every step from scratch (slow — especially `pip install`). Docker photographs each finished step as a **layer**. Next build, if a step's inputs didn't change, Docker says "I already cooked this" and reuses the photo — **instant**. The trick: put the ingredients that rarely change (dependencies) at the bottom of the pot, and the thing you tweak constantly (your code) on top — so you only re-cook the top.

## 🧠 Core concepts

- **[[Terminology#Container|Container]]** — a running instance of an image. Isolated process with its own filesystem and network identity.
- **[[Terminology#Image|Image]]** — the frozen, read-only template you `run` to get a container. Built from a Dockerfile.
- **[[Terminology#Docker network|Docker network]]** — a virtual switch that connects containers. On a **user-defined** network, Docker runs an embedded **DNS** so container **names resolve to IPs automatically**. The **default `bridge`** network does *not* do name-based DNS.
- **[[Terminology#Dockerfile|Dockerfile]]** — the text recipe. Each instruction (`FROM`, `COPY`, `RUN`, …) produces a build step.
- **[[Terminology#Layer|Layer]]** — one cached filesystem diff per instruction. Unchanged layers are reused → the heart of **build caching**.
- **[[Terminology#Port mapping|Port mapping]]** (`-p HOST:CONTAINER`) — publishes a container's internal port to a port on your host so the outside world can reach it. **Left = host, right = container.**
- **Base image size** — `python:3` is ~1 GB; `python:3-slim` is ~150 MB; `alpine` variants are tiny. Smaller image = faster pull, smaller attack surface.

## 🛠️ Guided walkthrough

### Lab A — Two nginx containers pinging by name 🏓

Straight from `class2/commands`:

```bash
docker network create nginx-network
docker run --name nginx1 --rm --network nginx-network -d nginx:alpine
docker run --name nginx2 --rm --network nginx-network -d nginx:alpine
docker exec -it nginx1 sh
# now inside nginx1:
ping nginx2
```

**Expected output** (inside `nginx1`):

```text
PING nginx2 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.18.0.3: seq=1 ttl=64 time=0.062 ms
```

✅ The name `nginx2` resolved to an IP **automatically** — that's the user-defined network's DNS. Type `exit` to leave the shell; stop with `docker stop nginx1 nginx2` (the `--rm` auto-removes them).

> [!tip] Prove the "by name" magic is the network's doing
> Run the same two containers **without** `--network` (they land on default bridge), `exec` in, and `ping nginx2`. You'll get `ping: bad address 'nginx2'` — no DNS. That's the whole point of building your own network.

### Lab B — Build the Flask image and curl it 🐍

The repo has `nginx-app/app.py` (a Flask app listening on `0.0.0.0:8080`). Give it this Dockerfile (in the same folder as `app.py`):

```dockerfile
FROM python:3-slim
WORKDIR /usr/src
COPY requirements.txt /usr/src/        # deps first  -> cached separately from code
RUN pip install -r requirements.txt
COPY app.py /usr/src/                  # code last   -> editing it won't re-run pip
EXPOSE 8080
CMD ["python3", "app.py"]
```

with `requirements.txt`:

```text
flask
```

Build and run:

```bash
docker build -t flask-app .
docker run --rm -p 8080:8080 flask-app
```

**Expected output** (in the run terminal):

```text
listening on port:  8080
 * Serving Flask app 'app'
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:8080
```

In a second terminal:

```bash
curl http://localhost:8080
```

**Expected:**

```text
Hello! I am a Flask application
```

> [!example] The teacher's original one-liner image
> `class2/Dockerfile` builds `first.py` (prints the date):
> ```dockerfile
> FROM python:3
> WORKDIR /usr/src
> COPY first.py /usr/src
> CMD ["python3","/usr/src/first.py"]
> ```
> `docker build -t datenow . && docker run --rm datenow` → prints something like `2026-07-08 14:03:11.482913`. No `-p` needed — it's not a server, it runs once and exits.

## 🔬 Drills (earn XP)

- [ ] **(10 XP)** Create `nginx-network`, attach `nginx1` + `nginx2`, and `ping nginx2` from inside `nginx1`. **Done when:** you see 64-byte replies.
- [ ] **(10 XP)** From inside `nginx1`, run `wget -qO- nginx2` and get nginx's default HTML back **by name**. **Done when:** the "Welcome to nginx!" HTML prints.
- [ ] **(10 XP)** Repeat the ping **without** `--network` (default bridge). **Done when:** you can quote the exact error (`bad address 'nginx2'`) and explain it.
- [ ] **(15 XP)** Build `class2/Dockerfile` as `datenow` and run it. **Done when:** it prints the current timestamp and exits.
- [ ] **(15 XP)** Build & run the Flask app with `-p 8080:8080` and `curl localhost:8080`. **Done when:** you get `Hello! I am a Flask application`.
- [ ] **(20 XP) Cache experiment:** build the Flask image, then edit **`app.py`** and rebuild — watch which steps say `CACHED`. Then edit **`requirements.txt`** and rebuild. **Done when:** you can explain why editing code kept `pip install` cached but editing requirements did not.
- [ ] **(20 XP) slim vs full:** build one image `FROM python:3` and one `FROM python:3-slim`, run `docker images`, compare the `SIZE` column. **Done when:** you can state the roughly ~5–7× size difference.

## 🖥️ From the class deck *(addition — mined from `Class2.Docker.pptx`)*

> [!info] Three things the slides stress that are easy to miss
> 1. **Containers vs VMs — the kernel is the difference.** A VM runs on a [[Terminology#Hypervisor]] and brings its *own* full OS; containers all share the **host kernel**, which is why a container starts in ~1s and weighs MBs while a VM takes minutes and GBs. Interview-favorite question.
> 2. **Don't forget the dot.** `docker build -t timestamp:1.0 .` — the trailing `.` is the **build context** (which files Docker may COPY). Forgetting it is the #1 first-build error.
> 3. **[[Terminology#Nginx]] is not "just a demo image"** — it's a reverse proxy / load balancer / cache serving ~a quarter of the busiest sites. The deck's exercise (`docker run --name docker-nginx -p 80:80 -d nginx`) is the same port-mapping muscle as the Flask lab.
>
> 📄 Full deck: [class2-docker.pptx](uploads/class-02-docker/class2-docker.pptx) *(in this repo's `uploads/`)*

## 📬 The REAL assignment (from Yariv's drive)

> [!important] 🐳 Docker Basics — Assignment 1: *From Application to Public Docker Hub Image*
> The actual graded homework. You get a small Python web server (`app.py`, port 8080, stdlib only) and must — **without copying a Dockerfile from the internet**:
> 1. **Write your own Dockerfile** — base image, workdir, copy `app.py`, expose the port, start command. *You must understand every instruction.*
> 2. **Build & tag** the image with a meaningful name, verify it exists locally.
> 3. **Create a Docker Hub account** — username = your permanent public namespace, lowercase.
> 4. **Create an access token** (Account Settings → Security → New Access Token, Read & Write) and `docker login` with it — **never your account password** in the terminal.
> 5. **Create a public repo** and push: re-tag as `<dockerhub-username>/<repo>` first. Remember: *public = anyone can pull; pushing always requires login.*
>
> Allowed sources: official Docker docs + `docker --help` only — which is exactly what the **Learn to fish** section below trains.
>
> 🗡️ **Practice it risk-free first:** mission 4 "Ship It ⚓" in [Shell Quest](https://github.com/iceteps/shell-quest) simulates this exact flow — including the classic `denied: requested access` errors.

## 🧗 Extra credit — beyond class *(addition)*

> [!example] 🆕 These drills are an **addition** — not covered in the class materials
> They're the next skill up from what class taught, chosen because you'll meet them in real work (and in the [[SkyWatch Capstone]]). Higher XP, higher payoff.

- [ ] **Multi-stage build (30 XP)** — class built a single-stage image. Rewrite the Flask Dockerfile with two stages: a `builder` stage that pip-installs into a venv, and a slim final stage that only `COPY --from=builder` the venv. Compare sizes. **Done when:** the final image is smaller and still serves.
- [ ] **HEALTHCHECK (20 XP)** — add `HEALTHCHECK CMD python -c "import urllib.request;urllib.request.urlopen('http://localhost:8080')"` to the Dockerfile; watch `docker ps` flip to `(healthy)`. **Done when:** you can explain who consumes health status (hint: Compose `depends_on: condition`).
- [ ] **Ship it (25 XP)** — create a free Docker Hub account, `docker tag` your image `yourname/my-flask-app:v1`, `docker push` it, then pull+run it on the other side (delete local first). **Done when:** your image runs from the registry, not the local cache.

## 🔎 Learn to fish — find it yourself (don't just copy)

> [!tip] Beat "monkey-see-monkey-do"
> The real skill isn't memorizing commands — it's finding the right one **fast**. Pros do this all day. Build the reflex:
> 1. **Ask the tool first:** `<tool> --help`, `<tool> <subcommand> --help`, `man <tool>`.
> 2. **Official docs = source of truth** (below) — not random blogs or old Stack Overflow answers.
> 3. **Search smart:** `<what you want> site:<official-docs-domain>`; add the tool's version if behaviour changed between releases.
> 4. **Read the WHOLE error message** — it almost always names the missing flag or the fix.
> 5. `tldr <command>` gives real-world examples (install a `tldr` client — it's the friendly `man`).

**📚 Docs & references**
- [Networking overview](https://docs.docker.com/network/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [docker build reference](https://docs.docker.com/reference/cli/docker/buildx/build/)

**⚡ Built-in help — try these BEFORE searching**
```bash
docker network --help    # create / ls / inspect / connect
docker build --help      # -t, -f, --build-arg, --no-cache
docker run --help        # find the -p / --network flags
```

> [!example] Power move for this class
> Stuck on a Dockerfile instruction? The Dockerfile reference lists every one (`FROM`,`COPY`,`RUN`,`CMD`,`ENTRYPOINT`...) with examples. Bookmark it.

## 🧪 Self-check quiz

> [!question]- Why can `nginx1` ping `nginx2` by name, but not on the default bridge network?
> A **user-defined** network runs an embedded **DNS server** that maps container **names → IPs** automatically. The **default `bridge`** network has no such name-based DNS — you'd need the raw IP (which is brittle, since it changes on restart).

> [!question]- In `docker run -p 8080:9090`, which number is the host and which is the container?
> **Left = host, right = container.** So host `8080` forwards to container `9090`. You'd `curl localhost:8080`, and the app inside must actually be listening on `9090`.

> [!question]- Why do we `COPY requirements.txt` and `RUN pip install` BEFORE `COPY app.py`?
> **Layer caching.** `pip install` is slow. If code is copied *before* deps, any code edit invalidates the deps layer and forces a re-install every build. Copying deps first means editing `app.py` only rebuilds the fast final layers — the `pip install` layer stays **CACHED**.

> [!question]- What's the difference between `EXPOSE 8080` and `-p 8080:8080`?
> `EXPOSE` is just **documentation/metadata** inside the image — it does NOT publish anything by itself. The actual host→container wiring happens at runtime with **`-p`** (`--publish`).

> [!question]- What does `CMD ["python3","app.py"]` do, and how is it different from `RUN`?
> **`CMD`** is the default command that runs when the **container starts**. **`RUN`** executes at **build time** to bake things into the image (e.g. `pip install`). Build-time vs run-time.

> [!question]- Why prefer `python:3-slim` over `python:3`?
> It's dramatically smaller (~150 MB vs ~1 GB) → **faster pulls/pushes, less disk, smaller attack surface**. Trade-off: slim strips many system libs, so occasionally you must `apt-get install` a build dependency.

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- - User-defined network
> A network you create with `docker network create`; provides automatic DNS so containers reach each other by name.

> [!question]- - Default bridge network
> Docker's out-of-the-box network — connectivity works, but **no** name-based DNS resolution between containers.

> [!question]- - `docker network create`
> Command that makes a user-defined network with embedded DNS.

> [!question]- - `--network <name>`
> `docker run` flag that attaches a container to a specific network.

> [!question]- - `docker exec -it <c> sh`
> Open an interactive shell inside a running container.

> [!question]- - Image
> Read-only template built from a Dockerfile; you `run` it to create containers.

> [!question]- - Dockerfile
> Text recipe of instructions that Docker builds into an image.

> [!question]- - `FROM`
> Sets the base image the build starts from.

> [!question]- - `WORKDIR`
> Sets the working directory for later instructions and the running container.

> [!question]- - `COPY`
> Copies files from build context into the image.

> [!question]- - `RUN`
> Executes a command at **build time** and bakes the result into a layer.

> [!question]- - `EXPOSE`
> Documents which port the app listens on (metadata only — does not publish).

> [!question]- - `CMD`
> The default command run when the container **starts**.

> [!question]- - Layer
> A cached filesystem diff per Dockerfile instruction; unchanged ones are reused.

> [!question]- - Layer caching
> Reusing unchanged layers to skip work — order deps before code to exploit it.

> [!question]- - `-p HOST:CONTAINER`
> Publishes a container port to a host port (left=host, right=container).

> [!question]- - `docker build -t name .`
> Builds an image from the Dockerfile in `.` and tags it `name`.

> [!question]- - slim/alpine base
> Much smaller base image variants → faster pulls, smaller attack surface.

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> - User-defined network::A network you create with `docker network create`; provides automatic DNS so containers reach each other by name.
> - Default bridge network::Docker's out-of-the-box network — connectivity works, but **no** name-based DNS resolution between containers.
> - `docker network create`::Command that makes a user-defined network with embedded DNS.
> - `--network <name>`::`docker run` flag that attaches a container to a specific network.
> - `docker exec -it <c> sh`::Open an interactive shell inside a running container.
> - Image::Read-only template built from a Dockerfile; you `run` it to create containers.
> - Dockerfile::Text recipe of instructions that Docker builds into an image.
> - `FROM`::Sets the base image the build starts from.
> - `WORKDIR`::Sets the working directory for later instructions and the running container.
> - `COPY`::Copies files from build context into the image.
> - `RUN`::Executes a command at **build time** and bakes the result into a layer.
> - `EXPOSE`::Documents which port the app listens on (metadata only — does not publish).
> - `CMD`::The default command run when the container **starts**.
> - Layer::A cached filesystem diff per Dockerfile instruction; unchanged ones are reused.
> - Layer caching::Reusing unchanged layers to skip work — order deps before code to exploit it.
> - `-p HOST:CONTAINER`::Publishes a container port to a host port (left=host, right=container).
> - `docker build -t name .`::Builds an image from the Dockerfile in `.` and tags it `name`.
> - slim/alpine base::Much smaller base image variants → faster pulls, smaller attack surface.

## ⚠️ Gotchas

> [!warning] The classics that bite everyone
> - **Name resolution needs a user-defined network.** `ping nginx2` fails on the default bridge — you must `docker network create` and `--network` both containers.
> - **`-p` order is host:container.** `-p 8080:8080` is host 8080 → container 8080. Flip them by accident and `curl localhost:8080` hangs.
> - **`EXPOSE` publishes nothing.** It's only documentation. Without `-p`, no traffic reaches the container from your host.
> - **Inline `#` comments can break Dockerfiles.** A trailing comment on the *same line* as an instruction argument (e.g. after a `CMD [...]` array) can be parsed as part of the value. Put comments on **their own line**.
> - **Cache order matters.** `COPY . .` *before* `RUN pip install` means every code edit re-runs the install. Copy `requirements.txt` first.
> - **The app must bind `0.0.0.0`, not `127.0.0.1`.** `nginx-app/app.py` correctly uses `host='0.0.0.0'` — binding to localhost inside the container makes it unreachable via `-p`.
> - **`--rm` removes the container when it stops**, so don't be surprised when `docker ps -a` shows nothing after exit.

## 🏆 Boss challenge

**Build a two-container app that talks over a user-defined network — no IPs allowed.**

1. Create a network `app-net`.
2. Run the Flask image (Lab B) on it as `--name api` (no `-p` — keep it internal).
3. Run a second container `--name client` on the same network (e.g. `nginx:alpine` or `python:3-slim`).
4. From `client`, `wget -qO- http://api:8080` and get `Hello! I am a Flask application` **by name**.
5. **Stretch:** re-tag your Flask image built `FROM python:3` vs `python:3-slim`, and screenshot the `docker images` SIZE difference.

**Boss defeated when:** `client` fetches from `api` purely by name, and you can explain in one sentence why that name resolves.

> [!success] Class 02 cleared when you can…
> - [ ] Make two containers talk by name and explain why the default bridge can't
> - [ ] Read a Dockerfile line-by-line and build/run it with correct port mapping
> - [ ] Demonstrate layer caching by editing code vs requirements
> - [ ] Compare slim vs full base image sizes
>
> 🎖️ **Badge: Image Smith**

> [!cheatsheet]- 🔒 Command cheat-sheet — try the drills FIRST (expanding = peeking 👀)
> ⚠️ **Wait!** Expanding this is basically giving up. Did you really attempt the drill from memory? If not, collapse me and try again — the struggle is what makes it stick. Still stuck? Fine, the commands are below. 👇
>
> ```bash
> # ── NETWORKING ──────────────────────────────────────────
> docker network create nginx-network            # make a user-defined network (with DNS)
> docker network ls                              # list networks
> docker network inspect nginx-network           # see which containers are attached + their IPs
>
> # run two containers on that network (‑d = detached, --rm = auto-clean on stop)
> docker run --name nginx1 --rm --network nginx-network -d nginx:alpine
> docker run --name nginx2 --rm --network nginx-network -d nginx:alpine
>
> docker exec -it nginx1 sh                       # hop a shell inside nginx1
> #   inside the container:
> ping nginx2                                      # reach the other container BY NAME
> wget -qO- nginx2                                 # fetch nginx2's homepage by name
>
> # ── BUILDING YOUR OWN IMAGE ─────────────────────────────
> docker build -t my-first-app .                  # build image from Dockerfile in "." , tag it
> docker images                                   # list images + SIZE column
> docker run --rm my-first-app                     # run it (prints the date, then exits)
>
> # ── PORT MAPPING (web app) ──────────────────────────────
> docker build -t flask-app .
> docker run --rm -p 8080:8080 flask-app          # host 8080  ->  container 8080
> curl http://localhost:8080                       # hit it from the host
> ```

## 🔗 Related

- [[Class 01 - Docker Basics]]
- [[Class 13 - RabbitMQ Messaging]]
- [[Terminology]]
- [[DevOps Experts — MOC]]
