Hotel Receptionist with WhatsApp, Gemini Model-Switching, Redis & Google Sheets

https://n8nworkflows.xyz/workflows/hotel-receptionist-with-whatsapp--gemini-model-switching--redis---google-sheets-10214


# Hotel Receptionist with WhatsApp, Gemini Model-Switching, Redis & Google Sheets

### 1. Workflow Overview

This workflow implements an AI-powered hotel receptionist system that interacts with guests via WhatsApp. It enables guests to send inquiries about hotel bookings, room availability, pricing, and other services through WhatsApp messages. The system automatically responds using AI to generate accurate, real-time answers based on live hotel data stored in a MySQL database and pricing details from Google Sheets.

The core logic is organized into these functional blocks:

- **1.1 Input Reception**: Receiving and validating incoming WhatsApp messages.
- **1.2 User Model Retrieval & Assignment**: Using Redis to track and assign AI language models per user for load balancing and cost optimization.
- **1.3 AI Query Processing**: Converting user questions into read-only SQL queries, executing them, and generating natural language responses via selected AI models.
- **1.4 Pricing Data Integration**: Supplementing AI answers with live pricing data from Google Sheets.
- **1.5 Response Delivery**: Sending the AI-generated response back to the guest through WhatsApp.

This design ensures a seamless loop: WhatsApp message → AI processing with database & spreadsheet integration → WhatsApp reply, with intelligent model switching for scalability.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives incoming WhatsApp messages, filters out non-text messages, and initiates user lookup.

**Nodes Involved:**  
- WhatsApp Trigger  
- Check Message  
- Check User Number

**Node Details:**

- **WhatsApp Trigger**  
  - *Type*: WhatsApp Trigger node  
  - *Role*: Listens for new WhatsApp messages from guests.  
  - *Config*: Webhook set up with WhatsApp OAuth credentials; triggers on message updates.  
  - *Inputs/Outputs*: Output to Check Message node.  
  - *Edge Cases*: Missing or unsupported message types (only text supported downstream).  
  - *Notes*: Critical entry point; requires valid WhatsApp OAuth credentials.

- **Check Message**  
  - *Type*: Code node (JavaScript)  
  - *Role*: Filters incoming messages, allowing only those with text content to proceed.  
  - *Config*: Checks if `messages[0].text` exists; returns empty otherwise to stop workflow early.  
  - *Inputs/Outputs*: Input from WhatsApp Trigger; output to Check User Number.  
  - *Edge Cases*: Non-text messages stop the workflow gracefully.

- **Check User Number**  
  - *Type*: Redis node (Get operation)  
  - *Role*: Checks Redis cache for previously stored AI model assignment for the user based on WhatsApp ID.  
  - *Config*: Key format `llm-user:<wa_id>`; Redis credentials required.  
  - *Inputs/Outputs*: Input from Check Message; output to Model Decider.  
  - *Edge Cases*: Cache miss leads to default model assignment downstream. Redis connectivity issues could cause failure.

---

#### 1.2 User Model Retrieval & Assignment

**Overview:**  
Determines which AI model instance to use for the current user to balance load and cost, storing the assignment back in Redis.

**Nodes Involved:**  
- Model Decider  
- Store User Number  
- Choose Model

**Node Details:**

- **Model Decider**  
  - *Type*: Code node (JavaScript)  
  - *Role*: Parses Redis data and flips the assigned model index (0 or 1) to alternate between two AI models per user session. Defaults to model 0 if no prior record.  
  - *Config*: Custom JS logic to parse JSON and toggle model index. Sets flag to store new assignment.  
  - *Inputs/Outputs*: Input from Check User Number; output to Store User Number and Choose Model.  
  - *Edge Cases*: Malformed Redis data handled by defaulting to model 0.

- **Store User Number**  
  - *Type*: Redis node (Set operation)  
  - *Role*: Stores the chosen model index keyed by the user's WhatsApp ID into Redis with a TTL of 1 hour.  
  - *Config*: Key `llm-user:<wa_id>`, TTL 3600 seconds, value is JSON string with modelIndex.  
  - *Inputs/Outputs*: Input from Model Decider; output to AI Agent.  
  - *Edge Cases*: Redis unavailability or write failure.

