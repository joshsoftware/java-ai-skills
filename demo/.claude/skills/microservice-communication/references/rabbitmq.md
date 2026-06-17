# RabbitMQ — Asynchronous Messaging (Spring AMQP)

Use RabbitMQ for flexible routing, task queues, and request-reply patterns. Ideal for:
- Work queues where multiple consumers share load (round-robin delivery)
- Routing messages to different queues based on routing key (Direct / Topic exchange)
- Fan-out broadcasting to multiple queues simultaneously (Fanout exchange)
- Delayed messages and scheduled jobs (via RabbitMQ delayed message plugin)

---

## Gradle Dependencies

```groovy
implementation 'org.springframework.boot:spring-boot-starter-amqp'
```

---

## Queue / Exchange / Routing Key Constants

```java
package com.joshsoftware.app.messaging;

public interface NotificationQueues {

    // Exchange names
    String NOTIFICATION_EXCHANGE     = "notification.exchange";
    String NOTIFICATION_DLX          = "notification.exchange.dlx";   // Dead Letter Exchange

    // Queue names
    String EMAIL_QUEUE               = "notification.email.queue";
    String SMS_QUEUE                 = "notification.sms.queue";
    String PUSH_QUEUE                = "notification.push.queue";
    String EMAIL_DLQ                 = "notification.email.queue.dlq";

    // Routing keys (for Topic exchange: supports * and # wildcards)
    String EMAIL_ROUTING_KEY         = "notification.email";
    String SMS_ROUTING_KEY           = "notification.sms";
    String PUSH_ROUTING_KEY          = "notification.push";
}
```

---

## RabbitMQ Configuration — Exchange, Queue, Bindings

```java
package com.joshsoftware.app.config;

import com.joshsoftware.app.messaging.NotificationQueues;
import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    // --- Dead Letter Exchange + Queue (must be declared first) ---

    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange(NotificationQueues.NOTIFICATION_DLX, true, false);
    }

    @Bean
    public Queue emailDeadLetterQueue() {
        return QueueBuilder.durable(NotificationQueues.EMAIL_DLQ).build();
    }

    @Bean
    public Binding emailDlqBinding() {
        return BindingBuilder.bind(emailDeadLetterQueue())
                .to(deadLetterExchange())
                .with(NotificationQueues.EMAIL_ROUTING_KEY);
    }

    // --- Main Exchange ---

    @Bean
    public TopicExchange notificationExchange() {
        return ExchangeBuilder
                .topicExchange(NotificationQueues.NOTIFICATION_EXCHANGE)
                .durable(true)
                .build();
    }

    // --- Queues (with DLX reference) ---

    @Bean
    public Queue emailQueue() {
        return QueueBuilder.durable(NotificationQueues.EMAIL_QUEUE)
                .withArgument("x-dead-letter-exchange", NotificationQueues.NOTIFICATION_DLX)
                .withArgument("x-dead-letter-routing-key", NotificationQueues.EMAIL_ROUTING_KEY)
                .withArgument("x-message-ttl", 300_000)  // 5-minute message TTL
                .build();
    }

    @Bean
    public Queue smsQueue() {
        return QueueBuilder.durable(NotificationQueues.SMS_QUEUE).build();
    }

    @Bean
    public Queue pushQueue() {
        return QueueBuilder.durable(NotificationQueues.PUSH_QUEUE).build();
    }

    // --- Bindings ---

    @Bean
    public Binding emailBinding() {
        return BindingBuilder.bind(emailQueue())
                .to(notificationExchange())
                .with(NotificationQueues.EMAIL_ROUTING_KEY);
    }

    @Bean
    public Binding smsBinding() {
        return BindingBuilder.bind(smsQueue())
                .to(notificationExchange())
                .with(NotificationQueues.SMS_ROUTING_KEY);
    }

    @Bean
    public Binding pushBinding() {
        return BindingBuilder.bind(pushQueue())
                .to(notificationExchange())
                .with(NotificationQueues.PUSH_ROUTING_KEY);
    }

    // --- Message Converter (JSON) ---

    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(jsonMessageConverter());
        template.setMandatory(true);   // trigger ReturnCallback if message is unroutable
        return template;
    }
}
```

---

## Message Payload (Immutable Record)

