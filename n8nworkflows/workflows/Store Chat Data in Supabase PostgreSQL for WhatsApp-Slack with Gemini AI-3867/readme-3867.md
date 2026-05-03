Store Chat Data in Supabase PostgreSQL for WhatsApp/Slack with Gemini AI

https://n8nworkflows.xyz/workflows/store-chat-data-in-supabase-postgresql-for-whatsapp-slack-with-gemini-ai-3867


# Store Chat Data in Supabase PostgreSQL for WhatsApp/Slack with Gemini AI

### 1. Workflow Overview

This n8n workflow is designed to capture chat data from platforms like WhatsApp or Slack, process it using AI, and store it in a Supabase PostgreSQL database. Its main goal is to handle unstructured chat inputs by mapping them into structured variables and persisting them for future use such as analytics or user interaction history.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Manual trigger and initial variable setting to simulate chat data input.
- **1.2 AI Processing:** Uses Google Gemini and a LangChain agent to process or validate the chat input.
- **1.3 Data Storage:** Persists the processed data into Supabase PostgreSQL.
- **1.4 Data Update Confirmation:** Updates additional fields (e.g., name) in the database after initial storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block simulates receiving chat data by manually triggering the workflow and setting sample input variables such as session ID, user name, and chat message.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Set sample Input Variables (Set Node)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually for testing purposes.  
  - Configuration: No parameters; triggers workflow on user action.  
  - Inputs: None  
  - Outputs: Triggers next node  
  - Edge Cases: None typical; manual trigger avoids runtime errors but not suitable for production.  
  - Notes: In production, this node should be replaced by a chat platform trigger (WhatsApp/Slack).

- **Set sample Input Variables**  
  - Type: Set  
  - Role: Defines mock chat data variables to simulate incoming unstructured chat data.  
  - Configuration: Sets three variables:  
    - `session_id` = "491634502879" (string, simulating a unique user/session identifier such as phone number)  
    - `name` = "Genn Sverster" (string, user name)  
    - `chatInput` = "wie gehts dir?" (string, sample chat message)  
  - Expressions: Values are assigned using expressions with `=` prefix (e.g., `=491634502879`).  
  - Inputs: Manual trigger node  
  - Outputs: Passes variables to AI processing nodes  
  - Edge Cases: Hardcoded values limit testing scope; in production, these would be dynamic inputs from chat triggers.

---

#### 2.2 AI Processing

**Overview:**  
Processes the chat input using AI models to clean, validate, or enrich the data before storage. This block chains two AI nodes: a Google Gemini model and a LangChain agent.

**Nodes Involved:**  
- GeminiFlash2.0 (Google Gemini Chat Model)  
- Sample Agent (LangChain Agent)

**Node Details:**

- **GeminiFlash2.0**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  - Role: Processes chat input using Google Gemini 2.0 Flash model.  
  - Configuration:  
    - Model: `models/gemini-2.0-flash`  
    - Credentials: Google Palm API key configured (`Stardawn#1`)  
  - Inputs: Receives data from "Set sample Input Variables" node.  
  - Outputs: Passes processed data to "Sample Agent" node.  
  - Edge Cases:  
    - API key invalid or expired could cause failures.  
    - Model unavailability or rate limits.  
    - Network timeouts.  
  - Notes: Placeholder for AI processing; can be customized or skipped.

- **Sample Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Further processes chat input text with a LangChain agent for contextual understanding or response generation.  
  - Configuration:  
    - Text input: Expression `={{ $json.chatInput }}` (takes chat message from previous node)  
    - System message: "You are a helpful assistant" (sets agent behavior)  
    - Prompt type: `define` (predefined prompt template)  
  - Inputs: Receives AI-processed data from GeminiFlash2.0 and raw variables from "Set sample Input Variables".  
  - Outputs: Sends data to "Update additional Values e.g. Name, Address ..." node.  
  - Edge Cases:  
    - Expression failures if `chatInput` is missing or malformed.  
    - API errors or timeouts.  
  - Notes: Can be tailored for specific AI tasks such as validation or enrichment.

---

#### 2.3 Data Storage

