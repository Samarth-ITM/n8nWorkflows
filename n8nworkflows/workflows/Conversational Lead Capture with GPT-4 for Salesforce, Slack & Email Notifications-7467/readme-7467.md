Conversational Lead Capture with GPT-4 for Salesforce, Slack & Email Notifications

https://n8nworkflows.xyz/workflows/conversational-lead-capture-with-gpt-4-for-salesforce--slack---email-notifications-7467


# Conversational Lead Capture with GPT-4 for Salesforce, Slack & Email Notifications

### 1. Workflow Overview

This workflow, named **"LeadBot Autopilot: Chat-to-Lead for Salesforce Automation"**, implements a conversational lead capture chatbot integrating GPT-4, Salesforce, Slack, and email notifications. It replaces traditional web forms with a step-by-step conversational interface to collect lead information: full name, email, mobile phone, and product interest. The workflow validates inputs, checks for duplicate leads in Salesforce, updates or creates records accordingly, and notifies both internal teams and clients upon successful lead capture.

**Logical Blocks:**

- **1.1 Chat Input Reception**  
  Entry point capturing user messages via webhook and initiating conversation.

- **1.2 AI Conversational Agent**  
  Core logic managing conversation flow, input validation, tool invocations (Salesforce queries and updates), and response generation.

- **1.3 Language Model Backend**  
  GPT-4 based OpenAI model supporting natural language processing and reasoning.

- **1.4 Memory Management**  
  Maintains conversation state and context via a windowed buffer memory.

- **1.5 Duplicate Lead Checking**  
  Queries Salesforce for existing leads by email to avoid duplication.

- **1.6 Internal Reasoning Tool**  
  AI internal "think" node to handle complex reasoning (e.g., name parsing).

- **1.7 Lead Record Management**  
  Salesforce nodes to create new leads or update existing ones based on duplicates.

- **1.8 Notifications**  
  Slack notifications for internal teams and personalized email notifications for clients.

---

### 2. Block-by-Block Analysis

#### 1.1 Chat Input Reception

- **Overview:**  
  Captures incoming chat messages from users via a public webhook, initiates the conversation with a greeting, and forwards messages to the AI Agent.

- **Nodes Involved:**  
  - `When chat message received`

- **Node Details:**  
  - **Type:** LangChain Chat Trigger  
  - **Role:** Entry trigger for chat messages via webhook  
  - **Configuration:**  
    - Public webhook enabled  
    - Initial greeting message: "Hi! I'm LeadBot. I'll help you submit your interest. Let's start with your full name."  
  - **Inputs:** External user chat messages  
  - **Outputs:** Forwards messages to AI Agent node  
  - **Version:** 1.1  
  - **Edge Cases:**  
    - Webhook connectivity issues  
    - Unexpected message formats  
  - **Sticky Note:**  
    - Describes the greeting and entry point role  
  - **Credentials:** None required

#### 1.2 AI Conversational Agent

- **Overview:**  
  The core chatbot logic node managing conversation flow, validations, tool invocations (duplicate checks, lead creation/updating, notifications), and user interactions based on a detailed system prompt.

- **Nodes Involved:**  
  - `AI Agent`

- **Node Details:**  
  - **Type:** LangChain Agent  
  - **Role:** Orchestrates conversational logic, decision-making, tool usage, and response generation  
  - **Configuration:**  
    - System message defines behavior rules for stepwise data collection (Name, Email, Mobile, Product Interest)  
    - Validates email and phone formats  
    - Calls tools for duplicate check, lead create/update, notifications, and internal reasoning  
    - Produces natural text output to users  
    - Limits replies to <100 words  
  - **Inputs:** Chat messages from trigger, memory, language model, and tool outputs  
  - **Outputs:** Responses to chat trigger, tool calls  
  - **Version:** 2  
  - **Edge Cases:**  
    - Input validation failures (email/phone)  
    - Ambiguous user input requiring reasoning  
    - Duplicate lead detection and update path  
    - Tool invocation failures or timeouts  
  - **Sticky Note:**  
    - Describes the core agent role and integrated tools  
  - **Credentials:** Uses linked OpenAI API, Salesforce OAuth, Slack API, SMTP credentials through individual nodes

