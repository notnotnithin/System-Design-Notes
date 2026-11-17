# Dual Write Problem – Designing Event-Driven Systems (HLD #33 – Lecture 70)

<!-- nav -->
⬅ [32 Common Attacks](32_Common_Attacks_Notes.md) · [Index](00_HLD_Index.md) · [34 Service Discovery](34_Service_Discovery_Notes.md) ➡
<!-- nav -->

Sources:
- Handwritten PDF (8 pages): https://drive.google.com/file/d/14wwKmtzMEgvctlQhiSrLU0gw9fhmMAGn/view?usp=drive_link
- OneNote (not readable automatically): https://1drv.ms/o/c/6b364628c29feb52/EpHzxZpvUj9ChPoZKABB6hYB3ThLz9J66-3csHIWs_G_ZA

---

<!-- simple-explanation -->
## In simple words – the dual write problem and the outbox pattern

### The problem
In microservices a service often has to do **two things** after a business action: **save data in its database** *and* **publish an event to a message broker (like Kafka)** so other services can react. For example, after creating a user, save the user and publish "UserCreated".

These are two **different systems**, so there is no single transaction covering both. Things can fail halfway:
- The event is published but the database save fails → others are told about something that doesn't exist.
- The database save works but publishing fails → data exists but nobody is told, so the rest of the system is out of sync.

This is the **dual write problem**: you must keep two systems consistent without a shared transaction. It appears everywhere in the SAGA pattern, where each service saves its own data and then notifies the next service.

### Why not just use 2PC (two-phase commit)?
2PC makes all participants "prepare", then "commit" together. It is unsuitable here because:
- Many message brokers **don't support** 2PC at all.
- It is **slow**, as everyone waits for everyone else.
- If the **coordinator crashes**, participants are stuck in limbo until it recovers.

### Solution 1 – Transactional Outbox pattern
Idea: **use only the database for the atomic part.**
1. In **one local database transaction**, write the business data (e.g. the User row) **and** the event into an extra table called the **outbox table**. Either both are saved or neither is.
2. A separate process, the **poller**, regularly reads new rows from the outbox table and **publishes them to the broker**.
3. Once published, the row is marked done or deleted.

Since the event is saved safely with the data, it can't be lost; if publishing fails, the poller just tries again.

**Things to handle (the trade-offs):**
- **Extra work and delay:** you must build the poller, and events are published slightly after the write.
- **Duplicates:** a poller may publish the same event more than once (at-least-once delivery). Kafka's `enable.idempotence=true` gives each producer an ID and sequence numbers so it drops duplicates from the *same* producer. But if **several pollers** pick the same event, they have different IDs, so **consumers must also be idempotent** (e.g. remember processed event IDs).
- **Ordering:** Kafka only guarantees order within one partition. If one entity's events (Created → Updated → Deleted) go to different partitions or are published by different pollers, a consumer could see them in the wrong order. Use the same message key for the same entity so they land in the same partition, or use a single poller/partition where ordering is critical.
- **Broker down:** retry with exponential backoff and jitter; after repeated failures, move the event to a failed-events table so it isn't lost.
- **Growing table:** delete rows after publishing, or clean up with a periodic batch job.

### Solution 2 – Listen-to-Yourself pattern
A variation: the service writes **only the event** (a single write). The poller publishes it, and the **service also consumes its own event** and only then updates its own database. This way there is just one write at the start, and all the real work is driven from the event.

### Solution 3 – Transaction log tailing (CDC)
Instead of polling an outbox table, a tool reads the **database's transaction log** and turns every committed change into an event. No polling and no extra writes by your code. (The source notes only name this one.)

### Takeaway
Don't write to two systems at once. Write to **one** (the database, atomically), and let a reliable second step deliver the event, then protect against duplicates and ordering problems. See also [Distributed Transactions / SAGA](20_Distributed_Transactions_Notes.md) and [Idempotent POST](14_Idempotent_POST_API_Notes.md).
<!-- simple-explanation -->


## 1. The Dual Write Problem
It occurs **when a component needs to persist a change in two different systems** – e.g. **a DB and a message broker like Kafka.**

We must make sure **both are in a consistent state** – there should **never be a case where the DB write succeeds but event publishing fails, or vice-versa.**

