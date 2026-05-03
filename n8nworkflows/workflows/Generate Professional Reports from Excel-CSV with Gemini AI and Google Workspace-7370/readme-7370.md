Generate Professional Reports from Excel/CSV with Gemini AI and Google Workspace

https://n8nworkflows.xyz/workflows/generate-professional-reports-from-excel-csv-with-gemini-ai-and-google-workspace-7370


# Generate Professional Reports from Excel/CSV with Gemini AI and Google Workspace

---

### 1. Workflow Overview

This workflow automates the generation of professional analytical reports from uploaded Excel (xlsx, xls) or CSV files using Google Gemini AI and Google Workspace. It is designed primarily for public sector decision-makers needing structured, high-quality reports based on data files. The workflow integrates intelligent data extraction, AI-driven analysis, and automated document creation and distribution.

Logical blocks:

- **1.1 Input Reception and File Handling:** Receives files via webhook or form submission, identifies file types, and extracts data.
- **1.2 Data Aggregation and Preparation:** Aggregates extracted data into a single JSON string for AI processing.
- **1.3 AI Knowledge Base Setup:** Embeds data into an in-memory vector store for retrieval-based AI querying.
- **1.4 AI Analytical Processing:** Runs multi-step AI analysis using LangChain agents and Google Gemini models to generate a structured JSON report.
- **1.5 AI JSON Parsing and Validation:** Cleans and parses AI JSON output to ensure validity and readiness for downstream processing.
- **1.6 Report Generation & Formatting:** Transforms JSON analysis into a formatted Google Docs report, ensuring professional styles and structure.
- **1.7 Automated Document Distribution:** Creates or updates Google Docs, generates download links, and sends notification emails with the report link.
- **1.8 Response and Webhook Reply:** Responds to initial webhook call with the final report link or error message.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and File Handling

**Overview:**  
This block handles incoming HTTP POST requests or form submissions containing files (xlsx, xls, csv, pdf). It extracts the file, determines its type, and prepares it for further processing.

**Nodes Involved:**  
- Webhook  
- On form submission (disabled)  
- Code (filename extraction)  
- Switch (file type routing)  
- Extract from File  
- Call Transform pdf data to markdown (for PDF files)

**Node Details:**  

- **Webhook**  
  - Type: n8n Webhook  
  - Role: Entry point receiving POST requests at `/generer-rapport`  
  - Config: Allows all origins (`*`), ignores bots, method POST, uses response node mode  
  - Inputs: External HTTP POST with file upload  
  - Outputs: Connected to Extract from File node  
  - Failures: Invalid or missing file, HTTP errors

- **On form submission** (disabled)  
  - Type: Form Trigger  
  - Role: Alternative file input via form (currently disabled)  
  - Config: Expects a required file field named "file"  
  - Failures: Disabled, so no active effect

- **Code**  
  - Type: Code node (JavaScript)  
  - Role: Extracts filename from binary data in form submission (legacy or alternative input)  
  - Key expression: Reads binary data keys, extracts filename property safely  
  - Inputs: From On form submission node  
  - Outputs: JSON with filename  
  - Failures: Missing binary data, no filename property

- **Switch**  
  - Type: Switch node  
  - Role: Routes flow based on file extension extracted from filename  
  - Conditions: Routes to outputs named pdf, xlsx, docx, pptx if file extension matches  
  - Inputs: From Code node  
  - Outputs:  
    - pdf → Call Transform pdf data to markdown  
    - xlsx → Extract from File  
    - docx → (no downstream node connected)  
    - pptx → (no downstream node connected)  
  - Failures: File extension missing or unsupported

- **Extract from File**  
  - Type: Extract from File node  
  - Role: Extracts data from xlsx files in binary property "file"  
  - Config: Operation set to "xlsx"  
  - Inputs: From Webhook node (POSTed file)  
  - Outputs: Extracted tabular data items  
  - Failures: Corrupt or unsupported files, extraction errors

- **Call Transform pdf data to markdown**  
  - Type: Execute Workflow (Sub-workflow)  
  - Role: Handles PDF files, converts them to markdown for further processing  
  - Inputs: From Switch node (pdf output)  
  - Workflow ID: `PAB8VMzM4HcZLqrm` (external workflow)  
  - Failures: Sub-workflow errors, PDF parsing errors

---

#### 1.2 Data Aggregation and Preparation

**Overview:**  
Aggregates all extracted data lines into a single JSON string to be used as AI input.

