Create Custom PDF Documents from Templates with Gemini & Google Drive

https://n8nworkflows.xyz/workflows/create-custom-pdf-documents-from-templates-with-gemini---google-drive-5808


# Create Custom PDF Documents from Templates with Gemini & Google Drive

### 1. Workflow Overview

This workflow automates the creation of custom PDF documents from Google Docs templates by interacting with users via a chat interface. It is designed for legal or professional document generation where templates contain placeholders and conditional blocks that are dynamically filled based on user input.

**Target Use Cases:**  
- Automating contract, form, or legal document generation.  
- Enabling users to select templates, provide required data interactively, and receive ready-to-sign PDFs quickly.  
- Managing conditional sections within documents to tailor outputs precisely.  
- Maintaining a scalable and no-code solution integrated with Google Drive and Google Apps Script.

**Logical Blocks:**

- **1.1 Input Reception and User Interaction:** Receives chat messages, manages conversation flow, and prompts users for information.  
- **1.2 Template Discovery and Metadata Retrieval:** Lists available templates from Google Drive and fetches metadata (placeholders, conditionals) for a selected template.  
- **1.3 User Input Processing and Validation:** Matches user selections with templates, verifies inputs, and formats user data according to the template metadata.  
- **1.4 Document Processing Sub-Workflow:** Copies the template, fills placeholders/conditional blocks using Apps Script, and generates PDF download links.  
- **1.5 Error Handling and Output Delivery:** Handles errors arising at various stages and delivers the final document or error messages to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and User Interaction

**Overview:**  
This block initiates the workflow by listening for chat messages, managing chat memory, and orchestrating the main agent logic that interacts with the user to select a document and collect required data.

**Nodes Involved:**  
- When chat message received  
- Postgres Chat Memory  
- Google Gemini Chat Model  
- DocAgent  

**Node Details:**  

- **When chat message received**  
  - *Type:* Langchain chat trigger webhook  
  - *Role:* Entry point for user chat messages; starts conversation.  
  - *Configuration:* Public webhook, initial greeting message: "Merhaba! 👋 My name is DocAgent."  
  - *Connections:* Output → DocAgent  
  - *Potential Issues:* Webhook connectivity or permissions.

- **Postgres Chat Memory**  
  - *Type:* Langchain Postgres memory node  
  - *Role:* Stores conversational context for coherent multi-turn dialogue.  
  - *Credentials:* Postgres connection named "DocAgentMemory."  
  - *Connected to:* DocAgent (memory input)  
  - *Notes:* Preferred for consistent memory management but optional.

- **Google Gemini Chat Model (Primary)**  
  - *Type:* Langchain Google Gemini language model  
  - *Role:* Generates AI responses in chat.  
  - *Model:* "models/gemini-2.5-pro"  
  - *Credentials:* Google Palm API key  
  - *Connected to:* DocAgent (language model input)  
  - *Potential Issues:* API quota, auth errors, latency.

- **DocAgent**  
  - *Type:* Langchain agent node  
  - *Role:* Main conversational agent implementing business logic for document generation; parses user intent, calls tools, and manages flow.  
  - *Configuration:* System prompt defines behavior as expert Turkish legal document drafter; includes detailed instructions for the flow and tool usage.  
  - *Tools:* Calls GetMetaData and DocProcess workflows as tools.  
  - *Error Handling:* Encourages asking users for missing data; returns JSON only in tool call messages.  
  - *Connections:* Receives from chat trigger, memory, and language model; outputs back to chat.  
  - *Potential Issues:* Expression errors in prompt, AI model misinterpretation.

---

#### 1.2 Template Discovery and Metadata Retrieval

**Overview:**  
This block discovers available templates via Google Drive API and retrieves metadata for the selected template using a Google Apps Script web app.

**Nodes Involved:**  
- Template List  
- GetMetaData  

**Node Details:**  

- **Template List**  
  - *Type:* HTTP Request  
  - *Role:* Fetches list of Google Docs templates from a specific Drive folder by querying Drive API.  
  - *Configuration:*  
    - URL includes folder ID placeholder (`<YOUR_PARENT_ID>`) for Drive folder containing templates.  
    - Returns files with id, name, and description.  
  - *Credentials:* Google Drive OAuth2  
  - *Potential Issues:* Invalid folder ID, OAuth permission errors, API rate limits.

