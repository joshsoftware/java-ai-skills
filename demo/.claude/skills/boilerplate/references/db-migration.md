# DB Migration — Flyway · Schema Design · Seed Data

## Package Structure

```
src/main/resources/db/changelog/
├── create_products_table.sql
├── create_categories_table.sql
├── alter_products_add_category_fk.sql
├── alter_products_rename_unit_price.sql
└── drop_products_barcode_column.sql
```

---

## Flyway Dependency

### Gradle (build.gradle)
```groovy
dependencies {
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql' // required for Flyway 10+ with PostgreSQL
    // MySQL / MariaDB: replace flyway-database-postgresql with flyway-mysql
    // SQL Server:      replace with flyway-sqlserver
    runtimeOnly 'org.postgresql:postgresql'
}
```

### Maven (pom.xml)
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <!-- version managed by Spring Boot BOM -->
</dependency>
<!-- Required for Flyway 10+ with PostgreSQL: -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
<!-- MySQL / MariaDB: replace the above with flyway-mysql -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

> **Using Liquibase instead?** This file covers Flyway only — it is the Josh Software default.
> For the Liquibase dependency snippet, see `gradle.md` / `maven.md`. The same schema design
> rules (UUID PKs, `TIMESTAMPTZ`, named constraints, no `ddl-auto` in production) apply regardless of tool.

---

## application.yml — Datasource + Flyway

> JPA config (`ddl-auto`, `show-sql`, Hibernate properties) is owned by `jpa-config.md`. Only datasource and Flyway belong here.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/${DB_NAME:inventory_db}
    username: ${DB_USERNAME}        # always from env — never hardcoded
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      pool-name: InventoryHikariPool

  flyway:
    enabled: true
    locations: classpath:db/changelog
    baseline-on-migrate: false      # true only when adopting Flyway on an existing DB
    out-of-order: false             # never allow out-of-order in production
    validate-on-migrate: true
    clean-disabled: true            # NEVER allow flyway:clean in production
```

> **Profile tip:** `flyway.clean-disabled=false` is only acceptable in `application-local.yml`.

---

## Naming Convention

Pattern: `{sql_command}_{table_name}.sql` — snake_case, SQL command first (`create`, `insert`, `alter`, `drop`). If a version is needed, append it at the end.

```
✅  create_products_table.sql
✅  insert_product_categories.sql
✅  alter_products_add_category_fk.sql
✅  drop_products_barcode_column.sql
✅  create_products_table_v2.sql          ← versioned variant, suffix only

❌  add_category_fk_to_products.sql       ← add is not a SQL command
❌  rename_unit_price_to_price.sql        ← rename is not a SQL command
❌  CreateProductsTable.sql               ← PascalCase
❌  init.sql                              ← too vague
❌  products.sql                          ← no SQL command
```

---

## Migration Templates

### Create Table

```sql
-- create_products_table.sql
-- Creates the core products table with UUID PK, audit columns, and named constraints.

CREATE TABLE products (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100) NOT NULL,
    description TEXT,
    price NUMERIC(19,2) NOT NULL,
    stock INTEGER NOT NULL DEFAULT 0,
    status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by VARCHAR(100),
    updated_by VARCHAR(100),

    CONSTRAINT pk_products PRIMARY KEY (id),
    CONSTRAINT uq_products_sku UNIQUE (sku),
    CONSTRAINT ck_products_price CHECK (price >= 0),
    CONSTRAINT ck_products_status CHECK (status IN ('ACTIVE', 'INACTIVE', 'DISCONTINUED'))
);
```

### Alter Table — Add FK + Index

```sql
-- alter_products_add_category_fk.sql
-- Adds category_id FK; nullable to preserve existing rows.
-- Every FK column must have a matching index in the same migration.

ALTER TABLE products ADD COLUMN category_id UUID;

ALTER TABLE products
    ADD CONSTRAINT fk_products_category
        FOREIGN KEY (category_id) REFERENCES categories (id)
        ON DELETE SET NULL;

CREATE INDEX idx_products_category_id ON products (category_id);
```

### Alter Table — Rename Column

```sql
-- alter_products_rename_unit_price.sql
-- Aligns column name with ubiquitous language agreed in sprint review.

ALTER TABLE products RENAME COLUMN unit_price TO price;
```

### Drop Column

```sql
-- drop_products_barcode_column.sql
-- Removes deprecated barcode column; data migrated to sku in create_products_table.sql.
-- IRREVERSIBLE — ensure backup before running in production.

ALTER TABLE products DROP COLUMN IF EXISTS barcode;
```

---

## Schema Design Rules

- PKs are always `UUID DEFAULT gen_random_uuid()` — never `BIGINT SERIAL` or `INT AUTO_INCREMENT`
- All `VARCHAR` columns must have an explicit length — never unbounded `VARCHAR`
- Monetary values use `NUMERIC(19,2)` — never `FLOAT` or `DOUBLE`
- Timestamps use `TIMESTAMPTZ` — matches `BaseEntity`'s `Instant` (UTC); prevents TZ ambiguity across environments
- Enum-like columns use `VARCHAR` + `CHECK` constraint — never DB-native `ENUM` (not portable)
- Every FK column gets a matching index in the same migration
- Name every constraint explicitly — pattern: `{prefix}_{table}_{column}` with prefixes `pk_`, `uq_`, `fk_`, `ck_`, `idx_`
- Always add a top-of-file comment explaining the purpose and rationale
- Destructive changes (`DROP`, `RENAME`) must carry an `-- IRREVERSIBLE` warning comment
- Data backfills belong in a **separate versioned file** from the schema change that precedes them

---

## Anti-Patterns (Strictly Forbidden)

| Anti-Pattern                                               | Why Forbidden                                             |
|------------------------------------------------------------|-----------------------------------------------------------|
| `ddl-auto: create` or `update` in non-local profile        | Irreversible data loss; bypasses migration history        |
| Editing an already-applied versioned migration file        | Flyway checksum mismatch — blocks startup                 |
| DB-native `ENUM` type                                      | Not portable; adding values requires DDL on all envs      |
| `FLOAT` / `DOUBLE` for monetary columns                    | Floating-point imprecision in financial calculations      |
| Unnamed constraints                                        | Non-portable; impossible to reference in later migrations |
| Schema change + data backfill in one migration             | Backfill failure locks the schema change; hard to retry   |
| `flyway.clean-disabled=false` in production                | `flyway:clean` drops entire schema — catastrophic         |
| Hardcoded credentials in `application.yml`                 | Security violation; always use env vars                   |
| Missing index on FK column                                 | Full-table scans on every join                            |