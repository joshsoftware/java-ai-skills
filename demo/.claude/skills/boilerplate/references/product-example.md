# Product Management — Full Working Example

This file shows every layer wired together for a real `Product` domain.
Use this as the reference when generating code for any new domain — substitute entity name, fields, and business rules accordingly.

---

## Project Structure (Product Module)

```
com.joshsoftware.inventory/
├── config/
│   └── OpenApiConfig.java
├── controller/
│   └── ProductController.java
├── service/
│   ├── ProductService.java
│   └── impl/
│       └── ProductServiceImpl.java
├── repository/
│   └── ProductRepository.java
├── entity/
│   ├── Product.java
│   └── Category.java
├── enums/
│   └── ProductStatus.java
├── dto/
│   ├── request/
│   │   ├── CreateProductRequestDTO.java
│   │   └── UpdateProductRequestDTO.java
│   ├── response/
│   │   ├── ProductResponseDTO.java
│   │   └── ProductSummaryResponseDTO.java
│   ├── ApiResponse.java
│   └── ErrorResponse.java
├── mapper/
│   └── ProductMapper.java
├── exception/
│   ├── AppException.java
│   ├── ResourceNotFoundException.java
│   ├── DuplicateResourceException.java
│   ├── BusinessException.java
│   └── handler/
│       └── GlobalExceptionHandler.java
└── constants/
    └── ProductConstants.java
```

---

## 1. Entity — Product.java

```java
package com.joshsoftware.inventory.entity;

/**
 * author : <AuthorName>
 **/

import com.joshsoftware.inventory.enums.ProductStatus;
import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table(
    name = "products",
    indexes = {
        @Index(name = "idx_product_sku",    columnList = "sku",    unique = true),
        @Index(name = "idx_product_status", columnList = "status")
    }
)
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    @Column(updatable = false, nullable = false)
    private UUID id;

    @Column(nullable = false, length = 200)
    private String name;

    @Column(nullable = false, unique = true, length = 50)
    private String sku;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stockQuantity = 0;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private ProductStatus status = ProductStatus.ACTIVE;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;

    @Column(updatable = false)
    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;

    @Column(updatable = false, length = 100)
    private String createdBy;

    @Column(length = 100)
    private String updatedBy;
}
```

---

## 2. Enum — ProductStatus.java

```java
package com.joshsoftware.inventory.enums;

import lombok.Getter;

@Getter
public enum ProductStatus {
    ACTIVE("Active"),
    INACTIVE("Inactive"),
    DISCONTINUED("Discontinued"),
    OUT_OF_STOCK("Out of Stock");

    private final String value;

    ProductStatus(String value) {
        this.value = value;
    }
}
```

---

## 3. Request DTOs

```java
// CreateProductRequestDTO.java
package com.joshsoftware.inventory.dto.request;

import jakarta.validation.constraints.*;
import lombok.Getter;
import lombok.Setter;
import java.math.BigDecimal;
import java.util.UUID;

@Getter
@Setter
public class CreateProductRequestDTO {

    @NotBlank(message = "Product name is required")
    @Size(max = 200, message = "Product name must not exceed 200 characters")
    private String name;

    @NotBlank(message = "SKU is required")
    @Size(max = 50, message = "SKU must not exceed 50 characters")
    private String sku;

    private String description;

    @NotNull(message = "Price is required")
    @DecimalMin(value = "0.0", inclusive = false, message = "Price must be greater than 0")
    @Digits(integer = 10, fraction = 2, message = "Price must have at most 2 decimal places")
    private BigDecimal price;

    @NotNull(message = "Stock quantity is required")
    private Integer stockQuantity;

    @NotNull(message = "Category ID is required")
    private UUID categoryId;
}
```

```java
// UpdateProductRequestDTO.java
package com.joshsoftware.inventory.dto.request;

import jakarta.validation.constraints.*;
import lombok.Getter;
import lombok.Setter;
import java.math.BigDecimal;

@Getter
@Setter
public class UpdateProductRequestDTO {

    @Size(max = 200, message = "Product name must not exceed 200 characters")
    private String name;

    private String description;

    @DecimalMin(value = "0.0", inclusive = false, message = "Price must be greater than 0")
    @Digits(integer = 10, fraction = 2, message = "Price must have at most 2 decimal places")
    private BigDecimal price;

    private Integer stockQuantity;

    private String status;
}
```

---

## 4. Response DTOs

