Medical Appointment Scheduler with OpenAI, Google Calendar & Messaging Apps

https://n8nworkflows.xyz/workflows/medical-appointment-scheduler-with-openai--google-calendar---messaging-apps-9211


# Medical Appointment Scheduler with OpenAI, Google Calendar & Messaging Apps

---

### 1. Workflow Overview

This workflow is a **Medical Appointment Scheduling Assistant** designed to integrate conversational AI (OpenAI) with calendar management (Google Calendar) and messaging platforms (Telegram and WhatsApp) to facilitate automated scheduling, rescheduling, and cancellation of medical appointments. It targets healthcare providers or services aiming to streamline appointment booking through natural language interactions via popular messaging apps.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Channel Identification:** Captures new incoming messages from Telegram and WhatsApp via webhooks and API triggers, detects the messaging channel, and prepares messages for processing.

- **1.2 Message Processing & Debounce Control:** Aggregates and concatenates incoming messages, manages message flooding through Redis-based debounce logic, and ensures only relevant messages are processed in a timely manner.

- **1.3 AI Conversational Agent with Memory:** Utilizes OpenAI’s chat model with Langchain integration and a simple window memory to interpret user intents and manage dialogue state.

- **1.4 Calendar Integration:** Interacts with Google Calendar API via specialized nodes to check schedules, create or cancel appointments, and verify timing constraints.

- **1.5 Outgoing Messaging Loop:** Sends responses back to users in batches over WhatsApp and Telegram, managing message flow and delivery confirmation.

- **1.6 Control and Utility Nodes:** Includes date/time checks, switches for flow control based on channel or timing, and variable/configuration setup.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Channel Identification

**Overview:**  
This block receives new messages from external messaging platforms and determines the communication channel (Telegram or WhatsApp). It sets up initial variables and routes the messages to the appropriate processing paths.

**Nodes Involved:**  
- `on new message (Evolution)` (WhatsApp webhook)  
- `Telegram Trigger` (Telegram webhook)  
- `WHATSAPP - CHANNEL` (Set node for WhatsApp channel context)  
- `TELEGRAM - CHANNEL` (Set node for Telegram channel context)  
- `CHANNEL` (Set node consolidating channel information)  
- `CHECK CHANNEL1` (Switch node to branch flow by channel)

**Node Details:**

- **on new message (Evolution)**  
  - Type: Webhook  
  - Role: Receives incoming WhatsApp messages via Evolution API webhook.  
  - Config: Requires Evolution API webhook credentials and configuration.  
  - Input: HTTP POST from WhatsApp.  
  - Output: Passes data to `WHATSAPP - CHANNEL`.  
  - Failure: Webhook misconfiguration, API auth errors.  

- **Telegram Trigger**  
  - Type: Webhook  
  - Role: Receives incoming Telegram messages.  
  - Config: Telegram Bot Token, webhook setup.  
  - Input: HTTP POST from Telegram.  
  - Output: Passes data to `TELEGRAM - CHANNEL`.  
  - Failure: Invalid bot token, webhook errors.  

- **WHATSAPP - CHANNEL & TELEGRAM - CHANNEL**  
  - Type: Set  
  - Role: Tag messages with channel metadata for downstream branching.  
  - Config: Static variables setting channel identifiers (e.g., "whatsapp" or "telegram").  
  - Input: From respective webhooks.  
  - Output: Connects to `CHANNEL` node.  

- **CHANNEL**  
  - Type: Set  
  - Role: Consolidates channel info, possibly normalizing or adding further metadata.  
  - Input: From `WHATSAPP - CHANNEL` or `TELEGRAM - CHANNEL`.  
  - Output: To `VARIABLES - CONFIG`.  

- **CHECK CHANNEL1**  
  - Type: Switch  
  - Role: Determines message processing path based on channel (WhatsApp or Telegram).  
  - Input: From `DATE NOW` (date node triggers channel checking).  
  - Outputs: Branches to `OBJECT MSG - WHASTAPP` or `OBJECT MSG - TELEGRAM`.  
  - Edge cases: Unknown/unset channels may cause routing errors.  

---

#### 2.2 Message Processing & Debounce Control

**Overview:**  
Aggregates incoming user messages, converts them into a structured array, concatenates message objects, and manages message flooding with Redis debounce keys to avoid duplicate or too frequent processing.

