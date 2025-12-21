# PRD Analysis: Money Planner Backoffice Application

**Document Information**
- **Analysis Date**: November 9, 2025
- **PRD Version Analyzed**: 1.0 (Draft)
- **Analyzed By**: Claude Code
- **Status**: Critical Issues Identified

---

## Executive Summary

This document provides a comprehensive analysis of the Money Planner Backoffice Application PRD against the actual codebase implementation. The analysis reveals **several critical misconceptions, missing prerequisites, and architectural issues** that must be addressed before implementation can begin.

**Key Findings:**
- ❌ Prerequisites timeline severely underestimated (4.4 weeks → 8-10 weeks)
- ❌ Critical missing features in User Service (event publishing, block/unblock endpoints)
- ❌ Budget Service incomplete event processing
- ❌ Eventual consistency issues not addressed
- ❌ WebSocket complexity underestimated
- ⚠️ Total project timeline should be **24-28 weeks** instead of 20 weeks

---

## Table of Contents

1. [Critical Issues](#critical-issues)
2. [Significant Issues](#significant-issues)
3. [Design Concerns](#design-concerns)
4. [Architecture Recommendations](#architecture-recommendations)
5. [Summary of Required Changes](#summary-of-required-prd-changes)
6. [Recommended Next Steps](#recommended-next-steps)

---

## 🔴 CRITICAL ISSUES

### 1. User Service Event Publishing - Fundamental Architecture Gap

**PRD Location**: Lines 816-886

**PRD Claim**:
- States User Service needs to publish Kafka events as "Critical for Standalone Service Approach"
- Assumes this is prerequisite work (5 days)
- Lists events: `user.created`, `user.updated`, `user.blocked`, `user.unblocked`

**Actual Reality**:
- ❌ User Service has **ZERO Kafka infrastructure**
- ❌ No `rdkafka` dependency in `user-service/Cargo.toml`
- ❌ No Kafka producer implementation
- ❌ No event schema definitions
- ❌ No Kafka configuration in run.sh or config files

**Evidence**:
```bash
# Checked: user-service/Cargo.toml
# Result: No rdkafka dependency found
# Checked: user-service/src/ for Kafka code
# Result: No Kafka producer infrastructure exists
```

**Impact**:
- The entire backoffice read model architecture depends on user events
- Without user events, the backoffice service cannot build `user_summaries` table
- Real-time dashboard metrics for user statistics impossible without events
- **Timeline underestimation**: This is not 5 days - it's 10-15 days including:
  - Adding Kafka dependencies (1 day)
  - Creating producer infrastructure (2 days)
  - Defining event schemas (1 day)
  - Implementing publishing logic in all user operations (3 days)
  - Error handling and retries (2 days)
  - Testing and idempotency (2 days)
  - Integration testing (1-2 days)

**Recommendation**:
1. **Reconsider architecture**: Should backoffice service directly query User Service API instead of building read model for Phase 1?
§3. **Alternative**: Start with BFF approach (direct API calls) and add event-driven read model in Phase 2

**Priority**: 🔴 Critical - Blocks entire Phase 1

---

### 2. Budget Service Event Publishing - Critical Misconception

**PRD Location**: Lines 928-1000

**PRD Claim**:
- Budget Service "does not publish events"
- Needs 5 days to implement budget event publishing
- Events needed: `budget.created`, `budget.updated`, `budget.deleted`

**Actual Reality**:
- ✅ **Correct** - Budget Service has no event publishing
- ⚠️ **However**: Budget Service **only processes `TransactionCreated` events**
- ❌ Budget Service **ignores `TransactionUpdated`** events
- ❌ Budget Service **ignores `TransactionDeleted`** events

**Evidence**:
```rust
// budget-service/src/domain/transaction_events.rs
pub enum TransactionEventType {
    #[serde(rename = "transaction.created")]
    Created,  // ONLY THIS TYPE IS HANDLED
}
```

**Missing from PRD**:
- Budget Service needs to handle ALL transaction event types before it can provide accurate budget data
- Current implementation will have stale budget calculations if transactions are updated or deleted

**Impact**:
- Budget analytics in backoffice will be **inaccurate**
- Budget spent amounts won't reflect transaction updates/deletions
- Example: User updates transaction from $100 to $50, but budget still shows $100 spent
- **Additional prerequisite work needed**: 3-5 days to handle all transaction event types

**Recommendation**:
1. Add to prerequisites: "Fix Budget Service to process TransactionUpdated and TransactionDeleted events"
2. Estimated time: 3-5 days
3. Must be completed before Phase 1 if budget data is used in backoffice
4. Consider this a critical data integrity issue

**Priority**: 🔴 Critical - Affects data accuracy

---

### 3. Block/Unblock User Endpoints - Completely Missing

**PRD Location**: Lines 794-806

**PRD Claim**:
- Lists as "Critical for Phase 1"
- Estimates 3 days
- Required endpoints: `POST /v1/users/{userId}/block` and `POST /v1/users/{userId}/unblock`
- Alternative: `PUT /v1/users/{userId}` with state field

**Actual Reality**:
- ✅ `UserState::Blocked` enum exists in domain model (`user-service/src/domain/user/entities.rs`)
- ✅ State is checked during login
- ❌ **NO API endpoints** to change user state
- ❌ `PUT /v1/users/{userId}` endpoint exists in routes but is marked as **TODO** (not implemented)

**Evidence**:
```rust
// user-service/src/api/user/handlers.rs:47
// TODO: Implement update_user handler

// user-service/src/api/user/routes.rs
.route("/users/:id", put(update_user))  // Handler is empty/todo
```

**Impact**:
- **Phase 1 cannot be completed** without these endpoints
- Core admin functionality (block/unblock users) completely blocked
- User search shows users but admin can't take action on them

**Recommendation**:
1. Implement `PUT /v1/users/{userId}` endpoint with full user update capability
2. Include state field in request body
3. Add audit logging for state changes (who blocked, when, why)
4. Increase estimate to **5 days** for:
   - Implement update handler (2 days)
   - Add state validation logic (1 day)
   - Add audit logging (1 day)
   - Testing (1 day)

**Priority**: 🔴 Critical - Blocks Phase 1 MVP

---

### 4. User Search/Filtering - Not Implemented

**PRD Location**: Lines 802-809

**PRD Claim**:
- Enhance `GET /v1/users` with query parameters
- Required: `?email=...&search=...&page=...&limit=...`
- Estimates 3 days

**Actual Reality**:
```rust
// user-service/src/repositories/user.rs
async fn list_users(&self) -> Result<Vec<User>, sqlx::Error> {
    sqlx::query_as!(
        User,
        r#"SELECT id, email, password, role as "role: UserRole",
           state as "state: UserState", created_at
           FROM users
           ORDER BY created_at DESC"#
    )
    .fetch_all(&*self.pool)
    .await
}
```

**Missing**:
- ❌ No query parameters
- ❌ No filtering by email, state, or role
- ❌ No pagination
- ❌ No search functionality
- ❌ Returns ALL users (scalability issue)

**Impact**:
- Admin staff cannot efficiently find users
- Will return all users (performance issue with 10,000+ users)
- No way to search by email or name
- UI will be unusable with large user bases

**Recommendation**:
1. Implement full-text search and pagination:
```rust
async fn list_users(
    &self,
    email: Option<String>,
    search: Option<String>,
    state: Option<UserState>,
    role: Option<UserRole>,
    page: i64,
    limit: i64,
) -> Result<(Vec<User>, i64), sqlx::Error>
```
2. Add indexes on email, state, role columns
3. Consider using PostgreSQL full-text search capabilities
4. Return total count for pagination
5. Increase estimate to **4-5 days** for complete implementation

**Priority**: 🔴 Critical - Blocks Phase 1 user search functionality

---

### 5. Session Storage - Architecture Misunderstanding

**PRD Location**: Lines 716-719

**PRD Claim**:
- "Sessions are managed by User Service, not stored in backoffice database"
- "Session validation is done via User Service /v1/auth/session endpoint"

**Actual Reality**:
- ✅ Correct - sessions stored in User Service PostgreSQL database
- ⚠️ **NOT Redis** - Uses database table
- ⚠️ Session expiry is **hardcoded to 24 hours** (not configurable)

**Evidence**:
```sql
-- user-service/migrations/18082024_200000_base_entities.sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    state VARCHAR(25) NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

```rust
// user-service/src/domain/user/entities.rs
impl Session {
    pub fn new(user_id: UserId) -> Self {
        let now = Utc::now();
        Self {
            id: SessionId::new(),
            user_id,
            created_at: now,
            expires_at: now + chrono::Duration::hours(24), // HARDCODED
            state: SessionState::Active,
        }
    }
}
```

**Missing from PRD**:
- ❌ No mention of session cleanup mechanism (expired sessions accumulate)
- ❌ No mention of active session tracking for statistics
- ⚠️ Database-backed sessions may have performance issues at scale
- ❌ No background job to clean up expired sessions

**Impact**:
- For "Users with active sessions" metric (Line 116), need database query to count non-expired sessions
- Expired sessions will accumulate in database without cleanup
- Performance degradation over time

**Recommendation**:
1. Document session storage mechanism accurately in PRD
2. Implement session cleanup background job:
```rust
// Periodic cleanup of expired sessions
DELETE FROM sessions WHERE expires_at < NOW()
```
3. For active session count, add database query:
```sql
SELECT COUNT(*) FROM sessions WHERE expires_at > NOW()
```
4. Consider adding session cleanup to Phase 1 (2 days)

**Priority**: ⚠️ Significant - Affects dashboard metrics and performance

---

### 6. Kafka Topic Naming Inconsistency

**PRD Location**: Lines 819, 899, 931, 1044-1048

**PRD Claim**:
- User events: `users.events`
- Transaction events: `transactions.events`
- Budget events: `budgets.events`

**Actual Reality**:
- ❌ Transaction Service uses: **`int.transaction-update.event`** (not `transactions.events`)
- ❌ Topic naming pattern: `int.<domain>-<action>.event` (internal prefix)

**Evidence**:
```bash
# transaction-service/run.sh
export KAFKA_TRANSACTION_TOPIC=int.transaction-update.event

# transaction-service/src/infrastructure/kafka_producer.rs
let topic = env::var("KAFKA_TRANSACTION_TOPIC")
    .unwrap_or_else(|_| "int.transaction-update.event".to_string());
```

**Impact**:
- PRD uses incorrect topic names throughout
- Backoffice service implementation will fail if using PRD topic names
- Code examples in PRD show wrong topic names
- Confusion during implementation

**Recommendation**:
- **Option 1**: Update PRD to use actual topic names (`int.transaction-update.event`, `int.user-update.event`, `int.budget-update.event`)
- **Option 2**: Standardize all services to use consistent naming (`users.events`, `transactions.events`, `budgets.events`)
- **Choose one approach** and update PRD accordingly
- Update all code examples in PRD

**Priority**: ⚠️ Significant - Will cause implementation errors

---

## ⚠️ SIGNIFICANT ISSUES

### 7. Staff Role - Undefined Implementation

**PRD Location**: Lines 24-27, 887-893

**PRD Claim**:
- Lists "Staff" as existing role with "currently undefined functionality"
- Includes Staff in MVP authentication (Phase 1)
- Differentiates between Staff and Admin permissions

**Actual Reality**:
```rust
// user-service/src/domain/user/entities.rs
pub enum UserRole {
    Customer,
    Staff,    // Exists in code but no distinct behavior
    Admin,
}
```

**Missing**:
- ❌ No permissions defined for Staff role
- ❌ No authorization logic differentiating Staff from Admin
- ❌ No documentation of Staff capabilities
- ❌ Currently, Staff is treated same as Customer in authorization checks

**Impact**:
- PRD assumes Staff role is ready for MVP
- Unclear what permissions Staff should have vs Admin
- Authorization logic needs to be implemented
- Example: Can Staff block users? View all users? Access all features?

**Recommendation**:
1. **Define Staff role permissions matrix BEFORE Phase 1**:
   - What can Staff do that Customer cannot?
   - What can Admin do that Staff cannot?
   - Create permission matrix table
2. Update User Service authorization middleware to enforce Staff vs Admin permissions
3. Add 5 days to prerequisites for Staff role implementation
4. **Alternative**: Remove Staff role from Phase 1 MVP, only support Admin
   - Simplifies MVP
   - Can add Staff role in Phase 2 after requirements are clear

**Priority**: ⚠️ Significant - Affects MVP scope and authorization

---

### 8. Authentication Flow - Complexity Not Acknowledged

**PRD Location**: Lines 514-521

**PRD Description**:
1. Admin logs in via User Service
2. User Service returns `sessionId` and `userId`
3. Frontend stores in localStorage
4. Backoffice Service validates session
5. Backoffice Service routes to backend services

**Missing Complexity**:
- ❌ Backoffice Service needs to **call User Service `/v1/auth/session` on every request**
- ❌ Performance impact of session validation on every admin request
- ❌ No caching strategy mentioned
- ❌ What happens if User Service is down?
- ❌ No circuit breaker pattern discussed

**Reality from BFF Implementation**:
```rust
// money-planner-bff/src/api/auth_utils.rs
impl axum::extract::FromRequestParts<Arc<AppState>> for SessionExtractor {
    async fn from_request_parts(...) -> Result<Self, Self::Rejection> {
        // Calls User Service on EVERY request
        let response = http_client
            .get(format!("{}/v1/auth/session", user_service_url))
            .header("Authorization", format!("Bearer {}", token))
            .send()
            .await?;

        // If User Service down, ALL admin requests fail
    }
}
```

**Impact**:
- Every admin request requires 2 HTTP calls: Frontend → Backoffice → User Service
- Latency increases by User Service response time
- If User Service is down, entire backoffice is unusable
- Load on User Service increases proportionally to admin activity

**Recommendation**:
1. Add session caching strategy:
   - Cache session validation results for 5 minutes
   - Use Redis or in-memory cache with TTL
   - Invalidate cache on user block/unblock
2. Implement circuit breaker for User Service calls:
   - Fall back to cached data if User Service unavailable
   - Return degraded service instead of complete failure
3. Add health check dependencies
4. Increase Phase 1 estimates by **2-3 days** for robust session handling
5. Document in PRD: "Session Validation and Caching Strategy"

**Priority**: ⚠️ Significant - Affects performance and reliability

---

### 9. WebSocket Server - Significant Technical Complexity

**PRD Location**: Lines 528-632

**PRD Claim**:
- WebSocket server for real-time dashboard updates
- "Backoffice Service WebSocket Server Responsibilities"
- Implementation appears simple in PRD

**Missing Technical Details**:
- ❌ Axum WebSocket implementation complexity not acknowledged
- ❌ Connection management (potentially hundreds of concurrent admin clients)
- ❌ WebSocket authentication (session validation on connection)
- ❌ Broadcasting strategy (all connected clients)
- ❌ Connection state management
- ❌ Heartbeat/ping-pong mechanism
- ❌ Graceful shutdown handling
- ❌ Reconnection logic on client side

**Reality Check**:
- WebSocket server is **NOT trivial** in Rust
- Requires `axum::extract::ws` or `tokio-tungstenite`
- Broadcasting to multiple clients requires channels/pubsub pattern
- Connection management needs careful state tracking
- Example complexity:
```rust
// Simplified WebSocket handler pseudocode
async fn websocket_handler(
    ws: WebSocketUpgrade,
    session: SessionExtractor,
) -> Response {
    ws.on_upgrade(|socket| handle_socket(socket, session))
}

async fn handle_socket(socket: WebSocket, session: Session) {
    // 1. Validate session on connection
    // 2. Add to active connections map
    // 3. Start broadcast listener
    // 4. Handle heartbeat/ping-pong
    // 5. Handle client messages
    // 6. Handle disconnection
    // 7. Remove from active connections
}
```

**Recommendation**:
1. Add 5-7 days to Phase 1 for WebSocket implementation
2. **Consider using Server-Sent Events (SSE) instead** for Phase 1:
   - SSE is much simpler to implement (unidirectional)
   - Sufficient for dashboard metrics updates
   - Easier error handling and reconnection
   - Can upgrade to WebSocket in Phase 2 if bidirectional needed
3. Add WebSocket as Phase 2 enhancement if bidirectional communication needed
4. Document WebSocket complexity in PRD
5. Add section on connection management and broadcasting

**Priority**: ⚠️ Significant - Affects Phase 1 timeline

---

### 10. Database Schema - Missing Critical Tables and Indexes

**PRD Location**: Lines 635-753

**PRD Shows**:
- `user_summaries` table
- `transaction_summaries` table
- `budget_summaries` table
- `processed_events` table for idempotency
- `audit_logs` table
- `system_alerts` table

**Missing from PRD**:
- ❌ No **indexes** defined on critical search columns
- ❌ No `dashboard_metrics_cache` table for performance optimization
- ❌ No `websocket_connections` table for connection tracking
- ❌ No partitioning strategy for large tables (transactions, events)
- ❌ No materialized views for analytics queries

**Schema Improvements Needed**:
```sql
-- Add indexes for fast queries
CREATE INDEX idx_user_summaries_email ON user_summaries(email);
CREATE INDEX idx_user_summaries_state ON user_summaries(state);
CREATE INDEX idx_user_summaries_role ON user_summaries(role);
CREATE INDEX idx_user_summaries_created_at ON user_summaries(created_at);

CREATE INDEX idx_transaction_summaries_user_id ON transaction_summaries(user_id);
CREATE INDEX idx_transaction_summaries_date ON transaction_summaries(date);
CREATE INDEX idx_transaction_summaries_category_id ON transaction_summaries(category_id);

-- Materialized view for dashboard metrics
CREATE MATERIALIZED VIEW dashboard_metrics AS
SELECT
    COUNT(*) as total_users,
    COUNT(*) FILTER (WHERE state = 'Active') as active_users,
    COUNT(*) FILTER (WHERE created_at > NOW() - INTERVAL '7 days') as recent_users_7d,
    COUNT(*) FILTER (WHERE created_at > NOW() - INTERVAL '30 days') as recent_users_30d
FROM user_summaries;

-- Partition audit_logs by month for performance
CREATE TABLE audit_logs_2025_01 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

**Recommendation**:
1. Add comprehensive index strategy to PRD
2. Add materialized views for dashboard metrics (refresh every 5 minutes)
3. Plan for table partitioning (monthly) for audit logs and events
4. Add 2-3 days to Phase 1 for database optimization
5. Document index maintenance strategy

**Priority**: ⚠️ Significant - Affects query performance

---

### 11. Read Model Eventual Consistency - Not Addressed

**PRD Location**: Lines 284-286

**PRD Architecture**:
- Read operations from denormalized read model
- Write operations direct to services
- Event flow updates read model

**Critical Missing Issue**:
- ❌ **Eventual consistency** between write and read
- ❌ Admin blocks user → User Service updates → Kafka event → Backoffice processes event → Read model updated
- ❌ **Delay between admin action and UI update** (could be seconds)
- ❌ No strategy for handling read-after-write consistency

**Example Scenario**:
```
1. Admin clicks "Block User" button
2. Backoffice Service → User Service (POST /v1/users/123/block)
3. User Service blocks user in database
4. User Service publishes user.blocked event to Kafka
5. Admin redirected to user list
6. Backoffice Service queries user_summaries table (read model)
7. **User still shows as Active** ← Kafka event not processed yet!
8. Admin confused - "Did it work? Should I click again?"
9. 2 seconds later, Kafka event processed, user shows as Blocked
10. Admin sees update but already clicked "Block" again (duplicate action)
```

**Impact**:
- Poor user experience (confusing UX)
- Potential duplicate actions
- Admin trust in system reduced
- Support tickets: "Backoffice is buggy, changes don't apply"

**Recommendation**:
1. **Implement write-through cache** for recent admin actions:
```rust
// On write operation
async fn block_user(user_id: Uuid) -> Result<()> {
    // 1. Call User Service
    user_service.block_user(user_id).await?;

    // 2. Update local cache immediately (optimistic)
    cache.set(user_id, UserState::Blocked, 60).await; // TTL 60s

    // 3. Return success
    Ok(())
}

// On read operation
async fn get_user(user_id: Uuid) -> Result<User> {
    // 1. Check cache first
    if let Some(cached_state) = cache.get(user_id).await {
        return Ok(cached_state);
    }

    // 2. Fall back to read model
    read_model.get_user(user_id).await
}
```

2. **Or implement optimistic UI updates** with confirmation:
   - Update UI immediately (optimistic)
   - Show "Processing..." indicator
   - Confirm via WebSocket when event processed
   - Rollback UI if error

3. **Or implement synchronous read-after-write**:
   - On write operations, return fresh data from source service
   - Don't rely on read model for recently updated data

4. Add to PRD: "Handling Eventual Consistency in Admin UX" section
5. Add 3-4 days to Phase 1 for consistency handling

**Priority**: ⚠️ Significant - Affects UX quality

---

### 12. Prerequisites Timeline - Severely Underestimated

**PRD Location**: Lines 1002-1014

**PRD Claims**:
- Total prerequisite work: **22 days (~4.4 weeks)**
- User Service: 16 days
- Transaction Service: 1 day
- Budget Service: 5 days

**Reality Check**:

#### User Service Work (PRD: 16 days):
1. Block/unblock endpoints: **3 days** ✅ (reasonable)
2. User search/filtering: **3 days** ✅ (reasonable if simple)
3. **Kafka event publishing: 5 days** ❌ **UNDERESTIMATED**
   - Add Kafka dependencies: 1 day
   - Implement producer infrastructure: 2 days
   - Define event schemas: 1 day
   - Implement publishing in all endpoints: 3 days
   - Error handling and retries: 2 days
   - Testing: 2 days
   - Integration testing: 1 day
   - **Realistic: 10-12 days**
4. Staff role implementation: **3 days** ⚠️ (depends on complexity)
5. User update endpoint: Not estimated but needed

**Revised User Service Total: 22-24 days (4.4-4.8 weeks)**

#### Budget Service Work (PRD: 5 days):
1. Kafka event publishing: **5 days** ✅ (reasonable)
2. **MISSING**: Fix transaction event processing (handle Updated/Deleted): **3-5 days**

**Revised Budget Service Total: 8-10 days (1.6-2 weeks)**

#### New Prerequisites:
1. User Service session event publishing: **+3 days**
2. Session cleanup mechanism: **+2 days**

#### Revised Total Prerequisites:
- User Service: **22-24 days**
- Transaction Service: **1 day**
- Budget Service: **8-10 days**
- Session management: **3 days**
- **Total: 34-38 days (6.8-7.6 weeks)**
- **With 20% buffer: 41-46 days (8-9 weeks)**

**Recommendation**:
1. Revise prerequisites section with realistic estimates
2. Add buffer time (20-30%) for unexpected issues
3. **Total prerequisites: 8-10 weeks** realistically
4. Consider phased approach:
   - **Phase 0**: Complete all prerequisites (8-10 weeks)
   - **Phase 1**: MVP implementation (6-7 weeks)
   - **Phase 2-4**: As planned

**Priority**: 🔴 Critical - Affects project timeline and planning

---

## 💡 DESIGN CONCERNS

### 13. Direct Service Calls vs Read Model - Inconsistency

**PRD Location**: Lines 393-408

**PRD Shows Inconsistency**:

Read operations use **mixed data sources**:
```
GET /v1/admin/users                      # From read model (Line 402)
GET /v1/admin/users/{userId}             # From read model (Line 403)
GET /v1/admin/users/{userId}/transactions # Direct API call to Transaction Service (Line 404)
GET /v1/admin/transactions/{transactionId} # Direct API call to Transaction Service (Line 407)

GET /v1/admin/transactions/              # From read model (Line 416)
```

**Problem**:
- Inconsistent data source (read model vs direct API)
- `GET /v1/admin/users/{userId}/transactions` should use read model (faster)
- Single transaction details could use read model too
- **Why bypass read model for some reads but not others?**

**Impact**:
- Performance inconsistency (some queries fast, others slow)
- Complexity in frontend (different response times)
- Potential data inconsistency between read model and direct API

**Recommendation**:
1. **Consistency**: ALL reads from read model
2. Only call source services on **write operations**
3. Update API design section:
```
# All reads from read model
GET /v1/admin/users                      # From user_summaries
GET /v1/admin/users/{userId}             # From user_summaries
GET /v1/admin/users/{userId}/transactions # From transaction_summaries (filtered by user_id)
GET /v1/admin/transactions/              # From transaction_summaries
GET /v1/admin/transactions/{id}          # From transaction_summaries

# All writes to source services
POST /v1/admin/users/{userId}/block      # → User Service
POST /v1/admin/users/{userId}/unblock    # → User Service
```

**Priority**: 💡 Design - Affects architecture consistency

---

### 14. Active Sessions Tracking - Implementation Unclear

**PRD Location**: Lines 115-117

**PRD Requirement**:
- Dashboard shows "Users with active sessions"
- Real-time metric

**Implementation Questions**:

1. **How does Backoffice Service track active sessions?**
   - Option A: Query User Service database directly (❌ bad architecture - violates service boundaries)
   - Option B: User Service publishes session events (⚠️ not in PRD)
   - Option C: Backoffice Service polls `/v1/auth/session` for all sessions (❌ inefficient)
   - Option D: Backoffice maintains session count from user events (⚠️ requires session events)

2. **How to get real-time updates?**
   - WebSocket updates require knowing when sessions change
   - Requires User Service to publish `session.created`, `session.expired`, `session.deleted` events

**Missing from PRD**:
- ❌ User Service session event publishing not mentioned
- ❌ Session events schema not defined
- ❌ How backoffice tracks active sessions count

**Recommendation**:
1. Add User Service session events to prerequisites:
   - `session.created` - when user logs in
   - `session.expired` - when session TTL expires
   - `session.deleted` - when user logs out
2. Add session event schema:
```json
{
  "event_id": "uuid",
  "event_type": "session.created",
  "timestamp": "2025-01-15T10:30:00Z",
  "version": "1.0",
  "data": {
    "session_id": "uuid",
    "user_id": "uuid",
    "created_at": "2025-01-15T10:30:00Z",
    "expires_at": "2025-01-16T10:30:00Z"
  },
  "metadata": {
    "source_service": "user-service",
    "correlation_id": "uuid"
  }
}
```
3. Update read model to track active sessions count
4. Add 2-3 days to User Service prerequisites for session events

**Priority**: ⚠️ Significant - Blocks dashboard metrics

---

### 15. Transaction Analytics - Double Storage Issue

**PRD Location**: Lines 658-675

**PRD Design**:
- Backoffice maintains `transaction_summaries` table
- Built from Kafka events from Transaction Service

**Actual Reality**:
Budget Service **already maintains transaction cache** from Kafka events:

```sql
-- budget-service/migrations/18082024_initial_schema.sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    description TEXT NOT NULL,
    amount DECIMAL NOT NULL,
    date TIMESTAMPTZ NOT NULL,
    transaction_category_id UUID NOT NULL,
    transaction_type VARCHAR(25) NOT NULL
);
```

**Problem**:
- **Two services maintaining identical transaction copies**
- Transaction Service (source of truth)
- Budget Service (for budget calculations)
- **Backoffice Service (for analytics)** ← Third copy!
- Increased storage costs
- Increased Kafka processing load (two consumers for same data)
- Data consistency issues if one service's processing fails

**Impact**:
- Storage: 3x transaction data
- Kafka load: 2x consumers for transaction events
- Operational overhead: Monitor 2 consumers, handle 2 failures

**Recommendation**:
1. **Option 1**: Backoffice Service queries Budget Service for transaction summaries
   - Reuse existing transaction cache
   - Add analytics endpoints to Budget Service
   - Backoffice becomes thin orchestration layer
2. **Option 2**: Create shared read model service (Transaction Analytics Service)
   - Single service maintains transaction cache
   - Both Budget and Backoffice query it
   - Centralized transaction analytics
3. **Option 3**: Accept duplication but acknowledge in PRD
   - Document that 3 services maintain transaction data
   - Justify duplication (performance, independence)
   - Add monitoring for data consistency
4. Update architecture diagram to show this duplication clearly

**Priority**: 💡 Design - Affects system efficiency

---

### 16. Idempotency - Incomplete Implementation

**PRD Location**: Lines 303, 710-712

**PRD Claims**:
- Event processing with idempotency checks using `event_id`
- `processed_events` table to track processed events

**PRD Schema**:
```sql
processed_events (
    event_id UUID PRIMARY KEY,
    event_type VARCHAR(255),
    topic VARCHAR(255),
    partition INTEGER,
    offset BIGINT,
    processed_at TIMESTAMP
)
```

**Missing Critical Details**:
- ❌ How to handle duplicate events?
- ❌ What happens if event processing partially fails?
- ❌ Retry strategy for failed events?
- ❌ Dead letter queue for poison messages?
- ❌ Event replay mechanism if read model corrupted?

**Reality from Transaction Service**:
```rust
// transaction-service/src/domain/transaction/models/events.rs
pub struct TransactionEvent {
    pub event_id: Uuid,  // Good - has event_id for idempotency
    // But no retry count, no processing status, no error tracking
}
```

**Enhanced Schema Needed**:
```sql
processed_events (
    event_id UUID PRIMARY KEY,
    event_type VARCHAR(255),
    topic VARCHAR(255),
    partition INTEGER,
    offset BIGINT,
    processing_status VARCHAR(50) NOT NULL, -- PROCESSING, COMPLETED, FAILED
    retry_count INTEGER DEFAULT 0,
    last_error TEXT,
    processed_at TIMESTAMP,
    failed_at TIMESTAMP,
    -- For idempotency checks
    CONSTRAINT check_status CHECK (processing_status IN ('PROCESSING', 'COMPLETED', 'FAILED'))
);

-- Dead letter queue for poison messages
CREATE TABLE event_dead_letter_queue (
    id UUID PRIMARY KEY,
    event_id UUID,
    event_type VARCHAR(255),
    event_payload JSONB,
    error_message TEXT,
    retry_count INTEGER,
    created_at TIMESTAMP,
    last_retry_at TIMESTAMP
);
```

**Idempotency Logic**:
```rust
async fn process_event(event: Event) -> Result<()> {
    // 1. Check if already processed
    if let Some(processed) = repo.get_processed_event(&event.event_id).await? {
        match processed.status {
            ProcessingStatus::Completed => return Ok(()), // Idempotent - already done
            ProcessingStatus::Failed => {
                // Retry with exponential backoff
                if processed.retry_count < MAX_RETRIES {
                    // Mark as PROCESSING again
                } else {
                    // Move to dead letter queue
                    return Err(Error::TooManyRetries);
                }
            }
            ProcessingStatus::Processing => {
                // Another instance is processing - skip
                return Ok(());
            }
        }
    }

    // 2. Mark as PROCESSING
    repo.insert_processing_event(&event.event_id).await?;

    // 3. Process event
    match process_event_logic(event).await {
        Ok(_) => {
            // 4. Mark as COMPLETED
            repo.mark_completed(&event.event_id).await?;
        }
        Err(e) => {
            // 5. Mark as FAILED with error
            repo.mark_failed(&event.event_id, &e.to_string()).await?;
        }
    }

    Ok(())
}
```

**Recommendation**:
1. Expand `processed_events` table schema with status tracking
2. Add dead letter queue table
3. Implement retry logic with exponential backoff
4. Add event replay mechanism for disaster recovery
5. Add 3-4 days to Phase 1 for robust event processing
6. Document idempotency strategy in PRD

**Priority**: ⚠️ Significant - Affects data integrity

---

### 17. OpenAPI Documentation - Missing Standards

**PRD Location**: Lines 299, 1451

**PRD Mentions**:
- "API Documentation: OpenAPI/Swagger with utoipa"
- "Appendix A: API Reference"

**Missing from PRD**:
- ❌ No examples of OpenAPI annotations
- ❌ No schema definitions for request/response bodies
- ❌ No error response standardization
- ❌ No API versioning strategy

**Current Services Reality**:
Transaction Service has good OpenAPI docs:
```rust
#[utoipa::path(
    post,
    path = "/v1/transactions",
    request_body = CreateTransactionRequest,
    responses(
        (status = 201, description = "Transaction created", body = Transaction),
        (status = 400, description = "Invalid request"),
        (status = 401, description = "Unauthorized")
    ),
    tag = "transactions"
)]
async fn create_transaction(...)
```

**Recommendation for Backoffice Service**:
1. Add section on API documentation standards:
```markdown
### OpenAPI Documentation Standards

All endpoints must include:
- Complete utoipa annotations
- Request/response schema definitions
- All possible status codes
- Example requests/responses
- Authentication requirements
- Error response format
```

2. Define standard error response format:
```rust
#[derive(Serialize, ToSchema)]
pub struct ErrorResponse {
    pub error: String,
    pub message: String,
    pub details: Option<HashMap<String, String>>,
    pub timestamp: DateTime<Utc>,
}
```

3. Add OpenAPI documentation time to each phase (1-2 days per phase)
4. Include Swagger UI setup in deployment

**Priority**: 💡 Design - Improves developer experience

---

### 18. Security - Missing Threat Model

**PRD Location**: Lines 755-786

**PRD Security Section**:
- Lists security features (authentication, authorization, encryption)
- No threat modeling or attack scenarios

**Missing Critical Security Concerns**:

1. **Privilege Escalation**:
   - ❓ Can Staff user promote themselves to Admin?
   - ❓ Can Admin modify their own permissions?
   - ❓ What prevents unauthorized role changes?

2. **Session Hijacking**:
   - ⚠️ Session tokens stored in localStorage (vulnerable to XSS)
   - ⚠️ If attacker injects JavaScript, can steal session token
   - No httpOnly cookie protection

3. **CSRF Protection**:
   - ❌ No mention of CSRF tokens
   - State-changing operations vulnerable to CSRF
   - Example: Malicious site can POST to block user endpoint

4. **Rate Limiting**:
   - ❌ Admin endpoints need rate limiting too
   - Prevent brute force attacks
   - Prevent DoS from compromised admin account

5. **Audit Log Tampering**:
   - ⚠️ Audit logs in PostgreSQL can be modified by DB admin
   - No cryptographic signatures
   - No append-only guarantees

6. **Data Exfiltration**:
   - ❓ Admin can export all user data - what controls?
   - ❓ How to prevent malicious admin from stealing customer data?
   - No data access monitoring mentioned

7. **WebSocket Security**:
   - ❌ No authentication mentioned for WebSocket connections
   - ❓ How to prevent unauthorized WebSocket subscriptions?
   - ❓ Session validation on WebSocket handshake?

**Recommendation**:
1. Add comprehensive security threat model section:
```markdown
### Security Threat Model

#### Threat 1: Privilege Escalation
**Risk**: Staff user modifies database to become Admin
**Mitigation**:
- Audit all role changes
- Require Admin approval for role promotions
- Monitor database access logs

#### Threat 2: Session Hijacking
**Risk**: XSS attack steals session token from localStorage
**Mitigation**:
- Use httpOnly cookies instead of localStorage
- Implement Content Security Policy (CSP)
- Add session IP binding

[Continue for all threats...]
```

2. Implement security controls:
   - CSRF protection for state-changing operations
   - Rate limiting (10 requests/minute per admin)
   - Immutable audit logs with cryptographic signatures
   - Data export controls and auditing
   - WebSocket authentication with session validation

3. Add security review gate before production deployment
4. Add 5-7 days to Phase 1 for security hardening
5. Third-party security audit before production

**Priority**: 🔴 Critical - Security is non-negotiable

---

## 📊 ARCHITECTURE RECOMMENDATIONS

### 19. Alternative Architecture Consideration

**Current PRD Approach**:
Standalone Backoffice Service with event-driven read model

#### Current Approach Analysis:

**Pros**:
- ✅ Fast analytics queries (denormalized read model)
- ✅ Real-time dashboard updates (via Kafka events)
- ✅ Complete separation from customer-facing services
- ✅ Can scale independently
- ✅ Optimized for admin use cases

**Cons**:
- ❌ Massive prerequisite work (8-10 weeks)
- ❌ Complex event processing and idempotency
- ❌ Eventual consistency issues (read-after-write)
- ❌ Data duplication (transactions stored in 3 services)
- ❌ High operational overhead (3+ Kafka consumers)
- ❌ Long time to MVP (18-20 weeks total)

---

#### Alternative Approach: **Extend BFF with Admin Routes**

**Architecture**:
```
Frontend (Admin) → BFF (/v1/admin/*) → User/Transaction/Budget Services
```

**Implementation**:
```rust
// money-planner-bff/src/api/admin/mod.rs

// Admin routes with role check
pub fn admin_routes(state: Arc<AppState>) -> Router<Arc<AppState>> {
    Router::new()
        .route("/admin/users", get(list_users))
        .route("/admin/users/:id", get(get_user).put(update_user))
        .route("/admin/users/:id/block", post(block_user))
        .route("/admin/users/:id/unblock", post(unblock_user))
        .route("/admin/users/:id/transactions", get(get_user_transactions))
        .route("/admin/dashboard/metrics", get(get_dashboard_metrics))
        .layer(axum::middleware::from_fn(require_admin_role))
}

// List users (forward to User Service)
async fn list_users(
    session: SessionExtractor,
    Query(params): Query<UserSearchParams>,
) -> Result<Json<Vec<User>>> {
    // Call User Service with admin JWT
    let jwt = create_admin_jwt(&session)?;
    let users = user_service_client.list_users(params, jwt).await?;
    Ok(Json(users))
}

// Dashboard metrics (aggregate from multiple services)
async fn get_dashboard_metrics(session: SessionExtractor) -> Result<Json<DashboardMetrics>> {
    let (users, transactions, budgets) = tokio::join!(
        user_service_client.get_user_stats(),
        transaction_service_client.get_transaction_stats(),
        budget_service_client.get_budget_stats(),
    );

    Ok(Json(DashboardMetrics {
        total_users: users?.total,
        active_users: users?.active,
        total_transactions: transactions?.total,
        // ... aggregate from all services
    }))
}
```

**Pros**:
- ✅ **Minimal prerequisites** (only block/unblock endpoints needed)
- ✅ **Immediate consistency** (no eventual consistency issues)
- ✅ **No Kafka event publishing needed initially**
- ✅ Simpler architecture (reuse existing patterns)
- ✅ **Can start Phase 1 in 2-3 weeks** instead of 10 weeks
- ✅ Faster time to MVP
- ✅ Lower operational overhead

**Cons**:
- ❌ Slower analytics queries (direct service calls, not cached)
- ❌ No real-time dashboard updates initially (need polling)
- ❌ More load on source services
- ❌ Doesn't fully leverage event-driven architecture

---

#### Hybrid Approach (Recommended):

**Phase 1 (Weeks 1-4)**: BFF Admin Routes
- Extend BFF with `/v1/admin/*` routes
- Direct API calls to User/Transaction/Budget services
- Polling for dashboard metrics (refresh every 30s)
- **Deliverables**: All Phase 1 features functional
- **Prerequisites**: Only block/unblock endpoints (1 week)

**Phase 2 (Weeks 5-8)**: Add Event-Driven Read Model
- Implement Backoffice Service with Kafka consumer
- Build read model for faster analytics
- Keep BFF admin routes for write operations
- **Deliverables**: Fast analytics, historical reporting
- **Prerequisites**: Kafka event publishing (parallel with Phase 1)

**Phase 3 (Weeks 9-12)**: Add Real-time Updates
- Add WebSocket server to Backoffice Service
- Real-time dashboard metrics
- **Deliverables**: Real-time admin experience

**Phase 4 (Weeks 13-16)**: Migrate to Standalone Service
- Migrate all admin routes from BFF to Backoffice Service
- BFF focuses only on customer features
- **Deliverables**: Complete separation, independent scaling

**Benefits of Hybrid Approach**:
- ✅ **Fast MVP**: Phase 1 in 3 weeks (vs 10 weeks)
- ✅ **Incremental complexity**: Add event-driven features gradually
- ✅ **Validate requirements**: Learn from Phase 1 usage
- ✅ **Lower risk**: Can stop after Phase 1 if sufficient
- ✅ **Flexible**: Can adjust Phase 2-4 based on Phase 1 learnings

---

#### Architecture Comparison:

| Aspect | Current PRD | BFF Extension | Hybrid (Recommended) |
|--------|-------------|---------------|---------------------|
| Prerequisites | 8-10 weeks | 1 week | 1 week (Phase 1) |
| Time to MVP | 18-20 weeks | 4-5 weeks | 4-5 weeks |
| Consistency | Eventual | Immediate | Immediate → Eventual |
| Real-time | Yes (from start) | No (polling) | Added in Phase 3 |
| Complexity | High | Low | Low → High |
| Scalability | Excellent | Good | Good → Excellent |
| Operational Overhead | High | Low | Low → High |
| Data Duplication | Yes (3 copies) | No | No → Yes |

**Recommendation**:
1. **Use Hybrid Approach** for faster MVP and lower risk
2. Start with BFF admin routes (Phase 1)
3. Add event-driven read model when analytics performance needed
4. Defer standalone service until scale requires it
5. Update PRD to present all 3 options with pros/cons
6. Let stakeholders choose based on priorities (speed vs features)

**Priority**: 🔴 Critical - Affects entire project approach

---

### 20. Missing: Disaster Recovery and Data Loss

**PRD Gaps**:
- ❌ No backup strategy for read model
- ❌ No event replay mechanism if read model corrupted
- ❌ No discussion of Kafka retention policies
- ❌ What happens if backoffice database is lost?
- ❌ No monitoring for event processing lag

**Critical Questions**:

1. **If `user_summaries` table is corrupted, how to rebuild?**
   - Option A: Replay all Kafka events from beginning
   - Option B: Query source services and rebuild
   - Option C: Restore from backup
   - Need defined procedure

2. **If Kafka events are purged, can read model be rebuilt?**
   - Kafka default retention: 7 days
   - After 7 days, events are deleted
   - Read model cannot be rebuilt from events
   - Need alternative rebuild strategy

3. **Kafka retention policy?**
   - 7 days? (default, too short for rebuild)
   - 30 days? (better, allows time to detect and fix issues)
   - Forever? (expensive, unlimited storage growth)
   - Need to define and configure

4. **Read model backup frequency?**
   - Daily backups? Weekly?
   - Backup retention period?
   - Backup validation process?

5. **Event processing lag monitoring?**
   - How to detect if consumer is falling behind?
   - Alert if lag > 1 hour?
   - Dashboard showing current offset vs latest offset?

**Recommendation**:

1. **Add "Disaster Recovery" section to PRD**:
```markdown
### Disaster Recovery Strategy

#### Scenario 1: Read Model Database Corruption
**Detection**: Database integrity checks fail
**Recovery Procedure**:
1. Stop backoffice service
2. Restore database from latest backup (RTO: 15 minutes)
3. Identify backup timestamp
4. Replay Kafka events from backup timestamp to current
5. Verify data consistency
6. Restart backoffice service
**RTO**: 30 minutes
**RPO**: Last backup (4 hours max)

#### Scenario 2: Complete Database Loss
**Recovery Procedure**:
1. Provision new database
2. Run migrations to create schema
3. Query source services to rebuild user data
4. Replay all Kafka events (if within retention period)
5. Verify data consistency
**RTO**: 2-4 hours
**RPO**: Depends on Kafka retention

#### Scenario 3: Kafka Event Loss (Beyond Retention)
**Recovery Procedure**:
1. Query User Service for all users → populate user_summaries
2. Query Transaction Service for all transactions → populate transaction_summaries
3. Query Budget Service for all budgets → populate budget_summaries
4. Recalculate all aggregates
**RTO**: 4-8 hours (depends on data volume)
**RPO**: Real-time (rebuild from source of truth)
```

2. **Define Kafka Retention Policy**:
```toml
# Kafka configuration for backoffice topics
[topics]
users.events:
  retention.ms = 2592000000  # 30 days
  retention.bytes = 10737418240  # 10 GB max

transactions.events:
  retention.ms = 2592000000  # 30 days
  retention.bytes = 53687091200  # 50 GB max

budgets.events:
  retention.ms = 2592000000  # 30 days
  retention.bytes = 10737418240  # 10 GB max
```
**Rationale**: 30 days allows time to detect and fix issues before events are purged

3. **Implement Event Replay Mechanism**:
```rust
// Rebuild read model from Kafka events
async fn rebuild_read_model(from_timestamp: DateTime<Utc>) -> Result<()> {
    // 1. Clear existing read model
    db.truncate_table("user_summaries").await?;
    db.truncate_table("transaction_summaries").await?;
    db.truncate_table("budget_summaries").await?;

    // 2. Reset Kafka consumer offset to timestamp
    consumer.seek_to_timestamp(from_timestamp).await?;

    // 3. Process all events from that timestamp
    loop {
        let events = consumer.poll_batch(Duration::from_secs(10)).await?;
        if events.is_empty() {
            break; // Caught up to latest
        }

        for event in events {
            process_event(event).await?;
        }
    }

    Ok(())
}

// Rebuild from source services (if Kafka retention expired)
async fn rebuild_from_sources() -> Result<()> {
    // 1. Query all users from User Service
    let users = user_service.list_all_users().await?;
    for user in users {
        db.insert_user_summary(user).await?;
    }

    // 2. Query all transactions from Transaction Service
    let transactions = transaction_service.list_all_transactions().await?;
    for transaction in transactions {
        db.insert_transaction_summary(transaction).await?;
    }

    // 3. Query all budgets from Budget Service
    let budgets = budget_service.list_all_budgets().await?;
    for budget in budgets {
        db.insert_budget_summary(budget).await?;
    }

    Ok(())
}
```

4. **Add Monitoring for Event Processing Lag**:
```rust
// Prometheus metrics
lazy_static! {
    static ref KAFKA_CONSUMER_LAG: IntGauge = IntGauge::new(
        "backoffice_kafka_consumer_lag",
        "Number of messages behind latest offset"
    ).unwrap();
}

// Monitor consumer lag
async fn monitor_consumer_lag(consumer: &Consumer) -> Result<()> {
    let current_offset = consumer.position().await?;
    let latest_offset = consumer.end_offsets().await?;
    let lag = latest_offset - current_offset;

    KAFKA_CONSUMER_LAG.set(lag as i64);

    if lag > 1000 {
        alert_team("Kafka consumer lag exceeded 1000 messages");
    }

    Ok(())
}
```

5. **Document Rebuild Procedure**:
- Add step-by-step runbook for DBAs/SREs
- Include commands to execute
- Define escalation path if rebuild fails
- Test rebuild procedure quarterly

6. **Add to Phase 1 tasks**:
- Implement event replay mechanism (+2 days)
- Add consumer lag monitoring (+1 day)
- Document disaster recovery procedures (+1 day)
- **Total: +4 days to Phase 1**

**Priority**: ⚠️ Significant - Affects operational resilience

---

## 📋 SUMMARY OF REQUIRED PRD CHANGES

### 1. Prerequisites Section Updates

**Current PRD Claims**:
- Total: 22 days (~4.4 weeks)
- User Service: 16 days
- Transaction Service: 1 day
- Budget Service: 5 days

**Recommended Updates**:

#### User Service (22-24 days):
- ✅ Block/unblock endpoints: **3 days** (unchanged)
- ✅ User search/filtering: **3 days** (unchanged)
- ❌ Kafka event publishing: **5 days** → **10-12 days** (increase)
  - Add infrastructure, schemas, publishing logic, error handling, testing
- ✅ User update endpoint: **+2 days** (add, not estimated)
- ✅ Session event publishing: **+3 days** (add, missing from PRD)
- ✅ Staff role implementation: **3 days** (unchanged)
- ✅ Session cleanup mechanism: **+2 days** (add, missing from PRD)
- **Revised Total: 26-29 days (5.2-5.8 weeks)**

#### Transaction Service (1 day):
- ✅ Event schema verification: **1 day** (unchanged)

#### Budget Service (8-10 days):
- ✅ Kafka event publishing: **5 days** (unchanged)
- ❌ Fix transaction event processing: **+3-5 days** (add, missing from PRD)
  - Handle TransactionUpdated and TransactionDeleted events
- **Revised Total: 8-10 days (1.6-2 weeks)**

#### Buffer Time:
- Add 20% buffer for unexpected issues: **+7-8 days**

**New Total Prerequisites**:
- **42-48 days (8.5-9.5 weeks)**
- **vs PRD claim of 22 days (4.4 weeks)**
- **Underestimate: 20-26 days (100-118%)**

---

### 2. Phase 1 Timeline Updates

**Current PRD**: 4 weeks

**Recommended Additions**:
- ✅ WebSocket complexity: **+5-7 days**
  - Connection management, broadcasting, authentication
  - Or: Use SSE instead (simpler)
- ✅ Session caching strategy: **+2-3 days**
  - Implement cache to reduce User Service calls
  - Circuit breaker for User Service failures
- ✅ Robust event processing: **+3-4 days**
  - Idempotency with retries
  - Dead letter queue
  - Processing status tracking
- ✅ Security hardening: **+5-7 days**
  - CSRF protection
  - Rate limiting
  - Audit log integrity
  - XSS prevention
- ✅ Database optimization: **+2-3 days**
  - Indexes, materialized views
- ✅ Disaster recovery: **+4 days**
  - Event replay, backup procedures
- ✅ Eventual consistency handling: **+3-4 days**
  - Write-through cache or optimistic updates

**Revised Phase 1**: **4 weeks** → **6-7 weeks**

---

### 3. Architecture Section Updates

#### Add/Clarify:
1. ✅ **Topic Names**: Update to actual names
   - `int.transaction-update.event` (not `transactions.events`)
   - Or standardize to `users.events`, `transactions.events`, `budgets.events`

2. ✅ **Session Storage**: Clarify implementation
   - PostgreSQL table (not Redis)
   - 24-hour hardcoded expiry
   - Need cleanup mechanism

3. ✅ **Eventual Consistency**: Add handling strategy
   - Write-through cache for recent admin actions
   - Or optimistic UI updates
   - Or read-from-source on write operations

4. ✅ **Active Session Tracking**: Define mechanism
   - Requires User Service session events
   - Or query User Service database (violates boundaries)
   - Choose and document approach

5. ✅ **Transaction Data Duplication**: Acknowledge
   - Budget Service already caches transactions
   - Backoffice Service adds third copy
   - Document rationale or reuse Budget Service data

---

### 4. New Sections to Add

#### Add to PRD:

1. **Eventual Consistency Handling**
   - Problem statement
   - UX impact
   - Solutions (write-through cache, optimistic updates)
   - Implementation details

2. **Security Threat Model**
   - Privilege escalation
   - Session hijacking
   - CSRF attacks
   - Data exfiltration
   - Audit log tampering
   - Mitigations for each threat

3. **Disaster Recovery Strategy**
   - Scenarios (DB corruption, complete loss, Kafka retention expiry)
   - Recovery procedures (RTO, RPO)
   - Event replay mechanism
   - Backup strategy
   - Monitoring and alerting

4. **Alternative Architecture Comparison**
   - Standalone service (current)
   - BFF extension
   - Hybrid approach
   - Pros/cons table
   - Recommendation

5. **Performance Optimization Strategy**
   - Database indexes
   - Materialized views
   - Caching strategy
   - Query optimization

---

### 5. Critical Decisions Needed

Before finalizing PRD, make these decisions:

#### Decision 1: Architecture Approach
- **Option A**: Standalone Backoffice Service (current PRD)
  - Pros: Fast analytics, real-time updates, clean separation
  - Cons: 8-10 weeks prerequisites, high complexity
- **Option B**: BFF Admin Routes Extension
  - Pros: 1 week prerequisites, simple, fast MVP
  - Cons: Slower analytics, no real-time initially
- **Option C**: Hybrid (BFF → Event-driven → Standalone)
  - Pros: Fast MVP, incremental complexity, flexible
  - Cons: More phases, requires migration

**Recommendation**: Choose **Option C (Hybrid)** for lowest risk and fastest MVP

---

#### Decision 2: Real-time Updates
- **Option A**: WebSocket (bidirectional)
  - Pros: Low latency, bidirectional
  - Cons: Complex implementation (5-7 days)
- **Option B**: Server-Sent Events (SSE, unidirectional)
  - Pros: Simpler, sufficient for dashboards
  - Cons: Unidirectional only
- **Option C**: HTTP Polling
  - Pros: Simplest implementation
  - Cons: Higher latency, more load

**Recommendation**: Start with **Option C (Polling)** for Phase 1, add **Option B (SSE)** in Phase 2

---

#### Decision 3: Staff Role in MVP
- **Option A**: Include Staff role in Phase 1
  - Pros: More complete MVP
  - Cons: +5 days prerequisites, unclear requirements
- **Option B**: Admin-only for Phase 1, add Staff in Phase 2
  - Pros: Simpler, faster MVP
  - Cons: Less functional MVP

**Recommendation**: Choose **Option B (Admin-only)** for Phase 1 unless Staff requirements are crystal clear

---

#### Decision 4: Kafka Topic Naming
- **Option A**: Keep current naming (`int.transaction-update.event`)
  - Pros: No changes to existing services
  - Cons: Inconsistent naming
- **Option B**: Standardize to new naming (`users.events`, `transactions.events`)
  - Pros: Clean, consistent
  - Cons: Requires changing Transaction Service topic name

**Recommendation**: Choose **Option A** for MVP, standardize in future

---

#### Decision 5: Consistency Model
- **Option A**: Accept eventual consistency
  - Pros: Simpler implementation
  - Cons: Confusing UX (read-after-write issues)
- **Option B**: Implement write-through cache
  - Pros: Better UX
  - Cons: +3-4 days complexity
- **Option C**: Read from source on writes
  - Pros: Immediate consistency for writes
  - Cons: Slower write operations

**Recommendation**: Choose **Option B (Write-through cache)** for better UX

---

### 6. Updated Timeline Summary

**Current PRD Timeline**:
- Prerequisites: 4-5 weeks
- Phase 1-4: 16 weeks
- **Total: 20-21 weeks**

**Realistic Timeline**:
- Prerequisites: **8-10 weeks**
- Phase 1: **6-7 weeks**
- Phase 2: **4-5 weeks**
- Phase 3: **4-5 weeks**
- Phase 4: **4-5 weeks**
- **Total: 26-32 weeks**

**Underestimate**: **6-12 weeks (30-60%)**

**Alternative with Hybrid Approach**:
- Prerequisites (BFF approach): **1-2 weeks**
- Phase 1 (BFF admin routes): **3-4 weeks**
- Phase 2 (Event-driven read model): **6-7 weeks** (parallel with Phase 1 if desired)
- Phase 3 (Real-time updates): **4-5 weeks**
- Phase 4 (Standalone migration): **4-5 weeks**
- **Total: 18-23 weeks** (faster MVP at week 4-6)

---

## 🎯 RECOMMENDED NEXT STEPS

### Immediate Actions (Week 0):

1. **Review this Analysis**
   - Share with technical lead and stakeholders
   - Discuss findings and recommendations
   - Prioritize which issues to address

2. **Make Critical Decisions**
   - Architecture approach (Standalone vs BFF vs Hybrid)
   - Real-time updates mechanism (WebSocket vs SSE vs Polling)
   - Staff role in MVP (Include vs Defer)
   - Consistency model (Eventual vs Write-through cache)

3. **Update PRD Document**
   - Revise prerequisites timeline (8-10 weeks)
   - Add missing sections (security, disaster recovery, consistency)
   - Update Phase 1 timeline (6-7 weeks)
   - Document all architecture decisions
   - Add alternative approaches comparison

---

### Prerequisites Work (Weeks 1-10):

#### If Choosing Standalone Architecture:

**Week 1-2: User Service - Block/Unblock + Search**
- Implement `POST /v1/users/{userId}/block`
- Implement `POST /v1/users/{userId}/unblock`
- Implement user search/filtering on `GET /v1/users`
- Add pagination support
- Add indexes on email, state, role

**Week 3-5: User Service - Kafka Event Publishing**
- Add rdkafka dependency
- Implement Kafka producer infrastructure
- Define event schemas (user.created, user.updated, user.blocked, user.unblocked)
- Implement publishing logic in all user operations
- Add error handling and retries
- Add idempotency
- Integration testing

**Week 5-6: User Service - Session Events**
- Implement session event publishing (session.created, session.expired, session.deleted)
- Add session cleanup background job

**Week 6: Transaction Service - Verification**
- Verify event schema matches requirements
- Document current topic name

**Week 7-8: Budget Service - Event Publishing**
- Implement budget event publishing (budget.created, budget.updated, budget.deleted)
- Define event schemas

**Week 8-9: Budget Service - Fix Transaction Event Handling**
- Handle TransactionUpdated events
- Handle TransactionDeleted events
- Update spent amounts correctly

**Week 10: Buffer and Integration Testing**

---

#### If Choosing Hybrid/BFF Architecture:

**Week 1: User Service - Block/Unblock + Search**
- Implement `POST /v1/users/{userId}/block`
- Implement `POST /v1/users/{userId}/unblock`
- Implement user search/filtering on `GET /v1/users`

**Week 2: BFF - Admin Routes**
- Implement `/v1/admin/*` routes in BFF
- Add admin role authorization middleware
- Forward requests to User/Transaction/Budget services

**Parallel Track: Kafka Event Publishing (if adding in Phase 2)**
- Can proceed in parallel with Phase 1 development
- Ready for Phase 2 event-driven read model

---

### Phase 1 Implementation (Weeks 11-17 or Weeks 3-9 for Hybrid):

**Week 1-2: Backoffice Service Setup**
- Create new Rust/Axum project
- Set up database schema
- Implement session validation
- Implement role-based authorization

**Week 3-4: Core Features**
- User search interface
- User details page
- Block/unblock functionality
- User transactions view

**Week 5-6: Dashboard**
- Implement dashboard metrics (from service calls or read model)
- User statistics
- Active sessions tracking
- Real-time updates (polling, SSE, or WebSocket)

**Week 7: Testing and Refinement**
- Integration testing
- Security hardening
- Performance optimization
- Bug fixes

---

### Phase 2-4 (As Originally Planned, with Adjustments):

Proceed with Phase 2-4 based on chosen architecture and Phase 1 learnings.

---

### Risk Mitigation:

1. **Weekly Progress Reviews**
   - Track actual vs estimated timelines
   - Adjust remaining estimates based on learnings

2. **Prototype Early**
   - Build WebSocket/SSE prototype before committing
   - Test Kafka event processing with sample data
   - Validate session caching strategy

3. **Incremental Deployment**
   - Deploy to staging after each phase
   - Gather feedback from internal users
   - Iterate before production

4. **Fallback Plans**
   - If prerequisites take too long, consider BFF approach
   - If WebSocket too complex, fall back to SSE or polling
   - If Staff role unclear, defer to Phase 2

---

## Conclusion

This PRD analysis reveals **significant gaps between the document's assumptions and actual codebase reality**. The most critical findings are:

1. **Prerequisites severely underestimated**: 4.4 weeks → 8-10 weeks (118% underestimate)
2. **Missing critical features**: User Service event publishing, block/unblock endpoints
3. **Incomplete implementations**: Budget Service event processing
4. **Unaddressed complexities**: Eventual consistency, WebSocket implementation, security threats
5. **Total timeline underestimate**: 20 weeks → 26-32 weeks (30-60% underestimate)

**Recommended Path Forward**:
- Consider **Hybrid Architecture** for fastest MVP (4-6 weeks to working backoffice)
- Complete realistic prerequisites (8-10 weeks if standalone, 1-2 weeks if BFF)
- Add missing sections to PRD (security, disaster recovery, consistency)
- Make critical architecture decisions before starting implementation
- Set realistic stakeholder expectations (26-32 weeks total, not 20 weeks)

The backoffice application is achievable and valuable, but requires **more time, more prerequisites, and more careful planning** than the current PRD suggests.

---

**End of Analysis**

**Generated**: November 9, 2025
**Analyzer**: Claude Code
**PRD Version**: 1.0 (Draft)
**Status**: Ready for Review
