Manage Google Calendar via WhatsApp with GPT-4 Virtual Assistant

https://n8nworkflows.xyz/workflows/manage-google-calendar-via-whatsapp-with-gpt-4-virtual-assistant-5368


# Manage Google Calendar via WhatsApp with GPT-4 Virtual Assistant

### 1. Workflow Overview

This workflow, titled **"Manage Google Calendar via WhatsApp with GPT-4 Virtual Assistant"**, serves as a virtual assistant named TimePilot that enables a user to manage their Google Calendar events through WhatsApp messages. It leverages GPT-4 to interpret natural language inputs, then routes calendar-related commands to specific Google Calendar operations — creating, updating, deleting, or searching events — and responds back via WhatsApp in real time.

**Target Use Cases:**

- Scheduling new meetings or calls via WhatsApp text
- Updating or rescheduling existing events
- Deleting calendar entries
- Querying upcoming meetings or availability
- Receiving confirmations or clarifications interactively on WhatsApp

**Logical Blocks:**

- **1.1 WhatsApp Trigger:** Entry point capturing incoming WhatsApp messages.
- **1.2 TimePilot Agent (AI Processing):** Core GPT-4-powered logic interpreting user intent and orchestrating calendar operations.
- **1.3 Google Calendar Modules:** Four distinct nodes performing Create, Update, Delete, and Search operations on Google Calendar.
- **1.4 Context Memory:** A simple memory buffer to maintain conversational context per session.
- **1.5 Output & Confirmation:** Preparing success messages and sending them back to the user on WhatsApp.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1 – WhatsApp Trigger

**Overview:**  
This is the workflow’s entry point that activates when a WhatsApp message is received. It captures the incoming message data and metadata to start processing.

**Nodes Involved:**  
- WhatsApp Trigger

**Node Details:**  

- **WhatsApp Trigger**  
  - *Type:* WhatsApp Trigger node  
  - *Role:* Listens for incoming WhatsApp messages (webhook-based trigger)  
  - *Configuration:*  
    - Listens to `messages` updates only  
    - Uses OAuth2 credentials tied to a WhatsApp Business Cloud account  
    - Webhook ID configured to receive WhatsApp messages  
  - *Expressions/Variables:* The raw WhatsApp message JSON is accessible as input data  
  - *Connections:* Output connected to the TimePilot agent node  
  - *Failure Modes:*  
    - Authentication errors if WhatsApp OAuth token is invalid or expired  
    - Webhook misconfiguration leading to missed messages  
    - Message format unexpected or unsupported types  
  - *Version:* v1 (standard for WhatsApp Trigger node)

---

#### 2.2 Block 2 – TimePilot Agent (AI Processing)

**Overview:**  
This block is the AI brain that interprets WhatsApp messages using GPT-4, decides the user intent, and triggers appropriate calendar actions. It also manages conversational context.

**Nodes Involved:**  
- TimePilot (Langchain Agent)  
- OpenAI Chat Model (GPT-4 Turbo)  
- Simple Memory (Memory Buffer)

**Node Details:**  

- **TimePilot** (Langchain Agent)  
  - *Type:* Langchain Agent node  
  - *Role:* Orchestrates the workflow’s logic by leveraging GPT-4 to interpret text and select calendar modules  
  - *Configuration:*  
    - Input text is the WhatsApp message body  
    - Uses a detailed system prompt describing assistant behavior, tool usage, SOP, and examples  
    - Instructs the AI to only invoke one of the four calendar modules per request  
    - Supports English and French messages  
    - Default meeting duration and time zone specified  
  - *Expressions:*  
    - Input: `{{$json.messages[0].text.body}}` — extracts user message text  
    - System prompt includes dynamic current date/time (`{{$now.format('yyyy-MM-dd')}}`)  
  - *Connections:*  
    - Receives AI language model output from OpenAI Chat Model node  
    - Sends commands to calendar modules (Create Event, Update Event, Delete Event, Search Event) as AI tools  
    - Uses Simple Memory for session context retention  
    - Outputs structured response to Success node  
  - *Failure Modes:*  
    - API quota or network errors with OpenAI  
    - Misinterpretation of ambiguous user inputs  
    - Missing critical data causing incomplete actions  
    - Expression evaluation errors on variables  
  - *Version:* 1.8 (latest support for Langchain agent features)  
  - *Sub-workflow:* None, but acts as central orchestrator invoking tools  

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI Chat Model node  
  - *Role:* Provides GPT-4 Turbo language model for natural language understanding  
  - *Configuration:*  
    - Model set to `gpt-4-turbo-preview` for cost-effective and fast responses  
    - No special options set beyond default  
  - *Input/Output:* Connected as AI language model for TimePilot agent  
  - *Failure Modes:* API key invalid, rate limits, model unavailability  
  - *Credentials:* OpenAI API key configured  

