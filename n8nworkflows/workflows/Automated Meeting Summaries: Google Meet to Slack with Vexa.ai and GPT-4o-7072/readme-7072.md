Automated Meeting Summaries: Google Meet to Slack with Vexa.ai and GPT-4o

https://n8nworkflows.xyz/workflows/automated-meeting-summaries--google-meet-to-slack-with-vexa-ai-and-gpt-4o-7072


# Automated Meeting Summaries: Google Meet to Slack with Vexa.ai and GPT-4o

### 1. Workflow Overview

This workflow automates the process of generating concise meeting summaries from Google Meet sessions and delivering them to a Slack channel. It integrates Google Calendar to detect meeting start events, uses Vexa.ai to join and transcribe the meeting, processes the transcript with an AI agent powered by GPT-4o, and posts the summary to Slack.

**Target Use Cases:**  
- Teams using Google Meet and Google Calendar who want automated, actionable meeting summaries delivered to Slack.  
- Organizations seeking to reduce manual note-taking and improve meeting follow-up efficiency.  
- Automations requiring integration between calendar events, AI transcription services, and messaging platforms.

**Logical Blocks:**

- **1.1 Input Reception:** Trigger on Google Calendar event start to detect meetings.  
- **1.2 Bot Injection:** Add a Vexa.ai bot to the Google Meet session to capture audio/transcript.  
- **1.3 Transcript Retrieval & Validation:** Poll Vexa.ai for the meeting transcript until it is ready.  
- **1.4 Transcript Processing:** Parse and format the transcript data, participants, and duration.  
- **1.5 AI Summarization:** Use an AI agent with GPT-4o to generate a concise meeting summary.  
- **1.6 Output Delivery:** Post the summary message to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Detects the start of a Google Calendar event (typically a Google Meet) to initiate the workflow.

**Nodes Involved:**  
- Google Calendar Trigger

**Node Details:**  
- **Google Calendar Trigger**  
  - Type: Trigger node for Google Calendar events  
  - Configuration:  
    - Trigger on event start (`eventStarted`)  
    - Poll frequency: every minute  
    - Calendar: primary calendar of the authenticated user  
  - Inputs: None (trigger node)  
  - Outputs: Emits event data including conference (Google Meet) details  
  - Requirements: Google Calendar OAuth2 credentials with read access  
  - Edge Cases:  
    - Events without Google Meet links (workflow will fail downstream)  
    - API quota limits or auth expiration  
  - Notes: Only triggers on calendar events with Google Meet conference data

#### 2.2 Bot Injection

**Overview:**  
Automatically adds a bot to the Google Meet session using Vexa.ai API, enabling recording and transcription.

**Nodes Involved:**  
- Add bot to meet

**Node Details:**  
- **Add bot to meet**  
  - Type: HTTP Request (POST) to Vexa.ai bots endpoint  
  - Configuration:  
    - URL: `https://gateway.dev.vexa.ai/bots`  
    - Body includes:  
      - Platform: `"google_meet"`  
      - Native meeting ID: dynamically extracted from Google Calendar event (`conferenceData.conferenceId`)  
      - Bot name: `"MyMeetingBot"` (customizable)  
    - Authentication: HTTP Header Auth with Vexa.ai API key (`X-API-Key`)  
  - Inputs: Google Calendar trigger JSON (to get meeting ID)  
  - Outputs: Vexa.ai bot creation response  
  - Edge Cases:  
    - Invalid or missing meeting ID leads to API errors  
    - Rate limits on Vexa.ai API  
    - Network or auth failures  
  - Notes: The bot joins the meeting ~30-60 seconds after start

#### 2.3 Transcript Retrieval & Validation

**Overview:**  
Waits and polls Vexa.ai API until the meeting transcript is fully generated, then validates completion.

**Nodes Involved:**  
- Wait  
- Get Vexa Transcript  
- If

**Node Details:**  
- **Wait**  
  - Type: Wait node  
  - Configuration: Default delay (unspecified, likely short wait before polling)  
  - Inputs: From Code node (processing initial transcript data)  
  - Outputs: Triggers Get Vexa Transcript node  
  - Edge Cases: Excessive wait without transcript availability may cause workflow delays or timeouts

- **Get Vexa Transcript**  
  - Type: HTTP Request (GET) to Vexa.ai transcripts endpoint  
  - Configuration:  
    - URL built dynamically: `https://gateway.dev.vexa.ai/transcripts/google_meet/{{ $json.native_meeting_id }}`  
    - Timeout: 30 seconds  
    - Authentication: HTTP Header Auth with Vexa.ai API key  
  - Inputs: From Wait and Add bot to meet nodes  
  - Outputs: Transcript JSON data including status and segments  
  - Edge Cases:  
    - Transcript not ready returns incomplete status  
    - API rate limits or auth issues  
    - Network timeouts

- **If**  
  - Type: Conditional node  
  - Configuration: Checks if `status` field in transcript JSON equals `"completed"`  
  - Inputs: From Get Vexa Transcript  
  - Outputs:  
    - True branch: proceed to Code node for transcript processing  
    - False branch: loop back to Wait node to retry  
  - Edge Cases:  
    - Status may never become "completed" if meeting failed or transcript unavailable (potential infinite loop)  
    - Need timeout or max retry management (not shown in current workflow)

