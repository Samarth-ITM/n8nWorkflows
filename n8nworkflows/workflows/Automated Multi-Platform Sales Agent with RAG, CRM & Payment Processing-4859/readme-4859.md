Automated Multi-Platform Sales Agent with RAG, CRM & Payment Processing

https://n8nworkflows.xyz/workflows/automated-multi-platform-sales-agent-with-rag--crm---payment-processing-4859


# Automated Multi-Platform Sales Agent with RAG, CRM & Payment Processing

### 1. Workflow Overview

This workflow implements an **Automated Multi-Platform Sales Agent** that integrates conversational AI, customer relationship management (CRM), calendar scheduling, and payment processing. It supports multiple communication platforms including WhatsApp, Facebook, Instagram, and a contact form, enabling seamless interaction with leads and customers. The workflow leverages Retrieval-Augmented Generation (RAG) with vector stores, AI chat agents (OpenAI and Google Gemini), and Postgres databases for chat memory and CRM data.

**Target Use Cases:**
- Automated sales lead qualification and follow-up across WhatsApp, Facebook, Instagram, and web forms
- CRM record creation, updating, and retrieval
- Calendar event management and scheduling with attendees
- Payment processing including customer creation, charges, and coupon management
- Multi-modal AI agent orchestration using LangChain nodes for conversational AI and tools

**Logical Blocks:**

- **1.1 Input Reception and Message Handling:** Receives messages from WhatsApp, Facebook, Instagram, and contact forms, filters and preprocesses them.
- **1.2 Lead Lookup and Message Routing:** Queries CRM leads and routes messages via a switch node to appropriate response paths.
- **1.3 AI Agent Processing and RAG Integration:** Uses a LangChain AI Agent with multiple tool workflows (CRM, Calendar, Billing) and vector stores for knowledge retrieval.
- **1.4 CRM Operations:** Create, update, delete contacts and opportunities in Postgres CRM with AI orchestration.
- **1.5 Calendar Operations:** Manage Google Calendar events (create, update, delete) via a calendar agent.
- **1.6 Billing and Payment Processing:** Stripe integration for customer and payment management invoked via a billing agent.
- **1.7 Outbound Messaging:** Responds to users on WhatsApp, Facebook, and Instagram with generated messages.
- **1.8 Memory and Chat History:** Chat memory stored in Postgres and in-memory buffers to maintain conversation context.
- **1.9 Error Handling and Retry Logic:** Nodes setting "Try Again" states and no-op fallbacks for robustness.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Message Handling

**Overview:**  
This block captures incoming messages and webhook events from multiple platforms (WhatsApp, Facebook, Instagram, Contact Form). It preprocesses input data, filters out echo messages, and routes valid messages for further handling.

**Nodes Involved:**  
- WhatsApp (Trigger)  
- Facebook Trigger (Webhook)  
- Instagram Trigger (Webhook)  
- Contact Form (Form Trigger)  
- Handle Message Types (Switch)  
- If is not echo - facebook (If)  
- If is not echo - instagram (If)  
- Edit Fields - facebook (Set)  
- Edit Fields - instagram (Set)  
- Respond to Webhook - facebook post / get  
- Respond to Webhook - instagram post / get

**Node Details:**

- **WhatsApp (Trigger)**  
  - Type: Webhook Trigger for WhatsApp messages  
  - Configuration: listens to incoming WhatsApp messages via webhook  
  - Input: Incoming WhatsApp messages  
  - Output: Passes data to "Get Lead" node  
  - Edge Cases: Missed webhook calls, malformed messages, authentication failures

- **Facebook Trigger**  
  - Type: Webhook Trigger for Facebook posts/gets  
  - Configuration: webhook listening for Facebook Messenger events  
  - Input: Facebook user messages and events  
  - Output: Branches to "Respond to Webhook - facebook post" and "Respond to Webhook - facebook get"  
  - Edge Cases: API permissions, webhook verification failure

- **Instagram Trigger**  
  - Type: Webhook Trigger for Instagram messages  
  - Configuration: webhook for Instagram messaging events  
  - Input: Instagram direct messages  
  - Output: Branches to "Respond to Webhook - instagram post" and "Respond to Webhook - instagram get"  
  - Edge Cases: API rate limits, webhook subscription errors

