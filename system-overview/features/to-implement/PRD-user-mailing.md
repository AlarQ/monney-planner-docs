# PRD: User Mailing System

## Document Information
- **Version**: 1.0
- **Date**: January 2025
- **Status**: Draft
- **Author**: Development Team

## Executive Summary

The User Mailing System enables Money Planner to communicate with users via email for transactional, authentication, and marketing purposes. This feature provides automated email delivery for critical user actions (password resets, account verification), system notifications (budget alerts, transaction summaries), and marketing communications (newsletters, feature announcements).

**Key Benefits:**
- Automated transactional emails reduce support burden
- Password reset functionality improves user experience and security
- Email verification enhances account security
- Marketing emails enable user engagement and retention
- Budget and transaction notifications keep users informed
- Centralized email service ensures consistent delivery and tracking

## Problem Statement

Currently, Money Planner has no email communication capabilities, which creates several critical gaps:

**Authentication & Security:**
- No password reset functionality - users who forget passwords cannot recover accounts
- No email verification - accounts can be created with invalid or unverified emails
- No security alerts - users are not notified of suspicious account activity

**User Engagement:**
- No welcome emails for new users
- No onboarding sequence to guide new users
- No notifications for important events (budget alerts, large transactions)
- No periodic summaries or reports

**Marketing & Growth:**
- No ability to send newsletters or feature announcements
- No re-engagement campaigns for inactive users
- No promotional emails for new features
- Limited ability to communicate product updates

**Operational:**
- No automated notifications for system maintenance
- No ability to notify users of policy changes
- No way to communicate service updates

## Current System Analysis

### Existing Architecture

**User Service (`user-service`)**:
- **Current State**: Handles user authentication, session management
- **Email Storage**: Users have `email` field in database
- **Missing**: No email verification, no password reset tokens, no email sending capability
- **Endpoints**: 
  - `POST /v1/users` - Create user (no email verification)
  - `POST /v1/users/login` - Login (no forgot password flow)
  - `GET /v1/users/{id}` - Get user info

**Backend for Frontend (`money-planner-bff`)**:
- **Current State**: Intermediary between frontend and user-service
- **Missing**: No email-related endpoints, no email service integration

**Frontend (`MoneyPlannerFE`)**:
- **Current State**: React application with user authentication UI
- **Missing**: No forgot password UI, no email verification UI, no email preferences

**Transaction Service (`transaction-service`)**:
- **Current State**: Handles transaction CRUD operations
- **Missing**: No email notifications for large transactions or unusual activity

**Budget Service (`budget-service`)**:
- **Current State**: Handles budget management and settlement
- **Missing**: No email notifications for budget alerts, period summaries, or settlement reports

### Database Schema Gaps

**User Service Database** (Port 5432):
- `users` table exists with `email` field
- **Missing**: 
  - `email_verified` boolean field
  - `email_verification_token` field
  - `email_verification_sent_at` timestamp
  - `password_reset_token` field
  - `password_reset_token_expires_at` timestamp
  - `email_preferences` JSON field (for opt-in/opt-out)

**New Tables Needed**:
- `email_templates` - Store email templates
- `email_queue` - Queue emails for delivery
- `email_logs` - Track email delivery status
- `email_preferences` - User email preferences (optional, can use JSON in users table)

### Integration Points

**Kafka Events** (if applicable):
- Transaction events could trigger email notifications
- Budget settlement events could trigger summary emails
- User events (registration, login from new device) could trigger security emails

## User Stories

### Primary User Stories

**As a user who forgot my password**, I want to reset it via email, so that I can regain access to my account without contacting support.

**As a new user**, I want to receive a welcome email with onboarding tips, so that I can get started quickly.

**As a user**, I want to receive email notifications when my budget is close to being exceeded, so that I can adjust my spending.

**As a user**, I want to receive periodic spending summaries via email, so that I can review my financial activity without logging in.

### Secondary User Stories

**As a user**, I want to verify my email address during registration, so that I can ensure account security and receive important notifications.

**As a user**, I want to receive security alerts when my account is accessed from a new device, so that I can detect unauthorized access.

**As a user**, I want to receive notifications for large transactions, so that I can quickly identify potential fraud.

**As a user**, I want to control which emails I receive, so that I can manage my inbox preferences.

**As a user**, I want to receive budget settlement summaries via email, so that I can review period performance without logging in.

**As a marketing team**, I want to send newsletters and feature announcements, so that I can engage users and drive adoption.

**As a system administrator**, I want to send maintenance notifications, so that users are informed of service disruptions.

## Functional Requirements

### FR-1: Email Service Infrastructure

**Requirement**: System must have a centralized email service capable of sending emails reliably.

**Details**:
- Email service should be a separate microservice or integrated into user-service
- Support for SMTP or email service provider APIs (SendGrid, AWS SES, Mailgun, etc.)
- Email queue system for reliable delivery (retry failed emails)
- Template engine for email content (HTML and plain text)
- Support for email attachments (PDF reports, CSV exports)
- Rate limiting to prevent abuse
- Bounce and complaint handling

