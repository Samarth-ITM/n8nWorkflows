Daily meetings summarization with Gemini AI

https://n8nworkflows.xyz/workflows/daily-meetings-summarization-with-gemini-ai-2789


# Daily meetings summarization with Gemini AI

### 1. Workflow Overview

This workflow automates the daily summarization of meetings using Google’s Gemini AI chat model and posts the summary to a Slack channel at a scheduled time (default 9 AM). It is designed for users who want an automated digest of their calendar events without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a specified hour.
- **1.2 AI Agent Processing:** Uses a LangChain AI Agent configured to interact with Google Calendar data and generate a summary prompt.
- **1.3 Google Calendar Data Retrieval:** Fetches all calendar events for the current day using Google Calendar API.
- **1.4 Google Gemini Chat Model:** Processes the AI Agent’s prompt and calendar data to generate a natural language summary.
- **1.5 Slack Notification:** Sends the generated summary text to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow once daily at a configured hour (default 9 AM). It acts as the entry point.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger daily at 9:00 AM (triggerAtHour: 9)  
    - Input: None (trigger node)  
    - Output: Triggers the "Calendar AI Agent" node  
    - Edge Cases:  
      - Timezone misconfiguration could cause unexpected trigger times.  
      - If the n8n instance is down at trigger time, the workflow will not run.  
    - Version: 1.2

---

#### 2.2 AI Agent Processing

- **Overview:**  
  This block uses a LangChain AI Agent node to create a prompt that summarizes today’s meetings. It sets the context and parameters for the AI model and orchestrates the use of tools (Google Calendar API) to retrieve event data.

- **Nodes Involved:**  
  - Calendar AI Agent

- **Node Details:**  
  - **Calendar AI Agent**  
    - Type: LangChain Agent Node (@n8n/n8n-nodes-langchain.agent)  
    - Configuration:  
      - Text prompt includes instructions to summarize today’s meetings with dynamic date range variables (`startdate` and `enddate`) set to the current day’s start and end times.  
      - System message defines the assistant’s role as a Google Calendar assistant, instructing it to use the Event Retrieval tool and format date ranges accordingly.  
      - No memory is assigned since the task is one-off without follow-up context.  
      - Prompt type: "define" (custom prompt with system message and user text)  
    - Inputs: Receives trigger from Schedule Trigger  
    - Outputs: Sends prompt to Google Gemini Chat Model (ai_languageModel), and receives calendar events from Google Calendar node (ai_tool)  
    - Key Expressions:  
      - `{{ $now.format('yyyy-MM-dd 00:00:00') }}` and `{{ $now.format('yyyy-MM-dd 23:59:59') }}` for date range  
      - System message uses Luxon DateTime formatting for current date  
    - Edge Cases:  
      - Expression evaluation errors if date formatting fails  
      - AI Agent may fail if Google Calendar API returns no data or errors  
      - If the AI model is unreachable or returns errors, the workflow will fail here  
    - Version: 1.7

---

#### 2.3 Google Calendar Data Retrieval

- **Overview:**  
  This block fetches all calendar events for the current day from the user’s Google Calendar account, based on the date range provided by the AI Agent.

- **Nodes Involved:**  
  - Google Calendar - Get Events

- **Node Details:**  
  - **Google Calendar - Get Events**  
    - Type: Google Calendar Tool Node (n8n-nodes-base.googleCalendarTool)  
    - Configuration:  
      - Operation: getAll (fetch all events)  
      - TimeMin and TimeMax parameters dynamically set using AI Agent’s date range variables (`start_date` and `end_date`)  
      - Calendar account selected (john@iKemo.io)  
      - Description set to "Use this tool when you’re asked to retrieve events data."  
    - Inputs: Receives date range from AI Agent (ai_tool input)  
    - Outputs: Sends event data back to AI Agent (ai_tool output)  
    - Credentials: Google Calendar OAuth2 configured for user john@iKemo.io  
    - Edge Cases:  
      - OAuth token expiration or invalid credentials causing auth errors  
      - No events found for the day (empty response)  
      - API rate limits or quota exceeded errors  
    - Version: 1.2

