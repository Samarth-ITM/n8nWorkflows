Job Brief Analysis with OpenAI to Find Relevant Templates and Log in Google Sheets

https://n8nworkflows.xyz/workflows/job-brief-analysis-with-openai-to-find-relevant-templates-and-log-in-google-sheets-9019


# Job Brief Analysis with OpenAI to Find Relevant Templates and Log in Google Sheets

### 1. Workflow Overview

This n8n workflow is designed to automate the process of analyzing job briefs (typically from Upwork proposals) using OpenAI, extracting relevant keywords, searching an external template library for matching workflow templates, and logging the results within a Google Sheet. It targets users who want to quickly find n8n workflow templates relevant to job descriptions and maintain an organized log for easy reference.

The workflow’s logic can be divided into these main functional blocks:

- **1.1 Input Reception:** Initiates the workflow when a chat message containing a job brief is received.
- **1.2 AI Keyword Extraction:** Uses an AI agent powered by OpenAI GPT-4 to extract exactly five concise, distinct keywords from the job description.
- **1.3 Keyword Processing:** Parses the AI output, converts it into an array of keywords, and iterates over each keyword.
- **1.4 Template Search:** For each keyword, queries the n8n template library API to find matching workflow templates.
- **1.5 Result Logging:** Processes each found template, appends details to a Google Sheet, builds a public URL for the template, and updates the sheet accordingly.
- **1.6 Configuration & Notes:** Includes setup instructions and general notes to ensure correct operation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block triggers the workflow upon receiving a chat message, which should contain a job brief or description. It serves as the entry point.
- **Nodes Involved:** 
  - When chat message received
- **Node Details:**
  - **When chat message received**
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`
    - Role: Webhook trigger that listens for incoming chat messages.
    - Configuration: Uses a webhook with a unique ID; no special options configured.
    - Input: External chat client message.
    - Output: Passes the chat message to the AI Agent node.
    - Failure modes: Webhook connectivity issues, malformed input.
    - Version: 1.3

#### 2.2 AI Keyword Extraction

- **Overview:** Extracts exactly five distinct, concise keywords from the job description using an OpenAI-powered AI agent.
- **Nodes Involved:**
  - AI Agent
  - OpenAI Chat Model
- **Node Details:**
  - **AI Agent**
    - Type: `@n8n/n8n-nodes-langchain.agent`
    - Role: Orchestrates the AI prompt and handles output parsing.
    - Configuration: Uses system message instructing to extract exactly 5 keywords and return as JSON object with keys "keyword1" to "keyword5".
    - Input: Job brief text from the chat trigger.
    - Output: JSON object with extracted keywords.
    - Failure modes: OpenAI API errors, malformed AI output.
    - Version: 2.2
  - **OpenAI Chat Model**
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
    - Role: The actual OpenAI GPT-4 model call.
    - Configuration: Model set to "gpt-4.1-mini", OpenAI credentials required.
    - Input: Prompt from AI Agent.
    - Output: AI-generated response.
    - Failure modes: API rate limits, auth errors, timeouts.
    - Version: 1.2

#### 2.3 Keyword Processing

- **Overview:** Parses the JSON string from AI output and converts it into an array of keyword objects for iteration.
- **Nodes Involved:**
  - Parse Keywords
  - Converting to Array
  - Loop Over Items
- **Node Details:**
  - **Parse Keywords**
    - Type: Code node
    - Role: Parses the AI response JSON string into an object.
    - Configuration: JavaScript code that parses the `output` field from AI Agent's JSON.
    - Input: AI Agent output.
    - Output: JSON object with keywords.
    - Failure modes: JSON parsing errors if AI response is malformed.
    - Version: 2
  - **Converting to Array**
    - Type: Code node
    - Role: Converts the keyword object into an array of individual keyword objects.
    - Configuration: JavaScript code creates an array of objects with keys: keyword1 to keyword5.
    - Input: Parsed keywords object.
    - Output: Array with 5 items, each containing a single keyword.
    - Failure modes: Missing keys or undefined values from previous node.
    - Version: 2
  - **Loop Over Items**
    - Type: SplitInBatches node
    - Role: Iterates over each keyword item sequentially.
    - Configuration: Default options for batch size (1 by default).
    - Input: Array of keyword objects.
    - Output: Single keyword per iteration.
    - Failure modes: Empty array input.
    - Version: 3

#### 2.4 Template Search

- **Overview:** For each extracted keyword, queries the n8n template library API to search for relevant workflow templates.
- **Nodes Involved:**
  - Search Templates
  - Split Out
- **Node Details:**
  - **Search Templates**
    - Type: HTTP Request node
    - Role: Sends GET requests to `https://api.n8n.io/templates/search` with query parameter `search` set to the current keyword.
    - Configuration: Query parameter dynamically set as `={{ $json.keyword }}`; expects JSON response.
    - Input: Keyword string from Loop Over Items.
    - Output: JSON response containing matching templates.
    - Failure modes: API errors, network issues, invalid response format.
    - Version: 4.2
  - **Split Out**
    - Type: SplitOut node
    - Role: Splits the array of workflows from API response into individual items for further processing.
    - Configuration: Splits on the field `workflows`.
    - Input: Search Templates response.
    - Output: Individual workflow template objects.
    - Failure modes: Missing or empty `workflows` field.
    - Version: 1

