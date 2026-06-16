# Data Masking — BFSI Standards

## Package Location
```
com.joshsoftware.<app>/
└── util/
    └── MaskingUtil.java
```

---

## MaskingUtil — Central Masking Library

```java
package com.joshsoftware.app.util;

import org.springframework.util.StringUtils;

/**
 * Centralised masking for BFSI-sensitive data.
 * Use these methods everywhere: logs, audit trails, API responses.
 * Never write ad-hoc masking inline — always call MaskingUtil.
 */
public final class MaskingUtil {

    private MaskingUtil() {}

    /**
     * PAN masking — PCI-DSS Req 3.4: show only last 4 digits.
     * Input: any of "4111111111111111", "4111-1111-1111-1111"
     * Output: "****-****-****-1111"
     */
    public static String maskPan(String pan) {
        if (!StringUtils.hasText(pan)) return "****-****-****-****";
        String digits = pan.replaceAll("[^0-9]", "");
        if (digits.length() < 4) return "****-****-****-****";
        String last4 = digits.substring(digits.length() - 4);
        return "****-****-****-" + last4;
    }

    /**
     * Aadhaar masking — UIDAI guidelines: show only last 4 digits.
     * Input: "123456789012" or "1234 5678 9012"
     * Output: "XXXX-XXXX-9012"
     */
    public static String maskAadhaar(String aadhaar) {
        if (!StringUtils.hasText(aadhaar)) return "XXXX-XXXX-XXXX";
        String digits = aadhaar.replaceAll("[^0-9]", "");
        if (digits.length() < 4) return "XXXX-XXXX-XXXX";
        String last4 = digits.substring(digits.length() - 4);
        return "XXXX-XXXX-" + last4;
    }

    /**
     * Bank account number masking — RBI standard: show only last 4 digits.
     * Input: "1234567890123456"
     * Output: "XXXXXXXXXXXX3456"
     */
    public static String maskAccountNumber(String accountNumber) {
        if (!StringUtils.hasText(accountNumber)) return "XXXXXXXXXXXX";
        String trimmed = accountNumber.trim();
        if (trimmed.length() <= 4) return "XXXX";
        String last4 = trimmed.substring(trimmed.length() - 4);
        return "X".repeat(trimmed.length() - 4) + last4;
    }

    /**
     * Mobile number masking — show first 2 and last 3 digits.
     * Input: "9876543210"
     * Output: "98XXXXX210"
     */
    public static String maskMobile(String mobile) {
        if (!StringUtils.hasText(mobile)) return "XXXXXXXXXX";
        String digits = mobile.replaceAll("[^0-9]", "");
        if (digits.length() < 5) return "X".repeat(digits.length());
        return digits.substring(0, 2)
                + "X".repeat(digits.length() - 5)
                + digits.substring(digits.length() - 3);
    }

    /**
     * Email masking — show first 2 chars of local part and full domain.
     * Input: "apurva@joshsoftware.com"
     * Output: "ap***@joshsoftware.com"
     */
    public static String maskEmail(String email) {
        if (!StringUtils.hasText(email) || !email.contains("@")) return "***@***.***";
        int atIndex = email.indexOf('@');
        String local = email.substring(0, atIndex);
        String domain = email.substring(atIndex);
        if (local.length() <= 2) return local + "***" + domain;
        return local.substring(0, 2) + "***" + domain;
    }

    /**
     * Insurance policy number masking — show only last 4 chars.
     * Input: "LIC/2024/123456"
     * Output: "**********3456"
     */
    public static String maskPolicyNumber(String policyNumber) {
        if (!StringUtils.hasText(policyNumber)) return "**********";
        String trimmed = policyNumber.trim();
        if (trimmed.length() <= 4) return "****";
        return "*".repeat(trimmed.length() - 4) + trimmed.substring(trimmed.length() - 4);
    }

    /**
     * Generic token / API key masking — show only first 4 and last 4 chars.
     * Input: "sk-abc123def456ghi789"
     * Output: "sk-a...i789"
     */
    public static String maskToken(String token) {
        if (!StringUtils.hasText(token)) return "[REDACTED]";
        if (token.length() <= 8) return "[REDACTED]";
        return token.substring(0, 4) + "..." + token.substring(token.length() - 4);
    }

    /**
     * Password / PIN — always fully redacted, no partial reveal.
     */
    public static String maskPassword(String ignored) {
        return "[REDACTED]";
    }

    /**
     * CVV — always fully redacted (PCI-DSS: must not be stored or logged at all).
     */
    public static String maskCvv(String ignored) {
        return "***";
    }

    /**
     * OTP — always fully redacted (transient secret).
     */
    public static String maskOtp(String ignored) {
        return "[REDACTED]";
    }

    /**
     * IFSC code — partially mask middle characters.
     * Input: "SBIN0001234"
     * Output: "SBIN****234"
     */
    public static String maskIfsc(String ifsc) {
        if (!StringUtils.hasText(ifsc) || ifsc.length() < 7) return "****";
        return ifsc.substring(0, 4) + "****" + ifsc.substring(ifsc.length() - 3);
    }
}
```

