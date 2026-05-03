Daily Meeting Summaries with Google Calendar & Gemini for Slack-Discord-WhatsApp

https://n8nworkflows.xyz/workflows/daily-meeting-summaries-with-google-calendar---gemini-for-slack-discord-whatsapp-6934


# Daily Meeting Summaries with Google Calendar & Gemini for Slack-Discord-WhatsApp

### 1. Workflow Overview

This workflow automates the generation and distribution of daily meeting summaries by integrating Google Calendar events with advanced AI processing via Google Gemini and LangChain. It targets teams using Slack, Discord, and WhatsApp for communication, providing concise AI-generated summaries of scheduled meetings.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily.
- **1.2 AI Processing:** Uses LangChain AI Agent combined with Google Gemini Chat Model to process calendar data.
- **1.3 Google Calendar Integration:** Fetches calendar events from two Google Calendar nodes to supply meeting data.
- **1.4 Message Distribution:** Sends the AI-generated summary to Slack, Discord, and WhatsApp channels.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block acts as the entry point, triggering the workflow automatically on a schedule (presumably daily, although no explicit schedule parameters are set).

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: scheduleTrigger  
    - Role: Initiates the workflow at defined intervals without manual intervention.  
    - Configuration: Default scheduling parameters; specific cron or interval not set in JSON, implying default or externally configured schedule.  
    - Inputs: None (trigger node).  
    - Outputs: Connects to AI Agent node.  
    - Edge Cases: If the trigger fails (n8n server down), the daily summary won't run. No retry logic shown.  
    - Version: 1.2  

---

#### 2.2 AI Processing

- **Overview:**  
  Central processing block where the AI Agent leverages LangChain with Google Gemini Chat Model and Google Calendar tools to analyze calendar data and generate meeting summaries.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Google Calendar 1  
  - Google Calendar 2  

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Orchestrates AI workflows using configured tools and language models.  
    - Configuration: Connects to two Google Calendar tools (Google Calendar 1 & 2) as AI tools and to Google Gemini Chat Model as the language model. Acts as the core AI orchestrator.  
    - Inputs: Receives trigger from Schedule Trigger; receives AI tool data from two calendar nodes and language model from Google Gemini.  
    - Outputs: Sends processed summary to messaging nodes (Slack, Discord, WhatsApp).  
    - Edge Cases: Possible errors include AI model unavailability, tool misconfiguration, or input data errors from calendars. Requires proper credentials.  
    - Version: 2.1  

  - **Google Gemini Chat Model**  
    - Type: Google Gemini Chat Model (AI language model node)  
    - Role: Provides the advanced natural language processing capability for generating meeting summaries.  
    - Configuration: No explicit parameters shown; expected to be linked to Google Cloud project with Gemini API credentials.  
    - Inputs: Feeds into AI Agent as the language model.  
    - Outputs: To AI Agent.  
    - Edge Cases: Authentication failures, API limits, or response timeouts may occur.  
    - Version: 1  

  - **Google Calendar 1 & Google Calendar 2**  
    - Type: Google Calendar Tool nodes  
    - Role: Serve as data sources, fetching calendar events to be summarized. Multiple calendar nodes allow simultaneous access to different calendars or calendars with different scopes.  
    - Configuration: Not explicitly detailed, but expected to be configured with Google OAuth2 credentials and calendar IDs or filters.  
    - Inputs: None (tool nodes).  
    - Outputs: Both connect as AI tools into AI Agent.  
    - Edge Cases: Authentication errors, empty event results, or API quota limits possible.  
    - Version: 1.3  

---

#### 2.3 Message Distribution

- **Overview:**  
  Sends the AI-generated meeting summaries to multiple communication platforms simultaneously: Slack, Discord, and WhatsApp.

- **Nodes Involved:**  
  - Send to Slack  
  - Send to Discord  
  - Send to WhatsApp  

- **Node Details:**  
  - **Send to Slack**  
    - Type: Slack node  
    - Role: Posts the summary message to a Slack workspace/channel.  
    - Configuration: Uses a webhook ID (f3e29a3d-e761-4842-a6e3-e5a4ac264302) indicating preconfigured Slack OAuth2 or webhook credentials.  
    - Inputs: Receives from AI Agent output.  
    - Outputs: None (terminal).  
    - Edge Cases: Message formatting errors, webhook invalidation, or rate limits.  
    - Version: 2.3  

  - **Send to Discord**  
    - Type: Discord node  
    - Role: Posts the summary message to a Discord channel.  
    - Configuration: Uses a webhook ID (43f7291e-f498-47dd-8d5d-ad7e8985f6cc).  
    - Inputs: Receives from AI Agent output.  
    - Outputs: None (terminal).  
    - Edge Cases: Webhook failure or Discord rate limits.  
    - Version: 2  

  - **Send to WhatsApp**  
    - Type: WhatsApp node  
    - Role: Sends the summary via WhatsApp messaging.  
    - Configuration: Configured with webhook ID (98fd8d5f-09b6-4f94-97b3-9f36ebcda4ce), requiring WhatsApp API credentials.  
    - Inputs: Receives from AI Agent output.  
    - Outputs: None (terminal).  
    - Edge Cases: Messaging limits, API failures, or invalid phone numbers.  
    - Version: 1  

