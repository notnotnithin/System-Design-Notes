# Distributed Messaging Queue (HLD #16 – Lecture 53)

<!-- nav -->
⬅ [15 High Availability Architecture](15_High_Availability_Architecture_Notes.md) · [Index](00_HLD_Index.md) · [17 Proxy vs Reverse Proxy](17_Proxy_vs_Reverse_Proxy_Notes.md) ➡
<!-- nav -->

Source (handwritten, 10 pages): https://drive.google.com/file/d/1kfnXbRlobCeK-U9kT56nyXI-4VY9EBAo/view?usp=drive_link

## Questions this note answers
1. What is a messaging queue? Why is it needed & its advantages?
2. What is Point-to-Point (P2P) and Pub/Sub messaging?
3. How does a messaging queue work? – **Kafka** and **RabbitMQ**
4. What happens when the queue size limit is reached? → **we add a different broker.**
5. What happens to messages when the queue goes down? → **we make a follower the leader (replication).**
6. What happens when a consumer goes down? → **another consumer takes over.**
7. What happens when a consumer is not able to process a message?
8. How does retry work? Different ways to retry?
9. How does a distributed messaging queue work?
10. What is a Dead Letter Queue?

---

<!-- simple-explanation -->
## In simple words – messaging queues, Kafka and RabbitMQ

### What is a messaging queue?
A **producer** puts a message into a **queue**, and a **consumer** takes messages out and processes them. The producer and consumer never talk to each other directly. It is like dropping letters in a post box: you don't wait for the receiver to read them.

### Why do we need it?
Example: a user clicks **Buy**. We should confirm the order immediately, but sending the email/SMS can happen a moment later. So the order service drops a "send notification" message in a queue and returns at once.
1. **Faster responses** – the user is not blocked by slow side tasks.
2. **Retries** – if the consumer is down or fails, the message stays in the queue and can be retried.
3. **Pace matching** – three services produce 10, 20 and 30 messages/second, but the notification service can only handle 15/second. The queue absorbs the difference and the consumer works at its own speed.
4. **Absorbing bursts** – e.g. thousands of cabs send GPS locations every 10 seconds; the queue smooths the flow for the dashboard to process.

### Two messaging styles
- **Point-to-Point:** each message is processed by **exactly one** consumer (even if several consumers listen). Use it for tasks to be done once.
- **Publish/Subscribe:** a message is **broadcast to many queues**, so **every subscriber** gets and processes its own copy.

### Kafka – the main ideas
- **Producer** sends messages; **consumer** reads them.
- **Topic** – a named category of messages (like a channel). A topic is divided into **partitions**; each partition is an ordered log, and each message has an **offset** (its position number: 0, 1, 2…).
- **Broker** – one Kafka server. A **cluster** is a group of brokers. **ZooKeeper** keeps metadata, such as which partition lives on which broker.
- **Message** has an optional **key** and a **value**. `hash(key) → partition`, so messages with the same key always go to the same partition (this preserves their order).
- **Consumer group** – consumers that share the work. **Within one group, a partition is read by only one consumer.** If a second application needs the same data, it uses a different group.
- **Committed offset** – how far a consumer group has read. After a crash, the consumer resumes from there instead of starting over.
- **Leader and followers (replication)** – each partition has a **leader** (handles all reads and writes) and **followers** (copies on other brokers). If the leader's broker dies, a follower is promoted. This protects against message loss.
- Kafka is **pull-based**: consumers ask for messages when ready.

### When a message can't be processed
Suppose offsets 0–6 are fine but offset 7 has a bug. The consumer **retries** it (e.g. 3 times). If it keeps failing, the message is moved to a **Dead Letter Queue (DLQ)**, a side queue for bad messages, so the consumer can continue with the rest, and engineers can inspect the failed ones later.

### RabbitMQ – the main ideas
- Producer → **exchange** → **queues** → consumers. The exchange uses a **routing key** and each queue's **binding key** to decide where to send a message.
  - **Fan-out:** copy to all bound queues.
  - **Direct:** exact key match.
  - **Topic:** wildcard match (like `india.*`).
- RabbitMQ is **push-based**: it pushes messages to consumers. A failed message is **requeued** and retried, and eventually goes to the DLQ.

### Handling failures at scale
- Queue full → **add more brokers/partitions**.
- Broker down → **a follower becomes leader**.
- Consumer down → **another consumer in the group takes over its partitions**.
<!-- simple-explanation -->


## 1. Messaging queue & why it's needed
**Messaging queue:** a **producer** produces a message; the message goes into a particular **queue**; a **consumer** reads the messages.
`Producer → [ queue ] → Consumer`

### Use case 1 – Asynchronous nature
E-commerce app (producer): the user clicks **Buy** → we must send a notification/mail – **but the user shouldn't have to actually wait for that notification.** The producer drops a message on the async queue; the **"Send notification" consumer** processes it (if the consumer is down, the message stays in the queue).

