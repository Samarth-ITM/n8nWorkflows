Chat Assistant (OpenAI assistant) with Postgres Memory And API Calling Capabalities

https://n8nworkflows.xyz/workflows/chat-assistant--openai-assistant--with-postgres-memory-and-api-calling-capabalities-2637


# Chat Assistant (OpenAI assistant) with Postgres Memory And API Calling Capabalities

### 1. Workflow Overview

This workflow implements an intelligent chatbot assistant using OpenAI, integrated with WhatsApp Business as the input channel and supported by persistent conversation memory via Postgres. It is designed primarily for sales and customer support scenarios, particularly in the health insurance domain, but can be adapted for other industries.

The workflow is logically divided into the following blocks:

- **1.1 Chat Input Reception:** Captures incoming customer messages from WhatsApp Business via the Chat Trigger node and extracts session information.
  
- **1.2 Conditional Data Check:** Determines if additional customer metadata (e.g., name, age, city) accompanies the message, influencing how the prompt is prepared.
  
- **1.3 Data Preparation:** Formats and compiles customer data and queries into a structured prompt for the AI assistant.
  
- **1.4 AI Processing & Memory:** Sends the prompt to the OpenAI assistant, maintaining conversation context using Postgres-based chat memory nodes to ensure continuity.
  
- **1.5 External API and Database Integration:** Performs calls to external HTTP APIs and queries a MySQL database to enrich responses with product and knowledge base data.
  
- **1.6 Response Delivery:** Processes AI output and sends the response back through WhatsApp Business (implicitly handled via Chat Trigger webhook).

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Input Reception

- **Overview:**  
  Captures incoming chat messages from WhatsApp Business customers, initializes session tracking, and starts the workflow.

- **Nodes Involved:**  
  - Chat Trigger

- **Node Details:**  
  - **Chat Trigger**  
    - Type: Chat Trigger (Langchain integration)  
    - Configuration: Public webhook, initial greeting message ("Olá 👋...")  
    - Key variables: `chatInput` (user message), `session_id` (session identifier)  
    - Inputs: External WhatsApp messages via webhook  
    - Outputs: Data to If node  
    - Edge cases: Missing or malformed session_id, webhook downtime  
    - Version: 1

---

#### 2.2 Conditional Data Check

- **Overview:**  
  Checks if incoming messages contain additional customer data (`leadData`). This condition influences prompt construction to enrich AI context.

- **Nodes Involved:**  
  - If

- **Node Details:**  
  - **If**  
    - Type: If node (conditional branch)  
    - Condition: Checks if JSON path `leadData` exists and is not empty  
    - Inputs: Chat Trigger output  
    - Outputs: Two branches — true (has leadData) and false (no leadData)  
    - Edge cases: Missing `leadData` property, empty objects, expression evaluation errors  
    - Version: 2

---

#### 2.3 Data Preparation

- **Overview:**  
  Prepares the chat prompt string to be sent to OpenAI. If additional customer data is present, composes a detailed introduction message to inform the assistant without requiring a response. Otherwise, passes the raw chat input forward.

- **Nodes Involved:**  
  - Edit Fields1 (for enriched prompt)  
  - Edit Fields2 (for raw prompt)

- **Node Details:**  
  - **Edit Fields1**  
    - Type: Set node  
    - Configuration: Constructs a multi-line string incorporating `leadData` fields (name, age, city, profession, education, entity, deviceType, channel, quotationType)  
    - Sets also `session_id` for downstream use  
    - Inputs: If node (true branch)  
    - Outputs: To OpenAI node (assistant)  
    - Expressions: Template strings referencing `$json.leadData.*` and `$json.session_id`  
    - Failure modes: Missing fields in `leadData` causing incomplete prompt  
    - Version: 3.4

  - **Edit Fields2**  
    - Type: Set node  
    - Configuration: Passes through the raw `chatInput` and `session_id` from Chat Trigger  
    - Inputs: OpenAI node (from false branch of If)  
    - Outputs: To OpenAI2 node  
    - Version: 3.4

---

#### 2.4 AI Processing and Memory

- **Overview:**  
  Sends the prepared prompt to OpenAI assistant nodes, which generate responses. Utilizes PostgreSQL chat memory nodes to maintain conversational context across sessions.

- **Nodes Involved:**  
  - OpenAI  
  - OpenAI2  
  - Postgres Chat Memory  
  - Postgres Chat Memory1

