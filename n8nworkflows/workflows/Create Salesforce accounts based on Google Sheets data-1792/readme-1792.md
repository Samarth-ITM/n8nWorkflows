Create Salesforce accounts based on Google Sheets data

https://n8nworkflows.xyz/workflows/create-salesforce-accounts-based-on-google-sheets-data-1792


# Create Salesforce accounts based on Google Sheets data

### 1. Workflow Overview

This workflow automates the creation and updating of Salesforce accounts and contacts based on data imported from a Google Sheet. It targets users who want to synchronize customer or prospect data stored in Google Sheets with Salesforce CRM without coding.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Trigger manual execution and read contact data from a specified Google Sheet.
- **1.2 Salesforce Account Search:** For each company name from the sheet, query Salesforce to check if the account exists.
- **1.3 Account Branching & Creation:** Separate new companies (not found in Salesforce) and existing ones. Create new Salesforce accounts for new companies.
- **1.4 Data Normalization and Merging:** Normalize and merge account data from both new and existing companies.
- **1.5 Contact Upsert:** Create or update Salesforce contacts linked to the appropriate accounts.

This structure ensures that only missing accounts are created, while existing accounts are reused, and all contact data is appropriately linked.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Starts the workflow manually and reads data from a Google Sheet containing company and contact information.

**Nodes Involved:**  
- On clicking 'execute'  
- Read Google Sheet

**Node Details:**

- **On clicking 'execute':**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow execution on demand.  
  - Configuration: Default; no parameters required.  
  - Input: None (manual trigger)  
  - Output: Triggers the next node.

- **Read Google Sheet:**  
  - Type: Google Sheets  
  - Role: Retrieves rows from a specified Google Sheet that contains companies and contact data.  
  - Configuration:  
    - Sheet ID set to `"1cz-4tVi7Nn3j1gh147hROq9l6S4ta06sMfhm2AAI6js"` (replaceable by user).  
    - Credentials: OAuth2 for Google Sheets authenticated as "Tom's Google Sheets account".  
  - Key Expressions: None (reads raw rows).  
  - Input: Trigger from manual node.  
  - Output: Data rows to be processed further.  
  - Potential failures: Authentication errors, invalid sheet ID, empty or malformed sheets.

---

#### 2.2 Salesforce Account Search

**Overview:**  
For each company name from the Google Sheet, searches Salesforce to find matching accounts by name.

**Nodes Involved:**  
- Search Salesforce accounts

**Node Details:**

- **Search Salesforce accounts:**  
  - Type: Salesforce node, resource: search  
  - Role: Queries Salesforce for existing accounts matching the company name.  
  - Configuration:  
    - SOQL query: `SELECT id, Name FROM Account WHERE Name = '{{Company Name with escaped single quotes}}'`  
    - Uses expression to escape single quotes in company names to avoid SOQL injection or syntax errors.  
    - Credentials: Salesforce OAuth2 ("Salesforce account").  
  - Input: Data rows from Google Sheets.  
  - Output: Salesforce account records corresponding to each row or empty if none found.  
  - Potential failures: Salesforce auth failure, query timeout, malformed queries due to unexpected characters in company names.

---

#### 2.3 Account Branching & Creation

**Overview:**  
Splits the data into two branches: one for companies not found in Salesforce (to be created) and one for existing companies (to be updated). Removes duplicates before creation.

**Nodes Involved:**  
- Keep new companies  
- Merge existing account data  
- Remove duplicate companies  
- Create Salesforce account

**Node Details:**

- **Keep new companies:**  
  - Type: Merge node, mode: removeKeyMatches  
  - Role: Filters companies that do not have a matching Salesforce account by comparing "Company Name" (from Google Sheets) with "Name" (from Salesforce).  
  - Input: From Search Salesforce accounts node (Google Sheet data and Salesforce search results).  
  - Output: Companies that are new (no matching account found).  
  - Edge cases: Case sensitivity issues could cause false positives/negatives.

