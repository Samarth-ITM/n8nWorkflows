Google Calendar Reminder System with GPT-4o and Telegram

https://n8nworkflows.xyz/workflows/google-calendar-reminder-system-with-gpt-4o-and-telegram-3393


# Google Calendar Reminder System with GPT-4o and Telegram

### 1. Workflow Overview

This workflow automates sending personalized, AI-enhanced reminders for upcoming Google Calendar events via Telegram exactly **1 hour before the event starts**. It is designed to prevent missed meetings or appointments by consolidating notifications into a friendly, conversational message delivered through a widely accessible messaging platform.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Periodically initiates the workflow every minute to check for imminent events.
- **1.2 Google Calendar Query**: Retrieves events starting within a specific 1-hour window.
- **1.3 Duplicate Filtering**: Ensures reminders are sent only once per event to avoid spamming.
- **1.4 AI-Powered Message Generation**: Uses GPT-4o to convert raw event data into a warm, professional reminder message.
- **1.5 Telegram Notification Delivery**: Sends the crafted reminder message to the user’s Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block initiates the workflow execution every minute, ensuring timely checks for upcoming events.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `n8n-nodes-base.scheduleTrigger`  
    - Role: Starts the workflow on a fixed time interval.  
    - Configuration: Runs every 1 minute (`minutesInterval: 1`).  
    - Input: None (trigger node).  
    - Output: Triggers "Get upcoming event" node.  
    - Version: 1.2  
    - Edge Cases:  
      - Disabled by default (must be enabled to activate workflow).  
      - If the system clock is off or n8n is paused, triggers may be delayed or missed.

---

#### 1.2 Google Calendar Query

- **Overview:**  
  Queries Google Calendar for events starting between exactly 1 hour and 1 hour + 1 minute from the current time, limiting results to 5 events.

- **Nodes Involved:**  
  - Get upcoming event

- **Node Details:**  
  - **Get upcoming event**  
    - Type: `n8n-nodes-base.googleCalendar`  
    - Role: Fetches calendar events within a defined time window.  
    - Configuration:  
      - Operation: `getAll` events.  
      - Time window:  
        - `timeMin` = current time + 1 hour  
        - `timeMax` = current time + 1 hour + 1 minute  
      - Limit: 5 events max per execution.  
      - Calendar: User’s Google Calendar email (`davide.boizza@gmail.com`).  
    - Input: Triggered by Schedule Trigger.  
    - Output: Passes event data to "Already sent?" node.  
    - Credentials: Google OAuth2 account linked.  
    - Version: 1.3  
    - Edge Cases:  
      - API rate limits or auth token expiration may cause failures.  
      - If no events exist in the time window, output is empty.  
      - Timezone mismatches could cause incorrect event retrieval.

---

#### 1.3 Duplicate Filtering

- **Overview:**  
  Prevents sending multiple reminders for the same event by filtering out events already processed in previous workflow executions.

- **Nodes Involved:**  
  - Already sent?

- **Node Details:**  
  - **Already sent?**  
    - Type: `n8n-nodes-base.removeDuplicates`  
    - Role: Filters out events whose IDs have been seen in prior runs.  
    - Configuration:  
      - Operation: `removeItemsSeenInPreviousExecutions`.  
      - Deduplication key: Event ID (`{{$json.id}}`).  
    - Input: Receives event list from "Get upcoming event".  
    - Output: Passes unique events to "Secretary Agent".  
    - Version: 2  
    - Edge Cases:  
      - If event IDs are missing or malformed, deduplication may fail.  
      - Persistent storage required to track seen IDs; data loss could cause duplicate reminders.

---

#### 1.4 AI-Powered Message Generation

- **Overview:**  
  Uses GPT-4o to generate a friendly, professional reminder message based on event details, simulating a virtual secretary’s tone.

- **Nodes Involved:**  
  - Secretary Agent  
  - OpenAI Chat Model (used as language model backend)