---

#### 2.4 Google Gemini Chat Model

- **Overview:**  
  This block uses Google’s Gemini AI chat model (gemini-flash) to generate a natural language summary of the meetings based on the prompt and calendar data provided by the AI Agent.

- **Nodes Involved:**  
  - Google Gemini Chat Model

- **Node Details:**  
  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model Node (@n8n/n8n-nodes-langchain.lmChatGoogleGemini)  
    - Configuration:  
      - Model name: "models/gemini-1.5-flash-latest" (Gemini Flash model)  
      - No additional options configured  
    - Inputs: Receives prompt from AI Agent (ai_languageModel input)  
    - Outputs: Returns AI-generated summary to AI Agent  
    - Credentials: Google Palm API key configured  
    - Edge Cases:  
      - API key invalid or expired  
      - Model service downtime or latency causing timeouts  
      - Unexpected model output format or errors  
    - Version: 1

---

#### 2.5 Slack Notification

- **Overview:**  
  This block sends the AI-generated meeting summary to a specified Slack channel, formatting the message to remove markdown and prefixing it with "Gemini :".

- **Nodes Involved:**  
  - Send response back to slack channel

- **Node Details:**  
  - **Send response back to slack channel**  
    - Type: Slack Node (n8n-nodes-base.slack)  
    - Configuration:  
      - Text message: `"Gemini : {{ $json.output.removeMarkdown() }}"` — removes markdown formatting from AI output  
      - Channel: Selected by channel ID (C07QMTJHR0A) from a cached Slack channel list  
      - Markdown enabled (mrkdwn: true)  
      - No link to workflow included  
    - Inputs: Receives AI summary text from AI Agent  
    - Outputs: None (final node)  
    - Credentials: Slack API OAuth2 configured  
    - Edge Cases:  
      - Slack API rate limits or invalid token errors  
      - Channel ID invalid or user not authorized to post  
      - Message formatting issues if AI output is empty or malformed  
    - Version: 2.1

---

### 3. Summary Table

| Node Name                     | Node Type                           | Functional Role                      | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                   |
|-------------------------------|-----------------------------------|------------------------------------|------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger                  | Initiates workflow daily at 9 AM   | None                   | Calendar AI Agent             | ## Trigger the task daily, receive the meetings data, process the data and return response for sending. No memory assigned to the model since the model is running one task and doesn't need a followup, then send the data to the user. |
| Calendar AI Agent             | LangChain Agent                   | Creates prompt and orchestrates AI | Schedule Trigger        | Google Gemini Chat Model, Google Calendar - Get Events, Send response back to slack channel |                                                                                              |
| Google Gemini Chat Model      | LangChain Google Gemini Chat Model | Generates meeting summary          | Calendar AI Agent       | Calendar AI Agent             | ### Gemini Flash model a base                                                                |
| Google Calendar - Get Events  | Google Calendar Tool              | Retrieves calendar events          | Calendar AI Agent       | Calendar AI Agent             | ### Access Google Calendar and fetch all the data                                           |
| Send response back to slack channel | Slack Node                      | Sends summary to Slack channel     | Calendar AI Agent       | None                         | ### Send the response from AI back to slack channel                                         |
| Sticky Note1                 | Sticky Note                      | Workflow overview note             | None                   | None                         | ## Trigger the task daily, receive the meetings data, process the data and return response for sending. No memory assigned to the model since the model is running one task and doesn't need a followup, then send the data to the user. |
| Sticky Note2                 | Sticky Note                      | Note on Google Calendar node      | None                   | None                         | ### Access Google Calendar and fetch all the data                                           |
| Sticky Note3                 | Sticky Note                      | Note on Gemini model               | None                   | None                         | ### Gemini Flash model a base                                                                |
| Sticky Note4                 | Sticky Note                      | Note on Slack node                 | None                   | None                         | ### Send the response from AI back to slack channel                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set the trigger to run daily at 9:00 AM (or your preferred time).  
   - No credentials needed.

