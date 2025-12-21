# Product Requirements Document: Generic Categories Setup

## 1. Overview

### Feature Summary
Add a "Setup Default Categories" feature that allows new users to preview and selectively add from a comprehensive set of generic transaction categories, eliminating the friction of manually creating categories from scratch while giving users control over their selection.

### Problem Statement
New users face a significant onboarding barrier when they have no transaction categories. The current empty state requires users to manually create each category one by one, which is time-consuming and may lead to abandonment. Users often don't know what categories they need or forget important ones.

### Solution
Provide a preview interface where users can see all available default categories and selectively choose which ones to add to their account, giving them control while reducing setup friction.

## 2. User Stories

### Primary User Story
**As a new user**, I want to preview and selectively choose from common transaction categories so that I can quickly set up only the categories I need without being overwhelmed by unnecessary options.

### Secondary User Stories
- **As a new user**, I want to see all available default categories with checkboxes so I can choose exactly which ones I want
- **As a user**, I want to be able to select/deselect all categories at once for convenience
- **As a user**, I want to see categories organized by type (Income/Expense) for better understanding
- **As a returning user**, I want the setup interface to only appear when I have no categories, so it doesn't clutter my interface

## 3. Functional Requirements

### 3.1 Core Functionality

#### FR-1: Default Categories Preview Display
- **Requirement**: The default categories preview interface must only appear when the user has zero categories
- **Location**: Transaction Categories page, replacing the current empty state
- **Trigger**: When `categories.length === 0`

#### FR-2: Default Categories Set
The system must offer the following categories for selection:

**Income Categories:**
- Salary
- Freelance
- Investment Returns
- Rental Income
- Other Income

**Expense Categories:**
- Housing (Rent/Mortgage)
- Utilities
- Groceries
- Transportation
- Healthcare
- Entertainment
- Dining Out
- Shopping
- Insurance
- Education
- Savings
- Debt Payment
- Other Expenses

#### FR-3: Category Selection Interface
- **Requirement**: Each category must have a checkbox for individual selection
- **Default State**: All categories should be pre-selected by default
- **Bulk Actions**: Provide "Select All" and "Deselect All" buttons
- **Organization**: Categories must be grouped by type (Income/Expense) with clear headers
- **Validation**: At least one category must be selected before proceeding

#### FR-4: Bulk Category Creation
- **Requirement**: Only selected categories must be created in a single API call
- **Performance**: Creation should complete within 2 seconds
- **Atomicity**: Either all selected categories are created successfully, or none are created (rollback on failure)

#### FR-5: User Feedback
- **Success**: Show success message with count of categories created
- **Error**: Show clear error message if creation fails
- **Loading**: Show loading state during creation process
- **Selection Count**: Show count of selected categories in real-time

### 3.2 User Experience Requirements

#### UX-1: Category Selection Interface
- **Requirement**: Replace empty state with category selection interface
- **Layout**: Two-column layout with Income and Expense sections
- **Checkboxes**: Material-UI Checkbox components for each category
- **Headers**: Clear section headers for "Income Categories" and "Expense Categories"
- **Bulk Controls**: "Select All" and "Deselect All" buttons at the top

#### UX-2: Selection Feedback
- **Requirement**: Show real-time count of selected categories
- **Location**: Display count near the "Create Selected Categories" button
- **Format**: "X of 18 categories selected"
- **Validation**: Disable create button when no categories are selected

#### UX-3: Post-Creation Experience
- **Requirement**: After successful creation, show categories list with newly created categories
- **State**: Categories list should show all newly created categories
- **Actions**: User can immediately edit, delete, or add more categories
- **Success Message**: Show confirmation with count of categories created

#### UX-4: Button State Management
- **Requirement**: Create button should be disabled during creation process
- **Visual**: Show loading spinner and "Creating..." text
- **Prevention**: Prevent multiple simultaneous creation attempts
- **Validation**: Button disabled when no categories selected

