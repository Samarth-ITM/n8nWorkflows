Create a WhatsApp Voice Assistant with Twilio, VAPI, Google Calendar & OpenAI

https://n8nworkflows.xyz/workflows/create-a-whatsapp-voice-assistant-with-twilio--vapi--google-calendar---openai-8284


# Create a WhatsApp Voice Assistant with Twilio, VAPI, Google Calendar & OpenAI

---

### 1. Workflow Overview

This workflow implements a **WhatsApp Voice Assistant** leveraging Twilio, VAPI (Voice API), Google Calendar, OpenAI, Supabase Vector Store, and Gmail. Its primary purpose is to process voice commands received via WhatsApp and execute calendar management, knowledge base queries, or email notifications accordingly. The workflow is designed for users who want to interact with their Google Calendar, knowledge repositories, or email through natural language voice commands sent via WhatsApp.

The workflow is logically divided into the following blocks:

- **1.1 Entry Point & Webhook Reception (VAPI Incoming Webhook)**: Receives voice commands routed through Twilio and VAPI, initiating the processing chain.
- **1.2 Response Dispatch (Respond to Webhook)**: Sends back the processed response to VAPI for delivery via WhatsApp.
- **1.3 MCP Servers (Multi-Channel Processing Servers)**: Three MCP servers handle specific types of requests:
  - **Calendar MCP**: Manages Google Calendar events (Create, Fetch, Update, Delete).
  - **Gmail MCP**: Sends email notifications based on voice commands.
  - **Knowledge Base MCP**: Uses OpenAI embeddings and Supabase vector store to answer general inquiries.
- **1.4 AI Processing Nodes**: OpenAI embeddings generation node and Supabase vector store node support knowledge base queries.
- **1.5 Sticky Notes**: Documentation nodes embedded within the workflow for clarity.

---

### 2. Block-by-Block Analysis

#### 2.1 Entry Point & Webhook Reception (VAPI Incoming Webhook)

**Overview:**  
This block receives incoming WhatsApp voice commands via the VAPI webhook. Twilio routes the audio stream to VAPI, which processes the voice input and forwards a structured request to this webhook in n8n to trigger further handling.

**Nodes Involved:**  
- Incoming Webhook (VAPI)  
- Respond to Webhook (VAPI)  
- Sticky Note (entry description)  
- Sticky Note (VAPI webhook explanation)  
- Sticky Note (response node explanation)

**Node Details:**  

- **Incoming Webhook (VAPI)**  
  - **Type & Role:** HTTP Webhook node; entry point for requests triggered by VAPI post voice command interpretation.  
  - **Configuration:** POST method; unique webhook path; response mode set to "responseNode" to defer response to downstream node.  
  - **Inputs:** External HTTP POST request from VAPI.  
  - **Outputs:** Connected to "Respond to Webhook (VAPI)" node.  
  - **Potential Failures:** HTTP connectivity issues; invalid/malformed requests from VAPI; webhook path misconfiguration.

- **Respond to Webhook (VAPI)**  
  - **Type & Role:** HTTP Response node; sends the final XML response back to VAPI.  
  - **Configuration:** Responds with XML content-type header; static XML body instructing VAPI to dial a SIP endpoint (`sip:15795503867@sip.vapi.ai`).  
  - **Inputs:** Receives input from "Incoming Webhook (VAPI)".  
  - **Outputs:** None (terminal node for this branch).  
  - **Potential Failures:** Failure to respond timely causing webhook timeout; incorrect XML formatting.

- **Sticky Notes**  
  - Provide contextual explanations of the incoming webhook’s role and response mechanics.

---

#### 2.2 MCP Servers (Multi-Channel Processing Servers)

**Overview:**  
This block contains three MCP server nodes that represent logical endpoints for handling different categories of voice commands routed by VAPI: Calendar management, Gmail notifications, and Knowledge Base queries.

**Nodes Involved:**  
- MCP Server – Calendar  
- MCP Server – Gmail  
- MCP Server – Knowledge Base  
- Sticky Notes describing each MCP server

**Node Details:**  

- **MCP Server – Calendar**  
  - **Type & Role:** MCP trigger node; processes calendar-related voice commands.  
  - **Configuration:** Unique webhook path; triggers calendar operations such as create, fetch, update, and delete events.  
  - **Inputs:** Activated by HTTP requests from VAPI when calendar commands are detected.  
  - **Outputs:** Connects to Google Calendar operations nodes: Create, Fetch, Update, Delete Calendar Event nodes.  
  - **Potential Failures:** Authentication failures with Google Calendar; malformed date/time inputs; concurrency conflicts on event updates/deletes.

