Automated Meeting Prep with GPT-5 & Gemini Research from Calendar to Slack via Attio CRM

https://n8nworkflows.xyz/workflows/automated-meeting-prep-with-gpt-5---gemini-research-from-calendar-to-slack-via-attio-crm-8199


# Automated Meeting Prep with GPT-5 & Gemini Research from Calendar to Slack via Attio CRM

---

# Automated Meeting Prep with GPT-5 & Gemini Research from Calendar to Slack via Attio CRM

---

### 1. Workflow Overview

This workflow automates meeting preparation by leveraging calendar data, CRM records, email history, external research, and AI summarization to generate concise meeting briefs and deliver them to Slack each weekday morning.

**Purpose:**  
To provide users with AI-generated, context-rich briefing documents for their external meetings every day, helping them prepare efficiently by aggregating relevant information about attendees and meeting objectives.

**Target Use Cases:**  
- Professionals with frequent external meetings who need fast, comprehensive prep notes.  
- Sales, customer success, or partnership teams integrating CRM, email, and calendar data.  
- Teams wanting automated daily summaries delivered conveniently via Slack.

**Logical Blocks:**

- **1.1 Trigger & Calendar Fetch:** Runs daily on weekdays, fetches today's calendar events for the user, filters for external meetings.  
- **1.2 Meeting Presence Check:** Determines if there are external meetings; sends a no-meetings Slack message if none.  
- **1.3 AI Meeting Research Agent:** For each meeting, uses AI to research attendees via CRM, Gmail, calendar history, and external tools (Perplexity). Creates a structured meeting brief.  
- **1.4 CRM Integration:** Searches Attio CRM for attendees, creates person records if missing, retrieves and updates notes.  
- **1.5 Meeting Brief Formatting:** Formats AI-generated briefs into markdown for CRM notes and Slack.  
- **1.6 Meeting Brief Optimization & Slack Formatting:** Optimizes combined meeting briefs via AI, parses structured output, formats to Slack Block Kit, then posts to Slack channel.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger & Calendar Fetch

**Overview:**  
Triggers workflow every weekday at 6 AM, fetches all events from the user's Google Calendar for the day, then filters events with external attendees.

**Nodes Involved:**  
- Daily Weekday Trigger  
- Fetch Today's Calendar Events  
- Filter External Meetings Only

**Node Details:**

- **Daily Weekday Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow at 6:00 AM, Monday to Friday (cron: `0 6 * * 1-5`)  
  - Inputs: None  
  - Outputs: To Fetch Today's Calendar Events  
  - Edge Cases: Missed runs if n8n is down; timezone must be carefully configured.

- **Fetch Today's Calendar Events**  
  - Type: Google Calendar  
  - Role: Retrieves all calendar events between start and end of the current day  
  - Config: Uses OAuth2 credentials; calendar email must be updated to user’s email  
  - Inputs: Trigger output  
  - Outputs: List of events (including internal and external)  
  - Edge Cases: API rate limits, auth token expiry.

- **Filter External Meetings Only**  
  - Type: Filter  
  - Role: Keeps only events with attendees (i.e., external meetings)  
  - Logic: Checks if `attendees` array exists and is non-empty  
  - Inputs: Calendar events  
  - Outputs: Filtered events with external attendees  
  - Edge Cases: Events without attendees or malformed data.

---

#### 1.2 Meeting Presence Check

**Overview:**  
Checks if any external meetings are scheduled and routes workflow accordingly.

**Nodes Involved:**  
- Check for External Meetings  
- Send No Meetings Message

**Node Details:**

- **Check for External Meetings**  
  - Type: If  
  - Role: Checks if any filtered event has an `id` (indicating existence of meetings)  
  - Inputs: Filter External Meetings Only output  
  - Outputs:  
    - True: Proceed to split individual meetings  
    - False: Send no meetings message  
  - Edge Cases: Empty event arrays, partial data.

