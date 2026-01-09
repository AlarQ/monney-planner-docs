# PRD: Enhanced CSV Import with AI-Powered Column Mapping

## Status: To Implement
## Priority: High
## Target Service: transaction-service, MoneyPlannerFE

---

## 1. Problem Statement

### Current Limitations
The current CSV import system has significant usability issues:

1. **Format Lock-in**: Only supports ING Bank format (Polish column names) and a basic generic fallback
2. **No User Control**: Users cannot map columns manually when auto-detection fails
3. **Silent Failures**: Unrecognized formats either fail completely or misparse data
4. **Limited Bank Support**: Adding new bank formats requires code changes
5. **Maintenance Burden**: Each bank format requires a dedicated adapter

### User Pain Points
- Users from other banks cannot import their transaction history
- Even ING users struggle when their export format slightly differs
- No visibility into why imports fail
- Forced to manually format CSV files before import

---

## 2. Proposed Solution

### Design Principles

1. **AI-First Approach**: Remove all bank-specific adapters. Use OpenAI for universal column mapping
2. **Single Upload Flow**: File uploaded once during analyze. Server returns pre-parsed transactions. Confirm endpoint receives parsed data, not the file again
3. **User Override**: Always allow manual column mapping when AI suggestions are wrong
4. **Full Replacement**: Replace existing import endpoints entirely (not a gradual migration)
5. **Code Migration**: Migrate useful utilities (delimiter detection, amount parsing) from existing implementation to new module

### New Import Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: Upload & Parse (Single File Upload)                    │
│  ┌─────────┐    ┌─────────────┐    ┌──────────────────┐        │
│  │ Upload  │───▶│ Parse CSV   │───▶│ Extract Headers  │        │
│  │ CSV     │    │ (detect     │    │ + ALL Rows       │        │
│  └─────────┘    │ delimiter)  │    └────────┬─────────┘        │
└─────────────────────────────────────────────┼──────────────────┘
                                              │
┌─────────────────────────────────────────────▼──────────────────┐
│  STEP 2: AI Column Mapping                                      │
│  ┌─────────────────┐    ┌────────────────────────────────┐     │
│  │ Send headers +  │───▶│ AI returns mapping proposal:   │     │
│  │ sample data to  │    │ - date_column: "Trans Date"    │     │
│  │ OpenAI          │    │ - amount_column: "Amount"      │     │
│  └─────────────────┘    │ - merchant_column: "Payee"     │     │
│                         │ - confidence: 0.92             │     │
│                         │ - date_format: "DD/MM/YYYY"    │     │
│                         └────────────────────────────────┘     │
│                                                                 │
│  Response includes:                                             │
│  - ai_mapping (AI suggestions)                                  │
│  - sample_rows (for UI preview)                                 │
│  - parsed_transactions (ALL rows pre-parsed with AI mapping)    │
└─────────────────────────────────────────────┬──────────────────┘
                                              │
┌─────────────────────────────────────────────▼──────────────────┐
│  STEP 3: User Review & Approval                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ UI shows:                                              │    │
│  │ ┌──────────────┬────────────────┬─────────────────┐   │    │
│  │ │ Our Field    │ AI Suggestion  │ Override        │   │    │
│  │ ├──────────────┼────────────────┼─────────────────┤   │    │
│  │ │ Date         │ "Trans Date" ▼ │ [dropdown]      │   │    │
│  │ │ Amount       │ "Amount"     ▼ │ [dropdown]      │   │    │
│  │ │ Merchant     │ "Payee"      ▼ │ [dropdown]      │   │    │
│  │ │ Description  │ "Memo"       ▼ │ [dropdown]      │   │    │
│  │ │ Currency     │ Not Found    ▼ │ [default: PLN]  │   │    │
│  │ └──────────────┴────────────────┴─────────────────┘   │    │
│  │                                                        │    │
│  │ Preview (first 5 rows with mapping applied):           │    │
│  │ ┌─────────┬─────────┬────────────┬────────┐           │    │
│  │ │ Date    │ Merchant│ Description│ Amount │           │    │
│  │ ├─────────┼─────────┼────────────┼────────┤           │    │
│  │ │ 2024-01│ Amazon  │ Books      │ -45.99 │           │    │
│  │ │ ...     │ ...     │ ...        │ ...    │           │    │
│  │ └─────────┴─────────┴────────────┴────────┘           │    │
│  │                                                        │    │
│  │ [  Cancel  ]                [ Approve & Import ]       │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────┬──────────────────┘
                                              │ (on Approve)