**Nodes Involved:**  
- `SPLIT USER MESSAGE (INPUT)`  
- `MESSAGES INTO JSON ARRAY`  
- `CHECK CHANNEL`  
- `LOOP MESSAGE - WHATSAPP`  
- `LOOP MESSAGE - TELEGRAM`  
- `IF EXIST MESSAGE`  
- `OBJECT MSG - WHASTAPP`  
- `OBJECT MSG - TELEGRAM`  
- `OBJECT MSG`  
- `OBJECT MSG - CONCATENATE`  
- `REDIS - DEBOUNCE - SET`  
- `REDIS - DEBOUNCE - GET`  
- `REDIS - DEBOUNCE - DELETE`  
- `HAS TIME PASSED?`  
- `WAIT TIME - 5 SECONDS DEBOUNCE`  

**Node Details:**

- **SPLIT USER MESSAGE (INPUT)**  
  - Type: Set  
  - Role: Standardizes incoming user messages into a structured format.  
  - Input: Raw message data from AI agent output.  
  - Output: Passes to `MESSAGES INTO JSON ARRAY`.  

- **MESSAGES INTO JSON ARRAY**  
  - Type: SplitOut  
  - Role: Splits concatenated messages into JSON array elements for processing.  
  - Input: Structured message set.  
  - Output: Routes to `CHECK CHANNEL`.  

- **CHECK CHANNEL**  
  - Type: Switch  
  - Role: Routes message arrays to appropriate loops based on channel.  
  - Outputs: `LOOP MESSAGE - WHATSAPP` and `LOOP MESSAGE - TELEGRAM`.  

- **LOOP MESSAGE - WHATSAPP / LOOP MESSAGE - TELEGRAM**  
  - Type: SplitInBatches  
  - Role: Processes messages in manageable batches for sending.  
  - Output: To `SEND MESSAGE WHATSAPP` or `SEND MESSAGE TELEGRAM` respectively.  
  - Edge cases: Large batch sizes may cause timeouts; empty batches skipped.  

- **IF EXIST MESSAGE**  
  - Type: If  
  - Role: Checks if a message exists before sending to Telegram.  
  - True: Send message.  
  - False: Loop message again or halt.  

- **OBJECT MSG - WHASTAPP & OBJECT MSG - TELEGRAM**  
  - Type: Code  
  - Role: Formats message objects according to platform-specific requirements.  
  - Input: Raw message data.  
  - Output: Passes formatted messages to `OBJECT MSG`.  
  - Failure: Formatting errors may cause send failures.  

- **OBJECT MSG**  
  - Type: Set  
  - Role: Sets common message properties after platform-specific formatting.  
  - Output: Passes to Redis debounce set node.  

- **OBJECT MSG - CONCATENATE**  
  - Type: Code  
  - Role: Concatenates multiple message parts into a single message object for AI processing.  
  - Input: From Redis debounce delete node.  
  - Output: To `Calendar AI Agent`.  

- **REDIS - DEBOUNCE - SET/GET/DELETE**  
  - Type: Redis  
  - Role: Implements debounce logic by storing keys for message timing control.  
  - Config: Redis connection with expiry for keys (e.g., 5 seconds).  
  - Input/Output: SET stores key, GET checks existence, DELETE removes key post-processing.  
  - Failures: Redis connectivity, key expiry misconfiguration.  

- **HAS TIME PASSED?**  
  - Type: Switch  
  - Role: Determines if enough time has elapsed to process new messages (debounce check).  
  - Routes to delete Redis key or wait node accordingly.  

- **WAIT TIME - 5 SECONDS DEBOUNCE**  
  - Type: Wait  
  - Role: Pauses execution to enforce debounce delay.  
  - Config: Fixed 5-second wait.  

---

#### 2.3 AI Conversational Agent with Memory

**Overview:**  
Handles the AI processing of user messages leveraging OpenAI’s chat model integrated with Langchain's agent and simple windowed memory to maintain context across interactions. The AI agent also interfaces with calendar tools as AI tools to perform specific appointment-related actions.

**Nodes Involved:**  
- `OpenAI Chat Model`  
- `Calendar AI Agent`  
- `Simple Memory`  
- `Check Schedule`  
- `Create Appointment`  
- `Cancel Appointment`  

**Node Details:**

