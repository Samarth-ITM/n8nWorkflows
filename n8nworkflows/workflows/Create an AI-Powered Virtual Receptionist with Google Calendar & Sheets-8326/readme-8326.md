Create an AI-Powered Virtual Receptionist with Google Calendar & Sheets

https://n8nworkflows.xyz/workflows/create-an-ai-powered-virtual-receptionist-with-google-calendar---sheets-8326


# Create an AI-Powered Virtual Receptionist with Google Calendar & Sheets

### 1. Workflow Overview

This n8n workflow implements an AI-powered virtual receptionist designed to handle customer interactions, manage appointment bookings, and maintain business records using Google Calendar and Google Sheets integration. It leverages OpenAI GPT models for natural language understanding and response generation, LangChain components for memory and agent orchestration, and Google APIs for calendar and spreadsheet operations.

**Target Use Cases:**  
- Automating customer conversation handling with AI for service inquiries and appointment scheduling  
- Checking real-time availability on Google Calendar  
- Booking appointments directly into Google Calendar  
- Persisting appointment and customer data into Google Sheets for record-keeping  
- Providing business-specific context and personalized interactions based on data fetched from Google Sheets  

**Logical Blocks:**  
- **1.1 Input Reception:** Capturing incoming chat messages from users  
- **1.2 Business Context Retrieval:** Loading business details and service information from Google Sheets to provide AI context  
- **1.3 AI Receptionist Agent:** Core AI logic processing user input, managing conversation memory, and orchestrating tools for booking and information retrieval  
- **1.4 Calendar Availability Check:** Verifying appointment slots against Google Calendar  
- **1.5 Appointment Booking:** Creating calendar events and saving records in Google Sheets  
- **1.6 Conversation Memory Management:** Maintaining context window for AI conversations  
- **1.7 Output Parsing (Disabled):** Parsing structured AI responses (disabled in this workflow)

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives chat messages from customers through a webhook to initiate the AI conversation flow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - *When chat message received*  
    - **Type:** LangChain chatTrigger (webhook trigger)  
    - **Role:** Entry point that triggers the workflow when a chat message arrives  
    - **Configuration:** Uses a webhook ID to listen for incoming chat messages  
    - **Inputs:** External webhook calls with chat input  
    - **Outputs:** Forwards message to “Get business details” node  
    - **Edge Cases:** Webhook downtime, malformed input, or missing chat content could break processing  

---

#### 1.2 Business Context Retrieval

- **Overview:**  
Fetches business-specific details such as services, hours, policies, and AI personality from a Google Sheet. Supplies this context to the AI receptionist for personalized and accurate responses.

- **Nodes Involved:**  
  - Get business details

- **Node Details:**  
  - *Get business details*  
    - **Type:** Google Sheets node  
    - **Role:** Reads business info from a predefined Google Sheets document and sheet (gid=0)  
    - **Configuration:** Uses OAuth2 credentials for Google Sheets access; fetches entire sheet content  
    - **Outputs:** Business data JSON to AI Receptionist node  
    - **Edge Cases:** Credential expiry, missing or malformed sheet, API rate limits  

---

#### 1.3 AI Receptionist Agent

- **Overview:**  
Central AI agent node that receives user input and business context, uses OpenAI GPT-4 to generate responses, manages conversation memory, and orchestrates downstream tool nodes for calendar checks and bookings.

- **Nodes Involved:**  
  - AI Receptionist  
  - OpenAI Chat Model  
  - Conversation Memory  
  - Book Calendar Appointment  
  - Check Calendar Availability  
  - Save Appointment Record  
  - Structured Output Parser (disabled)  
  - OpenAI Chat Model1  

