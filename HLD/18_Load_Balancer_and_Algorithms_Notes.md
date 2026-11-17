# Load Balancer & Different Algorithms (HLD #18 – Lecture 55)

<!-- nav -->
⬅ [17 Proxy vs Reverse Proxy](17_Proxy_vs_Reverse_Proxy_Notes.md) · [Index](00_HLD_Index.md) · [19 Distributed Cache and Caching Strategies](19_Distributed_Cache_and_Caching_Strategies_Notes.md) ➡
<!-- nav -->

Source (handwritten, 7 pages): https://drive.google.com/file/d/1dAg77qQ1t9mlHXc1VvO-dZXvaGWnvmaZ/view?usp=drive_link

---

<!-- simple-explanation -->
## In simple words – load balancer and its algorithms

### What is a load balancer and why do we need one?
If all clients send their requests to one server, that server gets overloaded and slows down or crashes. So we run **several servers** and put a **load balancer (LB)** in front. It receives every request and **forwards it to one of the servers**, so the work is spread out and no single server is overburdened. (It can also log traffic and do other jobs, but sharing the load is its main purpose.)

### Layer 4 vs Layer 7
A load balancer can work at different levels of the network:
- **L4 (transport layer) – network load balancer:** it only looks at **IP addresses and TCP/UDP ports**. It cannot read the content of the request. Because it does so little, it is **very fast**.
- **L7 (application layer) – application load balancer:** it can read the **headers, cookies, session and even the response**. So it can make smarter choices (e.g. send `/images` to one group of servers and `/payments` to another) and can also cache. It is **more powerful but a bit slower**.

### How does the LB pick a server? (algorithms)
Two families: **static** (fixed rules, don't look at the server's current state) and **dynamic** (look at the live load).

#### Static algorithms
1. **Round Robin** – take turns: request 1 → Server 1, request 2 → Server 2, request 3 → Server 1…
   - ✅ Very simple and shares requests equally.
   - ❌ Treats a big powerful server and a small weak server the same, so the weak one may get overloaded.
2. **Weighted Round Robin** – like round robin, but each server has a **weight based on its capacity**. A server with weight 3 gets 3 requests for every 1 that a weight-1 server gets.
   - ✅ Protects weaker servers; still easy.
   - ❌ Weights are fixed. If some requests are very heavy (taking 10–20 s) and happen to land on the small server, it still gets overloaded.
3. **IP Hash** – compute a hash of the client's IP address and use it to choose the server. The same client always lands on the same server.
   - ✅ Great when a user needs to stay on the same server (sticky sessions).
   - ❌ If many users sit behind one proxy, they all share one IP and hit the same server. It also can't guarantee an even spread.

#### Dynamic algorithms
1. **Least Connection** – send the request to the server that currently has the **fewest active connections**.
   - ✅ Adapts to the live situation.
   - ❌ A connection can be open but idle, which misleads the count, and it ignores server size.
2. **Weighted Least Connection** – compute `active connections ÷ weight` for each server and pick the lowest.
   Example: Server 1 has weight 10 and 2 connections → 0.2; Server 2 has weight 1 and 1 connection → 1. Server 1 gets the request, because relative to its capacity it is much less busy.
3. **Least Response Time** – uses **TTFB (Time To First Byte)**, the time between sending a request and receiving the first byte of the reply. It prefers the server with fewer connections and the quickest responses. If scores tie, it falls back to round robin.

### Summary
Start simple with round robin; use weights when servers differ; use IP hash for stickiness; use dynamic algorithms when request cost varies a lot.
<!-- simple-explanation -->


## 1. Why a load balancer?
Clients send requests; **not all requests should go to only one server** – one server must not get overloaded, so we **put a Load Balancer (LB)** between clients and servers (S1, S2, …).

> **Main purpose of a load balancer:** it can do logging and many other things, but the main purpose is to **distribute the traffic appropriately across all the servers so that no single server is overburdened.**

## 2. Layers: L4 vs L7 (OSI model)
- **Application layer = Layer 7 (L7) balancer** – can read **headers, session, data, response**.
- **Transport layer = Layer 4 (L4) balancer** – can see **TCP port** (UDP port, IP address, source and destination).

| | **Network LB (L4)** | **Application LB (L7)** |
|---|---|---|
| Decision based on | **TCP port / UDP port / IP address, source & destination** – decides only from this which server gets the traffic | **Header, session, cookies, data** |
| Extras | – | Can also do **caching**, because it **can read the response** |
| Speed | **Much faster** | **Much more advanced** |

