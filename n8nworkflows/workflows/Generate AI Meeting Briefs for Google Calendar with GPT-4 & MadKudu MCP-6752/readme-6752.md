Generate AI Meeting Briefs for Google Calendar with GPT-4 & MadKudu MCP

https://n8nworkflows.xyz/workflows/generate-ai-meeting-briefs-for-google-calendar-with-gpt-4---madkudu-mcp-6752


# Generate AI Meeting Briefs for Google Calendar with GPT-4 & MadKudu MCP

### 1. Workflow Overview

This workflow automates the preparation of meeting briefs for upcoming Google Calendar events that involve external attendees. It leverages AI tools including MadKudu MCP and OpenAI’s GPT-4 to research meeting participants and their associated companies, then generates and attaches a concise meeting summary directly into a duplicate calendar event. This enables users to arrive prepared for external-facing meetings without manual research.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Input Trigger:** Periodically checks the user’s Google Calendar for meetings scheduled within the next hour.
- **1.2 Event Extraction and Filtering:** Retrieves all relevant events, splits them individually, and filters for meetings containing external attendees based on a company domain variable.
- **1.3 AI Research and Summary Generation:** Uses MadKudu MCP and GPT-4 to research external attendees and companies, then generates a structured meeting brief.
- **1.4 Creation of Summary Event:** Creates a private calendar event at the same time as the original meeting with the AI-generated summary in the description.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Input Trigger

**Overview:**  
This block triggers the workflow every hour to check for upcoming meetings in the next 60 minutes.

**Nodes Involved:**  
- Schedule Trigger  
- Get many events  
- Split Calendar Events  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow on a recurring basis.  
  - Configuration: Set to trigger every 1 hour.  
  - Input: None (time-based trigger)  
  - Output: Triggers "Get many events" node.  
  - Edge Cases: Trigger timing misalignment could cause missed meetings; frequency should balance timeliness and API usage limits.

- **Get many events**  
  - Type: Google Calendar (Get All Events)  
  - Role: Fetches all calendar events up to one hour from the current time.  
  - Configuration:  
    - Calendar: User’s Google Calendar (email: margo.rey@madkudu.com)  
    - TimeMax: Now + 1 hour, dynamic expression `{{$now.plus({ hour: 1 })}}`  
    - Operation: Get all events  
  - Input: Trigger from Schedule Trigger  
  - Output: Passes full list of events to Split Calendar Events  
  - Credentials: OAuth2 Google Calendar account  
  - Edge Cases: API quota limits, network or auth failures, no events found.

- **Split Calendar Events**  
  - Type: Split Out  
  - Role: Separates the array of events into individual event items for processing.  
  - Configuration: Splits on `id` field, includes all other event fields.  
  - Input: List of events from Get many events  
  - Output: Single events streamed one-by-one to next node  
  - Edge Cases: Empty event list leads to no further processing.

#### 2.2 Event Extraction and Filtering

**Overview:**  
Filters the individual events to keep only those with at least one external attendee (email domain differs from the company domain specified in a variable).

**Nodes Involved:**  
- Keep meetings with external attendees

**Node Details:**

- **Keep meetings with external attendees**  
  - Type: Filter  
  - Role: Filters events to only those with external attendees.  
  - Configuration:  
    - Condition 1: Event must have more than 1 attendee (`attendees.length > 1`)  
    - Condition 2: At least one attendee’s email domain is different from the company domain variable `my_company_domain`  
  - Input: Individual event from Split Calendar Events  
  - Output: Passes filtered events with external attendees to AI Agent - Research Attendees  
  - Variables: Uses `$vars.my_company_domain` set externally (e.g., "madkudu.com")  
  - Edge Cases: No external attendees results in no output; missing or incorrect domain variable causes filtering errors.

#### 2.3 AI Research and Summary Generation

**Overview:**  
Uses MadKudu MCP and OpenAI GPT-4 to research and analyze external attendees and their companies, then generates a formatted meeting brief.

**Nodes Involved:**  
- AI Agent - Research Attendees  
- MadKudu MCP  
- OpenAI Model

**Node Details:**

- **AI Agent - Research Attendees**  
  - Type: LangChain Agent  
  - Role: Orchestrates AI research using MadKudu MCP and GPT-4 to generate a meeting summary.  
  - Configuration:  
    - Prompt includes meeting context (title, description) and a JSON list of external attendees’ emails.  
    - Calls MadKudu tools for data on attendees and accounts: value prop, account details, activities, top persons, person details and activities, and account brief instructions.  
    - Output format structured with sections: TL;DR, Person Summary, Account Summary, Account Brief.  
  - Inputs: Filtered event JSON with attendees and meeting details  
  - Outputs: Plain text meeting summary  
  - Connected AI Tools:  
    - MadKudu MCP (for data lookups)  
    - OpenAI Model (GPT-4) for language generation  
  - Edge Cases: API failures, rate limits, unexpected or missing attendee data, prompt expression errors.