- **OpenAI Chat Model**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Provides natural language understanding and response generation.  
  - Config: OpenAI API credentials, model selection (e.g., GPT-4), temperature, max tokens.  
  - Input: User messages from concatenated message object.  
  - Output: AI response to `Calendar AI Agent`.  
  - Failures: API rate limits, authentication errors, malformed prompt.  

- **Calendar AI Agent**  
  - Type: Langchain Agent  
  - Role: Acts as orchestrator AI, using OpenAI for language and Google Calendar nodes as tools.  
  - Config: Connects language model and AI tools (`Check Schedule`, `Create Appointment`, `Cancel Appointment`).  
  - Inputs: Receives message input and memory context.  
  - Outputs: Processed messages and commands for calendar manipulation.  
  - Edge cases: Misinterpretation of intent, API failures in calendar tools.  

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains conversational context window to support multi-turn dialogues.  
  - Config: Configurable window size (number of messages).  
  - Input: Conversation messages.  
  - Output: Provides memory context to AI agent.  

- **Check Schedule, Create Appointment, Cancel Appointment**  
  - Type: Google Calendar Tool nodes  
  - Role: Provide AI agent with calendar interaction capabilities: checking free/busy slots, creating events, cancelling events.  
  - Config: OAuth2 credentials for Google Calendar, calendar ID.  
  - Input: Commands from AI agent.  
  - Output: Calendar API responses back to AI agent.  
  - Failure: OAuth token expiry, API quota exceeded, calendar access denied.  

---

#### 2.4 Calendar Integration

**Overview:**  
These nodes interface directly with Google Calendar to manage appointment scheduling actions requested by the AI agent.

**Nodes Involved:**  
- `Check Schedule`  
- `Create Appointment`  
- `Cancel Appointment`  

**Node Details:**

- **Check Schedule**  
  - Type: Google Calendar Tool  
  - Role: Queries calendar availability for requested appointment times.  
  - Config: Calendar ID, time range parameters derived from AI agent input.  
  - Input: From AI agent tool interface.  
  - Output: Availability status for scheduling decisions.  

- **Create Appointment**  
  - Type: Google Calendar Tool  
  - Role: Creates a new event/appointment in the calendar.  
  - Config: Event details such as start/end times, description, attendees.  
  - Input: Appointment request from AI agent.  
  - Output: Confirmation of event creation.  

- **Cancel Appointment**  
  - Type: Google Calendar Tool  
  - Role: Removes an existing event from the calendar.  
  - Config: Event ID or identifiers to cancel.  
  - Input: Cancellation request from AI agent.  
  - Output: Confirmation of event deletion.  

Failures in these nodes include authentication issues, API call limits, or invalid event data leading to errors.

---

#### 2.5 Outgoing Messaging Loop

**Overview:**  
Splits outgoing messages into batches and sends them back to the user via the appropriate messaging platform (Telegram or WhatsApp), ensuring flow control and retries if needed.

**Nodes Involved:**  
- `LOOP MESSAGE - WHATSAPP`  
- `LOOP MESSAGE - TELEGRAM`  
- `SEND MESSAGE WHATSAPP`  
- `SEND MESSAGE TELEGRAM`  
- `IF EXIST MESSAGE`  

**Node Details:**

- **LOOP MESSAGE - WHATSAPP / TELEGRAM**  
  - Type: SplitInBatches  
  - Role: Manages batch-wise sending of messages to avoid flooding or API limits.  
  - Input: Messages ready to send, formatted per platform.  
  - Output: To respective send nodes.  

- **SEND MESSAGE WHATSAPP**  
  - Type: Evolution API node  
  - Role: Sends message through Evolution API for WhatsApp.  
  - Config: API credentials, message payload.  
  - Input: Batched WhatsApp messages.  
  - Outputs: To `LOOP MESSAGE - WHATSAPP` for next batch or end.  
  - Failures: API token expiration, network errors, message format errors.  

- **SEND MESSAGE TELEGRAM**  
  - Type: Telegram node  
  - Role: Sends message via Telegram Bot API.  
  - Config: Bot token, chat ID, message content.  
  - Input: Batched Telegram messages.  
  - Output: To `LOOP MESSAGE - TELEGRAM` or `IF EXIST MESSAGE` for flow control.  

- **IF EXIST MESSAGE**  
  - Type: If  
  - Role: Confirms presence of message before sending, avoiding empty sends.  