**Acceptance Criteria**:
- Email service can send emails via configured provider
- Failed emails are retried with exponential backoff
- Email delivery status is tracked and logged
- Service handles provider rate limits gracefully

**Implementation Options**:

**Option A**: Dedicated Email Service
- New `email-service` microservice
- Handles all email sending logic
- Other services call email-service via HTTP/gRPC
- Pro: Separation of concerns, scalable
- Con: Additional service to maintain

**Option B**: Email Module in User Service
- Email functionality as module in `user-service`
- Other services call user-service for emails
- Pro: Simpler architecture, fewer services
- Con: User-service becomes more complex

**Option C**: Shared Email Library
- Email library used by multiple services
- Each service sends emails directly
- Pro: Flexible, no single point of failure
- Con: Duplication, harder to track all emails

**Recommended**: Option A (Dedicated Email Service) for production, Option B for MVP

### FR-2: Password Reset Flow

**Requirement**: Users must be able to reset forgotten passwords via email.

**Details**:
- User requests password reset via frontend
- System generates secure, time-limited reset token
- Token stored in database with expiration (15 minutes default)
- Email sent with reset link containing token
- User clicks link, enters new password
- Token validated and invalidated after use
- Password updated, all sessions invalidated (optional)

**Flow**:
```
1. User clicks "Forgot Password" on login page
2. User enters email address
3. System validates email exists
4. System generates reset token (cryptographically secure)
5. Token stored in database with expiration timestamp
6. Email sent with reset link: https://app.moneyplanner.com/reset-password?token={token}
7. User clicks link (validates token, shows password reset form)
8. User enters new password
9. System validates token and updates password
10. Token invalidated, user redirected to login
```

**Security Requirements**:
- Tokens must be cryptographically secure (use secure random generator)
- Tokens expire after 15 minutes (configurable)
- Tokens are single-use (invalidated after password reset)
- Rate limiting: Max 3 reset requests per email per hour
- No indication if email exists (prevent email enumeration)
- Password requirements enforced (same as registration)

**Database Schema**:
```sql
ALTER TABLE users ADD COLUMN password_reset_token VARCHAR(255);
ALTER TABLE users ADD COLUMN password_reset_token_expires_at TIMESTAMP;
ALTER TABLE users ADD COLUMN password_reset_requested_at TIMESTAMP;
```

**API Endpoints**:
- `POST /v1/users/forgot-password` - Request password reset
  - Request: `{ "email": "user@example.com" }`
  - Response: `200 OK` (always, to prevent email enumeration)
  
- `POST /v1/users/reset-password` - Reset password with token
  - Request: `{ "token": "...", "new_password": "..." }`
  - Response: `200 OK` or `400 Bad Request` (invalid/expired token)

**Email Template**: Password Reset Email
- Subject: "Reset Your Money Planner Password"
- Content: Clear instructions, reset link, security notice
- Expiration notice (15 minutes)
- Support contact if user didn't request reset

**Acceptance Criteria**:
- Users can request password reset via email
- Reset tokens are secure and time-limited
- Password reset completes successfully
- Invalid/expired tokens are rejected
- Rate limiting prevents abuse
- Email enumeration is prevented

### FR-3: Email Verification

**Requirement**: New users must verify their email address during registration.

**Details**:
- After user registration, verification email is sent
- Email contains verification link with token
- Token is time-limited (24 hours default)
- User clicks link to verify email
- Account marked as verified
- Unverified accounts may have limited functionality (optional)

**Flow**:
```
1. User registers new account
2. Account created with email_verified = false
3. Verification token generated and stored
4. Verification email sent
5. User clicks verification link
6. Token validated, email_verified = true
7. Token invalidated
8. User redirected to app with success message
```

**Database Schema**:
```sql
ALTER TABLE users ADD COLUMN email_verified BOOLEAN NOT NULL DEFAULT false;
ALTER TABLE users ADD COLUMN email_verification_token VARCHAR(255);
ALTER TABLE users ADD COLUMN email_verification_sent_at TIMESTAMP;
ALTER TABLE users ADD COLUMN email_verified_at TIMESTAMP;
```

**API Endpoints**:
- `POST /v1/users` - Create user (triggers verification email)
  - Response includes: `email_verification_required: true`
  
- `POST /v1/users/verify-email` - Verify email with token
  - Request: `{ "token": "..." }`
  - Response: `200 OK` or `400 Bad Request`
  
- `POST /v1/users/resend-verification` - Resend verification email
  - Request: `{ "email": "user@example.com" }`
  - Response: `200 OK`
  - Rate limit: Max 3 requests per hour

**Email Template**: Email Verification
- Subject: "Verify Your Money Planner Account"
- Content: Welcome message, verification link, instructions
- Expiration notice (24 hours)

**Optional**: Unverified Account Restrictions
- Unverified accounts cannot:
  - Receive email notifications
  - Export data
  - Access certain features
- Banner in UI prompting verification

