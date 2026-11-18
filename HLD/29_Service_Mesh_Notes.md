# Service Mesh (HLD #29)

<!-- nav -->
⬅ [28 API Gateway Concepts](28_API_Gateway_Concepts_Notes.md) · [Index](00_HLD_Index.md) · [30 DNS](30_DNS_Notes.md) ➡
<!-- nav -->

Source: Typed Zoho note – https://notebook.zohopublic.in/public/notes/9wn9od9ab6075a35b4258aadedbf53aca805e

## 1. The question: how do 2 microservices communicate with each other?

In the previous topic (API Gateway) the picture is: **Client → DNS** (like AWS Route 53, Azure Traffic Manager) → one **API Gateway per region**. Each region has availability zones (AZ1, AZ2); in each AZ every microservice (Microservice1, Microservice2) has its own **load balancer** in front of several instances. The API Gateway asks **Service Discovery** where to send the request.

In simple form the page redraws it as:

```mermaid
flowchart LR
    GW[API Gateway] --> SD[Service Discovery]
    GW --> LB1[Application Load Balancer] --> MS1["Microservice1 (many instances)"]
    GW --> LB2[Application Load Balancer] --> MS2["Microservice2 (many instances)"]
```

**In simple words:** the gateway handles traffic coming **from the client**. The follow-up question is the traffic **between services**: Microservice A wants to call Microservice B. What does A need for that? The page lists seven capabilities.

## 2. Capability 1 – Service Discovery

- Microservice A wants to talk to Microservice B, so it needs B's **location (IP address and port no)**.
- Service Discovery is needed so A can get the location of **B's different instances**.

**Scenario 1:** Service Discovery returns **all of Microservice B's instance locations**.

```mermaid
sequenceDiagram
    participant A as Microservice1 (A)
    participant SD as Service Discovery
    A->>SD: Give me Microservice B location
    SD-->>A: Instance1: url:port, Instance2: url:port, Instance3: url:port
```

**Scenario 2:** Service Discovery returns the **address of Microservice B's application load balancer**.

```mermaid
sequenceDiagram
    participant A as Microservice1 (A)
    participant SD as Service Discovery
    A->>SD: Give me Microservice B location
    SD-->>A: Location of Microservice B Application Load Balancer
```

**In simple words:** either the registry hands A the addresses of all the instances of B, or it hands over just the address of B's load balancer, and the load balancer then picks an instance.

## 3. Capability 2 – Load balancing

**Scenario 1:** A got the instance locations of B and now wants **load balancing capability to distribute the traffic equally** to B. Config looks like `microserviceB.url={instance-url}:{port-no}`. A needs a **client-side load balance capability** inside itself.

```mermaid
flowchart LR
    MS1[Microservice1] -- Give me B location --> SD[Service Discovery]
    SD -- Instance1/2/3 url:port --> CLB
    subgraph MS1box[Microservice1]
        CLB[Client side Load Balance Capability]
    end
    CLB --> I1[Microservice2: Instance1]
    CLB --> I2[Microservice2: Instance2]
    CLB --> I3[Microservice2: Instance3]
```

**Scenario 2:** A got **B's application load balancer address** and now needs to invoke that. Config looks like `microserviceB.url={application-load-balancer-url}:{port-no}`. **An extra network call is required, which might cause a latency issue** (the "extra hop" in the diagram).

```mermaid
flowchart LR
    MS1[Microservice1] -- extra hop (network call) --> LB[Application Load Balancer] --> I[Microservice2: Instance1/2/3]
```

**In simple words:** with scenario 1 the caller picks the instance itself (no extra hop but the caller must contain the balancing logic). With scenario 2 the caller calls a load balancer which then forwards the call: simpler for the caller but one more network hop, so more delay.

## 4. Capability 3 – Authorization

Microservice2's instance must **validate: does Microservice A have permission to invoke this API?**

```mermaid
flowchart LR
    A[Microservice1 + client side LB] --> Z
    subgraph B[Microservice2: Instance1]
        Z[Authorization]
    end
```

**In simple words:** the called service checks who is calling it before it does the work.

## 5. Capability 4 – Circuit breaker

