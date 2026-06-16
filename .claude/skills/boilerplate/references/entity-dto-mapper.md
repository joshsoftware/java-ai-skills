# Entity · DTO · Mapper

## Package Structure

```
com.joshsoftware.<app>/
├── entity/           ← JPA entities
├── enums/            ← Enum types (top-level — NOT inside entity/enums/)
├── dto/
│   ├── request/      ← inbound payloads (validation here)
│   ├── response/     ← outbound shapes (POJO classes with @Getter @Setter)
│   ├── ApiResponse.java   ← lives here, NOT in common/
│   └── ErrorResponse.java ← lives here, NOT in common/
└── mapper/           ← MapStruct interfaces
```

---

## Entity Template

Every entity must include these fields directly — no base class, no auditing annotations:

```java
package com.joshsoftware.app.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.time.LocalDateTime;
import java.util.UUID;

/**
 * author : <AuthorName>
 **/

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    @Column(updatable = false, nullable = false)
    private UUID id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, unique = true, length = 150)
    private String email;

    @Column(nullable = false)
    private String password;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private Role role;

    @Column(nullable = false)
    private boolean active = true;

    @Column(updatable = false)
    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;

    @Column(updatable = false, length = 100)
    private String createdBy;

    @Column(length = 100)
    private String updatedBy;
}
```

**Entity rules:**
- **No `BaseEntity` or `@MappedSuperclass`** — add `id`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy` directly to every entity
- **No Spring auditing annotations** (`@CreatedDate`, `@LastModifiedDate`, etc.) — set `createdAt`/`updatedAt` manually in the mapper `toEntity` expression using `LocalDateTime.now()`
- Use `LocalDateTime` for timestamps — not `Instant`
- `createdAt` and `updatedAt` have **no** `nullable = false` constraint
- Always add `@AllArgsConstructor` and `@NoArgsConstructor` to every entity
- One entity per table — no multi-table inheritance unless explicitly needed
- Always use `@Column(nullable = false)` for required fields — don't rely on DB defaults alone
- Enums stored as `STRING` — never `ORDINAL`
- Enums live in `com.joshsoftware.<app>.enums` package — **not** inside `entity/enums/`
- Bidirectional relationships: owner side uses `@JoinColumn`; inverse side uses `mappedBy`
- No business logic in entity classes
- Author comment (`/**
 * author : <name>
 **/`) at the top of every entity class

---

## Request DTO Template (POJO class)

```java
package com.joshsoftware.app.dto.request;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.Getter;
import lombok.Setter;

/**
 * Payload for creating or updating a User.
 */
@Getter
@Setter
public class CreateUserRequestDTO {

    @NotBlank(message = "Name is required")
    @Size(max = 100, message = "Name must not exceed 100 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;

    @NotBlank(message = "Role is required")
    private String role;
}
```

**Request DTO rules:**
- DTOs are **POJO classes** with `@Getter @Setter` — **not** Java records
- Naming: always append `DTO` suffix — e.g. `CreateProductRequestDTO`, `UpdateProductRequestDTO`
- All validation annotations (`@NotBlank`, `@Size`, `@Email`, etc.) go here — not on the entity
- Custom `message` on every constraint — never rely on default Hibernate messages
- Never expose entity IDs or audit fields in request DTOs

---

## Response DTO Template (POJO class)

```java
package com.joshsoftware.app.dto.response;

import lombok.Getter;
import lombok.Setter;
import java.time.LocalDateTime;

/**
 * Read-only projection of a User returned to the client.
 */
@Getter
@Setter
public class UserResponseDTO {
    private Long id;
    private String name;
    private String email;
    private String role;
    private boolean active;
    private LocalDateTime createdAt;
}
```

**Response DTO rules:**
- DTOs are **POJO classes** with `@Getter @Setter` — **not** Java records
- Naming: always append `DTO` suffix — e.g. `ProductResponseDTO`, `ProductSummaryResponseDTO`
- Use `LocalDateTime` for timestamp fields — not `Instant`
- Never include `password` or any sensitive field
- Shape should be driven by what the client needs — not a 1:1 mirror of the entity
- Use `@JsonProperty` if the JSON key must differ from the field name

---

## ApiResponse and ErrorResponse

These live in `dto/` package — **not** `common/`. They are POJO classes, not records.

```java
// com.joshsoftware.<app>.dto.ApiResponse
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
    // Only success() and error() — no created() method
}
```

```java
// com.joshsoftware.<app>.dto.ErrorResponse
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class ErrorResponse {
    private int status;
    private String message;
    private List<FieldError> errors;
    private LocalDateTime timestamp;

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    public static class FieldError {
        private String field;
        private String message;
    }

    public static ErrorResponse of(int status, String message) {
        return new ErrorResponse(status, message, null, LocalDateTime.now());
    }

    public static ErrorResponse withFieldErrors(int status, String message, List<FieldError> errors) {
        return new ErrorResponse(status, message, errors, LocalDateTime.now());
    }
}
```

---

## Constants — Interface (not final class)

Constants classes must be **interfaces** — fields in an interface are implicitly `public static final`, so no keyword is needed:

```java
package com.joshsoftware.<app>.constants;

