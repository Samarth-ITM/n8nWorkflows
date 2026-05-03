Raw Materials Inventory Management with Google Sheets, Supabase and Email Approvals

https://n8nworkflows.xyz/workflows/raw-materials-inventory-management-with-google-sheets--supabase-and-email-approvals-3979


# Raw Materials Inventory Management with Google Sheets, Supabase and Email Approvals

### 1. Workflow Overview

This n8n workflow automates raw materials inventory management by integrating Google Sheets, Supabase, and Gmail. It is designed for small to medium-sized businesses and inventory managers who want to streamline inventory tracking, automate material issue approvals, and receive timely low stock alerts.

The workflow is organized into two main logical blocks with clear sub-flows:

**1.1 Raw Materials Receiving and Stock Update**  
- Receives raw material data via webhook (e.g., from forms).  
- Standardizes and validates input data.  
- Calculates total price of received materials.  
- Records raw material receipts in Google Sheets and Supabase.  
- Checks and updates current stock levels in both Google Sheets and Supabase.  
- Detects low stock levels and sends alert emails via Gmail.

**1.2 Material Issue Request and Approval**  
- Receives material issue requests via webhook.  
- Standardizes and validates request data.  
- Checks stock availability and sends approval request emails with actionable links.  
- Processes approver’s response via webhook.  
- Updates stock quantities post-approval.  
- Sends low stock alerts if necessary.  
- Updates request status in Google Sheets and Supabase.

---

### 2. Block-by-Block Analysis

#### 2.1 Raw Materials Receiving and Stock Update

**Overview:**  
Handles incoming raw materials data, validates and calculates costs, updates stock records in Google Sheets and Supabase, and triggers low stock alerts when needed.

**Nodes Involved:**  
- Receive Raw Materials Webhook  
- Standardize Raw Material Data  
- Calculate Total Price  
- Append Raw Materials  
- Validate Quantity Received  
- Lookup Existing Stock  
- Check If Product ID Exists (If node)  
- Calculate Updated Current Stock (True branch)  
- Update Current Stock (Google Sheets, True branch)  
- Current Stock Update (Supabase, True branch)  
- Retrieve Updated Stock for Check (LookUp Current stock)  
- Low Stock Detection (Low stock Detection2)  
- Trigger Low Stock Alert (Trigger Low Stock Alert)  
- Send Low Stock Email Alert (Send Low Stock Email Alert)  
- Format Response (Format response)  
- Initialize New Product stock (Google Sheets, False branch)  
- New Row Current Stock (Supabase, False branch)  
- New Record Raw (Supabase)  
- Merge (Merge)  

**Node Details:**

- **Receive Raw Materials Webhook**  
  - Type: Webhook  
  - Role: Entry point receiving POST requests containing raw materials data.  
  - Config: Path `/Pb-raw-materials`, expects JSON body from forms.  
  - Input: HTTP POST JSON payload.  
  - Output: Raw JSON data to next node.  
  - Potential Failures: Missing or malformed JSON, network timeout.

- **Standardize Raw Material Data**  
  - Type: Set  
  - Role: Maps and normalizes form input to consistent named fields (e.g., `Product ID`, `Quantity Received`).  
  - Key expressions: Maps fields from webhook `body` to standard field names.  
  - Input: Raw JSON from webhook.  
  - Output: Standardized JSON for downstream nodes.  
  - Edge Cases: Missing fields in input JSON.  
  - Version: v3.4

- **Calculate Total Price**  
  - Type: Code  
  - Role: Parses numeric fields (`Quantity Received`, `Unit Price`), validates them, computes `Total Price`.  
  - Key code: Robust number parsing stripping currency symbols, errors thrown on invalid data.  
  - Input: Standardized JSON.  
  - Output: JSON including calculated `Total Price`.  
  - Potential Failures: Invalid numeric inputs, runtime errors.

- **Append Raw Materials**  
  - Type: Google Sheets (Append)  
  - Role: Appends new raw material receipt record to Google Sheets "Raw Materials" sheet.  
  - Config: Auto-mapping input fields, credentials required.  
  - Input: Data with total price calculated.  
  - Output: Confirmation of append operation.  
  - Failures: Google API limits, auth errors.

- **Validate Quantity Received**  
  - Type: Code  
  - Role: Ensures `Quantity Received` is a positive number; throws error if invalid.  
  - Input: Output of Append Raw Materials.  
  - Output: JSON with validated numeric `Quantity Received`.  
  - Edge Cases: Zero or negative quantities cause error.

