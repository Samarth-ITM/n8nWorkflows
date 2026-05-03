Generate AI Newsletter Drafts from Google Sheets to Notion with GPT-5-mini

https://n8nworkflows.xyz/workflows/generate-ai-newsletter-drafts-from-google-sheets-to-notion-with-gpt-5-mini-7848


# Generate AI Newsletter Drafts from Google Sheets to Notion with GPT-5-mini

### 1. Workflow Overview

This workflow automates the generation of AI-driven newsletter drafts starting from data in a Google Sheets spreadsheet, then publishes these drafts as pages in a Notion database. The drafts are created using an advanced GPT-5-mini model configured with specific storytelling and stylistic guidelines to produce narrative-style newsletters inspired by Ali Abdaal’s writing voice.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Filtering**: Trigger and retrieve rows from Google Sheets that are marked as pending for newsletter generation.
- **1.2 Batch Processing Loop**: Handle each row individually by splitting inputs into batches for sequential processing.
- **1.3 AI Newsletter Draft Generation**: Use a LangChain AI agent powered by GPT-5-mini to create a polished newsletter draft based on the title and description from each sheet row.
- **1.4 Notion Page Creation and Content Upload**: Create a new page in Notion for each newsletter and upload the AI-generated content in chunks to avoid API size limits.
- **1.5 Status Update in Google Sheets**: Mark the processed rows as done in the Google Sheets to avoid duplicate processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

**Overview:**  
This block listens for manual workflow execution and retrieves all rows from a specific Google Sheets tab that have the status "Pending" in the "N8n Status" column. It serves as the entry point to fetch newsletter topics awaiting processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get row(s) in sheet (Google Sheets)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually  
  - Configuration: No parameters needed, triggers workflow execution on demand  
  - Inputs: None  
  - Outputs: Triggers "Get row(s) in sheet" node  
  - Edge cases: None significant beyond manual initiation  

- **Get row(s) in sheet**  
  - Type: Google Sheets (Read Rows)  
  - Role: Retrieves rows from Google Sheets where "N8n Status" = "Pending"  
  - Configuration:  
    - Document ID set to specific Google Sheets file ("Blog Automation")  
    - Sheet ID set to "newsletter" tab  
    - Filter applied: Only rows with "Pending" in "N8n Status" column are fetched  
  - Inputs: Trigger from manual node  
  - Outputs: Data passed to "Loop Over Items" node  
  - Possible failures: Google auth errors, sheet access issues, empty results if no pending rows  

---

#### 2.2 Batch Processing Loop

**Overview:**  
Splits the retrieved rows into batches and processes them sequentially, enabling controlled and manageable iteration through each newsletter item.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each newsletter row individually to allow sequential AI generation and Notion operations  
  - Configuration: Default batching without customized batch size (implied batch size = 1)  
  - Inputs: Rows with pending status from Google Sheets  
  - Outputs: Sends each row to AI Agent node for newsletter creation  
  - Edge cases: Batch size too large could cause rate limiting; empty batches if no rows fetched  

---

#### 2.3 AI Newsletter Draft Generation

**Overview:**  
This block uses a LangChain AI agent configured with detailed system instructions to generate a fully structured, narrative newsletter draft for each input row using the GPT-5-mini model.

**Nodes Involved:**  
- AI Agent (LangChain agent)  
- OpenAI Chat Model (GPT-5-mini)

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Implements the newsletter writing logic by applying a detailed system prompt and user inputs to generate a draft  
  - Configuration:  
    - System instructions define writing style, structure, tone, and output format based on Ali Abdaal's newsletter style  
    - Inputs use expressions referencing current batch item's "Newsletter Title" and "About the Newsletter" fields  
    - Connects to OpenAI Chat Model node as language model backend  
  - Inputs: Newsletter data from "Loop Over Items" node  
  - Outputs: Full newsletter draft JSON to "Create Page" node  
  - Edge cases: Prompt parsing or generation failures, API timeouts, rate limiting, malformed inputs  

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Executes the GPT-5-mini chat completion request based on AI Agent instructions  
  - Configuration:  
    - Model set explicitly to "gpt-5-mini"  
    - Uses stored OpenAI API credentials  
  - Inputs: Prompt from AI Agent node  
  - Outputs: Generated text passed back to AI Agent and downstream "Create Page"  
  - Edge cases: API errors, credential expiration, quota exceeded, network issues  

---

#### 2.4 Notion Page Creation and Content Upload

**Overview:**  
Creates a new Notion database page for each newsletter draft and uploads the AI-generated content in manageable chunks to comply with Notion’s API limits.

