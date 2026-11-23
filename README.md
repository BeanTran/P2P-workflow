# P2P Automation: Purchase Order – Invoice Reconciliation & AI-Powered Journal Entry Suggestions

An AI-powered Procure-to-Pay (P2P) automation workflow built with **n8n, OpenAI, Gmail, and Google Sheets**.

The workflow automates document processing and preliminary accounting tasks, from extracting Purchase Order (PO) information to matching supplier invoices, identifying discrepancies, and recommending accounting journal entries.

---

## 1. Project Overview

### Business Problem

In traditional Accounts Payable (AP) processes, accountants often spend significant time on repetitive tasks:

- Reading Purchase Orders and supplier invoices manually.
- Extracting supplier information, tax details, quantities, and prices.
- Matching invoices against their corresponding POs.
- Identifying differences in quantities, unit prices, VAT, and total amounts.
- Determining appropriate debit and credit accounts.
- Recording extracted information into accounting tracking spreadsheets.

These manual activities can become time-consuming, particularly when processing a large volume of purchasing documents.

### Solution

This project uses n8n to orchestrate an AI-powered P2P workflow that connects document processing, financial data extraction, invoice reconciliation, and accounting recommendations.

Instead of manually processing each document, users can upload POs through a form and forward supplier invoices to a designated Gmail account.

The workflow then processes the documents, extracts relevant information, performs reconciliation, and records the results in Google Sheets.

### Key Capabilities

- Automated PO document ingestion (PDF and image).
- AI-powered document understanding and data extraction.
- Automated invoice processing through Gmail.
- PO-to-invoice matching using multiple identification methods.
- Line-item and financial reconciliation.
- AI-generated accounting journal entry suggestions.
- Centralized AP records and reconciliation results in Google Sheets.

---

## 2. Workflow Architecture

The automation consists of two main processes that converge into a shared reconciliation and accounting workflow.

### Process A: Purchase Order (PO) Processing

**Step 1 – PO Submission**

Users upload a Purchase Order through an n8n form.

Supported formats:
- PDF
- PNG
- JPG / JPEG

An optional notes field allows users to provide additional information.

**Step 2 – Document Processing**

The workflow identifies the uploaded file format and routes it accordingly:

- PDF: Extracts text directly from the document using n8n's Extract From File node.
- Image: Uses an OpenAI vision-enabled model to read and transcribe document content.

**Step 3 – AI Data Extraction**

OpenAI extracts structured information from the PO, including:

| Data Category | Extracted Information |
|---|---|
| PO Information | PO number, PO date |
| Supplier | Supplier name, tax identification number, address |
| Payment | Payment terms |
| Delivery | Delivery date, delivery location |
| Product Details | Item description, unit, quantity, unit price, line amount |
| Financial Information | Currency, subtotal, VAT rate, VAT amount, total payable |

The extracted information is converted into a structured JSON format.

**Step 4 – Data Storage**

The workflow standardizes the extracted information and appends it to the `PO` worksheet in Google Sheets.

Each PO is stored with its item-level details and an initial status of `Chờ hóa đơn` (Awaiting Invoice).

---

### Process B: Supplier Invoice Processing

**Step 1 – Automated Email Monitoring**

The Gmail Trigger monitors incoming unread emails every minute.

It searches for emails that:
- Contain attachments.
- Have subjects matching invoice-related keywords, including `hóa đơn`, `hoá đơn`, `invoice`, or `HĐ`.

The configured maximum result count is 20 emails per polling cycle.

**Step 2 – Attachment Processing**

The workflow downloads attachments and filters supported document formats:

- PDF
- PNG
- JPG / JPEG

Documents are routed through the appropriate processing branch:

- PDF: Direct text extraction.
- Image: AI-powered document reading and transcription.

**Step 3 – AI Invoice Data Extraction**

OpenAI extracts structured invoice information, including:

| Data Category | Extracted Information |
|---|---|
| Invoice Identification | Invoice number, invoice series, invoice date |
| Supplier | Supplier name, supplier tax ID |
| PO Reference | Referenced PO number, if available |
| Product Details | Description, unit, quantity, unit price, line amount |
| Tax Information | VAT rate, VAT amount |
| Financial Information | Currency, subtotal, total payable |
| Payment | Payment due date |

The AI is instructed to distinguish the supplier's tax identification number from the buyer's tax identification number.

The extracted information is converted into structured JSON for subsequent processing.

---

### Process C: Purchase Order – Invoice Reconciliation

This is the core control mechanism of the workflow.

**Step 1 – Retrieve Existing Purchase Orders**

The workflow retrieves PO records from the `PO` worksheet in Google Sheets.

**Step 2 – Identify the Corresponding PO**

