Clean Company Names for Cold Emails with GPT-4 Mini and Google Sheets

https://n8nworkflows.xyz/workflows/clean-company-names-for-cold-emails-with-gpt-4-mini-and-google-sheets-5647


# Clean Company Names for Cold Emails with GPT-4 Mini and Google Sheets

### 1. Workflow Overview

This workflow automates the process of cleaning and standardizing company names from a lead list for cold email campaigns using GPT-4 Mini and Google Sheets. The main goal is to transform raw, inconsistent company names into polished, professional versions suitable for personalized outreach, improving engagement and response rates.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Accepts a Google Sheets URL via form submission and initiates processing.
- **1.2 Sheet Preparation and Data Retrieval:** Creates a destination sheet for cleaned data and fetches the original lead data.
- **1.3 AI-Powered Company Name Cleaning:** Uses OpenAI GPT-4 Mini to clean and standardize company names.
- **1.4 Data Merging and Output Saving:** Combines original lead data with cleaned company names and saves the results back to the newly created Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow when a user submits a form containing a Google Sheets URL with lead data. It serves as the entry point for the workflow.

**Nodes Involved:**  
- On Form Submit

**Node Details:**  

- **On Form Submit**  
  - *Type:* Form Trigger  
  - *Role:* Receives user input via a form submission.  
  - *Configuration:*  
    - Webhook ID assigned for external form submission.  
    - Form titled "Clean Up Company Names" with a required field "Lead Spreadsheet Url" for the Google Sheet URL.  
  - *Inputs:* External HTTP POST with form data.  
  - *Outputs:* JSON containing the submitted Google Sheets URL.  
  - *Edge Cases:*  
    - Missing or invalid URL submission.  
    - Unauthorized or malformed webhook calls.  
  - *Notes:* This node is critical as all subsequent processing depends on the input sheet URL.

---

#### 1.2 Sheet Preparation and Data Retrieval

**Overview:**  
Prepares the environment by creating a new worksheet in the provided Google Sheets document to store cleaned data and retrieves the original leads from the first sheet.

**Nodes Involved:**  
- Create Destination Sheet  
- Get Leads

**Node Details:**  

- **Create Destination Sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Creates a new worksheet titled "Cleaned Companies" with a green tab color in the submitted spreadsheet.  
  - *Configuration:*  
    - Operation: Create worksheet.  
    - Document ID: Extracted dynamically from the form submission URL.  
    - Tab color set to #0aa55c (green).  
  - *Inputs:* Output from "On Form Submit" node (spreadsheet URL).  
  - *Outputs:* JSON with the new sheet ID and spreadsheet ID for referencing.  
  - *Edge Cases:*  
    - Insufficient permissions on the spreadsheet.  
    - API rate limits or network issues.  
  - *Credential:* Google Sheets OAuth2 with edit access required.

- **Get Leads**  
  - *Type:* Google Sheets node  
  - *Role:* Fetches all lead data from the first sheet (sheet id "0") in the submitted Google Sheets document.  
  - *Configuration:*  
    - Operation: Read sheet data.  
    - Document ID: Dynamically set from the form submission URL.  
    - Sheet Name: First sheet (ID "0").  
  - *Inputs:* Output from "Create Destination Sheet".  
  - *Outputs:* JSON array of lead data.  
  - *Edge Cases:*  
    - Empty or malformed sheets.  
    - Unauthorized access or missing permissions.  
  - *Credential:* Same Google Sheets OAuth2 credential as above.  

---

#### 1.3 AI-Powered Company Name Cleaning

**Overview:**  
This block processes the raw company names through a GPT-4 Mini model, applying a detailed prompt to clean and standardize company names for professional cold emailing. The AI output is parsed into structured JSON.

**Nodes Involved:**  
- Clean Company Name  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**  

- **Clean Company Name**  
  - *Type:* Langchain LLM Chain node  
  - *Role:* Sends batches of company names to an AI chat model with a custom system prompt instructing the cleaning and formatting rules.  
  - *Configuration:*  
    - Batch size set to 5 to optimize throughput.  
    - Input message includes a detailed prompt explaining:  
      - Removal of suffixes (LLC, Inc., etc.)  
      - Removal of business-related words (Systems, Solutions, etc.)  
      - Removal of taglines and slogans after punctuation or key phrases  
      - Proper title casing with acronym preservation  
      - Examples provided for clarity  
    - Output is expected to be parsable JSON with "cleanCompanyName" field.  
  - *Inputs:* Raw lead data from "Get Leads", specifically company name field.  
  - *Outputs:* AI-generated cleaned company names, still as text.  
  - *Connections:* Uses OpenAI Chat Model for language processing and Structured Output Parser for JSON extraction.  
  - *Edge Cases:*  
    - Ambiguous company names causing poor AI output.  
    - API timeouts or quota limits on OpenAI.  
    - Parsing errors if AI output deviates from JSON schema.  
  - *Credential:* OpenAI API with GPT-4 Mini model access.