3. **Add a LangChain Agent node (Calendar AI Agent):**  
   - Connect the Schedule Trigger node’s output to this node’s input.  
   - Configure the prompt text as:  
     ```
     summarize today's meetings.
     startdate = {{ $now.format('yyyy-MM-dd 00:00:00') }}
     enddate = {{ $now.format('yyyy-MM-dd 23:59:59') }}
     ```  
   - Set the system message to:  
     ```
     You are a Google Calendar assistant.
     Your primary goal is to assist the user in managing their calendar effectively using Event Retrieval tool. 
     Always base your responses on the current date: 
     {{ DateTime.local().toFormat('cccc d LLLL yyyy') }}.
     General Guidelines:
     Always mention all meetings attendees
     Tool: Event Retrieval
     Format the date range:
     start_date: Start date and time in YYYY-MM-DD HH:mm:ss.
     end_date: End date and time in YYYY-MM-DD HH:mm:ss.
     ```  
   - Set Prompt Type to "define".  
   - No memory configured.

4. **Add a Google Gemini Chat Model node:**  
   - Connect the LangChain Agent node’s `ai_languageModel` output to this node’s input.  
   - Set the model name to `models/gemini-1.5-flash-latest`.  
   - Configure credentials with your Google Palm API key.

5. **Add a Google Calendar - Get Events node:**  
   - Connect the LangChain Agent node’s `ai_tool` output to this node’s input.  
   - Set operation to `getAll`.  
   - Set `timeMin` to `={{ $fromAI('start_date') }}` and `timeMax` to `={{ $fromAI('end_date') }}` to dynamically receive date range from AI Agent.  
   - Select the calendar account (e.g., john@iKemo.io).  
   - Configure Google Calendar OAuth2 credentials.

6. **Connect the Google Gemini Chat Model node’s output back to the LangChain Agent node’s `ai_languageModel` input** (this is already done in step 4).

7. **Add a Slack node (Send response back to slack channel):**  
   - Connect the LangChain Agent node’s main output to this Slack node.  
   - Set the message text to:  
     ```
     Gemini : {{ $json.output.removeMarkdown() }}
     ```  
   - Select the Slack channel by ID (e.g., C07QMTJHR0A).  
   - Enable Markdown formatting.  
   - Configure Slack API credentials with OAuth2.

8. **Verify all connections:**  
   - Schedule Trigger → Calendar AI Agent  
   - Calendar AI Agent → Google Gemini Chat Model (ai_languageModel)  
   - Calendar AI Agent → Google Calendar - Get Events (ai_tool)  
   - Google Gemini Chat Model → Calendar AI Agent (ai_languageModel)  
   - Calendar AI Agent → Slack node (main)

9. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow uses Google Gemini AI (Gemini Flash model) for natural language summarization.              | Sticky Note3                                                                                     |
| Slack message formatting removes markdown from AI output to ensure clean display.                    | Sticky Note4                                                                                     |
| The workflow requires Google Cloud project with Vertex AI API enabled and Google AI API key setup.  | Setup Steps in workflow description                                                             |
| Credentials needed: Google Palm API key, Google Calendar OAuth2, Slack OAuth2.                       | Setup Steps in workflow description                                                             |
| For more info on Google Gemini and Vertex AI, visit: https://cloud.google.com/vertex-ai             | External resource for API and model details                                                     |
| The workflow does not use memory in the AI Agent node, as it is designed for one-shot daily summaries. | Sticky Note1                                                                                     |

---

This document provides a complete and detailed reference for understanding, reproducing, and maintaining the "Daily meetings summarization with Gemini AI" workflow in n8n.