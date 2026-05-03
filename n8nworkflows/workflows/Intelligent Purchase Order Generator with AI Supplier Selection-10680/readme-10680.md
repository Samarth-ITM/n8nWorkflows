Intelligent Purchase Order Generator with AI Supplier Selection

https://n8nworkflows.xyz/workflows/intelligent-purchase-order-generator-with-ai-supplier-selection-10680


# Intelligent Purchase Order Generator with AI Supplier Selection

### 1. Workflow Overview

This workflow automates the end-to-end process of generating intelligent purchase orders (POs) with AI-powered supplier selection. It targets procurement teams seeking to optimize supplier choice based on cost, delivery speed, and item specialization while enforcing spending controls and approvals for high-value orders. The workflow receives purchase requests via webhook, validates and enriches the data, leverages AI to recommend the best supplier, routes orders for approval if over a threshold, generates professional PO documents as PDFs, archives them in Google Drive, sends the PO to the supplier by email, logs the order in the procurement system, and finally notifies the procurement team via Slack.

Logical blocks:

- **1.1 Input Reception & Validation:** Receive purchase order requests and validate the data structure and completeness.
- **1.2 AI Supplier Recommendation:** Prepare AI input context and use OpenAI (GPT-4) to recommend the optimal supplier.
- **1.3 Data Enrichment:** Parse AI output and enrich purchase order data with detailed supplier information and company context.
- **1.4 Approval Routing:** Determine if the order exceeds the approval threshold and route accordingly.
- **1.5 PO Document Generation:** Create an HTML purchase order document, convert it to PDF.
- **1.6 Archival and Distribution:** Archive the PDF in Google Drive, email it to the supplier, log the order in the procurement system, and notify the procurement team via Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:** Receives purchase order data via HTTP POST, validates required fields and item details, computes pricing totals, and sets defaults including PO number and approval necessity.
- **Nodes Involved:**  
  - Webhook Trigger  
  - Validate Request  
  - Sticky Note1 (documentation)

- **Node Details:**

  - **Webhook Trigger**  
    - Type: Webhook  
    - Role: Entry point receiving purchase order requests via HTTP POST at path `/purchase-order`.  
    - Configuration: POST method, no special options.  
    - Input: External HTTP request with JSON payload.  
    - Output: Emits JSON data for validation node.  
    - Edge Cases: Missing fields or invalid JSON in request; no explicit error handling beyond validation node.  
    - Version-specific: None.

  - **Validate Request**  
    - Type: Code (JavaScript)  
    - Role: Validates input JSON for required fields (`items`, `requestedBy`, `department`), ensures item array correctness, calculates line totals, subtotal, tax, shipping, total, and generates PO number if missing.  
    - Configuration: Custom JavaScript validates data, throws errors if missing or invalid. Defaults tax to 10% if not provided; default currency USD. Sets urgency default. Determines if approval required for totals over $5000.  
    - Key Expressions: Calculation of totals, default date formats, approval flag.  
    - Input: JSON from webhook.  
    - Output: Enriched JSON with validations and calculated fields.  
    - Edge Cases: Missing fields, malformed items, invalid quantities/prices, parsing errors. Throws error halting workflow if validation fails.  
    - Version-specific: Uses typeVersion 2 for code node.

  - **Sticky Note1**  
    - Type: Sticky Note (documentation)  
    - Role: Explains the data processing and AI analysis block.

#### 2.2 AI Supplier Recommendation

- **Overview:** Constructs a textual context summarizing the purchase order and supplier options, then uses OpenAI GPT-4 language model to generate a JSON-formatted recommendation for the optimal supplier.
- **Nodes Involved:**  
  - Prepare AI Context  
  - OpenAI Chat Model  
  - AI Supplier Selector  
  - Sticky Note (Sticky Note at webhook and section)

