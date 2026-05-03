Automated Receipt Processing with Tax Categorization using PDF Vector & Google Drive

https://n8nworkflows.xyz/workflows/automated-receipt-processing-with-tax-categorization-using-pdf-vector---google-drive-8501


# Automated Receipt Processing with Tax Categorization using PDF Vector & Google Drive

### 1. Workflow Overview

This workflow automates the processing of receipts and categorizes expenses for tax purposes. It is designed primarily for businesses or individuals who want to streamline expense management by extracting detailed receipt data from images or PDFs, categorizing expenses according to tax rules, validating data accuracy, and logging the processed results into a spreadsheet for tracking and accounting integration.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Handles receipt inputs by monitoring and retrieving files from Google Drive.
- **1.2 Data Extraction:** Uses PDF Vector AI-powered extraction to parse detailed receipt information from images or PDFs.
- **1.3 Tax Categorization:** Applies AI-driven classification to categorize expenses and determine deductibility.
- **1.4 Expense Processing:** Validates and processes extracted data, applies tax logic, and prepares the expense record.
- **1.5 Data Persistence:** Saves the processed expense data into a Google Sheets spreadsheet for record-keeping and further accounting workflows.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually and retrieves receipt files stored on Google Drive. It supports various input formats including photos, scanned PDFs, and email forwards, designed to handle even poor-quality images with OCR enhancement.

- **Nodes Involved:**  
  - Manual Trigger  
  - Google Drive - Get Receipt

- **Node Details:**

  - **Manual Trigger**  
    - Type: Manual trigger node  
    - Configuration: No parameters; triggers the workflow manually to process a receipt.  
    - Inputs: None  
    - Outputs: Connects to "Google Drive - Get Receipt"  
    - Failure modes: None, as it is a manual trigger.

  - **Google Drive - Get Receipt**  
    - Type: Google Drive node (download file)  
    - Configuration: Downloads a file based on `fileId` provided dynamically (`={{ $json.fileId }}`).  
    - Inputs: From Manual Trigger, expects `fileId` in input JSON.  
    - Outputs: Binary file data passed to next node.  
    - Failure modes:  
      - Invalid or missing `fileId` causing download failure.  
      - Authentication errors if Google Drive credentials expire or are misconfigured.  
      - Network or API rate limit errors.

---

#### 1.2 Data Extraction

- **Overview:**  
  This block extracts detailed receipt data from the retrieved file using PDF Vector, an AI-powered document processing node. It extracts merchant info, transaction details, line items, financials, payment, and loyalty data.

- **Nodes Involved:**  
  - PDF Vector - Extract Receipt

- **Node Details:**

  - **PDF Vector - Extract Receipt**  
    - Type: PDF Vector extraction node  
    - Configuration:  
      - Prompt instructs extraction of comprehensive receipt details, including merchant, items, totals, taxes, payment method, and loyalty info.  
      - Uses OCR automatically for scanned or image receipts.  
      - Input type: binary file from Google Drive node.  
      - Output: JSON structured receipt data according to a strict schema (merchant, transaction, items, financials, payment, loyalty).  
    - Inputs: Binary file data from Google Drive node.  
    - Outputs: Parsed JSON data passed to Tax Categorization node.  
    - Failure modes:  
      - Poor image quality affecting OCR accuracy.  
      - AI extraction errors or schema mismatches.  
      - Timeout or API request limits.  
      - Missing critical fields causing downstream errors.

---

#### 1.3 Tax Categorization

- **Overview:**  
  This block uses PDF Vector AI to classify the receipt into appropriate tax categories and determine deductibility percentages and special tax considerations.

- **Nodes Involved:**  
  - PDF Vector - Tax Categorization

- **Node Details:**

  - **PDF Vector - Tax Categorization**  
    - Type: PDF Vector question/answer node  
    - Configuration:  
      - Prompt asks for tax category classification (meals, travel, supplies, equipment, etc.), deductibility, and special tax notes.  
      - Uses OCR if needed for image input.  
      - Input: The same binary receipt file from Google Drive node.  
      - Output: Textual AI-generated classification answer.  
    - Inputs: Binary file (same as extraction node input).  
    - Outputs: Text categorization result passed to expense processing code node.  
    - Failure modes:  
      - Ambiguous or incorrect AI classification.  
      - API errors or downtime.  
      - Input file issues causing misclassification.

