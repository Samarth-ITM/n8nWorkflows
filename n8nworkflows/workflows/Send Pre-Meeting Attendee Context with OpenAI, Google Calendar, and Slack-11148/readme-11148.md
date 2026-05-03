Send Pre-Meeting Attendee Context with OpenAI, Google Calendar, and Slack

https://n8nworkflows.xyz/workflows/send-pre-meeting-attendee-context-with-openai--google-calendar--and-slack-11148


# Send Pre-Meeting Attendee Context with OpenAI, Google Calendar, and Slack

### 1. Workflow Overview

This workflow, titled **"Send Pre-Meeting Attendee Context with OpenAI, Google Calendar, and Slack"**, serves as an AI-powered pre-meeting assistant designed to automatically fetch, analyze, and summarize key contextual information about upcoming meetings and attendees. It targets users who want to be well-prepared for their scheduled meetings by receiving concise, AI-generated briefing messages delivered via Slack.

**Target Use Cases:**

- Professionals who want timely, context-rich pre-meeting summaries.
- Users with frequent meetings who need quick recaps of last email correspondences with attendees.
- Teams using Google Calendar, Gmail, OpenAI, and Slack integrations to automate meeting preparation.

**Logical Blocks:**

- **1.1 Scheduled Meeting Discovery:** Periodic hourly trigger to find meetings scheduled in the next hour from Google Calendar.
- **1.2 Attendee Extraction and Processing:** Extract attendees from meeting invites using AI-powered information extraction.
- **1.3 Last Correspondence Retrieval:** For each attendee, fetch the most recent email correspondence from Gmail.
- **1.4 Correspondence Summarization:** Use OpenAI language model to create a succinct summary of the email conversation.
- **1.5 Pre-Meeting Summary Generation:** Generate a personalized, structured message summarizing meeting details and attendee context.
- **1.6 Notification Dispatch:** Send the generated summary message to the user via Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Meeting Discovery

- **Overview:**  
  This block triggers the workflow on an hourly basis and queries Google Calendar for meetings scheduled to start during the next hour (i.e., between +1 hour and +2 hours from the current UTC time).

- **Nodes Involved:**  
  - Schedule Trigger  
  - Sticky Note (explanatory)  
  - Check For Upcoming Meetings

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Role: Initiates the workflow every hour.  
    - Configuration: Interval set to 1 hour.  
    - Inputs: None (trigger node)  
    - Outputs: Next node is "Check For Upcoming Meetings".  
    - Edge Cases: If the node fails or the workflow is inactive, meetings won't be fetched.

  - **Sticky Note** (Positioned near Schedule Trigger)  
    - Contains explanation about running hourly queries for meetings in the next hour window.

  - **Check For Upcoming Meetings**  
    - Type: `Google Calendar`  
    - Role: Retrieves up to 5 upcoming meetings scheduled to start during the hour following the next full hour.  
    - Configuration:  
      - Calendar: specific user email configured.  
      - Time Range: from +1 hour (start of hour) to +2 hours (start of hour) in UTC.  
      - Options: singleEvents=true, ordered by startTime.  
    - Inputs: From Schedule Trigger  
    - Outputs: To "Extract Attendee Information".  
    - Edge Cases:  
      - No meetings found: downstream nodes get no data.  
      - API or authentication errors could cause failure.

#### 1.2 Attendee Extraction and Processing

- **Overview:**  
  This block extracts attendee details (excluding the organizer) from each meeting using an AI-powered information extractor. It converts meeting JSON data into structured attendee info with names, emails, and optional LinkedIn URLs.

- **Nodes Involved:**  
  - Extract Attendee Information  
  - Sticky Note (explanatory)  
  - Attendees to List

