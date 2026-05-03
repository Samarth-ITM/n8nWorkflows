Create a WhatsApp Customer Support Bot with OpenAI, Calendar & Email Integration

https://n8nworkflows.xyz/workflows/create-a-whatsapp-customer-support-bot-with-openai--calendar---email-integration-8287


# Create a WhatsApp Customer Support Bot with OpenAI, Calendar & Email Integration

### 1. Workflow Overview

This workflow implements a **WhatsApp Customer Support Bot** integrated with OpenAI language models, Google Calendar, Gmail, and a Supabase vector store for knowledge base retrieval. Its primary purpose is to provide automated, intelligent customer service on WhatsApp by understanding user queries, managing calendar events, retrieving FAQ or pricing information, and sending emails.

The workflow is logically divided into the following main blocks:

- **1.1 Input Reception & Formatting**: Captures incoming WhatsApp messages via Twilio, extracts and formats user input and metadata.
- **1.2 Main Orchestrator Agent (WhatsApp AI Support Agent)**: Acts as the core decision-maker, interpreting user queries using an OpenAI-powered agent and routing tasks to specialized sub-agents.
- **1.3 Sub-Agents for Task Handling**:
    - **Calendar Agent**: Manages Google Calendar events, including querying availability, creating, updating, and deleting events.
    - **Knowledge Base Agent**: Retrieves structured answers from a Supabase vector store embedding database, primarily for FAQs and pricing.
    - **Email Agent**: Drafts and sends professional emails using Gmail based on user requests.
- **1.4 External Service Integrations**:
    - **Google Calendar Nodes**: For event management operations.
    - **Supabase Vector Store & OpenAI Embeddings**: For knowledge base retrieval.
    - **Gmail Node**: For sending emails.
    - **Twilio Nodes**: For receiving messages and sending WhatsApp replies.
- **1.5 Conversation Memory**: Stores and retrieves conversation context per user using a Postgres-based chat memory node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Formatting

- **Overview**: This block receives incoming WhatsApp messages via Twilio, extracts the message text and metadata, and formats them for downstream processing.
- **Nodes Involved**:  
  - `Incoming WhatsApp Message`  
  - `Format Incoming Message`

- **Node Details**:

  - **Incoming WhatsApp Message**  
    - Type: Twilio Trigger (webhook)  
    - Role: Listens for inbound WhatsApp messages via Twilio API.  
    - Configuration: Triggers on inbound message events (`com.twilio.messaging.inbound-message.received`).  
    - Inputs: External webhook trigger from Twilio.  
    - Outputs: Raw Twilio message JSON.  
    - Failures: Webhook misconfiguration, Twilio auth failures, network timeouts.  

  - **Format Incoming Message**  
    - Type: Set Node  
    - Role: Extracts user message text, sender mobile number (removes 'whatsapp:' prefix), and bot number for reply.  
    - Configuration: Assigns three variables:  
      - `usersMessage` = message body text  
      - `usersMobileNumber` = sender phone number without prefix  
      - `WhatsappAiAgentNumber` = bot's WhatsApp number without prefix  
    - Inputs: Raw Twilio message JSON from previous node.  
    - Outputs: Formatted JSON with these three fields.  
    - Failures: Expression errors if input data missing or malformed.

---

#### 2.2 Main Orchestrator Agent (WhatsApp AI Support Agent)

- **Overview**: This is the central AI agent that interprets the user's message, decides which sub-agent(s) to invoke (Calendar, Knowledge Base, Email), and orchestrates the overall conversation flow. It uses OpenAI GPT-4.1-mini as the language model and maintains conversational context using Postgres memory.
- **Nodes Involved**:  
  - `WhatsApp AI Support Agent`  
  - `Conversation Memory (Postgres)`  
  - `Send WhatsApp Reply`

