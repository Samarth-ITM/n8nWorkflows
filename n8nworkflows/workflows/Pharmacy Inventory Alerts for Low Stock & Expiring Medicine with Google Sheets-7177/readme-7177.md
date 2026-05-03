Pharmacy Inventory Alerts for Low Stock & Expiring Medicine with Google Sheets

https://n8nworkflows.xyz/workflows/pharmacy-inventory-alerts-for-low-stock---expiring-medicine-with-google-sheets-7177


# Pharmacy Inventory Alerts for Low Stock & Expiring Medicine with Google Sheets

### 1. Workflow Overview

This workflow automates daily monitoring of a pharmacy's medicine inventory by integrating Google Sheets data with alert generation and email notifications. It targets pharmacy operations teams aiming to maintain optimal stock levels and minimize risks associated with expired medicines.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow daily at 9 AM.
- **1.2 Data Retrieval:** Fetches current inventory data from a designated Google Sheet.
- **1.3 Data Synchronization:** Ensures all fetched data is fully available before analysis.
- **1.4 Inventory Analysis:** Processes stock quantities and expiration dates to detect low stock and near-expiry/expired medicines.
- **1.5 Data Update:** Records alert summaries and timestamps back into the Google Sheet.
- **1.6 Alert Notification:** Sends formatted email alerts to the pharmacist with actionable inventory warnings.
- **1.7 Documentation:** Provides a sticky note summarizing workflow operation and features for user reference.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Automatically triggers the workflow once per day at 9 AM to perform inventory checks.
- **Nodes Involved:** 
  - Daily Stock Check
- **Node Details:**

  - **Node Name:** Daily Stock Check  
  - **Type:** Cron Trigger  
  - **Role:** Initiates workflow execution based on scheduled time  
  - **Configuration:** Triggers once daily at 09:00 hours  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Activates "Fetch Stock Data" node  
  - **Failure Modes:** Cron misconfiguration or system downtime could delay triggering, but minimal risk otherwise  
  - **Version Requirements:** Compatible with n8n cron trigger node v1  

#### 1.2 Data Retrieval

- **Overview:** Retrieves the pharmacy's inventory data from a Google Sheet using a service account authentication.
- **Nodes Involved:** 
  - Fetch Stock Data
- **Node Details:**

  - **Node Name:** Fetch Stock Data  
  - **Type:** Google Sheets (Read)  
  - **Role:** Reads data from a specified sheet and range to provide inventory data for processing  
  - **Configuration:**  
    - Sheet ID parameterized as `{{your_google_sheet_id}}` (must be replaced with actual ID)  
    - Range set to `PharmacyInventory!A:E` (columns A to E in the sheet named PharmacyInventory)  
    - Authentication via Google Service Account credentials  
  - **Inputs:** Triggered by "Daily Stock Check"  
  - **Outputs:** Provides data to "Wait For All Data" node  
  - **Failure Modes:**  
    - Authentication failures if service account credentials are invalid or expired  
    - Invalid sheet ID or permission errors  
    - Empty or malformed data in sheet could cause downstream issues  
  - **Version Requirements:** Google Sheets node v2  

#### 1.3 Data Synchronization

- **Overview:** Waits for complete data retrieval before starting processing to avoid partial or inconsistent data analysis.
- **Nodes Involved:** 
  - Wait For All Data
- **Node Details:**

  - **Node Name:** Wait For All Data  
  - **Type:** Wait node  
  - **Role:** Ensures all upstream data is collected before forwarding to processing logic  
  - **Configuration:** Default parameters (wait indefinitely until data arrives)  
  - **Inputs:** Receives data from "Fetch Stock Data"  
  - **Outputs:** Passes data to "Check Expiry Date and Low Stock"  
  - **Failure Modes:** Workflow stalls if upstream data never arrives (e.g., "Fetch Stock Data" failure)  
  - **Version Requirements:** Wait node v1.1  

#### 1.4 Inventory Analysis

- **Overview:** Analyzes each medicine’s stock quantity and expiry date to identify low stock or expiry alerts, compiling a summary for notification.
- **Nodes Involved:** 
  - Check Expiry Date and Low Stock