- **Lookup Existing Stock**  
  - Type: Google Sheets (Lookup)  
  - Role: Searches "Current Stock" sheet for existing stock by `Product ID`.  
  - Input: Validated JSON.  
  - Output: Stock record if exists, empty if not.  
  - Failures: Google Sheets lookup errors, no matches found.

- **Check If Product ID Exists**  
  - Type: If  
  - Role: Branches workflow based on presence of `Product ID` in stock lookup result.  
  - Conditions: Checks if `Product ID` exists.  
  - Output: True branch (product exists), False branch (new product).

- **Calculate Updated Current Stock** (True Branch)  
  - Type: Code  
  - Role: Adds received quantity to existing stock.  
  - Input: Existing stock and validated quantity.  
  - Output: Updated stock number.

- **Update Current Stock** (Google Sheets, True Branch)  
  - Type: Google Sheets (Update)  
  - Role: Updates stock record in "Current Stock" sheet with new quantity.  
  - Key config: Matches row by `Product ID`.  
  - Input: Updated stock data.  
  - Output: Confirmation of update.

- **Current Stock Update** (Supabase, True Branch)  
  - Type: Supabase (Update)  
  - Role: Updates `Current Stock` table in Supabase database.  
  - Input: Updated stock data.  
  - Output: Updated Supabase record.  
  - Authentication: Requires Supabase credentials.

- **Retrieve Updated Stock for Check** (LookUp Current stock)  
  - Type: Google Sheets (Lookup)  
  - Role: Retrieves stock data post-update for low stock detection.  
  - Input: Updated stock data.  
  - Output: Current stock details.

- **Low Stock Detection** (Low stock Detection2)  
  - Type: Code  
  - Role: Compares current stock to minimum stock level (default 50).  
  - Output: Flag `Is Low` and `Alert Message` if stock is low.

- **Trigger Low Stock Alert**  
  - Type: If  
  - Role: Branches based on `Is Low` flag to send alert emails.  

- **Send Low Stock Email Alert**  
  - Type: Gmail  
  - Role: Sends HTML email alert to configured recipient with low stock details.  
  - Input: Alert message and product info.  
  - Failures: Gmail API limits or auth failure.

- **Format Response**  
  - Type: Item Lists  
  - Role: Removes duplicates from Supabase search results.  
  - Input: Supabase records array.  
  - Output: Cleaned unique records.

- **Initialize New Product stock** (False Branch)  
  - Type: Google Sheets (Append)  
  - Role: Adds a new product row with initial stock and default minimum stock level (50).  

- **New Row Current Stock** (Supabase, False Branch)  
  - Type: Supabase (Insert)  
  - Role: Inserts new product stock record into Supabase `Current Stock` table.

- **New Record Raw**  
  - Type: Supabase (Insert)  
  - Role: Inserts raw material receipt record into Supabase `Raw Materials` table.

- **Merge**  
  - Type: Merge  
  - Role: Combines the true/false branches of stock existence check to unify data flow.

---

#### 2.2 Material Issue Request and Approval

**Overview:**  
Manages incoming material issue requests, validates them, checks stock availability, sends approval emails, processes approval responses, updates stock, and sends low stock alerts post-issuance.

**Nodes Involved:**  
- Receive Issue Request (Webhook)  
- Standardize Data  
- Validate Issue Request Data  
- Verify Requested Quantity  
- Append Material Request (Google Sheets)  
- Check Available Stock for Issue (Google Sheets)  
- Merge Lookups  
- Prepare Approval (Code)  
- Send Approval Request (Gmail)  
- Get Approvals (Webhook)  
- Format Approval Response  
- Verify Approval Data (Code)  
- Retrieve Issue Request Details (Google Sheets)  
- Process Approval Decision (If)  
- Get Stock for Issue Update from Current (Google Sheets)  
- Update Stock (Code)  
- Update Stock After Issue (Google Sheets)  
- Materials Issue Table Update (Supabase)  
- Retrieve Stock After Issue (LookUp Current stock1)  
- Low Stock Detection1 (Code)  
- Is Stock is Low (If)  
- Low Stock Email Alert (Gmail)  
- Combine Stock Lookup Results (Merge1)  
- Create Record Issue (Supabase)  
- Search Stock by Product ID (Supabase)  
- Search Issue by Submission ID (Supabase)  
- Combine Issue Lookup Branches (Merge1)  

