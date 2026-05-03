Sync OVH Invoices to Google Sheets and Save PDFs to Google Drive

https://n8nworkflows.xyz/workflows/sync-ovh-invoices-to-google-sheets-and-save-pdfs-to-google-drive-4224


# Sync OVH Invoices to Google Sheets and Save PDFs to Google Drive

### 1. Workflow Overview

This workflow automates the synchronization of OVH invoices with a Google Sheets document and organizes their corresponding PDF files in Google Drive. It is designed primarily for users who want to track their OVH billing data systematically while storing invoice PDFs in structured folders by fiscal year. The workflow includes logical blocks for querying invoices from the OVH API, processing invoice data, updating Google Sheets, checking and creating Google Drive folders, downloading PDFs, and finally saving and linking these files in the spreadsheet.

Logical blocks:

- **1.1 Input Trigger**: Manual initiation of the workflow.
- **1.2 Query OVH Invoices**: Fetch a list of recent OVH invoices within a defined timeframe.
- **1.3 Data Transformation**: Convert the invoice list into an iterable array.
- **1.4 Invoice Details Processing Loop**: Iterate over each invoice ID to fetch detailed data.
- **1.5 Google Sheets Update**: Append or update invoice data into a Google Sheets document.
- **1.6 Google Drive Folder Verification & Creation**: Check for a fiscal year folder in Google Drive; create it if absent.
- **1.7 Invoice PDF Download and Storage**: Download the invoice PDF and save it to the appropriate Google Drive folder.
- **1.8 Update Sheet with PDF File Information**: Record the Google Drive file ID and filename back into the Google Sheets record.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:** Initiates the workflow manually.
- **Nodes Involved:** `When clicking ‘Test workflow’`
- **Node Details:**
  - Type: `Manual Trigger`
  - Role: Starts the workflow execution on demand.
  - Configuration: No parameters.
  - Inputs: None.
  - Outputs: Connects to `Query Latest OVH Invoices`.
  - Edge Cases: None; manual trigger may be forgotten or triggered unintentionally.

#### 1.2 Query OVH Invoices

- **Overview:** Fetches a list of invoices from OVH API within a set timeframe.
- **Nodes Involved:** `Query Latest OVH Invoices`
- **Node Details:**
  - Type: `HTTP Request`
  - Role: Calls OVH API endpoint `/me/bill` to retrieve invoices.
  - Configuration:
    - URL: `https://eu.api.ovh.com/v1/me/bill`
    - Query Parameters: `date.from` (3 months before current date), `date.to` (current date)
    - Headers: `accept: application/json`
    - Authentication: Generic HTTP header auth using OVH API bearer token.
  - Key Expressions: Uses `$now` and `$now.minus(3, 'months')` to define date range.
  - Inputs: From manual trigger.
  - Outputs: JSON list of invoice IDs.
  - Edge Cases:
    - Authentication failure (invalid or expired token).
    - Network timeouts or API rate limiting.
    - Empty response if no invoices in timeframe.
  - Sticky Note: Link to OVH API Console and advice on adjusting timeframe.

#### 1.3 Data Transformation

- **Overview:** Converts the array of invoice IDs from raw API response into a single-item JSON object containing all invoices for batch processing.
- **Nodes Involved:** `Convert to Array`
- **Node Details:**
  - Type: `Code` (JavaScript)
  - Role: Wraps all invoice IDs into one JSON property `invoices`.
  - Configuration: Returns `[{ json: { invoices: items.map(i => i.json) } }]`.
  - Inputs: List of invoice IDs.
  - Outputs: Single JSON object with invoices array.
  - Edge Cases: Empty input array results in empty invoices property.
  
#### 1.4 Invoice Details Processing Loop

- **Overview:** Iterates over each invoice ID to retrieve detailed invoice data from OVH API.
- **Nodes Involved:** `Split Out`, `Loop Over Items`, `Get Invoice Data`
- **Node Details:**
  - `Split Out`:
    - Type: `Split Out`
    - Role: Extracts the `invoices` array from the transformed object for iteration.
    - Configuration: Field to split out: `invoices`.
    - Inputs: From `Convert to Array`.
    - Outputs: Each invoice ID as separate item.
  - `Loop Over Items`:
    - Type: `Split In Batches`
    - Role: Processes invoices one by one (batch size default).
    - Inputs: From `Split Out`.
    - Outputs: Two outputs:
      - Output 1: Continues looping.
      - Output 2: Sends current batch invoice ID to `Get Invoice Data`.
  - `Get Invoice Data`:
    - Type: `HTTP Request`
    - Role: Queries detailed invoice data for each invoice ID from endpoint `/me/bill/{billId}`.
    - Configuration:
      - URL constructed dynamically using invoice ID from `$json.invoices`.
      - Authentication: Same header auth as before.
      - Headers: `accept: application/json`.
    - Inputs: From `Loop Over Items`.
    - Outputs: JSON with detailed invoice data.
    - Edge Cases:
      - Invalid invoice ID causing 404.
      - API rate limits.
      - Authentication issues.

