Extract invoice data from PDF with Claude AI and create vendor bill in Odoo 18

https://n8nworkflows.xyz/workflows/extract-invoice-data-from-pdf-with-claude-ai-and-create-vendor-bill-in-odoo-18-14573


# Extract invoice data from PDF with Claude AI and create vendor bill in Odoo 18

# Technical Documentation: Extract Invoice Data from PDF with Claude AI and Create Vendor Bill in Odoo 18

### 1. Workflow Overview

This workflow automates the accounts payable process by extracting key financial data from PDF invoices using Claude AI and synchronizing that data into Odoo 18 as a draft vendor bill. It handles the end-to-end pipeline from file reception to ERP entry, including duplicate prevention and automatic vendor management.

The logic is organized into four primary functional blocks:
- **1.1 PDF Upload & AI Extraction:** Handles the ingestion of the PDF file and uses a Large Language Model (LLM) to structure the unstructured document data.
- **1.2 Odoo Authentication & Duplicate Check:** Establishes a session with Odoo and ensures the invoice hasn't already been processed.
- **1.3 Vendor Lookup & Creation:** Matches the extracted vendor name against existing Odoo partners or creates a new supplier record if no match is found.
- **1.4 Invoice Creation & Response:** Generates the draft `account.move` record in Odoo and returns the final status to the requester.

---

### 2. Block-by-Block Analysis

#### 2.1 PDF Upload & AI Extraction
**Overview:** This block converts a binary PDF upload into a format readable by Claude AI and extracts structured JSON data.

- **Nodes Involved:** `Receive Invoice PDF`, `PDF to Base64`, `Configuration`, `Prepare Claude Request`, `Claude AI Extract`, `Parse Extraction`.
- **Node Details:**
    - **Receive Invoice PDF (Webhook):** Listens for `POST` requests at `/invoice-process`. It expects a multipart form-data upload.
    - **PDF to Base64 (Extract from File):** Converts the binary PDF file into a base64 string required for the Anthropic API.
    - **Configuration (Set):** Centralizes environment variables including Odoo URL, Database, Credentials, and the specific Claude model version.
    - **Prepare Claude Request (Code):** Constructs the JSON payload for Anthropic. It includes the base64 PDF and a strict prompt requesting specific fields: `vendor`, `invoice_number`, `invoice_date`, `total_amount`, `currency`, `net_amount`, `tax_rate`, and `tax_amount`.
    - **Claude AI Extract (HTTP Request):** Sends the request to `api.anthropic.com`. Uses `HTTP Header Auth` for the API key and includes the `anthropic-version` header.
    - **Parse Extraction (Code):** Cleans the AI response (removing markdown blocks), parses the JSON, validates that required fields exist, and normalizes numeric types (float conversion).
    - **Failure Types:** PDF too small/empty, Claude API timeout, or AI returning non-JSON text.

#### 2.2 Odoo Authentication & Duplicate Check
**Overview:** Manages secure access to Odoo and prevents the creation of redundant financial records.

- **Nodes Involved:** `Odoo Authenticate`, `Prepare Duplicate Check`, `Search Duplicate Invoice`, `Duplicate Found?`, `Handle Duplicate`, `Duplicate Response`.
- **Node Details:**
    - **Odoo Authenticate (HTTP Request):** Uses the Odoo JSON-RPC `common/authenticate` method to retrieve a `uid`.
    - **Prepare Duplicate Check (Code):** Builds a JSON-RPC call to `account.move` using `search_read` to find any existing invoice where the reference matches the extracted `invoice_number`.
    - **Search Duplicate Invoice (HTTP Request):** Executes the search query against the Odoo instance.
    - **Duplicate Found? (If):** Checks if the length of the result array is greater than 0.
    - **Handle Duplicate (Code):** Formats a "Failure" response containing details of the existing invoice.
    - **Duplicate Response (Respond to Webhook):** Returns the duplicate status to the user.
    - **Failure Types:** Odoo authentication failure (wrong password/db), network timeout.

#### 2.3 Vendor Lookup & Creation
**Overview:** Ensures the invoice is linked to the correct partner record in Odoo.

