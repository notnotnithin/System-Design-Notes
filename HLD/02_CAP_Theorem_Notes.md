# CAP Theorem (HLD #2 – Lecture 40)

<!-- nav -->
⬅ [01 Network Protocols](01_Network_Protocols_Notes.md) · [Index](00_HLD_Index.md) · [03 & 04 Microservices Design Patterns](03_04_Microservices_Design_Patterns_Notes.md) ➡
<!-- nav -->

> Transcribed from the 4-page handwritten PDF:
> https://drive.google.com/file/d/1rEs7p_Hrgdu57xakTyxgO1k3Zyp3zu-C/view?usp=drive_link
> Page references (p1–p4) point to the original pages.

---

<!-- simple-explanation -->
## In simple words – what the CAP theorem is really saying

### The problem it talks about
Imagine a company stores the same data on **several computers (nodes)** in different places – say one in India and one in the US – so that if one breaks, the other still works. This is called **replication**. Whenever someone writes new data on one node, that node must copy it to the others.

CAP is about what happens when that copying **can't happen** (for example, the network cable between the two data centres is cut). It says: *you have to choose which of two good things to give up.*

### The three letters, in everyday language
- **C – Consistency:** *"Everyone sees the same, latest data."* If you update a value to 5 on one node, then any other node you ask must also answer 5 – never the old value 4. It's like a bank balance: after you deposit money, every ATM should show the new balance.
- **A – Availability:** *"Every request gets an answer."* Any node that is working must respond (with data or with an error) and must not just hang or ignore you. It's like a shop that is always open.
- **P – Partition tolerance:** *"The system keeps working even if the nodes can't talk to each other."* A **partition** means the network between nodes is broken, so each side is cut off from the other. In real networks this **will** happen sometimes, so a distributed system must be built to survive it.

### Why you can only pick two
Take the example from the notes: nodes **B** and **C** hold the value `a`. A write sets `a = 6` on B. Now the link between B and C breaks, so C never hears about the change and still holds `a = 5`. A user asks C for `a`. C has only two choices:
1. **Answer `5`** → C replies (so it is *available*) but gives **old data** (so it is *not consistent*).
2. **Refuse / wait** until it can check with B → the data it gives (when it finally does) is correct (*consistent*), but during the break it is **not answering** (*not available*).

There is no third option – C cannot give the new value because it never received it. That is why **C, A and P cannot all be perfect at the same time.** During a partition you must give up **C or A**.

### "CA, CP, AP" – the three possible combinations
- **CA (Consistency + Availability, no partition tolerance):** works only while the network is healthy. If nodes get separated, the system breaks. In practice this really means "a single node or one tightly-connected cluster".
- **CP (Consistency + Partition tolerance):** during a network break the system **stops answering** (or returns an error) for data it cannot guarantee is correct. Good for money and inventory, where wrong data is worse than no data.
- **AP (Availability + Partition tolerance):** during a break the system **keeps answering**, even if some answers are slightly old, and fixes the differences later ("eventual consistency"). Good for social feeds, shopping carts, "likes".

### The practical takeaway
Because network failures can't be avoided in a distributed system, **partition tolerance is not optional**. So the real design question is simply: **"When things break, do I prefer correct answers (CP) or always-available answers (AP)?"** – and that depends on your business.
<!-- simple-explanation -->


## 1. What is the CAP Theorem? (p1)

- CAP describes the **desirable properties of a distributed system with replicated data**.
  - Sketch in notes: a **Master DB** replicating to a **Slave DB**.
- The three properties:

| Letter | Property |
|---|---|
| **C** | Consistency |
| **A** | Availability |
| **P** | Partition Tolerance |

- **Key rule:** we **can't use all three together; we can use only two** → **CA, CP, or AP**.
  - Venn diagram in notes: three overlapping circles C, A, P; the pairwise overlaps are CA, CP, AP. The centre (all three) is crossed out / marked as the "unique part" that is not achievable.

---

## 2. Running example used throughout

Three nodes holding the same data, replicated:

```
                 Application (A)
                 /            \
        write a=5              read query
               /                \
     DB node B (India)  <---->  DB node C (US)
        (a = 4 → 5)          (a = 4 → 5 after replication)
```

- **A** = the application node issuing queries.
- **B** = DB node in **India**, **C** = DB node in **US**.
- B and C replicate data between each other (the dashed arrow B → C in the notes).

---

## 3. C – Consistency (p1)

- **Scenario:** Suppose node B has data `a = 4`.
  1. Application **writes** an update query: `a = 5` to node B.
  2. B updates its data `a = 4 → a = 5`.
  3. B **replicates** the data to node C.
- **Definition:** Consistency means:
  - the application **writes** the data to **node B**, and
  - a later **query to read** the data from **node C**
  - **always finds the updated data** (`a = 5`).
- In short: *every read sees the most recent write, regardless of which node serves it.*

---

## 4. A – Availability (p2)

