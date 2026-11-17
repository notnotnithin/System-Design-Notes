# Thundering Herd Problem (HLD #27 – Lecture 64)

<!-- nav -->
⬅ [26 JWT Explained](26_JWT_Explained_Notes.md) · [Index](00_HLD_Index.md) · [28 API Gateway Concepts](28_API_Gateway_Concepts_Notes.md) ➡
<!-- nav -->

Source (handwritten, 2 pages): https://drive.google.com/file/d/1s_rhkuSserKURoOTwolrnMGifGHd9oKI/view?usp=drive_link

---

<!-- simple-explanation -->
## In simple words – the thundering herd problem

### The story
It's 12:00 PM and tickets for a popular show go on sale. Thousands of people hit **Book** at the same instant. The booking app has a few servers, each with a small waiting line (queue) for requests.

1. The servers are already **fully busy** and their queues are **full**.
2. New requests are **rejected** ("server busy").
3. Rejected users – and the app itself – **retry immediately**.
4. Now the servers face the **original traffic plus all the retries**, still at the same moment.
5. They reject even more, which triggers even more retries… The overload feeds itself and the whole app goes down.

That stampede of simultaneous requests and retries is the **thundering herd**. The retries meant to *help* are what keep the system down.

### Fix 1 – Exponential backoff
Make each client wait **longer after every failure**: the delay is `base × 2ⁿ`, where *n* is how many times it already failed. With a base of 100 ms:
- 1st retry → wait about 200 ms
- 2nd retry → wait about 400 ms
- 3rd retry → about 800 ms…

The retries spread out in time and the load on the server drops. **But** there's still a catch: if 1,000 clients all failed at the same moment, they all wait exactly 200 ms, then exactly 400 ms… so they still arrive together in "waves".

### Fix 2 – Exponential backoff **with jitter**
**Jitter** means *randomness*. Instead of waiting exactly `base × 2ⁿ`, each client waits a **random time between 0 and that value** (and never more than a maximum, e.g. 30 s). Now the 1,000 clients retry at 1,000 slightly different moments, so the herd turns into a smooth trickle. This is the standard recommendation for distributed systems.

### Fix 3 – Protect the front door with rate limiting
Put an **API gateway with a rate limiter** (e.g. the *token bucket* algorithm) in front of the load balancer. It only lets through the number of requests the servers can actually handle and rejects the rest quickly and cheaply. The servers never get flooded in the first place; excess users get a polite "try again shortly" message.

### Quick summary
| Problem | Cause | Cure |
|---|---|---|
| Many clients retry together | fixed or no waiting | backoff + **jitter** |
| Servers swamped at peak | no limit on incoming load | **rate limiting** at the gateway |
<!-- simple-explanation -->


## 1. Problem: "Why does a ticket-booking app go down during peak traffic?"

**Scenario:** the **12:00 PM sale has started** on a ticket-booking app.

Diagram in notes:
```
 new requests  + retried requests
          \   |   /
           [ Load Balancer ]      (annotated:  Min = 10, Max = 100)
            /     |     \
        TS 1     TS 2     TS 3        ← ticket-service instances,
        [queue]  [queue]  [queue]        each with its own request queue
```
- Every instance's **queue is also filled** and **all instance threads are busy**.
- When more requests come, **nothing can accept a new request**. What happens:
  1. **New requests get denied.**
  2. **Clients retry** – the denied requests come back again, on top of the new ones.
- So the load balancer is hit by *new requests **plus** retried requests*, all at the same moment → the surge feeds itself → the system **goes down**. This is the **Thundering Herd problem**: many clients retry at the same time, causing a surge of retry traffic.

---

## 2. Solutions

### ① Exponential Backoff
Make each retry wait longer than the previous one:

`delay = Base × 2ⁿ`   where **n = number of failures (retry attempt)**

Example with Base = 100 ms:
- Retry 1 ⇒ 100 × 2¹ = **200 ms**
- Retry 2 ⇒ 100 × 2² = **400 ms**

### ② Exponential Backoff + Jitter
Exponential backoff alone still makes all clients retry at the same moments (same formula). **Bring some randomness**:

`delay = min( MaxWaitTime , Random(0, Base × 2ⁿ) )`

This **spreads the retries over time** so the herd doesn't arrive together.

### ③ Protect the entry with a Rate Limiter (Application / API Gateway)
```
 Application Gateway ─ [Rate Limiter]  (Token Bucket algo)
          │
     Load Balancer
      /    |    \
   TS1    TS2    TS3
```
Put an **API/application gateway with a rate limiter (Token Bucket algorithm)** in front of the load balancer so that only an allowed rate of requests reaches the services.

---

## Quick revision
- Thundering herd = **synchronised retries (+ new traffic) overwhelm an already-saturated system**.
- Fixes: **exponential backoff**, **backoff + jitter (randomised delay)**, **rate limiting at the gateway** (token bucket).
- Related notes: [Retry Pattern](38_Retry_Pattern_Notes.md), [Rate Limiter](12_36_Rate_Limiter_Notes.md), [API Gateway](40_41_API_Gateway_Part1_Part2_Notes.md).

---

<!-- nav -->
⬅ [26 JWT Explained](26_JWT_Explained_Notes.md) · [Index](00_HLD_Index.md) · [28 API Gateway Concepts](28_API_Gateway_Concepts_Notes.md) ➡
<!-- nav -->