---

#### 2.6 Control and Utility Nodes

**Overview:**  
Support nodes that provide configuration, timing, and flow control.

**Nodes Involved:**  
- `VARIABLES - CONFIG`  
- `DATE NOW`  
- `HAS TIME PASSED?`  
- `WAIT TIME - 5 SECONDS DEBOUNCE`  
- Various sticky notes for documentation (empty content in this export)

**Node Details:**

- **VARIABLES - CONFIG**  
  - Type: Set  
  - Role: Holds global variables or configurations used throughout the workflow.  
  - Input: Entry point.  
  - Output: To `DATE NOW`.  

- **DATE NOW**  
  - Type: DateTime  
  - Role: Retrieves current date/time for time-based logic (e.g., debounce timing).  
  - Output: To `CHECK CHANNEL1`.  

- **HAS TIME PASSED?**  
  - Type: Switch  
  - Role: Logic gate to check if debounce duration has elapsed before processing.  

- **WAIT TIME - 5 SECONDS DEBOUNCE**  
  - Type: Wait  
  - Role: Enforces a 5-second wait to throttle message processing frequency.  

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                                   | Input Node(s)                      | Output Node(s)                     | Sticky Note                |
|-------------------------------|---------------------------------------|--------------------------------------------------|----------------------------------|----------------------------------|----------------------------|
| on new message (Evolution)     | Webhook                               | Receives WhatsApp messages                        | —                                | WHATSAPP - CHANNEL               |                            |
| Telegram Trigger               | Webhook                               | Receives Telegram messages                        | —                                | TELEGRAM - CHANNEL               |                            |
| WHATSAPP - CHANNEL            | Set                                   | Sets channel metadata for WhatsApp                | on new message (Evolution)        | CHANNEL                         |                            |
| TELEGRAM - CHANNEL            | Set                                   | Sets channel metadata for Telegram                 | Telegram Trigger                 | CHANNEL                         |                            |
| CHANNEL                       | Set                                   | Consolidates channel info                          | WHATSAPP - CHANNEL, TELEGRAM - CHANNEL | VARIABLES - CONFIG             |                            |
| VARIABLES - CONFIG            | Set                                   | Global variables/configuration                     | CHANNEL                         | DATE NOW                       |                            |
| DATE NOW                     | DateTime                              | Provides current datetime                           | VARIABLES - CONFIG               | CHECK CHANNEL1                 |                            |
| CHECK CHANNEL1               | Switch                                | Routes flow based on channel                        | DATE NOW                       | OBJECT MSG - WHASTAPP, OBJECT MSG - TELEGRAM |                            |
| OBJECT MSG - WHASTAPP        | Code                                  | Formats WhatsApp message objects                    | CHECK CHANNEL1                  | OBJECT MSG                    |                            |
| OBJECT MSG - TELEGRAM        | Code                                  | Formats Telegram message objects                    | CHECK CHANNEL1                  | OBJECT MSG                    |                            |
| OBJECT MSG                   | Set                                   | Sets common message properties                      | OBJECT MSG - WHASTAPP, OBJECT MSG - TELEGRAM | REDIS - DEBOUNCE - SET       |                            |
| REDIS - DEBOUNCE - SET       | Redis                                 | Stores debounce key to prevent flooding            | OBJECT MSG                     | REDIS - DEBOUNCE - GET          |                            |
| REDIS - DEBOUNCE - GET       | Redis                                 | Checks debounce key existence                        | REDIS - DEBOUNCE - SET          | HAS TIME PASSED?                |                            |
| HAS TIME PASSED?             | Switch                                | Determines if debounce wait time elapsed            | REDIS - DEBOUNCE - GET          | REDIS - DEBOUNCE - DELETE, WAIT TIME - 5 SECONDS DEBOUNCE |                            |
| REDIS - DEBOUNCE - DELETE    | Redis                                 | Deletes debounce key after processing               | HAS TIME PASSED?                | OBJECT MSG - CONCATENATE        |                            |
| WAIT TIME - 5 SECONDS DEBOUNCE | Wait                                 | Waits 5 seconds before retrying                      | HAS TIME PASSED?                | REDIS - DEBOUNCE - GET          |                            |
| OBJECT MSG - CONCATENATE     | Code                                  | Concatenates message parts for AI input             | REDIS - DEBOUNCE - DELETE       | Calendar AI Agent               |                            |
| Calendar AI Agent            | Langchain Agent                       | Orchestrates AI with LM and calendar tools          | OBJECT MSG - CONCATENATE, Simple Memory | SPLIT USER MESSAGE (INPUT)     |                            |
| Simple Memory                | Langchain Memory Buffer Window       | Maintains AI conversation context                    | —                              | Calendar AI Agent               |                            |
| OpenAI Chat Model            | Langchain LM Chat OpenAI              | Generates AI responses                               | Calendar AI Agent               | Calendar AI Agent               |                            |
| Check Schedule              | Google Calendar Tool                   | Checks calendar availability                         | Calendar AI Agent               | Calendar AI Agent               |                            |
| Create Appointment          | Google Calendar Tool                   | Creates new calendar events                           | Calendar AI Agent               | Calendar AI Agent               |                            |
| Cancel Appointment          | Google Calendar Tool                   | Cancels existing calendar events                      | Calendar AI Agent               | Calendar AI Agent               |                            |
| SPLIT USER MESSAGE (INPUT)  | Set                                   | Structures AI output messages for processing          | Calendar AI Agent               | MESSAGES INTO JSON ARRAY        |                            |
| MESSAGES INTO JSON ARRAY    | SplitOut                             | Splits messages into JSON array                       | SPLIT USER MESSAGE (INPUT)      | CHECK CHANNEL                  |                            |
| CHECK CHANNEL               | Switch                                | Routes messages by channel                             | MESSAGES INTO JSON ARRAY        | LOOP MESSAGE - WHATSAPP, LOOP MESSAGE - TELEGRAM |                            |
| LOOP MESSAGE - WHATSAPP     | SplitInBatches                       | Batch processes WhatsApp messages                      | CHECK CHANNEL                  | SEND MESSAGE WHATSAPP           |                            |
| LOOP MESSAGE - TELEGRAM     | SplitInBatches                       | Batch processes Telegram messages                      | CHECK CHANNEL                  | IF EXIST MESSAGE, SEND MESSAGE TELEGRAM |                            |
| IF EXIST MESSAGE            | If                                    | Checks message existence before sending               | LOOP MESSAGE - TELEGRAM         | SEND MESSAGE TELEGRAM, LOOP MESSAGE - TELEGRAM |                            |
| SEND MESSAGE WHATSAPP       | Evolution API node                   | Sends messages via Evolution WhatsApp API             | LOOP MESSAGE - WHATSAPP         | LOOP MESSAGE - WHATSAPP         |                            |
| SEND MESSAGE TELEGRAM       | Telegram                             | Sends messages via Telegram Bot API                    | IF EXIST MESSAGE               | LOOP MESSAGE - TELEGRAM         |                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes for Incoming Messages:**
   - Create a `Webhook` node named `on new message (Evolution)` for WhatsApp incoming messages via Evolution API. Configure with appropriate webhook URL and credentials.
   - Create a `Telegram Trigger` node for Telegram incoming messages. Set up Telegram Bot Credentials and webhook.

