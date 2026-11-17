# Circuit Breaker – Fault-Tolerant Microservice Part-4 (HLD #39)

<!-- nav -->
⬅ [38 Retry Pattern](38_Retry_Pattern_Notes.md) · [Index](00_HLD_Index.md) · [40 & 41 API Gateway Part1 Part2](40_41_API_Gateway_Part1_Part2_Notes.md) ➡
<!-- nav -->

Source: https://drive.google.com/file/d/1ApUVMYBXzsEiW_tCL_ZpfX0IGKZQ2IZe/view?usp=drive_link

---

<!-- simple-explanation -->
## In simple words – the circuit breaker

### The idea
In your house, a circuit breaker **cuts the power** when something goes wrong so that wires don't burn. A software circuit breaker does the same for calls to another service: when that service is clearly failing, it **stops calling it for a while**.

Why that helps. Suppose Order calls Product and Product is failing:
- Every extra call **adds load** to Product, so it takes **longer to recover**.
- Order **wastes its own threads and time** waiting on calls that will probably fail.
The breaker answers: *when should Order stop calling Product, and when should it try again?*

### The three states
- **CLOSED** (normal): the circuit is "closed" like a working switch, so **all calls go through**. The breaker silently counts successes and failures.
- **OPEN** (tripped): too many failures happened. Now **calls are blocked and fail immediately** (or return a fallback), without touching Product. This gives Product time to recover and saves Order's resources.
- **HALF-OPEN** (testing): after a waiting period, the breaker lets through **a few test calls**.
  - If they **all succeed** (100 %) → Product seems healthy → back to **CLOSED**.
  - If any **fail** → back to **OPEN** for another wait.

Cycle: **CLOSED → (failures cross the threshold) → OPEN → (after wait time) → HALF-OPEN → CLOSED or OPEN.**

### Using it in Spring Boot (Resilience4j)
Annotate the method with `@CircuitBreaker(name = "productService", fallbackMethod = "fallback")`. The fallback runs on every failed attempt and also whenever the circuit is open.

Key settings, in plain words:
| Setting | What it means |
|---|---|
| `sliding-window-type` and `sliding-window-size` | What to measure: **COUNT_BASED** looks at the last N calls (e.g. 10); **TIME_BASED** looks at calls in the last N seconds. |
| `minimum-number-of-calls` | Don't judge until at least this many calls were made (e.g. 5), so 1 failure out of 1 call doesn't trip it. |
| `failure-rate-threshold` | The failure percentage that trips the breaker (e.g. 50 %). |
| `wait-duration-in-open-state` | How long to stay OPEN before testing again (e.g. 10 s). |
| `permitted-number-of-calls-in-half-open-state` | How many test calls to allow in HALF-OPEN (e.g. 3). |
| `automatic-transition-from-open-to-half-open-enabled` | Move to HALF-OPEN by itself once the wait is over. |

With the example numbers: once at least 5 of the last 10 calls have been made and 50 % or more failed, the breaker opens for 10 s, then allows 3 test calls.

**Which errors count as failures?** By default every `RuntimeException` and `Error`. You can narrow it with `record-exceptions` (count only these) and `ignore-exceptions` (never count these, e.g. `IllegalArgumentException` caused by bad input, which isn't the downstream's fault).

### How it works inside (for the curious)
- Spring AOP intercepts your method and hands the call to the framework's **state machine** (`CircuitBreakerStateMachine`). After each result it checks whether the failure threshold was crossed; if so and the state is CLOSED, it switches to OPEN.
- **How does OPEN automatically become HALF-OPEN after 10 s?** When the circuit opens, the framework **schedules a task** ("switch to half-open") to run after the wait duration, using a `ScheduledThreadPoolExecutor`. That executor keeps tasks in a **delay queue**: worker threads look at the first task, and if its time hasn't come they sleep for the remaining time, then run it.

### Remember
Breaker = **stop hammering a failing service and probe it carefully later.** It works well together with a **fallback** (a sensible default answer) and, in the stack of protections, sits just before **Retry**.
<!-- simple-explanation -->


## 1. What & why

**Circuit Breaker prevents an application from making repeated calls to a downstream service that is likely to fail.**

