Sales CRM Chatbot with GPT-4o-mini, Google Sheets Lookup & Memory

https://n8nworkflows.xyz/workflows/sales-crm-chatbot-with-gpt-4o-mini--google-sheets-lookup---memory-10840


# Sales CRM Chatbot with GPT-4o-mini, Google Sheets Lookup & Memory

### 1. Workflow Overview

This workflow implements an **AI-powered Sales CRM Chatbot** that leverages OpenAI’s GPT-4o-mini model integrated with Google Sheets to provide conversational sales support. It is designed to handle user queries related to sales leads, outreach status, proposals, and CRM opportunities by dynamically looking up relevant data in Google Sheets and maintaining conversational memory across interactions.

**Target Use Cases:**
- Sales teams seeking instant, AI-assisted access to CRM data.
- Automating retrieval and summarization of outreach and opportunity information.
- Maintaining chat memory for contextual, multi-turn conversations.
- Logging all chat interactions for audit or training purposes.

**Logical Blocks:**

- **1.1 Input Reception:** Triggering on new chat messages from n8n’s chat interface.
- **1.2 AI Reasoning & Routing:** Processing user queries with GPT-4o-mini and deciding which Google Sheets tools to invoke.
- **1.3 Data Lookup Tools:** Querying Google Sheets for outreach and opportunity data.
- **1.4 Conversation Memory:** Buffering recent chat context to enable multi-turn dialogue.
- **1.5 Output Validation & Logging:** Ensuring AI responses are valid, logging conversations or errors.
- **1.6 Response Delivery:** Sending the AI-generated response back to the user live.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new chat messages from users via the n8n Chat interface and initiates the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Webhook trigger for incoming chat messages, starting point of the workflow.  
    - Configuration:  
      - Uses response mode "responseNodes" to enable asynchronous chat handling.  
      - Captures inputs: `chatInput` (user’s message text) and sessionId.  
    - Inputs: External chat interface webhook  
    - Outputs: Forwards message data to "AI Sales CRM Router"  
    - Potential Failures: Webhook connectivity issues, invalid incoming payloads.

#### 1.2 AI Reasoning & Routing

- **Overview:**  
  The core logic hub that interprets user intent, decides if data lookups are required, formats tool calls in JSON, and generates final summarized responses.

- **Nodes Involved:**  
  - AI Sales CRM Router  
  - OpenAI Chat Model  
  - Conversation Memory Buffer

- **Node Details:**

  - **AI Sales CRM Router**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Conversational agent orchestrating AI reasoning and tool invocation.  
    - Configuration:  
      - System message sets context as an AI Sales CRM Agent with access to Google Sheets tools.  
      - Contains rules to never guess data; must always call sheet lookup tools first.  
      - Uses JSON-formatted tool call instructions for outreachSheet and opportunitySheet.  
      - Receives input from chat trigger and memory buffer.  
      - Outputs AI-generated JSON commands or responses.  
    - Inputs:  
      - User chat input from "When chat message received"  
      - Memory context from "Conversation Memory Buffer"  
      - LLM model from "OpenAI Chat Model"  
      - Tool results from Google Sheets nodes  
    - Outputs:  
      - AI response payload forwarded to validation node  
    - Edge Cases:  
      - Expression evaluation failures in prompt.  
      - Tool call formatting errors.  
      - AI hallucination prevented by strict lookup rules.  
      - API rate limits or timeouts.  

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides GPT-4o-mini conversational intelligence to the router.  
    - Configuration:  
      - Model: gpt-4o-mini  
      - Credentials: OpenAI API key (configured via OpenAi account 2)  
    - Input: From AI Sales CRM Router (agent node)  
    - Output: AI-generated text or JSON tool calls  
    - Potential Failures: Authentication errors, quota exhaustion, network timeouts.  

  - **Conversation Memory Buffer**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains last 7 turns of conversation for context.  
    - Configuration:  
      - contextWindowLength: 7 (last 7 chat exchanges preserved)  
    - Inputs: From "When chat message received" and "AI Sales CRM Router"  
    - Outputs: Provides context to "AI Sales CRM Router"  
    - Edge Cases: Memory overflow, session mismatches.

#### 1.3 Data Lookup Tools

- **Overview:**  
  These nodes query specific Google Sheets documents to fetch relevant sales data required by the AI agent.