```java
// ProductResponseDTO.java
package com.joshsoftware.inventory.dto.response;

import lombok.Getter;
import lombok.Setter;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Getter
@Setter
public class ProductResponseDTO {
    private UUID id;
    private String name;
    private String sku;
    private String description;
    private BigDecimal price;
    private Integer stockQuantity;
    private String status;
    private String categoryName;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

```java
// ProductSummaryResponseDTO.java — lightweight projection for list endpoints
package com.joshsoftware.inventory.dto.response;

import lombok.Getter;
import lombok.Setter;
import java.math.BigDecimal;
import java.util.UUID;

@Getter
@Setter
public class ProductSummaryResponseDTO {
    private UUID id;
    private String name;
    private String sku;
    private BigDecimal price;
    private String status;
}
```

---

## 5. Mapper — ProductMapper.java

```java
package com.joshsoftware.inventory.mapper;

import com.joshsoftware.inventory.dto.request.CreateProductRequestDTO;
import com.joshsoftware.inventory.dto.request.UpdateProductRequestDTO;
import com.joshsoftware.inventory.dto.response.ProductResponseDTO;
import com.joshsoftware.inventory.dto.response.ProductSummaryResponseDTO;
import com.joshsoftware.inventory.entity.Product;
import com.joshsoftware.inventory.enums.ProductStatus;
import org.mapstruct.*;

import java.time.LocalDateTime;

@Mapper(
    componentModel = "spring",
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE,
    unmappedTargetPolicy = ReportingPolicy.ERROR,
    imports = {ProductStatus.class, LocalDateTime.class}
)
public interface ProductMapper {

    @Mapping(target = "id",         ignore = true)
    @Mapping(target = "createdAt",  expression = "java(LocalDateTime.now())")
    @Mapping(target = "updatedAt",  expression = "java(LocalDateTime.now())")
    @Mapping(target = "createdBy",  ignore = true)
    @Mapping(target = "updatedBy",  ignore = true)
    @Mapping(target = "category",   ignore = true)
    @Mapping(target = "status",     constant = "ACTIVE")
    Product toEntity(CreateProductRequestDTO request);

    @Mapping(target = "categoryName", source = "category.name")
    @Mapping(target = "status",       expression = "java(product.getStatus().name())")
    ProductResponseDTO toResponse(Product product);

    @Mapping(target = "status", expression = "java(product.getStatus().name())")
    ProductSummaryResponseDTO toSummaryResponse(Product product);

    @Mapping(target = "id",         ignore = true)
    @Mapping(target = "sku",        ignore = true)
    @Mapping(target = "createdAt",  ignore = true)
    @Mapping(target = "updatedAt",  expression = "java(LocalDateTime.now())")
    @Mapping(target = "createdBy",  ignore = true)
    @Mapping(target = "updatedBy",  ignore = true)
    @Mapping(target = "category",   ignore = true)
    @Mapping(target = "status",
             expression = "java(request.getStatus() != null ? ProductStatus.valueOf(request.getStatus()) : product.getStatus())")
    void updateEntityFromRequest(UpdateProductRequestDTO request, @MappingTarget Product product);
}
```

---

## 6. Constants — ProductConstants.java

```java
package com.joshsoftware.inventory.constants;

public interface ProductConstants {

    String PRODUCT_CREATED  = "Product created successfully";
    String PRODUCT_UPDATED  = "Product updated successfully";
    String PRODUCT_DELETED  = "Product deleted successfully";
    String PRODUCT_FETCHED  = "Product fetched successfully";
    String PRODUCTS_FETCHED = "Products fetched successfully";
}
```

> Constants class must be an **interface** — not a `final class`. Fields in an interface are implicitly `public static final` — do not add the keywords explicitly.

---

## 7. Shared Types — ApiResponse and ErrorResponse

These live in `dto/` package — **not** `common/`.

```java
// ApiResponse.java
package com.joshsoftware.inventory.dto;

import lombok.Builder;
import lombok.Getter;
import lombok.Setter;

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
    // No created() method — all create responses use ResponseEntity.ok() + ApiResponse.success()
}
```

```java
// ErrorResponse.java
package com.joshsoftware.inventory.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.time.LocalDateTime;
import java.util.List;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
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

## 8. ServiceImpl — ProductServiceImpl.java (with category resolution)