- **GetMetaData**  
  - *Type:* HTTP Request Tool  
  - *Role:* Calls Google Apps Script endpoint to get metadata (placeholders, conditional blocks) of a chosen template document.  
  - *Configuration:*  
    - URL to Apps Script web app with query parameters `mode=meta` and `id={{templateId}}`.  
    - Sends headers to accept JSON.  
  - *Credentials:* Google Drive OAuth2 (for Drive API access)  
  - *Input:* Template ID from user selection or agent.  
  - *Output:* JSON with placeholders and conditional blocks metadata.  
  - *Potential Issues:* Apps Script deployment or permission errors, invalid template ID.

---

#### 1.3 User Input Processing and Validation

**Overview:**  
This block validates user selection, retrieves updated metadata, formats user input data according to metadata, and prepares data for document generation.

**Nodes Involved:**  
- User Choice Match Check  
- Structured Output Parser  
- If (validation)  
- User Choice Match Correct  
- User Choice Matching Error  
- GetMetaData2  
- Edit Fields  
- Google Gemini Chat Model1  
- Format Control  
- Formatting Correction  

**Node Details:**  

- **User Choice Match Check**  
  - *Type:* Langchain chain LLM  
  - *Role:* Verifies that the user's selected document name matches the given template ID, preventing mismatches.  
  - *Configuration:* System prompt includes manual template list, expects JSON output with boolean match flag.  
  - *Input:* user_choice_name, user_choice_id  
  - *Output:* JSON with match status.  
  - *Potential Issues:* Incorrect manual template list, LLM parsing errors.

- **Structured Output Parser**  
  - *Type:* Langchain output parser  
  - *Role:* Parses JSON output from User Choice Match Check for structured data consumption.  
  - *Configuration:* JSON schema example provided.  
  - *Connections:* Output → User Choice Match Check.

- **If (validation)**  
  - *Type:* Conditional  
  - *Role:* Routes flow based on whether user choice matched or not.  
  - *True:* User Choice Match Correct  
  - *False:* User Choice Matching Error

- **User Choice Match Correct**  
  - *Type:* Set  
  - *Role:* Passes validated user choice ID forward.  
  - *Output:* user_choice_id to next node.

- **User Choice Matching Error**  
  - *Type:* Set  
  - *Role:* Sets an error message if mismatch occurs.  
  - *Output:* Message for user notification.

- **GetMetaData2**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves fresh metadata based on validated user_choice_id.  
  - *Input:* user_choice_id from User Choice Match Correct node.

- **Edit Fields**  
  - *Type:* Set  
  - *Role:* Prepares or adjusts fields before formatting.

- **Google Gemini Chat Model1**  
  - *Type:* Langchain language model  
  - *Role:* Used in conjunction with Format Control for formatting user data.

- **Format Control**  
  - *Type:* Langchain chain LLM  
  - *Role:* Formats user inputs strictly according to metadata rules, preserving placeholders and conditional flags exactly.  
  - *Input:* Metadata JSON and raw user data JSON.  
  - *Output:* Properly structured JSON for document filling.

- **Formatting Correction**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses AI output to extract JSON from formatted text, handling cases where JSON is embedded in markdown code blocks.  
  - *Output:* Parsed JSON object with final data structure.

---

#### 1.4 Document Processing Sub-Workflow

**Overview:**  
This sub-workflow performs the actual document generation: making a copy of the template, filling it with user data, and generating a PDF download link.

**Nodes Involved:**  
- When Executed by Another Workflow  
- User Choice Match Check (reused)  
- If (inside sub-workflow)  
- CopyTemplate  
- If1  
- FillDocument  
- if (status check)  
- Generate Download Link  
- Download Link Format  
- Switch  
- Template Technical Error  
- Incomplete Information Error  
- Other Errors  
- Cop. Document ID Matching Error  
- User Choice Match Correct  

**Node Details:**  

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point for the DocProcess sub-workflow when invoked by the main agent.  
  - *Inputs:* user_choice_name, user_choice_id, data (user inputs).

- **User Choice Match Check** (in sub-workflow)  
  - *Role:* Validates again that user choice name and ID match before proceeding with document generation.

- **If** (sub-workflow)  
  - *Role:* Routes flow based on match result.

- **User Choice Match Correct**  
  - *Role:* Proceeds to metadata retrieval and document processing.

- **User Choice Matching Error**  
  - *Role:* Sets error message about mismatch.

- **GetMetaData2**  
  - *Role:* Retrieves latest metadata for document copy.

- **Edit Fields**  
  - *Role:* Prepares data for formatting.

- **Format Control**  
  - *Role:* Formats data for Apps Script consumption.

- **Formatting Correction**  
  - *Role:* Parses formatted JSON.