- **Simple Memory**  
  - *Type:* Langchain Memory Buffer Window node  
  - *Role:* Maintains conversational context keyed by WhatsApp message type to enable coherent multi-turn dialogue  
  - *Configuration:*  
    - Session key derived from message type (e.g., text, image) for session isolation  
  - *Connections:* Feeds memory context into TimePilot agent  
  - *Failure Modes:* Incorrect session key usage may cause context mixing or loss  

---

#### 2.3 Block 3 – Google Calendar Modules

**Overview:**  
These nodes perform respective Google Calendar operations based on AI commands: create, update, delete, and search calendar events.

**Nodes Involved:**  
- Create Event  
- Update Event  
- Delete Event  
- Search Event

**Node Details:**  

- **Create Event**  
  - *Type:* Google Calendar Tool node  
  - *Role:* Creates new events on Google Calendar  
  - *Configuration:*  
    - Calendar selected from user’s account email  
    - Start and end times, and summary/title are dynamically provided by AI outputs  
    - Attendees list is empty by default (could be extended)  
    - Description manually typed or passed dynamically  
  - *Expressions:* Uses AI variables `$fromAI('Start')`, `$fromAI('End')`, `$fromAI('Summary')` for dynamic event details  
  - *Credentials:* Google Calendar OAuth2 authorized account  
  - *Failure Modes:*  
    - OAuth token expiry or invalid credentials  
    - Invalid date/time formats  
    - API quota exceeded  
  - *Version:* 1.3  

- **Update Event**  
  - *Type:* Google Calendar Tool node  
  - *Role:* Updates existing calendar events  
  - *Configuration:*  
    - Event ID dynamically obtained via AI (`$fromAI("eventID")`)  
    - New start and end times provided (`$fromAI("startTime")`, `$fromAI("endTime")`)  
  - *Credentials:* Same OAuth2 account  
  - *Failure Modes:*  
    - Event not found or invalid event ID  
    - Attempting to update locked or recurring events without proper handling  
  - *Version:* 1.3  

- **Delete Event**  
  - *Type:* Google Calendar Tool node  
  - *Role:* Deletes an event identified by event ID  
  - *Configuration:*  
    - Event ID passed dynamically from AI output  
    - Sends update notifications to attendees (`sendUpdates: "all"`)  
  - *Credentials:* Same OAuth2 account  
  - *Failure Modes:*  
    - Event ID invalid or event already deleted  
    - Permissions issues in calendar account  
  - *Version:* 1.3  

- **Search Event**  
  - *Type:* Google Calendar Tool node  
  - *Role:* Searches for events in a specified time range  
  - *Configuration:*  
    - Time range boundaries (`timeMin`, `timeMax`) passed from AI outputs  
    - Limit on number of events retrieved is dynamic  
  - *Credentials:* Same OAuth2 account  
  - *Failure Modes:*  
    - Invalid date range formats  
    - API response delays or failures  
  - *Version:* 1.3  

---

#### 2.4 Block 4 – Output & Confirmation

**Overview:**  
Prepares a success message based on the AI’s output and sends it back to the user via WhatsApp.

**Nodes Involved:**  
- Success (Set node)  
- WhatsApp Business Cloud (Send message)

**Node Details:**  

- **Success**  
  - *Type:* Set node  
  - *Role:* Prepares the final response message for WhatsApp by assigning the AI-generated response text to the `response` field  
  - *Configuration:*  
    - Sets `response` to the `output` field coming from the TimePilot agent  
  - *Connections:* Output linked to WhatsApp Business Cloud node  
  - *Failure Modes:* Missing or malformed AI output may cause empty messages  
  - *Version:* 3.4  