- **Node Details**:

  - **WhatsApp AI Support Agent**  
    - Type: Langchain Agent Node  
    - Role: Receives formatted user input, applies a detailed system prompt defining roles, sub-agent delegation rules, example conversations, and conversation policies.  
    - Configuration:  
      - Uses GPT-4.1-mini model.  
      - System message defines agent behavior: orchestrator delegating to Calendar, Knowledge Base, Email sub-agents, never acts alone on tasks.  
      - Handles multi-tool orchestration, maintains politeness, confirms ambiguous info, and presents results in WhatsApp-friendly style.  
    - Inputs: `usersMessage` from Format Incoming Message, session context memory.  
    - Outputs: Structured queries to sub-agents and final user response.  
    - Failures: Prompt or model errors, missing session memory, unhandled user inputs.  

  - **Conversation Memory (Postgres)**  
    - Type: Postgres Chat Memory Node  
    - Role: Stores and retrieves user conversation context keyed by user’s mobile number.  
    - Configuration:  
      - Session key derived from `usersMobileNumber`.  
      - Context window length set to 50 messages.  
    - Inputs: User message and response pairs.  
    - Outputs: Context history for agent.  
    - Failures: Database connectivity issues, key conflicts, data corruption.  

  - **Send WhatsApp Reply**  
    - Type: Twilio Node  
    - Role: Sends the final response message back to the user via WhatsApp.  
    - Configuration:  
      - `to` set to user mobile number from formatted input.  
      - `from` set to bot’s WhatsApp number.  
      - Message content from `WhatsApp AI Support Agent` output.  
      - Sends via Twilio WhatsApp channel.  
    - Inputs: Final text response from agent.  
    - Outputs: WhatsApp message sent confirmation.  
    - Failures: Twilio auth errors, invalid phone numbers, rate limits.

---

#### 2.3 Calendar Agent & Google Calendar Integration

- **Overview**: This block manages calendar-related requests such as checking availability, creating, updating, and deleting events in Google Calendar, ensuring precise handling of event details and conflict checking.
- **Nodes Involved**:  
  - `Calendar Agent`  
  - `When Executed by Another Workflow`  
  - `Get many events in Google Calendar`  
  - `Create an event in Google Calendar`  
  - `Delete an event in Google Calendar`  
  - `Create an event in Google Calendar1`  
  - `Calendar Tool` (Tool Workflow node invoking sub-workflow)

- **Node Details**:

  - **Calendar Agent**  
    - Type: Langchain Agent  
    - Role: Processes calendar-related user queries, translates into calendar operations (get, create, delete, update).  
    - Configuration:  
      - GPT-4.1-mini with extensive system prompt detailing calendar management rules, duration fixed at 30 minutes, conflict checking, and confirmation steps.  
      - Outputs concise confirmations or error messages.  
    - Inputs: Query string from orchestrator or external workflow.  
    - Outputs: Calendar operation commands with structured data.  
    - Failures: Date/time parsing errors, Google API errors, missing event IDs for update/delete.  

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for calendar sub-workflow calls from the main workflow.  
    - Configuration: Passthrough input source.  
    - Inputs: Called from main workflow.  
    - Outputs: Trigger to Calendar Agent.  

  - **Google Calendar Nodes** (`Get many events in Google Calendar`, `Create an event in Google Calendar`, `Delete an event in Google Calendar`, `Create an event in Google Calendar1`)  
    - Type: Google Calendar Tool Nodes  
    - Roles: Perform calendar operations via Google Calendar API.  
    - Configuration:  
      - Use OAuth2 credentials linked to Google Calendar account.  
      - Operations include getAll events (with timeMin/timeMax filters), create event (start/end date-times), delete event by eventId.  
      - Parameters such as eventId, start, end are dynamically populated from AI-generated variables.  
    - Inputs: Structured event data from Calendar Agent.  
    - Outputs: API responses confirming operations.  
    - Failures: OAuth token expiry, API quota limits, invalid event IDs, network issues.  

  - **Calendar Tool**  
    - Type: Langchain Tool Workflow (Sub-Workflow)  
    - Role: Represents calendar functionality as a callable tool from the main agent.  
    - Configuration: Points to a separate workflow (ID: `ZnQPXmxkpwVly7Ag`).  
    - Inputs/Outputs: Accepts structured calendar commands, returns operation results.  
    - Failures: Sub-workflow errors, input format mismatches.

---

#### 2.4 Knowledge Base Agent & Supabase Integration

- **Overview**: This block handles FAQ, pricing, and diagnostic queries by retrieving relevant information from a Supabase vector store using OpenAI embeddings, then responding with a structured summary.
- **Nodes Involved**:  
  - `Knowledge Base Agent`  
  - `OpenAI Chat Model2`  
  - `Supabase Vector Store`  
  - `Embeddings OpenAI`  
  - `Knowledge Base Tool` (Tool Workflow node)