┌─────────────────────────────────────────────▼──────────────────┐
│  STEP 4: Confirm Import (No Re-upload)                          │
│  ┌─────────────────┐    ┌─────────────────┐    ┌────────────┐  │
│  │ Send parsed_    │───▶│ Create          │───▶│ Start AI   │  │
│  │ transactions    │    │ Transactions    │    │ Categori-  │  │
│  │ from analyze    │    │                 │    │ zation Job │  │
│  └─────────────────┘    └─────────────────┘    └────────────┘  │
│                                                                 │
│  Note: Uses existing CategorizationJob infrastructure           │
│  (max 3 concurrent jobs per user, persistent job tracking)      │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. Technical Design

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **AI-first, no adapters** | Eliminates per-bank maintenance cost. AI handles any CSV format universally. |
| **Single file upload** | File uploaded once on analyze. Server returns pre-parsed transactions. Confirm receives parsed data, no re-upload. Saves bandwidth and improves UX. |
| **Clean new module** | New `domain/csv_import/` module instead of extending old adapter architecture. Old paradigm (bank detection → adapter) doesn't fit AI-first approach. |
| **Migrate useful utilities** | Delimiter detection and amount parsing logic are valuable. Migrate to new module rather than rewriting. |
| **Learned mappings cache (Phase 3)** | Store confirmed mappings by header hash. Skip AI for known formats. Reduces API costs over time. |

### 3.1 API Endpoints

#### POST `/v1/users/{user_id}/transactions/import/analyze`
Upload CSV and get AI-suggested column mapping with pre-parsed transactions.

**Request**: Multipart form with:
- `file`: CSV file (required)
- `mapping_override`: JSON string with user-specified mapping (optional)

**Behavior**:
- If `mapping_override` is NOT provided → Call AI for column mapping suggestions, parse with AI mapping
- If `mapping_override` IS provided → Skip AI call, parse directly with provided mapping

This allows users to re-analyze with corrected mapping without wasting an AI API call.

**Example `mapping_override`** (when user corrects AI suggestion):
```json
{
  "date_column": "Post Date",
  "amount_column": "Amount",
  "merchant_column": "Description",
  "description_column": null,
  "currency_column": null,
  "date_format": "MM/DD/YYYY",
  "amount_format": "standard"
}
```

**Response**:
```json
{
  "file_info": {
    "row_count": 150,
    "detected_delimiter": ",",
    "has_header_row": true
  },
  "headers": ["Trans Date", "Post Date", "Description", "Amount", "Category"],
  "sample_rows": [
    ["01/15/2024", "01/16/2024", "AMAZON.COM", "-45.99", "Shopping"],
    ["01/14/2024", "01/15/2024", "STARBUCKS", "-5.50", "Food"]
  ],
  "ai_mapping": {
    "date_column": "Trans Date",
    "amount_column": "Amount",
    "merchant_column": "Description",
    "description_column": null,
    "currency_column": null,
    "date_format": "MM/DD/YYYY",
    "amount_format": "standard",
    "has_separate_debit_credit": false,
    "debit_column": null,
    "credit_column": null,
    "confidence": 0.89,
    "reasoning": "Trans Date contains transaction dates, Amount has monetary values, Description contains merchant names"
  },
  "parsed_transactions": [
    {
      "row_index": 1,
      "date": "2024-01-15",
      "amount": -45.99,
      "merchant": "AMAZON.COM",
      "description": null,
      "currency": "PLN"
    },
    {
      "row_index": 2,
      "date": "2024-01-14",
      "amount": -5.50,
      "merchant": "STARBUCKS",
      "description": null,
      "currency": "PLN"
    }
  ],
  "parse_errors": [
    {
      "row_index": 45,
      "error": "Invalid date format",
      "raw_values": ["N/A", "01/16/2024", "UNKNOWN", "-10.00", ""]
    }
  ],
  "warnings": [
    {
      "type": "missing_currency",
      "message": "No currency column detected. Default currency will be used."
    }
  ]
}
```

**Key Design Decision**: The response includes `parsed_transactions` - ALL rows pre-parsed using the AI mapping. This eliminates the need for file re-upload on confirm. Frontend sends these parsed transactions back on confirm.

#### POST `/v1/users/{user_id}/transactions/import/confirm`
Confirm and create transactions. No file re-upload needed.

**Request**: JSON body with parsed transactions from analyze response

```json
{
  "transactions": [
    {
      "row_index": 1,
      "date": "2024-01-15",
      "amount": -45.99,
      "merchant": "AMAZON.COM",
      "description": null,
      "currency": "PLN"
    }
  ],
  "options": {
    "default_currency": "PLN",
    "skip_zero_amounts": true
  }
}
```

