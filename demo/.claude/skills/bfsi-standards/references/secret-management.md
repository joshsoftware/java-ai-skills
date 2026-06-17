# Secret Management — BFSI Standards

## Core Rule
**Secrets never appear in source code or committed config files.**
All credentials, keys, and tokens must come from environment variables, HashiCorp Vault, or AWS Secrets Manager.

---

## application.yml — Correct Pattern

```yaml
# application.yml  (safe to commit — no real values)
app:
  jwt:
    secret: ${JWT_SECRET}                        # min 256-bit key
    expiration-ms: ${JWT_EXPIRATION_MS:86400000} # default 24h

  encryption:
    aes-key: ${AES_ENCRYPTION_KEY}               # 32-byte Base64
    rsa-public-key: ${RSA_PUBLIC_KEY}
    rsa-private-key: ${RSA_PRIVATE_KEY}

  payment:
    gateway-api-key: ${PAYMENT_GATEWAY_API_KEY}
    webhook-secret: ${PAYMENT_WEBHOOK_SECRET}

spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}

  data:
    redis:
      password: ${REDIS_PASSWORD:}

cloud:
  aws:
    credentials:
      access-key: ${AWS_ACCESS_KEY_ID}
      secret-key: ${AWS_SECRET_ACCESS_KEY}
```

---

## Config Properties Class (Type-Safe Binding)

```java
package com.joshsoftware.app.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Min;

@Validated
@ConfigurationProperties(prefix = "app.payment")
public record PaymentSecretProperties(
        @NotBlank String gatewayApiKey,
        @NotBlank String webhookSecret
) {}
```

Enable in main class:
```java
@SpringBootApplication
@EnableConfigurationProperties({
    PaymentSecretProperties.class,
    EncryptionProperties.class
})
public class Application { ... }
```

**Rules:**
- Use `@Validated` + `@NotBlank` — fail-fast on startup if a required secret is missing
- `record` prevents accidental mutation
- Never inject raw `@Value("${app.payment.gateway-api-key}")` into service classes — use the Properties record

---

## HashiCorp Vault Integration (Spring Cloud Vault)

### Gradle dependency
```groovy
implementation 'org.springframework.cloud:spring-cloud-starter-vault-config'
```

### bootstrap.yml (Vault config)
```yaml
spring:
  cloud:
    vault:
      host: ${VAULT_HOST:localhost}
      port: ${VAULT_PORT:8200}
      scheme: https
      authentication: TOKEN
      token: ${VAULT_TOKEN}
      kv:
        enabled: true
        backend: secret
        default-context: bfsi-app
```

Secrets stored at `secret/bfsi-app` in Vault automatically bind to `@ConfigurationProperties`.

---

## AWS Secrets Manager Integration

### Gradle dependency
```groovy
implementation 'io.awspring.cloud:spring-cloud-aws-secrets-manager-config'
```

### application.yml
```yaml
spring:
  config:
    import: "aws-secretsmanager:/bfsi-app/prod/credentials"
```

Secrets stored as JSON in AWS Secrets Manager are flattened and injected as Spring properties.

---

## Local Development — .env File

For local dev only, use a `.env` file (never committed):

```bash
# .env  — add to .gitignore immediately
JWT_SECRET=your-local-dev-secret-min-32-chars-here
AES_ENCRYPTION_KEY=base64-encoded-32-byte-key-here
DATABASE_URL=jdbc:postgresql://localhost:5432/bfsidev
DATABASE_USERNAME=devuser
DATABASE_PASSWORD=devpass
```

Load in Spring Boot with:
```groovy
// build.gradle — only for local dev profile
tasks.named('bootRun') {
    systemProperty 'spring.profiles.active', 'local'
    environment(file('.env').readLines()
        .findAll { it && !it.startsWith('#') }
        .collectEntries { it.split('=', 2) })
}
```

**Mandatory `.gitignore` entries:**
```
.env
.env.*
*-secret.yml
*-secret.yaml
application-prod.yml
application-staging.yml
*.pem
*.p12
*.jks
```

---

## Secret Rotation Support

```java
package com.joshsoftware.app.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.context.refresh.ContextRefresher;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

/**
 * Periodically refreshes Spring Cloud Config / Vault-sourced secrets.
 * Triggered by @RefreshScope beans to pick up rotated credentials without restart.
 */
@Slf4j
@Component
public class SecretRefreshScheduler {

    private final ContextRefresher contextRefresher;

    public SecretRefreshScheduler(ContextRefresher contextRefresher) {
        this.contextRefresher = contextRefresher;
    }

    @Scheduled(fixedRateString = "${app.secret.refresh-interval-ms:3600000}") // default 1h
    public void refreshSecrets() {
        log.info("[SecretRefreshScheduler] Refreshing secrets from Vault/Config Server");
        contextRefresher.refresh();
        log.info("[SecretRefreshScheduler] Secret refresh complete");
    }
}
```

---

## Security Rules (Always Enforced)

- JWT secret minimum **256 bits (32 chars)** — reject shorter values at startup
- AES key must be exactly **32 bytes (256 bits)** — enforce in constructor
- Database password rotation: support hot-reload via `@RefreshScope` on datasource bean
- Never log the value of any injected `@Value` secret property
- TLS 1.2+ mandatory for all external service calls — configure `RestTemplate`/`WebClient` with TLS context
- Vault token must have the **least-privilege policy** — read-only on the required path