- **Nodes Involved:**  
  - outreachSheet  
  - opportunitySheet

- **Node Details:**

  - **outreachSheet**  
    - Type: `n8n-nodes-base.googleSheetsTool`  
    - Role: Looks up data in the “outreach automation” sheet for lead follow-up, proposal links, outreach status.  
    - Configuration:  
      - Document ID: Google Sheet with ID `17rcNd_ZpUQLm0uWEVbD-NY6GyFUkrD4BglvawlyBygM`  
      - Sheet Name: `46113423` (outreach automation tab)  
      - Credentials: Google Sheets OAuth2 (automations@techdome.ai)  
    - Input: Receives tool call JSON from AI Sales CRM Router specifying search column/value  
    - Output: Lookup results passed back to AI Sales CRM Router  
    - Edge Cases: OAuth token expiry, sheet access permissions, empty search results.

  - **opportunitySheet**  
    - Type: `n8n-nodes-base.googleSheetsTool`  
    - Role: Queries “ghl database” sheet for opportunity and CRM deal data.  
    - Configuration:  
      - Document ID: Same as outreachSheet  
      - Sheet Name: `1423466972` (ghl database tab)  
      - Credentials: Same as outreachSheet  
    - Input: Receives tool call JSON from AI Sales CRM Router  
    - Output: Lookup results to AI Sales CRM Router  
    - Edge Cases: Same as outreachSheet.

#### 1.4 Conversation Memory

- **Overview:**  
  Enables the chatbot to remember previous messages and AI responses during a session to maintain context.

- **Nodes Involved:**  
  - Conversation Memory Buffer

- **Node Details:**  
  - See above in 1.2 AI Reasoning & Routing.

#### 1.5 Output Validation & Logging

- **Overview:**  
  Validates if the AI produced a valid output. If valid, logs conversation memory to Google Sheets; if invalid, logs the error record for review.

- **Nodes Involved:**  
  - Validate AI Output Payload  
  - Write Chat Memory to Google Sheets  
  - Log Invalid Chat Records to Google Sheets

- **Node Details:**

  - **Validate AI Output Payload**  
    - Type: `n8n-nodes-base.if`  
    - Role: Checks that the `output` field from AI is not empty.  
    - Configuration:  
      - Condition: output field must be non-empty string.  
    - Inputs: From AI Sales CRM Router  
    - Outputs:  
      - True: to "Write Chat Memory to Google Sheets"  
      - False: to "Log Invalid Chat Records to Google Sheets"  
    - Edge Cases: Missing or malformed AI response payloads.  

  - **Write Chat Memory to Google Sheets**  
    - Type: `n8n-nodes-base.googleSheets`  
    - Role: Appends conversation data to “chat_memory” sheet for logging.  
    - Configuration:  
      - Document ID: Same Google Sheet as data lookups  
      - Sheet Name: `1064505346` ("chat_memory" tab)  
      - Columns: SessionID, Timestamp (ISO string), UserMessage, AssistantResponse  
      - Operation: Append  
      - Credentials: Google Sheets OAuth2  
    - Inputs: Valid AI output and chat context from validate node  
    - Outputs: Next node is "Send Chat Response"  
    - Edge Cases: Sheet locking, quota limits, data format errors.  

  - **Log Invalid Chat Records to Google Sheets**  
    - Type: `n8n-nodes-base.googleSheets`  
    - Role: Logs incomplete or invalid AI outputs for debugging.  
    - Configuration:  
      - Document ID & Sheet Name: Not configured (empty in JSON, requires setup)  
      - Operation: Append  
      - Credentials: Google Sheets OAuth2  
    - Inputs: From false branch of validate node  
    - Outputs: None  
    - Edge Cases: Missing config, permission issues.

#### 1.6 Response Delivery

- **Overview:**  
  Sends the AI-generated and validated response back to the user in the chat interface.

- **Nodes Involved:**  
  - Send Chat Response

- **Node Details:**

  - **Send Chat Response**  
    - Type: `@n8n/n8n-nodes-langchain.chat`  
    - Role: Delivers the AI’s answer in real-time to the chat UI.  
    - Configuration:  
      - Message uses expression to inject AI Sales CRM Router’s `output` JSON text.  
      - No waiting for user reply (one-way response).  
    - Inputs: From "Write Chat Memory to Google Sheets"  
    - Outputs: None (end of chat loop)  
    - Edge Cases: Delivery failure, message formatting errors.

