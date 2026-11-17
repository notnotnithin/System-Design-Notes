# Design a Key-Value Database / DynamoDB (HLD #9)

<!-- nav -->
⬅ [08 Back of the Envelope Estimation](08_Back_of_the_Envelope_Estimation_Notes.md) · [Index](00_HLD_Index.md) · [10 SQL vs NoSQL](10_SQL_vs_NoSQL_Notes.md) ➡
<!-- nav -->

Source (handwritten, pages 27–30 of the 36-page PDF): https://drive.google.com/file/d/1iegvlzgcLJtcGvbjJbPqF4hZaB5u_nrh/view?usp=drive_link

---

<!-- simple-explanation -->
## In simple words – key-value database (like DynamoDB)

Amazon uses DynamoDB for the shopping cart. Goals: **scalability, decentralization (no boss node), eventual consistency**. The design has five ideas:
1. **Partitioning:** one machine can't store everyone's carts, so spread keys across servers using a **consistent-hashing ring with virtual nodes** (see [Consistent Hashing](06_Consistent_Hashing_Notes.md)).
2. **Replication:** to survive failures, store each key on **N servers** (the first is the **coordinator**; the next ones on the ring are copies), ideally in different data centres. The ordered list of those servers is the **preference list**.
3. **Get/Put and quorum:** a write is successful when **W** replicas acknowledge; a read when **R** replicas respond. If **R + W > N**, the read set and write set always overlap, so a read sees at least one up-to-date copy. Requests reach the right node via a **generic load balancer** (simple, but may add a hop) or a **partition-aware** one (goes straight to the right node, lower latency).
4. **Data versioning with vector clocks:** if a node is down during a write, replicas can end up with different versions. Each version carries a **vector clock** (a list of `(server, counter)` pairs) so the system can tell whether one version is *newer* or whether two are *concurrent conflicts*. Conflicts are returned to the application to merge (for a cart: combine items). This is the AP choice: stay available now, become consistent later.
5. **Gossip protocol:** nodes regularly share their state with a few random peers, so everyone learns about nodes joining, leaving or failing, without a central coordinator. (Only named in the notes.)
<!-- simple-explanation -->


**Amazon uses DynamoDB in the "Add to cart" feature.** Goals: **1. Scalability · 2. Decentralization · 3. Eventual consistency.**
**Steps:** 1. **Partition** · 2. **Replication** · 3. **Get & Put operations** · 4. **Data versioning** · 5. **Gossip protocol.**

### 1. Partition
A plain mapping (person → cart list) doesn't scale: when users grow we can't hold all users' data in one place, and the mapping itself would have to be stored on multiple systems. **Solution: partition requests with consistent hashing** – a **virtual ring** to handle **ranges** (S1 → keys 1–50, S2 → 51–100, S3, S4 …). To **prevent bombarding only one server → virtual nodes** (each server appears at several ring points).

### 2. Replication (avoid single point of failure)
Example: key **CAR-45** is served by **S1 (coordinator)**; to maintain availability replicate to **N−1** other servers (copies on **S2 and S3**) – kept on **different data centres.** The **preference list**: S1 = coordinator, S2/S3 = backups.

### 3. Get & Put operations
- **Put(CAR) → find key (45)** → stored in **S1**, which keeps it and **replicates asynchronously** to S2, S3 (`45-car` placed in S1, S2, S3). **Quorum rule: R + W > N** – wait for the required number of success responses (**W** for writes, **R** for reads, **N** = number of replica nodes).
- **Get:** a request reaches a **load balancer** that must route it – **① Generic LB** (simple to implement but **high latency** – may hit any server; then the **preference list** works and the **coordinator** serves; if the coordinator is down, the next nodes on the preference list take over) or **② Partition-aware LB** (**low latency**, routes straight to the right node).
- **R + W > N → Get returns a successful response only if R nodes respond; a Put is successful only if W nodes acknowledge.**

### 4. Data versioning
Why: (1) `Put(CAR-45)` → S1 `[45-CAR]`, replicated to S2, S3. (2) `Put(CART-45)` → S1 is **down now**, so it goes to the next on the preference list, S2. (3) Server S1 comes back and a Get arrives: S1 still has `[45-CAR]` (**not updated**) – it must **talk with all** (S2, S3) to reconcile. **To implement it: Vector Clock (server, counter)** per version – example: v1 `[45-CAR]-1` on all; v2 `[45-CART]-2` on all; v3 `[45-CARM]-3` on S2 & S3 while S1 is down; v4 `[45-CARR]-4` concurrently on S3; on Get the coordinator sees CART(2), CARM(3), CARR(4) – **conflicting concurrent versions** – and **"eventual consistency – because here we can't compromise availability & partition tolerance, so we compromise consistency" (AP)**; the conflicts are resolved by the client/app via vector clocks.

### 5. Gossip protocol
(Named in the steps list – nodes periodically exchange state with random peers to detect failures and learn membership; the pages don't elaborate.)

---

## Quick revision
- **Key-value DB (Dynamo):** consistent-hashing ring + virtual nodes, **N replicas with R+W>N quorum**, **vector clocks** for versioning, **gossip** for membership – AP/eventual consistency.

---

<!-- nav -->
⬅ [08 Back of the Envelope Estimation](08_Back_of_the_Envelope_Estimation_Notes.md) · [Index](00_HLD_Index.md) · [10 SQL vs NoSQL](10_SQL_vs_NoSQL_Notes.md) ➡
<!-- nav -->
