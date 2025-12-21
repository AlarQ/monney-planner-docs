# Money Planner System Overview

## Architecture Overview

Money Planner is a microservices-based financial management application built with a modern tech stack. The system follows a clean architecture pattern with clear separation of concerns.

### System Components

#### Frontend (MoneyPlannerFE)
- **Technology**: React 18 with TypeScript
- **UI Framework**: Material-UI (MUI) v5.15.10 with Emotion for styling
- **State Management**: React Context API for auth and notifications
- **Routing**: React Router DOM v6
- **HTTP Client**: Axios for API communication
- **Form Handling**: Formik with Yup validation
- **Charts**: Recharts for data visualization
- **Testing**: Playwright for E2E testing
- **Development**: Runs on port 3000 in development mode

#### Backend for Frontend (BFF) - money-planner-bff
- **Technology**: Rust with Axum web framework
- **Purpose**: API gateway that aggregates data from microservices
- **Authentication**: Session-based auth with middleware
- **CORS**: Configured for localhost:3000
- **Port**: 8082 (local development)
- **Key Features**:
  - Session validation middleware
  - Error tracing middleware
  - Cookie management
  - Service orchestration

#### User Service - user-service
- **Technology**: Rust with Axum, SQLx for database operations
- **Database**: PostgreSQL with migrations
- **Authentication**: Argon2 password hashing, JWT tokens
- **Features**:
  - User registration and login
  - Session management
  - Password validation (8-100 chars, uppercase, lowercase, number, special)
  - User roles (Customer, Staff, Admin)
  - User states (Active, Blocked)
- **Port**: 8080 (local development)

#### Transaction Service - transaction-service
- **Technology**: Rust with Axum, SQLx, AI integration
- **Database**: PostgreSQL with connection pooling
- **Architecture**: Domain-Driven Design (DDD) with clean architecture principles
- **AI Integration**: OpenAI API for transaction categorization via ai_utils crate
- **CSV Processing**: Support for multiple bank formats (ING Bank, generic) with registry system
- **Event-Driven Architecture**: Kafka event publishing for transaction lifecycle events
- **Features**:
  - Transaction CRUD operations
  - Category management
  - AI-powered transaction categorization
  - CSV import with format detection and validation
  - Frequency tracking (Once, Monthly)
  - Payment day scheduling
  - Automatic transaction type determination (positive/negative amounts)
  - Kafka event publishing for budget service integration
- **Port**: 8081 (local development)

#### Budget Service - budget-service
- **Technology**: Rust with Axum, SQLx, Kafka integration
- **Database**: PostgreSQL with connection pooling
- **Architecture**: Domain-Driven Design (DDD) with clean architecture principles
- **Event-Driven**: Kafka consumer for real-time transaction updates
- **Features**:
  - **Advanced Budget Management**: Multi-period budgets (Monthly, Weekly, Yearly) with category allocations
  - **Budget Tracking**: Real-time progress monitoring with spending analysis
  - **User Settings**: Configurable budget warning thresholds
  - **Category Allocations**: Flexible budget-to-category mapping with optional specific amounts
  - **Budget Summaries**: Comprehensive spending analysis and progress tracking
  - **Event Processing**: Real-time budget updates via Kafka transaction events
  - **OpenAPI Documentation**: Interactive Swagger UI for API exploration
- **Port**: 8083 (local development)

#### Database
- **Technology**: PostgreSQL
- **Deployment**: Docker Compose with separate instances per service
- **User Service DB**: Port 5432
- **Transaction Service DB**: Port 5433
- **Budget Service DB**: Port 5434
- **Migrations**: SQLx-based migrations in each service

### Data Models

#### User Domain
```rust
struct User {
    id: UserId,
    password: UserPassword, // Argon2 hashed
    email: String,
    created_at: DateTime<Utc>,
    state: UserState, // Active, Blocked
    role: UserRole,   // Customer, Staff, Admin
}
```