---

### 3. Summary Table

| Node Name                        | Node Type                                    | Functional Role                               | Input Node(s)                      | Output Node(s)                        | Sticky Note                                                                                                                                                                                         |
|---------------------------------|----------------------------------------------|-----------------------------------------------|----------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received       | @n8n/n8n-nodes-langchain.chatTrigger         | Trigger start on new chat message              | External chat interface          | AI Sales CRM Router                 | ## ▶️ Chat Trigger — “When chat message received”  \nStarts whenever a user sends a message via the n8n Chat interface.  \nThe input (`chatInput`) contains user query text and session ID.           |
| AI Sales CRM Router              | @n8n/n8n-nodes-langchain.agent                | Core AI logic, routes queries, calls tools     | When chat message received, Conversation Memory Buffer, OpenAI Chat Model, outreachSheet, opportunitySheet | Validate AI Output Payload           | ## 🤖 AI Sales CRM Router  \nActs as the core logic hub for the chatbot.  \nResponsibilities:\n- Understands user intent  \n- Decides when to query Google Sheets tools  \n- Builds JSON tool calls automatically  \n- Generates final summarized responses  \n\n### Key Rules:\n- Never guess data; always lookup first  \n- Uses structured JSON tool calls like:\n  {\n    \"tool\": \"<toolName>\",\n    \"input\": {\n      \"searchColumn\": \"<column>\",\n      \"searchValue\": \"<value>\"\n    }\n  }\n" |
| OpenAI Chat Model                | @n8n/n8n-nodes-langchain.lmChatOpenAi         | Provides GPT-4o-mini AI model                   | AI Sales CRM Router             | AI Sales CRM Router (agent)         | ## 🧠 OpenAI Chat Model (GPT-4o-mini)  \nProvides the conversational intelligence for the chatbot.  \nUsed by the “AI Sales CRM Router” for reasoning and response creation.                          |
| outreachSheet                   | n8n-nodes-base.googleSheetsTool               | Lookup outreach data in Google Sheets           | AI Sales CRM Router (as tool)    | AI Sales CRM Router (tool response) | ## 📋 Outreach Sheet Tool  \nConnects to “outreach automation” sheet in Google Sheets.  \nUsed for lead follow-up, proposal link, and outreach status lookups.  \nCalled automatically by AI when user mentions leads, emails, or outreach progress. |
| opportunitySheet               | n8n-nodes-base.googleSheetsTool               | Lookup opportunity data in Google Sheets        | AI Sales CRM Router (as tool)    | AI Sales CRM Router (tool response) | ## 💼 Opportunity Sheet Tool  \nConnects to “ghl database” sheet for opportunity and CRM deal data.  \nUsed for company-level queries like pipeline stage, deal value, or assigned rep.               |
| Conversation Memory Buffer      | @n8n/n8n-nodes-langchain.memoryBufferWindow   | Maintains last 7 turns of chat context           | When chat message received       | AI Sales CRM Router                 | ## 🧠 Conversation Memory Buffer  \nStores short-term conversation context (last 7 turns).  \nAllows multi-turn chat: the bot remembers previous user messages in the same session.  \nKey for natural, contextual replies. |
| Validate AI Output Payload      | n8n-nodes-base.if                             | Checks AI output is not empty                    | AI Sales CRM Router             | Write Chat Memory to Google Sheets (if true), Log Invalid Chat Records (if false) | ## ✅ Validate AI Output Payload  \nChecks if the AI successfully produced a response (`output` field not empty).  \n✅ If yes → Writes to memory sheet and responds  \n❌ If no → Logs invalid interaction for review. |
| Write Chat Memory to Google Sheets | n8n-nodes-base.googleSheets                  | Logs valid chat exchanges to “chat_memory” sheet | Validate AI Output Payload (true) | Send Chat Response                 | ## 🗂️ Write Chat Memory to Google Sheets  \nAppends every conversation turn (timestamp, session ID, user message, assistant response) into the “chat_memory” sheet.  \nHelps in tracking conversation logs and fine-tuning the chatbot. |
| Log Invalid Chat Records to Google Sheets | n8n-nodes-base.googleSheets                  | Logs invalid AI outputs for debugging            | Validate AI Output Payload (false) | None                            | ## ⚠️ Log Invalid Chat Records  \nSaves interactions where AI output is missing or malformed.  \nUseful for debugging tool-call or API response issues.                                            |
| Send Chat Response              | @n8n/n8n-nodes-langchain.chat                  | Sends AI-generated response back to user        | Write Chat Memory to Google Sheets | None                            | ## 💬 Send Chat Response  \nDelivers AI’s generated response back to the user in the chat interface.  \nCompletes the interaction loop in real-time.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Chat Trigger node:**
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Name: "When chat message received"
   - Configure webhook to receive chat messages from n8n Chat interface.
   - Set `responseMode` to `"responseNodes"`.