- **Node Details:**

  - **Node Name:** Check Expiry Date and Low Stock  
  - **Type:** Code (JavaScript)  
  - **Role:** Custom JS code node that iterates inventory items to generate alerts and email summary  
  - **Configuration:**  
    - Reads all input items from Google Sheets  
    - Skips header rows or empty entries  
    - Uses default low stock threshold of 10 units if none specified per item  
    - Flags medicines with stock quantity ≤ threshold as low stock  
    - Checks expiry dates and flags medicines expiring within 30 days or already expired  
    - Compiles two lists: low stock items and expiry alerts, formatted for email  
  - **Key Expressions:**  
    - Uses `$input.all()` to access all input items  
    - Constructs alert messages as strings with medicine name, stock, and expiry info  
    - Produces a JSON object summarizing alerts and timestamp  
  - **Inputs:** Data from "Wait For All Data"  
  - **Outputs:**  
    - Summary object for email content  
    - Individual alert objects for logging or further use  
  - **Failure Modes:**  
    - Errors in parsing dates or missing fields may cause runtime exceptions  
    - Unexpected data format could break alert logic  
  - **Version Requirements:** Code node v2  

#### 1.5 Data Update

- **Overview:** Updates the Google Sheet to reflect the current alert status and timestamps, ensuring the inventory record is synchronized with alert generation.
- **Nodes Involved:** 
  - Update Google Sheet
- **Node Details:**

  - **Node Name:** Update Google Sheet  
  - **Type:** Google Sheets (Update)  
  - **Role:** Writes back to the inventory sheet to update alert status after analysis  
  - **Configuration:**  
    - Target sheet and range: `PharmacyInventory!A:E`  
    - Operation: Update existing rows  
    - Authentication: Service Account same as "Fetch Stock Data"  
  - **Inputs:** Receives alert data from "Check Expiry Date and Low Stock"  
  - **Outputs:** Passes control to "Send Email Alert"  
  - **Failure Modes:**  
    - Authorization errors if credentials lack write permissions  
    - Conflicts if Google Sheet is edited concurrently  
  - **Version Requirements:** Google Sheets node v2  

#### 1.6 Alert Notification

- **Overview:** Sends an email to the pharmacist listing medicines with low stock or nearing expiry, including instructions for action.
- **Nodes Involved:** 
  - Send Email Alert
- **Node Details:**

  - **Node Name:** Send Email Alert  
  - **Type:** Email Send  
  - **Role:** Dispatches plain-text email alerts to the pharmacist  
  - **Configuration:**  
    - Subject includes current date dynamically via `{{ $now.format('YYYY-MM-DD') }}`  
    - Recipient: `pharmacist@sunrisepharmacy.com`  
    - Sender: `inventory-alerts@sunrisepharmacy.com`  
    - Email body dynamically inserts low stock and expiry lists from JSON inputs  
    - SMTP credentials configured for secure email sending  
  - **Inputs:** Alert summary from "Update Google Sheet"  
  - **Outputs:** None (end node)  
  - **Failure Modes:**  
    - SMTP authentication failures or network issues preventing email delivery  
    - Missing or invalid email addresses cause sending errors  
  - **Version Requirements:** Email Send node v2.1  

#### 1.7 Documentation

- **Overview:** Provides an inline sticky note summarizing the workflow’s daily operation and key features for users working in the n8n editor.
- **Nodes Involved:** 
  - Sticky Note
- **Node Details:**

  - **Node Name:** Sticky Note  
  - **Type:** Sticky Note  
  - **Role:** Informational; no functional impact  
  - **Configuration:**  
    - Content includes bullet points on daily trigger, data fetch, processing, update, email alert  
    - Highlights features such as threshold monitoring, expiry tracking, automated notifications, and Google Sheets integration  
  - **Inputs/Outputs:** None  
  - **Failure Modes:** N/A  
  - **Version Requirements:** Sticky Note node v1  

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                    | Input Node(s)          | Output Node(s)                | Sticky Note                                                                                           |
|---------------------------|---------------------|----------------------------------|-----------------------|------------------------------|-----------------------------------------------------------------------------------------------------|
| Daily Stock Check          | Cron Trigger        | Scheduled daily workflow trigger | None                  | Fetch Stock Data              |                                                                                                     |
| Fetch Stock Data           | Google Sheets (Read)| Retrieves inventory data         | Daily Stock Check      | Wait For All Data             |                                                                                                     |
| Wait For All Data          | Wait                | Synchronizes data before analysis| Fetch Stock Data       | Check Expiry Date and Low Stock|                                                                                                     |
| Check Expiry Date and Low Stock | Code (JavaScript)  | Analyzes stock and expiry alerts | Wait For All Data      | Update Google Sheet           |                                                                                                     |
| Update Google Sheet        | Google Sheets (Update)| Updates sheet with alert info    | Check Expiry Date and Low Stock | Send Email Alert              |                                                                                                     |
| Send Email Alert           | Email Send          | Sends alert email to pharmacist  | Update Google Sheet    | None                         |                                                                                                     |
| Sticky Note               | Sticky Note         | Documentation and workflow summary| None                  | None                         | 📌 Automated Pharmacy Inventory Monitoring Workflow: Daily stock check, alerts, email, Google Sheets integration |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node:**  
   - Name: "Daily Stock Check"  
   - Schedule it to trigger daily at 09:00 (hour set to 9)  
   - No inputs, output connects to next node  

