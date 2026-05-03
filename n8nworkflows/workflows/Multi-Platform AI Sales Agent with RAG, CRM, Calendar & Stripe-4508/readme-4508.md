Multi-Platform AI Sales Agent with RAG, CRM, Calendar & Stripe

https://n8nworkflows.xyz/workflows/multi-platform-ai-sales-agent-with-rag--crm--calendar---stripe-4508


# Multi-Platform AI Sales Agent with RAG, CRM, Calendar & Stripe

### 1. Workflow Overview

This workflow implements a **Multi-Platform AI Sales Agent** that integrates AI-driven conversational capabilities with CRM, calendar management, and payment processing tools. It is designed to automate sales, customer engagement, lead handling, and billing across multiple communication platforms such as WhatsApp, Facebook, and Instagram. The workflow leverages Retrieval-Augmented Generation (RAG) techniques, vector stores for knowledge management, and AI language models including Google Gemini and OpenAI.

**Target Use Cases:**
- Automated multi-channel customer interaction and lead qualification.
- Integration of sales and billing workflows with CRM systems.
- Calendar event management for sales appointments.
- Handling payments and subscriptions via Stripe.
- Use of AI agents that dynamically switch between tools and memory stores for context-aware responses.

**Logical Blocks:**

- **1.1 Input Reception & Message Handling:** Receives messages from WhatsApp, Facebook, Instagram; differentiates message types.
- **1.2 AI Processing & Agent Coordination:** Uses Langchain AI agents, language models, and vector stores for understanding and response generation.
- **1.3 CRM Integration:** Manages customer, contact, lead, and opportunity data via PostgreSQL.
- **1.4 Calendar Management:** Handles event creation, updating, and deletion via Google Calendar.
- **1.5 Billing & Payments:** Manages customer billing, charges, discounts, and payment methods through Stripe.
- **1.6 Output & Response Delivery:** Sends replies back to users on relevant platforms, manages typing indicators.
- **1.7 Memory & Knowledge Management:** Uses Postgres chat memory and vector stores for conversation context and knowledge retrieval.
- **1.8 Workflow Triggers & Subflows:** Includes webhook triggers and sub-workflow executions for modularity and external event handling.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Message Handling

**Overview:**  
This block receives inbound messages from multiple platforms (WhatsApp, Facebook, Instagram), distinguishes message types, and routes for further processing.

**Nodes Involved:**  
- WhatsApp Trigger  
- Facebook Trigger  
- Instagram Trigger  
- Handle Message Types (Switch)  
- If is not echo - facebook (If)  
- If is not echo - instagram (If)  
- Edit Fields - chat1  
- Edit Fields - facebook  
- Edit Fields - instagram  
- WhatsApp Business Cloud  
- Reply To User1  
- Reply To User

**Node Details:**

- **WhatsApp Trigger**  
  - Type: Trigger node for WhatsApp messages  
  - Receives incoming WhatsApp messages via webhook  
  - Outputs messages to "Get Lead" for lead extraction  
  - Failure modes: webhook issues, message format errors

- **Facebook Trigger**  
  - Type: Webhook node for Facebook messages  
  - Receives Facebook Messenger events; splits into post and get requests  
  - Connected to corresponding Respond to Webhook nodes  
  - Failure modes: webhook authentication, API limit

- **Instagram Trigger**  
  - Type: Webhook node for Instagram messages  
  - Receives Instagram direct messages or events  
  - Connected to responses node for post/get requests  
  - Failure modes: webhook config, API quota

- **Handle Message Types (Switch)**  
  - Type: Switch node  
  - Differentiates incoming message types (chat, WhatsApp, Facebook, etc.)  
  - Routes messages to specific edit and reply nodes  
  - Edge cases: unexpected message types

- **If is not echo - facebook / instagram (If)**  
  - Conditional nodes to filter out echo or duplicate messages from Facebook/Instagram to avoid loops  
  - Passes only new messages downstream  
  - Failure modes: incorrect condition expression causing message loss

- **Edit Fields - chat1 / facebook / instagram (Set nodes)**  
  - Prepare or modify incoming message data for unified processing  
  - Set key variables or normalize message payloads  
  - Failure modes: expression errors if fields missing

- **WhatsApp Business Cloud**  
  - Node to send WhatsApp messages via WhatsApp Business Cloud API  
  - Sends outgoing messages after processing  
  - Failure modes: API authentication, rate limits

