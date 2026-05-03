Sync Companies from Google Sheets to Salesforce with Smart Duplicate Prevention

https://n8nworkflows.xyz/workflows/sync-companies-from-google-sheets-to-salesforce-with-smart-duplicate-prevention-9053


# Sync Companies from Google Sheets to Salesforce with Smart Duplicate Prevention

### 1. Workflow Overview

This workflow automates syncing company data from Google Sheets to Salesforce with intelligent duplicate detection and prevention. It is designed to:

- Read company records from a Google Sheets document.
- Search Salesforce Accounts by company name to detect existing entries.
- Separate companies into two logical paths:
  - Existing companies: update contacts under the already existing Salesforce account.
  - New companies: create new Salesforce accounts and related contacts.
- Prevent duplicates both within the input data and in Salesforce by checking existing records before creation.
- Maintain comprehensive contact management tied to accounts.

Logical blocks:

- **1.1 Input Reception & Preparation:** Manual trigger and Google Sheets data retrieval.
- **1.2 Existing Account Search & Data Merge:** Query Salesforce for existing accounts, merge with sheet data.
- **1.3 Duplicate Detection & Routing:** Conditional routing based on search results to either existing or new company processing.
- **1.4 New Company Creation:** Remove duplicates from new companies list, create Salesforce accounts, and associate contacts.
- **1.5 Existing Company Contact Update:** Map existing Salesforce account IDs and update/add contacts accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Preparation

**Overview:**  
Starts the workflow manually and retrieves company data from Google Sheets to prepare for further processing.

**Nodes Involved:**  
- On clicking 'execute'  
- Get row(s) in sheet

**Node Details:**  

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command.  
  - Parameters: None (simple manual trigger)  
  - Connections: Output to "Get row(s) in sheet"  
  - Potential Failures: None expected unless n8n is offline or user does not trigger.

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Role: Reads rows from a specified Google Sheets document and sheet.  
  - Configuration:  
    - `documentId` and `sheetName` must be set to the target Google Sheets file and worksheet.  
    - Credentials: Uses OAuth2 credentials ("xavtwl") for Google Sheets API access.  
  - Connections: Outputs data to "Search Salesforce accounts" and "Merge existing account data1".  
  - Edge Cases:  
    - Sheet not found or access denied (permissions).  
    - Empty or malformed rows.  
    - API rate limits from Google.  

---

#### 1.2 Existing Account Search & Data Merge

**Overview:**  
Search Salesforce Accounts for existing companies matching the names from the sheet, then merge sheet data with Salesforce data for further processing.

**Nodes Involved:**  
- Search Salesforce accounts  
- Merge existing account data1

**Node Details:**  

- **Search Salesforce accounts**  
  - Type: Salesforce node  
  - Role: Searches Salesforce Account records by exact company name matching.  
  - Configuration:  
    - Query: `SELECT id, Name FROM Account WHERE Name = '{{$json["Company Name"].replace(/'/g, '\\\'')}}'`  
    - Resource: Search (SOQL query)  
  - Connections: Outputs to "Merge existing account data1" (main output) and "Keep new companies" (secondary output).  
  - Edge Cases:  
    - No matches found (empty results).  
    - Salesforce API rate limits or authentication errors.  
    - Special characters in company names causing query injection issues (escaped in expression).  

- **Merge existing account data1**  
  - Type: Merge  
  - Role: Combines Google Sheets data (input1) with Salesforce search results (input2) by matching company names.  
  - Configuration:  
    - Mode: Combine (join)  
    - Advanced: true  
    - Merge by fields: Company Name (sheet) and Name (Salesforce)  
  - Connections: Output to "Account found?" node.  
  - Edge Cases:  
    - Missing company names in either source.  
    - Partial matches not handled (exact match required).  

---

#### 1.3 Duplicate Detection & Routing

**Overview:**  
Routes the companies based on whether an Account was found in Salesforce or not, enabling separate handling logic for existing and new companies.

**Nodes Involved:**  
- Account found?  
- Keep new companies

**Node Details:**  

- **Account found?**  
  - Type: If  
  - Role: Checks if the merged result contains a non-empty Salesforce Account ID, indicating an existing account.  
  - Condition: `$json["Id"]` is not empty  
  - Connections:  
    - True branch: to "Set Account ID for existing accounts"  
    - False branch: to "Keep new companies"  
  - Edge Cases:  
    - Missing or null Id fields could misroute companies.  

