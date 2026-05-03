Google Sheets and QuickBooks Expenses Automation Template

https://n8nworkflows.xyz/workflows/google-sheets-and-quickbooks-expenses-automation-template-7870


# Google Sheets and QuickBooks Expenses Automation Template

### 1. Workflow Overview

This n8n workflow automates the process of synchronizing and uploading categorized expense transactions from Google Sheets into QuickBooks Online (QBO). It is aimed primarily at small business owners, accountants, and bookkeepers who want to streamline expense management by leveraging spreadsheets for data entry and QuickBooks for accounting.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Initialization**: Trigger and initial data retrieval from Google Sheets.
- **1.2 Vendors Synchronization**: Importing new vendors from Google Sheets, removing duplicates, creating vendors in QBO, and refreshing vendor data in Google Sheets.
- **1.3 Chart of Accounts Synchronization**: Fetching active accounts from QBO and updating Google Sheets accordingly.
- **1.4 Expense Processing & Upload**: Retrieving new expenses from Google Sheets, filtering valid entries, uploading expenses to QBO, and updating Google Sheets with transaction IDs or error messages.
- **1.5 Realm ID Setup**: Setting the QuickBooks Realm ID used for API calls.
- **1.6 Documentation & User Guidance**: A sticky note providing detailed instructions, links, and explanations for users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initialization

- **Overview:**  
  This block initializes the workflow execution via manual trigger and fetches new vendors and sets the QuickBooks Realm ID before proceeding to other data retrieval and processing steps.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get New Vendors from Google Sheets  
  - Set Realm ID for Custom API Call

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow manually.  
    - *Configuration:* Default, no parameters.  
    - *Input/Output:* No inputs; outputs to "Get New Vendors from Google Sheets" and "Set Realm ID for Custom API Call".  
    - *Edge Cases:* None, manual start only.

  - **Get New Vendors from Google Sheets**  
    - *Type:* Google Sheets  
    - *Role:* Reads vendors data from a specific sheet named "Vendors" in the connected Google Sheets document.  
    - *Configuration:* Reads all rows without filters.  
    - *Input:* From manual trigger.  
    - *Output:* To "Remove Duplicates".  
    - *Credential:* Google Sheets OAuth2.  
    - *Edge Cases:* Google Sheets authentication failure, empty sheet, or connectivity issues.

  - **Set Realm ID for Custom API Call**  
    - *Type:* Set  
    - *Role:* Sets a variable `realmID` used for custom QuickBooks API calls.  
    - *Configuration:* Assigns an empty string by default; expected to be configured before running.  
    - *Input:* From manual trigger.  
    - *Output:* To "Get Chart of Accounts" and "Get New Expense Transactions".  
    - *Edge Cases:* If `realmID` is not set correctly, subsequent API calls will fail.

---

#### 2.2 Vendors Synchronization

- **Overview:**  
  This block handles importing new vendors from Google Sheets, removing duplicates, creating missing vendors in QuickBooks, and refreshing vendor data back into Google Sheets.

- **Nodes Involved:**  
  - Remove Duplicates  
  - Create New Vendors in QuickBooks  
  - Get Active Vendors in QuickBooks  
  - Refresh Vendors in Google Sheet Template

- **Node Details:**

  - **Remove Duplicates**  
    - *Type:* Remove Duplicates  
    - *Role:* Removes duplicate vendor entries based on the "Name" field to avoid creating duplicate vendors in QBO.  
    - *Configuration:* Compares "Name" field, outputs unique records.  
    - *Input:* From "Get New Vendors from Google Sheets".  
    - *Output:* To "Create New Vendors in QuickBooks".  
    - *Edge Cases:* If vendor names differ slightly, duplicates may not be detected.

  - **Create New Vendors in QuickBooks**  
    - *Type:* QuickBooks  
    - *Role:* Creates new vendor entries in QuickBooks Online for vendors not yet present.  
    - *Configuration:* Creates "vendor" resource with the "Name" field from input JSON.  
    - *Input:* From "Remove Duplicates".  
    - *Output:* On success, triggers "Get Active Vendors in QuickBooks" to refresh vendor list; on error, continues without interrupting workflow.  
    - *Credential:* QuickBooks OAuth2.  
    - *Edge Cases:* API errors, duplicate vendor creation errors, authentication failures.

  - **Get Active Vendors in QuickBooks**  
    - *Type:* QuickBooks  
    - *Role:* Retrieves all active vendors from QuickBooks Online to update Google Sheets.  
    - *Configuration:* `getAll` operation with no filters, returns all vendors.  
    - *Input:* From "Create New Vendors in QuickBooks".  
    - *Output:* To "Refresh Vendors in Google Sheet Template".  
    - *Credential:* QuickBooks OAuth2.  
    - *Edge Cases:* Timeout or API quota limits.

  - **Refresh Vendors in Google Sheet Template**  
    - *Type:* Google Sheets  
    - *Role:* Updates the "Vendors" sheet in Google Sheets with the latest vendor data from QuickBooks.  
    - *Configuration:* Uses "ID" as matching column, updates or appends vendor name and ID.  
    - *Input:* From "Get Active Vendors in QuickBooks".  
    - *Output:* None (end of this branch).  
    - *Credential:* Google Sheets OAuth2.  
    - *Edge Cases:* Google Sheets API rate limits, permission issues.