#### Transaction Domain
```rust
struct Transaction {
    id: TransactionId,
    user_id: UserId,
    description: Description,
    amount: Amount, // Decimal with validation
    date: DateTime<Utc>,
    transaction_type: TransactionType, // Auto-determined: positive = Income, negative = Expense
    transaction_category_id: TransactionCategoryId,
    frequency_type: TransactionFrequencyType, // Once, Monthly
    times_left: Option<i32>,
    payment_day: Option<i32>, // 1-31
}

struct Budget {
    id: BudgetId,
    user_id: UserId,
    name: String,
    description: Option<String>,
    amount: Amount, // Total budget amount
    period_type: BudgetPeriod, // Monthly, Weekly, Yearly
    period_start_date: DateTime<Utc>,
    period_end_date: DateTime<Utc>,
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
}

struct CategoryAllocation {
    category_id: TransactionCategoryId,
    category_name: String,
    allocated_amount: Option<Amount>, // Optional specific allocation
}

struct BudgetWithCategories {
    budget: Budget,
    categories: Vec<CategoryAllocation>,
}

struct BudgetSummary {
    id: BudgetId,
    name: String,
    total_budget: Amount,
    total_spent: Amount,
    remaining: Amount,
    period_type: BudgetPeriod,
    categories: Vec<CategoryBudgetSummary>,
}

struct UserSettings {
    user_id: UserId,
    budget_warning_threshold: Decimal, // Percentage (default: 80.0)
}
```

#### Frontend Types
```typescript
interface Spending {
    id: string;
    description: string;
    amount: number;
    date: string;
    category: string;
    frequency: SpendingFrequency;
}

interface SpendingFrequency {
    type: SpendingFrequencyType; // Once, Monthly
    timesLeft?: number;
    paymentDay?: number;
}
```

### Authentication Flow

1. **Registration**: User creates account with email/password
2. **Login**: User authenticates, receives JWT token
3. **Token Storage**: JWT token stored in localStorage
4. **Request Authorization**: JWT token sent as Bearer token in Authorization header
5. **Token Validation**: BFF validates JWT token with User Service
6. **User ID Extraction**: User ID extracted from JWT payload for all subsequent requests

### Event-Driven Architecture

The system uses Kafka for real-time communication between services:

#### Transaction Events
- **Publisher**: Transaction Service publishes events when transactions are created, updated, or deleted
- **Consumer**: Budget Service consumes transaction events to update budget calculations in real-time
- **Topic**: `int.transaction-update.event`
- **Event Types**: `transaction.created`, `transaction.updated`, `transaction.deleted`

#### Event Flow
1. **Transaction Creation**: User creates transaction via BFF → Transaction Service
2. **Event Publishing**: Transaction Service publishes transaction event to Kafka
3. **Event Consumption**: Budget Service consumes event and updates relevant budget calculations
4. **Real-time Updates**: Budget summaries and spending analysis updated automatically

#### Benefits
- **Decoupled Services**: Services can evolve independently
- **Real-time Updates**: Budget calculations update immediately when transactions change
- **Scalability**: Event-driven architecture supports horizontal scaling
- **Reliability**: Kafka provides message durability and delivery guarantees

### Key Features

#### Financial Management
- Track income and expenses
- Categorize transactions manually or via AI
- Set up recurring transactions (monthly)
- Payment day scheduling
- Transaction frequency tracking
- **Advanced Budget Management**: Multi-period budgets (Monthly, Weekly, Yearly) with category allocations and spending analysis
- **Budget Tracking**: Real-time progress monitoring with visual indicators and summaries
- **User Settings**: Customizable budget warning thresholds
- **Event-Driven Budget Updates**: Real-time budget calculations via Kafka transaction events

#### AI-Powered Categorization
- Automatic transaction categorization using OpenAI
- Confidence scoring for suggestions
- New category suggestions
- Merchant and description analysis

#### Data Import
- CSV import from various bank formats
- Automatic format detection
- Bulk transaction processing
- Data validation and sanitization

#### User Experience
- Material-UI based responsive design
- Real-time notifications via snackbars
- Form validation with Yup
- Protected routes with authentication
- User profile management