---

#### 1.4 Expense Processing

- **Overview:**  
  This block processes and validates the extracted receipt data alongside the AI tax categorization. It checks for discrepancies, assigns expense categories, calculates deductible amounts, and prepares a structured expense record.

- **Nodes Involved:**  
  - Process Expense Data (Code node)

- **Node Details:**

  - **Process Expense Data**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Reads receipt data and AI tax category from previous nodes.  
      - Validates subtotal against sum of item totals (tolerance 0.02).  
      - Checks tax amount consistency (tolerance 0.02).  
      - Assigns expense category based on tax categorization text (meals, travel, supplies, or other).  
      - Applies typical deductions, e.g., 50% for meals.  
      - Constructs a processed expense JSON object with metadata, validation results, and timestamps.  
    - Inputs: JSON from "PDF Vector - Extract Receipt" and text answer from "PDF Vector - Tax Categorization" nodes.  
    - Outputs: Processed expense JSON to "Save to Expense Sheet" node.  
    - Failure modes:  
      - Missing or malformed input JSON.  
      - Unexpected tax category text causing miscategorization.  
      - Date parsing errors.  
      - Validation errors logged but do not halt workflow.

---

#### 1.5 Data Persistence

- **Overview:**  
  Saves the finalized expense data into a Google Sheets spreadsheet, appending a row with key expense details for tracking and further integration.

- **Nodes Involved:**  
  - Save to Expense Sheet (Google Sheets node)

- **Node Details:**

  - **Save to Expense Sheet**  
    - Type: Google Sheets node (append operation)  
    - Configuration:  
      - Appends a row with date, merchant, category, amount, deductible amount, deductible percentage, tax notes, and processing timestamp.  
      - Uses dynamic sheet ID based on tax year (`{{ $json.taxYear }}-expenses`).  
      - Range set to columns A:H.  
    - Inputs: JSON from "Process Expense Data" node.  
    - Outputs: None (terminal node).  
    - Failure modes:  
      - Google Sheets API authentication or permission errors.  
      - Sheet not existing or named incorrectly.  
      - Data type mismatches causing append failure.

---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                      | Input Node(s)             | Output Node(s)                | Sticky Note                                                                                  |
|----------------------------|---------------------------|------------------------------------|---------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| Receipt Overview           | Sticky Note               | Provides high-level workflow summary | None                      | None                         | ## 🧾 Receipt & Tax Tracker\nAutomated expense management:\n• **Monitors** receipt folder hourly\n• **Extracts** data from photos/PDFs\n• **Categorizes** for tax purposes\n• **Calculates** deductions\n• **Syncs** with QuickBooks |
| Input Sources              | Sticky Note               | Describes receipt input formats    | None                      | None                         | ## 📸 Receipt Input\nHandles all formats:\n• Phone photos\n• Scanned PDFs\n• Email forwards\n• Poor quality images\n\n💡 OCR enhancement |
| Tax Categories             | Sticky Note               | Explains tax categorization logic  | None                      | None                         | ## 💰 Tax Logic\n**Auto-categorizes:**\n• Travel expenses\n• Office supplies\n• Meals (50% deduction)\n• Utilities\n\n⚠️ Consult tax advisor! |
| Manual Trigger             | Manual Trigger            | Initiates workflow manually         | None                      | Google Drive - Get Receipt    | Process receipt                                                                             |
| Google Drive - Get Receipt | Google Drive              | Downloads receipt file from Drive  | Manual Trigger            | PDF Vector - Extract Receipt  | Retrieve receipt from Drive                                                                |
| PDF Vector - Extract Receipt | PDF Vector Extraction   | Extracts receipt data from file    | Google Drive - Get Receipt | PDF Vector - Tax Categorization | Extract receipt data                                                                        |
| PDF Vector - Tax Categorization | PDF Vector QA         | Categorizes expense for tax purposes | PDF Vector - Extract Receipt | Process Expense Data          | Categorize for taxes                                                                        |
| Process Expense Data       | Code                      | Validates, categorizes, and processes expense data | PDF Vector - Tax Categorization | Save to Expense Sheet          | Validate and categorize                                                                    |
| Save to Expense Sheet      | Google Sheets             | Logs processed expense to spreadsheet | Process Expense Data       | None                         | Track in spreadsheet                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `Manual Trigger`  
   - Purpose: To start the workflow manually.