- **CopyTemplate**  
  - *Type:* Google Drive node  
  - *Role:* Copies the original template document to a specified folder for filling.  
  - *Configuration:*  
    - Name based on current date and document title.  
    - Copies file by ID, places in `<YOUR_FOLDER_ID>`.  
  - *Credentials:* Google Drive OAuth2  
  - *Output:* New copied document ID.

- **If1**  
  - *Role:* Checks if copied document ID matches expected ID before filling.

- **Cop. Document ID Matching Error**  
  - *Role:* Sets error message if mismatch occurs.

- **FillDocument**  
  - *Type:* HTTP Request (POST)  
  - *Role:* Calls Apps Script web app to fill placeholders and remove conditional blocks in the copied document.  
  - *Input:* docId (copied document ID) and formatted data JSON.  
  - *Potential Errors:* Apps Script errors, malformed data, permission issues.

- **if** (status check)  
  - *Role:* Checks if FillDocument returned status "OK".

- **Generate Download Link**  
  - *Type:* HTTP Request  
  - *Role:* Fetches Google Drive export links for the filled document.  
  - *Output:* Contains webContentLink and exportLinks (including PDF).

- **Download Link Format**  
  - *Type:* Set  
  - *Role:* Extracts the PDF export link from the previous node's output.

- **Switch**  
  - *Role:* Handles errors returned from FillDocument:  
    - Template Technical Error (unknown placeholders)  
    - Incomplete Information Error (missing fields)  
    - Other Errors (general errors)  

- **Template Technical Error / Incomplete Information Error / Other Errors**  
  - *Type:* Set  
  - *Role:* Format detailed error messages for user feedback.

---

#### 1.5 Error Handling and Output Delivery

**Overview:**  
The workflow contains multiple error checks to ensure users receive meaningful feedback and that the process can recover from common issues such as missing fields, template mismatches, or script errors.

**Nodes Involved:**  
- User Choice Matching Error  
- Cop. Document ID Matching Error  
- Template Technical Error  
- Incomplete Information Error  
- Other Errors  

**Node Details:**  
Each error node sets a descriptive message explaining the cause and, where appropriate, suggests corrective actions for the user or administrator.

---

### 3. Summary Table