- **Reply To User1 / Reply To User**  
  - Send replies back to users on WhatsApp  
  - Configured with webhook IDs to respond directly  
  - Failure modes: message delivery failures, formatting issues

---

#### 1.2 AI Processing & Agent Coordination

**Overview:**  
This block uses Langchain AI agents and multiple language models to process messages, manage context, and decide tool usage like CRM, Calendar, or Billing interactions.

**Nodes Involved:**  
- AI Agent  
- Gemini  
- OpenAI  
- Google Gemini Chat Models (multiple instances)  
- CRM Agent  
- Calendar Agent  
- Billing Agent  
- technical_and_sales_knowledge (VectorStore)  
- Postgres Chat Memory  
- Window Buffer Memory  
- No Operation, do nothing (NoOp)

**Node Details:**

- **AI Agent**  
  - Type: Langchain AI agent node  
  - Central orchestrator that receives input and calls specialized tool workflows (CRM, Calendar, Billing)  
  - Uses Postgres Chat Memory for conversation context  
  - Routes output to Switch1 for platform-specific response routing  
  - Version-specific: v1.7 with ai_memory and ai_tool integrations  
  - Potential failures: model timeouts, memory retrieval errors

- **Gemini & Google Gemini Chat Models**  
  - Type: AI language model nodes using Google Gemini (chat-based)  
  - Provide natural language understanding and generation  
  - Multiple instances used for different sub-agents (CRM, Calendar, Billing, Knowledge)  
  - Version: v1 or v1.8 depending on node  
  - Failures: API errors, token limits, latency

- **OpenAI**  
  - Type: OpenAI Langchain node for language models  
  - Used for complementary NLP tasks or fallback  
  - Integrated with HTTP Request node for custom calls  
  - Failures: API key issues, quota exceeded

- **CRM Agent, Calendar Agent, Billing Agent**  
  - Type: ToolWorkflow or Agent nodes  
  - Encapsulate CRM, calendar, and billing logic respectively as sub-workflows or multi-tool agents  
  - Connected as ai_tool to main AI Agent node  
  - Failures: sub-workflow errors, credential misconfiguration

- **technical_and_sales_knowledge (VectorStore)**  
  - Type: Vector Store node backed by Postgres PGVector  
  - Stores embeddings from Embeddings Google Gemini1 node  
  - Provides RAG capabilities to retrieve relevant knowledge snippets  
  - Failures: DB connection issues, embedding computation errors

- **Postgres Chat Memory & Window Buffer Memory**  
  - Memory nodes that store conversational context in Postgres or in-memory buffer  
  - Enable context-aware AI responses  
  - Failures: DB write/read errors, memory overflow

- **No Operation, do nothing (NoOp)**  
  - Placeholder node used to gracefully handle no-action paths in flows

---

#### 1.3 CRM Integration

**Overview:**  
Handles creation, updating, deletion, and listing of contacts, leads, and opportunities in a PostgreSQL database acting as CRM backend.

**Nodes Involved:**  
- Get Lead (Postgres)  
- Create Contact1 (Postgres)  
- Create Contact (PostgresTool)  
- Update Contact (PostgresTool)  
- Delete Contact (PostgresTool)  
- Create Opportunity (PostgresTool)  
- Update Opportunity (PostgresTool)  
- Delete Opportunity (PostgresTool)  
- List Records (PostgresTool)  
- CRM Agent2 (Langchain agent)  
- When Executed by Another Workflow (Execute Workflow Trigger)

**Node Details:**

- **Get Lead**  
  - Retrieves lead information from PostgreSQL based on incoming message data  
  - Used early in message processing to identify existing leads  
  - Failures: DB connection, query errors

- **Create / Update / Delete Contact & Opportunity**  
  - Perform CRUD operations on CRM tables  
  - Connected as ai_tools to CRM Agent2 for AI-driven CRM management  
  - Failures: data integrity, constraint violations, DB errors

- **List Records**  
  - Lists CRM records for retrieval or validation  
  - Connected to CRM Agent2 for context-aware operations

- **CRM Agent2**  
  - AI agent focused on CRM tasks, operates with Window Buffer Memory  
  - Handles commands or queries related to CRM data  
  - On error set to continue with error output for resilience

- **When Executed by Another Workflow**  
  - Entry point to trigger CRM operations from external workflows  
  - Accepts parameters like contact details and interest notes

