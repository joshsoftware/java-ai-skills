# API Layer — Controller · Service · Repository

## Package Structure

```
com.joshsoftware.<app>/
├── controller/          ← REST controllers
├── service/             ← Service interfaces
│   └── impl/            ← Service implementations
└── repository/          ← JPA Repository interfaces
```

---

## Controller Template — ProductController (Full CRUD, no pagination by default)

```java
package com.joshsoftware.inventory.controller;

/**
 * author : <AuthorName>
 **/

import com.joshsoftware.inventory.dto.ApiResponse;
import com.joshsoftware.inventory.dto.request.CreateProductRequestDTO;
import com.joshsoftware.inventory.dto.request.UpdateProductRequestDTO;
import com.joshsoftware.inventory.dto.response.ProductResponseDTO;
import com.joshsoftware.inventory.service.ProductService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.UUID;

@Slf4j
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @PostMapping
    public ResponseEntity<ApiResponse<ProductResponseDTO>> create(
            @Valid @RequestBody CreateProductRequestDTO request) {

        log.info("[ProductController#create] ENTRY - name={}, sku={}", request.getName(), request.getSku());
        ProductResponseDTO response = productService.create(request);
        log.info("[ProductController#create] EXIT  - productId={}", response.getId());
        return ResponseEntity.ok(ApiResponse.success("Product created successfully", response));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<ProductResponseDTO>> findById(@PathVariable UUID id) {
        log.info("[ProductController#findById] ENTRY - productId={}", id);
        ProductResponseDTO response = productService.findById(id);
        log.info("[ProductController#findById] EXIT  - productId={}", id);
        return ResponseEntity.ok(ApiResponse.success("Product fetched successfully", response));
    }

    @GetMapping
    public ResponseEntity<ApiResponse<List<ProductResponseDTO>>> findAll() {
        log.info("[ProductController#findAll] ENTRY");
        List<ProductResponseDTO> products = productService.findAll();
        log.info("[ProductController#findAll] EXIT  - count={}", products.size());
        return ResponseEntity.ok(ApiResponse.success("Products fetched successfully", products));
    }

    @PutMapping("/{id}")
    public ResponseEntity<ApiResponse<ProductResponseDTO>> update(
            @PathVariable UUID id,
            @Valid @RequestBody UpdateProductRequestDTO request) {

        log.info("[ProductController#update] ENTRY - productId={}", id);
        ProductResponseDTO response = productService.update(id, request);
        log.info("[ProductController#update] EXIT  - productId={}", id);
        return ResponseEntity.ok(ApiResponse.success("Product updated successfully", response));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<ApiResponse<Void>> delete(@PathVariable UUID id) {
        log.info("[ProductController#delete] ENTRY - productId={}", id);
        productService.delete(id);
        log.info("[ProductController#delete] EXIT  - productId={}", id);
        return ResponseEntity.ok(ApiResponse.success("Product deleted successfully", null));
    }
}
```

**Controller rules:**
- One controller per domain aggregate
- No `@Validated` on the controller class — not needed
- No `@Min`/`@Max` on `@RequestParam` — validate in service or skip
- No business logic — delegate to service only
- Use `@Valid` on every `@RequestBody`
- Versioned base paths: `/api/v1/...`
- Always log ENTRY with key identifier, EXIT with result identifier
- Return `ResponseEntity<ApiResponse<T>>` always
- Create APIs use `ResponseEntity.ok()` with `ApiResponse.success()` — **not** `ResponseEntity.status(201)`
- No pagination parameters on list endpoints by default — only add `Pageable` if the user explicitly requested pagination in Step 0
- Author comment (`/**
 * author : <name>
 **/`) at the top of every controller class (above imports)
- Use getter methods (`request.getName()`) since request DTOs are POJO classes

---

## Service Interface Template

```java
package com.joshsoftware.inventory.service;

import com.joshsoftware.inventory.dto.request.CreateProductRequestDTO;
import com.joshsoftware.inventory.dto.request.UpdateProductRequestDTO;
import com.joshsoftware.inventory.dto.response.ProductResponseDTO;

import java.util.List;
import java.util.UUID;

/**
 * Service contract for Product domain operations.
 */
public interface ProductService {

    // Creates a new product from the given request payload.
    ProductResponseDTO create(CreateProductRequestDTO request);

    // Retrieves a product by its UUID.
    ProductResponseDTO findById(UUID id);

    // Returns all products as a list. Add Pageable overload only if pagination was requested in Step 0.
    List<ProductResponseDTO> findAll();

    // Updates an existing product.
    ProductResponseDTO update(UUID id, UpdateProductRequestDTO request);

    // Deletes a product by UUID.
    void delete(UUID id);
}
```