- **Merge existing account data:**  
  - Type: Merge node, mode: mergeByKey  
  - Role: Joins Google Sheet data with existing Salesforce account data on company name to prepare for contact upsert.  
  - Input: From Search Salesforce accounts node outputs.  
  - Output: Combined data for existing accounts.

- **Remove duplicate companies:**  
  - Type: Item Lists node, operation: removeDuplicates by "Company Name"  
  - Role: Ensures no duplicate account creation by filtering unique company names.  
  - Input: Output from Keep new companies (new companies only).  
  - Output: List of unique new companies.

- **Create Salesforce account:**  
  - Type: Salesforce node, resource: account  
  - Role: Creates new Salesforce accounts for companies identified as new.  
  - Configuration:  
    - Name field set dynamically from "Company Name".  
    - Credentials: Salesforce OAuth2.  
  - Input: Unique new companies only.  
  - Output: Newly created Salesforce account records.  
  - Potential failures: Auth errors, Salesforce API limits, invalid company names.

---

#### 2.4 Data Normalization and Merging

**Overview:**  
Prepares and merges company and contact data from both new and existing accounts to unify downstream contact creation.

**Nodes Involved:**  
- Set new account name  
- Retrieve new company contacts  
- Set Account ID for existing accounts

**Node Details:**

- **Set new account name:**  
  - Type: Set node  
  - Role: Normalizes the newly created account data by setting fields `id` and `Name` with the Salesforce account ID and company name, respectively.  
  - Input: Output from Create Salesforce account node and Remove duplicate companies.  
  - Output: Normalized account data for merging.  
  - Key Expressions:  
    - `id` set to newly created account’s `id`.  
    - `Name` set from "Company Name" node output.  

- **Retrieve new company contacts:**  
  - Type: Merge node, mode: mergeByKey  
  - Role: Combines normalized new account data with their corresponding contacts from the Google Sheet by matching company names.  
  - Input: From Set new account name (new account IDs) and Remove duplicate companies (original contacts).  
  - Output: Merged data containing new account IDs and contact details for contact creation.

- **Set Account ID for existing accounts:**  
  - Type: Rename Keys node  
  - Role: Renames Salesforce `Id` field to `Account ID` for consistency with new accounts.  
  - Input: From Account found? conditional node (existing account data).  
  - Output: Existing accounts with `Account ID` field set.  
  - Potential failures: Missing Id fields if Salesforce query failed.

---

#### 2.5 Contact Upsert

**Overview:**  
Creates or updates Salesforce contacts using the merged contact data and associated account IDs.

**Nodes Involved:**  
- Create Salesforce contact  
- Account found? (conditional node)

**Node Details:**

- **Account found?:**  
  - Type: If node  
  - Role: Checks if the Salesforce account exists by inspecting if `Id` field is present and not empty.  
  - Input: Merge existing account data node output.  
  - Output: Routes to either existing account processing or new account processing.  
  - Edge cases: Missing or empty Id fields could cause false negatives.

- **Create Salesforce contact:**  
  - Type: Salesforce node, resource: contact, operation: upsert  
  - Role: Creates or updates contacts in Salesforce linked to correct accounts.  
  - Configuration:  
    - Upsert by external ID "Email".  
    - Fields: `lastname`, `email`, `firstName`, and `accountId` (linked account).  
    - Expression-based mapping from Google Sheet data and processed account IDs.  
    - Credentials: Salesforce OAuth2.  
  - Input: Merged contact data from both new and existing account branches (converged).  
  - Output: Salesforce contact records created or updated.  
  - Potential failures: Missing email (external ID), validation errors, Salesforce API limits.

---

### 3. Summary Table

