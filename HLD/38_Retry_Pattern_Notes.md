# Retry Pattern (HLD #38)

<!-- nav -->
⬅ [37 Bulkhead](37_Bulkhead_Notes.md) · [Index](00_HLD_Index.md) · [39 Circuit Breaker](39_Circuit_Breaker_Notes.md) ➡
<!-- nav -->

Source: https://drive.google.com/file/d/1F4e2EwjCfi11-q7RdujOqiPgt7oCbO7B/view?usp=drive_link

---

<!-- simple-explanation -->
## In simple words – the retry pattern

### What problem does it solve?
When your service calls another service, the call can fail for a **transient (temporary)** reason: a network glitch, a brief timeout, a server restarting. If you simply try again after a short pause, it often works. That is the **retry pattern**.

**"Why not let the client call our API again?"** By the time the downstream call fails, your service may have already done 90 % of the work (queries, calculations, memory). If the client re-sends the whole request you repeat all of that work. Retrying only the failed downstream call is cheaper and gives the user a smoother experience.

Two questions to answer: **when** should we retry, and **how** (how long to wait)?

### When to retry
**Retry only if the failure might go away on its own.**
- ❌ **Don't retry client errors (4xx)** such as 400 (bad input), 401, 403, 404. The same request will fail again. A 409 (conflict) also won't fix itself.
- ✅ **Retry server errors (5xx)**: 500 (internal error), 502 (bad gateway), 503 (service unavailable). They may be temporary.
- ✅ **429 (Too Many Requests)** is also retryable, but **with a delay**.
- ⚠️ **Network errors and timeouts** are retryable **only if the operation is idempotent** (repeating it causes no harm). If a non-idempotent call (like a payment) times out, you don't know whether it went through. A blind retry could duplicate it. See [Idempotent POST API](14_Idempotent_POST_API_Notes.md).

### How to retry – the four strategies
`maxAttempts` counts the **original call too** (4 attempts = 1 original + 3 retries).

**1. Fixed interval** – wait the same time every time (e.g. 2 s, 2 s, 2 s).
- ✅ Simple, easy to debug.
- ❌ If many clients fail together they all retry together, creating a **retry storm** (the [thundering herd](27_Thundering_Herd_Notes.md)).

**2. Exponential backoff** – the wait **doubles** after each failure: `delay = base × 2ⁿ`. With base 100 ms: 100 ms, 200 ms, 400 ms…
- ✅ Gives the struggling service more and more breathing room.
- ❌ A short outage may make the user wait longer than needed, and all clients still use the same formula, so some herd risk remains.

**3. Exponential backoff + jitter** – the same, but each wait is a **random value** between 0 and the calculated delay: `random(0, min(maxDelay, base × 2ⁿ))`.
- ✅ Retries spread out over time, so the herd is very unlikely. **Best for distributed systems.**
- ❌ A bit harder to reason about and debug because delays vary.

**4. Custom** – you write your own delay function (e.g. Fibonacci delays).
- ✅ Full control. ❌ More to maintain.

