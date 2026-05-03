Daily Magento 2 Customer Sync to Google Contacts & Sheets without Duplicates

https://n8nworkflows.xyz/workflows/daily-magento-2-customer-sync-to-google-contacts---sheets-without-duplicates-6783


# Daily Magento 2 Customer Sync to Google Contacts & Sheets without Duplicates

### 1. Workflow Overview

This workflow automates the daily synchronization of new Magento 2 customers into Google Contacts and logs their email addresses in a Google Sheet, ensuring no duplicates are created in either system. It is designed for e-commerce businesses using Magento 2 who want to maintain an updated contact list and a record of synced customers without manual intervention.

The workflow is structured into the following logical blocks:

- **1.1 Schedule Trigger & Initialization:** Starts the workflow on a schedule and initializes the date for fetching new customers.
- **1.2 Data Retrieval:** Fetches existing customer emails from Google Sheets and new customer data from Magento 2 via HTTP request.
- **1.3 Data Processing & Comparison:** Splits new customers and compares their emails with existing ones to filter out duplicates.
- **1.4 Contact Creation & Logging:** Creates Google Contacts for new unique customers and logs their emails into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger & Initialization

**Overview:**  
This block triggers the workflow at scheduled intervals, retrieves the previous date to fetch new customers created since then, and starts the data retrieval process.

**Nodes Involved:**  
- Schedule Trigger  
- GET PREVIOUS DATE  

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Trigger node  
  - *Role:* Initiates the workflow daily (default schedule, likely daily though not explicitly configured here)  
  - *Configuration:* No parameters customized; uses default scheduling  
  - *Inputs:* None  
  - *Outputs:* To ‘Get Existing Emails’ and ‘GET PREVIOUS DATE’ nodes  
  - *Edge cases:* Workflow will not trigger if n8n instance is down or scheduling fails  

- **GET PREVIOUS DATE**  
  - *Type:* Code node  
  - *Role:* Calculates the date from which to fetch new customers (likely “yesterday” or last successful sync date)  
  - *Configuration:* Contains JavaScript code executed once per workflow run  
  - *Inputs:* From Schedule Trigger  
  - *Outputs:* To ‘Get New Customers’ node  
  - *Edge cases:* Code execution errors may stop the workflow; date calculation must be compatible with Magento API’s requirements  

---

#### 1.2 Data Retrieval

**Overview:**  
This block fetches the current list of existing customer emails from a Google Sheet and requests new customer data from the Magento 2 API for the period since the previous date.

**Nodes Involved:**  
- Get Existing Emails  
- Get New Customers  

**Node Details:**

- **Get Existing Emails**  
  - *Type:* Google Sheets node  
  - *Role:* Reads existing synced customer emails from a Google Sheet to avoid duplicates  
  - *Configuration:* Reads a specific Google Sheets document and range containing emails  
  - *Inputs:* From Schedule Trigger  
  - *Outputs:* To ‘Compare Datasets’ node (as first input)  
  - *Credentials:* Requires Google Sheets OAuth2 credentials  
  - *Edge cases:* Sheet access errors, credential issues, or empty sheets could cause faulty comparisons  

- **Get New Customers**  
  - *Type:* HTTP Request node  
  - *Role:* Calls the Magento 2 API to retrieve new customers created since the previous date  
  - *Configuration:* Configured with Magento 2 API endpoint, likely with query parameters for filtering by creation date  
  - *Inputs:* From ‘GET PREVIOUS DATE’ node  
  - *Outputs:* To ‘Split Customers’ node  
  - *Credentials:* Requires Magento API credentials (not shown in JSON but assumed)  
  - *Edge cases:* API authentication failure, network errors, empty responses, rate limiting  

---

#### 1.3 Data Processing & Comparison

**Overview:**  
This block splits the new customer data array into individual customers, then compares these against existing emails to filter out already synced customers.

**Nodes Involved:**  
- Split Customers  
- Compare Datasets  

**Node Details:**

- **Split Customers**  
  - *Type:* Split Out node  
  - *Role:* Splits array of new customers into individual customer items for processing  
  - *Configuration:* Default behavior splitting each item into separate workflow items  
  - *Inputs:* From ‘Get New Customers’  
  - *Outputs:* To ‘Compare Datasets’ node (as second input)  
  - *Edge cases:* If input is not an array or empty, downstream nodes may fail or have no data to process  

- **Compare Datasets**  
  - *Type:* Compare Datasets node  
  - *Role:* Compares existing emails dataset with new customers to identify unique new customers  
  - *Configuration:* Compares based on email field or unique identifier; outputs only unique entries in the fourth output (index 3)  
  - *Inputs:*  
    - Input 0: Existing Emails (from Google Sheets)  
    - Input 1: New Customers (split items)  
  - *Outputs:*  
    - Output 3: Unique new customers (not in existing emails), sent to Google Contacts creation and logging  
  - *Edge cases:* Mismatched data formats, missing email fields, or case sensitivity could cause incorrect filtering  

---

#### 1.4 Contact Creation & Logging

**Overview:**  
This block creates Google Contacts for each unique new customer and logs their email addresses into the Google Sheet to update the existing emails list.

**Nodes Involved:**  
- Create Google Contact  
- Log Synced Email  

**Node Details:**

- **Create Google Contact**  
  - *Type:* Google Contacts node  
  - *Role:* Adds a new contact to Google Contacts for each unique new customer  
  - *Configuration:* Uses customer details (name, email, phone) from dataset to populate contact fields  
  - *Inputs:* From ‘Compare Datasets’ output 3 (unique customers)  
  - *Outputs:* None (end of branch)  
  - *Credentials:* Requires Google Contacts OAuth2 credentials  
  - *Edge cases:* API quota limits, duplicate contact errors, missing required fields  

