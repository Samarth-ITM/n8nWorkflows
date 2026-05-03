Semantic Prompt Search Assistant with Notion, ChatGPT, and HuggingFace

https://n8nworkflows.xyz/workflows/semantic-prompt-search-assistant-with-notion--chatgpt--and-huggingface-11194


# Semantic Prompt Search Assistant with Notion, ChatGPT, and HuggingFace

### 1. Workflow Overview

This workflow, titled **"Semantic Prompt Search Assistant with Notion, ChatGPT, and HuggingFace"**, is designed to create an intelligent chat interface that integrates with a Notion database of saved prompts. Its main purpose is to semantically search for the most relevant saved prompt matching a user’s query and suggest it before generating a direct AI response. If no suitable prompt is found, it answers normally. This enhances the user experience by reusing curated prompts and reducing redundant AI calls.

The workflow is logically divided into three main blocks:

- **1.1 Embeddings Generator**: Automatically generate or update vector embeddings for prompts stored in Notion based on page creation, update, or manual sync.
- **1.2 Prompt Finder**: Upon receiving a user query, convert the query to an embedding and compare it with stored prompt embeddings to find the best semantic match.
- **1.3 Chat Interface with AI Agent**: Intercepts each chat message, triggers the prompt search tool, and depending on results, either suggests a stored prompt for user confirmation or proceeds with normal AI response generation.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Embeddings Generator

**Overview:**  
This block listens to Notion database page creation and update events to generate or update embeddings for prompts. It can also sync all prompts on-demand via a webhook. It calculates a checksum to detect changes and updates the Notion page with embeddings and checksum to avoid redundant processing.

**Nodes Involved:**  
- On Page Create  
- On Page Update  
- Generate Checksum  
- If Embeddings Update Needed  
- Get Embeddings  
- Prepare Update Body  
- Save Embeddings + Checksum  
- Get All Prompts (used for full sync)  
- Embeddings - Sync All (webhook trigger)

**Node Details:**

- **On Page Create**  
  - Type: Notion Trigger  
  - Role: Detects newly created pages in the Notion prompt database every minute.  
  - Config: Polls Notion database for new pages.  
  - Inputs: Triggers workflow on new page.  
  - Outputs: Sends new page data to Generate Checksum.  
  - Edge Cases: Delays due to polling frequency; Notion API rate limits.

- **On Page Update**  
  - Type: Notion Trigger  
  - Role: Detects updated pages in the Notion prompt database every minute.  
  - Config: Polls Notion database for updated pages.  
  - Inputs: Triggers workflow on page update.  
  - Outputs: Sends updated page data to Generate Checksum.  
  - Edge Cases: Same as On Page Create.

- **Generate Checksum**  
  - Type: Code  
  - Role: Computes SHA-256 checksum of the prompt text to detect if embeddings need updating.  
  - Config: Uses a pure JavaScript SHA-256 implementation.  
  - Inputs: Prompt text and existing checksum from Notion page.  
  - Outputs: JSON object with pageId, prompt, oldChecksum, newChecksum, and needsUpdate boolean.  
  - Edge Cases: Missing prompt text; checksum mismatch logic.  
  - Notes: Prevents unnecessary embedding regeneration.

- **If Embeddings Update Needed**  
  - Type: If  
  - Role: Conditional check based on needsUpdate flag from checksum node.  
  - Config: Passes only if needsUpdate is true (boolean check).  
  - Inputs: Output from Generate Checksum.  
  - Outputs: Routes to Get Embeddings if true; stops otherwise.  
  - Edge Cases: False negatives if checksum logic fails.

- **Get Embeddings**  
  - Type: HTTP Request  
  - Role: Calls HuggingFace API to generate embedding vector for the prompt text.  
  - Config: POST to HuggingFace multilingual-e5-small model endpoint with prompt as input.  
  - Credentials: Uses HTTP Bearer Auth with HuggingFace token.  
  - Inputs: Prompt text from previous nodes.  
  - Outputs: Embedding vector string.  
  - Edge Cases: API errors, timeouts, auth failures.

