# Resilience Patterns — Resilience4j (Spring Boot 3.x)

Use Resilience4j for circuit breaking, retry, timeout, bulkhead, and rate limiting.
**Never use Hystrix** — it is end-of-life and incompatible with Spring Boot 3.x.
Resilience4j integrates with all three sync clients (RestClient, WebClient, OpenFeign) via AOP annotations or programmatic API.

---

## Gradle Dependencies

```groovy
implementation 'io.github.resilience4j:resilience4j-spring-boot3:2.2.0'
implementation 'org.springframework.boot:spring-boot-starter-aop'   // required for annotations
implementation 'org.springframework.boot:spring-boot-starter-actuator'  // exposes metrics
```

---

## application.yml — Full Resilience4j Config

```yaml
resilience4j:

  # --- Circuit Breaker ---
  circuitbreaker:
    configs:
      default:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10             # evaluate last 10 calls
        failure-rate-threshold: 50          # open circuit if >=50% fail
        slow-call-duration-threshold: 3s    # calls slower than 3s count as slow
        slow-call-rate-threshold: 80        # open if >=80% are slow
        wait-duration-in-open-state: 30s    # stay open for 30s before half-opening
        permitted-calls-in-half-open-state: 3
        register-health-indicator: true
    instances:
      order-service:
        base-config: default
      inventory-service:
        base-config: default
        failure-rate-threshold: 40          # stricter for inventory

  # --- Retry ---
  retry:
    configs:
      default:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2.0
        retry-exceptions:
          - org.springframework.web.client.ResourceAccessException
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.joshsoftware.app.exception.ResourceNotFoundException  # 404 — don't retry
          - com.joshsoftware.app.exception.ServiceCallException        # 4xx — don't retry
    instances:
      order-service:
        base-config: default
      inventory-service:
        base-config: default
        max-attempts: 2              # fewer retries for fast-fail

  # --- Timeout ---
  timelimiter:
    configs:
      default:
        timeout-duration: 5s
        cancel-running-future: true
    instances:
      order-service:
        timeout-duration: 5s
      inventory-service:
        timeout-duration: 3s       # stricter SLA

  # --- Bulkhead (thread pool isolation) ---
  bulkhead:
    configs:
      default:
        max-concurrent-calls: 20
        max-wait-duration: 10ms    # fail fast if all slots are busy
    instances:
      order-service:
        base-config: default
      inventory-service:
        max-concurrent-calls: 10   # fewer slots — protect against slow inventory calls

  # --- Rate Limiter ---
  ratelimiter:
    configs:
      default:
        limit-for-period: 100       # 100 calls per refresh period
        limit-refresh-period: 1s
        timeout-duration: 500ms     # wait up to 500ms for a permit
    instances:
      payment-gateway:
        limit-for-period: 50        # payment gateway has its own rate cap
        limit-refresh-period: 1s
```

---

## Annotation-Based Usage (on Client Wrapper Methods)

Apply annotations in this order (outermost to innermost): `@RateLimiter` → `@Bulkhead` → `@CircuitBreaker` → `@Retry` → `@TimeLimiter`

```java
package com.joshsoftware.app.client;

import com.joshsoftware.app.dto.response.OrderResponseDTO;
import com.joshsoftware.app.exception.ServiceCallException;
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import io.github.resilience4j.timelimiter.annotation.TimeLimiter;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

import java.util.UUID;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderServiceClient {

    @Qualifier("orderServiceRestClient")
    private final RestClient restClient;

    @CircuitBreaker(name = "order-service", fallbackMethod = "getOrderFallback")
    @Retry(name = "order-service")
    @Bulkhead(name = "order-service", type = Bulkhead.Type.SEMAPHORE)
    public OrderResponseDTO getOrderById(UUID orderId) {
        log.debug("[OrderServiceClient#getOrderById] ENTRY - orderId={}", orderId);
        OrderResponseDTO response = restClient.get()
                .uri("/api/v1/orders/{id}", orderId)
                .retrieve()
                .body(OrderResponseDTO.class);
        log.debug("[OrderServiceClient#getOrderById] EXIT - orderId={}", orderId);
        return response;
    }

    /**
     * Fallback invoked when circuit is open or all retries exhausted.
     * Must have the same signature as the primary method + a Throwable parameter.
     */
    private OrderResponseDTO getOrderFallback(UUID orderId, Throwable ex) {
        log.warn("[OrderServiceClient#getOrderFallback] Circuit open / retries exhausted - orderId={} cause={}",
                orderId, ex.getMessage());
        // Return a safe cached/default value, or throw a domain exception
        throw new ServiceCallException("Order service unavailable for orderId: " + orderId);
    }
}
```

---

## Programmatic Usage (Fine-Grained Control)

Use programmatic API when you need per-call config, testing, or conditional resilience:

```java
package com.joshsoftware.app.client;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.github.resilience4j.retry.Retry;
import io.github.resilience4j.retry.RetryRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.UUID;
import java.util.function.Supplier;

@Slf4j
@Component
public class ResilientInventoryClient {

    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final InventoryFeignClient feignClient;

    public ResilientInventoryClient(CircuitBreakerRegistry cbRegistry,
                                     RetryRegistry retryRegistry,
                                     InventoryFeignClient feignClient) {
        this.circuitBreaker = cbRegistry.circuitBreaker("inventory-service");
        this.retry = retryRegistry.retry("inventory-service");
        this.feignClient = feignClient;
    }

    public InventoryResponseDTO getStock(UUID productId) {
        Supplier<InventoryResponseDTO> decorated = CircuitBreaker.decorateSupplier(
                circuitBreaker,
                Retry.decorateSupplier(retry,
                        () -> feignClient.getStock(productId)
                )
        );

        return io.vavr.control.Try.ofSupplier(decorated)
                .recover(ex -> {
                    log.warn("[ResilientInventoryClient#getStock] Fallback - productId={}", productId);
                    return InventoryResponseDTO.unavailable();
                })
                .get();
    }
}
```

---

## Actuator Endpoints (Observability)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, circuitbreakers, retries
  endpoint:
    health:
      show-details: always
  health:
    circuitbreakers:
      enabled: true
```

Circuit breaker state is visible at:
- `GET /actuator/health` — shows `CIRCUIT_BREAKER_CLOSED / OPEN / HALF_OPEN`
- `GET /actuator/metrics/resilience4j.circuitbreaker.state?tag=name:order-service`

---

## Rules

| Pattern         | When to Use                                                                 |
|-----------------|-----------------------------------------------------------------------------|
| Circuit Breaker | Any external service call — prevents cascade failures                       |
| Retry           | Transient network / timeout errors only — never retry 4xx business errors   |
| Timeout         | Every HTTP call — pair with circuit breaker to limit blast radius           |
| Bulkhead        | Slow downstream service that would exhaust the thread pool                  |
| Rate Limiter    | External paid APIs or services with rate caps (payment gateways, SMS APIs)  |

- Fallback method signature must match the primary method + `Throwable` as the last parameter
- Never retry `ResourceNotFoundException` (404) or validation errors — only network/timeout exceptions
- Circuit breaker `name` in annotation must match the key in `application.yml` under `resilience4j.circuitbreaker.instances`
- Always expose `/actuator/health` with circuit breaker details — ops team needs visibility into circuit state
- Add `resilience4j.circuitbreaker.register-health-indicator: true` so health checks reflect circuit state