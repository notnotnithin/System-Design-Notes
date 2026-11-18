# Service Discovery (HLD #34 / Microservices #4)

<!-- nav -->
⬅ [33 Dual Write Problem](33_Dual_Write_Problem_Notes.md) · [Index](00_HLD_Index.md) · [35 Client Side Load Balancer](35_Client_Side_Load_Balancer_Notes.md) ➡
<!-- nav -->

Source: https://drive.google.com/file/d/11DAZgXFoArZ17xVLhkIxTYQtUYAJNnr-/view?usp=drive_link

---

## 1. Why Service Discovery?

<!-- diagram:34-0 -->
**Diagram – Hardcoded URL versus service discovery**

```mermaid
graph LR
  subgraph HC["Hardcoded URL"]
    O1["Order service"] -->|"http://localhost:8082"| P1["Product instance 1 only"]
  end
  subgraph SD["With Eureka"]
    O2["Order service"] -->|"where is product-service?"| E["Eureka server: the phone book"]
    E -->|"instance list"| O2
    O2 --> PA["Product instance 1"]
    O2 --> PB["Product instance 2"]
    PA -->|"register + heartbeat"| E
    PB -->|"register + heartbeat"| E
  end
```
<!-- /diagram:34-0 -->


In microservices, service/component **instances are created and deleted dynamically**. You **cannot hardcode the URL** of an instance – it is not scalable or feasible.

Example: *Order service* hardcodes the URL of *Product service*. This is fine only while Product service has a single instance. With multiple instances (or an instance being removed):

| Problem | Explanation |
|---|---|
| **Single point of failure** | If the hardcoded instance dies, Order service cannot reach any other instance. |
| **No load balancing** | Only one Product instance is overburdened; the others sit idle. |
| **Tight coupling** | Order service can't change/move Product service without being updated. |
| **Difficult testing** | Prod / QA / dev use different URLs → frequent config changes, error-prone. |

**Solution: Service Discovery (e.g. Netflix Eureka).**

- **Eureka Server** – acts like a **phone book**. Holds info of all registered clients: service name, instance id, IP, port, health status, etc.
- **Eureka Client** – **registers itself** with the server and **discovers** instances of other services via the server.

<!-- isw:34-0 -->
**In simple words**

In microservices, service instances are **created and deleted dynamically,** so you **cannot hardcode the URL** of an instance; it is not scalable or feasible. The example: the *Order service* hardcodes the URL of the *Product service.* That works only while Product has a single instance. With several instances (or when an instance is removed), the table in the notes lists the problems:
- **Single point of failure:** if the hardcoded instance dies, Order cannot reach any other instance.
- **No load balancing:** only one Product instance is overburdened and the others sit idle.
- **Tight coupling:** Order cannot change or move Product without being updated.
- **Difficult testing:** prod, QA and dev use different URLs, so config changes often and mistakes are likely.

The **solution is Service Discovery (for example Netflix Eureka).** The **Eureka Server acts like a phone book:** it holds the information of all registered clients (service name, instance id, IP, port, health status and so on). A **Eureka Client registers itself** with the server and **discovers** the instances of other services through it.
<!-- /isw:34-0 -->


---

## 2. Setting up Eureka Server

Generate project via Spring Initializr (start.spring.io).

**1. pom.xml**
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```
Version is resolved by dependency management:
```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>2023.0.1</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

**2. Enable Eureka Server**
```java
@SpringBootApplication
/* Tells Spring Boot to create the beans needed for Eureka Server:
   EurekaController, Dashboard etc. */
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

**3. application.properties**
```properties
spring.application.name=eureka-server
server.port=8761
# Since it's a server, don't register itself and don't fetch instance details
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```
Start the server → dashboard at **http://localhost:8761**.

<!-- isw:34-1 -->
**In simple words**

Generate the project with Spring Initializr (start.spring.io). Steps: (1) add the dependency **`spring-cloud-starter-netflix-eureka-server`** in `pom.xml` (the version is resolved by `spring-cloud-dependencies` 2023.0.1 in the dependency management); (2) put **`@EnableEurekaServer`** on the main class, which **tells Spring Boot to create the beans needed for a Eureka server (EurekaController, the dashboard, …);** (3) in `application.properties` set the name `eureka-server`, port **8761**, and **`register-with-eureka=false`** and **`fetch-registry=false`,** because it is a server and should neither register itself nor fetch instance details. Start the server and the **dashboard is at `http://localhost:8761`.**
<!-- /isw:34-1 -->