- **Send No Meetings Message**  
  - Type: Slack  
  - Role: Posts a friendly message to Slack channel notifying no external meetings today  
  - Config: OAuth2 Slack credentials; channel ID must be set  
  - Inputs: From false branch of check node  
  - Outputs: None  
  - Edge Cases: Slack API failures, invalid channel ID.

---

#### 1.3 AI Meeting Research Agent

**Overview:**  
Splits each external meeting, then uses an AI agent to research attendees and meeting context by querying CRM, Gmail, calendar history, and external research tools. Produces a structured meeting brief.

**Nodes Involved:**  
- Split Individual Meetings  
- AI Meeting Research Agent  
- Search Calendar History Tool  
- Search Gmail History Tool  
- Perplexity Research Tool  
- Attio Search People Tool  
- Attio List Notes Tool  
- Attio Read Notes Tool  
- Primary AI Model  
- Fallback AI Model  
- Meeting Data Parser  
- Format Meeting Brief  
- Filter External Attendees  
- Split Individual Attendees

**Node Details:**

- **Split Individual Meetings**  
  - Type: SplitOut  
  - Role: Splits multi-item calendar events array into individual meeting items  
  - Inputs: From check node true branch  
  - Outputs: Single meeting objects for AI processing  
  - Edge Cases: Empty input array.

- **AI Meeting Research Agent**  
  - Type: Langchain Agent (AI)  
  - Role: Central AI node orchestrating research on meeting attendees  
  - Configuration:  
    - System prompt: Expert meeting prep assistant with access to CRM, email, calendar, external tools  
    - Input prompt includes meeting summary, times, attendees (excluding user email)  
    - Tasks include CRM search, email/calendar history review, Perplexity research if needed, summarization  
  - Inputs: Single meeting JSON  
  - Outputs: Structured meeting prep data (via Meeting Data Parser)  
  - Edge Cases: AI model timeouts, API quota limits, fallback model usage.

- **Search Calendar History Tool**  
  - Type: Google Calendar Tool  
  - Role: Searches past calendar events for context on attendees  
  - Inputs: AI Agent dynamically passes query, date range  
  - Outputs: Event data for AI consumption  
  - Edge Cases: API limits, query errors.

- **Search Gmail History Tool**  
  - Type: Gmail Tool  
  - Role: Retrieves emails matching AI’s queries for attendee correspondence  
  - Inputs: Dynamic query from AI Agent  
  - Outputs: Email snippets  
  - Edge Cases: Gmail API quotas, auth issues.

- **Perplexity Research Tool**  
  - Type: Perplexity API  
  - Role: Provides external research on company/person if no prior interaction is found  
  - Inputs: AI-driven prompt  
  - Outputs: Research text  
  - Edge Cases: API limits, response errors.

- **Attio Search People Tool**  
  - Type: HTTP Request Tool  
  - Role: Searches Attio CRM people records by email  
  - Inputs: Person email from AI context  
  - Outputs: CRM person records  
  - Edge Cases: API auth failure, missing records.

- **Attio List Notes Tool**  
  - Type: HTTP Request Tool  
  - Role: Lists notes associated with CRM person records  
  - Inputs: Parent record ID from Attio search  
  - Outputs: Notes metadata  
  - Edge Cases: API errors, empty notes.

- **Attio Read Notes Tool**  
  - Type: HTTP Request Tool  
  - Role: Retrieves content of individual notes for AI analysis  
  - Inputs: Note ID from list notes output  
  - Outputs: Note content  
  - Edge Cases: API limits.

- **Primary AI Model & Fallback AI Model**  
  - Type: Langchain LM Chat (OpenRouter)  
  - Role: Primary and fallback AI models for generating research output  
  - Configuration: Gemini 2.5 Flash as primary, GPT-5 fallback  
  - Edge Cases: Model availability, retries on failure.