---

#### 1.4 Calendar Management

**Overview:**  
Manages Google Calendar events for appointments and scheduling, including creation, updating, deleting, and retrieval of events.

**Nodes Involved:**  
- Create Event  
- Create Event with Attendee  
- Update Event  
- Delete Event  
- Get Events  
- Calendar Agent1 (Langchain agent)  
- Calendar Agent (ToolWorkflow)  
- Success  
- Try Again1

**Node Details:**

- **Google Calendar Tool Nodes**  
  - Direct integration with Google Calendar API to manage events  
  - Create Event with Attendee supports adding participants  
  - Get Events retrieves calendar entries  
  - Failures: OAuth token expiry, API limits, invalid event data

- **Calendar Agent1**  
  - AI agent managing calendar-related queries and commands  
  - Connected to Google Gemini Chat Model3 for language understanding  
  - Outputs success or retry messages based on operation results

- **Calendar Agent (ToolWorkflow)**  
  - Higher-level agent node connected to main AI Agent for calendar tool usage

---

#### 1.5 Billing & Payments

**Overview:**  
Handles Stripe payment processing including customer creation, payment charges, discounts, and card management.

**Nodes Involved:**  
- Create Customer  
- Update Customer  
- Delete Customer  
- Get Customer  
- Get Customer Card  
- Create Charge  
- Create Discount Coupon  
- Billing Agent1 (Langchain agent)  
- Billing Agent (ToolWorkflow)  
- Simple Memory  
- Response1  
- Try Again2

**Node Details:**

- **Stripe Tool Nodes**  
  - Manage Stripe API interactions for customers, cards, charges, and coupons  
  - Connected as ai_tools to Billing Agent1  
  - Failures: API key invalid, network errors, payment declined

- **Billing Agent1**  
  - AI agent for handling payment-related interactions  
  - Utilizes Simple Memory for context  
  - Outputs response or retry instructions

- **Billing Agent (ToolWorkflow)**  
  - Connected to main AI Agent as a tool for billing operations

---

#### 1.6 Output & Response Delivery

**Overview:**  
Sends replies to users on respective platforms, manages typing indicators and response formatting.

**Nodes Involved:**  
- Reply To User1 (WhatsApp)  
- Reply To User (WhatsApp)  
- Facebook Graph API - Sales Agent Demo  
- Instagram Graph API - smb.sales.agent.demo  
- Send First Message (WhatsApp)  
- Sales Agent Demo - typing_on (Facebook Graph API)  
- Respond to Webhook - facebook post / get  
- Respond to Webhook - instagram post / get  
- Output - chat  
- Success  
- Try Again / Try Again1 / Try Again2  
- Response / Response1

**Node Details:**

- **Reply / Send Message Nodes**  
  - Send messages back via WhatsApp, Facebook, Instagram APIs  
  - Use webhook IDs for direct response correlation  
  - Include typing indicator node for Facebook to improve UX

- **Respond to Webhook Nodes**  
  - Handle HTTP response to Facebook and Instagram webhook calls  
  - Ensure proper acknowledgment of requests to avoid retries

- **Set Nodes (Response, Try Again, Success)**  
  - Prepare response messages or retry notifications  
  - Used for user feedback and error handling

---

#### 1.7 Memory & Knowledge Management

**Overview:**  
Stores conversational context and technical/sales knowledge to support RAG and contextual AI responses.

**Nodes Involved:**  
- Postgres Chat Memory  
- Window Buffer Memory  
- technical_and_sales_knowledge (Vector Store)  
- Embeddings Google Gemini1

**Node Details:**

- **Postgres Chat Memory & Window Buffer Memory**  
  - Store chat history and context for AI agents  
  - Postgres Chat Memory persists data, Window Buffer Memory holds recent context

- **technical_and_sales_knowledge**  
  - Vector store with embedded technical and sales knowledge documents  
  - Enables retrieval-augmented generation for accurate AI responses

- **Embeddings Google Gemini1**  
  - Generates embeddings for input texts to populate vector store

---

#### 1.8 Workflow Triggers & Subflows

**Overview:**  
Manages entry points and sub-workflow executions for modular design and external event handling.

**Nodes Involved:**  
- When Executed by Another Workflow (Execute Workflow Trigger)  
- CRM Agent2 (sub-workflow invoked here)  
- ToolWorkflow nodes (CRM Agent, Calendar Agent, Billing Agent)

