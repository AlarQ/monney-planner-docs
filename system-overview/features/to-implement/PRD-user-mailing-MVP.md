# PRD: User Mailing System - MVP

## Document Information
- **Version**: 1.0 MVP
- **Date**: January 2025
- **Status**: Implementation Ready
- **Author**: Development Team
- **Related Linear Tasks**: ALA-69, ALA-70, ALA-71
- **Timeline**: 3 weeks
- **Full PRD**: See [PRD-user-mailing.md](./PRD-user-mailing.md) for post-MVP features

## Executive Summary

This MVP focuses on critical authentication-related email functionality required for launch. The implementation delivers three core features: **password reset**, **email verification**, and **basic email service infrastructure**. This is a critical launch blocker - without password reset, users who forget passwords are permanently locked out.

**MVP Scope:**
- FR-1: Email Service Infrastructure (SendGrid integration)
- FR-2: Password Reset Flow
- FR-3: Email Verification Flow
- FR-9: Email Queue (simplified for reliability)

**Post-MVP (Deferred):**
- Welcome & onboarding emails
- Budget & transaction notifications
- Security alerts
- Marketing emails
- Advanced template management
- Email preferences UI

**Timeline**: 3 weeks implementation + 1 week testing/deployment

## Problem Statement

### Critical Authentication Gaps

**Password Reset** (ALA-69 - CRITICAL LAUNCH BLOCKER):
- Users who forget passwords cannot recover accounts
- No self-service password recovery mechanism
- Results in support burden and user frustration

**Email Verification** (ALA-71 - HIGH PRIORITY):
- No validation that user emails are real/accessible
- Accounts can be created with invalid emails
- Cannot reliably communicate with users

**Email Infrastructure** (ALA-70 - CRITICAL):
- No email sending capability exists
- Cannot send transactional emails
- Blocks password reset and verification features

## User Stories

### Primary User Stories (MVP)

**US-1: Password Reset**
As a user who forgot my password, I want to reset it via email, so that I can regain access to my account without contacting support.

**Acceptance Criteria:**
- User can request password reset from login page
- User receives email with reset link within 30 seconds
- Reset link expires after 15 minutes
- User can set new password using valid link
- After reset, all sessions are invalidated (forced re-login)

**US-2: Email Verification**
As a new user, I want to verify my email address, so that I can ensure my account is secure and receive important notifications.

**Acceptance Criteria:**
- User receives verification email after registration
- Verification link expires after 24 hours
- User can verify email by clicking link
- User can resend verification if email was lost
- Unverified users can still log in but see warning banner

**US-3: Reliable Email Delivery**
As a system, I need reliable email delivery with retry logic, so that critical authentication emails reach users even if temporary failures occur.

**Acceptance Criteria:**
- Failed emails are retried automatically (3 attempts max)
- Email queue ensures emails aren't lost
- Email delivery status is tracked
- System handles SendGrid rate limits gracefully

## Functional Requirements

### FR-1: Email Service Infrastructure

**Requirement**: System must have email sending capability using SendGrid.

**Implementation**:
- SendGrid API integration in user-service
- Email module in domain layer (`domain/email/`)
- Tera template engine for HTML emails
- Environment-based configuration

**Configuration**:
```bash
# Environment variables
SENDGRID_API_KEY=SG.xxx
SENDGRID_FROM_EMAIL=noreply@moneyplanner.com
SENDGRID_FROM_NAME="Money Planner"
PASSWORD_RESET_TOKEN_EXPIRY_MINUTES=15
EMAIL_VERIFICATION_TOKEN_EXPIRY_HOURS=24
```

**Email Templates** (Hardcoded in Rust):
- `templates/password_reset.html.tera`
- `templates/email_verification.html.tera`
- `templates/base.html.tera` (shared layout)

**Dependencies to Add**:
```toml
# Cargo.toml additions
lettre = "0.11"
tera = "1.19"
tokio-cron-scheduler = "0.9"
```

