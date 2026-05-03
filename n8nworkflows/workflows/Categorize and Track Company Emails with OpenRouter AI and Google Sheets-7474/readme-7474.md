Categorize and Track Company Emails with OpenRouter AI and Google Sheets

https://n8nworkflows.xyz/workflows/categorize-and-track-company-emails-with-openrouter-ai-and-google-sheets-7474


# Categorize and Track Company Emails with OpenRouter AI and Google Sheets

### 1. Workflow Overview

This workflow is designed to automate the categorization and tracking of company emails using AI-powered analysis and Google Sheets for data storage and updates. It listens for incoming emails via IMAP, retrieves relevant data from Google Sheets, processes the email content with an OpenRouter AI language model agent, determines the category of the email, and updates or appends rows in Google Sheets accordingly.  

The workflow’s logical structure is divided into the following blocks:

- **1.1 Input Reception**: Captures incoming emails from an IMAP email account.
- **1.2 Data Retrieval from Google Sheets**: Fetches existing data rows to aid AI processing.
- **1.3 AI Processing and Categorization**: Uses OpenRouter AI through LangChain nodes to analyze email content and produce structured category and request data.
- **1.4 Category Lookup in Google Sheets**: Searches for the derived category in Google Sheets to determine if it exists.
- **1.5 Conditional Update or Append**: Based on category existence, either updates an existing request count or appends a new row to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow when new emails arrive in an IMAP mailbox.

**Nodes Involved:**  
- Email Trigger (IMAP)

**Node Details:**  

- **Email Trigger (IMAP)**  
  - Type: Email Trigger (IMAP)  
  - Role: Watches an email inbox and triggers the workflow on new emails.  
  - Configuration: Default IMAP settings (credentials required, not shown in JSON).  
  - Inputs: None (trigger node).  
  - Outputs: Passes new email data downstream.  
  - Edge Cases:  
    - Authentication failures (wrong credentials or expired tokens).  
    - Network timeouts or connectivity issues.  
    - Empty inbox or no new emails.  
  - Version: 2.1  

---

#### 1.2 Data Retrieval from Google Sheets

**Overview:**  
This block retrieves existing rows from Google Sheets, likely to provide context or existing categories to the AI agent.

**Nodes Involved:**  
- Get row(s) in sheet

**Node Details:**  

- **Get row(s) in sheet**  
  - Type: Google Sheets - Get Rows  
  - Role: Pulls data from a Google Sheet to provide AI context or to check existing information.  
  - Configuration: Likely configured with spreadsheet ID, sheet name, and optional filters (exact config not visible).  
  - Inputs: Email data from the trigger node.  
  - Outputs: Sends sheet data to the AI Agent node.  
  - Edge Cases:  
    - Authentication errors (invalid or expired OAuth tokens).  
    - Sheet access permissions denied.  
    - Empty or malformed sheet data.  
  - Version: 4.6  
  - Note: `alwaysOutputData` set to true ensures output even if no rows found.  

---

#### 1.3 AI Processing and Categorization

**Overview:**  
This block processes the email content with an AI agent using LangChain nodes, leveraging OpenRouter as the language model, and parses AI output to structured data.

**Nodes Involved:**  
- AI Agent  
- oAI OSS (OpenRouter LM)  
- Category & Request (LangChain Structured Output Parser)

**Node Details:**  

- **oAI OSS**  
  - Type: LangChain LM Chat OpenRouter  
  - Role: Provides AI model interface for natural language processing.  
  - Configuration: Configured to use OpenRouter as the LLM backend. Credentials for OpenRouter must be set in n8n.  
  - Inputs: Message prompt from AI Agent node (reverse flow, attached as languageModel input).  
  - Outputs: AI responses to AI Agent.  
  - Edge Cases:  
    - API key invalid or quota exceeded.  
    - Latency or timeout from OpenRouter API.  
  - Version: 1  

