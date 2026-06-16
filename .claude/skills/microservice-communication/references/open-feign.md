# OpenFeign — Declarative HTTP Client (Spring Cloud)

OpenFeign generates a full HTTP client from a Java interface annotated with `@FeignClient`.
Best choice when consuming a well-defined external or internal API where you want minimal boilerplate
and a clean, testable contract.

---

## Gradle Dependencies

```groovy
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:2024.0.0"
    }
}

dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    // Resilience4j integration (if circuit breaker / retry needed)
    implementation 'io.github.resilience4j:resilience4j-spring-boot3:2.2.0'
    implementation 'org.springframework.boot:spring-boot-starter-aop'
}
```

Enable in main class:
```java
@SpringBootApplication
@EnableFeignClients(basePackages = "com.joshsoftware.app.client")
public class Application { ... }
```

---

## Client Properties

```yaml
# application.yml
app:
  clients:
    inventory-service:
      url: ${INVENTORY_SERVICE_BASE_URL:http://inventory-service:8082}

feign:
  client:
    config:
      inventory-service:            # matches @FeignClient(name = "inventory-service")
        connect-timeout: 3000
        read-timeout: 5000
        logger-level: BASIC         # NONE | BASIC | HEADERS | FULL
  compression:
    request:
      enabled: true
    response:
      enabled: true
```

---

## Feign Client Interface

```java
package com.joshsoftware.app.client;

import com.joshsoftware.app.config.FeignClientConfig;
import com.joshsoftware.app.dto.response.InventoryResponseDTO;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestParam;

import java.util.List;
import java.util.UUID;

@FeignClient(
    name = "inventory-service",
    url = "${app.clients.inventory-service.url}",
    configuration = FeignClientConfig.class,
    fallback = InventoryFeignClientFallback.class  // only if Resilience4j fallback is needed
)
public interface InventoryFeignClient {

    @GetMapping("/api/v1/inventory/{productId}")
    InventoryResponseDTO getStock(@PathVariable("productId") UUID productId);

    @GetMapping("/api/v1/inventory")
    List<InventoryResponseDTO> getStockForProducts(@RequestParam("productIds") List<UUID> productIds);

    @PostMapping("/api/v1/inventory/reserve")
    InventoryResponseDTO reserveStock(@RequestBody ReserveStockRequestDTO request);
}
```

---

## Global Feign Configuration (Auth + Logging)

```java
package com.joshsoftware.app.config;

import feign.Logger;
import feign.RequestInterceptor;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.util.StringUtils;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

/**
 * Applied to all @FeignClients via configuration = FeignClientConfig.class.
 * Do NOT annotate with @Configuration — Feign configs must not be in component scan.
 */
public class FeignClientConfig {

    @Bean
    public RequestInterceptor jwtForwardingInterceptor() {
        return template -> {
            var attrs = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            if (attrs != null) {
                HttpServletRequest request = attrs.getRequest();
                String auth = request.getHeader("Authorization");
                if (StringUtils.hasText(auth)) {
                    template.header("Authorization", auth);
                }
                String correlationId = request.getHeader("X-Correlation-ID");
                if (StringUtils.hasText(correlationId)) {
                    template.header("X-Correlation-ID", correlationId);
                }
            }
        };
    }

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.BASIC;  // logs method, URL, status, time — not body
    }
}
```

---

## Custom Error Decoder

```java
package com.joshsoftware.app.config;

import com.joshsoftware.app.exception.ResourceNotFoundException;
import com.joshsoftware.app.exception.ServiceCallException;
import feign.Response;
import feign.codec.ErrorDecoder;
import org.springframework.http.HttpStatus;

/**
 * Maps Feign HTTP error responses to domain exceptions.
 * Register as a @Bean inside FeignClientConfig.
 */
public class FeignErrorDecoder implements ErrorDecoder {

    private final ErrorDecoder defaultDecoder = new Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        HttpStatus status = HttpStatus.valueOf(response.status());
        return switch (status) {
            case NOT_FOUND -> new ResourceNotFoundException(
                    "Resource not found via: " + methodKey);
            case BAD_REQUEST -> new ServiceCallException(
                    "Bad request to downstream service: " + methodKey);
            case UNAUTHORIZED, FORBIDDEN -> new ServiceCallException(
                    "Auth failure calling: " + methodKey);
            default -> status.is5xxServerError()
                    ? new ServiceCallException("Downstream service error: " + methodKey)
                    : defaultDecoder.decode(methodKey, response);
        };
    }
}
```

Add to `FeignClientConfig`:
```java
@Bean
public ErrorDecoder errorDecoder() {
    return new FeignErrorDecoder();
}
```

---

## Fallback (Resilience4j Circuit Breaker)

```java
package com.joshsoftware.app.client;

import com.joshsoftware.app.dto.response.InventoryResponseDTO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.Collections;
import java.util.List;
import java.util.UUID;

@Slf4j
@Component
public class InventoryFeignClientFallback implements InventoryFeignClient {

    @Override
    public InventoryResponseDTO getStock(UUID productId) {
        log.warn("[InventoryFeignClientFallback#getStock] Circuit open - productId={}", productId);
        return InventoryResponseDTO.unavailable();  // return safe default
    }

    @Override
    public List<InventoryResponseDTO> getStockForProducts(List<UUID> productIds) {
        log.warn("[InventoryFeignClientFallback#getStockForProducts] Circuit open");
        return Collections.emptyList();
    }

    @Override
    public InventoryResponseDTO reserveStock(ReserveStockRequestDTO request) {
        log.error("[InventoryFeignClientFallback#reserveStock] Circuit open — reservation failed");
        throw new ServiceCallException("Inventory service unavailable — cannot reserve stock");
    }
}
```

Enable in `application.yml`:
```yaml
spring:
  cloud:
    openfeign:
      circuitbreaker:
        enabled: true
```

---

## Rules

- `FeignClientConfig` must **NOT** be annotated with `@Configuration` — prevents Feign config from being picked up globally via component scan. Register it only via the `configuration =` attribute on `@FeignClient`
- Always provide a `fallback` for mission-critical calls; fallbacks must either return a safe default or throw — never return null
- Use `@FeignClient(url = "${...}")` for URL binding — never hardcode the URL
- Each `@FeignClient` gets its own `feign.client.config.<name>` block in `application.yml` for independent timeouts
- Feign uses `ObjectMapper` for JSON — Spring Boot auto-configures it; do not create a second one
- Log at `BASIC` in prod (method + URL + status + time) — `FULL` (with body) only in dev/debug