---
name: microservice-communication
description: >
  Generates production-ready microservice inter-communication code following Josh Software standards.
  Use this skill whenever a user asks to:
  - Call another microservice (REST, gRPC, or message-based)
  - Set up a synchronous HTTP client (RestClient, WebClient, or OpenFeign)
  - Set up an asynchronous message channel (Kafka, RabbitMQ, or AWS SQS)
  - Add resilience patterns (circuit breaker, retry, timeout, bulkhead)
  - Propagate authentication headers (JWT, API key) between services
  - Set up service discovery (Eureka, Consul)
  - Handle dead-letter topics / queues for failed messages
  - Add distributed tracing (Micrometer + Zipkin / Jaeger)
  Always use this skill — even for partial requests like "call the order service",
  "set up a Kafka producer", "add a circuit breaker", or "configure OpenFeign".
  IMPORTANT: Never suggest or generate RestTemplate — it is in maintenance mode as of
  Spring 5. Use RestClient (Spring Boot 3.2+), WebClient, or OpenFeign instead.
---

# Microservice Inter-Communication Skill — Josh Software Standards

Generates clean, production-grade inter-service communication code for Spring Boot 3.4.x / Java 21.

**RestTemplate is in maintenance mode** (soft-deprecated since Spring 5; no new features added).
This skill always uses:
- `RestClient` (Spring Boot 3.2+) for simple synchronous HTTP
- `WebClient` (Spring WebFlux) for reactive / non-blocking HTTP
- `OpenFeign` (Spring Cloud) for declarative contract-first HTTP

---

## CRITICAL DEFAULT BEHAVIOURS — Always Active

- **Never generate RestTemplate** — always use RestClient, WebClient, or OpenFeign
- **Always propagate the `Authorization` header** from the incoming request to outgoing calls (JWT pass-through)
- **Always add a timeout** — no HTTP client call without `connectTimeout` + `readTimeout`
- **Always handle errors explicitly** — no silent swallowing of non-2xx responses
- **Always use DTOs** for request/response payloads — never pass raw `Map<String, Object>`
- **Never put base URLs in code** — always bind from `application.yml` via `@ConfigurationProperties`

---

## How to Use This Skill

### Step 0 — Ask These Questions Before Generating Anything

Ask each question in order, wait for the answer, then move to the next.
Never bundle questions. Only generate code once all answers are confirmed.

---

#### Question 1 — Communication Style
> "What **type of inter-service communication** do you need?"
>
> - **Synchronous** — caller waits for a response (REST over HTTP)
> - **Asynchronous** — fire-and-forget or event-driven (message broker)
> - **Both** — generate both synchronous and asynchronous channels

Record the answer. It determines which follow-up questions are asked.

---

#### Question 2 — Synchronous Client *(ask only if Synchronous or Both)*
> "Which **synchronous HTTP client** do you want to use?"
>
> - **RestClient** — Spring Boot 3.2+ fluent API; simple, synchronous, blocking. Best default for most services.
> - **WebClient** — Spring WebFlux reactive client; non-blocking. Use when the service is already reactive or you need streaming.
> - **OpenFeign** — Declarative `@FeignClient` interface; minimal boilerplate, auto-generated from the interface. Best when consuming a well-defined external API contract.
>
> After confirming choice, read `references/rest-client.md`, `references/web-client.md`, or `references/open-feign.md` accordingly.

**Follow-up if any sync client is chosen:**
> "What is the **name of the target service** and its **base URL config key**?"
> Example: `order-service` with base URL at `app.clients.order-service.base-url`
>
> Use the answer to name the client class and bind the URL from config.

> "Which **HTTP operations** do you need?"
> - [ ] GET (fetch by ID, fetch list)
> - [ ] POST (create resource)
> - [ ] PUT / PATCH (update resource)
> - [ ] DELETE (remove resource)

---

