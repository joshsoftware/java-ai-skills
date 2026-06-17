# BFSI Audit Logging — Compliance Standards

## Why Audit Logging
PCI-DSS Requirement 10, RBI IT Framework, and IRDAI guidelines all require an immutable
audit trail for every access to or modification of sensitive financial data.
This audit log is separate from the application debug log — it is a compliance record.

## Package Location
```
com.joshsoftware.<app>/
└── audit/
    ├── AuditLog.java              ← audit entity (append-only)
    ├── AuditLogRepository.java
    ├── AuditLogService.java
    ├── AuditLogServiceImpl.java
    └── AuditAspect.java           ← AOP-based auto-capture
```

---

## AuditLog Entity

```java
package com.joshsoftware.app.audit;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;

import java.time.Instant;
import java.util.UUID;

/**
 * Append-only audit record. Never update or delete rows in this table.
 * Satisfies PCI-DSS Req 10.2, RBI IT audit trail requirements.
 */
@Entity
@Table(name = "audit_logs", indexes = {
        @Index(name = "idx_audit_entity",    columnList = "entity_type, entity_id"),
        @Index(name = "idx_audit_actor",     columnList = "actor_id"),
        @Index(name = "idx_audit_timestamp", columnList = "event_timestamp"),
        @Index(name = "idx_audit_action",    columnList = "action")
})
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AuditLog {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "event_timestamp", nullable = false, updatable = false)
    private Instant eventTimestamp;

    @Column(name = "actor_id", nullable = false, updatable = false)
    private String actorId;           // userId or system service name

    @Column(name = "actor_ip", updatable = false)
    private String actorIp;           // client IP from request context

    @Column(name = "action", nullable = false, updatable = false)
    @Enumerated(EnumType.STRING)
    private AuditAction action;

    @Column(name = "entity_type", nullable = false, updatable = false)
    private String entityType;        // e.g. "CARD", "ACCOUNT", "CUSTOMER"

    @Column(name = "entity_id", nullable = false, updatable = false)
    private String entityId;          // masked/tokenised ID — never raw sensitive data

    @Column(name = "description", updatable = false, length = 1000)
    private String description;       // human-readable summary, no raw PII

    @Column(name = "outcome", nullable = false, updatable = false)
    @Enumerated(EnumType.STRING)
    private AuditOutcome outcome;

    @Column(name = "correlation_id", updatable = false)
    private String correlationId;     // ties to HTTP request tracing ID

    public enum AuditAction {
        VIEW_SENSITIVE_DATA,
        CREATE,
        UPDATE,
        DELETE,
        EXPORT,
        DECRYPT_ACCESS,
        LOGIN,
        LOGOUT,
        FAILED_LOGIN,
        PERMISSION_CHANGE,
        PAYMENT_INITIATE,
        PAYMENT_CONFIRM,
        PAYMENT_REVERSE,
        KYC_SUBMIT,
        KYC_VERIFY,
        POLICY_ISSUE,
        POLICY_CLAIM,
        LOAN_DISBURSE,
        LOAN_REPAY
    }

    public enum AuditOutcome {
        SUCCESS,
        FAILURE,
        PARTIAL
    }
}
```

---

## AuditLogRepository

```java
package com.joshsoftware.app.audit;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

import java.time.Instant;
import java.util.UUID;

public interface AuditLogRepository extends JpaRepository<AuditLog, UUID> {

    Page<AuditLog> findByEntityTypeAndEntityId(String entityType, String entityId, Pageable pageable);

    Page<AuditLog> findByActorId(String actorId, Pageable pageable);

    Page<AuditLog> findByEventTimestampBetween(Instant from, Instant to, Pageable pageable);
}
```

---

## AuditLogService

```java
package com.joshsoftware.app.audit;

public interface AuditLogService {
    void log(AuditLog.AuditAction action,
             String entityType,
             String entityId,
             String description,
             AuditLog.AuditOutcome outcome);
}
```

---

## AuditLogServiceImpl