---

#### 2.3 Chart of Accounts Synchronization

- **Overview:**  
  This block fetches the list of active accounts from QuickBooks Online and updates a designated sheet in Google Sheets to keep account data current.

- **Nodes Involved:**  
  - Get Chart of Accounts  
  - Split Out Accounts  
  - Add Accounts to Google Sheet Template

- **Node Details:**

  - **Get Chart of Accounts**  
    - *Type:* HTTP Request  
    - *Role:* Executes a custom API query to QuickBooks Online to retrieve active accounts with a maximum of 500 entries.  
    - *Configuration:* Uses QuickBooks OAuth2 credentials; queries "select * from Account where active=true maxResults 500" with minor version 75.  
    - *Input:* From "Set Realm ID for Custom API Call".  
    - *Output:* To "Split Out Accounts".  
    - *Edge Cases:* API limits, authentication errors, malformed queries.

  - **Split Out Accounts**  
    - *Type:* Split Out  
    - *Role:* Splits the array of accounts from the API response into individual items for further processing.  
    - *Configuration:* Splits on `QueryResponse.Account`.  
    - *Input:* From "Get Chart of Accounts".  
    - *Output:* To "Add Accounts to Google Sheet Template".  
    - *Edge Cases:* Empty or null account list.

  - **Add Accounts to Google Sheet Template**  
    - *Type:* Google Sheets  
    - *Role:* Updates the "Accounts" sheet with account ID, name, and type, appending or updating entries based on ID.  
    - *Configuration:* Matches on "ID" column, mapping ID, Name, and Account Type fields.  
    - *Input:* From "Split Out Accounts".  
    - *Output:* None (end of this branch).  
    - *Credential:* Google Sheets OAuth2.  
    - *Edge Cases:* API rate limits, permission issues.

---

#### 2.4 Expense Processing & Upload

- **Overview:**  
  This block processes new expense transactions from Google Sheets, filters out empty or invalid entries, uploads valid expenses to QuickBooks, and updates Google Sheets with transaction IDs or error messages.

- **Nodes Involved:**  
  - Get New Expense Transactions  
  - Remove Empties (If)  
  - Add an Expense to QBO (HTTP Request)  
  - Record Txn ID in Google Sheets  
  - Record Error Message in Google Sheets