**Node Details:**

- **When Executed by Another Workflow**  
  - Allows external workflows to trigger CRM operations with parameters  
  - Supports integration with other automation or external systems

- **ToolWorkflow Nodes**  
  - Encapsulate complex logic for CRM, Calendar, and Billing as reusable sub-workflows  
  - Connected as ai_tools to main AI Agent for modular AI tool invocation

---

### 3. Summary Table

| Node Name                          | Node Type                       | Functional Role                          | Input Node(s)                     | Output Node(s)                        | Sticky Note                         |
|-----------------------------------|--------------------------------|----------------------------------------|----------------------------------|-------------------------------------|-----------------------------------|
| WhatsApp Trigger                  | whatsappTrigger                | Receives WhatsApp messages              |                                  | Get Lead                            |                                   |
| Facebook Trigger                 | webhook                       | Receives Facebook webhook events        |                                  | Respond to Webhook - facebook post / get |                                   |
| Instagram Trigger                | webhook                       | Receives Instagram webhook events       |                                  | Respond to Webhook - instagram post / get |                                   |
| Handle Message Types             | switch                       | Routes message types                     | Get Lead, WhatsApp               | Edit Fields - chat1, WhatsApp Business Cloud, Reply To User1 |                                   |
| If is not echo - facebook        | if                           | Filters Facebook echo messages           | Respond to Webhook - facebook post | Edit Fields - facebook, Sales Agent Demo - typing_on |                                   |
| If is not echo - instagram       | if                           | Filters Instagram echo messages          | Respond to Webhook - instagram post | Edit Fields - instagram             |                                   |
| Edit Fields - chat1              | set                          | Prepares chat message data               | Handle Message Types             | No Operation, do nothing            |                                   |
| Edit Fields - facebook           | set                          | Prepares Facebook message data           | If is not echo - facebook        | No Operation, do nothing            |                                   |
| Edit Fields - instagram          | set                          | Prepares Instagram message data          | If is not echo - instagram       | No Operation, do nothing            |                                   |
| WhatsApp Business Cloud          | whatsApp                     | Sends WhatsApp messages                   | Handle Message Types             | HTTP Request                       |                                   |
| Reply To User1                  | whatsApp                     | Replies to WhatsApp user                   | Handle Message Types             | Get Lead                          |                                   |
| Reply To User                   | whatsApp                     | Replies to WhatsApp user                   | Switch1                        |                                   |                                   |
| Get Lead                       | postgres                     | Retrieves lead data                        | WhatsApp                        | Handle Message Types               |                                   |
| AI Agent                      | langchain.agent             | Main AI orchestrator                       | No Operation, do nothing, Postgres Chat Memory | Switch1                         |                                   |
| Switch1                      | switch                       | Routes AI Agent output to platform-specific responses | AI Agent                      | Reply To User, Facebook Graph API - Sales Agent Demo, Instagram Graph API - smb.sales.agent.demo, Output - chat |                                   |
| Gemini                        | langchain.lmChatGoogleGemini | AI language model for personalized messaging | Personalised First Message      | Personalised First Message         |                                   |
| Personalised First Message     | langchain.agent             | Generates personalized initial messages    | Create Contact1                | Send First Message                |                                   |
| Send First Message             | whatsApp                     | Sends first message on WhatsApp             | Personalised First Message      |                                   |                                   |
| OpenAI                        | langchain.openAi           | OpenAI language model for NLP tasks         | HTTP Request                  | Edit Fields - chat2               |                                   |
| HTTP Request                  | httpRequest                 | Makes HTTP calls (used with OpenAI)          | WhatsApp Business Cloud        | OpenAI                          |                                   |
| Postgres Chat Memory          | langchain.memoryPostgresChat | Stores conversation memory                   |                                 | AI Agent                        |                                   |
| Window Buffer Memory          | langchain.memoryBufferWindow | Stores short-term conversation context        |                                 | CRM Agent2                      |                                   |
| technical_and_sales_knowledge | langchain.toolVectorStore  | Stores technical/sales knowledge vectors      | Embeddings Google Gemini1       | AI Agent                        |                                   |
| Embeddings Google Gemini1     | langchain.embeddingsGoogleGemini | Generates embeddings for texts                |                                 | Postgres PGVector Store         |                                   |
| Postgres PGVector Store       | langchain.vectorStorePGVector | Vector store for knowledge base                | Embeddings Google Gemini1       | technical_and_sales_knowledge    |                                   |
| CRM Agent                    | langchain.toolWorkflow     | CRM operations sub-workflow                    | AI Agent                      | AI Agent                        |                                   |
| CRM Agent2                   | langchain.agent             | CRM agent for contact and opportunity management | When Executed by Another Workflow, List Records, Create Opportunity, etc. | Response, Try Again            |                                   |
| Get Events                   | googleCalendarTool         | Retrieves calendar events                       |                                 | Calendar Agent1                 |                                   |
| Create Event                 | googleCalendarTool         | Creates calendar events                          |                                 | Calendar Agent1                 |                                   |
| Create Event with Attendee   | googleCalendarTool         | Creates events with attendees                     |                                 | Calendar Agent1                 |                                   |
| Update Event                 | googleCalendarTool         | Updates calendar events                          |                                 | Calendar Agent1                 |                                   |
| Delete Event                 | googleCalendarTool         | Deletes calendar events                          |                                 | Calendar Agent1                 |                                   |
| Calendar Agent               | langchain.toolWorkflow     | Calendar management sub-workflow                   | AI Agent                      | AI Agent                        |                                   |
| Calendar Agent1              | langchain.agent             | Calendar agent handling event operations         | Get Events, Create Event, etc. | Success, Try Again1             |                                   |
| Success                     | set                        | Success response node                             | Calendar Agent1               |                                 |                                   |
| Try Again1                  | set                        | Retry response node                               | Calendar Agent1               |                                 |                                   |
| Create Customer             | stripeTool                 | Creates Stripe customer                           |                                 | Billing Agent1                 |                                   |
| Update Customer             | stripeTool                 | Updates Stripe customer                           |                                 | Billing Agent1                 |                                   |
| Delete Customer             | stripeTool                 | Deletes Stripe customer                           |                                 | Billing Agent1                 |                                   |
| Get Customer                | stripeTool                 | Retrieves Stripe customer info                     |                                 | Billing Agent1                 |                                   |
| Get Customer Card           | stripeTool                 | Retrieves Stripe customer card info                |                                 | Billing Agent1                 |                                   |
| Create Charge               | stripeTool                 | Creates Stripe charge                              |                                 | Billing Agent1                 |                                   |
| Create Discount Coupon      | stripeTool                 | Creates Stripe discount coupon                      |                                 | Billing Agent1                 |                                   |
| Billing Agent               | langchain.toolWorkflow     | Billing operations sub-workflow                     | AI Agent                      | AI Agent                        |                                   |
| Billing Agent1              | langchain.agent             | Billing agent handling payment processing           | Create Charge, Create Customer, etc. | Response1, Try Again2          |                                   |
| Response                    | set                        | Sets response message                              | CRM Agent2                   |                                 |                                   |
| Try Again                   | set                        | Sets retry message                                | CRM Agent2                   |                                 |                                   |
| Response1                   | set                        | Sets billing response message                      | Billing Agent1               |                                 |                                   |
| Try Again2                  | set                        | Sets billing retry message                          | Billing Agent1               |                                 |                                   |
| When Executed by Another Workflow | executeWorkflowTrigger | External workflow trigger for CRM                  |                                 | CRM Agent2                    |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Incoming Message Triggers:**
   - Add **WhatsApp Trigger** node configured with proper webhook ID.
   - Add **Facebook Trigger** node with webhook ID.
   - Add **Instagram Trigger** node with webhook ID.