| Node Name                   | Node Type                                  | Functional Role                                   | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                                                    |
|-----------------------------|--------------------------------------------|--------------------------------------------------|---------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received  | Langchain chat trigger webhook              | Entry point for chat input                        | —                               | DocAgent                      |                                                                                                                                               |
| Postgres Chat Memory        | Langchain Postgres memory                    | Stores conversation history                       | —                               | DocAgent                      | It is preferred for more consistent memory management. Other alternatives can also be tried.                                                   |
| Google Gemini Chat Model    | Langchain language model                     | Generates AI responses                            | —                               | DocAgent                      | Prioritized due to free API-Key trial.                                                                                                        |
| DocAgent                   | Langchain agent                              | Main chat agent and workflow orchestrator        | When chat message received, Memory, Language Model | Chat response output | Metadata of the template selected by the user is dynamically retrieved.                                                                       |
| Template List              | HTTP Request                                | Retrieves list of templates from Google Drive    | —                               | —                            | Manual template list retrieval. The response returned from this API request is manually added to the system prompt.                            |
| GetMetaData                | HTTP Request Tool                           | Retrieves metadata of selected template           | DocAgent                        | DocAgent                      | Metadata of the template selected by the user is dynamically retrieved.                                                                       |
| User Choice Match Check    | Langchain chain LLM                         | Validates user choice matches template ID         | When Executed by Another Workflow, DocProcess subworkflow | If                         |                                                                                                                                               |
| Structured Output Parser   | Langchain output parser                      | Parses JSON output from User Choice Match Check  | Google Gemini Chat Model2        | User Choice Match Check       |                                                                                                                                               |
| If                        | Conditional                                 | Routes based on user choice match result          | User Choice Match Check          | User Choice Match Correct, User Choice Matching Error |                                                                                                                                               |
| User Choice Match Correct  | Set                                         | Passes validated user choice forward               | If                             | GetMetaData2                  |                                                                                                                                               |
| User Choice Matching Error | Set                                         | Sets error message on mismatch                      | If                             | —                            |                                                                                                                                               |
| GetMetaData2              | HTTP Request                                | Fetches latest metadata for validated template    | User Choice Match Correct        | Edit Fields                   | Metadata of the template selected by the user is dynamically retrieved.                                                                       |
| Edit Fields               | Set                                         | Prepares fields before formatting                   | GetMetaData2                   | Format Control                |                                                                                                                                               |
| Google Gemini Chat Model1  | Langchain language model                     | Supports formatting control                         | —                             | Format Control                |                                                                                                                                               |
| Format Control            | Langchain chain LLM                         | Formats user inputs according to metadata          | Edit Fields, Google Gemini Chat Model1 | Formatting Correction       |                                                                                                                                               |
| Formatting Correction     | Code (JavaScript)                           | Parses and cleans AI output JSON                    | Format Control                 | CopyTemplate                  |                                                                                                                                               |
| CopyTemplate              | Google Drive                                | Copies template document for filling               | Formatting Correction          | If1                          |                                                                                                                                               |
| If1                       | Conditional                                 | Checks copied document ID match                     | CopyTemplate                  | Cop. Document ID Matching Error, FillDocument |                                                                                                                                               |
| Cop. Document ID Matching Error | Set                                     | Sets error message if copy ID mismatch              | If1                          | —                            |                                                                                                                                               |
| FillDocument              | HTTP Request                                | Calls Apps Script to fill placeholders              | If1                          | if                           |                                                                                                                                               |
| if                        | Conditional                                 | Checks FillDocument status                           | FillDocument                  | Generate Download Link, Switch |                                                                                                                                               |
| Generate Download Link    | HTTP Request                                | Gets export links for the filled doc                | if                           | Download Link Format          |                                                                                                                                               |
| Download Link Format      | Set                                         | Extracts PDF download link                           | Generate Download Link         | —                            |                                                                                                                                               |
| Switch                    | Conditional                                 | Handles errors from FillDocument                     | if                           | Template Technical Error, Incomplete Information Error, Other Errors |                                                                                                                                               |
| Template Technical Error  | Set                                         | Error message for unknown placeholders               | Switch                       | —                            |                                                                                                                                               |
| Incomplete Information Error | Set                                       | Error message for missing required fields            | Switch                       | —                            |                                                                                                                                               |
| Other Errors              | Set                                         | General error message                                 | Switch                       | —                            |                                                                                                                                               |
| When Executed by Another Workflow | Execute Workflow Trigger              | Entry trigger for DocProcess sub-workflow            | —                             | User Choice Match Check       |                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node:**  
   - Type: Langchain Chat Trigger  
   - Set Public: true  
   - Initial Messages: "Merhaba! 👋 My name is DocAgent."  
   - Position: top-left  

2. **Add Postgres Chat Memory Node:**  
   - Type: Langchain Postgres Memory  
   - Configure with Postgres credentials (e.g., "DocAgentMemory").  
   - Connect Webhook output → Postgres Memory → DocAgent.  

3. **Add Google Gemini Chat Model Node:**  
   - Type: Langchain Google Gemini LLM  
   - Model: "models/gemini-2.5-pro"  
   - Set Google Palm API credentials.  
   - Connect to DocAgent node as language model input.  

4. **Add DocAgent Node (Langchain Agent):**  
   - Paste the detailed system prompt describing role and flow.  
   - Define tools: GetMetaData and DocProcess (sub-workflow).  
   - Connect inputs: webhook, memory, language model.  
   - Output connects back to chat response.  

5. **Create Template List Node (HTTP Request):**  
   - URL: `https://www.googleapis.com/drive/v3/files?q='<YOUR_PARENT_ID>'+in+parents+and+trashed=false&fields=files(id,name,description)&pageSize=1000` (replace `<YOUR_PARENT_ID>` with Google Drive folder ID).  
   - Authentication: Google Drive OAuth2.  
   - Method: GET.  
   - Connect as needed for manual retrieval (used to build manual template list).  

6. **Create GetMetaData Node (HTTP Request Tool):**  
   - URL: `<WEB_APP_URL>?mode=meta&id={{ $json["id"] }}` (Apps Script Web App URL).  
   - Authentication: Google Drive OAuth2.  
   - Send query parameters: mode=meta, id from input.  
   - Used by DocAgent to get template metadata dynamically.  

7. **Create DocProcess Sub-Workflow:**  
   - Entry node: Execute Workflow Trigger, with inputs: user_choice_name (string), user_choice_id (string), data (object).  

