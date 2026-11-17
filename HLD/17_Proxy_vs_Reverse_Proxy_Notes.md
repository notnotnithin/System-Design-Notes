# Proxy vs Reverse Proxy – and how a proxy differs from VPN, Load Balancer, Firewall (HLD #17 – Lecture 54)

<!-- nav -->
⬅ [16 Distributed Messaging Queue](16_Distributed_Messaging_Queue_Notes.md) · [Index](00_HLD_Index.md) · [18 Load Balancer and Algorithms](18_Load_Balancer_and_Algorithms_Notes.md) ➡
<!-- nav -->

Source (handwritten, 6 pages): https://drive.google.com/file/d/18wyJRj2g11NE1Ll1c6GJSiqnhwSao01U/view?usp=drive_link

**Topics covered:** What is a proxy server · its types (Forward proxy, Reverse proxy) · Proxy vs VPN · Proxy vs Load Balancer · Proxy vs Firewall.

---

<!-- simple-explanation -->
## In simple words – proxy vs reverse proxy

### What is a proxy?
A **proxy** is a middleman. Instead of the client talking straight to the server, the client talks to the proxy, and the proxy talks to the server on its behalf. Think of a child who wants chocolate: the mother goes to the shop and brings it back. The shop only ever deals with the mother.

Both kinds below are middlemen. The difference is **whose side they are on**.

### Forward proxy – the middleman for *clients*
It sits in front of a group of clients (for example, all computers in an office) and sends their requests out to the Internet.
- When a computer asks for `google.com`, the proxy forwards the request using **its own IP address**. Google only sees the proxy, never the real computer. So the **clients stay hidden**.
- Useful things it gives you:
  1. **Anonymity** – your IP and location are not revealed to websites.
  2. **Grouping** – if many people ask for the same page, the proxy can fetch it once.
  3. **Bypassing restrictions** – if a site is blocked in your country, going through a proxy in another country makes the request look like it comes from there.
  4. **Security / filtering** – the company can block unwanted sites.
  5. **Caching** – popular pages are stored and served quickly.
- It works at the application level (it understands requests like HTTP).

### Reverse proxy – the middleman for *servers*
It sits in front of your backend servers. Users talk only to the reverse proxy, which decides which server should handle the request. Think of a restaurant: customers talk to the **waiter**, never directly to the **kitchen**.
- The **servers stay hidden**: the outside world only knows the proxy's address, not the servers' IPs.
- Benefits:
  - **Security** – attackers can't reach the real servers. In a DDoS attack, only the proxy takes the hit.
  - **Caching** – common responses are served directly, which also lowers latency.
  - **Load balancing** – it can spread requests across many servers.
  - **CDN** – a Content Delivery Network is basically many reverse proxies placed around the world. A user in Paris is served by the Paris CDN node, which fetches from the origin server (say in Singapore) only when needed.

**Easy memory trick:** forward proxy hides the *client*; reverse proxy hides the *server*.

### Proxy vs VPN
- A **proxy** mainly **masks your IP**, can cache and log. It usually does not encrypt your data.
- A **VPN** creates an **encrypted tunnel** between your device and the VPN server. Your data is encrypted before leaving your computer and decrypted at the VPN server, so no one in between can read it.

### Proxy vs load balancer
A **reverse proxy can also act as a load balancer**, but a plain load balancer only distributes traffic; it doesn't offer anonymity, caching or logging. So: reverse proxy ⊃ load balancer.

### Proxy vs firewall
A **firewall** is a rule-based gatekeeper. It inspects the **packet headers** (IP addresses, ports, source and destination) and allows or blocks traffic based on rules. A proxy, in contrast, passes on requests on someone's behalf. They are often used together.
<!-- simple-explanation -->


## 1. What is a Proxy Server?
**Analogy:** a child wants chocolate from the **shop** – instead of going directly, the **mom (proxy)** goes to the shop on the child's behalf and brings it back.

**Technical:** `Client 1 / Client 2 ⇄ Proxy Server ⇄ Server`

> A **proxy server sits between client and server. It acts like an intermediary** between the client and the server. Requests **pass through the proxy – there is no direct interaction** between client and server.

---

## 2. Types of proxy
They **differ in the direction of communication**: **Forward** and **Reverse**.

