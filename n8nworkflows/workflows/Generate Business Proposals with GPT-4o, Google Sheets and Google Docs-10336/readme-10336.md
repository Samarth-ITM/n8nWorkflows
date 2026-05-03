Generate Business Proposals with GPT-4o, Google Sheets and Google Docs

https://n8nworkflows.xyz/workflows/generate-business-proposals-with-gpt-4o--google-sheets-and-google-docs-10336


# Generate Business Proposals with GPT-4o, Google Sheets and Google Docs

### 1. Workflow Overview

This workflow automates the generation of professional business proposals by integrating Google Sheets, OpenAI’s GPT-4o, and Google Docs. It is designed for agencies or businesses that need to create customized client proposals at scale based on input data recorded in a Google Sheet. The workflow listens for new rows in a specified Google Sheet, processes only the latest entry, uses GPT-4o to generate detailed proposal content formatted precisely as JSON, populates a Google Docs template with this content, downloads the completed document, archives it to Google Drive, and resets the template for future use.

**Logical Blocks:**

- **1.1 Input Reception and Filtering:** Google Sheets trigger to detect new client job entries and code node to filter the latest row only.
- **1.2 AI Processing:** Uses LangChain AI nodes including memory buffer, GPT-4o model, structured JSON output parser, and an AI agent orchestrating the generation of tailored proposal content.
- **1.3 Document Population:** Populates a Google Docs template with the generated proposal fields by replacing placeholders.
- **1.4 File Management:** Downloads the updated Google Doc as a PDF, archives it to a designated Google Drive folder, and resets the template placeholders for reuse.
- **1.5 Instructional Sticky Notes:** Provide configuration and usage guidance at various steps.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Filtering

**Overview:**  
This block listens for new rows added to a Google Sheet and filters the incoming data to process only the most recent row to avoid duplicate or batch processing errors.

**Nodes Involved:**  
- Trigger: New Sheet Row  
- Filter: Latest Row Only

**Node Details:**

- **Trigger: New Sheet Row**  
  - *Type:* Google Sheets Trigger  
  - *Role:* Watches a specific Google Sheet (Sheet1) for new rows every minute.  
  - *Configuration:*  
    - Sheet Name: "Sheet1" (gid=0)  
    - Document ID: Must be replaced by user with own Google Sheet ID  
    - Polling Interval: Every minute  
  - *Input/Output:* No input; outputs all new rows detected.  
  - *Potential Failures:* Authentication errors, network issues, invalid Sheet ID, rate limits.

- **Filter: Latest Row Only**  
  - *Type:* Code (JavaScript) Node  
  - *Role:* Selects only the last (most recent) row from the batch of rows triggered by the Google Sheets node.  
  - *Key Expression:* Returns an array with only the last item from the input array.  
  - *Input:* All new rows from Google Sheets trigger.  
  - *Output:* Single-item array containing the latest row JSON.  
  - *Edge Cases:* Empty input array (returns empty array gracefully), multiple rows in input.

---

#### 1.2 AI Processing

**Overview:**  
This block uses LangChain nodes to generate a complete, structured business proposal in JSON format. It maintains client-specific context, uses GPT-4o with a system prompt tuned for professional proposal writing, and enforces output structure via a JSON parser.

**Nodes Involved:**  
- Memory: Client Context  
- Model: GPT-4o  
- Parser: JSON Output  
- AI Agent: Generate Proposal

**Node Details:**

- **Memory: Client Context**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Maintains conversation or generation context per client to ensure consistency across multiple runs if needed.  
  - *Configuration:*  
    - Session Key: Dynamic key based on clientName from the trigger data (e.g., "proposal_ClientName").  
  - *Input:* Receives data from the Google Sheets trigger filtered node.  
  - *Output:* Provides context memory to the AI Agent node.  
  - *Edge Cases:* Missing clientName field, session key collisions.

- **Model: GPT-4o**  
  - *Type:* LangChain Chat OpenAI Language Model  
  - *Role:* Generates natural language content based on the prompt and job description.  
  - *Configuration:*  
    - Model: gpt-4o  
    - Temperature: 0.7 (moderate creativity)  
  - *Input:* Prompt text from AI Agent node.  
  - *Output:* Raw generated text.  
  - *Potential Failures:* API key/auth errors, rate limits, model unavailability, timeout.