### Development Environment

#### Local Development Setup
- **Frontend**: `cd MoneyPlannerFE && ./run.sh` (port 3000)
- **BFF**: `cd money-planner-bff && ./run.sh` (port 8082)
- **User Service**: `cd user-service && ./run.sh` (port 8080, starts DB on 5432)
- **Transaction Service**: `cd transaction-service && ./run.sh` (port 8081, DB on 5433)
- **Budget Service**: `cd budget-service && ./run.sh` (port 8083, DB on 5434)
- **Database**: Three separate PostgreSQL instances via Docker Compose

#### Docker Compose Setup
- Each service has its own docker-compose.yaml for database
- User service DB: PostgreSQL on port 5432
- Transaction service DB: PostgreSQL on port 5433
- Budget service DB: PostgreSQL on port 5434
- Services run individually with run.sh scripts

#### Port Mapping
| Service | Port | Purpose |
|---------|------|----------|
| Frontend | 3000 | React Development Server |
| BFF | 8082 | API Gateway |
| User Service | 8080 | User management |
| Transaction Service | 8081 | Transaction management |
| Budget Service | 8083 | Budget management |
| User Service DB | 5432 | PostgreSQL (Docker) |
| Transaction Service DB | 5433 | PostgreSQL (Docker) |
| Budget Service DB | 5434 | PostgreSQL (Docker) |

### API Structure

#### BFF Endpoints
- `/v1/users/*` - User management (login, registration)
- `/v1/transactions/*` - Transaction operations
- `/v1/categories/*` - Category management
- `/v1/budgets/*` - Advanced budget management operations (create, update, summary, allocations)
- `/v1/user-settings` - User preferences and settings

#### Budget Service Endpoints
- `POST /v1/users/{user_id}/budgets` - Create a new budget
- `GET /v1/users/{user_id}/budgets` - List all budgets for user
- `GET /v1/users/{user_id}/budgets/{id}` - Get budget details
- `DELETE /v1/users/{user_id}/budgets/{id}` - Delete budget
- `PUT /v1/users/{user_id}/budgets/{id}/categories` - Update category allocations
- `GET /v1/users/{user_id}/budgets/summary` - Get budget summary with spending analysis

> **Note:** The BFF (API Gateway) is served from http://localhost:8082 in local development. The frontend run.sh script incorrectly points to port 8080 - this should be updated to 8082.

#### Authentication
- JWT-based with localStorage storage
- Public routes: `/v1/users/login`, `/v1/users/`
- Protected routes require valid JWT token in Authorization header

### Error Handling

#### Backend
- Structured error types with `thiserror`
- HTTP status codes mapped to domain errors
- Comprehensive logging with tracing
- Validation errors with detailed messages

#### Frontend
- Axios interceptors for error handling
- Snackbar notifications for user feedback
- Form validation with Yup schemas
- Graceful degradation for network issues

### Security Features

- **Password Security**: Argon2 hashing with salt
- **JWT Management**: Secure JWT tokens with localStorage
- **Input Validation**: Comprehensive validation at domain level with SQL injection and XSS pattern detection
- **CORS**: Configured for development and production
- **SQL Injection Protection**: SQLx with parameterized queries

### Monitoring and Observability

- **Logging**: Structured logging with tracing
- **Error Tracking**: Comprehensive error types and messages
- **Database Monitoring**: Connection pooling metrics
- **Performance**: Request tracing and timing
- **Event Monitoring**: Kafka event processing and budget service integration

### Testing Strategy

- **Unit Tests**: Domain logic testing (including advanced budget calculations and validations)
- **Integration Tests**: API endpoint testing (transactions, categories, advanced budgets, user settings)
- **Database Tests**: SQLx-based test setup with comprehensive test data
- **Event Tests**: Kafka integration testing for budget service
- **Frontend Tests**: React Testing Library
- **E2E Tests**: Playwright for end-to-end testing (including advanced budget workflows and multi-period scenarios)

This system provides a robust foundation for personal financial management with modern architecture patterns, AI integration, and comprehensive user experience features.

