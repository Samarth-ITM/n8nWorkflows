Deploy a Google Gemini AI Assistant on WhatsApp with Email & Calendar Tools

https://n8nworkflows.xyz/workflows/deploy-a-google-gemini-ai-assistant-on-whatsapp-with-email---calendar-tools-8826


# Deploy a Google Gemini AI Assistant on WhatsApp with Email & Calendar Tools

### 1. Workflow Overview

This workflow deploys a sophisticated AI assistant on WhatsApp leveraging Google Gemini AI models, integrated with email and calendar management tools. It is designed for users who want to interact conversationally via WhatsApp with an AI that can handle general chat, manage emails (send, draft, reply, label, mark unread, search), and control calendar events (create, update, delete, check availability).

The workflow is logically divided into four main blocks:

- **1.1 WhatsApp Input Reception:** Captures incoming WhatsApp messages to trigger the assistant.
- **1.2 Personal Assistant Core (Manager Agent):** Processes user input with a central AI agent that routes tasks intelligently.
- **1.3 Email Tool Block:** Specialized agent and nodes handling email-related requests and actions.
- **1.4 Calendar Tool Block:** Specialized agent and nodes managing calendar events and scheduling.

Each block uses a set of nodes including AI language models (Google Gemini), memory buffers, and service-specific API nodes integrated via OAuth2 or API keys.

---

### 2. Block-by-Block Analysis

#### 2.1 WhatsApp Input Reception

- **Overview:** Listens for WhatsApp messages via webhook, triggering the workflow for each incoming message.
- **Nodes Involved:**  
  - WhatsApp Trigger  
  - Send message  
  - Sticky Note (for annotation)

- **Node Details:**  
  - **WhatsApp Trigger**  
    - Type: Trigger node for WhatsApp Business API messages  
    - Configuration: Listens for message updates, configured with webhook ID  
    - Input: External WhatsApp message event  
    - Output: JSON with message content and metadata  
    - Failure Potential: Webhook misconfiguration, API authentication errors, rate limiting  
  - **Send message**  
    - Type: WhatsApp node to send messages back to users  
    - Configuration: Sends message text extracted from AI response output to a specified recipient phone number via WhatsApp API  
    - Key expression: `textBody` set dynamically to AI agent’s output (`={{ $json.output }}`)  
    - Input: AI agent’s response message  
    - Output: Confirmation of sent message  
    - Failure Potential: Incorrect phone number format, API quota exceeded, auth errors  
  - **Sticky Note**  
    - Provides a label “Whatsapp Trigger” for clarity in the workflow UI.

#### 2.2 Personal Assistant Core (Manager Agent)

- **Overview:** Central conversational AI agent that interprets user messages, maintains context, and delegates tasks to email and calendar sub-agents or general chat model.
- **Nodes Involved:**  
  - Personal Agent (LangChain Agent node)  
  - Simple Memory (Memory Buffer Window)  
  - Google Gemini Chat Model (for general conversation)  
  - Sticky Notes (for block annotation)

- **Node Details:**  
  - **Personal Agent**  
    - Type: LangChain agent node, orchestrating conversation and delegation  
    - Configuration: Uses a custom system message defining a witty, proactive assistant persona ("Main") with instructions on saving memory, calling sub-agents, and handling errors by scheduling retries  
    - Input: User message from WhatsApp Trigger  
    - Output: Text response sent to WhatsApp Send message node  
    - Error Handling: Configured to continue output on failure to avoid dead ends  
    - Potential Failures: AI model response delays, expression errors in prompt, memory access issues  
  - **Simple Memory**  
    - Type: LangChain memory buffer, stores conversation context with a window size of 10 messages  
    - Configuration: Session key set to WhatsApp API access token, enabling context persistence per user session  
    - Input: User messages and AI responses  
    - Output: Memory context passed to Personal Agent  
    - Edge Cases: Memory overflow if window too small; context loss if session key misconfigured  
  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini chat model node  
    - Configuration: Uses "models/gemini-2.5-pro" as the AI model for conversation  
    - Credentials: Google PaLM API account configured  
    - Input: Prompt from Personal Agent  
    - Output: Chat message response  
    - Potential Failures: API quota exhaustion, network issues, invalid credentials  
  - **Sticky Notes**  
    - Labels: “Chatbot Model” and “Receive Message from Whatsapp” provide visual context.

#### 2.3 Email Tool Block