#### 2.4 Transcript Processing

**Overview:**  
Formats the raw transcript data, extracts participants, calculates duration, and prepares data for AI summarization.

**Nodes Involved:**  
- Code

**Node Details:**  
- **Code**  
  - Type: Function code node (JavaScript)  
  - Configuration logic:  
    - Extracts `segments` array from transcript JSON  
    - If no segments, returns default empty transcript and zero duration  
    - Concatenates each segment into a readable transcript string with timestamps and speaker labels  
    - Collects unique participant names  
    - Calculates total meeting duration in seconds and minutes  
    - Passes meeting metadata (ID, platform, status) forward  
  - Inputs: From If node (transcript JSON confirmed completed)  
  - Outputs: JSON object with:  
    - `fullTranscript` (string)  
    - `participants` (comma-separated string)  
    - `durationSeconds`, `durationMinutes`  
    - Meeting metadata and segment count  
  - Edge Cases:  
    - Missing or malformed segments array  
    - Null or unknown speaker names  
    - Large transcripts may affect performance  
  - Notes: Prepares structured data for AI prompt context

#### 2.5 AI Summarization

**Overview:**  
Uses a GPT-4o-powered AI agent to generate a concise meeting summary formatted for Slack.

**Nodes Involved:**  
- AI Agent

**Node Details:**  
- **AI Agent**  
  - Type: Langchain AI Agent node (GPT-4o)  
  - Configuration:  
    - Prompt defines a structured summary format with sections: Meeting Summary, Duration, Participants, Key Points, Action Items, Outcome  
    - Input variables injected: full transcript, durationMinutes, participants  
    - Output is a short text summary (under 800 characters) focused on actionable items and decisions  
  - Inputs: From Code node (processed transcript data)  
  - Outputs: AI-generated summary text as JSON field `output`  
  - Edge Cases:  
    - API quota or auth failures with OpenAI  
    - Prompt formatting issues or unexpected model output  
  - Requirements: OpenAI API credentials with GPT-4o access and credits  
  - Notes: Uses custom-defined prompt for concise Slack-ready summaries

#### 2.6 Output Delivery

**Overview:**  
Posts the AI-generated meeting summary message to a designated Slack channel.

**Nodes Involved:**  
- Send a message

**Node Details:**  
- **Send a message**  
  - Type: Slack node (message sending)  
  - Configuration:  
    - Channel: `#general` (by name)  
    - Message text: output from AI Agent (`$json.output`)  
    - Other options: excludes workflow links in message  
  - Inputs: From AI Agent node  
  - Outputs: None (terminal node)  
  - Requirements: Slack API credentials with bot token having `chat:write` scope  
  - Edge Cases:  
    - Slack API auth failures or revoked tokens  
    - Channel not found or bot not invited  
    - Message length limits (unlikely due to prompt constraints)

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                     | Input Node(s)             | Output Node(s)         | Sticky Note                                                                                           |
|---------------------|----------------------------------|-----------------------------------|---------------------------|-----------------------|-----------------------------------------------------------------------------------------------------|
| Google Calendar Trigger | Google Calendar Trigger          | Detect meeting start event        | (Trigger)                 | Add bot to meet        | ## 🗓️ Google Calendar Setup: Steps to enable API, create OAuth2, trigger on eventStarted            |
| Add bot to meet      | HTTP Request (POST)               | Add Vexa.ai bot to Google Meet    | Google Calendar Trigger   | Get Vexa Transcript    | ## 🤖 Vexa.ai Setup: API key generation, bot config, rate limits                                    |
| Get Vexa Transcript  | HTTP Request (GET)                | Retrieve meeting transcript       | Add bot to meet, Wait     | If                     |                                                                                                     |
| If                   | If Condition                     | Check if transcript is complete   | Get Vexa Transcript       | Code (true), Wait (false) |                                                                                                     |
| Wait                 | Wait                            | Pause before retrying transcript  | If (false branch)         | Get Vexa Transcript    |                                                                                                     |
| Code                 | Function Code Node               | Parse and format transcript data  | If (true branch)          | AI Agent               |                                                                                                     |
| AI Agent             | Langchain AI Agent (GPT-4o)      | Generate meeting summary          | Code                      | Send a message         | ## 🧠 OpenAI + Slack Setup: OpenAI model config, Slack app & bot setup                              |
| Send a message       | Slack Node                      | Post summary message to Slack     | AI Agent                  | (End)                  |                                                                                                     |
| Sticky Note          | Sticky Note                     | Document prerequisites & costs    | -                         | -                      | ## 📋 Prerequisites & Requirements: Accounts, permissions, estimated cost                           |
| Sticky Note1         | Sticky Note                     | Document Google Calendar setup    | -                         | -                      | ## 🗓️ Google Calendar Setup: Enable API, OAuth2, test connection                                   |
| Sticky Note2         | Sticky Note                     | Document Vexa.ai setup             | -                         | -                      | ## 🤖 Vexa.ai Setup: API key, bot config, rate limits                                              |
| Sticky Note3         | Sticky Note                     | Document OpenAI and Slack setup   | -                         | -                      | ## 🧠 OpenAI + Slack Setup: Model, Slack bot scopes, channel config                                |
| Sticky Note4         | Sticky Note                     | Credential setup instructions     | -                         | -                      | ## 🔑 Credential Setup Guide: Steps for Google Calendar, Vexa.ai, OpenAI, Slack credentials       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Calendar Trigger node**  
   - Type: Google Calendar Trigger  
   - Credentials: Google Calendar OAuth2 with read access  
   - Settings: Trigger on event start (`eventStarted`), poll every minute, calendar ID: `primary`  

