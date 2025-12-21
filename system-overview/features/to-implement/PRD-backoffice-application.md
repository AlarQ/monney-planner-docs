# PRD: Money Planner Backoffice Application

## Document Information
- **Version**: 1.0
- **Date**: November , 2025
- **Status**: Draft

## Executive Summary

The Money Planner Backoffice Application is a comprehensive administrative platform designed to provide Money Planner staff and administrators with the tools necessary to manage users, monitor system health, oversee financial data, and maintain operational efficiency. This application uses a **standalone backoffice service** (port 8084) with event-driven architecture for real-time dashboards and analytics, combined with direct API calls for immediate consistency on write operations. The frontend connects directly to the Backoffice Service (HTTP + WebSocket), and the service consumes Kafka events from User, Transaction, and Budget services to build a denormalized read model. It includes a dedicated admin frontend and serves as a centralized hub for all administrative functions across the Money Planner microservices ecosystem.

## Current System Analysis

### Architecture Overview
Money Planner is built on a microservices architecture consisting of:
- **User Service** (port 8080): Authentication, user management, role-based access control
- **Transaction Service** (port 8081): Transaction management, AI categorization, CSV import
- **Budget Service** (port 8083): Budget tracking, analytics, real-time calculations
- **BFF (Backend for Frontend)** (port 8082): API gateway and request orchestration
- **Frontend** (port 3000): React-based user interface for customer-facing features
- **Infrastructure**: PostgreSQL databases, Kafka messaging, Kubernetes deployment

### Current User Roles
The system currently supports three user roles in the User Service:
- **Customer**: Standard users with access to personal financial data
- **Staff**: Mid-level administrative access (currently undefined functionality)
- **Admin**: Full system access with user management capabilities

### Existing Administrative Capabilities
- User role-based access control with Admin privileges
- User state management (Active/Blocked) for account suspension
- Session-based authentication system (not JWT for frontend-to-BFF)
  - Frontend → BFF: Session tokens validated via User Service `/v1/auth/session`
  - BFF → Backend Services: JWT service tokens for service-to-service communication
  - Sessions contain user_id and role (Admin/Staff/Customer) for authorization
- Cross-service authorization through shared JWT service tokens

## Business Case & Objectives

### Primary Goals
1. **Operational Efficiency**: Reduce manual administrative tasks and provide automated monitoring
2. **Customer Support**: Enable staff to efficiently assist customers with account and transaction issues
3. **System Monitoring**: Provide real-time visibility into system health and performance
4. **Compliance**: Ensure proper audit trails and regulatory compliance for financial operations
5. **Data Insights**: Deliver analytics and reporting for business decision-making

## Scope & Requirements

### In Scope
- Administrative dashboard with system overview
- User management and customer support tools
- Transaction monitoring
- System health monitoring and alerting
- Analytics and reporting functionality
- Audit logging and compliance features
- Administrative workflow automation

### Out of Scope
- Customer-facing features (handled by existing frontend)
- Direct database modifications (all changes through APIs)
- Financial calculations (delegated to existing services)
- External integrations beyond existing system scope

## User Personas

### Staff Administrator (MVP Role)
- **Role**: Multi-functional administrative staff handling operations, support, and monitoring
- **Needs**:
  - User account management and customer support capabilities
  - Transaction monitoring
  - System health visibility and alerting
  - Basic analytics and operational reporting
- **Pain Points**:
  - No centralized administrative interface
  - Cannot assist customers with account issues
  - Limited visibility into system operations
  - Manual processes for routine administrative tasks

## Feature Delivery Phases

### Phase 1: MVP - Core Admin Operations (Weeks 1-4)
**Business Value**: Enable basic administrative operations for user management and system visibility

#### Authentication & Basic Access
- **Session-Based Authentication**: Leverage existing session-based authentication system (not JWT)
  - Frontend → Backoffice: Uses session tokens (validated via User Service `/v1/auth/session`)
  - Backoffice → Backend Services: Uses JWT service tokens for service-to-service communication
  - Session validation ensures immediate revocation and proper session lifecycle management
- **Role-Based Access**: Basic Staff/Admin role validation
  - Admin/Staff roles are validated from session data (user role stored in User Service)
  - Session contains user_id and role information for authorization checks
- **Session Management**: Secure session handling
  - Reuse existing User Service session system (no separate admin sessions)
  - Admin/Staff users authenticate through same login flow as customers
  - Role-based access control enforced at API endpoint level

#### Core User Management
- **User Search**: Search users by email, ID, or name
  - **Prerequisite**: User Service must implement search/filtering on `GET /v1/users` endpoint
  - Requires query parameters: `?email=...&search=...&page=...&limit=...`
- **User Details**: View user profile, registration info, and account status
- **Account Actions**: Block/unblock users (critical for support)
  - **Prerequisite**: User Service must implement block/unblock endpoints
  - Required endpoints: `POST /v1/users/{userId}/block` and `POST /v1/users/{userId}/unblock`
  - Alternative: Implement full `PUT /v1/users/{userId}` endpoint with state field
  - **Note**: `UserState` enum exists (Active/Blocked) but no API to change it currently
- **User Transactions**: View transactions for a specific user
  - List user's transactions (from Transaction Service)
  - Basic transaction details (amount, date, description, category)
  - Filter by date range

#### Basic Dashboard & Statistics
- **User Statistics**:
  - Total users count
  - Active users (users with activity in last 30 days)
  - Users with active sessions
  - Recent registrations (last 7 days, last 30 days)
  - User growth trend (daily/weekly)
- **Service Status**: Basic health checks for all microservices
- **Quick Actions**: Access to primary administrative functions (user search, block/unblock)

**Success Criteria**: 
- Admin staff can search for users, view user details, and block/unblock accounts
- Admin staff can view a user's transactions
- Admin staff can see real-time statistics on active users and sessions
- Dashboard loads with current system statistics

### Phase 2: Transaction Investigation & Analytics (Weeks 5-8)
**Business Value**: Enable comprehensive transaction investigation and basic analytics

#### Advanced Transaction Management
- **Transaction Search**: Global search across all transactions (not just per-user)
  - Search by user, amount, date range, category, description
  - Advanced filtering and sorting
- **Transaction Details**: View complete transaction history with full context
- **Transaction Analytics**: 
  - Transaction volume trends (daily/weekly/monthly)
  - Revenue/expense breakdowns
  - Category distribution
  - Transaction patterns

#### Enhanced Dashboard
- **Transaction Metrics**: 
  - Daily transaction volume
  - Revenue/expense trends
  - Average transaction amounts
  - Transaction growth trends
- **System Performance**: Response times and error rates for services
- **Activity Overview**: Recent transactions, user activity patterns

**Success Criteria**: Staff can search and investigate transactions across the system and view transaction analytics

### Phase 3: Advanced Analytics & System Management (Weeks 9-12)
**Business Value**: Operational insights and system optimization