- **Contact Form**  
  - Type: Form Trigger for web-based contact forms  
  - Configuration: listens for form submissions  
  - Input: Contact form data (lead info)  
  - Output: "Create Contact1" node for CRM entry creation  
  - Edge Cases: Form validation failures, missing required fields

- **Handle Message Types (Switch)**  
  - Type: Switch node to filter message types for WhatsApp  
  - Configuration: Routes message flow based on the message type (e.g., text, command)  
  - Input: Output from "Get Lead"  
  - Output: Branches to "Edit Fields - chat1", "WhatsApp Business Cloud", or "Reply To User1"  
  - Edge Cases: Unknown message types, routing failures

- **If is not echo - facebook**  
  - Type: If node filtering out echo messages (messages sent by the bot itself) on Facebook  
  - Configuration: Checks message origin, passes only if not echo  
  - Input: Facebook webhook data  
  - Output: Proceeds to "Edit Fields - facebook" and "Sales Agent Demo - typing_on"  
  - Edge Cases: False positives/negatives for echo detection

- **If is not echo - instagram**  
  - Type: If node filtering echo messages on Instagram  
  - Configuration: Similar to Facebook echo filter  
  - Input: Instagram webhook data  
  - Output: Passes to "Edit Fields - instagram"  
  - Edge Cases: Same as Facebook echo filter

- **Edit Fields - facebook / instagram**  
  - Type: Set node to prepare and normalize incoming data fields for downstream processing  
  - Configuration: Maps raw webhook data fields to a common data structure  
  - Output: No Operation node as placeholder (no further processing at this stage)  
  - Edge Cases: Missing or malformed data fields

- **Respond to Webhook - facebook post / get**  
  - Type: Respond to webhook nodes  
  - Configuration: Sends HTTP 200 OK responses to Facebook webhook calls to acknowledge receipt  
  - Edge Cases: Failure to respond may cause webhook retries

- **Respond to Webhook - instagram post / get**  
  - Type: Respond to webhook nodes for Instagram  
  - Configuration: Same as Facebook response nodes  
  - Edge Cases: Same as Facebook response nodes

---

#### 1.2 Lead Lookup and Message Routing

**Overview:**  
This block queries the CRM database to find lead records corresponding to incoming messages and determines routing paths for replies or further processing.

**Nodes Involved:**  
- Get Lead (Postgres)  
- Handle Message Types (Switch)  
- Edit Fields - chat1 / chat2 (Set)  
- WhatsApp Business Cloud (WhatsApp node)  
- Reply To User1 (WhatsApp node)  
- No Operation, do nothing (NoOp)

**Node Details:**

- **Get Lead**  
  - Type: Postgres node querying the CRM database for lead info based on incoming message data (e.g., phone number, email)  
  - Configuration: Uses SQL to fetch lead details  
  - Input: Incoming WhatsApp message data  
  - Output: Leads to "Handle Message Types" for routing  
  - Edge Cases: DB connection issues, no lead found, SQL errors

- **Handle Message Types**  
  - Routes messages depending on their type as described above

- **Edit Fields - chat1 / chat2**  
  - Prepare message content or metadata for AI processing or reply nodes

- **WhatsApp Business Cloud**  
  - Sends WhatsApp messages using the Business Cloud API  
  - Edge Cases: API rate limits, auth token expiration

- **Reply To User1**  
  - Sends direct WhatsApp replies via webhook-triggered conversations

- **No Operation, do nothing**  
  - Placeholder node used to complete branches where no action is needed

---

#### 1.3 AI Agent Processing and RAG Integration

**Overview:**  
Central AI agent block combining LangChain agents for conversational AI. It uses multiple AI language models (Google Gemini and OpenAI), vector stores for retrieval (Postgres PGVector), and tool workflows for CRM, calendar, and billing integration.

**Nodes Involved:**  
- AI Agent (LangChain agent)  
- CRM Agent (toolWorkflow)  
- Calendar Agent (toolWorkflow)  
- Billing Agent (toolWorkflow)  
- technical_and_sales_knowledge (toolVectorStore)  
- Postgres PGVector Store (vector store)  
- Google Gemini Chat Model (multiple instances)  
- OpenAI  
- Postgres Chat Memory (memoryPostgresChat)  
- Window Buffer Memory (memoryBufferWindow)  
- Switch1 (Switch)

**Node Details:**