---

## 3. Setting up Eureka Client (Product Service)

**1. pom.xml**
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
(Same `spring-cloud-dependencies` 2023.0.1 dependencyManagement block as above.)

**2. application.properties**
```properties
server.port=8082
spring.application.name=product-service

# path of the Eureka Server
eureka.client.service-url.defaultZone=http://localhost:8761/eureka

# defaults are true, so these can be skipped
eureka.client.register-with-eureka=true
eureka.client.fetch-registry=true
```
Start Product Service and see it on the dashboard. Order Service is registered the same way.

<!-- isw:34-2 -->
**In simple words**

Steps for the Product service: (1) add **`spring-cloud-starter-netflix-eureka-client`** (with the same dependency-management block); (2) in `application.properties` set the port (8082), the application name (`product-service`), the **Eureka server path** (`eureka.client.service-url.defaultZone=http://localhost:8761/eureka`), and optionally `register-with-eureka=true` and `fetch-registry=true` (these defaults are already true, so they can be skipped). Start the Product service and it appears on the dashboard. The Order service is registered the same way.
<!-- /isw:34-2 -->


---

## 4. Invoking Product Service from Order Service

### Using RestTemplate

**Without Service Discovery** (hardcoded URL)
```java
public void callProductAPI(String id) {
    RestTemplate restTemplate = new RestTemplate();
    String response = restTemplate.getForObject(
        "http://localhost:8082/products/" + id, String.class);
    System.out.println("Response from Product api call is: " + response);
}
```

**With Service Discovery**
```java
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;

@Autowired DiscoveryClient discoveryClient;

public void callProductAPI(String id) {
    RestTemplate restTemplate = new RestTemplate();
    List<ServiceInstance> instances = discoveryClient.getInstances("product-service");
    URI uri = instances.get(0).getUri();
    String response = restTemplate.getForObject(uri + "/products/" + id, String.class);
    System.out.println("Response from Product api call is: " + response);
}
```
➡ With RestTemplate, **load balancing logic (choosing an instance) must be handled by you.**

<!-- isw:34-4 -->
**In simple words**

**Without service discovery** the URL is hardcoded: `restTemplate.getForObject("http://localhost:8082/products/" + id, String.class)`. **With service discovery** you inject the **`DiscoveryClient`,** call `getInstances("product-service")`, take `instances.get(0).getUri()` and call that URI. The notes point out: **with RestTemplate the load-balancing logic (choosing an instance) must be handled by you.**
<!-- /isw:34-4 -->


### Using FeignClient
➡ With FeignClient, **load balancing is handled automatically by the framework.** Add the load balancer dependency apart from eureka-client:
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```
Earlier (no discovery):
```java
@FeignClient(name = "product-service", url = "${feign.client.product-service.url}")
public interface ProductClient {
    @GetMapping("/products/{id}")
    String getProductById(@PathVariable("id") String id);
}
```
With Service Discovery:
```java
@FeignClient(name = "product-service")
public interface ProductClient {
    @GetMapping("/products/{id}")
    String getProductById(@PathVariable("id") String id);
}
```

<!-- isw:34-3 -->
**In simple words**

Order now calls Product in two possible ways, shown below.
<!-- /isw:34-3 -->

<!-- isw:34-5 -->
**In simple words**

**With FeignClient, load balancing is handled automatically by the framework.** Add the load-balancer dependency (`spring-cloud-starter-loadbalancer`) besides eureka-client. Earlier (without discovery) the Feign client has `@FeignClient(name = "product-service", url = "${feign.client.product-service.url}")`; **with service discovery** only the name is needed: `@FeignClient(name = "product-service")`.
<!-- /isw:34-5 -->



---

## 5. Curious-engineer questions

**Service Registration doubts**
1. How does Eureka Server know whether a client is UP or DOWN?
2. Where and how is the data stored?
3. What if Eureka Server itself goes down? Is it a single point of failure?

**Discovery doubts**
4. Latency – each call now needs 2 hops (Eureka Server, then the actual call)?
5. What if the local cache is stale? Can this lead to calling a dead instance?

### Q1. How does Eureka know a client is UP or DOWN?

<!-- diagram:34-1 -->
**Diagram – Registration, heartbeat and eviction**

```mermaid
sequenceDiagram
  participant C as Eureka client (Product service)
  participant S as Eureka server
  C->>S: register (name, IP, port, status)
  loop every 30 s (default)
    C->>S: heartbeat (lease renewal)
  end
  Note over S: no heartbeat for 90 s (default): instance removed
  C->>S: de-register on graceful shutdown: marked DOWN