**Acceptance Criteria**:
- New users receive verification email
- Email verification completes successfully
- Invalid/expired tokens are rejected
- Users can resend verification emails
- Unverified accounts are clearly marked

### FR-4: Welcome & Onboarding Emails

**Requirement**: New users receive welcome email and onboarding sequence.

**Details**:
- Welcome email sent immediately after registration
- Onboarding email sequence (3-5 emails over first week)
- Emails include tips, feature highlights, getting started guides
- Users can opt-out of onboarding sequence (keep transactional emails)

**Email Sequence**:
1. **Welcome Email** (Immediate)
   - Subject: "Welcome to Money Planner!"
   - Content: Thank you, quick start guide, key features
   - CTA: "Get Started" button

2. **Getting Started** (Day 1)
   - Subject: "Let's Set Up Your First Budget"
   - Content: Budget creation tutorial, example budgets
   - CTA: "Create Budget" button

3. **Transaction Tips** (Day 3)
   - Subject: "Track Your Spending Like a Pro"
   - Content: Transaction categorization, import tips
   - CTA: "Add Transaction" button

4. **Budget Insights** (Day 5)
   - Subject: "Understanding Your Budget Performance"
   - Content: Budget alerts, settlement summaries, reports
   - CTA: "View Budgets" button

5. **Advanced Features** (Day 7)
   - Subject: "Unlock Advanced Money Planner Features"
   - Content: Recurring transactions, categories, exports
   - CTA: "Explore Features" button

**Database Schema**:
```sql
CREATE TABLE email_sequences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    sequence_type VARCHAR(50) NOT NULL, -- 'onboarding', 're-engagement', etc.
    email_index INTEGER NOT NULL, -- 0, 1, 2, etc.
    sent_at TIMESTAMP,
    scheduled_for TIMESTAMP NOT NULL,
    status VARCHAR(20) NOT NULL, -- 'pending', 'sent', 'skipped', 'cancelled'
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

**Acceptance Criteria**:
- Welcome email sent after registration
- Onboarding sequence emails sent on schedule
- Users can opt-out of onboarding emails
- Sequence stops if user becomes inactive (optional)

### FR-5: Budget & Transaction Notifications

**Requirement**: Users receive email notifications for important budget and transaction events.

**Details**:
- Budget alerts (approaching limit, exceeded)
- Large transaction notifications
- Budget settlement summaries
- Weekly/monthly spending summaries
- Unusual activity alerts

**Budget Alerts**:

**Budget Threshold Alerts**:
- Triggered when budget spending reaches thresholds (50%, 75%, 90%, 100%)
- Configurable per budget or global defaults
- Daily digest option (one email per day with all alerts)

**Budget Exceeded Alerts**:
- Immediate notification when budget is exceeded
- Includes category breakdown, overspend amount
- Suggestions for reducing spending

**Budget Settlement Summaries**:
- Sent when budget period ends (monthly/weekly/yearly)
- Includes period summary, category breakdown, insights
- Link to detailed settlement report
- Optional PDF attachment

**Transaction Notifications**:

**Large Transaction Alerts**:
- Notify when transaction exceeds threshold (configurable, e.g., $500)
- Immediate notification
- Includes transaction details, category, merchant

**Unusual Activity Alerts**:
- Multiple transactions in short time
- Transactions in new categories
- Transactions in new locations (if location data available)

**Periodic Summaries**:

**Weekly Spending Summary**:
- Sent every Monday
- Previous week's spending by category
- Budget performance summary
- Top spending categories

**Monthly Spending Summary**:
- Sent on 1st of each month
- Previous month's complete summary
- Budget performance across all budgets
- Year-over-year comparison (if data available)
- Spending trends and insights

**Database Schema**:
```sql
CREATE TABLE notification_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    notification_type VARCHAR(50) NOT NULL, -- 'budget_alert', 'transaction_alert', etc.
    enabled BOOLEAN NOT NULL DEFAULT true,
    threshold DECIMAL(10,2), -- For threshold-based notifications
    frequency VARCHAR(20), -- 'immediate', 'daily_digest', 'weekly'
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, notification_type)
);
```

**API Endpoints**:
- `GET /v1/users/{id}/notification-preferences` - Get user preferences
- `PUT /v1/users/{id}/notification-preferences` - Update preferences
- `POST /v1/users/{id}/notification-preferences/test` - Send test notification

**Email Templates**:
- Budget Alert Email
- Budget Exceeded Email
- Budget Settlement Summary Email
- Large Transaction Alert Email
- Weekly Spending Summary Email
- Monthly Spending Summary Email

**Acceptance Criteria**:
- Budget alerts sent at configured thresholds
- Transaction notifications sent for large/unusual transactions
- Settlement summaries sent when periods end
- Periodic summaries sent on schedule
- Users can configure notification preferences
- Email preferences are respected

### FR-6: Security & Account Alerts

**Requirement**: Users receive email notifications for security-related events.

**Details**:
- Login from new device/location
- Password changed
- Email address changed
- Account locked (too many failed login attempts)
- Suspicious activity detected

**Security Alerts**:

**New Device Login**:
- Triggered when user logs in from unrecognized device/browser
- Includes device info, location (if available), timestamp
- Link to view active sessions
- "This wasn't me" link to secure account

**Password Changed**:
- Immediate notification when password is changed
- Includes timestamp, IP address (if available)
- "This wasn't me" link if unauthorized

**Email Changed**:
- Notification sent to old email address
- Confirmation required (optional)
- Notification sent to new email address

**Account Locked**:
- Notification when account locked due to failed login attempts
- Instructions for unlocking
- Support contact information

**Suspicious Activity**:
- Multiple failed login attempts
- Unusual transaction patterns (if integrated with transaction service)
- Account access from multiple locations simultaneously

**Email Templates**:
- New Device Login Alert
- Password Changed Alert
- Email Changed Alert
- Account Locked Alert
- Suspicious Activity Alert

**Acceptance Criteria**:
- Security alerts sent for all security events
- Alerts include actionable information
- Users can report unauthorized activity
- Alerts cannot be disabled (security-critical)

### FR-7: Marketing & Engagement Emails

**Requirement**: System can send marketing emails for user engagement and feature announcements.

**Details**:
- Newsletter emails
- Feature announcements
- Product updates
- Re-engagement campaigns for inactive users
- Promotional emails (optional, if applicable)

**Newsletter**:
- Periodic newsletter (monthly/quarterly)
- Product updates, tips, user stories
- Opt-in required (GDPR compliance)
- Unsubscribe link in every email

**Feature Announcements**:
- New feature releases
- Feature improvements
- UI/UX updates
- Sent to all active users or targeted segments

**Re-engagement Campaigns**:
- Triggered for inactive users (no login in 30/60/90 days)
- Personalized content based on user history
- Special offers or incentives (optional)
- "We miss you" messaging

**Email Segmentation**:
- Active users
- Inactive users
- New users (< 30 days)
- Power users (high transaction volume)
- Budget-focused users
- Transaction-focused users

**Database Schema**:
```sql
CREATE TABLE marketing_campaigns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    campaign_type VARCHAR(50) NOT NULL, -- 'newsletter', 'feature_announcement', 're_engagement'
    subject VARCHAR(255) NOT NULL,
    template_id UUID REFERENCES email_templates(id),
    target_segment VARCHAR(50), -- 'all', 'active', 'inactive', etc.
    scheduled_for TIMESTAMP,
    sent_at TIMESTAMP,
    status VARCHAR(20) NOT NULL, -- 'draft', 'scheduled', 'sending', 'sent', 'cancelled'
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by UUID REFERENCES users(id) -- Admin user
);