#### 1.3 Language Model Backend

- **Overview:**  
  Provides GPT-4.1 language model for natural language understanding, generation, and reasoning.

- **Nodes Involved:**  
  - `OpenAI Chat Model`

- **Node Details:**  
  - **Type:** LangChain OpenAI Chat Model node  
  - **Role:** Supplies AI language capabilities for the agent  
  - **Configuration:**  
    - Model set to "gpt-4.1"  
    - Minimal options (no temperature or other advanced parameters set)  
  - **Inputs:** Chat context and system prompt from agent  
  - **Outputs:** AI-generated text for agent responses and reasoning  
  - **Version:** 1.2  
  - **Edge Cases:**  
    - API key authentication failures  
    - API rate limits or timeouts  
  - **Sticky Note:**  
    - Describes language model role and credential requirements  
  - **Credentials:** OpenAI API key

#### 1.4 Memory Management

- **Overview:**  
  Maintains a windowed conversation buffer to provide context continuity during the chat session.

- **Nodes Involved:**  
  - `Simple Memory`

- **Node Details:**  
  - **Type:** LangChain Memory Buffer Window  
  - **Role:** Stores recent chat messages to maintain context for the AI Agent  
  - **Configuration:** Default windowing without custom parameters  
  - **Inputs:** Conversation messages  
  - **Outputs:** Context for AI Agent  
  - **Version:** 1.3  
  - **Edge Cases:**  
    - Memory overflow or loss causing context drop  
  - **Sticky Note:**  
    - Describes memory management and connection to Salesforce query  
  - **Credentials:** None

#### 1.5 Duplicate Lead Checking

- **Overview:**  
  Checks Salesforce for existing leads by email to prevent duplicate entries.

- **Nodes Involved:**  
  - `check_duplicate_lead`

- **Node Details:**  
  - **Type:** Salesforce Tool (getAll)  
  - **Role:** Queries Salesforce leads with email filter to detect duplicates  
  - **Configuration:**  
    - Operation: getAll  
    - Condition: Email equals AI-provided email variable  
    - Return all matching records  
  - **Inputs:** Email from AI Agent  
  - **Outputs:** Lead existence status, lead_id, and details (name, birthday, interest)  
  - **Version:** 1  
  - **Edge Cases:**  
    - Salesforce OAuth failure  
    - No matching leads found (normal case)  
    - Multiple matches (should handle gracefully)  
  - **Sticky Note:**  
    - Explains query operation, credentials, and dynamic email filtering  
  - **Credentials:** Salesforce OAuth2 API

#### 1.6 Internal Reasoning Tool

- **Overview:**  
  Facilitates AI internal step-by-step reasoning for complex decisions like ambiguous inputs or name parsing.

- **Nodes Involved:**  
  - `Think`

- **Node Details:**  
  - **Type:** LangChain Think Tool  
  - **Role:** Provides internal AI reflection without external calls  
  - **Configuration:** No parameters; returns AI thoughts as JSON  
  - **Inputs:** Invoked by AI Agent when needed  
  - **Outputs:** Reasoning text to AI Agent for decision making  
  - **Version:** 1  
  - **Edge Cases:**  
    - Failure to reason may cause incorrect decisions  
  - **Sticky Note:**  
    - Describes role in improving accuracy and handling edge cases  
  - **Credentials:** None

#### 1.7 Lead Record Management

- **Overview:**  
  Handles Salesforce lead creation or update based on duplicate check results.

- **Nodes Involved:**  
  - `update_lead`  
  - `create_lead`

