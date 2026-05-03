Chat with Your HubSpot Data (Contacts & Deals)

https://n8nworkflows.xyz/workflows/chat-with-your-hubspot-data--contacts---deals--8971


# Chat with Your HubSpot Data (Contacts & Deals)

### 1. Workflow Overview

This workflow enables dynamic conversational interaction with HubSpot CRM data, specifically contacts and deals, via an AI-driven chat interface. It is designed to receive chat messages, process them using an AI agent powered by Google Gemini language models, and query HubSpot for relevant contact and deal information based on the user's input. The workflow contains logical blocks for input reception, AI processing with memory, and HubSpot data retrieval tools.

Logical blocks:  
- **1.1 Input Reception:** Receives chat messages from users via a chat trigger node.  
- **1.2 AI Processing and Memory:** Processes the chat input using an AI Agent node configured with Google Gemini as the language model and a simple memory buffer to maintain conversational context.  
- **1.3 HubSpot Data Retrieval:** Performs searches on HubSpot CRM for contacts and deals using dynamic filters derived from the AI agent’s parsing of the chat input.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block serves as the entry point for the workflow. It listens for incoming chat messages from users, triggering the rest of the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Webhook/listener node that triggers the workflow on receiving chat messages.  
    - Configuration: Default options; uses a unique webhook ID to receive messages.  
    - Inputs: External chat messages from clients or users.  
    - Outputs: Triggers the AI Agent node with the chat content.  
    - Version: 1.3  
    - Edge Cases: Network or webhook errors, message payload format issues, webhook ID misconfiguration.  
    - Sub-workflow: None.

#### 1.2 AI Processing and Memory

- **Overview:**  
  This block processes incoming chat messages with an AI Agent that leverages the Google Gemini language model and maintains conversation context with a simple memory buffer.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Simple Memory

- **Node Details:**

  - **AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Orchestrates AI processing; coordinates language model, memory, and tools for dynamic response generation.  
    - Configuration: Default options; connects to language model, memory, and external tools.  
    - Inputs: Chat messages from "When chat message received" node and context from "Simple Memory."  
    - Outputs: Generates AI-driven responses and determines appropriate HubSpot data queries.  
    - Version: 2.2  
    - Edge Cases: Model response delays/timeouts, malformed AI output, tool invocation errors, memory overflow issues.  
    - Sub-workflow: None.

  - **Google Gemini Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
    - Role: Provides advanced natural language understanding and generation via Google Gemini 2.5 Pro model.  
    - Configuration: Uses model "models/gemini-2.5-pro"; authenticated with Google PaLM API credentials.  
    - Inputs: Receives prompts or context from AI Agent.  
    - Outputs: Returns generated text and intent for AI Agent.  
    - Version: 1  
    - Edge Cases: API authentication failures, quota limits, model latency.  
    - Credentials: Google Palm API with OAuth2.  
    - Sub-workflow: None.

  - **Simple Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains a sliding window memory buffer to preserve recent conversational context.  
    - Configuration: Default buffer settings (unspecified in detail).  
    - Inputs: Conversation context from AI Agent.  
    - Outputs: Provides memory context back to AI Agent.  
    - Version: 1.3  
    - Edge Cases: Memory overflow, context truncation, synchronization issues.  
    - Sub-workflow: None.

#### 1.3 HubSpot Data Retrieval

- **Overview:**  
  This block performs dynamic searches on HubSpot CRM for contacts and deals based on AI-parsed filters extracted from user queries.

- **Nodes Involved:**  
  - Search contacts in HubSpot  
  - Search for deals in HubSpot

- **Node Details:**

  - **Search contacts in HubSpot**  
    - Type: `n8n-nodes-base.hubspotTool`  
    - Role: Searches HubSpot contacts using filters dynamically derived from AI Agent outputs.  
    - Configuration:  
      - Operation: "search"  
      - Authentication: OAuth2 via HubSpot account credentials  
      - Filters: Dynamically injected using expressions from AI Agent outputs for `propertyName`, `operator`, and `value` (e.g., email, firstname).  
      - Properties to return: email, firstname, lastname, createdate, lifecyclestage, hs_lead_status, country, state, email-related fields, jobtitle, owner ID, engagement dates, last sales activity timestamp.  
      - Additional query field preset to "hans" (possibly default or fallback).  
    - Inputs: AI Agent's tool invocation with filters.  
    - Outputs: Search results passed back to AI Agent.  
    - Version: 2.1  
    - Credentials: HubSpot OAuth2 API  
    - Edge Cases: OAuth token expiration, invalid property names or operators, empty search results, API rate limiting, expression evaluation errors.

  - **Search for deals in HubSpot**  
    - Type: `n8n-nodes-base.hubspotTool`  
    - Role: Searches HubSpot deals using a fixed filter on `hs_all_owner_ids` property.  
    - Configuration:  
      - Resource: "deal"  
      - Operation: "search"  
      - Authentication: OAuth2 via HubSpot account credentials  
      - Filter: Fixed filter `hs_all_owner_ids = "="` (this likely requires dynamic adjustment or is a placeholder)  
      - Properties requested: dealname, amount, dealstage, pipeline, closedate, createdate, owner ID, last modified date.  
    - Inputs: Invoked as a tool by AI Agent.  
    - Outputs: Returns deal search results for AI Agent processing.  
    - Version: 2.1  
    - Credentials: HubSpot OAuth2 API  
    - Edge Cases: Same as above plus potential for insufficient or incorrect filters leading to large data returns or empty results.

