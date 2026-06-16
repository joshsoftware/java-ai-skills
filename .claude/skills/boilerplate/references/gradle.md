# Gradle Dependencies — build.gradle (Groovy DSL)

## Complete `build.gradle` Template (Java 21 + Spring Boot 3.4.x)

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.4.5'
    id 'io.spring.dependency-management' version '1.1.7'
}

group   = 'com.joshsoftware'
version = '1.0.0'
description = 'Inventory / Product Management Service'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

// ── Annotation Processors need to see Lombok-generated code ──────────────
configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

// ── Dependencies ──────────────────────────────────────────────────────────
dependencies {

    // ── Spring Boot Starters ─────────────────────────────────────────────
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-validation'

    // ── Database ─────────────────────────────────────────────────────────
    // PostgreSQL (comment out and add MySQL block below if needed)
    runtimeOnly 'org.postgresql:postgresql'

    /*
     * MySQL alternative — uncomment if using MySQL instead of PostgreSQL:
     * runtimeOnly 'com.mysql:mysql-connector-j'
     */

    // ── Lombok — version managed by Spring Boot BOM ───────────────────────
    compileOnly         'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // ── MapStruct ─────────────────────────────────────────────────────────
    implementation      'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'

    /*
     * CRITICAL — Lombok + MapStruct binding:
     * Without this, MapStruct cannot see Lombok-generated constructors/getters
     * and will produce compilation errors like "No property named 'xxx' exists".
     * Must come AFTER both lombok and mapstruct-processor in annotationProcessor order.
     */
    annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'

    // ── OpenAPI / Swagger UI ──────────────────────────────────────────────
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.14'

    // ── Testing ───────────────────────────────────────────────────────────
    testImplementation      'org.springframework.boot:spring-boot-starter-test'
    testImplementation      'org.springframework.security:spring-security-test'
    
    // H2 for integration tests — avoids needing a real DB to load context
    testRuntimeOnly         'com.h2database:h2'
}


tasks.named('test') {
    useJUnitPlatform()
}
```

---

## Annotation Processor Order Rule (CRITICAL)

The order of `annotationProcessor` declarations **matters**. Always declare in this exact order:

```groovy
// 1. Lombok first — generates getters, constructors, builders (version from BOM)
annotationProcessor 'org.projectlombok:lombok'

// 2. MapStruct second — uses Lombok-generated constructors/getters
annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'

// 3. Binding last — wires Lombok and MapStruct together
annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'
```

Reversing this order causes MapStruct to generate mappers that cannot see Lombok-generated
code, resulting in compile-time errors like:
- `No property named 'name' exists in source parameter`
- `Unknown property "id" in result type`

---

## Optional Dependency Snippets (add as needed)

```groovy
// ── Redis Cache ───────────────────────────────────────────────────────────
implementation 'org.springframework.boot:spring-boot-starter-data-redis'

// ── Kafka ─────────────────────────────────────────────────────────────────
implementation 'org.springframework.kafka:spring-kafka'

// ── JavaMail (Email) ──────────────────────────────────────────────────────
implementation 'org.springframework.boot:spring-boot-starter-mail'

// ── AWS S3 ────────────────────────────────────────────────────────────────
implementation 'software.amazon.awssdk:s3:2.25.0'

// ── Micrometer / Prometheus ───────────────────────────────────────────────
implementation 'io.micrometer:micrometer-registry-prometheus'

// ── H2 (in-memory — test / local dev only) ───────────────────────────────
runtimeOnly 'com.h2database:h2'

// ── Spring Cache Abstraction ──────────────────────────────────────────────
implementation 'org.springframework.boot:spring-boot-starter-cache'

// ── Async / Scheduling (already in spring-boot-starter, enable via @EnableAsync) ─
// No extra dep needed — just add @EnableAsync to a @Configuration class

// ── WebSocket ─────────────────────────────────────────────────────────────
implementation 'org.springframework.boot:spring-boot-starter-websocket'
```

---

## Dependency Quick-Reference Table

| Need                            | Gradle Dependency                                                             |
|---------------------------------|-------------------------------------------------------------------------------|
| Web (REST APIs)                 | `implementation 'org.springframework.boot:spring-boot-starter-web'`           |
| JPA + Hibernate                 | `implementation 'org.springframework.boot:spring-boot-starter-data-jpa'`      |
| Spring Security                 | `implementation 'org.springframework.boot:spring-boot-starter-security'`      |
| Bean Validation                 | `implementation 'org.springframework.boot:spring-boot-starter-validation'`    |
| Actuator (health/metrics)       | `implementation 'org.springframework.boot:spring-boot-starter-actuator'`      |
| PostgreSQL driver               | `runtimeOnly 'org.postgresql:postgresql'`                                     |
| MySQL driver                    | `runtimeOnly 'com.mysql:mysql-connector-j'`                                   |
| Flyway migrations               | `implementation 'org.flywaydb:flyway-core'`                                   |
| Liquibase migrations            | `implementation 'org.liquibase:liquibase-core'`                               |
| JWT (jjwt)                      | `implementation 'io.jsonwebtoken:jjwt-api:0.12.5'` + impl + jackson at runtime|
| Lombok                          | `compileOnly` + `annotationProcessor` (BOM-managed version)                  |
| MapStruct                       | `implementation` + `annotationProcessor` + binding                           |
| OpenAPI / Swagger               | `implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.8.14'`  |
| Redis                           | `implementation 'org.springframework.boot:spring-boot-starter-data-redis'`    |
| Kafka                           | `implementation 'org.springframework.kafka:spring-kafka'`                     |
| Email                           | `implementation 'org.springframework.boot:spring-boot-starter-mail'`          |
| AWS SDK S3                      | `implementation 'software.amazon.awssdk:s3:2.25.0'`                          |
| Prometheus metrics              | `implementation 'io.micrometer:micrometer-registry-prometheus'`               |
| H2 (test/local dev)             | `runtimeOnly 'com.h2database:h2'`                                             |

---

## Virtual Threads (Java 21 — Spring Boot 3.2+)

Enable Loom virtual threads for improved throughput under high concurrency.
No code changes needed — Spring Boot handles the thread pool swap automatically.

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

```properties
# application.properties
spring.threads.virtual.enabled=true
```

No extra Gradle dependency required — this is a JVM feature enabled via configuration.

---

## Application Configuration Format Reference

Both `application.yml` (YAML) and `application.properties` (flat key-value) are supported.
YAML is preferred for readability in nested configs. Pick one format and be consistent within a project.

### application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/${DB_NAME:mydb}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:secret}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    open-in-view: false
  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  port: 8080

app:
  jwt:
    secret: ${JWT_SECRET}
    expiration-ms: 86400000
```

### application.properties

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/${DB_NAME:mydb}
spring.datasource.username=${DB_USER:postgres}
spring.datasource.password=${DB_PASSWORD:secret}
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.open-in-view=false
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration

server.port=8080

app.jwt.secret=${JWT_SECRET}
app.jwt.expiration-ms=86400000
```

See `references/jpa-config.md` for the full JPA / HikariCP / Flyway configuration.

---

## Multi-Module Project Structure (reference)

If the project grows into multiple modules (e.g., `api`, `service`, `domain`):

```groovy
// settings.gradle
rootProject.name = 'inventory-platform'
include 'inventory-api', 'inventory-service', 'inventory-domain'
```

```groovy
// inventory-api/build.gradle
dependencies {
    implementation project(':inventory-service')
}
```

Keep the root `build.gradle` for shared plugin versions only.
Each submodule declares only the dependencies it directly uses.