- **Node Details:**

  - **Get New Expense Transactions**  
    - *Type:* Google Sheets  
    - *Role:* Reads new expense transaction data from the "Expenses" sheet in Google Sheets.  
    - *Configuration:* No filters applied; reads all entries.  
    - *Input:* From "Set Realm ID for Custom API Call".  
    - *Output:* To "Remove Empties".  
    - *Credential:* Google Sheets OAuth2.  
    - *Edge Cases:* Empty sheet, connectivity issues.

  - **Remove Empties**  
    - *Type:* If  
    - *Role:* Filters out rows where "Transaction ID" is empty but "Vendor" is not empty—i.e., selects new expenses not yet uploaded.  
    - *Configuration:* Condition: Transaction ID is empty AND Vendor is not empty.  
    - *Input:* From "Get New Expense Transactions".  
    - *Output:* True path to "Add an Expense to QBO". False path discarded.  
    - *Edge Cases:* Incorrect field names or data formatting.

  - **Add an Expense to QBO**  
    - *Type:* HTTP Request  
    - *Role:* Sends a POST request to QuickBooks API to create a purchase (expense) record.  
    - *Configuration:*  
      - URL uses dynamic realmID from "Set Realm ID" node.  
      - JSON body constructs expense with fields: PaymentType (Cash), TxnDate formatted to "yyyy-MM-dd", PrivateNote (Description), EntityRef (Vendor ID), AccountRef (Asset Account ID), and expense line with amount and Expense Account ID.  
      - Minor version 75 used.  
      - On error, continues workflow without stopping.  
    - *Input:* From "Remove Empties".  
    - *Output:* Two parallel outputs: success path to "Record Txn ID in Google Sheets", error path to "Record Error Message".  
    - *Credential:* QuickBooks OAuth2.  
    - *Edge Cases:* API errors, invalid data, authentication failure, rate limiting.

  - **Record Txn ID in Google Sheets**  
    - *Type:* Google Sheets  
    - *Role:* Updates the "Expenses" sheet with the QuickBooks Transaction ID returned after successful expense creation, matching by row number (#).  
    - *Configuration:* Matches on "#" column, updates "Transaction ID".  
    - *Input:* From successful output of "Add an Expense to QBO".  
    - *Output:* None (end of successful upload branch).  
    - *Credential:* Google Sheets OAuth2.  
    - *Edge Cases:* Google Sheets update failure, concurrency issues.

  - **Record Error Message in Google Sheets**  
    - *Type:* Google Sheets  
    - *Role:* Appends or updates the "Expenses" sheet with error messages for failed expense uploads, matching by row number (#).  
    - *Configuration:* Matches on "#" column, updates "Message" column with error info.  
    - *Input:* From error output of "Add an Expense to QBO".  
    - *Output:* None (end of error branch).  
    - *Credential:* Google Sheets OAuth2.  
    - *Edge Cases:* Error message format issues, Google Sheets API errors.

---

#### 2.5 Realm ID Setup

- **Overview:**  
  A single node sets the QuickBooks Realm ID value which is essential for constructing API URLs for custom calls.

- **Nodes Involved:**  
  - Set Realm ID for Custom API Call (also linked elsewhere)

- **Node Details:**  
  - Covered above in 2.1.

---

#### 2.6 Documentation & User Guidance

- **Overview:**  
  Contains a sticky note with detailed workflow documentation, instructions, use cases, setup steps, and a link to a free Google Sheets template.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - *Type:* Sticky Note  
    - *Role:* Provides comprehensive documentation and user guidance embedded directly in the workflow canvas.  
    - *Content Highlights:*  
      - Purpose and functionality overview  
      - Prerequisites including credential setup  
      - Stepwise explanation of workflow execution  
      - Use cases for small businesses and accounting professionals  
      - Setup instructions for Google Sheets and QuickBooks integration  
      - Link to free Google Sheets template:  
        https://docs.google.com/spreadsheets/d/1dmkXHeMghVp5AHrdyU1vrwjUHWNaoDfkk9UOuG-SNKI/edit?usp=sharing  
      - Customization options and benefits  
    - *Input/Output:* None  
    - *Edge Cases:* None, purely informational.

---

### 3. Summary Table

| Node Name                       | Node Type         | Functional Role                              | Input Node(s)                         | Output Node(s)                              | Sticky Note                                                                                      |
|--------------------------------|-------------------|----------------------------------------------|-------------------------------------|---------------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger    | Start workflow manually                      | -                                   | Get New Vendors from Google Sheets, Set Realm ID for Custom API Call | See detailed documentation in Sticky Note node                                                |
| Get New Vendors from Google Sheets | Google Sheets    | Reads new vendors data from Google Sheets   | When clicking ‘Execute workflow’    | Remove Duplicates                           |                                                                                              |
| Remove Duplicates               | Remove Duplicates  | Removes duplicate vendors by Name            | Get New Vendors from Google Sheets   | Create New Vendors in QuickBooks            |                                                                                              |
| Create New Vendors in QuickBooks | QuickBooks        | Creates new vendors in QuickBooks             | Remove Duplicates                   | Get Active Vendors in QuickBooks             |                                                                                              |
| Get Active Vendors in QuickBooks | QuickBooks        | Retrieves active vendors from QuickBooks     | Create New Vendors in QuickBooks    | Refresh Vendors in Google Sheet Template    |                                                                                              |
| Refresh Vendors in Google Sheet Template | Google Sheets    | Updates Vendors sheet in Google Sheets       | Get Active Vendors in QuickBooks    | -                                           |                                                                                              |
| Set Realm ID for Custom API Call | Set               | Sets QuickBooks Realm ID for API calls       | When clicking ‘Execute workflow’    | Get Chart of Accounts, Get New Expense Transactions |                                                                                              |
| Get Chart of Accounts           | HTTP Request       | Fetches active accounts from QuickBooks      | Set Realm ID for Custom API Call    | Split Out Accounts                          |                                                                                              |
| Split Out Accounts             | Split Out          | Splits accounts array into individual items  | Get Chart of Accounts               | Add Accounts to Google Sheet Template       |                                                                                              |
| Add Accounts to Google Sheet Template | Google Sheets    | Updates Accounts sheet in Google Sheets      | Split Out Accounts                 | -                                           |                                                                                              |
| Get New Expense Transactions    | Google Sheets      | Reads new expense transactions from Google Sheets | Set Realm ID for Custom API Call    | Remove Empties                             |                                                                                              |
| Remove Empties                 | If                 | Filters out empty or incomplete expenses     | Get New Expense Transactions        | Add an Expense to QBO                      |                                                                                              |
| Add an Expense to QBO           | HTTP Request       | Creates expense purchase in QuickBooks       | Remove Empties                     | Record Txn ID in Google Sheets, Record Error Message |                                                                                              |
| Record Txn ID in Google Sheets  | Google Sheets      | Updates Transactions sheet with QuickBooks Txn ID | Add an Expense to QBO (success)    | -                                           |                                                                                              |
| Record Error Message            | Google Sheets      | Logs error messages back to Google Sheets    | Add an Expense to QBO (error)      | -                                           |                                                                                              |
| Sticky Note                    | Sticky Note        | Provides workflow documentation and guidance | -                                   | -                                           | Contains detailed instructions, use cases, setup steps, and a link to free Google Sheets template |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: To manually start the workflow.

2. **Add Google Sheets Node to Get New Vendors**  
   - Name: `Get New Vendors from Google Sheets`  
   - Operation: Read rows from sheet named "Vendors"  
   - Document: Link your Google Sheets document (credential required)  
   - Output: Connect from manual trigger.

3. **Add Remove Duplicates Node**  
   - Name: `Remove Duplicates`  
   - Fields to compare: `Name`  
   - Input: Connect from `Get New Vendors from Google Sheets`.

4. **Add QuickBooks Node to Create New Vendors**  
   - Name: `Create New Vendors in QuickBooks`  
   - Resource: `vendor`  
   - Operation: `create`  
   - Display Name set to `={{ $json.Name }}`  
   - Credential: Connect your QuickBooks OAuth2 credential.  
   - Input: Connect from `Remove Duplicates`.  
   - On error: Set to continue without stopping.

5. **Add QuickBooks Node to Get Active Vendors**  
   - Name: `Get Active Vendors in QuickBooks`  
   - Resource: `vendor`  
   - Operation: `getAll`  
   - Return all: true  
   - Credential: Use same QuickBooks OAuth2.  
   - Input: Connect from `Create New Vendors in QuickBooks`.

6. **Add Google Sheets Node to Refresh Vendors Sheet**  
   - Name: `Refresh Vendors in Google Sheet Template`  
   - Operation: Append or update  
   - Sheet Name: "Vendors" (same as step 2)  
   - Matching column: `ID`  
   - Columns mapped: `ID` from QuickBooks Id, `Name` from DisplayName  
   - Input: Connect from `Get Active Vendors in QuickBooks`.

7. **Add Set Node to Define Realm ID**  
   - Name: `Set Realm ID for Custom API Call`  
   - Create string variable `realmID` (set to your QuickBooks Realm ID)  
   - Input: Connect from manual trigger.

8. **Add HTTP Request Node to Get Chart of Accounts**  
   - Name: `Get Chart of Accounts`  
   - Method: GET (default)  
   - URL: `https://sandbox-quickbooks.api.intuit.com/v3/company/{{ $json.realmID }}/query`  
   - Query parameters:  
     - `query`: `select * from Account where active=true maxResults 500`  
     - `minorversion`: `75`  
   - Authentication: Use QuickBooks OAuth2 credential.  
   - Input: Connect from `Set Realm ID for Custom API Call`.

9. **Add Split Out Node**  
   - Name: `Split Out Accounts`  
   - Field to split out: `QueryResponse.Account`  
   - Input: Connect from `Get Chart of Accounts`.

10. **Add Google Sheets Node to Add Accounts**  
    - Name: `Add Accounts to Google Sheet Template`  
    - Operation: Append or update  
    - Sheet Name: "Accounts"  
    - Matching column: `ID`  
    - Columns mapped: `ID`, `Name`, `Account Type` from QuickBooks account data  
    - Input: Connect from `Split Out Accounts`.

11. **Add Google Sheets Node to Get New Expense Transactions**  
    - Name: `Get New Expense Transactions`  
    - Operation: Read rows from "Expenses" sheet  
    - Input: Connect from `Set Realm ID for Custom API Call`.

12. **Add If Node to Remove Empties**  
    - Name: `Remove Empties`  
    - Condition:  
      - `Transaction ID` is empty AND  
      - `Vendor` is not empty  
    - Input: Connect from `Get New Expense Transactions`.

13. **Add HTTP Request Node to Add Expense to QBO**  
    - Name: `Add an Expense to QBO`  
    - Method: POST  
    - URL: `https://sandbox-quickbooks.api.intuit.com/v3/company/{{ $('Set Realm ID for Custom API Call').item.json.realmID }}/purchase`  
    - Body type: JSON  
    - JSON Body: Construct with fields from input JSON:  
      - `PaymentType`: `"Cash"`  
      - `TxnDate`: format date as `yyyy-MM-dd` from `Date` field  
      - `PrivateNote`: from `Description`  
      - `EntityRef.value`: from `Vendor ID`  
      - `AccountRef.value`: from `Asset ID`  
      - `Line`: array with one item detailing amount and expense account reference.  
    - Query Parameters: `minorversion=75`  
    - Authentication: QuickBooks OAuth2  
    - On error: Continue workflow (do not stop).  
    - Input: Connect from `Remove Empties`.

14. **Add Google Sheets Node to Record Transaction ID**  
    - Name: `Record Txn ID in Google Sheets`  
    - Operation: Update  
    - Sheet Name: "Expenses"  
    - Matching column: `#` (row number)  
    - Columns to update: `Transaction ID` from QuickBooks response Purchase.Id  
    - Input: Connect from success output of `Add an Expense to QBO`.

15. **Add Google Sheets Node to Record Error Message**  
    - Name: `Record Error Message`  
    - Operation: Append or update  
    - Sheet Name: "Expenses"  
    - Matching column: `#`  
    - Columns to update: `Message` with error details from response  
    - Input: Connect from error output of `Add an Expense to QBO`.

16. **Connect Workflow**  
    - Connect nodes according to the structure explained in Section 1 and 2, ensuring proper flow from manual trigger through vendors and accounts synchronization, then processing expenses.

17. **Credentials Setup**  
    - Set up Google Sheets OAuth2 credentials with access to the relevant Google Sheets document.  
    - Set up QuickBooks OAuth2 credentials with required scopes to read and write vendors, accounts, and expenses.

18. **Set Realm ID**  
    - Obtain the QuickBooks Realm ID from your QuickBooks Online developer account and enter it in the `Set Realm ID for Custom API Call` node.

19. **Test Workflow**  
    - Manually trigger the workflow to ensure smooth execution and correct data synchronization.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                            |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| This n8n workflow template automates uploading categorized expenses from Google Sheets into QuickBooks Online, saving time and reducing errors for small businesses and accounting professionals. It requires setting up credentials and configuring the Google Sheets document appropriately. The workflow includes comprehensive documentation embedded as a sticky note node for easy reference.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Embedded documentation sticky note inside the workflow.                                                                    |
| Free Google Sheets Template provided includes pre-configured sheets for bank transactions, vendors, and chart of accounts to simplify initial setup and usage.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | [Google Sheets Template](https://docs.google.com/spreadsheets/d/1dmkXHeMghVp5AHrdyU1vrwjUHWNaoDfkk9UOuG-SNKI/edit?usp=sharing) |
| Workflow benefits include automation of expense uploads, error reduction, enhanced efficiency, accuracy in financial reporting, and flexibility allowing spreadsheet-based categorization without granting direct QuickBooks access.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Workflow description and advantages as per sticky note documentation.                                                      |

---

**Disclaimer:** The provided content is exclusively derived from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.