2. **Add HTTP Request node named "Add bot to meet"**  
   - Type: HTTP Request (POST)  
   - URL: `https://gateway.dev.vexa.ai/bots`  
   - Authentication: HTTP Header Auth credential with header `X-API-Key` (Vexa.ai API key)  
   - Body parameters (JSON):  
     - `platform`: `"google_meet"`  
     - `native_meeting_id`: Expression: `{{ $json.conferenceData.conferenceId }}`  
     - `bot_name`: `"MyMeetingBot"` (customizable)  
   - Connect Google Calendar Trigger output to this node

3. **Add HTTP Request node named "Get Vexa Transcript"**  
   - Type: HTTP Request (GET)  
   - URL: Expression: `https://gateway.dev.vexa.ai/transcripts/google_meet/{{ $json.native_meeting_id }}`  
   - Authentication: Same HTTP Header Auth credential as above  
   - Timeout: 30000 ms (30 seconds)  
   - Connect "Add bot to meet" output to this node  
   - This node will also be connected from the Wait node for polling

4. **Add If node named "If"**  
   - Type: If condition  
   - Condition: Check if `{{$json.status}}` equals `"completed"`  
   - Connect "Get Vexa Transcript" output to this node

5. **Add Wait node named "Wait"**  
   - Type: Wait node (default delay, e.g., 30 seconds recommended)  
   - Connect "If" node – false branch to this node  
   - Connect "Wait" node output back to "Get Vexa Transcript" to create polling loop

6. **Add Code node named "Code"**  
   - Type: Function Code node  
   - Paste the JavaScript code to parse transcript segments, create full transcript string, extract participants, calculate duration  
   - Connect "If" node – true branch to this node

7. **Add AI Agent node named "AI Agent"**  
   - Type: Langchain AI Agent (GPT-4o)  
   - Configure prompt as per meeting summary format: concise, structured, under 800 characters  
   - Use input variables: `fullTranscript`, `durationMinutes`, `participants` from "Code" node output  
   - Connect "Code" node output to this node  
   - Credentials: OpenAI API with access to GPT-4o model

8. **Add Slack node named "Send a message"**  
   - Type: Slack node (Send message)  
   - Credentials: Slack API with bot token, scopes `chat:write`  
   - Channel: `#general` (or your preferred channel)  
   - Message text: Expression: `{{ $json.output }}` (from AI Agent output)  
   - Connect "AI Agent" node output to this node

9. **Credential Setup:**  
   - Google Calendar OAuth2 with Calendar API enabled  
   - Vexa.ai API key as HTTP Header Auth with header `X-API-Key`  
   - OpenAI API key with GPT-4o access  
   - Slack Bot Token with `chat:write` scope, bot added to target channel

10. **Additional Configuration:**  
    - Optional: Add Wait node delay time adjustment as needed  
    - Test each credential connection independently before full workflow run  
    - Add error handling or max retry logic if required to prevent infinite loops on transcript polling

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                               |
|------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow requires Google Calendar events to have Google Meet conference links for bot injection.     | See Sticky Note1 for Google Calendar Setup instructions                                                      |
| Vexa.ai free tier limits to 100 requests/hour; bot joins meeting ~30-60 seconds after start.          | See Sticky Note2 for Vexa.ai API key and bot configuration                                                   |
| Recommended OpenAI model is `chatgpt-4o-latest` for best summarization quality at reasonable cost.   | See Sticky Note3 for OpenAI and Slack setup details                                                          |
| Slack bot requires `chat:write` scope and must be invited to the target channel for message posting. | See Sticky Note3 for Slack bot setup steps                                                                   |
| Estimated cost per meeting summary is approximately $0.01 to $0.05 depending on OpenAI usage.         | See Sticky Note for Prerequisites & Requirements                                                             |
| Ensure all API credentials are tested and valid before running the workflow to avoid runtime errors.  | See Sticky Note4 for detailed credentials setup guide                                                        |
| This workflow implements a polling loop for transcript readiness; consider adding timeout or alerts. | To avoid infinite loops if transcript never becomes `completed`                                              |
| Prompt formatting focuses on actionable items and concise summaries for Slack readability.            | Custom prompt defined inside AI Agent node                                                                   |
| Vexa.ai API is used for transcription of Google Meet meetings; requires meeting ID extracted from calendar event. | Integration critical for transcript retrieval                                                                |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.