CREATE TABLE marketing_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    subscribed BOOLEAN NOT NULL DEFAULT true,
    subscribed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    unsubscribed_at TIMESTAMP,
    preferences JSONB, -- Granular preferences (newsletter, feature_announcements, etc.)
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(user_id)
);
```

**API Endpoints** (Admin/Backoffice):
- `POST /v1/admin/marketing/campaigns` - Create campaign
- `GET /v1/admin/marketing/campaigns` - List campaigns
- `POST /v1/admin/marketing/campaigns/{id}/send` - Send campaign
- `GET /v1/users/{id}/marketing-subscription` - Get user subscription
- `PUT /v1/users/{id}/marketing-subscription` - Update subscription
- `POST /v1/users/{id}/marketing-subscription/unsubscribe` - Unsubscribe

**Email Templates**:
- Newsletter Template
- Feature Announcement Template
- Re-engagement Template

**Acceptance Criteria**:
- Marketing emails can be created and scheduled
- Users can opt-in/opt-out of marketing emails
- Unsubscribe links work correctly
- Campaigns can target user segments
- Email delivery is tracked

### FR-8: Email Templates & Content Management

**Requirement**: System must support email templates with dynamic content.

**Details**:
- Template engine (Handlebars, Mustache, or similar)
- HTML and plain text versions
- Dynamic variables (user name, amounts, dates, etc.)
- Template versioning
- Preview functionality
- A/B testing support (optional)

**Template Variables**:
- User data: `{{user.name}}`, `{{user.email}}`
- Transaction data: `{{transaction.amount}}`, `{{transaction.category}}`
- Budget data: `{{budget.name}}`, `{{budget.spent}}`, `{{budget.limit}}`
- System data: `{{app.url}}`, `{{support.email}}`
- Dates: `{{date}}`, `{{date.format}}`

**Database Schema**:
```sql
CREATE TABLE email_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    template_type VARCHAR(50) NOT NULL, -- 'password_reset', 'welcome', 'budget_alert', etc.
    subject VARCHAR(255) NOT NULL,
    html_body TEXT NOT NULL,
    text_body TEXT,
    variables JSONB, -- Available variables for this template
    version INTEGER NOT NULL DEFAULT 1,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by UUID REFERENCES users(id)
);