Microservice1 also needs a **circuit breaker (Hystrix)** next to the client-side load balancer, in front of the call to Microservice2.

**In simple words:** if the target keeps failing, the circuit breaker stops further calls instead of letting them pile up. (See [39 Circuit Breaker](39_Circuit_Breaker_Notes.md).)

## 6. Capability 5 – Retry

Microservice1 also gets a **Retry** capability (the green circular arrow in the diagram), sitting together with the circuit breaker.

**In simple words:** if a call fails, it can be tried again. (See [38 Retry Pattern](38_Retry_Pattern_Notes.md).)

## 7. Capability 6 – Deployment strategy

- There are different deployment strategies like **Canary, Red-Black, Blue-Green** etc.
- Example: in **Canary**, I want **90% of traffic on old and 10% on new** initially, and gradually increase the amount on the new manifest.

```mermaid
flowchart LR
    M1[Microservice 1] -- 90% --> OLD["Microservice 2 (Old Manifest)"]
    M1 -- 10% --> NEW["Microservice 2 (New Manifest)"]
```

- Generally the **load balancer helps to control this**, which means **"the client-side load balancer also has to handle this different deployment strategy"**. So the box becomes *Client-side load balance capability + support for deployment strategy*.

## 8. Capability 7 – Telemetry

- It means we need to **collect data that helps in analysis and monitoring of a component.**
- Telemetry capability on Microservice2: **latency, error rate, traffic, etc.**
- This data **helps in monitoring as well as debugging.**

**Everything on one picture:** Microservice1 now carries client-side load balancing + deployment-strategy support, circuit breaker (Hystrix) and retry; Microservice2 carries telemetry and authorization; plus the Service Discovery lookup.

```mermaid
flowchart LR
    SD[Service Discovery]
    subgraph M1[Microservice1]
        A1[Client side LB + deployment strategy support]
        A2[Circuit Breaker - Hystrix]
        A3[Retry]
    end
    subgraph M2[Microservice2: Instance1]
        B1[Telemetry: latency, error rate, traffic]
        B2[Authorization]
    end
    M1 -- Give me Microservice B location --> SD
    SD -- Instance1/2/3 url:port --> M1
    M1 --> M2
```

**In simple words:** every microservice would have to carry all this extra code (discovery, balancing, authorization, circuit breaking, retry, deployment strategy, telemetry), which is a lot of repeated work inside each service.

## 9. Any better solution? – SERVICE MESH

**Service Mesh (like Istio).**

```mermaid
flowchart LR
    GW[API Gateway] --> SD[Service Discovery]
    GW --> LB1[Application Load Balancer] --> P1
    GW --> LB2[Application Load Balancer] --> P2
    subgraph DP[Data Plane]
        subgraph P1[Pod 1..N]
            I1[Microservice1 Instance]
            S1[Sidecar Proxy]
        end
        subgraph P2[Pod 1..N]
            I2[Microservice2 Instance]
            S2[Sidecar Proxy]
        end
    end
    subgraph CP[Control Plane]
        CM[Configuration Manager] --> TC[Traffic Controller]
        SM[Security Manager]
        T[Telemetry]
    end
    S1 <--> CP
    S2 <--> CP
    T -.-> OD[Observability Dashboards]
```

- **Data plane:** the pods, each with the microservice instance **and a sidecar proxy** next to it.
- **Control plane:** **Configuration Manager, Traffic Controller, Security Manager, Telemetry**; telemetry feeds **Observability Dashboards**.
- The sidecar proxies talk to the control plane (two-way arrow).

**In Istio:**
- Sidecar proxy is known as **Envoy**
- Configuration Manager is known as **Galley**
- Traffic Controller is known as **Pilot**
- Security Manager is known as **Citadel**

**In simple words:** instead of coding all seven capabilities inside every microservice, a small **sidecar proxy** sits beside each instance and does them. The **control plane** tells all the sidecars what rules to follow (traffic, security, configuration) and collects telemetry for dashboards.

---

<!-- nav -->
⬅ [28 API Gateway Concepts](28_API_Gateway_Concepts_Notes.md) · [Index](00_HLD_Index.md) · [30 DNS](30_DNS_Notes.md) ➡
<!-- nav -->