**Acceptance Criteria**:
- Email service can send emails via SendGrid
- Templates render correctly with dynamic variables
- Configuration managed via environment variables
- Basic error handling for failed sends

---

### FR-2: Password Reset Flow

**Requirement**: Users must be able to reset forgotten passwords via email.

**Endpoints**:
```
POST /v1/auth/forgot-password
POST /v1/auth/reset-password
```

**Flow**:
1. User clicks "Forgot Password" on login page
2. User enters email address
3. System validates email format
4. System generates secure reset token
5. Token hashed with Argon2 and stored in database
6. Email sent with reset link: `https://app.moneyplanner.com/reset-password?token={token}`
7. User clicks link, enters new password
8. System validates token and updates password
9. All user sessions invalidated (forced re-login)
10. Token marked as used/cleared from database

**Security Requirements**:
- Tokens cryptographically secure (32 bytes random)
- Tokens hashed with Argon2 before database storage
- Tokens expire after 15 minutes
- Tokens are single-use (cleared after password reset)
- Rate limiting: Max 1 request per 20 minutes per email
- Constant-time response to prevent email enumeration
- All sessions invalidated on password reset (mandatory)

**API Specification**:

**POST /v1/auth/forgot-password**
```json
Request:
{
  "email": "user@example.com"
}

Response (always 200 OK):
{
  "message": "If an account with that email exists, a password reset link has been sent."
}

Errors:
429 Too Many Requests (rate limit exceeded)
{
  "error_code": "RATE_LIMIT_EXCEEDED",
  "message": "Too many password reset requests. Please try again in 20 minutes.",
  "retry_after_seconds": 1200
}
```

**POST /v1/auth/reset-password**
```json
Request:
{
  "token": "abc123...",
  "new_password": "NewSecurePass123!"
}

Response (200 OK):
{
  "message": "Password has been reset successfully. Please log in with your new password."
}

Errors:
400 Bad Request (invalid/expired token):
{
  "error_code": "INVALID_RESET_TOKEN",
  "message": "Invalid or expired password reset link. Please request a new one."
}

400 Bad Request (weak password):
{
  "error_code": "INVALID_PASSWORD",
  "message": "Password must be at least 8 characters with uppercase, lowercase, number, and special character."
}
```

**Database Schema**:
```sql
-- Add to users table
ALTER TABLE users ADD COLUMN password_reset_token VARCHAR(255);
ALTER TABLE users ADD COLUMN password_reset_token_expires_at TIMESTAMPTZ;
ALTER TABLE users ADD COLUMN password_reset_requested_at TIMESTAMPTZ;

-- Index for token lookups
CREATE INDEX idx_users_password_reset_token
    ON users(password_reset_token)
    WHERE password_reset_token IS NOT NULL;

-- Index for rate limiting
CREATE INDEX idx_users_password_reset_requested
    ON users(password_reset_requested_at)
    WHERE password_reset_requested_at IS NOT NULL;
```

**Email Template**: Password Reset
```
Subject: Reset Your Money Planner Password

Hi {{user_name}},

You requested to reset your password. Click the link below to create a new password:

{{reset_link}}

This link will expire in 15 minutes.

If you didn't request this, please ignore this email or contact support if you have concerns.

Best regards,
Money Planner Team
```

**Acceptance Criteria**:
- ✅ Users can request password reset via email
- ✅ Reset tokens are secure, hashed, and time-limited (15 minutes)
- ✅ Password reset completes successfully with valid token
- ✅ Invalid/expired tokens are rejected with clear error
- ✅ Rate limiting prevents abuse (1 request per 20 minutes)
- ✅ Email enumeration is prevented (constant-time response)
- ✅ All sessions invalidated after password reset
- ✅ Integration tests cover full flow

---

### FR-3: Email Verification

**Requirement**: New users must verify their email address after registration.

**Endpoints**:
```
POST /v1/auth/verify-email
POST /v1/auth/resend-verification
```

