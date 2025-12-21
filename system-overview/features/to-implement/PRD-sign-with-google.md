# PRD: Google Sign-In Integration

## Document Information
- **Version**: 1.0
- **Date**: January 2025
- **Status**: Draft
- **Author**: Development Team

## Executive Summary

The Google Sign-In feature enables users to authenticate and register using their Google accounts, eliminating the need to create and remember separate credentials for Money Planner. This feature improves user onboarding experience, reduces friction in the registration process, and provides a secure, industry-standard authentication method alongside the existing email/password system.

**Key Benefits:**
- Reduced registration friction - users can sign up with one click
- Improved security - leverages Google's robust authentication infrastructure
- Better user experience - no need to remember another password
- Account linking - existing users can link their Google accounts
- Industry standard - OAuth 2.0 compliant authentication

## Problem Statement

Currently, users must create a new account with email and password to use Money Planner. This creates several friction points:

- **Registration Friction**: Users must fill out forms, create strong passwords, and verify emails
- **Password Fatigue**: Users already manage many passwords and may reuse weak passwords
- **Account Recovery**: Password reset flows add complexity and potential security issues
- **User Drop-off**: Complex registration processes lead to abandoned signups
- **Security Concerns**: Users may choose weak passwords or reuse passwords from other services

Without Google Sign-In, users who prefer OAuth authentication cannot use their preferred method, leading to potential user loss and reduced conversion rates.

## Current System Analysis

### Existing Authentication Architecture

**User Service** (port 8080):
- **Registration**: `POST /v1/users` - Creates user with email and password
- **Login**: `POST /v1/auth/login` - Authenticates with email/password, creates session
- **Session Management**: Session-based authentication with UUID session IDs
- **Password Security**: Argon2 hashing with validation (8-100 chars, uppercase, lowercase, number, special)
- **Database Schema**:
  ```sql
  CREATE TABLE users (
      id UUID PRIMARY KEY,
      password VARCHAR(250) NOT NULL,
      email VARCHAR(250) NOT NULL UNIQUE,
      created_at TIMESTAMPTZ NOT NULL,
      state VARCHAR(25) NOT NULL,  -- Active, Blocked
      role VARCHAR(25) NOT NULL    -- Customer, Staff, Admin
  );
  
  CREATE TABLE sessions (
      id UUID PRIMARY KEY,
      user_id UUID NOT NULL,
      created_at TIMESTAMPTZ NOT NULL,
      expires_at TIMESTAMPTZ NOT NULL,
      state VARCHAR(25) NOT NULL
  );
  ```

**BFF Service** (port 8082):
- **API Gateway**: Routes requests to appropriate microservices
- **Session Validation**: Validates session tokens via User Service `/v1/auth/session`
- **Authentication Flow**: Frontend → BFF (session token) → Backend Services (JWT service tokens)
- **Endpoints**:
  - `POST /v1/users` - User registration (no auth required)
  - `POST /v1/auth/login` - User login (no auth required)
  - `DELETE /v1/auth/logout` - User logout (requires session)

**Frontend** (port 3000):
- **Signup Form**: `SignupForm.tsx` - Email, password, confirm password fields
- **Signin Form**: `SigninForm.tsx` - Email and password fields
- **Auth Context**: `AuthContext.tsx` - Manages session tokens in localStorage
- **Token Storage**: `access_token` and `user_id` stored in localStorage
- **Session Management**: 30-minute session timeout with activity tracking

### Key Limitations

**Current System Constraints:**
- **Password Required**: `users.password` is `NOT NULL` - cannot create users without passwords
- **No OAuth Support**: No fields to store OAuth provider information or external IDs
- **No Account Linking**: Cannot link multiple authentication methods to same account
- **Email Uniqueness**: Email is unique, but no mechanism to handle OAuth accounts with same email
- **Session-Based Only**: Current flow assumes password-based authentication

**Integration Points:**
- User Service must support OAuth user creation
- BFF must handle OAuth callback flow
- Frontend must integrate Google OAuth SDK
- Database schema must support OAuth providers