## MVP Feature Planning

### Identified Potential Features

During system review, several feature categories were identified for potential addition:

#### AI & Automation
- Smart Budgeting: AI-analyzed spending patterns for personalized budgets
- Expense Forecasting: Predict future expenses based on historical data
- Anomaly Detection: Flag unusual spending patterns
- Receipt OCR: Scan receipts for automatic transaction extraction
- Voice Commands: Voice-based expense logging

#### Advanced Analytics
- Net Worth Tracking: Assets vs liabilities over time
- Investment Portfolio Integration: Brokerage connections
- Tax Optimization: Tax-deductible expense tracking
- Spending Insights: Comparative spending analysis
- Goal Progress Tracking: Visual progress for savings goals

#### Social & Collaboration
- Shared Household Budgets: Multi-user expense management
- Expense Splitting: Bill splitting with roommates/family
- Financial Goals Sharing: Accountability partnerships
- Community Features: Anonymous spending comparisons

#### Advanced Planning
- Debt Management: Snowball/avalanche payoff strategies
- Emergency Fund Calculator: Dynamic fund recommendations
- Retirement Planning: 401k/IRA tracking with projections
- Major Purchase Planning: Timeline-based savings goals

#### Integration & Connectivity
- Bank Account Sync: Real-time transaction import
- Credit Card Integration: Balance and payment tracking
- Bill Reminders: Due date notifications with auto-payment
- Subscription Tracking: Recurring subscription monitoring

#### Mobile-First Features
- Offline Mode: Expense tracking without internet
- Widgets: Quick expense logging from home screen
- Biometric Authentication: Face ID/fingerprint access
- Push Notifications: Budget alerts and reminders

#### Gamification
- Achievement System: Badges for financial milestones
- Challenges: No-spend weekends, savings challenges
- Streaks: Consecutive budget compliance tracking
- Leaderboards: Optional friend comparisons

#### Advanced Reporting
- Custom Date Ranges: Flexible reporting periods
- Export Options: PDF reports, Excel integration
- Tax Reports: Year-end summaries for tax prep
- Business Expense Tracking: Personal/business separation

#### Security & Privacy
- Two-Factor Authentication: Enhanced security
- Data Encryption: End-to-end encryption
- Privacy Controls: Granular data sharing
- Audit Logs: Account change tracking

#### Smart Notifications
- Contextual Alerts: Budget progress notifications
- Market Alerts: Investment opportunities/risks
- Life Event Planning: Financial guidance for major events

### MVP Feature Selection

For immediate MVP deployment, the following high-impact, low-effort features were prioritized:

#### Must-Have MVP Features

1. **Bank Account Sync** (High Impact)
   - Plaid API integration for real-time transaction import
   - Auto-categorization using existing AI system
   - Eliminates manual data entry

2. **Basic Budgeting** (Core Feature)
   - Monthly budget limits per category
   - Progress bars showing spending vs budget
   - Simple alerts when approaching limits
   - Leverages existing category system

3. **Bill Reminders** (User Retention)
   - Due date notifications for recurring transactions
   - Calendar view of upcoming payments
   - Uses existing recurring transaction system

4. **Export/Import** (Data Portability)
   - CSV export of transactions (complements existing import)
   - PDF monthly reports
   - Essential for user trust and data ownership

5. **Mobile Responsive** (Critical)
   - Ensure existing UI works perfectly on mobile
   - Touch-friendly expense logging
   - Responsive charts and tables

#### Implementation Timeline

- **Week 1-2**: Mobile responsive + basic budgeting
- **Week 3-4**: Bank sync (Plaid integration)
- **Week 5-6**: Bill reminders + export features

#### Features to Skip for MVP

- Social features (complex implementation)
- Advanced analytics (nice-to-have)
- Voice commands (gimmicky)
- Gamification (distraction from core value)

These 5 MVP features provide a solid, marketable foundation that solves real user pain points without over-engineering, with bank sync being a key differentiator from basic spreadsheet solutions.