- **Meeting Data Parser**  
  - Type: Langchain Output Parser  
  - Role: Parses AI output to a strict JSON schema describing meeting brief components  
  - Config: Manual schema includes meeting title, datetime, attendees, objectives, topics, outcomes  
  - Edge Cases: Parsing errors, incomplete AI output.

- **Format Meeting Brief**  
  - Type: Code  
  - Role: Converts parsed meeting brief JSON into markdown formatted text suitable for CRM notes and Slack message  
  - Logic: Formats attendees, context, objectives, topics, next steps as markdown  
  - Edge Cases: Missing fields, formatting errors.

- **Filter External Attendees**  
  - Type: Set  
  - Role: Removes the user’s own email from attendees list to focus on external participants  
  - Inputs: Parsed meeting JSON  
  - Outputs: Filtered attendee array  
  - Edge Cases: Empty attendees.

- **Split Individual Attendees**  
  - Type: SplitOut  
  - Role: Splits each external attendee into individual items for CRM lookup  
  - Inputs: Filter External Attendees output  
  - Outputs: Single attendee per item  
  - Edge Cases: Empty list.

---

#### 1.4 CRM Integration

**Overview:**  
For each attendee, checks if they exist in Attio CRM by email, creates a new person record if not found, then creates a meeting prep note attached to the person.

**Nodes Involved:**  
- Find Person in CRM  
- Check if Person Exists in CRM  
- Create New Person Record  
- Combine Person Records  
- Create Meeting Prep Note

**Node Details:**

- **Find Person in CRM**  
  - Type: HTTP Request  
  - Role: Queries Attio API for person by email address  
  - Config: POST JSON filter on email_addresses field  
  - Inputs: Single attendee email  
  - Outputs: CRM person data or empty  
  - Edge Cases: API failures, invalid email.

- **Check if Person Exists in CRM**  
  - Type: If  
  - Role: Checks if CRM query returned a record (by existence of record_id)  
  - Outputs:  
    - True: Person exists, continue flow  
    - False: Create new person record  
  - Edge Cases: False negatives due to API issues.

- **Create New Person Record**  
  - Type: HTTP Request  
  - Role: Creates new person in Attio with attendee email  
  - Inputs: Attendee email from split node  
  - Outputs: New person record info  
  - Edge Cases: API limits, duplicate creation risks.

- **Combine Person Records**  
  - Type: Merge  
  - Role: Merges results from existing or new person record into a single data stream  
  - Inputs: Outputs from 'Check if Person Exists' true and false branches  
  - Outputs: Unified person record data for note creation

- **Create Meeting Prep Note**  
  - Type: HTTP Request  
  - Role: Creates a markdown-formatted note titled "Meeting Prep" under the corresponding person in CRM  
  - Inputs: Person record ID and formatted markdown brief  
  - Outputs: Confirmation of note creation  
  - Edge Cases: API errors, invalid record ID.

---

#### 1.5 Meeting Brief Formatting & Aggregation

**Overview:**  
Aggregates individual meeting markdown briefs, optimizes and consolidates them with AI, then formats final content for Slack posting.

**Nodes Involved:**  
- Combine All Meeting Briefs  
- Meeting Brief Optimizer  
- Brief Optimizer AI Model  
- Brief Structure Parser  
- Brief Parser AI Model  
- Format Slack Block Kit Message

**Node Details:**

- **Combine All Meeting Briefs**  
  - Type: Aggregate  
  - Role: Aggregates all meeting markdown texts into an array for batch processing  
  - Inputs: Multiple formatted meeting briefs  
  - Outputs: Array of markdown strings  
  - Edge Cases: Large payloads.

- **Meeting Brief Optimizer**  
  - Type: Langchain Agent (AI)  
  - Role: Optimizes and summarizes multiple meeting briefs; extracts key fields like purpose, context, action, watch for  
  - Inputs: Aggregated markdown array  
  - Outputs: Structured brief array  
  - Edge Cases: AI response quality, model availability.

