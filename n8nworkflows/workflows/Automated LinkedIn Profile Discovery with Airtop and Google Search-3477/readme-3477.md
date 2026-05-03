Automated LinkedIn Profile Discovery with Airtop and Google Search

https://n8nworkflows.xyz/workflows/automated-linkedin-profile-discovery-with-airtop-and-google-search-3477


# Automated LinkedIn Profile Discovery with Airtop and Google Search

### 1. Workflow Overview

This workflow automates the discovery of LinkedIn profile URLs based on partial contact information using Google search and Airtop's AI-powered extraction capabilities. It is designed to save time and improve accuracy for recruiters, sales teams, or anyone needing to enrich contact data with validated LinkedIn profiles.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggering the workflow manually and reading input data from a Google Sheet.
- **1.2 Profile Search and Extraction:** Using Airtop to perform a Google search and extract the LinkedIn profile URL.
- **1.3 Response Parsing:** Processing the extracted data to isolate the LinkedIn URL.
- **1.4 Output Update:** Writing the validated LinkedIn URL back into the Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually and retrieves the input data (e.g., name, company) from a Google Sheet to be used for LinkedIn profile discovery.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Person info (Google Sheets)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow on user command for testing or manual runs.  
    - *Configuration:* No parameters; simply triggers the workflow.  
    - *Connections:* Outputs to "Person info".  
    - *Edge Cases:* None specific; workflow will not run unless triggered.  

  - **Person info**  
    - *Type:* Google Sheets  
    - *Role:* Reads rows from a specified Google Sheet containing contact information.  
    - *Configuration:*  
      - Document ID linked to a Google Sheet with columns like name, company, domain, etc.  
      - Sheet name set to the first sheet (gid=0).  
      - Uses OAuth2 credentials for Google Drive access.  
    - *Key Expressions:* None explicitly; reads all rows by default.  
    - *Connections:* Outputs to "Search profile".  
    - *Edge Cases:*  
      - Authentication failure if Google credentials expire or are revoked.  
      - Empty or malformed rows may cause downstream issues.  
      - Rate limits or API quota exceeded on Google Sheets API.  

#### 2.2 Profile Search and Extraction

- **Overview:**  
  This block constructs a Google search query from the input data and uses Airtop's AI extraction node to find and validate the LinkedIn profile URL.

- **Nodes Involved:**  
  - Search profile (Airtop)

- **Node Details:**

  - **Search profile**  
    - *Type:* Airtop (AI-powered extraction)  
    - *Role:* Performs a Google search query and extracts the LinkedIn URL from the search results.  
    - *Configuration:*  
      - URL parameter dynamically constructed using the input data field `Person Info` (which is expected to be a concatenated string of relevant contact info).  
      - Prompt instructs Airtop to return only the LinkedIn URL starting with `https://www.linkedin.com/in/` or an empty string if none found.  
      - Session mode set to "new" for each query to avoid session contamination.  
      - Uses Airtop API credentials.  
    - *Key Expressions:*  
      - URL: `https://www.google.com/search?q={{ encodeURI($json['Person Info']) }}`  
      - Prompt: Custom prompt guiding the AI to extract the LinkedIn URL only.  
    - *Connections:* Outputs to "Parse response".  
    - *Edge Cases:*  
      - API rate limits or quota exceeded on Airtop.  
      - No LinkedIn URL found returns empty string, which downstream nodes must handle.  
      - Network timeouts or errors during query.  
      - Incorrect or ambiguous input data leading to irrelevant search results.  

#### 2.3 Response Parsing

- **Overview:**  
  This block processes the raw response from Airtop, extracts the LinkedIn URL, and merges it with the original input data for updating.

- **Nodes Involved:**  
  - Parse response (Code node)

- **Node Details:**

  - **Parse response**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts the LinkedIn URL from Airtop's response and combines it with the original row data.  
    - *Configuration:*  
      - Runs once per item.  
      - JavaScript code accesses `data.modelResponse` from Airtop's output and assigns it to the `LinkedIn URL` field.  
      - Merges this with the original input data from "Person info".  
    - *Key Expressions:*  
      ```js
      const linkedInProfile = $json.data.modelResponse;
      const rowData = $('Person info').item.json;
      return { json: { ...rowData, 'LinkedIn URL': linkedInProfile } };
      ```  
    - *Connections:* Outputs to "Update row".  
    - *Edge Cases:*  
      - If Airtop returns unexpected data structure, code may fail or produce incorrect output.  
      - Empty LinkedIn URL handled as empty string.  
      - Dependency on correct naming of input node ("Person info").  

#### 2.4 Output Update

- **Overview:**  
  This block updates the original Google Sheet row with the discovered LinkedIn URL and optionally a validation status.

- **Nodes Involved:**  
  - Update row (Google Sheets)