### ① Forward proxy (the "simple proxy")
- Sits in front of a **group of clients in a closed network** (clients C1 `172.1.01`, C2 `172.2.01`, C3 `172.3.01`) and connects them to the Internet.
- **What does a forward proxy do?** It **hides the client networks** – the outside server **doesn't know the internal client.**
- **How?** Say client 1 (IP `172.1.01`) makes a request (e.g. `google.com`). The proxy forwards it using **its own IP address** (e.g. `192.30.1`) – "*the proxy is making a request on behalf of C1*". The server only sees the proxy's IP when it gets the request.

**Advantages**
1. **Anonymity for the internal network** – the IP address and location of the clients are **hidden from the outside world**; no server can know where the request comes from.
2. **Grouping of requests** – a proxy can take care of multiple client requests. If several clients make the same `google.com` request, the proxy can **club/group them and send a single request** outside.
3. **Access restricted data or content** – if your country blocks certain content/websites (e.g. in India), with the help of a proxy you can **bypass** it: the request appears to come from a user in e.g. **UK / US**, so country-wise location restrictions don't apply.
4. **Brings security.**
5. **Caching** – the proxy can cache responses (the diagram shows a *Cache* attached to the proxy).

**Disadvantage:** it **works at the application level.**

### ② Reverse proxy
```
 Internet → [ Reverse Proxy ] → Server 1 / Server 2 / … / Server n
```
- A reverse proxy is **like a security guard for your backend servers.** Instead of users talking directly to your servers, they talk to the **reverse proxy, which then forwards the request to the appropriate server.**
- **Example:** a restaurant with a **waiter (reverse proxy)** – customers don't go directly to the **kitchen (backend servers)**; they order through the waiter, who talks to the kitchen and brings the food back.

**Advantages**
- **Security** – the outside world **isn't aware of the servers' IP addresses**; everyone only knows the **reverse proxy's address.**
- **In case of a DDoS attack** – the **attacker can only attack the reverse-proxy server.**
- **CDN is a reverse proxy:** local clients (Paris, US, India) go to their **local CDN** (CDN-Paris, CDN-US, CDN-India) which fetches from the origin server (e.g. in Singapore). An attacker can attack only a CDN, not the origin.
- Also capable of **caching**.
- **Latency** benefit (served from a nearby/cached location).
- **Load-balancer capability** – a reverse proxy can distribute requests across servers.

---

## 3. Proxy vs VPN
| | **Proxy** | **VPN** |
|---|---|---|
| Path | `Client → Proxy → Server` | `Client → VPN client (on the home computer) → encrypted VPN tunnel across the Internet → VPN server → Server` |
| What it gives | **Anonymity (IP address masking), caching, logging** | **Encryption of data** – data is encrypted at the client, travels through the tunnel, and is **decrypted at the VPN server** |

---

## 4. Proxy vs Load Balancer
- **A reverse proxy offers IP anonymity, caching, logging – and can act as a load balancer.**
- **A load balancer cannot be a proxy** (it only distributes traffic across Server 1 … Server n).

---

## 5. Proxy vs Firewall
- A **firewall** is like a wall with **holes that decide what data may pass** between the Internet and your network – it follows **rules**.
- It does **packet scanning** – looks at the packet **header: port number, IP address, source / destination** – and allows or blocks accordingly.

---

## Quick revision
- **Forward proxy** = client-side intermediary → **hides clients** (anonymity, group requests, bypass geo-blocks, caching; works at application level).
- **Reverse proxy** = server-side intermediary → **hides servers** (security, DDoS shield, caching, load balancing, CDN = reverse proxy).
- **VPN** = encrypts all traffic via a tunnel; **proxy** = masks IP/caches/logs.
- **Reverse proxy can act as a load balancer; a load balancer can't act as a proxy.**
- **Firewall** = rule-based packet filtering on header info (ports, IPs).

---

<!-- nav -->
⬅ [16 Distributed Messaging Queue](16_Distributed_Messaging_Queue_Notes.md) · [Index](00_HLD_Index.md) · [18 Load Balancer and Algorithms](18_Load_Balancer_and_Algorithms_Notes.md) ➡
<!-- nav -->