**Nodes Involved:**  
- Aggregate (aggregateAllItemData)  
- Aggregate Data (Code node)  
- Set data for llm

**Node Details:**  

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines all extracted items into one dataset  
  - Config: Mode "aggregateAllItemData" to merge all item data into one  
  - Inputs: From Extract from File or pdf-to-markdown sub-workflow  
  - Outputs: Single aggregated dataset

- **Aggregate Data**  
  - Type: Code  
  - Role: Converts aggregated data array into a JSON-stringified string under key "data"  
  - Key code: Maps all input items `.json`, creates single output `{ data: JSON.stringify(jsonData) }`  
  - Inputs: From Aggregate node  
  - Outputs: Single item with stringified JSON data

- **Set data for llm**  
  - Type: Set  
  - Role: Assigns the aggregated JSON string to field "data" for AI agent input  
  - Inputs: From Aggregate Data  
  - Outputs: To AI Agent node

---

#### 1.3 AI Knowledge Base Setup

**Overview:**  
Embeds the data into an in-memory vector store for enhanced AI retrieval during analysis.

**Nodes Involved:**  
- Default Data Loader  
- Embeddings Google Gemini  
- Insert Data to Store

**Node Details:**  

- **Default Data Loader**  
  - Type: LangChain Document Default Data Loader  
  - Role: Loads data for embedding, expects binary data type  
  - Inputs: Not explicitly connected in main flow but part of vector store insertion  
  - Failures: Invalid data format

- **Embeddings Google Gemini**  
  - Type: LangChain Embeddings with Google Gemini API  
  - Role: Generates vector embeddings of the data  
  - Credentials: Google Palm API (Google Gemini)  
  - Inputs: From Default Data Loader  
  - Outputs: To Insert Data to Store

- **Insert Data to Store**  
  - Type: Vector Store In-Memory (LangChain)  
  - Role: Inserts embeddings into vector store under key "vector_store_key"  
  - Inputs: From Embeddings Google Gemini  
  - Outputs: To Query Data Tool1 for retrieval

---

#### 1.4 AI Analytical Processing

**Overview:**  
Executes a multi-stage AI analysis of the data using LangChain agents, Google Gemini chat models, and custom tools to generate a structured JSON report.

**Nodes Involved:**  
- AI Agent (LangChain agent)  
- Query Data Tool1 (vector store retrieval tool)  
- Think (LangChain toolThink)  
- Google Gemini Chat Model  
- AI Agent1 (LangChain agent for report generation)  
- Think1 (LangChain toolThink)  
- Google Gemini Chat Model2  
- AI Agent Tool1 (LangChain agent tool for document creation and sending)

**Node Details:**  

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Processes extracted data JSON, orchestrates retrieval and "Think" analysis to produce JSON report draft  
  - Prompt: Instructions to generate JSON format report with structured analysis, tailored for public service data  
  - Inputs: Aggregated JSON data from "Set data for llm"  
  - Outputs: JSON report draft string to Edit Fields node

- **Query Data Tool1**  
  - Type: Vector Store In-Memory (retrieve-as-tool)  
  - Role: Provides knowledge base retrieval for AI Agent using embedded data  
  - Inputs: From Embeddings Google Gemini2  
  - Outputs: To AI Agent as tool

- **Think**  
  - Type: LangChain ToolThink  
  - Role: Performs quick analysis of data structure and planning before final report generation  
  - Inputs: Used by AI Agent during processing

- **Google Gemini Chat Model**  
  - Type: LangChain LM Chat with Google Gemini API  
  - Role: Language model used by AI Agent for generation  
  - Credentials: Google Palm API  
  - Config: Max output tokens set very high (1,000,000) for large reports

- **AI Agent1**  
  - Type: LangChain Agent  
  - Role: Processes cleaned JSON from AI Agent output to convert JSON report into Google Docs formatted content and trigger sending  
  - Prompt: Instructions for JSON to Google Docs transformation and automated sending with strict format rules  
  - Inputs: From Parse and Clean AI Response node  
  - Outputs: To If node checking output presence

- **Think1**  
  - Type: LangChain ToolThink  
  - Role: Performs quick planning and validation for AI Agent1 report generation phase  
  - Inputs: Used by AI Agent1

- **Google Gemini Chat Model2**  
  - Type: LangChain LM Chat Google Gemini Model  
  - Role: Language model used by AI Agent1 and AI Agent Tool1 for generating formatted report and sending  
  - Credentials: Google Palm API  
  - Config: Model "models/gemini-2.5-pro", max tokens 10,000