- **WhatsApp Business Cloud**  
  - *Type:* WhatsApp node (message send)  
  - *Role:* Sends the final response text back to the user on WhatsApp  
  - *Configuration:*  
    - Text body dynamically set to `{{$json.response}}` from Success node  
    - Recipient phone number hardcoded placeholder (must be replaced with actual number or dynamic input)  
    - Phone number ID taken from WhatsApp Trigger metadata  
  - *Credentials:* WhatsApp OAuth credentials  
  - *Failure Modes:*  
    - Incorrect phone number format or missing recipient number  
    - WhatsApp API quota or permission issues  
  - *Version:* 1  

---

### 3. Summary Table

| Node Name            | Node Type                                | Functional Role                          | Input Node(s)         | Output Node(s)         | Sticky Note                                                                                                            |
|----------------------|----------------------------------------|----------------------------------------|-----------------------|------------------------|------------------------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger      | WhatsApp Trigger                       | Entry point; receives WhatsApp message | -                     | TimePilot              | 🟩 Block 1 – WhatsApp Trigger: Entry point capturing incoming WhatsApp messages                                          |
| TimePilot             | Langchain Agent                       | AI brain interpreting messages, routing actions | WhatsApp Trigger, OpenAI Chat Model, Simple Memory, Create/Update/Delete/Search Event | Success                | 🟦 Block 2 – TimePilot (Tools Agent): Main AI agent interpreting natural language and triggering calendar modules      |
| OpenAI Chat Model     | Langchain OpenAI Chat Model           | Provides GPT-4 Turbo for NLP processing | TimePilot              | TimePilot              | 🟦 Block 2 – TimePilot (Tools Agent)                                                                                    |
| Simple Memory         | Langchain Memory Buffer Window        | Maintains conversational context       | -                     | TimePilot              | 🟦 Block 2 – TimePilot (Tools Agent)                                                                                    |
| Create Event          | Google Calendar Tool                  | Creates calendar events                 | TimePilot              | TimePilot              | 🟦 Block 2 – TimePilot (Tools Agent)                                                                                    |
| Update Event          | Google Calendar Tool                  | Updates existing calendar events       | TimePilot              | TimePilot              | 🟦 Block 2 – TimePilot (Tools Agent)                                                                                    |
| Delete Event          | Google Calendar Tool                  | Deletes calendar events                 | TimePilot              | TimePilot              | 🟦 Block 2 – TimePilot (Tools Agent)                                                                                    |
| Search Event          | Google Calendar Tool                  | Searches calendar events                | TimePilot              | TimePilot              | 🟦 Block 2 – TimePilot (Tools Agent)                                                                                    |
| Success               | Set                                  | Prepares success message for WhatsApp  | TimePilot              | WhatsApp Business Cloud | 🟪 Part 3 – Output & Confirmation: Prepares success message                                                            |
| WhatsApp Business Cloud | WhatsApp (Send Message)              | Sends response back to user on WhatsApp| Success                | -                      | 🟪 Part 3 – Output & Confirmation                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure to listen for `messages` updates only  
   - Use OAuth2 credentials for WhatsApp Business Cloud (set up beforehand)  
   - Assign a webhook ID or leave default for webhook URL  

2. **Create OpenAI Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Model: `gpt-4-turbo-preview`  
   - Set OpenAI API credentials with your API key  

3. **Create Simple Memory Node**  
   - Type: Langchain Memory Buffer Window  
   - Session Key: Use expression `{{$json.messages[0].type}}` for session isolation  
   - Leave other defaults  

4. **Create TimePilot Agent Node**  
   - Type: Langchain Agent node  
   - Input Text: Set expression `{{$json.messages[0].text.body}}` from WhatsApp Trigger  
   - Set system prompt describing assistant behavior, instructions, examples, and SOP as given in the original prompt (see detailed system prompt above)  
   - Configure AI Language Model input to OpenAI Chat Model node  
   - Configure AI Memory input to Simple Memory node  
   - Set AI Tools inputs to Create Event, Update Event, Delete Event, and Search Event nodes  
   - Output: Connect to Success node  