**Node Details:**

- **Receive Issue Request**  
  - Type: Webhook  
  - Role: Entry point receiving POST requests for issue requests.  
  - Config: Path `/raw-materials-issue`, HTTP POST.  
  - Input: JSON payload with issue request data.  
  - Output: Raw JSON.

- **Standardize Data**  
  - Type: Set  
  - Role: Normalizes request data fields, adds `Approval Link`, sets `Status` to "Pending".  
  - Output: JSON with standardized fields.

- **Validate Issue Request Data**  
  - Type: Code  
  - Role: Validates `Quantity Requested` is positive.  
  - Errors on invalid quantity.

- **Verify Requested Quantity**  
  - Type: Code  
  - Role: Ensures `Product ID` and `Submission ID` present, quantity valid.  
  - Throws errors if missing.

- **Append Material Request**  
  - Type: Google Sheets (Append)  
  - Role: Records the issue request in "Materials Issued" sheet.  
  - Input: Verified data.

- **Check Available Stock for Issue**  
  - Type: Google Sheets (Lookup)  
  - Role: Retrieves current stock for requested `Product ID`.  

- **Merge Lookups**  
  - Type: Merge  
  - Role: Combines lookup results for approval preparation.

- **Prepare Approval**  
  - Type: Code  
  - Role: Checks if stock is enough for requested quantity, adds `Is Enough` flag.  

- **Send Approval Request**  
  - Type: Gmail  
  - Role: Sends HTML email to approver with detailed request and Approve/Reject links.  
  - Config: Email recipient must be set; uses dynamic template with buttons.  
  - Failures: Gmail API errors, invalid email addresses.

- **Get Approvals**  
  - Type: Webhook  
  - Role: Receives GET requests from approval email links with parameters (submissionId, action, quantity, approvedBy).  

- **Format Approval Response**  
  - Type: Set  
  - Role: Extracts and formats approval data, adds `Approval Date` timestamp.  

- **Verify Approval Data**  
  - Type: Code  
  - Role: Validates approval action is `approve` or `reject`, quantity positive if approved.  

- **Retrieve Issue Request Details**  
  - Type: Google Sheets (Lookup)  
  - Role: Fetches original issue request data by `Submission ID`.  

- **Process Approval Decision**  
  - Type: If  
  - Role: Branches flow into approved or rejected paths based on action.

- **Get Stock for Issue Update from Current**  
  - Type: Google Sheets (Lookup)  
  - Role: Retrieves stock before deduction.

- **Update Stock**  
  - Type: Code  
  - Role: Calculates new stock after deducting approved quantity; errors if insufficient stock.

- **Update Stock After Issue**  
  - Type: Google Sheets (Update)  
  - Role: Updates "Current Stock" sheet with new stock value.  

- **Materials Issue Table Update**  
  - Type: Supabase (Update)  
  - Role: Updates issue request status in Supabase `Materials Issued` table.  

- **Retrieve Stock After Issue** (LookUp Current stock1)  
  - Type: Google Sheets (Lookup)  
  - Role: Retrieves updated stock for low stock detection.

- **Low Stock Detection1**  
  - Type: Code  
  - Role: Checks if stock is below minimum after issue.

- **Is Stock is Low**  
  - Type: If  
  - Role: Branches flow to send alert if stock low.

- **Low Stock Email Alert**  
  - Type: Gmail  
  - Role: Sends low stock alert email post-issue.

- **Combine Stock Lookup Results** (Merge1)  
  - Type: Merge  
  - Role: Combines lookup results for consistency.

- **Create Record Issue**  
  - Type: Supabase (Insert)  
  - Role: Inserts issue request record into Supabase `Materials Issued` table.

- **Search Stock by Product ID**  
  - Type: Supabase (GetAll)  
  - Role: Fetches stock data from Supabase.

- **Search Issue by Submission ID**  
  - Type: Supabase (GetAll)  
  - Role: Fetches issue request records by `Submission ID`.

