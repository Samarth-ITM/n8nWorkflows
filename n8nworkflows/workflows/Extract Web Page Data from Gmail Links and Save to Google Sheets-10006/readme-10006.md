Extract Web Page Data from Gmail Links and Save to Google Sheets

https://n8nworkflows.xyz/workflows/extract-web-page-data-from-gmail-links-and-save-to-google-sheets-10006


# Extract Web Page Data from Gmail Links and Save to Google Sheets

---

### 1. Workflow Overview

This workflow automates the extraction of structured data from links contained within Gmail emails and appends that data into a Google Sheets spreadsheet. It is designed for scenarios where emails from a specific sender contain links to web pages with patient or case data that needs to be processed and stored in a tabular format for further analysis or record-keeping.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger and Email Retrieval:** Manual trigger initiates the workflow, which then searches Gmail inbox for emails from a defined sender within a specified date range.
- **1.2 Link Extraction from Email Body:** Extracts specific hyperlink elements from the HTML content of the email body.
- **1.3 HTTP Request to Access Link:** Opens each extracted link to retrieve its HTML content.
- **1.4 Data Capture from Web Page:** Extracts multiple patient and case-related fields using CSS selectors from the accessed web page.
- **1.5 Data Processing and Transformation:** Custom JavaScript code processes raw extracted data, including date parsing, case ID extraction, and age calculation.
- **1.6 Variable Setting for Output:** Assigns processed data fields to named variables for clarity and further use.
- **1.7 Data Storage:** Appends the processed and structured data as rows to a specific sheet in a Google Sheets document.

Each block directly feeds the next, creating a streamlined pipeline from email retrieval to data storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger and Email Retrieval

**Overview:**  
This block initiates the workflow manually and retrieves all emails from a specified Gmail sender within a certain date range.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Search Emails (Gmail)

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to start the workflow on demand.  
  - *Configuration:* No parameters; triggers workflow execution.  
  - *Input/Output:* No inputs; outputs trigger event to "Search Emails".  
  - *Edge Cases:* N/A for manual trigger.  
  - *Version:* 1

- **Search Emails**  
  - *Type:* Gmail node  
  - *Role:* Query Gmail account for emails from a specific sender within a date range.  
  - *Configuration:*  
    - Sender filter set to "exemplo.email@gmail.com".  
    - ReceivedAfter and ReceivedBefore parameters specify date range (dates appear reversed in JSON but should be corrected in practice).  
    - Operation: getAll, Return all matching emails.  
  - *Credentials:* Uses Gmail OAuth2 credentials named "Gmail account 2".  
  - *Inputs:* Trigger node output.  
  - *Outputs:* Email items to "search for an element in the email body".  
  - *Edge Cases:*  
    - Auth failure if credentials expire.  
    - No emails found results in empty downstream processing.  
    - Date filter misconfiguration may yield no results or unexpected emails.  
  - *Version:* 2.1

---

#### 2.2 Link Extraction from Email Body

**Overview:**  
Extracts specific hyperlinks styled with white text from the HTML content of each email.

**Nodes Involved:**  
- search for an element in the email body (HTML Extract)

**Node Details:**  

- **search for an element in the email body**  
  - *Type:* HTML Extract node  
  - *Role:* Parse the email HTML body to extract href attributes of anchor tags with inline style "color: #ffffff".  
  - *Configuration:*  
    - Operation: extractHtmlContent.  
    - Data property: 'html' (assumed from email body content).  
    - CSS Selector: `a[style*="color: #ffffff"]`  
    - Return value: attribute "href".  
  - *Inputs:* Emails from "Search Emails".  
  - *Outputs:* Extracted links to "open the link".  
  - *Edge Cases:*  
    - If no matching link, output may be empty, leading to errors downstream.  
    - Changes in email HTML structure could break extraction.  
  - *Version:* 1.2

---

#### 2.3 HTTP Request to Access Link

**Overview:**  
Accesses each extracted link to retrieve the HTML content of the linked web page for further data extraction.

**Nodes Involved:**  
- open the link (HTTP Request)

**Node Details:**  