**Nodes Involved:**  
- Create Page (Notion)  
- Code (JavaScript)  
- UpdateNotionBlock (HTTP Request)

**Node Details:**

- **Create Page**  
  - Type: Notion Node (databasePage resource)  
  - Role: Creates a new page in a specified Notion database using the newsletter title as page title  
  - Configuration:  
    - Database ID set to "Newsletter Automation" Notion database  
    - Title property mapped from current newsletter batch item's "Newsletter Title"  
    - Uses Notion API credentials  
  - Inputs: Newsletter draft from AI Agent output  
  - Outputs: Notion page ID passed to "Code" node  
  - Edge cases: API errors, permission denied, invalid database ID  

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Converts the large JSON newsletter draft into multiple paragraph blocks, chunking to ~1800 characters each to avoid Notion API size limits  
  - Configuration:  
    - Reads AI Agent output JSON string, slices into chunks, wraps each chunk into Notion paragraph block format  
    - Returns array of children blocks for Notion API PATCH call  
  - Inputs: Notion page ID and newsletter JSON from "Create Page" node  
  - Outputs: JSON body with chunked blocks for Notion update  
  - Edge cases: JSON serialization errors, overly large content  

- **UpdateNotionBlock**  
  - Type: HTTP Request  
  - Role: Sends PATCH request to Notion API to add the chunked newsletter content as children blocks to the created page  
  - Configuration:  
    - URL dynamically constructed with the Notion page ID from "Create Page" node  
    - PATCH method with JSON body containing "children" blocks from "Code" node  
    - Uses predefined Notion API credentials  
    - On error set to continue, avoiding workflow stop if Notion update partially fails  
  - Inputs: Chunked content blocks from "Code" node  
  - Outputs: Response passed to "Update row in sheet" node  
  - Edge cases: Network errors, API rate limits, partial content upload failures  

---

#### 2.5 Status Update in Google Sheets

**Overview:**  
Marks each processed newsletter row as "Done" in Google Sheets to prevent reprocessing and updates the newsletter title for tracking.

**Nodes Involved:**  
- Update row in sheet (Google Sheets)

**Node Details:**

- **Update row in sheet**  
  - Type: Google Sheets (Update Row)  
  - Role: Updates the "N8n Status" column to "Done" and sets the "Newsletter Title" in the original spreadsheet row  
  - Configuration:  
    - Matches rows by "Newsletter Title"  
    - Updates "N8n Status" to "Done"  
    - Uses Google Sheets OAuth2 credentials  
  - Inputs: Output from "UpdateNotionBlock" node confirming Notion update  
  - Outputs: Loops back to "Loop Over Items" node for next item processing  
  - Edge cases: Update failure due to row mismatch, API errors, credential issues  

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                          | Input Node(s)               | Output Node(s)            | Sticky Note                                      |
|-------------------------|-----------------------------------|----------------------------------------|----------------------------|---------------------------|-------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Starts workflow manually                | -                          | Get row(s) in sheet       |                                                 |
| Get row(s) in sheet      | Google Sheets                     | Fetch rows with status "Pending"       | When clicking ‘Execute workflow’ | Loop Over Items          |                                                 |
| Loop Over Items          | SplitInBatches                   | Processes each row individually        | Get row(s) in sheet         | AI Agent                  |                                                 |
| AI Agent                 | LangChain Agent                   | Generates newsletter draft via GPT-5-mini | Loop Over Items             | Create Page               |                                                 |
| OpenAI Chat Model        | LangChain OpenAI Chat Model       | Executes GPT-5-mini chat completion    | AI Agent (ai_languageModel) | AI Agent                  |                                                 |
| Create Page              | Notion (databasePage)             | Creates Notion page for newsletter     | AI Agent                   | Code                      |                                                 |
| Code                     | Code (JavaScript)                 | Splits newsletter JSON into Notion blocks | Create Page                | UpdateNotionBlock         |                                                 |
| UpdateNotionBlock        | HTTP Request                     | Uploads newsletter content to Notion   | Code                       | Update row in sheet       | On error: continue workflow                      |
| Update row in sheet      | Google Sheets                    | Marks newsletter row as "Done"          | UpdateNotionBlock           | Loop Over Items           |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start workflow execution manually.

2. **Add Google Sheets Node to Read Rows**  
   - Type: Google Sheets  
   - Operation: Read Rows  
   - Configure:  
     - Select your Google Sheets document by ID.  
     - Select the worksheet/tab named "newsletter" (or your equivalent).  
     - Set filter to only get rows where "N8n Status" equals "Pending".  
   - Connect output of Manual Trigger to this node.

