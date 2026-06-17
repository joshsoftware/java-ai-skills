# Unit Test Case Framework

## Testing Stack

Use the standard Spring Boot testing stack:

| Concern | Library |
|---------|---------|
| Unit testing | JUnit 5 |
| Mocking | Mockito |
| Assertions | AssertJ |
| Spring MVC controller tests | MockMvc |
| Service tests | JUnit 5 + Mockito |
| Integration tests | `@SpringBootTest` only when required |

Prefer lightweight unit and slice tests before full integration tests.

---

## Required Dependencies

### Gradle

```kotlin
testImplementation("org.springframework.boot:spring-boot-starter-test")
testImplementation("org.springframework.security:spring-security-test")
```

```kotlin
tasks.withType<Test> {
    useJUnitPlatform()
}
```

### Maven

```xml
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
```

---

## Test Package Structure

Mirror the main package structure under `src/test/java`.

```text
src/test/java/com/joshsoftware/<app>/
+-- controller/
+-- service/
   +-- impl/

```

Test class names must end with `Test`.

Examples:

```text
ProductControllerTest
ProductServiceImplTest
```

---

## Testing Rules

- Write tests for every public service method.
- Write controller tests for every REST endpoint.
- Mock only external dependencies of the class under test.
- Do not mock the class being tested.
- Do not use `@SpringBootTest` for simple unit tests.
- Use `@WebMvcTest` for controller tests.
- Use `@ExtendWith(MockitoExtension.class)` for service unit tests.
- Use AssertJ assertions: `assertThat(...)`.
- Test success, validation failure, not-found, and business error paths.
- Test method names should describe behavior clearly.

---

## Service Unit Test Pattern

Use this pattern for service implementation tests.

```java
@ExtendWith(MockitoExtension.class)
class ProductServiceImplTest {

    @Mock
    private ProductRepository productRepository;

    @Mock
    private ProductMapper productMapper;

    @InjectMocks
    private ProductServiceImpl productService;

    @Test
    void createProduct_shouldReturnProductResponse_whenRequestIsValid() {
        CreateProductRequest request = new CreateProductRequest("Keyboard", BigDecimal.valueOf(1200));
        Product product = Product.builder()
                .name("Keyboard")
                .price(BigDecimal.valueOf(1200))
                .build();
        Product savedProduct = Product.builder()
                .id(1L)
                .name("Keyboard")
                .price(BigDecimal.valueOf(1200))
                .build();
        ProductResponse response = new ProductResponse(1L, "Keyboard", BigDecimal.valueOf(1200));

        when(productMapper.toEntity(request)).thenReturn(product);
        when(productRepository.save(product)).thenReturn(savedProduct);
        when(productMapper.toResponse(savedProduct)).thenReturn(response);

        ProductResponse result = productService.createProduct(request);

        assertThat(result).isNotNull();
        assertThat(result.id()).isEqualTo(1L);
        assertThat(result.name()).isEqualTo("Keyboard");

        verify(productRepository).save(product);
        verify(productMapper).toEntity(request);
        verify(productMapper).toResponse(savedProduct);
    }
}
```

---

## Controller Test Pattern

Use `@WebMvcTest` for controller tests.

```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ProductService productService;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void createProduct_shouldReturnCreatedResponse_whenRequestIsValid() throws Exception {
        CreateProductRequest request = new CreateProductRequest("Keyboard", BigDecimal.valueOf(1200));
        ProductResponse response = new ProductResponse(1L, "Keyboard", BigDecimal.valueOf(1200));

        when(productService.createProduct(any(CreateProductRequest.class))).thenReturn(response);

        mockMvc.perform(post("/api/v1/products")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.status").value(201))
                .andExpect(jsonPath("$.message").exists())
                .andExpect(jsonPath("$.data.id").value(1))
                .andExpect(jsonPath("$.data.name").value("Keyboard"));

        verify(productService).createProduct(any(CreateProductRequest.class));
    }
}
```

---

## Anti-Patterns

| Anti-Pattern | Why Forbidden |
|--------------|---------------|
| Using `@SpringBootTest` for every test | Slow and unnecessary |
| Testing implementation details | Makes tests fragile |
| Mocking DTOs/entities | They are simple data objects |
| Calling private methods directly | Test public behavior |
| Using `System.out.println` in tests | Use assertions |
| Writing tests without assertions | Does not verify behavior |
| Ignoring failure paths | Misses real bugs |
| Sharing mutable test data across tests | Causes flaky tests |

---