## 4. Technical Requirements

### 4.1 Backend Requirements

#### TR-1: New API Endpoint
- **Endpoint**: `POST /v1/users/{user_id}/categories/default`
- **Authentication**: JWT token required
- **Authorization**: User can only create default categories for themselves
- **Request Body**: Array of category objects with name and type
- **Response**: List of created categories with IDs
- **Service**: Transaction Service (primary owner of categories)

#### TR-2: Bulk Creation Implementation
- **Database**: Use batch insert for performance in transaction_categories table
- **Transaction**: Wrap in database transaction for atomicity
- **Validation**: Validate all category names and types before insertion
- **Error Handling**: Return specific error codes for different failure scenarios
- **Service Boundary**: Transaction Service owns categories, Budget Service references them by ID

#### TR-3: Category Name Validation
- **Length**: 1-64 characters (existing validation)
- **Uniqueness**: Check for existing categories with same names per user
- **Sanitization**: Trim whitespace and validate format
- **Type Validation**: Ensure category_type is either "Income" or "Expense"

### 4.2 Frontend Requirements

#### TR-4: New API Integration
- **Function**: `createDefaultCategories(userId: string, categories: {name: string, type: string}[])`
- **Error Handling**: Handle network errors and validation errors
- **Type Safety**: TypeScript interfaces for request/response
- **Fix Existing**: Update `deleteCategory` to use correct endpoint with user context

#### TR-5: State Management
- **Hook**: Extend `useCategoryManagement` hook
- **State**: Add `isCreatingDefault` loading state and `selectedCategories` array
- **Cache**: Invalidate and refetch categories after creation
- **Types**: Add CategoryType enum and update Category interface

#### TR-6: Component Updates
- **File**: `TransactionCategories.tsx`
- **Changes**: Replace empty state with category selection interface
- **Components**: Create category selection component with checkboxes
- **State**: Manage selected categories state and validation
- **API Fix**: Fix `deleteCategory` endpoint inconsistency in `transactionsApi.ts`

### 4.3 Database Requirements

#### TR-7: Performance Optimization
- **Index**: Ensure `user_id` index exists on `transaction_categories` table
- **Batch Insert**: Use single INSERT statement with multiple VALUES
- **Connection**: Use connection pooling for concurrent requests
- **Schema Update**: Add `category_type` column to existing table

## 5. API Specification

### 5.1 Request Format

```typescript
// POST /v1/users/{user_id}/categories/default
interface CreateDefaultCategoriesRequest {
  categories: {
    name: string;
    type: 'Income' | 'Expense';
  }[];
}
```

### 5.2 Response Format

```typescript
interface CreateDefaultCategoriesResponse {
  categories: Category[];
  message: string;
}

interface Category {
  id: string;
  categoryId: string;
  name: string;
  type: 'Income' | 'Expense';
}
```

### 5.3 Error Responses

```typescript
interface ApiErrorResponse {
  code: string;
  description: string;
}

// Possible error codes:
// - "CATEGORIES_ALREADY_EXIST" - User already has categories
// - "VALIDATION_ERROR" - Category name/type validation failed
// - "DATABASE_ERROR" - Database operation failed
// - "UNAUTHORIZED" - Invalid or missing JWT token
// - "INVALID_CATEGORY_TYPE" - Category type must be Income or Expense
```

## 6. User Interface Design

### 6.1 Empty State Enhancement

**Current State:**
```
┌─────────────────────────────────────┐
│ No categories yet                   │
│ Create your first transaction       │
│ category to get started             │
└─────────────────────────────────────┘
```