```java
package com.joshsoftware.app.messaging.event;

import java.time.Instant;
import java.util.UUID;

public record NotificationEvent(
        UUID eventId,
        String recipientId,
        String templateCode,
        java.util.Map<String, String> templateParams,  // key-value for template placeholders
        Instant occurredAt
) {
    public static NotificationEvent of(String recipientId, String templateCode,
                                        java.util.Map<String, String> params) {
        return new NotificationEvent(UUID.randomUUID(), recipientId, templateCode, params, Instant.now());
    }
}
```

---

## Publisher

```java
package com.joshsoftware.app.messaging.producer;

import com.joshsoftware.app.messaging.NotificationQueues;
import com.joshsoftware.app.messaging.event.NotificationEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class NotificationMessagePublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishEmailNotification(NotificationEvent event) {
        log.info("[NotificationMessagePublisher#publishEmailNotification] Publishing - eventId={} recipient={}",
                event.eventId(), event.recipientId());
        rabbitTemplate.convertAndSend(
                NotificationQueues.NOTIFICATION_EXCHANGE,
                NotificationQueues.EMAIL_ROUTING_KEY,
                event
        );
        log.info("[NotificationMessagePublisher#publishEmailNotification] Published - eventId={}", event.eventId());
    }

    public void publishSmsNotification(NotificationEvent event) {
        log.info("[NotificationMessagePublisher#publishSmsNotification] Publishing - eventId={}", event.eventId());
        rabbitTemplate.convertAndSend(
                NotificationQueues.NOTIFICATION_EXCHANGE,
                NotificationQueues.SMS_ROUTING_KEY,
                event
        );
        log.info("[NotificationMessagePublisher#publishSmsNotification] Published - eventId={}", event.eventId());
    }
}
```

---

## Consumer (Listener)

```java
package com.joshsoftware.app.messaging.consumer;

import com.joshsoftware.app.messaging.NotificationQueues;
import com.joshsoftware.app.messaging.event.NotificationEvent;
import com.rabbitmq.client.Channel;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Slf4j
@Component
@RequiredArgsConstructor
public class NotificationMessageListener {

    private final EmailDispatchService emailDispatchService;

    @RabbitListener(
        queues = NotificationQueues.EMAIL_QUEUE,
        ackMode = "MANUAL",           // manual ack — commit only on successful processing
        concurrency = "2-5"           // min 2, max 5 consumer threads
    )
    public void onEmailNotification(NotificationEvent event,
                                     Channel channel,
                                     @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {
        log.info("[NotificationMessageListener#onEmailNotification] ENTRY - eventId={} recipient={}",
                event.eventId(), event.recipientId());
        try {
            emailDispatchService.dispatch(event);
            channel.basicAck(deliveryTag, false);  // ack single message
            log.info("[NotificationMessageListener#onEmailNotification] EXIT - eventId={}", event.eventId());
        } catch (Exception ex) {
            log.error("[NotificationMessageListener#onEmailNotification] FAILED - eventId={}",
                    event.eventId(), ex);
            // nack without requeue → routes to DLQ via x-dead-letter-exchange
            channel.basicNack(deliveryTag, false, false);
        }
    }
}
```

---

## application.yml — RabbitMQ Config

```yaml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST:localhost}
    port: ${RABBITMQ_PORT:5672}
    username: ${RABBITMQ_USERNAME:guest}
    password: ${RABBITMQ_PASSWORD:guest}
    virtual-host: ${RABBITMQ_VHOST:/}
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10          # max unacknowledged messages per consumer
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 500ms
          multiplier: 2.0
          max-interval: 10000ms
```

---

## Rules

- DLQ must be declared before the main queue — RabbitMQ validates the DLX reference at queue creation time
- Always use `ackMode = MANUAL` with `channel.basicAck` / `channel.basicNack` — never auto-ack in production
- `basicNack(deliveryTag, false, false)` — `requeue=false` routes to DLQ; `requeue=true` loops forever on a poison message
- `Jackson2JsonMessageConverter` must be on both publisher and consumer sides — mismatched converters cause deserialization failures
- Queue and exchange names must come from constants — never inline strings in `@RabbitListener`
- Monitor DLQ depth in production — messages accumulating there signal a processing bug