- **OpenAI Chat Model**  
  - *Type:* Langchain LLM node  
  - *Role:* Executes the GPT-4 Mini model inference for the cleaning prompt.  
  - *Configuration:* Model selected as "gpt-4.1-mini" for a balance between cost and performance.  
  - *Inputs:* Receives prompt text from "Clean Company Name" node.  
  - *Outputs:* Raw AI chat completion text.  
  - *Edge Cases:* API authentication failure, model unavailability, rate limiting.

- **Structured Output Parser**  
  - *Type:* Langchain output parser  
  - *Role:* Converts AI chat text output into structured JSON based on a defined schema.  
  - *Configuration:* Schema example expects a JSON object with a single field: "cleanCompanyName".  
  - *Inputs:* AI output text from "OpenAI Chat Model".  
  - *Outputs:* Parsed JSON objects with cleaned company names.  
  - *Edge Cases:* Parsing failure if AI output is malformed or not consistent with schema.

---

#### 1.4 Data Merging and Output Saving

**Overview:**  
Merges the cleaned company names back with the original lead data and appends the enriched dataset into the newly created Google Sheet.

**Nodes Involved:**  
- Merge  
- Edit Fields  
- Save Output

**Node Details:**  

- **Merge**  
  - *Type:* Merge node  
  - *Role:* Combines original lead data with the cleaned company names by position, ensuring each cleaned name aligns with the correct lead row.  
  - *Configuration:* Mode set to "combine" by position (index-based merge).  
  - *Inputs:*  
    - Original lead data from "Get Leads".  
    - Cleaned company names from "Clean Company Name".  
  - *Outputs:* Combined JSON with all original fields plus cleaned company name.

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Adjusts output data to include a new field "Clean Company Name" populated from the AI output.  
  - *Configuration:*  
    - Removes temporary fields "output" and "row_number".  
    - Adds "Clean Company Name" field with value from `$json.output.cleanCompanyName`.  
    - Keeps all other original fields.  
  - *Inputs:* Output from "Merge".  
  - *Outputs:* Prepared JSON for saving.

- **Save Output**  
  - *Type:* Google Sheets node  
  - *Role:* Appends the combined and cleaned lead data to the newly created "Cleaned Companies" sheet.  
  - *Configuration:*  
    - Operation: Append data.  
    - Document ID and Sheet ID dynamically taken from "Create Destination Sheet" output.  
    - Auto-mapping input fields to sheet columns, including "Clean Company Name".  
  - *Inputs:* Data from "Edit Fields".  
  - *Outputs:* Confirmation of data appended.  
  - *Credentials:* Same Google Sheets OAuth2 credential as above.  
  - *Edge Cases:*  
    - Write permission errors.  
    - Data schema mismatches causing append failures.

---

### 3. Summary Table

| Node Name              | Node Type                                  | Functional Role                               | Input Node(s)             | Output Node(s)        | Sticky Note                                                                                                            |
|------------------------|--------------------------------------------|-----------------------------------------------|---------------------------|-----------------------|------------------------------------------------------------------------------------------------------------------------|
| On Form Submit         | Form Trigger                               | Entry point; receives Google Sheets URL input | -                         | Create Destination Sheet | ## Fetch the lead data When a form is submitted with the url to a Google Sheet with the lead data, we create a new sheet in the document that we will use to store the results and fetch all the lead data from the first sheet. Note that the account associated with the Google Sheet credentials you use must have edit access to the sheet submitted in the form. |
| Create Destination Sheet | Google Sheets                             | Creates new sheet for storing cleaned data    | On Form Submit            | Get Leads             |                                                                                                                        |
| Get Leads              | Google Sheets                              | Reads original lead data from first sheet     | Create Destination Sheet  | Clean Company Name, Merge|                                                                                                                        |
| Clean Company Name     | Langchain LLM Chain                        | Prepares AI prompt and processes company names | Get Leads                 | Merge                 | ## Clean up the company name with AI **Important**: For better results, update the system prompt with more specific examples based on the company industry / patterns in your lead data. |
| OpenAI Chat Model      | Langchain LLM Model                        | Runs GPT-4 Mini to generate cleaned names     | Clean Company Name (prompt) | Clean Company Name (response) |                                                                                                                        |
| Structured Output Parser | Langchain Output Parser                   | Parses AI output into structured JSON         | OpenAI Chat Model         | Clean Company Name     |                                                                                                                        |
| Merge                  | Merge                                     | Combines original and cleaned data by position| Get Leads, Clean Company Name | Edit Fields           | ## Prep and save output Here we merge in the LLM's output with the original lead data and save everything back to the new sheet we created earlier in the workflow. |
| Edit Fields            | Set                                       | Adds cleaned company name field to data       | Merge                     | Save Output            |                                                                                                                        |
| Save Output            | Google Sheets                             | Appends cleaned and merged data to new sheet  | Edit Fields                | -                      |                                                                                                                        |
| Sticky Note            | Sticky Note                               | Provides explanation for the last block        | -                         | -                      | ## Prep and save output Here we merge in the LLM's output with the original lead data and save everything back to the new sheet we created earlier in the workflow. |
| Sticky Note1           | Sticky Note                               | Provides explanation for the first block       | -                         | -                      | ## Fetch the lead data When a form is submitted with the url to a Google Sheet with the lead data, we create a new sheet in the document that we will use to store the results and fetch all the lead data from the first sheet. Note that the account associated with the Google Sheet credentials you use must have edit access to the sheet submitted in the form. |
| Sticky Note2           | Sticky Note                               | Provides explanation for AI cleaning block     | -                         | -                      | ## Clean up the company name with AI **Important**: For better results, update the system prompt with more specific examples based on the company industry / patterns in your lead data. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node ("On Form Submit")**  
   - Type: Form Trigger  
   - Configure a webhook with a unique ID.  
   - Set form title: "Clean Up Company Names".  
   - Add a required field named "Lead Spreadsheet Url" for the Google Sheets URL input.  