- **Log Synced Email**  
  - *Type:* Google Sheets node  
  - *Role:* Appends the newly synced customer’s email to the Google Sheet to keep the existing emails list updated  
  - *Configuration:* Appends new rows with email data to specific Google Sheet and range  
  - *Inputs:* From ‘Compare Datasets’ output 3 (unique customers)  
  - *Outputs:* None (end of branch)  
  - *Credentials:* Requires Google Sheets OAuth2 credentials  
  - *Edge cases:* Sheet write permission errors, invalid data format, concurrent writes  

---

### 3. Summary Table

| Node Name            | Node Type              | Functional Role                         | Input Node(s)           | Output Node(s)                         | Sticky Note |
|----------------------|------------------------|---------------------------------------|------------------------|--------------------------------------|-------------|
| Schedule Trigger      | scheduleTrigger        | Starts workflow on schedule            | None                   | Get Existing Emails, GET PREVIOUS DATE |             |
| GET PREVIOUS DATE     | code                   | Calculates date to fetch new customers | Schedule Trigger       | Get New Customers                    |             |
| Get Existing Emails   | googleSheets           | Reads existing synced emails           | Schedule Trigger       | Compare Datasets                     |             |
| Get New Customers     | httpRequest            | Fetches new customers from Magento API | GET PREVIOUS DATE      | Split Customers                     |             |
| Split Customers       | splitOut               | Splits customer array into single items| Get New Customers      | Compare Datasets                    |             |
| Compare Datasets      | compareDatasets        | Filters new customers to unique only   | Get Existing Emails, Split Customers | Create Google Contact, Log Synced Email |             |
| Create Google Contact | googleContacts         | Creates Google Contacts for new customers | Compare Datasets      | None                               |             |
| Log Synced Email      | googleSheets           | Logs new emails in Google Sheet        | Compare Datasets       | None                               |             |
| Sticky Note          | stickyNote             | (empty)                               | None                   | None                               |             |
| Sticky Note1         | stickyNote             | (empty)                               | None                   | None                               |             |
| Sticky Note2         | stickyNote             | (empty)                               | None                   | None                               |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger` node:**  
   - Type: Schedule Trigger  
   - Configuration: Set to run daily at desired time (default is daily). No special parameters needed.  

2. **Create `GET PREVIOUS DATE` node:**  
   - Type: Code  
   - Execute Once: Enabled  
   - Code: Implement JavaScript to compute the previous date (e.g., yesterday’s date in ISO format) to use as a filter for new customers.  
   - Connect input from `Schedule Trigger`.  

3. **Create `Get Existing Emails` node:**  
   - Type: Google Sheets (Read)  
   - Configure credentials for Google Sheets.  
   - Select the spreadsheet and range where synced emails are stored.  
   - Connect input from `Schedule Trigger`.  

4. **Create `Get New Customers` node:**  
   - Type: HTTP Request  
   - Configure credentials for Magento 2 API (OAuth or API key).  
   - Set HTTP method to GET.  
   - Set URL to Magento 2 customers endpoint with query parameter for ‘created_at’ greater than date from `GET PREVIOUS DATE`.  
   - Connect input from `GET PREVIOUS DATE`.  

5. **Create `Split Customers` node:**  
   - Type: Split Out  
   - Default configuration to split array of customer objects.  
   - Connect input from `Get New Customers`.  

6. **Create `Compare Datasets` node:**  
   - Type: Compare Datasets  
   - Configure:  
     - Dataset 1: Existing Emails (from Google Sheets)  
     - Dataset 2: New Customers (split items)  
     - Key for comparison: Customer email field (ensure matching field names)  
     - Output: Use the 4th output (unique items in dataset 2)  
   - Connect inputs from `Get Existing Emails` (Input 0) and `Split Customers` (Input 1).  

7. **Create `Create Google Contact` node:**  
   - Type: Google Contacts  
   - Configure credentials for Google Contacts OAuth2.  
   - Map customer fields (name, email, etc.) from the unique customers dataset.  
   - Connect input from 4th output of `Compare Datasets`.  

8. **Create `Log Synced Email` node:**  
   - Type: Google Sheets (Append)  
   - Configure credentials for Google Sheets.  
   - Select spreadsheet and sheet where to log emails.  
   - Append new rows with customer emails.  
   - Connect input from 4th output of `Compare Datasets`.  

9. **Connect nodes according to the flow:**  
   - `Schedule Trigger` → `Get Existing Emails` and `GET PREVIOUS DATE`  
   - `GET PREVIOUS DATE` → `Get New Customers`  
   - `Get New Customers` → `Split Customers`  
   - `Get Existing Emails` → `Compare Datasets` (Input 0)  
   - `Split Customers` → `Compare Datasets` (Input 1)  
   - `Compare Datasets` → `Create Google Contact` (Output 3)  
   - `Compare Datasets` → `Log Synced Email` (Output 3)  

10. **Test the workflow:**  
    - Manually trigger or wait for scheduled execution.  
    - Verify new Google Contacts are created only for unique new customers and emails are logged properly.  

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                               |
|--------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow uses OAuth2 credentials for Google APIs and requires Magento 2 API access credentials.    | Credential setup in n8n integrations.          |
| Ensure Magento 2 API endpoint supports filtering by customer creation date for incremental sync.        | Magento 2 API documentation.                    |
| The ‘Compare Datasets’ node requires consistent and normalized email formats for accurate comparison.   | Data normalization best practices.              |
| For large datasets, consider API rate limits and Google Sheets row limits to avoid errors or delays.    | Google API quotas and limits documentation.     |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.