**Note**: If user overrides column mapping in UI, frontend can either:
1. Accept the pre-parsed data as-is (95% of cases - user accepts AI mapping)
2. Re-upload file for re-parsing with new mapping (5% edge case - user changes mapping)

**Response**:
```json
{
  "import_result": {
    "total_rows": 150,
    "imported_rows": 148,
    "skipped_rows": 2,
    "transaction_ids": ["uuid1", "uuid2", "..."]
  },
  "categorization_job_id": "uuid"
}
```

**Note**: Categorization uses existing `CategorizationJob` infrastructure (max 3 concurrent jobs per user, persistent job tracking, cancellation support).

---

### 3.2 AI Prompt Design for Column Mapping

```text
SYSTEM PROMPT:
You are a CSV column mapping assistant. Analyze the provided CSV headers and sample data to determine which columns map to our standard transaction format.

Required fields:
- date_column: Transaction date (required)
- amount_column: Transaction amount (required)

Optional fields:
- merchant_column: Merchant/payee name
- description_column: Transaction description/memo
- currency_column: Currency code (if present)

Also determine:
- date_format: One of "YYYY-MM-DD", "DD/MM/YYYY", "MM/DD/YYYY", "DD.MM.YYYY", "DD-MM-YYYY"
- amount_format: One of "standard" (1234.56), "european" (1.234,56)

Rules:
1. If multiple date columns exist, prefer "transaction date" over "posting date"
2. If separate debit/credit columns exist, set has_separate_debit_credit to true and specify debit_column/credit_column
3. If a column contains both merchant and description, map to merchant_column
4. Return null for fields that cannot be determined
5. Provide a confidence score (0.0-1.0) based on how certain you are
6. Include brief reasoning for your mapping decisions

Respond with valid JSON only, no additional text.

USER PROMPT:
Headers: ["Trans Date", "Post Date", "Description", "Amount", "Category"]

Sample rows (first 3):
Row 1: ["01/15/2024", "01/16/2024", "AMAZON.COM*123ABC", "-45.99", "Shopping"]
Row 2: ["01/14/2024", "01/15/2024", "STARBUCKS #1234", "-5.50", "Food & Drink"]
Row 3: ["01/13/2024", "01/14/2024", "DEPOSIT - PAYROLL", "2500.00", "Income"]

Analyze and provide the column mapping.
```

**Expected Response**:
```json
{
  "date_column": "Trans Date",
  "amount_column": "Amount",
  "merchant_column": "Description",
  "description_column": null,
  "currency_column": null,
  "date_format": "MM/DD/YYYY",
  "amount_format": "standard",
  "has_separate_debit_credit": false,
  "debit_column": null,
  "credit_column": null,
  "confidence": 0.92,
  "reasoning": "Trans Date contains transaction dates in MM/DD/YYYY format. Amount column has monetary values with standard decimal format. Description contains merchant names with location identifiers. Post Date is the posting date, not primary transaction date. No currency column present."
}
```

---

### 3.3 Data Models

#### New Domain Models