**Overview:**  
Stores the chat session data into a Supabase PostgreSQL table, associating the session ID as a key and including chat context memory.

**Nodes Involved:**  
- Supabase Postgres Database (LangChain PostgreSQL Memory Node)

**Node Details:**

- **Supabase Postgres Database**  
  - Type: `@n8n/n8n-nodes-langchain.memoryPostgresChat`  
  - Role: Persists chat data into Supabase PostgreSQL as a memory store for chat sessions.  
  - Configuration:  
    - Table name: `whatsapp_messages3`  
    - Session key: Expression `={{ $json.session_id }}` (uses session_id as unique key)  
    - Session ID type: `customKey` (indicates custom identifier)  
    - Context window length: 20 (number of chat messages to retain in context)  
    - Credentials: Supabase PostgreSQL credentials configured (`Supabase SD - N8N Demo Chatbot`)  
  - Inputs: Receives AI-processed data from GeminiFlash2.0 and "Set sample Input Variables".  
  - Outputs: Feeds into "Sample Agent" node as memory context.  
  - Edge Cases:  
    - Database connection failures (network, auth).  
    - Table or schema mismatch errors.  
    - Data type conflicts for session key or fields.  
  - Notes: Essential for chat history persistence and context-aware AI responses.

---

#### 2.4 Data Update Confirmation

**Overview:**  
Updates additional user information fields (e.g., name) in the Supabase table after initial data insertion, ensuring user profile completeness.

**Nodes Involved:**  
- Update additional Values e.g. Name, Address ... (Supabase Node)

**Node Details:**

- **Update additional Values e.g. Name, Address ...**  
  - Type: `n8n-nodes-base.supabase`  
  - Role: Updates existing records in Supabase table `whatsapp_messages3` to add or correct fields like `name`.  
  - Configuration:  
    - Table ID: `whatsapp_messages3`  
    - Filters:  
      - `session_id` equals the session_id from "Set sample Input Variables" (`={{ $('Set sample Input Variables').item.json.session_id }}`)  
      - `name` is `NULL` (only update records missing a name)  
    - Fields to update:  
      - `name` set to value from "Set sample Input Variables" (`={{ $('Set sample Input Variables').item.json.name }}`)  
    - Match type: `allFilters` (both conditions must be true)  
    - Operation: `update`  
    - Credentials: Supabase API credentials (`N8N Chatbot`)  
  - Inputs: Receives data from "Sample Agent" node.  
  - Outputs: None (end of workflow)  
  - Edge Cases:  
    - No matching records found (update affects zero rows).  
    - Permission or authentication errors with Supabase API.  
    - Network or timeout issues.  
  - Notes: Ensures user metadata is complete; can be extended for other fields like address.

---

### 3. Summary Table

| Node Name                                | Node Type                                   | Functional Role                                    | Input Node(s)                    | Output Node(s)                             | Sticky Note                                                                                                   |
|-----------------------------------------|---------------------------------------------|---------------------------------------------------|---------------------------------|--------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’            | Manual Trigger                              | Starts the workflow manually for testing          | None                            | Set sample Input Variables                  | In production, replace with WhatsApp/Slack message trigger for real chat input.                              |
| Set sample Input Variables               | Set                                         | Defines mock chat data variables                    | When clicking ‘Test workflow’    | GeminiFlash2.0, Sample Agent               | Maps unstructured chat data to structured variables for database storage.                                    |
| GeminiFlash2.0                          | Google Gemini Chat Model                     | AI processing of chat input                         | Set sample Input Variables       | Sample Agent                               | Placeholder AI node; can be customized or skipped.                                                           |
| Supabase Postgres Database               | LangChain PostgreSQL Memory Node             | Stores chat session data in Supabase PostgreSQL    | GeminiFlash2.0, Set sample Input Variables | Sample Agent                               | Persists chat context for session-based AI memory.                                                          |
| Sample Agent                            | LangChain Agent                             | Further AI processing and validation                | GeminiFlash2.0, Set sample Input Variables | Update additional Values e.g. Name, Address ... | Processes chat input text with system prompt "You are a helpful assistant".                                  |
| Update additional Values e.g. Name, Address ... | Supabase Node                               | Updates additional user info fields in Supabase    | Sample Agent                    | None                                       | Updates records where name is NULL; ensures user profile completeness.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: `When clicking ‘Test workflow’`  
   - No parameters needed. Connect this node as the workflow entry point.