- **Node Details:**  

  - **update_lead:**  
    - **Type:** Salesforce Tool (update)  
    - **Role:** Updates existing lead record with collected data  
    - **Configuration:**  
      - Uses lead_id from duplicate check  
      - Updates firstname, lastname, email, mobilePhone, and custom ProductInterest__c field  
    - **Inputs:** Data from AI Agent variables  
    - **Outputs:** Success status and messages  
    - **Version:** 1  
    - **Edge Cases:** Auth failure, invalid lead_id, update conflicts  
    - **Sticky Note:** Details parameters and dynamic field usage  
    - **Credentials:** Salesforce OAuth2 API  

  - **create_lead:**  
    - **Type:** Salesforce Tool (create)  
    - **Role:** Creates new lead record with collected data  
    - **Configuration:**  
      - Default company name (empty string)  
      - Sets lastname, email, firstname, mobilePhone, ProductInterest__c  
    - **Inputs:** Data from AI Agent variables  
    - **Outputs:** Success status and new lead_id  
    - **Version:** 1  
    - **Edge Cases:** Auth failure, invalid data format  
    - **Sticky Note:** Details parameters and dynamic field injection  
    - **Credentials:** Salesforce OAuth2 API

#### 1.8 Notifications

- **Overview:**  
  Sends notifications internally via Slack and externally to clients via email after lead creation or update.

- **Nodes Involved:**  
  - `send_notification_internal`  
  - `send_notification_client`