2. **Channel Identification:**
   - Create `Set` nodes named `WHATSAPP - CHANNEL` and `TELEGRAM - CHANNEL` connected respectively to the above webhooks.
   - In each, set a static variable (e.g., `channel = "whatsapp"` or `channel = "telegram"`).
   - Create a `Set` node named `CHANNEL` that consolidates the channel information from the above two nodes.
   
3. **Global Configuration and Date Node:**
   - Add a `Set` node named `VARIABLES - CONFIG` after `CHANNEL` to hold global variables and configurations.
   - Connect `VARIABLES - CONFIG` to a `DateTime` node named `DATE NOW` to fetch current timestamp.

4. **Channel-based Routing:**
   - Create a `Switch` node `CHECK CHANNEL1` connected from `DATE NOW`.
   - Configure it to route messages based on the `channel` variable (e.g., "whatsapp" -> `OBJECT MSG - WHASTAPP`, "telegram" -> `OBJECT MSG - TELEGRAM`).

5. **Message Formatting:**
   - Create two `Code` nodes named `OBJECT MSG - WHASTAPP` and `OBJECT MSG - TELEGRAM` to format incoming messages according to platform requirements.
   - Connect both to a common `Set` node `OBJECT MSG`.

6. **Debounce Logic:**
   - Add a Redis node `REDIS - DEBOUNCE - SET` connected from `OBJECT MSG`, configured with Redis credentials to set a key with a TTL (e.g., 5 seconds).
   - Link to `REDIS - DEBOUNCE - GET` to check if key exists.
   - Connect to a `Switch` node `HAS TIME PASSED?` to branch depending on whether the key is expired.
   - If time passed, connect to `REDIS - DEBOUNCE - DELETE` to remove the key and then to `OBJECT MSG - CONCATENATE` (a Code node to concatenate messages).
   - If not, connect to a `Wait` node `WAIT TIME - 5 SECONDS DEBOUNCE` and loop back to `REDIS - DEBOUNCE - GET`.

