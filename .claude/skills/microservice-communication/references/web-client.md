# WebClient — Reactive HTTP Client (Spring WebFlux)

`WebClient` is non-blocking and reactive. Use it when:
- The service is already built on Spring WebFlux (reactive stack)
- You need to make **concurrent** downstream calls efficiently (non-blocking I/O)
- You need **streaming** responses
- You want to use `RestClient` but your service must handle high concurrency without threads blocking

WebClient can also be used in a traditional (blocking) MVC service — call `.block()` at the boundary.
However, prefer `RestClient` for purely blocking MVC services; use WebClient when reactive semantics are needed.

---

## Gradle Dependencies

```groovy
implementation 'org.springframework.boot:spring-boot-starter-webflux'
```

If using WebClient **only** for outbound calls in an MVC service (not a reactive server):
```groovy
// webflux brings in reactor-netty; that's fine even in MVC
implementation 'org.springframework.boot:spring-boot-starter-webflux'
```

---

## Client Properties

```java
package com.joshsoftware.app.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;

@Validated
@ConfigurationProperties(prefix = "app.clients.payment-service")
public record PaymentServiceClientProperties(
        @NotBlank String baseUrl,
        @Positive int connectTimeoutMs,
        @Positive int responseTimeoutMs
) {}
```

```yaml
# application.yml
app:
  clients:
    payment-service:
      base-url: ${PAYMENT_SERVICE_BASE_URL:http://payment-service:8083}
      connect-timeout-ms: 3000
      response-timeout-ms: 10000
```

---

## WebClient Bean Configuration

```java
package com.joshsoftware.app.config;

import io.netty.channel.ChannelOption;
import lombok.RequiredArgsConstructor;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import org.springframework.web.reactive.function.client.ExchangeFilterFunction;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.netty.http.client.HttpClient;

import java.time.Duration;

@Configuration
@RequiredArgsConstructor
@EnableConfigurationProperties(PaymentServiceClientProperties.class)
public class WebClientConfig {

    private final PaymentServiceClientProperties props;

    @Bean("paymentServiceWebClient")
    public WebClient paymentServiceWebClient() {
        HttpClient httpClient = HttpClient.create()
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, props.connectTimeoutMs())
                .responseTimeout(Duration.ofMillis(props.responseTimeoutMs()));

        return WebClient.builder()
                .baseUrl(props.baseUrl())
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .defaultHeader("Content-Type", "application/json")
                .filter(jwtForwardingFilter())
                .filter(loggingFilter())
                .build();
    }

    private ExchangeFilterFunction jwtForwardingFilter() {
        return ExchangeFilterFunction.ofRequestProcessor(clientRequest -> {
            var attrs = org.springframework.web.context.request.RequestContextHolder
                    .getRequestAttributes();
            if (attrs instanceof org.springframework.web.context.request.ServletRequestAttributes sra) {
                String auth = sra.getRequest().getHeader("Authorization");
                if (org.springframework.util.StringUtils.hasText(auth)) {
                    return reactor.core.publisher.Mono.just(
                        org.springframework.web.reactive.function.client.ClientRequest
                            .from(clientRequest)
                            .header("Authorization", auth)
                            .build()
                    );
                }
            }
            return reactor.core.publisher.Mono.just(clientRequest);
        });
    }

    private ExchangeFilterFunction loggingFilter() {
        return ExchangeFilterFunction.ofResponseProcessor(response -> {
            if (response.statusCode().isError()) {
                return response.bodyToMono(String.class)
                        .doOnNext(body -> org.slf4j.LoggerFactory
                                .getLogger(WebClientConfig.class)
                                .error("WebClient error response: status={} body={}",
                                        response.statusCode(), body))
                        .thenReturn(response);
            }
            return reactor.core.publisher.Mono.just(response);
        });
    }
}
```

---

## Client Wrapper — Reactive Usage

```java
package com.joshsoftware.app.client;

import com.joshsoftware.app.dto.response.PaymentResponseDTO;
import com.joshsoftware.app.exception.ServiceCallException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.HttpStatusCode;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import org.springframework.web.reactive.function.client.WebClientResponseException;
import reactor.core.publisher.Mono;

import java.util.UUID;

@Slf4j
@Component
@RequiredArgsConstructor
public class PaymentServiceClient {

    @Qualifier("paymentServiceWebClient")
    private final WebClient webClient;

    /**
     * Reactive call — returns Mono. Subscribe in the calling service.
     */
    public Mono<PaymentResponseDTO> getPayment(UUID paymentId) {
        log.debug("[PaymentServiceClient#getPayment] ENTRY - paymentId={}", paymentId);
        return webClient.get()
                .uri("/api/v1/payments/{id}", paymentId)
                .retrieve()
                .onStatus(HttpStatusCode::is4xxClientError, response ->
                        Mono.error(new ServiceCallException("Payment not found: " + paymentId)))
                .onStatus(HttpStatusCode::is5xxServerError, response ->
                        Mono.error(new ServiceCallException("Payment service error for: " + paymentId)))
                .bodyToMono(PaymentResponseDTO.class)
                .doOnSuccess(r -> log.debug("[PaymentServiceClient#getPayment] EXIT - paymentId={}", paymentId))
                .doOnError(ex -> log.error("[PaymentServiceClient#getPayment] ERROR - paymentId={}", paymentId, ex));
    }
}
```

---

## Client Wrapper — Blocking Usage (in MVC service)

When called from a traditional MVC service, block at the service layer boundary:

```java
package com.joshsoftware.app.service.impl;

import com.joshsoftware.app.client.PaymentServiceClient;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {

    private final PaymentServiceClient paymentServiceClient;

    @Override
    public void processOrder(UUID orderId, UUID paymentId) {
        log.debug("[OrderServiceImpl#processOrder] ENTRY - orderId={}", orderId);

        // Block at the service layer — never block inside a reactive chain
        var payment = paymentServiceClient.getPayment(paymentId)
                .block(Duration.ofSeconds(10));  // match response timeout

        log.debug("[OrderServiceImpl#processOrder] EXIT - orderId={}", orderId);
    }
}
```

---

## Parallel Calls with WebClient

```java
// Call two downstream services concurrently — Mono.zip() runs them in parallel
public OrderSummary getOrderSummary(UUID orderId, UUID customerId) {
    Mono<OrderResponseDTO>    orderMono    = orderServiceClient.getOrder(orderId);
    Mono<CustomerResponseDTO> customerMono = customerServiceClient.getCustomer(customerId);

    return Mono.zip(orderMono, customerMono)
            .map(tuple -> new OrderSummary(tuple.getT1(), tuple.getT2()))
            .block(Duration.ofSeconds(10));
}
```

---

## Rules

- Use `responseTimeout` on `HttpClient` — controls the full response wait, not just connect
- Use `ExchangeFilterFunction` for cross-cutting concerns (auth, logging) — never inline them
- One `WebClient` bean per target service
- In MVC services, call `.block()` only at the service layer — never inside repository or controller
- For parallel calls, use `Mono.zip()` — never block-then-call sequentially