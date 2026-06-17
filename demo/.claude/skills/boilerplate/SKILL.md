---
name: springboot-boilerplate
description: >
  Generates production-ready Spring Boot boilerplate code following Josh Software standards.
  Use this skill whenever a user asks to:
  - Initialize, scaffold, or bootstrap a new Spring Boot service or module
  - Create or generate any layer: Controller, Service, ServiceImpl, Repository, Entity,
    DTO (Request/Response), Mapper, Exception, Config, or Util class
  - Add a new domain/feature/entity to an existing project (e.g. "add Product module", "create Order API")
  - Detect which package/directory a new class belongs to and enforce naming conventions
  - Set up JPA, Hibernate auditing, datasource config, or DB migrations
  - Write, name, or structure Flyway migration scripts (versioned or repeatable)
  - Design DB schema: UUID PKs, TIMESTAMPTZ columns, named constraints, indexes
  - Add seed data via repeatable migrations
  - Add entry/exit logging to controller or service methods
  - Set up JWT security, OncePerRequestFilter, SecurityFilterChain, or auth plumbing
  - Set up OpenAPI / Swagger documentation using springdoc-openapi
  - Add global exception handling, custom error responses, or @ControllerAdvice
  - Set up cloud storage (S3, GCS) with provider abstraction and pre-signed URLs
  - Add file upload, download, or delete endpoints with validation
  - Wire up Maven (pom.xml) or Gradle (build.gradle.kts) dependencies
  - Refactor existing code for layer violations, anti-patterns, or naming convention breaches
  - Review PRs for structure violations or missing standards
  Always use this skill — even for partial requests like "just give me the service layer",
  "add entry-exit logs", "create the Product entity", or "which package does this class go in?".
---

# Spring Boot Boilerplate Skill — Josh Software Standards

Generates clean, idiomatic, production-grade Spring Boot code.  
Working domain used throughout all examples: **Product Management System**.

---

## Stack at a Glance

| Concern          | Choice                                                          |
|------------------|-----------------------------------------------------------------|
| Language         | Java 21 (records, sealed classes, text blocks where appropriate)|
| Build            | **Gradle** (Kotlin DSL) **or Maven** — see references below    |
| Framework        | Spring Boot 3.4.x                                               |
| Persistence      | Spring Data JPA + Hibernate 6                                   |
| Mapping          | MapStruct 1.5+                                                  |
| Boilerplate      | Lombok                                                          |
| Validation       | Hibernate Validator (Jakarta `@Valid`)                          |
| Security         | Spring Security 6 — custom `OncePerRequestFilter` JWT           |
| Response shape   | `ApiResponse<T> { status, message, data }`                      |
| Logging          | SLF4J + Logback — entry/exit on every controller & service method |
| DB Migration     | Flyway (preferred) or Liquibase                                 |

---

## How to Use This Skill

### Step 0 — Ask These Questions Before Generating Anything

When scaffolding a **new project or module**, always ask these questions first and wait for answers before writing any code:

1. **Build tool** — "Do you want **Gradle** (Groovy DSL) or **Maven**?"
   - Use `references/gradle.md` for Gradle, `references/maven.md` for Maven.

2. **Config file format** — "Do you want **application.yml** or **application.properties**?"
   - Generate the chosen format; do not produce both.

3. **JWT security** — "Do you want to set up **JWT authentication** now?"
   - If **yes**: read `references/security.md` and generate the full JWT setup.
   - If **no**: skip entirely. See the **Opt-In Features** table for the full rule.

4. **Pagination** — "Do you want **pagination** on list APIs?"
   - If **yes**: add `Pageable` parameter and return `Page<T>`.
   - If **no**: return `List<T>` with no pagination parameters.

5. **Author** — "Who is the **author**?" (ask once at the start of full project generation)
   - Use the answer for the `/**
 * author : <name>
 **/` comment at the top of every generated class.

