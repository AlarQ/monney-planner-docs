# Money Planner MVP - P0 Launch Blockers Implementation Plan

**Total Effort:** 110-142 hours (3-4 weeks)
**Critical Path:** Email service → Password reset → Email verification
**Parallel Workstreams:** Secrets, health checks, legal docs, rate limiting
**Architecture Note:** All frontend requests go through BFF (port 8082) → Backend services

---

## Implementation Roadmap

### Week 1: Foundation (Parallel Workstreams)
1. ✅ Email Service Integration (8-10h) - COMPLETED 2025-12-24
2. Secrets Management (6-8h)
3. Health Checks Enhancement (4-6h)

### Week 2: User Experience (Sequential)
4. Password Reset Flow (15-18h) - requires email, includes BFF
5. Email Verification (12-15h) - requires email, includes BFF
6. Transaction Deduplication (8-10h)

### Week 3: Security & Compliance
7. Rate Limiting Middleware (10-12h)
8. Legal Documents + UI (8-10h) - includes BFF
9. OWASP Top 10 Assessment (12-16h)

### Week 4: Infrastructure & Final
10. GDPR Endpoints (12-15h) - includes BFF
11. Database Backup Automation (6-8h)
12. Circuit Breaker for OpenAI (6-8h)

---

## P0A: User Experience Blockers

### 1. Email Service Integration ✅ COMPLETED
**Effort:** 8-10 hours | **Dependencies:** None | **Blocks:** Password reset, email verification
**Status:** ✅ Implemented in PR #23 | **Branch:** `feat/user-service-email-integration`
**Completion Date:** 2025-12-24

**New Dependencies:**
- `aws-sdk-ses = "1.14"` (production)
- `lettre = "0.11"` (development SMTP)

**Files Created:** ✅
- `/user-service/src/domain/email/mod.rs` - EmailProvider trait + free functions
- `/user-service/src/domain/email/entities.rs` - EmailAddress + EmailTemplate value objects
- `/user-service/src/repositories/email/mock_provider.rs` - Mock implementation (default)
- `/user-service/src/repositories/email/ses_provider.rs` - AWS SES implementation
- `/user-service/src/repositories/email/smtp_provider.rs` - SMTP implementation (Lettre)
- `/user-service/templates/password-reset.html` - Password reset email template
- `/user-service/templates/email-verification.html` - Email verification template
- `/user-service/templates/welcome.html` - Welcome email template

**Implementation Steps:** ✅ All Complete
1. ✅ Define `EmailProvider` trait in `domain/email/mod.rs` (following user-service pattern)
2. ✅ Create `EmailAddress` value object with security validation (LDAP, null bytes, format)
3. ✅ Implement `SesEmailProvider` with AWS SDK (aws-sdk-ses 1.14)
4. ✅ Implement `SmtpEmailProvider` with Lettre (0.11, tokio1 support)
5. ✅ Implement `MockEmailProvider` for development/testing
6. ✅ Create HTML email templates with professional styling
7. ✅ Add email configuration to `AppConfig` with EmailProviderType enum
8. ✅ Update AppState to include email_provider
9. ✅ Integration tests updated and passing (131 tests total)

**Configuration:**
```rust
// Add to user-service/src/config.rs
pub struct EmailConfig {
    pub provider: EmailProviderType, // Ses or Smtp
    pub from_address: String,
    pub from_name: String,
    pub aws_region: Option<String>,
    pub smtp_host: Option<String>,
    pub smtp_port: Option<u16>,
}
```

**Testing:** ✅ Complete
- ✅ 23 unit tests for domain layer (EmailAddress, EmailTemplate)
- ✅ 7 unit tests for MockEmailProvider
- ✅ Integration tests updated with email provider (84 tests passing)
- ✅ All 131 tests passing
- ⚠️ SMTP provider: Not yet tested with real server
- ⚠️ SES provider: Not yet tested with AWS/localstack

**Implementation Notes:**
- Mock provider is development default (no external dependencies)
- Template paths: `templates/*.html` (relative to service root)
- Configuration via `USER_SERVICE__EMAIL__*` environment variables
- Optional features: `email-mock`, `email-smtp`, `email-ses`
- AppState signature updated (breaking change for tests - fixed)

**Remaining Risks:**
- AWS SES requires domain verification (DNS setup)
- Sending limits in sandbox mode (50 emails/day until production access)
- SMTP and SES providers not yet tested in real environments

**PR:** https://github.com/AlarQ/user-service/pull/23

---