---

### 3. Summary Table

| Node Name                 | Node Type                                   | Functional Role                          | Input Node(s)             | Output Node(s)           | Sticky Note                          |
|---------------------------|---------------------------------------------|----------------------------------------|---------------------------|--------------------------|------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger        | Entry trigger for chat messages        | External webhook           | AI Agent                 |                                    |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent               | AI orchestration, response generation  | When chat message received, Simple Memory, Google Gemini Chat Model, HubSpot nodes | HubSpot nodes or response |                                    |
| Google Gemini Chat Model  | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Language model for AI processing       | AI Agent                   | AI Agent                 | Requires Google PaLM API credential |
| Simple Memory             | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains chat conversation context    | AI Agent                   | AI Agent                 |                                    |
| Search contacts in HubSpot| n8n-nodes-base.hubspotTool                   | Searches contacts with dynamic filters | AI Agent                   | AI Agent                 | OAuth2 HubSpot credential used     |
| Search for deals in HubSpot| n8n-nodes-base.hubspotTool                   | Searches deals with fixed filter       | AI Agent                   | AI Agent                 | OAuth2 HubSpot credential used     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure webhook to receive chat messages, no special options needed.  
   - Position: (0,0) or as preferred.

2. **Create "Google Gemini Chat Model" node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Set `modelName` to `"models/gemini-2.5-pro"`.  
   - Configure credentials: Google Palm API OAuth2 with valid API key and access.  
   - Position: (80,208).

3. **Create "Simple Memory" node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Use default buffer settings or configure window size as needed.  
   - Position: (224,208).

4. **Create "AI Agent" node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - No special options required, default suffices.  
   - Connect inputs:  
     - From "When chat message received" (main input).  
     - From "Google Gemini Chat Model" (ai_languageModel input).  
     - From "Simple Memory" (ai_memory input).  
     - From HubSpot search nodes (ai_tool input).  
   - Position: (208,0).

5. **Create "Search contacts in HubSpot" node**  
   - Type: `n8n-nodes-base.hubspotTool`  
   - Operation: "search" with "authentication" set to OAuth2.  
   - Configure filter groups with dynamic expressions to receive filter values from AI Agent outputs:  
     - Use expressions to extract `propertyName`, `operator`, and `value` from AI Agent context (`$fromAI()`).  
   - Set properties to fetch: email, firstname, lastname, createdate, lifecyclestage, hs_lead_status, country, state, hs_email_last_open_date, hs_email_last_reply_date, hs_sequences_is_enrolled, hs_sequences_enrolled_count, jobtitle, hubspot_owner_id, hs_sa_first_engagement_date, hs_last_sales_activity_timestamp.  
   - Credentials: Attach HubSpot OAuth2 API credentials.  
   - Position: (432,208).

6. **Create "Search for deals in HubSpot" node**  
   - Type: `n8n-nodes-base.hubspotTool`  
   - Resource: "deal"  
   - Operation: "search" with OAuth2 authentication.  
   - Filter groups: Set a filter on `hs_all_owner_ids` property with operator "="; consider making this dynamic or adjusting for real use.  
   - Properties to return: dealname, amount, dealstage, pipeline, closedate, createdate, hubspot_owner_id, hs_lastmodifieddate.  
   - Credentials: Use the same HubSpot OAuth2 API credentials.  
   - Position: (608,208).

7. **Connect nodes:**  
   - "When chat message received" → "AI Agent" (main).  
   - "Google Gemini Chat Model" → "AI Agent" (ai_languageModel).  
   - "Simple Memory" → "AI Agent" (ai_memory).  
   - "Search contacts in HubSpot" → "AI Agent" (ai_tool).  
   - "Search for deals in HubSpot" → "AI Agent" (ai_tool).

8. **Credentials Setup:**  
   - Ensure Google Palm API credentials are configured for the Google Gemini Chat Model node.  
   - Ensure HubSpot OAuth2 credentials are configured for both HubSpot search nodes.

9. **Test the workflow:**  
   - Send a chat message through the webhook URL provided by "When chat message received".  
   - Verify that AI Agent processes the message, queries HubSpot contacts and deals, and returns relevant information.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                     |
|----------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| This workflow leverages Google Gemini (PaLM) API for advanced NLP capabilities in conversational AI.     | Requires Google Cloud account with PaLM API access |
| HubSpot API integration uses OAuth2 for secure access; ensure tokens are valid and scopes include CRM read.| HubSpot developer documentation for OAuth2         |
| Expressions in HubSpot search filters dynamically extract parameters from AI output for flexible queries.| n8n expression documentation: https://docs.n8n.io/ |
| The "Simple Memory" node uses a sliding window buffer to maintain context without growing indefinitely.    | n8n Langchain memory nodes documentation            |
| The "Search for deals in HubSpot" node filter is currently fixed; consider enhancing for dynamic queries. | Workflow improvement suggestion                      |

---

**Disclaimer:** The text provided is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.