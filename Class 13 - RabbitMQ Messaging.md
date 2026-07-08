---
tags: [devops, rabbitmq, messaging, class-13]
aliases: [RabbitMQ, Message Queues, AMQP, Class 13, Work Queues, pika]
class: 13
difficulty: intermediate
---

# 📨 Class 13 — RabbitMQ Messaging

> [!info]- 🔢 Which class is this, really?
> The number in this note's title follows the **course repo folder** `class13/` — **not** the calendar session. The real session numbering is shifted: calendar-session 1 was the DevOps intro (see [[DevOps Foundations]]), so e.g. Docker basics was *taught* in session 2 even though its repo folder is `class1`. When in doubt, navigate by **topic** via [[DevOps Experts — MOC]], not by number.


> [!abstract] TL;DR
> **RabbitMQ** is a message broker: a smart mailbox that sits *between* your services. A **producer** drops a message into a **queue**; a **consumer** picks it up whenever it's ready. This **decouples** them — the producer doesn't wait for the consumer, and if consumers fall behind, the queue **buffers** the backlog instead of dropping work or crashing.
> - Talk to it two ways: **AMQP** on port **5672** (the pika Python client), or the **HTTP API / management UI** on port **15672** (`guest`/`guest`).
> - Run many identical consumers on one queue → **competing consumers**: RabbitMQ round-robins messages between them → instant horizontal scaling of workers.
> - Core pika calls: `queue_declare` (make sure the mailbox exists) → `basic_publish` (send) → `basic_consume` (receive) → `basic_ack` (confirm you handled it).

---

## 🎯 Learning goals

- [ ] Explain **why** messaging beats direct calls (buffering, resilience, scaling)
- [ ] Draw the **Producer → Queue → Consumer** flow from memory
- [ ] Publish messages with pika's `basic_publish` and `queue_declare`
- [ ] Consume messages with `basic_consume` + a `callback`
- [ ] Run **two consumers** and watch RabbitMQ **load-balance** messages between them
- [ ] Publish a message via the **HTTP API** with `curl` (the `%2F` vhost trick)
- [ ] Read **queue depth** and **consumer count** in the management UI (15672)
- [ ] Explain **ack vs auto-ack** and why it matters for not losing work

---

## 🧩 The big idea

> [!example] Analogy — the kitchen ticket rail 🍳
> Picture a busy restaurant. **Waiters** (producers) take orders and clip **tickets** onto a rail. **Cooks** (consumers) grab the next ticket *when their hands are free*.
> - The waiter never waits for a cook — they clip the ticket and walk away. **Decoupled.**
> - A dinner rush? Tickets pile up on the rail. Nothing is lost; the cooks just work through the backlog. **Buffering.**
> - Slammed? Add a second cook. Now tickets get split between them. **Scaling / competing consumers.**
> - A cook drops a plate mid-cook? The ticket wasn't torn off yet, so it goes to another cook. **Acknowledgements.**
>
> RabbitMQ *is* the ticket rail. The rail is the **queue**.

```text
                  ┌──────────────────┐
 producer.py ───► │  QUEUE  "orders" │ ───►  consumer.py       (1 cook)
 (waiter)         │  [#4][#3][#2][#1]│
                  └──────────────────┘

 With TWO cooks (competing consumers) — RabbitMQ round-robins:

                  ┌──────────────────┐  ──► consumer-A  (gets #1, #3, #5…)
 producer.py ───► │  QUEUE  "orders" │
                  └──────────────────┘  ──► consumer-B  (gets #2, #4, #6…)
```

> [!tip] Why this matters in real DevOps
> Messaging is the backbone of resilient systems: order processing, email/SMS sending, video encoding, log pipelines, and every "do this heavy thing later" job. It turns a fragile synchronous chain (if one service is down, everything fails) into an async, buffered, independently-scalable pipeline.

---

## 🧠 Core concepts