CREATE TABLE email_template_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id UUID NOT NULL REFERENCES email_templates(id),
    version INTEGER NOT NULL,
    subject VARCHAR(255) NOT NULL,
    html_body TEXT NOT NULL,
    text_body TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by UUID REFERENCES users(id),
    UNIQUE(template_id, version)
);
```

**API Endpoints** (Admin):
- `GET /v1/admin/email-templates` - List templates
- `GET /v1/admin/email-templates/{id}` - Get template
- `POST /v1/admin/email-templates` - Create template
- `PUT /v1/admin/email-templates/{id}` - Update template
- `POST /v1/admin/email-templates/{id}/preview` - Preview with sample data
- `POST /v1/admin/email-templates/{id}/test` - Send test email

**Acceptance Criteria**:
- Templates support dynamic variables
- HTML and plain text versions available
- Templates can be created and updated
- Template preview works correctly
- Version history is maintained

### FR-9: Email Queue & Delivery Tracking

**Requirement**: System must queue emails and track delivery status.

**Details**:
- Email queue for reliable delivery
- Retry failed emails with exponential backoff
- Delivery status tracking (sent, delivered, bounced, failed)
- Bounce and complaint handling
- Rate limiting per email provider

**Database Schema**:
```sql
CREATE TABLE email_queue (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    to_email VARCHAR(255) NOT NULL,
    from_email VARCHAR(255) NOT NULL,
    subject VARCHAR(255) NOT NULL,
    template_id UUID REFERENCES email_templates(id),
    template_variables JSONB,
    priority INTEGER NOT NULL DEFAULT 5, -- 1-10, higher = more important
    status VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending', 'sending', 'sent', 'delivered', 'bounced', 'failed'
    attempts INTEGER NOT NULL DEFAULT 0,
    max_attempts INTEGER NOT NULL DEFAULT 3,
    scheduled_for TIMESTAMP NOT NULL DEFAULT NOW(),
    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    error_message TEXT,
    provider_response JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_email_queue_status ON email_queue(status, scheduled_for);
CREATE INDEX idx_email_queue_user ON email_queue(user_id);

CREATE TABLE email_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email_queue_id UUID REFERENCES email_queue(id),
    user_id UUID REFERENCES users(id),
    to_email VARCHAR(255) NOT NULL,
    subject VARCHAR(255) NOT NULL,
    template_type VARCHAR(50),
    status VARCHAR(20) NOT NULL,
    provider VARCHAR(50), -- 'sendgrid', 'ses', 'smtp', etc.
    provider_message_id VARCHAR(255),
    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    opened_at TIMESTAMP,
    clicked_at TIMESTAMP,
    bounced_at TIMESTAMP,
    bounce_reason TEXT,
    complaint_at TIMESTAMP,
    complaint_reason TEXT,
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_email_logs_user ON email_logs(user_id, created_at);
CREATE INDEX idx_email_logs_status ON email_logs(status, created_at);
```

**Queue Processing**:
- Background worker processes email queue
- Processes emails in priority order
- Retries failed emails with exponential backoff
- Marks emails as failed after max attempts
- Handles provider rate limits

**Bounce Handling**:
- Track hard bounces (invalid email)
- Track soft bounces (temporary issues)
- Mark emails as invalid after multiple hard bounces
- Suppress future emails to invalid addresses

**Complaint Handling**:
- Track spam complaints
- Automatically unsubscribe users who complain
- Monitor complaint rate (high rate = provider issues)

**Acceptance Criteria**:
- Emails are queued reliably
- Failed emails are retried
- Delivery status is tracked
- Bounces and complaints are handled
- Queue processing is efficient

### FR-10: Email Preferences Management

**Requirement**: Users can manage their email preferences.

**Details**:
- Granular control over email types
- Opt-in/opt-out for marketing emails
- Frequency preferences (immediate, daily digest, weekly)
- Transactional emails cannot be disabled (security-critical)

**Email Categories**:
1. **Transactional** (Cannot disable):
   - Password reset
   - Email verification
   - Security alerts
   - Account changes

2. **Notifications** (Can disable):
   - Budget alerts
   - Transaction notifications
   - Settlement summaries
   - Periodic summaries

3. **Marketing** (Opt-in):
   - Newsletter
   - Feature announcements
   - Promotional emails

**Database Schema**:
```sql
-- Already defined in FR-5 (notification_preferences)
-- Add marketing preferences:

ALTER TABLE users ADD COLUMN email_preferences JSONB DEFAULT '{
    "marketing": {
        "newsletter": false,
        "feature_announcements": false,
        "promotional": false
    },
    "notifications": {
        "budget_alerts": true,
        "transaction_alerts": true,
        "settlement_summaries": true,
        "periodic_summaries": true
    },
    "frequency": {
        "budget_alerts": "immediate",
        "transaction_alerts": "immediate",
        "periodic_summaries": "weekly"
    }
}'::jsonb;
```

**UI Components**:
- Email preferences page in user settings
- Toggle switches for each email type
- Frequency dropdowns
- "Unsubscribe from all marketing" button
- Preview of email types

**API Endpoints**:
- `GET /v1/users/{id}/email-preferences` - Get preferences
- `PUT /v1/users/{id}/email-preferences` - Update preferences
- `POST /v1/users/{id}/email-preferences/unsubscribe-all` - Unsubscribe from all marketing

**Acceptance Criteria**:
- Users can view and update email preferences
- Preferences are respected when sending emails
- Transactional emails cannot be disabled
- Unsubscribe links work correctly

## Non-Functional Requirements

### NFR-1: Email Delivery Reliability
- **Requirement**: 99.9% email delivery success rate
- **Details**: Retry mechanism, provider failover, monitoring

### NFR-2: Email Delivery Speed
- **Requirement**: Transactional emails delivered within 30 seconds
- **Details**: Priority queue, immediate processing for critical emails

### NFR-3: Scalability
- **Requirement**: Support 10,000+ emails per hour
- **Details**: Queue-based architecture, horizontal scaling

### NFR-4: Security
- **Requirement**: Secure token generation, email content encryption (optional)
- **Details**: Cryptographically secure tokens, HTTPS for email links

### NFR-5: Compliance
- **Requirement**: GDPR, CAN-SPAM compliance
- **Details**: Unsubscribe links, opt-in for marketing, data retention policies

### NFR-6: Monitoring & Observability
- **Requirement**: Email delivery metrics, error tracking, alerting
- **Details**: Delivery rates, bounce rates, complaint rates, failed email alerts

## Technical Architecture

### Service Architecture Options

**Option A: Dedicated Email Service** (Recommended for Production)

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
       ├──────────────┬──────────────┬──────────────┐
       ▼              ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│User Service │ │Transaction  │ │Budget      │ │Email Service│
│             │ │Service      │ │Service     │ │             │
└─────────────┘ └─────────────┘ └─────────────┘ └──────┬──────┘
                                                        │
                                                        ▼
                                              ┌─────────────────┐
                                              │ Email Provider  │
                                              │ (SendGrid/SES)  │
                                              └─────────────────┘
```