### 2. Password Reset Flow ⭐ CRITICAL
**Effort:** 15-18 hours | **Dependencies:** Email service | **Blocks:** User recovery
**BFF Integration:** Required (frontend → BFF → user-service)

**Database Migration:**
```sql
-- /user-service/migrations/20250122_000001_password_reset_tokens.sql
CREATE TABLE password_reset_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL,
    used BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE(token_hash)
);

CREATE INDEX idx_reset_tokens_user ON password_reset_tokens(user_id);
CREATE INDEX idx_reset_tokens_expiry ON password_reset_tokens(expires_at) WHERE NOT used;
```

**Files to Create:**

**User Service (Implementation Layer):**
- `/user-service/src/domain/password_reset/reset_service.rs` - Domain logic
  - `initiate_password_reset(email: &EmailAddress) -> Result<ResetToken>`
  - `verify_reset_token(token: &str) -> Result<UserId>`
  - `complete_password_reset(token: &str, new_password: &str) -> Result<()>`
- `/user-service/src/domain/interfaces/reset_token_repository.rs` - Repository trait
- `/user-service/src/infrastructure/repositories/postgres_reset_token_repo.rs` - Implementation
- `/user-service/src/api/handlers/password_reset.rs` - API handlers

**BFF Service (Routing Layer):**
- `/money-planner-bff/src/api/routes/password_reset.rs` - Password reset proxying
- `/money-planner-bff/src/services/user_service_client.rs` - Add methods:
  - `forgot_password(email: &str) -> Result<()>`
  - `verify_reset_token(token: &str) -> Result<bool>`
  - `reset_password(token: &str, new_password: &str) -> Result<()>`

**Frontend Files:**
- `/MoneyPlannerFE/src/pages/ForgotPassword.tsx` - Request reset form
- `/MoneyPlannerFE/src/pages/ResetPassword.tsx` - New password form

**API Endpoints:**
```
Frontend calls BFF:
POST   http://localhost:8082/v1/auth/forgot-password      { email: string }
GET    http://localhost:8082/v1/auth/reset-password/verify/:token  → { valid: bool }
POST   http://localhost:8082/v1/auth/reset-password       { token: string, new_password: string }

BFF proxies to User Service:
POST   http://localhost:8080/v1/auth/forgot-password
GET    http://localhost:8080/v1/auth/reset-password/verify/:token
POST   http://localhost:8080/v1/auth/reset-password
```

**Request Flow:**
```
1. Frontend (port 3000) → BFF (port 8082)
2. BFF validates request → User Service (port 8080)
3. User Service sends email via EmailProvider
4. BFF returns response → Frontend
```

**Security Implementation:**
- Generate crypto-secure 32-byte random token
- Hash token with Argon2 before database storage
- 1-hour expiration from creation
- Constant-time comparison for token verification
- Email enumeration prevention (always return 200 OK)
- Rate limiting: 3 requests per day per email address

**Testing:**
- Unit tests: Token generation, validation, expiration
- Integration tests: Full reset flow
- Security tests: Timing attack prevention, expired token rejection

---

### 3. Email Verification Post-Registration
**Effort:** 12-15 hours | **Dependencies:** Email service
**BFF Integration:** Required (frontend → BFF → user-service)

**Database Migration:**
```sql
-- /user-service/migrations/20250122_000002_email_verification.sql
ALTER TABLE users ADD COLUMN email_verified BOOLEAN NOT NULL DEFAULT FALSE;

CREATE TABLE email_verification_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL,
    used BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE(token_hash)
);

CREATE INDEX idx_verification_tokens_user ON email_verification_tokens(user_id);
```

**Files to Create:**

**User Service (Implementation Layer):**
- `/user-service/src/domain/user/verification_service.rs` - Verification logic
- `/user-service/src/api/handlers/email_verification.rs` - API handlers

**Files to Modify:**
- `/user-service/src/domain/user/entities.rs` - Add `email_verified` field to User
- `/user-service/src/api/handlers/registration.rs` - Send verification email on signup

**BFF Service (Routing Layer):**
- `/money-planner-bff/src/api/routes/email_verification.rs` - Email verification proxying
- `/money-planner-bff/src/services/user_service_client.rs` - Add methods:
  - `verify_email(token: &str) -> Result<()>`
  - `resend_verification(user_id: Uuid) -> Result<()>`

**Frontend Files:**
- `/MoneyPlannerFE/src/components/auth/EmailVerificationBanner.tsx` - Warning banner
- `/MoneyPlannerFE/src/pages/VerifyEmail.tsx` - Verification confirmation page