**New State:**
```
┌─────────────────────────────────────┐
│ Setup Your Categories               │
│ Choose from common categories below │
│                                     │
│ [Select All] [Deselect All]         │
│                                     │
│ Income Categories:                  │
│ ☑ Salary                           │
│ ☑ Freelance                        │
│ ☑ Investment Returns               │
│ ☑ Rental Income                    │
│ ☑ Other Income                     │
│                                     │
│ Expense Categories:                 │
│ ☑ Housing (Rent/Mortgage)          │
│ ☑ Utilities                        │
│ ☑ Groceries                        │
│ ☑ Transportation                   │
│ ☑ Healthcare                       │
│ ☑ Entertainment                    │
│ ☑ Dining Out                       │
│ ☑ Shopping                         │
│ ☑ Insurance                        │
│ ☑ Education                        │
│ ☑ Savings                          │
│ ☑ Debt Payment                     │
│ ☑ Other Expenses                   │
│                                     │
│ 18 of 18 categories selected        │
│ [Create Selected Categories]        │
│ [Add Custom Category]               │
└─────────────────────────────────────┘
```

## 7. Implementation Plan

### 7.1 Phase 1: Backend Implementation (2-3 days) ✅ **COMPLETED**

#### Day 1: API Endpoint ✅ **COMPLETED**
- [x] Add `category_type` field to TransactionCategory model and database schema
- [x] Create new endpoint in transaction-service that accepts array of category objects
- [ ] Add route to BFF service
- [x] Implement bulk category creation logic with selected categories only
- [x] Add comprehensive error handling

#### Day 2: Testing & Validation ✅ **COMPLETED**
- [x] Write unit tests for bulk creation with various category selections
- [x] Write integration tests for API endpoint
- [x] Test error scenarios and edge cases (empty array, invalid names)
- [x] Performance testing with concurrent requests

### 7.2 Phase 2: BFF Service Implementation (1 day) 🔄 **IN PROGRESS**

#### BFF Service Requirements
- [ ] Add route for bulk category creation: `POST /api/v1/users/{user_id}/categories/default`
- [ ] Forward requests to transaction-service with proper authentication
- [ ] Handle error responses and status code mapping
- [ ] Add request/response logging for debugging
- [ ] Update OpenAPI specification

#### BFF Implementation Details
```rust
// Route to add in BFF service
.route("/api/v1/users/:user_id/categories/default", post(create_default_categories))

// Handler function signature
async fn create_default_categories(
    Path(user_id): Path<String>,
    State(app_state): State<AppState>,
    headers: HeaderMap,
    Json(request): Json<CreateDefaultCategoriesRequest>,
) -> Result<Json<CreateDefaultCategoriesResponse>, ApiError>
```

### 7.3 Phase 3: Frontend Implementation (2-3 days)

#### Day 1: API Integration
- [ ] Fix `deleteCategory` endpoint inconsistency in `transactionsApi.ts`
- [ ] Add `createDefaultCategories` function to API client with category objects array
- [ ] Extend `useCategoryManagement` hook with selection state
- [ ] Add TypeScript interfaces for selection state and CategoryType enum

#### Day 2: UI Components
- [ ] Create category selection component with checkboxes
- [ ] Update `TransactionCategories.tsx` empty state with selection interface
- [ ] Add "Select All" and "Deselect All" functionality
- [ ] Implement real-time selection count display

#### Day 3: Testing & Polish
- [ ] Write component tests for selection interface
- [ ] E2E testing with Playwright for category selection flow
- [ ] UI/UX polish and accessibility
- [ ] Cross-browser testing

### 7.4 Phase 4: Integration & Deployment (1 day)

- [ ] Integration testing across all services
- [ ] Database migration verification
- [ ] Performance monitoring setup
- [ ] Documentation updates

## 8. Success Metrics

### 8.1 Primary Metrics
- **Adoption Rate**: % of new users who use the default categories feature
- **Time to First Transaction**: Reduction in time from signup to first transaction entry
- **User Retention**: 7-day retention rate for users who use default categories vs. those who don't

### 8.2 Secondary Metrics
- **Category Customization**: % of users who modify default categories within 30 days
- **Support Tickets**: Reduction in category-related support requests
- **Feature Satisfaction**: User feedback on the default categories feature

