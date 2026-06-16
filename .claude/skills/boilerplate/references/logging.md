# Logging — Entry / Exit Pattern

## Setup

Add `@Slf4j` (Lombok) to every Controller and ServiceImpl class.  
Never use `System.out.println`. Never use `java.util.logging`.

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductServiceImpl implements ProductService { ... }
```

---

## Entry / Exit Log Rules

| Layer      | Entry log level | Exit log level | What to log                              |
|------------|-----------------|----------------|------------------------------------------|
| Controller | `INFO`          | `INFO`         | HTTP method, path, key request param/id  |
| Service    | `DEBUG`         | `DEBUG`        | Method name, key business identifier     |
| Write ops  | `INFO`          | `INFO`         | Mutating operations always at INFO       |
| Errors     | `WARN` / `ERROR`| —              | WARN for expected; ERROR for unexpected  |

---

## Controller — Entry / Exit Pattern

```java
package com.joshsoftware.inventory.controller;

import com.joshsoftware.inventory.common.ApiResponse;
import com.joshsoftware.inventory.dto.request.CreateProductRequest;
import com.joshsoftware.inventory.dto.request.UpdateProductRequest;
import com.joshsoftware.inventory.dto.response.ProductResponse;
import com.joshsoftware.inventory.service.ProductService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@Slf4j
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @PostMapping
    public ResponseEntity<ApiResponse<ProductResponse>> create(
            @Valid @RequestBody CreateProductRequest request) {

        log.info("[ProductController#create] ENTRY - name={}", request.name());
        ProductResponse response = productService.create(request);
        log.info("[ProductController#create] EXIT  - productId={}", response.id());
        return ResponseEntity.status(201)
                .body(ApiResponse.created("Product created successfully", response));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<ProductResponse>> findById(@PathVariable UUID id) {
        log.info("[ProductController#findById] ENTRY - productId={}", id);
        ProductResponse response = productService.findById(id);
        log.info("[ProductController#findById] EXIT  - productId={}", id);
        return ResponseEntity.ok(ApiResponse.success("Product fetched", response));
    }

    @GetMapping
    public ResponseEntity<ApiResponse<Page<ProductResponse>>> findAll(Pageable pageable) {
        log.info("[ProductController#findAll] ENTRY - page={}, size={}", 
                pageable.getPageNumber(), pageable.getPageSize());
        Page<ProductResponse> page = productService.findAll(pageable);
        log.info("[ProductController#findAll] EXIT  - totalElements={}", page.getTotalElements());
        return ResponseEntity.ok(ApiResponse.success("Products fetched", page));
    }

    @PutMapping("/{id}")
    public ResponseEntity<ApiResponse<ProductResponse>> update(
            @PathVariable UUID id,
            @Valid @RequestBody UpdateProductRequest request) {

        log.info("[ProductController#update] ENTRY - productId={}", id);
        ProductResponse response = productService.update(id, request);
        log.info("[ProductController#update] EXIT  - productId={}", id);
        return ResponseEntity.ok(ApiResponse.success("Product updated", response));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<ApiResponse<Void>> delete(@PathVariable UUID id) {
        log.info("[ProductController#delete] ENTRY - productId={}", id);
        productService.delete(id);
        log.info("[ProductController#delete] EXIT  - productId={}", id);
        return ResponseEntity.ok(ApiResponse.success("Product deleted", null));
    }
}
```

---

## ServiceImpl — Entry / Exit Pattern

```java
package com.joshsoftware.inventory.service.impl;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductServiceImpl implements ProductService {

    private final ProductRepository productRepository;
    private final ProductMapper productMapper;

    @Override
    @Transactional
    public ProductResponse create(CreateProductRequest request) {
        log.debug("[ProductServiceImpl#create] ENTRY - name={}", request.name());

        if (productRepository.existsBySku(request.sku())) {
            throw new DuplicateResourceException("Product", "SKU", request.sku());
        }

        Product product = productMapper.toEntity(request);
        Product saved = productRepository.save(product);

        log.debug("[ProductServiceImpl#create] EXIT  - productId={}", saved.getId());
        return productMapper.toResponse(saved);
    }

    @Override
    @Transactional(readOnly = true)
    public ProductResponse findById(UUID id) {
        log.debug("[ProductServiceImpl#findById] ENTRY - productId={}", id);

        ProductResponse response = productRepository.findById(id)
                .map(productMapper::toResponse)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));

        log.debug("[ProductServiceImpl#findById] EXIT  - productId={}", id);
        return response;
    }

    @Override
    @Transactional(readOnly = true)
    public Page<ProductResponse> findAll(Pageable pageable) {
        log.debug("[ProductServiceImpl#findAll] ENTRY - page={}", pageable.getPageNumber());

        Page<ProductResponse> page = productRepository.findAll(pageable)
                .map(productMapper::toResponse);

        log.debug("[ProductServiceImpl#findAll] EXIT  - totalElements={}", page.getTotalElements());
        return page;
    }

    @Override
    @Transactional
    public ProductResponse update(UUID id, UpdateProductRequest request) {
        log.debug("[ProductServiceImpl#update] ENTRY - productId={}", id);

        Product existing = productRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        productMapper.updateEntityFromRequest(request, existing);
        Product updated = productRepository.save(existing);

        log.debug("[ProductServiceImpl#update] EXIT  - productId={}", id);
        return productMapper.toResponse(updated);
    }

    @Override
    @Transactional
    public void delete(UUID id) {
        log.debug("[ProductServiceImpl#delete] ENTRY - productId={}", id);

        if (!productRepository.existsById(id)) {
            throw new ResourceNotFoundException("Product", "id", id);
        }
        productRepository.deleteById(id);

        log.debug("[ProductServiceImpl#delete] EXIT  - productId={}", id);
    }
}
```

---

## Log Format Rules

| Rule                                  | Good                                                     | Bad                                  |
|---------------------------------------|----------------------------------------------------------|--------------------------------------|
| Prefix with class + method            | `[ProductServiceImpl#create] ENTRY`                      | `"Creating product..."`              |
| Use `{}` placeholders, not `+`        | `log.info("id={}", id)`                                  | `log.info("id=" + id)`               |
| Log the **key identifier**            | `productId={}`, `orderId={}`, `email={}`                 | logging entire request object        |
| Don't log passwords / tokens          | log `userId` not `password`                              | `log.info("pwd={}", request.password())` |
| WARN on expected business errors      | `log.warn("Product not found, id={}", id)`               | `log.error(...)` for 404s            |
| ERROR only for truly unexpected       | `log.error("Unexpected error processing product", ex)`   | WARN for NPE                         |

---

## Logback Configuration (`src/main/resources/logback-spring.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <springProfile name="local,dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
        <logger name="com.joshsoftware" level="DEBUG"/>
        <logger name="org.hibernate.SQL" level="DEBUG"/>
        <logger name="org.hibernate.orm.jdbc.bind" level="TRACE"/>
    </springProfile>

    <springProfile name="prod">
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/app.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>logs/app.%d{yyyy-MM-dd}.log</fileNamePattern>
                <maxHistory>30</maxHistory>
            </rollingPolicy>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="FILE"/>
        </root>
        <logger name="com.joshsoftware" level="INFO"/>
    </springProfile>

</configuration>
```