- **Nodes Involved:** `Prepare Vendor Search`, `Search Vendor`, `Vendor Exists?`, `Use Existing Vendor`, `Prepare Create Vendor`, `Create Vendor in Odoo`, `Get New Vendor ID`.
- **Node Details:**
    - **Prepare Vendor Search (Code):** Constructs a JSON-RPC call searching `res.partner` using the `ilike` operator on the vendor name and filtering for `supplier_rank > 0`.
    - **Search Vendor (HTTP Request):** Executes the search in Odoo.
    - **Vendor Exists? (If):** Evaluates if the search returned a partner.
    - **Use Existing Vendor (Code):** Extracts the `partnerId` from the search result.
    - **Prepare Create Vendor (Code):** If not found, prepares a `create` call for `res.partner` with `supplier_rank: 1` and `company_type: 'company'`.
    - **Create Vendor in Odoo (HTTP Request):** Creates the new partner record.
    - **Get New Vendor ID (Code):** Extracts the ID of the newly created partner.
    - **Failure Types:** Vendor name too generic leading to wrong match, or API permission errors when creating partners.

#### 2.4 Invoice Creation & Response
**Overview:** Finalizes the process by creating the financial record in Odoo.

- **Nodes Involved:** `Prepare Invoice`, `Create Invoice in Odoo`, `Build Response`, `Success Response`, `Error Response`.
- **Node Details:**
    - **Prepare Invoice (Code):** Constructs the `account.move` create call. It sets `move_type` to `in_invoice` (Vendor Bill) and creates a single invoice line with the `net_amount`.
    - **Create Invoice in Odoo (HTTP Request):** Sends the creation request to Odoo.
    - **Build Response (Code):** Validates the Odoo response and prepares a success JSON object.
    - **Success Response (Respond to Webhook):** Returns a 200 OK with the Odoo Invoice ID and extracted data.
    - **Error Response (Respond to Webhook):** A global error handler that catches failures from AI or Odoo nodes and returns a 500 error with the specific node's error message.
    - **Failure Types:** Validation errors in Odoo (e.g., missing mandatory fields), currency mismatches.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Invoice PDF | Webhook | Entry point / File upload | - | PDF to Base64 | 1. PDF Upload & AI Extraction |
| PDF to Base64 | Extract from File | Binary conversion | Receive Invoice PDF | Configuration | 1. PDF Upload & AI Extraction |
| Configuration | Set | Env variable management | PDF to Base64 | Prepare Claude Req | 1. PDF Upload & AI Extraction |
| Prepare Claude Request | Code | Prompt Engineering | Configuration | Claude AI Extract | 1. PDF Upload & AI Extraction |
| Claude AI Extract | HTTP Request | Data extraction via LLM | Prep Claude Request | Parse Extraction, Error Response | 1. PDF Upload & AI Extraction |
| Parse Extraction | Code | JSON Cleaning & Validation | Claude AI Extract | Odoo Authenticate | 1. PDF Upload & AI Extraction |
| Odoo Authenticate | HTTP Request | Session management | Parse Extraction | Prep Duplicate Check, Error Response | 2. Odoo Auth & Duplicate Check |
| Prepare Duplicate Check | Code | Search query builder | Odoo Authenticate | Search Duplicate Invoice | 2. Odoo Auth & Duplicate Check |
| Search Duplicate Invoice | HTTP Request | Duplicate verification | Prep Duplicate Check | Duplicate Found? | 2. Odoo Auth & Duplicate Check |
| Duplicate Found? | If | Logic switch (Duplicate) | Search Duplicate Invoice | Handle Duplicate, Prep Vendor Search | 2. Odoo Auth & Duplicate Check |
| Handle Duplicate | Code | Error response builder | Duplicate Found? | Duplicate Response | 2. Odoo Auth & Duplicate Check |
| Duplicate Response | Respond to Webhook | API Response (Duplicate) | Handle Duplicate | - | 2. Odoo Auth & Duplicate Check |
| Prepare Vendor Search | Code | Partner search builder | Duplicate Found? | Search Vendor | 3. Vendor Lookup & Creation |
| Search Vendor | HTTP Request | Partner verification | Prep Vendor Search | Vendor Exists? | 3. Vendor Lookup & Creation |
| Vendor Exists? | If | Logic switch (Vendor) | Search Vendor | Use Existing Vendor, Prep Create Vendor | 3. Vendor Lookup & Creation |
| Use Existing Vendor | Code | ID extraction | Vendor Exists? | Prepare Invoice | 3. Vendor Lookup & Creation |
| Prepare Create Vendor | Code | Partner creation builder | Vendor Exists? | Create Vendor in Odoo | 3. Vendor Lookup & Creation |
| Create Vendor in Odoo | HTTP Request | Partner creation | Prep Create Vendor | Get New Vendor ID | 3. Vendor Lookup & Creation |
| Get New Vendor ID | Code | ID extraction | Create Vendor in Odoo | Prepare Invoice | 3. Vendor Lookup & Creation |
| Prepare Invoice | Code | Bill creation builder | Use Exist Vendor, Get New Vendor ID | Create Invoice in Odoo | 4. Invoice Creation & Response |
| Create Invoice in Odoo | HTTP Request | Bill creation | Prepare Invoice | Build Response, Error Response | 4. Invoice Creation & Response |
| Build Response | Code | Success data builder | Create Invoice in Odoo | Success Response | 4. Invoice Creation & Response |
| Success Response | Respond to Webhook | API Response (Success) | Build Response | - | 4. Invoice Creation & Response |
| Error Response | Respond to Webhook | API Response (Failure) | Claude AI, Odoo Auth, Odoo Create | - | Global Error Handling |