```rust
// Value object for AI mapping response
pub struct AiColumnMapping {
    pub date_column: String,              // Required
    pub amount_column: String,            // Required
    pub merchant_column: Option<String>,
    pub description_column: Option<String>,
    pub currency_column: Option<String>,
    pub date_format: DateFormat,
    pub amount_format: AmountFormat,
    pub has_separate_debit_credit: bool,
    pub debit_column: Option<String>,
    pub credit_column: Option<String>,
    pub confidence: f32,
    pub reasoning: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DateFormat {
    #[serde(rename = "YYYY-MM-DD")]
    IsoYmd,
    #[serde(rename = "DD/MM/YYYY")]
    EuropeanSlash,
    #[serde(rename = "MM/DD/YYYY")]
    AmericanSlash,
    #[serde(rename = "DD.MM.YYYY")]
    EuropeanDot,
    #[serde(rename = "DD-MM-YYYY")]
    EuropeanDash,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AmountFormat {
    #[serde(rename = "standard")]
    Standard,      // 1234.56 (dot decimal, optional comma thousands)
    #[serde(rename = "european")]
    European,      // 1.234,56 (comma decimal, dot thousands)
}

// User-provided mapping (may override AI suggestions)
pub struct ColumnMapping {
    pub date_column: String,
    pub amount_column: String,
    pub merchant_column: Option<String>,
    pub description_column: Option<String>,
    pub currency_column: Option<String>,
}

pub struct ImportOptions {
    pub date_format: DateFormat,
    pub amount_format: AmountFormat,
    pub default_currency: String,
    pub skip_header_rows: u32,
    pub skip_zero_amounts: bool,
    pub invert_amounts: bool,  // For banks that report expenses as positive
}

// Pre-parsed transaction (returned from /analyze, sent back to /confirm)
pub struct ParsedTransaction {
    pub row_index: u32,
    pub date: NaiveDate,
    pub amount: Decimal,
    pub merchant: Option<String>,
    pub description: Option<String>,
    pub currency: String,
}

// Parse error for rows that failed parsing
pub struct ParseError {
    pub row_index: u32,
    pub error: String,
    pub raw_values: Vec<String>,
}

// Analysis result (returned from /analyze endpoint)
pub struct CsvAnalysisResult {
    pub file_info: FileInfo,
    pub headers: Vec<String>,
    pub sample_rows: Vec<Vec<String>>,          // Raw data for UI display
    pub ai_mapping: AiColumnMapping,
    pub parsed_transactions: Vec<ParsedTransaction>,  // Pre-parsed with AI mapping
    pub parse_errors: Vec<ParseError>,
    pub warnings: Vec<ImportWarning>,
}

pub struct FileInfo {
    pub row_count: usize,
    pub detected_delimiter: char,
    pub has_header_row: bool,
}

pub struct ImportWarning {
    pub warning_type: String,
    pub message: String,
}

// Confirm request (receives parsed transactions, no file re-upload)
pub struct ConfirmImportRequest {
    pub transactions: Vec<ParsedTransaction>,
    pub options: ConfirmOptions,
}

pub struct ConfirmOptions {
    pub default_currency: String,
    pub skip_zero_amounts: bool,
}
```

---

### 3.4 Backend Changes

#### Remove Existing Adapters

Delete the following files/modules:
- `src/services/csv_formats/ing_bank.rs`
- `src/services/csv_formats/generic_csv.rs`
- `src/services/csv_formats/registry.rs`
- `src/services/csv_formats/mod.rs`

The `CsvFormat` trait and all adapters will be replaced with the AI-powered approach.

#### Migrate Useful Utilities

Before deleting, migrate these utilities to the new `domain/csv_import/` module:

| Source File | Utility | Target |
|-------------|---------|--------|
| `detection.rs` | `detect_delimiter()` - scoring algorithm for delimiter detection | `domain/csv_import/parsing.rs` |
| `amount_parsing.rs` | Amount parsing logic (PLN symbols, comma/dot normalization, unicode minus) | `domain/csv_import/parsing.rs` |

These utilities are well-tested and handle edge cases. Rewrite only the interface, keep the core logic.

#### New Domain Module: `domain/csv_import/`

| Component | Description |
|-----------|-------------|
| `AiColumnMapping` | Value object for AI mapping response |
| `ColumnMapping` | Value object for user-confirmed mapping |
| `ImportOptions` | Value object for import configuration |
| `DateFormat` | Enum for date format variants |
| `AmountFormat` | Enum for amount format variants |
| `CsvAnalysisResult` | Aggregate for analyze endpoint response |

#### Extend AiService Trait

Add new method to existing `domain/interfaces/ai_service.rs`:

```rust
#[async_trait]
pub trait AiService: Send + Sync {
    // Existing method
    async fn categorize_transaction(&self, request: CategorizationRequest)
        -> Result<CategorizationResponse, TransactionError>;

    // New method for column mapping
    async fn analyze_csv_columns(&self, request: CsvColumnAnalysisRequest)
        -> Result<AiColumnMapping, TransactionError>;
}

pub struct CsvColumnAnalysisRequest {
    pub headers: Vec<String>,
    pub sample_rows: Vec<Vec<String>>,  // First 3-5 rows
}
```

#### New CSV Parsing Service

Create `domain/csv_import/parsing.rs` with free functions:

```rust
/// Detect delimiter by trying common options and scoring
pub fn detect_delimiter(csv_content: &str) -> char

/// Parse CSV and extract headers + sample rows
pub fn extract_csv_structure(csv_content: &str, delimiter: char)
    -> Result<(Vec<String>, Vec<Vec<String>>, usize), TransactionError>

/// Parse a single row using confirmed mapping
pub fn parse_row_with_mapping(
    row: &[String],
    headers: &[String],
    mapping: &ColumnMapping,
    options: &ImportOptions,
) -> Result<ParsedTransaction, RowParseError>

/// Parse date string according to format
pub fn parse_date(value: &str, format: &DateFormat) -> Result<NaiveDate, ParseError>

/// Parse amount string according to format
pub fn parse_amount(value: &str, format: &AmountFormat) -> Result<Decimal, ParseError>
```