- **Node Details:**

  - **Extract Attendee Information**  
    - Type: `Information Extractor (LangChain)`  
    - Role: Parses meeting details and extracts a JSON array of attendees excluding the organizer.  
    - Configuration:  
      - Input text constructed from meeting attributes (start time, URL, summary, description, organizer email, attendee emails).  
      - System prompt guides extraction to link description details to attendees.  
      - Output schema expects an array of objects with name, email, linkedin_url fields.  
    - Inputs: From "Check For Upcoming Meetings"  
    - Outputs: To "Attendees to List"  
    - Edge Cases:  
      - Extraction may fail or miss details if meeting data is incomplete.  
      - AI output validation required to avoid malformed data.

  - **Sticky Note1**  
    - Explains use of Information Extractor node with AI to avoid complex manual parsing.

  - **Attendees to List**  
    - Type: `Split Out`  
    - Role: Splits the array of attendees into individual items for further processing.  
    - Inputs: Output of "Extract Attendee Information" (array of attendees)  
    - Outputs: To "Has Email Address?" node  
    - Edge Cases: Empty attendee list leads to no further processing.

#### 1.3 Last Correspondence Retrieval

- **Overview:**  
  For each attendee with an email address, this block fetches the latest Gmail correspondence from that attendee to the user. This gives context on the most recent communications before the meeting.

- **Nodes Involved:**  
  - Has Email Address? (conditional)  
  - Get Last Correspondence  
  - Has Emails? (conditional)  
  - Get Message Contents  
  - Simplify Emails  
  - Return Email Success  
  - Return Email Error (fallback)  
  - Return Email Error2 (fallback)  
  - Sticky Note5

- **Node Details:**

  - **Has Email Address?**  
    - Type: `If` node  
    - Role: Checks if the attendee object contains a valid email property.  
    - Inputs: From "Attendees to List" (individual attendee)  
    - Outputs:  
      - True: to "Get Last Correspondence"  
      - False: to "Return Email Error" (sets default "No correspondence found")  
    - Edge Cases: Missing or malformed email results in error fallback.

  - **Get Last Correspondence**  
    - Type: `Gmail` node  
    - Role: Fetch the most recent email from the attendee's email address (sender) to the user.  
    - Configuration:  
      - Limit: 1  
      - Filters: sender equals attendee email  
      - Operation: getAll  
    - Inputs: From "Has Email Address?" (true branch)  
    - Outputs: To "Has Emails?"  
    - Edge Cases: API rate limits, no emails found, or auth errors.

  - **Has Emails?**  
    - Type: `If` node  
    - Role: Checks if any email data was retrieved.  
    - Inputs: From "Get Last Correspondence"  
    - Outputs:  
      - True: to "Get Message Contents" (to fetch full content)  
      - False: to "Return Email Error2" (sets "No correspondence found")  
    - Edge Cases: Empty result sets handled gracefully.

  - **Get Message Contents**  
    - Type: `Gmail` node  
    - Role: Retrieves detailed content of the specific email message identified by ID.  
    - Inputs: From "Has Emails?" (true branch)  
    - Outputs: To "Simplify Emails"  
    - Edge Cases: Message may be deleted or inaccessible.

  - **Simplify Emails**  
    - Type: `Set` node  
    - Role: Extracts and simplifies key email fields into a flat structure: date, subject, text, from, to.  
    - Inputs: From "Get Message Contents"  
    - Outputs: To "Correspondance Recap Agent"  
    - Edge Cases: Missing fields handled by empty strings.

  - **Return Email Success**  
    - Type: `Set` node  
    - Role: Assigns the email text from the simplified email to `email_summary` property for downstream use.  
    - Inputs: From "Correspondance Recap Agent"  
    - Outputs: To "Attendee Research Agent"  
    - Edge Cases: None significant.

  - **Return Email Error / Return Email Error2**  
    - Type: `Set` nodes  
    - Role: Provide default summary "No correspondence found." when no valid email exists or found.  
    - Inputs: From "Has Email Address?" false branch or "Has Emails?" false branch  
    - Outputs: To "Attendee Research Agent"  
    - Edge Cases: Used to avoid null or undefined downstream data.

  - **Sticky Note5**  
    - Explains the rationale for fetching last email correspondence to help the user pick up prior discussions.