### 8.3 Technical Metrics
- **API Performance**: Response time < 2 seconds for bulk creation
- **Error Rate**: < 1% failure rate for default category creation
- **Database Performance**: No impact on existing category queries

## 9. Risk Assessment

### 9.1 Technical Risks

#### Risk: Database Performance Impact
- **Impact**: High
- **Probability**: Low
- **Mitigation**: Use batch inserts, add database indexes, performance testing

#### Risk: Concurrent User Issues
- **Impact**: Medium
- **Probability**: Medium
- **Mitigation**: Proper transaction handling, unique constraints

### 9.2 Product Risks

#### Risk: Users Don't Like Default Categories
- **Impact**: Medium
- **Probability**: Low
- **Mitigation**: Allow easy deletion/modification, gather user feedback

#### Risk: Feature Complexity
- **Impact**: Low
- **Probability**: Medium
- **Mitigation**: Keep implementation simple, focus on core functionality

## 10. Future Enhancements

### 10.1 Short-term (Next Sprint)
- **Category Templates**: Multiple sets of default categories (e.g., "Student", "Family", "Business")
- **Customization**: Allow users to select which default categories to create
- **Analytics**: Track which default categories are most/least used

### 10.2 Long-term (Future Releases)
- **AI-Powered Categories**: Use AI to suggest personalized categories based on user's transaction history
- **Category Import**: Allow users to import categories from other financial apps
- **Category Sharing**: Allow users to share custom category sets with others

## 11. Acceptance Criteria

### 11.1 Functional Acceptance
- [ ] Selection interface only appears when user has zero categories
- [ ] All 18 default categories are displayed with checkboxes
- [ ] All categories are pre-selected by default
- [ ] "Select All" and "Deselect All" buttons work correctly
- [ ] Only selected categories are created in single API call
- [ ] Real-time selection count is displayed
- [ ] Create button is disabled when no categories selected
- [ ] User can immediately edit/delete created categories
- [ ] Proper error handling for all failure scenarios

### 11.2 Performance Acceptance
- [ ] API response time < 2 seconds
- [ ] No database performance degradation
- [ ] Handles concurrent requests properly

### 11.3 User Experience Acceptance
- [ ] Clear visual feedback during creation process
- [ ] Intuitive category selection interface with organized layout
- [ ] Smooth transition to categories list after creation
- [ ] Accessible design (WCAG 2.1 AA compliance)
- [ ] Responsive design works on mobile devices
- [ ] Keyboard navigation support for checkboxes

## 12. Implementation Status & Details

### 12.1 Transaction Service ✅ **COMPLETED**
**Commit**: `6f4dcc2` on branch `feat/generic_categories`

**Implemented Features:**
- ✅ `category_type` field (Income/Expense) added to TransactionCategory model
- ✅ Database migration with constraints and indexes
- ✅ Bulk category creation endpoint: `POST /v1/users/{user_id}/categories/default`
- ✅ Comprehensive validation and error handling
- ✅ 7 new integration tests covering all scenarios
- ✅ CSV import logic updated to determine category_type from transaction amount
- ✅ All 16 transaction category tests passing

**API Endpoint Details:**
```rust
// Transaction Service Endpoint
POST /v1/users/{user_id}/categories/default

// Request Body
{
  "categories": [
    {"name": "Salary", "category_type": "Income"},
    {"name": "Groceries", "category_type": "Expense"}
  ]
}

// Response
{
  "categories": [
    {
      "id": "uuid",
      "categoryId": "uuid", 
      "name": "Salary",
      "categoryType": "Income"
    }
  ],
  "message": "Successfully created 2 default categories"
}
```

### 12.2 BFF Service ✅ **COMPLETED**

**Commit**: Implementation completed on branch `feat/generic_categories`

