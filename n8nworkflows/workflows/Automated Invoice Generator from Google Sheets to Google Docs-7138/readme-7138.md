Automated Invoice Generator from Google Sheets to Google Docs

https://n8nworkflows.xyz/workflows/automated-invoice-generator-from-google-sheets-to-google-docs-7138


# Automated Invoice Generator from Google Sheets to Google Docs

### 1. Workflow Overview

This workflow automates the generation of invoice documents by extracting invoice data from a Google Sheets spreadsheet and populating a Google Docs invoice template with this data. The final invoices are dynamically created as new Google Docs files in a specified Google Drive folder, with all placeholder fields replaced by actual invoice details.

The workflow is structured into the following logical blocks:

- **1.1 Manual Trigger & Input Acquisition:** Starts the workflow manually and loads invoice data from Google Sheets.
- **1.2 Template Loading & Document Creation:** Fetches the Google Docs invoice template and creates a new Google Doc for each invoice.
- **1.3 Data Merging & Content Insertion:** Combines template data and invoice data, inserts initial content into the new document.
- **1.4 Placeholder Replacement:** Replaces placeholders in the new document with actual invoice details from the spreadsheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger & Input Acquisition

- **Overview:**  
  This block initiates the workflow manually and loads all invoice records from a specific Google Sheets spreadsheet.

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`  
  - `Google Sheets`

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type & Role:* Manual Trigger node; initiates workflow runs on-demand.  
    - *Configuration:* No parameters required.  
    - *Expressions:* None.  
    - *Input/Output:* No input; outputs trigger signal to next nodes.  
    - *Edge Cases:* None specific; manual trigger expected to always succeed unless n8n is offline.  
    - *Sub-workflow:* None.

  - **Google Sheets**  
    - *Type & Role:* Google Sheets node; reads invoice data rows from a Google Sheets spreadsheet.  
    - *Configuration:*  
      - Spreadsheet ID: `1MHVZRVo5aPs5VqRXk7lBNPVlZ2gilKqZ8J9yeg4taW4`  
      - Sheet name: `gid=0` (Sheet1)  
      - Reads all rows with columns expected: `Company From`, `Company To`, `Terms`, `Invoice`, `Description`, `Amount`.  
    - *Expressions:* None dynamic; static sheet and document IDs.  
    - *Input/Output:* Input triggered by Manual Trigger; outputs rows of invoice data one by one.  
    - *Credentials:* Requires Google Sheets OAuth2 credentials with read access.  
    - *Edge Cases:*  
      - Authentication failure if credentials invalid or revoked.  
      - API rate limits or quota exceeded errors.  
      - Empty or malformed spreadsheet rows could cause downstream issues.  
    - *Sub-workflow:* None.

---

#### 2.2 Template Loading & Document Creation

- **Overview:**  
  Loads the static Google Docs invoice template and creates a new Google Doc copy for each invoice to be populated.

- **Nodes Involved:**  
  - `Get Invoice Template`  
  - `Create New Doc`

- **Node Details:**

  - **Get Invoice Template**  
    - *Type & Role:* Google Docs node; fetches the template document metadata.  
    - *Configuration:*  
      - Operation: `get`  
      - Document URL: `https://docs.google.com/document/d/18n0HTqabDldi7fVbhbI1aG12qbFWsjyTXdduwDDOUu8`  
    - *Expressions:* Static document URL.  
    - *Input/Output:* Triggered by Manual Trigger; outputs template document metadata including document ID.  
    - *Credentials:* Requires Google Docs OAuth2 credentials with read access.  
    - *Edge Cases:*  
      - Permission denied if credential lacks access.  
      - Document deleted or URL invalid.  
    - *Sub-workflow:* None.

  - **Create New Doc**  
    - *Type & Role:* Google Docs node; creates a new Google Doc file for each invoice by duplicating the template.  
    - *Configuration:*  
      - Operation: `create`  
      - Title: Dynamic, set as `Invoice: {{ $json.Invoice }}` using invoice number from sheet row.  
      - Folder ID: `1TnDibwPPPUm3VbmETiqWDVhtaUTLJ6mn` (destination folder in Google Drive).  
    - *Expressions:* Uses invoice number from Google Sheets node (`$json.Invoice`).  
    - *Input/Output:* Input from Google Sheets node rows; output is new document metadata including ID.  
    - *Credentials:* Google Docs OAuth2 with write access to specified Drive folder.  
    - *Edge Cases:*  
      - Permission errors if folder access denied.  
      - Title conflicts (Google Docs allows duplicates, but naming collisions could cause confusion).  
    - *Sub-workflow:* None.

---

#### 2.3 Data Merging & Content Insertion

- **Overview:**  
  Combines the template document metadata and invoice data row into one payload for easier processing, then inserts initial content into the new document.

