# RestClient — Synchronous HTTP Client (Spring Boot 3.2+)

`RestClient` is the modern replacement for `RestTemplate`. It offers a fluent API, is synchronous and blocking, and requires no WebFlux dependency. Default choice for simple service-to-service HTTP calls.

---

## Gradle Dependency

No extra dependency — `RestClient` is bundled in `spring-boot-starter-web` (Spring Boot 3.2+).

```groovy
// Already present if you have:
implementation 'org.springframework.boot:spring-boot-starter-web'
```

---

## Client Properties (Config Binding)

```java
package com.joshsoftware.app.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;

@Validated
@ConfigurationProperties(prefix = "app.clients.order-service")
public record OrderServiceClientProperties(
        @NotBlank String baseUrl,
        @Positive int connectTimeoutMs,
        @Positive int readTimeoutMs
) {}
```

```yaml
# application.yml
app:
  clients:
    order-service:
      base-url: ${ORDER_SERVICE_BASE_URL:http://order-service:8081}
      connect-timeout-ms: 3000
      read-timeout-ms: 5000
```

---

## RestClient Bean Configuration

```java
package com.joshsoftware.app.config;

import lombok.RequiredArgsConstructor;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.SimpleClientHttpRequestFactory;
import org.springframework.web.client.RestClient;

@Configuration
@RequiredArgsConstructor
@EnableConfigurationProperties(OrderServiceClientProperties.class)
public class RestClientConfig {

    private final OrderServiceClientProperties props;

    @Bean("orderServiceRestClient")
    public RestClient orderServiceRestClient() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(props.connectTimeoutMs());
        factory.setReadTimeout(props.readTimeoutMs());

        return RestClient.builder()
                .baseUrl(props.baseUrl())
                .requestFactory(factory)
                .defaultHeader("Accept", "application/json")
                .defaultHeader("Content-Type", "application/json")
                .requestInterceptor(new JwtForwardingInterceptor())  // see auth section
                .build();
    }
}
```

---

## JWT Forwarding Interceptor

```java
package com.joshsoftware.app.config;

import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.HttpRequest;
import org.springframework.http.client.ClientHttpRequestExecution;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.util.StringUtils;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import java.io.IOException;

/**
 * Forwards the caller's Authorization header to downstream service calls.
 * Only attaches the header when a request context is present (HTTP thread).
 */
public class JwtForwardingInterceptor implements ClientHttpRequestInterceptor {

    @Override
    public ClientHttpResponse intercept(HttpRequest request,
                                        byte[] body,
                                        ClientHttpRequestExecution execution) throws IOException {
        var attrs = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attrs != null) {
            HttpServletRequest incoming = attrs.getRequest();
            String auth = incoming.getHeader("Authorization");
            if (StringUtils.hasText(auth)) {
                request.getHeaders().set("Authorization", auth);
            }
        }
        return execution.execute(request, body);
    }
}
```

---

## Client Wrapper

```java
package com.joshsoftware.app.client;

import com.joshsoftware.app.dto.response.OrderResponseDTO;
import com.joshsoftware.app.exception.ServiceCallException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.HttpStatusCode;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.RestClientResponseException;

import java.util.List;
import java.util.UUID;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderServiceClient {

    @Qualifier("orderServiceRestClient")
    private final RestClient restClient;

    public OrderResponseDTO getOrderById(UUID orderId) {
        log.debug("[OrderServiceClient#getOrderById] ENTRY - orderId={}", orderId);
        try {
            OrderResponseDTO response = restClient.get()
                    .uri("/api/v1/orders/{id}", orderId)
                    .retrieve()
                    .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
                        throw new ServiceCallException("Order not found or bad request: " + orderId);
                    })
                    .onStatus(HttpStatusCode::is5xxServerError, (req, res) -> {
                        throw new ServiceCallException("Order service error for orderId: " + orderId);
                    })
                    .body(OrderResponseDTO.class);

            log.debug("[OrderServiceClient#getOrderById] EXIT - orderId={}", orderId);
            return response;
        } catch (RestClientResponseException ex) {
            log.error("[OrderServiceClient#getOrderById] HTTP error - status={} orderId={}",
                    ex.getStatusCode(), orderId);
            throw new ServiceCallException("Failed to fetch order " + orderId, ex);
        }
    }

    public List<OrderResponseDTO> getOrdersByCustomer(UUID customerId) {
        log.debug("[OrderServiceClient#getOrdersByCustomer] ENTRY - customerId={}", customerId);
        try {
            List<OrderResponseDTO> orders = restClient.get()
                    .uri("/api/v1/orders?customerId={id}", customerId)
                    .retrieve()
                    .onStatus(HttpStatusCode::isError, (req, res) -> {
                        throw new ServiceCallException("Order service error for customerId: " + customerId);
                    })
                    .body(new org.springframework.core.ParameterizedTypeReference<>() {});

            log.debug("[OrderServiceClient#getOrdersByCustomer] EXIT - count={}", orders.size());
            return orders;
        } catch (RestClientResponseException ex) {
            log.error("[OrderServiceClient#getOrdersByCustomer] HTTP error - status={}", ex.getStatusCode());
            throw new ServiceCallException("Failed to fetch orders for customer " + customerId, ex);
        }
    }
}
```

---

## ServiceCallException

```java
package com.joshsoftware.app.exception;

public class ServiceCallException extends RuntimeException {
    public ServiceCallException(String message) { super(message); }
    public ServiceCallException(String message, Throwable cause) { super(message, cause); }
}
```

---

## application.yml — Full Client Config

```yaml
app:
  clients:
    order-service:
      base-url: ${ORDER_SERVICE_BASE_URL:http://order-service:8081}
      connect-timeout-ms: ${ORDER_SERVICE_CONNECT_TIMEOUT_MS:3000}
      read-timeout-ms: ${ORDER_SERVICE_READ_TIMEOUT_MS:5000}
    inventory-service:
      base-url: ${INVENTORY_SERVICE_BASE_URL:http://inventory-service:8082}
      connect-timeout-ms: 2000
      read-timeout-ms: 4000
```

---

## Rules

- Always use `.onStatus()` for 4xx and 5xx — never let non-2xx responses silently return null
- Always set both `connectTimeout` and `readTimeout` — no defaults are safe in production
- Base URL always from `@ConfigurationProperties` — never hardcoded
- One `RestClient` bean per target service — different timeouts, base URLs, interceptors
- Use `@Qualifier` when injecting if multiple `RestClient` beans exist