| Node Name                  | Node Type                | Functional Role                                  | Input Node(s)                     | Output Node(s)                 | Sticky Note                                                                                                                             |
|----------------------------|--------------------------|-------------------------------------------------|----------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'       | Manual Trigger           | Starts workflow execution                        | None                             | Read Google Sheet              |                                                                                                                                         |
| Read Google Sheet           | Google Sheets            | Reads contact and company data from Google Sheet| On clicking 'execute'             | Search Salesforce accounts, Keep new companies, Merge existing account data | Requires Google Sheets OAuth2 authentication. Replace sheet ID accordingly.                                                             |
| Search Salesforce accounts  | Salesforce (search)      | Searches for existing Salesforce accounts       | Read Google Sheet                 | Keep new companies, Merge existing account data | Escapes single quotes in company names to avoid SOQL errors. Requires Salesforce OAuth2 authentication.                                 |
| Keep new companies          | Merge (removeKeyMatches) | Filters new companies not found in Salesforce   | Search Salesforce accounts        | Remove duplicate companies, Retrieve new company contacts |                                                                                                                                        |
| Merge existing account data | Merge (mergeByKey)       | Merges existing Salesforce account data with sheet data | Search Salesforce accounts        | Account found?                |                                                                                                                                        |
| Remove duplicate companies  | Item Lists (removeDuplicates) | Removes duplicate company names before creation | Keep new companies                | Create Salesforce account      |                                                                                                                                        |
| Create Salesforce account   | Salesforce (account)     | Creates new Salesforce accounts                  | Remove duplicate companies        | Set new account name           | Requires Salesforce OAuth2 authentication.                                                                                             |
| Set new account name        | Set                      | Normalizes new account data                       | Create Salesforce account         | Retrieve new company contacts  |                                                                                                                                        |
| Retrieve new company contacts| Merge (mergeByKey)      | Merges new accounts with related contacts       | Set new account name, Remove duplicate companies | Create Salesforce contact     |                                                                                                                                        |
| Account found?              | If                       | Checks if account exists in Salesforce           | Merge existing account data       | Set Account ID for existing accounts |                                                                                                                                        |
| Set Account ID for existing accounts | Rename Keys        | Renames Salesforce account Id to Account ID     | Account found?                   | Create Salesforce contact      |                                                                                                                                        |
| Create Salesforce contact   | Salesforce (contact)     | Creates or updates Salesforce contacts           | Retrieve new company contacts, Set Account ID for existing accounts | None                         | Upserts contacts by Email. Requires Salesforce OAuth2 authentication. Note typo in field 'acconuntId' (should be 'accountId').          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `On clicking 'execute'`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Add Google Sheets Node:**  
   - Name: `Read Google Sheet`  
   - Type: Google Sheets (v2)  
   - Set **Sheet ID** to your actual Google Sheet ID (e.g., `"1cz-4tVi7Nn3j1gh147hROq9l6S4ta06sMfhm2AAI6js"`).  
   - Connect credentials for Google Sheets OAuth2.  
   - Connect output from Manual Trigger.

3. **Add Salesforce Search Node:**  
   - Name: `Search Salesforce accounts`  
   - Type: Salesforce, resource: Search  
   - Set query as:  
     ```sql
     SELECT id, Name FROM Account WHERE Name = '{{$json["Company Name"].replace(/'/g, "\\'")}}'
     ```  
   - Use Salesforce OAuth2 credentials.  
   - Connect input from `Read Google Sheet`.

4. **Add Merge Node to Keep New Companies:**  
   - Name: `Keep new companies`  
   - Type: Merge, mode: removeKeyMatches  
   - Set `propertyName1` to "Company Name" (Google Sheet) and `propertyName2` to "Name" (Salesforce).  
   - Connect input from `Search Salesforce accounts` (two inputs: original data and Salesforce search results).

5. **Add Merge Node for Existing Accounts:**  
   - Name: `Merge existing account data`  
   - Type: Merge, mode: mergeByKey  
   - Set keys: `propertyName1` = "Company Name", `propertyName2` = "Name".  
   - Connect input from `Search Salesforce accounts` (both outputs).