**Option B: Email Module in User Service** (Recommended for MVP)

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
│User Service │ │Transaction  │ │Budget      │
│(with email) │ │Service      │ │Service     │
└──────┬──────┘ └─────────────┘ └─────────────┘
       │
       ▼
┌─────────────────┐
│ Email Provider  │
│ (SendGrid/SES)  │
└─────────────────┘
```

### Technology Stack Recommendations

**Email Service Provider**:
- **SendGrid** (Recommended): Easy integration, good deliverability, free tier
- **AWS SES**: Cost-effective, good for high volume
- **Mailgun**: Developer-friendly, good API
- **SMTP**: For self-hosted (not recommended for production)

**Template Engine**:
- **Handlebars**: Popular, good Rust support
- **Tera**: Rust-native template engine
- **Askama**: Compile-time templates in Rust

**Queue System**:
- **PostgreSQL-based queue**: Simple, uses existing database
- **Redis Queue**: Fast, good for high volume
- **RabbitMQ**: Robust, good for complex workflows
- **Background job library**: `tokio-cron-scheduler` or `background-jobs` for Rust

**Email Library (Rust)**:
- `lettre`: SMTP client
- `sendgrid-rs`: SendGrid API client
- `rusoto_ses`: AWS SES client

### Database Migrations

**User Service Database**:
```sql
-- Email verification
ALTER TABLE users ADD COLUMN email_verified BOOLEAN NOT NULL DEFAULT false;
ALTER TABLE users ADD COLUMN email_verification_token VARCHAR(255);
ALTER TABLE users ADD COLUMN email_verification_sent_at TIMESTAMP;
ALTER TABLE users ADD COLUMN email_verified_at TIMESTAMP;

-- Password reset
ALTER TABLE users ADD COLUMN password_reset_token VARCHAR(255);
ALTER TABLE users ADD COLUMN password_reset_token_expires_at TIMESTAMP;
ALTER TABLE users ADD COLUMN password_reset_requested_at TIMESTAMP;

-- Email preferences
ALTER TABLE users ADD COLUMN email_preferences JSONB DEFAULT '{
    "marketing": {"newsletter": false, "feature_announcements": false},
    "notifications": {"budget_alerts": true, "transaction_alerts": true},
    "frequency": {"budget_alerts": "immediate", "periodic_summaries": "weekly"}
}'::jsonb;