- **Node Details:**  
  - *AI Receptionist*  
    - **Type:** LangChain Agent  
    - **Role:** Orchestrates AI chat response generation, tool usage, and context application  
    - **Configuration:**  
      - Uses user input from “When chat message received”  
      - System prompt dynamically built with business details (name, services, hours, policies, personality)  
      - Defines booking workflow steps and guidelines in prompt  
      - Outputs JSON response including “ai_reply”  
      - Connects to multiple AI tools (calendar check, booking, data saving)  
    - **Inputs:** User chat, business details, conversation memory  
    - **Outputs:** AI-generated replies, tool invocations  
    - **Edge Cases:** Model availability, prompt formatting errors, API rate limits, improper tool response handling  

  - *OpenAI Chat Model*  
    - **Type:** LangChain OpenAI Chat Model (GPT-4.1-mini)  
    - **Role:** Provides language model completions as part of AI Receptionist agent  
    - **Credentials:** OpenAI API key with free credits  
    - **Edge Cases:** API key invalid/expired, request timeouts, token limits  

  - *Conversation Memory*  
    - **Type:** LangChain memoryBufferWindow  
    - **Role:** Maintains a sliding window of last 15 conversation exchanges for context retention  
    - **Configuration:** Context window length set to 15  
    - **Inputs:** AI conversation data  
    - **Outputs:** Context updates to AI Receptionist agent  
    - **Edge Cases:** Memory overflow or truncation of important context  

  - *Book Calendar Appointment*  
    - **Type:** Google Calendar Tool  
    - **Role:** Creates an event in Google Calendar based on AI-provided start/end datetime and event details  
    - **Configuration:**  
      - Calendar ID hardcoded to ris362720@gmail.com  
      - Uses AI output fields for event title and description  
      - No default reminders set  
      - OAuth2 Google Calendar credentials configured  
    - **Inputs:** AI tool invocation from AI Receptionist  
    - **Outputs:** Confirmation or failure of booking  
    - **Edge Cases:** Calendar API quota, overlapping events, invalid datetime formats  

  - *Check Calendar Availability*  
    - **Type:** Google Calendar Tool  
    - **Role:** Checks calendar free/busy status between AI-provided start and end datetime  
    - **Configuration:** Calendar ID same as booking node  
    - **Inputs:** AI invocation with start/end times  
    - **Outputs:** Availability status for AI decision-making  
    - **Edge Cases:** API errors, timezone mismatches, no events found edge case  

  - *Save Appointment Record*  
    - **Type:** Google Sheets Tool  
    - **Role:** Appends appointment and customer data (summary, event_id, services, patient name/number) to a Google Sheet for record-keeping  
    - **Configuration:**  
      - Targets specific spreadsheet and sheet (Sheet2, gid=1454968607)  
      - Maps AI output fields to sheet columns  
      - OAuth2 Google Sheets credentials configured  
    - **Inputs:** AI tool invocation after booking completion  
    - **Outputs:** Confirmation of record save  
    - **Edge Cases:** Sheet permission issues, data format mismatch, API quota  

  - *OpenAI Chat Model1*  
    - **Type:** LangChain OpenAI Chat Model (GPT-4.1-mini)  
    - **Role:** Connected to the disabled Structured Output Parser (likely a fallback or experimental node)  
    - **Edge Cases:** Currently disabled, so no active role  

  - *Structured Output Parser* (Disabled)  
    - **Type:** LangChain outputParserStructured  
    - **Role:** Intended to parse AI JSON formatted replies automatically, currently disabled  
    - **Configuration:** Example JSON schema provided for ai_reply message  
    - **Edge Cases:** Parsing errors, format mismatches  

---

#### 1.4 Conversation Memory Management

- **Overview:**  
Maintains a rolling window of conversation history to provide relevant context to the AI model on each interaction.

- **Nodes Involved:**  
  - Conversation Memory (also part of AI Receptionist block)

- **Node Details:**  
  See details in AI Receptionist section above.

---

#### 1.5 Sticky Notes (Documentation Nodes)

- **Overview:**  
Non-executable nodes providing human-readable documentation and workflow explanations. Two sticky notes summarize the business details fetching and overall AI receptionist functionality.