- **Brief Optimizer AI Model**  
  - Type: LM Chat OpenRouter  
  - Role: GPT-5-mini model to generate optimized briefs  
  - Inputs: From Meeting Brief Optimizer  
  - Outputs: AI-generated structured data  
  - Edge Cases: API limits.

- **Brief Structure Parser**  
  - Type: Langchain Output Parser  
  - Role: Parses optimized AI output into JSON array of meeting briefs with defined schema  
  - Edge Cases: Parsing failures.

- **Brief Parser AI Model**  
  - Type: LM Chat OpenRouter  
  - Role: Gemini 2.5 Flash model used to parse and validate AI outputs into structured data  
  - Edge Cases: Model timeouts.

- **Format Slack Block Kit Message**  
  - Type: Code  
  - Role: Converts structured meeting briefs into Slack Block Kit JSON format for rich message posting  
  - Logic: Includes header, date, meeting sections with times, purpose, context, action, and watch-for notes; truncates long texts  
  - Outputs: JSON string for Slack blocks  
  - Edge Cases: Slack block limits, JSON formatting errors.

---

#### 1.6 Slack Notification

**Overview:**  
Sends the final aggregated and formatted meeting briefs as a Slack message to a configured channel.

**Nodes Involved:**  
- Send Meeting Brief to Slack

**Node Details:**