**API Endpoints:**
```
Frontend calls BFF:
GET    http://localhost:8082/v1/auth/verify-email/:token
POST   http://localhost:8082/v1/auth/resend-verification

BFF proxies to User Service:
GET    http://localhost:8080/v1/auth/verify-email/:token
POST   http://localhost:8080/v1/auth/resend-verification
```

**Request Flow:**
```
Registration: Frontend → BFF → User Service → Send verification email
Verification: User clicks link → Frontend → BFF → User Service → Mark verified
Resend: Frontend (authenticated) → BFF → User Service → Send email
```

**Implementation:**
- Send verification email immediately after registration
- 24-hour token expiration
- Resend throttling: 1 per 5 minutes per user
- Show banner until verified

**Testing:**
- Integration tests for verification flow
- Resend throttling tests
- Expired token handling

---

### 4. Transaction Deduplication for CSV Imports
**Effort:** 8-10 hours | **Dependencies:** None

**Database Migration:**
```sql
-- /transaction-service/migrations/20250122_000001_add_content_hash.sql
ALTER TABLE transactions ADD COLUMN content_hash VARCHAR(64);
CREATE INDEX idx_transactions_hash ON transactions(user_id, content_hash);
```

**Files to Create:**
- `/transaction-service/src/domain/transaction/deduplication.rs` - Hash calculation logic

**Files to Modify:**
- `/transaction-service/src/domain/transaction/csv_services.rs` - Add duplicate detection
- `/transaction-service/src/domain/transaction/entities.rs` - Add content_hash field

**Frontend Files:**
- `/MoneyPlannerFE/src/components/transactions/import/DuplicatesList.tsx` - Show duplicates

**Implementation:**
```rust
// Deduplication hash: SHA-256(user_id + date + amount + description)
pub fn calculate_content_hash(
    user_id: &Uuid,
    date: &NaiveDate,
    amount: &Decimal,
    description: &str,
) -> String {
    use sha2::{Sha256, Digest};
    let mut hasher = Sha256::new();
    hasher.update(user_id.as_bytes());
    hasher.update(date.to_string().as_bytes());
    hasher.update(amount.to_string().as_bytes());
    hasher.update(description.as_bytes());
    format!("{:x}", hasher.finalize())
}
```

**CSV Import Flow:**
1. Calculate hash for each imported transaction
2. Query existing transactions by hash
3. Show duplicate count to user
4. Allow user to skip or force import

**Testing:**
- Unit tests for hash calculation
- Integration tests with duplicate data
- CSV import with mixed new/duplicate transactions

---

## P0B: Legal & Compliance

### 5. Terms of Service & Privacy Policy ⭐ CRITICAL
**Effort:** 8-10 hours | **Dependencies:** None | **LEGAL REVIEW REQUIRED**
**BFF Integration:** Required (frontend → BFF → user-service)

**Database Migration:**
```sql
-- /user-service/migrations/20250122_000003_legal_documents.sql
CREATE TABLE legal_document_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_type VARCHAR(50) NOT NULL,
    version VARCHAR(20) NOT NULL,
    effective_date TIMESTAMPTZ NOT NULL,
    content TEXT NOT NULL,
    is_current BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(document_type, version)
);

CREATE TABLE legal_acceptances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    document_id UUID NOT NULL REFERENCES legal_document_versions(id),
    accepted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ip_address VARCHAR(45),
    user_agent TEXT
);

CREATE INDEX idx_legal_acceptances_user ON legal_acceptances(user_id);
```

**Files to Create:**

**User Service (Implementation Layer):**
- `/user-service/src/domain/legal/acceptance_service.rs` - Acceptance tracking logic
- `/user-service/src/domain/interfaces/legal_repository.rs` - Repository trait
- `/user-service/src/api/handlers/legal.rs` - Legal document handlers
- `/user-service/legal/terms-of-service-v1.0.md` - ToS content ⚠️ NEEDS LEGAL REVIEW
- `/user-service/legal/privacy-policy-v1.0.md` - Privacy policy ⚠️ NEEDS LEGAL REVIEW

**Files to Modify:**
- `/user-service/src/api/handlers/registration.rs` - Require acceptance during signup

**BFF Service (Routing Layer):**
- `/money-planner-bff/src/api/routes/legal.rs` - Legal document proxying
- `/money-planner-bff/src/services/user_service_client.rs` - Add methods:
  - `get_terms_of_service() -> Result<LegalDocument>`
  - `get_privacy_policy() -> Result<LegalDocument>`
  - `accept_legal_document(document_type, version, ip, user_agent) -> Result<()>`