8. **Inside DocProcess Sub-Workflow:**  

   a. Add User Choice Match Check (Langchain Chain LLM)  
      - System prompt includes manual template list for validation.  
      - Outputs JSON with match boolean.  
      - Connect output to If node.

   b. Add If node to check match:  
      - True: continue  
      - False: set error message & stop.  

   c. Add GetMetaData2 HTTP Request to get fresh metadata (same URL pattern as GetMetaData).  

   d. Add Edit Fields Set node to prepare data.  

   e. Add Google Gemini Chat Model1 for formatting.  

   f. Add Format Control chain LLM node:  
      - Input metadata and user data JSON.  
      - Output formatted data JSON.  

   g. Add Formatting Correction Code node:  
      - Parses JSON from AI output with error handling.  

   h. Add CopyTemplate Google Drive node:  
      - Operation: copy file using docId from formatted data.  
      - Destination folder `<YOUR_FOLDER_ID>`.  
      - Name: current date + template title.  

   i. Add If1 node to verify copied document ID matches expected.  
      - True: proceed to FillDocument.  
      - False: set error message.  

   j. Add FillDocument HTTP Request node:  
      - POST to Apps Script URL.  
      - Body parameters: docId (from copied doc), data (formatted JSON).  

   k. Add conditional if node to check FillDocument response status == "OK".  
      - True: Generate Download Link.  
      - False: Switch node for error routing.  

   l. Add Generate Download Link HTTP Request node:  
      - GET Google Drive file metadata with exportLinks.  

   m. Add Download Link Format Set node:  
      - Extract PDF link from exportLinks.  

   n. Add Switch node to handle FillDocument errors:  
      - Template Technical Error (unknown placeholders)  
      - Incomplete Information Error (missing placeholders)  
      - Other errors (generic).  

   o. Add Set nodes for error messages accordingly.  

9. **Wire all nodes according to logical flow and connections above.**

10. **Credential Setup:**  
    - Google Drive OAuth2 with Drive API full access.  
    - Google Docs OAuth2 with Docs API access.  
    - Google Palm API key for Gemini model.  
    - Postgres credentials for chat memory (optional).  

11. **Apps Script Deployment:**  
    - Deploy Apps Script project with GetMetaData and FillDocument functions as a Web App.  
    - Set permissions: Execute as Me, accessible by Anyone.  
    - Copy Web App URL to be used in GetMetaData and FillDocument nodes.  

12. **Template Preparation:**  
    - Prepare Google Docs templates in a Drive folder.  
    - Use placeholder format `{{PLACEHOLDER}}` (uppercase ASCII).  
    - Optional blocks wrapped with `[[BLOCK_NAME:START]]` / `[[BLOCK_NAME:END]]`.  
    - Add META_JSON block at document end for metadata.  
    - Maintain a Generated folder for copies.  

13. **Manual Template List:**  
    - Run Template List node to fetch template metadata.  
    - Paste JSON output into system prompts for DocAgent and User Choice Match Check nodes.  
    - Update whenever templates change.  

14. **Activate workflow and test:**  
    - Use chat interface to start (`/start`).  
    - Select template, answer prompts, confirm data.  
    - Receive PDF download link.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow automates Google Docs-based contract and form template filling into ready-to-sign PDFs through chat interaction, eliminating manual document creation and errors.                                                                                                                                                                                                | Described in Sticky Note4 content.                                                                                |
| Manual template list caching is recommended to save LLM tokens; update manually after adding templates.                                                                                                                                                                                                                                                                         | Sticky Note regarding Template List and manual insertion.                                                        |
| Apps Script deployment must enable Google Docs and Drive APIs; set Web App permissions to "Anyone" and "Execute as Me" to avoid 403 errors.                                                                                                                                                                                                                                  | Setup steps in Sticky Note4 and detailed instructions.                                                           |
| Placeholders must be uppercase ASCII, no Turkish characters or spaces; conditional blocks require blank line before start tag.                                                                                                                                                                                                                                                | Setup instructions in Sticky Note4.                                                                               |
| Workflow author: Özgür Karateke. Support and consulting available via LinkedIn.                                                                                                                                                                                                                                                                                               | LinkedIn link in Sticky Note4.                                                                                     |
| Sample template and Apps Script source code links provided for reference.                                                                                                                                                                                                                                                                                                     | [Simple sample template → Template Link](https://www.notion.so/Simple-sample-template-Template-Link-22b3f8a1e57f8070beacd034ba6f557f?pvs=21)  
[Apps Script source code → Notion Link](https://www.notion.so/Apps-Script-source-code-Notion-Link-22b3f8a1e57f8015a280d90de16c031f?pvs=21) |

---

**Disclaimer:** This workflow is created with n8n and strictly complies with content policies. It processes only legal and publicly available data.