- **Overview:** Handles email management tasks such as sending, drafting, replying, labeling, marking unread, and searching emails through a specialized email agent using Gmail API.
- **Nodes Involved:**  
  - email_agent (LangChain Agent Tool)  
  - Google Gemini Chat Model4 (AI language model for email context)  
  - Window Buffer Memory2 (Email-specific memory buffer)  
  - Gmail nodes: Send Email, Create Draft, Email Reply, Get Labels, Label Emails, Mark Unread, Gmail (search emails)  
  - Sticky Note4 (annotation)

- **Node Details:**  
  - **email_agent**  
    - Type: LangChain agent tool node specialized for email tasks  
    - Configuration: System message provides detailed email rules (sending, drafting, labeling, replying) and enforces plain-text email body formatting with signature  
    - Input: Text from Personal Agent requesting email operations  
    - Output: Parsed commands and data for Gmail nodes  
    - Output parser enabled to structure AI responses into actionable commands  
    - Potential Failures: Parsing errors, missing recipient data, Gmail API errors  
  - **Google Gemini Chat Model4**  
    - Type: Google Gemini AI model configured for email tasks  
    - Credentials: Google PaLM API  
    - Role: Provides natural language understanding specific to email domain  
  - **Window Buffer Memory2**  
    - Type: Memory buffer tied to email agent session  
    - Manages context specifically for email conversations  
  - **Gmail Nodes**  
    - Send Email: Sends emails immediately  
    - Create Draft: Creates email drafts only if requested  
    - Email Reply: Sends replies to specific message IDs  
    - Get Labels: Retrieves available Gmail labels  
    - Label Emails: Adds labels to emails  
    - Mark Unread: Marks emails as unread by message ID  
    - Gmail (Get All Emails): Retrieves emails with optional filters and limits  
    - All nodes authenticate via OAuth2 Gmail credentials  
    - Edge Cases: OAuth2 token expiration, API rate limits, malformed inputs  
  - **Sticky Note4**  
    - Annotates the Email Tool block for clarity.

#### 2.4 Calendar Tool Block

- **Overview:** Manages Google Calendar events including creation (with or without attendees), updates, deletions, availability checks, and retrieval of single or multiple events.
- **Nodes Involved:**  
  - calendar_agent (LangChain Agent Tool)  
  - Google Gemini Chat Model5 (AI model for calendar tasks)  
  - Google Calendar nodes: Get all event, Delete event, Get a single event, Update event, Availability operation agent, Create Event with attendee, Create Event without attendee  
  - Sticky Note5 (annotation)

- **Node Details:**  
  - **calendar_agent**  
    - Type: LangChain agent tool specialized for calendar management  
    - Configuration: System message defines calendar management capabilities (checking availability, scheduling, rescheduling, reminders) with instructions to output event name and time after creation  
    - Input: Text commands parsed from Personal Agent  
    - Output: Structured commands to Google Calendar nodes  
    - Output parser enabled for structured data  
    - Potential Failures: Calendar API quota, invalid event parameters, network issues  
  - **Google Gemini Chat Model5**  
    - Type: Google Gemini AI model for calendar tasks  
    - Credentials: Google PaLM API  
  - **Google Calendar Nodes**  
    - Get all event: Retrieves events within a date range  
    - Delete event: Deletes event by ID  
    - Get a single event: Retrieves event details by ID  
    - Update event: Updates event fields, optionally with reminders  
    - Availability operation agent: Checks calendar availability over a time range  
    - Create Event with attendee: Creates event including attendees list  
    - Create Event without attendee: Creates event without attendees  
    - All nodes authenticate via Google Calendar OAuth2 credentials  
    - Edge Cases: Overlapping events, invalid attendee emails, permission errors  
  - **Sticky Note5**  
    - Marks the Calendar Tool block.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                            | Input Node(s)         | Output Node(s)       | Sticky Note                                      |