- **AI Agent**  
  - Orchestrates interaction among AI language models and tool workflows  
  - Inputs: Chat memory, vector store results, user messages  
  - Outputs: Routed through "Switch1" to different response nodes or APIs  
  - Edge Cases: Model API failures, timeout, malformed input, tool invocation errors

- **CRM Agent**  
  - A toolWorkflow node wrapping CRM operations with AI control  
  - Invoked as a tool by the AI Agent to perform CRM CRUD operations  
  - Edge Cases: Postgres DB errors, concurrency conflicts

- **Calendar Agent**  
  - ToolWorkflow handling calendar event operations  
  - Integrated with Google Calendar nodes for event CRUD  
  - Edge Cases: Google API quota limits, auth token expiration

- **Billing Agent**  
  - ToolWorkflow managing Stripe payment and customer operations  
  - Edge Cases: Stripe API errors, payment failures

- **technical_and_sales_knowledge**  
  - Vector store holding domain-specific knowledge for RAG  
  - Connected to AI Agent for knowledge retrieval

- **Postgres PGVector Store**  
  - Stores embeddings for retrieval-based generation  
  - Edge Cases: DB storage issues, vector similarity search failures

- **Google Gemini Chat Model (multiple instances)**  
  - Language models used for different AI agents (personalized messages, CRM, calendar, billing agents)  
  - Edge Cases: API rate limits, model version changes

- **OpenAI**  
  - Used for language generation in some flows, possibly fallback or complementary to Gemini

- **Postgres Chat Memory**  
  - Stores conversation history persistently

- **Window Buffer Memory**  
  - In-memory context window for short-term conversation state

- **Switch1**  
  - Routes AI Agent output to appropriate messaging platform nodes or final output sets

---

#### 1.4 CRM Operations

**Overview:**  
This block manages CRM data by creating, updating, deleting contacts and opportunities in a Postgres database, all controlled via AI agents.

**Nodes Involved:**  
- Create Contact (PostgresTool)  
- Update Contact (PostgresTool)  
- Delete Contact (PostgresTool)  
- Create Opportunity (PostgresTool)  
- Update Opportunity (PostgresTool)  
- Delete Opportunity (PostgresTool)  
- List Records (PostgresTool)  
- CRM Agent2 (LangChain agent)  
- Response (Set)  
- Try Again (Set)  
- When Executed by Another Workflow (ExecuteWorkflowTrigger)  
- Create Contact1 (Postgres)

**Node Details:**

- **PostgresTool Nodes**  
  - Perform CRUD operations on CRM tables  
  - Inputs come from CRM Agent2 invoking these as tools  
  - Edge Cases: SQL injection risks, DB transaction failures

- **CRM Agent2**  
  - AI agent for CRM management, uses Google Gemini Chat Model2  
  - On failure, continues error output for graceful degradation

- **Response / Try Again**  
  - Set nodes to prepare success or retry messages for calling workflows or front-end

- **When Executed by Another Workflow**  
  - Allows this workflow to be invoked with parameters for CRM operations from other workflows

- **Create Contact1**  
  - Entry point from contact form submissions to create CRM contact

---

#### 1.5 Calendar Operations

**Overview:**  
Manages Google Calendar event lifecycle (create, update, delete, get) via a calendar agent AI workflow.

**Nodes Involved:**  
- Create Event  
- Create Event with Attendee  
- Get Events  
- Update Event  
- Delete Event  
- Calendar Agent1 (LangChain agent)  
- Success (Set)  
- Try Again1 (Set)  
- Sales Agent Demo - typing_on (Facebook Graph API, typing indicator)  
- Google Gemini Chat Model3

**Node Details:**

- **Google Calendar Tool Nodes**  
  - Perform event CRUD operations on Google Calendar  
  - Inputs from Calendar Agent1 invoking these as tools  
  - Edge Cases: OAuth token expiration, API rate limiting

- **Calendar Agent1**  
  - AI agent controlling calendar tool operations  
  - Uses Google Gemini Chat Model3

- **Success / Try Again1**  
  - Manage operation status responses and retries

- **Sales Agent Demo - typing_on**  
  - Sends typing indicator via Facebook Graph API during calendar interactions for UX improvement

---

#### 1.6 Billing and Payment Processing

**Overview:**  
Handles Stripe payment processing including customer and card management, charge creation, and coupon discounts, orchestrated by a billing agent AI workflow.