- **Send Meeting Brief to Slack**  
  - Type: Slack  
  - Role: Posts the daily meeting brief Block Kit message to Slack channel  
  - Config: OAuth2 Slack credentials, configured channel ID, fallback plain text message  
  - Inputs: Slack Block Kit JSON from the formatting node  
  - Edge Cases: Slack API errors, invalid channel, message size limits.

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                          | Input Node(s)                      | Output Node(s)                | Sticky Note                                                                                                                                        |
|-------------------------------|--------------------------------------|----------------------------------------|----------------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                   | Sticky Note                          | Describes AI-powered meeting prep      | None                             | None                         | ## AI-Powered Meeting Preparation Automation (Full workflow description and requirements)                                                         |
| Sticky Note1                  | Sticky Note                          | Describes the daily schedule trigger   | None                             | None                         | ### Daily Schedule Trigger Runs Monday-Friday at 6:00 AM Cron: `0 6 * * 1-5`                                                                       |
| Daily Weekday Trigger         | Schedule Trigger                     | Triggers workflow every weekday at 6AM| None                             | Fetch Today's Calendar Events |                                                                                                                                                    |
| Sticky Note2                  | Sticky Note                          | Describes calendar integration         | None                             | None                         | ### Calendar Integration Fetches today's events and filters for external meetings **Important**: Update the calendar email to your own             |
| Fetch Today's Calendar Events | Google Calendar                      | Fetches all calendar events for the day| Daily Weekday Trigger            | Filter External Meetings Only |                                                                                                                                                    |
| Filter External Meetings Only | Filter                              | Filters for meetings with external attendees| Fetch Today's Calendar Events | Check for External Meetings    | Keep only events with attendees (external meetings)                                                                                               |
| Check for External Meetings   | If                                  | Checks if any external meetings exist  | Filter External Meetings Only    | Split Individual Meetings, Send No Meetings Message |                                                                                                                                                    |
| Send No Meetings Message      | Slack                               | Sends no meetings notification to Slack| Check for External Meetings (false branch)| None                      |                                                                                                                                                    |
| Sticky Note3                  | Sticky Note                          | Describes the AI meeting research block| None                             | None                         | ### AI Meeting Research When meetings are found, this agent researches CRM, email, calendar, Perplexity and generates briefs                       |
| Split Individual Meetings     | SplitOut                            | Splits calendar events into single meetings| Check for External Meetings (true branch)| AI Meeting Research Agent |                                                                                                                                                    |
| AI Meeting Research Agent     | Langchain Agent                     | Orchestrates AI research on meetings   | Split Individual Meetings        | Format Meeting Brief          |                                                                                                                                                    |
| Search Calendar History Tool  | Google Calendar Tool                | Searches past calendar events           | AI Meeting Research Agent (tool) | AI Meeting Research Agent     |                                                                                                                                                    |
| Search Gmail History Tool     | Gmail Tool                         | Searches past email history             | AI Meeting Research Agent (tool) | AI Meeting Research Agent     |                                                                                                                                                    |
| Perplexity Research Tool      | Perplexity Tool                    | Performs external research when needed  | AI Meeting Research Agent (tool) | AI Meeting Research Agent     |                                                                                                                                                    |
| Attio Search People Tool      | HTTP Request Tool                  | Searches Attio CRM for person by email | AI Meeting Research Agent (tool) | AI Meeting Research Agent     |                                                                                                                                                    |
| Attio List Notes Tool         | HTTP Request Tool                  | Lists notes related to Attio person    | AI Meeting Research Agent (tool) | AI Meeting Research Agent     |                                                                                                                                                    |
| Attio Read Notes Tool         | HTTP Request Tool                  | Reads content of Attio notes            | AI Meeting Research Agent (tool) | AI Meeting Research Agent     |                                                                                                                                                    |
| Primary AI Model              | Langchain LM Chat OpenRouter       | Primary AI model for research agent    | AI Meeting Research Agent (lmChat) | AI Meeting Research Agent (fallback) |                                                                                                                                                    |
| Fallback AI Model             | Langchain LM Chat OpenRouter       | Fallback AI model if primary fails      | AI Meeting Research Agent (lmChat) | AI Meeting Research Agent     |                                                                                                                                                    |
| Meeting Data Parser           | Langchain Output Parser            | Parses AI output into structured JSON   | AI Meeting Research Agent (outputParser) | Format Meeting Brief          |                                                                                                                                                    |
| Format Meeting Brief          | Code                              | Formats structured meeting data to markdown| Meeting Data Parser            | Filter External Attendees, Combine All Meeting Briefs |                                                                                                                                                    |
| Filter External Attendees     | Set                               | Removes user's own email from attendee list| Format Meeting Brief           | Split Individual Attendees    |                                                                                                                                                    |
| Split Individual Attendees    | SplitOut                          | Splits attendees into single items      | Filter External Attendees        | Find Person in CRM            |                                                                                                                                                    |
| Find Person in CRM            | HTTP Request                      | Searches Attio CRM for person by email | Split Individual Attendees       | Check if Person Exists in CRM |                                                                                                                                                    |
| Check if Person Exists in CRM | If                                | Checks if CRM person record exists      | Find Person in CRM               | Combine Person Records, Create New Person Record |                                                                                                                                                    |
| Create New Person Record      | HTTP Request                      | Creates new person in Attio CRM         | Check if Person Exists in CRM    | Combine Person Records        |                                                                                                                                                    |
| Combine Person Records        | Merge                             | Merges person record outputs             | Check if Person Exists in CRM, Create New Person Record | Create Meeting Prep Note |                                                                                                                                                    |
| Create Meeting Prep Note      | HTTP Request                      | Creates meeting prep note in Attio CRM  | Combine Person Records           | None                         |                                                                                                                                                    |
| Combine All Meeting Briefs    | Aggregate                        | Aggregates all markdown meeting briefs  | Format Meeting Brief             | Meeting Brief Optimizer       |                                                                                                                                                    |
| Meeting Brief Optimizer       | Langchain Agent                  | Optimizes and summarizes meeting briefs| Combine All Meeting Briefs       | Format Slack Block Kit Message|                                                                                                                                                    |
| Brief Optimizer AI Model      | Langchain LM Chat OpenRouter     | AI model for meeting brief optimizer    | Meeting Brief Optimizer          | Meeting Brief Optimizer       |                                                                                                                                                    |
| Brief Structure Parser        | Langchain Output Parser          | Parses optimized meeting briefs         | Meeting Brief Optimizer          | Meeting Brief Optimizer       |                                                                                                                                                    |
| Brief Parser AI Model         | Langchain LM Chat OpenRouter     | Parses brief structure with AI model    | Meeting Brief Optimizer          | Meeting Brief Optimizer       |                                                                                                                                                    |
| Format Slack Block Kit Message| Code                            | Formats meeting briefs into Slack Block Kit JSON | Meeting Brief Optimizer       | Send Meeting Brief to Slack  | Converts AI Agent output to Slack Block Kit format                                                                                               |
| Send Meeting Brief to Slack   | Slack                           | Sends final meeting brief message to Slack channel | Format Slack Block Kit Message | None                         | Configure your channel ID and credentials. The 'text' field is required as fallback for notifications.                                            |
| Sticky Note4                  | Sticky Note                      | Describes the CRM integration points    | None                           | None                         | ### CRM Integration The workflow includes Attio CRM integration: search people, create records, list/read notes, create meeting prep notes         |
| Sticky Note5                  | Sticky Note                      | Describes Slack integration              | None                           | None                         | ### Slack Integration Final step sends formatted briefs to Slack; update channel ID                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Cron expression: `0 6 * * 1-5` (6:00 AM Monday-Friday)  
   - Purpose: Start workflow daily on weekdays.