6. **Cloud storage** — "Do you want to set up **cloud file storage** (upload / download / pre-signed URLs) now?"
   - If **yes**: ask the follow-up: "**AWS S3** or **Google Cloud Storage (GCS)**?" then read `references/cloud-storage.md`.
     - **AWS S3**: generate `S3Config`, `S3StorageServiceImpl`, AWS SDK v2 deps only. Skip all GCS classes.
     - **GCS**: generate `GcsConfig`, `GcsStorageServiceImpl`, GCS BOM dep only. Skip all S3 classes.
     - Both paths: also generate `StorageProperties`, `StorageService` interface, `FileController`, both response DTOs, `FileValidationUtil`, all storage exceptions, and the storage config block.
   - If **no**: skip entirely. See the **Opt-In Features** table for the full rule.

> Only proceed to code generation once all six answers are confirmed.

---

### Step 1 onwards — Code Generation

1. Identify which **layers** the user needs.
2. Read **every matching reference file** before generating code.
3. Generate **complete, compilable classes** — never pseudocode or stubs.
4. Include the relevant **build dependency snippet** (Gradle or Maven) when a new library is introduced.
   - **Before adding any dependency**, read the existing `build.gradle` or `pom.xml` to check whether it is already present.
   - If the dependency already exists, do **not** add it again — not even with a different version or scope.
   - When multiple reference files are read in one session (e.g. `gradle.md` + `security.md`), deduplicate across all of them before writing to the build file.
5. Enforce **naming conventions** and **package placement** rules (Section below).
6. Follow **cross-cutting rules** at all times.

---

## Reference Files — Read Before Generating

Reference files are split into two tiers. **Always-included** files are read whenever the matching layer is generated. **Opt-in** files must never be read or acted on until the user explicitly confirms they want that feature (see confirmation rules below).

### Always-Included (read automatically when the matching layer is needed)

| User asks about…                                      | Read this file                              |
|-------------------------------------------------------|---------------------------------------------|
| Controller / REST endpoints                           | `references/api-layer.md`                  |
| Service interface + ServiceImpl                       | `references/api-layer.md`                  |
| Repository (JPA queries)                              | `references/api-layer.md`                  |
| Entity, DTO (Request/Response), Mapper                | `references/entity-dto-mapper.md`          |
| JPA config, datasource, auditing, relationships       | `references/jpa-config.md`                 |
| Entry/exit logging pattern                            | `references/logging.md`                    |
| Global exception handling, custom errors              | `references/exception-handling.md`         |
| Gradle (build.gradle) dependencies                    | `references/gradle.md`                     |
| Maven (pom.xml) dependencies                          | `references/maven.md`                      |
| Full working example (Product Management)             | `references/product-example.md`            |

### Opt-In Features — Ask Before Reading or Generating

These features involve custom infrastructure, external integrations, or non-trivial setup. **Never read the reference file or generate any code for these without explicit user confirmation.** Ask the confirmation question, wait for the answer, then proceed only if the user says yes.

| Feature                          | Confirmation question to ask                                              | Reference file                         |
|----------------------------------|---------------------------------------------------------------------------|----------------------------------------|
| JWT Security                     | "Do you want to set up **JWT authentication**?"                           | `references/security.md`              |
| Cloud Storage (S3 / GCS)         | "Do you want to set up **cloud file storage** (upload / download / pre-signed URLs)?" — if yes, follow up: "**AWS S3** or **GCS**?" | `references/cloud-storage.md` |
| DB Migrations (Flyway)           | "Do you want to set up **Flyway DB migrations**?"                         | `references/db-migration.md`          |
| Encryption (AES / RSA)           | "Do you want to add **AES / RSA field-level encryption**?"                | `references/encryption.md`            |
| OpenAPI / Swagger Documentation  | "Do you want to generate **OpenAPI / Swagger documentation**?"            | `references/swagger-documentation.md` |
| Unit & Integration Tests         | "Do you want me to generate **unit and integration tests**?"              | `references/unit-testing.md`          |

**Rules for opt-in features:**
- Ask the confirmation question **before** reading the reference file or writing any code for that feature.
- If the user answers **no** or does not mention the feature: skip the reference file entirely and do not generate any related classes, config, or dependencies.
- If the user answers **yes**: read the reference file, then ask any follow-up questions required by that feature (e.g. S3 vs GCS, base path for FileController) before generating code.
- Never bundle opt-in feature questions together — ask each one separately and wait for the answer.

---

## Mandatory Architecture

```
Controller → Service (interface) → ServiceImpl → Repository → Database
```