**Frontend Files:**
- `/MoneyPlannerFE/src/pages/TermsOfService.tsx` - ToS page
- `/MoneyPlannerFE/src/pages/PrivacyPolicy.tsx` - Privacy page
- `/MoneyPlannerFE/src/components/auth/LegalAcceptanceCheckbox.tsx` - Signup checkbox

**API Endpoints:**
```
Frontend calls BFF:
GET    http://localhost:8082/v1/legal/terms-of-service
GET    http://localhost:8082/v1/legal/privacy-policy
POST   http://localhost:8082/v1/legal/accept              { document_type, version, ip, user_agent }

BFF proxies to User Service:
GET    http://localhost:8080/v1/legal/terms-of-service
GET    http://localhost:8080/v1/legal/privacy-policy
POST   http://localhost:8080/v1/legal/accept
```

**Critical:** Documents MUST be reviewed by legal counsel before production

---

### 6. OWASP Top 10 Security Assessment ⭐ CRITICAL
**Effort:** 12-16 hours | **Dependencies:** None

**Files to Create:**
- `/user-service/src/api/middleware/security_headers.rs` - Security headers middleware
- `/user-service/src/api/middleware/authorization.rs` - Resource ownership validation
- `/.github/workflows/security-audit.yml` - Automated security scanning
- `/.github/dependabot.yml` - Dependency updates

**Files to Modify:**
- All services: Add security headers to server configuration
- All services: Implement resource authorization checks

**Security Headers Implementation:**
```rust
// Add to all services
pub async fn security_headers_middleware(
    req: Request<Body>,
    next: Next<Body>,
) -> Response {
    let mut response = next.run(req).await;
    response.headers_mut().insert(
        "Strict-Transport-Security",
        "max-age=31536000; includeSubDomains".parse().unwrap()
    );
    response.headers_mut().insert("X-Content-Type-Options", "nosniff".parse().unwrap());
    response.headers_mut().insert("X-Frame-Options", "DENY".parse().unwrap());
    response.headers_mut().insert(
        "Content-Security-Policy",
        "default-src 'self'".parse().unwrap()
    );
    response
}
```

**OWASP Checklist:**
- ✅ A01 - Broken Access Control: Resource ownership middleware
- ✅ A02 - Cryptographic Failures: Secure cookies (HttpOnly, Secure, SameSite)
- ✅ A03 - Injection: SQLx parameterized queries (already done)
- ✅ A04 - Insecure Design: Failed login tracking
- ✅ A05 - Security Misconfiguration: Security headers
- ✅ A06 - Vulnerable Components: Dependabot + cargo audit
- ✅ A07 - Authentication Failures: Account lockout (5 attempts)
- ✅ A08 - Software/Data Integrity: Dependency verification
- ✅ A09 - Logging Failures: Security event logging
- ✅ A10 - SSRF: Input validation on URLs

**Testing:**
- OWASP ZAP scan on all endpoints
- Manual penetration testing
- Cargo audit in CI pipeline

---

### 7. GDPR Compliance Endpoints
**Effort:** 12-15 hours | **Dependencies:** None | **Legal requirement for EU users**
**BFF Integration:** Required (frontend → BFF → user-service → other services)

**Files to Create:**

**User Service (Orchestration Layer):**
- `/user-service/src/domain/gdpr/data_export.rs` - Cross-service export orchestration
- `/user-service/src/domain/gdpr/data_deletion.rs` - Cascade deletion logic
- `/user-service/src/api/handlers/gdpr.rs` - GDPR API handlers

**Transaction Service:**
- `/transaction-service/src/domain/gdpr/data_management.rs` - Transaction data handling
- `/transaction-service/src/api/handlers/gdpr.rs` - Internal GDPR endpoints

**Budget Service:**
- `/budget-service/src/domain/gdpr/data_management.rs` - Budget data handling
- `/budget-service/src/api/handlers/gdpr.rs` - Internal GDPR endpoints

**BFF Service (Routing Layer):**
- `/money-planner-bff/src/api/routes/gdpr.rs` - GDPR endpoint proxying
- `/money-planner-bff/src/services/user_service_client.rs` - Add methods:
  - `request_data_export(user_id: Uuid) -> Result<ExportJob>`
  - `get_export_status(job_id: Uuid) -> Result<ExportStatus>`
  - `request_account_deletion(user_id: Uuid) -> Result<DeletionSchedule>`
  - `cancel_account_deletion(user_id: Uuid) -> Result<()>`
  - `get_deletion_status(user_id: Uuid) -> Result<DeletionStatus>`

