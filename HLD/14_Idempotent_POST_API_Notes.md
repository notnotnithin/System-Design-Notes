# Design Idempotency Handler – Idempotent POST API (HLD #14 – Lecture 51)

<!-- nav -->
⬅ [12 & 36 Rate Limiter](12_36_Rate_Limiter_Notes.md) · [Index](00_HLD_Index.md) · [15 High Availability Architecture](15_High_Availability_Architecture_Notes.md) ➡
<!-- nav -->

Source (handwritten, 4 pages): https://drive.google.com/file/d/1pzl_Pr0iUD3jyRUtS58Sq9DyCsrr2dmE/view?usp=drive_link

---

<!-- simple-explanation -->
## In simple words – idempotent POST API

### What does "idempotent" mean?
An operation is **idempotent** if doing it **once or many times gives the same final result**. Think of pressing the "lift down" button five times: the lift still comes only once. Pressing "Pay ₹10" five times, however, should *not* pay ₹50.

Why it matters: networks fail. A client sends a request, the response gets lost or times out, and the client **retries** because it doesn't know whether the first one worked. If the operation isn't idempotent, the retry may do the work twice (double payment, two identical cart items).

### Which HTTP methods are safe to repeat?
- **GET** only reads data, so repeating it changes nothing.
- **PUT** sets something to a value ("name = Himanshu"). Doing it again leaves the same state.
- **DELETE** removes something. Deleting an already-deleted item leaves the same state (gone).
- **POST** usually *creates* or *triggers* something, so each repeat has a new effect. A payment of ₹10 from HS to Hardik, sent twice, moves ₹20. **POST is not idempotent, so we have to make it so.**