**Flow**:
1. User registers new account via `POST /v1/users`
2. Account created with `email_verified = false`
3. Verification token generated (32 bytes random, plain text storage OK for MVP)
4. Verification email sent
5. User clicks verification link: `https://app.moneyplanner.com/verify-email?token={token}`
6. Token validated, `email_verified = true`
7. Token cleared from database
8. User redirected to app with success message

**API Specification**:

**POST /v1/users** (Modified)
```json
Response (201 Created):
{
  "id": "uuid",
  "email": "user@example.com",
  "email_verified": false,
  "created_at": "2025-01-22T10:00:00Z",
  "message": "Account created successfully. Please check your email to verify your account."
}
```

**POST /v1/auth/verify-email**
```json
Request:
{
  "token": "abc123..."
}

Response (200 OK):
{
  "message": "Email verified successfully!"
}

Errors:
400 Bad Request (invalid/expired token):
{
  "error_code": "INVALID_VERIFICATION_TOKEN",
  "message": "Invalid or expired verification link. Please request a new one."
}

400 Bad Request (already verified):
{
  "error_code": "EMAIL_ALREADY_VERIFIED",
  "message": "This email address has already been verified."
}
```

**POST /v1/auth/resend-verification**
```json
Request:
{
  "email": "user@example.com"
}

Response (200 OK):
{
  "message": "Verification email has been sent."
}

Errors:
429 Too Many Requests (rate limit):
{
  "error_code": "RATE_LIMIT_EXCEEDED",
  "message": "Too many verification requests. Please try again in 20 minutes.",
  "retry_after_seconds": 1200
}
```

**Database Schema**:
```sql
-- Add to users table
ALTER TABLE users ADD COLUMN email_verified BOOLEAN NOT NULL DEFAULT false;
ALTER TABLE users ADD COLUMN email_verification_token VARCHAR(255);
ALTER TABLE users ADD COLUMN email_verification_sent_at TIMESTAMPTZ;
ALTER TABLE users ADD COLUMN email_verified_at TIMESTAMPTZ;

-- Index for token lookups
CREATE INDEX idx_users_email_verification_token
    ON users(email_verification_token)
    WHERE email_verification_token IS NOT NULL;

-- Comment for documentation
COMMENT ON COLUMN users.email_verified IS 'Whether user has verified their email address';
COMMENT ON COLUMN users.email_verification_token IS 'Token for email verification (expires in 24 hours)';
```

**Unverified User Behavior** (MVP):

✅ **Unverified users CAN:**
- Log in with email/password
- Create transactions
- Create budgets
- View all data
- Update profile (except email address)

❌ **Unverified users CANNOT:**
- Receive email notifications (when implemented post-MVP)
- Export data to CSV/PDF (when implemented post-MVP)
- Change email address

⚠️ **Warning Banner:**
- Displayed in UI: "Please verify your email address to enable notifications and exports"
- Link to resend verification email
- Dismissible but reappears on next login until verified

**Email Template**: Email Verification
```
Subject: Verify Your Money Planner Account

Hi {{user_name}},

Welcome to Money Planner! Please verify your email address by clicking the link below:

{{verification_link}}

This link will expire in 24 hours.

If you didn't create this account, please ignore this email.

Best regards,
Money Planner Team
```

**Acceptance Criteria**:
- ✅ New users receive verification email after registration
- ✅ Email verification completes successfully with valid token
- ✅ Invalid/expired tokens are rejected with clear error
- ✅ Users can resend verification emails (rate limited)
- ✅ Unverified accounts are marked in database
- ✅ Unverified users see warning banner in UI
- ✅ Integration tests cover full flow

---

### FR-9: Email Queue (Simplified MVP)

**Requirement**: Email sending must be reliable with retry logic for failed sends.

**Implementation**:
- Simple PostgreSQL-based queue
- Async background worker processing
- 3-retry logic with exponential backoff
- No priority queuing (MVP)
- No email analytics (MVP)