#### Advanced Analytics
- **Budget Performance**: System-wide budget utilization and performance metrics
- **User Behavior**: Usage patterns and engagement metrics
- **Category Analytics**: Popular categories and spending patterns
- **Custom Reports**: Configurable reporting with export functionality
- **Trend Analysis**: Historical trends and predictive insights

#### System Administration
- **Configuration Management**: View and update system settings
- **Performance Monitoring**: Detailed service metrics and database status
- **Kafka Monitoring**: Topic health and message throughput
- **System Health**: Comprehensive system health dashboard

#### Enhanced Security & Audit
- **Granular Permissions**: Define specific permissions for functions
- **Activity History**: Detailed user login and transaction history
- **Advanced Audit**: Comprehensive audit trails for all administrative actions

**Success Criteria**: Operations team has visibility into system performance, user behavior, and can manage system configuration

### Phase 4: Automation, Compliance & Advanced Features (Weeks 13-16)
**Business Value**: Automated operations, regulatory compliance, and advanced administrative features

#### Compliance & Reporting
- **Regulatory Reports**: Generate compliance reports (GDPR, CCPA, etc.)
- **Audit Trails**: Immutable audit logs with tamper detection
- **Data Privacy**: GDPR/CCPA compliance monitoring and reporting
- **Data Export**: User data export for compliance requests

#### Alert & Notification System
- **Real-time Alerts**: Immediate notification of critical issues
- **Alert Routing**: Route alerts to appropriate team members
- **Integration**: Slack/email integration for notifications
- **Alert Management**: Alert history and resolution tracking

#### Advanced Administration
- **Staff Management**: Create and manage administrative accounts
- **Data Management**: Data integrity checks and backup monitoring
- **Feature Flags**: Toggle features without deployment
- **System Maintenance**: Maintenance mode and scheduled tasks

**Success Criteria**: System operates with minimal manual intervention, full compliance capabilities, and advanced administrative features

## Technical Specifications

### Architecture & Technology Stack

#### Backend Services

**Standalone Backoffice Service Architecture**: Complete backoffice service with WebSocket server, database, and admin APIs

**Rationale for Standalone Service**:
- **Real-time Dashboards**: Event-driven read model enables fast, real-time dashboards without polling
- **Fast Analytics**: Denormalized read model provides fast queries for reporting and analytics
- **Immediate Consistency for Writes**: Direct API calls ensure immediate consistency for admin actions
- **Separation of Concerns**: Admin service completely separate from customer-facing BFF
- **Simpler Architecture**: Frontend connects directly to Backoffice Service (HTTP + WebSocket)
- **No BFF Complexity**: BFF doesn't need WebSocket server, admin routing, or connection management
- **Scalability**: Can scale Backoffice Service independently
- **Clean Boundaries**: Clear separation between customer and admin functionality

**Architecture**:
```
┌─────────────────────────────────────┐
│         Frontend (port 3000)       │
│         Admin Frontend              │
└─────────────────────────────────────┘
            │
    ┌───────┴───────┐
    │               │
    │ (HTTP)        │ (WebSocket)
    ▼               ▼
┌─────────────────────────────────────┐
│  Backoffice Service (port 8084)    │
│  ┌─────────────────────────────────┐ │
│  │  Admin APIs                     │ │
│  │  - Read operations (from model) │ │
│  │  - Write operations (to services)│ │
│  │  - Dashboards                   │ │
│  │  - Analytics                    │ │
│  └─────────────────────────────────┘ │
│  ┌─────────────────────────────────┐ │
│  │  WebSocket Server                │ │
│  │  - Real-time updates             │ │
│  │  - Dashboard metrics             │ │
│  │  - Alert notifications           │ │
│  └─────────────────────────────────┘ │
│  ┌─────────────────────────────────┐ │
│  │  Kafka Consumer                  │ │
│  │  - Consumes user events          │ │
│  │  - Consumes transaction events   │ │
│  │  - Consumes budget events        │ │
│  └─────────────────────────────────┘ │
│  ┌─────────────────────────────────┐ │
│  │  Denormalized Read Model         │ │
│  │  - User summaries                │ │
│  │  - Transaction summaries         │ │
│  │  - Budget summaries              │ │
│  │  - Aggregated views              │ │
│  └─────────────────────────────────┘ │
└─────────────────────────────────────┘
            │
            │ (Consumes events)
            ▼
┌─────────────────────────────────────┐
│           Kafka                    │
│  ┌─────────┬─────────┬─────────────┐ │
│  │users    │trans-   │budgets      │ │
│  │.events  │actions  │.events      │ │
│  │         │.events  │             │ │
│  └─────────┴─────────┴─────────────┘ │
└─────────────────────────────────────┘
            ▲
            │ (Publish events)
            │
┌─────────────────────────────────────┐
│        Existing Services            │
│  ┌─────────┬─────────┬─────────────┐ │
│  │User     │Trans-   │Budget       │ │
│  │Service  │action   │Service      │ │
│  │:8080    │Service  │:8083        │ │
│  │         │:8081    │             │ │
│  └─────────┴─────────┴─────────────┘ │
└─────────────────────────────────────┘
```

**Request Flow**:
- **Read Operations** (GET): Frontend → Backoffice Service (from denormalized read model)
- **Write Operations** (POST/PUT/DELETE): Frontend → Backoffice Service → User/Transaction/Budget Services (direct API calls)
- **Real-time Updates** (WebSocket): Frontend → Backoffice Service (WebSocket connection)
- **Event Flow**: Services → Kafka → Backoffice Service (updates read model)

**Technical Stack**:
- **Language**: Rust with Axum framework (consistent with existing services)
- **Database**: PostgreSQL with dedicated backoffice schema
  - **Denormalized Read Model**: User summaries, transaction summaries, budget summaries
  - **Aggregated Views**: Materialized views for analytics and reporting
  - **Audit Logs**: Event-sourced audit trail from consumed events
- **Authentication**: Session-based authentication with existing User Service
  - Frontend → Backoffice Service: Session tokens validated via User Service `/v1/auth/session`
  - Backoffice Service → Backend Services: JWT service tokens (same pattern as BFF)
  - Admin/Staff roles validated from session user data
  - Session validation: Backoffice Service calls User Service to validate session and get user role
- **API Documentation**: OpenAPI/Swagger with utoipa
- **Event Processing**: Kafka consumer for real-time updates
  - **Consumer**: Consumes events from `users.events`, `transactions.events`, `budgets.events`
  - **Event Handlers**: Process events and update denormalized read model
  - **Idempotency**: Event processing with idempotency checks using `event_id`

**Backoffice Service Responsibilities**:
- **Admin APIs**: Provides all admin endpoints (read and write operations)
  - Read operations: From denormalized read model (fast queries)
  - Write operations: Direct API calls to User/Transaction/Budget Services
- **WebSocket Server**: Maintains WebSocket connections for real-time dashboard updates
  - WebSocket endpoint: `/v1/admin/ws`
  - Broadcasts real-time updates to all connected admin frontend clients
  - Dashboard metrics updates
  - Alert notifications
  - User activity events
  - Transaction events
