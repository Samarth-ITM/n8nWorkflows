Generate Template Descriptions from Google Drive with Azure GPT-4

https://n8nworkflows.xyz/workflows/generate-template-descriptions-from-google-drive-with-azure-gpt-4-9344


# Generate Template Descriptions from Google Drive with Azure GPT-4

### 1. Workflow Overview

This workflow automates the generation of structured template descriptions for n8n workflows stored as JSON files in a specific Google Drive folder. It is designed to streamline the documentation process by leveraging Azure OpenAI's GPT-4 model combined with LangChain agents for intelligent content analysis and generation.

**Target Use Cases:**  
- Automatically generating descriptive titles and detailed, structured descriptions for n8n workflow templates.  
- Batch processing multiple JSON workflow files stored in Google Drive.  
- Formatting output for publishing, logging, and communication via email.

**Logical Blocks:**

- **1.1 Manual Trigger:** Initiates the workflow manually for testing or batch processing.  
- **1.2 Google Drive File Search & Retrieval:** Searches a specific Drive folder for JSON workflow files, downloads them, and extracts JSON content.  
- **1.3 AI Processing:** Sends the JSON workflow data to an AI agent (Azure OpenAI + LangChain) to generate a template title and structured description.  
- **1.4 Parsing & Formatting:** Parses the AI response into structured JSON, formats it into Markdown and HTML email content.  
- **1.5 Output Handling:** Saves the generated description to Google Sheets and sends the formatted email with the description.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  Allows manual initiation of the entire workflow, useful for controlled testing or batch execution.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)  
  - **Configuration:** No parameters; triggers workflow on manual execution.  
  - **Connections:** Outputs to "Search files and folders".  
  - **Failure Modes:** None significant; manual trigger depends on user execution.

#### 1.2 Google Drive File Search & Retrieval

- **Overview:**  
  Searches the specified Google Drive folder for all JSON workflow files, downloads each file, and extracts the JSON content for AI processing.

- **Nodes Involved:**  
  - Search files and folders  
  - Loop Over JSONS  
  - Download file  
  - Extract from File

- **Node Details:**  

  - **Search files and folders**  
    - *Type:* Google Drive (n8n-nodes-base.googleDrive)  
    - *Role:* Searches Google Drive folder by folder ID for JSON files.  
    - *Configuration:* Folder ID hardcoded (`1HP3LnTPLwe81xUrp0P6aV2nJKdX6BIcM`), returns all matching files.  
    - *Credentials:* Google Drive OAuth2 ("Techdome Account").  
    - *Connections:* Outputs to "Loop Over JSONS".  
    - *Edge Cases:* Folder ID incorrect or inaccessible; OAuth token expired; no files found.

  - **Loop Over JSONS**  
    - *Type:* SplitInBatches (n8n-nodes-base.splitInBatches)  
    - *Role:* Processes each found file individually to handle large datasets efficiently.  
    - *Configuration:* Uses default batch size (can be adjusted).  
    - *Connections:* Outputs to "Download file".  
    - *Edge Cases:* Large batch sizes may cause timeouts; empty input.

  - **Download file**  
    - *Type:* Google Drive (n8n-nodes-base.googleDrive)  
    - *Role:* Downloads the file content for each Drive item.  
    - *Configuration:* Uses current item's file ID to download.  
    - *Credentials:* Same Google Drive OAuth2.  
    - *Connections:* Outputs to "Extract from File".  
    - *Edge Cases:* File permissions preventing download; network issues.

  - **Extract from File**  
    - *Type:* ExtractFromFile (n8n-nodes-base.extractFromFile)  
    - *Role:* Parses downloaded file content as JSON.  
    - *Configuration:* Operation set to "fromJson".  
    - *Connections:* Outputs to "AI Agent".  
    - *Edge Cases:* Malformed JSON; empty file content.

#### 1.3 AI Processing

- **Overview:**  
  Sends the extracted JSON workflow data to an AI agent powered by Azure OpenAI GPT-4 and LangChain agents to generate a concise template title and a structured description following strict formatting rules.

- **Nodes Involved:**  
  - AI Agent  
  - Connect to Azure OpenAI GPT Model  
  - Store AI Context (LangChain Memory)  
  - Parse AI Response into Structured JSON