- **Node Details:**

  - **Prepare AI Context**  
    - Type: Set node  
    - Role: Creates a string summarizing purchase order details and supplier options for AI input.  
    - Configuration: Uses expressions to list items, totals, urgency, department, preferred supplier, budget limit, and static supplier descriptions.  
    - Input: Validated purchase order JSON.  
    - Output: JSON with a string property `aiContext`.  
    - Edge Cases: Missing optional fields replaced by defaults in expression.  
    - Version-specific: n8n v3.4 or higher for advanced expression support.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Node  
    - Role: Provides the underlying OpenAI GPT-4 model for AI processing.  
    - Configuration: Model set to `"gpt-4.1-mini"`, linked with OpenAI API credentials.  
    - Input: Receives prompt context to generate supplier recommendation.  
    - Output: AI response passed to next node.  
    - Edge Cases: API key invalid or rate limits; network timeouts; model errors.  
    - Version-specific: Requires configured OpenAI credentials.

  - **AI Supplier Selector**  
    - Type: LangChain Agent Node  
    - Role: Defines prompt instructions and formats AI output as strict JSON containing recommended supplier and reasoning.  
    - Configuration: System message sets expert persona; prompt includes purchase order context; expects valid JSON only.  
    - Input: AI chat model output.  
    - Output: Parsed AI recommendation JSON.  
    - Edge Cases: AI may produce invalid JSON or unexpected output; downstream node parses and defaults if necessary.

#### 2.3 Data Enrichment

- **Overview:** Parses AI JSON output, enriches data with detailed supplier contact info and company/shipping details, merges with original request data.
- **Nodes Involved:**  
  - Enrich with Supplier Data  
  - Sticky Note5 (documentation)

- **Node Details:**

  - **Enrich with Supplier Data**  
    - Type: Code (JavaScript)  
    - Role: Parses AI recommendation JSON safely, loads supplier database (hardcoded), merges supplier and company info with purchase order.  
    - Configuration: Includes fallback defaults if AI output cannot be parsed; enriches with company address, contact, shipping info, special instructions, and notes.  
    - Input: AI Supplier Selector output and original validated request.  
    - Output: Fully enriched purchase order JSON with AI recommendation and supplier details.  
    - Edge Cases: AI output parsing failure defaults to standard supplier; missing fields replaced with defaults.  
    - Version-specific: Code node typeVersion 2.

  - **Sticky Note5**  
    - Type: Sticky Note  
    - Role: Explains approval routing logic.

#### 2.4 Approval Routing

- **Overview:** Checks if order total exceeds approval threshold ($5000) and routes either to an email approval request or directly to PO generation.
- **Nodes Involved:**  
  - Needs Approval? (IF node)  
  - Request Approval (Gmail node)  
  - Generate PO Document (Code node)  
  - Sticky Note4 (documentation)

- **Node Details:**

  - **Needs Approval?**  
    - Type: IF node  
    - Role: Evaluates boolean condition `requiresApproval` set during validation.  
    - Configuration: If true, routes to Request Approval; else to Generate PO Document.  
    - Input: Enriched purchase order JSON.  
    - Output: Conditional branches.  
    - Edge Cases: If flag missing or malformed, default is false (no approval).  
    - Version-specific: None.

  - **Request Approval**  
    - Type: Gmail node  
    - Role: Sends an email to approvers requesting order approval for orders > $5k.  
    - Configuration: Uses OAuth2 credentials; parameters not detailed but expected to include email content with PO details.  
    - Input: Purchase order data from approval branch.  
    - Output: Email sent confirmation.  
    - Edge Cases: Authentication failure, SMTP errors, missing recipient.  
    - Version-specific: Gmail OAuth2 credentials required.

  - **Generate PO Document**  
    - Type: Code (JavaScript)  
    - Role: Constructs a styled HTML purchase order document including item details, supplier info, AI insights, and totals.  
    - Configuration: Uses template literals with embedded expressions for dynamic content. Currency formatting, conditional urgency badges, AI insights box, and company/shipment info included.  
    - Input: Purchase order JSON (enriched).  
    - Output: JSON containing `html` string, file name, supplier email, and other metadata for PDF conversion.  
    - Edge Cases: Missing supplier or AI data handled by conditional rendering; large orders with many items may affect performance.  
    - Version-specific: Code node typeVersion 2.

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Documents PO generation step.

#### 2.5 PO PDF Conversion and Archival

- **Overview:** Converts generated HTML PO to PDF, archives the PDF in Google Drive, emails it to supplier, and logs the order in procurement system.
- **Nodes Involved:**  
  - HTML to PDF  
  - Archive in Drive  
  - Email to Supplier (Gmail)  
  - Log in Procurement System (Code)  
  - Sticky Note10 (documentation)