- No layer may skip to a non-adjacent layer
- No circular dependencies between services
- Dependencies flow **downward only**

---

## Standard Project Structure

```
com.joshsoftware.<app>/
├── config/                     ← Spring config beans (Security, JPA, OpenAPI, etc.)
├── controller/                 ← REST controllers only
├── service/                    ← Service interfaces
│   └── impl/                   ← Service implementations
├── repository/                 ← JPA Repository interfaces
├── entity/                     ← JPA entity classes
├── enums/                      ← Enum types (top-level package, NOT inside entity/)
├── dto/
│   ├── request/                ← Inbound payloads (validation annotations here)
│   ├── response/               ← Outbound shapes (POJO classes with @Getter @Setter)
│   ├── ApiResponse.java        ← Generic response wrapper (lives in dto/, not common/)
│   └── ErrorResponse.java      ← Error response wrapper (lives in dto/, not common/)
├── mapper/                     ← MapStruct mapper interfaces
├── exception/                  ← Custom exception classes
│   └── handler/                ← GlobalExceptionHandler (@RestControllerAdvice)
├── util/                       ← Stateless helper/utility classes
├── constants/                  ← String/int constants (interfaces — not final classes)
└── security/                   ← JWT filter, UserDetailsService impl, SecurityConfig (only if JWT enabled)
```

> **Do NOT generate** `src/test/resources/` or any `application.yml` inside the test module.

---

## Class Placement — Auto-Detection Rules

When a new class is added, determine its package by these rules **in order**:

| Class characteristics                                          | Package                        |
|----------------------------------------------------------------|--------------------------------|
| Annotated `@RestController` or `@Controller`                   | `controller/`                  |
| Interface with only method signatures (domain contract)        | `service/`                     |
| `implements XxxService`, annotated `@Service`                  | `service/impl/`                |
| Extends `JpaRepository` / `CrudRepository`                     | `repository/`                  |
| Annotated `@Entity`                                            | `entity/`                      |
| Name ends with `Request`, used as `@RequestBody`               | `dto/request/`                 |
| Name ends with `Response`, returned from service               | `dto/response/`                |
| Annotated `@Mapper` (MapStruct)                                | `mapper/`                      |
| Extends `AppException` / `RuntimeException` (domain errors)    | `exception/`                   |
| Annotated `@RestControllerAdvice` / `@ControllerAdvice`        | `exception/handler/`           |
| Annotated `@Configuration` or `@EnableXxx`                     | `config/`                      |
| Stateless, no Spring annotations, pure helper methods          | `util/`                        |
| Only `static final` constants                                  | `constants/`                   |
| Implements `UserDetailsService` or extends `OncePerRequestFilter` | `security/`               |
| Shared wrapper types (`ApiResponse`, `ErrorResponse`)          | `dto/`                         |

---

## Naming Conventions (Strictly Enforced)

| Artifact            | Convention                                      | Example                          |
|---------------------|-------------------------------------------------|----------------------------------|
| Class               | `PascalCase`                                    | `ProductController`              |
| Package             | `com.joshsoftware.<app>.<layer>`                | `com.joshsoftware.inventory.service` |
| Service interface   | `<Domain>Service`                               | `ProductService`                 |
| Service impl        | `<Domain>ServiceImpl`                           | `ProductServiceImpl`             |
| Repository          | `<Domain>Repository`                            | `ProductRepository`              |
| Entity              | singular noun, `PascalCase`                     | `Product`, `OrderItem`           |
| DB table            | `snake_case`, plural                            | `products`, `order_items`        |
| Request DTO         | `<Action><Domain>RequestDTO`                    | `CreateProductRequestDTO`, `UpdateProductRequestDTO` |
| Response DTO        | `<Domain>ResponseDTO` or `<Domain>SummaryResponseDTO` | `ProductResponseDTO`, `ProductSummaryResponseDTO` |
| Mapper              | `<Domain>Mapper`                                | `ProductMapper`                  |
| Exception           | `<Reason>Exception`                             | `ResourceNotFoundException`      |
| Constants class     | `<Domain>Constants` or `AppConstants`           | `ProductConstants`               |
| Util class          | `<Concern>Util` or `<Concern>Helper`            | `DateUtil`, `PriceCalculator`    |
| Controller method   | `camelCase`, verb-first                         | `createProduct`, `findById`      |
| Variable            | `camelCase`                                     | `productId`, `totalPrice`        |
| Constant field      | `UPPER_SNAKE_CASE`                              | `MAX_RETRY_COUNT`                |