- **MCP Server – Gmail**  
  - **Type & Role:** MCP trigger node; handles email notification commands.  
  - **Configuration:** Unique webhook path; triggers sending emails as confirmation or reminders.  
  - **Inputs:** Activated when Gmail-related commands are detected in voice input.  
  - **Outputs:** Connects to "Send Email Notification" node.  
  - **Potential Failures:** Gmail OAuth token expiration; email delivery failures; invalid recipient addresses.

- **MCP Server – Knowledge Base**  
  - **Type & Role:** MCP trigger node; processes knowledge base queries by leveraging AI embeddings and vector search.  
  - **Configuration:** Unique webhook path for knowledge base queries.  
  - **Inputs:** Activated when general inquiry commands are detected.  
  - **Outputs:** Connects to "Supabase Vector Store" and "Embeddings OpenAI" nodes for retrieval and embedding generation.  
  - **Potential Failures:** API rate limits (OpenAI, Supabase); data retrieval failures; embedding generation errors.

- **Sticky Notes**  
  - Describe the purpose and responsibilities of each MCP server node for clarity.

---

#### 2.3 Google Calendar Operations

**Overview:**  
This block contains nodes that perform Create, Fetch, Update, and Delete operations on Google Calendar based on commands received through the MCP Server – Calendar node.

**Nodes Involved:**  
- Create Calendar Event  
- Fetch Calendar Events  
- Update Calendar Event  
- Delete Calendar Event

**Node Details:**  

- **Create Calendar Event**  
  - **Type & Role:** Google Calendar node; creates new calendar events.  
  - **Configuration:** Uses calendar `nabin.busines@gmail.com`; start and end times dynamically set from AI override expressions.  
  - **Inputs:** Triggered via MCP Server – Calendar node.  
  - **Outputs:** Passes result downstream or back to MCP Server.  
  - **Potential Failures:** Invalid date/time formats; permission denied errors; API quota limits.

- **Fetch Calendar Events**  
  - **Type & Role:** Google Calendar node; retrieves events in a date/time range.  
  - **Configuration:** Uses calendar `nabin.busines@gmail.com`; timeMin and timeMax dynamically set from AI override expressions; operation "getAll".  
  - **Inputs:** Triggered via MCP Server – Calendar node.  
  - **Outputs:** Event list to MCP Server or further processing.  
  - **Potential Failures:** Large result sets causing timeouts; incorrect time ranges; API errors.

- **Update Calendar Event**  
  - **Type & Role:** Google Calendar node; updates existing calendar events.  
  - **Configuration:** Event ID dynamically set; calendar selected; update fields dynamically populated from AI overrides.  
  - **Inputs:** Triggered via MCP Server – Calendar node.  
  - **Outputs:** Confirmation of update.  
  - **Potential Failures:** Event not found; concurrent modification conflicts; invalid changes.

- **Delete Calendar Event**  
  - **Type & Role:** Google Calendar node; deletes calendar events by event ID.  
  - **Configuration:** Event ID dynamically set; calendar selected; operation "delete".  
  - **Inputs:** Triggered via MCP Server – Calendar node.  
  - **Outputs:** Confirmation of deletion.  
  - **Potential Failures:** Event not found; permission issues; API errors.

---

#### 2.4 Gmail Email Notification

**Overview:**  
This block sends email notifications such as confirmations or reminders triggered by voice commands routed through the MCP Server – Gmail node.

**Nodes Involved:**  
- Send Email Notification

**Node Details:**  

- **Send Email Notification**  
  - **Type & Role:** Gmail node; sends email messages.  
  - **Configuration:** Uses Gmail OAuth2 credentials; recipient, subject, and message body dynamically set via AI override expressions; email type set to text.  
  - **Inputs:** Triggered by MCP Server – Gmail node.  
  - **Outputs:** Confirmation of email sent.  
  - **Potential Failures:** OAuth token expiration; invalid recipient email addresses; Gmail sending limits.

---

#### 2.5 Knowledge Base AI Processing

**Overview:**  
This block generates AI embeddings from input queries and retrieves relevant information from the Supabase vector store to respond to knowledge base inquiries.

**Nodes Involved:**  
- Embeddings OpenAI  
- Supabase Vector Store

**Node Details:**  

- **Embeddings OpenAI**  
  - **Type & Role:** OpenAI embeddings node; generates embeddings for input text.  
  - **Configuration:** Uses OpenAI API credentials; default embeddings model; no special options configured.  
  - **Inputs:** Triggered by MCP Server – Knowledge Base node.  
  - **Outputs:** Embeddings passed to Supabase Vector Store node.  
  - **Potential Failures:** API rate limits; invalid input text; network issues.