Without it (e.g. Order → Product while Product is failing):
- It **unnecessarily adds load** to Product service, so Product may take **longer to recover**.
- Order service **wastes its own resources** (latency and thread blocking) making calls that are likely to fail.

Question: *how does Order service know when to stop calling Product and when to restart?* → **Circuit Breaker**.

Flow in the diagram: calls to Product fail → **failure threshold reached → stop future calls to Product** → calls are **blocked for some time and fail immediately** → afterwards **check if Product is healthy and allow further calls only if it is**.

---

## 2. States of Circuit Breaker

```
 Initial ──► CLOSED ──(failure rate crosses threshold)──► OPEN
               ▲                                           │  │
               │                          after wait/timeout│  │ test calls' success
               │                                  completed │  │ rate not 100%
  test calls'  │                                           ▼  │
  success 100% └──────────────────────────────────── HALF-OPEN ┘
```
| State | Behaviour |
|---|---|
| **CLOSED** | Initial state. **All calls are allowed** to the downstream service. |
| **OPEN** | **No calls allowed** to downstream; they **fail immediately**. |
| **HALF-OPEN** | After the timeout period, allows only a **limited number of test calls** to downstream and **tracks their success rate**. |

Transitions:
- CLOSED → OPEN: failure limit crossed the threshold.
- OPEN → HALF-OPEN: after the timeout/wait period completes.
- HALF-OPEN → CLOSED: test calls' success rate is **100 %**.
- HALF-OPEN → OPEN: test calls' success rate is **not 100 %**.

---

## 3. Implementation (Resilience4j)

**pom.xml**
```xml
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-spring-boot3</artifactId>
  <version>2.1.0</version>
</dependency>
```

**Controller / Feign**
```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    @Autowired OrderService orderService;

    @GetMapping("/{id}")
    public void callProductAPI(@PathVariable String id) {
        orderService.invokeProductAPI(id);
    }
}

@FeignClient(name = "product-service")
public interface ProductClient {
    @GetMapping(value = "/products/{id}")
    String getProductById(@PathVariable("id") String id);
}
```

**Service**
```java
@Component
public class OrderService {
    @Autowired ProductClient productClient;

    /* Fallback method is invoked for each failure attempt. */
    @CircuitBreaker(name = "productService", fallbackMethod = "fallback")
    public void invokeProductAPI(String id) {
        productClient.getProductById(id);
    }

    public void fallback(Throwable ex) {
        System.out.println("not able to invoke product service");
    }
}
```

**application.properties**
```properties
server.port=8081
spring.application.name=order-service
eureka.client.service-url.defaultZone=http://localhost:8761/eureka

# product service - circuit breaker configurations
resilience4j.circuitbreaker.instances.productService.sliding-window-type=COUNT_BASED
resilience4j.circuitbreaker.instances.productService.sliding-window-size=10
resilience4j.circuitbreaker.instances.productService.minimum-number-of-calls=5
resilience4j.circuitbreaker.instances.productService.failure-rate-threshold=50
resilience4j.circuitbreaker.instances.productService.wait-duration-in-open-state=10s
resilience4j.circuitbreaker.instances.productService.permitted-number-of-calls-in-half-open-state=3
resilience4j.circuitbreaker.instances.productService.automatic-transition-from-open-to-half-open-enabled=true
```

### What each property means
| Property | Meaning |
|---|---|
| `sliding-window-type=COUNT_BASED`, `sliding-window-size=10` | Tracks the **last N (here 10) calls**. |
| `sliding-window-type=TIME_BASED`, `sliding-window-size=10s` | Tracks the calls made in the **last N seconds** (here 10 s). |
| `minimum-number-of-calls` | Minimum calls required in a window **before it starts evaluating the failure rate**. |
| `failure-rate-threshold` | Circuit moves to **OPEN** once this **% limit** is reached. |
| `wait-duration-in-open-state` | How long the circuit stays **OPEN** before moving to **HALF_OPEN** (automatically, if `automatic-transition-from-open-to-half-open-enabled=true`). |
| `permitted-number-of-calls-in-half-open-state` | Number of **test calls allowed in HALF_OPEN** to find out if downstream is healthy. |