- **Category & Request**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI response into structured JSON for category and request data extraction.  
  - Configuration: Uses a structured output parser, likely with a defined schema to extract fields such as "Category" and "Request."  
  - Inputs: AI output from oAI OSS.  
  - Outputs: Structured data to AI Agent.  
  - Edge Cases:  
    - Parsing errors if AI output does not conform to expected structure.  
  - Version: 1.3  

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates prompt construction, sends to OpenRouter LM, and parses output.  
  - Configuration: Uses sub-nodes for languageModel (oAI OSS) and outputParser (Category & Request). Executes once per email.  
  - Inputs: Email data and Google Sheets rows from "Get row(s) in sheet."  
  - Outputs: Sends categorized data to "Find Category."  
  - Edge Cases:  
    - Execution failures if input data is malformed.  
    - AI model or parser errors.  
  - Version: 2.1  

---

#### 1.4 Category Lookup in Google Sheets

**Overview:**  
This block searches Google Sheets to find if the predicted category already exists.

**Nodes Involved:**  
- Find Category

**Node Details:**  

- **Find Category**  
  - Type: Google Sheets - Get Rows  
  - Role: Looks up a category in Google Sheets to check if it exists.  
  - Configuration: Uses filters or queries to find rows matching the AI-determined category.  
  - Inputs: Categorized data from AI Agent.  
  - Outputs: Determines flow for conditional logic in the next node.  
  - Edge Cases:  
    - Authentication or permission errors.  
    - No matching rows found (handled downstream).  
  - Version: 4.6  
  - `alwaysOutputData` is true to ensure downstream nodes receive output regardless of matches.  

---

#### 1.5 Conditional Update or Append

**Overview:**  
Based on whether the category exists, this block updates the request count of an existing entry or appends a new row to Google Sheets.

**Nodes Involved:**  
- If  
- Edit Fields  
- Update Request Count  
- Append row in sheet

**Node Details:**  

- **If**  
  - Type: Conditional (If)  
  - Role: Checks if category exists (based on output from "Find Category").  
  - Configuration: Likely checks if length of found rows > 0.  
  - Inputs: Output of "Find Category."  
  - Outputs: Two branches:  
    - True: Category exists → Update flow  
    - False: Category not found → Append flow  
  - Edge Cases:  
    - Expression errors if input data does not contain expected fields.  
  - Version: 2.2  

- **Edit Fields**  
  - Type: Set Node  
  - Role: Prepares data fields for updating the request count, possibly increments the count.  
  - Configuration: Sets or modifies specific fields for the Google Sheets update.  
  - Inputs: True branch from If node.  
  - Outputs: Data sent to "Update Request Count."  
  - Edge Cases:  
    - Data type mismatches.  
  - Version: 3.4  

- **Update Request Count**  
  - Type: Google Sheets - Update Row  
  - Role: Updates the existing sheet row with incremented request count or other changes.  
  - Configuration: Uses row ID or unique identifier to update correct row.  
  - Inputs: Data from "Edit Fields."  
  - Outputs: None further downstream (end of update branch).  
  - Edge Cases:  
    - Row not found due to concurrency issues.  
    - Permission errors.  
  - Version: 4.6  

- **Append row in sheet**  
  - Type: Google Sheets - Append Row  
  - Role: Adds a new row for a new category and request.  
  - Configuration: Maps AI output fields to sheet columns for insertion.  
  - Inputs: False branch from If node.  
  - Outputs: None further downstream (end of append branch).  
  - Edge Cases:  
    - Permission denied.  
    - Invalid data formats.  
  - Version: 4.6  

---

### 3. Summary Table

| Node Name           | Node Type                                | Functional Role                         | Input Node(s)        | Output Node(s)           | Sticky Note                                  |
|---------------------|----------------------------------------|---------------------------------------|----------------------|--------------------------|----------------------------------------------|
| Email Trigger (IMAP) | Email Trigger (IMAP)                    | Triggers on new incoming emails       | —                    | Get row(s) in sheet      |                                              |
| Get row(s) in sheet  | Google Sheets (Get Rows)                | Retrieves existing data from sheet    | Email Trigger (IMAP)  | AI Agent                 |                                              |
| AI Agent            | LangChain Agent                        | Processes email with AI and parses output | Get row(s) in sheet, oAI OSS, Category & Request | Find Category            |                                              |
| oAI OSS             | LangChain LM Chat OpenRouter            | Provides AI language model             | AI Agent (languageModel input) | AI Agent (LM output)       |                                              |
| Category & Request   | LangChain Output Parser Structured      | Parses AI response into structured data | oAI OSS              | AI Agent (outputParser)  |                                              |
| Find Category       | Google Sheets (Get Rows)                | Checks if category exists in sheet    | AI Agent              | If                       |                                              |
| If                  | Conditional (If)                        | Branches flow based on category presence | Find Category        | Edit Fields (true), Append row in sheet (false) |                                              |
| Edit Fields         | Set Node                               | Prepares updated data for sheet update | If (true)             | Update Request Count      |                                              |
| Update Request Count | Google Sheets (Update Row)              | Updates existing row with incremented count | Edit Fields          | —                        |                                              |
| Append row in sheet | Google Sheets (Append Row)              | Adds new row for new category         | If (false)             | —                        |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an IMAP Email Trigger node**:  
   - Set node type to "Email Trigger (IMAP)."  
   - Configure IMAP credentials with your mail server details.  
   - Save and connect it as the starting node.  

