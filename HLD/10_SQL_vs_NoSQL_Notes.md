# SQL vs NoSQL (HLD #10 – Lecture 48)

<!-- nav -->
⬅ [09 Key Value DB](09_Key_Value_DB_Notes.md) · [Index](00_HLD_Index.md) · [11 Design WhatsApp Chat App](11_Design_WhatsApp_Chat_App_Notes.md) ➡
<!-- nav -->

Source (handwritten, 7 pages): https://drive.google.com/file/d/1CuxKzs00W6meH_6PdF2la8xBkXnyvhcO/view?usp=drive_link

**Compared on 4 dimensions:** 1. **Structure** · 2. **Nature** · 3. **Scalability** · 4. **Properties**

---

<!-- simple-explanation -->
## In simple words – SQL vs NoSQL

### What problem are we solving?
Every application needs to store data. There are two big families of databases. The choice affects how you design tables, how you scale to millions of users and how safe your data is.

### SQL databases (MySQL, PostgreSQL, Oracle)
- **Structure:** data lives in **tables** made of **rows and columns**, like a spreadsheet. Tables are linked by **relationships** (a `Salary` table points to an `Employee` table).
- **Fixed schema:** before storing anything you must define the columns and their types (name = text, roll = number). This planning up front keeps data tidy.
- **Query language:** you ask questions with SQL (`SELECT`, `JOIN`…), which is very flexible – you can combine tables in almost any way.
- **ACID guarantees:** a transaction is **A**tomic (all-or-nothing), **C**onsistent (rules always hold), **I**solated (parallel users don't disturb each other) and **D**urable (once saved, it survives crashes). This is why banks use SQL.
- **Scaling:** the easy way is **vertical** – buy a bigger machine (more CPU/RAM). Splitting the data across machines (**sharding**) is possible but awkward, because related rows should live on the same machine for joins to work. So SQL data tends to be **centralised on one server**.

### NoSQL databases ("Not only SQL")
They give up some SQL features in exchange for **huge scale and flexibility**. Four common kinds:
1. **Key-value** (DynamoDB, Redis) – like a giant dictionary: give a key, get a value. The database doesn't understand the value (it is "opaque"), so you can only look things up **by key**. Very fast.
2. **Document** (MongoDB) – like key-value, but the value is a **JSON document** whose fields the database understands, so you can query *inside* it ("find users with age > 30").
3. **Column-family / wide-column** (Cassandra) – each row stores just the columns it needs; different rows can have different columns. Great for huge write volumes.
4. **Graph** (Neo4j) – stores **nodes and relationships** (Himanshu –*friend*→ XYZ). Perfect for social networks and recommendations, because following relationships is fast (SQL would need many expensive joins).

- **Distributed by nature:** data is spread over many machines (e.g. 2 million users on each of 5 nodes). To grow, you simply **add more nodes** (**horizontal scaling**).
- **BASE instead of ACID:** **B**asically **A**vailable (always answers), **S**oft state (data may still be changing as copies sync), **E**ventually consistent (all copies become equal after a short time). So right after a write, one node might briefly return old data – an acceptable trade-off for many apps.

### How to choose
| Choose SQL when… | Choose NoSQL when… |
|---|---|
| you need flexible queries and joins | you mostly look things up by key or simple filters |
| data is naturally relational (customers → orders → items) | data is huge, fast-growing or has no fixed shape |
| correctness matters most (money, stock) – ACID | availability and speed matter most; a few seconds of staleness is fine |
| data fits on one strong server (or a primary + replicas) | you need to spread data over many servers |

Many real systems use **both**: SQL for orders and payments, NoSQL for caches, activity feeds, chat messages and logs.
<!-- simple-explanation -->


# PART A – SQL

**SQL (Structured Query Language)** is used to **query / manage a Relational Database Management System (RDBMS)** – with `SELECT`, `INSERT`, `UPDATE` – over **tables → rows → columns**. Tables can be related (**Parent → Child** via relations).

## 1. Structure
1. Data lives in **tables, rows and columns**.
2. **Predetermined schema** – the schema must be defined **before you start using it** (e.g. `Table(name: String, roll: Number(20))`).
3. **We plan the schema before the insert/update operations.**

## 2. Nature – *concentrated / centralized*
- Example: DB **Employee** with Table 1 (employee id, name, dept e.g. `1, IT`), Table 2 (**Salary**), Table 3 (**Address**) – related to each other.
- For a particular user/employee (e.g. "Himanshu") **the whole data is present in one server.**
- ⇒ Nature: **concentrated / centralized.**

## 3. Scalability
We want to scale a `User` table with e.g. **10 million users**:
1. **Vertical scaling** – increase **RAM, storage capacity** of the machine. **Vertical fits SQL much better.**
2. **Horizontal scaling (sharding)** – based on rows and columns. **Not well supported in SQL.**

## 4. Properties – **ACID**
**A**tomicity, **C**onsistency, **I**solation, **D**urability → together they **make sure data integrity** (consistency).

---

# PART B – NoSQL
**NoSQL = "Non-relational", "Not only SQL".**

## 1. Structure
NoSQL **works on unstructured data.** Four types:

### ① Key–Value DB (e.g. **DynamoDB**)
- Table of `Key | Value`; the value can be a **String / Integer / JSON**.
- The value is **opaque** to the DB – **you search / query only on the key.**

### ② Document DB (e.g. **MongoDB**)
- `Key | Value` where the value is a **JSON (or XML) document**.
- **Search based on the key, and we can also query on the value** (fields inside the document).

### ③ Column-wise DB
- Each **key** maps to a set of **column → value** pairs; different keys can have different columns.
- Example:
  - key `1001` → `first-name: Himansh`, `department: IT`, `interest: BasketBall`
  - key `1002` → `first-name: Raj`, `department: Sales`

### ④ Graph DB
- Stores data as **Nodes and Edges (relationships)**.
- Example: `Himanshu —friend→ XYZ —friend→ …`.
- Used by **social networking, recommendation engines.**
- It is **very, very fast.** SQL also has relationships (joins) but they are costly (**SQL has to scan fully**), while a **Graph DB has the relationship directly.**

## 2. Nature – *distributed*
- Data is **distributed across nodes** – e.g. 10 M users spread as **Node 1 = 2 M, Node 2 = 2 M, Node 3 = 2 M, …**

## 3. Scalability – *horizontal*
- "My user table is growing fast (1 M, …)" → **we can add any number of nodes to store user data.**

## 4. Properties – **BASE** (not ACID)
| Letter | Meaning | Explanation |
|---|---|---|
| **BA** | **Basically Available** | highly available DB – if one node (e.g. the one holding "Sha…") is down, another replica/node still serves the data |
| **S** | **Soft state** (written as "Safe state") | the **state of data can change without interaction** – replicas (N1/S1, N2/S2, N3/S3, e.g. `Him : 1 : S1`) sync among themselves in the background |
| **E** | **Eventual consistency** | a query right after a write may return **stale** data; **after some time** the replicas converge and the **latest** value is returned |

---

# PART C – When to use what?

| | **SQL** | **NoSQL** |
|---|---|---|
| 1 | **Flexible query functionality** needed | ✗ – only **basic query / search**, or you must know in advance **which column needs search capability** |
| 2 | Data is **relational in nature** (parent–child) | – |
| 3 | **Data integrity (ACID) / consistency** is required | ✗ ACID – **NoSQL is developed to handle large amounts of data**; choose it for **high availability, high performance / search (query) with some consistency (eventual)** |

---

## Quick revision
| Dimension | SQL | NoSQL |
|---|---|---|
| Structure | tables/rows/columns, predefined schema | unstructured – key-value, document, column-family, graph |
| Nature | centralized (data of an entity on one server) | distributed |
| Scalability | mainly **vertical** (sharding poorly supported) | **horizontal** (add nodes) |
| Properties | **ACID** | **BASE** (Basically Available, Soft state, Eventual consistency) |
| Examples | MySQL / Postgres (RDBMS) | DynamoDB (KV), MongoDB (document), column-wise stores, Graph DBs |

---

## Extra from the 36-page handwritten compendium (pages 30–32)
These pages are in the same PDF as notes #1, #7, #9 and #11 (https://drive.google.com/file/d/1iegvlzgcLJtcGvbjJbPqF4hZaB5u_nrh/view?usp=drive_link); kept here because they belong to this topic.

(Full version: [10_SQL_vs_NoSQL_Notes.md](10_SQL_vs_NoSQL_Notes.md).)
- **SQL (RDBMS):** tables (row-column) with relationships; **predetermined schema**; **centralized/concentrated** – complete data placed on one server; **scalability** – vertical (RAM, storage) – horizontal isn't well supported; **ACID** → data integrity/consistency (same at every place).
- **NoSQL:** non-relational / "not only SQL"; **unstructured data** – **① Key-value** (opaque value – *you can't query on the value*, e.g. string/JSON), **② Document DB** (`Key | JSON` – can **query through values**, e.g. value > 30), **③ Column-wise** (column–value pairs, e.g. Name: Sujal Sharma), **④ Graph DB** (node + edge/relationship, e.g. Sujal —learning→ SD); **distributed** nature; **horizontal** scalability; **BASE** (Basically available, Soft state, Eventual consistency).
- **When to use:** SQL – flexible query functionality, relational, data integrity (ACID); availability is lower, "just to provide consistent data." NoSQL – no complex queries, non-relational, no strict consistency; **high availability / performance / search queries / less consistency.**

---

<!-- nav -->
⬅ [09 Key Value DB](09_Key_Value_DB_Notes.md) · [Index](00_HLD_Index.md) · [11 Design WhatsApp Chat App](11_Design_WhatsApp_Chat_App_Notes.md) ➡
<!-- nav -->
