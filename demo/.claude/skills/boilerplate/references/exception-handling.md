# Exception Handling

## Package Structure

```
com.joshsoftware.<app>/
└── exception/
    ├── ResourceNotFoundException.java
    ├── BusinessException.java
    ├── DuplicateResourceException.java
    └── handler/
        └── GlobalExceptionHandler.java
```

---

## Base Exception

```java
package com.joshsoftware.app.exception;

import lombok.Getter;
import org.springframework.http.HttpStatus;

/**
 * Base class for all application-level exceptions.
 * Carries an HTTP status so the handler doesn't need a mapping table.
 */
@Getter
public abstract class AppException extends RuntimeException {

    private final HttpStatus status;

    protected AppException(String message, HttpStatus status) {
        super(message);
        this.status = status;
    }

    protected AppException(String message, HttpStatus status, Throwable cause) {
        super(message, cause);
        this.status = status;
    }
}
```

---

## Common Exceptions

```java
// ResourceNotFoundException.java
package com.joshsoftware.app.exception;

import org.springframework.http.HttpStatus;

public class ResourceNotFoundException extends AppException {

    public ResourceNotFoundException(String resource, String field, Object value) {
        super("%s not found with %s: '%s'".formatted(resource, field, value),
              HttpStatus.NOT_FOUND);
    }
}
```

```java
// DuplicateResourceException.java
package com.joshsoftware.app.exception;

import org.springframework.http.HttpStatus;

public class DuplicateResourceException extends AppException {

    public DuplicateResourceException(String resource, String field, Object value) {
        super("%s already exists with %s: '%s'".formatted(resource, field, value),
              HttpStatus.CONFLICT);
    }
}
```

```java
// BusinessException.java  — for domain rule violations
package com.joshsoftware.app.exception;

import org.springframework.http.HttpStatus;

public class BusinessException extends AppException {

    public BusinessException(String message) {
        super(message, HttpStatus.UNPROCESSABLE_ENTITY);
    }
}
```

---

## Error Response Shape

```java
package com.joshsoftware.app.common;

import com.fasterxml.jackson.annotation.JsonInclude;
import java.time.Instant;
import java.util.List;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record ErrorResponse(
        int status,
        String message,
        List<FieldError> errors,   // populated for validation failures only
        Instant timestamp
) {
    public record FieldError(String field, String message) {}

    public static ErrorResponse of(int status, String message) {
        return new ErrorResponse(status, message, null, Instant.now());
    }

    public static ErrorResponse withFieldErrors(int status, String message,
                                                List<FieldError> errors) {
        return new ErrorResponse(status, message, errors, Instant.now());
    }
}
```

---

## Global Exception Handler

```java
package com.joshsoftware.app.exception.handler;

import com.joshsoftware.app.common.ErrorResponse;
import com.joshsoftware.app.exception.AppException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.List;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /** Handles all custom application exceptions uniformly. */
    @ExceptionHandler(AppException.class)
    public ResponseEntity<ErrorResponse> handleAppException(AppException ex) {
        log.warn("Application exception [{}]: {}", ex.getStatus(), ex.getMessage());
        return ResponseEntity
                .status(ex.getStatus())
                .body(ErrorResponse.of(ex.getStatus().value(), ex.getMessage()));
    }

    /** Handles @Valid / @Validated bean-validation failures. */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(
            MethodArgumentNotValidException ex) {

        List<ErrorResponse.FieldError> fieldErrors = ex.getBindingResult()
                .getFieldErrors()
                .stream()
                .map(fe -> new ErrorResponse.FieldError(fe.getField(), fe.getDefaultMessage()))
                .toList();

        log.warn("Validation failed: {}", fieldErrors);
        return ResponseEntity.badRequest()
                .body(ErrorResponse.withFieldErrors(400, "Validation failed", fieldErrors));
    }

    /** Safety net — log and return 500 without leaking internals. */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception ex) {
        log.error("Unexpected error", ex);
        return ResponseEntity.internalServerError()
                .body(ErrorResponse.of(500, "An unexpected error occurred"));
    }
}
```

**Exception handling rules:**
- **Never** catch and swallow exceptions in service/repository layers
- **Never** expose stack traces or internal class names in API responses
- All new exception types must extend `AppException` — carry the HTTP status in the exception itself
- Log `WARN` for expected business errors; `ERROR` for unexpected `Exception`
- The global handler is the single place HTTP status codes are set — services only throw domain exceptions