```java
package com.joshsoftware.inventory.service.impl;

/**
 * author : <AuthorName>
 **/

import com.joshsoftware.inventory.dto.request.CreateProductRequestDTO;
import com.joshsoftware.inventory.dto.request.UpdateProductRequestDTO;
import com.joshsoftware.inventory.dto.response.ProductResponseDTO;
import com.joshsoftware.inventory.entity.Category;
import com.joshsoftware.inventory.entity.Product;
import com.joshsoftware.inventory.exception.DuplicateResourceException;
import com.joshsoftware.inventory.exception.ResourceNotFoundException;
import com.joshsoftware.inventory.mapper.ProductMapper;
import com.joshsoftware.inventory.repository.CategoryRepository;
import com.joshsoftware.inventory.repository.ProductRepository;
import com.joshsoftware.inventory.service.ProductService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductServiceImpl implements ProductService {

    private final ProductRepository  productRepository;
    private final CategoryRepository categoryRepository;
    private final ProductMapper      productMapper;

    // Creates a new product and persists it.
    @Override
    @Transactional
    public ProductResponseDTO create(CreateProductRequestDTO request) {
        log.debug("[ProductServiceImpl#create] ENTRY - sku={}", request.getSku());

        if (productRepository.existsBySku(request.getSku())) {
            throw new DuplicateResourceException("Product", "SKU", request.getSku());
        }

        Category category = categoryRepository.findById(request.getCategoryId())
                .orElseThrow(() -> new ResourceNotFoundException("Category", "id", request.getCategoryId()));

        Product product = productMapper.toEntity(request);
        product.setCategory(category);

        Product createdProduct = productRepository.save(product);
        log.debug("[ProductServiceImpl#create] EXIT  - productId={}", createdProduct.getId());
        return productMapper.toResponse(createdProduct);
    }

    // Retrieves a product by its UUID.
    @Override
    public ProductResponseDTO findById(UUID id) {
        log.debug("[ProductServiceImpl#findById] ENTRY - productId={}", id);

        ProductResponseDTO response = productRepository.findByIdWithCategory(id)
                .map(productMapper::toResponse)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));

        log.debug("[ProductServiceImpl#findById] EXIT  - productId={}", id);
        return response;
    }

    // Returns all products.
    @Override
    public List<ProductResponseDTO> findAll() {
        log.debug("[ProductServiceImpl#findAll] ENTRY");

        List<ProductResponseDTO> products = productRepository.findAll()
                .stream()
                .map(productMapper::toResponse)
                .toList();

        log.debug("[ProductServiceImpl#findAll] EXIT  - count={}", products.size());
        return products;
    }

    // Updates an existing product by UUID.
    @Override
    @Transactional
    public ProductResponseDTO update(UUID id, UpdateProductRequestDTO request) {
        log.debug("[ProductServiceImpl#update] ENTRY - productId={}", id);

        Product existing = productRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));

        productMapper.updateEntityFromRequest(request, existing);
        Product updatedProduct = productRepository.save(existing);

        log.debug("[ProductServiceImpl#update] EXIT  - productId={}", id);
        return productMapper.toResponse(updatedProduct);
    }

    // Deletes a product by UUID.
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

## 9. Flyway Migration — V2__create_products_table.sql

```sql
CREATE TABLE products (
    id             UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    name           VARCHAR(200)  NOT NULL,
    sku            VARCHAR(50)   NOT NULL UNIQUE,
    description    TEXT,
    price          NUMERIC(12,2) NOT NULL,
    stock_quantity INT           NOT NULL DEFAULT 0,
    status         VARCHAR(20)   NOT NULL DEFAULT 'ACTIVE',
    category_id    UUID          NOT NULL REFERENCES categories(id),
    created_at     TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ   NOT NULL DEFAULT now(),
    created_by     VARCHAR(100),
    updated_by     VARCHAR(100)
);

CREATE INDEX idx_product_sku    ON products(sku);
CREATE INDEX idx_product_status ON products(status);
```

---

## How to Adapt This Example for a New Domain

| Product domain               | Your new domain example (e.g. Order)     |
|------------------------------|------------------------------------------|
| `Product`                    | `Order`                                  |
| `ProductService`             | `OrderService`                           |
| `ProductServiceImpl`         | `OrderServiceImpl`                       |
| `ProductRepository`          | `OrderRepository`                        |
| `CreateProductRequestDTO`    | `CreateOrderRequestDTO`                  |
| `UpdateProductRequestDTO`    | `UpdateOrderRequestDTO`                  |
| `ProductResponseDTO`         | `OrderResponseDTO`                       |
| `ProductSummaryResponseDTO`  | `OrderSummaryResponseDTO`                |
| `ProductMapper`              | `OrderMapper`                            |
| `ProductStatus` (enum)       | `OrderStatus` (enum in `enums/`)         |
| `ProductConstants`           | `OrderConstants` (interface, not class)  |
| `products` (table)           | `orders` (table)                         |
| `sku` (business key)         | `orderNumber` or `trackingId`            |