3. **Add the OpenAI Chat Model node:**
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
   - Name: "OpenAI Chat Model"
   - Model: Select or specify "gpt-4o-mini"
   - Credentials: Set up OpenAI API credentials (OpenAi account 2)
   - Leave options default.

4. **Add Google Sheets Tool nodes:**

   - **outreachSheet:**
     - Type: `n8n-nodes-base.googleSheetsTool`
     - Name: "outreachSheet"
     - Document ID: `17rcNd_ZpUQLm0uWEVbD-NY6GyFUkrD4BglvawlyBygM`
     - Sheet Name/Tab: `46113423` (corresponding to "outreach automation")
     - Credentials: Google Sheets OAuth2 for automations@techdome.ai

   - **opportunitySheet:**
     - Type: `n8n-nodes-base.googleSheetsTool`
     - Name: "opportunitySheet"
     - Document ID: Same as above
     - Sheet Name/Tab: `1423466972` (corresponding to "ghl database")
     - Credentials: Same as above

5. **Add the AI Sales CRM Router node:**
   - Type: `@n8n/n8n-nodes-langchain.agent`
   - Name: "AI Sales CRM Router"
   - Parameters:
     - System Message:  
       ```
       You are an AI Sales CRM Agent inside n8n.

       You have access to the following tools:
       1. outreachSheet – lookup in Outreach Automation sheet.
       2. opportunitySheet – lookup in GHL Opportunities sheet.

       RULES FOR TOOL USE:
       - If user mentions email, company name, contact name, status, lead, opportunity, or proposal:
         YOU MUST call the appropriate tool BEFORE answering.

       TOOL CALL FORMAT:
       Respond ONLY using this JSON format when calling a tool:

       {
         "tool": "<toolName>",
         "input": {
           "searchColumn": "<column>",
           "searchValue": "<value>"
         }
       }

       Never guess data. Always lookup first.
       If no match is found, ask for more details.
       After a tool returns data, summarize it for the user.
       ```
     - Text prompt:  
       ```
       You are interacting with a sales assistant chatbot.

       User Query:{{ $json.chatInput }}

       Your tasks:
       1. Understand the user’s question.
       2. Use the Google Sheets tool to search for relevant data:
          - Lead Status
          - Contact Details
          - Company Details
          - Outreach Status
          - Proposal Links
          - GHL Opportunity Details
       3. Use only the data returned from the sheet lookups.
       4. If data is missing, ask the user for more information.
       5. Give a clear and helpful answer based strictly on available data.
       ```
   - Connect this node to receive inputs from:
     - "When chat message received" (main input)
     - "OpenAI Chat Model" (LLM)
     - "Conversation Memory Buffer" (memory context)
     - "outreachSheet" and "opportunitySheet" (as tools)

6. **Add Conversation Memory Buffer node:**
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
   - Name: "Conversation Memory Buffer"
   - Parameter: `contextWindowLength` = 7 (retain last 7 conversation turns)
   - Connect input from "When chat message received"
   - Connect output to "AI Sales CRM Router" (memory input)

7. **Add Validate AI Output Payload node:**
   - Type: `n8n-nodes-base.if`
   - Name: "Validate AI Output Payload"
   - Condition: Check if `{{$json.output}}` is a non-empty string (string operation: notEmpty)
   - Connect input from "AI Sales CRM Router"
   - True output → "Write Chat Memory to Google Sheets"
   - False output → "Log Invalid Chat Records to Google Sheets"