- **Prepare Update Body**  
  - Type: Code  
  - Role: Formats embedding data and checksum into a Notion-compatible update payload.  
  - Config: Splits embedding string into chunks below 2000 characters to fit Notion rich_text limits.  
  - Inputs: Embedding string from Get Embeddings and pageId/checksum from Generate Checksum.  
  - Outputs: JSON body for Notion API PATCH request.  
  - Edge Cases: Large embeddings, JSON parsing errors.

- **Save Embeddings + Checksum**  
  - Type: HTTP Request  
  - Role: Updates the Notion page with new embeddings and checksum properties.  
  - Config: PATCH request to Notion API with prepared JSON body.  
  - Credentials: Notion API OAuth2 credentials.  
  - Inputs: JSON body from Prepare Update Body, pageId for URL.  
  - Outputs: Confirmation of update.  
  - Edge Cases: API rate limits, permission errors.

- **Get All Prompts**  
  - Type: Notion  
  - Role: Retrieves all prompts from Notion database for full sync.  
  - Config: Get all pages, no limit.  
  - Credentials: Notion API credentials.  
  - Inputs: Triggered by Embeddings - Sync All webhook.  
  - Outputs: List of all prompts to Generate Checksum.  
  - Edge Cases: Large database pagination, API throttling.

- **Embeddings - Sync All**  
  - Type: Webhook  
  - Role: Manual trigger to regenerate embeddings for all prompts.  
  - Config: POST webhook at path `/embeddings/sync-all`.  
  - Inputs: External manual or automated POST request.  
  - Outputs: Triggers Get All Prompts node.  
  - Edge Cases: Heavy load on large databases.

---

#### 2.2 Prompt Finder

**Overview:**  
This block receives a user query, generates its embedding, retrieves all saved prompts with their embeddings, and calculates cosine similarity to find the best matching prompt. It returns the best prompt, similarity score, and a notFound flag.

**Nodes Involved:**  
- Search Entrypoint (Execute Workflow Trigger)  
- Get Question Embeddings  
- Get All Prompts for search  
- Find a prompt  
- Search a Prompt (Tool Workflow node used by AI Agent)

**Node Details:**

- **Search Entrypoint**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for semantic prompt search tool invoked by AI Agent.  
  - Config: Accepts `query` parameter from calling workflow.  
  - Inputs: Receives user query string.  
  - Outputs: Passes query to Get Question Embeddings.  
  - Edge Cases: Missing or empty query.

- **Get Question Embeddings**  
  - Type: HTTP Request  
  - Role: Generates embedding vector for user query using HuggingFace multilingual-e5-small model.  
  - Config: POST request with query as input.  
  - Credentials: HuggingFace Bearer token.  
  - Inputs: Query string from Search Entrypoint.  
  - Outputs: Embedding vector string.  
  - Edge Cases: API failures, invalid input.

- **Get All Prompts for search**  
  - Type: Notion  
  - Role: Fetches all saved prompts with their embeddings from Notion database for similarity comparison.  
  - Config: Get all pages, no limit.  
  - Credentials: Notion API.  
  - Inputs: Triggered after query embedding.  
  - Outputs: List of prompt pages with embeddings.  
  - Edge Cases: Large datasets, pagination.

- **Find a prompt**  
  - Type: Code  
  - Role: Performs cosine similarity calculation between query embedding and each saved prompt embedding.  
  - Config:  
    - Parses embeddings from JSON stored as rich_text in Notion.  
    - Computes cosine similarity score.  
    - Applies threshold (0.4) to decide if a prompt is found.  
    - Returns best prompt text, similarity score, notFound flag, and pageId.  
  - Inputs: Embeddings from Get Question Embeddings and prompt data from Get All Prompts for search.  
  - Outputs: JSON with prompt search result.  
  - Edge Cases: Missing or invalid embeddings, low similarity scores.

