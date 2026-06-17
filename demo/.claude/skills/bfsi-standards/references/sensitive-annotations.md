# Sensitive Data Annotations — BFSI Standards

## Package Location
```
com.joshsoftware.<app>/
└── annotation/
    ├── SensitiveData.java     ← marks fields/params as BFSI-sensitive
    └── MaskField.java         ← drives serialisation-level masking
```

---

## @SensitiveData Annotation

Marks any field, parameter, or return value as containing BFSI-regulated data.
Serves as: documentation marker, static analysis target, and runtime serialisation hook.

```java
package com.joshsoftware.app.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a field or parameter as containing BFSI-regulated sensitive data.
 *
 * Rules enforced by this annotation (checked in code review and at runtime):
 * - Field must never appear in any log statement without masking
 * - Field must be excluded from toString() (use @ToString.Exclude)
 * - If persisted, field must use an AttributeConverter for encryption
 * - If returned via API, field must use @MaskField or @JsonIgnore
 */
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface SensitiveData {

    SensitiveDataType type();

    enum SensitiveDataType {
        PAN,              // Credit/debit card primary account number
        CVV,              // Card verification value — must never be stored
        AADHAAR,          // Aadhaar UID
        BANK_ACCOUNT,     // Bank account number
        IFSC,             // IFSC code
        PASSWORD,         // User password or PIN
        OTP,              // One-time password
        API_KEY,          // External service API key
        JWT_SECRET,       // JWT signing secret
        OAUTH_TOKEN,      // OAuth access/refresh token
        MOBILE,           // Mobile phone number (PII)
        EMAIL,            // Email address (PII)
        POLICY_NUMBER,    // Insurance policy number
        LOAN_ACCOUNT,     // Loan account number
        GENERIC_PII       // Other personally identifiable information
    }
}
```

---

## @MaskField Annotation

Controls how a field is serialised in API responses. Works with `MaskingSerializer`.

```java
package com.joshsoftware.app.annotation;

import com.fasterxml.jackson.annotation.JacksonAnnotationsInside;
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import com.joshsoftware.app.serializer.MaskingSerializer;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Applied to response DTO fields that should be masked during JSON serialisation.
 * Combine with @SensitiveData to document intent and drive serialisation masking.
 *
 * Usage:
 *   @MaskField(type = MaskType.PAN)
 *   private String maskedPan;
 */
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@JacksonAnnotationsInside
@JsonSerialize(using = MaskingSerializer.class)
public @interface MaskField {

    MaskType type() default MaskType.GENERIC;

    enum MaskType {
        PAN,
        AADHAAR,
        ACCOUNT_NUMBER,
        MOBILE,
        EMAIL,
        POLICY_NUMBER,
        TOKEN,
        GENERIC
    }
}
```

---

## MaskingSerializer (Jackson)

```java
package com.joshsoftware.app.serializer;

import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.BeanProperty;
import com.fasterxml.jackson.databind.JsonSerializer;
import com.fasterxml.jackson.databind.SerializerProvider;
import com.fasterxml.jackson.databind.ser.ContextualSerializer;
import com.joshsoftware.app.annotation.MaskField;
import com.joshsoftware.app.util.MaskingUtil;

import java.io.IOException;

/**
 * Jackson serialiser that applies MaskingUtil based on the @MaskField type.
 * Registered automatically via @JsonSerialize on @MaskField.
 */
public class MaskingSerializer extends JsonSerializer<String> implements ContextualSerializer {

    private MaskField.MaskType maskType = MaskField.MaskType.GENERIC;

    @Override
    public void serialize(String value, JsonGenerator gen, SerializerProvider serializers)
            throws IOException {
        if (value == null) {
            gen.writeNull();
            return;
        }
        gen.writeString(applyMask(value));
    }

    @Override
    public JsonSerializer<?> createContextual(SerializerProvider prov, BeanProperty property) {
        if (property != null) {
            MaskField annotation = property.getAnnotation(MaskField.class);
            if (annotation != null) {
                MaskingSerializer serializer = new MaskingSerializer();
                serializer.maskType = annotation.type();
                return serializer;
            }
        }
        return this;
    }

    private String applyMask(String value) {
        return switch (maskType) {
            case PAN            -> MaskingUtil.maskPan(value);
            case AADHAAR        -> MaskingUtil.maskAadhaar(value);
            case ACCOUNT_NUMBER -> MaskingUtil.maskAccountNumber(value);
            case MOBILE         -> MaskingUtil.maskMobile(value);
            case EMAIL          -> MaskingUtil.maskEmail(value);
            case POLICY_NUMBER  -> MaskingUtil.maskPolicyNumber(value);
            case TOKEN          -> MaskingUtil.maskToken(value);
            default             -> "[REDACTED]";
        };
    }
}
```

---

## Applying Annotations to Entity Fields

```java
package com.joshsoftware.app.entity;

import com.joshsoftware.app.annotation.SensitiveData;
import com.joshsoftware.app.converter.PanAttributeConverter;
import com.joshsoftware.app.converter.AadhaarAttributeConverter;
import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;

@Entity
@Table(name = "customers")
@Getter
@Setter
@ToString(exclude = {"pan", "aadhaarNumber", "passwordHash"})  // never in toString()
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private java.util.UUID id;

    private String name;

    @SensitiveData(type = SensitiveData.SensitiveDataType.EMAIL)
    private String email;

    @SensitiveData(type = SensitiveData.SensitiveDataType.MOBILE)
    private String mobile;

    @SensitiveData(type = SensitiveData.SensitiveDataType.PAN)
    @Convert(converter = PanAttributeConverter.class)   // encrypted at rest
    @Column(name = "pan_encrypted")
    private String pan;

    @SensitiveData(type = SensitiveData.SensitiveDataType.AADHAAR)
    @Convert(converter = AadhaarAttributeConverter.class)  // encrypted at rest
    @Column(name = "aadhaar_encrypted")
    private String aadhaarNumber;

    @SensitiveData(type = SensitiveData.SensitiveDataType.PASSWORD)
    @Column(name = "password_hash")
    private String passwordHash;   // BCrypt hash — never the plain password
}
```

---

## Rules

- `@SensitiveData` is documentation + enforcement — place on every sensitive field in entities and DTOs
- `@ToString.Exclude` is mandatory alongside `@SensitiveData` — never skip it
- `@JsonIgnore` for fields that must never appear in any API response (CVV, password, OTP)
- `@MaskField` for fields that may appear in responses but must be masked (PAN last-4, masked email)
- Never use `@Data` on entities or classes with sensitive fields — use targeted Lombok annotations
- CVV must have no field declared in entity at all — it is validated transiently and discarded