- **Nodes Involved:**  
  - Sticky Note1  
  - Sticky Note

- **Node Details:**  
  - *Sticky Note1*  
    - Content explains the role of “Get Business Details” node: pulls service list, hours, policies, AI personality from Google Sheets to provide business-specific context to AI.  
  - *Sticky Note*  
    - Describes the AI Receptionist Agent block: handles customer chat, books appointments, saves records leveraging GPT and Google APIs.

---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                           | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                                             |
|-------------------------|----------------------------------------|-----------------------------------------|-----------------------------|--------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger  | Entry point webhook for chat input      | -                           | Get business details            |                                                                                                                         |
| Get business details     | n8n-nodes-base.googleSheets             | Fetch business context from Google Sheets | When chat message received   | AI Receptionist                | ## Get Business Details Pulls service list, hours, policies, and AI personality from Google Sheets. Provides AI context. |
| AI Receptionist         | @n8n/n8n-nodes-langchain.agent          | Core AI agent managing conversation, tools | Get business details, Conversation Memory, OpenAI Chat Model, Book Calendar Appointment, Check Calendar Availability, Save Appointment Record | -                              | ## AI Receptionist Agent Handles customer chat with GPT + business context, books appointments via Google Calendar, saves records.|
| Conversation Memory      | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains recent conversation history   | -                           | AI Receptionist                |                                                                                                                         |
| OpenAI Chat Model        | @n8n/n8n-nodes-langchain.lmChatOpenAi  | Provides GPT-4 language model completions | -                           | AI Receptionist                |                                                                                                                         |
| Check Calendar Availability | n8n-nodes-base.googleCalendarTool     | Checks calendar availability for booking | AI Receptionist             | AI Receptionist                |                                                                                                                         |
| Book Calendar Appointment | n8n-nodes-base.googleCalendarTool      | Books appointment in Google Calendar    | AI Receptionist             | AI Receptionist                |                                                                                                                         |
| Save Appointment Record  | n8n-nodes-base.googleSheetsTool          | Saves appointment/customer data          | AI Receptionist             | -                              |                                                                                                                         |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured (disabled) | Intended to parse AI JSON replies        | OpenAI Chat Model1          | AI Receptionist                |                                                                                                                         |
| OpenAI Chat Model1       | @n8n/n8n-nodes-langchain.lmChatOpenAi  | GPT-4 model connected to disabled parser | -                           | Structured Output Parser       |                                                                                                                         |
| Sticky Note1             | n8n-nodes-base.stickyNote                | Documentation on business details node   | -                           | -                              | ## Get Business Details Pulls service list, hours, policies, and AI personality from Google Sheets. Provides AI context. |
| Sticky Note              | n8n-nodes-base.stickyNote                | Documentation on AI Receptionist agent   | -                           | -                              | ## AI Receptionist Agent Handles customer chat with GPT + business context, books appointments via Google Calendar, saves records.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add **When chat message received** node (LangChain chatTrigger).  
   - Configure webhook with a unique webhook ID to receive chat messages.

2. **Add Google Sheets Node for Business Details:**  
   - Add **Get business details** node (Google Sheets).  
   - Configure OAuth2 credentials for Google Sheets access.  
   - Set Document ID to `1sEfkvCT_4GGlleZKcQfWy1Z8V2FEg2BH2eY0LDzrnt8`.  
   - Set Sheet Name to `gid=0` (Sheet1).  
   - Connect output of "When chat message received" to input of this node.

3. **Add Conversation Memory Node:**  
   - Add **Conversation Memory** node (LangChain memoryBufferWindow).  
   - Set contextWindowLength to 15.  
   - This node manages recent chat context for AI.

4. **Add OpenAI Chat Model Node:**  
   - Add **OpenAI Chat Model** node (LangChain lmChatOpenAi).  
   - Select model `gpt-4.1-mini`.  
   - Configure OpenAI API credentials with a valid API key.  
   - This node will generate AI chat completions.