- **AI Agent Tool1**  
  - Type: LangChain Agent Tool  
  - Role: Final agent that formats text for Google Docs, creates document, sends email, and generates download link  
  - Inputs: From AI Agent1, Google Docs and Gmail nodes triggered inside  
  - Prompt: Strict instructions for report formatting, styling, and automatic delivery  
  - Outputs: Final download link

---

#### 1.5 AI JSON Parsing and Validation

**Overview:**  
Cleans the raw AI JSON string output, extracts valid JSON object, and throws errors if parsing fails.

**Nodes Involved:**  
- Parse and Clean AI Response (Code node)  
- If1 (check non-empty output)  
- Wait2 (delay)  
- Set data for llm (re-trigger AI Agent with cleaned data)

**Node Details:**  

- **Parse and Clean AI Response**  
  - Type: Code node  
  - Role:  
    - Removes markdown code blocks (```json ...```), trims extraneous text  
    - Extracts JSON between first and last curly braces  
    - Parses JSON, returns as object for downstream nodes  
  - Inputs: Raw AI Agent output string  
  - Outputs: JSON object report or error thrown if invalid  
  - Failures: Empty response, invalid JSON, parsing errors

- **If1**  
  - Type: If node  
  - Role: Checks if output field is non-empty to decide continuation  
  - Inputs: From Edit Fields output  
  - Outputs: True → Parse and Clean AI Response, False → error branch

- **Wait2**  
  - Type: Wait node  
  - Role: Adds minor delay before re-processing cleaned data  
  - Inputs: From Parse and Clean AI Response  
  - Outputs: To Set data for llm for next AI Agent iteration

- **Set data for llm**  
  - Role: Re-assign cleaned and parsed JSON data as string for AI Agent input  
  - Inputs: From Wait2  
  - Outputs: To AI Agent node (loop)

---

#### 1.6 Report Generation & Formatting

**Overview:**  
Transforms the validated JSON report into a professionally formatted Google Docs document with proper headings, tables, and styles.

**Nodes Involved:**  
- Edit Fields (set output for further processing)  
- AI Agent1  
- Create a document in Google Docs  
- Update a document in Google Docs  
- Get a document in Google Docs  
- AI Agent Tool1  
- Think1  
- Google Gemini Chat Model2

**Node Details:**  

- **Edit Fields**  
  - Type: Set node  
  - Role: Transfers AI Agent output string to output field  
  - Inputs: From AI Agent

- **AI Agent1**  
  - Type: LangChain Agent  
  - Role: Converts JSON report to Google Docs textual content and commands for formatting  
  - Inputs: Cleaned JSON report from Parse and Clean AI Response  
  - Outputs: To If node for output presence check

- **Create a document in Google Docs**  
  - Type: Google Docs Tool  
  - Role: Creates new Google Doc report with title from AI  
  - Credentials: Google Docs OAuth2  
  - Inputs: From AI Agent Tool1 (indirect)

- **Update a document in Google Docs**  
  - Type: Google Docs Tool  
  - Role: Applies formatting updates to Google Doc (headings, tables, styles)  
  - Credentials: Google Docs OAuth2

- **Get a document in Google Docs**  
  - Type: Google Docs Tool  
  - Role: Retrieves document details or URL for download link generation  
  - Credentials: Google Docs OAuth2

---

#### 1.7 Automated Document Distribution

**Overview:**  
Automatically sends the generated report link via Gmail and optionally creates a Google Slides presentation.

**Nodes Involved:**  
- Send a message in Gmail  
- Create a presentation in Google Slides  
- Replace text in a presentation in Google Slides  
- AI Agent Tool1

**Node Details:**  

- **Send a message in Gmail**  
  - Type: Gmail Tool  
  - Role: Sends email with report link and message generated by AI  
  - Credentials: Gmail OAuth2  
  - Inputs: From AI Agent Tool1 (email fields from AI)

- **Create a presentation in Google Slides**  
  - Type: Google Slides Tool  
  - Role: Optionally creates a slides presentation from AI-generated title  
  - Credentials: Google Slides OAuth2

- **Replace text in a presentation in Google Slides**  
  - Type: Google Slides Tool  
  - Role: Replaces placeholder text in slides with AI-generated content  
  - Credentials: Google Slides OAuth2

- **AI Agent Tool1**  
  - Coordinates sending emails, updating documents, and generating links