**Advantages**
1. **Latency improves** (the user isn't blocked).
2. **Retry capabilities.**
3. **Pace matching** – E-commerce produces 10 msg/s, Inventory management 20 msg/s and App XYZ 30 msg/s, but the notification service can handle only **15 msg/s**. All three put messages into one queue and the consumer **consumes them at its own pace.**
4. **Smoothing bursts** – e.g. cab GPS: every cab sends its car ID + current location every 10 s; the consumer application builds a **dashboard** of car locations from the queue and processes at its own rate.

## 2. Point-to-Point (P2P) vs Pub/Sub
- **P2P:** publisher → queue → **only one consumer** processes each message. A message is **consumed/processed only once** – by either Consumer 1 or Consumer 2, **but only once.**
- **Pub/Sub:** publisher → **exchange (with logic)** → **multiple queues (Q1, Q2)** → each consumer. Whenever a message comes in, based on the logic the exchange **puts/broadcasts the message to all the queues**, and **both consumers consume and process the message.**

---

## 3. How a messaging queue works – ① Kafka

### Components
1. **Producer** – sends messages.
2. **Consumer** – **reads from a partition.**
3. **Consumer Group** – each consumer is part of a consumer group.
4. **Topic** – like a category/channel where data is stored and organised (data is stored on partitions only).
5. **Partition** – a small part of a topic; helps in **distributing and processing data efficiently**; partitions can have different lengths.
6. **Offset** – the position of a message inside a partition (like an index in a queue; each partition has its own offsets 0,1,2,…).
7. **Broker** – a **Kafka server**; the producer talks to the broker.
8. **Cluster** – **a group of brokers** (each running on a different machine/node).
9. **ZooKeeper** – helps the brokers **interact with each other** and keeps internal metadata (e.g. which partition of Topic 1 lives on which broker).

**Rule:** inside one consumer group, **one partition is consumed by only one consumer** – e.g. Topic A has partitions P0 and P1; if Consumer 1 uses P0, Consumer 2 takes P1, but Consumer 2 can't also take P0. If P0 must be read by another consumer, that consumer must be in a **different consumer group.**

### How a message flows
- **Message has:** **Key** (string/id, optional), **Value** (actual message), **Partition** (empty unless specified), **Topic** (topic name).
- The producer sends `key: 123456…` → **hash(key) → partition** (e.g. P1). Messages with the same key always land in the same partition.
- Consumer 1 (Consumer Group A) reads Topic A / Partition P0 and has a **committed offset = 3** (kept in Kafka/ZooKeeper per consumer-group+topic+partition) – **the offset tells how far the consumer has read**, so after a crash it resumes from there.

### Cluster, leader & followers (replication)
- Broker 1 has Topic 1 / partition P0 and Broker 2 has Topic 1 / partition P1 – **partitions can reside on different Kafka servers.**
- **What if a partition goes down – what happens to its messages?** → that's why we keep a **replica.**
- The partition that is **not a replica is the Leader**; **replicas are Followers.** (e.g. P0 leader on Broker 1 replicated to Broker 2; P1 leader on Broker 2 replicated to Broker 3.)
- **All reads/writes happen through the Leader.** **Whenever the leader goes down, a replica takes over and becomes the leader.**
- A **follower always follows the leader** – any new message that comes in, it takes and adds to its own copy, so it keeps syncing itself from the leader.

### 7. What happens when a consumer can't process a message?
Example: partition offsets 0…6 processed properly; **offset 7 = a buggy message** → the consumer polls it, fails to process.
- **Committed offset** stays at 6 (not moved to 7) so the message would be re-delivered.
- **Retry = 3** – every message is retried 3 times.
- After the retries are exhausted the message is moved to a **Failure / Dead Letter Queue (DLQ)** so the consumer can move on.

**RabbitMQ vs Kafka (delivery model):** **RabbitMQ = push-based approach**; **Kafka = pull-based approach** (consumers poll).

---

## 4. RabbitMQ
`Producer → Exchange → (binding/routing key) → Queue 1 / Queue 2 / Queue 3 → Consumer 1 / 2 / 3`
- Producer sends a message with a **routing key** (e.g. `india-123`); the exchange routes it to the queue(s) whose **binding key** matches (Q1 ↔ routing key 1 / `*123`, Q2 ↔ key 2, Q3 ↔ key 3). There can be multiple exchanges.

### Different types of exchange
1. **Fan-out exchange** – pushes the message **to all the queues associated with this exchange.**
2. **Direct exchange** – **exact match** between **message (routing) key and binding key.**
3. **Topic exchange** – can use **wildcards** (e.g. binding `*123` matches routing key `india-123`).

### Failure handling in RabbitMQ
If Consumer 1 reads a message but **can't process it**, the message is **re-queued to the back of the queue** (MSG1 requeue) and retried again; after repeated failures it goes to the **Dead Letter Queue.**

---

## Quick revision
- Queues give **async processing, retry, pace-matching, decoupling**.
- **P2P** = each message consumed once; **Pub/Sub** = broadcast to all subscribed queues.
- **Kafka:** topic → partitions (ordered, offsets) → consumer groups (1 partition ↔ 1 consumer per group); hash(key) → partition; **leader/follower replication** for fault tolerance; **committed offset** = progress; **pull-based**.
- **RabbitMQ:** producer → exchange (**fan-out / direct / topic**) → queues → consumers; **push-based**; failed messages requeued then **DLQ**.
- Scale by **adding brokers/partitions**; queue/broker failure → **follower promoted**; consumer failure → **another consumer in the group takes over**.
- Related: [Scale from 0 to Million](05_Scale_from_Zero_to_Million_Users_Notes.md), [Retry Pattern](38_Retry_Pattern_Notes.md).

---

<!-- nav -->
⬅ [15 High Availability Architecture](15_High_Availability_Architecture_Notes.md) · [Index](00_HLD_Index.md) · [17 Proxy vs Reverse Proxy](17_Proxy_vs_Reverse_Proxy_Notes.md) ➡
<!-- nav -->
