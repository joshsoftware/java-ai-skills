---
name: bfsi-sensitive-data
description: >
  Enforces BFSI (Banking, Financial Services, Insurance) standards for sensitive data protection
  in Java Spring Boot applications. Use this skill whenever a user asks to:
  - Handle, store, or transmit PAN, CVV, Aadhaar, bank account numbers, IFSC, or insurance policy numbers
  - Add or review fields for passwords, PINs, API keys, JWT secrets, OAuth tokens, or encryption keys
  - Generate code that logs, serialises, or returns financial or personal data
  - Set up secret/credential management (Vault, AWS Secrets Manager, environment variables)
  - Add field-level encryption for PII or financial data columns
  - Write masking utilities for logs, responses, or audit trails
  - Review existing code for credential leaks, unmasked sensitive fields, or PCI-DSS / RBI violations
  - Create or review any class whose name or fields suggest financial data (Account, Card, Loan, Policy, KYC, Customer, Transaction)
  - Add audit logging for financial operations
  Always use this skill BEFORE generating code for any of the above — do not produce unprotected
  financial or credential-related code without running this skill first.
---

# BFSI Sensitive Data Protection Skill

Enforces BFSI-grade data security for Java Spring Boot services.
Applies to: Banking, Payments, Lending, Insurance, Wealth Management, and any system handling
financial or personally identifiable data regulated under PCI-DSS, RBI, IRDAI, or DPDPA.

---

## CRITICAL DEFAULT BEHAVIOURS — Always Active

These rules apply **without any opt-in confirmation**. Never override them.

### 1. Never Read or Suggest Hardcoded Credentials
- **Never read files** named `.env`, `application-secret.yml`, `application-prod.yml`,
  `credentials.json`, `secrets.properties`, or any file the user identifies as containing live keys.
- If the user pastes a real credential, API key, or secret into the chat: **do not echo it back**,
  warn the user it should be revoked immediately, and replace it with an environment variable reference.
- All generated `application.yml` snippets must use `${ENV_VAR_NAME}` placeholders — never literal values.

### 2. Sensitive Field Auto-Protection
When generating or reviewing any class with BFSI-sensitive fields, automatically apply all of:
- `@JsonIgnore` on password, PIN, CVV, OTP, and private key fields (never expose in API response)
- `@ToString.Exclude` (Lombok) on all sensitive fields (prevent Lombok-generated `toString()` leaks)
- `@SensitiveData` custom annotation (see `references/sensitive-annotations.md`) on PAN, Aadhaar, bank account number, IFSC, and token fields
- Field-level encryption via JPA `AttributeConverter` (see `references/pii-field-encryption.md`) for any column storing PAN, Aadhaar, or account number

### 3. Log Masking — Zero Tolerance
- **Never log** raw PAN, CVV, Aadhaar, passwords, PINs, API keys, JWT tokens, or OTPs
- PAN in logs: show only last 4 digits → `****-****-****-1234`
- Aadhaar in logs: show only last 4 digits → `XXXX-XXXX-5678`
- Bank account in logs: show only last 4 digits → `XXXXXX1234`
- Mobile/email in logs: partial mask → `98XXXXX321`, `ap***@domain.com`
- Use `MaskingUtil` (see `references/data-masking.md`) — never write ad-hoc masking inline

### 4. Never Return Raw Sensitive Data in API Responses
- Response DTOs for financial entities must never include CVV, full PAN, full Aadhaar, PIN, or password fields
- PAN in responses: masked by default unless caller has explicit `ROLE_COMPLIANCE` or `ROLE_ADMIN`
- When in doubt, omit the field from the response DTO entirely

---

## How to Use This Skill

### Step 0 — Ask These Questions Before Generating Anything

When this skill is triggered, ask **each question below in order** and wait for an answer before moving to the next.
Never bundle questions — ask one at a time. Only proceed to code generation once all answers are confirmed.

---

#### Question 1 — Domain
> "What is the **domain** of this feature?"
> Options: **Banking / Payments**, **Lending**, **Insurance**, **Wealth Management**, **KYC / Onboarding**, **Other**

Use the answer to determine the applicable regulatory framework (PCI-DSS for Payments, IRDAI for Insurance, etc.)
and which data types are most likely present. Record the domain — it drives follow-up questions.