## User Stories

### Primary User Stories

**As a new user**, I want to sign up using my Google account, so that I can start using Money Planner without creating a new password.

**As an existing user**, I want to sign in using my Google account, so that I don't need to remember my Money Planner password.

**As an existing user with email/password**, I want to link my Google account, so that I can use either authentication method.

**As a user**, I want to unlink my Google account, so that I can remove OAuth access if needed.

### Secondary User Stories

- **As a user**, I want to see clear indication when signing in with Google, so that I understand the authentication method being used
- **As a user**, I want to be able to switch between Google and email/password login, so that I have flexibility in authentication
- **As a user**, I want my Google account information to be securely handled, so that my privacy is protected
- **As a user**, I want to see which authentication methods are linked to my account, so that I can manage my account security

## Functional Requirements

### FR-1: Google OAuth Integration

**Requirement**: System must support Google OAuth 2.0 authentication flow.

**Details**:
- Use Google OAuth 2.0 OpenID Connect (OIDC) for authentication
- Support both sign-up and sign-in flows
- Verify Google ID tokens on the backend
- Extract user information (email, name, profile picture) from Google tokens
- Handle OAuth callback redirects securely

**Acceptance Criteria**:
- Users can initiate Google sign-in from frontend
- Google OAuth consent screen displays correctly
- ID tokens are validated against Google's public keys
- User information is extracted from verified tokens
- OAuth flow completes without exposing sensitive data

### FR-2: OAuth User Registration

**Requirement**: System must create user accounts from Google OAuth authentication.

**Details**:
- Create user record when Google sign-in is used for new users
- Store Google user ID for future authentication
- Set default user role (Customer) and state (Active)
- Extract and store email from Google profile
- Handle email conflicts (existing user with same email)

**Acceptance Criteria**:
- New users can register using Google account
- User record created with Google ID stored
- Email extracted from Google profile
- Default role and state set correctly
- Session created after successful registration

### FR-3: OAuth User Login

**Requirement**: System must authenticate existing users via Google OAuth.

**Details**:
- Verify Google ID token on backend
- Look up user by Google ID or email
- Create session for authenticated user
- Return session token to frontend
- Handle blocked users appropriately

**Acceptance Criteria**:
- Existing users can sign in with Google
- User lookup works by Google ID or email
- Session created successfully
- Blocked users cannot authenticate
- Session token returned to frontend

### FR-4: Account Linking

**Requirement**: Existing email/password users must be able to link Google accounts.

**Details**:
- Allow authenticated users to link Google account
- Store Google ID for existing users
- Support multiple authentication methods per user
- Prevent duplicate account creation
- Handle email mismatches (Google email != account email)

**Acceptance Criteria**:
- Authenticated users can link Google account
- Google ID stored for existing user
- User can login with either method after linking
- Email mismatch handled gracefully
- Cannot link Google account already linked to another user

### FR-5: Account Unlinking

**Requirement**: Users must be able to unlink Google accounts.

**Details**:
- Allow users to remove Google account link
- Require at least one authentication method (cannot unlink if no password)
- Prevent account lockout (must have password before unlinking Google)
- Clear Google ID from user record
- Maintain user account and data

**Acceptance Criteria**:
- Users can unlink Google account
- Cannot unlink if no password set
- Google ID removed from user record
- User can still login with password
- Account data preserved

### FR-6: Database Schema Updates

**Requirement**: Database must support OAuth provider information.

