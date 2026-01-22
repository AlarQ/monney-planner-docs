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
2. **Two-Step Flow**: Analyze returns AI mapping + sample preview. Confirm receives file + confirmed mapping for full parsing
3. **User Override**: Always allow manual column mapping when AI suggestions are wrong
4. **Full Replacement**: Replace existing import endpoints entirely (not a gradual migration)
5. **Extend Existing Structure**: Add new models to existing `domain/transaction/models/`. Keep parsing utilities in `services/`

### New Import Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: Upload & Analyze (File Upload #1)                      │
│  ┌─────────┐    ┌─────────────┐    ┌──────────────────┐        │
│  │ Upload  │───▶│ Parse CSV   │───▶│ Extract Headers  │        │
│  │ CSV     │    │ (detect     │    │ + Sample Rows    │        │
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
│  - sample_rows (raw data for UI preview - first 5-10 rows)     │
│  - file_info (row count, delimiter, etc.)                      │
│  - warnings (missing columns, etc.)                            │
│  NOTE: Does NOT include parsed transactions (parse on confirm) │
└─────────────────────────────────────────────┬──────────────────┘
                                              │
┌─────────────────────────────────────────────▼──────────────────┐
│  STEP 3: User Review & Mapping Confirmation                     │
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
│  │ Preview (sample rows with current mapping applied):    │    │
│  │ ┌─────────┬─────────┬────────────┬────────┐           │    │
│  │ │ Date    │ Merchant│ Description│ Amount │           │    │
│  │ ├─────────┼─────────┼────────────┼────────┤           │    │
│  │ │ 2024-01│ Amazon  │ Books      │ -45.99 │           │    │
│  │ │ ...     │ ...     │ ...        │ ...    │           │    │
│  │ └─────────┴─────────┴────────────┴────────┘           │    │
│  │                                                        │    │
│  │ [  Cancel  ]                [ Approve & Import ]       │    │
│  └────────────────────────────────────────────────────────┘    │
│  Note: Preview is rendered client-side using sample_rows +     │
│  current mapping. No server call needed for preview updates.   │
└─────────────────────────────────────────────┬──────────────────┘
                                              │ (on Approve)
┌─────────────────────────────────────────────▼──────────────────┐
│  STEP 4: Confirm Import (File Upload #2)                        │
│  ┌─────────────────┐    ┌─────────────────┐    ┌────────────┐  │
│  │ Re-upload file  │───▶│ Parse ALL rows  │───▶│ Create     │  │
│  │ + confirmed     │    │ with confirmed  │    │ Transactions│  │
│  │ mapping + opts  │    │ mapping         │    │            │  │
│  └─────────────────┘    └─────────────────┘    └─────┬──────┘  │
│                                                       │         │
│                                              ┌────────▼───────┐ │
│                                              │ Start AI       │ │
│                                              │ Categorization │ │
│                                              │ Job            │ │
│                                              └────────────────┘ │
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
| **Parse on confirm, not analyze** | Analyze returns sample rows + AI mapping only. Full parsing happens on confirm. Keeps response payload small and avoids memory issues with large files. |
| **Two file uploads** | File uploaded on analyze (for AI mapping) and confirm (for full parsing). Stateless design - no server-side file storage needed. |
| **Extend existing domain** | Add new models to `domain/transaction/models/csv.rs`. Keep parsing utilities in `services/csv_parsing/`. No new bounded context needed. |
| **Client-side preview** | Frontend renders preview using sample_rows + current mapping. Mapping changes don't require server calls. |
| **Learned mappings cache (Phase 3)** | Store confirmed mappings by header hash. Skip AI for known formats. Reduces API costs over time. |

### 3.1 API Endpoints

#### POST `/v1/users/{user_id}/transactions/import/analyze`
Upload CSV and get AI-suggested column mapping. Returns sample rows for preview, NOT parsed transactions.

**Request**: Multipart form with:
- `file`: CSV file (required)

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
    ["01/14/2024", "01/15/2024", "STARBUCKS", "-5.50", "Food"],
    ["01/13/2024", "01/14/2024", "WALMART", "-123.45", "Shopping"],
    ["01/12/2024", "01/13/2024", "NETFLIX", "-15.99", "Entertainment"],
    ["01/11/2024", "01/12/2024", "UBER", "-22.00", "Transportation"]
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
  "warnings": [
    {
      "type": "missing_currency",
      "message": "No currency column detected. Default currency will be used."
    }
  ]
}
```

**Key Design Decision**: Response does NOT include `parsed_transactions`. Full parsing happens on `/confirm`. This keeps the response small and avoids memory issues with large files.

#### POST `/v1/users/{user_id}/transactions/import/confirm`
Re-upload CSV with confirmed mapping. Parses all rows and creates transactions.

**Request**: Multipart form with:
- `file`: CSV file (required, same file as analyze)
- `mapping`: JSON string with confirmed column mapping (required)
- `options`: JSON string with import options (required)

**Example `mapping`**:
```json
{
  "date_column": "Trans Date",
  "amount_column": "Amount",
  "merchant_column": "Description",
  "description_column": null,
  "currency_column": null,
  "date_format": "MM/DD/YYYY",
  "amount_format": "standard"
}
```

**Example `options`**:
```json
{
  "default_currency": "PLN",
  "skip_zero_amounts": true,
  "invert_amounts": false
}
```

**Response**:
```json
{
  "import_result": {
    "total_rows": 150,
    "imported_rows": 148,
    "skipped_rows": 2,
    "transaction_ids": ["uuid1", "uuid2", "..."]
  },
  "parse_errors": [
    {
      "row_index": 45,
      "error": "Invalid date format",
      "raw_values": ["N/A", "01/16/2024", "UNKNOWN", "-10.00", ""]
    }
  ],
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

#### Location: `domain/transaction/models/csv.rs`
Extend the existing CSV models file with new types. These are value objects for the AI-powered import flow.

```rust
// Value object for AI mapping response
#[derive(Debug, Clone, Serialize, Deserialize)]
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

// User-confirmed mapping (sent with /confirm request)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ColumnMapping {
    pub date_column: String,
    pub amount_column: String,
    pub merchant_column: Option<String>,
    pub description_column: Option<String>,
    pub currency_column: Option<String>,
    pub date_format: DateFormat,
    pub amount_format: AmountFormat,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ImportOptions {
    pub default_currency: String,
    pub skip_zero_amounts: bool,
    pub invert_amounts: bool,  // For banks that report expenses as positive
}

// Parse error for rows that failed parsing (returned from /confirm)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ParseError {
    pub row_index: u32,
    pub error: String,
    pub raw_values: Vec<String>,
}

// Analysis result (returned from /analyze endpoint)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CsvAnalysisResult {
    pub file_info: FileInfo,
    pub headers: Vec<String>,
    pub sample_rows: Vec<Vec<String>>,  // Raw data for UI preview (first 5-10 rows)
    pub ai_mapping: AiColumnMapping,
    pub warnings: Vec<ImportWarning>,
    // NOTE: No parsed_transactions - parsing happens on /confirm
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FileInfo {
    pub row_count: usize,
    pub detected_delimiter: char,
    pub has_header_row: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ImportWarning {
    pub warning_type: String,
    pub message: String,
}

// Import result (returned from /confirm endpoint)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CsvImportResult {
    pub total_rows: usize,
    pub imported_rows: usize,
    pub skipped_rows: usize,
    pub transaction_ids: Vec<Uuid>,
    pub parse_errors: Vec<ParseError>,
    pub categorization_job_id: Option<Uuid>,
}
```

---

### 3.4 Backend Changes

#### Remove Existing Adapters

Delete the following files/modules:
- `src/services/csv_formats/ing_bank.rs`
- `src/services/csv_formats/generic.rs`
- `src/services/csv_formats/registry.rs`

The `CsvFormatAdapter` trait and all adapters will be replaced with the AI-powered approach.

#### Keep & Refactor Utilities in Services Layer

Refactor `src/services/csv_formats/` → `src/services/csv_parsing/`:

| Keep | Refactor | Description |
|------|----------|-------------|
| `detection.rs` | Minor cleanup | `detect_delimiter()` - scoring algorithm |
| `amount_parsing.rs` | Minor cleanup | Amount parsing (unicode minus, European format) |
| - | New file | `date_parsing.rs` - date parsing for all DateFormat variants |
| `mod.rs` | Rewrite | Remove adapter exports, expose parsing functions |

These utilities are infrastructure concerns and belong in `services/`, not `domain/`.

#### Extend Domain Models

Add new types to existing `domain/transaction/models/csv.rs`:
- `AiColumnMapping` - AI response value object
- `ColumnMapping` - User-confirmed mapping
- `ImportOptions` - Import configuration
- `DateFormat` and `AmountFormat` enums
- `CsvAnalysisResult` - Analyze endpoint response
- `ImportResult` - Confirm endpoint response
- `ParseError`, `FileInfo`, `ImportWarning`

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

#### New Parsing Functions in `services/csv_parsing/`

```rust
// detection.rs (keep existing, minor cleanup)
pub fn detect_delimiter(csv_content: &str) -> char

// parsing.rs (new file)
/// Parse CSV and extract headers + sample rows
pub fn extract_csv_structure(csv_content: &str, delimiter: char)
    -> Result<(Vec<String>, Vec<Vec<String>>, usize), TransactionError>

/// Parse all rows using confirmed mapping
pub fn parse_all_rows(
    csv_content: &str,
    delimiter: char,
    headers: &[String],
    mapping: &ColumnMapping,
    options: &ImportOptions,
) -> Result<(Vec<Transaction>, Vec<ParseError>), TransactionError>

// date_parsing.rs (new file)
pub fn parse_date(value: &str, format: &DateFormat) -> Result<NaiveDate, TransactionError>

// amount_parsing.rs (keep existing)
pub fn parse_amount(value: &str, format: &AmountFormat) -> Result<Decimal, TransactionError>
```

#### New API Handlers

| Endpoint | Handler |
|----------|---------|
| `POST /import/analyze` | `analyze_csv_handler` - multipart file upload, AI call |
| `POST /import/confirm` | `confirm_import_handler` - multipart file + mapping, full parsing |

Remove old handlers:
- `preview_csv_import` (in `handlers.rs`)
- `confirm_csv_import_async` (in `handlers.rs`)

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
// Response from /analyze endpoint
interface CsvAnalysisResponse {
  file_info: {
    row_count: number;
    detected_delimiter: string;
    has_header_row: boolean;
  };
  headers: string[];
  sample_rows: string[][];  // Raw data for client-side preview
  ai_mapping: AiColumnMapping;
  warnings: ImportWarning[];
  // NOTE: No parsed_transactions - parsing happens on /confirm
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

// User-confirmed mapping (sent with /confirm request)
interface ColumnMapping {
  date_column: string;
  amount_column: string;
  merchant_column: string | null;
  description_column: string | null;
  currency_column: string | null;
  date_format: DateFormat;
  amount_format: AmountFormat;
}

interface ImportOptions {
  default_currency: string;
  skip_zero_amounts: boolean;
  invert_amounts: boolean;
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

// Response from /confirm endpoint
interface ImportResult {
  import_result: {
    total_rows: number;
    imported_rows: number;
    skipped_rows: number;
    transaction_ids: string[];
  };
  parse_errors: ParseError[];
  categorization_job_id: string;
}

type DateFormat = 'YYYY-MM-DD' | 'DD/MM/YYYY' | 'MM/DD/YYYY' | 'DD.MM.YYYY' | 'DD-MM-YYYY';
type AmountFormat = 'standard' | 'european';
```

#### New API Service: `src/services/csvImportService.ts`

```typescript
export const csvImportService = {
  // Step 1: Upload file, get AI mapping + sample rows
  analyzeCsv: async (file: File): Promise<CsvAnalysisResponse> => {
    const formData = new FormData();
    formData.append('file', file);
    return api.post('/import/analyze', formData);
  },

  // Step 2: Re-upload file with confirmed mapping, parse and create transactions
  confirmImport: async (
    file: File,
    mapping: ColumnMapping,
    options: ImportOptions
  ): Promise<ImportResult> => {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('mapping', JSON.stringify(mapping));
    formData.append('options', JSON.stringify(options));
    return api.post('/import/confirm', formData);
  },
};
```

#### State Management

| Hook | Purpose |
|------|---------|
| `useCsvImportWizard` | Custom hook for wizard state (step, file, mapping, options) |

```typescript
function useCsvImportWizard() {
  const [step, setStep] = useState<'upload' | 'review' | 'importing' | 'complete'>('upload');
  const [file, setFile] = useState<File | null>(null);
  const [analysisResult, setAnalysisResult] = useState<CsvAnalysisResponse | null>(null);
  const [mapping, setMapping] = useState<ColumnMapping | null>(null);
  const [options, setOptions] = useState<ImportOptions>(defaultOptions);

  const handleAnalyze = async (uploadedFile: File) => {
    setFile(uploadedFile);
    const result = await csvImportService.analyzeCsv(uploadedFile);
    setAnalysisResult(result);
    // Initialize mapping from AI suggestion
    setMapping({
      date_column: result.ai_mapping.date_column,
      amount_column: result.ai_mapping.amount_column,
      merchant_column: result.ai_mapping.merchant_column,
      description_column: result.ai_mapping.description_column,
      currency_column: result.ai_mapping.currency_column,
      date_format: result.ai_mapping.date_format,
      amount_format: result.ai_mapping.amount_format,
    });
    setStep('review');
  };

  const handleConfirm = async () => {
    if (!file || !mapping) return;
    setStep('importing');
    // Re-upload file with confirmed mapping
    const result = await csvImportService.confirmImport(file, mapping, options);
    setStep('complete');
    return result;
  };

  // User can modify mapping in UI - no server call needed
  const handleMappingChange = (newMapping: ColumnMapping) => {
    setMapping(newMapping);
    // Preview updates client-side using sample_rows + new mapping
  };

  return {
    step,
    file,
    analysisResult,
    mapping,
    options,
    setOptions,
    handleAnalyze,
    handleConfirm,
    handleMappingChange
  };
}
```

#### Client-Side Preview Logic

The preview table renders sample rows using the current mapping. When user changes mapping in dropdowns, preview updates instantly without server calls:

```typescript
function applyMappingToRow(
  row: string[],
  headers: string[],
  mapping: ColumnMapping
): PreviewTransaction | null {
  const getColumnValue = (columnName: string | null) => {
    if (!columnName) return null;
    const index = headers.indexOf(columnName);
    return index >= 0 ? row[index] : null;
  };

  const dateStr = getColumnValue(mapping.date_column);
  const amountStr = getColumnValue(mapping.amount_column);

  if (!dateStr || !amountStr) return null;

  return {
    date: parseDate(dateStr, mapping.date_format),
    amount: parseAmount(amountStr, mapping.amount_format),
    merchant: getColumnValue(mapping.merchant_column),
    description: getColumnValue(mapping.description_column),
    currency: getColumnValue(mapping.currency_column) || options.default_currency,
  };
}
```

---

### 3.6 BFF Changes

#### Route Updates

| Route | Content Type | Action |
|-------|--------------|--------|
| `POST /v1/users/{user_id}/transactions/import/analyze` | multipart/form-data | Proxy to transaction-service (file upload, AI call) |
| `POST /v1/users/{user_id}/transactions/import/confirm` | multipart/form-data | Proxy to transaction-service (file upload, parsing) |

Remove old routes:
- `POST /import/preview`
- `POST /import/confirm-async`

#### Considerations
- Handle multipart/form-data forwarding for both endpoints
- Set appropriate timeout for analyze endpoint (AI call may take 5-10 seconds)
- Set appropriate timeout for confirm endpoint (parsing large files may take time)

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
| **AI service unavailable** | API timeout/error | Return error with clear message, user must retry |
| **Missing required fields** | AI returns null for date/amount | Return error with clear message |
| **User overrides mapping** | User changes column selection in UI | Preview updates client-side, confirmed mapping sent on /confirm |

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

1. **Extend Domain Models**
   - Add new types to `domain/transaction/models/csv.rs`
   - `AiColumnMapping`, `ColumnMapping`, `ImportOptions`, etc.
   - `DateFormat` and `AmountFormat` enums

2. **Refactor CSV Utilities**
   - Rename `services/csv_formats/` → `services/csv_parsing/`
   - Keep `detection.rs` and `amount_parsing.rs`
   - Add `date_parsing.rs` for DateFormat variants
   - Add `parsing.rs` with `parse_all_rows()` function
   - Remove adapter files (`ing_bank.rs`, `generic.rs`, `registry.rs`)

3. **Extend AI Service**
   - Add `analyze_csv_columns` method to `AiService` trait
   - Implement in `AiUtilsService`
   - Add response validation (ensure columns exist in headers)

4. **Create New API Endpoints**
   - `POST /import/analyze` - upload file, call AI, return sample + mapping
   - `POST /import/confirm` - upload file + mapping, parse all rows, create transactions

5. **Remove Old Implementation**
   - Remove old handlers (`preview_csv_import`, `confirm_csv_import_async`)
   - Update tests

### Phase 2: Frontend Integration
**Scope**: New wizard UI

1. **Create Types and Services**
   - Add types in `src/types/csvImport.ts`
   - Add `csvImportService.ts` with analyze/confirm methods

2. **Create Import Wizard**
   - `CsvImportWizard` container with step management
   - `ColumnMappingReview` with AI suggestions + override dropdowns
   - `MappingPreviewTable` with client-side preview rendering
   - `ImportOptionsForm` for currency, skip options
   - `useCsvImportWizard` custom hook for state management

3. **Client-Side Preview**
   - Implement date/amount parsing utilities for preview
   - Preview updates instantly when user changes mapping

4. **Replace Existing Flow**
   - Update transactions page to use new wizard
   - Remove old import components
   - Error handling with snackbar notifications

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

- [ ] Extend domain models in `domain/transaction/models/csv.rs`
  - [ ] `AiColumnMapping` value object
  - [ ] `ColumnMapping` value object
  - [ ] `ImportOptions` value object
  - [ ] `DateFormat` and `AmountFormat` enums
  - [ ] `ParseError` struct
  - [ ] `CsvAnalysisResult` struct
  - [ ] `ImportResult` struct
  - [ ] `FileInfo` and `ImportWarning` structs
- [ ] Refactor `services/csv_formats/` → `services/csv_parsing/`
  - [ ] Keep `detection.rs` (minor cleanup)
  - [ ] Keep `amount_parsing.rs` (minor cleanup)
  - [ ] Create `date_parsing.rs` with `parse_date()` for all DateFormat variants
  - [ ] Create `parsing.rs` with `extract_csv_structure()` and `parse_all_rows()`
  - [ ] Update `mod.rs` to expose new API
  - [ ] Delete `ing_bank.rs`
  - [ ] Delete `generic.rs`
  - [ ] Delete `registry.rs`
- [ ] Extend `AiService` trait with `analyze_csv_columns()` method
- [ ] Implement AI column analysis in `AiUtilsService`
  - [ ] Build prompt with headers + sample rows
  - [ ] Parse and validate AI response
  - [ ] Ensure suggested columns exist in actual headers
  - [ ] Handle AI errors gracefully
- [ ] Create API handlers
  - [ ] `analyze_csv_handler` - multipart upload, AI call, return sample + mapping
  - [ ] `confirm_import_handler` - multipart upload with mapping/options, parse all rows, create transactions
- [ ] Remove old handlers
  - [ ] Delete `preview_csv_import` handler
  - [ ] Delete `confirm_csv_import_async` handler
- [ ] Add OpenAPI documentation with `utoipa`
- [ ] Update/add tests
  - [ ] Unit tests for date parsing
  - [ ] Unit tests for amount parsing
  - [ ] Integration tests for new endpoints
  - [ ] AI response validation tests

### Frontend

- [ ] Create types in `src/types/csvImport.ts`
- [ ] Create `csvImportService.ts`
- [ ] Create `useCsvImportWizard` hook
- [ ] Implement client-side preview utilities
  - [ ] Date parsing for preview
  - [ ] Amount parsing for preview
  - [ ] `applyMappingToRow()` function
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
- [ ] Add route for `/import/confirm` (multipart forwarding)
- [ ] Remove old `/import/preview` and `/import/confirm-async` routes
- [ ] Configure multipart forwarding for both endpoints
- [ ] Set appropriate timeouts (15 seconds for AI, longer for large file parsing)

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
3. **Bank Format Templates** - Community-contributed mapping templates
4. **Import History** - Track past imports with rollback capability
5. **Server-Side File Storage** - File re-uploaded on confirm instead of stored server-side (simpler, stateless)