- **Node Details:**  
  - **Secretary Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Generates conversational reminder text from event data.  
    - Configuration:  
      - Input text template includes event name, description, location, start/end times, and creator email, dynamically injected from "Get upcoming event".  
      - System message defines the AI’s persona as a warm, professional virtual secretary with a conversational tone, occasional light humor, and clear communication style.  
      - Output parser enabled to extract the generated message.  
    - Input: Receives unique events from "Already sent?" and language model from "OpenAI Chat Model".  
    - Output: Passes generated reminder text to "Send reminder".  
    - Version: 1.8  
    - Edge Cases:  
      - AI API rate limits or errors may cause failures.  
      - If event data fields are missing or empty, message quality may degrade.  
      - Unexpected AI output format could cause parsing errors.  
    - Sub-workflow: Uses "OpenAI Chat Model" node as language model backend.

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides GPT-4o language model capabilities to the "Secretary Agent".  
    - Configuration:  
      - Model: `gpt-4o-mini` (GPT-4o variant).  
      - No additional options set.  
    - Input: Connected as AI language model for "Secretary Agent".  
    - Output: Provides AI-generated completions.  
    - Credentials: OpenAI API key configured.  
    - Version: 1.2  
    - Edge Cases:  
      - API key invalid or quota exceeded causes errors.  
      - Model unavailability or network issues may interrupt processing.

---

#### 1.5 Telegram Notification Delivery

- **Overview:**  
  Sends the AI-generated reminder message to the user’s Telegram chat, ensuring accessible and timely notification.

- **Nodes Involved:**  
  - Send reminder

- **Node Details:**  
  - **Send reminder**  
    - Type: `n8n-nodes-base.telegram`  
    - Role: Sends text messages via Telegram Bot API.  
    - Configuration:  
      - Message text: Uses the output from "Secretary Agent" (`{{$json.output}}`).  
      - Chat ID: Placeholder `CHAT_ID` to be replaced with user’s Telegram chat identifier.  
      - Additional fields: Attribution disabled to keep message clean.  
    - Input: Receives reminder text from "Secretary Agent".  
    - Output: None (end node).  
    - Credentials: Telegram Bot API credentials configured.  
    - Version: 1.2  
    - Edge Cases:  
      - Invalid or missing `CHAT_ID` prevents message delivery.  
      - Telegram API rate limits or connectivity issues may cause failures.  
      - Bot permissions must allow sending messages to the specified chat.

---

### 3. Summary Table

| Node Name          | Node Type                              | Functional Role                    | Input Node(s)       | Output Node(s)    | Sticky Note                                                                                          |
|--------------------|--------------------------------------|----------------------------------|---------------------|-------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger    | n8n-nodes-base.scheduleTrigger       | Initiates workflow every minute  | None                | Get upcoming event | Disabled by default; triggers workflow every 1 minute                                              |
| Get upcoming event  | n8n-nodes-base.googleCalendar        | Fetches events starting in 1 hour| Schedule Trigger    | Already sent?      | Step 1: Set notification time before event; currently 1 hour                                       |
| Already sent?       | n8n-nodes-base.removeDuplicates      | Filters out duplicate events     | Get upcoming event   | Secretary Agent   | Prevent multiple reminders for the same event                                                      |
| Secretary Agent     | @n8n/n8n-nodes-langchain.agent       | Generates AI reminder message    | Already sent?        | Send reminder     | Uses GPT-4o to create warm, conversational reminders                                               |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi| Provides GPT-4o language model   | None (AI model node) | Secretary Agent   | OpenAI GPT-4o-mini model configured                                                                |
| Send reminder      | n8n-nodes-base.telegram               | Sends reminder via Telegram      | Secretary Agent      | None              | Replace `CHAT_ID` with your Telegram chat ID                                                      |
| Sticky Note        | n8n-nodes-base.stickyNote             | Documentation note               | None                | None              | Google Calendar Event Reminder overview                                                            |
| Sticky Note1       | n8n-nodes-base.stickyNote             | Documentation note               | None                | None              | Step 1 instructions for "Get upcoming event" and Telegram node                                    |
| Sticky Note2       | n8n-nodes-base.stickyNote             | Documentation note               | None                | None              | Prevent multiple reminders for the same event                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: `Schedule Trigger`  
   - Parameters: Set interval to every 1 minute (`minutesInterval: 1`).  
   - Position: Start of workflow.  
   - Enable node to activate workflow trigger.