- **Both node B or C – at least one response.**
- **All nodes should respond – at least a success or a failure.**
  - i.e. every request to a working node gets a response (it can't hang/ignore); the response may be success or failure, but there must be one.
- Diagram: App A connected to B and C; a query to either node gets a response.

---

## 5. P – Partition Tolerance (p2)

- **System is up** even though the network between nodes is broken.
- The system is **hidden** from the user:
  - The user **doesn't know** data is replicated (or not).
  - The user only knows they are **querying "the system"**.
- Diagram: the DB nodes B and C sit inside a "hidden" box; the application talks to the box. A **breakage** is drawn on the B–C link.
- **Definition:** Partition tolerance tells us that **if B and C have a communication break, even then the application continues querying** → that is partition tolerance.

---

## 6. Which combinations are possible? (p3)

| Combination | Possible? | Note |
|---|---|---|
| **CA** | ✅ Possible | (system up – i.e. only when there is no partition) |
| **CP** | ✅ Possible | |
| **AP** | ✅ Possible | |
| **CAP** (all three) | ❌ Not possible | cannot have C, A and P together |

---

## 7. Why can't C, A and P co-exist? (p3)

Setup: A writes `a = 6`; the link between B and C is **broken** (partition, drawn as `$$ / Break`).

- B gets the write → `a = 6`.
- C could not receive the replication → still `a = 5`.

Now a **read query** hits C (**Available + Partition Tolerance** case, drawn on the left of the page):

- If C **answers (success)** with `a = 5` → the answer is **stale → Consistency is broken**.
- If C **refuses (failure)** to avoid returning stale data → it is **not responding with data → Availability is broken**.

So during a partition you must **give up either C or A**. You can't keep all three.

(Sketch labels on the page: "Success / failure" next to B and C, with values `a=5→6` on B and `a=5` on C.)

---

## 8. Pairs in practice (p4)

### CP-style loss example – "I do not lose consistency"
- Write `d = 6` goes to B; C is **down** (still has `d = 5`).
- Notes' reading: with C down and a failed replication, the system sacrifices something – the figure is labelled "**CAP → ✗**" (not all three), showing that when a node can't be reached you can't have everything.

### "I do Consistency and Availability" → **CA**
- Diagram: A is **down** / link between B and C is shown with the break symbol and ✗.
- Reading: CA = consistent + available, but **no partition tolerance**; it works only while the network between nodes is healthy.

> ⚠️ Page 4's two small diagrams are rough sketches (rotated, partly scribbled). The interpretation above follows the sketches and the captions; double-check against the original page if you need exact wording.

---

## 9. Trade-off conclusion (p4)

- **"We never trade off partition tolerance."**
  - In a real distributed system, network partitions *will* happen, so **P is mandatory**.
  - Therefore the real choice is between:
    - **CP** – keep consistency, sacrifice availability during a partition.
    - **AP** – keep availability, sacrifice consistency during a partition.
  - (The notes box lists: `CP`, `AP` as the practical choices; `CA` is circled beside it as the option that drops P.)

---

## 10. Quick revision sheet

- **C** – every read returns the latest write.
- **A** – every request to a live node gets a (success/failure) response.
- **P** – system keeps working when nodes can't talk to each other.
- Only **2 of 3** at a time → **CA / CP / AP**.
- Partition tolerance is non-negotiable in distributed systems → choose **CP vs AP**.
- Proof sketch: A writes `a=6` to B, B–C link breaks, C still has `a=5`; reading from C must either return stale data (lose C) or refuse (lose A).

---

## Extra from the 36-page handwritten compendium (pages 5–6)
These pages are in the same PDF as notes #1, #7, #9 and #11 (https://drive.google.com/file/d/1iegvlzgcLJtcGvbjJbPqF4hZaB5u_nrh/view?usp=drive_link); kept here because they belong to this topic.

- **CAP** = desirable properties of a **distributed system with replicated data**: **C**onsistency, **A**vailability, **P**artition tolerance (Venn: CA, CP, AP).
- **In a distributed system we can't use all 3 properties.** Causes: (1) if we make the application consistent and available, a **partition is not possible**, because to be consistent the DB instances must be connected; (2) if you partition, then **forget about consistency or availability.**
- **Consistency:** B (a=4) and A writes on B (a=5) → B replicates the same value to C (a=5) – the changes must be done at each place in one shot.
- **Availability:** all nodes must be responsive, even though data is inconsistent.
- **Partition:** B and C are not connected, yet A still gets a response → that's fault tolerance.
- *"Always compromise between consistency & availability" – **Brewer's theorem.***
(See [02_CAP_Theorem_Notes.md](02_CAP_Theorem_Notes.md).)

**ACID** (SQL): **Atomicity** – a transaction must happen in one shot; **Consistency** – data must be the same on all nodes; **Isolation** – instances must work individually; **Durability** – instances must be backed up.
**BASE (AP databases):** **BA**sically **A**vailable · **S**oft state · **E**ventually consistent.

**Examples**
- **PostgreSQL (CA):** uses SQL; follows ACID; used in **banking** where consistency and availability are crucial; provides **foreign data wrappers** that allow multiple DBs to communicate.
- **MongoDB (CP):** primarily BASE-oriented (document store), but now it is adopting ACID.
- **Cassandra (AP):** sacrifices consistency at the cost of availability; it is **peer-to-peer.**

---

<!-- nav -->
⬅ [01 Network Protocols](01_Network_Protocols_Notes.md) · [Index](00_HLD_Index.md) · [03 & 04 Microservices Design Patterns](03_04_Microservices_Design_Patterns_Notes.md) ➡
<!-- nav -->