- **Node Details:**  

  - **Connect to Azure OpenAI GPT Model**  
    - *Type:* LangChain Chat Model (lmChatAzureOpenAi)  
    - *Role:* Connects to Azure OpenAI GPT-4 model ("gpt-4o") for text generation.  
    - *Configuration:* Model set to "gpt-4o".  
    - *Credentials:* Azure OpenAI API credential linked.  
    - *Connections:* Provides AI language model input to "AI Agent".  
    - *Edge Cases:* API key invalid/expired; model name mismatch; request timeouts.

  - **Store AI Context (LangChain Memory)**  
    - *Type:* LangChain Memory Buffer (memoryBufferWindow)  
    - *Role:* Maintains short-term AI context over 7 previous interactions to improve prompt continuity.  
    - *Configuration:* Custom session key `"json_review"`, context window length 7.  
    - *Connections:* Supplies AI memory to "AI Agent".  
    - *Edge Cases:* Memory overflow if context too large; session ID conflicts.

  - **AI Agent**  
    - *Type:* LangChain Agent (@n8n/n8n-nodes-langchain.agent)  
    - *Role:* Runs prompt using the Azure OpenAI model and memory to generate the template title and structured description.  
    - *Configuration:*  
      - Prompt includes full JSON workflow input.  
      - System message defines expert role and strict formatting rules.  
      - Output parser attached for structured JSON compliance.  
    - *Connections:*  
      - Receives AI model and memory inputs.  
      - Outputs to "Format Markdown Description".  
    - *Edge Cases:* AI response not conforming to schema; model rate limits; prompt failures.

  - **Parse AI Response into Structured JSON**  
    - *Type:* LangChain Output Parser (outputParserStructured)  
    - *Role:* Parses AI output into a JSON schema with fields: title, overview, steps, benefits, features, requirements, audience, setup instructions.  
    - *Configuration:* Uses example JSON schema for validation.  
    - *Connections:* Feeds parsed JSON back to "AI Agent".  
    - *Edge Cases:* Parsing errors if AI output malformed; schema mismatches.

#### 1.4 Parsing & Formatting

- **Overview:**  
  Converts the structured JSON description from the AI into human-readable Markdown and HTML email formats for publishing and communication.

- **Nodes Involved:**  
  - Format Markdown Description  
  - Format HTML Email

- **Node Details:**  

  - **Format Markdown Description**  
    - *Type:* Code (n8n-nodes-base.code)  
    - *Role:* Takes AI JSON output and formats it into a Markdown description with emojis and structured lists for clarity.  
    - *Configuration:* Custom JavaScript code formats sections: description, steps, benefits, features, requirements, audience, setup instructions.  
    - *Connections:* Outputs to "Save To Google Sheets".  
    - *Edge Cases:* Input JSON missing expected fields; formatting logic errors.

  - **Format HTML Email**  
    - *Type:* Code (n8n-nodes-base.code)  
    - *Role:* Converts Markdown description into styled HTML email content and plain-text version for logging.  
    - *Configuration:* Custom JavaScript with inline CSS styling for visual clarity and branding.  
    - *Connections:* Outputs to "Send Email With Description".  
    - *Edge Cases:* HTML injection if input not sanitized; email client rendering issues.

#### 1.5 Output Handling

- **Overview:**  
  Saves the generated template description to a Google Sheet for record-keeping and sends a formatted HTML email containing the description.

- **Nodes Involved:**  
  - Save To Google Sheets  
  - Send Email With Description

- **Node Details:**  

  - **Save To Google Sheets**  
    - *Type:* Google Sheets (n8n-nodes-base.googleSheets)  
    - *Role:* Appends or updates the sheet with title and formatted description data.  
    - *Configuration:* Auto-maps input data columns for title and description, targets specific Google Sheet and sheet tab by ID.  
    - *Credentials:* Google Sheets OAuth2 ("automations@techdome.ai").  
    - *Connections:* Outputs to "Format HTML Email" (feedback loop for email sending).  
    - *Edge Cases:* Sheet access issues; quota limits; data type mismatches.

  - **Send Email With Description**  
    - *Type:* Gmail (n8n-nodes-base.gmail)  
    - *Role:* Sends the formatted HTML email to a specified recipient.  
    - *Configuration:* Sends to static email (placeholder: "your-email@example.com"), subject is dynamic with template title, message body is HTML.  
    - *Credentials:* Gmail OAuth2 credentials linked.  
    - *Connections:* Loops back to "Loop Over JSONS" to process next file.  
    - *Edge Cases:* Invalid email address; Gmail API quota or auth errors.

---

### 3. Summary Table