**Nodes Involved:**  
- Create Customer  
- Get Customer  
- Update Customer  
- Delete Customer  
- Get Customer Card  
- Create Charge  
- Create Discount Coupon  
- Delete Customer  
- Billing Agent1 (LangChain agent)  
- Billing Agent (toolWorkflow)  
- Simple Memory (memoryBufferWindow)  
- Google Gemini Chat Model4  
- Response1 (Set)  
- Try Again2 (Set)

**Node Details:**

- **Stripe Tool Nodes**  
  - Manage Stripe customer lifecycle and transactions  
  - Inputs controlled by Billing Agent1 invoking these tools  
  - Edge Cases: Payment failures, invalid card info, API errors

- **Billing Agent1**  
  - AI agent managing billing/payment processes with Google Gemini Chat Model4  
  - Uses Simple Memory for short-term context

- **Billing Agent**  
  - ToolWorkflow integrated with AI Agent to expose billing operations as tools

- **Response1 / Try Again2**  
  - Prepare success or retry responses for billing workflows

---

#### 1.7 Outbound Messaging

**Overview:**  
Sends responses back to users on WhatsApp, Facebook Messenger, Instagram, and via form triggers after AI processing.

**Nodes Involved:**  
- Reply To User1 (WhatsApp)  
- WhatsApp Business Cloud  
- Reply To User (WhatsApp)  
- Send First Message (WhatsApp)  
- Personalised First Message (AI Agent)  
- Facebook Graph API - Sales Agent Demo  
- Instagram Graph API - smb.sales.agent.demo  
- Output - chat (Set)

**Node Details:**

- **WhatsApp Nodes**  
  - Send direct messages, initial messages, and replies on WhatsApp platforms  
  - Use webhook IDs for proper session management  
  - Edge Cases: Message send failures, platform rate limits

- **Facebook / Instagram Graph API Nodes**  
  - Outbound messaging and typing indicators for Facebook and Instagram  
  - Edge Cases: API permission issues, message formatting errors

- **Personalised First Message**  
  - AI agent node generating personalized initial outreach messages

- **Output - chat**  
  - Final data set for chat responses that might be logged or sent to other endpoints

---

#### 1.8 Memory and Chat History

**Overview:**  
Maintains conversation context for AI agents using persistent Postgres chat memory and short-term window buffer memory.

**Nodes Involved:**  
- Postgres Chat Memory  
- Window Buffer Memory  
- Simple Memory (for billing agent)

**Node Details:**

- **Postgres Chat Memory**  
  - Stores full chat history persistently in Postgres for context retrieval

- **Window Buffer Memory**  
  - Holds recent chat context in memory for efficient access during conversation

- **Simple Memory**  
  - Used specifically by Billing Agent1 for payment context

---

#### 1.9 Error Handling and Retry Logic

**Overview:**  
Sets workflow states for retrying failed operations and no-op fallbacks to maintain workflow stability.

**Nodes Involved:**  
- Try Again, Try Again1, Try Again2 (Set)  
- No Operation, do nothing (NoOp)

**Node Details:**

- **Try Again Nodes**  
  - Prepare retry messages or flags for orchestrating re-execution of failed steps