---

## Usage in Log Statements

```java
// CORRECT — always mask before logging
log.info("[PaymentService#processPayment] ENTRY - pan={}, amount={}",
        MaskingUtil.maskPan(request.getPan()), request.getAmount());

log.info("[KycService#verifyAadhaar] ENTRY - aadhaar={}",
        MaskingUtil.maskAadhaar(request.getAadhaarNumber()));

log.info("[AccountService#transfer] ENTRY - from={}, to={}",
        MaskingUtil.maskAccountNumber(request.getFromAccount()),
        MaskingUtil.maskAccountNumber(request.getToAccount()));

// WRONG — never log raw sensitive data
log.info("Processing PAN: {}", request.getPan());             // PCI-DSS violation
log.info("Aadhaar: {}", customer.getAadhaarNumber());         // UIDAI violation
log.info("Password: {}", request.getPassword());              // Auth credential leak
```

---

## Masked Response DTO Pattern

For APIs where partial data must be returned (e.g. display last 4 of card on dashboard):

```java
package com.joshsoftware.app.dto.response;

import com.fasterxml.jackson.annotation.JsonIgnore;
import lombok.Builder;
import lombok.Getter;
import com.joshsoftware.app.util.MaskingUtil;

@Getter
@Builder
public class CardSummaryResponseDTO {

    private String id;
    private String cardType;           // VISA, MASTERCARD
    private String maskedPan;          // set at build time via MaskingUtil.maskPan()
    private String expiryMonth;
    private String expiryYear;
    private String cardHolderName;

    // CVV and full PAN are never included in any response DTO — no field declared at all

    public static CardSummaryResponseDTO from(Card card) {
        return CardSummaryResponseDTO.builder()
                .id(card.getId().toString())
                .cardType(card.getCardType().name())
                .maskedPan(MaskingUtil.maskPan(card.getPan()))  // always masked
                .expiryMonth(card.getExpiryMonth())
                .expiryYear(card.getExpiryYear())
                .cardHolderName(card.getCardHolderName())
                .build();
    }
}
```

---

## Logback Masking Filter (Defence-in-Depth)

Adds a last-resort log pattern filter to catch any unmasked PAN that slips through:

```xml
<!-- logback-spring.xml -->
<conversionRule conversionWord="mask"
                converterClass="com.joshsoftware.app.logging.SensitiveDataMaskingConverter"/>

<appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
        <!-- %mask{%msg} applies regex masking to the final log line -->
        <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %mask{%msg}%n</pattern>
    </encoder>
</appender>
```

```java
package com.joshsoftware.app.logging;

import ch.qos.logback.classic.pattern.ClassicConverter;
import ch.qos.logback.classic.spi.ILoggingEvent;

import java.util.regex.Pattern;

/**
 * Logback converter that scrubs PAN-like digit sequences from log output.
 * Defence-in-depth: primary protection is MaskingUtil at the call site.
 */
public class SensitiveDataMaskingConverter extends ClassicConverter {

    // Matches 13-19 consecutive digits (PAN range) — may have spaces or dashes
    private static final Pattern PAN_PATTERN =
            Pattern.compile("\\b(?:\\d[ -]?){12,18}\\d\\b");

    // Matches 12-digit Aadhaar
    private static final Pattern AADHAAR_PATTERN =
            Pattern.compile("\\b\\d{4}[- ]?\\d{4}[- ]?\\d{4}\\b");

    @Override
    public String convert(ILoggingEvent event) {
        String message = event.getFormattedMessage();
        message = PAN_PATTERN.matcher(message).replaceAll("****-****-****-XXXX");
        message = AADHAAR_PATTERN.matcher(message).replaceAll("XXXX-XXXX-XXXX");
        return message;
    }
}
```

---

## Masking Rules Summary

| Data Type           | Mask Format             | Method                        | Never Log/Return          |
|---------------------|-------------------------|-------------------------------|---------------------------|
| PAN                 | `****-****-****-1234`   | `maskPan()`                   | Full PAN                  |
| CVV                 | `***`                   | `maskCvv()`                   | Any digits                |
| Aadhaar             | `XXXX-XXXX-5678`        | `maskAadhaar()`               | Full number               |
| Bank account        | `XXXXXXXXXX1234`        | `maskAccountNumber()`         | Full number               |
| Mobile              | `98XXXXX210`            | `maskMobile()`                | Full number               |
| Email               | `ap***@domain.com`      | `maskEmail()`                 | Full address in some contexts |
| Password / PIN      | `[REDACTED]`            | `maskPassword()`              | Any chars                 |
| OTP                 | `[REDACTED]`            | `maskOtp()`                   | Any digits                |
| API key / JWT token | `sk-a...i789`           | `maskToken()`                 | Full token                |
| IFSC                | `SBIN****234`           | `maskIfsc()`                  | Full code in some contexts |
| Policy number       | `**********3456`        | `maskPolicyNumber()`          | Full number               |