2. **Create Google Drive node**  
   - Name: `Google Drive - Get Receipt`  
   - Operation: `download`  
   - Parameter: `fileId` set dynamically as `={{ $json.fileId }}`  
   - Connect: Output of `Manual Trigger` → Input of this node.  
   - Configure Google Drive OAuth2 credentials with access to the receipt folder.

3. **Create PDF Vector Extraction node**  
   - Name: `PDF Vector - Extract Receipt`  
   - Operation: `extract`  
   - Resource: `document`  
   - Input Type: `file` (binary data)  
   - Binary Property Name: `data`  
   - Prompt:  
     ```
     Extract all receipt information from this document or image including merchant name and address, transaction date and time, all items with descriptions and prices, subtotal, tax amount and rate, tip if applicable, total amount, payment method, and any loyalty or membership numbers. Use OCR if this is a scanned receipt or image.
     ```  
   - Schema: Use the provided JSON schema defining merchant, transaction, items, financial, payment, and loyalty properties.  
   - Connect: Output of `Google Drive - Get Receipt` → Input of this node.

4. **Create PDF Vector Tax Categorization node**  
   - Name: `PDF Vector - Tax Categorization`  
   - Operation: `ask`  
   - Resource: `document`  
   - Input Type: `file` (binary data)  
   - Binary Property Name: `data`  
   - Prompt:  
     ```
     Based on this receipt document or image, determine: 1) The most appropriate tax category (meals, travel, supplies, equipment, etc.), 2) Whether this is likely tax deductible for business, 3) If this is a meal receipt, what percentage would typically be deductible, 4) Any special considerations for tax purposes. Process any image format using OCR if needed.
     ```  
   - Connect: Output of `PDF Vector - Extract Receipt` → Input of this node.

5. **Create Code node**  
   - Name: `Process Expense Data`  
   - Language: JavaScript  
   - Code: Copy the provided JS code exactly, which:  
     - Reads JSON from extraction and tax categorization nodes  
     - Validates subtotals and tax amounts  
     - Determines expense category and deductibility  
     - Constructs processed expense data with metadata and validation results  
   - Connect: Output of `PDF Vector - Tax Categorization` → Input of this node.

6. **Create Google Sheets node**  
   - Name: `Save to Expense Sheet`  
   - Operation: `append`  
   - Sheet Range: `A:H`  
   - Sheet ID: Use expression `{{ $json.taxYear }}-expenses` to dynamically select sheet based on tax year.  
   - Data: Append array with fields:  
     - Date  
     - Merchant  
     - Expense Category  
     - Amount  
     - Deductible Amount  
     - Deductible Percentage (string with `%`)  
     - Tax Notes  
     - Processed Timestamp  
   - Connect: Output of `Process Expense Data` → Input of this node.  
   - Configure Google Sheets credentials with write access to the target spreadsheet.

7. **Optional: Add Sticky Notes**  
   - Add notes summarizing the purpose of each block as per the initial sticky notes to improve documentation and user understanding.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                |
|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow handles multiple receipt formats including photos and scanned PDFs with OCR enhancement. | Input Sources sticky note                                      |
| Tax categorization includes common categories like meals (50% deductible), travel, and supplies.     | Tax Categories sticky note                                     |
| Consult a tax advisor to verify and customize tax logic for your jurisdiction and business needs.    | Tax Categories sticky note                                     |
| The workflow syncs processed expenses with QuickBooks (mentioned in overview, requires external setup). | Receipt Overview sticky note                                   |
| Use Google Drive and Google Sheets credentials with appropriate access and scopes configured in n8n. | Credential requirement                                         |
| PDF Vector nodes require valid API access and can be sensitive to input quality; monitor for errors. | AI processing nodes                                            |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.