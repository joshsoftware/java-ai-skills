# Apache Kafka — Asynchronous Event Streaming (Spring Kafka)

Use Kafka for high-throughput, ordered, durable event streaming. Ideal for:
- Audit trails and event sourcing
- Decoupling services that process high-volume data (payments, transactions, notifications)
- Exactly-once event delivery (with transactional producers)

---

## Gradle Dependencies

```groovy
implementation 'org.springframework.kafka:spring-kafka'
// Jackson for JSON serialization (already present with spring-boot-starter-web)
implementation 'com.fasterxml.jackson.core:jackson-databind'
```

---

## Topic Constants

```java
package com.joshsoftware.app.messaging;

public interface PaymentTopics {
    String PAYMENT_INITIATED  = "payment.initiated";
    String PAYMENT_CONFIRMED  = "payment.confirmed";
    String PAYMENT_FAILED     = "payment.failed";
    String PAYMENT_INITIATED_DLT = "payment.initiated.DLT";  // Dead Letter Topic
}
```

---

## Event Payload (Immutable Record)

```java
package com.joshsoftware.app.messaging.event;

import com.fasterxml.jackson.annotation.JsonFormat;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

/**
 * Immutable event payload. Versioned via eventType field.
 * Adding fields is backwards-compatible; removing fields is a breaking change.
 */
public record PaymentInitiatedEvent(
        UUID eventId,
        String eventType,               // "PaymentInitiatedEvent" — for consumer deserialization
        UUID paymentId,
        UUID orderId,
        UUID customerId,
        BigDecimal amount,
        String currency,
        @JsonFormat(shape = JsonFormat.Shape.STRING) Instant occurredAt
) {
    public static PaymentInitiatedEvent of(UUID paymentId, UUID orderId,
                                            UUID customerId, BigDecimal amount, String currency) {
        return new PaymentInitiatedEvent(
                UUID.randomUUID(),
                "PaymentInitiatedEvent",
                paymentId, orderId, customerId,
                amount, currency,
                Instant.now()
        );
    }
}
```

---

## Producer Configuration

```java
package com.joshsoftware.app.config;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaProducerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "all");           // strongest durability guarantee
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);  // exactly-once at producer level
        config.put(JsonSerializer.ADD_TYPE_INFO_HEADERS, false);     // avoid type header coupling
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

---

## Producer

```java
package com.joshsoftware.app.messaging.producer;

import com.joshsoftware.app.messaging.PaymentTopics;
import com.joshsoftware.app.messaging.event.PaymentInitiatedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Component;

import java.util.concurrent.CompletableFuture;

@Slf4j
@Component
@RequiredArgsConstructor
public class PaymentEventProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    /**
     * Publish a payment event. Key = paymentId (ensures ordering per payment).
     */
    public void publishPaymentInitiated(PaymentInitiatedEvent event) {
        log.info("[PaymentEventProducer#publishPaymentInitiated] Publishing - paymentId={}",
                event.paymentId());

        CompletableFuture<SendResult<String, Object>> future =
                kafkaTemplate.send(PaymentTopics.PAYMENT_INITIATED,
                        event.paymentId().toString(),   // partition key
                        event);

        future.whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("[PaymentEventProducer#publishPaymentInitiated] FAILED - paymentId={}",
                        event.paymentId(), ex);
            } else {
                log.info("[PaymentEventProducer#publishPaymentInitiated] SUCCESS - paymentId={} offset={}",
                        event.paymentId(),
                        result.getRecordMetadata().offset());
            }
        });
    }
}
```

---

## Consumer Configuration

```java
package com.joshsoftware.app.config;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer;
import org.springframework.kafka.listener.DefaultErrorHandler;
import org.springframework.kafka.support.serializer.JsonDeserializer;
import org.springframework.util.backoff.FixedBackOff;

import java.util.HashMap;
import java.util.Map;

@EnableKafka
@Configuration
public class KafkaConsumerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${spring.kafka.consumer.group-id}")
    private String groupId;

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);     // manual ack for reliability
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "com.joshsoftware.*");
        config.put(JsonDeserializer.USE_TYPE_INFO_HEADERS, false);
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory(
            KafkaTemplate<String, Object> kafkaTemplate) {

        var factory = new ConcurrentKafkaListenerContainerFactory<String, Object>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties().setAckMode(
                org.springframework.kafka.listener.ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        factory.setConcurrency(3);  // number of consumer threads per listener

        // Dead Letter Topic: after 3 retries (500ms apart), publish to <topic>.DLT
        var recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);
        var errorHandler = new DefaultErrorHandler(recoverer, new FixedBackOff(500L, 3L));
        factory.setCommonErrorHandler(errorHandler);

        return factory;
    }
}
```

---

## Consumer

```java
package com.joshsoftware.app.messaging.consumer;

import com.joshsoftware.app.messaging.PaymentTopics;
import com.joshsoftware.app.messaging.event.PaymentInitiatedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.annotation.RetryableTopic;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.retry.annotation.Backoff;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class PaymentEventConsumer {

    private final PaymentProcessingService paymentProcessingService;

    @RetryableTopic(
        attempts = "3",
        backoff = @Backoff(delay = 500, multiplier = 2.0),
        dltTopicSuffix = ".DLT"
    )
    @KafkaListener(
        topics = PaymentTopics.PAYMENT_INITIATED,
        groupId = "${spring.kafka.consumer.group-id}",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void onPaymentInitiated(@Payload PaymentInitiatedEvent event, Acknowledgment ack) {
        log.info("[PaymentEventConsumer#onPaymentInitiated] ENTRY - paymentId={} eventId={}",
                event.paymentId(), event.eventId());
        try {
            paymentProcessingService.process(event);
            ack.acknowledge();
            log.info("[PaymentEventConsumer#onPaymentInitiated] EXIT - paymentId={}", event.paymentId());
        } catch (Exception ex) {
            log.error("[PaymentEventConsumer#onPaymentInitiated] FAILED - paymentId={}", event.paymentId(), ex);
            throw ex;  // re-throw to trigger retry / DLT routing
        }
    }
}
```

---

## application.yml — Kafka Config

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      acks: all
      retries: 3
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: ${KAFKA_CONSUMER_GROUP_ID:payment-service}
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.joshsoftware.*"
```

---

## Rules

- Partition key = the business entity ID (e.g. `paymentId`) — ensures ordered processing per entity
- `enable.auto.commit = false` + manual ack — prevents offset commit on processing failure
- `ACKS_ALL` + `enable.idempotence = true` — no duplicate messages from producer retries
- `@RetryableTopic` over `@KafkaListener` retry — uses separate retry topics, keeps main topic healthy
- Re-throw exceptions from listener — triggers retry + DLT routing; never swallow and ack on failure
- DLT consumers must be monitored and alerted — a message in DLT means a processing failure that needs investigation
- Event records are immutable (`record`) — never mutate a consumed event; create a new one if needed
- Never call another service synchronously inside a Kafka listener — breaks isolation; publish a follow-up event instead