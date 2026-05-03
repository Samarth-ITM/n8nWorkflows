Create a Knowledge Base & Scheduling AI Assistant with GPT-5, Google Calendar & Sheets

https://n8nworkflows.xyz/workflows/create-a-knowledge-base---scheduling-ai-assistant-with-gpt-5--google-calendar---sheets-8597


# Create a Knowledge Base & Scheduling AI Assistant with GPT-5, Google Calendar & Sheets

---

## 1. Workflow Overview

This workflow automates client interaction for scheduling and knowledge queries by leveraging ChatGPT-5, Google Sheets, Google Calendar, and Gmail. Its primary goal is to streamline answering company-related questions from a knowledge base and managing appointment scheduling with real-time availability checks and confirmations. The workflow is designed for businesses needing a conversational AI assistant that integrates knowledge lookup, calendar availability checks, appointment creation, and confirmation email dispatching—all localized to Paris time.

**Logical Blocks:**

- **1.1 Input Reception:** Captures user queries and interactions via a chat trigger.
- **1.2 AI Reasoning and Response Generation:** Processes input using ChatGPT-5, LangChain reasoning, and memory to generate contextual replies.
- **1.3 Knowledge Base Access:** Retrieves relevant factual information from Google Sheets acting as the company knowledge base.
- **1.4 Calendar Availability & Appointment Management:** Checks availability, proposes alternative slots, and creates appointments in Google Calendar.
- **1.5 Confirmation Communication:** Sends booking confirmation emails via Gmail.
- **1.6 User Guidance and Setup Notes:** Provides embedded tutorial links, documentation, and usage instructions via sticky notes.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block initiates the workflow by capturing user input through a chat interface, triggering downstream AI processing and integration logic.

**Nodes Involved:**  
- Chat with Your Data

**Node Details:**

- **Name:** Chat with Your Data  
- **Type:** LangChain Chat Trigger (`@n8n/n8n-nodes-langchain.chatTrigger`)  
- **Role:** Entry point capturing chat messages from users to start conversation sessions.  
- **Configuration:** Uses a webhook ID to receive chat data; no additional parameters specified.  
- **Input:** External user chat input via webhook.  
- **Output:** Passes chat content to AI Agent Support node for processing.  
- **Version:** 1.3  
- **Edge Cases:** Webhook connectivity issues, malformed requests, or unsupported message formats may cause failures.  
- **Notes:** Serves as the single entry point for user interaction.

---

### 2.2 AI Reasoning and Response Generation

**Overview:**  
Processes user input intelligently using the OpenAI GPT-5 language model combined with LangChain reasoning and a contextual memory buffer to maintain conversational state.

**Nodes Involved:**  
- AI Agent Support  
- OpenAI GPT-5 Model  
- Reasoning Tool (LangChain)  
- Memory

**Node Details:**