- **Keep new companies**  
  - Type: Merge  
  - Role: Filters out companies that do not have matching Salesforce accounts (non-matches) to proceed with new account creation.  
  - Configuration:  
    - Mode: Combine  
    - Join mode: Keep non-matches only  
    - Merge fields: Company Name (sheet) and Name (Salesforce)  
    - Output data from input1 (sheet data)  
  - Connections: Outputs to "Remove duplicate companies"  
  - Edge Cases:  
    - Partial or inconsistent company name formats causing misses.  

---

#### 1.4 New Company Creation

**Overview:**  
Removes duplicates within new companies, creates Salesforce accounts for them, sets account names, and retrieves their contact data.

**Nodes Involved:**  
- Remove duplicate companies  
- Create Salesforce account  
- Set new account name  
- Retrieve new company contacts

**Node Details:**  

- **Remove duplicate companies**  
  - Type: Item Lists  
  - Role: Eliminates duplicate company entries in the new companies list based on company name.  
  - Configuration:  
    - Operation: Remove duplicates  
    - Fields to compare: Company Name  
  - Connections: Outputs to "Create Salesforce account"  
  - Edge Cases:  
    - Case sensitivity or whitespace differences in company names may affect deduplication.  

- **Create Salesforce account**  
  - Type: Salesforce  
  - Role: Creates new Account records in Salesforce with the company name.  
  - Configuration:  
    - Resource: Account  
    - Name: Taken from `$json["Company Name"]`  
  - Connections: Outputs to "Set new account name"  
  - Edge Cases:  
    - Salesforce required fields missing (only Name is set here).  
    - API errors or limits.  

- **Set new account name**  
  - Type: Set  
  - Role: Sets fields `id` and `Name` for further processing, using the created account's ID and original company name.  
  - Configuration:  
    - id: `$json["id"]` (Salesforce Account ID from created record)  
    - Name: Company Name from the "Remove duplicate companies" node  
  - Connections: Outputs to "Retrieve new company contacts"  
  - Edge Cases:  
    - Missing id if account creation failed.  

- **Retrieve new company contacts**  
  - Type: Merge  
  - Role: Merges new account data with the original company contact data by company name to prepare for contact creation.  
  - Configuration:  
    - Mode: Combine  
    - Merge by fields: Company Name (sheet) and Name (Salesforce account)  
  - Connections: Outputs to "Create Salesforce contact"  
  - Edge Cases:  
    - Mismatches in company names.  

---

#### 1.5 Existing Company Contact Update

**Overview:**  
Maps existing Salesforce Account IDs to the contacts and creates or updates contacts linked to those accounts.

**Nodes Involved:**  
- Set Account ID for existing accounts  
- Create Salesforce contact

**Node Details:**  

- **Set Account ID for existing accounts**  
  - Type: Rename Keys  
  - Role: Renames the Salesforce Account `Id` field to `Account ID` to align with contact creation data structure.  
  - Configuration:  
    - Rename "Id" → "Account ID"  
  - Connections: Outputs to "Create Salesforce contact"  
  - Edge Cases:  
    - Missing or null Id fields if account not found.  