| Node Name                      | Node Type                              | Functional Role                         | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                   |
|--------------------------------|--------------------------------------|---------------------------------------|----------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger (n8n-nodes-base.manualTrigger) | Starts workflow manually               | —                                | Search files and folders          | ⚙️ Step 1: Manual Trigger. No setup required.                                                 |
| Search files and folders        | Google Drive (n8n-nodes-base.googleDrive) | Searches JSON files in specific Drive folder | When clicking ‘Execute workflow’ | Loop Over JSONS                  | 📂 Step 2: Search Files and Folders. Setup: Google Drive OAuth2, replace folder ID if needed. |
| Loop Over JSONS                 | SplitInBatches (n8n-nodes-base.splitInBatches) | Processes files individually           | Search files and folders          | Download file                   | 🔁 Step 3: Loop Over JSONS. Adjust batch size if needed.                                     |
| Download file                  | Google Drive (n8n-nodes-base.googleDrive) | Downloads file contents                 | Loop Over JSONS                   | Extract from File                | ⬇️ Step 4: Download File. Setup: Google Drive OAuth2, ensure permissions.                    |
| Extract from File               | ExtractFromFile (n8n-nodes-base.extractFromFile) | Parses downloaded content as JSON      | Download file                    | AI Agent                       | 📦 Step 5: Extract JSON Data. No credentials needed.                                         |
| Connect to Azure OpenAI GPT Model | LangChain Chat Model (@n8n/n8n-nodes-langchain.lmChatAzureOpenAi) | Connects to Azure OpenAI GPT-4 model  | —                                | AI Agent (ai_languageModel)      | 💬 Step 7: Connect to Azure OpenAI GPT Model. Setup: Azure OpenAI credentials, verify model. |
| Store AI Context (LangChain Memory) | LangChain Memory Buffer (@n8n/n8n-nodes-langchain.memoryBufferWindow) | Maintains short-term AI context        | —                                | AI Agent (ai_memory)             | 🧠 Step 8: Store AI Context (LangChain Memory). Keep context window small (7–10).             |
| AI Agent                      | LangChain Agent (@n8n/n8n-nodes-langchain.agent) | Runs AI prompt, generates title & description | Extract from File, Connect to Azure OpenAI GPT Model, Store AI Context | Format Markdown Description     | 🤖 Step 6: AI Agent. Setup: AI model and output parser. Well-structured prompt.              |
| Parse AI Response into Structured JSON | LangChain Output Parser (@n8n/n8n-nodes-langchain.outputParserStructured) | Parses AI response to structured JSON | AI Agent (ai_outputParser)        | AI Agent                       |                                                                                               |
| Format Markdown Description    | Code (n8n-nodes-base.code)            | Formats AI JSON output into Markdown  | AI Agent                        | Save To Google Sheets            | 🧾 Step 10: Format Markdown Description. Adjust emojis or section titles if needed.           |
| Save To Google Sheets           | Google Sheets (n8n-nodes-base.googleSheets) | Saves generated data to spreadsheet    | Format Markdown Description      | Format HTML Email                | 🧾 Step 12: Save to Google Sheets.                                                           |
| Format HTML Email               | Code (n8n-nodes-base.code)            | Formats Markdown into HTML email & plain text | Save To Google Sheets            | Send Email With Description      | ✨ Step 11: Format HTML Email. Adjust CSS styling as needed.                                 |
| Send Email With Description     | Gmail (n8n-nodes-base.gmail)           | Sends formatted HTML email             | Format HTML Email                | Loop Over JSONS                 | 📧 Step 13: Send Email with Description. Setup Gmail OAuth2 credentials.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger (n8n-nodes-base.manualTrigger)  
   - No special configuration.  
   - Position at the start of the workflow.

2. **Add Google Drive "Search files and folders" Node**  
   - Type: Google Drive (n8n-nodes-base.googleDrive)  
   - Operation: Search files/folders in folder  
   - Configure: Set Folder ID to your target Drive folder containing JSON workflows.  
   - Credentials: Connect Google Drive OAuth2 credentials.  
   - Connect output from Manual Trigger.

3. **Add SplitInBatches Node "Loop Over JSONS"**  
   - Type: SplitInBatches (n8n-nodes-base.splitInBatches)  
   - Default batch size is fine or adjust as needed.  
   - Connect input from "Search files and folders".

4. **Add Google Drive "Download file" Node**  
   - Type: Google Drive (n8n-nodes-base.googleDrive)  
   - Operation: Download  
   - File ID: Use expression to pass current item's `id` (`={{$json["id"]}}`).  
   - Credentials: Use same Google Drive OAuth2.  
   - Connect output from "Loop Over JSONS".

5. **Add ExtractFromFile Node "Extract from File"**  
   - Type: ExtractFromFile (n8n-nodes-base.extractFromFile)  
   - Operation: fromJson  
   - Connect output from "Download file".

