# Scale from Zero to Million Users (HLD #5 – Lecture 43)

<!-- nav -->
⬅ [03 & 04 Microservices Design Patterns](03_04_Microservices_Design_Patterns_Notes.md) · [Index](00_HLD_Index.md) · [06 Consistent Hashing](06_Consistent_Hashing_Notes.md) ➡
<!-- nav -->

Source (handwritten, 9 pages): https://drive.google.com/file/d/16FIwh_zZ2nowjFh_vzeUq7nAwoaQct6f/view?usp=drive_link

## Roadmap – the 9 steps
1. Single server
2. Application & DB server separation
3. Load balancer + multiple app servers
4. Database replication
5. Cache
6. CDN
7. Data centre
8. Messaging queue
9. Database scaling

---

<!-- simple-explanation -->
## In simple words – scaling from one user to a million

Think of it as a story: you start with one tiny server and, every time something becomes a bottleneck, you add one new building block. Here are the nine steps and the problem each one solves.

**1. Single server.** Your website, business logic and database all run on one machine. Simple and cheap, but if it's busy or crashes, everything stops.

**2. Separate the application server from the database server.** Run the app on one machine and the database on another. Now each can be upgraded and scaled independently, and a heavy database query no longer steals CPU from the web app.

**3. Add a load balancer and several app servers.** One app server can only handle so many users. So run several copies and put a **load balancer** in front. Clients talk only to the load balancer, which forwards each request to one of the servers. Bonus: the servers use **private IPs**, so they aren't directly exposed to the Internet (better security).

**4. Replicate the database (master–slave).** A single database is a single point of failure and also a bottleneck. Keep one **master** that accepts all **writes** and one or more **slaves** (replicas) that copy the master and serve **reads**. If a slave dies, others still answer. If the master dies, one slave is **promoted** to be the new master.

**5. Add a cache (e.g. Redis).** Calling the database over the network is slow and expensive. Put a fast in-memory cache next to the app servers and **always check it first**. Found = **cache hit** (fast). Not found = **cache miss**: read from the DB and save the value in the cache for next time. Every entry has a **TTL (time to live)**, after which it expires so that the cache doesn't serve outdated data forever.

**6. Add a CDN (Content Delivery Network).** If your servers are in India, a user in the USA has a long network trip, so pages feel slow. A CDN keeps copies of **static files** (images, videos, CSS, HTML) on servers around the world. The user is served from the nearest CDN node; only if it doesn't have the file does it go to another node and finally to your origin server. Result: lower latency, less load on your servers and lower cost.

**7. Multiple data centres.** Even a whole data centre can fail (power cut, flood). Run a second one (e.g. India and USA) and let the load balancer send traffic to whichever is healthy. The catch: the second data centre needs the same data, so the databases must **replicate between data centres**.

**8. Message queue (RabbitMQ, Kafka).** Some work doesn't need to happen while the user waits, e.g. sending an email after a purchase. The app (**producer**) drops a message in a queue, and a separate worker (**consumer**) handles it later. This makes the system **asynchronous**, decoupled and better at absorbing bursts.
- In RabbitMQ the producer sends to an **exchange**, which routes messages to queues:
  - **Direct** – goes to the queue whose binding key exactly matches the routing key.
  - **Fan-out** – copied to every bound queue.
  - **Topic** – matched with wildcard patterns.

**9. Scale the database itself.**
- **Vertical scaling:** give the DB machine more CPU and RAM. Easy, but there is always a ceiling.
- **Horizontal scaling (sharding):** split the data across many machines.
  - *Horizontal sharding* splits a table by **rows** (rows 1–100 on shard 1, 101–200 on shard 2, or names A–P vs Q–Z).
  - *Vertical sharding* splits by **columns**.
- Sharding has costs: a shard can become **hot** and overfill (fix with re-sharding / [consistent hashing](06_Consistent_Hashing_Notes.md)), and you can **no longer JOIN across shards** (fix by denormalizing, i.e. storing the needed data together).

### The big picture
Load balancer → many app servers → cache → replicated, sharded databases, with a CDN for static content, a queue for background work, and several data centres for safety. Each piece fixes exactly one problem of the previous design.
<!-- simple-explanation -->


## 1. Single server
`Client (web, mobile) ⇄ Server (Application + DB on the same machine)`.

## 2. Application & DB server separation
`Client ⇄ Application server (mid tier) ⇄ DB server (data / DB tier)` – the app and the database now run on separate machines so each can be scaled/managed independently.

## 3. Load balancer + multiple app servers
```
 Client ⇄ LOAD BALANCER ──► App S1 | App S2 | App S3   (mid tier, private IPs)
                                   ⇅
                               DB server (data / DB tier)
```
- The **client can only talk to the load balancer**; the LB decides **to whom to forward the request.**
- The LB brings **security and privacy** – **the LB and app servers talk only over private IPs.**
- The LB can **equally divide the traffic/requests.**

## 4. Database replication (to handle the failure issue)
`App servers → Master DB (writes) ⇄ replication ⇄ Slave DB(s) (reads)`
- **Write requests go to the master; read requests go to the slave(s).**
- If one slave is down (damaged), **another DB node is up and should handle the request.**
- If the **master DB fails, any slave DB can get promoted to master.**