- **Combine Issue Lookup Branches**  
  - Type: Merge  
  - Role: Merges issue fetch branches to unify data.

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                                   | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                                             |
|------------------------------|---------------------|-------------------------------------------------|----------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Receive Raw Materials Webhook | Webhook             | Entry point for raw materials data               | -                                | Standardize Raw Material Data      | # Raw Materials Receiving and Stock Update                                                                                             |
| Standardize Raw Material Data | Set                 | Normalize raw materials input fields              | Receive Raw Materials Webhook     | Calculate Total Price              | # Raw Materials Receiving and Stock Update                                                                                             |
| Calculate Total Price         | Code                | Calculate total price and validate numbers       | Standardize Raw Material Data     | Append Raw Materials               | # Raw Materials Receiving and Stock Update                                                                                             |
| Append Raw Materials          | Google Sheets       | Append raw materials record                        | Calculate Total Price             | Validate Quantity Received         | # Raw Materials Receiving and Stock Update                                                                                             |
| Validate Quantity Received    | Code                | Validate quantity received                         | Append Raw Materials              | Lookup Existing Stock              | # Raw Materials Receiving and Stock Update                                                                                             |
| Lookup Existing Stock         | Google Sheets       | Lookup current stock by Product ID                 | Validate Quantity Received        | Check If Product ID Exists         | # Raw Materials Receiving and Stock Update                                                                                             |
| Check If Product ID Exists    | If                  | Branch on existence of product in stock            | Lookup Existing Stock             | Calculate Updated Current Stock, Format Response | # Raw Materials Receiving and Stock Update                                                                                             |
| Calculate Updated Current Stock| Code               | Calculate updated stock (add received quantity)    | Check If Product ID Exists (True) | Update Current Stock, Current Stock Update | # Raw Materials Receiving and Stock Update                                                                                             |
| Update Current Stock          | Google Sheets       | Update current stock sheet                          | Calculate Updated Current Stock   | LookUp Current stock              | # Raw Materials Receiving and Stock Update                                                                                             |
| Current Stock Update          | Supabase            | Update current stock in Supabase                    | Calculate Updated Current Stock   | -                                 | # Raw Materials Receiving and Stock Update                                                                                             |
| LookUp Current stock          | Google Sheets       | Retrieve updated stock for low stock check          | Update Current Stock              | Low stock Detection2              | # Raw Materials Receiving and Stock Update                                                                                             |
| Low stock Detection2          | Code                | Detect low stock condition                           | LookUp Current stock              | Trigger Low Stock Alert           | # Raw Materials Receiving and Stock Update                                                                                             |
| Trigger Low Stock Alert       | If                  | Branch to send alert if stock low                    | Low stock Detection2              | Send Low Stock Email Alert        | # Raw Materials Receiving and Stock Update                                                                                             |
| Send Low Stock Email Alert    | Gmail               | Send low stock alert email                           | Trigger Low Stock Alert           | -                                 | # Raw Materials Receiving and Stock Update                                                                                             |
| Format response              | Item Lists          | Remove duplicates from Supabase stock search         | Check If Product ID Exists (False)| Initialize New Product stock, New Row Current Stock | # Raw Materials Receiving and Stock Update                                                                                             |
| Initialize New Product stock  | Google Sheets       | Add new product to current stock sheet               | Format response                  | Update Current Stock              | # Raw Materials Receiving and Stock Update                                                                                             |
| New Row Current Stock         | Supabase            | Insert new product stock record in Supabase            | Format response                  | -                                 | # Raw Materials Receiving and Stock Update                                                                                             |
| New Record Raw                | Supabase            | Insert raw material receipt record in Supabase         | Calculate Total Price            | Search Current Stock             | # Raw Materials Receiving and Stock Update                                                                                             |
| Merge                        | Merge               | Combine branches for product existence                | Lookup Existing Stock, Check If Product ID Exists | Check If Product ID Exists        | # Raw Materials Receiving and Stock Update                                                                                             |
| Receive Issue Request         | Webhook             | Entry point for material issue requests               | -                                | Standardize Data                 | # Material Issue Request and Approval                                                                                                 |
| Standardize Data              | Set                 | Normalize issue request data and add approval link    | Receive Issue Request             | Validate Issue Request Data       | # Material Issue Request and Approval                                                                                                 |
| Validate Issue Request Data   | Code                | Validate quantity requested                             | Standardize Data                 | Verify Requested Quantity         | # Material Issue Request and Approval                                                                                                 |
| Verify Requested Quantity     | Code                | Validate required fields for issue request             | Validate Issue Request Data      | Append Material Request           | # Material Issue Request and Approval                                                                                                 |
| Append Material Request       | Google Sheets       | Append issue request to "Materials Issued" sheet         | Verify Requested Quantity        | Check Available Stock for Issue   | # Material Issue Request and Approval                                                                                                 |
| Check Available Stock for Issue| Google Sheets      | Lookup current stock for requested product              | Append Material Request          | Merge Lookups                   | # Material Issue Request and Approval                                                                                                 |
| Merge Lookups                 | Merge               | Combine stock lookup results                             | Check Available Stock for Issue, Search Product ID | Prepare Approval                | # Material Issue Request and Approval                                                                                                 |
| Prepare Approval              | Code                | Check stock sufficiency and add `Is Enough` flag          | Merge Lookups                   | Send Approval Request            | # Material Issue Request and Approval                                                                                                 |
| Send Approval Request         | Gmail               | Send approval email with Approve/Reject buttons            | Prepare Approval                | -                               | # Material Issue Request and Approval                                                                                                 |
| Get Approvals                 | Webhook             | Receive approval response from email link                  | -                              | Format Approval Response         | # Material Issue Request and Approval                                                                                                 |
| Format Approval Response      | Set                 | Format approval webhook data and add approval date          | Get Approvals                  | Verify Approval Data             | # Material Issue Request and Approval                                                                                                 |
| Verify Approval Data          | Code                | Validate approval action and quantity                        | Format Approval Response       | Retrieve Issue Request Details   | # Material Issue Request and Approval                                                                                                 |
| Retrieve Issue Request Details| Google Sheets       | Retrieve original issue request by Submission ID             | Verify Approval Data           | Merge1                         | # Material Issue Request and Approval                                                                                                 |
| Process Approval Decision     | If                  | Branch workflow based on approval (approve or reject)         | Retrieve Issue Request Details | Update Stock After Issue, Materials Issue Table Update, Get Stock for Issue Update from Current | # Material Issue Request and Approval                                                                                                 |
| Get Stock for Issue Update from Current| Google Sheets | Lookup stock before update                                  | Process Approval Decision      | Update Stock                   | # Material Issue Request and Approval                                                                                                 |
| Update Stock                 | Code                | Calculate new stock after deduction                           | Get Stock for Issue Update from Current | Update Current Stock1         | # Material Issue Request and Approval                                                                                                 |
| Update Stock After Issue      | Google Sheets       | Update stock sheet with new stock                              | Process Approval Decision      | -                             | # Material Issue Request and Approval                                                                                                 |
| Materials Issue Table Update  | Supabase            | Update issue status in Supabase                                | Process Approval Decision      | -                             | # Material Issue Request and Approval                                                                                                 |
| LookUp Current stock1         | Google Sheets       | Retrieve updated stock for low stock detection                  | Update Current Stock1          | Low stock Detection1           | # Material Issue Request and Approval                                                                                                 |
| Low stock Detection1          | Code                | Detect low stock post-issue                                     | LookUp Current stock1          | Is Stock is Low               | # Material Issue Request and Approval                                                                                                 |
| Is Stock is Low               | If                  | Branch to send low stock alert if applicable                     | Low stock Detection1           | Low Stock Email Alert          | # Material Issue Request and Approval                                                                                                 |
| Low Stock Email Alert         | Gmail               | Send low stock alert email                                      | Is Stock is Low               | -                             | # Material Issue Request and Approval                                                                                                 |
| Create Record Issue           | Supabase            | Insert issue record into Supabase                               | Verify Requested Quantity      | Search Product ID             | # Material Issue Request and Approval                                                                                                 |
| Search Product ID             | Supabase            | Get stock records from Supabase                                  | Create Record Issue            | Merge Lookups                 | # Material Issue Request and Approval                                                                                                 |
| Search Issue by Submission ID | Supabase            | Get issue records by Submission ID                               | Verify Approval Data           | Combine Issue Lookup Branches  | # Material Issue Request and Approval                                                                                                 |
| Combine Issue Lookup Branches | Merge               | Combine issue lookup branches                                    | Retrieve Issue Request Details, Search Issue by Submission ID | Process Approval Decision    | # Material Issue Request and Approval                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Set Up Credentials**  
- Create and store Google Sheets credentials with Sheets API access.  
- Create and store Supabase credentials (API URL and key).  
- Create and store Gmail credentials with Gmail API access.  