- **No Operation**  
  - Used as placeholders or default outputs to prevent workflow breakage

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                            | Input Node(s)                         | Output Node(s)                          | Sticky Note                                              |
|----------------------------------|----------------------------------|--------------------------------------------|-------------------------------------|----------------------------------------|----------------------------------------------------------|
| WhatsApp                         | WhatsApp Trigger                 | Receives WhatsApp messages                  | -                                   | Get Lead                              |                                                          |
| Get Lead                        | Postgres                        | Query lead info from CRM                     | WhatsApp                            | Handle Message Types                   |                                                          |
| Handle Message Types             | Switch                         | Routes message types                         | Get Lead                           | Edit Fields - chat1, WhatsApp Business Cloud, Reply To User1 |                                                          |
| Edit Fields - chat1              | Set                            | Prepares chat fields                         | Handle Message Types                | No Operation, do nothing               |                                                          |
| WhatsApp Business Cloud          | WhatsApp                       | Sends WhatsApp messages                       | Handle Message Types                | HTTP Request                          |                                                          |
| Reply To User1                  | WhatsApp                       | Sends WhatsApp replies                        | Handle Message Types                | -                                    |                                                          |
| No Operation, do nothing         | NoOp                          | Placeholder node                             | Edit Fields - chat1 / chat2 / chat | AI Agent                            |                                                          |
| AI Agent                       | LangChain Agent                | Central AI orchestration                      | No Operation, do nothing            | Switch1                             |                                                          |
| Switch1                        | Switch                         | Routes AI output                             | AI Agent                          | Reply To User, Facebook Graph API, Instagram Graph API, Output - chat |                                                          |
| Reply To User                   | WhatsApp                      | Sends WhatsApp reply                         | Switch1                           | -                                    |                                                          |
| Facebook Graph API - Sales Agent Demo | Facebook Graph API            | Facebook messaging and typing indicator      | Switch1 / If is not echo - facebook | -                                    |                                                          |
| Instagram Graph API - smb.sales.agent.demo | HTTP Request                  | Instagram messaging                           | Switch1                          | -                                    |                                                          |
| Output - chat                   | Set                           | Final chat output                            | Switch1                          | No Operation, do nothing              |                                                          |
| CRM Agent                      | LangChain toolWorkflow         | CRM operations tool                          | AI Agent                         | AI Agent (via ai_tool)                 |                                                          |
| CRM Agent2                     | LangChain agent               | CRM AI orchestration                         | When Executed by Another Workflow, List Records | Response, Try Again                    |                                                          |
| When Executed by Another Workflow | ExecuteWorkflowTrigger        | Entry point for external workflow calls      | -                                 | CRM Agent2                          |                                                          |
| Create Contact1                | Postgres                      | Create CRM contact                           | Contact Form                     | Personalised First Message             |                                                          |
| Personalised First Message      | LangChain Agent               | Generate personalized first message          | Create Contact1                  | Send First Message                    |                                                          |
| Send First Message             | WhatsApp                      | Send first WhatsApp message                   | Personalised First Message        | -                                    |                                                          |
| Contact Form                   | Form Trigger                  | Receive web contact form submissions          | -                               | Create Contact1                      |                                                          |
| Postgres Chat Memory            | LangChain memoryPostgresChat  | Persistent conversation memory                | AI Agent (ai_memory)             | AI Agent (ai_memory)                   |                                                          |
| Window Buffer Memory           | LangChain memoryBufferWindow  | Short-term conversation memory                | CRM Agent2 (ai_memory)           | CRM Agent2 (ai_memory)                 |                                                          |
| technical_and_sales_knowledge  | LangChain toolVectorStore     | Vector store of domain knowledge              | Postgres PGVector Store (ai_vectorStore) | AI Agent (ai_tool)                     |                                                          |
| Postgres PGVector Store         | LangChain vectorStorePGVector | Stores embeddings for retrieval               | Embeddings Google Gemini1 (ai_embedding) | technical_and_sales_knowledge (ai_vectorStore) |                                                          |
| Embeddings Google Gemini1       | LangChain embeddingsGoogleGemini | Generates embeddings for vector store         | -                               | Postgres PGVector Store (ai_embedding) |                                                          |
| Google Gemini Chat Model        | LangChain lmChatGoogleGemini  | Language model for AI Agent                    | -                               | Personalised First Message, CRM Agent2, Calendar Agent1, Billing Agent1 |                                                          |
| OpenAI                         | LangChain openAi              | Alternative/fallback language model            | HTTP Request                    | Edit Fields - chat2                   |                                                          |
| HTTP Request                   | HTTP Request                 | Sends HTTP requests (e.g., WhatsApp Business Cloud) | WhatsApp Business Cloud          | OpenAI                              |                                                          |
| Calendar Agent                 | LangChain toolWorkflow         | Calendar operations tool                      | AI Agent (ai_tool)              | AI Agent (ai_tool)                     |                                                          |
| Calendar Agent1                | LangChain agent               | AI agent managing Google Calendar             | Get Events, Create Event, Delete Event, Update Event, Create Event with Attendee (ai_tool) | Success, Try Again1                   |                                                          |
| Create Event                  | Google Calendar Tool          | Creates calendar event                         | Calendar Agent1 (ai_tool)        | Calendar Agent1                      |                                                          |
| Create Event with Attendee    | Google Calendar Tool          | Creates event with attendees                    | Calendar Agent1 (ai_tool)        | Calendar Agent1                      |                                                          |
| Get Events                   | Google Calendar Tool          | Retrieves calendar events                       | Calendar Agent1 (ai_tool)        | Calendar Agent1                      |                                                          |
| Update Event                 | Google Calendar Tool          | Updates calendar event                          | Calendar Agent1 (ai_tool)        | Calendar Agent1                      |                                                          |
| Delete Event                 | Google Calendar Tool          | Deletes calendar event                          | Calendar Agent1 (ai_tool)        | Calendar Agent1                      |                                                          |
| Billing Agent                | LangChain toolWorkflow         | Payment and billing tool                       | AI Agent (ai_tool)              | AI Agent (ai_tool)                     |                                                          |
| Billing Agent1               | LangChain agent               | AI agent managing Stripe billing                | Get Customer, Create Charge, Create Discount Coupon, etc. (ai_tool) | Response1, Try Again2                |                                                          |
| Create Customer             | Stripe Tool                  | Creates Stripe customer                         | Billing Agent1 (ai_tool)         | Billing Agent1                      |                                                          |
| Get Customer                | Stripe Tool                  | Retrieves Stripe customer                       | Billing Agent1 (ai_tool)         | Billing Agent1                      |                                                          |
| Update Customer             | Stripe Tool                  | Updates Stripe customer                         | Billing Agent1 (ai_tool)         | Billing Agent1                      |                                                          |
| Delete Customer             | Stripe Tool                  | Deletes Stripe customer                         | Billing Agent1 (ai_tool)         | Billing Agent1                      |                                                          |
| Get Customer Card           | Stripe Tool                  | Retrieves customer card details                  | Billing Agent1 (ai_tool)         | Billing Agent1                      |                                                          |
| Create Charge               | Stripe Tool                  | Creates payment charge                           | Billing Agent1 (ai_tool)         | Billing Agent1                      |                                                          |
| Create Discount Coupon      | Stripe Tool                  | Creates discount coupons                         | Billing Agent1 (ai_tool)         | Billing Agent1                      |                                                          |
| Response                    | Set                          | Sets response messages for CRM operations       | CRM Agent2                      | -                                    |                                                          |
| Try Again                   | Set                          | Sets retry messages for CRM operations          | CRM Agent2                      | -                                    |                                                          |
| Response1                   | Set                          | Sets response messages for billing operations   | Billing Agent1                  | -                                    |                                                          |
| Try Again2                  | Set                          | Sets retry messages for billing operations      | Billing Agent1                  | -                                    |                                                          |
| Success                    | Set                          | Sets success message for calendar operations    | Calendar Agent1                 | -                                    |                                                          |
| Try Again1                 | Set                          | Sets retry message for calendar operations       | Calendar Agent1                 | -                                    |                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Input Triggers:**  
   - Add **WhatsApp Trigger** node, set webhook ID, configure API credentials for WhatsApp Business Cloud.  
   - Add **Facebook Trigger** webhook node, configure Facebook app webhook URL and permissions.  
   - Add **Instagram Trigger** webhook node, configure Instagram messaging webhook subscription.  
   - Add **Contact Form** node to receive web-based lead submissions.