A custom JavaScript node applies a prioritized matching approach:

1. **PO Number Matching:** Matches the PO number extracted from the invoice against the stored PO number.
2. **Supplier Tax ID Matching:** If no PO number match is found, searches for an open PO with the same supplier tax ID.
3. **Supplier Name Matching:** If the previous methods fail, normalizes supplier names to identify an equivalent supplier name.

Supplier name normalization removes:
- Vietnamese diacritics.
- Differences in uppercase and lowercase.
- Special characters and spacing.
- Common corporate identifiers such as TNHH and Công ty.

When multiple open POs match the same supplier, the workflow selects the latest matching PO in the retrieved records.

The matching method is also recorded for traceability.

**Step 3 – AI-Powered Financial Reconciliation**

The AI compares the invoice against the corresponding PO.

The reconciliation covers:

| Reconciliation Item | Comparison |
|---|---|
| Supplier | Supplier name and tax ID |
| Product / Service | Item descriptions and semantic similarity |
| Quantity | Ordered quantity vs. invoiced quantity |
| Unit Price | PO unit price vs. invoice unit price |
| Line Amount | PO line amount vs. invoice line amount |
| VAT Rate | PO VAT rate vs. invoice VAT rate |
| VAT Amount | PO VAT amount vs. invoice VAT amount |
| Subtotal | PO subtotal vs. invoice subtotal |
| Total Payable | PO total vs. invoice total |
| Currency | PO currency vs. invoice currency |

The AI applies the following reconciliation rules:

- Differences below VND 1,000 caused by tax rounding are treated as matching.
- Partial deliveries are classified as discrepancies and explicitly identified.
- Missing corresponding POs are classified as `Không tìm thấy PO`.
- Differences are explained with specific figures rather than simply returning a pass/fail result.

**Reconciliation Outcomes**

- `Khớp` – Matched.
- `Lệch` – Discrepancy identified.
- `Không tìm thấy PO` – No corresponding PO found.

The workflow also calculates the total difference:

Invoice Total – PO Total

A detailed explanation of discrepancies and item-level reconciliation results is generated.

---

### Process D: AI-Powered Accounting Journal Entry Suggestions

After reconciliation, the workflow retrieves the company's Chart of Accounts from the `HETHONGTAIKHOAN` worksheet.

**Step 1 – Retrieve the Chart of Accounts**

Google Sheets provides the accounting account codes and descriptions available to the AI.

**Step 2 – Determine Accounting Treatment**

OpenAI analyzes:

- Invoice line-item descriptions.
- The corresponding PO information.
- Reconciliation results.
- The company's existing Chart of Accounts.

The AI determines the likely accounting nature of purchased goods or services, such as:

- Merchandise.
- Raw materials.
- Tools and supplies.
- Fixed assets.
- Operating expenses.
- Other purchased services.

**Step 3 – Generate Journal Entry Suggestions**

The AI proposes debit and credit accounts based on the company's Chart of Accounts.

For example:

| Account | Description | Debit | Credit |
|---|---|---:|---:|
| 1561 | Merchandise inventory | 5,000,000 | - |
| 1331 | Deductible input VAT | 500,000 | - |
| 331 | Accounts payable | - | 5,500,000 |
| **Total** | | **5,500,000** | **5,500,000** |

*Illustrative example based on the workflow's structured output schema.*

The AI follows several accounting constraints:

- Only uses account codes available in the company's Chart of Accounts.
- Separates the purchase amount from input VAT where applicable.
- Ensures that the proposed journal entry balances to the invoice's total payable.
- Assigns a confidence level: High, Medium, or Low.
- Provides an explanation of the proposed accounting treatment.

**Important:** These are AI-generated recommendations, not automatically posted accounting entries. Accountant review and approval are required before recording them in the accounting system.

---

## 3. Data Storage & Output

All processed information is centralized in Google Sheets.

The workflow uses three main worksheets:

### 1. PO

Stores extracted Purchase Order information.

Key fields:
- PO number and date.
- Supplier name and tax ID.
- Payment terms.
- Delivery information.
- Item descriptions and quantities.
- Unit prices and line amounts.
- VAT and total payable.
- Import date and processing status.

### 2. HOADON

Stores invoice information and processing results.

Key fields:
- Invoice number, series, and date.
- Supplier information.
- Invoice line items.
- Subtotal, VAT, and total payable.
- Matched PO number.
- PO matching method.
- Reconciliation status.
- Total discrepancy.
- Detailed discrepancy explanation.
- Item-level reconciliation results.
- Proposed journal entries.
- Debit and credit account codes.
- Accounting rationale.
- AI confidence level.

### 3. HETHONGTAIKHOAN

Contains the company's Chart of Accounts used as the reference for AI-generated accounting recommendations.

