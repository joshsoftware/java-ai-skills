# OpenAPI / Swagger Documentation

## Documentation Stack

Use Springdoc OpenAPI for Swagger documentation in Spring Boot 3.x applications.

| Concern | Choice |
|---------|--------|
| OpenAPI generator | springdoc-openapi |
| Swagger UI | springdoc-openapi-starter-webmvc-ui |
| Spec endpoint | `/v3/api-docs` |
| Swagger UI endpoint | `/swagger-ui/index.html` |
| Controller docs | `@Tag`, `@Operation`, `@ApiResponses` |
| DTO schema docs | `@Schema` |
| Security docs | `@SecurityRequirement` |

API documentation must describe the public contract clearly without exposing internal implementation details.

---

## Required Dependencies

### Gradle

``` groovy
def springdocVersion = "2.5.0"

implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:${springdocVersion}'
```

### Maven

```xml
<properties>
    <springdoc.version>2.5.0</springdoc.version>
</properties>

<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>${springdoc.version}</version>
</dependency>
```

---

## OpenAPI Configuration Class

Place OpenAPI configuration under `config/`.

```java
package com.joshsoftware.inventory.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import io.swagger.v3.oas.models.security.SecurityRequirement;
import io.swagger.v3.oas.models.security.SecurityScheme;
import io.swagger.v3.oas.models.Components;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    private static final String SECURITY_SCHEME_NAME = "bearerAuth";

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("Inventory Service API")
                        .description("REST API documentation for Inventory Service")
                        .version("v1")
                        .contact(new Contact()
                                .name("Josh Software")
                                .email("support@joshsoftware.com"))
                        .license(new License()
                                .name("Internal Use Only")))
                .addSecurityItem(new SecurityRequirement().addList(SECURITY_SCHEME_NAME))
                .components(new Components()
                        .addSecuritySchemes(SECURITY_SCHEME_NAME,
                                new SecurityScheme()
                                        .name(SECURITY_SCHEME_NAME)
                                        .type(SecurityScheme.Type.HTTP)
                                        .scheme("bearer")
                                        .bearerFormat("JWT")));
    }
}
```

---

## Application Properties

Use these properties when custom paths or UI behavior are required.
```
springdoc.api-docs.path=/v3/api-docs
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.swagger-ui.operations-sorter=method
springdoc.swagger-ui.tags-sorter=alpha
```

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
    tags-sorter: alpha
```

Keep default paths unless the project has a clear reason to customize them.

---

## Controller Documentation Pattern

Every REST controller must use `@Tag`. Every endpoint must use `@Operation` and `@ApiResponses`.

```java
package com.joshsoftware.inventory.controller;

import com.joshsoftware.inventory.common.ApiResponse;
import com.joshsoftware.inventory.dto.request.CreateProductRequest;
import com.joshsoftware.inventory.dto.response.ProductResponse;
import com.joshsoftware.inventory.service.ProductService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Tag(name = "Products", description = "Product management APIs")
@SecurityRequirement(name = "bearerAuth")
public class ProductController {

    private final ProductService productService;

    @PostMapping
    @Operation(
            summary = "Create product",
            description = "Creates a new product using the supplied product details."
    )
    public ResponseEntity<ApiResponse<ProductResponse>> create(
            @Valid @RequestBody CreateProductRequest request) {

        ProductResponse response = productService.create(request);
        return ResponseEntity.status(201)
                .body(ApiResponse.created("Product created successfully", response));
    }

    @GetMapping("/{id}")
    @Operation(
            summary = "Get product by id",
            description = "Returns product details for the given product id."
    )
    public ResponseEntity<ApiResponse<ProductResponse>> findById(
            @Parameter(description = "Product UUID", required = true)
            @PathVariable UUID id) {

        ProductResponse response = productService.findById(id);
        return ResponseEntity.ok(ApiResponse.success("Product fetched", response));
    }
}
```

---

## DTO Schema Documentation Pattern

Document request and response DTOs with `@Schema`.

```java
package com.joshsoftware.inventory.dto.request;

import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Positive;
import lombok.Getter;
import lombok.Setter;

import java.math.BigDecimal;

@Getter
@Setter
@Schema(description = "Request payload used to create a product")
public class CreateProductRequestDTO {

    @Schema(description = "Product name", example = "Keyboard")
    @NotBlank(message = "Product name is required")
    private String name;

    @Schema(description = "Product SKU", example = "KEY-001")
    @NotBlank(message = "Product SKU is required")
    private String sku;

    @Schema(description = "Product price", example = "1200.00")
    @Positive(message = "Product price must be greater than zero")
    private BigDecimal price;
}
```

Rules:

- Add `@Schema` to every request and response DTO.
- Provide clear descriptions and realistic examples.

---

## Response Documentation Rules

Every endpoint must document expected status codes.

| Scenario | Status |
|----------|--------|
| Successful create | `201` |
| Successful read/update/delete | `200` |
| Validation error | `400` |
| Unauthenticated request | `401` |
| Forbidden request | `403` |
| Resource not found | `404` |
| Duplicate/conflict | `409` |
| Unexpected server error | `500` |

---

## Security Configuration Rules

Swagger endpoints must be publicly accessible in secured applications:

```java
private static final String[] PUBLIC_ENDPOINTS = {
        "/api/v1/auth/**",
        "/actuator/health",
        "/v3/api-docs/**",
        "/swagger-ui/**",
        "/swagger-ui.html"
};
```

Rules:

- Permit Swagger and API docs paths in `SecurityFilterChain`.
- Keep business APIs protected unless explicitly public.
- Add `@SecurityRequirement(name = "bearerAuth")` to protected controllers or operations.
- Do not require JWT authentication just to open Swagger UI.

---

## Documentation Rules

- Use one `@Tag` per controller.
- Use short, action-based operation summaries.
- Use descriptions to explain behavior, not implementation details.
- Document path variables with `@Parameter`.
- Document query parameters, filters, pagination, and sorting.
- Keep response codes consistent with `GlobalExceptionHandler`.
- Keep DTO examples realistic and non-sensitive.
- Do not document database table names, internal classes, or private business rules.

---

## Anti-Patterns

| Anti-Pattern | Better Approach |
|--------------|-----------------|
| Missing `@Operation` on endpoints | Add summary and description for every endpoint |
| Documenting entities directly | Document request and response DTOs |
| Exposing internal fields in Swagger | Show only public API contract fields |
| Wrong response codes | Match controller and exception handler behavior |
| Generic descriptions like "API for product" | Explain the actual endpoint behavior |
| Hardcoding secrets or real tokens in examples | Use safe placeholder examples |
| Requiring auth for Swagger UI | Permit Swagger docs paths in security config |
| Adding Swagger annotations to service classes | Keep API documentation on controllers and DTOs |
 
---