## 3. Algorithms (for the network LB) – static vs dynamic
```
 Static algorithms                    Dynamic algorithms
  • Round Robin                        • Least Connection
  • Weighted Round Robin               • Weighted Least Connection
  • IP Hash                            • Least Response Time
```

---

# STATIC LOAD BALANCING

## ① Round Robin
Requests are handed to servers **in turn**: with 4 clients and 2 servers → requests 1, 3, 5, 7 → **Server 1**; 2, 4, 6, 8 → **Server 2**.

**Advantages**
- Very easy to implement.
- Equal load distribution to all servers.

**Disadvantages**
- One server with **high capacity** and another with **low** capacity are **treated the same.**
- Chance that the **low-capacity server will go down** because of request overload.

## ② Weighted Round Robin
Each server has a **weight = its capacity** – e.g. **Server 1 weight = 3×**, **Server 2 weight = 1×** → for every 4 requests, Server 1 gets 3 (requests 1,2,3,5,6,7…) and Server 2 gets 1 (4, 8…).

**Advantages**
- **Low-capacity server is saved** from receiving a large number of requests.
- **Easy to implement** – weights are **static**, no dynamic computation.

**Disadvantages**
- If **requests have different processing times** (e.g. request 4 → 10 s, request 8 → 20 s while others take ~50 ms), the low-capacity server may get the heavy requests and **get overburdened.**

## ③ IP Hash
The LB computes **Hash(source IP address)** and, using that value, **assigns the request to a server** (e.g. clients `192.178.10.4`, `182.178.14.2`, `172.178.10.242`, `152.198.10.14` → hash → server 1 or 2).

**Advantages**
- Good where the **same client needs to connect to the same server** (session stickiness).
- Easy to implement.

**Disadvantages**
- If client requests come **through a proxy**, **all clients appear with the same source IP** (e.g. `162.16.0.1`) → **one server gets overloaded.**
- **Cannot ensure equal distribution.**

---

# DYNAMIC LOAD BALANCING

## ① Least Connection
The LB **checks which server has fewer active connections** and sends the new request there (Server 1 has active connections from Client 1 & 3; Server 2 from Client 2 → new request from Client 4 goes to Server 2).

**Advantages**
- **Dynamic** – it considers the **current load on each server**, so the chance of a server being overburdened is less **when every server has equal capacity.**

**Disadvantages**
- **TCP connections can be active but possibly carry no traffic** → purpose is defeated.
- **No difference between low-capacity and high-capacity servers** → the low-capacity server can get overburdened.

## ② Weighted Least Connection
- **Calculate the ratio of number of active connections to the server's weight.**
- **The server with the minimum ratio gets the request.**
- Example: Server 1 weight = 10 with 2 active connections → ratio `2/10 = 0.2`; Server 2 weight = 1 with 1 active connection → ratio `1/1 = 1` → **Server 1 gets the request.**

## ③ Least Response Time
- Uses **TTFB (Time To First Byte)** = the **time interval between sending a request and receiving the first byte of the response from the server.**
- **Picks the server which has fewer active connections and the least TTFB**; **if there is a clash, follow round robin.**
- Example: S1 → TTFB 3 × 2 active connections = **6**; S2 → TTFB 1 × 4 active connections = **4**; S3 → TTFB 2 × (0–1 active connection) = **2**/lowest → choose the smallest score.
  *(the S3 figures are partly illegible in the scan)*

---

## Quick revision
| Algorithm | Type | Key idea | Weakness |
|---|---|---|---|
| Round Robin | static | take turns | ignores capacity |
| Weighted RR | static | more requests to bigger servers (static weights) | ignores differing request cost |
| IP Hash | static | hash(client IP) → server (sticky) | proxies share one IP → imbalance |
| Least Connection | dynamic | fewest active connections | idle-but-open connections; ignores capacity |
| Weighted Least Connection | dynamic | min(active connections ÷ weight) | – |
| Least Response Time | dynamic | lowest TTFB (× active connections) | needs measurement; ties → round robin |

- **L4** LB = fast, routes on IP/port. **L7** LB = smart, routes on headers/cookies/data and can cache.
- Client-side LB in Spring: see [Client Side Load Balancer](35_Client_Side_Load_Balancer_Notes.md).

---

<!-- nav -->
⬅ [17 Proxy vs Reverse Proxy](17_Proxy_vs_Reverse_Proxy_Notes.md) · [Index](00_HLD_Index.md) · [19 Distributed Cache and Caching Strategies](19_Distributed_Cache_and_Caching_Strategies_Notes.md) ➡
<!-- nav -->