---

#### 1.8 Response and Webhook Reply

**Overview:**  
Final step returns response to the initial webhook with either the generated report link or error message.

**Nodes Involved:**  
- If (check for empty output)  
- Wait (delay)  
- Edit Fields (prepare output)  
- Respond to Webhook

**Node Details:**  

- **If**  
  - Checks if output field is empty or not from AI Agent1 results  
  - Routes to Wait node (if empty) or Respond to Webhook (if not)

- **Wait**  
  - Adds delay before retry or error handling

- **Edit Fields**  
  - Prepares final output field with report download link

- **Respond to Webhook**  
  - Sends HTTP response with text body containing report link or error  
  - Inputs: From If node output

---

### 3. Summary Table

| Node Name                      | Node Type                             | Functional Role                                  | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                       |
|--------------------------------|-------------------------------------|-------------------------------------------------|------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| Webhook                        | n8n-nodes-base.webhook               | Receives file upload POST requests               | External HTTP POST            | Extract from File               |                                                                                                 |
| On form submission             | n8n-nodes-base.formTrigger (disabled)| Alternative file input via form (disabled)       |                              | Code                           |                                                                                                 |
| Code                          | n8n-nodes-base.code                  | Extract filename from binary form data           | On form submission            | Switch                        |                                                                                                 |
| Switch                        | n8n-nodes-base.switch                | Routes flow by file extension                     | Code                         | Extract from File, Call Transform pdf data to markdown |                                                                                                 |
| Extract from File              | n8n-nodes-base.extractFromFile      | Extracts tabular data from Excel files            | Webhook                      | Aggregate                      |                                                                                                 |
| Call Transform pdf data to markdown | n8n-nodes-base.executeWorkflow      | Converts PDF files to markdown                     | Switch (pdf output)           | Aggregate                      |                                                                                                 |
| Aggregate                     | n8n-nodes-base.aggregate             | Aggregates all data items into one collection     | Extract from File, Call Transform pdf data to markdown | Aggregate Data                |                                                                                                 |
| Aggregate Data                | n8n-nodes-base.code                  | Converts aggregated data array to JSON string     | Aggregate                    | Set data for llm               |                                                                                                 |
| Set data for llm              | n8n-nodes-base.set                   | Prepares JSON string data for AI agent input      | Aggregate Data               | AI Agent                      |                                                                                                 |
| Default Data Loader           | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads data for vector store embedding              |                              | Insert Data to Store           |                                                                                                 |
| Embeddings Google Gemini      | @n8n/n8n-nodes-langchain.embeddingsGoogleGemini | Generates vector embeddings                         | Default Data Loader          | Insert Data to Store           |                                                                                                 |
| Insert Data to Store          | @n8n/n8n-nodes-langchain.vectorStoreInMemory | Inserts embeddings into in-memory vector store     | Embeddings Google Gemini     | Query Data Tool1               |                                                                                                 |
| Query Data Tool1              | @n8n/n8n-nodes-langchain.vectorStoreInMemory | Retrieves data from vector store as AI tool        | Embeddings Google Gemini2    | AI Agent                      |                                                                                                 |
| AI Agent                     | @n8n/n8n-nodes-langchain.agent       | Processes data JSON and generates JSON report     | Set data for llm, Query Data Tool1, Think, Google Gemini Chat Model | Edit Fields                  |                                                                                                 |
| Think                        | @n8n/n8n-nodes-langchain.toolThink    | Performs quick analysis/planning for AI            | AI Agent                    | AI Agent                      |                                                                                                 |
| Google Gemini Chat Model     | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Language model for AI Agent                         | AI Agent                    | AI Agent                      |                                                                                                 |
| Edit Fields                  | n8n-nodes-base.set                   | Assigns AI Agent output to output field            | AI Agent                    | If1                          |                                                                                                 |
| If1                         | n8n-nodes-base.if                    | Checks for non-empty AI output                      | Edit Fields                 | Parse and Clean AI Response, Wait2 |                                                                                                 |
| Parse and Clean AI Response  | n8n-nodes-base.code                  | Cleans and parses AI JSON output                    | If1                         | AI Agent1                    | Throws error if JSON invalid                                                                    |
| Wait2                       | n8n-nodes-base.wait                  | Delay before re-processing cleaned data             | Parse and Clean AI Response | Set data for llm               |                                                                                                 |
| AI Agent1                   | @n8n/n8n-nodes-langchain.agent       | Converts JSON report to Google Docs format and triggers sending | Parse and Clean AI Response | If                          |                                                                                                 |
| Think1                      | @n8n/n8n-nodes-langchain.toolThink    | Planning/validation for AI Agent1                    | AI Agent1                   | AI Agent1                    |                                                                                                 |
| Google Gemini Chat Model2   | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LM for AI Agent1 and AI Agent Tool1                  | AI Agent1, AI Agent Tool1    | AI Agent1, AI Agent Tool1      |                                                                                                 |
| AI Agent Tool1              | @n8n/n8n-nodes-langchain.agentTool   | Formats report, creates Google Docs, sends email    | AI Agent1                   | Send a message in Gmail, Create a document in Google Docs, Update a document in Google Docs, Get a document in Google Docs, Create a presentation in Google Slides, Replace text in a presentation in Google Slides |                                                                                                 |
| Create a document in Google Docs | n8n-nodes-base.googleDocsTool         | Creates Google Docs report document                   | AI Agent Tool1              | AI Agent Tool1                |                                                                                                 |
| Update a document in Google Docs | n8n-nodes-base.googleDocsTool         | Applies formatting updates to Google Docs document   | AI Agent Tool1              | AI Agent Tool1                |                                                                                                 |
| Get a document in Google Docs | n8n-nodes-base.googleDocsTool         | Retrieves Google Docs document details                | AI Agent Tool1              | AI Agent Tool1                |                                                                                                 |
| Send a message in Gmail     | n8n-nodes-base.gmailTool              | Sends email with report link and message              | AI Agent Tool1              | AI Agent Tool1                |                                                                                                 |
| Create a presentation in Google Slides | n8n-nodes-base.googleSlidesTool       | Optionally creates Google Slides presentation          | AI Agent Tool1              | AI Agent Tool1                |                                                                                                 |
| Replace text in a presentation in Google Slides | n8n-nodes-base.googleSlidesTool       | Replaces text in Google Slides                          | AI Agent Tool1              | AI Agent Tool1                |                                                                                                 |
| If                         | n8n-nodes-base.if                    | Checks if AI Agent1 output is empty                    | AI Agent1                   | Wait, Respond to Webhook       |                                                                                                 |
| Wait                       | n8n-nodes-base.wait                  | Delay before responding or retry                       | If                         | Edit Fields                  |                                                                                                 |
| Respond to Webhook          | n8n-nodes-base.respondToWebhook      | Sends final HTTP response with report link             | If                         |                              |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `generer-rapport`  
   - HTTP Method: POST  
   - Response Mode: responseNode  
   - Allowed Origins: `*`  
   - Option: Ignore Bots = true