- **[[Terminology#Message queue|Message queue]]** — an ordered buffer that holds messages until a consumer takes them. FIFO-ish. It's the "ticket rail." In our labs the queue is named `orders`.
- **[[Terminology#RabbitMQ|RabbitMQ]]** — the **message broker**: the server process that stores queues, routes messages, and delivers them to consumers. We run it from the `rabbitmq:3-management` Docker image (the `-management` tag bundles the web UI).
- **[[Terminology#Producer|Producer]]** — code that **sends** messages. `producer.py` publishes 20 orders (`Order #1`…`Order #20`), one per second.
- **[[Terminology#Consumer|Consumer]]** — code that **receives** and processes messages. `consumer.py` prints each order and acks it.
- **[[Terminology#Exchange|Exchange]]** — the "post office" that decides *which* queue(s) a message goes to. You **always** publish to an exchange, never straight to a queue. The empty string `''` is the **default exchange**, which routes a message to the queue whose name equals the **routing key**. That's the shortcut our labs use.
- **[[Terminology#Routing key|Routing key]]** — the address label on the message. With the default exchange, `routing_key='orders'` means "deliver to the queue named `orders`."
- **[[Terminology#AMQP|AMQP]]** — Advanced Message Queuing Protocol, the wire protocol RabbitMQ speaks on port **5672**. pika is a Python AMQP client.
- **Channel** — a lightweight virtual connection multiplexed over one TCP connection. All pika calls (`queue_declare`, `basic_publish`…) go through a `channel`.
- **Acknowledgement (ack)** — the consumer telling RabbitMQ "I finished this one, delete it." Until it's acked (or auto-acked), RabbitMQ keeps the message and will redeliver it if the consumer dies.

### Queues vs Exchanges (the one that trips everyone up)

| | **Queue** | **Exchange** |
|---|---|---|
| Job | **Stores** messages | **Routes** messages |
| Analogy | The ticket rail | The post office sorting desk |
| Consumers read from | ✅ Queues | ❌ Never exchanges |
| Producers publish to | ❌ Never directly | ✅ Exchanges |
| In our labs | `orders` | `''` (default) → routes by routing key |

> [!warning] You publish to an exchange, not a queue
> `channel.basic_publish(exchange='', routing_key='orders', ...)` looks like it publishes "to the orders queue," but really it publishes to the **default exchange** with routing key `orders`, and *that* exchange forwards it to the matching queue. Subtle, but it's why the `curl` example targets `amq.default` and not a queue.

### Competing consumers (work queues)

Start **N identical consumers** on the **same queue** and RabbitMQ **round-robins** messages across them. No code change, no coordinator — just launch more processes. This is the "work queue" pattern and it's how you scale throughput horizontally. `consumer-multi.py` gives each instance a random name (`consumer-ab12cd`) so you can *see* which worker grabbed which message.

---

## 🛠️ Guided walkthrough — mini-lab

> [!example] Watch a queue absorb a rush and split work across workers
> You'll need `pika` installed: `pip install pika`. Run each consumer in its **own terminal**.

1. **Start RabbitMQ:**
   ```bash
   docker compose up -d
   ```
   Wait ~10s for it to boot.

2. **Open the management UI:** browse to `http://localhost:15672`, log in `guest` / `guest`. Click the **Queues** tab (it'll be empty for now).

3. **Fire the producer** (Terminal 1):
   ```bash
   python producer.py
   ```
   Expected — one line per second:
   ```
   Sent: Order #1
   Sent: Order #2
   ...
   Sent: Order #20
   ```
   Flip to the UI **Queues** tab → an `orders` queue appears with **Ready** count climbing. That's the buffer filling. 📈

4. **Start ONE consumer** (Terminal 2, while the producer still runs):
   ```bash
   python consumer.py
   ```
   Expected:
   ```
   Waiting for messages. Press CTRL+C to exit.
   Received: Order #1
   Received: Order #2
   ...
   ```
   The **Ready** count in the UI drains as it catches up.

5. **Now the competing-consumers demo.** Stop that consumer (Ctrl+C). Open **two** terminals and run `consumer-multi.py` in **each**:
   ```bash
   python consumer-multi.py     # Terminal A
   python consumer-multi.py     # Terminal B
   ```
   Each prints its identity, e.g. `Consumer started: consumer-4f9a1c`.

6. **Re-run the producer** (Terminal 1): `python producer.py`. Watch A and B:
   ```
   # Terminal A
   [consumer-4f9a1c] Received: Order #1
   [consumer-4f9a1c] Received: Order #3
   # Terminal B
   [consumer-9b2e07] Received: Order #2
   [consumer-9b2e07] Received: Order #4
   ```
   RabbitMQ **round-robins** odd/even-ish across the two workers. 🎉 In the UI **Queues → orders**, the **Consumers** column now shows **2**.

7. **Publish from curl too** (any terminal), and watch it land in a consumer:
   ```bash
   curl -u guest:guest -H "content-type:application/json" -X POST -d '{
     "properties": {}, "routing_key": "orders",
     "payload": "Order created from curl", "payload_encoding": "string"
   }' http://localhost:15672/api/exchanges/%2F/amq.default/publish
   ```
   One of your consumers prints `Received: Order created from curl`.

---

## 🔬 Drills (earn XP)

- [ ] **(10 XP)** Bring the broker up with `docker compose up -d` and confirm both ports (`5672`, `15672`) are mapped via `docker compose ps`. **Done when:** you can log into the UI at 15672 with guest/guest.
- [ ] **(15 XP)** Run `producer.py` alone (no consumers) and watch the `orders` queue **Ready** count grow to 20 in the UI. **Done when:** you've screenshotted queue depth = 20 with 0 consumers.
- [ ] **(15 XP)** Start `consumer.py` and watch the queue **drain** in real time in the UI. **Done when:** Ready count returns to 0.
- [ ] **(20 XP)** Run **two** `consumer-multi.py` instances, then `producer.py`, and confirm messages **split** across the two named consumers. **Done when:** each terminal shows roughly half the orders and the UI **Consumers** column reads 2.
- [ ] **(20 XP)** Publish a message with the `curl` HTTP-API command and have it appear in a running consumer. **Done when:** a consumer prints `Received: Order created from curl`.
- [ ] **(25 XP)** Scale to **three** consumers and eyeball the load-balancing shift vs two. **Done when:** you can describe how per-worker share dropped from ~1/2 to ~1/3.
- [ ] **(25 XP)** In `consumer-multi.py`, `time.sleep(2)` simulates slow work. Explain *why* you'd add `basic_qos(prefetch_count=1)` here. **Done when:** you can state the fair-dispatch reason in one sentence.

---

## 🧗 Extra credit — beyond class *(addition)*

> [!example] 🆕 These drills are an **addition** — not covered in the class materials
> They're the next skill up from what class taught, chosen because you'll meet them in real work (and in the [[SkyWatch Capstone]]). Higher XP, higher payoff.

- [ ] **Survive a broker restart (30 XP)** — class queues were transient. Declare the queue `durable=True` and publish with `delivery_mode=2`, then `docker compose restart rabbitmq` mid-stream. **Done when:** unconsumed messages are still there after the restart (and you know BOTH flags were needed).
- [ ] **Manual ack + prefetch (25 XP)** — switch the consumer to `auto_ack=False` + `basic_qos(prefetch_count=1)`, and kill a worker mid-message. **Done when:** the unacked message gets redelivered to the other worker instead of vanishing.
- [ ] **Dead-letter queue (35 XP)** — declare a queue with `x-dead-letter-exchange` and reject a message with `basic_nack(requeue=False)`. **Done when:** the poison message lands in the DLQ instead of looping forever.

## 🔎 Learn to fish — find it yourself (don't just copy)

> [!tip] Beat "monkey-see-monkey-do"
> The real skill isn't memorizing commands — it's finding the right one **fast**. Pros do this all day. Build the reflex:
> 1. **Ask the tool first:** `<tool> --help`, `<tool> <subcommand> --help`, `man <tool>`.
> 2. **Official docs = source of truth** (below) — not random blogs or old Stack Overflow answers.
> 3. **Search smart:** `<what you want> site:<official-docs-domain>`; add the tool's version if behaviour changed between releases.
> 4. **Read the WHOLE error message** — it almost always names the missing flag or the fix.
> 5. `tldr <command>` gives real-world examples (install a `tldr` client — it's the friendly `man`).

**📚 Docs & references**
- [RabbitMQ tutorials](https://www.rabbitmq.com/tutorials)
- [pika (Python client) docs](https://pika.readthedocs.io/en/stable/)
- [Management HTTP API](https://www.rabbitmq.com/docs/management#http-api)

**⚡ Built-in help — try these BEFORE searching**
```bash
python -c "import pika; help(pika.BlockingConnection)"   # inspect the client
# Explore the Management UI at http://localhost:15672 — every action shows the underlying API
# The RabbitMQ tutorials (1-6) cover work queues, pub/sub, RPC with runnable code
```

> [!example] Power move for this class
> The 6 official RabbitMQ tutorials each come with full Python (pika) code — work through tutorial 2 (Work Queues) to truly get competing consumers, instead of copying snippets.

## 🧪 Self-check quiz

> [!question]- What problem does putting a queue between producer and consumer actually solve?
> **Decoupling + buffering + scaling.** The producer doesn't block waiting for the consumer; if consumers are slow or down, the queue holds the backlog (nothing is lost); and you can add more consumers to drain faster without changing the producer.

> [!question]- Ports 5672 and 15672 — which is which?
> **5672 = AMQP** (the binary protocol pika uses to publish/consume). **15672 = the management UI + HTTP API** (browser + curl). The `-management` image tag is what enables 15672.

> [!question]- In `basic_publish(exchange='', routing_key='orders')`, where does the message actually go, and how?
> To the **default exchange** (`''`), which routes it to the queue whose **name equals the routing key** — so the queue named `orders`. You never publish straight to a queue; you always go through an exchange.

> [!question]- Why call `queue_declare` in BOTH the producer and the consumer?
> It's **idempotent** — it creates the queue only if it doesn't exist. Declaring in both means whichever process starts first guarantees the queue exists, so publishing/consuming never fails on a missing queue.

> [!question]- What's the difference between `basic_ack` (consumer.py) and `auto_ack=True` (consumer-multi.py)?
> With **manual ack**, RabbitMQ keeps the message until the consumer confirms success — if the worker crashes mid-processing, the message is **redelivered**. With **auto-ack**, RabbitMQ deletes the message the instant it's delivered — faster, but a crash **loses** that message.

> [!question]- You run two consumers on `orders`. Producer sends 20 messages. Roughly how are they distributed?
> RabbitMQ **round-robins**, so ~10 each. With `prefetch_count=1` (fair dispatch), a slower worker automatically gets fewer, keeping throughput balanced.

---

## 🃏 Flashcards
#flashcards/devops

> [!tip] Two ways to study these
> **Flip a card:** click any ❓ below to reveal the answer. **Spaced repetition:** click the 🃏 ribbon icon (or `Ctrl+P → Spaced Repetition: Review flashcards`) and review the **devops** deck — the collapsed deck at the bottom feeds it.

> [!question]- Message broker
> A server (like RabbitMQ) that sits between services, storing and routing messages from producers to consumers.

> [!question]- Producer
> Code that publishes/sends messages to an exchange.

> [!question]- Consumer
> Code that subscribes to a queue and processes messages as they arrive.

> [!question]- Queue
> An ordered buffer that holds messages until a consumer takes them.

> [!question]- Exchange
> The router that decides which queue(s) a published message goes to; you always publish to an exchange.

> [!question]- Default exchange
> The empty-string exchange `''` (`amq.default`) that routes a message to the queue named by its routing key.

> [!question]- Routing key
> The address label on a message; with the default exchange it must equal the target queue's name.

> [!question]- AMQP
> The messaging wire protocol RabbitMQ speaks, on port 5672.

> [!question]- Port 5672
> RabbitMQ's AMQP port (pika connects here).

> [!question]- Port 15672
> RabbitMQ's management UI + HTTP API port (browser/curl, guest/guest).

> [!question]- queue_declare
> pika call that idempotently creates a queue if it doesn't already exist.

> [!question]- basic_publish
> pika call that sends a message to an exchange with a routing key.

> [!question]- basic_consume
> pika call that registers a callback to receive messages from a queue.

> [!question]- basic_ack
> Consumer's confirmation that a message was handled, so RabbitMQ can delete it.

> [!question]- auto_ack=True
> Ack a message on delivery instead of after processing — faster but loses messages on crash.

> [!question]- Competing consumers
> Multiple consumers on one queue; RabbitMQ round-robins messages to scale work horizontally.

> [!question]- prefetch_count=1
> Fair-dispatch setting — don't send a worker a new message until it acks the current one.

> [!question]- %2F in the HTTP API
> URL-encoded `/`, naming the default virtual host in RabbitMQ's HTTP API path.

> [!question]- rabbitmq:3-management
> The Docker image that bundles RabbitMQ with the web management UI enabled.

> [!srdeck]- 🔁 Raw review deck — the plugin reads this (collapsed on purpose; looks like text by design)
> #flashcards/devops
> Message broker::A server (like RabbitMQ) that sits between services, storing and routing messages from producers to consumers.
> Producer::Code that publishes/sends messages to an exchange.
> Consumer::Code that subscribes to a queue and processes messages as they arrive.
> Queue::An ordered buffer that holds messages until a consumer takes them.
> Exchange::The router that decides which queue(s) a published message goes to; you always publish to an exchange.
> Default exchange::The empty-string exchange `''` (`amq.default`) that routes a message to the queue named by its routing key.
> Routing key::The address label on a message; with the default exchange it must equal the target queue's name.
> AMQP::The messaging wire protocol RabbitMQ speaks, on port 5672.
> Port 5672::RabbitMQ's AMQP port (pika connects here).
> Port 15672::RabbitMQ's management UI + HTTP API port (browser/curl, guest/guest).
> queue_declare::pika call that idempotently creates a queue if it doesn't already exist.
> basic_publish::pika call that sends a message to an exchange with a routing key.
> basic_consume::pika call that registers a callback to receive messages from a queue.
> basic_ack::Consumer's confirmation that a message was handled, so RabbitMQ can delete it.
> auto_ack=True::Ack a message on delivery instead of after processing — faster but loses messages on crash.
> Competing consumers::Multiple consumers on one queue; RabbitMQ round-robins messages to scale work horizontally.
> prefetch_count=1::Fair-dispatch setting — don't send a worker a new message until it acks the current one.
> %2F in the HTTP API::URL-encoded `/`, naming the default virtual host in RabbitMQ's HTTP API path.
> rabbitmq:3-management::The Docker image that bundles RabbitMQ with the web management UI enabled.

## ⚠️ Gotchas

> [!warning] Common traps
> - **`guest`/`guest` only works from `localhost`.** RabbitMQ blocks the default guest user over the network by design. Fine for our labs; create a real user for anything remote.
> - **Declare the queue before publishing.** Publishing to a queue that was never declared silently goes nowhere with the default exchange. That's why both producer and consumer call `queue_declare`.
> - **auto-ack loses messages.** `consumer-multi.py` uses `auto_ack=True` — if it crashes during its `time.sleep(2)`, that message is gone. Use manual `basic_ack` (like `consumer.py`) when the work is important.
> - **`%2F`, not `/`, in the HTTP API path.** The vhost is a path segment and must be URL-encoded. `.../api/exchanges/%2F/amq.default/publish`. A literal `/` breaks the route.
> - **You publish to an exchange, not a queue.** `exchange=''` is the default exchange doing the queue-name → routing magic. Forgetting this makes custom routing confusing later.
> - **Give the broker time to boot.** Connecting immediately after `docker compose up -d` can fail with a connection-refused; wait a few seconds.
> - **Without `prefetch_count=1`, dispatch isn't fair.** RabbitMQ front-loads messages to whichever consumer is ready, so a slow worker can hoard a batch. Set QoS for even load-balancing.

---

## 🏆 Boss challenge

> [!example] Prove load balancing end-to-end 🥊
> Build and demonstrate a working **producer → queue → 2 workers** pipeline:
> 1. `docker compose up -d` the `rabbitmq:3-management` broker.
> 2. Launch **two** `consumer-multi.py` workers (note their random names).
> 3. Run `producer.py` to fire 20 orders.
> 4. **Prove** the split: capture both terminals showing each worker got roughly half the orders, and show the UI **Queues → orders** with **Consumers = 2**.
> 5. **Bonus (+50 XP):** kill one worker mid-run and show the survivor picking up the remaining messages — resilience in action.
>
> **Win condition:** you can point at the two terminals + UI and explain, out loud, exactly how RabbitMQ decoupled the producer and round-robined work across the consumers.

---

> [!success] Class 13 cleared when you can…
> - Explain **why** a queue beats a direct call (decouple, buffer, scale) using the kitchen-ticket analogy
> - Publish and consume with pika (`queue_declare` → `basic_publish` → `basic_consume` → `basic_ack`)
> - Run multiple consumers and **demonstrate** competing-consumer load balancing in the UI
> - Publish via the HTTP API with `curl` and explain the `%2F` vhost and `amq.default`
> - Contrast **ack vs auto-ack** and say when each loses (or saves) messages
>
> 🎖️ **Badge: Queue Conductor**

---

> [!cheatsheet]- 🔒 Command cheat-sheet — try the drills FIRST (expanding = peeking 👀)
> ⚠️ **Wait!** Expanding this is basically giving up. Did you really attempt the drill from memory? If not, collapse me and try again — the struggle is what makes it stick. Still stuck? Fine, the commands are below. 👇
>
> **Start the broker (with the management UI):**
>
> ```bash
> # docker-compose.yaml pulls rabbitmq:3-management, maps 5672 (AMQP) + 15672 (UI)
> docker compose up -d
>
> # Confirm it's up, then open the UI
> docker compose ps
> # → http://localhost:15672   login: guest / guest
> ```
>
> **Publish a message with plain HTTP — no Python needed (from `commands`):**
>
> ```bash
> curl -u guest:guest -H "content-type:application/json" -X POST -d '{
>   "properties": {},
>   "routing_key": "orders",
>   "payload": "Order created from curl",
>   "payload_encoding": "string"
> }' http://localhost:15672/api/exchanges/%2F/amq.default/publish
> ```
>
> > [!warning] `%2F` is the default vhost, URL-encoded
> > The path segment `%2F` is `/` URL-encoded — it names the **default virtual host** `/`. `amq.default` is the default exchange (same one as `exchange=''` in pika). So this curl is the HTTP twin of `producer.py`'s publish.
>
> **Producer — send (annotated, from `producer.py`):**
>
> ```python
> import pika, time
>
> # guest/guest only works from localhost (see Gotchas)
> credentials = pika.PlainCredentials('guest', 'guest')
> params = pika.ConnectionParameters(host='localhost', credentials=credentials)
>
> connection = pika.BlockingConnection(params)   # open AMQP connection (port 5672)
> channel = connection.channel()                 # get a channel
> queue_name = 'orders'
> channel.queue_declare(queue=queue_name)        # idempotent: create queue if absent
>
> for i in range(1, 21):
>     message = f"Order #{i}"
>     channel.basic_publish(
>         exchange='',                # default exchange...
>         routing_key=queue_name,     # ...routes by name → queue "orders"
>         body=message,
>     )
>     print(f"Sent: {message}")
>     time.sleep(1)                   # 1 msg/sec so you can watch it live
> connection.close()
> ```
>
> **Consumer — receive (annotated, from `consumer.py`):**
>
> ```python
> import pika
>
> credentials = pika.PlainCredentials('guest', 'guest')
> params = pika.ConnectionParameters(host='localhost', credentials=credentials)
> connection = pika.BlockingConnection(params)
> channel = connection.channel()
> channel.queue_declare(queue='orders')          # declare again — safe & idempotent
>
> def callback(ch, method, properties, body):    # runs per delivered message
>     print(f"Received: {body.decode()}")
>     ch.basic_ack(delivery_tag=method.delivery_tag)   # "done — delete it"
>
> channel.basic_qos(prefetch_count=1)   # fair dispatch: one unacked msg at a time
> channel.basic_consume(queue='orders', on_message_callback=callback)
>
> print("Waiting for messages. Press CTRL+C to exit.")
> channel.start_consuming()             # blocks forever, calling callback per msg
> ```
>
> > [!tip] `basic_qos(prefetch_count=1)` = fair dispatch
> > Without it, RabbitMQ shoves messages at a consumer as fast as it can, so a busy worker gets flooded while an idle one starves. `prefetch_count=1` means "don't give me a new message until I've acked the current one" → work spreads evenly across slow/fast workers.
>
> ---

## 🔗 Related

- [[Class 02 - Docker Networking and Images]] — the compose/ports/image knowledge that runs the broker
- [[SkyWatch Capstone]] — where async messaging fits a real pipeline
- [[Terminology]] — glossary of all bolded terms
- [[DevOps Experts — MOC]] — course map