#### New API Handlers

| Endpoint | Handler |
|----------|---------|
| `POST /import/analyze` | `analyze_csv_handler` |
| `POST /import/confirm` | `confirm_import_handler` |

Remove old handlers:
- `preview_csv_handler`
- `confirm_csv_import_async_handler`

---

### 3.5 Frontend Components

#### New Components: `src/components/transactions/import/`

| Component | Props | Description |
|-----------|-------|-------------|
| `CsvImportWizard` | `onComplete`, `onCancel` | Main wizard container with step management |
| `ColumnMappingReview` | `headers`, `aiMapping`, `sampleRows`, `onChange` | Column mapping table with dropdowns |
| `MappingPreviewTable` | `sampleRows`, `mapping`, `options` | Live preview of parsed transactions |
| `ImportOptionsForm` | `options`, `onChange` | Date format, currency, skip options |

#### New Types: `src/types/csvImport.ts`

```typescript
interface CsvAnalysisResponse {
  file_info: {
    row_count: number;
    detected_delimiter: string;
    has_header_row: boolean;
  };
  headers: string[];
  sample_rows: string[][];
  ai_mapping: AiColumnMapping;
  parsed_transactions: ParsedTransaction[];  // Pre-parsed with AI mapping
  parse_errors: ParseError[];
  warnings: ImportWarning[];
}

interface AiColumnMapping {
  date_column: string;
  amount_column: string;
  merchant_column: string | null;
  description_column: string | null;
  currency_column: string | null;
  date_format: DateFormat;
  amount_format: AmountFormat;
  has_separate_debit_credit: boolean;
  debit_column: string | null;
  credit_column: string | null;
  confidence: number;
  reasoning: string;
}

// Pre-parsed transaction (returned from analyze, sent to confirm)
interface ParsedTransaction {
  row_index: number;
  date: string;  // ISO format YYYY-MM-DD
  amount: number;
  merchant: string | null;
  description: string | null;
  currency: string;
}

interface ParseError {
  row_index: number;
  error: string;
  raw_values: string[];
}

interface ImportWarning {
  type: string;
  message: string;
}

// Confirm request - no file re-upload, sends parsed transactions
interface ConfirmImportRequest {
  transactions: ParsedTransaction[];
  options: ConfirmOptions;
}

interface ConfirmOptions {
  default_currency: string;
  skip_zero_amounts: boolean;
}

interface ImportResult {
  import_result: {
    total_rows: number;
    imported_rows: number;
    skipped_rows: number;
    transaction_ids: string[];
  };
  categorization_job_id: string;
}

type DateFormat = 'YYYY-MM-DD' | 'DD/MM/YYYY' | 'MM/DD/YYYY' | 'DD.MM.YYYY' | 'DD-MM-YYYY';
type AmountFormat = 'standard' | 'european';
```

#### New API Service: `src/services/csvImportService.ts`

```typescript
export const csvImportService = {
  // Step 1: Upload file, get AI mapping + pre-parsed transactions
  analyzeCsv: async (file: File): Promise<CsvAnalysisResponse> => {
    const formData = new FormData();
    formData.append('file', file);
    return api.post('/import/analyze', formData);
  },

  // Step 2: Confirm with parsed transactions (no file re-upload)
  confirmImport: async (request: ConfirmImportRequest): Promise<ImportResult> => {
    return api.post('/import/confirm', request);  // JSON body, not multipart
  },

  // Optional: Re-analyze with different mapping (edge case - user overrides mapping)
  reAnalyzeWithMapping: async (file: File, mapping: ColumnMapping): Promise<CsvAnalysisResponse> => {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('mapping_override', JSON.stringify(mapping));
    return api.post('/import/analyze', formData);
  },
};
```

#### State Management

| Hook | Purpose |
|------|---------|
| `useCsvImportWizard` | Custom hook for wizard state (step, file, analysis result with parsed transactions) |