**Database Schema**:
```sql
CREATE TABLE email_queue (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    to_email VARCHAR(255) NOT NULL,
    subject VARCHAR(255) NOT NULL,
    template_type VARCHAR(50) NOT NULL,  -- 'password_reset', 'email_verification'
    template_variables JSONB NOT NULL,   -- Dynamic template data
    status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- 'pending', 'sent', 'failed'
    attempts INTEGER NOT NULL DEFAULT 0,
    max_attempts INTEGER NOT NULL DEFAULT 3,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    sent_at TIMESTAMPTZ
);

CREATE INDEX idx_email_queue_status ON email_queue(status, created_at);
CREATE INDEX idx_email_queue_user ON email_queue(user_id);
```

**Queue Processing**:
- Background worker runs every 30 seconds
- Processes emails in FIFO order (created_at ASC)
- Retries failed emails with exponential backoff (1min, 5min, 15min)
- Marks as 'failed' after 3 attempts
- Logs errors for manual intervention

**Acceptance Criteria**:
- ✅ Emails are queued reliably
- ✅ Failed emails are retried automatically (3 attempts)
- ✅ Queue processing is efficient (< 5 min lag)
- ✅ Error logging for debugging

---

## Non-Functional Requirements

### NFR-1: Email Delivery Reliability
- **Requirement**: 99% email delivery success rate
- **Implementation**: Retry mechanism with 3 attempts, SendGrid reliability

### NFR-2: Email Delivery Speed
- **Requirement**: Password reset emails delivered within 30 seconds
- **Implementation**: Async queue processing, immediate processing for pending emails

### NFR-3: Security
- **Requirement**: Secure token generation, hashed storage for password reset tokens
- **Implementation**:
  - Cryptographically secure random tokens (32 bytes)
  - Argon2 hashing for password reset tokens
  - Constant-time responses to prevent enumeration
  - HTTPS for all email links

### NFR-4: Rate Limiting
- **Requirement**: Prevent abuse of email sending
- **Implementation**:
  - Max 1 password reset request per 20 minutes per email
  - Max 1 verification resend per 20 minutes per email
  - HTTP 429 response with Retry-After header

### NFR-5: Monitoring
- **Requirement**: Track email delivery status
- **Implementation**:
  - Log all email sends with status
  - Alert on high failure rate (> 10%)
  - Queue lag monitoring (alert if > 5 minutes)

---

## Technical Architecture

### Service Architecture (MVP)

```
┌─────────────┐
│  Frontend   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│     BFF     │
└──────┬──────┘
       │
       ├──────────────┬──────────────┐
       ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│User Service │ │Transaction  │ │Budget       │
│(with email) │ │Service      │ │Service      │
└──────┬──────┘ └─────────────┘ └─────────────┘
       │
       ▼
┌─────────────────┐
│    SendGrid     │
└─────────────────┘
```

**Email Module Location**: `user-service/src/domain/email/`

**Rationale**:
- Email functionality integrated into user-service for MVP simplicity
- Avoids creating new microservice
- Can be extracted to dedicated service post-MVP if needed

### Technology Stack

**Email Provider**: SendGrid
- **Why**: Easy integration, good deliverability, generous free tier (100 emails/day)
- **Alternative**: AWS SES (cheaper at scale, requires more setup)

**Template Engine**: Tera
- **Why**: Rust-native, compile-time safety, good performance
- **Alternative**: Handlebars (more familiar syntax but external dependency)

**Queue System**: PostgreSQL-based
- **Why**: Uses existing database, simple implementation, no new infrastructure
- **Alternative**: Redis (faster but adds infrastructure complexity)

---

## Database Migrations

### Migration 1: Add Email Fields to Users Table

**File**: `user-service/migrations/20250122_add_email_fields.sql`