- **Nodes Involved:**  
  - `Merge`  
  - `Insert Content into Doc`

- **Node Details:**

  - **Merge**  
    - *Type & Role:* Merge node; combines data from two input sources into one unified set.  
    - *Configuration:*  
      - Mode: `combine`  
      - Combine By: `combineAll` (merges all items from both inputs into one item per pair).  
    - *Expressions:* None.  
    - *Input/Output:*  
      - Input 1: Output from `Get Invoice Template` (template metadata).  
      - Input 2: Output from `Create New Doc` (new doc metadata).  
      - Output: Combined JSON including template and new doc info along with invoice data.  
    - *Edge Cases:*  
      - Mismatched input lengths could cause unexpected merges.  
    - *Sub-workflow:* None.

  - **Insert Content into Doc**  
    - *Type & Role:* Google Docs node; inserts initial content into the newly created document.  
    - *Configuration:*  
      - Operation: `update`  
      - Actions: Insert text from merged content field `{{ $json.content }}` at the current cursor or document start.  
      - Document URL: dynamically set as `{{ $json.id }}` from the new document ID.  
    - *Expressions:* Uses merged JSON for document ID and content.  
    - *Input/Output:* Input from `Merge` node; output carries forward updated document metadata.  
    - *Credentials:* Google Docs OAuth2 with write access.  
    - *Edge Cases:*  
      - Failure if document ID missing or invalid.  
      - API errors or timeouts.  
    - *Sub-workflow:* None.

---

#### 2.4 Placeholder Replacement

- **Overview:**  
  Replaces placeholder strings in the newly created Google Doc with actual invoice details from the spreadsheet, completing the invoice generation.

- **Nodes Involved:**  
  - `Input Invoice Details`

- **Node Details:**

  - **Input Invoice Details**  
    - *Type & Role:* Google Docs node; performs multiple text replacements to fill placeholders with real data.  
    - *Configuration:*  
      - Operation: `update`  
      - Actions: Multiple `replaceAll` actions targeting placeholders in the template:  
        - `FromCompany#` → `Company From` (sheet)  
        - `ToCompany#` → `Company To` (sheet)  
        - `Terms#` → `Terms` (sheet)  
        - `Invoice#` → `Invoice` (sheet)  
        - `Description#` → `Description` (sheet)  
        - `Amount#` → `Amount` (sheet)  
      - Document URL: dynamically from merged data (`$('Merge').item.json.id`).  
    - *Expressions:* Uses Google Sheets data referenced via node expressions for each placeholder replacement.  
    - *Input/Output:* Input from `Insert Content into Doc`; output is the final updated document.  
    - *Credentials:* Google Docs OAuth2 with write access.  
    - *Edge Cases:*  
      - Placeholders missing or misspelled in template will not be replaced.  
      - Replacement text missing or blank could result in incomplete invoices.  
      - Document access/permission issues.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                          | Input Node(s)                          | Output Node(s)              | Sticky Note                                                                                              |
|-------------------------------|---------------------|----------------------------------------|--------------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Starts workflow manually                | None                                 | Get Invoice Template, Google Sheets | Step 1: Manual trigger for testing or on-demand execution.                                             |
| Google Sheets                 | Google Sheets       | Loads invoice data from sheet           | When clicking ‘Execute workflow’     | Create New Doc               | Step 2: Loads invoice data from specified Google Sheet. Credentials require Google Sheets OAuth2 setup.|
| Get Invoice Template          | Google Docs         | Loads invoice template document          | When clicking ‘Execute workflow’     | Merge                       | Step 3: Loads static Google Docs invoice template. Requires Google Docs OAuth2.                        |
| Create New Doc                | Google Docs         | Creates new Google Doc per invoice       | Google Sheets                        | Merge                       | Step 4: Creates new invoice document in Drive folder with dynamic title.                              |
| Merge                        | Merge               | Combines template data and new doc data | Get Invoice Template, Create New Doc | Insert Content into Doc      | Step 5: Merges template and invoice data for downstream processing.                                   |
| Insert Content into Doc       | Google Docs         | Inserts initial content into new doc    | Merge                               | Input Invoice Details        |                                                                                                        |
| Input Invoice Details         | Google Docs         | Replaces placeholders with actual data  | Insert Content into Doc               | None                        | Step 7: Replaces placeholders in Google Doc with invoice values from sheet.                           |
| Sticky Note                  | Sticky Note         | Documentation and instructions          | None                                 | None                        | Step 1 and Step 2 detailed instructions and credential setup guidance.                                |
| Sticky Note1                 | Sticky Note         | Documentation and instructions          | None                                 | None                        | Steps 3, 4, and 5 detailed instructions and credential notes.                                        |
| Sticky Note2                 | Sticky Note         | Documentation and instructions          | None                                 | None                        | Step 7 detailed instructions and final output explanation.                                           |
| Sticky Note3                 | Sticky Note         | Contact help information                 | None                                 | None                        | Contact email and LinkedIn link for support.                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - No parameters needed.  
   - This node starts the workflow on demand.

