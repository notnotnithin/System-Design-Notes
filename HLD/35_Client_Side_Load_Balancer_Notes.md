# Client Side Load Balancer (HLD #35 / Microservices #5)

<!-- nav -->
⬅ [34 Service Discovery](34_Service_Discovery_Notes.md) · [Index](00_HLD_Index.md) · [37 Bulkhead](37_Bulkhead_Notes.md) ➡
<!-- nav -->

Source: https://drive.google.com/file/d/1uERH11qv6xWBvwHc0BoOJlnn89IzWt95/view?usp=drive_link

**Pre-requisite:** Service Discovery (see [34_Service_Discovery_Notes.md](34_Service_Discovery_Notes.md)).

---

## 1. Load Balancer basics

<!-- diagram:35-0 -->
**Diagram – Server-side versus client-side load balancer**

![Diagram](diagrams/35_Client_Side_Load_Balancer_Notes-01.png)
<!-- /diagram:35-0 -->


- Distributes traffic to multiple instances of a server.
- Prevents a single server from being overloaded with huge traffic.

### Types
| Type | Description | Examples |
|---|---|---|
| **Server-side** | Centralized load balancer; like a separate microservice | Nginx, ELB (Elastic Load Balancer) |
| **Client-side** | Load-balancing capability lives inside the client (caller), or it uses a library to decide | Spring Cloud LoadBalancer, Netflix Ribbon, Istio with Sidecar |

<!-- isw:35-0 -->
**In simple words**

A load balancer **distributes traffic to several instances of a server** and **prevents a single server from being overloaded with huge traffic.**
<!-- /isw:35-0 -->

<!-- isw:35-1 -->
**In simple words**

**Server-side:** a **centralized load balancer, like a separate microservice** (examples: Nginx, ELB, the Elastic Load Balancer). **Client-side:** the **load-balancing capability lives inside the client (the caller),** or it uses a library to decide (examples: Spring Cloud LoadBalancer, Netflix Ribbon, Istio with sidecar).
<!-- /isw:35-1 -->



---

## 2. Before: RestTemplate with manual selection

After Service Discovery is set up, with RestTemplate we fetch the list of instances using the `DiscoveryClient` and **manually choose** an instance.

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Autowired DiscoveryClient discoveryClient;
    @Autowired RestTemplate restTemplate;

    @GetMapping("/{id}")
    public void callProductAPI(@PathVariable String id) {
        List<ServiceInstance> instances = discoveryClient.getInstances("product-service");
        URI uri = instances.get(0).getUri();
        String response = restTemplate.getForObject(uri + "/products/" + id, String.class);
        System.out.println("Response from Product api call is: " + response);
    }
}

@Configuration
public class Config {
    @Bean
    public RestTemplate restTemplateObj() { return new RestTemplate(); }
}
```

<!-- isw:35-2 -->
**In simple words**

After service discovery is set up, with RestTemplate we get the list of instances through the **`DiscoveryClient`** and **choose an instance manually** (the code takes `instances.get(0)` and builds the URL from it).
<!-- /isw:35-2 -->


---

## 3. With Spring Cloud LoadBalancer

<!-- diagram:35-1 -->
**Diagram – What @LoadBalanced does on a call**

![Diagram](diagrams/35_Client_Side_Load_Balancer_Notes-02.png)
<!-- /diagram:35-1 -->


**1. Dependency**
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

**2. Controller – no manual instance selection**
```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @Autowired RestTemplate restTemplate;

    @Value("${product.service.baseurl}")
    String productBaseURL;

    @GetMapping("/{id}")
    public void callProductAPI(@PathVariable String id) {
        String response = restTemplate.getForObject(productBaseURL + "/products/" + id, String.class);
        System.out.println("Response from Product api call is: " + response);
    }
}

@Configuration
public class Config {
    @Bean
    /* With this annotation, the framework starts intercepting the request,
       calls Service Discovery, gets instances & applies the load balancing algorithm. */
    @LoadBalanced
    public RestTemplate restTemplateObj() { return new RestTemplate(); }
}
```

**3. application.properties**
```properties
server.port=8081
spring.application.name=order-service
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
product.service.baseurl=http://product-service
```

<!-- isw:35-3 -->
**In simple words**

(1) Add the dependency **`spring-cloud-starter-loadbalancer`.** (2) In the controller there is **no manual instance selection:** it just calls `productBaseURL + "/products/" + id`, where the base URL is `http://product-service` (the service name). (3) On the `RestTemplate` bean put **`@LoadBalanced`:** *with this annotation the framework starts intercepting the request, calls service discovery, gets the instances and applies the load-balancing algorithm.* The `application.properties` only needs the server port, the application name, the Eureka path and `product.service.baseurl=http://product-service`.
<!-- /isw:35-3 -->


