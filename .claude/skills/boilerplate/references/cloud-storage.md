# Cloud Storage — S3 · GCS · Provider Abstraction

## Package Structure

```
com.joshsoftware.<app>/
├── config/
│   └── storage/
│       ├── StorageProperties.java        ← @ConfigurationProperties(prefix = "app.storage")
│       ├── S3Config.java                 ← S3Client + S3Presigner beans
│       └── GcsConfig.java                ← Storage bean
├── service/
│   └── storage/
│       ├── StorageService.java           ← provider-agnostic interface — no SDK or Spring Web types
│       └── impl/
│           ├── S3StorageServiceImpl.java
│           └── GcsStorageServiceImpl.java
├── dto/
│   └── response/
│       ├── FileUploadResponseDTO.java
│       └── PreSignedUrlResponseDTO.java
├── controller/
│   └── FileController.java
├── exception/
│   ├── FileStorageException.java     ← 500
│   ├── FileValidationException.java  ← 400
│   └── FileNotFoundException.java    ← 404
└── util/
    └── FileValidationUtil.java       ← static, no Spring annotations
```

---

## Dependencies

> For Maven equivalents see `maven.md`. Do NOT add `aws-java-sdk-s3` (SDK v1).

**Gradle — AWS S3**
```groovy
implementation platform('software.amazon.awssdk:bom:2.25.60')
implementation 'software.amazon.awssdk:s3'
implementation 'software.amazon.awssdk:sts'   // assume-role / temporary credentials
```

**Gradle — GCS**
```groovy
implementation platform('com.google.cloud:libraries-bom:26.37.0')
implementation 'com.google.cloud:google-cloud-storage'
```

---

## application.yml / application.yaml / application.properties

> Credentials are **never** in config files — use IAM roles, env vars, or secrets manager.

### YAML Format (`application.yml` or `application.yaml`)
```yaml
app:
  storage:
    provider: s3                          # s3 | gcs — drives @ConditionalOnProperty
    bucket-name: ${STORAGE_BUCKET_NAME}
    region: ${AWS_REGION:ap-south-1}      # AWS only — ignored by GCS
    project-id: ${GCS_PROJECT_ID:}        # GCS only — ignored by AWS
    prefix: uploads/
    presigned-url-expiry-minutes: 15
    max-file-size-mb: 10
    allowed-content-types:
      - image/jpeg
      - image/png
      - image/webp
      - application/pdf

spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 10MB               # must match app.storage.max-file-size-mb
      max-request-size: 15MB
      file-size-threshold: 2KB
```

### Properties Format (`application.properties`)
```properties
# s3 | gcs — drives @ConditionalOnProperty
app.storage.provider=s3
app.storage.bucket-name=${STORAGE_BUCKET_NAME}
# AWS only — ignored by GCS
app.storage.region=${AWS_REGION:ap-south-1}
# GCS only — ignored by AWS
app.storage.project-id=${GCS_PROJECT_ID:}
app.storage.prefix=uploads/
app.storage.presigned-url-expiry-minutes=15
app.storage.max-file-size-mb=10
app.storage.allowed-content-types=image/jpeg,image/png,image/webp,application/pdf

spring.servlet.multipart.enabled=true
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=15MB
spring.servlet.multipart.file-size-threshold=2KB
```

---

## StorageProperties

```java
@Getter @Setter @Validated @Configuration
@ConfigurationProperties(prefix = "app.storage")
public class StorageProperties {
    @NotBlank private String provider;
    @NotBlank private String bucketName;
    private String region = "ap-south-1";
    private String projectId;
    private String prefix = "uploads/";
    @Min(1) private int presignedUrlExpiryMinutes = 15;
    @Min(1) private int maxFileSizeMb = 10;
    @NotEmpty private List<String> allowedContentTypes;
}
```

---

## S3Config