- **MadKudu MCP**  
  - Type: MadKudu MCP Client Tool  
  - Role: Provides streaming data on accounts and persons to AI Agent.  
  - Configuration: SSE endpoint uses variable `{{$vars.madkudu_api_key}}` for authentication.  
  - Input: Called by AI Agent node  
  - Output: Streaming data feed to AI Agent  
  - Edge Cases: Authentication errors, network issues, invalid API key.

- **OpenAI Model**  
  - Type: Language Model Chat OpenAI  
  - Role: Executes GPT-4.1-mini model for natural language generation.  
  - Configuration: Model set to “gpt-4.1-mini”  
  - Credentials: Uses OpenAI API key (n8n free credits)  
  - Input: Prompt from AI Agent  
  - Output: AI-generated text summary  
  - Edge Cases: API quota exceeded, timeouts, invalid prompt format.

#### 2.4 Creation of Summary Event

**Overview:**  
Creates a new calendar event synchronized with the original meeting time, embedding the AI-generated summary in the event description for user reference.

**Nodes Involved:**  
- Send summary as event

**Node Details:**

- **Send summary as event**  
  - Type: Google Calendar (Create Event)  
  - Role: Adds a new calendar event with the AI-generated meeting brief.  
  - Configuration:  
    - Calendar: Same as original user’s calendar  
    - Start and End time: Matches original meeting start and end (`start.dateTime`, `end.dateTime`)  
    - Summary: Prep meeting + original meeting summary title  
    - Description: AI-generated summary text from AI Agent output  
  - Input: Output from AI Agent - Research Attendees  
  - Credentials: Google Calendar OAuth2  
  - Edge Cases: API write failures, event duplication, calendar permission issues.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                             | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                             |
|-------------------------------|----------------------------------|---------------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger                 | Triggers workflow hourly                     | None                        | Get many events             | ### 1. Check for new meetings every hour: The Scheduler is configured for every hour. It will check... |
| Get many events              | Google Calendar (Get All Events) | Retrieves upcoming calendar events           | Schedule Trigger            | Split Calendar Events       |                                                                                                       |
| Split Calendar Events         | Split Out                       | Splits event array to individual events      | Get many events             | Keep meetings with external attendees | ### 2. Keep External Meetings only: Set your company domain variable. Filters meetings with externals. |
| Keep meetings with external attendees | Filter                    | Filters events to retain those with external attendees | Split Calendar Events       | AI Agent - Research Attendees | ### 2. Keep External Meetings only: Filters out internal-only meetings based on your company domain.  |
| AI Agent - Research Attendees | LangChain Agent                 | Researches attendees and generates summary   | Keep meetings with external attendees | Send summary as event       | ### 3. Research Attendees and Generate Summary: Enrich attendee & company data using MadKudu and AI.   |
| MadKudu MCP                  | MadKudu MCP Client Tool         | Provides account and person data streams     | AI Agent - Research Attendees (ai_tool) | AI Agent - Research Attendees (ai_tool) |                                                                                                       |
| OpenAI Model                 | LangChain LM Chat OpenAI        | Executes GPT-4 for text generation            | AI Agent - Research Attendees (ai_languageModel) | AI Agent - Research Attendees (ai_languageModel) |                                                                                                       |
| Send summary as event         | Google Calendar (Create Event)  | Creates calendar event with AI-generated brief | AI Agent - Research Attendees | None                        | ### 4. Send Meeting invitation with summary: Creates private calendar event with AI brief.             |
| Sticky Note                  | Sticky Note                    | Documentation and user guidance               | None                        | None                        | # Try it Out! This workflow builds a Meeting Prep AI Assistant...                                      |
| Sticky Note1                 | Sticky Note                    | Documentation and user guidance               | None                        | None                        | ### 1. Check for new meetings every hour: The Scheduler is configured for every hour...                |
| Sticky Note2                 | Sticky Note                    | Documentation and user guidance               | None                        | None                        | ### 2. Keep External Meetings only: Set your company email domain first as a Variable...               |
| Sticky Note3                 | Sticky Note                    | Documentation and user guidance               | None                        | None                        | ### 3. Research Attendees and Generate Summary: Enrich attendee & company data using MadKudu and AI.   |
| Sticky Note4                 | Sticky Note                    | Documentation and user guidance               | None                        | None                        | ### 4. Send Meeting invitation with summary: Create a private calendar event with the AI-generated brief. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Parameters: Set to trigger every 1 hour (interval field: hours, value: 1).