**Implemented Features:**
- ✅ `POST /v1/users/{user_id}/categories/default` route added
- ✅ `create_default_categories` handler with JWT authentication
- ✅ Request/response types: `CreateDefaultCategoriesRequest/Response`
- ✅ Transaction service client integration with proper error mapping
- ✅ OpenAPI specification updated with new endpoint
- ✅ Comprehensive logging and error handling

**API Endpoint Details:**
```rust
// BFF Service Endpoint
POST /v1/users/{user_id}/categories/default

// Request Body
{
  "categories": [
    {"name": "Salary", "category_type": "Income"},
    {"name": "Groceries", "category_type": "Expense"}
  ]
}

// Response
{
  "categories": [
    {
      "category_id": "uuid",
      "name": "Salary"
    }
  ],
  "message": "Successfully created 2 default categories"
}
```

**Implementation Files:**
- `src/api/models/transaction.rs` - Request/response types
- `src/api/transaction_actions.rs` - Handler function with OpenAPI docs
- `src/services/transaction_service.rs` - HTTP client implementation
- `src/domain/transaction/mod.rs` - Domain function
- `src/api/mod.rs` - Route configuration
- `src/main.rs` & `src/lib.rs` - OpenAPI specification updates

### 12.3 Frontend 🔄 **NEXT TO IMPLEMENT**

**Required Changes:**

#### 12.3.1 API Client Updates
- **File**: `src/api/transactionsApi.ts`
- **Function**: Add `createDefaultCategories(userId: string, categories: {name: string, type: string}[])`
- **Fix**: Update `deleteCategory` to use correct endpoint with user context
- **Error Handling**: Handle network errors and validation errors

#### 12.3.2 TypeScript Interfaces
- **File**: `src/types/transaction.ts`
- **Add**: `CategoryType` enum (`'Income' | 'Expense'`)
- **Update**: `Category` interface to include `type` field
- **Add**: `CreateDefaultCategoriesRequest` and `CreateDefaultCategoriesResponse` interfaces

#### 12.3.3 State Management
- **File**: `src/hooks/useCategoryManagement.ts`
- **Add**: `isCreatingDefault` loading state
- **Add**: `selectedCategories` array state
- **Add**: `createDefaultCategories` function
- **Update**: Cache invalidation after bulk creation

#### 12.3.4 UI Components
- **File**: `src/components/TransactionCategories.tsx`
- **Replace**: Empty state with category selection interface
- **Add**: Category selection component with checkboxes
- **Add**: "Select All" and "Deselect All" functionality
- **Add**: Real-time selection count display
- **Add**: Loading states and error handling

#### 12.3.5 Default Categories Data
- **File**: `src/data/defaultCategories.ts`
- **Content**: Array of 18 default categories (5 Income, 13 Expense)
- **Structure**: `{name: string, type: CategoryType, selected: boolean}`

**Frontend Implementation Details:**

```typescript
// Default categories data structure
export const DEFAULT_CATEGORIES = [
  // Income Categories
  { name: "Salary", type: "Income" as CategoryType },
  { name: "Freelance", type: "Income" as CategoryType },
  { name: "Investment Returns", type: "Income" as CategoryType },
  { name: "Rental Income", type: "Income" as CategoryType },
  { name: "Other Income", type: "Income" as CategoryType },
  
  // Expense Categories
  { name: "Housing (Rent/Mortgage)", type: "Expense" as CategoryType },
  { name: "Utilities", type: "Expense" as CategoryType },
  { name: "Groceries", type: "Expense" as CategoryType },
  { name: "Transportation", type: "Expense" as CategoryType },
  { name: "Healthcare", type: "Expense" as CategoryType },
  { name: "Entertainment", type: "Expense" as CategoryType },
  { name: "Dining Out", type: "Expense" as CategoryType },
  { name: "Shopping", type: "Expense" as CategoryType },
  { name: "Insurance", type: "Expense" as CategoryType },
  { name: "Education", type: "Expense" as CategoryType },
  { name: "Savings", type: "Expense" as CategoryType },
  { name: "Debt Payment", type: "Expense" as CategoryType },
  { name: "Other Expenses", type: "Expense" as CategoryType },
];

// API client function
export const createDefaultCategories = async (
  userId: string, 
  categories: {name: string, type: string}[]
): Promise<CreateDefaultCategoriesResponse> => {
  const response = await apiClient.post(
    `/v1/users/${userId}/categories/default`,
    { categories }
  );
  return response.data;
};
```