- **Event Consumer**: Consumes Kafka events from User, Transaction, and Budget services
- **Read Model Builder**: Maintains denormalized read model from consumed events
- **Authentication**: Validates session tokens via User Service and enforces role-based authorization
- **System Monitoring**: Health checks and metrics aggregation
- **Audit Logging**: Event-sourced audit trail from consumed events
- **Administrative Data**: Maintains admin-specific data (alerts, configurations, audit logs)
  - Note: Sessions are managed by User Service, not stored in backoffice database

**Benefits of Standalone Service**:
- **Real-time Dashboards**: Event-driven read model enables fast, real-time dashboards with WebSocket
- **Fast Analytics**: Denormalized read model provides fast queries for reporting
- **Immediate Consistency for Writes**: Direct API calls ensure immediate consistency for admin actions
- **Complete Separation**: Admin service completely independent from customer-facing BFF
- **Simpler Architecture**: Frontend connects directly to Backoffice Service (no BFF routing complexity)
- **No BFF Bloat**: BFF stays focused on customer features only
- **Scalability**: Can scale Backoffice Service independently
- **Optimized for Admin**: Can optimize queries, WebSocket connections, and APIs specifically for admin use cases
- **Clean Boundaries**: Clear separation between customer and admin functionality

#### Frontend Application

**Architecture**: Backoffice frontend connects directly to Backoffice Service (HTTP + WebSocket)

**API Consumption Pattern**:
- **Base URL**: Backoffice Service (`http://localhost:8084/v1` or configured via `REACT_APP_ADMIN_API_BASE_URL`)
- **Admin Routes**: All admin operations use `/v1/admin/*` endpoints on Backoffice Service
- **Authentication**: Session-based authentication (validated via User Service)
  - Session token stored in `localStorage` as `access_token`
  - Token sent in `Authorization: Bearer <session_token>` header
  - Backoffice Service validates session via User Service `/v1/auth/session` and enforces Admin/Staff role authorization
- **API Client**: Axios instance with interceptors (consistent with existing frontend pattern)
  - Request interceptor: Adds `Authorization` header with session token
  - Response interceptor: Handles 401 (unauthorized) and redirects to login
- **WebSocket**: Direct WebSocket connection to Backoffice Service
  - WebSocket endpoint: `ws://localhost:8084/v1/admin/ws` (or `wss://` in production)
  - Session token sent in WebSocket handshake (query param or header)
  - Real-time updates for dashboards, alerts, and metrics

**Request Flow**:
```
Admin Frontend → Backoffice Service (HTTP) → User/Transaction/Budget Services (write ops)
Admin Frontend → Backoffice Service (HTTP) → Read Model (read ops)
Admin Frontend → Backoffice Service (WebSocket) → Real-time updates
```

**Example API Calls**:
```typescript
// Read operations (from Backoffice Service read model)
GET /v1/admin/users              // List users (from read model)
GET /v1/admin/users/{userId}     // Get user details (from read model)
GET /v1/admin/dashboard/metrics  // Dashboard metrics (from aggregated views)

// Write operations (Backoffice Service calls User Service)
POST /v1/admin/users/{userId}/block    // Block user (Backoffice Service → User Service)
POST /v1/admin/users/{userId}/unblock  // Unblock user (Backoffice Service → User Service)
```

**Technical Stack**:
- **Framework**: React 18 with TypeScript (consistent with existing frontend)
- **UI Library**: Material-UI v5 (consistent with existing design system)
- **State Management**: React Context API with custom hooks
- **Charts & Analytics**: Recharts for data visualization
- **Real-time Updates**: WebSocket connection to Backoffice Service for live dashboard updates
- **API Client**: Axios with interceptors (same pattern as customer frontend)
- **Configuration**: Environment variables via `REACT_APP_ADMIN_API_BASE_URL` (separate from customer frontend)
- **WebSocket Client**: WebSocket connection to Backoffice Service for real-time metrics and alerts

#### Infrastructure
- **Deployment**: Kubernetes with Helm charts
- **Service Mesh**: Integrate with existing microservices architecture
- **Monitoring**: Prometheus and Grafana integration
- **Security**: TLS encryption, network policies, secret management

### API Design

#### RESTful Endpoints

**Read Operations** (routed to Backoffice Service - from denormalized read model):

**Phase 1 (MVP)**:
```
/v1/admin/dashboard
  GET /metrics              # User statistics (total users, active users, active sessions, recent registrations)
  GET /health              # Service health status

/v1/admin/users
  GET /                    # List users with pagination and filtering (from read model)
  GET /{userId}            # Get user details (from read model)
  GET /{userId}/transactions # Get user's transactions (direct API call to Transaction Service)

/v1/admin/transactions
  GET /{transactionId}      # Get transaction details (direct API call to Transaction Service)
```

**Phase 2+**:
```
/v1/admin/dashboard
  GET /alerts              # Active alerts summary (Phase 4)

/v1/admin/transactions
  GET /                    # List transactions with advanced filtering (from read model) - Phase 2
  GET /analytics           # Transaction analytics (from aggregated views) - Phase 2

/v1/admin/budgets
  GET /analytics           # Budget system analytics (from aggregated views) - Phase 3
  GET /performance         # Budget performance metrics (from aggregated views) - Phase 3

/v1/admin/system
  GET /audit-logs          # Administrative audit logs (from event-sourced audit trail) - Phase 3
```

**Write Operations** (routed to User/Transaction/Budget Services - direct API calls):

**Phase 1 (MVP)**:
```
/v1/admin/users
  POST /{userId}/block     # Block user account → User Service
  POST /{userId}/unblock   # Unblock user account → User Service
```

**Phase 2+**:
```
/v1/admin/users
  PUT /{userId}            # Update user information → User Service (Phase 2)

/v1/admin/transactions
  POST /{transactionId}/review # Mark transaction as reviewed → Transaction Service (Phase 2)

/v1/admin/system
  GET /configuration       # System configuration → Backoffice Service (Phase 3)
  PUT /configuration       # Update system settings → Backoffice Service (Phase 3)
```

**Note**: Read operations use denormalized read model (fast, eventually consistent), write operations use direct API calls (immediate consistency). Endpoints are implemented incrementally across phases.

#### Frontend API Integration

**API Client Setup**:
```typescript
// api/axiosConfig.ts
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_ADMIN_API_BASE_URL || 'http://localhost:8084/v1',
  timeout: 10000,
});

// Request interceptor: Add session token
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor: Handle 401 (unauthorized)
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Redirect to login
      localStorage.removeItem('access_token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

**Admin API Client**:
```typescript
// api/adminApi.ts
import apiClient from './axiosConfig';

// Read operations (from Backoffice Service read model)
export const getUsers = async (params?: { page?: number; limit?: number; search?: string }) => {
  return apiClient.get('/admin/users', { params });
};

export const getUser = async (userId: string) => {
  return apiClient.get(`/admin/users/${userId}`);
};

export const getDashboardMetrics = async () => {
  return apiClient.get('/admin/dashboard/metrics');
};

// Write operations (direct API calls to services)
export const blockUser = async (userId: string) => {
  return apiClient.post(`/admin/users/${userId}/block`);
};