#### 2.5 Result Logging

- **Overview:** Appends each found template to a Google Sheet, builds a public URL for the template, retrieves the corresponding row, edits fields with the URL, and updates the row.
- **Nodes Involved:**
  - Append row in sheet2
  - Get row(s) in sheet
  - Edit Fields
  - Update row in sheet
- **Node Details:**
  - **Append row in sheet2**
    - Type: Google Sheets node
    - Role: Appends a new row for each template with columns: id, Name, User, Description.
    - Configuration: DocumentId and SheetName set to the target Google Sheet; columns mapped explicitly.
    - Input: Individual workflow template data.
    - Output: Confirmation with appended row info.
    - Failure modes: Google Sheets API auth errors, rate limits, missing columns.
    - Version: 4.7
  - **Get row(s) in sheet**
    - Type: Google Sheets node
    - Role: Retrieves the newly appended row by matching on `id` and `Name`.
    - Configuration: Filters set to look up by `id` and `Name`.
    - Input: Output from append operation.
    - Output: Row data including row number.
    - Failure modes: Lookup failures if row not found.
    - Version: 4.7
  - **Edit Fields**
    - Type: Set node
    - Role: Builds a public-facing URL for the template based on id and name.
    - Configuration: Creates a URL string using template ID and sanitized template name.
    - Input: Row data from Google Sheets.
    - Output: Modified JSON with `URL` field.
    - Failure modes: String manipulation errors if fields missing.
    - Version: 3.4
  - **Update row in sheet**
    - Type: Google Sheets node
    - Role: Updates the previously appended row with the generated URL.
    - Configuration: Matches on `row_number` to update the correct row.
    - Input: Edited row data with URL.
    - Output: Confirmation of the update.
    - Failure modes: Google Sheets API errors, incorrect row number.
    - Version: 4.7

#### 2.6 Configuration & Notes

- **Overview:** Provides setup instructions and general notes for correct workflow operation.
- **Nodes Involved:**
  - Sticky Note
  - Sticky Note1
- **Node Details:**
  - **Sticky Note**
    - Type: Sticky Note node
    - Role: Setup instructions for credentials and Google Sheet configuration.
    - Content highlights:
      - Add OpenAI credentials to LLM nodes.
      - Add Google Sheets OAuth credentials.
      - Set environment/config variables for Google Sheet Document ID, Sheet Name, API URL.
      - Ensure Google Sheet columns for Template ID, Name, User, Description, URL.
    - No input/output connections.
    - Version: 1
  - **Sticky Note1**
    - Type: Sticky Note node
    - Role: Describes the logical blocks of the workflow and general notes.
    - Content highlights:
      - Explains the purpose of input trigger, AI keyword extraction, template search, and Google Sheets logging.
      - Notes on no hardcoded API keys, ability to swap Google Sheets with other databases.
      - Reminders to ensure required columns exist.
    - No input/output connections.
    - Version: 1