```typescript
function useCsvImportWizard() {
  const [step, setStep] = useState<'upload' | 'review' | 'importing' | 'complete'>('upload');
  const [file, setFile] = useState<File | null>(null);  // Kept for edge case re-analysis
  const [analysisResult, setAnalysisResult] = useState<CsvAnalysisResponse | null>(null);
  const [options, setOptions] = useState<ConfirmOptions>(defaultOptions);

  const handleAnalyze = async (file: File) => {
    setFile(file);
    const result = await csvImportService.analyzeCsv(file);
    setAnalysisResult(result);  // Contains parsed_transactions
    setStep('review');
  };

  const handleConfirm = async () => {
    if (!analysisResult) return;
    setStep('importing');
    // Send parsed_transactions back - no file re-upload
    const result = await csvImportService.confirmImport({
      transactions: analysisResult.parsed_transactions,
      options,
    });
    setStep('complete');
    return result;
  };

  // Edge case: user overrides mapping, need to re-parse
  const handleMappingOverride = async (newMapping: ColumnMapping) => {
    if (!file) return;
    const result = await csvImportService.reAnalyzeWithMapping(file, newMapping);
    setAnalysisResult(result);
  };

  return { step, analysisResult, options, setOptions, handleAnalyze, handleConfirm, handleMappingOverride };
}
```

---

### 3.6 BFF Changes

#### Route Updates

| Route | Content Type | Action |
|-------|--------------|--------|
| `POST /v1/users/{user_id}/transactions/import/analyze` | multipart/form-data | Proxy to transaction-service (file upload) |
| `POST /v1/users/{user_id}/transactions/import/confirm` | application/json | Proxy to transaction-service (no file) |

Remove old routes:
- `POST /import/preview`
- `POST /import/confirm-async`

#### Considerations
- Handle multipart/form-data forwarding for analyze endpoint only
- Confirm endpoint is standard JSON - no special handling needed
- Set appropriate timeout for analyze endpoint (AI call may take 5-10 seconds)

---

## 4. Edge Cases & Error Handling

### 4.1 CSV Parsing Issues

| Issue | Detection | Handling |
|-------|-----------|----------|
| **Wrong delimiter** | Try common delimiters (`,`, `;`, `\t`, `\|`) | Auto-detect based on column consistency |
| **No header row** | AI determines from sample data | Allow user to specify skip_header_rows = 0 |
| **Multiple header rows** | Metadata before actual headers | Allow user to specify skip_header_rows > 1 |
| **Duplicate column names** | Count unique vs total headers | Append index to duplicates (Amount, Amount_2) |
| **Empty columns** | All values in column are empty | Warn but allow mapping |
| **Quoted values with delimiters** | Standard CSV parsing | Use proper CSV parser (csv crate handles this) |

### 4.2 AI Mapping Issues

| Issue | Detection | Handling |
|-------|-----------|----------|
| **Low confidence mapping** | confidence < 0.7 | Show warning in UI, highlight fields |
| **AI hallucination** | Suggested column doesn't exist | Validate AI response against actual headers, return error |
| **Ambiguous columns** | Multiple date/amount columns | AI explains choice in reasoning, user can override |
| **AI service unavailable** | API timeout/error | Return error, show manual mapping UI only |
| **Missing required fields** | AI returns null for date/amount | Return error with clear message |
| **User overrides mapping** | User changes column selection in UI | Call `/analyze` with `mapping_override` parameter, skip AI, re-parse with user mapping |

### 4.3 Data Parsing Issues

| Issue | Detection | Handling |
|-------|-----------|----------|
| **Ambiguous dates** | 01/02/2024 (US vs EU) | AI suggests format, preview shows result, user confirms |
| **Mixed date formats** | Different formats in same column | Report rows as errors in response |
| **Negative amounts** | Parentheses vs minus sign | Handle both: (50.00) = -50.00 |
| **Separate debit/credit** | AI detects two amount columns | Combine: expense = -debit, income = +credit |
| **Currency symbols in amount** | "$50.00", "50.00 USD" | Strip symbols during parsing |
| **Invalid amount** | "N/A", "-", empty | Skip row, report in errors array |

### 4.4 Business Logic Edge Cases

| Issue | Handling |
|-------|----------|
| **All rows fail parsing** | Block import, return detailed errors |
| **> 50% rows fail** | Warn user in response, allow import of valid rows |
| **Zero amounts** | Skip by default (configurable via skip_zero_amounts) |
| **Future dates** | Allow (scheduled transactions are valid) |
| **Very old dates** | Allow (historical imports are valid) |

---

## 5. Implementation Plan

### Phase 1: Backend Core
**Scope**: New endpoints and AI integration

1. **Create CSV Import Domain Module**
   - Add `domain/csv_import/` with value objects and parsing functions
   - Implement delimiter detection
   - Implement date/amount parsing for all formats

2. **Extend AI Service**
   - Add `analyze_csv_columns` method to `AiService` trait
   - Implement in `AiUtilsService`
   - Add response validation (ensure columns exist in headers)