#### Question 3 — Asynchronous Broker *(ask only if Asynchronous or Both)*
> "Which **message broker** will you use?"
>
> - **Apache Kafka** — high-throughput, ordered event streaming; partition-based; best for audit trails, event sourcing, and high-volume data pipelines. Read `references/kafka.md`.
> - **RabbitMQ** — flexible routing via exchanges and bindings; best for task queues, work distribution, and request-reply patterns. Read `references/rabbitmq.md`.
> - **AWS SQS / SNS** — managed cloud queue; best when already on AWS and want no-ops overhead. Read `references/aws-messaging.md`.

**Follow-up if Kafka:**
> - "What is the **topic name**?" (e.g. `order-created`, `payment-processed`)
> - "What is the **consumer group ID**?"
> - "Do you need a **Dead Letter Topic (DLT)** for failed messages?" (Yes / No)
> - "Do you need a **transactional producer** (exactly-once semantics)?" (Yes / No)

**Follow-up if RabbitMQ:**
> - "What is the **exchange name** and **routing key**?"
> - "What is the **queue name**?"
> - "Do you need a **Dead Letter Queue (DLQ)** for failed messages?" (Yes / No)
> - "Exchange type?" (Direct / Topic / Fanout / Headers)

**Follow-up if AWS SQS:**
> - "What is the **SQS queue URL config key**?"
> - "Do you need a **Dead Letter Queue (DLQ)**?" (Yes / No)
> - "Standard queue or **FIFO**?" (Standard / FIFO)

---

#### Question 4 — Resilience Patterns
> "Do you need **resilience patterns** (circuit breaker, retry, timeout, bulkhead)?"
>
> - **Yes** → ask the follow-up below, then read `references/resilience.md`
> - **No** → skip, but still enforce mandatory timeouts on all HTTP clients

**Follow-up if Yes:**
> "Which patterns do you need?" *(select all that apply)*
> - [ ] **Circuit Breaker** — stop calling a failing service after a threshold; fast-fail for a cooldown window
> - [ ] **Retry with exponential backoff** — retry transient failures with increasing delay (max 3 retries by default)
> - [ ] **Timeout** — fail fast if the call takes too long (always included even if other patterns are skipped)
> - [ ] **Bulkhead** — limit concurrent calls to a downstream service to protect thread pool
> - [ ] **Rate Limiter** — cap calls per second to avoid overwhelming a downstream service

> "Should resilience be applied via **annotations** (`@CircuitBreaker`, `@Retry`) or **programmatic** `Resilience4j` config?"
> - **Annotations** (simpler, AOP-based)
> - **Programmatic** (fine-grained control, test-friendly)

---