```java
@Configuration
@RequiredArgsConstructor
@ConditionalOnProperty(name = "app.storage.provider", havingValue = "s3")
public class S3Config {

    private final StorageProperties storageProperties;

    @Bean
    public S3Client s3Client() {
        return S3Client.builder()
                .region(Region.of(storageProperties.getRegion()))
                .credentialsProvider(DefaultCredentialsProvider.create())
                .build();
    }

    @Bean
    public S3Presigner s3Presigner() {
        return S3Presigner.builder()
                .region(Region.of(storageProperties.getRegion()))
                .credentialsProvider(DefaultCredentialsProvider.create())
                .build();
    }
}
```

---

## GcsConfig

```java
@Configuration
@ConditionalOnProperty(name = "app.storage.provider", havingValue = "gcs")
public class GcsConfig {

    @Bean
    public Storage gcsStorage(StorageProperties props) {
        return StorageOptions.newBuilder()
                .setProjectId(props.getProjectId())
                .build()
                .getService();
    }
}
```

---

## StorageService Interface

```java
public interface StorageService {
    // Controller extracts InputStream + metadata from MultipartFile — never pass MultipartFile here
    FileUploadResponseDTO upload(String fileName, String contentType, long size, InputStream inputStream);
    PreSignedUrlResponseDTO generatePreSignedUrl(String objectKey);
    void delete(String objectKey);
    boolean exists(String objectKey);
}
```

---

## S3StorageServiceImpl (full pattern — GCS follows same structure)

```java
@Slf4j @Service @RequiredArgsConstructor
@ConditionalOnProperty(name = "app.storage.provider", havingValue = "s3")
public class S3StorageServiceImpl implements StorageService {

    private final S3Client s3Client;
    private final S3Presigner s3Presigner;
    private final StorageProperties storageProperties;

    @Override
    public FileUploadResponseDTO upload(String fileName, String contentType,
                                     long size, InputStream inputStream) {
        log.debug("[S3StorageServiceImpl#upload] ENTRY - fileName={}", fileName);

        FileValidationUtil.validate(fileName, contentType, size,
                storageProperties.getMaxFileSizeMb(), storageProperties.getAllowedContentTypes());

        String objectKey = buildObjectKey(fileName);
        try {
            s3Client.putObject(
                    PutObjectRequest.builder()
                            .bucket(storageProperties.getBucketName())
                            .key(objectKey).contentType(contentType).contentLength(size)
                            .build(),
                    RequestBody.fromInputStream(inputStream, size));
        } catch (S3Exception e) {
            throw new FileStorageException("Failed to upload file to S3: " + e.getMessage(), e);
        }

        log.debug("[S3StorageServiceImpl#upload] EXIT - objectKey={}", objectKey);
        return new FileUploadResponseDTO(objectKey, fileName, contentType, size);
    }

    @Override
    public PreSignedUrlResponseDTO generatePreSignedUrl(String objectKey) {
        log.debug("[S3StorageServiceImpl#generatePreSignedUrl] ENTRY - objectKey={}", objectKey);
        if (!exists(objectKey)) throw new FileNotFoundException("File", "objectKey", objectKey);

        String url = s3Presigner.presignGetObject(
                GetObjectPresignRequest.builder()
                        .signatureDuration(Duration.ofMinutes(storageProperties.getPresignedUrlExpiryMinutes()))
                        .getObjectRequest(b -> b.bucket(storageProperties.getBucketName()).key(objectKey))
                        .build()
        ).url().toString();

        log.debug("[S3StorageServiceImpl#generatePreSignedUrl] EXIT - objectKey={}", objectKey);
        return new PreSignedUrlResponseDTO(url, storageProperties.getPresignedUrlExpiryMinutes());
    }

    @Override
    public void delete(String objectKey) {
        log.debug("[S3StorageServiceImpl#delete] ENTRY - objectKey={}", objectKey);
        try {
            s3Client.deleteObject(DeleteObjectRequest.builder()
                    .bucket(storageProperties.getBucketName()).key(objectKey).build());
        } catch (S3Exception e) {
            throw new FileStorageException("Failed to delete file: " + e.getMessage(), e);
        }
        log.debug("[S3StorageServiceImpl#delete] EXIT - objectKey={}", objectKey);
    }

    @Override
    public boolean exists(String objectKey) {
        try {
            s3Client.headObject(HeadObjectRequest.builder()
                    .bucket(storageProperties.getBucketName()).key(objectKey).build());
            return true;
        } catch (NoSuchKeyException e) { return false; }
    }

    private String buildObjectKey(String fileName) {
        return storageProperties.getPrefix() + UUID.randomUUID() + "/" + fileName;
    }
}
```

