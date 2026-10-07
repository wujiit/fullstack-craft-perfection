# Backend Resilience, Parameter Defense & Concurrency Architecture Guide

This guide defines production-grade architectural standards for mission-critical API services, concurrent write operations, external distributed communications, and long-running daemon workers.

---

## 1. Strict DTO Parameter Whitelisting (Mass Assignment Defense)

### Core Principle
Inbound HTTP payloads must never penetrate directly into persistent storage (ORM/SQL) or internal domain models. Every inbound request must pass through a strictly typed **DTO (Data Transfer Object)** or schema validator for type casting and property whitelisting.

### Architectural Pattern (Pseudocode Implementation)
```php
// 1. Strictly typed whitelist DTO
class UpdateUserProfileDTO {
    public int $userId;
    public string $nickname;
    public ?string $bio;

    public static function fromRequest(array $input): self {
        $dto = new self();
        $dto->userId = (int)($input['user_id'] ?? 0);
        $dto->nickname = trim(strip_tags((string)($input['nickname'] ?? '')));
        $dto->bio = isset($input['bio']) ? trim((string)$input['bio']) : null;
        
        // Validation: Block mass assignment attacks (e.g., unauthorized is_admin, role attributes)
        if ($dto->userId <= 0 || mb_strlen($dto->nickname) === 0) {
            throw new InvalidArgumentException("Invalid request payload parameters.");
        }
        return $dto;
    }
}
```

---

## 2. Server-Side Write Idempotency

### Core Principle
Operations modifying financial assets, order lifecycles, account credits, or crucial state machines **must never rely solely on frontend button disabling**. Network timeouts, retries, and concurrent API clients can trigger duplicate executions; idempotency must be guaranteed server-side.

### Three Lines of Defense
1. **Tier 1: In-Flight Mutex Lock**:
   - Extract an `Idempotency-Key` header, or derive a hash from `user_id + action + resource_id`.
   - Acquire a short-lived distributed mutex via Redis: `SET resource_key token NX EX 10`. If acquisition fails, return `409 Conflict: Request is already being processed`.
2. **Tier 2: Database Unique Constraint**:
   - Tables must define a business idempotency column backed by a `UNIQUE KEY` (e.g., `order_no` or `(user_id, biz_token)`).
   - Leverage database engine atomicity to prevent concurrent insertion of duplicate records.
3. **Tier 3: Unidirectional State Machine Guard**:
   - State updates must enforce preconditions:
     ```sql
     UPDATE orders SET status = 'PAID' WHERE id = :id AND status = 'PENDING';
     ```
   - If affected row count is 0, the operation was either previously executed or represents an illegal transition. Return the prior successful state idempotently.

---

## 3. End-to-End Tracing (Trace-ID) & Audit Trails

### Core Principle
Production observability requires tracing any event from its inception. From the moment a request reaches ingress, assign or propagate a globally unique trace identifier.

### Specification
1. **Header Propagation**: Check incoming `X-Trace-Id` headers; if absent, generate `trace-{UUID}-{timestamp}` and echo it back in HTTP response headers.
2. **Structured Log Injection**: All application errors, access logs, and slow queries must include the `trace_id` field:
   ```json
   {
     "trace_id": "trace-8f9a2b-1727960000",
     "level": "ERROR",
     "timestamp": "2026-10-03T21:50:00Z",
     "message": "Third-party payment gateway timeout",
     "context": { "order_id": 10023, "duration_ms": 3002 }
   }
   ```
3. **Audit Trails**: Privilege escalations, financial adjustments, and bulk data exports must be recorded into immutable audit logs (capturing operator, IP, payload diff, timestamp, and Trace-ID).

---

## 4. Exponential Backoff with Random Jitter

### Core Principle
When calling external third-party services or distributed dependencies, blind fixed-interval retries cause thundering herds. Implement **exponential backoff with randomized jitter**.

### Algorithm
$$\text{Delay} = \min\left(\text{MaxDelay}, \text{BaseDelay} \times 2^{\text{Attempt}}\right) + \text{RandomJitter}(0, \text{BaseDelay})$$

```javascript
async function fetchWithRetry(url, options = {}, maxAttempts = 3) {
  const baseDelay = 300; // 300ms base
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fetchWithTimeout(url, options);
    } catch (err) {
      if (attempt === maxAttempts) throw err;
      const jitter = Math.random() * baseDelay;
      const delay = Math.min(3000, baseDelay * Math.pow(2, attempt - 1)) + jitter;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

---

## 5. Graceful Daemon Shutdown

### Core Principle
Message queue consumers, WebSocket workers, and batch scripts must never be abruptly killed (`kill -9`) during deployments or auto-scaling events. They must capture OS termination signals and shut down gracefully.

### Shutdown Protocol
1. **Signal Handling**: Listen for `SIGTERM` (container termination) and `SIGINT` (Ctrl+C).
2. **State Transition**: Switch service state to `SHUTTING_DOWN`, rejecting new incoming requests/messages.
3. **Drain In-Flight Work**: Allow currently executing transactions to conclude (subject to a graceful timeout, e.g., 15s).
4. **Resource Teardown**: Close database connection pools, Redis handles, and file locks cleanly inside `finally` blocks before calling `exit(0)`.

---

## 6. Row-Level Authorization (IDOR / BOLA Defense)

### Core Principle
Insecure Direct Object References (IDOR) allow attackers to manipulate resource IDs to access or modify data belonging to other users. Never query or mutate private user resources based solely on client-supplied primary keys.

### Implementation Pattern (Persistent Owner Binding)
```php
// ❌ Dangerous IDOR Vulnerability: Any user can modify order_id to cancel another user's order
$stmt = $pdo->prepare("UPDATE orders SET status = 'CANCELLED' WHERE id = :id");
$stmt->execute([':id' => $input['order_id']]);