---

## 4. Technology Stack

| Technology | Purpose |
|---|---|
| n8n | Workflow orchestration and process automation |
| OpenAI GPT-5.4-mini | Document understanding, structured extraction, reconciliation, and accounting recommendations |
| Gmail API | Automated invoice email monitoring and attachment retrieval |
| Google Sheets API | Centralized PO, invoice, and accounting reference data |
| JavaScript | File normalization, data transformation, supplier matching, and business logic |
| Structured Output Parser | Enforces structured JSON outputs from AI processing |

### Technical Components

The workflow combines several n8n capabilities:

- Form Trigger
- Gmail Trigger
- Code Node
- IF Node
- Extract From File
- Set Node
- Google Sheets
- LangChain LLM Chain
- OpenAI Chat Model
- Structured Output Parser
- Aggregate Node

---

## 5. Setup & Configuration

To deploy this workflow, the following components must be configured.

### Prerequisites

- An active n8n instance.
- An OpenAI API connection.
- A Gmail account with appropriate OAuth permissions.
- A Google account with Google Sheets API access.
- A Google Spreadsheet containing the required worksheets.

### Required Google Sheets Structure

Create the following worksheets:

1. `PO`
2. `HOADON`
3. `HETHONGTAIKHOAN`

Ensure that the column headers in the spreadsheet match the fields expected by the workflow.

### Credentials

Configure the following credentials in n8n:

- OpenAI API.
- Gmail OAuth2.
- Google Sheets OAuth2.

### Deployment Steps

1. Import the workflow JSON into n8n.
2. Configure OpenAI, Gmail, and Google Sheets credentials.
3. Update the Google Spreadsheet reference to your own spreadsheet.
4. Prepare the three worksheets and their column headers.
5. Verify the Gmail search filters and attachment permissions.
6. Activate the workflow.
7. Submit a test PO through the n8n form.
8. Send a test invoice email with a PDF or image attachment.
9. Review the extracted information, reconciliation results, and proposed journal entries in Google Sheets.

---

## 6. Controls, Limitations & Further Improvements

This project is designed as an AI-assisted accounting workflow. It reduces repetitive processing tasks while retaining human oversight over accounting decisions.

### Current Workflow Limitations

- AI extraction accuracy depends on document quality, layout, and readability.
- Text extraction from PDF files may not work reliably for scanned PDFs without an embedded text layer.
- Supplier-based PO matching can be ambiguous when multiple open POs exist.
- The workflow does not currently implement a dedicated duplicate-invoice detection mechanism.
- The configured Gmail polling cycle has a maximum result count of 20 emails.
- AI-generated accounting entries require professional review.
- The workflow does not automatically post journal entries into an ERP or accounting system.

### Potential Improvements

- Implement duplicate invoice detection using invoice number, supplier tax ID, and invoice date.
- Add human approval steps for discrepancies and accounting recommendations.
- Improve PO matching using delivery status, dates, and remaining quantities.
- Add exception handling, retry mechanisms, and processing logs.
- Integrate with ERP and accounting systems.
- Implement scalable batch processing for high-volume document processing.
- Add automated validation for accounting balances and tax calculations.

---

## 7. Business Value

The project demonstrates how AI and workflow automation can support accounting operations beyond simple data entry.

Key benefits include:

- **Reduced manual data entry:** Automatically extracts structured information from purchasing documents.
- **Improved AP efficiency:** Streamlines invoice processing and PO reconciliation.
- **Stronger financial controls:** Highlights discrepancies in supplier details, quantities, pricing, VAT, and total amounts.
- **Accounting assistance:** Provides preliminary journal entry recommendations based on the company's Chart of Accounts.
- **Better traceability:** Centralizes document information, matching methods, reconciliation results, and accounting suggestions.
- **Scalable architecture:** Provides a foundation for further automation of the Procure-to-Pay process.

---

## 8. Project Objective

The objective of this project is to explore the practical application of AI and workflow automation in accounting operations, particularly in Accounts Payable and Procure-to-Pay processes.

By combining accounting knowledge, structured financial data, and AI-driven document processing, the workflow demonstrates an approach to transforming repetitive accounting tasks into a more efficient, standardized, and reviewable process.

**Disclaimer:** This is an independent automation project for demonstration and learning purposes. AI-generated outputs are preliminary suggestions and should not replace professional accounting judgment or formal internal approval procedures.

---

**Project:** P2P – Purchase Order & Invoice Reconciliation

**Built with:** n8n | OpenAI | Gmail | Google Sheets | JavaScript

**Focus Areas:** Accounts Payable | Procure-to-Pay | Accounting Automation | AI Workflow Orchestration | Financial Data Processing