- **Node Details:**

  - **HTML to PDF**  
    - Type: HTML CSS to PDF node  
    - Role: Converts HTML string from PO document node to a print-ready PDF format.  
    - Configuration: No special parameters shown. Requires HTML input from previous node.  
    - Input: HTML document JSON.  
    - Output: Binary PDF data with metadata.  
    - Edge Cases: Very large HTML may cause timeout; styling may behave differently in PDF.  
    - Version-specific: Requires API key from htmlcsstoimage.com service.

  - **Archive in Drive**  
    - Type: Google Drive node  
    - Role: Uploads generated PDF file to Google Drive root folder with dynamic file name.  
    - Configuration: Uses OAuth2 for Google Drive; folder is root "My Drive".  
    - Input: PDF binary from HTML to PDF node.  
    - Output: Drive file metadata including webViewLink.  
    - Edge Cases: Authentication failure, quota limits, network errors.  
    - Version-specific: Google Drive OAuth2 credentials required.

  - **Email to Supplier**  
    - Type: Gmail node  
    - Role: Sends email to supplier with PO PDF attachment.  
    - Configuration: Uses OAuth2 Gmail credentials. Email parameters not fully shown but expected to include supplier email and attached PDF.  
    - Input: PDF archive metadata and purchase order JSON.  
    - Output: Email sent confirmation.  
    - Edge Cases: Authentication errors, invalid email addresses, attachment size limits.  
    - Version-specific: Gmail OAuth2 required.

  - **Log in Procurement System**  
    - Type: Code (JavaScript)  
    - Role: Prepares a structured log object with PO details, status, AI recommendations, and activity log for external procurement system API or record keeping.  
    - Configuration: Builds JSON with all relevant fields, sets status as "Sent to Supplier", includes timestamps.  
    - Input: Confirmation and data from email node.  
    - Output: Enriched JSON with log and timestamp.  
    - Edge Cases: No actual API call shown; future integration may need HTTP request node.  
    - Version-specific: Code node typeVersion 2.

  - **Sticky Note10**  
    - Type: Sticky Note  
    - Role: Describes the distribution and tracking operations.

#### 2.6 Procurement Team Notification

- **Overview:** Sends a Slack notification summarizing the PO creation, including AI insights and Google Drive link for compliance and visibility.
- **Nodes Involved:**  
  - Notify Procurement Team (HTTP Request)  

- **Node Details:**

  - **Notify Procurement Team**  
    - Type: HTTP Request  
    - Role: Posts a formatted Slack message to a predefined Slack webhook URL with PO number, supplier, total, urgency, requester, AI reasoning, and Drive link.  
    - Configuration: JSON payload with Slack blocks, using expressions to insert data from prior nodes.  
    - Input: Procurement log data and Drive file URL.  
    - Output: Slack API response.  
    - Edge Cases: Invalid webhook URL, Slack API rate limits, network failures.  
    - Version-specific: Supports Slack block formatting.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                         | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                  |