2. **Create Set Node to Define Sample Input Variables**  
   - Type: Set  
   - Name: `Set sample Input Variables`  
   - Add the following fields with values (all string type):  
     - `session_id`: `491634502879`  
     - `name`: `Genn Sverster`  
     - `chatInput`: `wie gehts dir?`  
   - Connect output of Manual Trigger to this node.

3. **Create Google Gemini Chat Model Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Name: `GeminiFlash2.0`  
   - Parameters:  
     - Model Name: `models/gemini-2.0-flash`  
   - Credentials: Configure Google Palm API credentials with a valid API key.  
   - Connect output of `Set sample Input Variables` to this node.

4. **Create Supabase PostgreSQL Memory Node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryPostgresChat`  
   - Name: `Supabase Postgres Database`  
   - Parameters:  
     - Table Name: `whatsapp_messages3`  
     - Session Key: Expression `={{ $json.session_id }}`  
     - Session ID Type: `customKey`  
     - Context Window Length: `20`  
   - Credentials: Configure with Supabase PostgreSQL credentials:  
     - Host, Port, Database, User, Password (from your Supabase connection string)  
     - Enable SSL  
   - Connect output of `GeminiFlash2.0` and `Set sample Input Variables` to this node as AI memory and input respectively.

5. **Create LangChain Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Name: `Sample Agent`  
   - Parameters:  
     - Text: Expression `={{ $json.chatInput }}`  
     - System Message: `"You are a helpful assistant"`  
     - Prompt Type: `define`  
   - Connect output of `GeminiFlash2.0` and `Set sample Input Variables` to this node.

6. **Create Supabase Node to Update Additional Values**  
   - Type: `n8n-nodes-base.supabase`  
   - Name: `Update additional Values e.g. Name, Address ...`  
   - Parameters:  
     - Table ID: `whatsapp_messages3`  
     - Filters (all must match):  
       - `session_id` equals expression `={{ $('Set sample Input Variables').item.json.session_id }}`  
       - `name` is `NULL`  
     - Fields to update:  
       - `name` set to expression `={{ $('Set sample Input Variables').item.json.name }}`  
     - Operation: `update`  
   - Credentials: Configure with Supabase API credentials with proper permissions.  
   - Connect output of `Sample Agent` node to this node.

7. **Connect Nodes in Order:**  
   - `When clicking ‘Test workflow’` → `Set sample Input Variables`  
   - `Set sample Input Variables` → `GeminiFlash2.0`  
   - `GeminiFlash2.0` → `Sample Agent`  
   - `Set sample Input Variables` → `Sample Agent` (parallel input)  
   - `GeminiFlash2.0` → `Supabase Postgres Database` (AI memory input)  
   - `Set sample Input Variables` → `Supabase Postgres Database` (main input)  
   - `Sample Agent` → `Update additional Values e.g. Name, Address ...`

8. **Test Workflow:**  
   - Run manually using the trigger.  
   - Verify data is inserted and updated correctly in Supabase table `whatsapp_messages3`.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Guide with images and detailed setup instructions is available at: https://github.com/JimPresting/Supabase-n8n-Self-Hosted-Integration/edit/main/README.md | Project GitHub repository README for setup and troubleshooting.                                  |
| Firewall configuration is critical to allow n8n to connect to Supabase PostgreSQL; restrict IPs for security. | See Step 1 in the description for firewall setup instructions.                                  |
| Store Supabase passwords securely using n8n environment variables to avoid exposing credentials in workflows. | Security best practice for credential management.                                               |
| SSL must be enabled in Supabase PostgreSQL node for secure communication.                                  | Supabase requires SSL connections by default.                                                  |
| This workflow is built to be extended for real chat platform triggers replacing the manual trigger.       | Production integration with WhatsApp or Slack requires platform-specific webhook or API triggers.|

---

This document fully describes the workflow structure, node configurations, and setup instructions to allow advanced users or AI agents to understand, reproduce, and modify the workflow confidently.