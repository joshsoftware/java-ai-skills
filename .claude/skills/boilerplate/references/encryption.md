# Encryption & Decryption

## Package Structure

```
com.joshsoftware.<app>/
└── security/crypto/
    ├── AesEncryptionService.java    ← symmetric AES-256-GCM
    ├── RsaEncryptionService.java    ← asymmetric RSA-OAEP
    └── EncryptionProperties.java   ← config binding
```

---

## EncryptionProperties

```java
package com.joshsoftware.app.security.crypto;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "app.encryption")
public record EncryptionProperties(
        String aesSecretKey,    // 32-byte Base64-encoded key
        String rsaPublicKey,    // Base64-encoded DER public key
        String rsaPrivateKey    // Base64-encoded DER private key
) {}
```

Enable in main class: `@EnableConfigurationProperties(EncryptionProperties.class)`

```yaml
# application.yml
app:
  encryption:
    aes-secret-key: ${AES_SECRET_KEY}     # 32-byte Base64
    rsa-public-key: ${RSA_PUBLIC_KEY}     # Base64 DER
    rsa-private-key: ${RSA_PRIVATE_KEY}   # Base64 DER
```

---

## AES-256-GCM Encryption Service

```java
package com.joshsoftware.app.security.crypto;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import javax.crypto.Cipher;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.security.SecureRandom;
import java.util.Base64;

/**
 * Symmetric encryption using AES-256-GCM.
 * GCM provides both confidentiality and authenticity (AEAD).
 *
 * Output format: Base64( IV[12 bytes] + CipherText + AuthTag[16 bytes] )
 */
@Slf4j
@Service
public class AesEncryptionService {

    private static final String ALGORITHM    = "AES/GCM/NoPadding";
    private static final int    IV_LENGTH    = 12;   // 96-bit IV recommended for GCM
    private static final int    TAG_LENGTH   = 128;  // 128-bit auth tag

    private final SecretKey secretKey;

    public AesEncryptionService(EncryptionProperties props) {
        byte[] keyBytes = Base64.getDecoder().decode(props.aesSecretKey());
        if (keyBytes.length != 32) {
            throw new IllegalArgumentException(
                    "AES key must be 32 bytes (256 bits). Got: " + keyBytes.length);
        }
        this.secretKey = new SecretKeySpec(keyBytes, "AES");
    }

    /**
     * Encrypts plaintext and returns a Base64-encoded string
     * containing the IV prepended to the ciphertext+tag.
     */
    public String encrypt(String plaintext) {
        try {
            byte[] iv = generateIv();
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, secretKey, new GCMParameterSpec(TAG_LENGTH, iv));

            byte[] encrypted = cipher.doFinal(plaintext.getBytes());
            byte[] combined  = new byte[IV_LENGTH + encrypted.length];

            System.arraycopy(iv,        0, combined, 0,         IV_LENGTH);
            System.arraycopy(encrypted, 0, combined, IV_LENGTH, encrypted.length);

            return Base64.getEncoder().encodeToString(combined);
        } catch (Exception ex) {
            log.error("AES encryption failed", ex);
            throw new RuntimeException("Encryption failed", ex);
        }
    }

    /**
     * Decrypts a Base64-encoded ciphertext produced by {@link #encrypt(String)}.
     */
    public String decrypt(String encryptedBase64) {
        try {
            byte[] combined  = Base64.getDecoder().decode(encryptedBase64);
            byte[] iv        = new byte[IV_LENGTH];
            byte[] cipherText = new byte[combined.length - IV_LENGTH];

            System.arraycopy(combined, 0,         iv,         0, IV_LENGTH);
            System.arraycopy(combined, IV_LENGTH, cipherText, 0, cipherText.length);

            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, secretKey, new GCMParameterSpec(TAG_LENGTH, iv));

            return new String(cipher.doFinal(cipherText));
        } catch (Exception ex) {
            log.error("AES decryption failed", ex);
            throw new RuntimeException("Decryption failed", ex);
        }
    }

    private byte[] generateIv() {
        byte[] iv = new byte[IV_LENGTH];
        new SecureRandom().nextBytes(iv);
        return iv;
    }
}
```

---

## RSA-OAEP Encryption Service

```java
package com.joshsoftware.app.security.crypto;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import javax.crypto.Cipher;
import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.spec.PKCS8EncodedKeySpec;
import java.security.spec.X509EncodedKeySpec;
import java.util.Base64;

/**
 * Asymmetric encryption using RSA-2048 with OAEP-SHA256 padding.
 * Use for small payloads (e.g. encrypting AES keys, tokens, PII fields).
 * For large data, prefer AES-GCM; use RSA only to encrypt the AES key.
 */
@Slf4j
@Service
public class RsaEncryptionService {

    private static final String ALGORITHM = "RSA/ECB/OAEPWithSHA-256AndMGF1Padding";

    private final PublicKey  publicKey;
    private final PrivateKey privateKey;

    public RsaEncryptionService(EncryptionProperties props) {
        try {
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            byte[] pubBytes  = Base64.getDecoder().decode(props.rsaPublicKey());
            byte[] privBytes = Base64.getDecoder().decode(props.rsaPrivateKey());

            this.publicKey  = keyFactory.generatePublic(new X509EncodedKeySpec(pubBytes));
            this.privateKey = keyFactory.generatePrivate(new PKCS8EncodedKeySpec(privBytes));
        } catch (Exception ex) {
            throw new IllegalStateException("Failed to initialise RSA keys", ex);
        }
    }

    /**
     * Encrypts plaintext with the RSA public key.
     *
     * @param plaintext data to encrypt (keep small — RSA-2048/OAEP limit ~190 bytes)
     * @return Base64-encoded ciphertext
     */
    public String encrypt(String plaintext) {
        try {
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, publicKey);
            return Base64.getEncoder().encodeToString(
                    cipher.doFinal(plaintext.getBytes()));
        } catch (Exception ex) {
            log.error("RSA encryption failed", ex);
            throw new RuntimeException("RSA encryption failed", ex);
        }
    }

    /**
     * Decrypts Base64-encoded ciphertext with the RSA private key.
     */
    public String decrypt(String encryptedBase64) {
        try {
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, privateKey);
            return new String(cipher.doFinal(
                    Base64.getDecoder().decode(encryptedBase64)));
        } catch (Exception ex) {
            log.error("RSA decryption failed", ex);
            throw new RuntimeException("RSA decryption failed", ex);
        }
    }
}
```

---

## Encryption Rules

- **AES-256-GCM** for all symmetric encryption — never ECB or CBC
- Always generate a **fresh random IV** per encryption — never reuse an IV with the same key
- **RSA-OAEP-SHA256** for asymmetric — never PKCS#1 v1.5 (vulnerable to padding oracle)
- RSA is for small payloads only (< ~190 bytes for RSA-2048); for larger data use hybrid (encrypt data with AES, encrypt AES key with RSA)
- Keys loaded from **environment variables or Vault** — never hardcoded or committed to source control
- Log encryption **failures** at ERROR; never log plaintext or key material