2. **Lead Lookup:**  
   - Add **Postgres** node "Get Lead" to query leads by phone/email from the incoming message.  
   - Configure database credentials and SQL query to fetch lead info.

3. **Message Type Handling:**  
   - Add a **Switch** node "Handle Message Types" to route messages based on type (text, command, etc.).  
   - Configure conditions for routing to different paths.

4. **Prepare Fields for Chat:**  
   - Add **Set** nodes "Edit Fields - chat1" and "Edit Fields - chat2" to format data for AI processing.

5. **Outbound WhatsApp Messaging:**  
   - Add **WhatsApp Business Cloud** node to send messages via WhatsApp API.  
   - Add **Reply To User1** node for replying to users.  
   - Create **No Operation** node as placeholder for branches requiring no action.

6. **AI Agent Setup:**  
   - Add **LangChain AI Agent** node "AI Agent" to orchestrate AI interactions.  
   - Connect to **LangChain toolWorkflow** nodes: CRM Agent, Calendar Agent, Billing Agent for integrated tool operations.  
   - Configure AI language models: add **Google Gemini Chat Model** and **OpenAI** nodes as required.  
   - Attach **Postgres Chat Memory** and **Window Buffer Memory** nodes for conversation context.  
   - Configure vector store with **Embeddings Google Gemini1** and **Postgres PGVector Store** nodes.  
   - Create **toolVectorStore** node "technical_and_sales_knowledge" connected to vector store for RAG.

