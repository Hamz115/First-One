# PDF Processing Strategy: Vision AI Approach

## Overview

This document outlines the strategy for implementing PDF processing capabilities in the Argus financial analyst application. After removing traditional PDF parsing methods (table detection, text extraction) due to reliability issues, we've adopted a **Vision AI-based approach** using local models to extract structured data from PDF documents converted to images.

### Context
- **Previous Approach**: Traditional table parsing with libraries like pdfplumber, camelot
- **Issues**: Inconsistent table detection, complex layout handling, poor reliability
- **New Approach**: Convert PDF pages to images → Process with Vision AI models → Extract structured JSON data
- **Key Advantage**: Better handling of complex layouts, visual table structures, and mixed content types

### Important: Existing Parsers Remain Untouched
**CRITICAL**: The Excel/CSV parsing functionality and existing parsers (`src/parsers/financial_statement_parser.py`, `src/parsers/csv_parser.py`) should **NOT** be modified or touched. This PDF processing capability is being added as a **separate, additional feature** that works alongside the existing parsers without interfering with their functionality.

## Document Type Analysis

### 1. Financial Statements (e.g., Watani Iron Steel Company)

**Structure**: 41-page document with distinct sections
- **Pages 3-7**: Auditor's Report (critical for reliability assessment)
- **Pages 8-11**: Core Financial Tables
  - Balance Sheet (Statement of Financial Position)
  - Income Statement (Statement of Profit or Loss)
  - Statement of Changes in Equity
  - Cash Flow Statement
- **Pages 12-34**: Notes to Financial Statements
- **Pages 35-41**: Additional disclosures and signatures

**Complexity**: High
- Nested table structures
- Multiple year comparisons
- Complex hierarchical labels
- Cross-references between sections
- Subtotals and calculated fields

### 2. Bank Transaction Statements (e.g., SAB Account Statement)

**Structure**: 10-page document with consistent format
- **Page 1**: Account information header
- **Pages 2-10**: Transaction listings
  - Date, Description, Debit, Credit, Running Balance
  - Sequential chronological data
  - Simple tabular format

**Complexity**: Low
- Flat table structure
- Consistent column headers
- Sequential data flow
- No nested hierarchies

## Vision AI Workflow with Qwen2.5-VL

### Phase 1: Document Type Detection

```
PDF Input → First Page Text Extraction → Type Classification:

Financial Statement Keywords:
- "Statement of Financial Position"
- "Statement of Profit or Loss" 
- "Auditor's Report"
- "Notes to Financial Statements"

Bank Statement Keywords:
- "Account Statement"
- "Transaction History"
- "Opening Balance"
- "Closing Balance"
```

### Phase 2: Page Classification & Processing

#### Financial Statements Processing

```
For each page:
1. Text extraction to identify section type
2. Route to appropriate processor:

   a) Core Financial Tables (Pages 8-11):
      → Convert to image (pdf2image)
      → Send to Qwen2.5-VL with specialized prompt
      → Extract structured financial data as JSON
      
   b) Auditor's Report (Pages 3-7):
      → Text extraction
      → Store in vector database for RAG queries
      → Flag audit opinion (qualified/unqualified)
      
   c) Notes Section (Pages 12-34):
      → Text extraction with section headers
      → Chunk into semantic segments
      → Store in vector database for context
```

#### Bank Statements Processing

```
For each page:
1. Check for transaction data presence
2. If transactions found:
   → Convert to image (pdf2image)
   → Send to Qwen2.5-VL with transaction extraction prompt
   → Validate running balance continuity
   → Store as structured transaction records
```

### Phase 3: Vision AI Model Integration

**Model**: Qwen2.5:7b-vl (locally installed via Ollama)

**Prompting Strategy**:

*For Financial Tables*:
```
Extract all financial data from this statement page as structured JSON.
Include: line items, current year values, prior year values, section totals.
Preserve the hierarchical structure of assets, liabilities, equity.
Return empty object if no financial table is present.
```

*For Bank Transactions*:
```
Extract all bank transactions from this statement as JSON array.
Include: date, description, debit amount, credit amount, running balance.
Ensure amounts are properly classified as debit/credit.
Return empty array if no transactions found.
```

## Technical Implementation

### Data Structures

#### Financial Statement Output
```json
{
  "document_type": "financial_statement",
  "company_info": {
    "name": "Watani Iron Steel Company",
    "period": "2024",
    "audit_status": "Unqualified Opinion"
  },
  "balance_sheet": {
    "assets": {
      "current": {
        "cash_and_equivalents": {"current": 1500000, "prior": 1200000},
        "total": {"current": 5000000, "prior": 4500000}
      },
      "non_current": {...},
      "total": {"current": 15000000, "prior": 14000000}
    },
    "liabilities": {...},
    "equity": {...}
  },
  "income_statement": {...},
  "cash_flow": {...},
  "ratios": {
    "current_ratio": 2.5,
    "debt_to_equity": 0.6,
    "roe": 15.2
  }
}
```

