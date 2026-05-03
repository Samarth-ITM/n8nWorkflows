Monitor Construction Stock & Send Low Inventory Alerts with Google Sheets

https://n8nworkflows.xyz/workflows/monitor-construction-stock---send-low-inventory-alerts-with-google-sheets-6636


# Monitor Construction Stock & Send Low Inventory Alerts with Google Sheets

### 1. Workflow Overview

This workflow automates the monitoring of construction stock inventory by integrating Google Sheets as the data source and sending email alerts when stock levels fall below defined thresholds. It is designed primarily for construction project managers or inventory teams to maintain sufficient stock levels and avoid operational disruptions.

The workflow operates through the following logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a specific time to perform the stock check.
- **1.2 Stock Data Retrieval:** Fetches current stock inventory data from a Google Sheet.
- **1.3 Stock Level Update:** Processes any additions or usage of stock to update quantities.
- **1.4 Low Stock Detection:** Identifies any items whose stock levels are equal to or below their threshold.
- **1.5 Data Persistence:** Updates the Google Sheet with the latest stock quantities.
- **1.6 Notification Dispatch:** Sends an email alert to the inventory team listing low-stock items.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
Schedules the workflow to run once daily at 8 AM, ensuring automatic monitoring without manual intervention.

- **Nodes Involved:**  
  - Daily Stock Check

- **Node Details:**  
  - **Daily Stock Check**  
    - Type: Cron Trigger  
    - Configuration: Set to trigger at hour 8 (8 AM) every day.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Fetch Stock Data" node.  
    - Edge Cases: Cron misconfiguration could cause no runs; time zone considerations may affect timing.  

#### 2.2 Stock Data Retrieval

- **Overview:**  
Retrieves the current stock inventory data from a specified Google Sheet tab ("StockInventory") covering columns A to E.

- **Nodes Involved:**  
  - Fetch Stock Data

- **Node Details:**  
  - **Fetch Stock Data**  
    - Type: Google Sheets (Read operation)  
    - Configuration: Reads range "StockInventory!A:E" from a Google Sheet identified by a dynamic sheet ID placeholder `{{your_google_sheet_id}}`.  
    - Authentication: Uses Google Service Account credentials.  
    - Inputs: Triggered by "Daily Stock Check"  
    - Outputs: Passes data to "Update Stock Levels"  
    - Edge Cases:  
      - Invalid or expired credentials will cause auth errors.  
      - Incorrect sheet ID or range will cause read failure or empty data.  
      - Large data volume may cause timeouts or API limits.  

#### 2.3 Stock Level Update

- **Overview:**  
Processes stock additions or deductions based on predefined input actions (currently hardcoded for demonstration) and updates each item's quantity and last updated timestamp.

- **Nodes Involved:**  
  - Update Stock Levels

- **Node Details:**  
  - **Update Stock Levels**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Reads all stock items from input.  
      - Applies updates from a static array of stock changes (e.g., adding 50 units of Cement, using 10 units of Paint).  
      - Ensures quantities do not fall below zero.  
      - Updates `last_updated` field with current ISO timestamp.  
    - Inputs: Data from "Fetch Stock Data"  
    - Outputs: Updated stock array, forwarded to "Check Low Stock"  
    - Expressions/Variables: Uses `$input.all()` to fetch incoming data; manipulates JavaScript objects.  
    - Edge Cases:  
      - Hardcoded stockUpdates array means no dynamic input without modification.  
      - If the stockUpdates list does not match items in inventory, no update occurs for those missing.  
      - Failure in code execution (syntax errors) would halt workflow.  

#### 2.4 Low Stock Detection

- **Overview:**  
Scans updated stock data to find items with quantities less than or equal to their defined threshold (default threshold 20 if none specified).

- **Nodes Involved:**  
  - Check Low Stock