- **Node Details:**  
  - **OpenAI**  
    - Type: OpenAI Assistant node (Langchain integration)  
    - Configuration: Uses assistant ID `asst_numdCoMZPQ6GwfiJg5drg9hr` tailored for sales/support use cases  
    - Credentials: OpenAI API key  
    - Inputs: From Edit Fields1 (detailed prompt)  
    - Outputs: To Edit Fields2 (passes to OpenAI2)  
    - Edge cases: API authentication errors, rate limits, malformed prompts  
    - Version: 1.4

  - **OpenAI2**  
    - Type: OpenAI Assistant node  
    - Configuration: Uses assistant ID `asst_x2qfc7EuoPv7XGOL84ClEZ3L` (possibly for summarization or further processing)  
    - Credentials: OpenAI API key  
    - Inputs: From Edit Fields2 (raw prompt) and from multiple AI tool nodes (External API, Knowledge Base, Products in Database) and memory nodes  
    - Outputs: Final AI-generated text  
    - Edge cases: Same as OpenAI node, plus potential input format inconsistencies  
    - Version: 1.4

  - **Postgres Chat Memory** (two instances)  
    - Type: Langchain Postgres memory nodes  
    - Configuration: Stores conversation messages in `aimessages` table, keying by session concatenation (`session_id` + internal sessionId)  
    - Context window length: 30 for one node (longer context), 1 for the other (short-term context)  
    - Credentials: Postgres DB connection  
    - Inputs: AI nodes use this memory to provide context for assistant replies  
    - Edge cases: DB connection failures, query timeouts, schema mismatches  
    - Version: 1.3

---

#### 2.5 External API and Database Integration

- **Overview:**  
  Enhances AI responses by querying external APIs and databases, providing real-time data such as product listings and knowledge base information.

- **Nodes Involved:**  
  - Products in Daatabase (MySQL query)  
  - Knowledge Base (HTTP request)  
  - External API (HTTP POST request)

- **Node Details:**  
  - **Products in Daatabase**  
    - Type: MySQL Tool node  
    - Configuration: Executes complex SQL query filtering products by city, state, modality, holder age, and dependents count; returns top 3 matching products sorted by price and creation date  
    - Inputs: AI tool inputs with dynamic parameters parsed from AI responses (`$fromAI(...)`)  
    - Credentials: MySQL database  
    - Edge cases: SQL injection if inputs not sanitized, DB connection errors, empty result sets  
    - Version: 2.4

  - **Knowledge Base**  
    - Type: HTTP Request node (Langchain tool)  
    - Configuration: GET request to a knowledge base URL with placeholders for modality, state, city, operator  
    - Inputs: AI tool inputs, parameters interpolated dynamically  
    - Edge cases: Endpoint downtime, invalid parameter replacement, HTTP errors  
    - Version: 1.1

  - **External API**  
    - Type: HTTP Request node (Langchain tool)  
    - Configuration: POST JSON request to external API sending user name and birthdate in camel case and ISO date format  
    - Inputs: AI tool inputs with JSON body constructed with expressions  
    - Edge cases: API authentication, malformed JSON, network failures  
    - Version: 1.1

---

#### 2.6 Response Delivery

- **Overview:**  
  The AI-generated response, enriched by external data and memory context, is returned via the OpenAI2 node output, which connects back to the Chat Trigger webhook response, thus delivering the message to the customer on WhatsApp Business.

- **Nodes Involved:**  
  - OpenAI2  
  - Chat Trigger (implicitly for message response)

- **Node Details:**  
  - Responses and final message formatting occur in OpenAI2 node outputs, which n8n returns through the webhook response.  
  - Edge cases: Delayed responses, webhook timeout, message formatting errors.

---

### 3. Summary Table

| Node Name           | Node Type                     | Functional Role                            | Input Node(s)           | Output Node(s)          | Sticky Note                   |
|---------------------|-------------------------------|-------------------------------------------|-------------------------|-------------------------|-------------------------------|
| Chat Trigger        | Chat Trigger (Langchain)       | Receive customer message & session input  | External webhook        | If                      |                               |
| If                  | If                            | Check presence of additional customer data| Chat Trigger            | Edit Fields1, OpenAI2    |                               |
| Edit Fields1        | Set                           | Prepare detailed prompt with customer data| If (true branch)        | OpenAI                  |                               |
| OpenAI              | OpenAI Assistant (Langchain)  | Generate AI response with enriched prompt | Edit Fields1            | Edit Fields2, Postgres Chat Memory1 |                               |
| Edit Fields2        | Set                           | Prepare prompt pass-through for raw input | OpenAI                  | OpenAI2                 |                               |
| OpenAI2             | OpenAI Assistant (Langchain)  | Final AI response processing & output     | Edit Fields2, External API, Knowledge Base, Products in Daatabase, Postgres Chat Memory | Chat Trigger (response) |                               |
| Postgres Chat Memory| Postgres Chat Memory (Langchain)| Store and retrieve conversation history   | -                       | OpenAI2                 |                               |
| Postgres Chat Memory1| Postgres Chat Memory (Langchain)| Store short-term conversation context      | -                       | OpenAI                  |                               |
| Products in Daatabase| MySQL Tool                    | Query products database for insurance plans| AI tool input           | OpenAI2                 |                               |
| Knowledge Base      | HTTP Request (Langchain tool) | Query knowledge base for shop info         | AI tool input           | OpenAI2                 | TOOLS (Sticky Note)            |
| External API        | HTTP Request (Langchain tool) | Call external API with user info           | AI tool input           | OpenAI2                 | TOOLS (Sticky Note)            |
| Sticky Note         | Sticky Note                   | Label "TOOLS" for API/DB nodes             | -                       | -                       | TOOLS                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: Chat Trigger (Langchain)  
   - Configuration: Set public webhook, initial greeting message "Olá 👋\nSou Jovelino..."  
   - Save the webhook URL for WhatsApp Business integration.