|---------------------------|----------------------------------|--------------------------------------------|-----------------------|----------------------|-------------------------------------------------|
| WhatsApp Trigger          | WhatsApp Trigger                  | Receives WhatsApp messages                   | -                     | Personal Agent       | ## Whatsapp Trigger                             |
| Send message              | WhatsApp                         | Sends response message back via WhatsApp    | Personal Agent        | -                    | ## Receive Message from Whatsapp                |
| Google Gemini Chat Model  | LangChain LM Chat (Google Gemini)| General chatbot AI model                      | Personal Agent        | Personal Agent       | ## Chatbot Model                                |
| Simple Memory             | LangChain Memory Buffer          | Stores WhatsApp conversation context         | WhatsApp Trigger      | Personal Agent       |                                                 |
| Personal Agent            | LangChain Agent                  | Central AI assistant that routes tasks       | WhatsApp Trigger, email_agent, calendar_agent, Google Gemini Chat Model | Send message |                                                 |
| email_agent               | LangChain Agent Tool             | Handles email-related AI processing          | Personal Agent, Google Gemini Chat Model4, Window Buffer Memory2 | Personal Agent | ## Email Tool                                   |
| Google Gemini Chat Model4 | LangChain LM Chat (Google Gemini)| AI model specialized for email domain        | email_agent           | email_agent          |                                                 |
| Window Buffer Memory2     | LangChain Memory Buffer          | Email conversation context storage           | email_agent           | email_agent          |                                                 |
| Send Email                | Gmail Tool                      | Sends emails immediately                      | email_agent           | -                    |                                                 |
| Create Draft              | Gmail Tool                      | Creates email drafts                          | email_agent           | -                    |                                                 |
| Email Reply               | Gmail Tool                      | Sends replies to emails                       | email_agent           | -                    |                                                 |
| Get Labels                | Gmail Tool                      | Retrieves Gmail labels                        | email_agent           | -                    |                                                 |
| Label Emails              | Gmail Tool                      | Adds labels to emails                         | email_agent           | -                    |                                                 |
| Mark Unread               | Gmail Tool                      | Marks emails as unread                        | email_agent           | -                    |                                                 |
| Gmail                    | Gmail Tool                      | Retrieves emails with filters                 | email_agent           | -                    |                                                 |
| calendar_agent            | LangChain Agent Tool             | Handles calendar management tasks             | Personal Agent, Google Gemini Chat Model5 | Personal Agent | ## Calendar Tool                                |
| Google Gemini Chat Model5 | LangChain LM Chat (Google Gemini)| AI model specialized for calendar domain      | calendar_agent        | calendar_agent       |                                                 |
| Get all event             | Google Calendar Tool             | Retrieves all calendar events in range        | calendar_agent        | -                    |                                                 |
| Delete event             | Google Calendar Tool             | Deletes specified calendar event              | calendar_agent        | -                    |                                                 |
| Get a single event        | Google Calendar Tool             | Retrieves details of one calendar event       | calendar_agent        | -                    |                                                 |
| Update event              | Google Calendar Tool             | Updates calendar event details                 | calendar_agent        | -                    |                                                 |
| Availablity operation agent | Google Calendar Tool           | Checks calendar availability in a time range | calendar_agent        | -                    |                                                 |
| Create Event with attendee| Google Calendar Tool             | Creates event with attendees                   | calendar_agent        | -                    |                                                 |
| Create Event without attendee | Google Calendar Tool         | Creates event without attendees                | calendar_agent        | -                    |                                                 |
| Sticky Note               | Sticky Note                     | Annotation in UI                              | -                     | -                    | # 🤖 WhatsApp AI Assistant with Email & Calendar Agents... (long content) |
| Sticky Note1              | Sticky Note                     | Annotation for WhatsApp message reception    | -                     | -                    | ## Receive Message from Whatsapp                |
| Sticky Note2              | Sticky Note                     | Annotation for chatbot model                  | -                     | -                    | ## Chatbot Model                                |
| Sticky Note3              | Sticky Note                     | Overview and project description              | -                     | -                    | # 🤖 WhatsApp AI Assistant with Email & Calendar Agents (detailed) |
| Sticky Note4              | Sticky Note                     | Annotation for Email Tool                      | -                     | -                    | ## Email Tool                                   |
| Sticky Note5              | Sticky Note                     | Annotation for Calendar Tool                   | -                     | -                    | ## Calendar Tool                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure webhook to receive messages, enable "messages" update event  
   - Save webhook ID for WhatsApp Business API credentials

2. **Create Simple Memory Node for WhatsApp Context**  
   - Type: LangChain Memory Buffer Window  
   - Set `sessionKey` to WhatsApp API access token  
   - Set `contextWindowLength` to 10  
   - Use sessionIdType as "customKey"

3. **Create Google Gemini Chat Model Node for General Chat**  
   - Type: LangChain LM Chat Google Gemini  
   - Set modelName to "models/gemini-2.5-pro"  
   - Provide Google PaLM API credentials  