- **Choose Model**  
  - *Type*: LangChain Model Selector node  
  - *Role*: Routes the request to one of two AI language model nodes based on `modelIndex`.  
  - *Config*: Rules map modelIndex 0 to one model, modelIndex 1 to another.  
  - *Inputs/Outputs*: Input from Model Decider; output to AI Agent node connected to the selected model.  
  - *Edge Cases*: Unexpected modelIndex values default handling.

---

#### 1.3 AI Query Processing

**Overview:**  
Processes the user's question into a SQL SELECT query through an AI agent, executes the query on MySQL, and prepares the response using memory context and AI language models.

**Nodes Involved:**  
- AI Agent  
- Execute a SQL query in MySQL  
- Simple Memory  
- Google Gemini Chat Model  
- Google Gemini Chat Model1

**Node Details:**

- **AI Agent**  
  - *Type*: LangChain Agent node  
  - *Role*: Receives user’s text message, converts it into a read-only SQL SELECT query per strict security rules.  
  - *Config*: System message instructs agent to ONLY generate SELECT queries, forbidding any write operations.  
  - *Inputs/Outputs*: Input text from WhatsApp Trigger messages; output is SQL query to Execute SQL node and final response to Send message node.  
  - *Edge Cases*: Misinterpretation of input could lead to invalid SQL; strict prompt reduces risk.  
  - *Notes*: Uses session-based memory from Simple Memory node.

- **Execute a SQL query in MySQL**  
  - *Type*: MySQL node  
  - *Role*: Executes the AI-generated SQL SELECT query on the hotel’s MySQL database.  
  - *Config*: Query parameter dynamically set from AI Agent output.  
  - *Inputs/Outputs*: Input SQL query from AI Agent; output results back to AI Agent for formatting.  
  - *Edge Cases*: SQL syntax errors, database connectivity failures.

- **Simple Memory**  
  - *Type*: LangChain Memory Buffer Window  
  - *Role*: Maintains conversational context for each user session keyed by WhatsApp ID to improve AI responses.  
  - *Config*: Session key = user’s WhatsApp ID.  
  - *Inputs/Outputs*: Linked to AI Agent node for memory management.  
  - *Edge Cases*: Memory overflow or session key mismatches.

- **Google Gemini Chat Model** & **Google Gemini Chat Model1**  
  - *Type*: LangChain LM Chat Google Gemini nodes  
  - *Role*: Two separate Google Gemini language models used alternately via model switching logic.  
  - *Config*: Each has separate Google Palm API credentials.  
  - *Inputs/Outputs*: Input routed from Choose Model node; output sent to AI Agent.  
  - *Edge Cases*: API limits, auth failures, response latency.

---

#### 1.4 Pricing Data Integration

**Overview:**  
Fetches live pricing data from a Google Sheet to enrich AI responses with up-to-date hotel pricing information.

**Nodes Involved:**  
- Pricing

**Node Details:**

- **Pricing**  
  - *Type*: Google Sheets Tool node  
  - *Role*: Reads the pricing sheet (Sheet1, gid=0) from a specific Google Sheets document.  
  - *Config*: Uses Google OAuth credentials to access the spreadsheet by document ID.  
  - *Inputs/Outputs*: Connected as an AI tool input to AI Agent for pricing data retrieval.  
  - *Edge Cases*: OAuth token expiration, sheet access permissions, network issues.

---

#### 1.5 Response Delivery

**Overview:**  
Sends the AI-generated natural language response back to the guest via WhatsApp.

**Nodes Involved:**  
- Send message

**Node Details:**

- **Send message**  
  - *Type*: WhatsApp node  
  - *Role*: Sends a text message to the guest’s WhatsApp number containing the AI agent’s response.  
  - *Config*: Uses WhatsApp API credentials; recipient number dynamically extracted from incoming message data; message body set to AI output.  
  - *Inputs/Outputs*: Input from AI Agent node; output ends workflow.  
  - *Edge Cases*: WhatsApp API rate limits, invalid recipient numbers, message send failures.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                                     | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                 |