```java
package com.joshsoftware.app.audit;

import jakarta.servlet.http.HttpServletRequest;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import java.time.Instant;

@Slf4j
@Service
@RequiredArgsConstructor
public class AuditLogServiceImpl implements AuditLogService {

    private final AuditLogRepository auditLogRepository;

    @Override
    @Transactional(propagation = Propagation.REQUIRES_NEW)  // audit always commits, even if main tx rolls back
    public void log(AuditLog.AuditAction action,
                    String entityType,
                    String entityId,
                    String description,
                    AuditLog.AuditOutcome outcome) {

        String actorId = resolveActorId();
        String actorIp = resolveActorIp();
        String correlationId = resolveCorrelationId();

        AuditLog entry = AuditLog.builder()
                .eventTimestamp(Instant.now())
                .actorId(actorId)
                .actorIp(actorIp)
                .action(action)
                .entityType(entityType)
                .entityId(entityId)
                .description(description)
                .outcome(outcome)
                .correlationId(correlationId)
                .build();

        auditLogRepository.save(entry);
        log.info("[AuditLog] action={} entity={}:{} actor={} outcome={}",
                action, entityType, entityId, actorId, outcome);
    }

    private String resolveActorId() {
        try {
            var auth = SecurityContextHolder.getContext().getAuthentication();
            return (auth != null && auth.isAuthenticated()) ? auth.getName() : "SYSTEM";
        } catch (Exception ex) {
            return "UNKNOWN";
        }
    }

    private String resolveActorIp() {
        try {
            var attrs = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            if (attrs == null) return "INTERNAL";
            HttpServletRequest request = attrs.getRequest();
            String forwarded = request.getHeader("X-Forwarded-For");
            return (forwarded != null) ? forwarded.split(",")[0].trim() : request.getRemoteAddr();
        } catch (Exception ex) {
            return "UNKNOWN";
        }
    }

    private String resolveCorrelationId() {
        try {
            var attrs = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            if (attrs == null) return null;
            return attrs.getRequest().getHeader("X-Correlation-ID");
        } catch (Exception ex) {
            return null;
        }
    }
}
```

---

## @Audited Annotation (AOP-Based Auto-Capture)

```java
package com.joshsoftware.app.annotation;

import com.joshsoftware.app.audit.AuditLog;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Triggers automatic audit log entry via AuditAspect.
 * Place on service methods that access or modify sensitive financial data.
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Audited {
    AuditLog.AuditAction action();
    String entityType();
    String description() default "";
}
```

---

## AuditAspect

```java
package com.joshsoftware.app.audit;

import com.joshsoftware.app.annotation.Audited;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Slf4j
@Aspect
@Component
@RequiredArgsConstructor
public class AuditAspect {

    private final AuditLogService auditLogService;

    @Around("@annotation(audited)")
    public Object auditMethod(ProceedingJoinPoint joinPoint, Audited audited) throws Throwable {
        String entityId = resolveEntityId(joinPoint.getArgs());
        try {
            Object result = joinPoint.proceed();
            auditLogService.log(
                    audited.action(),
                    audited.entityType(),
                    entityId,
                    audited.description(),
                    AuditLog.AuditOutcome.SUCCESS
            );
            return result;
        } catch (Exception ex) {
            auditLogService.log(
                    audited.action(),
                    audited.entityType(),
                    entityId,
                    "FAILED: " + ex.getClass().getSimpleName(),
                    AuditLog.AuditOutcome.FAILURE
            );
            throw ex;
        }
    }

    private String resolveEntityId(Object[] args) {
        if (args == null || args.length == 0) return "N/A";
        Object first = args[0];
        if (first == null) return "N/A";
        // UUID or String IDs are safe to log — never a full entity object with PII
        return (first instanceof java.util.UUID || first instanceof String)
                ? first.toString() : first.getClass().getSimpleName();
    }
}
```

---

## Usage on Service Methods

```java
@Audited(
    action = AuditLog.AuditAction.VIEW_SENSITIVE_DATA,
    entityType = "CUSTOMER",
    description = "Fetched customer PAN for compliance review"
)
public CustomerPanResponseDTO getCustomerPan(UUID customerId) {
    log.debug("[CustomerServiceImpl#getCustomerPan] ENTRY - customerId={}", customerId);
    // ...
}

@Audited(
    action = AuditLog.AuditAction.PAYMENT_INITIATE,
    entityType = "PAYMENT"
)
@Transactional
public PaymentResponseDTO initiatePayment(InitiatePaymentRequestDTO request) {
    // ...
}
```

---

## DB Migration — Audit Table

```sql
-- V3__create_audit_logs_table.sql
CREATE TABLE audit_logs (
    id               UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    event_timestamp  TIMESTAMPTZ NOT NULL,
    actor_id         VARCHAR(255) NOT NULL,
    actor_ip         VARCHAR(45),
    action           VARCHAR(50)  NOT NULL,
    entity_type      VARCHAR(100) NOT NULL,
    entity_id        VARCHAR(255) NOT NULL,
    description      VARCHAR(1000),
    outcome          VARCHAR(20)  NOT NULL,
    correlation_id   VARCHAR(255)
);

-- Append-only: revoke UPDATE and DELETE from application DB user
REVOKE UPDATE, DELETE ON audit_logs FROM app_user;
```

---

## Audit Rules

- Audit table is **append-only** — revoke `UPDATE`/`DELETE` from the application DB user
- Use `Propagation.REQUIRES_NEW` — audit entry must commit even when the main transaction rolls back
- `entityId` in audit records must be a UUID or masked value — never raw PAN, Aadhaar, or account number
- Retain audit logs for **minimum 5 years** (PCI-DSS Req 10.7 / RBI IT Framework)
- Audit log must capture: who (actorId + IP), what (action + entity), when (timestamp), result (outcome)
- `X-Correlation-ID` header must be propagated from gateway to service to audit record for full traceability