- **Node Details:**

  - **Update row**  
    - *Type:* Google Sheets  
    - *Role:* Updates the corresponding row in the Google Sheet with the new LinkedIn URL and validation status.  
    - *Configuration:*  
      - Uses the same Google Sheet document and sheet as "Person info".  
      - Matching rows by `row_number` to ensure correct update.  
      - Auto-maps input data fields to sheet columns including "Person Info", "Linkedin URL", and "Validated".  
      - Uses OAuth2 credentials for Google Drive.  
    - *Key Expressions:* None explicitly; relies on auto-mapping.  
    - *Connections:* Terminal node.  
    - *Edge Cases:*  
      - Row matching failure if `row_number` is missing or incorrect.  
      - Google Sheets API quota or permission issues.  
      - Concurrent updates causing race conditions if multiple runs overlap.  

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                       | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                          |
|-------------------------|-------------------|------------------------------------|-------------------------|------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger    | Starts the workflow manually        | —                       | Person info            |                                                                                                    |
| Person info             | Google Sheets     | Reads input contact data from sheet | When clicking ‘Test workflow’ | Search profile         |                                                                                                    |
| Search profile          | Airtop            | Performs Google search & extracts LinkedIn URL | Person info             | Parse response         | This could take a few minutes depending on the number of rows                                      |
| Parse response          | Code              | Parses Airtop response & merges data | Search profile          | Update row             |                                                                                                    |
| Update row              | Google Sheets     | Updates Google Sheet with LinkedIn URL | Parse response          | —                      |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `When clicking ‘Test workflow’`  
   - No configuration needed. This node will manually start the workflow.

3. **Add a Google Sheets node to read input data:**  
   - Name: `Person info`  
   - Operation: Read rows from a sheet.  
   - Document ID: Use your Google Sheet ID containing contact data (e.g., names, companies).  
   - Sheet Name: Use the first sheet or the one containing your data (e.g., `gid=0`).  
   - Credentials: Set up and select your Google Drive OAuth2 credentials.  
   - Connect output of Manual Trigger to this node.

4. **Add an Airtop node to perform the search and extraction:**  
   - Name: `Search profile`  
   - Resource: `extraction`  
   - Operation: `query`  
   - URL parameter: Set to `https://www.google.com/search?q={{ encodeURI($json['Person Info']) }}` where `Person Info` is the concatenated string of contact info from the Google Sheet.  
   - Prompt parameter:  
     ```
     This is Google Search results. the first results should be the Linkedin Page of {{ $json['Person Info'] }} 
     Return the Linkedin URL and nothing else.
     If you cannot find the Linkedin URL, return an empty string. 
     A valid Linkedin profile URL starts with "https://www.linkedin.com/in/"
     ```  
   - Session Mode: `new`  
   - Credentials: Add and select your Airtop API credentials.  
   - Connect output of `Person info` to this node.

5. **Add a Code node to parse the Airtop response:**  
   - Name: `Parse response`  
   - Mode: Run once per item.  
   - JavaScript code:  
     ```js
     const linkedInProfile = $json.data.modelResponse;
     const rowData = $('Person info').item.json;
     return { json: { ...rowData, 'LinkedIn URL': linkedInProfile } };
     ```  
   - Connect output of `Search profile` to this node.

6. **Add a Google Sheets node to update the original sheet:**  
   - Name: `Update row`  
   - Operation: Update row.  
   - Document ID and Sheet Name: Same as `Person info`.  
   - Mapping Mode: Auto map input data to columns.  
   - Matching Columns: Use `row_number` to identify the row to update.  
   - Columns to update: At minimum, `Linkedin URL` and optionally `Validated`.  
   - Credentials: Use the same Google Drive OAuth2 credentials.  
   - Connect output of `Parse response` to this node.

7. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Airtop API key is required for the extraction node to function.                                                | https://portal.airtop.ai/?utm_campaign=n8n                                                        |
| Google Workspace account (or Gmail) is needed for Google Sheets integration.                                   |                                                                                                    |
| Example Google Sheet template with input and output columns is provided for easy setup.                        | https://docs.google.com/spreadsheets/d/1rjlKzphEbknNh_ToS9pR_dP_Tw93FsxDte5AI4LH5_E/edit?usp=sharing |
| Use full names and company information in input data for best LinkedIn profile discovery accuracy.            |                                                                                                    |
| Consider rate limiting and proxy usage to avoid LinkedIn or Google blocking automated requests.               | https://docs.airtop.ai/guides/how-to/using-a-proxy                                                  |
| Workflow can be adapted to other data sources like Airtable or Notion and integrated with webhooks.           |                                                                                                    |
| Real-world use cases include recruiting firms saving on data enrichment costs and sales teams enriching leads.|                                                                                                    |

---

This documentation provides a detailed, structured understanding of the LinkedIn Profile Discovery workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.