- **Node Details:**  

  - **send_notification_internal:**  
    - **Type:** Slack Tool  
    - **Role:** Sends Slack message to internal user/channel about new or updated interest  
    - **Configuration:**  
      - Text dynamically composed by AI (e.g., "New/updated interest: [Name] - [Email] - [Interest]")  
      - Sends to specific Slack user ID (U08QYRBEE3V)  
    - **Inputs:** Lead info from AI Agent  
    - **Outputs:** Slack API response  
    - **Version:** 2.3  
    - **Edge Cases:** Slack auth failure, API rate limits  
    - **Sticky Note:** Explains Slack notification role and credentials  
    - **Credentials:** Slack API  

  - **send_notification_client:**  
    - **Type:** Email Send Tool  
    - **Role:** Sends personalized email to client based on product interest  
    - **Configuration:**  
      - Dynamic "fromEmail" (fixed as test@example.com)  
      - Dynamic "toEmail", subject, and HTML body from AI Agent  
    - **Inputs:** Email details from AI Agent  
    - **Outputs:** Email send status  
    - **Version:** 2.1  
    - **Edge Cases:** SMTP auth failure, invalid email addresses  
    - **Sticky Note:** Describes email notification parameters and credentials  
    - **Credentials:** SMTP account

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                               | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                              |
|-----------------------------|----------------------------------|-----------------------------------------------|----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received   | LangChain Chat Trigger            | Entry point: receives chat messages, starts conversation | External webhook            | AI Agent                    | This node starts the workflow on incoming chat messages with initial greeting                           |
| AI Agent                    | LangChain Agent                   | Core conversational logic, validation, tool calls | When chat message received, Simple Memory, OpenAI Chat Model, Tools (Think, Salesforce, Notifications) | None (outputs to chat trigger) | The heart of the bot managing conversation, validations, and tool calls                                |
| OpenAI Chat Model           | LangChain OpenAI Chat Model       | Provides GPT-4.1 NLP backend for AI Agent    | AI Agent                   | AI Agent                    | Provides the LLM backend (GPT-4.1) for the AI Agent                                                    |
| Simple Memory               | LangChain Memory Buffer Window    | Maintains chat context for AI Agent           | AI Agent                   | AI Agent                    | Manages conversation memory for context retention                                                     |
| check_duplicate_lead        | Salesforce Tool (getAll)           | Checks for existing Salesforce leads by email | AI Agent                   | AI Agent                    | Queries leads by email to detect duplicates, uses Salesforce OAuth                                    |
| Think                       | LangChain Think Tool               | Provides AI internal reasoning for complex cases | AI Agent                   | AI Agent                    | Used for internal step-by-step reasoning to improve decision making                                   |
| update_lead                 | Salesforce Tool (update)           | Updates existing Salesforce lead record       | AI Agent                   | AI Agent                    | Updates lead with collected info if duplicate found, Salesforce OAuth required                         |
| create_lead                 | Salesforce Tool (create)           | Creates new Salesforce lead record             | AI Agent                   | AI Agent                    | Creates lead if no duplicate found, Salesforce OAuth required                                         |
| send_notification_internal  | Slack Tool                       | Sends Slack notification to internal team     | AI Agent                   | AI Agent                    | Sends Slack message about new/updated leads to internal user                                          |
| send_notification_client    | Email Send Tool                  | Sends personalized email notification to client | AI Agent                   | AI Agent                    | Sends client email with product interest details, uses SMTP credentials                               |
| Sticky Note                 | Sticky Note                      | Documentation                                  | -                          | -                           | Workflow overview, explains purpose and flow                                                          |
| Sticky Note1                | Sticky Note                      | Documentation                                  | -                          | -                           | Memory Management explanation for Salesforce duplicate check                                          |
| Sticky Note2                | Sticky Note                      | Documentation                                  | -                          | -                           | Internal Reasoning tool description                                                                    |
| Sticky Note3                | Sticky Note                      | Documentation                                  | -                          | -                           | Language Model node explanation                                                                         |
| Sticky Note4                | Sticky Note                      | Documentation                                  | -                          | -                           | Chat Trigger step explanation                                                                           |
| Sticky Note5                | Sticky Note                      | Documentation                                  | -                          | -                           | Internal Slack notification details                                                                    |
| Sticky Note6                | Sticky Note                      | Documentation                                  | -                          | -                           | Client Email notification details                                                                      |
| Sticky Note7                | Sticky Note                      | Documentation                                  | -                          | -                           | Update Lead Salesforce tool details                                                                    |
| Sticky Note8                | Sticky Note                      | Documentation                                  | -                          | -                           | Duplicate Check Salesforce tool details                                                                |
| Sticky Note9                | Sticky Note                      | Documentation                                  | -                          | -                           | Create Lead Salesforce tool details                                                                    |
| Sticky Note10               | Sticky Note                      | Documentation                                  | -                          | -                           | Core AI Agent explanation                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `When chat message received` node**  
   - Type: LangChain Chat Trigger  
   - Set public webhook enabled  
   - Initial message: "Hi! I'm LeadBot. I'll help you submit your interest. Let's start with your full name."  
   - Position near workflow start

