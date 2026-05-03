Bulk generate payment reminder PDFs from NocoDB with Autype

https://n8nworkflows.xyz/workflows/bulk-generate-payment-reminder-pdfs-from-nocodb-with-autype-13786


# Bulk generate payment reminder PDFs from NocoDB with Autype

### 1. Workflow Overview

This workflow automates the process of identifying overdue invoices, generating professional payment reminder PDF documents in bulk, and distributing them via email as a compressed ZIP archive. It is designed for businesses using **NocoDB** as their database and **Autype** for high-quality PDF generation.

The workflow consists of two main logical paths:
1.  **The Setup Path (Manual):** A one-time initialization sequence to create the necessary project and document template within Autype.
2.  **The Execution Path (Scheduled):** A weekly routine that fetches overdue data, calculates aging, renders PDFs in bulk, and sends them to a designated recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 One-time Setup Logic
**Overview:** This block is used once to initialize the Autype environment. It creates a container (Project) and defines the visual layout and variable placeholders (Document Template) for the reminders.

*   **Nodes Involved:** `Run Setup Once`, `Create Project`, `Create Document`.
*   **Node Details:**
    *   **Run Setup Once:** Manual trigger used to start the initialization.
    *   **Create Project:** 
        *   **Type:** Autype Node.
        *   **Role:** Creates a project named "Payment Reminders".
        *   **Input:** Manual Trigger.
        *   **Output:** Project ID (used by the next node).
    *   **Create Document:**
        *   **Type:** Autype Node.
        *   **Role:** Creates a PDF template with complex styling (headers, footers, tables, and brand colors). 
        *   **Configuration:** Defines variables for `customerName`, `amountDue`, `invoiceNumber`, `dueDate`, `daysOverdue`, etc.
        *   **Failure Modes:** API authentication failure or invalid JSON structure in the template definition.

#### 2.2 Data Retrieval (NocoDB)
**Overview:** This block triggers the process and pulls the raw data from the database.

*   **Nodes Involved:** `Weekly Schedule`, `Get Overdue Invoices`.
*   **Node Details:**
    *   **Weekly Schedule:** 
        *   **Type:** Schedule Trigger.
        *   **Config:** Set to run every Monday.
    *   **Get Overdue Invoices:**
        *   **Type:** NocoDB Node.
        *   **Role:** Performs a `getAll` operation on a specific Table ID to retrieve invoice records.
        *   **Requirements:** Requires `customer_name`, `amount_due`, and `due_date` fields to exist in NocoDB.

#### 2.3 Data Transformation & Logic
**Overview:** This block processes the raw database rows into a format compatible with bulk document rendering.

*   **Nodes Involved:** `Build Bulk Items`.
*   **Node Details:**
    *   **Type:** Code Node (JavaScript).
    *   **Logic:**
        1. Iterates through NocoDB rows.
        2. Calculates `daysOverdue` by comparing `today` with the `due_date`.
        3. Maps database fields to Autype variable names.
        4. Aggregates all items into a single JSON array.
    *   **Key Variables:** `documentId` (The ID of the template created in Step 2.1).
    *   **Edge Cases:** Handles missing fields by providing default empty strings or "0.00" values.

#### 2.4 Document Rendering & Distribution
**Overview:** The final stage where data is converted into files and sent via email.

*   **Nodes Involved:** `Bulk Render Payment Reminders`, `Send ZIP via Email`.
*   **Node Details:**
    *   **Bulk Render Payment Reminders:**
        *   **Type:** Autype Node.
        *   **Role:** Sends the array of data to Autype’s bulk engine.
        *   **Operation:** `bulkRender`.
        *   **Output:** A binary ZIP file containing one PDF for every record found in NocoDB.
    *   **Send ZIP via Email:**
        *   **Type:** Email Send (SMTP) Node.
        *   **Config:** Attaches the binary output from the previous node.
        *   **Subject Expression:** Dynamic subject line including the current date and the total count of documents generated.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Weekly Schedule** | Schedule Trigger | Workflow entry point | None | Get Overdue Invoices | 1. Weekly Schedule — Triggers every Monday (configurable). |
| **Get Overdue Invoices** | NocoDB | Data fetching | Weekly Schedule | Build Bulk Items | 1. Get Overdue Invoices: Reads all rows from NocoDB. Columns: customer_name, customer_address, invoice_number, amount_due, due_date, company_name |
| **Build Bulk Items** | Code | Data mapping & logic | Get Overdue Invoices | Bulk Render... | 2. Build Bulk Items: Maps NocoDB rows to Autype variable sets. Calculates days overdue from due_date vs. today. Sets the document ID. |
| **Bulk Render Payment Reminders** | Autype | PDF Batch Generation | Build Bulk Items | Send ZIP via Email | 3. Bulk Render: Sends all variable sets in one request. Downloads the output as a ZIP archive containing one PDF per invoice. |
| **Send ZIP via Email** | Email Send | File distribution | Bulk Render... | None | 4. Send ZIP via Email: Emails the ZIP to a printer service, accounting team, or archive mailbox via SMTP. |
| **Run Setup Once** | Manual Trigger | Setup entry point | None | Create Project | Run once to create the Autype project and payment reminder template document. |
| **Create Project** | Autype | Resource creation | Run Setup Once | Create Document | Run once to create the Autype project and payment reminder template document. |
| **Create Document** | Autype | Template definition | Create Project | None | Run once to create the Autype project and payment reminder template document. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Initialize Autype:**
    *   Create a **Manual Trigger** node.
    *   Add an **Autype Node** (Operation: `createProject`) and name the project "Payment Reminders".
    *   Add a second **Autype Node** (Operation: `createDocument`). Use a JSON structure defining sections (headers, tables, text) and variables (`customerName`, `invoiceNumber`, `amountDue`, `daysOverdue`).
    *   Run this sequence once and **copy the `id`** of the created document.

2.  **Set Up Trigger & Data Source:**
    *   Create a **Schedule Trigger** set to "Weekly" on Mondays.
    *   Connect a **NocoDB Node**. Configure your credentials, Base ID, and Table ID. Ensure the operation is set to `Get All`.

3.  **Implement Transformation Logic:**
    *   Connect a **Code Node** (JavaScript). 
    *   Paste logic to calculate the difference between `new Date()` and the row's `due_date`.
    *   Map the output so it returns a single object containing the `documentId` (copied in step 1) and an `items` string (the stringified array of all row data).

4.  **Bulk PDF Generation:**
    *   Add an **Autype Node** and set the resource to `Bulk Render`.
    *   Set the `Document ID` field to point to the `documentId` from the Code node.
    *   Set the `Items` field to the `items` variable from the Code node.
    *   **Crucial:** Enable "Download Output" to receive the binary ZIP file.

5.  **Email Delivery:**
    *   Add an **Email Send Node**.
    *   Configure SMTP credentials.
    *   Set the "Attachments" option to use the binary data from the Bulk Render node.
    *   Set the recipient and a dynamic subject line (e.g., `Payment Reminders - {{ $now }}`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Community node required for self-hosted instances | **n8n-nodes-autype** |
| Official Autype Dashboard for API Keys | [app.autype.com](https://app.autype.com) |
| Recommended Variable Types | String (names, dates), Number (amounts, aging) |
| Workflow Use Case | Ideal for print-by-email services or automated accounting archives |