2. **Add Message Type Handling:**
   - Add a **Switch** node named "Handle Message Types" to route based on message platform/type.
   - Connect WhatsApp Trigger → Get Lead → Handle Message Types.
   - Connect Facebook and Instagram triggers to corresponding Respond to Webhook nodes.

3. **Add Echo Filters:**
   - Add **If** nodes "If is not echo - facebook" and "If is not echo - instagram" to filter duplicates.
   - Connect Facebook and Instagram post webhook responses through these filters.

4. **Add Edit Fields Nodes:**
   - Add **Set** nodes to normalize incoming chat, Facebook, and Instagram messages.
   - Connect outputs of echo filters and switch node accordingly.

5. **Configure WhatsApp Business Cloud Node:**
   - Add WhatsApp Business Cloud node for sending messages.
   - Connect from Edit Fields - chat and Handle Message Types.

6. **Add AI Agents and Language Models:**
   - Add **AI Agent** node as the central orchestrator.
   - Connect Postgres Chat Memory as AI memory to AI Agent.
   - Connect AI Agent output to Switch1 for routing responses by platform.
   - Add Langchain Google Gemini and OpenAI nodes as language models.
   - Connect Gemini node output to Personalised First Message agent node.
   - Connect Personalised First Message output to Send First Message WhatsApp node.