#### 1.4 Correspondence Summarization

- **Overview:**  
  This block uses OpenAI's GPT model to summarize the last email correspondence, condensing it into a succinct recap to prepare the user for the meeting discussion.

- **Nodes Involved:**  
  - Correspondance Recap Agent (LangChain Chain LLM)  
  - OpenAI Model2 (OpenAI GPT)  
  - Sticky Note12

- **Node Details:**

  - **Correspondance Recap Agent**  
    - Type: `Chain LLM` (LangChain)  
    - Role: Summarizes last email correspondence with a prompt framing the task as helping the user recap prior discussions.  
    - Configuration:  
      - Input text includes from, to, date, subject, and full email text.  
      - Prompt instructs succinct summary of discussion, changes, agreements.  
    - Inputs: From "Simplify Emails" or "Return Email Error" fallback  
    - Outputs: To "Return Email Success"  
    - Edge Cases: Potential GPT API rate limits or generation errors.

  - **OpenAI Model2**  
    - Type: `lmChatOpenAi` (OpenAI GPT)  
    - Role: The actual language model node used inside the chain node.  
    - Inputs/Outputs: Linked internally with "Correspondance Recap Agent".  
    - Credentials: Requires valid OpenAI API credential.  
    - Edge Cases: API key expiry, usage limits.

  - **Sticky Note12**  
    - Highlights the purpose of summarizing email chains to remind the user of prior interactions and expectations.

#### 1.5 Pre-Meeting Summary Generation

- **Overview:**  
  Combines meeting and attendee data, including email summaries, to create a personalized pre-meeting notification message using OpenAI GPT.

- **Nodes Involved:**  
  - Attendee Research Agent (LangChain Chain LLM)  
  - OpenAI Model3 (OpenAI GPT)  
  - Return Email Success (input)  
  - Sticky Note3

- **Node Details:**

  - **Attendee Research Agent**  
    - Type: `Chain LLM` (LangChain)  
    - Role: Generates a casual, markdown-formatted SMS-style summary message for the user.  
    - Configuration:  
      - Input text compiles meeting date, URL, summary, description, attendee emails, and last correspondence summaries for each attendee.  
      - Prompt instructs to highlight relevant todos, talking points, and recent attendee activities.  
    - Inputs: Receives attendee email summaries (`email_summary`) and meeting info.  
    - Outputs: To "Send a message" (Slack node).  
    - Edge Cases: GPT errors or incomplete data may degrade message quality.

  - **OpenAI Model3**  
    - Type: `lmChatOpenAi`  
    - Role: Language model node used by "Attendee Research Agent".  
    - Credentials: OpenAI API credentials required.  
    - Edge Cases: Same as other OpenAI nodes.

  - **Sticky Note3**  
    - Describes this node's role in summarizing discussion content to create pre-meeting notifications.

#### 1.6 Notification Dispatch

- **Overview:**  
  Sends the AI-generated pre-meeting summary message to a specified Slack user.

- **Nodes Involved:**  
  - Send a message (Slack)  
  - Sticky Note4