**GCS key operation differences** (same class structure, swap SDK calls):
```java
// upload
BlobInfo blobInfo = BlobInfo.newBuilder(BlobId.of(bucket, objectKey)).setContentType(contentType).build();
gcsStorage.createFrom(blobInfo, inputStream);

// presigned URL — always use V4 signing
gcsStorage.signUrl(blobInfo, expiryMinutes, TimeUnit.MINUTES, Storage.SignUrlOption.withV4Signature());

// delete — returns boolean; log warn if false, never throw
boolean deleted = gcsStorage.delete(BlobId.of(bucket, objectKey));

// exists
Blob blob = gcsStorage.get(BlobId.of(bucket, objectKey));
return blob != null && blob.exists();
```

---

## DTOs (POJO classes)

```java
// dto/response/FileUploadResponseDTO.java
@Getter
@Setter
@AllArgsConstructor
public class FileUploadResponseDTO {
    private String objectKey;
    private String fileName;
    private String contentType;
    private long sizeBytes;
}
```

```java
// dto/response/PreSignedUrlResponseDTO.java
@Getter
@Setter
@AllArgsConstructor
public class PreSignedUrlResponseDTO {
    private String url;
    private int expiresInMinutes;
}
```

---

## FileValidationUtil

```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class FileValidationUtil {

    public static void validate(String fileName, String contentType,
                                long size, int maxMb, List<String> allowed) {
        if (fileName == null || fileName.isBlank())
            throw new FileValidationException("File name is required");

        if (size <= 0)
            throw new FileValidationException("File is empty");

        if (size > (long) maxMb * 1024 * 1024)
            throw new FileValidationException(
                    "File size exceeds maximum allowed %d MB".formatted(maxMb));

        if (contentType == null || !allowed.contains(contentType))
            throw new FileValidationException(
                    "Content type '%s' is not allowed. Allowed: %s".formatted(contentType, allowed));
    }
}
```

---

## FileController

> **Prompt Instruction for Claude:** Before generating the controller, you MUST ask the user what the REST API base path should be (e.g., `/api/v1/files` vs `/api/v1/documents`). Wait for the user to type the path name before proceeding.

```java
@Slf4j @RestController @RequiredArgsConstructor
@RequestMapping("/api/v1/files")
public class FileController {

    private final StorageService storageService;

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<ApiResponse<FileUploadResponseDTO>> upload(
            @RequestParam("file") MultipartFile file) throws IOException {
        log.info("[FileController#upload] ENTRY - fileName={}", file.getOriginalFilename());

        try (InputStream is = file.getInputStream()) {
            FileUploadResponseDTO response = storageService.upload(
                    file.getOriginalFilename(), file.getContentType(), file.getSize(), is);
            log.info("[FileController#upload] EXIT - objectKey={}", response.objectKey());
            return ResponseEntity.ok(ApiResponse.success("File uploaded successfully", response));
        }
    }

    // objectKey contains slashes — always @RequestParam, never @PathVariable
    @GetMapping("/url")
    public ResponseEntity<ApiResponse<PreSignedUrlResponseDTO>> getPreSignedUrl(
            @RequestParam("key") String objectKey) {
        log.info("[FileController#getPreSignedUrl] ENTRY - objectKey={}", objectKey);
        PreSignedUrlResponseDTO response = storageService.generatePreSignedUrl(objectKey);
        log.info("[FileController#getPreSignedUrl] EXIT");
        return ResponseEntity.ok(ApiResponse.success("Pre-signed URL generated", response));
    }

    @DeleteMapping
    public ResponseEntity<ApiResponse<Void>> delete(@RequestParam("key") String objectKey) {
        log.info("[FileController#delete] ENTRY - objectKey={}", objectKey);
        storageService.delete(objectKey);
        log.info("[FileController#delete] EXIT");
        return ResponseEntity.ok(ApiResponse.success("File deleted successfully", null));
    }
}
```