2. **Create Google Calendar node:**  
   - Type: Google Calendar  
   - Operation: Get All Events  
   - Time Min: `={{ $today }}`  
   - Time Max: `={{ $today.plus({ day: 1 }) }}`  
   - Calendar: Set to your email address  
   - Credentials: Google Calendar OAuth2  
   - Connect from Schedule Trigger.

3. **Create Filter node:**  
   - Type: Filter  
   - Condition: Check if `attendees` field exists (array non-empty)  
   - Connect from Google Calendar node.

4. **Create If node:**  
   - Type: If  
   - Condition: Check if filtered events have any `id` field (exists)  
   - True: Proceed with meetings  
   - False: Send no meetings message.

5. **Create Slack node for no meetings:**  
   - Type: Slack  
   - Message: "Good morning! No external meetings scheduled for today. Enjoy your focus time!"  
   - Channel: Your Slack channel ID  
   - Credentials: Slack OAuth2  
   - Connect from If node false branch.

6. **Create SplitOut node to split meetings:**  
   - Type: SplitOut  
   - Field to split: `id` (event ID)  
   - Connect from If node true branch.

7. **Create Langchain Agent node (AI Meeting Research Agent):**  
   - Type: Langchain Agent (agent)  
   - System Message: Expert meeting prep assistant with CRM, email, calendar, and company research access  
   - Prompt: Pass meeting details (summary, time, attendees excluding self) and tasks to research attendees via CRM, Gmail, calendar, Perplexity  
   - Models: Primary (Gemini 2.5 Flash), fallback (GPT-5)  
   - Connect from Split Individual Meetings.

8. **Add supporting tool nodes for AI Agent:**  
   - Google Calendar Tool: To search past calendar events  
   - Gmail Tool: To search past emails  
   - Perplexity Tool: For external research  
   - HTTP Request Tools for Attio CRM: Search people, list notes, read notes  
   - Connect all tools as AI tools for the AI Meeting Research Agent node.

9. **Add Langchain Output Parser node (Meeting Data Parser):**  
   - Define a strict JSON schema for meeting briefs with fields like meetingTitle, attendees array, meetingObjective, discussionTopics, etc.  
   - Connect AI Meeting Research Agent’s output.

10. **Add Code node (Format Meeting Brief):**  
    - Run once per item  
    - JavaScript to format parsed JSON data into markdown with sections for attendees, context, objectives, topics, next steps  
    - Connect from Meeting Data Parser.

11. **Add Set node (Filter External Attendees):**  
    - Remove user’s own email from attendees array  
    - Connect from Format Meeting Brief.

12. **Add SplitOut node (Split Individual Attendees):**  
    - Split attendees array into single attendee items  
    - Connect from Filter External Attendees.