- **Create Salesforce contact**  
  - Type: Salesforce  
  - Role: Upserts contacts into Salesforce, linked to accounts either newly created or existing.  
  - Configuration:  
    - Resource: Contact  
    - Operation: Upsert by external ID "Email"  
    - Fields:  
      - lastName: `$json["Last Name"]`  
      - email: `$json["Email"]`  
      - firstName: `$json["First Name"]`  
      - accountId: `$json["Account ID"]` (note: typo in JSON as "acconuntId" should be "accountId")  
  - Connections: None (end node)  
  - Edge Cases:  
    - Salesforce required contact fields missing.  
    - Email duplicates or mismatches.  
    - Typo in field name `acconuntId` may cause failure or incorrect behavior.  
    - Salesforce API limits or auth errors.  

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                                      | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                     |
|-----------------------------|-------------------------|-----------------------------------------------------|-------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'        | Manual Trigger          | Starts workflow execution                            |                               | Get row(s) in sheet            | # 🎯 Salesforce-Google Sheets Company Sync (Smart Duplicate Prevention)<br>Made by: Xavier Tai<br>Instructions and requirements listed |
| Get row(s) in sheet          | Google Sheets           | Reads company data from Google Sheets                | On clicking 'execute'          | Search Salesforce accounts, Merge existing account data1 | # 📊 Part 1: Data Input & Preparation<br>Extract data from Google Sheets for Salesforce processing                              |
| Search Salesforce accounts   | Salesforce              | Searches Salesforce Accounts by company name         | Get row(s) in sheet            | Merge existing account data1, Keep new companies | # 📊 Part 1: Data Input & Preparation<br>Search Salesforce for existing companies                                                |
| Merge existing account data1 | Merge                   | Combines sheet data with Salesforce search results   | Get row(s) in sheet, Search Salesforce accounts | Account found?                | # 📊 Part 1: Data Input & Preparation<br>Merge data for further routing                                                         |
| Account found?               | If                      | Routes based on existence of Salesforce Account ID   | Merge existing account data1   | Set Account ID for existing accounts (true), Keep new companies (false) | # ✅ Part 4: Existing Company Processing<br>Identify existing companies                                                        |
| Keep new companies           | Merge                   | Keeps companies not found in Salesforce              | Search Salesforce accounts     | Remove duplicate companies     | # 🔄 Part 2: Smart Routing & Duplicate Detection<br>Route companies to new account creation path                               |
| Remove duplicate companies   | Item Lists              | Removes duplicate companies from new companies list  | Keep new companies             | Create Salesforce account      | # 🆕 Part 3: New Company Processing<br>Remove duplicates before creation                                                       |
| Create Salesforce account    | Salesforce              | Creates new Salesforce Account                        | Remove duplicate companies     | Set new account name           | # 🆕 Part 3: New Company Processing<br>Create new Salesforce Accounts                                                           |
| Set new account name         | Set                     | Sets ID and Name fields for new accounts             | Create Salesforce account      | Retrieve new company contacts  | # 🆕 Part 3: New Company Processing<br>Prepare data for contact association                                                    |
| Retrieve new company contacts| Merge                   | Merges new accounts with contact data                 | Set new account name, Keep new companies | Create Salesforce contact       | # 🆕 Part 3: New Company Processing<br>Associate contacts with new accounts                                                    |
| Set Account ID for existing accounts | Rename Keys       | Renames Salesforce Account Id to Account ID          | Account found?                 | Create Salesforce contact      | # ✅ Part 4: Existing Company Processing<br>Prepare existing accounts for contact creation                                      |
| Create Salesforce contact    | Salesforce              | Upserts contacts linked to accounts                   | Set Account ID for existing accounts, Retrieve new company contacts |                               | # ✅ Part 4: Existing Company Processing<br>Creates or updates Salesforce contacts<br>Note: Typo in "acconuntId" field may cause issues |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "On clicking 'execute'"  
   - No parameters needed.  

2. **Create Google Sheets Node**  
   - Type: Google Sheets  
   - Name: "Get row(s) in sheet"  
   - Parameters:  
     - Document ID: Set to your Google Sheets document containing company data.  
     - Sheet Name: Set to the specific worksheet name.  
   - Credentials: Configure OAuth2 for Google Sheets API (e.g., "xavtwl").  
   - Connect Manual Trigger output to this node input.  

3. **Create Salesforce Search Node**  
   - Type: Salesforce  
   - Name: "Search Salesforce accounts"  
   - Resource: Search  
   - Query:  
     ```
     SELECT id, Name FROM Account WHERE Name = '{{$json["Company Name"].replace(/'/g, '\\\'')}}'
     ```  
   - Connect output of Google Sheets node to this node.  

4. **Create Merge Node for Existing Account Data**  
   - Type: Merge  
   - Name: "Merge existing account data1"  
   - Mode: Combine (join)  
   - Advanced: true  
   - Merge by fields: Company Name (input1 - sheet data) and Name (input2 - Salesforce search)  
   - Connect Google Sheets node to input1 and Salesforce search node to input2.  

5. **Create If Node for Account Detection**  
   - Type: If  
   - Name: "Account found?"  
   - Condition: Check if `$json["Id"]` is not empty.  
   - Connect output of Merge node to this node.  

6. **Create Rename Keys Node (For Existing Accounts)**  
   - Type: Rename Keys  
   - Name: "Set Account ID for existing accounts"  
   - Rename key: "Id" → "Account ID"  
   - Connect true output of If node here.  

7. **Create Merge Node to Filter New Companies**  
   - Type: Merge  
   - Name: "Keep new companies"  
   - Mode: Combine  
   - Join Mode: Keep Non-Matches  
   - Merge by fields: Company Name (sheet) and Name (Salesforce)  
   - Output data from input1 (sheet data).  
   - Connect Salesforce search node output to input2 and Google Sheets output to input1.  
   - Connect false output of If node to this node.  

