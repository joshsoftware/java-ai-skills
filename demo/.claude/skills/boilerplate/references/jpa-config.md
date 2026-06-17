# JPA Configuration — Entities, Auditing, Relationships, application.yml

## Dependencies Required

See `gradle.md` or `maven.md` — ensure these are present:
- `spring-boot-starter-data-jpa`
- Your JDBC driver (`postgresql` or `mysql-connector-j`)
- For migrations: `flyway-core` (preferred) or `liquibase-core`

---

## application.yml — Complete JPA + DataSource Config

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/${DB_NAME:inventory_db}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:secret}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      pool-name: InventoryHikariPool

  jpa:
    hibernate:
      ddl-auto: update             # Creates missing tables and updates schema automatically
    show-sql: false                 # true only in local dev; use logback for SQL in dev
    open-in-view: false             # ALWAYS false — prevents lazy-load issues in web layer
    properties:
      hibernate:
        format_sql: true
        default_schema: public
        jdbc:
          batch_size: 25           # enables batch inserts
        order_inserts: true
        order_updates: true

  flyway:
    enabled: true
    locations: classpath:db/migration

# For MySQL, replace datasource driver and dialect:
# driver-class-name: com.mysql.cj.jdbc.Driver
# url: jdbc:mysql://localhost:3306/${DB_NAME}?useSSL=false&serverTimezone=UTC
# dialect: org.hibernate.dialect.MySQLDialect
```

---

## Main Application Class

```java
package com.joshsoftware.inventory;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class InventoryApplication {
    public static void main(String[] args) {
        SpringApplication.run(InventoryApplication.class, args);
    }
}
```

> Do **not** add `@EnableJpaAuditing` — auditing annotations are not used in this standard.

---

## Audit Fields — Add Directly to Every Entity

Do **not** use `BaseEntity`, `@MappedSuperclass`, or Spring auditing annotations. Add these fields directly into every entity class. Set `createdAt` and `updatedAt` via MapStruct `expression = "java(LocalDateTime.now())"` in the mapper — not manually in the service:

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
@Column(updatable = false, nullable = false)
private UUID id;

@Column(updatable = false)          // No nullable = false on timestamp fields
private LocalDateTime createdAt;

private LocalDateTime updatedAt;    // No nullable = false on timestamp fields

@Column(updatable = false, length = 100)
private String createdBy;

@Column(length = 100)
private String updatedBy;
```

**Set via mapper expressions (NOT in the service layer):**
```java
// In toEntity mapping:
@Mapping(target = "createdAt", expression = "java(LocalDateTime.now())")
@Mapping(target = "updatedAt", expression = "java(LocalDateTime.now())")

// In updateEntityFromRequest mapping:
@Mapping(target = "updatedAt", expression = "java(LocalDateTime.now())")
```

---

## Entity Template — Product (Full Example)

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
import lombok.ToString;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@ToString(exclude = "category")
@Entity
@Table(
    name = "products",
    indexes = {
        @Index(name = "idx_product_sku", columnList = "sku", unique = true),
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

    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<ProductImage> images = new ArrayList<>();

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

## Enum Best Practices

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

- Always `@Enumerated(EnumType.STRING)` on entity fields — never `ORDINAL`
- Place enums in `com.joshsoftware.<app>.enums` — **not** inside `entity/enums/`
- Every enum must have `@Getter`, a `private final String value` field, and a constructor
- Import enums from the top-level `enums/` package in entities, repositories, and mappers

---

## JPA Relationship Rules

| Relationship    | Fetch Type       | Cascade Rule                            |
|-----------------|------------------|-----------------------------------------|
| `@ManyToOne`    | `LAZY` (always)  | No cascade — parent owns child          |
| `@OneToMany`    | `LAZY` (default) | `CascadeType.ALL` + `orphanRemoval` if child is owned |
| `@OneToOne`     | `LAZY`           | `CascadeType.ALL` if tightly coupled    |
| `@ManyToMany`   | `LAZY`           | Use join entity instead of `@ManyToMany` for extra columns |

**Never use `FetchType.EAGER`** — causes N+1 problems and uncontrolled joins.

## Repository Patterns — JPA Query Reference

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

    @Query(value = "SELECT * FROM products WHERE to_tsvector(name) @@ plainto_tsquery(:query)",
           nativeQuery = true)
    List<Product> fullTextSearch(@Param("query") String query);
}
```