## 5. Cache (performance improvement)
- Every time the application server talks to the DB it is a **network call – it takes a lot of time; an expensive call.**
- Add a cache (e.g. **Redis**) next to the app servers.
- **Always check the cache before going to the DB:** data present in cache = **cache hit**; not present = **cache miss** (then read from DB and populate the cache).
- **TTL (Time To Live)** – how long an entry stays in cache; chosen **depending upon the application requirement.**

## 6. CDN (Content Delivery Network)
- **CDN does caching – but not everything that does caching is a CDN.**
- Problem: an origin **data centre in India** is far from users: India user ≈ **1 ms**, SA ≈ **2 ms**, USA ≈ **3 ms**, Japan ≈ **4 ms** (latency grows with distance → **latency issue**) – a **CDN solves this.**
- **CDN does caching of static data** (HTML / video / CSS / images). CDN nodes sit near users (USA, India, France, Japan…).
- **Request flow:** the user's request first goes to **their own (nearest) CDN node**; if the data is there → done; if not present it goes to **another nearest CDN**; if still not present, it goes to the **original server** (and is cached).
- Benefits: **performance increases, security enhanced, cost cutting.**

## 7. Data centre
```
 Client ⇄ CDN
   │
   LB ──► India DC: App S1-S3, Cache, MQ, Master DB ⇄ Slave DB
      └─► USA  DC: App S1-S3, Cache, MQ, Master DB ⇄ Slave DB     (replication between the two DCs)
```
**Advantage:** if the India data centre is down, the **load balancer can fulfil the requirement from the USA data centre.**
**Problem:** when India is down and requests are served by USA, **only USA data will be found** (India's data isn't there). **Solution: replication** between data centres.

## 8. Messaging queue (RabbitMQ, Kafka)
- Brings **asynchronous nature** in the codebase: **Producer → Messaging Queue (queue/topic) → Subscriber (consumer)**; failed messages can go to a **requeue**.

### Terminology (RabbitMQ)
`Producer ──(routing key)──► Exchange ──(binding)──► Queue Q1 / Q2 / … ──► Subscriber Su1 / Su2 / Su3`

Exchange types:
1. **Direct** – **routing key == binding key** (message goes to the queue whose binding key matches exactly).
2. **Fan-out** – the exchange **copies the message to all bound queues** (Q1, Q2, Q3, Q4).
3. **Topic** – routing by **wildcard** pattern (e.g. `*`) matching.

## 9. Database scaling
Two types: **① Vertical** and **② Horizontal.**

### ① Vertical scaling
Increase the capacity of the existing DB server – **more CPU and RAM**, because when more requests come, CPU and RAM take care.
**Disadvantage:** **there is always a limit** – you cannot increase CPU/RAM beyond a point.

### ② Horizontal scaling
**Add more nodes.** Implemented through **SHARDING**, which is of two kinds:

**Horizontal sharding** – a table is **divided into multiple tables based on rows**: e.g. table T1 with rows 1–100 → Shard T1 (R1–R100 …) and Shard T2 (R101–…); another example by name: A–P → ST1, Q–Z → ST2.
**Vertical sharding** – a table is **divided into multiple tables based on columns**: e.g. columns C1–C10 → table with C1–C5 and table with C6–C10.

### Drawbacks of sharding
1. **Hot shard / re-sharding:** if the "A" set is very large and "A to P" is going on ST1, ST1 **immediately fills up** → we **shard again** (ST1(1), ST1(2)…). In sharding, **a "tree" of shards should be possible.** *Tree can solve with **consistent hashing**.* (See [Consistent Hashing](06_Consistent_Hashing_Notes.md).)
2. **Joins:** once rows are divided, **we can't JOIN** across shards – we can solve it with **denormalization.**

---

## Quick revision
| Step | Component | Why |
|---|---|---|
| 2 | Separate app & DB tiers | independent scaling |
| 3 | LB + many app servers | spread load, hide servers (private IPs) |
| 4 | Master–slave replication | failure handling, read scaling |
| 5 | Cache (Redis, TTL) | avoid expensive DB calls |
| 6 | CDN | cache static content near users, lower latency |
| 7 | Multiple data centres | availability / geo failover (needs replication) |
| 8 | Message queue | async processing, decoupling |
| 9 | DB scaling | vertical (limited) vs horizontal (sharding: by rows/columns; consistent hashing; denormalize for joins) |

---

## Extra from the 36-page handwritten compendium (pages 14–18)
These pages are in the same PDF as notes #1, #7, #9 and #11 (https://drive.google.com/file/d/1iegvlzgcLJtcGvbjJbPqF4hZaB5u_nrh/view?usp=drive_link); kept here because they belong to this topic.


**Database scaling (page 18):** **Vertical** – increase hardware (RAM, ROM, processor). **Horizontal** – add multiple instances; implementation is **sharding**:
- **Horizontal sharding** – divide into multiple tables on the basis of **rows** (T1 → A–P, T2 → Q–Z).
- **Vertical sharding** – divide into multiple tables by **columns** (T1 → C1–C6, T2 → C7–C10).
- It can become **hierarchical**, and **joining isn't possible** → remove redundancy/dependency and **make the table normalized/denormalized**; hierarchical sharding is solved with **consistent hashing.**

---

<!-- nav -->
⬅ [03 & 04 Microservices Design Patterns](03_04_Microservices_Design_Patterns_Notes.md) · [Index](00_HLD_Index.md) · [06 Consistent Hashing](06_Consistent_Hashing_Notes.md) ➡
<!-- nav -->