---

### 3. Summary Table

| Node Name             | Node Type                                | Functional Role                          | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                          |
|-----------------------|----------------------------------------|----------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Input trigger receiving job brief      | —                           | AI Agent                    | This node starts the workflow when a chat message is received. The message should contain a job brief or description. |
| AI Agent              | @n8n/n8n-nodes-langchain.agent         | Extracts 5 keywords from job description | When chat message received  | Parse Keywords              | The agent extracts 5 distinct keywords from the job description using OpenAI.                      |
| OpenAI Chat Model      | @n8n/n8n-nodes-langchain.lmChatOpenAi  | Runs OpenAI GPT-4 model                 | AI Agent (as ai_languageModel) | AI Agent                    |                                                                                                    |
| Parse Keywords        | Code                                   | Parses AI JSON output to object        | AI Agent                    | Converting to Array         |                                                                                                    |
| Converting to Array   | Code                                   | Converts keywords object to array       | Parse Keywords              | Loop Over Items             |                                                                                                    |
| Loop Over Items       | SplitInBatches                         | Iterates over each keyword              | Converting to Array         | Search Templates, Split Out |                                                                                                    |
| Search Templates      | HTTP Request                          | Searches n8n templates by keyword       | Loop Over Items             | Loop Over Items             | For each keyword, the workflow searches the n8n template library API.                              |
| Split Out             | SplitOut                              | Splits template search results          | Loop Over Items             | Append row in sheet2        | Results are split and processed individually.                                                      |
| Append row in sheet2  | Google Sheets                         | Appends new template entry to sheet     | Split Out                  | Get row(s) in sheet         | Appends template search results to Google Sheets.                                                  |
| Get row(s) in sheet   | Google Sheets                         | Retrieves appended row by id and name   | Append row in sheet2        | Edit Fields                |                                                                                                    |
| Edit Fields           | Set                                   | Builds public URL for template          | Get row(s) in sheet         | Update row in sheet         | Builds a public URL for each template and updates the corresponding row.                           |
| Update row in sheet   | Google Sheets                         | Updates sheet row with URL               | Edit Fields                 | —                           |                                                                                                    |
| Sticky Note           | Sticky Note                          | Setup instructions                       | —                           | —                           | Add OpenAI and Google Sheets credentials; set config vars; create required columns in Google Sheet. |
| Sticky Note1          | Sticky Note                          | Workflow overview and general notes     | —                           | —                           | Describes each block and general notes on API keys and sheet columns.                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**
   - Add node: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Name: `When chat message received`
   - Configure webhook to listen for chat messages containing job briefs.
   - No special options needed.
   
2. **Add AI Agent Node**
   - Add node: `@n8n/n8n-nodes-langchain.agent`
   - Name: `AI Agent`
   - Configure system message:
     ```
     You are a helpful assistant.
     Extract exactly 5 distinct search keywords or short phrases from the following job description.
     Return them strictly as a JSON object with the keys "keyword1", "keyword2", "keyword3", "keyword4", "keyword5".
     Do not include explanations or extra text.
     The values must be concise search-ready terms taken directly from the job description context.
     ```
   - Connect `When chat message received` → `AI Agent`.
   
3. **Add OpenAI Chat Model Node**
   - Add node: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
   - Name: `OpenAI Chat Model`
   - Set model to `gpt-4.1-mini`.
   - Add OpenAI API credentials.
   - Connect `OpenAI Chat Model` as AI language model input for `AI Agent`.
   
4. **Add Parse Keywords Node**
   - Add node: Code node.
   - Name: `Parse Keywords`
   - JavaScript:
     ```js
     const items = $input.all();
     const updatedItems = items.map((item) => {
       const output = JSON.parse(item?.json?.output);
       return output;
     });
     return updatedItems;
     ```
   - Connect `AI Agent` → `Parse Keywords`.
   