(Don't confuse this with **concurrency**: that is many *different* users competing for the same resource, like two people grabbing the last seat on BookMyShow. Idempotency is about the *same* request arriving more than once.)

### Two ways duplicates arrive
1. **Sequential:** the first request times out, so the client sends it again. The server may have already finished the first one.
2. **Parallel:** the original and the duplicate reach the server at almost the same moment, and both may start working.

### The solution: an Idempotency Key (IK)
An idempotency key is a **unique ID (a UUID) that identifies one logical operation**. The agreement is:
- The **client creates** the key, once per operation.
- **Retries of the same operation reuse the same key**; a different operation gets a new key.
- The client sends it in a header, for example `Idempotency-Key: IK1`.

### What the server does (step by step)
1. **Is the key present in the request?** If not → reply **400 Bad Request**.
2. **Look the key up in the database** (a small table with `Key` and `Status`).
3. **Key not found** → this is a new operation. Insert `IK1 = Created`, do the work, then mark it `Consumed` and reply **201 Created**.
4. **Key found, status `Created`** → the first request is still being processed. Reply **409 Conflict** ("wait, it's in progress").
5. **Key found, status `Consumed`** → the work is already done. Reply **200 OK** with the earlier result, **without doing it again**.

### Handling the parallel case
If two requests with the same key arrive together, both might check the table, see "not found", and both insert. To prevent this, the "check and insert" step must be **atomic** – protected by a lock (mutex / `synchronized`) or, better, by a **unique constraint** in the database so that only one insert can succeed. The winner proceeds; the other gets 409 or 200.

### Takeaway
Idempotency keys let clients **retry safely**. This is exactly what makes retry patterns and "at-least-once" messaging safe to use (see the [Retry Pattern](38_Retry_Pattern_Notes.md) note).
<!-- simple-explanation -->


## 1. Idempotency vs Concurrency
- **Concurrency** – many requests (R) hitting the **same resource** at the same time (example in notes: **BookMyShow** seat).
- **Idempotency** helps to **take care of duplicate requests**.
- ✱ It enables clients to **safely retry an operation without worrying about the side effect** the operation can cause.

Example operation: **adding an item into a cart** – `POST` followed by a duplicate `POST` would **create another row**.

---

## 2. Which HTTP methods are idempotent?
1. **By default GET, PUT and DELETE are idempotent in nature.**
2. **POST is NOT idempotent**, and **we have to make it idempotent.**

| Method | Behaviour on repeats |
|---|---|
| **GET** | Repeated `GET`s – **no side effect**; any time the client sends a GET the server returns the data. |
| **PUT** | e.g. "update name HS → Himanshu": repeating the same PUT many times leaves the **same final state** (name = Himanshu). |
| **POST** | Not idempotent. Example – payment **HS → Hardik ₹10**: 1st time: HS −10, Hardik +10. A repeated POST debits HS another −10 and credits Hardik another +10, and so on. **Money is moved multiple times.** |

---

## 3. How can duplicate POSTs come? (two types of side effect)
1. **Sequential** – client sends `POST` (e.g. "add item to cart"), the call **times out**, client sends a **duplicate/retry POST**. Server may already have processed the first one → *how do we handle this?*
2. **Parallel** – the **original POST** and a **duplicate POST** arrive at (almost) the same time → *how to handle? Only **one** request should be processed.*

---

## 4. Approach: Idempotency Key (IK)
Use an **Idempotency Key** – a **unique UUID**, appended/associated with the operation.

**Agreement with clients**
1. The **client should generate the idempotency key**.
2. **For each different operation, a new idempotency key** must be generated (retries of the *same* operation reuse the same key).

**IK life-cycle:** `Created / Acquired` → `Consumed / Claimed`

### Flow – sequential duplicates
`POST /addItem` (header `IK: IK1`)

1. **Client generates** an Idempotency Key (the original and its duplicate/retry use the same key).
2. Client **sets the IK in the request header** and calls the server.
3. **Validation – is the IK present in the request?**
   - **No → HTTP 400** (validation error).
   - Yes → continue.
4. Server **reads the idempotency key** and **checks the DB – is the IK present?**
5. **IK present in DB →** check its **status**:
   - `Created` (still being processed) → **HTTP 409** (conflict).
   - `Consumed` (already completed) → **HTTP 200** (return the earlier result, don't redo the work).
6. **IK not present in DB →** the server **puts an entry in the DB** (`Key = IK1, Status = Created`), then performs the operation. If the call/operation **succeeds**, it **consumes/claims the IK** (status → `Consumed`) and returns **HTTP 201**.

DB table: `| Key | Status |` e.g. `| IK1 | Created → Consumed |`.

### Flow – parallel duplicates
Two requests `POST /addItem` with **the same IK (IK1)** arrive simultaneously.

1. Client generates IK and sets it in the request header.
2. Server receives both requests (header = IK1 and header = IK1).
3. Validate that the IK is present (else HTTP 400).
4. Server reads the IK from the DB – the **check-and-insert must be a *critical section***, protected by a **Mutex / synchronized** (or an atomic DB operation such as a unique key insert), so that **only one request wins**:
   - The winner **puts the entry in DB** (`IK1, Created`), executes the operation, then marks it `Consumed` → **HTTP 201**.
   - The other request sees the IK already present (`Created`/`Consumed`) → handled per the table above (409 / 200) – **never processed twice**.

> ⚠️ Pages 3–4 of the original are rotated hand-drawn flowcharts; the exact status-code labels (409 for Created, 200 for Consumed, 201 for first success, 400 for missing key) were read from the diagram and are the most likely interpretation.

---

## Quick revision
- GET / PUT / DELETE are naturally idempotent; **POST must be made idempotent**.
- Duplicates occur **sequentially (timeout → retry)** or **in parallel**.
- Solution: **client-generated unique Idempotency Key** sent in a header, **stored in DB with status** (`Created` → `Consumed`), with the check-and-insert done **atomically (mutex/unique constraint)**.
- Responses: missing key → 400; in-progress duplicate → 409; completed duplicate → 200 (same result); first success → 201.
- Needed so that **retries** (see [Retry Pattern](38_Retry_Pattern_Notes.md)) are safe.

---

<!-- nav -->
⬅ [12 & 36 Rate Limiter](12_36_Rate_Limiter_Notes.md) · [Index](00_HLD_Index.md) · [15 High Availability Architecture](15_High_Availability_Architecture_Notes.md) ➡
<!-- nav -->