- **Node Details**:

  - **Knowledge Base Agent**  
    - Type: Langchain Agent  
    - Role: Receives query input from main agent, queries knowledge base for relevant info, returns structured, fact-based answers only.  
    - Configuration:  
      - GPT-4.1-mini model with a system prompt specifying input/output formats, retrieval guidelines, and safety-critical flags.  
      - Emphasizes no fabrication, only fact-based responses.  
    - Inputs: Customer query details.  
    - Outputs: Structured knowledge base response with likely faults, fees, price range, safety info.  
    - Failures: Retrieval failures, prompt misinterpretation, model errors.  

  - **OpenAI Chat Model2**  
    - Type: Langchain LM Chat OpenAI  
    - Role: Language model used by Knowledge Base Agent to process query and generate response.  
    - Inputs: Embedded query results.  
    - Outputs: Formatted answer text.  

  - **Supabase Vector Store**  
    - Type: Langchain Vector Store Supabase  
    - Role: Retrieves relevant documents from Supabase embedding database based on vector similarity.  
    - Configuration: Table named `documents`, retrieval mode as a tool.  
    - Inputs: OpenAI embeddings of user query.  
    - Outputs: Matched documents for knowledge base agent.  
    - Failures: Supabase API errors, misconfigured table, empty results.  

  - **Embeddings OpenAI**  
    - Type: Langchain Embeddings OpenAI  
    - Role: Generates vector embeddings from user queries for similarity search in Supabase.  
    - Inputs: Raw query text.  
    - Outputs: Vector embedding.  
    - Failures: API key issues, rate limits.  

  - **Knowledge Base Tool**  
    - Type: Langchain Tool Workflow (Sub-Workflow)  
    - Role: Represents the knowledge base functionality as a callable tool by main agent.  
    - Configuration: Points to a separate workflow (ID: `VzMwnIb1MrJgCvNb`).  
    - Inputs/Outputs: Query input, structured knowledge response.  

---

#### 2.5 Email Agent & Gmail Integration

- **Overview**: This block automates creation and sending of professional emails based on WhatsApp requests, such as confirmations or follow-ups.
- **Nodes Involved**:  
  - `Email Agent`  
  - `OpenAI Chat Model3`  
  - `Send a message in Gmail`  
  - `Email Tool` (Tool Workflow node)

- **Node Details**:

  - **Email Agent**  
    - Type: Langchain Agent  
    - Role: Receives instructions to draft and send emails, generates full email drafts, and sends only upon explicit instruction.  
    - Configuration:  
      - GPT-4.1-mini with system prompt defining input structure (sender, recipient, subject, context, tone, action), task rules, and formatting.  
      - Outputs “STATUS: Email Sent ✅” upon sending.  
    - Inputs: Email request details from main agent.  
    - Outputs: Drafted email and send confirmation.  
    - Failures: Email address validation, model generation errors.  

  - **OpenAI Chat Model3**  
    - Type: Langchain LM Chat OpenAI  
    - Role: Language model used to generate email content based on input format.  

  - **Send a message in Gmail**  
    - Type: Gmail Node  
    - Role: Sends email messages via Gmail API with OAuth2 credentials.  
    - Configuration:  
      - Dynamically sets To, Subject, Message body from AI-generated variables.  
      - Uses plain text email type.  
    - Inputs: Email draft and recipient info from Email Agent.  
    - Outputs: Gmail send confirmation.  
    - Failures: OAuth token expiry, send limits, invalid addresses.  

  - **Email Tool**  
    - Type: Langchain Tool Workflow (Sub-Workflow)  
    - Role: Email sending functionality exposed as callable tool.  
    - Configuration: Points to workflow ID `asdMM8Ip0p1BpgKo`.  

---

### 3. Summary Table