- **Node Details:**  
  - **Check Low Stock**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Iterates over each item, comparing quantity vs threshold.  
      - Collects low-stock items along with alert messages.  
      - Returns an array of low-stock items with relevant details.  
    - Inputs: Updated stock data from "Update Stock Levels"  
    - Outputs: List of low-stock items to "Update Google Sheet"  
    - Expressions/Variables: Uses `$input.all()`; constructs alert_message strings dynamically.  
    - Edge Cases:  
      - Missing threshold data defaults to 20, may not suit all items.  
      - If no items meet criteria, output is empty array; subsequent nodes must handle this gracefully.  

#### 2.5 Data Persistence

- **Overview:**  
Updates the Google Sheet to reflect the latest stock quantities and timestamps after processing.

- **Nodes Involved:**  
  - Update Google Sheet

- **Node Details:**  
  - **Update Google Sheet**  
    - Type: Google Sheets (Update operation)  
    - Configuration:  
      - Updates range "StockInventory!A:E" in the same Google Sheet.  
      - Uses the same Service Account credentials as before.  
    - Inputs: Low stock items from "Check Low Stock" (though logically this should be updated stock data — see note below)  
    - Outputs: Passes data to "Send Email Alert"  
    - Edge Cases:  
      - Updates can fail if data format mismatches the sheet or if API limits are exceeded.  
      - Concurrent edits in the Sheet during update might cause conflicts or data loss.  

    **Note:** Based on the connections, "Check Low Stock" passes data to "Update Google Sheet". However, logically the sheet update should reflect the full updated stock, not just low-stock items. This could be a design oversight or simplification.

#### 2.6 Notification Dispatch

- **Overview:**  
Sends an email alert listing all low-stock items to a designated inventory team email address.

- **Nodes Involved:**  
  - Send Email Alert

- **Node Details:**  
  - **Send Email Alert**  
    - Type: Email Send  
    - Configuration:  
      - Subject includes current date (formatted `YYYY-MM-DD`).  
      - Email text body includes a greeting, explanation, and lists low-stock items from the JSON input as `{{ $json.lowStockList }}`.  
      - Recipient: inventory@realestatecompany.com  
      - Sender: alerts@realestatecompany.com  
      - Email format: Plain text  
    - Credentials: SMTP authentication configured separately.  
    - Inputs: Data from "Update Google Sheet"  
    - Edge Cases:  
      - Sending fails if SMTP credentials are invalid or server unreachable.  
      - If low stock list is empty, email content might be confusing or empty.  
      - Email body relies on correct JSON path for low stock list; mismatch may cause missing data.  

#### 2.7 Documentation Note

- **Overview:**  
Provides a sticky note summarizing the workflow blocks for user clarity.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Sticky Note**  
    - Type: Sticky Note (Visual aid)  
    - Content: Summarizes the workflow with bullet points describing each block.  
    - Position: Separate from data flow, used for user reference only.  
    - No inputs or outputs.  

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role               | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                   |
|---------------------|-------------------------|------------------------------|------------------------|----------------------|----------------------------------------------------------------------------------------------|
| Daily Stock Check    | Cron Trigger            | Scheduled daily workflow start | None                   | Fetch Stock Data      | 📌 Workflow Overview: Daily Stock Check triggers the workflow daily at 8 AM.                 |
| Fetch Stock Data     | Google Sheets (Read)    | Retrieve current stock data   | Daily Stock Check       | Update Stock Levels   | 📌 Fetch Stock Data retrieves stock levels from Google Sheets.                               |
| Update Stock Levels  | Code (JavaScript)       | Process stock additions/usage| Fetch Stock Data        | Check Low Stock       | 📌 Updates stock quantities based on predefined stock updates.                              |
| Check Low Stock      | Code (JavaScript)       | Identify low stock items      | Update Stock Levels     | Update Google Sheet   | 📌 Detects items below threshold for alerting.                                              |
| Update Google Sheet  | Google Sheets (Update)  | Persist updated stock data    | Check Low Stock         | Send Email Alert      | 📌 Saves updated inventory back to Google Sheets.                                           |
| Send Email Alert     | Email Send              | Send low stock alert emails   | Update Google Sheet     | None                 | 📌 Sends notification emails to inventory team with low stock details.                      |
| Sticky Note         | Sticky Note             | Documentation and overview    | None                   | None                 | 📌 Workflow Overview summarizes all workflow blocks.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger Node**  
   - Name: Daily Stock Check  
   - Type: Cron Trigger  
   - Parameters: Set the trigger time to daily at 8:00 AM.  
   - No credentials needed.