**API Endpoints:**
```
Frontend calls BFF:
POST   http://localhost:8082/v1/gdpr/export                → { job_id, status }
GET    http://localhost:8082/v1/gdpr/export/:job_id        → { download_url, expires_at }
POST   http://localhost:8082/v1/gdpr/delete-account        → { scheduled_deletion_date }
POST   http://localhost:8082/v1/gdpr/cancel-deletion       → { cancelled: true }
GET    http://localhost:8082/v1/gdpr/deletion-status       → { scheduled, date }

BFF proxies to User Service (which orchestrates other services):
POST   http://localhost:8080/v1/gdpr/export
GET    http://localhost:8080/v1/gdpr/export/:job_id
POST   http://localhost:8080/v1/gdpr/delete-account
POST   http://localhost:8080/v1/gdpr/cancel-deletion
GET    http://localhost:8080/v1/gdpr/deletion-status

User Service calls other services (internal):
GET    http://localhost:8081/internal/gdpr/user/:user_id  (Transaction Service)
GET    http://localhost:8083/internal/gdpr/user/:user_id  (Budget Service)
```

**Data Export Format (JSON):**
```json
{
  "user": { "email": "...", "created_at": "..." },
  "transactions": [ { "id": "...", "amount": "..." } ],
  "budgets": [ { "id": "...", "period": "..." } ],
  "categories": [ { "name": "..." } ],
  "ai_categorization_jobs": [ { "id": "...", "status": "..." } ]
}
```

**Deletion Implementation:**
- 30-day grace period before permanent deletion
- Cancellation allowed during grace period
- Cascade deletion across all services
- Mark account as "pending_deletion" state
- CronJob for permanent deletion after 30 days

**Testing:**
- Integration tests for complete data export
- Cascade deletion tests across services
- Cancellation flow tests

---

## P0C: Infrastructure Blockers

### 8. Health Check Endpoints Enhancement ⭐ CRITICAL
**Effort:** 4-6 hours | **Dependencies:** None | **Kubernetes required**

**Files to Create:**
- `/user-service/src/domain/health/checks.rs` - Database health check
- `/transaction-service/src/domain/health/checks.rs` - DB + Kafka + OpenAI checks
- `/budget-service/src/domain/health/checks.rs` - DB + Kafka checks

**Files to Modify:**
- All services: Enhance existing `/health` and `/ready` endpoints

**Health Check Response:**
```json
{
  "status": "healthy",
  "version": "0.1.0",
  "checks": {
    "database": {
      "status": "healthy",
      "latency_ms": 5
    },
    "kafka": {
      "status": "healthy",
      "connected": true
    },
    "openai": {
      "status": "healthy",
      "circuit_breaker": "closed"
    }
  }
}
```

**Kubernetes Probe Configuration:**
```yaml
# Add to all service deployments
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 30
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 2
```

**Health Check Logic:**
- `/health`: Basic liveness (process alive)
- `/ready`: Readiness with dependency checks (database latency < 100ms)

---

### 9. Rate Limiting Middleware ⭐ CRITICAL
**Effort:** 10-12 hours | **Dependencies:** None

**New Dependencies:**
- `tower-governor = "0.4"` (rate limiting)

**Files to Create:**
- `/user-service/src/api/middleware/rate_limiter.rs` - Rate limiting logic
- `/transaction-service/src/api/middleware/rate_limiter.rs` - Same pattern
- `/budget-service/src/api/middleware/rate_limiter.rs` - Same pattern
- `/money-planner-bff/src/api/middleware/rate_limiter.rs` - BFF limits

**Rate Limit Configuration:**
```rust
// Per-user limits (authenticated requests)
const USER_RATE_LIMIT: u32 = 100; // requests per minute

// Auth endpoint limits (per IP)
const AUTH_RATE_LIMIT: u32 = 10; // requests per minute

// Global limits
const GLOBAL_RATE_LIMIT: u32 = 1000; // requests per second
```

**Implementation:**
```rust
use tower_governor::{GovernorLayer, GovernorConfigBuilder};

let governor_conf = Box::new(
    GovernorConfigBuilder::default()
        .per_second(60)
        .burst_size(100)
        .finish()
        .unwrap(),
);

let governor_limiter = governor_conf.limiter().clone();
let governor_layer = GovernorLayer {
    config: Box::leak(governor_conf),
};
```

**HTTP Response Headers:**
```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1642694400
Retry-After: 30
```

**Testing:**
- Unit tests for rate limit calculation
- Integration tests exceeding limits
- Per-user vs per-IP isolation

**Future Enhancement:** Migrate to Redis for distributed rate limiting (multi-pod support)

---