// ✅ Enforce Row-Level Ownership: Bind authenticated user context directly in persistent queries
$stmt = $pdo->prepare("UPDATE orders SET status = 'CANCELLED' WHERE id = :id AND user_id = :userId");
$stmt->execute([
    ':id'     => (int)$input['order_id'],
    ':userId' => (int)$currentUser->id
]);
if ($stmt->rowCount() === 0) {
    // Resource not found or unauthorized (return 404 to avoid ID enumeration)
    throw new ResourceNotFoundOrForbiddenException("Resource not found or unauthorized.");
}
```

---

## 7. Soft Delete Architecture & Uniqueness Integrity

### Core Principle
Core business assets (orders, user profiles, financial ledgers, audit records) must never be physically deleted with `DELETE` in production. Preserve complete historical integrity using timestamped soft deletes (`deleted_at TIMESTAMP NULL`).

### Implementation & Unique Index Design
1. **Uniform Column Standard**: Name soft-delete columns `deleted_at`, defaulting to `NULL` (indicating active/undeleted). Set to `NOW()` upon deletion.
2. **Global Query Filtering**:
   ```sql
   SELECT id, title, updated_at FROM posts WHERE user_id = :userId AND deleted_at IS NULL;
   ```
3. **Unique Key Collision Resolution**:
   - If a business value must remain unique among active records (e.g., `account` username) but may be reused after soft deletion:
   - **Composite Unique Key (MySQL Compatible)**: Create `UNIQUE KEY uk_account_del (account, deleted_at)`. In MySQL/InnoDB, multiple `NULL` values in a unique index do not collide, while active records share `NULL` and deleted records hold unique timestamps.

---

## 8. PII Masking & Sensitive Log Sanitization

### Core Principle
Personally Identifiable Information (PII)—including phone numbers, government IDs, email addresses, and bank accounts—must never be displayed unmasked on user interfaces or leaked in server logs.

### Masking Implementation
```php
class PiiMasker {
    // Phone numbers: Keep first 3 and last 4, mask middle 4 (e.g., 138****1234)
    public static function maskMobile(string $mobile): string {
        return preg_replace('/(\d{3})\d{4}(\d{4})/', '$1****$2', $mobile) ?? '';
    }

    // Email addresses: Keep initial character and domain (e.g., z***y@domain.com)
    public static function maskEmail(string $email): string {
        return preg_replace('/(?<=.).(?=.*@)/u', '*', $email) ?? '';
    }

    // National IDs: Keep first 6 and last 4, mask middle section
    public static function maskIdCard(string $idCard): string {
        return preg_replace('/(\d{6})\d+(\d{4})/', '$1********$2', $idCard) ?? '';
    }
}
```

### Log Filter Masking
Configure centralized log formatters to automatically replace values matching sensitive keys (`password`, `token`, `secret`, `access_key`, `card_no`) with `******`.

---

## 9. Non-Destructive Expand-Contract Migrations

### Core Principle
Production systems are upgraded via rolling deployments across multiple instances; new and old application versions inevitably coexist. Database migrations (DDL) must be re-entrant, idempotent, and backward-compatible.

### Idempotent DDL Guidelines
```sql
CREATE TABLE IF NOT EXISTS sys_configs (
    id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    cfg_key VARCHAR(64) NOT NULL,
    cfg_value TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_key (cfg_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Four-Phase Expand-Contract Methodology
Never execute instantaneous destructive operations (such as dropping or renaming columns) in production:
1. **Phase 1 (Expand)**: Add the new column `new_col` as nullable. Legacy application code continues reading and writing `old_col`.
2. **Phase 2 (Dual-Write & Backfill)**: Deploy the updated application. Write operations write to both `old_col` and `new_col`. Run an asynchronous script to backfill historical rows.
3. **Phase 3 (Contract)**: Shift read queries to `new_col`. Cease writing to `old_col`. Monitor stability.
4. **Phase 4 (Cleanup)**: Drop `old_col` during low-traffic maintenance windows, completing zero-downtime evolution.

---

## 10. Holistic Cohesion, Clean Refactoring & API Realism

### 10.1 Anti-Overengineering (KISS & Anti-YAGNI)
- Seek the most straightforward, flat, and intuitive implementation. If 100 lines solve the problem, never author 500 lines of multi-layer factories or abstract interfaces for hypothetical requirements.

### 10.2 Clean-Cut Refactoring
- When migrating from Solution A to Solution B, completely eliminate Solution A. Never leave both architectures half-implemented and entangled. Purge zombie code and obsolete parameters.

### 10.3 Version Alignment & Zero API Hallucination
- Validate runtime versions and library manifests.
- When calling third-party APIs, strictly adhere to official documentation for URLs, parameters, headers, and schemas. Never invent unverified endpoints or fields.

---

## 11. Blast Radius Audit, Asset Deduplication & Edge-Case Defense

### 11.1 Blast Radius Control
- Reverse-search all references across the codebase before altering public methods or database columns to guarantee backward compatibility.

### 11.2 Pre-Coding Deduplication & Single Source of Truth (SSOT)
- Search existing repositories for utilities before writing new ones. Reusable business logic must have exactly one authoritative implementation.

### 11.3 Edge-Case Simulation
- Rigorously simulate nulls, empty collections, negative numbers, numeric overflows, dropped connections, and concurrency race conditions. Trace bugs back to their root causes rather than applying cosmetic patches.