8. **Create Item Lists Node to Remove Duplicates**  
   - Type: Item Lists  
   - Name: "Remove duplicate companies"  
   - Operation: Remove duplicates based on "Company Name" field.  
   - Connect output of "Keep new companies" node here.  

9. **Create Salesforce Node to Create Account**  
   - Type: Salesforce  
   - Name: "Create Salesforce account"  
   - Resource: Account  
   - Set "Name" to `{{$json["Company Name"]}}`  
   - Connect output of "Remove duplicate companies" node here.  

10. **Create Set Node to Prepare New Account Data**  
    - Type: Set  
    - Name: "Set new account name"  
    - Set fields:  
      - `id` = `{{$json["id"]}}` (from created account)  
      - `Name` = Company Name from "Remove duplicate companies" node  
    - Connect output of "Create Salesforce account" node here.  

11. **Create Merge Node to Retrieve New Company Contacts**  
    - Type: Merge  
    - Name: "Retrieve new company contacts"  
    - Mode: Combine  
    - Merge by fields: Company Name (sheet) and Name (Salesforce created accounts)  
    - Connect output of "Set new account name" node to input1.  
    - Connect output of "Remove duplicate companies" node to input2.  

12. **Create Salesforce Node to Upsert Contact**  
    - Type: Salesforce  
    - Name: "Create Salesforce contact"  
    - Resource: Contact  
    - Operation: Upsert  
    - External ID: "Email"  
    - External ID Value: `{{$json["Email"]}}`  
    - Fields:  
      - lastName: `{{$json["Last Name"]}}`  
      - email: `{{$json["Email"]}}`  
      - firstName: `{{$json["First Name"]}}`  
      - accountId: `{{$json["Account ID"]}}` (Note: fix typo from "acconuntId")  
    - Connect outputs of "Set Account ID for existing accounts" and "Retrieve new company contacts" to this node as parallel inputs.  

13. **Connect Outputs**  
    - Connect Manual Trigger → Get row(s) in sheet  
    - Get row(s) in sheet → Search Salesforce accounts and Merge existing account data1  
    - Merge existing account data1 → Account found?  
    - Account found? (true) → Set Account ID for existing accounts → Create Salesforce contact  
    - Account found? (false) → Keep new companies → Remove duplicate companies → Create Salesforce account → Set new account name → Retrieve new company contacts → Create Salesforce contact  

14. **Credentials Setup**  
    - Google Sheets OAuth2 credentials for accessing the spreadsheet.  
    - Salesforce OAuth2 or API credentials with permissions to read Accounts and create/update Accounts and Contacts.  

15. **Validation & Testing**  
    - Test with small dataset, ensuring correct field mappings.  
    - Monitor API quotas and errors.  
    - Fix the typo in "Create Salesforce contact": replace `"acconuntId"` with `"accountId"` in Additional Fields.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow was created by Xavier Tai to automate syncing company data from Google Sheets to Salesforce with smart duplicate prevention and contact management. It requires Google Sheets API and Salesforce API credentials configured in n8n.                                                                                                                                                 | Workflow description and author credit.                                                                             |
| Estimated execution time per company is approximately 15-45 seconds, depending on API response times and batch sizes.                                                                                                                                                                                                                                                                       | Workflow performance note.                                                                                           |
| Security best practices include using environment variables for API keys, restricting workflow access in n8n, and testing with small datasets before full deployment.                                                                                                                                                                                                                      | Security recommendations.                                                                                           |
| Troubleshooting tips: Check OAuth tokens for authentication errors, verify required Salesforce fields, watch for duplicate creation via exact name matching, monitor API rate limits, and confirm Google Sheets permissions and sheet names.                                                                                                                                                | Troubleshooting guide.                                                                                              |
| For detailed Salesforce field mapping, ensure knowledge of required vs optional fields for Account and Contact objects in your Salesforce org.                                                                                                                                                                                                                                              | Salesforce configuration reminder.                                                                                   |
| The workflow includes sticky notes visually explaining each part of the process: data input, smart routing, new company processing, and existing company processing.                                                                                                                                                                                                                        | Workflow inline documentation via sticky notes.                                                                     |
| Link for OAuth2 Setup in Salesforce and Google Sheets API can be found in official documentation: https://developer.salesforce.com/docs and https://developers.google.com/sheets/api                                                                                                                                                                                                        | External API documentation links.                                                                                   |

---

**Disclaimer:** The provided text is extracted solely from an n8n automated workflow. It fully complies with content policies and contains no illegal or offensive material. All data processed is legal and public.