```
<!-- /diagram:34-1 -->


**a) Client de-registration request** – on graceful shutdown, the Eureka client sends a de-registration request; the server marks the client **DOWN**. (Visible in both client and server logs.)

**b) Client heartbeat** – the client periodically sends a heartbeat to the server.
If the client dies without de-registering (e.g. network issue), the server waits for a heartbeat for a particular interval (decided at registration); if none arrives, **the server removes the instance itself**.

Client `application.properties`:
```properties
server.port=8082
spring.application.name=product-service
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
eureka.client.register-with-eureka=true
eureka.client.fetch-registry=true

# every 60 seconds client sends heartbeat to server
eureka.instance.lease-renewal-interval-in-seconds=60
# server waits this long for a heartbeat, else removes me (kept < renewal interval for testing)
eureka.instance.lease-expiration-duration-in-seconds=5
```
- `lease-renewal-interval-in-seconds` – after this many seconds the client sends a new heartbeat. **Default 30 s.**
- `lease-expiration-duration-in-seconds` – how long the server waits for a heartbeat before removing the instance. **Default 90 s.**

Eureka Server `application.properties`:
```properties
spring.application.name=eureka-server
server.port=8761
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false

# by default the server does NOT remove clients even if heartbeats are missed (self-preservation).
# turn it OFF so the server can remove clients that miss heartbeats.
eureka.server.enable-self-preservation=false

# how frequently the eviction task runs
eureka.server.eviction-interval-timer-in-ms=6000
```
With the above test config: client sends a heartbeat after 60 s, but server waits only 5 s → server **removes the instance from the list even though the client is running and up.**

<!-- isw:34-7 -->
**In simple words**

Two ways: **(a) the client's de-registration request** (on a graceful shutdown the Eureka client sends a de-registration request and the server marks it **DOWN;** visible in both logs); **(b) the client heartbeat** (the client regularly sends a heartbeat; if it dies without de-registering, for example from a network issue, the server waits for a heartbeat for a certain interval decided at registration, and if none arrives **the server removes the instance itself**).

The client properties: **`lease-renewal-interval-in-seconds`** (after this many seconds the client sends a new heartbeat; **default 30 s**) and **`lease-expiration-duration-in-seconds`** (how long the server waits for a heartbeat before removing the instance; **default 90 s**). On the server side, **by default the server does NOT remove clients even if heartbeats are missed (self-preservation)**; you turn it off with `eureka.server.enable-self-preservation=false` so the server can remove clients, and `eviction-interval-timer-in-ms` sets how often the eviction task runs. In the notes' test configuration the client sends a heartbeat after 60 s but the server waits only 5 s, so the server **removes the instance even though the client is running and up.**
<!-- /isw:34-7 -->


### Q2. How does Eureka store data?
**Only in memory**: `Map<String, Lease<InstanceInfo>>`
- Key: `appName/instanceId`, e.g. `PRODUCT-SERVICE/192.157.2.27:product-service:8082`
- Value: `InstanceInfo` object containing: instance id, app name, IP address, host name, port, status (UP/DOWN), last renewed timestamp, lease duration, etc.

<!-- isw:34-8 -->
**In simple words**

**Only in memory,** in a `Map<String, Lease<InstanceInfo>>`. The **key** is `appName/instanceId` (for example `PRODUCT-SERVICE/192.157.2.27:product-service:8082`). The **value** is an `InstanceInfo` object with the instance id, app name, IP address, host name, port, status (UP / DOWN), last renewed timestamp, lease duration and so on.
<!-- /isw:34-8 -->


### Q3. What if Eureka Server goes down?

<!-- diagram:34-2 -->
**Diagram – Three-node Eureka cluster**

```mermaid
graph LR
  E1["eureka-1 :8761"] <--> E2["eureka-2 :8762"]
  E2 <--> E3["eureka-3 :8763"]
  E3 <--> E1
  C["Clients list all three URLs"] --> E1
  C --> E2
  C --> E3