#### Question 5 — Authentication Header Propagation
> "Does the target service require **authentication header propagation** from the caller's JWT or an API key?"
>
> - **JWT pass-through** — extract `Authorization: Bearer <token>` from the incoming request and forward it to outgoing calls
> - **Service-to-service API key** — attach a fixed API key from config (not from the caller's token)
> - **OAuth2 client credentials** — obtain a service token via client credentials flow and attach it
> - **None** — the target service is internal and unauthenticated

After confirming, generate the appropriate `ExchangeFilterFunction` (for WebClient / RestClient) or `RequestInterceptor` (for OpenFeign).

---

#### Question 6 — Service Discovery
> "Do you need **service discovery** so service URLs are resolved dynamically rather than hardcoded?"
>
> - **Spring Cloud Eureka** — client-side discovery; service registers with Eureka server; caller resolves via `@LoadBalanced`
> - **Consul** — similar to Eureka with health-check support
> - **Kubernetes DNS** — no client-side library; use `http://service-name.namespace.svc.cluster.local` base URL
> - **No** — base URL is static, loaded from config

> If Eureka or Consul: "Should I add the **`@LoadBalanced`** annotation to the client bean and the `spring-cloud-starter-loadbalancer` dependency?"

---

#### Question 7 — Distributed Tracing
> "Do you need **distributed tracing** to correlate logs and spans across services?"
>
> - **Yes — Zipkin** → add Micrometer Tracing + Zipkin exporter; read tracing config
> - **Yes — Jaeger** → add Micrometer Tracing + OTLP exporter for Jaeger
> - **No** → skip, but remind user to propagate `X-Correlation-ID` headers manually

---

> Only proceed to code generation once **all seven questions** are answered.

---

### Step 1 — Read Reference Files

Read only the files for features confirmed in Step 0:

| Confirmed Feature                       | Read this file                              |
|-----------------------------------------|---------------------------------------------|
| RestClient (sync)                       | `references/rest-client.md`               |
| WebClient (reactive sync/async)         | `references/web-client.md`                |
| OpenFeign (declarative sync)            | `references/open-feign.md`                |
| Kafka (async)                           | `references/kafka.md`                     |
| RabbitMQ (async)                        | `references/rabbitmq.md`                  |
| Resilience4j patterns                   | `references/resilience.md`                |

### Step 2 — Generate Code

- Generate **complete, compilable** classes — never stubs or pseudocode
- Include the **Gradle dependency snippet** for every new library introduced
- Before adding a dependency, check `build.gradle` to avoid duplicates
- All config values (base URLs, topic names, queue names) bound from `application.yml` via `@ConfigurationProperties`
- Entry/exit logs on every client method following the pattern in `boilerplate/references/logging.md`

---

## Package Structure

```
com.joshsoftware.<app>/
├── client/                              ← sync HTTP clients (RestClient / WebClient / Feign)
│   ├── <ServiceName>Client.java         ← RestClient or WebClient wrapper
│   └── <ServiceName>FeignClient.java    ← if OpenFeign
├── config/
│   ├── RestClientConfig.java            ← RestClient bean config
│   ├── WebClientConfig.java             ← WebClient bean config
│   ├── FeignConfig.java                 ← global Feign config (interceptors)
│   └── ClientProperties.java           ← @ConfigurationProperties for base URLs / timeouts
├── messaging/
│   ├── producer/
│   │   └── <Domain>EventProducer.java   ← Kafka / RabbitMQ publisher
│   ├── consumer/
│   │   └── <Domain>EventConsumer.java   ← Kafka / RabbitMQ listener
│   └── event/
│       └── <Domain>Event.java           ← event payload record (immutable)
└── dto/
    ├── request/                         ← outbound call request DTOs
    └── response/                        ← inbound response DTOs from other services
```

---

## Naming Conventions

| Artifact                   | Convention                        | Example                              |
|----------------------------|-----------------------------------|--------------------------------------|
| Sync client wrapper        | `<ServiceName>Client`             | `OrderServiceClient`                 |
| Feign interface            | `<ServiceName>FeignClient`        | `InventoryFeignClient`               |
| Kafka producer             | `<Domain>EventProducer`           | `PaymentEventProducer`               |
| Kafka consumer             | `<Domain>EventConsumer`           | `PaymentEventConsumer`               |
| RabbitMQ producer          | `<Domain>MessagePublisher`        | `NotificationMessagePublisher`       |
| RabbitMQ consumer          | `<Domain>MessageListener`         | `NotificationMessageListener`        |
| Event payload              | `<Domain>Event`                   | `OrderCreatedEvent`                  |
| Client config properties   | `<ServiceName>ClientProperties`   | `OrderServiceClientProperties`       |
| Topic / queue constant     | `<Domain>Topics` or `<Domain>Queues` | `PaymentTopics`, `NotificationQueues` |

---

## Anti-Patterns (Strictly Forbidden)

| Anti-Pattern                                              | Why Forbidden                                            |
|-----------------------------------------------------------|----------------------------------------------------------|
| `new RestTemplate()`                                      | Maintenance mode — use `RestClient` or `WebClient`       |
| Base URL hardcoded in client class                        | Breaks per-environment deployment                        |
| No timeout on HTTP client                                 | One slow service cascades to full outage                 |
| Catching `Exception` and returning null                   | Silent failure masks downstream errors                   |
| Raw `Map<String, Object>` as request/response payload     | No type safety, no validation                            |
| `@Autowired` field injection into client                  | Hidden dependency; use constructor injection             |
| Consuming Kafka without a DLT for poison-pill messages    | One bad message blocks the entire partition              |
| Sharing a consumer group across unrelated services        | Partition assignment conflicts                           |
| Publishing events inside `@Transactional` without outbox  | Event fires even if DB transaction rolls back            |
| `ObjectMapper` instantiated per call                      | Expensive; inject the Spring-managed singleton bean      |