5. **Create Google Calendar Tool Nodes**  
   - Create four separate nodes for Create Event, Update Event, Delete Event, and Search Event  
   - Authenticate each with Google Calendar OAuth2 credentials  
   - Set calendar to your Google Calendar email account  
   - Create Event: Configure to accept dynamic inputs for start, end, summary via `$fromAI('Start')`, `$fromAI('End')`, `$fromAI('Summary')`  
   - Update Event: Accept `eventID`, `startTime`, `endTime` via `$fromAI()` variables  
   - Delete Event: Accept `eventID` via `$fromAI("eventID")`, set `sendUpdates` to "all"  
   - Search Event: Accept `timeMin`, `timeMax`, `limit` dynamically for event search  

6. **Create Success Node (Set Node)**  
   - Type: Set  
   - Assign field `response` to expression `{{$json.output}}` from TimePilot output  

7. **Create WhatsApp Business Cloud Node**  
   - Type: WhatsApp (Send Message)  
   - Text Body: Set to `{{$json.response}}` from Success node  
   - Phone Number ID: Use expression `{{$('WhatsApp Trigger').item.json.metadata.phone_number_id}}`  
   - Recipient Phone Number: Replace placeholder with actual WhatsApp user number or make dynamic based on incoming message sender  
   - Authenticate with WhatsApp OAuth2 credentials  

8. **Connect Flow**  
   - WhatsApp Trigger → TimePilot Agent  
   - OpenAI Chat Model → TimePilot Agent (AI Language Model input)  
   - Simple Memory → TimePilot Agent (AI Memory input)  
   - Create/Update/Delete/Search Event nodes → TimePilot Agent (AI Tools input)  
   - TimePilot Agent → Success Node  
   - Success Node → WhatsApp Business Cloud Node  

9. **Credentials Setup**  
   - Configure OpenAI API credentials with valid API key  
   - Configure Google Calendar OAuth2 credentials with calendar access  
   - Configure WhatsApp OAuth2 credentials with WhatsApp Business Cloud tokens and verified phone number  

10. **Testing**  
    - Send WhatsApp messages such as “Create a meeting with Anna at 4pm tomorrow”  
    - Verify that the calendar event is created and confirmation is received via WhatsApp  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Prerequisites include WhatsApp Business Cloud account, Google Calendar OAuth2, OpenAI API key, verified Meta phone number, and n8n instance accessible via webhook for Meta.                                                                                                                                                                                              | Block 4 Sticky Note (How to use TimePilot Agent)                                                                        |
| Step-by-step Meta App creation and WhatsApp product setup, including webhook registration and token generation, are necessary before using this workflow.                                                                                                                                                                                                                  | Block 4 Sticky Note (How to use TimePilot Agent)                                                                        |
| TimePilot AI agent uses a detailed system prompt enforcing strict usage of calendar modules, multi-language support (English/French), and best practices to avoid ambiguous or incomplete commands. It prevents unauthorized assumptions and ensures context-aware follow-up questions.                                                                                     | Block 2 Sticky Note (TimePilot Agent details)                                                                           |
| The workflow uses GPT-4 turbo-preview model for cost-effective and fast AI responses.                                                                                                                                                                                                                                                                                      | Node details for OpenAI Chat Model                                                                                       |
| WhatsApp Business Cloud node currently has a placeholder for recipient phone number; this must be replaced with the actual WhatsApp user number or improved to dynamically retrieve sender number for two-way communication.                                                                                                                                             | WhatsApp Business Cloud node details                                                                                     |
| The workflow is designed for a single user (you) managing your personal Google Calendar exclusively, ensuring privacy and simplicity.                                                                                                                                                                                                                                     | TimePilot system prompt                                                                                                  |
| The default meeting duration is one hour unless otherwise specified in user input and processed by AI.                                                                                                                                                                                                                                                                     | TimePilot system prompt                                                                                                  |
| For more details on creating Meta Apps and configuring WhatsApp Business Cloud, see https://developers.facebook.com/docs/whatsapp/getting-started/                                                                                                                                                                                                                         | Meta Developer Portal documentation                                                                                      |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow. It complies strictly with content policies, contains no illegal or offensive content, and handles only legal and public data.