### Using it in Spring Boot (Resilience4j)
- Put `@Retry(name = "productService", fallbackMethod = "productFallback")` on a method that contains **only the downstream call**, so only that part is repeated.
- The method must **throw the exception** for a retry to trigger (don't swallow it in a try/catch without rethrowing).
- When all attempts are used up, the **fallback method** runs, so you can return a friendly default.
- Fixed interval config: `maxAttempts=3`, `waitDuration=2s`.
- Exponential: add `enableExponentialBackoff=true` and `exponentialBackoffMultiplier=2`.
- With jitter: also add `enableRandomizedWait=true`.
- Custom strategies can't be configured in properties, so you create a `Retry` bean from a `RetryConfig` (with your `IntervalFunction`) and wrap the call manually with `customRetry.executeSupplier(...)`.

### Where it sits in the stack
RateLimiter → Bulkhead → TimeLimiter → CircuitBreaker → **Retry**. Retry is applied last so that it doesn't amplify traffic that the earlier protections would have blocked.
<!-- simple-explanation -->


## 1. Why Retry?

<!-- diagram:38-0 -->
**Diagram – Retry only the failed downstream call**

```mermaid
sequenceDiagram
  participant C as Client
  participant O as Order service
  participant P as Product service
  C->>O: request
  O->>O: 90% of the work done
  O->>P: call
  P-->>O: transient failure (5xx / timeout)
  Note over O: wait, then retry (no need to redo the 90%)
  O->>P: retry
  P-->>O: success
  O-->>C: response
```
<!-- /diagram:38-0 -->


In a distributed system, a call to a downstream service might fail because of **transient issues** – short, temporary problems like network glitches or timeouts. **Retrying the same request after a short delay can succeed.**

**"Why should *we* retry? Why can't the client just call our API again?"**
Each call does a lot of heavy processing that is **not free** (memory, time, resources). Say 90 % of the API work is done and then a transient issue hits during the downstream call:
- Throwing away that 90 % and redoing it all on a fresh API call wastes cost.
- Retrying only the failed downstream call, if it succeeds, **saves cost and gives a better user experience.**

Two questions: **When to retry?** and **How to retry?**

---

## 2. When to retry?

- **Do NOT retry on permanent failures** – e.g. **4xx** errors. They are validation / input errors; the same request will fail again.
- **Do NOT retry if the operation is non-idempotent.** Idempotency ensures that if a request is retried multiple times, **duplicate processing does not happen.**
  - Problem scenario: a timeout happens on an endpoint that is **not idempotent** – what if the user (or we) retry? Could create duplicates. (How to design idempotent APIs: see HLD #14 *Idempotent POST API*.)
- **Rule of thumb: retry on 5xx errors** – 5xx means a system issue and the request wasn't successful; could be transient and is pretty safe to retry.
- **Retry network errors and timeouts only if the endpoint ensures idempotency.**

### Retry decision table

<!-- diagram:38-1 -->
**Diagram – Should I retry?**

```mermaid
graph TD
  E["Call failed"] --> Q1{"4xx client error?"}
  Q1 -->|"yes"| N["Do not retry"]
  Q1 -->|"no"| Q2{"5xx or 429?"}
  Q2 -->|"yes"| Y["Retry (429: with a delay)"]
  Q2 -->|"no"| Q3{"network error or timeout?"}
  Q3 -->|"yes"| Q4{"endpoint idempotent?"}
  Q4 -->|"yes"| Y
  Q4 -->|"no"| N
```
<!-- /diagram:38-1 -->


| HTTP status / event | Meaning | Retry? |
|---|---|---|
| 200 OK | Success | No |
| 400 Bad Request | Client error (invalid input) | No |
| 401 Unauthorized | Authentication needed | No |
| 403 Forbidden | Access denied | No |
| 404 Not Found | Resource not found | No |
| 409 Conflict | (listed as "Resource not found" in the notes – really a state conflict) | No |
| 429 Too Many Requests | Rate limiting | **Yes (with delay)** |
| 500 Internal Server Error | Server issue | **Yes** |
| 502 Bad Gateway | Downstream service failure | **Yes** |
| 503 Service Unavailable | Service down/overloaded | **Yes** |
| Network error | Connection failure, timeout | **Yes (idempotent only)** |

---

## 3. How to retry? Types of retry

### 3.1 Fixed Interval

<!-- diagram:38-2 -->
**Diagram – Fixed interval: 2 s every time**

```mermaid
graph LR
  A["original call"] --> W1["wait 2 s"] --> R1["retry 1"] --> W2["wait 2 s"] --> R2["retry 2"] --> W3["wait 2 s"] --> R3["retry 3"]
```
<!-- /diagram:38-2 -->


Retry after a **constant delay**.
Example: total attempts = 4 (original + 3 retries), fixed delay 2 s → original call at t=2, 1st retry at 4, 2nd retry at 6, 3rd retry at 8.

- **Advantages:** very simple; easy to configure and debug.
- **Disadvantage:** high chance of a **retry storm (Thundering Herd problem)** – many clients retry at the same time → surge of retry traffic → the system spends resources on retries instead of new requests. (Thundering Herd is covered in HLD #27.)

**Code**
```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    @Autowired OrderService orderService;
    @GetMapping("/{id}")
    public void callProductAPI(@PathVariable String id) { orderService.invokeProductAPI(id); }
}

@FeignClient(name = "product-service")
public interface ProductClient {
    @GetMapping(value = "/products/{id}")
    String getProductById(@PathVariable("id") String id);
}

@Component
public class OrderService {
    @Autowired ProductClient productClient;

    /* @Retry is applied on top of the method where ONLY the downstream invocation is present.
       Make sure the method contains only the code you want to retry.
       When all retries are exhausted, the fallback method is invoked. */
    @Retry(name = "productService", fallbackMethod = "productFallback")
    public void invokeProductAPI(String id) {
        String response = productClient.getProductById(id);
    }

    public void productFallback(String id, Throwable t) {
        System.out.println("Product Service is busy");
    }
}
```
```properties
server.port=8081
spring.application.name=order-service
eureka.client.service-url.defaultZone=http://localhost:8761/eureka

# fixed delay interval
resilience4j.retry.instances.productService.maxAttempts=3
resilience4j.retry.instances.productService.waitDuration=2s
```
**Testing:** stop the Product service so the failure happens at invocation; add a try/catch with a `System.out.println("not able to invoke Product")` and **rethrow** `e` (so retry triggers). **Output:** max attempts is 3 → **1 original call + 2 retry calls**.

**What AOP's proxy might do internally**
```java
public Retry fixedRetry() {
    RetryConfig config = RetryConfig.custom()
        .maxAttempts(4)
        /* IntervalFunction helps determine the delay between retries.
           Pre-defined functions:
            - IntervalFunction.of(long intervalMillis)                       // fixed delay
            - IntervalFunction.ofExponentialBackoff(long initialIntervalMillis,
                                   double multiplier, long maxIntervalMillis)
            etc. */
        .intervalFunction(IntervalFunction.of(2000))
        /* Retry will only happen for these exceptions or their subclasses. */
        .retryExceptions(IOException.class, ArithmeticException.class)
        .build();
    return Retry.of("fixedRetry", config);
}
```

### 3.2 Exponential Backoff

<!-- diagram:38-3 -->
**Diagram – Exponential backoff: base x 2^n**

```mermaid
graph LR
  A["original call"] --> W1["wait 100 ms"] --> R1["retry 1"] --> W2["wait 200 ms"] --> R2["retry 2"] --> W3["wait 400 ms"] --> R3["retry 3"]
```
<!-- /diagram:38-3 -->


**Delay increases exponentially:** `delay = Base × 2^(no. of failed retry attempts)`

Example: total attempts = 4 (original + 3 retries), Base = 100 ms
- 1st retry: 100 × 2⁰ = **100 ms**
- 2nd retry: 100 × 2¹ = **200 ms**
- 3rd retry: 100 × 2² = **400 ms**

- **Advantages:** reduces load on downstream during an outage; helps avoid Thundering Herd **to some extent**.
- **Disadvantages:** if the outage is short, the user may wait longer than needed; **still some risk of Thundering Herd** because all clients use the same formula (`Base × 2^retry attempt`).

Code is all the same, only config changes:
```properties
# exponential backoff
resilience4j.retry.instances.productService.maxAttempts=4
resilience4j.retry.instances.productService.waitDuration=1s
resilience4j.retry.instances.productService.enableExponentialBackoff=true
resilience4j.retry.instances.productService.exponentialBackoffMultiplier=2
```
Framework class `Backoff.java`:
```java
try {
    /* Formula: initial wait time * (factor ^ no. of failed retry attempts)
       1st retry = 1000ms * (2^0) = 1000ms = 1sec delay
       2nd retry = 1000ms * (2^1) = 2000ms = 2sec delay
       3rd retry = 1000ms * (2^2) = 4000ms = 4sec delay */
    nextBackoff = firstBackoff.multipliedBy((long) Math.pow(factor, (context.iteration() - 1)));
    if (nextBackoff.compareTo(maxBackoffInterval) >= 0) {
        nextBackoff = maxBackoffInterval;
    }
}
```
**Output:** max attempts 4 → 1 original + 3 retries.

### 3.3 Exponential Backoff + Jitter

<!-- diagram:38-4 -->
**Diagram – Backoff with jitter: random wait up to the limit**

```mermaid
graph LR
  A["original call"] --> W1["wait random(0, 100 ms)"] --> R1["retry 1"] --> W2["wait random(0, 200 ms)"] --> R2["retry 2"] --> W3["wait random(0, 400 ms)"] --> R3["retry 3"]
```
<!-- /diagram:38-4 -->


Like exponential backoff but adds **jitter / randomness** to the delay to avoid retry storms.

`delay = random_between(0, min(maxDelay, baseDelay × 2^failed_retry_attempt))`

- **Advantages:** best for distributed systems – Thundering Herd probability becomes very low; **spreads retries over time**, reducing peak load.
- **Disadvantages:** not easy to debug; a little complex.

```java
public Retry exponentialJitterRetry() {
    RetryConfig config = RetryConfig.custom()
        .maxAttempts(6)
        .intervalFunction(
            /* Use the appropriate IntervalFunction method for the delay computation */
            IntervalFunction.ofExponentialRandomBackoff(
                /* initialIntervalMillis */ 2000,
                /* multiplier */ 2))
        .retryExceptions(Exception.class)
        .build();
    return Retry.of("exponentialBackOffRetry", config);
}
```
```properties
server.port=8081
spring.application.name=order-service
eureka.client.service-url.defaultZone=http://localhost:8761/eureka

# exponential backoff + jitter
resilience4j.retry.instances.productService.maxAttempts=4
resilience4j.retry.instances.productService.waitDuration=1s
resilience4j.retry.instances.productService.enableExponentialBackoff=true
resilience4j.retry.instances.productService.exponentialBackoffMultiplier=2
resilience4j.retry.instances.productService.enableRandomizedWait=true
```

### 3.4 Custom Retry
Define **your own retry delay strategy**.
- **Advantage:** full control and flexibility. **Disadvantage:** harder to maintain.

```java
@Configuration
public class Config {
    @Bean
    public Retry customRetry() {
        /* IntervalFunction defines the wait time between retries. */
        IntervalFunction fibonacciInterval = attempt -> {
            return 2000L; // return milliseconds as long
        };

        /* RetryConfig holds all configs: max attempts, which IntervalFunction to use, etc. */
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(6)
            .intervalFunction(fibonacciInterval)
            .retryExceptions(Exception.class)
            .build();

        /* Create the Retry object from RetryConfig. Any name can be given. */
        return Retry.of("customRetry", config);
    }
}

@Component
public class OrderService {
    @Autowired ProductClient productClient;
    @Autowired Retry customRetry;

    public void invokeProductAPI(String id) {
        try {
            /* Need to manually wrap the call within Retry, as the @Retry annotation
               works only for retry types whose config can be given in application.properties. */
            customRetry.executeSupplier(() -> {
                System.out.println("calling product service at " + java.time.LocalTime.now());
                return productClient.getProductById(id);
            });
        } catch (Exception e) {
            System.out.println("all retries failed, this is fallback");
        }
    }
}
```

| Strategy | Delay | Pros | Cons |
|---|---|---|---|
| Fixed interval | constant | simple, easy to debug | retry storm / thundering herd |
| Exponential backoff | Base × 2^n | less load on downstream | slow for short outages; some herd risk |
| Exp. backoff + jitter | random(0, min(max, Base×2^n)) | best for distributed systems | harder to debug |
| Custom | your function | full control | harder to maintain |

---

## Quick revision
- Retry **transient** failures only: 5xx, 429 (with delay), network errors **if idempotent**; never 4xx or non-idempotent ops.
- `@Retry(name, fallbackMethod)` – annotate a method containing only the downstream call; method must **throw** to trigger retry; fallback after all attempts fail.
- `maxAttempts` includes the original call.
- Prefer **exponential backoff + jitter** in distributed systems to avoid thundering herd.
- Custom strategies need `Retry` bean + `executeSupplier(...)`.
- Order of patterns: RateLimiter → Bulkhead → TimeLimiter → CircuitBreaker → Retry.

---

<!-- nav -->
⬅ [37 Bulkhead](37_Bulkhead_Notes.md) · [Index](00_HLD_Index.md) · [39 Circuit Breaker](39_Circuit_Breaker_Notes.md) ➡
<!-- nav -->