---

### 3. Summary Table

| Node Name            | Node Type                        | Functional Role                          | Input Node(s)     | Output Node(s)               | Sticky Note |
|----------------------|---------------------------------|----------------------------------------|-------------------|-----------------------------|-------------|
| Sticky Note          | Sticky Note                     | Visual annotation (empty content)      | None              | None                        |             |
| Schedule Trigger     | scheduleTrigger                 | Initiates workflow on schedule         | None              | AI Agent                    |             |
| AI Agent             | LangChain Agent                 | AI processing orchestrator              | Schedule Trigger, Google Calendar 1, Google Calendar 2, Google Gemini Chat Model | Send to Slack, Send to Discord, Send to WhatsApp |             |
| Google Gemini Chat Model | Google Gemini Chat Model       | AI language model for summarization    | None              | AI Agent                    |             |
| Google Calendar 1    | Google Calendar Tool            | Fetches calendar events (source 1)     | None              | AI Agent                    |             |
| Google Calendar 2    | Google Calendar Tool            | Fetches calendar events (source 2)     | None              | AI Agent                    |             |
| Send to Slack        | Slack Node                     | Sends summary message to Slack          | AI Agent          | None                        |             |
| Send to Discord      | Discord Node                   | Sends summary message to Discord        | AI Agent          | None                        |             |
| Send to WhatsApp     | WhatsApp Node                  | Sends summary message to WhatsApp       | AI Agent          | None                        |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node**  
   - Type: scheduleTrigger  
   - Configure it to run daily at a suitable time (e.g., 8:00 AM) to trigger the workflow automatically.

2. **Add two Google Calendar Tool nodes** (Google Calendar 1 and Google Calendar 2)  
   - Type: Google Calendar Tool (version 1.3)  
   - Configure each with valid Google OAuth2 credentials.  
   - Set up calendar IDs or filters to fetch events relevant for the daily summary (e.g., today’s events).  
   - No inputs; outputs connect as AI tools to AI Agent.

3. **Add Google Gemini Chat Model node**  
   - Type: Google Gemini Chat Model (version 1)  
   - Configure with Google Cloud credentials for access to the Gemini API.  
   - No inputs; output connects as AI language model to AI Agent.

4. **Add AI Agent node**  
   - Type: LangChain Agent (version 2.1)  
   - Configure to use the two Google Calendar nodes as AI tools. Connect their outputs to AI Agent’s ai_tool input.  
   - Connect Google Gemini Chat Model output to AI Agent’s ai_languageModel input.  
   - Connect Schedule Trigger output to AI Agent main input.  
   - Set AI Agent parameters as needed for summarization context (e.g., prompt templates, max tokens).

5. **Add messaging nodes to send summary**  
   - Slack Node (version 2.3)  
     - Configure with Slack credentials or webhook ID.  
     - Connect AI Agent main output to Slack node input.  
   - Discord Node (version 2)  
     - Configure with Discord webhook credentials.  
     - Connect AI Agent main output to Discord node input.  
   - WhatsApp Node (version 1)  
     - Configure with WhatsApp API credentials/webhook.  
     - Connect AI Agent main output to WhatsApp node input.

6. **Wire the nodes**  
   - Connect Schedule Trigger → AI Agent (main input).  
   - Connect Google Calendar 1 → AI Agent (ai_tool input).  
   - Connect Google Calendar 2 → AI Agent (ai_tool input).  
   - Connect Google Gemini Chat Model → AI Agent (ai_languageModel input).  
   - Connect AI Agent (main output) → Send to Slack.  
   - Connect AI Agent (main output) → Send to Discord.  
   - Connect AI Agent (main output) → Send to WhatsApp.

7. **Credentials and Permissions**  
   - Ensure Google OAuth2 credentials have permissions to read calendar events.  
   - Google Gemini API credentials with access to Gemini chat endpoint.  
   - Slack workspace app or webhook with permissions to post messages.  
   - Discord webhook URL with posting permissions.  
   - WhatsApp API credentials/webhooks authorized to send messages to intended recipients.

8. **Testing and Validation**  
   - Test each calendar node independently to confirm event retrieval.  
   - Test AI Agent with sample data to verify summarization.  
   - Test each messaging node to confirm message delivery.  
   - Confirm the entire workflow triggers on schedule and produces expected summaries.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                      |
|--------------------------------------------------------------------------------------------------------------------|------------------------------------|
| The workflow uses Google Gemini via LangChain integration for advanced AI summarization capabilities.              | Google Gemini API & LangChain docs |
| Webhook IDs in messaging nodes indicate preconfigured credentials; ensure these are kept secure and valid.         | n8n Credential Management          |
| For multi-calendar summaries, ensure calendars do not have overlapping or duplicate events to avoid redundancy.    | Google Calendar API documentation  |
| Consider adding error handling nodes or retries for API rate limits or failures to improve reliability.             | n8n documentation on error handling|

---

**Disclaimer:** The provided content is derived exclusively from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and publicly accessible.