---

**Step 2: Raw Materials Receiving and Stock Update Flow**

1. **Create Webhook node:**  
   - Name: Receive Raw Materials Webhook  
   - HTTP Method: POST  
   - Path: `/Pb-raw-materials`  
   - Response Headers: `Content-Type: application/json`  

2. **Create Set node:**  
   - Name: Standardize Raw Material Data  
   - Map webhook body fields (e.g., `product_id` to `Product ID`, `quantity_received` to `Quantity Received`, etc.)  
   - Include a `Timestamp` field (use `{{$json.body.timestamp}}` or current ISO date if missing)  

3. **Create Code node:**  
   - Name: Calculate Total Price  
   - JavaScript: Parse and validate `Quantity Received` and `Unit Price` fields, compute `Total Price`, throw errors if invalid.  

4. **Create Google Sheets Append node:**  
   - Name: Append Raw Materials  
   - Target Sheet: "Raw Materials"  
   - Document: Your Google Sheet ID  
   - Configure auto-mapping of standardized fields.  
   - Assign Google Sheets credentials.

5. **Create Code node:**  
   - Name: Validate Quantity Received  
   - Logic: Ensure quantity is > 0, else throw error.  

6. **Create Google Sheets Lookup node:**  
   - Name: Lookup Existing Stock  
   - Sheet: "Current Stock"  
   - Filter by `Product ID` from validated data.  