---

## 4. Algorithms & how they are attached

- Each load balancing algorithm is **attached to a ServiceId** – the client name, e.g. `"product-service"`.
- **Default:** `RoundRobinLoadBalancer`.
- If you don't want the default, you must define **which algorithm for which client (serviceId)**.
- Spring Cloud LoadBalancer supports **only 2 algorithms: RoundRobin and Random.**
- For others (Weighted, Least Connection, …) either write a **custom implementation** or switch to a **service mesh like Istio with sidecar**.

<!-- isw:35-4 -->
**In simple words**

Each load-balancing algorithm is **attached to a ServiceId,** which is the client name, for example `"product-service"`. The **default is `RoundRobinLoadBalancer`.** If you do not want the default, you must define **which algorithm for which client (serviceId).** Spring Cloud LoadBalancer supports **only 2 algorithms: RoundRobin and Random.** For others (Weighted, Least Connection, …) either write a **custom implementation** or switch to a **service mesh like Istio with a sidecar.**
<!-- /isw:35-4 -->


---

## 5. Changing the algorithm for one service (Random for product-service)

Controller / Config / properties stay the same as in section 3. Add `@LoadBalancerClient` on the application class:

```java
@SpringBootApplication
@EnableFeignClients
@LoadBalancerClient(
    name = "product-service",
    configuration = LoadBalancerProductClientConfig.class
)
public class OrderserviceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderserviceApplication.class, args);
    }
}
```
> A normal config class is invoked generally only at application startup. **This `@LoadBalancerClient` config is invoked at runtime too**, when `http://product-service/xyz` ("product-service") is used.

```java
@Configuration
public class LoadBalancerProductClientConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> productClientLoadBalancer(
            LoadBalancerClientFactory factory) {
        return new RandomLoadBalancer(
            factory.getLazyProvider("product-service", ServiceInstanceListSupplier.class),
            "product-service");
    }
}
```
- `ServiceInstanceListSupplier` is the one that **invokes Service Discovery and asks for the instances** of `"product-service"`.
- `new RandomLoadBalancer(ServiceInstanceListSupplier, serviceId)` – we pass the serviceId because the algorithm is mapped to **that service ID only**. So only for `product-service` is `RandomLoadBalancer` used.

<!-- isw:35-5 -->
**In simple words**

To use Random for product-service: add **`@LoadBalancerClient(name = "product-service", configuration = LoadBalancerProductClientConfig.class)`** on the application class. The notes remark: *a normal config class is generally invoked only at application startup, but this `@LoadBalancerClient` config is invoked at runtime too,* when `http://product-service/xyz` is used. The config class returns `new RandomLoadBalancer(factory.getLazyProvider("product-service", ServiceInstanceListSupplier.class), "product-service")`. The **`ServiceInstanceListSupplier` is the one that invokes service discovery and asks for the instances** of `"product-service"`, and the serviceId is passed because the algorithm is mapped to **that service ID only.**
<!-- /isw:35-5 -->


---

## 6. Algorithm A for product-service, Algorithm B for all others

<!-- diagram:35-2 -->
**Diagram – Per-service algorithm and a default**

![Diagram](diagrams/35_Client_Side_Load_Balancer_Notes-03.png)
<!-- /diagram:35-2 -->


```java
@SpringBootApplication
@EnableFeignClients
/* First the child client's config is loaded, then the default one. */
@LoadBalancerClients(
    defaultConfiguration = LoadBalancerGlobalConfig.class,
    value = {
        @LoadBalancerClient(
            name = "product-service",
            configuration = LoadBalancerProductClientConfig.class
        )
    }
)
public class OrderserviceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderserviceApplication.class, args);
    }
}
```
- The `@LoadBalancerClients` config is loaded **at runtime for each service ID**, so `environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME)` is **resolved at runtime**.
- `@ConditionalOnMissingBean` is required on the Global config class because, for `product-service`, **two `ReactorLoadBalancer` beans should not be created** (one from `LoadBalancerProductClientConfig`, another from `LoadBalancerGlobalConfig`).
- Result: **product-service → Random**, **all other services → RoundRobin**.

```java
@Configuration
@ConditionalOnMissingBean(ReactorLoadBalancer.class)
public class LoadBalancerGlobalConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            LoadBalancerClientFactory factory, Environment environment) {
        String serviceId = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(
            factory.getLazyProvider(serviceId, ServiceInstanceListSupplier.class),
            serviceId);
    }
}

@Configuration
public class LoadBalancerProductClientConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> productClientLoadBalancer(
            LoadBalancerClientFactory factory) {
        return new RandomLoadBalancer(
            factory.getLazyProvider("product-service", ServiceInstanceListSupplier.class),
            "product-service");
    }
}
```