|-------------------------|--------------------------------|---------------------------------------|--------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note             | Sticky Note                   | Workflow overview and instructions    |                          |                             | Explains workflow purpose, setup steps, and required data                                    |
| Webhook Trigger         | Webhook                      | Entry point receiving purchase orders |                          | Validate Request            |                                                                                              |
| Sticky Note1            | Sticky Note                   | Data validation and AI analysis intro | Webhook Trigger          | Validate Request            | Explains validation and AI analysis block                                                    |
| Validate Request        | Code                         | Validates order data, calculates totals| Webhook Trigger          | Prepare AI Context          |                                                                                              |
| Prepare AI Context      | Set                          | Builds AI input summary string         | Validate Request          | AI Supplier Selector        |                                                                                              |
| OpenAI Chat Model       | LangChain OpenAI Chat         | Provides GPT-4 language model           | AI Supplier Selector (agent) | AI Supplier Selector      |                                                                                              |
| AI Supplier Selector    | LangChain Agent               | Runs AI prompt and parses supplier rec | Prepare AI Context / OpenAI Chat Model | Enrich with Supplier Data |                                                                                              |
| Enrich with Supplier Data| Code                        | Parses AI JSON, adds supplier & company data | AI Supplier Selector    | Needs Approval?             |                                                                                              |
| Sticky Note5            | Sticky Note                   | Explains approval routing logic        | Enrich with Supplier Data | Needs Approval?             | Explains orders > $5k require email approval, others proceed directly                         |
| Needs Approval?         | IF                           | Routes based on approval requirement   | Enrich with Supplier Data | Request Approval / Generate PO Document |                                                                                              |
| Request Approval        | Gmail                        | Sends approval request email            | Needs Approval?           |                             |                                                                                              |
| Generate PO Document    | Code                         | Creates purchase order HTML document    | Needs Approval?           | HTML to PDF                 | Explains PO document generation                                                             |
| Sticky Note4            | Sticky Note                   | PO generation documentation             | Generate PO Document      | HTML to PDF                 |                                                                                              |
| HTML to PDF             | HTML CSS to PDF               | Converts HTML to PDF                    | Generate PO Document      | Archive in Drive            |                                                                                              |
| Archive in Drive        | Google Drive                 | Uploads PDF to Drive                     | HTML to PDF               | Email to Supplier           |                                                                                              |
| Email to Supplier       | Gmail                        | Emails PO PDF to supplier                | Archive in Drive          | Log in Procurement System   |                                                                                              |
| Log in Procurement System| Code                        | Logs PO details and activity             | Email to Supplier         | Notify Procurement Team     |                                                                                              |
| Sticky Note10           | Sticky Note                   | Explains distribution and tracking      | Log in Procurement System | Notify Procurement Team     |                                                                                              |
| Notify Procurement Team | HTTP Request                 | Sends Slack notification                 | Log in Procurement System |                             |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger node:**  
   - Type: Webhook  
   - Name: "Webhook Trigger"  
   - HTTP Method: POST  
   - Path: `purchase-order`  
   - No authentication required (adjust as needed).

2. **Add Code node "Validate Request":**  
   - Type: Code (JavaScript)  
   - Connect input from "Webhook Trigger".  
   - Paste validation and calculation script:  
     - Check required fields: `items`, `requestedBy`, `department`.  
     - Validate each item has `productName` and `quantity`.  
     - Convert quantities and prices to float, calculate totals.  
     - Generate PO number if missing (format: PO-YYYYMM-XXXX).  
     - Set defaults for currency, symbol, urgency, dates.  
     - Determine approval requirement if total > 5000.  
   - Node version: 2.

3. **Add Set node "Prepare AI Context":**  
   - Connect input from "Validate Request".  
   - Create one string field `aiContext` with expression that formats:  
     - List of items with product name, quantity, category.  
     - Total value with currency symbol.  
     - Urgency, department, preferred supplier, budget limit.  
     - Static description of available suppliers.  
   - Node version: 3.4 (enables advanced expressions).

4. **Add LangChain OpenAI Chat model node:**  
   - Connect input from "Prepare AI Context".  
   - Set model to `"gpt-4.1-mini"` or GPT-4 equivalent.  
   - Use valid OpenAI API credentials.  
   - No special options.

5. **Add LangChain Agent node "AI Supplier Selector":**  
   - Connect input from OpenAI Chat node (as agent input).  
   - Configure system message defining AI as procurement expert.  
   - Set prompt with instructions to analyze order and respond ONLY with JSON in specified format: recommendedSupplier, reasoning, estimatedDelivery, costOptimization, alternativeSupplier, specialNotes.  
   - Node version: 1.7.

6. **Add Code node "Enrich with Supplier Data":**  
   - Connect input from "AI Supplier Selector".  
   - Paste enrichment code:  
     - Parse AI JSON output safely, fallback to default supplier if parse fails.  
     - Define hardcoded supplier database with contact info, payment terms, delivery days.  
     - Merge AI recommendation and supplier info with original validated request.  
     - Add company and shipping details with defaults.  
     - Include special instructions and notes.  
   - Node version: 2.

7. **Add IF node "Needs Approval?":**  
   - Connect input from "Enrich with Supplier Data".  
   - Condition: Boolean equals `{{$json.requiresApproval}}` to `true`.  
   - If true branch: connect to "Request Approval".  
   - Else branch: connect to "Generate PO Document".