### Which exceptions count as failure?
By default the Circuit Breaker records **all `RuntimeException`s and `Error`s as failures**. You can choose what to record or ignore:
```properties
resilience4j.circuitbreaker.instances.productService.record-exceptions=java.io.IOException,org.springframework.web.client.HttpServerErrorException
resilience4j.circuitbreaker.instances.productService.ignore-exceptions=java.lang.IllegalArgumentException
```

**Output observed:** once the max trial-call limit is reached in HALF_OPEN and success is not 100 %, the status changed **HALF_OPEN → OPEN**.

---

## 4. How it works internally

- AOP intercepts the call and passes it to **`CircuitBreakerStateMachine.java`**, which has the complete logic for changing from one state to another whenever there is a failure.
- Sample framework method:
```java
// Transitions to open state when thresholds have been exceeded.
private void checkIfThresholdsExceeded(Result result) {
    if (Result.hasExceededThresholds(result) && isClosed.compareAndSet(true, false)) {
        /* After every error, it checks if state needs to be changed.
           Here: if threshold limit reached and state is CLOSED, transit to OPEN.
           A similar method is present for the other states with their own transition logic. */
        publishCircuitThresholdsExceededEvent(result, circuitBreakerMetrics);
        transitionToOpenState();
    }
}
```
- **Q: after moving to OPEN, how does it automatically go to HALF_OPEN after e.g. 10 s?**
  **A: it uses a `ScheduledThreadPoolExecutor`.** `OpenState` in `CircuitBreakerStateMachine`:
```java
OpenState(final int attempts, final long waitDurationInMillis,
          final Instant retryAfterWaitDuration, CircuitBreakerMetrics circuitBreakerMetrics) {
    this.attempts = attempts;
    this.retryAfterWaitDuration = retryAfterWaitDuration;
    this.circuitBreakerMetrics = circuitBreakerMetrics;
    if (circuitBreakerConfig.isAutomaticTransitionFromOpenToHalfOpenEnabled()) {
        ScheduledExecutorService scheduledExecutorService = schedulerFactory.getScheduler();
        /* It passes the request to ScheduledThreadPoolExecutor.
           1st parameter: the Task, i.e. "toHalfOpenState" to transit OPEN → HALF_OPEN
           2nd parameter: the delay (like 10 or 20 or 30)
           3rd parameter: Time Unit (second / minutes / millisecond) */
        transitionToHalfOpenFuture = scheduledExecutorService
            .schedule(this::toHalfOpenState, waitDurationInMillis, TimeUnit.MILLISECONDS);
    } else {
        transitionToHalfOpenFuture = null;
    }
    isOpen = new AtomicBoolean(true);
}
```
- (`ScheduledThreadPoolExecutor` usage is covered in the Java notes.)
- Internally `ScheduledThreadPoolExecutor` uses the concept of a **DelayedQueue**:
  - Each available worker thread looks at the **head** of the Delayed Queue.
  - If the task's delay hasn't expired, the thread **waits (blocks)** for the remaining delay; once the delay is over, the OS wakes the thread.
  - One thread picks the task from the head and executes it; other threads move to the next task in the queue – if it's not yet ready they wait (block) again.

---

## Quick revision
- 3 states: **CLOSED → OPEN → HALF_OPEN → (CLOSED | OPEN)**.
- Key knobs: sliding-window type/size, `minimum-number-of-calls`, `failure-rate-threshold`, `wait-duration-in-open-state`, `permitted-number-of-calls-in-half-open-state`.
- Default failure = any `RuntimeException`/`Error`; customise via `record-exceptions` / `ignore-exceptions`.
- Fallback is invoked on **every** failed attempt (and when circuit is open).
- OPEN → HALF_OPEN automatic switch is scheduled with `ScheduledThreadPoolExecutor` (DelayedQueue).

---

<!-- nav -->
⬅ [38 Retry Pattern](38_Retry_Pattern_Notes.md) · [Index](00_HLD_Index.md) · [40 & 41 API Gateway Part1 Part2](40_41_API_Gateway_Part1_Part2_Notes.md) ➡
<!-- nav -->