2. **Create Extract from File Node**  
   - Type: Extract from File  
   - Operation: `xlsx`  
   - Binary Property Name: `file`  
   - Connect Webhook output to this node

3. **Create Switch Node**  
   - Type: Switch  
   - Condition: Check file extension from filename (expression: `{{$json["filename"].split('.').pop()}}`)  
   - Outputs: pdf, xlsx, docx, pptx  
   - Connect from Code node (see next)

4. **Create Code Node (Filename Extraction)**  
   - Extract filename from binary data (JS code provided in original workflow)  
   - Connect On form submission (optional) or Webhook files to this node

5. **Create Call Transform pdf data to markdown (Execute Workflow)**  
   - Workflow ID: `PAB8VMzM4HcZLqrm` (must import or create separately)  
   - Connect Switch pdf output to this node

6. **Create Aggregate Node**  
   - Operation: `aggregateAllItemData`  
   - Connect Extract from File and Call Transform pdf data to markdown outputs here

7. **Create Aggregate Data Code Node**  
   - JS code to stringify aggregated data array under key `data`  
   - Connect Aggregate output here

8. **Create Set Node (Set data for llm)**  
   - Assign field `data` with expression: `={{ JSON.stringify($('Extract from File').all()) }}` or from previous aggregate code node output  
   - Connect Aggregate Data output here

9. **Create Default Data Loader Node**  
   - Data Type: binary  
   - Connect as needed for embedding preparation

10. **Create Embeddings Google Gemini Node**  
    - Credentials: Google Palm API (Google Gemini)  
    - Connect Default Data Loader output here

11. **Create Insert Data to Store Node**  
    - Mode: insert  
    - Memory Key: `vector_store_key`  
    - Connect Embeddings Google Gemini output here

