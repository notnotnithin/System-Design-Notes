# Distributed Cache and Caching Strategies (HLD #19 – Lecture 56)

<!-- nav -->
⬅ [18 Load Balancer and Algorithms](18_Load_Balancer_and_Algorithms_Notes.md) · [Index](00_HLD_Index.md) · [20 Distributed Transactions](20_Distributed_Transactions_Notes.md) ➡
<!-- nav -->

Sources:
- Handwritten PDF (9 pages): https://drive.google.com/file/d/1C7f5kvsWQ45ECA3sR0jgpym1cy96CgR0/view?usp=drive_link
- Typed Zoho note (JavaScript-rendered, not readable automatically): https://notebook.zohopublic.in/public/notes/u3i1s6d12db6764b94cff9ad7f4ddef5b5348

## Agenda (as listed on page 1)
- What is caching? · What is distributed caching and how is it done?
- Caching strategies: **Cache-aside, Read-through, Write-around, Write-through, Write-back**
- Cache eviction policies: **LRU, FIFO, LFU, etc.** *(listed in the agenda; the handwritten PDF ends after Write-back – eviction policies are not written out in these 9 pages)*

---

<!-- simple-explanation -->
## In simple words – distributed cache and caching strategies

### What is caching?
Reading from a database (disk, over a network) is slow. A **cache** keeps frequently used data in **fast memory** so the next read is quick. This makes the app faster, reduces load on the database, and even helps when the database is struggling. Caching can happen in many places: the **browser**, a **CDN**, the **load balancer**, and in the **server** (e.g. Redis).

### Why "distributed" cache?
One cache server has limited memory and is a single point of failure. If all app servers depend on it, it limits **scalability** and **availability**. So we use a **pool of cache servers**. Each app server has a **cache client** that picks which cache server holds a given key, typically using **consistent hashing** (see [Consistent Hashing](06_Consistent_Hashing_Notes.md)) so that adding or removing a cache server only moves a few keys.

### The five caching strategies
They differ in **who talks to the cache/DB and in what order**.

**1. Cache-aside (lazy loading)** – *the app manages the cache.*
Read: app checks the cache → hit: return it; miss: read the DB, put the value in the cache, return it.
- ✅ Good for read-heavy apps; if the cache is down the app still works (reads the DB); cache data can have a different shape from the DB.
- ❌ The first read of new data is always a miss; the cache can go **stale** if updates aren't handled.

**2. Read-through** – *the cache manages the DB read.*
Same idea, but the **cache library itself** fetches from the DB on a miss, so the app only talks to the cache.
- ✅ Simpler app code (loading logic is hidden).
- ❌ First read is a miss; can go stale; the cache's structure must match the DB table.

**3. Write-around** – *write straight to the DB.*
On a write, update only the DB and **invalidate (delete)** the cache entry. The next read will reload fresh data.
- ✅ No stale data; good for read-heavy workloads.
- ❌ The next read after a write is a miss; if the DB is down, the write fails.

**4. Write-through** – *write to cache and DB together.*
Write to the cache **and then synchronously to the DB** before returning; if either fails, raise an error.
- ✅ Cache and DB stay consistent; high hit rate.
- ❌ Slower writes (two writes); usually combined with read-through or cache-aside; if the DB is down, writes fail.

**5. Write-back (write-behind)** – *write to cache now, DB later.*
Write to the cache, put the DB write on a **queue**, and return immediately. A worker writes to the DB afterwards.
- ✅ Very fast writes; great for write-heavy apps; survives a short DB outage because the queue and cache hold the data.
- ❌ If the cache entry expires or the cache fails **before** the DB is updated, data can be **lost**. Mitigate with a longer TTL and a durable queue.

### Which to choose?
| Need | Pick |
|---|---|
| Mostly reads, simple | Cache-aside (or read-through) |
| Never serve stale data | Write-through or write-around |
| Many writes, low latency | Write-back |
| Best overall | Combine, e.g. **read-through + write-back** |