```sql
-- Email verification fields
ALTER TABLE users ADD COLUMN email_verified BOOLEAN NOT NULL DEFAULT false;
ALTER TABLE users ADD COLUMN email_verification_token VARCHAR(255);
ALTER TABLE users ADD COLUMN email_verification_sent_at TIMESTAMPTZ;
ALTER TABLE users ADD COLUMN email_verified_at TIMESTAMPTZ;

-- Password reset fields
ALTER TABLE users ADD COLUMN password_reset_token VARCHAR(255);
ALTER TABLE users ADD COLUMN password_reset_token_expires_at TIMESTAMPTZ;
ALTER TABLE users ADD COLUMN password_reset_requested_at TIMESTAMPTZ;

-- Indexes for performance
CREATE INDEX idx_users_email_verification_token
    ON users(email_verification_token)
    WHERE email_verification_token IS NOT NULL;

CREATE INDEX idx_users_password_reset_token
    ON users(password_reset_token)
    WHERE password_reset_token IS NOT NULL;

CREATE INDEX idx_users_password_reset_requested
    ON users(password_reset_requested_at)
    WHERE password_reset_requested_at IS NOT NULL;

-- Documentation
COMMENT ON COLUMN users.email_verified IS 'Whether user has verified their email address';
COMMENT ON COLUMN users.password_reset_token IS 'Hashed token for password reset (expires in 15 minutes)';
COMMENT ON COLUMN users.email_verification_token IS 'Token for email verification (expires in 24 hours)';
```

**Rollback**:
```sql
DROP INDEX IF EXISTS idx_users_email_verification_token;
DROP INDEX IF EXISTS idx_users_password_reset_token;
DROP INDEX IF EXISTS idx_users_password_reset_requested;

ALTER TABLE users DROP COLUMN IF EXISTS email_verified;
ALTER TABLE users DROP COLUMN IF EXISTS email_verification_token;
ALTER TABLE users DROP COLUMN IF EXISTS email_verification_sent_at;
ALTER TABLE users DROP COLUMN IF EXISTS email_verified_at;
ALTER TABLE users DROP COLUMN IF EXISTS password_reset_token;
ALTER TABLE users DROP COLUMN IF EXISTS password_reset_token_expires_at;
ALTER TABLE users DROP COLUMN IF EXISTS password_reset_requested_at;
```

### Migration 2: Create Email Queue Table

**File**: `user-service/migrations/20250123_create_email_queue.sql`

```sql
CREATE TABLE email_queue (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    to_email VARCHAR(255) NOT NULL,
    subject VARCHAR(255) NOT NULL,
    template_type VARCHAR(50) NOT NULL,
    template_variables JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    attempts INTEGER NOT NULL DEFAULT 0,
    max_attempts INTEGER NOT NULL DEFAULT 3,
    error_message TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    sent_at TIMESTAMPTZ
);

CREATE INDEX idx_email_queue_status ON email_queue(status, created_at);
CREATE INDEX idx_email_queue_user ON email_queue(user_id);

COMMENT ON TABLE email_queue IS 'Queue for reliable email delivery with retry logic';
```

**Rollback**:
```sql
DROP TABLE IF EXISTS email_queue;
```

---

## Implementation Plan (3 Weeks)

### Week 1: Email Infrastructure + Password Reset

**Goal**: Basic email sending + password reset flow

**Tasks**:
1. **Day 1-2**: SendGrid Setup & Email Module
   - Create SendGrid account and API key
   - Add email configuration to `config.rs`
   - Create `domain/email/mod.rs` module
   - Implement basic email sending function
   - Create Tera templates (password_reset.html.tera)

2. **Day 3**: Database Migration
   - Create migration for password reset fields
   - Run migration on dev database
   - Test rollback script

3. **Day 4-5**: Password Reset Endpoints
   - Implement `POST /v1/auth/forgot-password` handler
   - Implement `POST /v1/auth/reset-password` handler
   - Add token generation/hashing functions
   - Add rate limiting logic
   - Update error codes enum

**Deliverables**:
- Working email sending to SendGrid
- Password reset flow functional end-to-end
- Rate limiting active
- Integration tests passing

---

### Week 2: Email Verification