---

#### Question 2 — Sensitive Data Types Present
> "Which **sensitive data types** does this feature handle? (select all that apply)"
>
> - [ ] PAN / credit / debit card number
> - [ ] CVV / CVV2
> - [ ] Aadhaar number
> - [ ] Bank account number + IFSC
> - [ ] Password or PIN
> - [ ] OTP (One-Time Password)
> - [ ] JWT secret / API key / OAuth token
> - [ ] Name, DOB, Address (PII)
> - [ ] Mobile number / Email
> - [ ] Insurance policy number
> - [ ] Loan account number
> - [ ] Other (ask user to specify)

Use the answer to determine which protections must be applied (see the Sensitivity Level table below).
**This answer is mandatory** — do not generate any entity, DTO, or service class without it.

---

#### Question 3 — Custom Annotations
> "Do you want to add **custom BFSI annotations** (`@SensitiveData`, `@MaskField`) to mark and drive protection on sensitive fields?"
>
> - **Yes** → ask the follow-up below, then read `references/sensitive-annotations.md`
> - **No** → skip. Still apply `@JsonIgnore` and `@ToString.Exclude` manually without the custom annotation.

**Follow-up if Yes:**
> "Which annotation capabilities do you need?"
> - [ ] `@SensitiveData` — documentation + static-analysis marker on entity/DTO fields
> - [ ] `@MaskField` — Jackson serialiser-level masking in API responses (works with `MaskingSerializer`)
> - [ ] `@Audited` — AOP-based automatic audit log capture on service methods
> - [ ] All three

---

#### Question 4 — Data Masking
> "Do you need a **data masking utility** (`MaskingUtil`) for logs, audit trails, or partial API responses?"
>
> - **Yes** → ask the follow-up below, then read `references/data-masking.md`
> - **No** → skip. Remind user that any logging of sensitive fields without masking is a compliance violation.

**Follow-up if Yes:**
> "Which data types need masking methods generated?"
> - [ ] PAN → `maskPan()` — last 4 digits visible
> - [ ] Aadhaar → `maskAadhaar()` — last 4 digits visible
> - [ ] Bank account number → `maskAccountNumber()` — last 4 digits visible
> - [ ] Mobile number → `maskMobile()` — first 2 + last 3 visible
> - [ ] Email → `maskEmail()` — first 2 chars + domain visible
> - [ ] Insurance policy number → `maskPolicyNumber()` — last 4 visible
> - [ ] API key / JWT token → `maskToken()` — first 4 + last 4 visible
> - [ ] All of the above (generate complete `MaskingUtil`)

> "Do you also want the **Logback defence-in-depth filter** (`SensitiveDataMaskingConverter`) to scrub any unmasked values from log output?"
> - **Yes** → generate the converter + update `logback-spring.xml`
> - **No** → skip

---

#### Question 5 — Secret / Credential Management
> "Do you need **secret management** setup (so credentials never appear in source code)?"
>
> - **Yes** → ask the follow-up below, then read `references/secret-management.md`
> - **No** → skip. Still enforce `${ENV_VAR}` placeholders in any generated `application.yml`.

**Follow-up if Yes:**
> "Which secret management approach will you use?"
> - **Environment variables only** → generate `.env` example + `.gitignore` rules + `@ConfigurationProperties` binding
> - **HashiCorp Vault** → generate `bootstrap.yml` Vault config + Spring Cloud Vault dependency + `SecretRefreshScheduler`
> - **AWS Secrets Manager** → generate `spring.config.import` entry + Spring Cloud AWS dependency
> - **All three (show me all options)** → document all patterns

> "Which secret categories need `@ConfigurationProperties` records generated?"
> - [ ] JWT signing secret + expiration
> - [ ] AES / RSA encryption keys
> - [ ] Payment gateway API key + webhook secret
> - [ ] Database credentials (URL, username, password)
> - [ ] Cloud storage credentials (S3 / GCS)
> - [ ] Other (ask user to specify)

---

#### Question 6 — Field-Level PII Encryption
> "Do you need **field-level encryption** for sensitive columns in the database (JPA `AttributeConverter`)?"
>
> - **Yes** → ask the follow-up below, then read `references/pii-field-encryption.md`
> - **No** → skip. Warn user that storing PAN or Aadhaar without encryption violates PCI-DSS Req 3.5 and UIDAI circular.