3. **Add SplitInBatches Node**  
   - Type: SplitInBatches  
   - Purpose: To process one row at a time.  
   - Default batch size is 1 (adjust if needed).  
   - Connect output of Google Sheets read node to this node.

4. **Add LangChain AI Agent Node**  
   - Type: LangChain Agent (Requires n8n-nodes-langchain package)  
   - Configure System Role: Use the provided detailed prompt instructing the AI to write newsletters with the specified style and structure.  
   - Use expressions to pass:  
     - `Newsletter Title` from current batch item as input.  
     - `About the Newsletter` from current batch item as input.  
   - Connect output of SplitInBatches node to AI Agent node.

5. **Add OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Set model to "gpt-5-mini".  
   - Configure OpenAI API credentials (create and connect).  
   - Connect AI Agent’s `ai_languageModel` input to this node.

6. **Add Notion Create Page Node**  
   - Type: Notion  
   - Operation: Create database page  
   - Configure with your Notion integration credentials.  
   - Set database ID to your newsletter database.  
   - Set page title to current batch item’s `Newsletter Title`.  
   - Connect AI Agent’s output to this node.

7. **Add Code Node (JavaScript)**  
   - Purpose: Chunk the generated newsletter content into smaller blocks for Notion.  
   - Paste provided JS code that:  
     - Serializes AI output JSON string.  
     - Slices it into chunks (~1800 characters).  
     - Wraps each chunk as a Notion paragraph block.  
   - Connect output of Notion Create Page node to this Code node.

8. **Add HTTP Request Node to Update Notion Page Blocks**  
   - Type: HTTP Request  
   - Method: PATCH  
   - URL: `https://api.notion.com/v1/blocks/{{ $json["id"] }}/children` (replace with dynamic page ID from Create Page node)  
   - Authentication: Use your Notion API credentials.  
   - Body: Pass the chunked children blocks from Code node.  
   - Set "On Error" to "Continue" so the workflow does not stop on failure.  
   - Connect Code node output to this node.

9. **Add Google Sheets Update Row Node**  
   - Type: Google Sheets  
   - Operation: Update Row  
   - Configure to update the same row in the sheet:  
     - Match on "Newsletter Title" to identify the row.  
     - Update "N8n Status" to "Done".  
   - Connect output of HTTP Request node to this node.

10. **Connect Update Row Node Back to SplitInBatches Node**  
    - This closes the loop to process the next batch item.

11. **Credentials Setup**  
    - Google Sheets OAuth2 credentials for sheet access.  
    - OpenAI API credentials for GPT-5-mini access.  
    - Notion API credentials with database write permissions.

12. **Test the Workflow**  
    - Place some rows in Google Sheets with "Pending" status and valid newsletter titles and descriptions.  
    - Execute the workflow manually and verify:  
      - AI-generated newsletter drafts created.  
      - Pages created in Notion with content uploaded properly.  
      - Rows updated to "Done" in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| The AI Agent system prompt is carefully crafted to emulate Ali Abdaal’s newsletter style, focusing on storytelling, reflection, and a warm, conversational tone.                                                                           | System message inside AI Agent node configuration.                                                                   |
| The workflow respects Notion API limits by chunking content into ~1800 character blocks before uploading, preventing potential errors with large text payloads.                                                                           | Code node JavaScript logic.                                                                                           |
| This workflow requires n8n’s LangChain nodes and a GPT-5-mini model accessible via OpenAI API, which may require specific API access or subscription plans.                                                                              | OpenAI Chat Model node configuration.                                                                                 |
| Google Sheets and Notion credentials must have appropriate permissions to read/write data to the specified documents and databases.                                                                                                      | Credential setup for Google Sheets and Notion API nodes.                                                              |
| The workflow initiates only on manual trigger, allowing controlled runs to avoid unintended batch processing. Automation triggers can be added if desired for scheduled or event-based execution.                                        | Manual Trigger node.                                                                                                  |
| Consider adding error handling or notifications for API failures or when no pending rows are found to improve robustness.                                                                                                                | Not implemented in this workflow but recommended for production use.                                                  |
| Useful links: Notion API docs [https://developers.notion.com/], LangChain [https://python.langchain.com/en/latest/], OpenAI API [https://platform.openai.com/docs].                                                                     | External resource references.                                                                                          |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. This processing fully complies with content policies and contains no illegal, offensive, or protected material. All data manipulated is legal and public.