2. **Add Google Sheets Node ("Create Destination Sheet")**  
   - Connect from "On Form Submit".  
   - Operation: "Create" worksheet.  
   - Title: "Cleaned Companies".  
   - Tab Color: Set to green (#0aa55c).  
   - Document ID: Use an expression to extract from the form field `={{ $('On Form Submit').item.json['Lead Spreadsheet Url'] }}`.  
   - Use Google Sheets OAuth2 credential with edit rights.

3. **Add Google Sheets Node ("Get Leads")**  
   - Connect from "Create Destination Sheet".  
   - Operation: Read data from sheet.  
   - Sheet Name: Set to first sheet by ID "0".  
   - Document ID: Same expression as above.  
   - Credential: Same Google Sheets OAuth2 credential.

4. **Add Langchain LLM Chain Node ("Clean Company Name")**  
   - Connect from "Get Leads".  
   - Input: Company name field from lead data. Use expression `=Company Name:  {{ $json.Company }}` for each item.  
   - Batch size: 5 for efficient processing.  
   - Messages:  
     - System prompt with detailed instructions for cleaning company names, including removal of suffixes, business descriptors, taglines, and proper title casing.  
     - Include comprehensive examples as provided.  
   - Enable output parser.  

5. **Add Langchain LLM Model Node ("OpenAI Chat Model")**  
   - Connect as the language model for "Clean Company Name".  
   - Model: Select "gpt-4.1-mini".  
   - Credential: OpenAI API key with access to GPT-4 Mini.

6. **Add Langchain Output Parser Node ("Structured Output Parser")**  
   - Connect from "OpenAI Chat Model".  
   - JSON schema example: `{"cleanCompanyName": "Techwave"}`.  
   - This ensures AI output is parsed into structured JSON.

7. **Add Merge Node ("Merge")**  
   - Connect two inputs:  
     - Original lead data from "Get Leads" (main).  
     - Cleaned company name output from "Clean Company Name" (main).  
   - Mode: Combine by position (index-based alignment).

8. **Add Set Node ("Edit Fields")**  
   - Connect from "Merge".  
   - Remove temporary fields `output` and `row_number`.  
   - Add new field "Clean Company Name" with value from the AI output: `={{ $json.output.cleanCompanyName }}`.  
   - Include all other original fields.

9. **Add Google Sheets Node ("Save Output")**  
   - Connect from "Edit Fields".  
   - Operation: Append rows.  
   - Document ID and Sheet Name: Dynamic values from "Create Destination Sheet" output using expressions:  
     - Document ID: `={{ $('Create Destination Sheet').item.json.spreadsheetId }}`  
     - Sheet Name: `={{ $('Create Destination Sheet').item.json.sheetId }}`  
   - Use auto mapping to match fields, including "Clean Company Name".  
   - Credential: Same Google Sheets OAuth2 credential.

10. **Add Sticky Notes (Optional for Documentation)**  
    - Add notes near blocks to explain process steps and best practices.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                           |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| The Google Sheets OAuth2 credential used must have edit access to all sheets involved in the workflow. | Google Sheets API Permissions                                                                                              |
| For better AI cleaning results, customize the system prompt with company-industry specific examples. | Enhances prompt relevance and accuracy with domain-specific data.                                                         |
| This workflow uses GPT-4 Mini to balance cost and performance; consider model upgrades for higher accuracy. | OpenAI GPT Models: https://platform.openai.com/docs/models                                                                 |
| The workflow appends data to a new sheet named "Cleaned Companies" to preserve original lead data.  | Best practice for data integrity and auditability.                                                                        |
| The batch size in the LLM chain is set to 5 for efficient API usage without hitting rate limits.     | Adjust batch size based on input volume and API quotas.                                                                    |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.