2. **Add Google Sheets Node to Read Invoice Data**  
   - Node Type: Google Sheets  
   - Operation: Read rows from spreadsheet.  
   - Set Document ID: `1MHVZRVo5aPs5VqRXk7lBNPVlZ2gilKqZ8J9yeg4taW4`  
   - Set Sheet Name: `gid=0` (Sheet1)  
   - Credential: Connect Google Sheets OAuth2 API with read permission.  
   - Connect Manual Trigger node output to this node input.

3. **Add Google Docs Node to Get Invoice Template**  
   - Node Type: Google Docs  
   - Operation: Get document metadata.  
   - Document URL: `https://docs.google.com/document/d/18n0HTqabDldi7fVbhbI1aG12qbFWsjyTXdduwDDOUu8`  
   - Credential: Google Docs OAuth2 API with read access.  
   - Connect Manual Trigger output to this node input (parallel to Google Sheets).

4. **Add Google Docs Node to Create New Invoice Doc**  
   - Node Type: Google Docs  
   - Operation: Create new document.  
   - Title: Use expression `Invoice: {{ $json.Invoice }}` to dynamically set title from sheet data.  
   - Folder ID: `1TnDibwPPPUm3VbmETiqWDVhtaUTLJ6mn` (Google Drive folder for invoices).  
   - Credential: Google Docs OAuth2 API with write access.  
   - Connect Google Sheets node output to this node input (one document per invoice row).

5. **Add Merge Node to Combine Template and New Doc Data**  
   - Node Type: Merge  
   - Mode: Combine  
   - Combine By: combineAll  
   - Connect `Get Invoice Template` node as first input.  
   - Connect `Create New Doc` node as second input.

6. **Add Google Docs Node to Insert Initial Content**  
   - Node Type: Google Docs  
   - Operation: Update document.  
   - Actions: Insert text action with content set by expression `={{ $json.content }}` (can be empty or template content if defined).  
   - Document URL: Use expression `={{ $json.id }}` to refer to the newly created doc ID from merged data.  
   - Credential: Google Docs OAuth2 API with write access.  
   - Connect Merge node output to this node input.

7. **Add Google Docs Node to Replace Placeholders with Invoice Data**  
   - Node Type: Google Docs  
   - Operation: Update document.  
   - Actions: Multiple `replaceAll` actions configured as:  
     - Replace `FromCompany#` with expression `={{ $('Google Sheets').item.json['Company From'] }}`  
     - Replace `ToCompany#` with `={{ $('Google Sheets').item.json['Company To'] }}`  
     - Replace `Terms#` with `={{ $('Google Sheets').item.json.Terms }}`  
     - Replace `Invoice#` with `={{ $('Google Sheets').item.json.Invoice }}`  
     - Replace `Description#` with `={{ $('Google Sheets').item.json.Description }}`  
     - Replace `Amount#` with `={{ $('Google Sheets').item.json.Amount }}` (append newline if desired)  
   - Document URL: Use expression `={{ $('Merge').item.json.id }}` (new document ID from merged data).  
   - Credential: Google Docs OAuth2 API with write access.  
   - Connect `Insert Content into Doc` node output to this node input.

8. **Save and Test**  
   - Ensure all OAuth2 credentials (Google Sheets and Google Docs) are correctly set up with required API access and permissions.  
   - Test by manually triggering the workflow and verifying new invoices appear in the specified Google Drive folder with placeholders replaced.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Google Sheets OAuth2 credential requires enabling Google Sheets API and setting redirect URI to:                         | https://console.cloud.google.com/ (Google Cloud Console)                                            |
| https://api.n8n.cloud/oauth2-credential/callback                                                                         |                                                                                                     |
| The invoice template must contain placeholders exactly as: `FromCompany#`, `ToCompany#`, `Terms#`, `Invoice#`, `Description#`, `Amount#` | Template URL: https://docs.google.com/document/d/18n0HTqabDldi7fVbhbI1aG12qbFWsjyTXdduwDDOUu8/edit    |
| Google Docs OAuth2 credentials must have write access to the target Google Drive folder for creating and updating docs. | Folder ID used: `1TnDibwPPPUm3VbmETiqWDVhtaUTLJ6mn`                                                |
| Contact for help: Robert Breen, email: rbreen@ynteractive.com LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ |                                                                                                     |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, adhering strictly to valid content policies. It contains no illegal, offensive, or protected elements. All processed data are lawful and public.