export const unblockUser = async (userId: string) => {
  return apiClient.post(`/admin/users/${userId}/unblock`);
};
```

**Authentication Flow**:
1. Admin/Staff user logs in via User Service `/v1/auth/login` (via Backoffice Service or directly)
2. User Service returns `sessionId` and `userId` (same response format)
3. Frontend stores `sessionId` in `localStorage` as `access_token`
4. All subsequent requests include `Authorization: Bearer <session_token>` header
5. Backoffice Service validates session via User Service `/v1/auth/session` and checks Admin/Staff role
6. Backoffice Service routes write operations to User/Transaction/Budget Services

**Error Handling**:
- **401 Unauthorized**: Session expired or invalid → Redirect to login
- **403 Forbidden**: User doesn't have Admin/Staff role → Show access denied
- **404 Not Found**: Resource not found → Show not found message
- **500 Server Error**: Backend error → Show error notification

#### Real-time Data Streams

**Architecture**: Frontend uses WebSocket connection to Backoffice Service for real-time dashboard updates

**WebSocket Flow**:
```
Frontend (WebSocket) → Backoffice Service (WebSocket Server)
                              ↓
                        Kafka Events (consumed by Backoffice Service)
```

**WebSocket Connection**:
- **Endpoint**: `ws://localhost:8084/v1/admin/ws` (or `wss://` in production)
- **Protocol**: WebSocket for bidirectional communication
- **Authentication**: Session token sent in WebSocket handshake (query param or header)
- **Reconnection**: Automatic reconnection on disconnect with exponential backoff

**Real-time Updates**:
- **Dashboard Metrics**: Live system metrics (user counts, transaction volumes, etc.)
- **Alerts**: Real-time alert notifications
- **User Activity**: Live user activity updates
- **Transaction Events**: Real-time transaction creation/updates
- **System Health**: Live service health status

**Frontend WebSocket Client**:
```typescript
// hooks/useWebSocket.ts
import { useEffect, useRef, useState } from 'react';

export const useAdminWebSocket = (onMessage: (data: any) => void) => {
  const [connected, setConnected] = useState(false);
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectTimeoutRef = useRef<NodeJS.Timeout>();

  useEffect(() => {
      const connect = () => {
      const token = localStorage.getItem('access_token');
      const wsUrl = `ws://localhost:8084/v1/admin/ws?token=${token}`;
      
      const ws = new WebSocket(wsUrl);
      
      ws.onopen = () => {
        setConnected(true);
        console.log('WebSocket connected');
      };
      
      ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        onMessage(data);
      };
      
      ws.onerror = (error) => {
        console.error('WebSocket error:', error);
      };
      
      ws.onclose = () => {
        setConnected(false);
        // Reconnect with exponential backoff
        const timeout = Math.min(1000 * Math.pow(2, reconnectAttempts), 30000);
        reconnectTimeoutRef.current = setTimeout(connect, timeout);
      };
      
      wsRef.current = ws;
    };
    
    connect();
    
    return () => {
      if (wsRef.current) {
        wsRef.current.close();
      }
      if (reconnectTimeoutRef.current) {
        clearTimeout(reconnectTimeoutRef.current);
      }
    };
  }, [onMessage]);
  
  return { connected };
};
```

**Backoffice Service WebSocket Server Responsibilities**:
- **WebSocket Server**: Maintains WebSocket connections to admin frontend clients
- **Event Broadcasting**: Broadcasts real-time updates to all connected admin clients
  - Updates triggered by consumed Kafka events
  - Dashboard metrics updates
  - Alert notifications
  - User activity events
  - Transaction events
- **Authentication**: Validates session token on WebSocket connection via User Service
- **Role Authorization**: Ensures only Admin/Staff users can connect
- **Update Frequency**: Configurable (e.g., every second for metrics, immediate for alerts)

**Alternative: Server-Sent Events (SSE)**:
- **Endpoint**: `GET /v1/admin/events` (SSE stream)
- **Protocol**: Server-Sent Events (unidirectional from server to client)
- **Use Case**: Simpler than WebSocket if bidirectional communication not needed
- **Pros**: Simpler implementation, automatic reconnection
- **Cons**: Unidirectional only, less efficient than WebSocket

**Recommendation**: Use WebSocket for real-time dashboards to enable:
- Bidirectional communication (client can send commands)
- Lower latency than HTTP polling
- Efficient real-time updates
- Better user experience for live dashboards

### Database Schema

#### Backoffice-Specific Tables

**Denormalized Read Model** (built from consumed Kafka events):
```sql
-- User summaries (from user events)
user_summaries (
  id UUID PRIMARY KEY,
  email VARCHAR(255),
  role VARCHAR(50),
  state VARCHAR(50),
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  last_login_at TIMESTAMP,
  total_transactions INTEGER DEFAULT 0,
  total_budgets INTEGER DEFAULT 0,
  -- Denormalized fields for fast queries
  INDEX idx_email (email),
  INDEX idx_role (role),
  INDEX idx_state (state),
  INDEX idx_created_at (created_at)
);

-- Transaction summaries (from transaction events)
transaction_summaries (
  id UUID PRIMARY KEY,
  user_id UUID,
  description TEXT,
  amount DECIMAL(10,2),
  transaction_type VARCHAR(50),
  category_id UUID,
  category_name VARCHAR(255),
  date TIMESTAMP,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  -- Denormalized fields for fast queries
  INDEX idx_user_id (user_id),
  INDEX idx_date (date),
  INDEX idx_category_id (category_id),
  INDEX idx_transaction_type (transaction_type)
);

-- Budget summaries (from budget events)
budget_summaries (
  id UUID PRIMARY KEY,
  user_id UUID,
  name VARCHAR(255),
  amount DECIMAL(10,2),
  period_type VARCHAR(50),
  spent DECIMAL(10,2) DEFAULT 0,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  -- Denormalized fields for fast queries
  INDEX idx_user_id (user_id),
  INDEX idx_period_type (period_type)
);

-- Aggregated views for analytics
daily_transaction_metrics (
  date DATE PRIMARY KEY,
  total_transactions INTEGER DEFAULT 0,
  total_amount DECIMAL(10,2) DEFAULT 0,
  total_users INTEGER DEFAULT 0,
  updated_at TIMESTAMP
);

-- Event processing tracking
processed_events (
  event_id UUID PRIMARY KEY,
  event_type VARCHAR(255),
  topic VARCHAR(255),
  partition INTEGER,
  offset BIGINT,
  processed_at TIMESTAMP,
  -- For idempotency and event replay
  INDEX idx_event_type (event_type),
  INDEX idx_processed_at (processed_at)
);
```

**Administrative Tables**:
```sql
-- Note: Sessions are managed by User Service, not stored in backoffice database
-- Admin/Staff users use the same session system as customers
-- Session validation is done via User Service /v1/auth/session endpoint