- **Node Details:**

  - **Send a message**  
    - Type: `Slack`  
    - Role: Posts the formatted message text to the target Slack user.  
    - Configuration:  
      - Text: dynamically set from previous node output.  
      - User: selected from a list by Slack user ID (e.g., "U09C3LU03EX").  
    - Inputs: From "Attendee Research Agent"  
    - Outputs: None (end of workflow)  
    - Credentials: Slack API OAuth2 credentials required.  
    - Edge Cases: Slack API rate limits, invalid user ID, or permission errors.

  - **Sticky Note4**  
    - Mentions that Slack can be swapped for other messaging platforms if desired (e.g., Twilio, WhatsApp).

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                                  | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                  |
|----------------------------|----------------------------------|-------------------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger                  | Hourly trigger to start workflow                 | None                             | Check For Upcoming Meetings       | ## 1. Periodically Search For Upcoming Meetings; explains hourly interval setup |
| Check For Upcoming Meetings| Google Calendar                  | Fetch meetings scheduled in the next hour        | Schedule Trigger                 | Extract Attendee Information       |                                                              |
| Extract Attendee Information| Information Extractor (LangChain)| Extract structured attendee info from meeting    | Check For Upcoming Meetings       | Attendees to List                 | ## 2. Extract Attendee Details From Invite; explains AI extraction usage     |
| Attendees to List          | Split Out                        | Split attendees array into individual items       | Extract Attendee Information      | Has Email Address?               |                                                              |
| Has Email Address?         | If                              | Check if attendee has an email address            | Attendees to List                | Get Last Correspondence (true), Return Email Error (false)   |                                                              |
| Get Last Correspondence    | Gmail                           | Fetch last email from attendee                     | Has Email Address? (true)        | Has Emails?                     | ## 3. Fetch Last Email Correspondance; explains importance of last email    |
| Has Emails?                | If                              | Check if any emails were found                     | Get Last Correspondence          | Get Message Contents (true), Return Email Error2 (false)    |                                                              |
| Get Message Contents       | Gmail                           | Retrieve full content of the last email            | Has Emails? (true)               | Simplify Emails                 |                                                              |
| Simplify Emails            | Set                             | Simplify email data fields                          | Get Message Contents             | Correspondance Recap Agent       |                                                              |
| Correspondance Recap Agent | Chain LLM (LangChain)            | Summarize last email correspondence                 | Simplify Emails                 | Return Email Success             | ## 4. Summarize Correspondance For Attendee; explains email summarization  |
| Return Email Success       | Set                             | Store summarized email text                         | Correspondance Recap Agent       | Attendee Research Agent          |                                                              |
| Return Email Error         | Set                             | Default message if no email found (missing email) | Has Email Address? (false)       | Attendee Research Agent          |                                                              |
| Return Email Error2        | Set                             | Default message if no email found (no emails)      | Has Emails? (false)              | Attendee Research Agent          |                                                              |
| Attendee Research Agent    | Chain LLM (LangChain)            | Generate pre-meeting notification message          | Return Email Success, Return Email Error, Return Email Error2 | Send a message                 | ## 5. Pre-Meeting Summary Notification; describes generating summary       |
| Send a message             | Slack                           | Send pre-meeting summary message to Slack user     | Attendee Research Agent          | None                           | ## 6. Send Notification via Slack; suggests alternative messaging platforms  |
| Sticky Note                | Sticky Note                     | Explanatory notes                                  | None                             | None                           | Used multiple times with context-specific explanatory content             |
| OpenAI Model               | lmChatOpenAi                   | OpenAI model for attendee info extraction           | Extract Attendee Information     | Extract Attendee Information (internal) |                                                              |
| OpenAI Model2              | lmChatOpenAi                   | OpenAI model for correspondence recap               | Correspondance Recap Agent       | Correspondance Recap Agent (internal) |                                                              |
| OpenAI Model3              | lmChatOpenAi                   | OpenAI model for pre-meeting summary generation     | Attendee Research Agent          | Attendee Research Agent (internal) |                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: `Schedule Trigger`  
   - Set the interval to every 1 hour.

2. **Add Google Calendar node "Check For Upcoming Meetings"**  
   - Credentials: Configure Google Calendar OAuth2 with proper scopes.  
   - Operation: `getAll`  
   - Calendar: Set to your email/calendar ID.  
   - Set `timeMin` to `{{$now.toUTC().plus(1, 'hour').startOf('hour')}}`  
   - Set `timeMax` to `{{$now.toUTC().plus(2, 'hour').startOf('hour')}}`  
   - Enable `singleEvents` and order by `startTime`.  
   - Connect Schedule Trigger's output to this node's input.