7. **Create If node:**  
   - Name: Check If Product ID Exists  
   - Condition: `Product ID` exists (not empty) in lookup result.  

8. **Create Code node (True branch):**  
   - Name: Calculate Updated Current Stock  
   - Logic: Add `Quantity Received` to `Current Stock`.  

9. **Create Google Sheets Update node (True branch):**  
   - Name: Update Current Stock  
   - Update row matching `Product ID` in "Current Stock" sheet with updated stock and timestamp.  

10. **Create Supabase Update node (True branch):**  
    - Name: Current Stock Update  
    - Update `Current Stock` table with new stock data.  

11. **Create Google Sheets Lookup node (True branch):**  
    - Name: LookUp Current stock  
    - Retrieve updated stock data for low stock check.  

12. **Create Code node:**  
    - Name: Low stock Detection2  
    - Logic: Compare `Current Stock` to `Minimum Stock Level` (default 50), set `Is Low` flag and alert message.  

13. **Create If node:**  
    - Name: Trigger Low Stock Alert  
    - Condition: `Is Low` == true  

14. **Create Gmail node:**  
    - Name: Send Low Stock Email Alert  
    - Configure recipient email, subject, and HTML message template with stock details.  

15. **Create Item Lists node:**  
    - Name: Format response  
    - Remove duplicates from Supabase search results.  

16. **Create Google Sheets Append node (False branch):**  
    - Name: Initialize New Product stock  
    - Append new product row with initial stock and minimum stock level 50.  

17. **Create Supabase Insert node (False branch):**  
    - Name: New Row Current Stock  
    - Insert new stock record for product.  

18. **Create Supabase Insert node:**  
    - Name: New Record Raw  
    - Insert raw material receipt record in Supabase.  

19. **Create Merge node:**  
    - Name: Merge  
    - Combine True and False branches of product existence check.  

---

**Step 3: Material Issue Request and Approval Flow**

1. **Create Webhook node:**  
   - Name: Receive Issue Request  
   - HTTP Method: POST  
   - Path: `/raw-materials-issue`  

2. **Create Set node:**  
   - Name: Standardize Data  
   - Map webhook body fields to standard fields, add `Approval Link` using submission ID, set `Status` to "Pending".  

3. **Create Code node:**  
   - Name: Validate Issue Request Data  
   - Check `Quantity Requested` > 0.  

4. **Create Code node:**  
   - Name: Verify Requested Quantity  
   - Check presence of `Product ID` and `Submission ID` and valid quantity.  

5. **Create Google Sheets Append node:**  
   - Name: Append Material Request  
   - Append issue request to "Materials Issued" sheet.  

6. **Create Google Sheets Lookup node:**  
   - Name: Check Available Stock for Issue  
   - Lookup "Current Stock" by `Product ID`.  

7. **Create Supabase Search node:**  
   - Name: Search Product ID  
   - Lookup Supabase `Current Stock` by `Product ID`.  

8. **Create Merge node:**  
   - Name: Merge Lookups  
   - Combine Google Sheets and Supabase stock lookups.  

9. **Create Code node:**  
   - Name: Prepare Approval  
   - Check if stock is sufficient (`Is Enough` flag).  