2. **Create a Google Sheets Node (Read Operation)**  
   - Name: Fetch Stock Data  
   - Type: Google Sheets  
   - Set Operation to "Read"  
   - Parameters:  
     - Sheet ID: Enter your Google Sheet ID (replace `{{your_google_sheet_id}}`).  
     - Range: "StockInventory!A:E"  
   - Authentication: Use Google Service Account credentials configured with read access to the sheet.  
   - Connect the output of "Daily Stock Check" to the input of this node.

3. **Create a Code Node (JavaScript)**  
   - Name: Update Stock Levels  
   - Type: Code  
   - Parameters: Paste the JavaScript code that iterates over input items and applies stock additions or usage.  
   - Note: Replace the hardcoded `stockUpdates` array with dynamic input if needed.  
   - Connect the output of "Fetch Stock Data" to this node.

4. **Create a Code Node (JavaScript)**  
   - Name: Check Low Stock  
   - Type: Code  
   - Parameters: Paste JavaScript code that compares each stock quantity against its threshold (default 20 if missing) and returns low-stock items with alert messages.  
   - Connect the output of "Update Stock Levels" to this node.

5. **Create a Google Sheets Node (Update Operation)**  
   - Name: Update Google Sheet  
   - Type: Google Sheets  
   - Set Operation to "Update"  
   - Parameters:  
     - Sheet ID: The same Google Sheet ID as before.  
     - Range: "StockInventory!A:E"  
   - Authentication: Use the same Google Service Account credentials.  
   - Connect the output of "Check Low Stock" to this node.

6. **Create an Email Send Node**  
   - Name: Send Email Alert  
   - Type: Email Send  
   - Parameters:  
     - To Email: inventory@realestatecompany.com  
     - From Email: alerts@realestatecompany.com  
     - Subject: "Low Stock Alert - {{ $now.format('YYYY-MM-DD') }}"  
     - Text: Use the provided template including `{{ $json.lowStockList }}` to list low-stock items.  
     - Email Format: Plain text  
   - Credentials: Configure SMTP credentials with a valid SMTP server.  
   - Connect the output of "Update Google Sheet" to this node.

7. **Add a Sticky Note (Optional)**  
   - Name: Sticky Note  
   - Type: Sticky Note  
   - Content: Copy the overview content summarizing all blocks for documentation and user clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| The workflow depends on a Google Sheet named "StockInventory" with columns A to E containing stock items and thresholds. | Sheet structure should be consistent for proper data handling.     |
| SMTP credentials must be valid and tested to ensure email delivery is successful.                                      | SMTP server setup instructions depend on your email provider.      |
| The hardcoded stock update array in the "Update Stock Levels" node should be replaced with dynamic input for production use. | Consider integrating form input, API, or webhook for real updates. |
| Time zone considerations for the Cron Trigger may require adjustment depending on server location and requirements.  | n8n Cron triggers run in server time zone by default.              |
| To avoid errors, ensure Google Service Account has proper permissions to read and update the specified Google Sheet.  | Refer to Google API documentation for Service Account setup.       |
| Email body template uses n8n expressions, ensure correct JSON paths for dynamic data insertion.                        | Documentation: https://docs.n8n.io/nodes/n8n-nodes-base.email-send/ |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.