3. **Add Information Extractor node "Extract Attendee Information"**  
   - Type: `Information Extractor (LangChain)`  
   - Input: Construct input text from meeting JSON fields: start time, hangout link, summary, description, organizer email, and attendee emails excluding organizer.  
   - Use a system prompt instructing extraction of attendee name, email, and LinkedIn URL if present.  
   - Define output schema as an array of objects with `name`, `email`, and `linkedin_url` string fields.  
   - Connect "Check For Upcoming Meetings" output here.

4. **Add Split Out node "Attendees to List"**  
   - Split the `output.attendees` array into individual items.  
   - Connect "Extract Attendee Information" output here.

5. **Add If node "Has Email Address?"**  
   - Condition: Check if `email` field exists and is non-empty for the attendee.  
   - Connect "Attendees to List" output here.

6. **Add Gmail node "Get Last Correspondence"**  
   - Credentials: Gmail OAuth2 with proper Gmail API scopes.  
   - Operation: `getAll` with limit 1, filter by sender equal to attendee's email.  
   - Connect "Has Email Address?" true output.

7. **Add If node "Has Emails?"**  
   - Condition: Check if email data is not empty.  
   - Connect "Get Last Correspondence" output.

8. **Add Gmail node "Get Message Contents"**  
   - Operation: `get` message details by passing messageId from first email.  
   - Connect "Has Emails?" true output.

9. **Add Set node "Simplify Emails"**  
   - Extract date, subject, text, from (name and address), to (name and address) fields into flat properties.  
   - Connect "Get Message Contents" output.

10. **Add Chain LLM node "Correspondance Recap Agent"**  
    - Use OpenAI GPT model with prompt: summarize succinctly the last correspondence from provided email fields.  
    - Connect "Simplify Emails" output.

11. **Add Set node "Return Email Success"**  
    - Assign property `email_summary` with the summarized text from previous node.  
    - Connect "Correspondance Recap Agent" output.

12. **Add Set node "Return Email Error"**  
    - Assign `email_summary` = "No correspondance found."  
    - Connect "Has Email Address?" false output.

13. **Add Set node "Return Email Error2"**  
    - Assign `email_summary` = "No correspondance found."  
    - Connect "Has Emails?" false output.

14. **Add Chain LLM node "Attendee Research Agent"**  
    - Use OpenAI GPT model with prompt to generate pre-meeting notification: includes meeting date, URL, summary, description, attendee emails, and each attendee’s last correspondence summary.  
    - Format output as casual SMS message with markdown bullet points.  
    - Connect "Return Email Success", "Return Email Error", and "Return Email Error2" outputs here.

15. **Add Slack node "Send a message"**  
    - Credentials: Slack OAuth2 with appropriate chat permissions.  
    - Set text to the output of "Attendee Research Agent".  
    - Select Slack user by ID to send the message to.  
    - Connect "Attendee Research Agent" output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                 | Context or Link                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow builds an AI meeting assistant that sends information-dense pre-meeting notifications for upcoming meetings.                                                                                   | Provided in workflow's main sticky note.                                                                             |
| Google Cloud setup requires enabling Gmail API and Google Calendar API, and creating OAuth2 credentials.                                                                                                      | [Google Cloud Console Credentials](https://console.cloud.google.com/apis/credentials)                                 |
| OpenAI credentials must be created and connected as per n8n documentation.                                                                                                                                   | [OpenAI Credential Setup](https://docs.n8n.io/integrations/builtin/credentials/openai/)                               |
| Slack credentials require OAuth2 with chat permissions.                                                                                                                                                        | [Slack Credential Setup](https://docs.n8n.io/integrations/builtin/credentials/slack/)                                 |
| Information Extractor node uses AI to parse meeting invite details, simplifying complex extraction logic.                                                                                                     | [Information Extractor Documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.information-extractor/) |
| Slack can be swapped for other notification channels such as Twilio, Telegram, or WhatsApp by replacing the Slack node accordingly.                                                                            | Mentioned in sticky note near Slack node.                                                                             |

---

This documentation fully describes the workflow logic, node configurations, data flows, failure considerations, and setup instructions to enable reproduction or modification by users or automation agents.