| Node Name                    | Node Type                                   | Functional Role                             | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                                 |
|------------------------------|---------------------------------------------|---------------------------------------------|------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Incoming WhatsApp Message     | Twilio Trigger                              | Receives inbound WhatsApp messages          | External webhook              | Format Incoming Message       |                                                                                                                             |
| Format Incoming Message       | Set                                         | Extracts and formats message and metadata   | Incoming WhatsApp Message     | WhatsApp AI Support Agent     |                                                                                                                             |
| WhatsApp AI Support Agent     | Langchain Agent                             | Core orchestrator AI agent                   | Format Incoming Message, Conversation Memory | Send WhatsApp Reply           | 🤖 **Main WhatsApp AI Agent** - Core assistant handling WhatsApp messages, routing to sub-agents, manages context            |
| Conversation Memory (Postgres)| Postgres Chat Memory                        | Stores conversation context per user        | WhatsApp AI Support Agent     | WhatsApp AI Support Agent     |                                                                                                                             |
| Send WhatsApp Reply           | Twilio Node                                | Sends reply back to WhatsApp user            | WhatsApp AI Support Agent     |                             |                                                                                                                             |
| Calendar Agent               | Langchain Agent                             | Handles calendar-related queries             | When Executed by Another Workflow | Google Calendar nodes         | 📅 **Calendar Agent** - Manages Google Calendar events: check, create, delete                                                |
| When Executed by Another Workflow | Execute Workflow Trigger                  | Entry trigger for calendar sub-workflow      | WhatsApp AI Support Agent     | Calendar Agent               |                                                                                                                             |
| Get many events in Google Calendar | Google Calendar Tool                      | Retrieves calendar events                     | Calendar Agent               | Calendar Agent               |                                                                                                                             |
| Create an event in Google Calendar | Google Calendar Tool                      | Creates calendar event                        | Calendar Agent               | Calendar Agent               |                                                                                                                             |
| Delete an event in Google Calendar | Google Calendar Tool                      | Deletes calendar event                        | Calendar Agent               | Calendar Agent               |                                                                                                                             |
| Create an event in Google Calendar1 | Google Calendar Tool                      | Additional create event node (likely duplicate or variant) | Calendar Agent               | Calendar Agent               |                                                                                                                             |
| Calendar Tool               | Langchain Tool Workflow                     | Calendar sub-agent called by main agent      | WhatsApp AI Support Agent     | WhatsApp AI Support Agent     |                                                                                                                             |
| Knowledge Base Agent         | Langchain Agent                             | Retrieves FAQ and pricing info from knowledge base | Knowledge Base Tool           |                             | 📚 **Knowledge Base Agent** - Answers FAQs using Supabase vector DB and OpenAI embeddings                                     |
| OpenAI Chat Model2           | Langchain LM Chat OpenAI                    | Language model for knowledge base agent      | Knowledge Base Agent          | Knowledge Base Agent          |                                                                                                                             |
| Supabase Vector Store        | Langchain Vector Store Supabase             | Retrieves relevant documents for FAQ queries | Embeddings OpenAI            | Knowledge Base Agent          |                                                                                                                             |
| Embeddings OpenAI            | Langchain Embeddings OpenAI                  | Creates vector embeddings for query          | Knowledge Base Agent          | Supabase Vector Store         |                                                                                                                             |
| Knowledge Base Tool          | Langchain Tool Workflow                     | Knowledge base sub-agent called by main agent | WhatsApp AI Support Agent     | WhatsApp AI Support Agent     |                                                                                                                             |
| Email Agent                 | Langchain Agent                             | Drafts and sends emails                       | Email Tool                   | Send a message in Gmail       | 📧 **Email Agent** - Drafts and sends emails using Gmail API                                                                |
| OpenAI Chat Model3           | Langchain LM Chat OpenAI                    | Language model for email generation           | Email Agent                  | Email Agent                  |                                                                                                                             |
| Send a message in Gmail      | Gmail Node                                  | Sends email messages                          | Email Agent                  |                             |                                                                                                                             |
| Email Tool                  | Langchain Tool Workflow                     | Email sub-agent called by main agent          | WhatsApp AI Support Agent     | WhatsApp AI Support Agent     |                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Incoming WhatsApp Message (Twilio Trigger) node**  
   - Set trigger to inbound WhatsApp messages (`com.twilio.messaging.inbound-message.received`).  
   - Connect Twilio account credentials with WhatsApp enabled.  

2. **Create Format Incoming Message (Set) node**  
   - Add three string variables:  
     - `usersMessage` = `{{$json.data.body}}`  
     - `usersMobileNumber` = `{{$json.data.from.replaceAll('whatsapp:', '')}}`  
     - `WhatsappAiAgentNumber` = `{{$json.data.to.replaceAll('whatsapp:', '')}}`  
   - Connect output of Incoming WhatsApp Message to this node.

3. **Create Conversation Memory (Postgres) node**  
   - Set `sessionKey` to `{{$json.usersMobileNumber}}`.  
   - Use Postgres credentials.  
   - Configure context window length to 50.  

4. **Create WhatsApp AI Support Agent (Langchain Agent) node**  
   - Use GPT-4.1-mini model with OpenAI credentials.  
   - Paste the provided detailed system prompt describing orchestrator role, sub-agents, example flows, and rules.  
   - Connect memory node as `ai_memory` input.  
   - Connect Format Incoming Message as input.  

5. **Create Send WhatsApp Reply (Twilio) node**  
   - Set `to` to `{{$node["Format Incoming Message"].json.usersMobileNumber}}`.  
   - Set `from` to `{{$node["Format Incoming Message"].json.WhatsappAiAgentNumber}}`.  
   - Set message content to output of WhatsApp AI Support Agent.  
   - Use Twilio credentials with WhatsApp enabled.  
   - Connect WhatsApp AI Support Agent output to this node.  