2. **Add If Node**  
   - Connect Chat Trigger output to If node input.  
   - Condition: Check if `leadData` exists and is non-empty (`exists($json.leadData) == true`).

3. **Add Edit Fields1 Node (Detailed Prompt)**  
   - Connect If node "true" branch to Edit Fields1.  
   - Configure Set fields:  
     - `chatInput`: Multiline string incorporating customer fields from `leadData` such as name, age, city, profession, educationLevel, entity, deviceType, channel, quotationType.  
     - `session_id`: Set to `{{$json.session_id}}`.

4. **Add OpenAI Node (Assistant for Detailed Prompt)**  
   - Connect Edit Fields1 output to OpenAI node.  
   - Credentials: Configure OpenAI API with valid key.  
   - Parameters: Use assistant resource with assistant ID `asst_numdCoMZPQ6GwfiJg5drg9hr`.  
   - Input text: Use `chatInput` from previous node.

5. **Add Edit Fields2 Node (Raw Prompt Pass-through)**  
   - Connect OpenAI node output to Edit Fields2.  
   - Set fields:  
     - `chatInput`: Pass through `chatInput` from Chat Trigger node.  
     - `session_id`: Pass through `session_id` from Chat Trigger node.

6. **Add OpenAI2 Node (Final AI Assistant)**  
   - Connect Edit Fields2 output to OpenAI2 node.  
   - Credentials: Use OpenAI API credentials.  
   - Assistant ID: `asst_x2qfc7EuoPv7XGOL84ClEZ3L`.  
   - Input text: `chatInput` from Edit Fields2.

7. **Add Postgres Chat Memory Node (Long Context)**  
   - Configure with table `aimessages` for chat history.  
   - Session key: Concatenate `session_id` from Chat Trigger and internal session ID.  
   - Context window length: 30.  
   - Connect as AI memory input to OpenAI2.

8. **Add Postgres Chat Memory1 Node (Short Context)**  
   - Same configuration as above but context window length = 1.  
   - Connect as AI memory input to OpenAI node.

9. **Add Products in Daatabase Node**  
   - Type: MySQL Tool node.  
   - Credentials: Configure MySQL connection.  
   - Query: Use provided SQL template filtering products by city, state, modality, holder age/count.  
   - Connect as AI tool input to OpenAI2 node.

10. **Add Knowledge Base Node**  
    - Type: HTTP Request (Langchain tool).  
    - URL with placeholders for modality, state, city, operator.  
    - Connect as AI tool input to OpenAI2.

11. **Add External API Node**  
    - Type: HTTP Request (Langchain tool).  
    - POST JSON body with user name and birthdate from AI tool input.  
    - Connect as AI tool input to OpenAI2.

12. **Connect If node "false" branch directly to OpenAI2 node**  
    - To handle cases when no additional `leadData` is present.

13. **Set workflow execution order accordingly**  
    - Ensure Chat Trigger initiates flow, and data flows through condition, formatting, AI, memory, tools, and back to response.

14. **Test the entire flow using WhatsApp Business messages**  
    - Validate session tracking, prompt formatting, API calls, and response delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Prevent hallucinations by clearly describing every API’s purpose in the OpenAI prompt.                                       | Best practice in prompt engineering              |
| Workflow modularity allows flexible adaptation for sales and customer support use cases.                                     | Workflow design principle                         |
| Persistent context is maintained via Postgres chat memory to support continuous conversations.                              | Important for multi-turn dialogs                  |
| Include practical examples and decision flows in the assistant prompt for better AI guidance.                               | Enhances AI response quality                      |
| Log interactions extensively for future analysis and optimization.                                                          | Workflow monitoring and improvement              |
| Sticky Note “TOOLS” visually groups the API and database integration nodes, indicating their collective role.                | Workflow documentation clarity                    |
| OpenAI assistant IDs used: `asst_numdCoMZPQ6GwfiJg5drg9hr` (primary), `asst_x2qfc7EuoPv7XGOL84ClEZ3L` (secondary).            | Assistants configured in Langchain n8n integration |
| Initial greeting in Chat Trigger: “Olá 👋\nSou Jovelino, o serviço de IA do Joov...”                                          | User-friendly bot introduction                     |

---

This documentation provides a detailed understanding of the chatbot workflow, enabling developers and automation agents to comprehend, reproduce, and extend its functionality confidently.