5. **Add AI Receptionist Agent Node:**  
   - Add **AI Receptionist** node (LangChain agent).  
   - Set input text to `={{ $('When chat message received').item.json.chatInput }}`.  
   - Construct a detailed system prompt including business details from "Get business details" node JSON data, specifying AI personality, business hours, services, booking workflow, and guidelines.  
   - Attach Conversation Memory node as AI memory source.  
   - Connect OpenAI Chat Model as AI language model.  
   - Configure AI tools with:  
     - Book Calendar Appointment  
     - Check Calendar Availability  
     - Save Appointment Record  
   - Enable output parsing (optional).

6. **Add Calendar Availability Node:**  
   - Add **Check Calendar Availability** node (Google Calendar Tool).  
   - Configure OAuth2 Google Calendar credentials.  
   - Set calendar email to `ris362720@gmail.com`.  
   - Set `timeMin` and `timeMax` dynamically from AI output (`start_datetime`, `end_datetime`).  
   - Connect as AI tool to AI Receptionist node.

7. **Add Calendar Booking Node:**  
   - Add **Book Calendar Appointment** node (Google Calendar Tool).  
   - Configure OAuth2 Google Calendar credentials.  
   - Set calendar email to `ris362720@gmail.com`.  
   - Set `start` and `end` dynamically from AI output.  
   - Set event summary and description from AI output fields (`event_title`, `event_description`).  
   - Disable default reminders.  
   - Connect as AI tool to AI Receptionist node.

8. **Add Save Appointment Record Node:**  
   - Add **Save Appointment Record** node (Google Sheets Tool).  
   - Configure OAuth2 Google Sheets credentials.  
   - Set Document ID to same as business details sheet.  
   - Set Sheet Name to `1454968607` (Sheet2).  
   - Map columns `summary`, `event_id`, `services`, `patient name `, `patient number` to AI output fields with the same names.  
   - Operation: Append row.  
   - Connect as AI tool to AI Receptionist node.

9. **Optional: Add Structured Output Parser Node:**  
   - Add **Structured Output Parser** node (LangChain outputParserStructured).  
   - Provide example JSON schema with field `"ai_reply"`.  
   - Connect downstream of OpenAI Chat Model1 (optional, currently disabled).

10. **Connect Nodes:**  
    - "When chat message received" → "Get business details" → "AI Receptionist"  
    - "Conversation Memory" → "AI Receptionist" (memory input)  
    - "OpenAI Chat Model" → "AI Receptionist" (language model)  
    - "Check Calendar Availability", "Book Calendar Appointment", "Save Appointment Record" → "AI Receptionist" (tools)  

11. **Test and Validate:**  
    - Ensure all OAuth2 credentials are valid and authorized.  
    - Confirm Google Sheets and Calendar APIs are accessible.  
    - Verify webhook triggers correctly receive chat messages.  
    - Perform test conversations to validate booking flow and data saving.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The workflow combines AI natural language processing with business data to automate receptionist tasks. | Workflow title: "Create an AI-Powered Virtual Receptionist with Google Calendar & Sheets"                   |
| Business context (services, hours, policies) is stored and updated in Google Sheets for easy management. | Google Sheets document ID: `1sEfkvCT_4GGlleZKcQfWy1Z8V2FEg2BH2eY0LDzrnt8`                                  |
| Google Calendar integration uses OAuth2 credentials to access and modify calendar events.                | Calendar email configured: `ris362720@gmail.com`                                                           |
| AI persona and role are dynamically injected via system prompt for customized conversational style.     | Uses LangChain Agent with GPT-4.1-mini as the language model.                                               |
| Sticky notes within the workflow provide useful explanations for business users and developers.          | Sticky Note1 explains business details retrieval; Sticky Note summarizes AI Receptionist functionality.    |

---

**Disclaimer:** This document is generated from an automated n8n workflow export. All data manipulated are legal and publicly accessible. No illegal or offensive content is included.