- **Supabase Vector Store**  
  - **Type & Role:** Supabase vector store node; retrieves relevant documents based on embeddings.  
  - **Configuration:** Connects to Supabase with specified credentials; queries `documents` table; operates in "retrieve-as-tool" mode; tool description provided for pricing/general inquiries.  
  - **Inputs:** Receives embeddings from Embeddings OpenAI node.  
  - **Outputs:** Returns relevant knowledge base results to MCP Server – Knowledge Base node.  
  - **Potential Failures:** Database connectivity issues; empty query results; permission errors.

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                          | Input Node(s)            | Output Node(s)                   | Sticky Note                                                                                                        |
|---------------------------|-----------------------------------------|----------------------------------------|--------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Incoming Webhook (VAPI)    | Webhook (HTTP)                         | Entry point for WhatsApp voice commands via VAPI | External HTTP request      | Respond to Webhook (VAPI)         | This webhook is triggered by VAPI when a WhatsApp voice command is received via Twilio. VAPI forwards requests here. |
| Respond to Webhook (VAPI)  | Respond to Webhook                      | Sends XML response back to VAPI        | Incoming Webhook (VAPI)   | None                            | Sends processed response back to VAPI for delivery via WhatsApp.                                                   |
| MCP Server – Calendar      | MCP Trigger                            | Handles calendar-related voice commands | External HTTP request      | Create, Fetch, Update, Delete Calendar Event nodes | 📅 Calendar MCP Handles Calendar actions: Create, Fetch, Update, Delete.                                            |
| Create Calendar Event      | Google Calendar Tool                    | Creates new calendar events            | MCP Server – Calendar     | MCP Server – Calendar           | 📅 Calendar MCP                                                                                                    |
| Fetch Calendar Events      | Google Calendar Tool                    | Fetches calendar events in date range | MCP Server – Calendar     | MCP Server – Calendar           | 📅 Calendar MCP                                                                                                    |
| Update Calendar Event      | Google Calendar Tool                    | Updates existing calendar events       | MCP Server – Calendar     | MCP Server – Calendar           | 📅 Calendar MCP                                                                                                    |
| Delete Calendar Event      | Google Calendar Tool                    | Deletes calendar events by ID          | MCP Server – Calendar     | MCP Server – Calendar           | 📅 Calendar MCP                                                                                                    |
| MCP Server – Gmail         | MCP Trigger                            | Handles email notification commands    | External HTTP request      | Send Email Notification         | 📧 Gmail MCP Sends confirmation/reminder emails based on voice commands.                                           |
| Send Email Notification    | Gmail Tool                            | Sends emails                           | MCP Server – Gmail        | MCP Server – Gmail              | 📧 Gmail MCP                                                                                                       |
| MCP Server – Knowledge Base| MCP Trigger                            | Handles knowledge base queries         | External HTTP request      | Embeddings OpenAI, Supabase Vector Store | 📚 Knowledge Base MCP OpenAI generates embeddings; Supabase vector store saves/retrieves data.                      |
| Embeddings OpenAI          | OpenAI Embeddings                      | Generates text embeddings for queries  | MCP Server – Knowledge Base| Supabase Vector Store           | 📚 Knowledge Base MCP                                                                                              |
| Supabase Vector Store      | Supabase Vector Store                   | Retrieves documents based on embeddings| Embeddings OpenAI         | MCP Server – Knowledge Base     | 📚 Knowledge Base MCP                                                                                              |
| Sticky Note1               | Sticky Note                           | Documentation for Calendar MCP block   | None                     | None                           | 📅 Calendar MCP Handles Calendar actions: Create, Fetch, Update, Delete                                           |
| Sticky Note2               | Sticky Note                           | Documentation for Gmail MCP block       | None                     | None                           | 📧 Gmail MCP Sends confirmation/reminder emails based on voice commands                                           |
| Sticky Note3               | Sticky Note                           | Documentation for Knowledge Base MCP    | None                     | None                           | 📚 Knowledge Base MCP OpenAI generates embeddings; Supabase vector store saves/retrieves data                     |
| Sticky Note                | Sticky Note                           | Documentation for Entry Point           | None                     | None                           | Entry Point – Incoming Webhook (VAPI) flow explanation                                                           |
| Sticky Note4               | Sticky Note                           | Documentation for Incoming Webhook (VAPI) | None                     | None                           | Webhook triggered by VAPI; routes to MCP servers                                                                 |
| Sticky Note5               | Sticky Note                           | Documentation for Respond to Webhook    | None                     | None                           | Sends processed response back to VAPI                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Incoming Webhook (VAPI) Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Generate a unique path (e.g., `81e240fc-d3fb-4ccd-b91a-0aacbf2d8f2a`)  
   - Response Mode: Respond with response node (defer response)  

2. **Create Respond to Webhook (VAPI) Node**  
   - Type: Respond to Webhook  
   - Connect input from Incoming Webhook node  
   - Response options: Add header `Content-Type: application/xml`  
   - Response body:  
     ```xml
     <?xml version="1.0" encoding="UTF-8"?>
     <Response>
        <Dial>
           <Sip>sip:15795503867@sip.vapi.ai</Sip>
        </Dial>
     </Response>
     ```
   - Position this node to the right of the Incoming Webhook node  