-- Audit logs (event-sourced from consumed events)
audit_logs (
  id UUID PRIMARY KEY,
  event_id UUID UNIQUE, -- From consumed event
  user_id UUID,
  action VARCHAR(255),
  resource_type VARCHAR(100),
  resource_id TEXT,
  old_values JSONB,
  new_values JSONB,
  timestamp TIMESTAMP,
  ip_address INET,
  -- Indexed for fast queries
  INDEX idx_user_id (user_id),
  INDEX idx_action (action),
  INDEX idx_timestamp (timestamp)
);

-- System alerts
system_alerts (
  id UUID PRIMARY KEY,
  alert_type VARCHAR(100),
  severity VARCHAR(20),
  title VARCHAR(255),
  description TEXT,
  source_service VARCHAR(100),
  metadata JSONB,
  created_at TIMESTAMP,
  resolved_at TIMESTAMP,
  resolved_by UUID REFERENCES users(id)
);

```

### Security Considerations

#### Authentication & Authorization
- **Session-Based Authentication**: Use existing session-based authentication system
  - Admin/Staff users authenticate through User Service login endpoint (same as customers)
  - Session tokens validated via User Service `/v1/auth/session` endpoint
  - Session contains user_id and role (Admin/Staff) for authorization
  - No separate admin session system - reuse existing User Service sessions
- **Role Validation**: Admin/Staff roles validated from session user data
  - Session validation returns user object with role information
  - Role-based middleware enforces access control at endpoint level
  - Staff role permissions defined separately from Admin (see Permission Matrix)
- **Permission Matrix**: Define granular permissions for each administrative function
  - Admin: Full access to all administrative functions
  - Staff: Limited access based on defined permission set (see Staff Role Definition)
- **Session Security**: Secure session management with encryption and timeout
  - Leverage existing User Service session security (immediate revocation, timeout)
  - Session tokens are UUIDs managed by User Service
- **API Security**: Rate limiting, input validation, SQL injection prevention

#### Data Protection
- **Sensitive Data**: Encrypt sensitive data at rest and in transit
- **Data Masking**: Mask sensitive customer data in administrative interfaces
- **Access Controls**: Strict access controls for customer financial data
- **Audit Requirements**: Comprehensive audit logging for all data access

#### Network Security
- **Network Segmentation**: Isolate administrative services from customer-facing services
- **VPN Access**: Require VPN for administrative access
- **IP Whitelisting**: Restrict access to known administrative IP addresses
- **Certificate Management**: Proper TLS certificate management and rotation

## Prerequisites

Before starting development of the Backoffice Application, the following changes must be implemented in existing services. These are **critical dependencies** that must be completed before starting each phase.

### User Service Prerequisites (Critical for Phase 1)

#### Required Endpoints (Critical for Phase 1)
1. **Block/Unblock User Endpoints**
   - `POST /v1/users/{userId}/block` - Block a user account
   - `POST /v1/users/{userId}/unblock` - Unblock a user account
   - **Alternative**: Implement full `PUT /v1/users/{userId}` endpoint with `state` field
   - **Current State**: `UserState` enum exists (Active/Blocked) but no API to change user state
   - **Impact**: Phase 1 cannot be completed without these endpoints
   - **Estimated Time**: 3 days

2. **User Search/Filtering**
   - Enhance `GET /v1/users` endpoint with query parameters:
     - `?email=...` - Filter by email
     - `?search=...` - Search by email, ID, or name (if name field exists)
     - `?page=...&limit=...` - Pagination support
   - **Current State**: Only `GET /v1/users` exists (Admin only, no filtering/pagination)
   - **Impact**: Phase 1 user search functionality requires this
   - **Estimated Time**: 3 days

3. **User Update Endpoint** (Optional but Recommended)
   - `PUT /v1/users/{userId}` - Update user information including state
   - **Current State**: Endpoint exists but is commented out/TODO
   - **Note**: If full update endpoint is implemented, block/unblock endpoints may not be needed
   - **Estimated Time**: 2 days

#### Kafka Event Publishing (Critical for Standalone Service Approach)

**Topic**: `users.events`

**Required Events**:
1. **`user.created`** - Published when a new user is created
   - **Trigger**: After successful user creation in `create_user` function
   - **Event Data**:
     ```json
     {
       "event_id": "uuid",
       "event_type": "user.created",
       "timestamp": "2024-01-15T10:30:00Z",
       "version": "1.0",
       "data": {
         "user": {
           "id": "uuid",
           "email": "user@example.com",
           "role": "Customer|Staff|Admin",
           "state": "Active|Blocked",
           "created_at": "2024-01-15T10:30:00Z"
         }
       },
       "metadata": {
         "source_service": "user-service",
         "correlation_id": "uuid"
       }
     }
     ```

2. **`user.updated`** - Published when user information is updated
   - **Trigger**: After successful user update (if update endpoint is implemented)
   - **Event Data**: Include old and new values for audit trail

3. **`user.blocked`** - Published when a user is blocked
   - **Trigger**: After successful user block in `POST /v1/users/{userId}/block`
   - **Event Data**: Include user ID, blocked by (admin user ID), timestamp

4. **`user.unblocked`** - Published when a user is unblocked
   - **Trigger**: After successful user unblock in `POST /v1/users/{userId}/unblock`
   - **Event Data**: Include user ID, unblocked by (admin user ID), timestamp

**Implementation Requirements**:
- **Kafka Producer Setup**: Add Kafka producer to User Service
  - Use existing Kafka infrastructure (bootstrap servers: `localhost:9092`)
  - Create producer for `users.events` topic
  - Handle producer errors and retries
  - Add `rdkafka` dependency to Cargo.toml
  - Configure producer with appropriate settings (acks, retries, idempotence)
- **Event Schema**: Standardized event schema matching Transaction Service pattern
  - `event_id`: UUID for idempotency
  - `event_type`: String (user.created, user.updated, user.blocked, user.unblocked)
  - `timestamp`: ISO 8601 timestamp
  - `version`: String (e.g., "1.0")
  - `data`: Event-specific data (user object)
  - `metadata`: Source service, correlation ID
- **Publishing Points**:
  - After user creation in `create_user` function
  - After user block in block endpoint handler
  - After user unblock in unblock endpoint handler
  - After user update in update endpoint handler (if implemented)
- **Error Handling**:
  - Log errors but don't fail user operations if Kafka publish fails
  - Consider async publishing to avoid blocking user operations
  - Implement retry logic with exponential backoff
  - Add dead letter queue for failed events
- **Testing**:
  - Integration tests with Kafka testcontainers
  - Verify idempotency handling
  - Test error scenarios and retries
- **Current State**: User Service does not publish events
- **Impact**: Backoffice service cannot build read model without user events

#### Staff Role Implementation (If Using Staff Role in MVP)
- Define Staff role permissions and access levels
- Implement Staff role checks in User Service authorization logic
- Create permission matrix differentiating Staff from Admin capabilities
- **Current State**: `UserRole::Staff` exists in code but has no implementation
- **Impact**: Phase 1 authentication requirements are unclear without Staff role definition

### Transaction Service Prerequisites (For Phase 2)

#### Event Publishing (Already Implemented ✅)
**Topic**: `int.transaction-update.event` or `transactions.events`

**Current Implementation**:
- ✅ Already publishes events to Kafka
- ✅ Event types: `transaction.created`, `transaction.updated`, `transaction.deleted`
- ✅ Event schema exists

**Verification Required**:
1. **Event Schema Verification**: Verify event schema matches backoffice requirements
   - Ensure `event_id` is present for idempotency
   - Ensure `data` contains all fields needed for backoffice read model:
     - Transaction ID, user ID, amount, description, date
     - Category ID and name (if available)
     - Transaction type (EXPENSE/INCOME)
   - Ensure `metadata` contains source service and correlation ID
2. **Topic Name Standardization**: 
   - Current: `int.transaction-update.event`
   - Consider standardizing to `transactions.events` for consistency
   - Or document current topic name for backoffice service
3. **Event Completeness**: Ensure all transaction operations publish events
   - Transaction creation ✅
   - Transaction update ✅
   - Transaction deletion ✅
   - Category assignment (if applicable)
4. **Estimated Time**: 1 day (verification and documentation)

**Transaction Service Prerequisites Total**: 1 day

### Budget Service Prerequisites (For Phase 2)

#### Kafka Event Publishing (Critical for Standalone Service Approach)

**Topic**: `budgets.events`

**Required Events**:
1. **`budget.created`** - Published when a new budget is created
   - **Trigger**: After successful budget creation in `create_budget` function
   - **Event Data**:
     ```json
     {
       "event_id": "uuid",
       "event_type": "budget.created",
       "timestamp": "2024-01-15T10:30:00Z",
       "version": "1.0",
       "data": {
         "budget": {
           "id": "uuid",
           "user_id": "uuid",
           "name": "Monthly Essentials",
           "amount": "1500.00",
           "period_type": "MONTHLY|WEEKLY|YEARLY",
           "created_at": "2024-01-15T10:30:00Z"
         }
       },
       "metadata": {
         "source_service": "budget-service",
         "correlation_id": "uuid"
       }
     }
     ```

2. **`budget.updated`** - Published when a budget is updated
   - **Trigger**: After successful budget update in `update_budget` or `patch_budget` functions
   - **Event Data**: Include old and new values for audit trail
   - **Note**: Should be published for:
     - Budget amount changes
     - Budget status changes (DRAFT → ACTIVE → ARCHIVED)
     - Category allocation changes

3. **`budget.deleted`** - Published when a budget is deleted
   - **Trigger**: After successful budget deletion in `delete_budget` function
   - **Event Data**: Include budget ID, user ID, deleted at timestamp

**Implementation Requirements**:
- **Kafka Producer Setup**: Add Kafka producer to Budget Service
  - Use existing Kafka infrastructure (bootstrap servers: `localhost:9092`)
  - Create producer for `budgets.events` topic
  - Handle producer errors and retries
- **Event Schema**: Standardized event schema matching Transaction Service pattern
  - `event_id`: UUID for idempotency
  - `event_type`: String (budget.created, budget.updated, budget.deleted)
  - `timestamp`: ISO 8601 timestamp
  - `version`: String (e.g., "1.0")
  - `data`: Event-specific data (budget object)
  - `metadata`: Source service, correlation ID
- **Publishing Points**: 
  - After budget creation in `create_budget` function
  - After budget update in `update_budget` or `patch_budget` functions
  - After budget deletion in `delete_budget` function
  - After budget status changes (if applicable)
- **Error Handling**: 
  - Log errors but don't fail budget operations if Kafka publish fails
  - Consider async publishing to avoid blocking budget operations
- **Current State**: Budget Service does not publish events
- **Impact**: Backoffice service cannot build read model without budget events
- **Estimated Time**: 5 days
  - Kafka producer setup: 1 day
  - Event schema implementation: 1 day
  - Publishing logic in endpoints: 2 days
  - Testing and error handling: 1 day

**Budget Service Prerequisites Total**: 5 days (~1 week)

### Timeline Impact

**Prerequisite Work**: 22 days (~4.4 weeks) before Phase 1 start
- User Service block/unblock endpoints: 3 days
- User Service search/filtering: 3 days
- User Service event publishing: 5 days (critical for standalone service)
- Staff role definition (if needed): 3 days
- Transaction Service verification: 1 day
- Buffer time: 2 days

**Phase 2 Prerequisites**: 5 days (~1 week) before Phase 2 start
- Budget Service event publishing: 5 days

### Event Schema Standardization

**All services must use consistent event schema** for easier consumption by Backoffice Service:

**Standard Event Schema**:
```json
{
  "event_id": "uuid",              // UUID for idempotency (required)
  "event_type": "string",          // Event type (e.g., "user.created", "transaction.updated")
  "timestamp": "ISO8601",          // ISO 8601 timestamp (required)
  "version": "string",             // Schema version (e.g., "1.0")
  "data": {                        // Event-specific data (required)
    // Service-specific data structure
  },
  "metadata": {                    // Metadata (required)
    "source_service": "string",    // Service name (e.g., "user-service")
    "correlation_id": "uuid"       // Correlation ID for tracing
  }
}
```

**Implementation Requirements**:
- All events must include `event_id` for idempotency checks
- All events must include `timestamp` in ISO 8601 format
- All events must include `version` for schema evolution
- All events must include `data` with event-specific payload
- All events must include `metadata` with source service and correlation ID
- Event types must follow pattern: `{resource}.{action}` (e.g., `user.created`, `transaction.updated`)

**Topic Naming Convention**:
- User events: `users.events`
- Transaction events: `transactions.events` (or keep existing `int.transaction-update.event`)
- Budget events: `budgets.events`

**Event Schema Examples**:

**User Service Events** (`users.events`):
```json
// user.created
{
  "event_id": "uuid",
  "event_type": "user.created",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": "1.0",
  "data": {
    "user": {
      "id": "uuid",
      "email": "user@example.com",
      "role": "Customer|Staff|Admin",
      "state": "Active|Blocked",
      "created_at": "2024-01-15T10:30:00Z"
    }
  },
  "metadata": {
    "source_service": "user-service",
    "correlation_id": "uuid"
  }
}