- **Search a Prompt**  
  - Type: Langchain Tool Workflow  
  - Role: Exposes Prompt Finder as a callable tool for AI Agent nodes.  
  - Config: Calls the above workflow block with `query` input.  
  - Inputs: User message from AI Agent.  
  - Outputs: Found prompt or notFound flag to AI Agent.  
  - Edge Cases: Workflow invocation errors.

---

#### 2.3 Chat Interface with AI Agent

**Overview:**  
This block handles incoming chat messages, always calls the Prompt Finder tool first, and based on whether a matching prompt is found, either suggests it for user confirmation or directly responds using OpenAI’s GPT-4. It maintains conversation context with a simple memory buffer.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- OpenAI Chat Model  
- Simple Memory  
- Search a Prompt (tool node referenced by AI Agent)

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Public webhook endpoint for receiving chat messages from users.  
  - Config: Initial greeting message "Hi there! 👋 My name is JARVIS. How can I assist you today?"  
  - Inputs: Incoming chat message via webhook.  
  - Outputs: Passes message to AI Agent.  
  - Edge Cases: Network errors, webhook misconfiguration.

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Orchestrates chat message processing. Always calls `Search a Prompt` tool first with user message.  
  - Config: System message describes logic: first search prompt; if found, suggest prompt and wait user confirmation; if not found, answer normally.  
  - Inputs: User message from chat trigger.  
  - Outputs: Depending on prompt search, routes to OpenAI Chat Model or waits user confirmation.  
  - Edge Cases: Tool call failures, logic errors in branching, user confirmation handling.

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Generates AI response using GPT-4.1-mini model when no prompt match is found or user accepts AI answer.  
  - Config: Uses OpenAI API with GPT-4.1-mini model.  
  - Credentials: OpenAI API key.  
  - Inputs: User message and context memory.  
  - Outputs: AI-generated chat response.  
  - Edge Cases: API rate limits, model errors.

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains short conversation history of last 10 messages for context in AI responses.  
  - Config: Context window length set to 10.  
  - Inputs: Incoming messages and AI responses.  
  - Outputs: Provides conversation context to AI Agent.  
  - Edge Cases: Memory overflow or loss.

---

### 3. Summary Table