(The notes also list **eviction policies** – LRU, FIFO, LFU – which decide what to throw out when the cache is full; the handwritten pages don't detail them.)
<!-- simple-explanation -->


## 1. What is caching?
- **A technique to store frequently used data in a fast-access memory rather than accessing data every time from slow-access memory.**
- Makes the system **very fast**, **reduces latency**, and also helps to achieve **fault tolerance**.

**Caching exists at different layers of a system:**
- **Client-side caching** (browser caching)
- **CDN** (stores static data)
- **Load balancer**
- **Server-side application caching** (like **Redis**, etc.)

Basic picture: `Client → LB → App servers → Cache (Redis) → DB`.

---

## 2. What is distributed caching?
**Single cache service problem:** all app servers (1, 2, 3) hit one cache service – **limited space/resources**.
Limitations: ① **scalability**, ② **single point of failure.**

**Distributed caching** to the rescue:
```
 App Server ── Cache Client ──► Cache Pool: Cache1 | Cache2 | Cache3 | Cache4 | Cache5
```
- Each **app server has its own cache client**, and using that client it gets **one of the cache servers** where it stores or reads the data.
- **How is one of the servers allotted? – Consistent Hashing technique** (cache servers CS1…CS5 placed on a hash ring; a key is stored on the next server clockwise; "App Server 1 → put"). See [Consistent Hashing](06_Consistent_Hashing_Notes.md).

---

## 3. Caching strategies

### ① Cache-Aside (lazy loading)
Flow (`Client → Server → Cache → DB`):
1. Client sends **read request (GET)** to the server.
2. Server **reads from cache**.
3. **Cache hit** → data returned to client. **Cache miss** → **fetch data from DB**, **write into cache**, then send response.

**Pros**
- Good for **heavy-read applications.**
- **Even if the cache is down, the request won't fail** – it fetches from DB.
- **The cache document structure can differ** from the data in the DB.

**Cons**
- For **new data, the first read is always a cache miss** (resolve by **pre-heating** the cache).
- If the cache is not handled properly **during write operations**, there is a chance of **inconsistency between cache and DB** (e.g. cache holds a stale value of id 10 while the DB has the latest value after an update).

### ② Read-Through
- Application first checks the cache.
- **Cache hit** → return data.
- **Cache miss** → **the cache library itself fetches the data from the DB, stores it back in the cache and returns it to the application.**

**Pros**
- Good for **heavy-read applications.**
- **Logic of fetching from the DB and updating the cache is separated from the application.**

**Cons**
- For new data, **first read is always a cache miss** (pre-heat the cache).
- If caching is not used properly during writes → **inconsistency between cache and DB.**
- **Cache document structure should be the same as the DB table.**

### ③ Write-Around
- **Directly writes data into the DB; doesn't touch the cache.**
- It does **not update the cache data**; it **invalidates the cached entry (marks it dirty).**
- Flow: Client → **write (PUT / PATCH)** → Server → **directly write to DB** → **invalidate the data in cache.**

**Pros**
- Good for **heavy-read applications.**
- **Resolves the inconsistency problem between cache and DB.**

**Cons**
- For **new data read there will always be a cache miss** (pre-heat the cache).
- **If the DB is down, the write operation fails.**

### ④ Write-Through
- **First writes data into the cache, then synchronously writes data into the DB.**
- Flow: Client → write request (**POST**) → Server → first write into cache → then write into DB.
- **If any of the insertions fails, throw an exception.**

**Pros**
- **Cache and DB always remain consistent.**
- **Cache hit chance increases a lot.**

**Cons**
- **Alone it's not useful – it increases latency** (that's why it's always used together with **Read-through or Cache-aside**).
- **2-phase commit needs to be supported with this** to maintain the transactional property.
- **If the DB is down, the write fails** – *it's still not fully fault tolerant.*

### ⑤ Write-Back (Write-Behind)
- **First writes data into the cache, then asynchronously writes data into the DB.**
- Flow: Client → write (POST) → Server → first write into cache → **push the message to a Queue** → (later) **write into DB**.
- Example: if the DB is down, the **cache (turn-around/TTL ≈ 24 h)** keeps the data and the **queue holds the writes for e.g. 2 h**; **when the DB comes back the queue writes into the DB** → **brings fault tolerance.**

**Pros**
- **Good for write-heavy applications.**
- **Improves write latency** because writing into the DB happens asynchronously.
- **Cache hit chance increases a lot.**
- **Gives much better performance when used with Read-through cache.**

**Cons**
- **If data is removed from the cache (TTL expiry) and the DB write still hasn't happened, there's a chance of data loss/issue** (e.g. TTL = 3 h but the DB has been down for 5 h). **Handled by keeping the cache TTL a little higher, e.g. 2 days.**

---

## Quick revision
| Strategy | Read path | Write path | Best for | Watch out |
|---|---|---|---|---|
| **Cache-aside** | app checks cache, on miss loads DB + fills cache | app writes DB (+ must update/invalidate cache) | read-heavy; resilient if cache is down | cold start misses; stale cache |
| **Read-through** | cache library loads from DB on miss | – | read-heavy; app code simpler | cold start; schema must match DB |
| **Write-around** | – | write DB only, **invalidate cache** | read-heavy; no stale data | DB down ⇒ write fails; first read misses |
| **Write-through** | – | write cache **then** DB synchronously | consistency; high hit-rate | extra latency; needs 2PC; DB down ⇒ fail |
| **Write-back** | – | write cache, **async** to DB via queue | write-heavy, low write latency | data loss if cache evicts before DB write |

- Distributed cache = **cache pool + cache client + consistent hashing** (avoids SPOF & scales).
- Combine strategies: e.g. **Read-through + Write-back** for best performance.

---

<!-- nav -->
⬅ [18 Load Balancer and Algorithms](18_Load_Balancer_and_Algorithms_Notes.md) · [Index](00_HLD_Index.md) · [20 Distributed Transactions](20_Distributed_Transactions_Notes.md) ➡
<!-- nav -->