**Details**:
- Add `google_id` field to users table (nullable, unique)
- Add `auth_provider` field to track authentication method
- Add `password` nullable constraint (OAuth users don't need passwords)
- Add index on `google_id` for efficient lookups
- Support migration of existing users

**Acceptance Criteria**:
- Database migration runs successfully
- Existing users unaffected
- Google ID stored and retrievable
- Efficient queries by Google ID
- Nullable password constraint works

### FR-7: Frontend Google Sign-In Button

**Requirement**: Frontend must provide Google sign-in UI.

**Details**:
- Add "Sign in with Google" button to signup and signin forms
- Integrate Google OAuth JavaScript SDK
- Handle OAuth callback redirects
- Display loading states during authentication
- Show error messages for failed authentication

**Acceptance Criteria**:
- Google sign-in button visible on forms
- OAuth flow initiates correctly
- Callback handled properly
- Loading states displayed
- Error messages shown on failure

### FR-8: Backend Token Verification

**Requirement**: Backend must verify Google ID tokens.

**Details**:
- Verify token signature using Google's public keys
- Validate token expiration
- Verify token audience (client ID)
- Extract user information from verified token
- Handle invalid or expired tokens

**Acceptance Criteria**:
- Invalid tokens rejected
- Expired tokens rejected
- Token audience validated
- User info extracted correctly
- Security errors logged

## Technical Specifications

### Architecture Changes

#### Database Schema Updates

**Migration: Add OAuth Support to Users Table**

```sql
-- Add Google ID column (nullable, unique)
ALTER TABLE users ADD COLUMN google_id VARCHAR(255) UNIQUE;

-- Add auth provider tracking
ALTER TABLE users ADD COLUMN auth_provider VARCHAR(50) DEFAULT 'email';

-- Make password nullable (OAuth users don't need passwords)
ALTER TABLE users ALTER COLUMN password DROP NOT NULL;

-- Add index for Google ID lookups
CREATE INDEX idx_users_google_id ON users(google_id) WHERE google_id IS NOT NULL;

-- Update existing users to have 'email' as auth_provider
UPDATE users SET auth_provider = 'email' WHERE auth_provider IS NULL;
```

**New User Entity Structure**:
```rust
pub struct User {
    pub id: UserId,
    pub password: Option<UserPassword>,  // Nullable for OAuth users
    pub email: String,
    pub google_id: Option<String>,       // Google OAuth ID
    pub auth_provider: AuthProvider,     // email, google, or both
    pub created_at: DateTime<Utc>,
    pub state: UserState,
    pub role: UserRole,
}

#[derive(Debug, Clone, sqlx::Type, PartialEq, Eq)]
#[sqlx(type_name = "varchar")]
pub enum AuthProvider {
    Email,      // Password-based only
    Google,     // Google OAuth only
    Both,       // Both methods linked
}
```

#### User Service API Changes

**New Endpoints**:

1. **Google Sign-In Endpoint**
   - `POST /v1/auth/google`
   - Request: `{ "id_token": "..." }`
   - Response: `{ "session_id": "...", "user_id": "..." }`
   - Behavior:
     - Verifies Google ID token
     - Creates user if new, or logs in existing user
     - Creates session
     - Returns session token

2. **Link Google Account**
   - `POST /v1/users/{id}/link-google`
   - Requires: Authenticated session
   - Request: `{ "id_token": "..." }`
   - Response: `{ "message": "Google account linked" }`
   - Behavior:
     - Verifies user is authenticated
     - Verifies Google ID token
     - Stores Google ID for user
     - Updates auth_provider to 'Both'

3. **Unlink Google Account**
   - `DELETE /v1/users/{id}/unlink-google`
   - Requires: Authenticated session, password must exist
   - Response: `{ "message": "Google account unlinked" }`
   - Behavior:
     - Verifies user has password
     - Removes Google ID
     - Updates auth_provider to 'Email'

**Modified Endpoints**:

1. **User Registration** (`POST /v1/users`)
   - Make password optional in request
   - Support OAuth user creation
   - Set auth_provider based on input

2. **User Login** (`POST /v1/auth/login`)
   - Keep existing email/password flow
   - No changes required (separate endpoint for Google)

#### BFF Service Changes

**New Endpoints**:

1. **Google Sign-In**
   - `POST /v1/auth/google`
   - Forwards to User Service
   - Returns session token to frontend

2. **Link Google Account**
   - `POST /v1/users/{id}/link-google`
   - Requires session authentication
   - Forwards to User Service

3. **Unlink Google Account**
   - `DELETE /v1/users/{id}/unlink-google`
   - Requires session authentication
   - Forwards to User Service

#### Frontend Changes

**New Components**:

1. **GoogleSignInButton Component**
   - Renders Google sign-in button
   - Handles OAuth flow initiation
   - Manages OAuth callback

2. **AccountLinking Component**
   - Shows linked authentication methods
   - Allows linking/unlinking Google account
   - Displays in user profile

**Modified Components**:

1. **SignupForm.tsx**
   - Add "Sign up with Google" button
   - Handle Google OAuth callback
   - Show loading state during OAuth

2. **SigninForm.tsx**
   - Add "Sign in with Google" button
   - Handle Google OAuth callback
   - Show loading state during OAuth

3. **UserProfile.tsx**
   - Display linked authentication methods
   - Show link/unlink Google account option
   - Prevent unlinking if no password

### Google OAuth Configuration

#### Required Google Cloud Setup

1. **Create OAuth 2.0 Client ID**
   - Google Cloud Console → APIs & Services → Credentials
   - Create OAuth 2.0 Client ID
   - Application type: Web application
   - Authorized JavaScript origins: `http://localhost:3000` (dev), production domain
   - Authorized redirect URIs: `http://localhost:3000/auth/google/callback` (dev), production callback

2. **Enable Google+ API**
   - Google Cloud Console → APIs & Services → Library
   - Enable Google+ API (for user profile access)

3. **Environment Variables**
   ```
   GOOGLE_CLIENT_ID=<client-id>
   GOOGLE_CLIENT_SECRET=<client-secret>  # For server-side verification
   GOOGLE_REDIRECT_URI=<callback-url>
   ```

#### Frontend OAuth Flow

1. **Load Google OAuth SDK**
   ```html
   <script src="https://accounts.google.com/gsi/client" async defer></script>
   ```

2. **Initialize Google Sign-In**
   ```typescript
   window.google.accounts.id.initialize({
     client_id: GOOGLE_CLIENT_ID,
     callback: handleGoogleSignIn
   });
   ```

3. **Render Sign-In Button**
   ```typescript
   window.google.accounts.id.renderButton(
     document.getElementById('google-signin-button'),
     { theme: 'outline', size: 'large' }
   );
   ```

4. **Handle Credential Response**
   ```typescript
   function handleGoogleSignIn(response: CredentialResponse) {
     // Send credential.id_token to backend
     fetch('/v1/auth/google', {
       method: 'POST',
       body: JSON.stringify({ id_token: response.credential })
     });
   }
   ```

#### Backend Token Verification

**Rust Implementation** (using `google-oauth2` or `jsonwebtoken` with Google public keys):

```rust
use jsonwebtoken::{decode, DecodingKey, Validation, Algorithm};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct GoogleClaims {
    sub: String,           // Google user ID
    email: String,
    email_verified: bool,
    name: Option<String>,
    picture: Option<String>,
    aud: String,            // Client ID (must match)
    exp: usize,            // Expiration
    iss: String,           // Must be "accounts.google.com" or "https://accounts.google.com"
}

async fn verify_google_token(id_token: &str, client_id: &str) -> Result<GoogleClaims, Error> {
    // Fetch Google's public keys (cache these)
    let keys = fetch_google_public_keys().await?;
    
    // Decode token header to get key ID
    let header = decode_header(id_token)?;
    let kid = header.kid.ok_or("Missing kid in token header")?;
    
    // Find matching public key
    let key = keys.find(&kid)?;
    let decoding_key = DecodingKey::from_rsa_components(&key.n, &key.e)?;
    
    // Verify token
    let mut validation = Validation::new(Algorithm::RS256);
    validation.set_audience(&[client_id]);
    validation.iss = Some("accounts.google.com".to_string());
    
    let token_data = decode::<GoogleClaims>(id_token, &decoding_key, &validation)?;
    
    // Additional checks
    if !token_data.claims.email_verified {
        return Err("Email not verified".into());
    }
    
    Ok(token_data.claims)
}
```

### Security Considerations

#### Token Verification

- **Public Key Validation**: Verify token signature using Google's public keys
- **Token Expiration**: Reject expired tokens
- **Audience Validation**: Ensure token is intended for our application
- **Issuer Validation**: Verify token is from Google
- **Email Verification**: Only accept tokens with verified email addresses

#### Account Security

- **Email Conflicts**: Handle cases where Google email matches existing account
- **Account Linking**: Require authentication before linking accounts
- **Unlinking Protection**: Prevent account lockout (require password before unlinking)
- **Session Security**: Use same session mechanism as password-based auth
- **CSRF Protection**: Validate OAuth state parameter

#### Data Privacy

- **Minimal Data Collection**: Only collect necessary information (email, name)
- **Profile Picture**: Optional, user can choose to display
- **Google ID Storage**: Store only Google user ID, not full tokens
- **Token Handling**: Never store Google ID tokens, only verify and discard

#### Error Handling

- **Invalid Tokens**: Return 401 Unauthorized
- **Expired Tokens**: Return 401 with clear error message
- **Email Mismatch**: Return 400 with explanation
- **Account Blocked**: Return 403 Forbidden
- **Network Errors**: Handle Google API unavailability gracefully

## Implementation Plan

### Phase 1: Backend Foundation (Weeks 1-2)

**Week 1: Database & Domain Layer**
- [ ] Create database migration for OAuth fields
- [ ] Update User entity to support nullable password and Google ID
- [ ] Add AuthProvider enum
- [ ] Update User repository to handle OAuth fields
- [ ] Add Google token verification service
- [ ] Write unit tests for OAuth domain logic

**Week 2: API Layer**
- [ ] Implement Google sign-in endpoint in User Service
- [ ] Implement account linking endpoint
- [ ] Implement account unlinking endpoint
- [ ] Update user creation to support OAuth
- [ ] Add BFF endpoints for Google authentication
- [ ] Write integration tests for OAuth endpoints

**Deliverables**:
- Database migration script
- Updated User domain model
- Google token verification service
- OAuth API endpoints
- Integration tests

### Phase 2: Frontend Integration (Weeks 3-4)

**Week 3: OAuth UI Components**
- [ ] Integrate Google OAuth JavaScript SDK
- [ ] Create GoogleSignInButton component
- [ ] Add Google sign-in to SignupForm
- [ ] Add Google sign-in to SigninForm
- [ ] Handle OAuth callback flow
- [ ] Add loading and error states

**Week 4: Account Management**
- [ ] Create AccountLinking component
- [ ] Add account linking to UserProfile
- [ ] Implement link/unlink Google account UI
- [ ] Add authentication method display
- [ ] Handle edge cases (no password, already linked)
- [ ] Write E2E tests for OAuth flow

**Deliverables**:
- Google sign-in buttons on forms
- OAuth callback handling
- Account linking UI
- E2E tests

### Phase 3: Testing & Refinement (Week 5)

**Week 5: Testing & Polish**
- [ ] End-to-end testing of OAuth flows
- [ ] Security audit of token verification
- [ ] Performance testing
- [ ] Error handling improvements
- [ ] Documentation updates
- [ ] User acceptance testing

**Deliverables**:
- Comprehensive test suite
- Security review
- Updated documentation
- Production-ready feature

### Phase 4: Deployment & Monitoring (Week 6)

**Week 6: Production Deployment**
- [ ] Configure Google OAuth in production
- [ ] Deploy database migrations
- [ ] Deploy backend services
- [ ] Deploy frontend updates
- [ ] Monitor OAuth authentication metrics
- [ ] Collect user feedback

**Deliverables**:
- Production deployment
- Monitoring dashboards
- User feedback collection

## Testing Requirements

### Unit Tests

**User Service**:
- Google token verification logic
- User creation with OAuth
- User lookup by Google ID
- Account linking logic
- Account unlinking logic
- Auth provider enum handling

**Domain Layer**:
- User entity with nullable password
- OAuth user creation
- Email conflict resolution

### Integration Tests

**API Endpoints**:
- `POST /v1/auth/google` - New user registration
- `POST /v1/auth/google` - Existing user login
- `POST /v1/users/{id}/link-google` - Account linking
- `DELETE /v1/users/{id}/unlink-google` - Account unlinking
- Error cases (invalid token, expired token, blocked user)

**BFF Integration**:
- OAuth flow through BFF
- Session creation after OAuth
- Error propagation

### E2E Tests

**Frontend Flows**:
- Google sign-up flow
- Google sign-in flow
- Account linking flow
- Account unlinking flow
- Error handling (invalid token, network errors)
- Session persistence after OAuth login

### Security Tests

- Token signature verification
- Token expiration handling
- Audience validation
- CSRF protection
- Account lockout prevention
- Email verification requirement

## Success Criteria

### Functional Success

- ✅ Users can sign up using Google account
- ✅ Users can sign in using Google account
- ✅ Existing users can link Google accounts
- ✅ Users can unlink Google accounts (with password protection)
- ✅ OAuth and password authentication work independently
- ✅ Account data preserved across authentication methods

### Technical Success

- ✅ Google ID tokens verified correctly
- ✅ Database schema supports OAuth users
- ✅ No breaking changes to existing authentication
- ✅ Session management works for OAuth users
- ✅ Error handling covers all edge cases
- ✅ Performance impact < 100ms for token verification

### User Experience Success

- ✅ Google sign-in button visible and accessible
- ✅ OAuth flow completes in < 3 seconds
- ✅ Clear error messages for failures
- ✅ Account linking/unlinking intuitive
- ✅ No user data loss during migration

### Security Success

- ✅ All tokens verified before acceptance
- ✅ No security vulnerabilities introduced
- ✅ Account lockout prevented
- ✅ CSRF protection implemented
- ✅ Privacy requirements met

## Open Questions & Decisions

### Q1: Email Conflict Resolution

**Question**: What happens when a user tries to sign in with Google using an email that already exists with a password-based account?

**Options**:
- **Option A**: Auto-link accounts (require password verification first)
- **Option B**: Show error, require password login first, then link
- **Option C**: Create separate account (not recommended - violates email uniqueness)

**Recommendation**: Option B - Show clear message that account exists, require password login, then offer to link Google account.

### Q2: Profile Picture Storage

**Question**: Should we store Google profile pictures in our database?

**Options**:
- **Option A**: Store URL in database
- **Option B**: Fetch from Google on demand
- **Option C**: Don't store, use Google URL directly

**Recommendation**: Option C - Use Google profile picture URL directly, no storage needed.

### Q3: Name Field

**Question**: Should we add a `name` field to users table for Google users?

**Options**:
- **Option A**: Add `name` field, populate from Google
- **Option B**: Keep email-only, add name later if needed
- **Option C**: Store in separate profile table

**Recommendation**: Option B - Keep minimal for MVP, add name field in future if needed.

### Q4: Multiple OAuth Providers

**Question**: Should we design for future OAuth providers (Facebook, Apple, etc.)?

**Options**:
- **Option A**: Generic OAuth table structure
- **Option B**: Google-specific fields, refactor later
- **Option C**: Hybrid approach

**Recommendation**: Option B - Start Google-specific, refactor to generic structure when adding second provider.

## Dependencies

### External Dependencies

- **Google OAuth 2.0**: Google Cloud Platform OAuth configuration
- **Google OAuth JavaScript SDK**: Frontend OAuth library
- **Google Public Keys**: For token verification (can be cached)

### Internal Dependencies

- **User Service**: Must support OAuth user creation and authentication
- **BFF Service**: Must route OAuth requests
- **Frontend**: Must integrate Google OAuth SDK
- **Database**: Must support schema changes

### Prerequisites

- Google Cloud Platform account
- OAuth 2.0 client credentials configured
- Database migration capability
- Frontend deployment pipeline

## Risks & Mitigations

### Risk 1: Google API Unavailability

**Risk**: Google OAuth service unavailable, users cannot authenticate.

**Mitigation**:
- Fallback to email/password authentication
- Cache Google public keys with TTL
- Monitor Google API status
- Graceful error messages

### Risk 2: Token Verification Performance

**Risk**: Token verification adds latency to authentication.

**Mitigation**:
- Cache Google public keys (24-hour TTL)
- Use efficient JWT verification library
- Monitor performance metrics
- Consider async verification if needed

### Risk 3: Account Linking Conflicts

**Risk**: Users link wrong Google account or create duplicate accounts.

**Mitigation**:
- Clear UI showing which account is being linked
- Require password verification before linking
- Prevent linking if Google ID already in use
- Provide account unlinking capability

### Risk 4: Migration Complexity

**Risk**: Database migration affects existing users.

**Mitigation**:
- Test migration on staging environment
- Make password nullable (not dropping column)
- Preserve all existing user data
- Rollback plan ready

### Risk 5: Security Vulnerabilities

**Risk**: OAuth implementation introduces security issues.

**Mitigation**:
- Security code review
- Use well-tested OAuth libraries
- Follow OAuth 2.0 best practices
- Regular security audits

## Future Enhancements

### Phase 2 Features (Post-MVP)

- **Additional OAuth Providers**: Facebook, Apple Sign-In, Microsoft
- **Profile Picture Sync**: Sync Google profile picture
- **Name Field**: Store and display user names
- **OAuth Token Refresh**: Handle token refresh for long sessions
- **Account Merging**: Merge accounts with same email from different providers

### Advanced Features

- **Social Features**: Share budgets with Google contacts
- **Calendar Integration**: Sync with Google Calendar for budget reminders
- **Email Integration**: Use Gmail for transaction import
- **Multi-Factor Authentication**: Combine OAuth with MFA

## Appendix

### Google OAuth Scopes

**Required Scopes**:
- `openid` - OpenID Connect
- `email` - User email address
- `profile` - Basic profile information (name, picture)

**Optional Scopes** (Future):
- `https://www.googleapis.com/auth/userinfo.profile` - Extended profile

### Token Verification Flow

1. Receive `id_token` from frontend
2. Decode token header to get `kid` (key ID)
3. Fetch Google's public keys (or use cached)
4. Find public key matching `kid`
5. Verify token signature
6. Validate `aud` (audience) matches client ID
7. Validate `iss` (issuer) is Google
8. Check `exp` (expiration) is in future
9. Verify `email_verified` is true
10. Extract user information

### Database Migration Script

```sql
-- Migration: Add OAuth support to users table
-- Date: 2025-01-XX
-- Description: Adds Google OAuth support with nullable password

BEGIN;

-- Add Google ID column
ALTER TABLE users 
ADD COLUMN google_id VARCHAR(255) UNIQUE;

-- Add auth provider tracking
ALTER TABLE users 
ADD COLUMN auth_provider VARCHAR(50) DEFAULT 'email';

-- Make password nullable (OAuth users don't need passwords)
ALTER TABLE users 
ALTER COLUMN password DROP NOT NULL;

-- Add index for Google ID lookups
CREATE INDEX idx_users_google_id 
ON users(google_id) 
WHERE google_id IS NOT NULL;

-- Update existing users
UPDATE users 
SET auth_provider = 'email' 
WHERE auth_provider IS NULL;

-- Add constraint to ensure at least one auth method
ALTER TABLE users 
ADD CONSTRAINT check_auth_method 
CHECK (
    (password IS NOT NULL) OR 
    (google_id IS NOT NULL)
);

COMMIT;
```

### API Request/Response Examples

**Google Sign-In Request**:
```json
POST /v1/auth/google
{
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjEyMzQ1NiJ9..."
}
```

**Google Sign-In Response**:
```json
{
  "session_id": "550e8400-e29b-41d4-a716-446655440000",
  "user_id": "660e8400-e29b-41d4-a716-446655440000",
  "is_new_user": true
}
```

**Link Google Account Request**:
```json
POST /v1/users/{id}/link-google
Authorization: Bearer {session_token}
{
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjEyMzQ1NiJ9..."
}
```

**Link Google Account Response**:
```json
{
  "message": "Google account linked successfully",
  "auth_provider": "Both"
}
```