| Node Name                | Node Type                           | Functional Role                                    | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                  |
|--------------------------|-----------------------------------|---------------------------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| When chat message received| Langchain Chat Trigger             | Receives user chat messages                        | —                                | AI Agent                         | ChatGPT interface that searches for prompts on each message. If found, suggests the prompt. |
| AI Agent                 | Langchain Agent                    | Controls chat logic, calls prompt search, responds| When chat message received        | OpenAI Chat Model, Search a Prompt| ChatGPT interface that searches for prompts on each message. If found, suggests the prompt. |
| OpenAI Chat Model        | Langchain OpenAI LM                | Generates AI answers                               | AI Agent                        | —                                 | ChatGPT interface that searches for prompts on each message. If found, suggests the prompt. |
| Simple Memory            | Langchain Memory Buffer Window    | Conversation history context                       | AI Agent                        | AI Agent                         | ChatGPT interface that searches for prompts on each message. If found, suggests the prompt. |
| Embeddings - Sync All    | Webhook                           | Manual sync trigger for embeddings regeneration   | —                                | Get All Prompts                   | Generates embeddings for new/updated prompts. Triggers: Notion page create/update or manual sync webhook. |
| Get All Prompts          | Notion                            | Retrieves all prompts for sync                     | Embeddings - Sync All             | Generate Checksum                | Generates embeddings for new/updated prompts. Triggers: Notion page create/update or manual sync webhook. |
| On Page Update           | Notion Trigger                    | Detects updated prompt pages                       | —                                | Generate Checksum                | Generates embeddings for new/updated prompts. Triggers: Notion page create/update or manual sync webhook. |
| On Page Create           | Notion Trigger                    | Detects newly created prompt pages                 | —                                | Generate Checksum                | Generates embeddings for new/updated prompts. Triggers: Notion page create/update or manual sync webhook. |
| Generate Checksum        | Code                             | Computes checksum to detect prompt changes        | On Page Update, On Page Create, Get All Prompts | If Embeddings Update Needed      | Generates embeddings for new/updated prompts. Triggers: Notion page create/update or manual sync webhook. |
| If Embeddings Update Needed | If                             | Checks if embeddings need update                   | Generate Checksum                 | Get Embeddings                  | Generates embeddings for new/updated prompts. Triggers: Notion page create/update or manual sync webhook. |
| Get Embeddings           | HTTP Request                     | Calls HuggingFace API to generate prompt embedding| If Embeddings Update Needed       | Prepare Update Body             | Generates embeddings for new/updated prompts. Triggers: Notion page create/update or manual sync webhook. |
| Prepare Update Body      | Code                             | Formats embedding data for Notion update           | Get Embeddings                   | Save Embeddings + Checksum      | Generates embeddings for new/updated prompts. Triggers: Notion page create/update or manual sync webhook. |
| Save Embeddings + Checksum | HTTP Request                   | Updates Notion page with new embeddings/checksum   | Prepare Update Body              | —                             | Generates embeddings for new/updated prompts. Triggers: Notion page create/update or manual sync webhook. |
| Search Entrypoint        | Execute Workflow Trigger          | Entry point for prompt search tool                  | —                                | Get Question Embeddings         | Searches Notion prompts using semantic similarity. Compares user message embeddings with saved prompt embeddings. |
| Get Question Embeddings  | HTTP Request                     | Generates embedding for user query                  | Search Entrypoint                | Get All Prompts for search      | Searches Notion prompts using semantic similarity. Compares user message embeddings with saved prompt embeddings. |
| Get All Prompts for search | Notion                         | Retrieves all prompts for prompt search             | Get Question Embeddings          | Find a prompt                  | Searches Notion prompts using semantic similarity. Compares user message embeddings with saved prompt embeddings. |
| Find a prompt            | Code                             | Calculates cosine similarity, finds best matching prompt | Get All Prompts for search       | Search a Prompt (Tool Workflow) | Searches Notion prompts using semantic similarity. Compares user message embeddings with saved prompt embeddings. |
| Search a Prompt          | Langchain Tool Workflow          | Exposes prompt search as callable tool              | AI Agent (ai_tool)               | AI Agent                       | ChatGPT interface that searches for prompts on each message. If found, suggests the prompt. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Notion Database:**  
   - Columns:  
     - `Prompt` (Text)  
     - `Embeddings` (Text)  
     - `Checksum` (Text)  
   - Obtain Database ID for configuration.

2. **Configure Credentials:**  
   - OpenAI API Key (for GPT-4.1-mini)  
   - Notion API OAuth2 credentials with access to the database  
   - HuggingFace Bearer token (with access to multilingual-e5-small model)

3. **Setup Embeddings Generator Block:**

   - **Add Notion Triggers:**  
     - `On Page Create` and `On Page Update` nodes configured to poll the Notion database every minute for new/updated pages.

   - **Add `Generate Checksum` Code Node:**  
     - Implement SHA-256 hash of prompt text.  
     - Outputs `needsUpdate` boolean.

   - **Add `If Embeddings Update Needed` Node:**  
     - Condition: `needsUpdate == true`.

   - **Add `Get Embeddings` HTTP Request Node:**  
     - POST to HuggingFace model endpoint with prompt text as input.  
     - Use Bearer auth with HuggingFace token.

   - **Add `Prepare Update Body` Code Node:**  
     - Split embedding string into chunks under 2000 characters.  
     - Format JSON for Notion page update with `Embeddings` and `Checksum` properties.

   - **Add `Save Embeddings + Checksum` HTTP Request Node:**  
     - PATCH Notion page using pageId and JSON body.  
     - Use Notion OAuth2 credentials.

   - **Link nodes:**  
     - On Page Create → Generate Checksum → If Embeddings Update Needed → Get Embeddings → Prepare Update Body → Save Embeddings + Checksum  
     - On Page Update → same as above.

   - **Add `Embeddings - Sync All` Webhook Node:**  
     - POST `/embeddings/sync-all` path.  
     - Connect to `Get All Prompts` node to fetch all prompts.  
     - Chain `Get All Prompts` → `Generate Checksum` (for each prompt).