- **Parser: JSON Output**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses the raw AI output to enforce JSON format matching a strict schema, ensuring downstream steps receive predictable structured data.  
  - *Configuration:*  
    - JSON Schema example defining all expected fields (executive_summary, scope_of_work, month1_focus, month1_cost, ..., conclusion).  
  - *Input:* AI Agent output text.  
  - *Output:* Parsed JSON object with proposal fields.  
  - *Edge Cases:* AI output not matching schema, parser failures, malformed JSON.

- **AI Agent: Generate Proposal**  
  - *Type:* LangChain Agent  
  - *Role:* Orchestrates prompt construction and calls to the language model, using memory and parser nodes.  
  - *Configuration:*  
    - System Message: Defines expert proposal writer persona and detailed formatting & content rules (e.g., bullet points, paragraph breaks, cost formatting).  
    - Prompt: Injects clientName and jobDescription dynamically from input JSON.  
    - Output Parser: Attached to enforce JSON output.  
    - Memory: Uses client-specific memory buffer.  
  - *Input:* Latest Google Sheet row JSON.  
  - *Output:* Structured JSON proposal content.  
  - *Edge Cases:* Incorrect or incomplete input data, AI generation errors, prompt injection risks.

---

#### 1.3 Document Population

**Overview:**  
This block replaces placeholders in a Google Docs template with the generated proposal content fields, producing a fully populated professional proposal document.

**Nodes Involved:**  
- Populate: Template Document

**Node Details:**

- **Populate: Template Document**  
  - *Type:* Google Docs Node  
  - *Role:* Updates a Google Docs template by replacing predefined placeholders with proposal data fields.  
  - *Configuration:*  
    - Operation: Update document  
    - Document URL: User must replace with their Google Docs template ID  
    - Actions: Multiple "replaceAll" actions mapping placeholders (e.g., {{executive_summary}}) to corresponding JSON output fields from AI Agent.  
  - *Input:* Proposal JSON from AI Agent node.  
  - *Output:* Updated Google Doc metadata including document ID.  
  - *Potential Failures:* Invalid document ID, permission errors, placeholder mismatches, Google Docs API errors.

---

#### 1.4 File Management

**Overview:**  
This block downloads the populated proposal document, archives it to a specified Google Drive folder with client and date-based naming, and resets the Google Docs template placeholders to prepare for the next generation cycle.

**Nodes Involved:**  
- Download: Completed Proposal  
- Archive: Save to Drive  
- Reset: Template Placeholders

**Node Details:**