Where it shows up: the **SAGA pattern** → *dual-write problem*. In a saga `SA → SB → SC`, each service has its own DB (e.g. SA's DB: "Invoice created") and must also publish an event to the next service through a queue.

Two failure sequences (service → DB / message broker):
1. Message to broker ✔, then DB write ✗ → event published but data not saved.
2. DB write ✔, then publish to broker ✗ → data saved but nobody is told.

---

## 2. How to resolve it? First idea – 2PC (Two-Phase Commit)
- A **coordinator** sends a **Prepare** message to all participant services and waits for **Ack**.
- Once all Acks arrive it sends **Commit** and again waits for **Ack** – ensuring **all participant services properly committed the txn.**

### But – 2PC is **not a viable solution** for the dual-write problem. Why?
- Dual-write scenarios involve **heterogeneous systems (DB + message queue)**, and **some message queues / brokers do not support 2PC.**
- In a distributed system **2PC is slow (high latency)** because it provides **strong consistency by waiting for all participants to commit.**
- **If the 2PC coordinator crashes, participants may be stuck in an uncertain state till it recovers.**

---

## 3. Answer: Event-Driven Architecture patterns
1. **Transactional Outbox Pattern**
2. **Listen-to-Yourself Pattern**
3. **Transactional Log Tailing Pattern** *(named in the notes; the PDF doesn't detail it)*

### ① Transactional Outbox Pattern
- Instead of writing to both the DB **and** publishing an event, **write the event into a separate table called the OUTBOX table** – **in the same local DB transaction** as the business data.
- **Later, a separate process (poller) reads the Outbox table and publishes the event to the message broker asynchronously.**

Example – User Service:
```
Begin Txn
   Insert User            ┐
   Insert User Event      ├─ 1 Txn  (User table + UserEvent table in the same DB)
Commit Txn                ┘
Poller Service ── get events from UserEvent table ──► Publisher ── publish ──► Message Broker
```
Table design options: **one common event table** (UE, OE, IE – user/order/inventory events together) **or separate tables per event type** (UE table, OE table).
- The **Outbox table is regularly cleaned up** once messages are successfully published.

**Challenges**
1. **Extra effort** to write the poller service.
2. **Delay** between writing an event and publishing it to the broker.
3. **A poller may publish the same event multiple times** to the broker → **idempotency must be handled.**
   - Producer sets **`enable.idempotence=true`**.
   - Kafka then **assigns a producer ID (PID) to each producer** when it starts.
   - Every message from the producer now carries a **monotonically increasing sequence number.**
   - If Kafka sees a duplicate **(producer id + sequence no.)**, it **discards the event automatically.**
4. **Multiple pollers can pick the same events and cause duplication.**
   - `enable.idempotence=true` **won't help here** because **different producers (pollers) get different PIDs.**
   - So **idempotency must be handled on the consumer side too** – e.g. **store the unique IDs of processed events in a cache.**
5. **Events might be published out of order** if different pollers pick events in parallel.
   - Kafka **only guarantees order if there is only 1 producer and 1 partition.**
   - With **multiple producers and multiple partitions, ordering is not guaranteed.**
   - If ordering is a must with multiple producers/partitions → **Kafka Streams** can be used (it re-orders).
   - *Ordering issue example:* events for one entity – **Creation, Updation, Deletion** – may reach the notification service as *Deleted → Updated → Created* if they land in different partitions. **Message key → hash % no. of partitions** decides the partition (same key ⇒ same partition). **1 publisher → 1 partition = order is maintained** (4,3,2,1,0 stays in order in P0); with **more partitions** (P0: 2,1,0 and P1: 5,4,3) the consumer may interleave them.
6. **What if the poller can't publish to the message broker?** The event must not be lost:
   1. **Retry with exponential backoff and jitter.**
   2. After the retries, **move to a Failed-event table**, or – depending on how the outbox table is used (if a **status column** is maintained) – keep it in the same table.
7. **Outbox table can grow** – clean it regularly:
   1. **After every message/event is published successfully**, or
   2. **A batch job.**

### ② Listen-to-Yourself Pattern
- **A variation of the Transactional Outbox pattern.**
- The **application writes only the event** in the Outbox/event table (single write).
- The **poller fetches the event** from the table and **publishes it** to the message broker.
- **The application also listens to the same event** (via a consumer / "User Service daemon application") **and only now updates its own DB** (writes into the User table).

```
User Service ─ Begin Txn / Insert User Event / Commit Txn ─► DB (UserEvent table)
Poller ── get event ──► publish ──► Message Broker ──► Consumer
                                           └─► User Service Daemon application ── writes into DB (User table)
```

### ③ Transactional Log Tailing
Listed as the third pattern (read the DB's transaction log instead of polling a table); the PDF doesn't go into details.

---

## Quick revision
- **Dual write** = updating DB + broker separately → can end up inconsistent.
- **2PC isn't suitable** (brokers often don't support it, high latency, coordinator crash blocks).
- **Transactional Outbox:** write data + event in **one local DB txn**, a **poller** publishes later → **at-least-once**, so make producers/consumers **idempotent**; watch **ordering**, retries/failed-event table, and **cleanup**.
- **Listen-to-yourself:** write only the event, then consume your own event to update your DB.
- **Log tailing:** CDC from the DB transaction log.
- Related: [Distributed Transactions / SAGA](20_Distributed_Transactions_Notes.md), [Idempotent POST](14_Idempotent_POST_API_Notes.md).

---

<!-- nav -->
⬅ [32 Common Attacks](32_Common_Attacks_Notes.md) · [Index](00_HLD_Index.md) · [34 Service Discovery](34_Service_Discovery_Notes.md) ➡
<!-- nav -->