### 10. Secrets Management ⭐ CRITICAL
**Effort:** 6-8 hours | **Dependencies:** None | **Security blocker**

**Current Problem:** Hardcoded secrets in all `run.sh` files
- JWT secrets
- OpenAI API keys
- Database passwords

**Files to Create:**
- `/user-service/.env.example`
- `/transaction-service/.env.example`
- `/budget-service/.env.example`
- `/money-planner-bff/.env.example`
- `/scripts/generate-secrets.sh` - Secret generation helper
- `/helm/templates/user-service-secret.yaml`
- `/helm/templates/transaction-service-secret.yaml`
- `/helm/templates/budget-service-secret.yaml`
- `/helm/templates/bff-secret.yaml`

**Files to Modify:**
- All `run.sh` files - Remove hardcoded secrets, source `.env`
- `.gitignore` - Add `*.env` (not `.env.example`)

**Example .env.example:**
```bash
# User Service Environment Variables
USER_SERVICE__JWT_SECRET="CHANGE-ME-generate-with-openssl-rand-base64-32"
USER_SERVICE__DATABASE_URL="postgresql://postgres:postgres@localhost:5432/postgres"
USER_SERVICE__EMAIL__PROVIDER="smtp"
USER_SERVICE__EMAIL__FROM_ADDRESS="noreply@moneyplanner.local"
USER_SERVICE__EMAIL__AWS_REGION="us-east-1"
```

**Secret Generation Script:**
```bash
#!/bin/bash
# scripts/generate-secrets.sh

echo "=== Money Planner Secret Generation ==="
echo ""
echo "JWT_SECRET=$(openssl rand -base64 32)"
echo "OPENAI_API_KEY=sk-... (get from OpenAI dashboard)"
echo "AWS_SES_ACCESS_KEY=... (get from AWS IAM)"
echo "AWS_SES_SECRET_KEY=... (get from AWS IAM)"
```

**Kubernetes Secrets (Base64 encoded):**
```yaml
# helm/templates/user-service-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secrets
type: Opaque
data:
  jwt-secret: {{ .Values.userService.jwtSecret | b64enc }}
  database-url: {{ .Values.userService.databaseUrl | b64enc }}
```

**Testing:**
- Verify services start with `.env` files
- Verify Kubernetes pods mount secrets correctly
- Security scan: No secrets in git history

---

### 11. Database Backup Automation
**Effort:** 6-8 hours | **Dependencies:** S3 bucket or equivalent

**Files to Create:**
- `/scripts/backup/backup-databases.sh` - pg_dump script
- `/scripts/backup/restore-database.sh` - Restore script
- `/helm/templates/backup-cronjob.yaml` - Kubernetes CronJob
- `/scripts/backup/verify-backup.sh` - Quarterly restore test

**Backup Script:**
```bash
#!/bin/bash
# scripts/backup/backup-databases.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/${DATE}"

# Backup all databases
pg_dump -h localhost -p 5432 -U postgres -d user_service > ${BACKUP_DIR}/user_service.sql
pg_dump -h localhost -p 5433 -U postgres -d transaction_service > ${BACKUP_DIR}/transaction_service.sql
pg_dump -h localhost -p 5434 -U postgres -d budget_service > ${BACKUP_DIR}/budget_service.sql

# Compress and upload to S3
tar -czf ${BACKUP_DIR}.tar.gz ${BACKUP_DIR}
aws s3 cp ${BACKUP_DIR}.tar.gz s3://moneyplanner-backups/daily/

# Cleanup old backups (30-day retention)
find /backups -type f -mtime +30 -delete
```

**Kubernetes CronJob:**
```yaml
# helm/templates/backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *"  # 2 AM UTC daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15-alpine
            command: ["/scripts/backup-databases.sh"]
            volumeMounts:
            - name: backup-scripts
              mountPath: /scripts
```

**Backup Strategy:**
- **Schedule:** Daily at 2 AM UTC
- **Retention:** 30 days online, 365 days in Glacier
- **Storage:** S3 Standard-IA → Glacier (automated lifecycle)
- **Testing:** Quarterly restore verification

**Monitoring:**
- Alert on backup failure
- Alert on backup size anomaly (>50% change)
- Monthly backup integrity report

---

### 12. Circuit Breaker for OpenAI API
**Effort:** 6-8 hours | **Dependencies:** None | **Resilience critical**

**New Dependencies:**
- `failsafe = "1.3"` (circuit breaker implementation)

**Files to Create:**
- `/transaction-service/src/services/circuit_breaker.rs` - Circuit breaker wrapper