2. **Add a Google Sheets node to get rows**:  
   - Set node type to "Google Sheets" → Operation: "Get Rows."  
   - Link credentials for Google Sheets OAuth2.  
   - Configure spreadsheet ID and sheet name containing category/request data.  
   - Connect this node to the IMAP trigger node's main output.  
   - Enable "Always Output Data" to ensure downstream receives data even if empty.  

3. **Add LangChain AI Agent node**:  
   - Set node type to "@n8n/n8n-nodes-langchain.agent."  
   - Enable "Execute Once" to process one email at a time.  
   - Connect the "Get row(s) in sheet" node output to the AI Agent main input.  

4. **Configure OpenRouter LM node**:  
   - Add "@n8n/n8n-nodes-langchain.lmChatOpenRouter" node.  
   - Set up OpenRouter API credentials in n8n (API key).  
   - Connect this node to the AI Agent node's `ai_languageModel` input.  

5. **Add LangChain Structured Output Parser node**:  
   - Add "@n8n/n8n-nodes-langchain.outputParserStructured" node.  
   - Configure expected output schema to extract "Category" and "Request" fields.  
   - Connect this node to the AI Agent's `ai_outputParser` input.  

6. **Connect AI Agent main output to Google Sheets node for category lookup**:  
   - Add a new Google Sheets node to "Get Rows" to find if the category exists.  
   - Configure spreadsheet and sheet with categories.  
   - Use a filter (e.g., "Category" column equals AI Agent output category).  
   - Enable "Always Output Data."  
   - Connect AI Agent output to this node.  

7. **Add an If node for conditional branching**:  
   - Add an "If" node.  
   - Configure condition: Check if the length of rows from "Find Category" > 0.  
   - Connect "Find Category" output to this node.  

8. **Add Set node to edit fields for update**:  
   - Add "Set" node.  
   - Configure fields to increment the request count or modify necessary fields.  
   - Connect "If" node's "true" output to this node.  

9. **Add Google Sheets node to update row**:  
   - Add Google Sheets node configured to "Update Row."  
   - Use row ID from "Find Category" or "Set" node data.  
   - Connect "Set" node output to this node.  

10. **Add Google Sheets node to append new row**:  
    - Add another Google Sheets node configured to "Append Row."  
    - Map AI Agent output fields for new category and request.  
    - Connect "If" node's "false" output to this node.  

11. **Finalize and test the workflow**:  
    - Ensure all Google Sheets nodes use proper OAuth2 credentials.  
    - Verify OpenRouter API credentials are valid.  
    - Test with sample emails.  
    - Monitor for errors in node executions and adjust filters or mappings as needed.  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow uses OpenRouter API as an LLM backend requiring valid API credentials and quota.       | OpenRouter official site: https://openrouter.ai        |
| Google Sheets nodes require OAuth2 credentials with edit and read permissions on the target sheets. | Google Sheets API docs: https://developers.google.com/sheets/api |
| LangChain integration in n8n facilitates structured AI output parsing for reliable data extraction. | LangChain docs: https://docs.langchain.com              |
| Be aware of rate limits on both Google Sheets API and OpenRouter API to prevent workflow failures.   | Monitor API dashboard for usage and errors.             |

---

*Disclaimer: The text provided originates exclusively from an n8n automated workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.*