2. **Create Google Calendar node**  
   - Type: `Google Calendar`  
   - Operation: `getAll`  
   - Parameters:  
     - `timeMin`: Expression `{{$now.plus({ hour: 1 })}}`  
     - `timeMax`: Expression `{{$now.plus({ hour: 1, minute: 1 })}}`  
     - Limit: 5  
     - Calendar: Select your Google Calendar email.  
   - Credentials: Connect your Google OAuth2 account.  
   - Connect Schedule Trigger output to this node input.

3. **Create Remove Duplicates node**  
   - Type: `Remove Duplicates`  
   - Operation: `removeItemsSeenInPreviousExecutions`  
   - Deduplication key: Expression `{{$json.id}}` (event ID)  
   - Connect Google Calendar node output to this node input.

4. **Create OpenAI Chat Model node**  
   - Type: `Langchain OpenAI Chat Model`  
   - Parameters:  
     - Model: `gpt-4o-mini`  
   - Credentials: Configure OpenAI API key.  
   - No input connections (used as AI model backend).

5. **Create Secretary Agent node**  
   - Type: `Langchain Agent`  
   - Parameters:  
     - Text template:  
       ```
       These are the details of the event/appointment:

       Event Name: {{ $('Get upcoming event').item.json.summary }}
       Description: {{ $('Get upcoming event').item.json.description }}
       Location: {{ $('Get upcoming event').item.json.location }}
       Start: {{ $('Get upcoming event').item.json.start.dateTime }}
       End: {{ $('Get upcoming event').item.json.end.dateTime }}
       Created by: {{ $('Get upcoming event').item.json.creator.email }}
       ```
     - System message:  
       ```
       ## Core Identity
       You are a professional and friendly virtual secretary, dedicated to reminder appointments with efficiency and a warm personal touch.

       ## Communication Style
       - Communicate in a conversational, approachable manner
       - Maintain a balance between professional competence and friendly rapport
       - Use a tone that is informal yet precise
       - Inject occasional light humor and personality into interactions

       ## Key Responsibilities
       1. Calendar Management
          - Provide timely reminders and scheduling updates

       2. Communication Approach
          - Respond promptly and clearly
          - Maintain confidentiality and discretion

       ## Interaction Guidelines
       - Use a friendly, conversational tone
       - Just describe the details of the event without asking questions

       ## Tone and Language
       - Warm and approachable
       - Professional but not overly formal
       - Direct and clear in communication
       - Use simple, straightforward language
       - Show genuine care and attentiveness

       Remember: Your primary goal is to make the user's life easier, more organized, and less stressful through efficient and friendly administrative support.
       ```
     - Enable output parser.  
   - Connect Remove Duplicates node output to this node input.  
   - Connect OpenAI Chat Model node as AI language model backend.

6. **Create Telegram node**  
   - Type: `Telegram`  
   - Parameters:  
     - Text: Expression `{{$json.output}}` (output from Secretary Agent)  
     - Chat ID: Replace placeholder `CHAT_ID` with your Telegram chat identifier.  
     - Additional fields: Disable attribution.  
   - Credentials: Configure Telegram Bot API credentials.  
   - Connect Secretary Agent output to this node input.

7. **Set Workflow Settings**  
   - Timezone: Set to your local timezone (e.g., `Europe/Rome`).  
   - Activate the workflow.  
   - Ensure all credentials are valid and nodes enabled.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow sends a friendly reminder exactly 1 hour before your event starts, via Telegram, powered by AI. | Overview sticky note in the workflow.                                                          |
| To customize notification time, adjust the "Get upcoming event" node's `timeMin` and `timeMax` expressions.   | Sticky Note1 in the workflow.                                                                  |
| Replace `CHAT_ID` in the Telegram node with your personal Telegram Bot chat ID to receive messages.           | Sticky Note1 in the workflow.                                                                  |
| Prevents duplicate reminders by filtering events already notified in previous executions.                      | Sticky Note2 in the workflow.                                                                  |
| For help customizing or consulting, contact via email: info@n3w.it or LinkedIn: https://www.linkedin.com/in/davideboizza/ | Contact information provided in the workflow description.                                      |

---

This documentation provides a complete, structured understanding of the "Google Calendar Event Reminder" workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.