2. **Add a Google Sheets node (Read):**  
   - Name: "Fetch Stock Data"  
   - Operation: Read  
   - Sheet ID: set parameter `{{your_google_sheet_id}}` (replace with actual ID)  
   - Range: `PharmacyInventory!A:E`  
   - Authentication: Use Google API Service Account credentials (configure or import credential)  
   - Connect input from "Daily Stock Check"  

3. **Add a Wait node:**  
   - Name: "Wait For All Data"  
   - Default configuration (waits for data)  
   - Connect input from "Fetch Stock Data"  

4. **Add a Code node:**  
   - Name: "Check Expiry Date and Low Stock"  
   - Language: JavaScript  
   - Paste the provided JS code to process inventory items, check thresholds, expiry dates, and prepare alert summaries  
   - Connect input from "Wait For All Data"  

5. **Add a Google Sheets node (Update):**  
   - Name: "Update Google Sheet"  
   - Operation: Update  
   - Sheet ID: same as step 2 (`{{your_google_sheet_id}}`)  
   - Range: `PharmacyInventory!A:E`  
   - Authentication: same Google Service Account credentials as in step 2  
   - Connect input from "Check Expiry Date and Low Stock"  

6. **Add an Email Send node:**  
   - Name: "Send Email Alert"  
   - Email Format: Plain Text  
   - To: `pharmacist@sunrisepharmacy.com`  
   - From: `inventory-alerts@sunrisepharmacy.com`  
   - Subject: `Pharmacy Inventory Alert - Low Stock & Expiry Notice - {{ $now.format('YYYY-MM-DD') }}`  
   - Body: Use the template with dynamic insertion of `{{ $json.lowStockList }}` and `{{ $json.expiryList }}` for alert lists  
   - Credentials: Configure SMTP credentials for email sending  
   - Connect input from "Update Google Sheet"  

7. **Add a Sticky Note node:**  
   - Name: "Sticky Note"  
   - Content: Add the provided summary content describing workflow purpose, daily operation, and key features for user reference  
   - Position it near the nodes for clarity  

8. **Connect all nodes as per the order:**  
   - Daily Stock Check → Fetch Stock Data → Wait For All Data → Check Expiry Date and Low Stock → Update Google Sheet → Send Email Alert  

9. **Save and activate the workflow:**  
   - Ensure Google Sheets service account has read/write access to the target spreadsheet  
   - Verify SMTP credentials for email sending  
   - Replace placeholder sheet ID with your actual Google Sheet ID  
   - Test with sample data in the sheet to confirm alerts and emails trigger correctly  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                 | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| The workflow provides a 30-day advance warning for medicine expiry to ensure proactive stock management and patient safety.                                                                                  | Business logic detail                                                                                         |
| Email alerts include detailed instructions for pharmacists to restock or remove expired items, promoting operational clarity.                                                                                | Email notification content                                                                                     |
| Google Sheets integration uses service account authentication for secure and automated access without manual OAuth prompts.                                                                                 | Google Sheets API best practices                                                                              |
| Sticky Note node offers an at-a-glance summary within the n8n editor to aid maintenance and onboarding.                                                                                                       | n8n editor usability                                                                                           |
| Replace `{{your_google_sheet_id}}` with the actual Google Sheet ID for live operation.                                                                                                                       | Setup requirement                                                                                             |
| SMTP credentials must be configured with valid email server details for sending alerts reliably.                                                                                                             | Email infrastructure setup                                                                                     |
| Ensure timezone settings in n8n or cron node match pharmacy operational hours to prevent timing mismatches.                                                                                                  | Scheduling best practice                                                                                       |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow constructed with n8n, an integration and automation tool. This processing strictly adheres to prevailing content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and publicly available.