**Goal**: Email verification flow + unverified user handling

**Tasks**:
1. **Day 1**: Database Migration
   - Create migration for email verification fields
   - Run migration on dev database

2. **Day 2-3**: Email Verification Endpoints
   - Implement `POST /v1/auth/verify-email` handler
   - Implement `POST /v1/auth/resend-verification` handler
   - Update `POST /v1/users` to send verification email
   - Create verification email template

3. **Day 4**: Frontend Integration
   - Add verification warning banner component
   - Add resend verification button
   - Update user profile to show verification status

4. **Day 5**: Testing
   - Integration tests for verification flow
   - E2E test: register → verify → login

**Deliverables**:
- Email verification flow functional
- Unverified users see warning banner
- Resend verification works
- Tests passing

---

### Week 3: Email Queue + Production Readiness

**Goal**: Reliable email delivery + production deployment

**Tasks**:
1. **Day 1-2**: Email Queue Implementation
   - Create email_queue table migration
   - Implement queue repository
   - Create background worker for queue processing
   - Migrate email sending to use queue
   - Add retry logic with exponential backoff

2. **Day 3**: OpenAPI Documentation
   - Update Swagger docs for new endpoints
   - Add request/response examples
   - Document error codes

3. **Day 4**: Security Audit
   - Review token generation (secure randomness)
   - Test rate limiting effectiveness
   - Test email enumeration prevention
   - Test session invalidation on password reset

4. **Day 5**: Deployment & Monitoring
   - Deploy to staging environment
   - Set up email monitoring (delivery rate, failures)
   - Create runbook for common issues
   - Load test email queue

**Deliverables**:
- Email queue operational with retry logic
- OpenAPI docs updated
- Security audit complete
- Deployed to staging
- Ready for production

---

## Testing Strategy

### Unit Tests
- Token generation and validation
- Template rendering
- Rate limiting logic
- Email enumeration prevention (constant-time response)

### Integration Tests
- Password reset flow end-to-end
- Email verification flow end-to-end
- Email queue processing
- Rate limiting enforcement
- Session invalidation on password reset

### E2E Tests (Playwright)
- User registers → receives verification email → verifies → logs in
- User forgets password → requests reset → receives email → resets → logs in
- User clicks expired token → sees error → requests new token

### Manual Testing Checklist
- [ ] Password reset email received within 30 seconds
- [ ] Password reset link works correctly
- [ ] Password reset link expires after 15 minutes
- [ ] Verification email received after registration
- [ ] Verification link works correctly
- [ ] Verification link expires after 24 hours
- [ ] Rate limiting blocks excessive requests
- [ ] Unverified users see warning banner
- [ ] All sessions invalidated after password reset

---

## Security Considerations

### Token Security
✅ **Use cryptographically secure random generators**
- Use `rand::rngs::OsRng` for token generation
- 32 bytes = 256 bits of entropy

✅ **Hash password reset tokens before storage**
- Use Argon2 (same as passwords)
- Prevents token theft if database is compromised

✅ **Tokens must be single-use**
- Clear token from database after successful use
- Concurrent token generation invalidates previous token

✅ **Tokens must expire**
- Password reset: 15 minutes
- Email verification: 24 hours

### Email Enumeration Prevention
✅ **Constant-time responses**
- Always return same message regardless of email existence
- Add fake work delay to match real email sending time

✅ **Rate limiting**
- Limit requests per email AND per IP
- Prevents brute-force email discovery

### Session Management
✅ **Invalidate all sessions on password reset**
- Mandatory, not optional
- Forces re-login after password change
- Prevents compromised sessions from remaining active

---

## Monitoring & Observability

### Metrics to Track
- Email delivery success rate (target: > 99%)
- Email delivery time (target: < 30 seconds)
- Queue processing lag (target: < 5 minutes)
- Failed email count (alert if > 10 failures/hour)
- Rate limit hit count