<!-- isw:35-6 -->
**In simple words**

To use one algorithm for product-service and another for all the others, use **`@LoadBalancerClients(defaultConfiguration = LoadBalancerGlobalConfig.class, value = { @LoadBalancerClient(name = "product-service", configuration = LoadBalancerProductClientConfig.class) })`.** The child client's config is loaded first, then the default one; both are loaded **at runtime for each service ID,** so `environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME)` is **resolved at runtime.** **`@ConditionalOnMissingBean` is needed on the global config** so that for `product-service` **two `ReactorLoadBalancer` beans are not created.** The **result:** product-service → Random, all other services → RoundRobin.
<!-- /isw:35-6 -->


---

## 7. Custom load balancer

```java
public class MyCustomLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final ObjectProvider<ServiceInstanceListSupplier> serviceInstanceSuppliers;
    private final String serviceId;

    public MyCustomLoadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> serviceInstanceSuppliers,
            String serviceId) {
        this.serviceInstanceSuppliers = serviceInstanceSuppliers;
        this.serviceId = serviceId;
    }

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        return serviceInstanceSuppliers
            .getIfAvailable()
            .get()
            .next()
            .map(instances -> {
                if (instances == null || instances.isEmpty()) {
                    return new EmptyResponse();
                }
                // Custom load balancing algorithm (demo: pick first instance)
                return new DefaultResponse(instances.get(0));
            });
    }
}
```
Plug it in:
```java
@Configuration
public class LoadBalancerProductClientConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> productClientLoadBalancer(
            LoadBalancerClientFactory factory) {
        return new MyCustomLoadBalancer(
            factory.getLazyProvider("product-service", ServiceInstanceListSupplier.class),
            "product-service");
    }
}
```

<!-- isw:35-7 -->
**In simple words**

To write your own, implement **`ReactorServiceInstanceLoadBalancer`** with its **`choose(Request)`** method: get the instances from the `ServiceInstanceListSupplier`, and if the list is empty return an `EmptyResponse`, otherwise apply your algorithm (the demo just picks the first instance) and return `new DefaultResponse(instance)`. Then plug it in by returning `new MyCustomLoadBalancer(...)` from the config bean for `"product-service"`.
<!-- /isw:35-7 -->


---

## 8. With FeignClient

Load balancing is already automatic (seen in Service Discovery). **Everything is the same as with RestTemplate – no change.**

```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    @Autowired ProductClient productClient;

    @GetMapping("/{id}")
    public void callProductAPI(@PathVariable String id) {
        String response = productClient.getProductById(id);
        System.out.println("Response from Product api call is: " + response);
    }
}
```
Algorithm override uses the same pieces:
```java
@SpringBootApplication
@EnableFeignClients
@LoadBalancerClient(name = "product-service", configuration = LoadBalancerProductClientConfig.class)
public class OrderserviceApplication { ... }

@Configuration
public class LoadBalancerProductClientConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> productClientLoadBalancer(LoadBalancerClientFactory factory) {
        return new RandomLoadBalancer(
            factory.getLazyProvider("product-service", ServiceInstanceListSupplier.class),
            "product-service");
    }
}
```

<!-- isw:35-8 -->
**In simple words**

Load balancing with Feign is already automatic (seen in the Service Discovery note), and **everything is the same as with RestTemplate, with no change:** the controller just calls `productClient.getProductById(id)`, and the algorithm override uses the same `@LoadBalancerClient` and config class.
<!-- /isw:35-8 -->


---

## Quick revision
- Server-side LB = separate component (Nginx/ELB). Client-side LB = inside the caller (Spring Cloud LoadBalancer, Ribbon, Istio sidecar).
- `@LoadBalanced` on `RestTemplate` → framework intercepts, asks discovery, applies algorithm; use `http://<serviceId>` as base URL.
- Default algorithm = **RoundRobin**; built-in options = **RoundRobin, Random** only.
- Per-service algorithm: `@LoadBalancerClient(name, configuration)`; global default + overrides: `@LoadBalancerClients(defaultConfiguration, value)` with `@ConditionalOnMissingBean` on the global config.
- Weighted / least-connection → custom `ReactorServiceInstanceLoadBalancer` or Istio.
- FeignClient needs no code change.

> Note: diagrams from the original PDF are not reproduced; all text and code is.

---

<!-- nav -->
⬅ [34 Service Discovery](34_Service_Discovery_Notes.md) · [Index](00_HLD_Index.md) · [37 Bulkhead](37_Bulkhead_Notes.md) ➡
<!-- nav -->