|-------------------------|--------------------------------------|----------------------------------------------------|------------------------|--------------------------|---------------------------------------------------------------------------------------------|
| WhatsApp Trigger         | WhatsApp Trigger                     | Entry point: receives WhatsApp messages            | —                      | Check Message             |                                                                                             |
| Check Message            | Code                                | Filters non-text messages                           | WhatsApp Trigger         | Check User Number         |                                                                                             |
| Check User Number        | Redis (Get)                        | Retrieves user’s model assignment from Redis       | Check Message            | Model Decider             | Redis Get Node: Checks Redis for user’s assigned AI model by WhatsApp ID                     |
| Model Decider            | Code                                | Decides and toggles user’s AI model assignment     | Check User Number        | Store User Number, Choose Model | ⚙️ Model Switching System: Uses Redis to distribute user load between AI models         |
| Store User Number        | Redis (Set)                        | Stores user’s AI model assignment with TTL         | Model Decider            | AI Agent                  | Redis Set Node: Saves assigned model index with 1-hour TTL                                  |
| Choose Model             | LangChain Model Selector            | Routes request to proper AI language model          | Model Decider            | Google Gemini Chat Model, Google Gemini Chat Model1 |                                                                                     |
| AI Agent                 | LangChain Agent                    | Converts text to SQL, interprets user query         | Store User Number        | Send message              | 🧠 AI-Powered Hotel Assistant: Generates read-only SQL and natural language answers          |
| Execute a SQL query in MySQL | MySQL Tool                       | Executes AI-generated SQL SELECT queries             | AI Agent (ai_tool)       | AI Agent (results)        |                                                                                             |
| Simple Memory            | LangChain Memory Buffer Window     | Maintains session memory keyed by WhatsApp ID       | —                        | AI Agent                  |                                                                                             |
| Google Gemini Chat Model | LangChain LM Chat (Google Gemini) | AI language model instance 1                          | Choose Model             | Choose Model              |                                                                                             |
| Google Gemini Chat Model1| LangChain LM Chat (Google Gemini) | AI language model instance 2                          | Choose Model             | Choose Model              |                                                                                             |
| Pricing                 | Google Sheets Tool                  | Retrieves live pricing data from Google Sheets       | AI Agent (ai_tool)       | AI Agent                  |                                                                                             |
| Send message             | WhatsApp                           | Sends AI response back to user on WhatsApp          | AI Agent                 | —                        |                                                                                             |
| Sticky Note              | Sticky Note                       | Workflow overview                                   | —                        | —                        | 💬 Workflow Overview: AI receptionist combining WhatsApp → AI → DB → WhatsApp loop           |
| Sticky Note1             | Sticky Note                       | Model switching explanation                         | —                        | —                        | ⚙️ Model Switching System: Redis-based model assignment for load balancing                   |
| Sticky Note2             | Sticky Note                       | AI agent role description                           | —                        | —                        | 🧠 AI-Powered Hotel Assistant: Safe, read-only SQL generation with real-time hotel data      |
| Sticky Note3             | Sticky Note                       | Redis Get explanation                               | —                        | —                        | Redis Get Node: Retrieves user’s model assignment from Redis                                |
| Sticky Note4             | Sticky Note                       | Redis Set explanation                               | —                        | —                        | Redis Set Node: Stores user model assignment with expiration                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure webhook with WhatsApp OAuth2 credentials.  
   - Set to trigger on new message updates.

2. **Add Code Node "Check Message"**  
   - Type: Code  
   - JavaScript: Check if incoming message contains text, else return empty to stop workflow.  
   ```js
   const msg = $json.messages?.[0]?.text;
   if (!msg) return [];
   return items;
   ```  
   - Connect WhatsApp Trigger output to this node.

3. **Add Redis Node "Check User Number" (Get)**  
   - Type: Redis  
   - Operation: Get  
   - Key: `llm-user:{{$json.contacts[0].wa_id}}`  
   - Connect output of Check Message to this node.  
   - Use Redis credentials.

4. **Add Code Node "Model Decider"**  
   - Type: Code  
   - JavaScript to parse Redis result, toggle model index between 0 and 1, default 0.  
   - Outputs JSON with `modelIndex` and `shouldSet` flags.  
   - Connect Check User Number output to this node.

