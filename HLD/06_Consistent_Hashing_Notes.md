# Consistent Hashing (HLD #6 – Lecture 44)

<!-- nav -->
⬅ [05 Scale from Zero to Million Users](05_Scale_from_Zero_to_Million_Users_Notes.md) · [Index](00_HLD_Index.md) · [07 URL Shortener](07_URL_Shortener_Notes.md) ➡
<!-- nav -->

Source (handwritten, 5 pages): https://drive.google.com/file/d/1dMSY_fHaKwADw_zeQJfoWQUiHwgwDL6H/view?usp=drive_link

---

<!-- simple-explanation -->
## In simple words – consistent hashing, step by step

### First, what is hashing?
A **hash function** takes any input (a name, an ID, a URL) and turns it into a **number**. The same input always gives the same number. For example, `hash("Himanshu") = 14321`.

To decide *where* to store something, we often use **mod hashing**: `hash(key) % N`, where **N is the number of servers**. With 3 servers, `14321 % 3 = 2`, so "Himanshu" goes to server 2. Every machine can calculate this on its own, so no lookup table is needed. It's a neat way to spread data evenly.

### The problem: servers change
What if you **add a 4th server**? Now the formula is `hash % 4`, and **almost every key suddenly maps to a different server**. The data would have to be moved around – with millions of records that is slow, expensive and can break the system. The same happens when a server crashes (N becomes smaller). This is called **rebalancing**, and mod hashing makes it terrible.

### The idea behind consistent hashing
Instead of using `% N`, imagine a **circle (ring)** with positions 0 to 11 (in real systems, billions of positions).
1. Place each **server** on the ring by hashing its name. Example: S1 at position 3, S2 at 7, S3 at 11.
2. Place each **key** on the ring by hashing it too. Example: key K1 at position 1.
3. **Rule: a key belongs to the first server you meet moving clockwise from the key's position.** K1 at 1 → moves clockwise → meets S1 at 3. So S1 stores K1.

That's all. Every machine can compute this independently.

### Why this fixes the rebalancing problem
- **Adding a server:** add S4 at position 5. Only the keys sitting between the previous server (S1 at 3) and S4 (at 5) – i.e. positions 4 and 5 – now belong to S4 instead of S2. **Every other key stays exactly where it was.** On average only about **1/N of the keys move** (with 10 servers, roughly 10%), instead of nearly all of them.
- **Removing a server:** its keys simply move to the **next server clockwise**. Nobody else is affected.

### A weakness: uneven load
If servers happen to land **close together** on the ring (e.g. S1 at 2, S2 at 3, S3 at 4), one server will cover a huge arc and receive most keys, while others get almost nothing.

### The fix: virtual nodes
Give each real server **many positions** on the ring. Example: S1 sits at positions 2, 8 and 11; S2 at 3, 6 and 10; S3 at 4 and 9. Now each server owns several small arcs scattered around the ring, so the load averages out. When a server is added or removed, its load is shared among **many** neighbours instead of dumped on just one.