7. **AI Agent Setup:**
   - Create an `OpenAI Chat Model` node configured with OpenAI API credentials (GPT-4 recommended).
   - Create a `Simple Memory` node configured for windowed conversation memory.
   - Create a `Calendar AI Agent` Langchain agent node that integrates the `OpenAI Chat Model` and calendar tool nodes (`Check Schedule`, `Create Appointment`, `Cancel Appointment`).
   - Connect `OBJECT MSG - CONCATENATE` output to `Calendar AI Agent` input.
   - Connect `Simple Memory` as AI memory to `Calendar AI Agent`.

8. **Google Calendar Tool Nodes:**
   - Create three Google Calendar Tool nodes: `Check Schedule`, `Create Appointment`, and `Cancel Appointment`.
   - Configure each with OAuth2 credentials and calendar ID.
   - Connect each as AI tools in the `Calendar AI Agent` node’s tool inputs.

9. **Post-AI Message Processing:**
   - Connect `Calendar AI Agent` output to a `Set` node `SPLIT USER MESSAGE (INPUT)` for formatting AI responses.
   - Connect it to a `SplitOut` node `MESSAGES INTO JSON ARRAY` to split messages.

10. **Channel Routing for Outgoing Messages:**
    - Add a `Switch` node `CHECK CHANNEL` to route messages based on their channel metadata.
    - Connect outputs to two `SplitInBatches` nodes: `LOOP MESSAGE - WHATSAPP` and `LOOP MESSAGE - TELEGRAM`.

11. **Message Sending Nodes:**
    - Create an `Evolution API` node `SEND MESSAGE WHATSAPP` configured with WhatsApp API credentials.
    - Create a `Telegram` node `SEND MESSAGE TELEGRAM` configured with Telegram Bot credentials.
    - Connect `LOOP MESSAGE - WHATSAPP` to `SEND MESSAGE WHATSAPP` and `LOOP MESSAGE - TELEGRAM` to `SEND MESSAGE TELEGRAM`.
    - Add an `If` node `IF EXIST MESSAGE` after `LOOP MESSAGE - TELEGRAM` to confirm message existence before sending.

12. **Loops and Flow Control:**
    - Configure the output connections of send nodes back to their respective `SplitInBatches` nodes or `IF EXIST MESSAGE` to continue sending batches or halt.

13. **Additional Setup:**
    - Create a `DateTime` node `DATE NOW` to trigger date-dependent logic.
    - Ensure Redis credentials are properly set for debounce nodes.
    - Configure all API credentials (OpenAI, Google Calendar OAuth2, Telegram Bot, Evolution API) securely in n8n credential manager.

14. **Testing and Validation:**
    - Test each webhook and messaging node independently.
    - Validate AI agent responses and calendar API actions.
    - Test debounce functionality by rapid message sending.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                               |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow utilizes Redis for debounce control to prevent message flooding and duplicate processing.         | Redis server required; configure connection in n8n credentials |
| The AI agent is powered by OpenAI GPT-4 via Langchain integration, enabling multi-turn conversational context. | OpenAI API documentation: https://platform.openai.com/docs  |
| Google Calendar API nodes require OAuth2 credentials with calendar access enabled.                             | Google Cloud Console for API credentials setup                |
| Messaging via WhatsApp uses Evolution API, which requires specific API credentials and webhook setup.          | Evolution API docs and support                                |
| Telegram messaging requires a Telegram bot with webhook configured.                                            | Telegram Bot API: https://core.telegram.org/bots/api          |

---

**Disclaimer:**  
The provided text is exclusively generated from an n8n automated workflow export. The processing respects all applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---