**Files to Modify:**
- `/transaction-service/src/services/ai_service.rs` - Integrate circuit breaker
- `/transaction-service/src/domain/transaction/categorization_service.rs` - Fallback logic

**Circuit Breaker Configuration:**
```rust
use failsafe::{CircuitBreaker, Config, backoff};

let circuit_breaker = CircuitBreaker::new(
    Config::new()
        .failure_threshold(5)        // Open after 5 failures
        .success_threshold(2)        // Close after 2 successes
        .timeout(Duration::from_secs(30))
);
```

**States:**
- **Closed:** Normal operation, calls pass through
- **Open:** Circuit tripped, fail fast without calling OpenAI
- **Half-Open:** Testing recovery, allow limited calls

**Fallback Strategy:**
```rust
pub async fn categorize_with_fallback(
    transaction: &Transaction,
) -> Result<Category> {
    match circuit_breaker.call(|| ai_service.categorize(transaction)).await {
        Ok(category) => Ok(category),
        Err(_) => {
            // Fallback to keyword-based categorization
            keyword_categorizer.categorize(transaction)
        }
    }
}
```

**Keyword-Based Fallback:**
- "GROCERY" → Groceries
- "GAS" / "FUEL" → Transportation
- "RESTAURANT" / "CAFE" → Dining Out
- Default → Uncategorized (manual review)

**Monitoring:**
- Track circuit breaker state changes
- Alert when circuit opens (OpenAI failures)
- Dashboard showing fallback categorization rate

**Testing:**
- Unit tests for circuit breaker states
- Integration tests with simulated OpenAI failures
- Verify fallback categorization accuracy

---

## Critical Files Summary

**Top 12 Most Critical Files:**

1. `/user-service/src/domain/email/email_service.rs` - Email sending (blocks password reset)
2. `/user-service/src/domain/password_reset/reset_service.rs` - Password reset logic
3. `/money-planner-bff/src/services/user_service_client.rs` - BFF-to-user-service client (all auth flows)
4. `/user-service/src/api/middleware/rate_limiter.rs` - Security protection
5. `/transaction-service/src/services/circuit_breaker.rs` - System resilience
6. `/scripts/backup/backup-databases.sh` - Data protection
7. `/user-service/legal/terms-of-service-v1.0.md` - Legal compliance
8. `/user-service/src/api/middleware/security_headers.rs` - OWASP protection
9. `/user-service/src/domain/gdpr/data_export.rs` - GDPR compliance
10. `/money-planner-bff/src/api/routes/password_reset.rs` - Password reset routing
11. `.env.example` (all services) - Secrets management
12. `/user-service/src/domain/health/checks.rs` - Kubernetes health

**Additional Important BFF Files:**
- `/money-planner-bff/src/api/routes/email_verification.rs` - Email verification routing
- `/money-planner-bff/src/api/routes/legal.rs` - Legal document routing
- `/money-planner-bff/src/api/routes/gdpr.rs` - GDPR endpoint routing

**Additional Important Files:**
- All migration files for new tables
- Frontend pages for user flows
- Kubernetes secret manifests
- Security audit CI workflow

---

## Success Criteria

### P0A: User Experience
- ✅ Users can reset password via email link (< 2 min workflow)
- ✅ Email verification sent on registration (< 30 sec delivery)
- ✅ CSV imports skip duplicates with notification
- ✅ Email delivery rate >95%

### P0B: Legal & Compliance
- ✅ Terms of Service and Privacy Policy accessible and legally reviewed
- ✅ Users must accept ToS during registration
- ✅ All OWASP Top 10 mitigations implemented
- ✅ GDPR data export produces complete JSON in < 5 min
- ✅ Account deletion removes all data within 30 days

### P0C: Infrastructure
- ✅ Health checks respond < 1 second
- ✅ Kubernetes probes prevent traffic to unhealthy pods
- ✅ Rate limiting active (429 responses for abuse)
- ✅ Zero hardcoded secrets in repository
- ✅ Daily automated database backups to S3 (100% success rate)
- ✅ Circuit breaker prevents OpenAI cascade failures

---

## Risk Mitigation

**Critical Risks:**

1. **Email Deliverability (HIGH)**
   - Risk: Emails land in spam, users can't reset passwords
   - Mitigation: Complete AWS SES verification, SPF/DKIM/DMARC, warm-up sending

2. **Legal Non-Compliance (HIGH)**
   - Risk: GDPR violations, lawsuits, regulatory fines
   - Mitigation: Professional legal review, comprehensive testing, privacy audit

3. **Secret Exposure (MEDIUM)**
   - Risk: Secrets committed to git, leaked in logs
   - Mitigation: Git-ignored .env files, audit git history, secret scanning in CI