- **open the link**  
  - *Type:* HTTP Request  
  - *Role:* Fetch HTML content from each extracted hyperlink.  
  - *Configuration:*  
    - URL set dynamically from extracted link (`={{ $json.dados }}`).  
    - No special headers or authentication configured.  
    - On error: continue execution without failing workflow (continueErrorOutput).  
  - *Inputs:* Links from "search for an element in the email body".  
  - *Outputs:* HTML page content to "capture data".  
  - *Edge Cases:*  
    - HTTP errors (404, 500) handled by continuing, but data extraction may fail.  
    - Broken links or network issues could lead to missing data.  
  - *Version:* 4.2

---

#### 2.4 Data Capture from Web Page

**Overview:**  
Extracts multiple fields of patient and case information from the HTML content of the fetched web page using CSS selectors.

**Nodes Involved:**  
- capture data (HTML Extract)

**Node Details:**  

- **capture data**  
  - *Type:* HTML Extract node  
  - *Role:* Extract specific data points like date of birth, case ID, request date, full address, patient name, complaint, etc., from the web page.  
  - *Configuration:*  
    - Operation: extractHtmlContent.  
    - Extraction values include keys (dtnasc, id, data_solicitacao, Local_pt01, Local_pt02, Nome, sobrenome, Queixa, Link).  
    - CSS Selectors are used to target specific element IDs like `#lblMemberDateOfBirth`, `#txtCase`, `#lblCaseOpenOn`, etc.  
    - The key "Link" extracts a value with an empty CSS selector and "value" return type (likely a placeholder or misconfiguration).  
  - *Inputs:* HTML content from "open the link".  
  - *Outputs:* Extracted raw data to "processes information".  
  - *Edge Cases:*  
    - Missing or changed element IDs could lead to empty or wrong data.  
    - The "Link" extraction with empty selector might fail or always return empty.  
  - *Version:* 1.2

---

#### 2.5 Data Processing and Transformation

**Overview:**  
Processes raw extracted data with custom JavaScript code to parse dates, clean case IDs, calculate patient age, and determine attendance type based on email subject.

**Nodes Involved:**  
- processes information (Code node)

**Node Details:**  

- **processes information**  
  - *Type:* Code (JavaScript) node  
  - *Role:* Transform raw extracted strings into structured, typed data.  
  - *Configuration:*  
    - Parses date strings from "DD/MM/YYYY HH:MM" format to Date objects.  
    - Extracts numeric case ID from strings with prefixes like "Caso ID :".  
    - Calculates age based on birth date and request date.  
    - Reads email subjects to set attendance type ("PRESENCIAL" or "TELEMEDICINA").  
    - Handles missing data gracefully by returning partial info.  
    - Combines data from multiple preceding nodes using the `$` syntax referencing node outputs by name.  
  - *Key expressions:*  
    - Uses jQuery-like selectors `$()` and `$input.all()` to access multiple node data.  
    - Returns array of transformed JSON objects with added fields like age, requestDate, requestTime, caseId, attendanceType, etc.  
  - *Inputs:* Output from "capture data" and indirectly from "Search Emails" and "search for an element in the email body".  
  - *Outputs:* Processed structured data to "set variables".  
  - *Edge Cases:*  
    - Date parsing may fail if input formats differ.  
    - Missing or malformed IDs might lead to null fields.  
    - Email subject absence or unexpected formats could misclassify attendance type.  
  - *Version:* 2

---

#### 2.6 Variable Setting for Output

**Overview:**  
Assigns processed data fields to named variables with clear labels to prepare for spreadsheet insertion.

**Nodes Involved:**  
- set variables (Set node)

**Node Details:**  

- **set variables**  
  - *Type:* Set node  
  - *Role:* Create clean, well-labeled fields for final data output.  
  - *Configuration:*  
    - Maps processed JSON fields to variables such as "id", "Nome_Paciente" (concatenation of first and last names), "Dt_solicitacao", "Queixa", "Idade", "Local" (combined address fields), "horario", "Link", and "Tipo atendimento".  
    - Uses expressions like `={{ $json.caseId }}`, `={{ $json.Nome }} {{ $json.sobrenome }}`.  
  - *Inputs:* Processed data from "processes information".  
  - *Outputs:* Variables to "Save data in spreadsheet".  
  - *Edge Cases:*  
    - Missing fields from previous steps propagate as empty variables.  
  - *Version:* 3.4