```
<!-- /diagram:34-2 -->


Yes – a **single Eureka server is a single point of failure.** Usually a **3-node cluster** is used.

Example: eureka-1 (8761), eureka-2 (8762), eureka-3 (8763) on different machines/containers. **Each server is also a client** – they register themselves, fetch the registry and replicate changes.

Server 1:
```properties
spring.application.name=eureka-server
server.port=8761
eureka.instance.hostname=localhost
eureka.client.register-with-eureka=true
eureka.client.fetch-registry=true
eureka.client.service-url.defaultZone=http://localhost:8762/eureka/,http://localhost:8763/eureka/
```
Server 2: `server.port=8762`, `defaultZone=http://localhost:8761/eureka/,http://localhost:8763/eureka/`
Server 3: `server.port=8763`, `defaultZone=http://localhost:8761/eureka/,http://localhost:8762/eureka/`

Client:
```properties
eureka.client.service-url.defaultZone=http://eureka-1:8761/eureka,http://eureka-2:8762/eureka,http://eureka-3:8763/eureka
```
Now even if one server is down, client availability is not impacted.

<!-- isw:34-9 -->
**In simple words**

Yes, a **single Eureka server is a single point of failure.** Usually a **3-node cluster** is used: for example eureka-1 (8761), eureka-2 (8762) and eureka-3 (8763) on different machines or containers. **Each server is also a client:** they register themselves, fetch the registry and replicate the changes (server 1 lists servers 2 and 3 in its `defaultZone`, and so on). The clients list all three URLs in their `defaultZone`. Now **even if one server is down, the client's availability is not affected.**
<!-- /isw:34-9 -->


### Q4. Latency – 2 hops per call?

<!-- diagram:34-3 -->
**Diagram – The client caches the registry locally**

```mermaid
sequenceDiagram
  participant O as Order service
  participant E as Eureka server
  participant P as Product service
  O->>E: at startup: fetch registry (cached locally)
  loop every 30 s
    O->>E: refresh the local copy
  end
  O->>P: normal calls use the local copy (no extra hop)
```
<!-- /diagram:34-3 -->


Eureka Server is **not called for every request.** At startup the client (e.g. order-service) **fetches the registry** (`eureka.client.fetch-registry=true`) and **caches it locally**; future requests use the local copy to find the instance. Cache refresh is configurable:
```properties
# refresh local cache copy every 30 seconds
eureka.client.registry-fetch-interval-seconds=30
```

<!-- isw:34-10 -->
**In simple words**

**The Eureka server is not called for every request.** At startup the client (for example order-service) **fetches the registry** (`fetch-registry=true`) and **caches it locally;** later requests use the local copy to find the instance. The refresh is configurable: `registry-fetch-interval-seconds=30` means the local copy is refreshed every 30 seconds.
<!-- /isw:34-10 -->


### Q5. Stale local cache → calling a dead instance?
Valid scenario – **it's a trade-off** (availability/performance vs. freshness).

<!-- isw:34-6 -->
**In simple words**

The notes list five doubts. **Registration:** (1) How does Eureka Server know whether a client is UP or DOWN? (2) Where and how is the data stored? (3) What if Eureka Server itself goes down; is it a single point of failure? **Discovery:** (4) Latency: does each call now need 2 hops (Eureka Server, then the actual call)? (5) What if the local cache is stale; can it lead to calling a dead instance? The answers follow.
<!-- /isw:34-6 -->

<!-- isw:34-11 -->
**In simple words**

This is a valid scenario: **it is a trade-off** (availability and performance versus freshness).
<!-- /isw:34-11 -->



---

## Quick revision
- Hardcoded URLs → SPOF, no LB, tight coupling, test pain → use **Service Discovery**.
- Eureka Server = registry (in-memory map); Eureka Client = registers + fetches registry + caches locally.
- Liveness via **de-registration** + **heartbeats** (defaults 30 s renew / 90 s expiry); self-preservation can be disabled.
- HA = **3-node Eureka cluster** where each node is also a client.
- RestTemplate → manual instance selection; FeignClient (+ spring-cloud-starter-loadbalancer) → automatic LB.
- Local registry cache trades freshness for fewer hops.

> Note: diagrams in the original PDF (architecture pictures, dashboard/log screenshots) are not reproduced here; all text and code is.

---

<!-- nav -->
⬅ [33 Dual Write Problem](33_Dual_Write_Problem_Notes.md) · [Index](00_HLD_Index.md) · [35 Client Side Load Balancer](35_Client_Side_Load_Balancer_Notes.md) ➡
<!-- nav -->