#### 1.5 Google Sheets Update

- **Overview:** Appends or updates invoice details into a Google Sheets spreadsheet.
- **Nodes Involved:** `Record in OVH Sheet`
- **Node Details:**
  - Type: `Google Sheets`
  - Role: Write invoice data into a specific sheet and update if invoice already exists.
  - Configuration:
    - Document ID: Linked to a specific Google Sheets file (template provided).
    - Sheet Name: Specific sheet by ID.
    - Operation: Append or update based on the "Invoice" column.
    - Columns mapped include fiscal year (FY), date (localized format), tax info, URLs, and pricing details.
    - Date formatting uses expression with `DateTime.fromISO` localized to German (`de`) locale.
  - Inputs: From `Get Invoice Data`.
  - Outputs: Sends updated invoice data to check folder existence.
  - Edge Cases:
    - Google Sheets API quota limits.
    - Incorrect document or sheet ID.
    - Data type conversion errors.
  - Sticky Note: Instruction on date formatting and link to API curl examples.

#### 1.6 Google Drive Folder Verification & Creation

- **Overview:** Ensures a folder exists in Google Drive for the invoice's fiscal year; creates it if missing.
- **Nodes Involved:** `Check for FY Folder ID`, `Does Folder exist`, `Create Folder`
- **Node Details:**
  - `Check for FY Folder ID`:
    - Type: `Google Drive`
    - Role: Search for a folder named by the invoice fiscal year inside the main invoice folder.
    - Configuration:
      - Search folder ID: User must replace placeholder with main invoice folder ID.
      - Query string: Fiscal year derived from invoice date formatted as year (`y`).
      - Resource: `fileFolder`, filtered to folders.
    - Inputs: From `Record in OVH Sheet`.
    - Outputs: Folder search results.
  - `Does Folder exist`:
    - Type: `If`
    - Role: Checks if search result is empty (folder missing) or exists.
    - Condition: True branch if folder does NOT exist.
    - Inputs: From `Check for FY Folder ID`.
    - Outputs: 
      - True: Create folder.
      - False: Proceed to PDF download.
  - `Create Folder`:
    - Type: `Google Drive`
    - Role: Creates new folder named as fiscal year inside main invoices folder.
    - Configuration:
      - Folder name from invoice FY.
      - Parent folder ID from user input placeholder.
      - Folder color set to blue (#0B54E4).
    - Inputs: True branch of `Does Folder exist`.
    - Outputs: Folder creation success.
  - Edge Cases:
    - Missing or incorrect main folder ID.
    - Permission errors on Google Drive.
    - Duplicate folder names.
  - Sticky Note: Highlights the block purpose of saving PDFs into yearly folders.

#### 1.7 Invoice PDF Download and Storage

- **Overview:** Downloads the invoice PDF from the URL and saves it into the respective fiscal year folder in Google Drive.
- **Nodes Involved:** `Download PDF`, `Safe File`
- **Node Details:**
  - `Download PDF`:
    - Type: `HTTP Request`
    - Role: Downloads the invoice PDF binary file from the `pdfUrl` property.
    - Configuration:
      - URL dynamically taken from the invoice data JSON.
      - No authentication required (public PDF link).
    - Inputs: From `Create Folder` success or false branch of `Does Folder exist`.
    - Outputs: Binary PDF data with filename.
    - Edge Cases:
      - Broken or expired PDF URL.
      - Network errors.
  - `Safe File`:
    - Type: `Google Drive`
    - Role: Uploads the downloaded PDF into the folder with the fiscal year ID.
    - Configuration:
      - File name derived from `Download PDF` node’s binary data filename.
      - Folder ID dynamically set to fiscal year folder ID.
      - Drive: Default "My Drive".
    - Inputs: From `Download PDF`.
    - Outputs: Google Drive file metadata including file ID.
    - Edge Cases:
      - Insufficient permissions or quota on Google Drive.
      - Invalid folder ID.
      
#### 1.8 Update Sheet with PDF File Information

- **Overview:** Updates the Google Sheet with the Google Drive file ID and PDF filename for each invoice record.
- **Nodes Involved:** `Update Record Data with FileID`
- **Node Details:**
  - Type: `Google Sheets`
  - Role: Append or update the sheet row corresponding to the invoice with the PDF file name and Google Drive file ID.
  - Configuration:
    - Uses matching column "Invoice" to locate the record.
    - Writes `Filename` and `GDrive_ID`.
    - Same document and sheet as earlier.
  - Inputs: From `Safe File`.
  - Outputs: Loops back to `Loop Over Items` to continue processing next invoice.
  - Edge Cases:
    - Google Sheets API limits.
    - Mismatched or missing invoice IDs preventing updates.

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role                                  | Input Node(s)                        | Output Node(s)                   | Sticky Note                                                                                                     |
|------------------------------|--------------------|-------------------------------------------------|------------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Start workflow manually                         | -                                  | Query Latest OVH Invoices        |                                                                                                                 |
| Query Latest OVH Invoices     | HTTP Request       | Fetch list of invoices from OVH API             | When clicking ‘Test workflow’       | Convert to Array                 | Set the timeframe for the query; link to API console [here](https://eu.api.ovh.com/console/?section=%2Fme&branch=v1#get-/me/bill) |
| Convert to Array              | Code               | Convert invoice array for iteration             | Query Latest OVH Invoices           | Split Out                       |                                                                                                                 |
| Split Out                    | Split Out           | Extract invoices array for looping               | Convert to Array                   | Loop Over Items                 |                                                                                                                 |
| Loop Over Items              | Split In Batches    | Iterate over invoice IDs                         | Split Out                         | Get Invoice Data (batch output 2), Continue batch (output 1) |                                                                                                                 |
| Get Invoice Data             | HTTP Request       | Retrieve detailed invoice data for each invoice | Loop Over Items (batch output 2)    | Record in OVH Sheet             | Query invoice details; date formatting note included                                                            |
| Record in OVH Sheet           | Google Sheets      | Append/update invoice data in sheet              | Get Invoice Data                  | Check for FY Folder ID           | Amend date expression for locale; link to API curl examples                                                    |
| Check for FY Folder ID        | Google Drive       | Search for fiscal year folder in Google Drive    | Record in OVH Sheet               | Does Folder exist               |                                                                                                                 |
| Does Folder exist             | If                 | Check if fiscal year folder exists                | Check for FY Folder ID             | Create Folder (true branch), Download PDF (false branch) |                                                                                                                 |
| Create Folder                | Google Drive       | Create fiscal year folder if missing              | Does Folder exist (true branch)    | Download PDF                   |                                                                                                                 |
| Download PDF                 | HTTP Request       | Download invoice PDF from URL                      | Create Folder / Does Folder exist  | Safe File                      |                                                                                                                 |
| Safe File                   | Google Drive       | Upload PDF to fiscal year folder                   | Download PDF                     | Update Record Data with FileID  |                                                                                                                 |
| Update Record Data with FileID | Google Sheets      | Update sheet with PDF filename and file ID        | Safe File                        | Loop Over Items (batch continue) |                                                                                                                 |
| Sticky Note8                 | Sticky Note        | Setup Guide and author credit                      | -                                | -                             | Detailed setup guide with links and OVH API credential instructions                                            |
| Sticky Note                 | Sticky Note        | Query timeframe and API documentation             | -                                | -                             | Notes on query timeframe and API parameters                                                                     |
| Sticky Note1                | Sticky Note        | Invoice detail query and date formatting           | -                                | -                             | Notes on invoice detail API and date formatting                                                                 |
| Sticky Note2                | Sticky Note        | PDF download to Google Drive yearly folders        | -                                | -                             | Block description for PDF download and folder organization                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: `When clicking ‘Test workflow’`
   - Type: `Manual Trigger`
   - No parameters needed.

2. **Create HTTP Request Node for Invoice List**
   - Name: `Query Latest OVH Invoices`
   - Type: `HTTP Request`
   - URL: `https://eu.api.ovh.com/v1/me/bill`
   - Authentication: Generic Credential Type → HTTP Header Auth
   - Header: `accept: application/json`
   - Query Parameters:
     - `date.from` = `={{ $now.minus(3, 'months') }}`
     - `date.to` = `={{ $now }}`
   - Connect from `When clicking ‘Test workflow’` output.

3. **Create Code Node to Convert Array**
   - Name: `Convert to Array`
   - Type: `Code`
   - Code:
     ```javascript
     return [{ json: { invoices: items.map(i => i.json) } }];
     ```
   - Connect from `Query Latest OVH Invoices`.

4. **Create Split Out Node**
   - Name: `Split Out`
   - Type: `Split Out`
   - Field to split out: `invoices`
   - Connect from `Convert to Array`.

5. **Create Split In Batches Node**
   - Name: `Loop Over Items`
   - Type: `Split In Batches`
   - Use default batch size (1).
   - Connect from `Split Out`.

6. **Create HTTP Request Node for Invoice Details**
   - Name: `Get Invoice Data`
   - Type: `HTTP Request`
   - URL: `=https://eu.api.ovh.com/v1/me/bill/{{ $json.invoices }}`
   - Authentication: Same as `Query Latest OVH Invoices`
   - Header: `accept: application/json`
   - Connect from output 2 of `Loop Over Items`.

7. **Create Google Sheets Node to Record Invoice Data**
   - Name: `Record in OVH Sheet`
   - Type: `Google Sheets`
   - Operation: `appendOrUpdate`
   - Document ID: Use your copied Google Sheets template ID.
   - Sheet Name: Use the relevant sheet ID.
   - Matching Columns: `Invoice`
   - Columns mapping: Map all relevant invoice fields, including localized date formatting:
     ```javascript
     FY: ={{ $json.date.toDateTime().format('y') }}
     date: ={{DateTime.fromISO($json.date,  { locale: 'de' }).toFormat('dd.MM.yyyy')}}
     ```
   - Connect from `Get Invoice Data`.

8. **Create Google Drive Search Node**
   - Name: `Check for FY Folder ID`
   - Type: `Google Drive`
   - Resource: `fileFolder`
   - Operation: Search folders inside your main invoice folder.
   - Query String: Fiscal year from invoice date (`={{DateTime.fromISO($('Get Invoice Data').item.json.date,  { locale: 'de' }).toFormat('y')}}`).
   - Connect from `Record in OVH Sheet`.

9. **Create If Node to Check Folder Existence**
   - Name: `Does Folder exist`
   - Type: `If`
   - Condition: Check if output from `Check for FY Folder ID` is empty (folder does not exist).
   - Connect from `Check for FY Folder ID`.

10. **Create Google Drive Node to Create Folder**
    - Name: `Create Folder`
    - Type: `Google Drive`
    - Resource: `folder`
    - Folder Name: Fiscal year from sheet data (`={{ $('Record in OVH Sheet').item.json.FY }}`)
    - Parent Folder ID: Your main invoice folder ID.
    - Connect from true branch of `Does Folder exist`.

11. **Create HTTP Request Node to Download PDF**
    - Name: `Download PDF`
    - Type: `HTTP Request`
    - URL: `={{ $('Get Invoice Data').item.json.pdfUrl }}`
    - Connect from false branch of `Does Folder exist` and from `Create Folder`.

12. **Create Google Drive Node to Save PDF**
    - Name: `Safe File`
    - Type: `Google Drive`
    - File Name: `={{ $('Download PDF').item.binary.data.fileName }}`
    - Folder ID: Fiscal year folder ID from `Check for FY Folder ID` or `Create Folder`.
    - Connect from `Download PDF`.

13. **Create Google Sheets Node to Update with File Info**
    - Name: `Update Record Data with FileID`
    - Type: `Google Sheets`
    - Operation: `appendOrUpdate`
    - Document ID and Sheet Name: Same as previous Google Sheets node.
    - Matching Columns: `Invoice`
    - Columns: `Filename` and `GDrive_ID` from `Safe File`.
    - Connect from `Safe File`.
    - Connect back to batch continuation output of `Loop Over Items` to process next invoice.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| **Setup Guide authored by Oliver Bardenheier** with detailed instructions on OVH API credential creation, Google Sheets template usage, and OAuth2 header authentication for OVH API calls. Includes links for Google Sheets template and OVH API console login.                                                                                                                                                                                                                                               | [Setup Guide Sticky Note](https://docs.google.com/spreadsheets/d/1NDQVUayKCUktwhDfgdyueW15JPeQZl_oExO_4O5GV-I/edit?usp=sharing) and OVH API Console https://eu.api.ovh.com/console |
| Adjust the timeframe of the query in `Query Latest OVH Invoices` node to suit your needs; for example, setting it to 1 day if using an email trigger before execution.                                                                                                                                                                                                                                                                                                                                          | Sticky note near `Query Latest OVH Invoices` node                                                       |
| For locale-specific date formatting in Google Sheets, amend the date expression in the `Record in OVH Sheet` node accordingly; example uses German locale (`de`).                                                                                                                                                                                                                                                                                                                                              | Sticky note near `Record in OVH Sheet` node                                                            |
| The workflow requires user input of the main Google Drive folder ID where yearly invoice folders will be created and PDFs stored. Replace all `[Put here Your main Invoice Folder]` placeholders with your actual Google Drive folder ID to ensure proper operation.                                                                                                                                                                                                                                              | Multiple Google Drive nodes                                                                              |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow crafted in n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.