5. **Add Redis Node "Store User Number" (Set)**  
   - Type: Redis  
   - Operation: Set  
   - Key: `llm-user:{{ $('WhatsApp Trigger').item.json.contacts[0].wa_id }}`  
   - Value: JSON string `{"modelIndex": modelIndex}` from Model Decider output.  
   - TTL: 3600 seconds (1 hour)  
   - Connect Model Decider output to this node.

6. **Add LangChain Model Selector Node "Choose Model"**  
   - Type: LangChain Model Selector  
   - Rules:  
     - If `modelIndex == 0` → route to Google Gemini Chat Model node  
     - If `modelIndex == 1` → route to Google Gemini Chat Model1 node  
   - Connect Model Decider output to this node.

7. **Add Two LangChain Google Gemini Chat Model Nodes**  
   - Name one "Google Gemini Chat Model", the other "Google Gemini Chat Model1"  
   - Assign separate Google Palm API credentials for each.  
   - Connect Choose Model outputs accordingly.

8. **Add LangChain Agent Node "AI Agent"**  
   - Type: LangChain Agent  
   - Text input expression: `{{$('WhatsApp Trigger').item.json.messages[0].text.body}}`  
   - System Message:  
     ```
     You are a helpful AI assistant tasked with answering questions about hotel bookings.
     You have access to a MySQL database with tables like 'bookings', 'guests', 'rooms', etc.
     IMPORTANT SECURITY RULE: YOU ARE STRICTLY FORBIDDEN FROM PERFORMING ANY DATABASE WRITE OPERATIONS (INSERT, UPDATE, DELETE, CREATE, ALTER, DROP, etc.).
     You must ONLY generate valid SQL SELECT statements.
     When a user asks a question, translate it into an appropriate SQL SELECT query.
     ```  
   - Connect output of Store User Number node to AI Agent.  
   - Configure AI Agent to use Simple Memory node (see next step).

9. **Add LangChain Memory Buffer Window Node "Simple Memory"**  
   - Session Key: `{{$('WhatsApp Trigger').item.json.contacts[0].wa_id}}`  
   - Connect to AI Agent node as memory source.

10. **Add MySQL Node "Execute a SQL query in MySQL"**  
    - Operation: Execute Query  
    - Query: `{{$('AI Agent').item.json.query}}`  
    - Use MySQL credentials with access to hotel booking database.  
    - Connect AI Agent output (SQL query) to this node as ai_tool input.  
    - Connect MySQL node output back to AI Agent for processing results.

11. **Add Google Sheets Tool Node "Pricing"**  
    - Document ID: ID of hotel pricing Google Sheet  
    - Sheet Name: Sheet1 (gid=0)  
    - Use Google Sheets OAuth2 credentials.  
    - Connect as ai_tool input to AI Agent node for pricing data retrieval.

12. **Connect AI Agent output to WhatsApp Node "Send message"**  
    - Type: WhatsApp  
    - Operation: Send  
    - Recipient Phone Number: `{{$('WhatsApp Trigger').item.json.messages[0].from}}`  
    - Text Body: `{{$json.output}}` (AI-generated response)  
    - Use WhatsApp API credentials.  

13. **Add Sticky Notes for Documentation** (optional)  
    - Add sticky notes describing workflow overview, model switching system, Redis Get/Set explanations, and AI agent role.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| 💬 Workflow Overview: AI receptionist combining WhatsApp → AI Agent → Database → WhatsApp seamless loop.                         | Workflow purpose summary                                                                                     |
| ⚙️ Model Switching System: Redis-based user AI model assignment to balance load and reduce costs, ideal for large-scale systems.  | Model Decider, Redis nodes explanation                                                                       |
| 🧠 AI-Powered Hotel Assistant: Strictly read-only SQL generation by AI with real-time hotel data ensures safe, accurate answers. | AI Agent node prompt and role                                                                                 |
| Redis Get Node: Retrieves user’s model assignment by WhatsApp ID to maintain session consistency.                                 | Check User Number node                                                                                         |
| Redis Set Node: Stores user’s assigned model with TTL = 3600 seconds (1 hour), enabling model alternation.                        | Store User Number node                                                                                        |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.