#### Bank Statement Output
```json
{
  "document_type": "bank_statement",
  "account_info": {
    "account_number": "****1234",
    "period": "13/06/2025 to 13/07/2025",
    "currency": "SAR",
    "bank": "Saudi Awwal Bank"
  },
  "transactions": [
    {
      "date": "2025-06-13",
      "description": "ATM WITHDRAWAL - RIYADH",
      "debit": 500.00,
      "credit": null,
      "balance": 4500.00,
      "reference": "ATM001234"
    },
    {
      "date": "2025-06-14",
      "description": "SALARY CREDIT",
      "debit": null,
      "credit": 8000.00,
      "balance": 12500.00,
      "reference": "SAL202406"
    }
  ],
  "summary": {
    "opening_balance": 5000.00,
    "closing_balance": 12500.00,
    "total_credits": 15000.00,
    "total_debits": 7500.00,
    "transaction_count": 45
  }
}
```

### Processing Efficiency Comparison

| Aspect | Financial Statements | Bank Statements |
|--------|---------------------|-----------------|
| **Pages requiring Vision AI** | 4-5 pages (tables only) | All transaction pages |
| **Processing Complexity** | High (nested structures) | Low (flat tables) |
| **Validation Requirements** | Cross-totals, ratio checks | Balance continuity |
| **Additional Processing** | Notes via RAG | Transaction categorization |
| **Model Prompt Complexity** | High (financial context) | Low (simple extraction) |
| **Expected Accuracy** | 85-90% (complex layouts) | 95%+ (simple tables) |

### Integration with Existing Architecture

#### Parser Factory Pattern
```python
def create_pdf_parser(pdf_path: str) -> BasePDFParser:
    doc_type = detect_document_type(pdf_path)
    
    if doc_type == "financial_statement":
        return FinancialStatementPDFParser()  # NEW - PDF-specific parser
    elif doc_type == "bank_statement":
        return BankStatementPDFParser()       # NEW - PDF-specific parser
    else:
        return GenericPDFParser()             # NEW - PDF-specific parser

# NOTE: Existing Excel/CSV parsers remain completely separate:
# - src/parsers/financial_statement_parser.py (Excel) - UNTOUCHED
# - src/parsers/csv_parser.py (Bank CSV) - UNTOUCHED
```

#### Vector Database Integration
- **Financial Tables**: Stored as structured data + searchable descriptions
- **Auditor Reports**: Chunked and stored for reliability context
- **Notes**: Semantic chunking for detailed accounting policy queries
- **Transactions**: Individual records + categorized summaries

#### Error Handling & Fallbacks
1. **Vision AI Failure**: Fall back to text extraction + manual validation
2. **Model Unavailable**: Queue for later processing, show warning
3. **Parse Errors**: Manual review queue with confidence scores
4. **Validation Failures**: Flag for user review with specific issues

## Implementation Phases

### Phase 1: Core Infrastructure
- PDF to image conversion (pdf2image)
- Ollama client for Qwen2.5-VL
- Document type detection
- Base parser classes

### Phase 2: Financial Statement Parser
- Page classification logic
- Vision AI integration for tables
- Text extraction for notes/reports
- Financial ratio calculations

### Phase 3: Bank Statement Parser
- Transaction extraction from images
- Balance validation
- Transaction categorization
- Summary statistics

### Phase 4: Integration & Testing
- Parser factory implementation
- Vector database storage
- End-to-end testing
- Performance optimization

## Example Processing Flows

### Watani Iron Steel Company (Financial Statement)
```
Input: 1941_0_2025-03-05_14-51-22_En (1).pdf (41 pages)
↓
Document Type Detection: "financial_statement"
↓
Page Classification:
- Pages 3-7: auditor_report → Text extraction + RAG storage
- Pages 8-11: financial_tables → Vision AI extraction
- Pages 12-34: notes → Text chunking + RAG storage
- Pages 35-41: disclosures → Text extraction
↓
Output: Structured financial data + searchable notes
```

### SAB Account Statement (Bank Statement)
```
Input: Account Statement_13-07-2025.pdf (10 pages)
↓
Document Type Detection: "bank_statement"
↓
Page Processing:
- Page 1: account_info → Text extraction
- Pages 2-10: transactions → Vision AI extraction
↓
Validation: Running balance continuity check
↓
Output: Transaction records + account summary
```

## Benefits of Vision AI Approach

### Advantages
1. **Layout Independence**: Handles any table format or visual structure
2. **Context Awareness**: Understands visual relationships between data
3. **Robustness**: Works with scanned, rotated, or poor-quality PDFs
4. **Scalability**: Same model handles different document types
5. **Local Processing**: Complete privacy, no cloud dependencies

### Considerations
1. **Processing Time**: Slower than pure text extraction
2. **Model Requirements**: Needs Qwen2.5-VL model (~4GB)
3. **Accuracy Tuning**: Requires prompt optimization
4. **Hardware**: Benefits from GPU acceleration

## Success Metrics

### Accuracy Targets
- **Financial Statements**: 90% field extraction accuracy
- **Bank Statements**: 95% transaction extraction accuracy
- **Document Classification**: 98% type detection accuracy

### Performance Targets
- **Financial Statement**: <30 seconds per document
- **Bank Statement**: <2 seconds per page
- **Memory Usage**: <2GB during processing

### Quality Assurance
- Cross-validation with existing CSV parsers
- Manual review for edge cases
- Confidence scoring for each extraction
- User feedback integration for continuous improvement

---

*This strategy serves as the technical foundation for implementing robust PDF processing capabilities that can handle the complexity and variety of financial documents while maintaining the privacy-first, local-processing approach of the Argus application.*