7. **CRM Operations:**  
   - Add PostgresTool nodes to create/update/delete contacts and opportunities.  
   - Add **LangChain agent** node "CRM Agent2" to orchestrate CRM functions.  
   - Add **Set** nodes "Response" and "Try Again" for operation feedback.  
   - Add **ExecuteWorkflowTrigger** "When Executed by Another Workflow" for external invocation.

8. **Calendar Operations:**  
   - Add Google Calendar nodes for event creation, update, deletion, and retrieval.  
   - Add **LangChain agent** "Calendar Agent1" for managing calendar via AI.  
   - Add **Set** nodes "Success" and "Try Again1" for operation statuses.  
   - Optionally add Facebook Graph API node for typing indicators.

9. **Billing Operations:**  
   - Add Stripe Tool nodes for customer and payment management (create, update, delete).  
   - Add **LangChain agent** "Billing Agent1" for billing AI orchestration.  
   - Add **Set** nodes "Response1" and "Try Again2" for billing responses.  
   - Add **Simple Memory** node for billing context.

10. **Outbound Messaging for Other Platforms:**  
    - Add Facebook Graph API and HTTP Request nodes for Facebook and Instagram messaging replies.

11. **Routing AI Agent Outputs:**  
    - Add **Switch1** node to route AI Agent outputs to appropriate messaging nodes or final output.

12. **Final Output and Error Handling:**  
    - Add **Set** nodes for final chat output and try again logic as needed.  
    - Use **No Operation** nodes to safely terminate branches.

13. **Credential Setup:**  
    - Configure credentials for WhatsApp Business Cloud, Facebook Graph API, Instagram API, Google Calendar (OAuth2), Postgres DB, and Stripe API.

14. **Test and Deploy:**  
    - Test each input source independently.  
    - Verify AI agent responses and tool integrations.  
    - Monitor logs for errors, adjust retry nodes as necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                 | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow demonstrates advanced multi-channel sales automation integrating AI with CRM, calendar, and billing tools.                                                   | Workflow title and description                                                                      |
| The AI agents leverage both Google Gemini and OpenAI models via LangChain nodes for enhanced conversational capabilities.                                                  | Nodes: AI Agent, Google Gemini Chat Model, OpenAI                                                  |
| Vector store uses Postgres PGVector for embedding-based retrieval augmenting AI responses with domain knowledge.                                                            | Nodes: Postgres PGVector Store, Embeddings Google Gemini1                                           |
| Stripe payment integration supports full customer lifecycle and charge management controlled by AI workflow nodes.                                                         | Nodes: Stripe Tool nodes, Billing Agent, Billing Agent1                                            |
| Facebook and Instagram messaging use webhook triggers and outgoing Graph API interaction with safeguards to avoid echo messages.                                           | Nodes: Facebook Trigger, Instagram Trigger, If is not echo - facebook/instagram, Facebook Graph API |
| Google Calendar integration supports event creation, update, deletion with attendee management and typing indicators for UX.                                               | Nodes: Google Calendar Tool nodes, Calendar Agent1, Sales Agent Demo - typing_on                    |
| Contact form trigger allows external web leads to enter the CRM and initiate automated outreach via AI agents.                                                             | Node: Contact Form, Create Contact1, Personalised First Message                                    |
| Error handling uses set nodes like Try Again to flag retry logic and no-op nodes to ensure workflow stability on failures.                                                | Nodes: Try Again, Try Again1, Try Again2, No Operation                                            |
| For more information on LangChain integration with n8n, see: https://n8n.io/integrations/n8n-nodes-langchain                                                              | Official n8n LangChain integration documentation                                                  |
| Stripe API documentation for payment node configuration: https://stripe.com/docs/api                                                                                         | Stripe API reference                                                                               |
| Google Calendar API OAuth2 setup guidelines: https://developers.google.com/calendar/api/quickstart/js                                                                       | Google Calendar API developer guide                                                              |
| Facebook and Instagram webhook setup instructions: https://developers.facebook.com/docs/messenger-platform/getting-started/webhook-setup                                  | Facebook Developer documentation                                                                  |

---

This document fully describes the workflow structure, logic, node configurations, and integration points to enable advanced users and automation agents to understand, reproduce, and maintain this multi-platform sales automation setup.