6. **Create Calendar Agent (Langchain Agent) node**  
   - Use GPT-4.1-mini model with OpenAI credentials.  
   - Paste detailed calendar management system prompt.  
   - Connect an Execute Workflow Trigger node as entry point (`When Executed by Another Workflow`).  
   - Create Google Calendar nodes:  
     - `Get many events in Google Calendar` for getAll operation with OAuth2 credentials.  
     - `Create an event in Google Calendar` nodes for creating events.  
     - `Delete an event in Google Calendar` node for deleting events by eventId.  
   - Connect corresponding outputs/inputs as per calendar workflow logic.  

7. **Create Calendar Tool (Langchain Tool Workflow) node**  
   - Point to calendar sub-workflow ID.  
   - Expose inputs/outputs for structured calendar commands.  

8. **Create Knowledge Base Agent (Langchain Agent) node**  
   - Use GPT-4.1-mini model with OpenAI credentials.  
   - Paste knowledge base system prompt.  
   - Connect OpenAI Chat Model2 node.  
   - Create Supabase Vector Store node with Supabase credentials and configure with `documents` table.  
   - Create Embeddings OpenAI node to generate query embeddings.  
   - Connect Embeddings OpenAI to Supabase Vector Store; connect Supabase results to Knowledge Base Agent.  

9. **Create Knowledge Base Tool (Langchain Tool Workflow) node**  
   - Link to knowledge base sub-workflow ID.  

10. **Create Email Agent (Langchain Agent) node**  
    - Use GPT-4.1-mini model with OpenAI credentials.  
    - Paste detailed email drafting system prompt.  
    - Connect OpenAI Chat Model3 node.  
    - Create Send a message in Gmail node with Gmail OAuth2 credentials.  
    - Map email fields dynamically from Email Agent output.  

11. **Create Email Tool (Langchain Tool Workflow) node**  
    - Link to email sub-workflow ID.  

12. **Connect WhatsApp AI Support Agent to all tool workflows via ai_tool inputs**:  
    - Calendar Tool  
    - Knowledge Base Tool  
    - Email Tool  

13. **Connect all sub-agents’ language models and tools as per the designed workflow connections**.  

14. **Test end-to-end**: Send WhatsApp message → Format → Main agent → Sub-agent(s) → Operations → Reply sent.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| 🤖 **Main WhatsApp AI Agent** is the orchestrator that routes WhatsApp user queries to specialized sub-agents based on intent and task type.                                                                                                                                                                                                                 | Sticky Note near WhatsApp AI Support Agent node                  |
| 📅 **Calendar Agent** uses Google Calendar API to manage scheduling tasks, always confirming details and checking for conflicts.                                                                                                                                                                                                                             | Sticky Note near Calendar Agent nodes                            |
| 📚 **Knowledge Base Agent** retrieves FAQ and pricing info from a Supabase vector store enhanced with OpenAI embeddings, ensuring fact-based answers only.                                                                                                                                                                                                   | Sticky Note near Knowledge Base Agent nodes                      |
| 📧 **Email Agent** drafts and sends professional emails using Gmail API, following strict input formats and confirmation rules.                                                                                                                                                                                                                                | Sticky Note near Email Agent nodes                               |
| Workflow uses **OpenAI GPT-4.1-mini** model variants for natural language understanding and generation across agents, requiring valid OpenAI API credentials.                                                                                                                                                                                                  | Credential requirement                                           |
| WhatsApp integration is via **Twilio API** with WhatsApp sandbox or production numbers, requiring proper webhook setup and credentials.                                                                                                                                                                                                                      | Twilio account setup                                             |
| Conversation memory uses **Postgres** to maintain context per user, critical for multi-turn dialog coherence.                                                                                                                                                                                                                                                 | Postgres credential and setup                                   |
| Google Calendar integration requires OAuth2 credentials authorized for the target calendar.                                                                                                                                                                                                                                                                  | Google Calendar OAuth2 credentials                               |
| Supabase vector store requires a configured table `documents` with pre-loaded embeddings for knowledge base content.                                                                                                                                                                                                                                         | Supabase account and data preparation                            |
| Gmail node requires OAuth2 credentials with send email permissions for the connected Gmail account.                                                                                                                                                                                                                                                          | Gmail OAuth2 credentials                                         |
| The sub-workflows (Calendar Tool, Knowledge Base Tool, Email Tool) must be created separately and linked via workflow IDs as per the tool nodes.                                                                                                                                                                                                            | Sub-workflow setup required                                      |

---

**Disclaimer**: The provided text is extracted exclusively from an automated workflow designed using n8n, respecting all applicable content policies. It does not contain any illegal, offensive, or protected elements. All processed data is legal and public.