---

#### 2.7 Data Storage

**Overview:**  
Appends the structured and labeled data rows into a specified Google Sheets spreadsheet and sheet.

**Nodes Involved:**  
- Save data in spreadsheet (Google Sheets)

**Node Details:**  

- **Save data in spreadsheet**  
  - *Type:* Google Sheets node  
  - *Role:* Append rows to a Google Sheets document with mapped columns.  
  - *Configuration:*  
    - Operation: append rows.  
    - Document ID and Sheet Name specified by spreadsheet ID and sheet number.  
    - Columns mapped using explicit JSON keys to spreadsheet column names, e.g., "DATA" maps to "Dt_solicitacao", "NOME" to "Nome_Paciente", etc.  
    - Mapping mode: defineBelow (explicit column mapping).  
  - *Credentials:* Google Sheets OAuth2 credentials named "Google Sheets account".  
  - *Inputs:* Variables from "set variables".  
  - *Outputs:* None (end node).  
  - *Edge Cases:*  
    - Auth failures or quota limits on Google Sheets API.  
    - Mismatched column names or schema changes in the sheet could cause data not to append correctly.  
  - *Version:* 4.7

---

### 3. Summary Table

| Node Name                         | Node Type          | Functional Role                                   | Input Node(s)                     | Output Node(s)                       | Sticky Note                                                                                 |
|----------------------------------|--------------------|-------------------------------------------------|----------------------------------|------------------------------------|---------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger     | Manual workflow start                            |                                  | Search Emails                      |                                                                                             |
| Search Emails                    | Gmail              | Retrieve emails from sender/date filters         | When clicking ‘Test workflow’     | search for an element in the email body | **earch for emails** received from a sender within a specific time period.                  |
| search for an element in the email body | HTML Extract       | Extract specific link hrefs from email HTML       | Search Emails                    | open the link                      | **Search for a specific css element** in the email code                                    |
| open the link                   | HTTP Request       | Access link URL to fetch HTML page content        | search for an element in the email body | capture data                     | **access the link** found in the HTML body of the email                                    |
| capture data                   | HTML Extract       | Extract patient/case data from web page HTML       | open the link                   | processes information             | **Search for a specific css element** in the email code                                    |
| processes information          | Code               | Process and transform raw extracted data           | capture data                   | set variables                    | js code to process the extracted data, **customize as you wish**                          |
| set variables                 | Set                | Assign processed data to named output variables    | processes information          | Save data in spreadsheet          | set the desired variables to save                                                          |
| Save data in spreadsheet         | Google Sheets      | Append data rows to specified Google Sheet         | set variables                  |                                  | save the information to the desired Google spreadsheet.                                   |
| Sticky Note                     | Sticky Note        | Comment for Search Emails node                      |                                  |                                  | **earch for emails** received from a sender within a specific time period.                  |
| Sticky Note1                    | Sticky Note        | Comment for "search for an element in the email body" |                                  |                                  | **Search for a specific css element** in the email code                                    |
| Sticky Note2                    | Sticky Note        | Comment for "open the link" node                    |                                  |                                  | **access the link** found in the HTML body of the email                                    |
| Sticky Note3                    | Sticky Note        | Comment for "capture data" node                      |                                  |                                  | **Search for a specific css element** in the email code                                    |
| Sticky Note4                    | Sticky Note        | Comment for "processes information" node            |                                  |                                  | js code to process the extracted data, **customize as you wish**                          |
| Sticky Note5                    | Sticky Note        | Comment for "set variables" node                     |                                  |                                  | set the desired variables to save                                                          |
| Sticky Note6                    | Sticky Note        | Comment for "Save data in spreadsheet" node         |                                  |                                  | save the information to the desired Google spreadsheet.                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: Manual start of the workflow.

2. **Add a Gmail node**  
   - Name: "Search Emails"  
   - Credentials: Connect Gmail OAuth2 account.  
   - Operation: getAll emails.  
   - Filters:  
     - Sender: "exemplo.email@gmail.com"  
     - ReceivedAfter: Set start date (e.g., 2025-09-22T00:00:00)  
     - ReceivedBefore: Set end date (e.g., 2025-09-30T00:00:00)  
   - Connect "When clicking ‘Test workflow’" → "Search Emails".