8. **Add Gmail node "Request Approval":**  
   - Connect from true branch of "Needs Approval?".  
   - Configure Gmail OAuth2 credentials.  
   - Set email parameters to request approval (recipient, subject, body referencing PO details).  
   - Node version: 2.1.

9. **Add Code node "Generate PO Document":**  
   - Connect from false branch of "Needs Approval?" or after approval.  
   - Paste HTML generation script:  
     - Dynamically create HTML for purchase order with company info, supplier details, itemized table, AI insights, totals, and notes.  
     - Format currency, include urgency badge, render AI recommendation box conditionally.  
     - Output JSON with fields: `html`, `poNumber`, `supplierEmail`, `supplierName`, `total`, `fileName`.  
   - Node version: 2.

10. **Add HTML CSS to PDF node "HTML to PDF":**  
    - Connect input from "Generate PO Document".  
    - No extra parameters required.  
    - Requires API key from htmlcsstoimage.com for HTML to PDF conversion.

11. **Add Google Drive node "Archive in Drive":**  
    - Connect input from "HTML to PDF".  
    - Set drive to "My Drive" and folder to root.  
    - File name: use expression from PO Document node output `fileName`.  
    - Authenticate with Google Drive OAuth2 credentials.

12. **Add Gmail node "Email to Supplier":**  
    - Connect input from "Archive in Drive".  
    - Configure Gmail OAuth2 credentials.  
    - Set recipient to the supplier email from PO Document output.  
    - Attach the generated PDF file.  
    - Compose email subject and body referencing PO number and order details.  
    - Node version: 2.1.

13. **Add Code node "Log in Procurement System":**  
    - Connect input from "Email to Supplier".  
    - Script builds a procurement log JSON object with PO details, status, AI recommendation, approval status, and activity log entry.  
    - Optionally prepare for API integration to external system.  
    - Node version: 2.

14. **Add HTTP Request node "Notify Procurement Team":**  
    - Connect input from "Log in Procurement System".  
    - Configure HTTP POST to Slack Incoming Webhook URL.  
    - JSON body includes Slack block formatting with PO summary, AI insights, and Google Drive link.  
    - Use expressions to fill dynamic data fields.  
    - Node version: 4.1.

15. **Add Sticky Notes:**  
    - Add descriptive sticky notes at logical places to document workflow blocks and key information for users.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                      | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow automates purchase order creation with AI-powered supplier selection, approval routing, PO PDF generation, archival, email distribution, and team notification.                                      | Full workflow overview sticky note at start of workflow.                                               |
| Requires OpenAI API key with GPT-4 recommended for best supplier recommendations.                                                                                                                                | Setup instructions in main sticky note.                                                                |
| HTML to PDF conversion depends on an API key from https://htmlcsstoimage.com.                                                                                                                                     | Mentioned in setup sticky note.                                                                         |
| Google Drive OAuth2 credentials needed for archival of purchase orders.                                                                                                                                           | Setup instructions and Archive in Drive node.                                                          |
| Gmail OAuth2 credentials required for sending approval and supplier emails.                                                                                                                                        | Setup instructions and Gmail nodes.                                                                    |
| Slack webhook URL required for procurement team notifications.                                                                                                                                                    | Setup instructions and Notify Procurement Team node.                                                   |
| Approval threshold set at $5000 by default; can be adjusted in the Validate Request node JavaScript.                                                                                                               | Validation node comments and Sticky Note5.                                                             |
| Supplier database is hardcoded in the Enrich with Supplier Data node; for production, replace with dynamic database or API call.                                                                                  | Enrich with Supplier Data node comments.                                                               |
| AI recommendation JSON is strictly parsed; fallback to default supplier if parsing fails to avoid workflow failure.                                                                                               | Enrich with Supplier Data node comments.                                                               |
| Email to Supplier node assumes supplier email is valid and reachable; ensure data correctness to avoid bounce-backs.                                                                                              | Email to Supplier node comments.                                                                        |
| Slack notification uses block kit formatting for rich message display.                                                                                                                                             | Notify Procurement Team node comments.                                                                 |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.