### Where it is used
Distributed caches (so adding a cache node doesn't flush everything), database sharding, load balancers that need the same user to hit the same server, and key-value stores like DynamoDB and Cassandra.
<!-- simple-explanation -->


## 1. Hashing basics (p1)
- **Hash function** takes a **key of arbitrary length** (e.g. `"Himanshu"`) and produces a **fixed-length** value.
  Example: `f("Himanshu") ⇒ 14321 % 6 ⇒ 2` → stored at **index 2** of a fixed-size table (size 6).
- This is **mod hashing**: `hash(key) % N`, N = table size / number of nodes.

### What is the problem?
- If the **mod hashing size is fixed**, there is **no problem**.
- Otherwise – when the **size is not fixed** – the problem occurs.

Two places where size changes:
1. **Load balancing for the application server**
2. **Horizontal sharding**

### 1) Load balancing for application servers
```
 key/request → LB: f(key) % 3 → App1 | App2 | App3        (the node count is dynamic:
                                                           one app goes down, or a new node is added)
```
**Note:** the load balancer should **divide the traffic equally** across my app servers so that **only one server doesn't carry all the load** (e.g. 10 requests are spread evenly).

### 2) Horizontal sharding
- Example: 100 rows split by name range → **A–P → DB1 (50 rows), Q–Z → DB2 (50 rows)**.
- Problem: if **many names start with "A"**, DB1 fills early → **load is not equally divided**.
- With mod hashing instead: `LB: f(Himanshu) % 3 = 1 → Node 1`, `f(Rahul) % 3 = 2 → Node 2`, `f(Shivam) % 3 = 3 → Node 3` – evenly spread **as long as the node count stays 3.**

---

## 2. Use case: increase the servers/nodes (p3)

<!-- diagram:06-0 -->
**Diagram – Mod hashing: adding a node moves almost every key**

```mermaid
graph LR
  subgraph B["Before: f(key) % 3"]
    direction TB
    K1["Himanshu"] --> N1["N1"]
    K2["Rahul"] --> N2["N2"]
    K3["Shivam"] --> N3["N3"]
  end
  subgraph A["After adding N4: f(key) % 4"]
    direction TB
    K4["Himanshu"] --> M2["N2 (moved)"]
    K5["Rahul"] --> M3["N3 (moved)"]
    K6["Shivam"] --> M1["N1 (moved)"]
  end
```
<!-- /diagram:06-0 -->


Nodes N1, N2, N3 with `f(key) % 3`. **Add a new node N4** → formula becomes `f(key) % 4`:
- e.g. `Himanshu` was on N1 and now maps to **N2** → **almost all keys get a new location.**
- **Rebalancing = moving millions of DB entries.** Very costly.

---

## 3. Consistent Hashing – the idea
- In consistent hashing, **rebalancing occurs when nodes (servers) are added or removed**. The **goal is to minimize the number of keys that need to be reassigned while maintaining uniform distribution.**
- The **main motto: when nodes are added or removed, keep rebalancing minimal.**
- Rebalancing moves only about **1/N of the total number of keys** (N = number of nodes) – "good".
  - e.g. N1 holds keys K1–K10, N2 holds K…, N3 (removed) – only its keys move.

---

## 4. How does Consistent Hashing work? (p4)

### ① Virtual ring

<!-- diagram:06-1 -->
**Diagram – Ring with 12 slots (clockwise): keys go to the next server**

```mermaid
graph LR
  n0(("0"))
  n1(("1<br/>K1")):::key
  n2(("2<br/>K2")):::key
  n3(("3<br/>S1")):::srv
  n4(("4<br/>K7")):::key
  n5(("5"))
  n6(("6<br/>K5")):::key
  n7(("7<br/>S2")):::srv
  n8(("8<br/>K6")):::key
  n9(("9<br/>K3")):::key
  n10(("10<br/>K4")):::key
  n11(("11<br/>S3")):::srv
  n0 --> n1 --> n2 --> n3 --> n4 --> n5 --> n6 --> n7 --> n8 --> n9 --> n10 --> n11 --> n0
  classDef srv fill:#fde68a,stroke:#b45309,color:#000
  classDef key fill:#dbeafe,stroke:#1d4ed8,color:#000
```
<!-- /diagram:06-1 -->


- Take a **virtual ring** with slots (example: **0–11, i.e. mod 12**), traversed **clockwise**.
- **Servers** are placed on the ring using mod hash:
  - `S1 = hash(server1) % 12` → slot **3**
  - `S2 = hash(server2) % 12` → slot **7**
  - `S3 = hash(server3) % 12` → slot **11**
- **Keys** are placed the same way: `Key1 = hash(key1) % 12`, … (K1 at 1, K2 at 2, K7 at 4, K5 at 6, K6 at 8, K3 at 9, K4 at 10).
- **Rule:** a key is handled by the **first server found moving clockwise** from the key's position.

**Whose server handles which key?**
- `S1 = K1, K2`
- `S2 = K5, K7`
- `S3 = K6, K3, K4`

### Addition of servers

<!-- diagram:06-2 -->
**Diagram – After adding S4 at slot 5 only K7 moves (S2 to S4)**

```mermaid
graph LR
  n0(("0"))
  n1(("1<br/>K1")):::key
  n2(("2<br/>K2")):::key
  n3(("3<br/>S1")):::srv
  n4(("4<br/>K7")):::key
  n5(("5<br/>S4")):::srv
  n6(("6<br/>K5")):::key
  n7(("7<br/>S2")):::srv
  n8(("8<br/>K6")):::key
  n9(("9<br/>K3")):::key
  n10(("10<br/>K4")):::key
  n11(("11<br/>S3")):::srv
  n0 --> n1 --> n2 --> n3 --> n4 --> n5 --> n6 --> n7 --> n8 --> n9 --> n10 --> n11 --> n0
  classDef srv fill:#fde68a,stroke:#b45309,color:#000
  classDef key fill:#dbeafe,stroke:#1d4ed8,color:#000
```
<!-- /diagram:06-2 -->


Add **S4 at slot 5**:
- K7 (slot 4) now finds **S4** first instead of S2.
- `S1 = K1, K2` · `S2 = K5` · `S3 = K6, K3, K4` · **`S4 = K7`**
- ✱ **We can rebalance only a limited number of keys** (only the keys between the previous server and the new one move).
- (Removing a server works the same way in reverse: only that server's keys move to the next server clockwise.)

---

## 5. Disadvantages of consistent hashing (p5)

<!-- diagram:06-3 -->
**Diagram – Skewed servers: S1 takes almost all keys**

```mermaid
graph LR
  n0(("0"))
  n1(("1"))
  n2(("2<br/>S1")):::srv
  n3(("3<br/>S2")):::srv
  n4(("4<br/>S3")):::srv
  n5(("5<br/>K1")):::key
  n6(("6<br/>K2")):::key
  n7(("7<br/>K3")):::key
  n8(("8<br/>K4")):::key
  n9(("9<br/>K5")):::key
  n10(("10<br/>K6")):::key
  n11(("11"))
  n0 --> n1 --> n2 --> n3 --> n4 --> n5 --> n6 --> n7 --> n8 --> n9 --> n10 --> n11 --> n0
  classDef srv fill:#fde68a,stroke:#b45309,color:#000
  classDef key fill:#dbeafe,stroke:#1d4ed8,color:#000
```
<!-- /diagram:06-3 -->


- Example ring where servers are clustered close together: **S1 at 2, S2 at 3, S3 at 4**. Keys K1–K6 are spread around the ring but **all of them walk clockwise to S1** (the next server) → S1 takes almost all the load.
- So consistent hashing **fails to divide the load equally** when server positions are skewed.

### Solution: create virtual nodes (virtual objects)

<!-- diagram:06-4 -->
**Diagram – Virtual nodes spread each server over the ring**

```mermaid
graph LR
  S1["Server S1"] --- a1["slot 2"]
  S1 --- a2["slot 8"]
  S1 --- a3["slot 11"]
  S2["Server S2"] --- b1["slot 3"]
  S2 --- b2["slot 6"]
  S2 --- b3["slot 10"]
  S3["Server S3"] --- c1["slot 4"]
  S3 --- c2["slot 9"]
```
<!-- /diagram:06-4 -->


- **Replicate each server at different points on the ring.**
- Example: **S1 → slots 2, 8, 11; S2 → slots 3, 6, 10; S3 → slots 4, 9.**
- Result in the example: `S3 = K2, K1`; `S2 = K6, K7`; `S1 = K5, K4, K3` – a much more even spread.
- More virtual nodes per server ⇒ smoother distribution (and when a server leaves, its load is shared among many servers).

---

## Quick revision
- `hash % N` breaks when **N changes** (nodes added/removed) → nearly all keys move.
- **Consistent hashing:** hash servers **and** keys onto a **ring**; key → **next server clockwise**; adding/removing a node moves only **~1/N of keys**.
- Weakness: **uneven distribution** if servers land close together → fix with **virtual nodes** (each physical server placed at many ring points).
- Used for: load balancing app servers, sharding DBs, distributed caches, key-value stores.

---

## Extra from the 36-page handwritten compendium (pages 18–19)
These pages are in the same PDF as notes #1, #7, #9 and #11 (https://drive.google.com/file/d/1iegvlzgcLJtcGvbjJbPqF4hZaB5u_nrh/view?usp=drive_link); kept here because they belong to this topic.

- **Hashing** converts arbitrary-length values to a **fixed-length** value (for fast data access); **mod hashing** places data in a fixed place (array length 7: hash 1451 → 1451 % 7 = **2**).
- Hashing is used mainly in two places: **load-balancing application servers** and **load-balancing database sharding.**
- **Problem – rebalancing:** whenever the array size changes (3 instances → 4 instances) values must be re-placed on different nodes (e.g. 25%3, 20%3, 39%3 vs 25%4, 20%4, 39%4, 52%4) – with millions of records that is huge.
- **Solution: consistent hashing** – only **~1/n of keys** need rebalancing ("good"). A **virtual ring**: servers placed by mod hashing; **each key is served by the next server clockwise**; since servers are continuous this can be unbalanced → **virtual replication of servers** (S1, S2, S3 placed at several ring points).

---

<!-- nav -->
⬅ [05 Scale from Zero to Million Users](05_Scale_from_Zero_to_Million_Users_Notes.md) · [Index](00_HLD_Index.md) · [07 URL Shortener](07_URL_Shortener_Notes.md) ➡
<!-- nav -->