10. **Create Gmail node:**  
    - Name: Send Approval Request  
    - Send email with approval request details and Approve/Reject buttons linking to approval webhook URL.  

11. **Create Webhook node:**  
    - Name: Get Approvals  
    - HTTP Method: GET or POST (matching links)  
    - Path: `/approve-issue`  

12. **Create Set node:**  
    - Name: Format Approval Response  
    - Extract query params: `submissionId`, `action`, `quantity`, `approvedBy`, add `Approval Date`.  

13. **Create Code node:**  
    - Name: Verify Approval Data  
    - Validate `Action` as approve/reject and approved quantity for approval.  

14. **Create Google Sheets Lookup node:**  
    - Name: Retrieve Issue Request Details  
    - Lookup original request by `Submission ID` in "Materials Issued".  

15. **Create If node:**  
    - Name: Process Approval Decision  
    - Branch by approval: `Action` == "approve" (True) or "reject" (False).  

16. **True branch:**  
    - Lookup current stock (Google Sheets)  
    - Code node: Update Stock (Calculate new stock after deduction)  
    - Google Sheets Update node: Update Stock After Issue  
    - Google Sheets Lookup node: Retrieve Stock After Issue  
    - Code node: Low Stock Detection1  
    - If node: Is Stock is Low  
    - Gmail node: Send Low Stock Email Alert (if low)  
    - Supabase Update node: Materials Issue Table Update  
    - Supabase Update node: Update Current Stock  

17. **False branch:**  
    - Update stock request status as "Rejected" in Google Sheets and Supabase (Materials Issue Table Update node).  

18. **Create Supabase Insert node:**  
    - Name: Create Record Issue  
    - Insert issue request record in Supabase.  

19. **Create Supabase Lookup node:**  
    - Name: Search Issue by Submission ID  

20. **Create Merge node:**  
    - Name: Combine Issue Lookup Branches  
    - Combine issue data from Google Sheets and Supabase.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Diagram image labeled "INVENTORY AUTOMATION SYSTEM.png" visually illustrating workflow logic, useful for understanding flow and dependencies.                                                                                                                                                                                                                                                                                                                                                                                        | Workflow overview image (not included here)                                                                     |
| Approval email HTML template uses responsive design with gradient buttons for Approve/Reject, including dynamic fields populated from workflow data. It can be customized in the Send Approval Request Gmail node.                                                                                                                                                                                                                                                                                                                     | Approval email template in Send Approval Request node                                                           |
| Low stock alert emails have styled HTML notifications with reorder link placeholder (`https://forms.gle/reorder-form`) which should be customized to actual reorder form URL.                                                                                                                                                                                                                                                                                                                                                     | Low Stock Email Alert & Send Low Stock Email Alert Gmail nodes                                                  |
| Minimum stock level threshold is currently set to 50 units by default; adjust in code nodes or Google Sheets as needed.                                                                                                                                                                                                                                                                                                                                                                                                             | Low Stock Detection code nodes and stock initialization nodes                                                   |
| Ensure API credentials for Google Sheets, Supabase, and Gmail are configured securely in n8n credentials and assigned to respective nodes to avoid runtime errors.                                                                                                                                                                                                                                                                                                                                                                  | Setup Instructions section                                                                                       |
| Approval webhook URLs must exactly match links in approval emails including query parameters (`submissionId`, `action`, `quantity`, `approvedBy`) for proper response handling.                                                                                                                                                                                                                                                                                                                                                      | Send Approval Request and Get Approvals webhook nodes                                                           |
| Error handling is implemented via code nodes throwing exceptions on invalid data which stops workflow execution and provides clear error messages in n8n UI or logs.                                                                                                                                                                                                                                                                                                                                                                | Code nodes Validate Quantity Received, Verify Requested Quantity, Verify Approval Data                           |
| Workflow assumes Google Sheets sheets named "Raw Materials", "Current Stock", and "Materials Issued" exist with appropriate columns matching node configurations.                                                                                                                                                                                                                                                                                                                                                                   | Google Sheets nodes parameters                                                                                   |
| Supabase tables `Raw Materials`, `Current Stock`, and `Materials Issued` must exist with columns matching the fields used in workflow nodes.                                                                                                                                                                                                                                                                                                                                                                                      | Supabase nodes parameters                                                                                        |

---

This document fully describes the workflow structure, node functions, configurations, and instructions to reproduce it from scratch. It enables developers and AI agents to understand, modify, or extend the inventory management automation confidently.