public interface ProductConstants {
    String PRODUCT_CREATED  = "Product created successfully";
    String PRODUCT_UPDATED  = "Product updated successfully";
    String PRODUCT_DELETED  = "Product deleted successfully";
    String PRODUCT_FETCHED  = "Product fetched successfully";
    String PRODUCTS_FETCHED = "Products fetched successfully";
}
```

---

## MapStruct Mapper Template

```java
package com.joshsoftware.app.mapper;

import com.joshsoftware.app.dto.request.CreateUserRequestDTO;
import com.joshsoftware.app.dto.response.UserResponseDTO;
import com.joshsoftware.app.enums.Role;
import com.joshsoftware.app.entity.User;
import org.mapstruct.*;

import java.time.LocalDateTime;

@Mapper(
    componentModel = "spring",
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE,
    imports = {Role.class, LocalDateTime.class}
)
public interface UserMapper {

    /**
     * Maps a CreateUserRequestDTO to a new User entity.
     * Password encoding is handled in the service layer, not here.
     * createdAt and updatedAt are set via expression so they are populated on creation.
     */
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", expression = "java(LocalDateTime.now())")
    @Mapping(target = "updatedAt", expression = "java(LocalDateTime.now())")
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "updatedBy", ignore = true)
    @Mapping(target = "active", constant = "true")
    @Mapping(target = "role", expression = "java(Role.valueOf(request.getRole()))")
    User toEntity(CreateUserRequestDTO request);

    UserResponseDTO toResponse(User user);

    /**
     * Partially updates an existing entity from a request — null fields are skipped.
     * updatedAt is refreshed via expression on every update.
     */
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", expression = "java(LocalDateTime.now())")
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "updatedBy", ignore = true)
    void updateEntityFromRequest(CreateUserRequestDTO request, @MappingTarget User user);
}
```

**Mapper rules:**
- `componentModel = "spring"` — always; allows `@Autowired` / constructor injection
- `NullValuePropertyMappingStrategy.IGNORE` — safe for partial updates
- Always `@Mapping(target = "id", ignore = true)` on `toEntity`
- Set `createdAt` and `updatedAt` via `expression = "java(LocalDateTime.now())"` in `toEntity` — do **not** ignore them
- Set `updatedAt` via `expression = "java(LocalDateTime.now())"` in `updateEntityFromRequest` — do **not** ignore it
- Import `LocalDateTime.class` in the `@Mapper` `imports` attribute
- No business logic (e.g. password encoding) inside mappers — do it in the service before/after mapping
- Use `expression = "java(...)"` for type conversions like `String → Enum`
- Use getter methods (`request.getXxx()`) since DTOs are POJO classes, not records