---

## Exceptions

```java
// 500 — wraps SDK errors
public class FileStorageException extends AppException {
    public FileStorageException(String message, Throwable cause) {
        super(message, HttpStatus.INTERNAL_SERVER_ERROR, cause);
    }
}

// 400 — validation failure
public class FileValidationException extends AppException {
    public FileValidationException(String message) { super(message, HttpStatus.BAD_REQUEST); }
}

// 404 — delegates to existing ResourceNotFoundException
public class FileNotFoundException extends ResourceNotFoundException {
    public FileNotFoundException(String resource, String field, Object value) {
        super(resource, field, value);
    }
}
```

> Add `FileStorageException` → 500 and `FileValidationException` → 400 handler methods to the
> existing `GlobalExceptionHandler` — never create a second `@RestControllerAdvice`.

---

## Design Rules

- `StorageService` interface never imports SDK types or `MultipartFile` — JDK types only (`String`, `InputStream`, `long`)
- Controller extracts `InputStream` + metadata from `MultipartFile` before calling service
- Provider swap requires changes only in `config/storage/` and `service/storage/impl/` — zero changes elsewhere
- `@ConditionalOnProperty(app.storage.provider)` drives provider selection — no `if/else` in code
- Credentials via `DefaultCredentialsProvider` (AWS) or ADC (GCS) — never hardcoded
- `FileValidationUtil` is static — no `@Component`, no injected dependencies
- Validation runs inside service `upload()` before any SDK call
- Object keys always UUID-prefixed — prevents collisions and enumeration
- Object keys passed as `@RequestParam` — never `@PathVariable` (slashes break path resolution)
- S3 `deleteObject` is idempotent — never call `exists()` before `delete()`
- GCS `signUrl` always uses `Storage.SignUrlOption.withV4Signature()` — V2 is deprecated

---

## Anti-Patterns

| Anti-Pattern                                              | Why Forbidden                                          |
|-----------------------------------------------------------|--------------------------------------------------------|
| SDK types in `StorageService` interface or DTOs           | Couples entire app to provider; impossible to swap     |
| `MultipartFile` in service interface                      | Couples service to Spring Web                          |
| Hardcoded credentials anywhere                            | Security violation — use IAM roles or env vars         |
| `byte[]` for file content                                 | OOM risk; always stream via `InputStream`              |
| Permanent public URLs                                     | Security violation — always use time-limited presigned |
| No content-type validation                                | Allows malicious uploads disguised as allowed types    |
| AWS SDK v1 (`com.amazonaws:aws-java-sdk-s3`)              | Deprecated — always SDK v2                             |
| One bucket across all environments                        | Cross-environment data leaks                           |
| File metadata stored only in cloud provider               | No queryability — store metadata in DB, file in bucket |
| `if provider == s3` branching in service                  | Use `@ConditionalOnProperty` with separate impls       |
| `exists()` before `delete()` on S3                        | Extra round trip + race condition; delete is idempotent|
| `@PathVariable` for object keys                           | Slashes in keys break Spring MVC path resolution       |
| `@Component` on utility classes                           | Utils must be static and stateless                     |