13. **Add HTTP Request nodes for Attio CRM:**  
    - Find Person in CRM: POST query by email  
    - Check if Person Exists: If node checking existence of record_id  
    - Create New Person Record: POST creating person if not found  
    - Combine Person Records: Merge node combining existing/new records  
    - Create Meeting Prep Note: POST note with markdown content linked to person  
    - Connect these nodes in sequence from Split Individual Attendees.

14. **Add Aggregate node (Combine All Meeting Briefs):**  
    - Aggregate all individual markdown meeting briefs into an array  
    - Connect from Format Meeting Brief node.

15. **Add Langchain Agent node (Meeting Brief Optimizer):**  
    - Input: Aggregated markdown array  
    - Task: Optimize and summarize each meeting brief extracting title, time, purpose, context, action, watch-for  
    - Connect from Aggregate node.

16. **Add Langchain LM Chat nodes:**  
    - Brief Optimizer AI Model (GPT-5-mini)  
    - Brief Parser AI Model (Gemini 2.5 Flash)  
    - Connect to Meeting Brief Optimizer for model calls and output parsing.

17. **Add Langchain Output Parser node (Brief Structure Parser):**  
    - Define JSON schema for array of meeting briefs with fields like meetingTitle, meetingTimePST, purpose, action, etc.  
    - Connect from Meeting Brief Optimizer.

18. **Add Code node (Format Slack Block Kit Message):**  
    - Convert structured meeting briefs into Slack Block Kit JSON with header, context, sections, dividers  
    - Truncate content to meet Slack limits  
    - Connect from Brief Structure Parser.

19. **Add Slack node (Send Meeting Brief to Slack):**  
    - Post message as Block Kit to your Slack channel  
    - Text fallback: “Daily meeting brief”  
    - Credentials: Slack OAuth2  
    - Connect from Format Slack Block Kit Message.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                               | Context or Link                                                                                                                                                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow requires valid API credentials for Google Calendar, Gmail, Slack, Attio CRM, OpenRouter (for AI models), and Perplexity. Update all credential nodes accordingly.                                                                             | Credentials setup in n8n for OAuth2 and API keys.                                                                                                                                                                                        |
| OpenRouter models used include "google/gemini-2.5-flash" as primary AI and "openai/gpt-5" or "openai/gpt-5-mini" as fallback or optimizer models. Adjust model names based on availability and cost preferences.                                                | OpenRouter API documentation: https://docs.openrouter.ai/                                                                                                                                                                                |
| Slack nodes require Bot token with channel access and correct channel ID to post messages. Channels should be pre-created.                                                                                                                                 | Slack API: https://api.slack.com/                                                                                                                                                                                                        |
| Attio CRM API integration uses bearer token authentication. The workflow creates and updates person records and notes via REST API.                                                                                                                        | Attio API: https://docs.attio.com/reference                                                                                                                                                                                             |
| The AI Meeting Research Agent uses a multi-tool approach combining CRM, Gmail, calendar, and external research tools orchestrated via Langchain agent node, leveraging AI for summarization and context extraction.                                          | Langchain Agent node details: https://docs.n8n.io/nodes/n8n-nodes-langchain/agent/                                                                                                                                                        |
| Slack Block Kit formatting is custom-coded to provide rich, readable meeting brief messages with headers, sections, lists, and quotes.                                                                                                                    | Slack Block Kit reference: https://api.slack.com/block-kit                                                                                                                                                                              |
| The workflow assumes user email is set in nodes for filtering own meetings and attendees. Update all occurrences of `YOUR_EMAIL@example.com` and `YOUR_SLACK_CHANNEL_ID` before running.                                                                     | Replace placeholders in relevant node parameters.                                                                                                                                                                                       |
| Costs apply for AI model usage; monitor usage to manage expenses.                                                                                                                                                                                          |                                                                                                                                                                                                                                           |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, respecting all current content policies and containing no illegal, offensive, or protected elements. All data processed is legal and public.

---