- **Download: Completed Proposal**  
  - *Type:* Google Drive Node  
  - *Role:* Downloads the populated Google Docs file as a PDF or original format (default).  
  - *Configuration:*  
    - Operation: Download file by ID (passed dynamically from Populate node's output).  
  - *Input:* Document ID from Populate node.  
  - *Output:* Binary data of the downloaded file.  
  - *Edge Cases:* File not found, permission denied, network errors.

- **Archive: Save to Drive**  
  - *Type:* Google Drive Node  
  - *Role:* Uploads the downloaded proposal file into a designated Drive folder for archival.  
  - *Configuration:*  
    - Folder ID: User must replace with their desired Drive folder ID  
    - File Name: Dynamic, composed of clientName and current date (e.g., "ClientName_Proposal_2024-06-01")  
    - Drive ID: "My Drive" (default)  
  - *Input:* Binary data from Download node and clientName from trigger.  
  - *Output:* Metadata of uploaded file.  
  - *Potential Failures:* Upload errors, folder permission issues.

- **Reset: Template Placeholders**  
  - *Type:* Google Docs Node  
  - *Role:* Reverts the placeholder replacements in the Google Docs template back to their original placeholder strings, so the template is clean for next use.  
  - *Configuration:*  
    - Operation: Update document  
    - Document URL: Same template document ID as Populate node  
    - Actions: Replace each filled content field back to its placeholder string (e.g., executive_summary text → "{{executive_summary}}")  
  - *Input:* Uses AI Agent output JSON to know what text to replace back.  
  - *Output:* Updated template document ready for next generation.  
  - *Edge Cases:* Partial replacements, concurrent usage conflicts.

---

#### 1.5 Instructional Sticky Notes

**Overview:**  
These nodes provide inline documentation and usage guidance for each key step, aiding users to configure and understand the workflow.

**Nodes Involved:**  
- Workflow Overview  
- Trigger Instructions  
- Data Filter Guide  
- AI Components Guide  
- Document Setup Guide  
- File Operations Guide

**Node Details:**  
Each sticky note contains detailed instructions, setup tips, expected input/output formats, and reminders for critical configuration points such as replacing IDs and credentials.

---

### 3. Summary Table

| Node Name                   | Node Type                                    | Functional Role                        | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                       |
|-----------------------------|----------------------------------------------|-------------------------------------|-------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow Overview           | Sticky Note                                  | Documentation overview               |                               |                               | ## 🎯 AI-Powered Proposal Generator Workflow Purpose and flow description                        |
| Trigger Instructions        | Sticky Note                                  | Trigger setup instructions           |                               |                               | ## 📥 Step 1: Trigger Setup Google Sheets columns and polling instructions                       |
| Data Filter Guide           | Sticky Note                                  | Data filtering instructions          |                               |                               | ## 🔄 Step 2: Data Processing Explanation why last row only is processed                         |
| AI Components Guide         | Sticky Note                                  | AI generation instructions           |                               |                               | ## 🤖 Step 3: AI Generation components and output fields                                        |
| Document Setup Guide        | Sticky Note                                  | Document template setup instructions |                               |                               | ## 📝 Step 4: Document Population placeholders and template setup                               |
| File Operations Guide       | Sticky Note                                  | File management instructions         |                               |                               | ## 💾 Step 5: File Management steps: download, upload, reset                                   |
| Trigger: New Sheet Row      | Google Sheets Trigger                        | Input event listener                  |                               | Filter: Latest Row Only        | Replace YOUR_GOOGLE_SHEET_ID with your actual Google Sheet ID                                  |
| Filter: Latest Row Only     | Code                                         | Filters latest row only               | Trigger: New Sheet Row         | AI Agent: Generate Proposal    | Filters only the most recently appended row to avoid duplicates                                |
| Memory: Client Context      | LangChain Memory Buffer Window               | Maintains client-specific context    | Filter: Latest Row Only        | AI Agent: Generate Proposal    |                                                                                                 |
| Model: GPT-4o              | LangChain Chat OpenAI Model                   | Language model for content generation| AI Agent: Generate Proposal    | AI Agent: Generate Proposal    | Uses GPT-4o with temperature 0.7 for balanced creativity                                       |
| Parser: JSON Output         | LangChain Output Parser Structured            | Parses AI output into structured JSON| AI Agent: Generate Proposal    | AI Agent: Generate Proposal    | Enforces strict JSON schema for proposal fields                                              |
| AI Agent: Generate Proposal | LangChain Agent                              | Orchestrates AI prompt, memory, parsing | Filter: Latest Row Only, Memory, Model, Parser | Populate: Template Document | Complex prompt with detailed formatting and content instructions                             |
| Populate: Template Document | Google Docs                                  | Replaces placeholders with AI output| AI Agent: Generate Proposal    | Download: Completed Proposal   | Replace YOUR_TEMPLATE_DOCUMENT_ID with your Google Docs template ID                           |
| Download: Completed Proposal| Google Drive                                 | Downloads populated proposal document| Populate: Template Document    | Archive: Save to Drive         |                                                                                                 |
| Archive: Save to Drive      | Google Drive                                 | Uploads finalized proposal to Drive | Download: Completed Proposal   | Reset: Template Placeholders   | Replace YOUR_OUTPUT_FOLDER_ID with your Google Drive folder ID                                |
| Reset: Template Placeholders| Google Docs                                  | Resets template placeholders to original | Archive: Save to Drive         |                               | Ensures the template is clean for next use                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node:**  
   - Node type: Google Sheets Trigger  
   - Set Document ID to your Google Sheet ID with columns `clientName` in column A and `jobDescription` in column B.  
   - Set Sheet Name to "Sheet1" or your actual sheet name.  
   - Polling: Configure to poll every minute.

2. **Add Code Node to Filter Latest Row Only:**  
   - Node type: Code (JavaScript)  
   - Paste the code that returns only the last row from the input array:  
     ```js
     const items = $input.all();
     if (!items || items.length === 0) {
       return [];
     }
     const lastItem = items[items.length - 1];
     return [{ json: lastItem.json }];
     ```
   - Connect trigger output to this node input.

3. **Add LangChain Memory Buffer Window Node:**  
   - Node type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Set `sessionKey` to expression: `="proposal_" + $('Trigger: New Sheet Row').item.json.clientName`  
   - Connect from Filter node.

4. **Add LangChain Chat OpenAI Model Node:**  
   - Node type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Set model to "gpt-4o"  
   - Set temperature to 0.7  
   - Configure OpenAI credentials.  
   - Connect from AI Agent node (see below).

5. **Add LangChain Output Parser Structured Node:**  
   - Node type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Paste JSON schema example defining all fields: executive_summary, scope_of_work, month1_focus, month1_cost, ..., conclusion.  
   - Connect from AI Agent node.

6. **Add LangChain Agent Node:**  
   - Node type: `@n8n/n8n-nodes-langchain.agent`  
   - Set `text` prompt with dynamic expressions for `jobDescription` and `clientName` from latest row JSON.  
   - Configure detailed system message with professional proposal style instructions (as per original prompt).  
   - Link AI model, memory buffer, and parser nodes appropriately in node options.  
   - Connect from Filter node and Memory node inputs. Connect Model and Parser nodes as subcomponents.  
   - Output connects to Google Docs node.

7. **Add Google Docs Node to Populate Template:**  
   - Node type: Google Docs  
   - Operation: Update document  
   - Document URL: Insert your Google Docs template document ID.  
   - Add multiple replaceAll actions for each placeholder (`{{executive_summary}}`, `{{scope_of_work}}`, etc.) mapped to corresponding AI Agent output JSON fields.  
   - Connect from AI Agent node.

8. **Add Google Drive Node to Download Document:**  
   - Node type: Google Drive  
   - Operation: Download  
   - File ID: Expression to get documentId from Populate node output.  
   - Connect from Populate node.

9. **Add Google Drive Node to Archive File:**  
   - Node type: Google Drive  
   - Operation: Upload (default)  
   - Folder ID: Your destination Google Drive folder ID for proposals  
   - File name: Expression combining clientName and current date (e.g., `={{ $('Trigger: New Sheet Row').item.json.clientName + '_Proposal_' + $now.format('yyyy-MM-dd') }}`)  
   - Connect from Download node.

10. **Add Google Docs Node to Reset Template Placeholders:**  
    - Node type: Google Docs  
    - Operation: Update document  
    - Document URL: Same template document ID as step 7.  
    - Replace all proposal content fields back to their original placeholders using "replaceAll" actions.  
    - Connect from Archive node.

11. **Add Sticky Notes for Documentation (Optional):**  
    - Add multiple sticky note nodes to provide user guidance and instructions at each step for clarity.

12. **Credential Setup:**  
    - OpenAI API key for LangChain nodes.  
    - Google Sheets OAuth2 credential with read access for Sheets trigger.  
    - Google Docs and Drive OAuth2 credentials with appropriate scopes for document editing, downloading, and file uploads.

13. **Activate Workflow and Test:**  
    - Replace all placeholder IDs with actual user-specific IDs.  
    - Test by adding a new row to the Google Sheet and confirm the final proposal document is generated, archived, and the template resets correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow is optimized for agencies generating client proposals at scale using dynamic AI generation. | Workflow Overview Sticky Note                                                                           |
| Replace all placeholder IDs (Google Sheet ID, Google Docs Template ID, Google Drive Folder ID) with yours.| Trigger Instructions, Document Setup Guide, File Operations Guide Sticky Notes                         |
| GPT-4o model is used with temperature 0.7 for balanced creativity and professionalism in proposals.       | AI Components Guide Sticky Note                                                                         |
| Proposal content strictly follows JSON schema for easy template mapping and predictable downstream use.  | AI Agent prompt and Parser node details                                                                |
| Template placeholders must exactly match those in Google Docs template and the node replacement actions. | Document Setup Guide Sticky Note                                                                        |
| Files are archived with client name and date for easy retrieval and versioning.                          | File Operations Guide Sticky Note                                                                       |
| Resetting the template placeholders is critical to avoid data leakage between runs.                      | File Operations Guide Sticky Note                                                                       |
| For detailed styling and prompt customization, modify the system message in the AI Agent node.           | AI Components Guide Sticky Note                                                                         |
| Workflow polling interval is every minute; adjust as needed for your use case and API limits.            | Trigger Instructions Sticky Note                                                                        |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.