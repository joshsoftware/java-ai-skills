# PII Field-Level Encryption — BFSI Standards

## Overview

Field-level encryption stores each sensitive column as ciphertext in the DB.
The application encrypts on write and decrypts on read transparently via JPA `AttributeConverter`.
Uses `AesEncryptionService` from `references/encryption.md` (boilerplate skill).

## Package Location
```
com.joshsoftware.<app>/
└── converter/
    ├── PanAttributeConverter.java
    ├── AadhaarAttributeConverter.java
    ├── AccountNumberConverter.java
    └── EncryptedStringConverter.java   ← generic base for other PII fields
```

---

## Generic Encrypted String Converter (Base)

```java
package com.joshsoftware.app.converter;

import com.joshsoftware.app.security.crypto.AesEncryptionService;
import jakarta.persistence.AttributeConverter;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * Base JPA converter for AES-256-GCM field-level encryption.
 * Subclass for each sensitive column type to make intent explicit.
 *
 * DB stores: Base64(IV + CipherText + AuthTag)
 * Application sees: plain-text string
 */
@Slf4j
@Component
public class EncryptedStringConverter implements AttributeConverter<String, String> {

    @Autowired
    private AesEncryptionService aesEncryptionService;

    @Override
    public String convertToDatabaseColumn(String plaintext) {
        if (plaintext == null) return null;
        try {
            return aesEncryptionService.encrypt(plaintext);
        } catch (Exception ex) {
            log.error("Field encryption failed — blocking write to prevent plain-text storage", ex);
            throw new IllegalStateException("Failed to encrypt sensitive field", ex);
        }
    }

    @Override
    public String convertToEntityAttribute(String ciphertext) {
        if (ciphertext == null) return null;
        try {
            return aesEncryptionService.decrypt(ciphertext);
        } catch (Exception ex) {
            log.error("Field decryption failed", ex);
            throw new IllegalStateException("Failed to decrypt sensitive field", ex);
        }
    }
}
```

---

## PAN Converter

```java
package com.joshsoftware.app.converter;

import jakarta.persistence.Converter;

/**
 * Encrypts/decrypts PAN (Primary Account Number) at the JPA layer.
 * PCI-DSS Requirement 3.5: protect stored cardholder data.
 */
@Converter
public class PanAttributeConverter extends EncryptedStringConverter {}
```

## Aadhaar Converter

```java
package com.joshsoftware.app.converter;

import jakarta.persistence.Converter;

/**
 * Encrypts/decrypts Aadhaar UID at the JPA layer.
 * UIDAI circular: Aadhaar must never be stored in plain text.
 */
@Converter
public class AadhaarAttributeConverter extends EncryptedStringConverter {}
```

## Account Number Converter

```java
package com.joshsoftware.app.converter;

import jakarta.persistence.Converter;

/**
 * Encrypts/decrypts bank account numbers at the JPA layer.
 * RBI guidelines: account numbers classified as sensitive financial data.
 */
@Converter
public class AccountNumberConverter extends EncryptedStringConverter {}
```

---

## DB Migration — Encrypted Column Definitions

Always use `TEXT` (not `VARCHAR`) for encrypted columns — ciphertext is always longer than plaintext.

```sql
-- V2__add_customer_sensitive_columns.sql
ALTER TABLE customers
    ADD COLUMN pan_encrypted       TEXT,
    ADD COLUMN aadhaar_encrypted   TEXT,
    ADD COLUMN account_encrypted   TEXT;

COMMENT ON COLUMN customers.pan_encrypted     IS 'AES-256-GCM encrypted PAN — PCI-DSS Req 3.5';
COMMENT ON COLUMN customers.aadhaar_encrypted IS 'AES-256-GCM encrypted Aadhaar — UIDAI circular';
COMMENT ON COLUMN customers.account_encrypted IS 'AES-256-GCM encrypted bank account — RBI guidelines';
```

---

## Entity Usage

```java
@SensitiveData(type = SensitiveData.SensitiveDataType.PAN)
@Convert(converter = PanAttributeConverter.class)
@Column(name = "pan_encrypted")
@ToString.Exclude
private String pan;

@SensitiveData(type = SensitiveData.SensitiveDataType.AADHAAR)
@Convert(converter = AadhaarAttributeConverter.class)
@Column(name = "aadhaar_encrypted")
@ToString.Exclude
private String aadhaarNumber;

@SensitiveData(type = SensitiveData.SensitiveDataType.BANK_ACCOUNT)
@Convert(converter = AccountNumberConverter.class)
@Column(name = "account_encrypted")
@ToString.Exclude
private String bankAccountNumber;
```

---

## Tokenisation (PCI-DSS Preferred for PAN)

For systems where PAN is used repeatedly (e.g. recurring payments), prefer tokenisation over
raw encrypted PAN. Store the token; call the payment vault for the real PAN only when needed.

```java
package com.joshsoftware.app.service.impl;

import com.joshsoftware.app.annotation.SensitiveData;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

/**
 * Tokenises PAN before persistence — keeps raw PAN out of your DB entirely.
 * Integrate with your payment vault (e.g. Stripe, Razorpay token API).
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class CardTokenisationService {

    private final PaymentVaultClient vaultClient;

    /**
     * Returns an opaque token that represents the card.
     * The token is safe to store and log (it carries no card data).
     *
     * @param pan raw PAN — handled transiently, never persisted
     */
    public String tokenise(@SensitiveData(type = SensitiveData.SensitiveDataType.PAN) String pan) {
        log.info("[CardTokenisationService#tokenise] Tokenising card — pan={}",
                com.joshsoftware.app.util.MaskingUtil.maskPan(pan));
        String token = vaultClient.createToken(pan);
        log.info("[CardTokenisationService#tokenise] Token created successfully");
        return token;
    }

    public String detokenise(String token) {
        // Only called by authorised payment processing path, never for display
        return vaultClient.retrievePan(token);
    }
}
```

---

## Encryption Rules

- AES-256-GCM only — never ECB, never CBC (CBC is vulnerable to padding oracle attacks)
- Fresh random IV per encryption call — never reuse IV with the same key
- CVV **must never be stored** — validate transiently, then discard; no converter, no column
- Key rotation: implement a migration job that re-encrypts all rows when the AES key rotates
- Encrypted columns must be `TEXT` in the DB — ciphertext is larger than plaintext
- All encrypted column names must end with `_encrypted` — makes schema intent obvious
- Test both encrypt and decrypt paths in integration tests against a real test DB