# Maven Dependencies — pom.xml

## Complete `pom.xml` Template (Java 21 + Spring Boot 3.3.x)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- ── Parent ─────────────────────────────────────────────────── -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.0</version>
        <relativePath/>
    </parent>

    <!-- ── Project Coordinates ────────────────────────────────────── -->
    <groupId>com.joshsoftware</groupId>
    <artifactId>inventory-service</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    <name>inventory-service</name>
    <description>Inventory / Product Management Service</description>

    <!-- ── Properties ─────────────────────────────────────────────── -->
    <properties>
        <java.version>21</java.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
        <lombok.version>1.18.32</lombok.version>
        <lombok-mapstruct-binding.version>0.2.0</lombok-mapstruct-binding.version>
        <jjwt.version>0.12.5</jjwt.version>
        <springdoc.version>2.5.0</springdoc.version>
    </properties>

    <!-- ── Dependencies ───────────────────────────────────────────── -->
    <dependencies>

        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Database — PostgreSQL (comment out, add MySQL below if needed) -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!--
        MySQL (uncomment if using MySQL instead of PostgreSQL):
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>
        -->

        <!-- DB Migration — Flyway (preferred) -->
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <!-- Required for Flyway 10+ with PostgreSQL -->
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-database-postgresql</artifactId>
        </dependency>

        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>${jjwt.version}</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <optional>true</optional>
        </dependency>

        <!-- MapStruct -->
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>${mapstruct.version}</version>
        </dependency>

        <!-- OpenAPI / Swagger UI -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>${springdoc.version}</version>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <!-- ── Build ───────────────────────────────────────────────────── -->
    <build>
        <plugins>

            <!-- Spring Boot Maven Plugin -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <!-- Exclude Lombok from the final JAR -->
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>

            <!-- Maven Compiler Plugin — CRITICAL for Lombok + MapStruct ordering -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <release>${java.version}</release>
                    <annotationProcessorPaths>
                        <!--
                            ORDER MATTERS:
                            1. Lombok must run FIRST so MapStruct can see generated getters/constructors
                            2. MapStruct processor second
                            3. Lombok-MapStruct binding last to wire them
                        -->
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>${lombok.version}</version>
                        </path>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>${mapstruct.version}</version>
                        </path>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok-mapstruct-binding</artifactId>
                            <version>${lombok-mapstruct-binding.version}</version>
                        </path>
                    </annotationProcessorPaths>
                    <compilerArgs>
                        <!-- MapStruct: use constructor injection (matches @RequiredArgsConstructor) -->
                        <arg>-Amapstruct.defaultComponentModel=spring</arg>
                        <arg>-Amapstruct.defaultInjectionStrategy=constructor</arg>
                    </compilerArgs>
                </configuration>
            </plugin>

        </plugins>
    </build>

</project>
```

---

## Optional Dependency Snippets (add as needed)

```xml
<!-- Redis Cache -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- Kafka -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>

<!-- JavaMail (Email) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>

<!-- AWS S3 -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
    <version>2.25.0</version>
</dependency>

<!-- Micrometer / Prometheus -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

<!-- Liquibase (alternative to Flyway) -->
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>

<!-- H2 (in-memory — test / local dev only) -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

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
  threads:
    virtual:
      enabled: true

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
spring.threads.virtual.enabled=true

server.port=8080

app.jwt.secret=${JWT_SECRET}
app.jwt.expiration-ms=86400000
```

See `references/jpa-config.md` for the full JPA / HikariCP / Flyway configuration.

---

## Maven vs Gradle Quick Comparison

| Feature                       | Maven (`pom.xml`)                          | Gradle (`build.gradle.kts`)                    |
|-------------------------------|---------------------------------------------|------------------------------------------------|
| Annotation processor order    | `annotationProcessorPaths` in compiler plugin | `annotationProcessor(...)` order in deps      |
| Spring Boot plugin            | `spring-boot-maven-plugin`                 | `id("org.springframework.boot")`               |
| Build speed                   | Slower (no incremental by default)          | Faster (incremental + build cache)             |
| Preferred at Josh Software    | Both supported; **Gradle preferred**        | —                                              |