- **AI Agent Support**  
  - **Type:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`)  
  - **Role:** Central AI orchestrator handling user intents with predefined goals and rules integrating multiple tools.  
  - **Configuration:**  
    - System message defines three-tier goal priority: knowledge answering, availability checking, and appointment booking.  
    - Strict rules enforce a single call to knowledge base and appointment creation per request.  
    - Timezone handling ensures all user-facing times are in Paris time; API calls convert Paris time to CST for calendar operations.  
    - Response style is concise (max 3 sentences or bullets).  
    - Enforces privacy and limits mentions of internal tools.  
  - **Inputs:** User messages from Chat with Your Data, memory context, and outputs from tools.  
  - **Outputs:** Generates AI responses and triggers tool calls (knowledge base lookup, calendar checks, appointment creation, email sending).  
  - **Version:** 2.2  
  - **Edge Cases:**  
    - Violations of call limits per request cause errors.  
    - Timezone conversion errors could cause scheduling mishaps.  
    - Missing required user info (e.g., email) triggers additional prompts.  
    - Model or API rate limits and connectivity issues.  
  - **Sub-workflows:** None; this node orchestrates all logic internally.

- **OpenAI GPT-5 Model**  
  - **Type:** LangChain OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)  
  - **Role:** Provides the core language understanding and generation capabilities using GPT-5-mini model variant.  
  - **Configuration:**  
    - Model set explicitly to "gpt-5-mini".  
    - Credentials linked to OpenAI API key.  
  - **Input:** Receives prompts and instructions from AI Agent Support.  
  - **Output:** Returns AI-generated text responses.  
  - **Version:** 1.2  
  - **Edge Cases:** API key errors, rate limits, or model availability issues.

- **Reasoning Tool (LangChain)**  
  - **Type:** LangChain Tool for Thought (`@n8n/n8n-nodes-langchain.toolThink`)  
  - **Role:** Enables structured reasoning steps for the AI agent to deliberate before responding or invoking tools.  
  - **Configuration:** Default, no parameters set.  
  - **Input/Output:** Called by AI Agent Support to enhance decision-making.  
  - **Version:** 1.1  
  - **Edge Cases:** Processing delays or failures in reasoning logic.

- **Memory**  
  - **Type:** LangChain Memory Buffer Window (`@n8n/n8n-nodes-langchain.memoryBufferWindow`)  
  - **Role:** Maintains conversational context with a window of last 10 interactions to provide continuity.  
  - **Configuration:** `contextWindowLength` set to 10.  
  - **Input:** Conversation history from AI Agent Support.  
  - **Output:** Provides context to AI Agent Support for subsequent turns.  
  - **Version:** 1.3  
  - **Edge Cases:** Memory overflow if context is too large, causing truncation of older messages.

---

### 2.3 Knowledge Base Access

**Overview:**  
Fetches relevant company or training information from a Google Sheets-based knowledge base to respond factually to user queries.

**Nodes Involved:**  
- Knowledgebase Lookup

**Node Details:**

- **Name:** Knowledgebase Lookup  
- **Type:** Google Sheets Tool (`n8n-nodes-base.googleSheetsTool`)  
- **Role:** Reads data from a configured Google Sheets document containing FAQs, policies, and training packs.  
- **Configuration:**  
  - Uses OAuth2 credentials for Google Sheets access.  
  - Document ID and Sheet Name set via expressions to dynamically target the knowledge base.  
  - Intended for a single call per request as constrained by AI Agent Support.  
- **Input:** Triggered by AI Agent Support when a knowledge query is detected.  
- **Output:** Returns matching knowledge data to AI Agent Support for response generation.  
- **Version:** 4.7  
- **Edge Cases:**  
  - Credential expiry or permission errors.  
  - Malformed or missing data in the sheet.  
  - Network timeouts or API rate limits.

---

### 2.4 Calendar Availability & Appointment Management

**Overview:**  
Manages checking calendar availability, proposing alternative times, and creating confirmed appointments in Google Calendar localized to Paris time.

**Nodes Involved:**  
- Calendar: Check Availability  
- Calendar: Create Appointment

**Node Details:**

- **Calendar: Check Availability**  
  - **Type:** Google Calendar Tool (`n8n-nodes-base.googleCalendarTool`)  
  - **Role:** Queries the Google Calendar for free/busy slots within a specified time range.  
  - **Configuration:**  
    - Uses OAuth2 credentials for Google Calendar.  
    - TimeMin and TimeMax parameters are dynamically provided by AI Agent Support using expressions.  
    - Returns all matching events within the queried timeframe.  
  - **Input:** Called multiple times by AI Agent Support to validate availability and find alternative slots.  
  - **Output:** Provides availability status to AI Agent Support.  
  - **Version:** 1.3  
  - **Edge Cases:**  
    - Timezone conversion errors (must convert Paris → CST).  
    - API quota or permission issues.  
    - If calendar IDs are misconfigured, queries can return incorrect data.

- **Calendar: Create Appointment**  
  - **Type:** Google Calendar Tool (`n8n-nodes-base.googleCalendarTool`)  
  - **Role:** Creates a new calendar event (1-hour appointment) after client confirmation.  
  - **Configuration:**  
    - OAuth2 credentials for calendar access.  
    - Start, End times, summary, and description are dynamically set by AI Agent Support.  
    - Enforces 1-hour duration.  
  - **Input:** Triggered once per request by AI Agent Support upon booking confirmation.  
  - **Output:** Returns event creation confirmation.  
  - **Version:** 1.3  
  - **Edge Cases:**  
    - Attempting to book an already taken slot.  
    - Failure if called multiple times in a single request.  
    - Timezone misalignment causing incorrect booking time.

---

### 2.5 Confirmation Communication

**Overview:**  
Sends a single confirmation email with booking details to the client, ensuring proper communication post-appointment creation.

**Nodes Involved:**  
- Send Booking Confirmation

**Node Details:**

- **Name:** Send Booking Confirmation  
- **Type:** Gmail Tool (`n8n-nodes-base.gmailTool`)  
- **Role:** Sends an email with appointment details and instructions.  
- **Configuration:**  
  - Uses OAuth2 credentials for Gmail access.  
  - Recipient email, subject, and message body are dynamically set by AI Agent Support.  
  - Only one email sent per booking request.  
- **Input:** Triggered by AI Agent Support after appointment creation success.  
- **Output:** Email dispatch confirmation.  
- **Version:** 2.1  
- **Edge Cases:**  
  - Missing or invalid client email address triggers additional prompts.  
  - Gmail API quota or auth failures.  
  - Email formatting errors if variables are missing.

---

### 2.6 User Guidance and Setup Notes

**Overview:**  
Provides embedded documentation, tutorial references, and setup instructions through sticky notes directly in the workflow canvas to assist users in installation, customization, and understanding.

**Nodes Involved:**  
- Sticky Note8  
- Sticky Note2  
- Sticky Note7  
- Sticky Note9  
- Sticky Note

**Node Details:**

- **Sticky Note8**  
  - Large tutorial and documentation reference including YouTube video link and Notion docs.  
  - Explains problem solved, setup steps, customization, and support contacts.

- **Sticky Note2**  
  - Summarizes the workflow capabilities and integration with knowledge base, calendar, and email.

- **Sticky Note7**  
  - Highlights user interaction capabilities: asking questions, checking availability, and booking.

- **Sticky Note9**  
  - Details on setting up OpenAI API key and billing.

- **Sticky Note** (unnamed)  
  - Instructions on knowledge base formatting, scheduling rules, timezone handling, and tutorial references.

- **Type:** Sticky Note (`n8n-nodes-base.stickyNote`)  
- **Role:** Informational only, no runtime function.  
- **Edge Cases:** None (static content).

---

## 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                      | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                                                                                                                                                                                                           |
|---------------------------|-----------------------------------------|------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Chat with Your Data        | LangChain Chat Trigger                   | Captures user chat input           | -                           | AI Agent Support            |                                                                                                                                                                                                                                                                                                     |
| AI Agent Support           | LangChain Agent                         | Central AI logic orchestrator      | Chat with Your Data, Memory, Reasoning Tool, OpenAI GPT-5 Model, Knowledgebase Lookup, Calendar nodes, Send Booking Confirmation | Reasoning Tool, Knowledgebase Lookup, Calendar: Check Availability, Calendar: Create Appointment, Send Booking Confirmation, Memory, OpenAI GPT-5 Model |                                                                                                                                                                                                                                                                                                     |
| OpenAI GPT-5 Model         | LangChain OpenAI Chat Model             | Language model for AI responses    | AI Agent Support             | AI Agent Support             |                                                                                                                                                                                                                                                                                                     |
| Reasoning Tool (LangChain) | LangChain ToolThink                     | AI reasoning and thought process  | AI Agent Support             | AI Agent Support             |                                                                                                                                                                                                                                                                                                     |
| Memory                    | LangChain Memory Buffer Window          | Maintains conversation context    | AI Agent Support             | AI Agent Support             |                                                                                                                                                                                                                                                                                                     |
| Knowledgebase Lookup       | Google Sheets Tool                      | Reads knowledge base data          | AI Agent Support             | AI Agent Support             |                                                                                                                                                                                                                                                                                                     |
| Calendar: Check Availability| Google Calendar Tool                   | Checks calendar availability      | AI Agent Support             | AI Agent Support             |                                                                                                                                                                                                                                                                                                     |
| Calendar: Create Appointment| Google Calendar Tool                   | Creates confirmed appointments     | AI Agent Support             | AI Agent Support             |                                                                                                                                                                                                                                                                                                     |
| Send Booking Confirmation  | Gmail Tool                             | Sends booking confirmation email  | AI Agent Support             | -                           |                                                                                                                                                                                                                                                                                                     |
| Sticky Note8               | Sticky Note                            | Tutorial, documentation, support  | -                           | -                           | ### 🎥 Watch This Tutorial @[youtube](MYLub9KYlu8)  \n\n### 📥  [Open full documentation on Notion](https://automatisation.notion.site/Build-Your-First-AI-Agent-with-ChatGPT-5-By-Dr-Firas-26f3d6550fd9801eb00dc0c578fc5f2c?source=copy_link)  \n---\nManually handles client questions, scheduling, and booking automation. |
| Sticky Note2               | Sticky Note                            | Workflow summary overview          | -                           | -                           | Allows chatting with AI Agent powered by GPT-5 linked to Google Sheets knowledge base, Google Calendar availability checking, and booking with Paris time enforcement.                                                                                                                              |
| Sticky Note7               | Sticky Note                            | User query capabilities overview   | -                           | -                           | Ask questions on company/training, check Paris time availability, confirm and book appointments.                                                                                                                                                                                                    |
| Sticky Note9               | Sticky Note                            | OpenAI API setup instructions      | -                           | -                           | Setup OpenAI API key and billing: [OpenAI Platform](https://platform.openai.com/api-keys), [OpenAI Billing](https://platform.openai.com/settings/organization/billing/overview)                                                                                                                     |
| Sticky Note                | Sticky Note                            | Scheduling and knowledge base setup| -                           | -                           | Connect Google Sheets knowledge base, check availability in Paris time, propose 3 alternatives, create 1-hour appointments after confirmation, send confirmation email, tutorial for Google Sheets, Gmail, Calendar credentials setup: [YouTube](https://youtu.be/fDzVmdw7bNU)                        |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Node:** Chat with Your Data  
   - Type: LangChain Chat Trigger  
   - Configuration: Assign a webhook ID to receive user chat messages.  
   - No credentials required.  

2. **Create Node:** AI Agent Support  
   - Type: LangChain Agent  
   - Configuration:  
     - Set system message with defined goals and rules: prioritize knowledge answering, then availability checking, then booking.  
     - Enforce single calls to knowledge base and appointment creation per request.  
     - Enforce Paris time usage and conversion to CST for calendar operations.  
     - Response style: concise (max 3 sentences/bullets).  
   - No credentials assigned here; it orchestrates other nodes.  

3. **Create Node:** OpenAI GPT-5 Model  
   - Type: LangChain OpenAI Chat Model  
   - Configuration: Model selected as "gpt-5-mini".  
   - Credentials: Provide OpenAI API key credentials.  

4. **Create Node:** Reasoning Tool (LangChain)  
   - Type: LangChain ToolThink  
   - Configuration: Default parameters.  

5. **Create Node:** Memory  
   - Type: LangChain Memory Buffer Window  
   - Configuration: Set context window length to 10 interactions.  

6. **Connect AI Agent Support Inputs:**  
   - Connect Chat with Your Data output to AI Agent Support input.  
   - Connect OpenAI GPT-5 Model, Reasoning Tool, Memory outputs to AI Agent Support respective inputs.  

7. **Create Node:** Knowledgebase Lookup  
   - Type: Google Sheets Tool  
   - Configuration:  
     - Set Google Sheets OAuth2 credentials.  
     - Specify the knowledge base document ID and sheet name via expressions or static values.  
   - Connect AI Agent Support “ai_tool” output to this node’s input.  

8. **Create Node:** Calendar: Check Availability  
   - Type: Google Calendar Tool  
   - Configuration:  
     - Use Google Calendar OAuth2 credentials.  
     - Set operation to “getAll”.  
     - Parameters for timeMin and timeMax are dynamically set by AI Agent Support.  
   - Connect AI Agent Support “ai_tool” output to this node’s input.  

9. **Create Node:** Calendar: Create Appointment  
   - Type: Google Calendar Tool  
   - Configuration:  
     - Use Google Calendar OAuth2 credentials.  
     - Parameters for start time, end time, summary, description are dynamically set by AI Agent Support.  
   - Connect AI Agent Support “ai_tool” output to this node’s input.  

10. **Create Node:** Send Booking Confirmation  
    - Type: Gmail Tool  
    - Configuration:  
      - Use Gmail OAuth2 credentials.  
      - Recipient, subject, and message body parameters are dynamically set by AI Agent Support.  
    - Connect AI Agent Support “ai_tool” output to this node’s input.  

11. **Connect AI Agent Support outputs:**  
    - Connect to Knowledgebase Lookup, Calendar nodes, Reasoning Tool, OpenAI GPT-5 Model, Memory, and Send Booking Confirmation as per the roles above.  

12. **Setup Sticky Notes:**  
    - Add informational sticky notes on canvas with the following content:  
      - Tutorial video link: `MYLub9KYlu8` (YouTube)  
      - Notion documentation link: https://automatisation.notion.site/Build-Your-First-AI-Agent-with-ChatGPT-5-By-Dr-Firas-26f3d6550fd9801eb00dc0c578fc5f2c?source=copy_link  
      - Instructions on Google Sheets knowledge base format (link sample provided).  
      - OpenAI API key setup instructions with billing links.  
      - Scheduling rules and Paris timezone enforcement notes.  

13. **Credential Setup:**  
    - Configure OAuth2 credentials for:  
      - Google Sheets API  
      - Google Calendar API  
      - Gmail API  
      - OpenAI API (using API key)  

14. **Final Testing:**  
    - Import or test trigger via Chat with Your Data webhook.  
    - Validate AI Agent answers questions from the knowledge base.  
    - Validate calendar availability checks and alternative proposals.  
    - Verify appointment booking creates 1-hour events in Paris time.  
    - Confirm booking confirmation emails are sent correctly.  

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| 🎥 Watch the detailed tutorial video on YouTube for setup and customization guidance.                                                                                                                                                                                                                          | https://youtu.be/MYLub9KYlu8                                                                                                  |
| 📥 Full documentation hosted on Notion provides extensive explanation and customization tips.                                                                                                                                                                                                                  | https://automatisation.notion.site/Build-Your-First-AI-Agent-with-ChatGPT-5-By-Dr-Firas-26f3d6550fd9801eb00dc0c578fc5f2c        |
| Sample Knowledgebase template for Google Sheets to structure FAQs, policies, and training packs.                                                                                                                                                                                                                | https://docs.google.com/spreadsheets/d/1TIaqkiVRr-Z3VLC-mvXW2Ak1zO0becK1-wqcgVeop0E/copy                                        |
| OpenAI API key and billing setup instructions to ensure uninterrupted AI service access.                                                                                                                                                                                                                        | https://platform.openai.com/api-keys & https://platform.openai.com/settings/organization/billing/overview                      |
| Tutorial for configuring Google Sheets, Gmail, and Google Calendar OAuth2 credentials in n8n.                                                                                                                                                                                                                   | https://youtu.be/fDzVmdw7bNU                                                                                                  |
| Contact Dr. Firas for consulting and workflow customization support via LinkedIn or YouTube.                                                                                                                                                                                                                    | LinkedIn: https://www.linkedin.com/in/dr-firas/  YouTube: https://www.youtube.com/@DRFIRASS                                    |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is lawful and public.

---