-- Email queue and logs (if email service in user-service)
CREATE TABLE email_queue (...);
CREATE TABLE email_logs (...);
CREATE TABLE email_templates (...);
CREATE TABLE notification_preferences (...);
```

## Implementation Plan

### Phase 0: Foundation (Weeks 1-2)

**Goal**: Set up email infrastructure and basic sending capability

**Tasks**:
1. Choose email service provider (SendGrid recommended)
2. Set up email service account and API keys
3. Create email service module/library
4. Implement basic email sending function
5. Set up email templates (basic HTML/text)
6. Configure environment variables
7. Add email configuration to user-service

**Deliverables**:
- Email service integrated
- Can send test emails
- Configuration management

### Phase 1: Password Reset (Weeks 3-4)

**Goal**: Implement password reset functionality

**Tasks**:
1. Add password reset database fields
2. Implement password reset token generation
3. Create password reset API endpoints
4. Create password reset email template
5. Implement frontend password reset UI
6. Add rate limiting
7. Write integration tests

**Deliverables**:
- Password reset flow complete
- Users can reset passwords via email
- Security requirements met

### Phase 2: Email Verification (Weeks 5-6)

**Goal**: Implement email verification for new users

**Tasks**:
1. Add email verification database fields
2. Implement verification token generation
3. Create verification API endpoints
4. Create verification email template
5. Update user registration to send verification email
6. Implement frontend verification UI
7. Add resend verification functionality
8. Write integration tests

**Deliverables**:
- Email verification flow complete
- New users verify emails
- Unverified account handling (optional)

### Phase 3: Email Queue & Templates (Weeks 7-8)

**Goal**: Set up email queue and template system

**Tasks**:
1. Create email queue database tables
2. Implement queue processing worker
3. Implement template engine
4. Create email template management API
5. Migrate existing emails to template system
6. Add retry logic and error handling
7. Add email logging

**Deliverables**:
- Email queue operational
- Template system functional
- Reliable email delivery

### Phase 4: Welcome & Onboarding (Weeks 9-10)

**Goal**: Implement welcome and onboarding email sequence

**Tasks**:
1. Create welcome email template
2. Create onboarding email templates (3-5 emails)
3. Implement email sequence scheduler
4. Create sequence management API
5. Add opt-out functionality
6. Update user registration to trigger welcome email
7. Write integration tests

**Deliverables**:
- Welcome email sent on registration
- Onboarding sequence functional
- Users can opt-out

### Phase 5: Budget & Transaction Notifications (Weeks 11-14)

**Goal**: Implement budget and transaction email notifications

**Tasks**:
1. Create notification preferences database
2. Implement budget alert detection
3. Create budget alert email templates
4. Integrate with budget service (Kafka events or HTTP)
5. Implement large transaction detection
6. Create transaction notification templates
7. Implement periodic summary emails
8. Create notification preferences API
9. Add frontend notification preferences UI
10. Write integration tests

**Deliverables**:
- Budget alerts functional
- Transaction notifications functional
- Periodic summaries functional
- User preferences respected

### Phase 6: Security Alerts (Weeks 15-16)

**Goal**: Implement security-related email alerts

**Tasks**:
1. Create security alert email templates
2. Implement new device detection
3. Add security alert triggers to user service
4. Integrate with session management
5. Add "This wasn't me" functionality
6. Write integration tests

**Deliverables**:
- Security alerts functional
- Users notified of security events

### Phase 7: Marketing Emails (Weeks 17-20)

**Goal**: Implement marketing email capabilities

**Tasks**:
1. Create marketing subscription database
2. Create marketing campaign management API
3. Create marketing email templates
4. Implement user segmentation
5. Create campaign scheduling
6. Add unsubscribe functionality
7. Create admin UI for campaigns (if backoffice exists)
8. Write integration tests

**Deliverables**:
- Marketing emails functional
- Campaign management operational
- GDPR compliance met

### Phase 8: Polish & Optimization (Weeks 21-22)

**Goal**: Optimize and polish email system

**Tasks**:
1. Performance optimization
2. Monitoring and alerting setup
3. Documentation
4. Load testing
5. Security audit
6. User acceptance testing

**Deliverables**:
- Production-ready email system
- Monitoring in place
- Documentation complete

## Testing Strategy

### Unit Tests
- Email service functions
- Token generation and validation
- Template rendering
- Queue processing logic

### Integration Tests
- Password reset flow end-to-end
- Email verification flow end-to-end
- Email sending with mock provider
- Queue processing
- Notification triggering

### E2E Tests
- Complete user flows (registration → verification → password reset)
- Email delivery verification
- Unsubscribe flow
- Notification preferences

### Load Tests
- Email queue processing under load
- Concurrent email sending
- Provider rate limit handling

## Security Considerations

### Token Security
- Use cryptographically secure random generators
- Tokens must be sufficiently long (32+ bytes)
- Tokens must be single-use
- Tokens must expire
- Tokens must be stored hashed (optional, for password reset)

### Email Security
- HTTPS for all email links
- No sensitive data in email content (use tokens)
- Rate limiting to prevent abuse
- Email enumeration prevention

### Data Privacy
- GDPR compliance (opt-in for marketing)
- Unsubscribe functionality
- Data retention policies
- Email content encryption (optional)

### Provider Security
- API keys stored securely (environment variables, secrets management)
- Rotate API keys regularly
- Monitor for compromised keys
- Use separate keys for different environments

## Monitoring & Observability

### Metrics to Track
- Email delivery rate
- Email bounce rate
- Email complaint rate
- Queue processing time
- Failed email count
- Provider API errors
- Token generation/validation errors

### Alerts
- High bounce rate (> 5%)
- High complaint rate (> 0.1%)
- Queue backup (emails not processing)
- Provider API failures
- High failed email rate

### Logging
- All email sends (with user ID, template, status)
- Token generation/validation
- Queue processing events
- Provider API calls and responses
- Errors and exceptions

## Open Questions & Decisions Needed

1. **Email Service Architecture**: Dedicated service vs. module in user-service?
   - **Recommendation**: Start with module in user-service (MVP), migrate to dedicated service later

2. **Email Provider**: SendGrid, AWS SES, Mailgun, or SMTP?
   - **Recommendation**: SendGrid for MVP (easy setup, good free tier)

3. **Template Engine**: Handlebars, Tera, or Askama?
   - **Recommendation**: Tera (Rust-native, good performance)

4. **Queue System**: PostgreSQL queue, Redis, or RabbitMQ?
   - **Recommendation**: PostgreSQL queue for MVP (uses existing DB), migrate to Redis later if needed

5. **Unverified Account Restrictions**: Should unverified accounts have limited functionality?
   - **Recommendation**: Yes, but minimal restrictions (can't receive emails, can't export data)

6. **Marketing Email Opt-in**: Default opt-in or opt-out?
   - **Recommendation**: Opt-out for MVP (GDPR requires opt-in, but can be implemented later)

7. **Email Frequency Limits**: Should there be daily/weekly limits per user?
   - **Recommendation**: Yes, prevent email fatigue (max 10 transactional + 5 marketing per day)

8. **Budget Alert Thresholds**: Default thresholds and configurability?
   - **Recommendation**: Default 50%, 75%, 90%, 100%; user-configurable per budget

## Success Metrics

### User Engagement
- Email open rate > 20%
- Email click rate > 5%
- Password reset completion rate > 80%
- Email verification rate > 90%

### System Performance
- Email delivery success rate > 99.9%
- Average delivery time < 30 seconds (transactional)
- Queue processing lag < 5 minutes
- Bounce rate < 2%

### Business Impact
- Reduced support tickets for password resets
- Increased user activation (email verification)
- Improved user retention (onboarding emails)
- Higher engagement (notifications)

## Dependencies

### External Dependencies
- Email service provider account (SendGrid/SES)
- SMTP server (if self-hosting)

### Internal Dependencies
- User service (authentication, user data)
- Budget service (budget data, settlement events)
- Transaction service (transaction data, events)
- BFF (API endpoints)
- Frontend (UI components)

### Infrastructure Dependencies
- Database (PostgreSQL)
- Background job processing
- Monitoring and logging infrastructure

## Risks & Mitigations

### Risk 1: Email Deliverability Issues
- **Impact**: High - Emails not reaching users
- **Mitigation**: Use reputable provider, monitor bounce rates, maintain sender reputation

### Risk 2: Provider Rate Limits
- **Impact**: Medium - Emails delayed or failed
- **Mitigation**: Queue system, rate limiting, provider failover

### Risk 3: Security Vulnerabilities
- **Impact**: High - Account compromise
- **Mitigation**: Secure token generation, rate limiting, security audits

### Risk 4: Email Spam Classification
- **Impact**: Medium - Emails marked as spam
- **Mitigation**: SPF/DKIM/DMARC setup, content best practices, monitor complaint rates

### Risk 5: High Costs at Scale
- **Impact**: Medium - Email provider costs
- **Mitigation**: Optimize email frequency, use cost-effective provider, monitor usage

## Future Enhancements

### Post-MVP Features
- Email A/B testing
- Advanced segmentation
- Personalized content based on user behavior
- Email analytics dashboard
- SMS notifications (alternative channel)
- Push notifications (in-app)
- Email scheduling optimization (send time optimization)
- Multi-language email support
- Rich email templates with images
- Email attachments (PDF reports, CSV exports)

## Appendix

### Email Template Examples

**Password Reset Email**:
```
Subject: Reset Your Money Planner Password

Hi {{user.name}},

You requested to reset your password. Click the link below to create a new password:

[Reset Password]({{reset_link}})

This link will expire in 15 minutes.

If you didn't request this, please ignore this email or contact support.

Best regards,
Money Planner Team
```

**Welcome Email**:
```
Subject: Welcome to Money Planner!

Hi {{user.name}},

Welcome to Money Planner! We're excited to help you take control of your finances.

Get started by:
1. Creating your first budget
2. Adding your transactions
3. Setting up categories

[Get Started]({{app_url}}/dashboard)

Need help? Check out our [Getting Started Guide]({{docs_url}}).

Best regards,
Money Planner Team
```

**Budget Alert Email**:
```
Subject: Budget Alert: {{budget.name}} at {{percentage}}%

Hi {{user.name}},

Your budget "{{budget.name}}" has reached {{percentage}}% of its limit.

Current spending: ${{spent}}
Budget limit: ${{limit}}
Remaining: ${{remaining}}

[View Budget]({{budget_url}})

Best regards,
Money Planner Team
```

### API Endpoint Specifications

See OpenAPI/Swagger documentation for detailed API specifications (to be created during implementation).

### Database Schema Diagrams

See database migration files for complete schema definitions (to be created during implementation).