4. **Create Personal Agent Node (Central AI Assistant)**  
   - Type: LangChain Agent  
   - Set `text` input to incoming WhatsApp message (`={{ $json.text }}`)  
   - Enter a detailed system message defining “Main” assistant personality and task delegation rules  
   - Enable output parser  
   - Connect memory buffer and Google Gemini Chat Model as AI memory and language model respectively  
   - Set error handling to “continueRegularOutput”  
   - Connect output to Send message node  

5. **Create Send message Node (WhatsApp Send Message)**  
   - Type: WhatsApp  
   - Set operation to “send”  
   - Set textBody to output from Personal Agent (`={{ $json.output }}`)  
   - Specify `phoneNumberId` and recipient phone number (with country code)  
   - Provide WhatsApp Business API credentials  

6. **Email Tool Setup**  
   - Create a new LangChain Agent Tool node named `email_agent`  
     - Input: user message extracted from AI prompt variable  
     - System message with detailed email management instructions including sending, drafting, labeling, replying rules, signature format, and current date/time  
     - Enable output parser  
   - Create Google Gemini Chat Model node for email (`Google Gemini Chat Model4`)  
     - Connect to `email_agent` node as language model  
   - Create Window Buffer Memory node for email memory context (`Window Buffer Memory2`)  
     - Attach to `email_agent` as AI memory  
   - Add Gmail nodes with OAuth2 credentials for:  
     - Send Email (immediate sending)  
     - Create Draft (only when requested)  
     - Email Reply  
     - Get Labels  
     - Label Emails  
     - Mark Unread  
     - Gmail (Get All Emails) for inbox search  
   - Connect these Gmail nodes as AI tools to the `email_agent` node  

7. **Calendar Tool Setup**  
   - Create LangChain Agent Tool node named `calendar_agent`  
     - Input: user message from prompt variable  
     - System message describing calendar assistant capabilities (availability, scheduling, rescheduling, reminders)  
     - Enable output parser  
   - Create Google Gemini Chat Model node for calendar (`Google Gemini Chat Model5`)  
     - Connect as language model for `calendar_agent`  
   - Add Google Calendar nodes with OAuth2 credentials for:  
     - Get all event (range query)  
     - Delete event  
     - Get a single event  
     - Update event  
     - Availability check  
     - Create Event with attendee  
     - Create Event without attendee  
   - Connect these calendar nodes as AI tools for `calendar_agent`  

8. **Connect Agents to Personal Agent**  
   - Link `email_agent` and `calendar_agent` as AI tools to Personal Agent for task routing  
   - Connect Personal Agent to Send message for final WhatsApp output  

9. **Add Sticky Notes for Documentation**  
   - Add descriptive sticky notes to annotate each block: WhatsApp Trigger, Chatbot Model, Email Tool, Calendar Tool, and overall workflow description  

10. **Configure Credentials**  
    - WhatsApp Business API credentials for WhatsApp nodes  
    - Google PaLM API keys for Google Gemini nodes  
    - Gmail OAuth2 credentials for Gmail nodes  
    - Google Calendar OAuth2 credentials for calendar nodes  

11. **Activate workflow** and test by sending WhatsApp messages to the configured number. The workflow will interpret, delegate, and respond accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This workflow transforms WhatsApp into an AI assistant with email and calendar capabilities using Google Gemini AI and specialized sub-agents. It requires setting up credentials for WhatsApp Business API, Google PaLM API, Gmail OAuth2, and Google Calendar OAuth2.                                                                                                                                                                                                                                                                                       | See Sticky Note3 for detailed project overview and setup instructions                                           |
| The Personal Agent (“Main”) is designed with a witty, human-like personality inspired by Donna Paulsen from Suits, providing engaging and expressive responses rather than robotic ones.                                                                                                                                                                                                                                                                                                                                                                                                        | Personal Agent system prompt in node configuration                                                              |
| The email agent enforces plain-text emails with a standard signature and automates sending unless explicitly asked to draft or confirm.                                                                                                                                                                                                                                                                                                                                                                                                            | email_agent system message                                                                                        |
| The calendar agent manages scheduling proactively, checking availability, and sending reminders, ensuring efficient time management.                                                                                                                                                                                                                                                                                                                                                                                                                       | calendar_agent system message                                                                                     |
| For more details on Google Gemini (PaLM) API usage and n8n LangChain integration, refer to the official n8n documentation and Google Cloud AI resources.                                                                                                                                                                                                                                                                                                                                                                                                    | https://www.n8n.io/integrations/n8n-nodes-langchain, https://cloud.google.com/vertex-ai/docs/generative-ai        |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.