12. **Create Query Data Tool1 Node**  
    - Mode: retrieve-as-tool  
    - Tool Name: knowledge_base  
    - Memory Key: `vector_store_key`  
    - Tool Description: "Use this knowledge base to answer questions from the user"  
    - Connect Embeddings Google Gemini2 output here

13. **Create AI Agent Node**  
    - Type: LangChain agent  
    - Text prompt: As per detailed instructions for JSON report generation  
    - System message: Expert AI agent for public services analysis  
    - Connect Set data for llm and Query Data Tool1 nodes here  
    - Make sure Google Gemini Chat Model is connected as LM

14. **Create Think Node**  
    - Type: LangChain toolThink  
    - Connect as AI tool for AI Agent

15. **Create Google Gemini Chat Model Node**  
    - Credentials: Google Palm API  
    - Max Output Tokens: 1,000,000  
    - Connect to AI Agent as languageModel

16. **Create Edit Fields Node**  
    - Assign field `output` with AI Agent output string  
    - Connect AI Agent main output here

17. **Create If1 Condition Node**  
    - Check if `output` field is not empty  
    - Connect Edit Fields output here

18. **Create Parse and Clean AI Response Code Node**  
    - JS code to clean markdown, extract valid JSON, parse it, throw error if invalid  
    - Connect If1 true output here

19. **Create Wait2 Node**  
    - Delay node with default wait time  
    - Connect Parse and Clean AI Response output here

20. **Connect Wait2 output back to Set data for llm node**  
    - To re-trigger AI Agent with cleaned data (loop)

21. **Create AI Agent1 Node**  
    - LangChain agent for JSON to Google Docs text transformation and sending  
    - System message includes formatting instructions and strict output rules  
    - Connect Parse and Clean AI Response output here

22. **Create Think1 Node**  
    - LangChain toolThink for AI Agent1 planning  
    - Connect as AI tool for AI Agent1

23. **Create Google Gemini Chat Model2 Node**  
    - Credentials: Google Palm API  
    - Model Name: `models/gemini-2.5-pro`  
    - Max Output Tokens: 10,000  
    - Connect as LM for AI Agent1 and AI Agent Tool1

24. **Create AI Agent Tool1 Node**  
    - LangChain agent tool for formatting report, creating Google Docs, sending email  
    - Connect AI Agent1 output here  
    - Connect Google Docs and Gmail nodes to this agent

25. **Create Google Docs Tool Nodes**  
    - Create a document in Google Docs: title from AI  
    - Update a document in Google Docs: formatting commands  
    - Get a document in Google Docs: retrieve document URL  
    - Connect these nodes in sequence from AI Agent Tool1

26. **Create Gmail Tool Node**  
    - Send a message in Gmail  
    - Recipient, subject, message from AI Agent Tool1 outputs  
    - Connect AI Agent Tool1 output here

27. **Optionally, create Google Slides Tool nodes**  
    - Create a presentation in Google Slides  
    - Replace text in a presentation  
    - Connect AI Agent Tool1 outputs here if needed

28. **Create If Node**  
    - Check if AI Agent1 output is empty  
    - True → Wait node  
    - False → Respond to Webhook node

29. **Create Wait Node**  
    - Delay before retry or error handling  
    - Connect If true output here

30. **Create Respond to Webhook Node**  
    - Responds with text body containing final report download link or error message  
    - Connect If false output here

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|----------------|
| Workflow is designed for public sector reports in sectors like water, sanitation, waste, and transport. | System messages in AI Agent nodes |
| Google Gemini (PaLM) API credentials are required for embedding and LM nodes. | Credential setup in nodes "Embeddings Google Gemini", "Google Gemini Chat Model" |
| Sub-workflow "Call Transform pdf data to markdown" must be imported separately with ID `PAB8VMzM4HcZLqrm`. | Execute Workflow node configuration |
| Output Google Docs and Gmail nodes require OAuth2 credentials properly configured for Google Workspace and Gmail accounts. | Google Docs OAuth2 and Gmail OAuth2 credentials |
| AI JSON output parsing is critical; errors in response format will throw exceptions. | Parse and Clean AI Response node code |
| The workflow uses LangChain nodes with specific prompt engineering for professional, structured output. | Agent node prompts in French and English |
| The workflow supports large document generation with high max token limits. | Google Gemini Chat Model maxOutputTokens |

---

**Disclaimer:** The content provided is extracted exclusively from an automated n8n workflow using public and legal data. It adheres strictly to content policies and contains no illegal, offensive, or protected elements.

---