2. **Create `OpenAI Chat Model` node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4.1`  
   - Connect OpenAI API credentials  
   - Minimal options, no custom temperature  
   - Position beside AI Agent node

3. **Create `Simple Memory` node**  
   - Type: LangChain Memory Buffer Window  
   - Default configuration, no custom parameters  
   - Position near AI Agent for context retention

4. **Create `Think` node**  
   - Type: LangChain Think Tool  
   - No parameters  
   - Used for internal AI reasoning steps  
   - Position near AI Agent node

5. **Create `check_duplicate_lead` node**  
   - Type: Salesforce Tool (getAll)  
   - Operation: `getAll`  
   - Return all results: true  
   - Condition: Email equals `{{$fromAI('Email')}}` (use AI variable injection)  
   - Connect Salesforce OAuth2 credentials  
   - Position near AI Agent node

6. **Create `update_lead` node**  
   - Type: Salesforce Tool (update)  
   - Operation: `update`  
   - Lead ID: `{{$fromAI('Lead_ID')}}`  
   - Fields to update: firstname, lastname, email, mobilePhone, custom ProductInterest__c (all from AI variables)  
   - Connect Salesforce OAuth2 credentials  
   - Position near Salesforce duplicate check node

7. **Create `create_lead` node**  
   - Type: Salesforce Tool (create)  
   - Company: empty or default string  
   - Fields: lastname, firstname, email, mobilePhone, ProductInterest__c from AI variables  
   - Connect Salesforce OAuth2 credentials  
   - Position near update_lead node

8. **Create `send_notification_internal` node**  
   - Type: Slack Tool  
   - Text: `{{$fromAI('Message_Text')}}` (dynamic message from AI)  
   - User: Fixed Slack user ID (e.g., U08QYRBEE3V)  
   - Connect Slack API credentials  
   - Position near lead creation/updating nodes

9. **Create `send_notification_client` node**  
   - Type: Email Send Tool  
   - From Email: `test@example.com` (fixed)  
   - To Email, Subject, HTML Body: all dynamically injected from AI variables  
   - Connect SMTP credentials  
   - Position near send_notification_internal node

10. **Create `AI Agent` node**  
    - Type: LangChain Agent  
    - System message: Use detailed prompt specifying stepwise data collection (name, email, phone, product interest), validation rules, tool usage (check_duplicate_lead, create_lead, update_lead, notifications, think)  
    - Connect inputs from:  
      - `When chat message received` (main)  
      - `OpenAI Chat Model` (ai_languageModel)  
      - `Simple Memory` (ai_memory)  
      - `check_duplicate_lead`, `update_lead`, `create_lead`, `send_notification_internal`, `send_notification_client`, and `Think` nodes as ai_tools  
    - Position centrally to orchestrate flow

11. **Connect Nodes**  
    - From `When chat message received` main output to `AI Agent` main input  
    - From `OpenAI Chat Model` ai_languageModel output to `AI Agent` ai_languageModel input  
    - From `Simple Memory` ai_memory output to `AI Agent` ai_memory input  
    - From `check_duplicate_lead`, `update_lead`, `create_lead`, `send_notification_internal`, `send_notification_client`, and `Think` nodes outputs as ai_tools back to `AI Agent` ai_tool inputs  

12. **Configure Credentials**  
    - Set up OpenAI API key under credentials and assign to OpenAI Chat Model node  
    - Configure Salesforce OAuth2 API credentials and assign to Salesforce tool nodes  
    - Add Slack API credentials and assign to Slack Tool node  
    - Add SMTP credentials and assign to Email Send Tool node  

13. **Validation and Testing**  
    - Ensure all credentials are working and authorized  
    - Test webhook connectivity  
    - Verify AI Agent prompt and tool calls respond correctly to input flows  
    - Monitor logs for errors such as API timeouts or input validation failures  

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow creates a Chat-to-Lead conversational chatbot replacing web forms with AI-driven interaction.                         | Workflow Overview (Sticky Note)                                                                         |
| Salesforce OAuth2 credentials are required for querying and modifying leads.                                                    | Duplicate Check and Lead Management nodes                                                             |
| Slack API credentials and user/channel ID needed for internal notifications.                                                    | Internal Notification node                                                                              |
| SMTP credentials necessary for sending client emails.                                                                          | Client Notification node                                                                                |
| AI Agent uses LangChain tools and GPT-4.1 model for stepwise guided conversation, validation, and tool invocation.              | Core AI Agent node and Language Model node                                                             |
| Potential enhancements include adding more product options or regex validation in system prompt.                                | Workflow Overview sticky note suggestions                                                              |
| Slack notification sends to user ID `U08QYRBEE3V` (can be customized).                                                          | Slack Notification sticky note                                                                          |
| Client emails use fixed sender email `test@example.com`, customize as needed.                                                   | Client Notification sticky note                                                                         |
| AI Agent uses a "think" node for internal reasoning to handle ambiguous inputs or complex decisions without external calls.    | Internal Reasoning sticky note                                                                          |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow and complies fully with content policies. No illegal, offensive, or protected content is included. All processed data is legal and public.