---

## Cross-Cutting Rules (Always Apply)

### Dependency Injection
- **Always** constructor injection via `@RequiredArgsConstructor`
- **Never** `@Autowired` on fields
- **Never** `@Autowired` on setter methods

### Lombok Usage
| Annotation                  | When to use                                      |
|-----------------------------|--------------------------------------------------|
| `@Getter @Setter`           | Mutable entities only                            |
| `@Builder`                  | DTOs, config objects                             |
| `@RequiredArgsConstructor`  | All Spring-managed beans                         |
| `@Slf4j`                    | Any class needing a logger                       |
| `@Data`                     | **Avoid** — too permissive; use targeted annotations |
| `@ToString(exclude=...)`    | Entities with lazy relationships                 |

### Response Wrapper
Every controller method returns `ResponseEntity<ApiResponse<T>>`.

```java
// com.joshsoftware.<app>.dto.ApiResponse
// Lives in dto/ package — NOT common/
@Getter
@Setter
@Builder
public class ApiResponse<T> {
    private int status;
    private String message;
    private T data;

    public static <T> ApiResponse<T> success(String message, T data) {
        return ApiResponse.<T>builder().status(200).message(message).data(data).build();
    }

    public static <T> ApiResponse<T> error(int status, String message) {
        return ApiResponse.<T>builder().status(status).message(message).data(null).build();
    }
    // No created() method — create endpoints use ResponseEntity.ok() with ApiResponse.success()
}
```

### Transaction Rules
- `@Transactional` on write methods (create, update, delete) and complex operations only
- **No** `@Transactional(readOnly = true)` on simple read methods like `findById`, `findAll`
- Transactions belong on **ServiceImpl**, never on Controller or Repository

### Validation
- All `@NotBlank`, `@Size`, `@Email`, `@Min`, `@Max` annotations go on **Request DTOs only**
- Use `@Valid` on every `@RequestBody` in the controller
- Custom `message` on every validation annotation — no default Hibernate messages

### Error Handling
- Always throw **custom exceptions** (extend `AppException`) — never raw `RuntimeException`
- HTTP status codes are set **only** in `GlobalExceptionHandler`, not in services
- Log `WARN` for expected business errors; `ERROR` for unexpected `Exception`

---

## Anti-Patterns (Strictly Forbidden)

| Anti-Pattern                                    | Why Forbidden                                  |
|-------------------------------------------------|------------------------------------------------|
| Business logic inside controllers               | Violates SRP, untestable                       |
| Returning `@Entity` objects directly from API   | Leaks DB schema, circular serialisation risk   |
| Direct `@Repository` injection into controller  | Skips business layer                           |
| `@Autowired` field injection                    | Hides dependencies, hard to test               |
| `@Data` on entities                             | Generates `equals/hashCode` on mutable state   |
| Catching and swallowing exceptions silently     | Masks bugs                                     |
| Native SQL when JPQL suffices                   | Couples to DB dialect                          |
| `Optional.get()` without `isPresent()` check    | NPE risk                                       |
| Storing enum as `ORDINAL`                       | Breaks on enum reorder                         |
| Hardcoded strings / magic numbers in logic      | Use constants                                  |
| `System.out.println` for logging                | Use SLF4J                                      |
| Adding a dependency that already exists in the build file | Creates duplicate entries, can cause version conflicts or build warnings |

---

## Java Version Notes

| Java Version | Features used in this skill                                          |
|--------------|----------------------------------------------------------------------|
| Java 17+     | Records, sealed classes, pattern matching for `instanceof`, text blocks |
| Java 21      | Virtual threads (`spring.threads.virtual.enabled=true`), sequenced collections |
| Spring Boot  | 3.x requires **Java 17 minimum**; 3.4.x targets Java 21             |

Set the toolchain in Gradle or `<java.version>` in Maven accordingly (see build references).

---