// user.blocked
{
  "event_id": "uuid",
  "event_type": "user.blocked",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": "1.0",
  "data": {
    "user_id": "uuid",
    "blocked_by": "uuid",  // Admin user ID
    "reason": "string"     // Optional
  },
  "metadata": {
    "source_service": "user-service",
    "correlation_id": "uuid"
  }
}
```

**Transaction Service Events** (`int.transaction-update.event` or `transactions.events`):
```json
// transaction.created
{
  "event_id": "uuid",
  "event_type": "transaction.created",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": "1.0",
  "data": {
    "transaction": {
      "id": "uuid",
      "user_id": "uuid",
      "amount": "89.99",
      "description": "Whole Foods Market",
      "date": "2024-01-15T10:30:00Z",
      "transaction_type": "EXPENSE|INCOME",
      "category_id": "uuid",
      "category_name": "Groceries"
    }
  },
  "metadata": {
    "source_service": "transaction-service",
    "correlation_id": "uuid"
  }
}
```

**Budget Service Events** (`budgets.events`):
```json
// budget.created
{
  "event_id": "uuid",
  "event_type": "budget.created",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": "1.0",
  "data": {
    "budget": {
      "id": "uuid",
      "user_id": "uuid",
      "name": "Monthly Essentials",
      "amount": "1500.00",
      "period_type": "MONTHLY|WEEKLY|YEARLY",
      "status": "DRAFT|ACTIVE|ARCHIVED",
      "created_at": "2024-01-15T10:30:00Z"
    }
  },
  "metadata": {
    "source_service": "budget-service",
    "correlation_id": "uuid"
  }
}
```

### Summary of Service Changes Required

**User Service** (Critical for Phase 1):
1. ✅ Block/unblock endpoints: `POST /v1/users/{userId}/block`, `POST /v1/users/{userId}/unblock` (3 days)
2. ✅ User search/filtering: Enhance `GET /v1/users` with query parameters (3 days)
3. ✅ Kafka event publishing: Publish `user.created`, `user.updated`, `user.blocked`, `user.unblocked` to `users.events` (5 days)
4. ⚠️ Staff role implementation: Define permissions and authorization (3 days, if using Staff role)
5. ⚠️ User update endpoint: `PUT /v1/users/{userId}` (2 days, optional)

**Transaction Service** (For Phase 2):
1. ✅ Event schema verification: Verify existing events match requirements (1 day)
2. ⚠️ Topic name standardization: Consider renaming `int.transaction-update.event` to `transactions.events` (optional)

**Budget Service** (For Phase 2):
1. ✅ Kafka event publishing: Publish `budget.created`, `budget.updated`, `budget.deleted` to `budgets.events` (5 days)

**Total Prerequisite Work**: 22 days (~4.4 weeks)
- User Service: 16 days (3.2 weeks)
- Transaction Service: 1 day (0.1 weeks)
- Budget Service: 5 days (1 week, can be done in parallel with Phase 1)

**Recommendation**: 
- Complete all User Service prerequisites (including event publishing) before starting Phase 1
- Complete Budget Service event publishing before starting Phase 2 (can be done in parallel with Phase 1)
- Event publishing is **critical** for standalone service approach - backoffice service needs events to build read model
- All services should use consistent event schema for easier consumption
- Consider implementing event publishing asynchronously to avoid blocking user operations

## Implementation Plan

### Resource Requirements
- **Team Size**: 2-3 full-stack developers (Rust + React experience)
- **Timeline**: 16 weeks total (4 phases of 4 weeks each) + 4-5 weeks prerequisite work
  - **Prerequisite Work**: 4-5 weeks before Phase 1 (User Service, Transaction Service, Budget Service changes)
    - User Service: 3.2 weeks (endpoints + event publishing)
    - Transaction Service: 0.1 weeks (verification)
    - Budget Service: 1 week (event publishing - can be done in parallel with Phase 1)
  - **Phase 1**: 4 weeks (MVP - Core Admin Operations)
  - **Phase 2**: 4 weeks (Transaction Support & Monitoring)
  - **Phase 3**: 4 weeks (Advanced Analytics & System Management)
  - **Phase 4**: 4 weeks (Automation & Compliance)
- **Dependencies**: 
  - **Prerequisites**: All service changes must be implemented first (see Prerequisites section)
    - User Service: Block/unblock endpoints, search/filtering, Kafka event publishing (critical)
    - Transaction Service: Event schema verification
    - Budget Service: Kafka event publishing (can be done in parallel with Phase 1)
  - Existing User, Transaction, Budget services must be operational
  - Staff role must be defined if using in MVP
  - Kafka infrastructure must be operational
- **Infrastructure**: Kubernetes cluster, PostgreSQL instance, Kafka access

### Phase 1: MVP - Core Admin Operations (Weeks 1-4)
**Deliverables**:
- New Backoffice Service (Rust/Axum) with Kafka consumer and read model
- WebSocket server for real-time dashboard statistics
- Admin frontend application (React/TypeScript)
- User search, user details, and account management (block/unblock)
- User transaction viewing (per-user transaction list)
- Basic dashboard with user statistics and active session tracking
- Deployment to development environment

**Technical Tasks**:
- Create new Backoffice Service Rust project with Axum
- Implement Kafka consumer for user events (`users.events` topic)
- Build denormalized read model (user_summaries table)
- Implement event handlers for user events (user.created, user.updated, user.blocked, user.unblocked)
- Create PostgreSQL schema and migrations for read model and admin data
- Implement session tracking (store active sessions in read model or query User Service)
- Implement session validation via User Service `/v1/auth/session`
- Implement role-based authorization (Admin/Staff) middleware
- Implement JWT service token generation for calling backend services
- Implement admin read APIs:
  - User search (from read model)
  - User details (from read model)
  - User transactions (direct API call to Transaction Service)
  - Dashboard statistics (active users, active sessions, total users, recent registrations)
- Implement admin write APIs (block/unblock users via User Service)
- Implement WebSocket server for real-time dashboard statistics updates
- Build React admin frontend with Material-UI
- Implement user search interface (from read model)
- Implement user details page with transaction list
- Implement account blocking/unblocking UI
- Implement dashboard with real-time statistics:
  - Total users count
  - Active users (last 30 days)
  - Users with active sessions (real-time)
  - Recent registrations (last 7/30 days)
  - User growth trends
- Set up CI/CD pipeline for Backoffice Service
- Basic integration tests (Kafka consumer, read model updates, API endpoints, WebSocket, statistics aggregation)

### Phase 2: Transaction Investigation & Analytics (Weeks 5-8)
**Deliverables**:
- Transaction read model in Backoffice Service (from transaction events)
- Global transaction search and investigation tools
- Transaction analytics and reporting
- Enhanced dashboard with transaction metrics
- Transaction timeline and user activity views
- Staging environment deployment

**Technical Tasks**:
- Extend Kafka consumer for transaction events (`transactions.events` topic)
- Build denormalized read model (transaction_summaries table)
- Implement event handlers for transaction events (transaction.created, transaction.updated, transaction.deleted)
- Implement transaction aggregation APIs (from read model)
- Build global transaction search interface (from read model)
  - Search across all transactions (not just per-user)
  - Advanced filtering (user, amount, date range, category, description)
  - Sorting and pagination
- Create transaction detail views (from read model)
- Build transaction analytics engine:
  - Transaction volume trends
  - Revenue/expense breakdowns
  - Category distribution
  - Transaction patterns
- Enhance dashboard with transaction metrics
- Implement audit logging (from consumed events)
- Performance optimization (read model queries, indexing)
- End-to-end testing

### Phase 3: Advanced Analytics & System Management (Weeks 9-12)
**Deliverables**:
- Budget read model in Backoffice Service (from budget events)
- Advanced analytics and reporting (from aggregated views)
- System configuration management
- Comprehensive monitoring dashboard
- Custom report generation
- Production environment setup

**Technical Tasks**:
- Extend Kafka consumer for budget events (`budgets.events` topic)
- Build denormalized read model (budget_summaries table)
- Implement event handlers for budget events (budget.created, budget.updated, budget.deleted)
- Build aggregated views for analytics (daily_transaction_metrics, etc.)
- Build analytics aggregation engine (from aggregated views)
- Implement custom reporting with export (from read model)
- Create system configuration interfaces
- Add Kafka monitoring integration
- Implement granular permissions
- Load testing and optimization (read model queries)
- Security hardening

### Phase 4: Automation, Compliance & Advanced Features (Weeks 13-16)
**Deliverables**:
- Compliance reporting features (GDPR, CCPA)
- Real-time alerting system
- Staff management functionality
- Data management and integrity checks
- Feature flags system
- Complete documentation and training
- Production deployment

**Technical Tasks**:
- Build compliance reporting engine:
  - Regulatory report generation
  - Data privacy compliance monitoring
  - User data export for compliance requests
- Implement immutable audit trails with tamper detection
- Build real-time alerting system:
  - Alert configuration and routing
  - Real-time WebSocket updates for alerts
  - Alert history and resolution tracking
- Integrate external notification systems (Slack/email)
- Implement staff management:
  - Create and manage administrative accounts
  - Permission management
- Build data management tools:
  - Data integrity checks
  - Backup monitoring
- Implement feature flags system
- Complete security audit
- Performance monitoring setup
- User training and documentation

## Risk Assessment

### Technical Risks
- **Performance Impact**: Additional load on existing services
  - *Mitigation*: Implement caching and optimize database queries
- **Service Dependencies**: Dependency on all existing microservices
  - *Mitigation*: Implement circuit breakers and graceful degradation
- **Data Consistency**: Maintaining consistency across services
  - *Mitigation*: Use event-driven architecture with eventual consistency

### Security Risks
- **Privileged Access**: Administrative access to sensitive data
  - *Mitigation*: Implement principle of least privilege and comprehensive audit logging
- **Insider Threats**: Potential abuse of administrative privileges
  - *Mitigation*: Regular access reviews and behavioral monitoring
- **Data Breaches**: Centralized access to customer data
  - *Mitigation*: Data encryption, access controls, and monitoring

### Operational Risks
- **Staff Training**: Learning curve for administrative staff
  - *Mitigation*: Comprehensive training program and documentation
- **Operational Dependencies**: Single point of failure for administration
  - *Mitigation*: High availability deployment and backup procedures
- **Compliance**: Meeting regulatory requirements
  - *Mitigation*: Built-in compliance features and regular audits

## Success Criteria

### Functional Success
- ✅ All user management operations can be performed through the interface
- ✅ Real-time system monitoring with 99.9% accuracy
- ✅ Transaction investigation capabilities with sub-second response times
- ✅ Comprehensive audit logging for all administrative actions

### Performance Success
- ✅ Dashboard loads in under 2 seconds
- ✅ Search operations complete in under 1 second
- ✅ System can handle 1000+ concurrent administrative users
- ✅ 99.9% uptime for Backoffice Service
- ✅ Real-time updates with less than 100ms latency

### Business Success
- ✅ 50% reduction in customer support response time
- ✅ 30% decrease in operational overhead
- ✅ 100% compliance with audit requirements
- ✅ Zero security incidents related to administrative access
- ✅ 95% user satisfaction score from administrative staff

## Future Considerations

### Scalability
- **Horizontal Scaling**: Design for multi-instance deployment
- **Database Scaling**: Consider read replicas for reporting queries
- **Microservice Evolution**: Plan for additional microservices integration
- **Global Deployment**: Multi-region deployment capabilities

### Feature Enhancements
- **Predictive Analytics**: Predictive insights for business planning
- **Mobile App**: Mobile administrative interface for on-the-go management
- **Third-party Integrations**: CRM, helpdesk, and BI tool integrations

### Technology Evolution
- **API Versioning**: Plan for API evolution and backward compatibility
- **Service Mesh**: Consider service mesh adoption for advanced traffic management
- **Observability**: Enhanced observability with distributed tracing
- **Automation**: Increased automation for routine administrative tasks

## Testing Strategy

### Unit Testing
- **Backend**: Rust unit tests for domain logic and API handlers
- **Frontend**: React component testing with Jest and React Testing Library
- **Coverage**: Minimum 80% code coverage for critical paths

### Integration Testing
- **API Testing**: Comprehensive API testing with real database
- **Cross-Service**: Test integration with User, Transaction, and Budget services
- **Authentication**: JWT validation and role-based access testing

### End-to-End Testing
- **User Workflows**: Complete administrative workflows from login to task completion
- **Browser Testing**: Multi-browser compatibility testing
- **Performance**: Load testing for concurrent administrative users

### Security Testing
- **Penetration Testing**: Third-party security assessment
- **Vulnerability Scanning**: Automated security scanning
- **Access Control**: Thorough permission and authorization testing

## Monitoring & Observability

### Application Monitoring
- **Metrics**: Prometheus metrics for API performance and business KPIs
- **Logging**: Structured logging with correlation IDs
- **Tracing**: Distributed tracing across service calls
- **Health Checks**: Kubernetes-ready health and readiness probes

### Business Monitoring
- **Administrative Actions**: Track user management operations
- **System Usage**: Dashboard views and feature utilization
- **Performance**: Response times and error rates

### Alerting
- **Critical Alerts**: System failures, authentication issues
- **Performance Alerts**: High response times, error rate spikes
- **Business Alerts**: Unusual administrative activity patterns
- **Integration**: Slack/email notifications with escalation

## Deployment Strategy

### Infrastructure
- **Kubernetes**: Deploy new Backoffice Service alongside existing services
- **Ingress**: Separate subdomain (admin.moneyplanner.local) routing to BFF admin routes
- **Database**: Dedicated PostgreSQL schema in existing database instance (separate from BFF)
- **Secrets**: Kubernetes secrets for Kafka consumer credentials and service credentials
- **Kafka**: Access to Kafka topics (`users.events`, `transactions.events`, `budgets.events`)

### Environments
- **Development**: Local development with docker-compose
- **Staging**: Full staging environment for integration testing
- **Production**: High-availability production deployment with monitoring

### CI/CD Pipeline
- **Build**: Docker image building with multi-stage builds for Backoffice Service
- **Testing**: Automated testing pipeline with quality gates for Backoffice Service
- **Security**: Container scanning and dependency vulnerability checks
- **Deployment**: GitOps deployment with Helm charts for Backoffice Service
- **Note**: Backoffice Service has its own CI/CD pipeline (independent from BFF)

## Appendices

### Appendix A: API Reference
Complete OpenAPI specification for all backoffice endpoints

### Appendix B: Database Schema
Complete database schema with relationships and constraints

### Appendix C: Security Matrix
Detailed permission matrix for all administrative roles

### Appendix D: Deployment Guide
Step-by-step deployment instructions for all environments

---

*This PRD serves as the foundational document for the Money Planner Backoffice Application development. It should be reviewed and updated regularly as requirements evolve and implementation progresses.*