3. **Create MCP Server – Calendar Node**  
   - Type: MCP Trigger (Langchain MCP trigger node)  
   - Set unique webhook path (e.g., `1902a1d2-f8a8-4601-b20c-90e824fe478d`)  
   - This node acts as trigger for calendar operations  

4. **Create Google Calendar Nodes** (all using the same Google Calendar OAuth2 credential)  
   - **Create Calendar Event**  
     - Operation: Create  
     - Calendar: Select appropriate calendar (e.g., `nabin.busines@gmail.com`)  
     - Start and End: Use expressions to dynamically get start/end from AI overrides  
   - **Fetch Calendar Events**  
     - Operation: Get All  
     - Calendar: Same as above  
     - TimeMin and TimeMax: Use expressions for date range from AI overrides  
   - **Update Calendar Event**  
     - Operation: Update  
     - Calendar: Same calendar  
     - Event ID: Use expression from AI override  
     - Update fields: Map from AI override data  
   - **Delete Calendar Event**  
     - Operation: Delete  
     - Calendar: Same calendar  
     - Event ID: Use expression from AI override  

   - Connect all these calendar nodes to the MCP Server – Calendar node using the `ai_tool` connection.

5. **Create MCP Server – Gmail Node**  
   - Type: MCP Trigger  
   - Unique webhook path (e.g., `41a2ab5f-1a7d-440b-a1e3-1b2308dee744`)  
   - Acts as trigger for email notifications  

6. **Create Send Email Notification Node**  
   - Type: Gmail Tool  
   - Gmail OAuth2 credential configured  
   - Parameters:  
     - Send To: Expression from AI override  
     - Subject: Expression from AI override  
     - Message: Expression from AI override  
     - Email Type: Text  
   - Connect input from MCP Server – Gmail node via `ai_tool` connection  

7. **Create MCP Server – Knowledge Base Node**  
   - Type: MCP Trigger  
   - Unique webhook path (e.g., `3643c062-c554-43d8-84d1-692b886b780f`)  
   - Acts as trigger for knowledge base queries  

8. **Create Embeddings OpenAI Node**  
   - Type: Langchain Embeddings OpenAI  
   - OpenAI API credential configured  
   - No special options needed  
   - Connect input from MCP Server – Knowledge Base node via `ai_embedding` connection  

9. **Create Supabase Vector Store Node**  
   - Type: Langchain Vector Store Supabase  
   - Supabase API credential configured  
   - Table Name: `documents`  
   - Mode: Retrieve-as-tool  
   - Tool Description: "Retrieve this tool to get pricing Information or any other General Inquiry."  
   - Connect input from Embeddings OpenAI node via `ai_embedding` connection  
   - Connect output back to MCP Server – Knowledge Base node via `ai_tool` connection  

10. **Connect Outputs of MCP Servers to Incoming Webhook Node**  
    - Ensure the overall routing logic is handled externally by VAPI to call the correct MCP server webhook based on command intent.  

11. **Add Sticky Notes for Documentation**  
    - Create sticky notes describing entry point, MCP servers, and functional blocks as per the provided content to enhance maintainability.  

12. **Configure Credentials**  
    - Google Calendar OAuth2 with access to `nabin.busines@gmail.com`  
    - Gmail OAuth2 for sending emails  
    - OpenAI API key for embeddings and AI services  
    - Supabase API key for vector store access  

13. **Test the Workflow**  
    - Validate webhook endpoints with VAPI and Twilio integration  
    - Test calendar event creation, fetching, updating, deleting via voice commands  
    - Test sending email notifications triggered by voice commands  
    - Test knowledge base queries via natural language voice input  

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow leverages VAPI to interpret voice commands routed from WhatsApp via Twilio, enabling multi-modal input. | Integration flow: WhatsApp → Twilio (TwiML app) → VAPI → n8n Incoming Webhook                   |
| The calendar operations use Google Calendar OAuth2 with real email `nabin.busines@gmail.com`.                         | Ensure OAuth scopes include calendar read/write permissions                                     |
| The Gmail node uses OAuth2 credentials for sending emails; ensure Gmail API enabled on Google Cloud Console.          | OAuth2 refresh tokens should be handled to avoid auth failures                                  |
| OpenAI embeddings enable semantic search in the knowledge base stored in Supabase vector store.                       | OpenAI API rate limits and Supabase usage quotas should be monitored                            |
| Sticky notes embedded throughout the workflow provide valuable inline documentation for maintainers and developers. | Review sticky notes for quick understanding of node roles and connections                      |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or proprietary elements. All manipulated data is legal and public.

---