2. **Create Google Calendar Node ("Get many events")**  
   - Type: Google Calendar (Get All Events)  
   - Parameters:  
     - Operation: Get All  
     - Calendar: Select your Google Calendar account (e.g., margo.rey@madkudu.com).  
     - TimeMax: Use expression `{{$now.plus({ hour: 1 })}}` to fetch events up to 1 hour ahead.  
   - Credentials: Configure OAuth2 credentials for your Google account.  
   - Connect Schedule Trigger → Get many events.

3. **Create Split Out Node ("Split Calendar Events")**  
   - Type: Split Out  
   - Parameters:  
     - Field to split out: `id` (splits array of events into single items).  
     - Include all other fields.  
   - Connect Get many events → Split Calendar Events.

4. **Create Filter Node ("Keep meetings with external attendees")**  
   - Type: Filter  
   - Parameters:  
     - Condition 1: `attendees.length > 1`  
     - Condition 2: At least one attendee’s email domain is different from the company domain variable.  
       Use expression: `$json.attendees.some(a => !a.email.includes('@' + $vars.my_company_domain))`  
   - Variables: Create a variable named `my_company_domain` with your company domain string (e.g., "madkudu.com").  
   - Connect Split Calendar Events → Keep meetings with external attendees.

5. **Create LangChain Agent Node ("AI Agent - Research Attendees")**  
   - Type: LangChain Agent  
   - Parameters:  
     - Prompt: Insert a detailed multi-section prompt with meeting context, attendee emails filtered for external domains, and instructions to use MadKudu MCP and OpenAI.  
     - Prompt Type: Define  
     - Enable output parser for plain text output.  
   - Connect Keep meetings with external attendees → AI Agent - Research Attendees.

6. **Create MadKudu MCP Node**  
   - Type: MadKudu MCP Client Tool  
   - Parameters:  
     - SSE Endpoint: Use expression `https://mcp.madkudu.com/{{$vars.madkudu_api_key}}/sse`  
   - Variables: Add `madkudu_api_key` with your API key value.  
   - Connect MadKudu MCP → AI Agent - Research Attendees (ai_tool connection).

7. **Create OpenAI Model Node**  
   - Type: LangChain LM Chat OpenAI  
   - Parameters:  
     - Model: Select "gpt-4.1-mini".  
   - Credentials: Configure OpenAI API credentials with your API key.  
   - Connect OpenAI Model → AI Agent - Research Attendees (ai_languageModel connection).

8. **Create Google Calendar Node ("Send summary as event")**  
   - Type: Google Calendar (Create Event)  
   - Parameters:  
     - Calendar: Same Google Calendar as before.  
     - Start: Set to original meeting start `{{$node["Keep meetings with external attendees"].json["start"]["dateTime"]}}`  
     - End: Set to original meeting end `{{$node["Keep meetings with external attendees"].json["end"]["dateTime"]}}`  
     - Summary: Use expression `"Prep meeting " + $node["Keep meetings with external attendees"].json["summary"]`  
     - Description: Use AI-generated summary output from AI Agent node `$json.output`.  
   - Credentials: Use the same Google Calendar OAuth2 credentials.  
   - Connect AI Agent - Research Attendees → Send summary as event.

9. **Add Variables Setup**  
   - Create workflow variables:  
     - `my_company_domain`: Your company email domain (e.g., "madkudu.com")  
     - `madkudu_api_key`: Your MadKudu API key  

10. **Optional: Add Sticky Notes for Documentation**  
    - Add sticky notes to explain each block: hourly schedule, filtering logic, AI research, and event creation for user clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow builds a Meeting Prep AI Assistant that automatically prepares meeting briefs for external meetings, helping users show up prepared without manual research.                    | See first Sticky Note content in the workflow.                                                              |
| Set your company email domain as a variable `my_company_domain` to filter out internal attendees reliably.                                                                                   | Variable documentation: https://docs.n8n.io/code/variables/#create-variables                                  |
| The AI Agent leverages MadKudu MCP tools and OpenAI GPT-4 to research attendees and generate a structured meeting summary.                                                                  | MadKudu MCP docs and OpenAI API references.                                                                 |
| Duplicate calendar event is created privately with the AI-generated brief to avoid modifying the original meeting.                                                                           |                                                                                                             |
| For assistance or community support, join the n8n Discord or Forum.                                                                                                                          | https://community.n8n.io/                                                                                    |

---

**Disclaimer:** The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.