3. **Create New API Endpoints**
   - `POST /import/analyze` - upload, parse, call AI
   - `POST /import/confirm` - parse with mapping, create transactions

4. **Remove Old Implementation**
   - Delete `src/services/csv_formats/` directory
   - Remove old `/import/preview` and `/import/confirm-async` endpoints
   - Update tests

### Phase 2: Frontend Integration
**Scope**: New wizard UI

1. **Create Import Wizard**
   - `CsvImportWizard` container with step management
   - `ColumnMappingReview` with AI suggestions + override dropdowns
   - `MappingPreviewTable` with live preview
   - `ImportOptionsForm` for date format, currency, etc.

2. **API Integration**
   - `csvImportService.ts` with analyze/confirm methods
   - `useCsvImportWizard` custom hook
   - Error handling with snackbar notifications

3. **Replace Existing Flow**
   - Update transactions page to use new wizard
   - Remove old import components

### Phase 3: Polish & Optimization
**Scope**: Edge cases, UX improvements, and cost optimization

1. **Enhanced Parsing**
   - Debit/credit column combination
   - Amount inversion option
   - Better error messages

2. **UI Improvements**
   - Confidence indicator for AI mapping
   - Warning badges for low confidence fields
   - Better error display in preview

3. **Learned Mappings Cache** (Cost Optimization)
   - When user confirms AI mapping, hash the headers pattern (sorted, normalized)
   - Store confirmed mapping in database: `{ header_hash, mapping, usage_count }`
   - On future imports: check if headers match a cached mapping
   - If match found with high usage_count → skip AI call, use cached mapping
   - Benefits:
     - Reduced AI API costs for repeat formats
     - Faster imports for known formats
     - Self-improving system over time

---

## 6. Implementation Checklist

### Transaction Service

- [ ] Create `domain/csv_import/` module
  - [ ] `AiColumnMapping` value object
  - [ ] `ColumnMapping` value object
  - [ ] `ImportOptions` value object
  - [ ] `DateFormat` and `AmountFormat` enums
  - [ ] `ParsedTransaction` struct
  - [ ] `ParseError` struct
  - [ ] `CsvAnalysisResult` struct
  - [ ] `ConfirmImportRequest` struct
- [ ] Migrate utilities from existing code
  - [ ] Copy `detect_delimiter()` logic from `detection.rs`
  - [ ] Copy amount parsing logic from `amount_parsing.rs`
  - [ ] Adapt interfaces to new module structure
- [ ] Implement parsing functions in `domain/csv_import/parsing.rs`
  - [ ] `detect_delimiter()` (migrated)
  - [ ] `extract_csv_structure()`
  - [ ] `parse_all_rows()` - parse entire CSV with mapping, return `Vec<ParsedTransaction>`
  - [ ] `parse_date()` for all DateFormat variants
  - [ ] `parse_amount()` for all AmountFormat variants (migrated)
- [ ] Extend `AiService` trait with `analyze_csv_columns()` method
- [ ] Implement AI column analysis in `AiUtilsService`
  - [ ] Build prompt with headers + sample rows
  - [ ] Parse and validate AI response
  - [ ] Handle AI errors gracefully
- [ ] Create API handlers
  - [ ] `analyze_csv_handler` - multipart upload, parse, AI call, return pre-parsed transactions
    - [ ] Support optional `mapping_override` parameter to skip AI and use provided mapping
  - [ ] `confirm_import_handler` - JSON body with parsed transactions (no file upload), create transactions
- [ ] Add OpenAPI documentation with `utoipa`
- [ ] Delete old CSV format adapters
  - [ ] Remove `src/services/csv_formats/` directory
  - [ ] Remove old API handlers (`preview_csv_handler`, `confirm_csv_import_async_handler`)
- [ ] Update/add tests
  - [ ] Unit tests for parsing functions
  - [ ] Integration tests for new endpoints
  - [ ] AI response validation tests
  - [ ] Test that confirm works without file re-upload

### Frontend

- [ ] Create types in `src/types/csvImport.ts`
- [ ] Create `csvImportService.ts`
- [ ] Create `useCsvImportWizard` hook
- [ ] Create components in `src/components/transactions/import/`
  - [ ] `CsvImportWizard`
  - [ ] `ColumnMappingReview`
  - [ ] `MappingPreviewTable`
  - [ ] `ImportOptionsForm`
- [ ] Update transactions page
- [ ] Remove old import components
- [ ] Add Playwright E2E tests

### BFF