---

## ServiceImpl Template

```java
package com.joshsoftware.inventory.service.impl;

/**
 * author : <AuthorName>
 **/

import com.joshsoftware.inventory.dto.request.CreateProductRequestDTO;
import com.joshsoftware.inventory.dto.request.UpdateProductRequestDTO;
import com.joshsoftware.inventory.dto.response.ProductResponseDTO;
import com.joshsoftware.inventory.entity.Product;
import com.joshsoftware.inventory.exception.DuplicateResourceException;
import com.joshsoftware.inventory.exception.ResourceNotFoundException;
import com.joshsoftware.inventory.mapper.ProductMapper;
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

    private final ProductRepository productRepository;
    private final ProductMapper     productMapper;

    // Creates a new product and persists it.
    @Override
    @Transactional
    public ProductResponseDTO create(CreateProductRequestDTO request) {
        log.debug("[ProductServiceImpl#create] ENTRY - sku={}", request.getSku());

        if (productRepository.existsBySku(request.getSku())) {
            throw new DuplicateResourceException("Product", "SKU", request.getSku());
        }

        Product product = productMapper.toEntity(request);
        Product createdProduct = productRepository.save(product);

        log.debug("[ProductServiceImpl#create] EXIT  - productId={}", createdProduct.getId());
        return productMapper.toResponse(createdProduct);
    }

    // Retrieves a product by its UUID.
    @Override
    public ProductResponseDTO findById(UUID id) {
        log.debug("[ProductServiceImpl#findById] ENTRY - productId={}", id);

        ProductResponseDTO response = productRepository.findById(id)
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

**ServiceImpl rules:**
- `@Transactional` on write methods (create, update, delete) and complex multi-step operations only
- **No** `@Transactional(readOnly = true)` on simple reads like `findById`, `findAll`
- `Optional.map(...).orElseThrow(...)` — never call `Optional.get()` directly
- Use descriptive variable names: `createdProduct`, `updatedProduct` — never `saved` or `updated` alone
- Comments only on public service methods (one-line comment above the method) — no inline or block comments elsewhere
- Log ENTRY at DEBUG with key business identifier; writes may log at INFO if significant
- Delegate all mapping to MapStruct; no manual `new Entity()` construction in service
- Use getter methods (`request.getSku()`) since DTOs are POJO classes, not records
- Author comment (`/**
 * author : <name>
 **/`) at the top of every ServiceImpl class (above imports)

---

## Repository Template

```java
package com.joshsoftware.inventory.repository;

import com.joshsoftware.inventory.entity.Product;
import com.joshsoftware.inventory.enums.ProductStatus;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Repository
public interface ProductRepository extends JpaRepository<Product, UUID> {

    Optional<Product> findBySku(String sku);

    boolean existsBySku(String sku);

    List<Product> findByStatus(ProductStatus status);

    List<Product> findByCategoryId(UUID categoryId);

    @Query("SELECT p FROM Product p WHERE p.status = :status AND p.price BETWEEN :min AND :max")
    List<Product> findByStatusAndPriceRange(
            @Param("status") ProductStatus status,
            @Param("min") BigDecimal min,
            @Param("max") BigDecimal max);

    @Query("SELECT p FROM Product p JOIN FETCH p.category WHERE p.id = :id")
    Optional<Product> findByIdWithCategory(@Param("id") UUID id);

    @Modifying
    @Query("UPDATE Product p SET p.status = :status WHERE p.category.id = :categoryId")
    int bulkUpdateStatusByCategoryId(
            @Param("categoryId") UUID categoryId,
            @Param("status") ProductStatus status);
}
```

**Repository rules:**
- Extend `JpaRepository<Entity, PrimaryKeyType>` — primary key type is `UUID` in all new entities
- Derived method names (`findByXxx`) for simple single-field lookups
- `@Query` (JPQL) for multi-condition, join, or aggregate queries
- `JOIN FETCH` to load associations in one query — prevents N+1
- `@Modifying` is mandatory on `UPDATE` / `DELETE` `@Query` methods
- Native SQL only when JPQL cannot express it (e.g. full-text search, DB-specific functions)
- List endpoints return `List<Entity>` by default — only return `Page<Entity>` if pagination was explicitly requested in Step 0
- **No comments at all** in repository — no section dividers, no inline comments
- Import enums from `com.joshsoftware.<app>.enums` — not from `entity/enums/`