**UI Component Structure:**
```tsx
// Category selection interface
<Box>
  <Typography variant="h5">Setup Your Categories</Typography>
  <Typography variant="body2">Choose from common categories below</Typography>
  
  <Box sx={{ display: 'flex', gap: 1, mb: 2 }}>
    <Button onClick={selectAll}>Select All</Button>
    <Button onClick={deselectAll}>Deselect All</Button>
  </Box>
  
  <Grid container spacing={2}>
    <Grid item xs={6}>
      <Typography variant="h6">Income Categories</Typography>
      {incomeCategories.map(category => (
        <FormControlLabel
          key={category.name}
          control={<Checkbox checked={category.selected} onChange={...} />}
          label={category.name}
        />
      ))}
    </Grid>
    
    <Grid item xs={6}>
      <Typography variant="h6">Expense Categories</Typography>
      {expenseCategories.map(category => (
        <FormControlLabel
          key={category.name}
          control={<Checkbox checked={category.selected} onChange={...} />}
          label={category.name}
        />
      ))}
    </Grid>
  </Grid>
  
  <Box sx={{ mt: 2 }}>
    <Typography variant="body2">
      {selectedCount} of 18 categories selected
    </Typography>
    <Button 
      variant="contained" 
      disabled={selectedCount === 0 || isCreatingDefault}
      onClick={handleCreateCategories}
    >
      {isCreatingDefault ? 'Creating...' : 'Create Selected Categories'}
    </Button>
  </Box>
</Box>
```

## 13. Dependencies

### 13.1 Internal Dependencies
- ✅ Transaction Service API (primary owner) - **COMPLETED**
- ✅ BFF Service routing - **COMPLETED**
- ⏳ Frontend category management components - **NEXT TO IMPLEMENT**
- ✅ Database schema (add category_type column) - **COMPLETED**
- ⏳ Budget Service (references categories by ID) - **NO CHANGES NEEDED**

### 13.2 External Dependencies
- Material-UI Dialog component
- Existing authentication system
- Database connection pooling

## 14. Rollback Plan

### 14.1 Feature Flag
- Implement feature flag to disable default categories button
- Allow quick rollback without code deployment

### 14.2 Database Rollback
- Remove category_type column if needed
- Categories can be deleted individually if needed
- No data migration required (new column has default value)

### 14.3 Code Rollback
- Revert to previous version of TransactionCategories component
- Remove new API endpoint
- Remove BFF route

## 15. Next Steps

### 15.1 Immediate Next Steps
1. **Frontend Implementation** (Priority 1)
   - Add API client function for bulk category creation
   - Create default categories data file
   - Implement category selection UI component
   - Update TransactionCategories component with empty state replacement
   - Add TypeScript interfaces and state management

2. **Integration Testing** (Priority 2)
   - Test BFF → Transaction Service integration
   - E2E testing of full category selection flow
   - Performance testing with bulk creation

### 15.2 Testing Strategy
- **Integration Testing**: BFF ↔ Transaction Service
- **E2E Testing**: Full user flow from frontend to database
- **Performance Testing**: Bulk creation with large category sets

---

**Document Version**: 1.2  
**Last Updated**: January 25, 2025  
**Author**: Development Team  
**Status**: Transaction Service Complete, BFF Service Complete, Frontend Implementation Ready