- [ ] Add route for `/import/analyze` (multipart forwarding)
- [ ] Add route for `/import/confirm` (JSON forwarding - no file)
- [ ] Remove old `/import/preview` and `/import/confirm-async` routes
- [ ] Configure multipart forwarding for analyze endpoint
- [ ] Set appropriate timeout (15 seconds for AI on analyze endpoint)

---

## 7. Security Considerations

1. **File Upload Security**
   - Validate file size limits (max 10MB)
   - Validate content type (text/csv, application/csv)
   - Sanitize file content before AI processing
   - Don't expose raw file content in error messages

2. **AI Prompt Injection**
   - Sanitize CSV headers before sending to AI
   - Validate AI response format strictly
   - Ensure suggested columns exist in actual headers
   - Don't execute any code from AI responses

3. **Data Privacy**
   - Don't log full transaction data
   - Inform users about AI processing (sample rows sent to OpenAI)
   - No server-side storage of uploaded files

---

## 8. Testing Strategy

### Unit Tests
- Delimiter detection with various CSV formats
- Date parsing for all DateFormat variants
- Amount parsing for standard and European formats
- AI response validation (column existence check)

### Integration Tests
- Full flow: upload → analyze → confirm → transactions created
- AI service error handling (timeout, invalid response)
- Multipart file upload handling
- Categorization job triggered after import

### E2E Tests (Playwright)
- Complete wizard flow in browser
- Column override via dropdown
- Error handling and recovery
- Large file import

### Test Data
- Sample CSVs from various banks (see Appendix A)
- Edge case files (duplicate columns, mixed formats)
- Large files for performance testing

---

## Appendix A: Sample Bank Formats

### ING Bank (Poland)
```csv
"Data transakcji";"Data księgowania";"Dane kontrahenta";"Tytuł";"Nr rachunku";"Kwota transakcji (waluta rachunku)";"Waluta"
"2024-01-15";"2024-01-15";"AMAZON EU SARL";"Zakup książek";"PL123...";"−45,99";"PLN"
```

### Chase Bank (USA)
```csv
Trans Date,Post Date,Description,Amount,Category
01/15/2024,01/16/2024,AMAZON.COM*123ABC,-45.99,Shopping
```

### Revolut
```csv
Type,Product,Started Date,Completed Date,Description,Amount,Fee,Currency,State,Balance
CARD_PAYMENT,Current,2024-01-15 10:30:00,2024-01-15 10:30:00,Amazon,-45.99,0.00,PLN,COMPLETED,1234.56
```

### N26
```csv
"Date","Payee","Account number","Transaction type","Payment reference","Amount (EUR)","Amount (Foreign Currency)","Type Foreign Currency","Exchange Rate"
"2024-01-15","Amazon EU","DE123...","MasterCard Payment","","−45.99","","",""
```

### Santander (Poland)
```csv
Data operacji;Rodzaj operacji;Tytuł;Odbiorca/Nadawca;Numer rachunku;Kwota;Waluta
2024-01-15;Przelew wychodzący;Zakupy online;AMAZON;PL123...;-45,99;PLN
```

---

## Appendix B: AI Mapping Response Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["date_column", "amount_column", "date_format", "amount_format", "confidence"],
  "properties": {
    "date_column": { "type": "string" },
    "amount_column": { "type": "string" },
    "merchant_column": { "type": ["string", "null"] },
    "description_column": { "type": ["string", "null"] },
    "currency_column": { "type": ["string", "null"] },
    "date_format": {
      "type": "string",
      "enum": ["YYYY-MM-DD", "DD/MM/YYYY", "MM/DD/YYYY", "DD.MM.YYYY", "DD-MM-YYYY"]
    },
    "amount_format": {
      "type": "string",
      "enum": ["standard", "european"]
    },
    "has_separate_debit_credit": { "type": "boolean" },
    "debit_column": { "type": ["string", "null"] },
    "credit_column": { "type": ["string", "null"] },
    "confidence": { "type": "number", "minimum": 0, "maximum": 1 },
    "reasoning": { "type": "string" }
  }
}
```

---

## Appendix C: Removed Features (Deferred)

The following features have been removed to reduce complexity. They may be reconsidered in future iterations:

1. ~~**Saved Column Mappings**~~ - Moved to Phase 3 as "Learned Mappings Cache" (automatic, not user-managed)
2. **Encoding Detection** - Auto-detect non-UTF8 encodings (chardetng/encoding_rs)
3. ~~**Session Storage**~~ - Not needed with single-upload flow (parsed transactions returned in analyze response)
4. **Bank Format Templates** - Community-contributed mapping templates
5. **Import History** - Track past imports with rollback capability