8. **Add Write Chat Memory to Google Sheets node:**
   - Type: `n8n-nodes-base.googleSheets`
   - Name: "Write Chat Memory to Google Sheets"
   - Document ID: Same as above Google Sheets
   - Sheet Name: `1064505346` ("chat_memory" tab)
   - Operation: Append
   - Columns mapping:  
     - SessionID = `{{$('When chat message received').item.json.sessionId}}`  
     - Timestamp = `{{ new Date().toISOString() }}`  
     - UserMessage = `{{$('When chat message received').item.json.chatInput}}`  
     - AssistantResponse = `{{$json.output}}`
   - Credentials: Google Sheets OAuth2
   - Connect input from Validate AI Output Payload (true branch)
   - Output to "Send Chat Response"

9. **Add Log Invalid Chat Records to Google Sheets node:**
   - Type: `n8n-nodes-base.googleSheets`
   - Name: "Log Invalid Chat Records to Google Sheets"
   - Document ID: (Must configure a Google Sheet to log failed AI outputs)  
   - Sheet Name: (Configure appropriately)
   - Operation: Append
   - Credentials: Google Sheets OAuth2
   - Connect input from Validate AI Output Payload (false branch)

10. **Add Send Chat Response node:**
    - Type: `@n8n/n8n-nodes-langchain.chat`
    - Name: "Send Chat Response"
    - Message: Set expression to `{{$("AI Sales CRM Router").item.json.output}}`
    - Options: Wait user reply = false
    - Connect input from "Write Chat Memory to Google Sheets"

11. **Connect nodes as follows:**

    - "When chat message received" → "AI Sales CRM Router"  
    - "Conversation Memory Buffer" input ← "When chat message received"  
    - "Conversation Memory Buffer" output → "AI Sales CRM Router" (memory input)  
    - "OpenAI Chat Model" → "AI Sales CRM Router" (LLM input)  
    - "AI Sales CRM Router" → "Validate AI Output Payload"  
    - "Validate AI Output Payload" true → "Write Chat Memory to Google Sheets" → "Send Chat Response"  
    - "Validate AI Output Payload" false → "Log Invalid Chat Records to Google Sheets"  
    - "AI Sales CRM Router" (tool calls) → "outreachSheet" and "opportunitySheet" → back to "AI Sales CRM Router"

12. **Credentials Setup:**

    - OpenAI API key configured under OpenAi account 2 credentials.  
    - Google Sheets OAuth2 credentials configured for automations@techdome.ai user, with access to the referenced Google Sheets document.

13. **Test the flow:**

    - Send chat messages through n8n’s chat interface.  
    - Verify AI responses contain accurate sales CRM data from Google Sheets.  
    - Check that conversation memory helps maintain context.  
    - Confirm chat logs appear in "chat_memory" Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                 | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow is branded as **AI-Powered Sales CRM Chatbot with Google Sheets Lookup & Memory (GPT-4o-mini)**. It acts as an intelligent assistant for lead status, proposals, and opportunity lookups.           | Sticky Note on main workflow canvas                                                                      |
| Workflow highlights include dynamic Google Sheets lookups, multi-turn memory, and logging all chat turns to a dedicated sheet for monitoring and training purposes.                                         | Sticky Note summary                                                                                       |
| Tools and integrations used: GPT-4o-mini for AI, Google Sheets Tools for data access, Memory Buffer for context retention, n8n Chat UI for interface, Google Sheets for historical logging.                   | Sticky Note summary                                                                                       |
| The AI Sales CRM Router enforces strict rules to never guess data and always perform lookups first, using structured JSON calls to Google Sheets tools.                                                     | Sticky Note on AI Sales CRM Router node                                                                  |
| The Google Sheets document ID and sheet tabs must be accessible and permissions granted to the OAuth2 account used by n8n to ensure successful lookups and logging.                                         | Credential and access note                                                                                 |
| The Log Invalid Chat Records node requires configuration with a valid Google Sheet and sheet name to capture malformed AI responses for debugging.                                                          | Important setup note for error logging                                                                    |
| The conversation memory buffer length is set to 7 turns to balance context retention and performance.                                                                                                        | Memory Buffer sticky note                                                                                  |
| Chat responses are sent back via n8n’s chat interface node without waiting for user reply, enabling smooth conversational flow.                                                                             | Send Chat Response sticky note                                                                             |

---

This document fully describes the workflow’s structure, node roles, configurations, and how to reproduce it manually in n8n, along with important operational notes and potential failure points.