7. **Set Up CRM Integration:**
   - Add Postgres nodes for Get Lead, Create Contact, Update Contact, Delete Contact, Create Opportunity, Update Opportunity, Delete Opportunity, and List Records.
   - Add CRM Agent2 Langchain agent node.
   - Connect When Executed by Another Workflow trigger node to CRM Agent2.
   - Connect CRM Agent2 to response and retry set nodes.
   - Wire CRM Postgres nodes as ai_tools to CRM Agent2.

8. **Set Up Calendar Management:**
   - Add Google Calendar Tool nodes for Create Event, Create Event with Attendee, Update Event, Delete Event, Get Events.
   - Add Calendar Agent1 Langchain agent node with Google Gemini Chat Model3.
   - Connect Calendar tool nodes as ai_tools to Calendar Agent1.
   - Connect Calendar Agent1 output to Success and Try Again1 set nodes.
   - Connect Calendar Agent1 as ai_tool to main AI Agent.

9. **Configure Billing & Payments:**
   - Add Stripe Tool nodes for Create Customer, Update Customer, Delete Customer, Get Customer, Get Customer Card, Create Charge, Create Discount Coupon.
   - Add Billing Agent1 Langchain agent node with Google Gemini Chat Model4.
   - Connect Stripe nodes as ai_tools to Billing Agent1.
   - Add Simple Memory node for Billing context.
   - Connect Billing Agent1 output to Response1 and Try Again2 set nodes.
   - Connect Billing Agent1 as ai_tool to main AI Agent.

10. **Add Knowledge Base and Memory Stores:**
    - Add Embeddings Google Gemini1 node for text embedding.
    - Add Postgres PGVector Store node connected to embeddings.
    - Add technical_and_sales_knowledge vector store node connected to PGVector Store.
    - Connect technical_and_sales_knowledge as ai_tool to main AI Agent.

11. **Set Up Response Delivery:**
    - Add Reply To User and Reply To User1 WhatsApp nodes.
    - Add Facebook Graph API and Instagram Graph API nodes for message replies.
    - Add Sales Agent Demo - typing_on Facebook API node for typing indicators.
    - Connect Switch1 node outputs to these response nodes accordingly.

12. **Add No Operation Nodes:**
    - Add No Operation nodes as placeholders for unused branches.

13. **Connect All Nodes According to Dependencies:**
    - Ensure all nodes are connected as per the detailed connections above.
    - Set webhook IDs, credentials for WhatsApp Business Cloud, Facebook, Instagram, Google Calendar, Stripe, PostgreSQL, OpenAI, Google Gemini.

14. **Validate and Test:**
    - Test each platform trigger with sample messages.
    - Verify AI agents respond and call appropriate sub-workflows.
    - Confirm CRM, Calendar, and Billing operations function correctly.
    - Check error handling and retry logic.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow leverages advanced AI models including Google Gemini and OpenAI via Langchain integration for multi-agent orchestration. | AI Models & Langchain integration                |
| The workflow uses PostgreSQL with PGVector extension for vector embeddings to enable RAG (Retrieval-Augmented Generation).          | RAG implementation with Postgres PGVector       |
| Stripe nodes require proper API key credentials and permissions for customer/payment management.                                     | Stripe API integration                            |
| Google Calendar nodes require OAuth2 credentials with calendar scope granted.                                                        | Google Calendar OAuth2 setup                      |
| WhatsApp Business Cloud and Facebook/Instagram APIs require proper webhook configuration and app permissions.                      | Social media platform API setup                   |
| The workflow includes modular sub-workflows for CRM, Calendar, and Billing to allow reusability and separation of concerns.        | Modular workflow design                            |
| Careful error handling is implemented using Try Again and Success nodes to improve resilience.                                       | Error handling and retry logic                     |
| Sticky Notes nodes are present for internal documentation or future comments; they currently contain no content.                    | Internal workflow notes                            |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow built with n8n, adhering strictly to content policies without illegal or protected elements. All data handled is legal and public.