4. **Rate Limiting False Positives (MEDIUM)**
   - Risk: Legitimate users blocked, poor UX
   - Mitigation: Conservative limits, monitoring, user feedback mechanism

5. **Backup Restore Failures (MEDIUM)**
   - Risk: Backups exist but can't be restored when needed
   - Mitigation: Quarterly restore testing, automated verification, runbook documentation

6. **Circuit Breaker Misconfiguration (LOW)**
   - Risk: Circuit too sensitive or too lenient
   - Mitigation: Load testing, monitoring, gradual threshold tuning

---

## Testing Strategy

### Unit Tests
- Email address validation
- Token generation and hashing
- Circuit breaker state transitions
- Deduplication hash calculation
- Rate limiting logic

### Integration Tests
- Full password reset flow
- Email verification end-to-end
- GDPR data export completeness
- CSV import with duplicates
- Health check with real dependencies

### Security Tests
- OWASP ZAP automated scan
- Manual penetration testing
- Timing attack prevention (password reset)
- SQL injection attempts (already protected)
- XSS attempts in transaction descriptions

### Infrastructure Tests
- Kubernetes pod health checks
- Database backup and restore
- Circuit breaker failover
- Rate limiting under load
- Secret rotation procedures

---

## Implementation Notes

**BFF Architecture Pattern:**
```
┌─────────────┐
│  Frontend   │  (port 3000)
│  React App  │
└──────┬──────┘
       │ HTTP
       ▼
┌─────────────┐
│     BFF     │  (port 8082) - API Gateway
│  Routing    │  - Request validation
│   Layer     │  - Service orchestration
└──────┬──────┘  - Response aggregation
       │ HTTP
       ▼
┌─────────────┐
│ User Service│  (port 8080) - Business logic
│ Domain Logic│  - Email sending
│ Persistence │  - Token management
└─────────────┘  - Database operations
```

**BFF Responsibilities:**
- Route frontend requests to appropriate backend services
- Validate JWT tokens and extract user claims
- Transform DTOs between frontend and backend formats
- Aggregate responses from multiple services (GDPR export)
- Handle cross-cutting concerns (rate limiting at gateway level)

**Implementation Pattern for Each Feature:**
1. **User Service**: Implement domain logic and API handlers
2. **BFF Service**: Add proxy routes and service client methods
3. **Frontend**: Call BFF endpoints (never direct backend calls)

**Architecture Patterns to Follow:**
- Domain-Driven Design (DDD) with free functions in domain layer
- Traits defined in `domain/interfaces`
- SQLx for all database operations
- Proper error handling with `thiserror`
- OpenAPI documentation with `utoipa`
- Structured logging with `tracing`
- **BFF Pattern**: All frontend requests go through BFF (no direct backend access)

**Code Quality:**
- Comprehensive test coverage (aim for 90%+)
- Integration tests for critical flows
- Proper error messages for user-facing errors
- Consistent naming conventions
- Documentation for public APIs

**Deployment:**
- Test locally with `.env` files first
- Test in Kubernetes with Secrets
- Verify health checks in staging
- Load test before production
- Gradual rollout with monitoring

---

## Next Steps After Plan Approval

1. **Week 1 Kickoff:**
   - Set up AWS SES account and verify domain
   - Create S3 bucket for backups
   - Generate development secrets
   - Start email service integration

2. **Parallel Workstreams:**
   - Developer 1: Email + Password Reset + Verification (User Service + BFF)
   - Developer 2: Secrets + Health Checks + Rate Limiting (All Services)
   - Developer 3: Legal docs + GDPR + OWASP assessment (User Service + BFF)

3. **Weekly Checkpoints:**
   - End of Week 1: Email service working
   - End of Week 2: Password reset complete
   - End of Week 3: All P0A+P0B done
   - End of Week 4: All P0C done + testing

4. **Go/No-Go Decision (Week 4):**
   - All P0 items complete ✅
   - Security audit passed ✅
   - Load testing passed ✅
   - Legal review complete ✅
   - → Proceed to private beta launch

---

**Total Implementation Timeline: 3-4 weeks**
**Total Effort: 110-142 hours** (increased from 98-125h to include BFF integration)
**Production Launch Readiness: After Week 4 + 2 weeks beta testing**

**Key Architecture Update:**
All new features require implementation in TWO layers:
1. **User Service** - Domain logic, business rules, database operations
2. **BFF Service** - API routing, request proxying, response aggregation

This ensures proper separation of concerns and maintains the established microservices architecture pattern.