**Follow-up if Yes:**
> "Which fields need encrypted database columns?"
> - [ ] PAN → `PanAttributeConverter` + `pan_encrypted TEXT` column
> - [ ] Aadhaar → `AadhaarAttributeConverter` + `aadhaar_encrypted TEXT` column
> - [ ] Bank account number → `AccountNumberConverter` + `account_encrypted TEXT` column
> - [ ] Other PII field (ask user: field name + data type)

> "Should I also generate the **Flyway migration** to add the encrypted columns?"
> - **Yes** → generate `V<n>__add_encrypted_columns.sql` with column definitions and SQL comments
> - **No** → skip

---

#### Question 7 — Audit Logging
> "Do you need **BFSI audit logging** to track who accessed or modified sensitive financial data?"
>
> - **Yes** → ask the follow-up below, then read `references/bfsi-audit-log.md`
> - **No** → skip. Warn user that PCI-DSS Req 10 and RBI IT Framework require audit trails for cardholder data access.

**Follow-up if Yes:**
> "Which financial operations should generate audit log entries?"
> - [ ] View / read sensitive data (PAN, Aadhaar, account number)
> - [ ] Create / Update / Delete financial records
> - [ ] Payment initiation and confirmation
> - [ ] Loan disbursement and repayment
> - [ ] Insurance policy issuance and claims
> - [ ] KYC submission and verification
> - [ ] Login, logout, and failed login
> - [ ] All of the above

> "Should the `@Audited` AOP annotation be used on service methods, or manual `auditLogService.log()` calls?"
> - **AOP annotation** → generate `AuditAspect` + `@Audited` annotation
> - **Manual calls** → generate `AuditLogService` + `AuditLogServiceImpl` only; user places calls manually
> - **Both** → generate everything

> "Should I generate the **Flyway migration** for the `audit_logs` table with append-only DB permissions?"
> - **Yes** → generate `V<n>__create_audit_logs_table.sql` including `REVOKE UPDATE, DELETE`
> - **No** → skip

---

> Only proceed to code generation once **all seven questions** are answered.

---

### Step 1 — Sensitivity Classification Reference

Use this table to verify the mandatory controls for each data type confirmed in Question 2:

| Data Type                          | Sensitivity Level | Mandatory Controls                                      |
|------------------------------------|-------------------|---------------------------------------------------------|
| PAN (credit/debit card number)     | PCI-DSS Critical  | Encrypt at rest, mask in logs/responses, tokenise        |
| CVV / CVV2                         | PCI-DSS Critical  | Never store (PCI-DSS Req 3.2); validate transiently only |
| Aadhaar number                     | UIDAI / DPDPA     | Encrypt at rest, mask in logs, never in plain responses  |
| Bank account number + IFSC         | RBI Sensitive     | Encrypt at rest, mask in logs                           |
| Password / PIN                     | Authentication    | BCrypt hash only; never store or log plain text          |
| JWT secret / API key / OAuth token | Credential        | Env var / Vault only; `@JsonIgnore`; never log          |
| Name, DOB, Address, Email, Mobile  | PII / DPDPA       | Mask in logs; encrypt if stored for compliance          |
| Insurance policy number            | IRDAI Sensitive   | Mask in logs; access-controlled responses               |
| Loan account number                | RBI Sensitive     | Mask in logs; encrypt at rest                           |
| OTP                                | Transient Secret  | Never log, never persist after use, TTL enforced        |

---

### Step 2 — Read Reference Files

Read only the reference files for features the user confirmed in Step 0:

| Confirmed Feature          | Read this file                               |
|----------------------------|----------------------------------------------|
| Custom annotations         | `references/sensitive-annotations.md`       |
| Data masking               | `references/data-masking.md`                |
| Secret management          | `references/secret-management.md`           |
| Field-level encryption     | `references/pii-field-encryption.md`        |
| Audit logging              | `references/bfsi-audit-log.md`              |

**Rules for skipped features:**
- If the user answers **No** to a question: skip that reference file entirely and do not generate any related classes.
- If the user answers **Yes**: read the reference file, generate complete compilable classes, and never leave a feature half-done.
- Never bundle skipped features together — treat each as independently opted out.