4. **Setup Prompt Finder Block:**

   - **Add `Search Entrypoint` Execute Workflow Trigger Node:**  
     - Accepts input parameter `query` (string).

   - **Add `Get Question Embeddings` HTTP Request Node:**  
     - POST to HuggingFace with user query.  
     - Use Bearer auth.

   - **Add `Get All Prompts for search` Notion Node:**  
     - Fetch all prompt pages from Notion.

   - **Add `Find a prompt` Code Node:**  
     - Parse embeddings, compute cosine similarity with query embedding.  
     - Return best prompt, score, and notFound flag based on threshold 0.4.

   - **Connect Nodes:**  
     - Search Entrypoint → Get Question Embeddings → Get All Prompts for search → Find a prompt

5. **Setup Chat Interface Block:**

   - **Add `When chat message received` Langchain Chat Trigger:**  
     - Public webhook, initial greeting message configured.

   - **Add `AI Agent` Langchain Agent Node:**  
     - System message instructing:  
       1) Always call `Search a Prompt` tool with user message as `query`.  
       2) If prompt found, suggest prompt and wait for user confirmation.  
       3) Else proceed with normal answer.  
     - Connect input from chat trigger.

   - **Add `OpenAI Chat Model` Node:**  
     - Model: GPT-4.1-mini  
     - Credentials: OpenAI API key

   - **Add `Simple Memory` Node:**  
     - Context window length: 10 messages  
     - Connect as AI Agent memory.

   - **Configure AI Agent connections:**  
     - AI Agent calls `Search a Prompt` (the tool workflow above).  
     - AI Agent uses `OpenAI Chat Model` for answer generation.  
     - AI Agent uses `Simple Memory` for conversation context.

6. **Integrate Search Tool in AI Agent:**  
   - In AI Agent node, add tool `Search a Prompt` referencing the prompt finder workflow.  
   - Map the user message as `query` input parameter.

7. **Activate all nodes and workflows.**

8. **Test by sending chat messages to the webhook URL of `When chat message received`.**  
   - Verify prompt search and suggestion logic.  
   - Verify fallback to AI response when no prompt found.  
   - Test prompt create/update in Notion to confirm embeddings update.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow automatically searches your saved prompts in Notion when you chat with ChatGPT. It converts messages and prompts into embeddings using HuggingFace, compares them by cosine similarity, and suggests the best prompt before answering normally.                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note3 content describing overall architecture and usage.                                   |
| Setup instructions: 1) Import workflow, 2) Configure OpenAI, HuggingFace, Notion credentials, 3) Create Notion database with required columns, 4) Set database IDs in nodes, 5) Activate workflow and use webhook URL.                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note3 content.                                                                              |
| Embeddings Generator triggers on Notion page create/update and via manual webhook. It calculates checksums to avoid redundant embedding calls and updates Notion with new embeddings.                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note content near Embeddings Generator block.                                              |
| Chat interface uses Langchain AI Agent and memory buffer. It always calls prompt search tool first, then acts based on search results.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note1 content near chat interface nodes.                                                   |
| Prompt Finder compares user query embeddings with stored prompt embeddings using cosine similarity and a threshold of 0.4 to decide matches.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note2 content near Prompt Finder nodes.                                                    |
| HuggingFace model used: `intfloat/multilingual-e5-small` for feature extraction (embeddings). OpenAI model used: `gpt-4.1-mini`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Configuration details from HTTP Request nodes.                                                    |
| Notion API: Ensure OAuth2 credentials have read/write access to the database; rich_text fields are used to store embeddings split into chunks due to size limits.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Notion API usage notes inferred from nodes.                                                      |
| SHA-256 checksum implemented in pure JavaScript for sandbox compatibility to detect prompt text changes reliably without external libraries.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Code node implementation detail.                                                                  |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing fully complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.