5. **Add Converting to Array Node**
   - Add node: Code node.
   - Name: `Converting to Array`
   - JavaScript:
     ```js
     const input = $json;
     return [
       { json: { keyword: input.keyword1 } },
       { json: { keyword: input.keyword2 } },
       { json: { keyword: input.keyword3 } },
       { json: { keyword: input.keyword4 } },
       { json: { keyword: input.keyword5 } },
     ];
     ```
   - Connect `Parse Keywords` → `Converting to Array`.
   
6. **Add Loop Over Items Node**
   - Add node: `SplitInBatches`
   - Name: `Loop Over Items`
   - Default batch size (1).
   - Connect `Converting to Array` → `Loop Over Items`.
   
7. **Add Search Templates Node**
   - Add node: `HTTP Request`
   - Name: `Search Templates`
   - Method: GET
   - URL: `https://api.n8n.io/templates/search`
   - Query parameter: `search = {{$json.keyword}}`
   - Header: `Content-Type: application/json`
   - Connect `Loop Over Items` → `Search Templates`.
   
8. **Add Split Out Node**
   - Add node: `SplitOut`
   - Name: `Split Out`
   - Field to split out: `workflows`
   - Connect `Search Templates` → `Loop Over Items` (first output)
   - Connect `Search Templates` → `Split Out` (second output)
   
9. **Add Append Row in Sheet Node**
   - Add node: `Google Sheets`
   - Name: `Append row in sheet2`
   - Operation: Append
   - Document ID: Google Sheets document ID (from config/environment)
   - Sheet Name: Sheet identifier (e.g., `gid=0`)
   - Columns: Map `id`, `Name`, `User`, `Description` from incoming JSON.
   - Add Google Sheets OAuth2 credentials.
   - Connect `Split Out` → `Append row in sheet2`.
   
10. **Add Get Row(s) in Sheet Node**
    - Add node: `Google Sheets`
    - Name: `Get row(s) in sheet`
    - Operation: Lookup rows
    - Filter by `id` and `Name` matching appended row.
    - Connect `Append row in sheet2` → `Get row(s) in sheet`.
    
11. **Add Edit Fields Node**
    - Add node: `Set`
    - Name: `Edit Fields`
    - Add field `URL` with expression:
      ```
      =https://n8n.io/workflows/{{$json.id}}-{{$json.Name.toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/(^-|-$)/g, "")}}
      ```
    - Connect `Get row(s) in sheet` → `Edit Fields`.
    
12. **Add Update Row in Sheet Node**
    - Add node: `Google Sheets`
    - Name: `Update row in sheet`
    - Operation: Update
    - Use `row_number` from previous node to update the correct row.
    - Set `id` to URL field.
    - Connect `Edit Fields` → `Update row in sheet`.
    
13. **Add Sticky Note Nodes for Documentation**
    - Add two `Sticky Note` nodes.
    - One with setup instructions for credentials, environment variables, and Google Sheet setup.
    - One with workflow overview and general notes.
    - Place them appropriately for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Add OpenAI credentials to the LLM nodes and Google Sheets OAuth credentials to all Google Sheets nodes.      | Setup instructions for external services authentication.                                           |
| Set environment variables for GOOGLE_SHEETS_DOC_ID, GOOGLE_SHEET_NAME, and N8N_TEMPLATES_API_URL in n8n Config.| Ensures flexibility and ease of configuration without hardcoding IDs or URLs.                       |
| Create columns in Google Sheets: Template ID (id), Name, User, Description, URL                              | Required schema for logging and updating template records accurately.                               |
| No hardcoded API keys in HTTP Request nodes to follow security best practices.                                | Enhances security and maintainability.                                                             |
| You can replace Google Sheets nodes with Airtable or Notion nodes if preferred.                              | Flexibility to adapt the workflow to other data storage services.                                  |
| Workflow uses OpenAI GPT-4 mini model for keyword extraction; ensure API quota and rate limits are managed.  | Prevents API call failures due to quota exhaustion.                                                |

---

**Disclaimer:**  
The provided content derives exclusively from an automated n8n workflow designed with strict compliance to content policies. No illegal, offensive, or protected materials are involved. All processed data is legal and publicly available.