---

### 4. Reproducing the Workflow from Scratch

1.  **Webhook Setup:** Create a **Webhook** node. Set path to `invoice-process`, method to `POST`, and response mode to `Response Node`.
2.  **File Processing:** Add an **Extract from File** node. Set operation to `Binary to Property` to convert the uploaded PDF to a base64 string.
3.  **Global Config:** Add a **Set** node. Define a JSON object with: `odooUrl`, `odooDb`, `odooUser`, `odooPassword`, and `anthropicModel` (e.g., `claude-sonnet-4-5-20250929`).
4.  **AI Prompting:** Add a **Code** node to build the Anthropic request. Use the base64 data and a system prompt requesting specific JSON fields (`vendor`, `invoice_number`, etc.).
5.  **AI Request:** Add an **HTTP Request** node. 
    - URL: `https://api.anthropic.com/v1/messages`. 
    - Method: `POST`. 
    - Auth: `HTTP Header Auth` (`x-api-key`). 
    - Header: `anthropic-version: 2023-06-01`.
6.  **Data Parsing:** Add a **Code** node to remove Markdown backticks from the AI response and `JSON.parse()` the result. Validate that required fields are present.
7.  **Odoo Session:** Add an **HTTP Request** node to `{{odooUrl}}/jsonrpc`. Body should call `common.authenticate` with the db, user, and password from the Config node.
8.  **Duplicate Logic:**
    - Use a **Code** node to build a `search_read` call for `account.move` where `ref` equals the extracted invoice number.
    - Execute via **HTTP Request** node.
    - Use an **If** node to check if the result array length `> 0`.
    - If TRUE: Route to a **Code** node (Handle Duplicate) $\rightarrow$ **Respond to Webhook**.
9.  **Vendor Logic:**
    - If FALSE: Use a **Code** node to build a `search_read` for `res.partner` where name `ilike` the extracted vendor.
    - Execute via **HTTP Request**.
    - Use an **If** node to check if the partner exists.
    - If TRUE: Extract `partnerId`.
    - If FALSE: Use a **Code** node to build a `create` call for `res.partner` $\rightarrow$ **HTTP Request** $\rightarrow$ **Code** node to extract new ID.
10. **Invoice Entry:**
    - Use a **Code** node to build the `account.move` creation JSON. Ensure `move_type` is `in_invoice` and include one line in `invoice_line_ids`.
    - Execute via **HTTP Request**.
11. **Finalization:**
    - Route success to a **Code** node (Build Response) $\rightarrow$ **Respond to Webhook**.
    - Route all "Error" outputs from the AI and Odoo HTTP nodes to a final **Respond to Webhook** node with a `500` status code.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Author: Florian Eiche | [eiche-digital.de](https://eiche-digital.de) |
| Data Privacy Warning: PDFs are sent to Anthropic. Ensure DPA compliance. | [Anthropic DPA](https://www.anthropic.com/legal/data-processing-addendum) |
| Recommended Model: Claude Sonnet 4.5 | High accuracy for document extraction |
| Integration Requirement: Odoo 18 Invoicing module must be installed | Functional dependency |