### Alerts
- **Critical**: Email delivery success rate < 95%
- **Warning**: Queue processing lag > 5 minutes
- **Warning**: Failed email count > 10/hour

### Logging
- All email sends (user_id, template_type, status)
- All token generation (user_id, type, expiration)
- All rate limit hits (email, IP, timestamp)
- All errors with stack traces

---

## Success Metrics

### User Engagement
- Password reset completion rate > 80%
- Email verification rate > 90%

### System Performance
- Email delivery success rate > 99%
- Average delivery time < 30 seconds
- Queue processing lag < 5 minutes

### Business Impact
- Reduced support tickets for password resets
- Increased user activation (email verification)

---

## Dependencies

### External Dependencies
- SendGrid account (free tier: 100 emails/day, sufficient for MVP)
- Domain configuration (SPF/DKIM records for moneyplanner.com)

### Internal Dependencies
- User service (authentication, user data) ✅ Exists
- BFF (API endpoints) ✅ Exists
- Frontend (UI components) - Requires new components
- PostgreSQL database ✅ Exists

### New Rust Dependencies
```toml
[dependencies]
lettre = "0.11"           # Email sending
tera = "1.19"             # Template engine
tokio-cron-scheduler = "0.9"  # Background jobs
```

---

## Risks & Mitigations

### Risk 1: SendGrid Deliverability Issues
- **Impact**: High - Emails not reaching users
- **Probability**: Low (SendGrid is reliable)
- **Mitigation**: Monitor delivery rates, set up SPF/DKIM, have AWS SES as backup

### Risk 2: Rate Limits
- **Impact**: Medium - Emails delayed
- **Probability**: Low (free tier supports 100/day)
- **Mitigation**: Queue system with retry logic, upgrade to paid tier if needed

### Risk 3: Email Spam Classification
- **Impact**: Medium - Emails marked as spam
- **Probability**: Medium (transactional emails generally safe)
- **Mitigation**: SPF/DKIM/DMARC setup, avoid spam trigger words, monitor complaint rates

---

## Post-MVP Roadmap

After MVP launch, the following features from the full PRD will be prioritized:

**Phase 1** (Post-MVP +1 month):
- Welcome & onboarding email sequence
- Basic budget alert notifications

**Phase 2** (Post-MVP +2 months):
- Transaction notifications (large transactions)
- Budget settlement summaries

**Phase 3** (Post-MVP +3 months):
- Security alerts (new device login)
- Email preferences UI

**Phase 4** (Post-MVP +6 months):
- Marketing email campaigns
- Advanced template management
- Email analytics

See [PRD-user-mailing.md](./PRD-user-mailing.md) for complete post-MVP feature specifications.

---

## Appendix

### Error Codes Reference

```rust
pub enum EmailErrorCode {
    InvalidResetToken,
    ExpiredResetToken,
    InvalidVerificationToken,
    ExpiredVerificationToken,
    RateLimitExceeded,
    EmailSendingFailed,
    EmailAlreadyVerified,
}
```

### Email Template Variables

**Password Reset Template**:
- `{{user_name}}` - User's name or email
- `{{reset_link}}` - Full reset URL with token
- `{{expiry_minutes}}` - Token expiration time (15)

**Email Verification Template**:
- `{{user_name}}` - User's name or email
- `{{verification_link}}` - Full verification URL with token
- `{{expiry_hours}}` - Token expiration time (24)

### API Endpoint Summary

| Endpoint | Method | Description | Auth Required |
|----------|--------|-------------|---------------|
| `/v1/auth/forgot-password` | POST | Request password reset | No |
| `/v1/auth/reset-password` | POST | Reset password with token | No |
| `/v1/auth/verify-email` | POST | Verify email with token | No |
| `/v1/auth/resend-verification` | POST | Resend verification email | No |
| `/v1/users` | POST | Create user (triggers verification) | No |

---

**END OF MVP PRD**

For complete feature specifications including welcome emails, notifications, marketing, and advanced features, see [PRD-user-mailing.md](./PRD-user-mailing.md).