3. **Add an HTML Extract node**  
   - Name: "search for an element in the email body"  
   - Operation: extractHtmlContent  
   - Data Property Name: "html" (use email body content)  
   - Extraction Values:  
     - Key: "dados"  
     - Attribute: "href"  
     - CSS Selector: `a[style*="color: #ffffff"]`  
     - Return Value: attribute  
   - Connect "Search Emails" → "search for an element in the email body".

4. **Add an HTTP Request node**  
   - Name: "open the link"  
   - URL: Expression `={{ $json.dados }}` to use extracted href.  
   - On Error: Continue on error (to avoid workflow failure).  
   - Connect "search for an element in the email body" → "open the link".

5. **Add another HTML Extract node**  
   - Name: "capture data"  
   - Operation: extractHtmlContent  
   - Extraction Values: Add keys and CSS selectors as follows:  
     - dtnasc: `#lblMemberDateOfBirth`  
     - id: `#txtCase`  
     - data_solicitacao: `#lblCaseOpenOn`  
     - Local_pt01: `#lblMemberFullAddress`  
     - Local_pt02: `#lblLocationHotelName`  
     - Nome: `#lblMemberFirstName`  
     - sobrenome: `#lblMemberLastName`  
     - Queixa: `#lblCaseReportedIssue`  
     - Link: leave CSS Selector empty and return value as "value" (verify usage or adjust if needed)  
   - Connect "open the link" → "capture data".

6. **Add a Code node**  
   - Name: "processes information"  
   - Paste the provided JavaScript code that:  
     - Parses dates (birth and request)  
     - Cleans case IDs  
     - Calculates age  
     - Determines attendance type from email subject  
   - Use references to previous node data via `$` syntax as in the original code.  
   - Connect "capture data" → "processes information".

7. **Add a Set node**  
   - Name: "set variables"  
   - Add assignments for:  
     - id = `={{ $json.caseId }}`  
     - Nome_Paciente = `={{ $json.Nome }} {{ $json.sobrenome }}`  
     - Dt_solicitacao = `={{ $json.requestDate }}`  
     - Queixa = `={{ $json.Queixa }}`  
     - Idade = `={{ $json.age }}`  
     - Local = `={{ $json.Local_pt01 }} {{ $json.Local_pt02 }}`  
     - horario = `={{ $json.requestTime }}`  
     - Link = `={{ $json.link }}`  
     - Tipo atendimento = `={{ $json.attendanceType }}`  
   - Connect "processes information" → "set variables".

8. **Add a Google Sheets node**  
   - Name: "Save data in spreadsheet"  
   - Credentials: Connect Google Sheets OAuth2 account.  
   - Operation: Append rows.  
   - Document ID: Set the Google Sheets document ID (e.g., "1bbcFzO77RBr0t8hYaAEuzEsr24MJrFWuIPzoC-XMjx8").  
   - Sheet Name: Set the sheet number or name (e.g., 311390915).  
   - Columns mapping: Map each variable to corresponding sheet column as per the original configuration:  
     - DATA ← Dt_solicitacao  
     - LINK ← Link  
     - NOME ← Nome_Paciente  
     - CASO ← id  
     - IDADE ← Idade  
     - LOCAL ← Local  
     - QUEIXA ← Queixa  
     - HORÁRIO ← horario  
     - TIPO CONSULTA ← Tipo atendimento  
   - Connect "set variables" → "Save data in spreadsheet".

9. **Test the workflow by executing the Manual Trigger node and verify data flows correctly and appends to Google Sheets.**

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                  |
|------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow relies on consistent HTML structures in emails and linked pages.| Changes to email formats or target web pages may require updating CSS selectors or code logic. |
| Date filters in Gmail node appear reversed; ensure correct chronological order.| Setting accurate date ranges is critical for retrieving intended emails.                        |
| The JavaScript code block is customizable for additional data processing needs.| Modify the code to handle new fields or different date formats as required.                    |
| Google Sheets column names are case-sensitive and must match exactly.        | Adjust mapping if spreadsheet columns are renamed or reordered.                                |
| Workflow tested with OAuth2 credentials for Gmail and Google Sheets.         | Ensure credentials have sufficient permission scopes and are refreshed as needed.              |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---