---

### Step 3 — Generate Code

Generate **complete, compilable** classes with all protections applied.
Never generate partial protection (e.g., encrypt the field but forget `@JsonIgnore`).
Every generated class must satisfy all mandatory controls from the Sensitivity Classification table for each data type confirmed in Question 2.

---

## Package Structure (additions to base boilerplate)

```
com.joshsoftware.<app>/
├── security/
│   └── crypto/                         ← AES / RSA encryption services (from boilerplate)
├── annotation/
│   ├── SensitiveData.java              ← marks fields as BFSI-sensitive
│   └── MaskField.java                  ← drives log/serialisation masking
├── util/
│   └── MaskingUtil.java                ← centralised masking logic for all sensitive types
├── config/
│   └── SensitiveDataConfig.java        ← binds secret properties from env/Vault
├── audit/
│   ├── AuditLog.java                   ← audit entity
│   ├── AuditLogRepository.java
│   ├── AuditLogService.java
│   └── AuditAspect.java                ← AOP-based audit capture
└── converter/
    ├── PanAttributeConverter.java      ← JPA encrypt/decrypt PAN
    ├── AadhaarAttributeConverter.java  ← JPA encrypt/decrypt Aadhaar
    └── AccountNumberConverter.java     ← JPA encrypt/decrypt account number
```

---

## Naming Conventions

| Artifact                        | Convention                            | Example                           |
|---------------------------------|---------------------------------------|-----------------------------------|
| Custom annotation               | `@PascalCase`                         | `@SensitiveData`, `@MaskField`    |
| JPA converter                   | `<FieldType>AttributeConverter`       | `PanAttributeConverter`           |
| Masking utility method          | `mask<DataType>()`                    | `maskPan()`, `maskAadhaar()`      |
| Audit entity                    | `AuditLog`                            | lives in `audit/` package         |
| Secret config properties class  | `<Domain>SecretProperties`            | `PaymentSecretProperties`         |

---

## BFSI Anti-Patterns (Strictly Forbidden)

| Anti-Pattern                                             | Violation                            |
|----------------------------------------------------------|--------------------------------------|
| `String pan = "4111111111111111";` (hardcoded card)      | PCI-DSS Req 3 + credential leak      |
| `log.info("PAN: {}", pan)`                               | PCI-DSS Req 3.4 + audit failure      |
| `log.info("password: {}", password)`                     | Auth credential exposure             |
| Storing CVV in any database column                       | PCI-DSS Req 3.2 — strictly forbidden |
| Returning full PAN in API response without role check    | PCI-DSS Req 4 + RBI data exposure    |
| `@Value("${db.password}") String dbPassword` logged      | Credential exposure                  |
| `password = request.getPassword()` without hashing       | Plain-text password storage          |
| Aadhaar in log without masking                           | UIDAI circular violation             |
| JWT secret in `application.yml` committed to git         | Credential in source control         |
| Using MD5 or SHA-1 for password hashing                  | Broken hash — use BCrypt / Argon2    |
| Storing OTP longer than its TTL                          | Transient secret persistence         |
| Skipping `@JsonIgnore` on sensitive entity fields        | Schema / secret leak via API         |
| Printing `toString()` of entity with card/account data   | Lombok `@Data` log leak              |

---

## Compliance Reference

| Standard   | Key Requirements Enforced by This Skill                                      |
|------------|------------------------------------------------------------------------------|
| PCI-DSS    | Req 3: No CVV storage; encrypt PAN; tokenise where possible                  |
| PCI-DSS    | Req 4: TLS 1.2+ in transit (enforced at config level)                        |
| PCI-DSS    | Req 10: Audit logging of all access to cardholder data                       |
| RBI        | Sensitive data (account, IFSC) encrypted at rest and masked in transit       |
| UIDAI      | Aadhaar number never stored in plain text; only VID or tokenised form        |
| DPDPA 2023 | PII (name, DOB, email, mobile) protected, minimised, and consent-tracked     |
| IRDAI      | Policy number and health data masked and access-controlled                   |
| ISO 27001  | Secrets managed via Vault/SSM; no credentials in source control              |