6. **Add LangChain Chat Model Node "Connect to Azure OpenAI GPT Model"**  
   - Type: LangChain Chat Model (@n8n/n8n-nodes-langchain.lmChatAzureOpenAi)  
   - Model: Set to "gpt-4o" or your Azure GPT-4 equivalent.  
   - Credentials: Connect Azure OpenAI API credentials.  
   - Position near AI Agent nodes.

7. **Add LangChain Memory Buffer Node "Store AI Context (LangChain Memory)"**  
   - Type: LangChain Memory Buffer (@n8n/n8n-nodes-langchain.memoryBufferWindow)  
   - Session Key: `"json_review"` (string)  
   - Context Window Length: 7  
   - No credentials needed.

8. **Add LangChain Agent Node "AI Agent"**  
   - Type: LangChain Agent (@n8n/n8n-nodes-langchain.agent)  
   - Text prompt: Use expression to insert JSON workflow content, instructing AI to generate title and structured description with strict formatting rules (as per the system message in the original).  
   - System message: Define expert role and output formatting rules (copy from original for accuracy).  
   - Connect AI model input from "Connect to Azure OpenAI GPT Model".  
   - Connect AI memory input from "Store AI Context".  
   - Connect its output parser input to the next node (see next step).

9. **Add LangChain Output Parser Node "Parse AI Response into Structured JSON"**  
   - Type: LangChain Output Parser (@n8n/n8n-nodes-langchain.outputParserStructured)  
   - JSON Schema Example: Use the schema defining fields like title, overview, steps, benefits, features, requirements, audience, setup_instructions.  
   - Connect output parser to "AI Agent" node's output parser input.

10. **Connect "Extract from File" output to "AI Agent" input**  
    - This links the JSON workflow data directly into the AI prompt.

11. **Add Code Node "Format Markdown Description"**  
    - Type: Code (n8n-nodes-base.code)  
    - JavaScript: Use the provided code to format AI JSON output into a Markdown string with emojis and lists.  
    - Connect output from "AI Agent".

12. **Add Google Sheets Node "Save To Google Sheets"**  
    - Type: Google Sheets (n8n-nodes-base.googleSheets)  
    - Operation: Append or Update  
    - Document ID: Set to your target Google Sheet ID.  
    - Sheet Name: Specify tab (gid=0 or name).  
    - Columns: Map `title` and `formatted_description` fields automatically.  
    - Credentials: Connect Google Sheets OAuth2 credentials.  
    - Connect output from "Format Markdown Description".

13. **Add Code Node "Format HTML Email"**  
    - Type: Code (n8n-nodes-base.code)  
    - JavaScript: Use provided code to convert Markdown description into styled HTML email and plain text.  
    - Connect output from "Save To Google Sheets".

14. **Add Gmail Node "Send Email With Description"**  
    - Type: Gmail (n8n-nodes-base.gmail)  
    - Parameters:  
      - Send To: Replace with your recipient email.  
      - Subject: Use expression `={{ 'n8n Template: ' + $json.title }}`  
      - Message: Use expression for HTML email content.  
    - Credentials: Connect Gmail OAuth2 credentials.  
    - Connect output from "Format HTML Email".

15. **Connect "Send Email With Description" output back to "Loop Over JSONS"**  
    - This loops the workflow to process the next JSON file in batch.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow automates the creation of structured n8n template descriptions using AI and Google Drive JSON files. | Workflow Overview sticky note in the JSON.                                                                  |
| Use your own Google Drive, Azure OpenAI, Gmail, and Google Sheets credentials for authentication. | Credential setup notes throughout the workflow sticky notes.                                                |
| For best AI results, ensure the JSON workflow files are well-formed and accessible to Google Drive OAuth2 credentials. | Extract from File and Download file nodes notes.                                                             |
| The AI prompt follows strict n8n Template Publishing Guidelines ensuring consistent output structure. | AI Agent node sticky note with detailed prompt instructions.                                                |
| Adjust batch size in "Loop Over JSONS" node to manage large numbers of files and avoid timeouts. | Loop Over JSONS sticky note.                                                                                 |
| CSS styling in "Format HTML Email" node can be customized for branding purposes.                   | Format HTML Email sticky note.                                                                               |
| Gmail step uses OAuth2; test with a non-personal account if publishing descriptions publicly.      | Send Email With Description sticky note.                                                                     |
| For more insights on AI integration with n8n, visit the official n8n documentation and LangChain GitHub. | External resource suggestion (not in workflow but relevant for users).                                        |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected material. All data processed is legal and public.