6. **Add Item Lists Node to Remove Duplicate Companies:**  
   - Name: `Remove duplicate companies`  
   - Type: Item Lists  
   - Operation: removeDuplicates  
   - Compare by field: "Company Name".  
   - Connect input from `Keep new companies`.

7. **Add Salesforce Node to Create Accounts:**  
   - Name: `Create Salesforce account`  
   - Type: Salesforce, resource: Account, operation: Create  
   - Set field `Name` to `{{$json["Company Name"]}}`.  
   - Use Salesforce OAuth2 credentials.  
   - Connect input from `Remove duplicate companies`.

8. **Add Set Node to Normalize New Account Data:**  
   - Name: `Set new account name`  
   - Type: Set  
   - Set fields:  
     - `id` = `{{$json["id"]}}` (from created account)  
     - `Name` = `{{$node["Remove duplicate companies"].json["Company Name"]}}`  
   - Connect input from `Create Salesforce account`.

9. **Add Merge Node to Retrieve New Company Contacts:**  
   - Name: `Retrieve new company contacts`  
   - Type: Merge, mode: mergeByKey  
   - Set keys: `propertyName1` = "Company Name", `propertyName2` = "Name".  
   - Connect inputs from `Set new account name` (new accounts) and `Remove duplicate companies` (original contacts).

10. **Add If Node to Check Account Existence:**  
    - Name: `Account found?`  
    - Type: If  
    - Condition: Check if `$json["Id"]` is not empty.  
    - Connect input from `Merge existing account data`.

11. **Add Rename Keys Node to Set Account ID:**  
    - Name: `Set Account ID for existing accounts`  
    - Type: Rename Keys  
    - Rename key: from `Id` to `Account ID`.  
    - Connect input from `Account found?` (true branch).

12. **Add Salesforce Node to Upsert Contacts:**  
    - Name: `Create Salesforce contact`  
    - Type: Salesforce, resource: Contact, operation: Upsert  
    - External ID field: `Email`  
    - External ID value: `{{$json["Email"]}}`  
    - Fields:  
      - `LastName`: `{{$json["Last Name"]}}`  
      - `Email`: `{{$json["Email"]}}`  
      - `FirstName`: `{{$json["First Name"]}}`  
      - `AccountId`: `{{$json["Account ID"]}}` (note correction from `acconuntId`)  
    - Use Salesforce OAuth2 credentials.  
    - Connect inputs from:  
      - `Retrieve new company contacts` (for new accounts)  
      - `Set Account ID for existing accounts` (for existing accounts)  

13. **Connect Nodes per Workflow Logic:**  
    - `On clicking 'execute'` → `Read Google Sheet`  
    - `Read Google Sheet` → `Search Salesforce accounts`  
    - `Search Salesforce accounts` → `Keep new companies` and `Merge existing account data`  
    - `Keep new companies` → `Remove duplicate companies` → `Create Salesforce account` → `Set new account name`  
    - `Set new account name` and `Remove duplicate companies` → `Retrieve new company contacts` → `Create Salesforce contact`  
    - `Merge existing account data` → `Account found?` → `Set Account ID for existing accounts` → `Create Salesforce contact`  

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------|
| To run the workflow, ensure both Google Sheets and Salesforce credentials are authenticated in n8n first.      | Workflow description instructions                                   |
| Google Sheet ID must be copied from the sheet URL and set correctly in the Google Sheets node.                  | Screenshot reference in original workflow description               |
| Salesforce query escapes single quotes in company names to avoid SOQL syntax errors.                             | Important note on query construction                                |
| The contact creation node has a typo in the field name: `acconuntId` should be corrected to `accountId`.        | Potential source of failure in contact creation                     |
| The workflow is designed to be triggered manually but can be adapted to scheduled or webhook triggers.           | General operational note                                            |

---

This detailed reference fully covers the workflow’s structure, node roles, configurations, error points, and reproduction steps, enabling advanced users and automation agents to understand, recreate, and enhance the process confidently.