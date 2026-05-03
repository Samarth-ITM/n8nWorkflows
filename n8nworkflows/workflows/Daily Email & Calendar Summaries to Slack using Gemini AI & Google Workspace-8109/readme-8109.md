Daily Email & Calendar Summaries to Slack using Gemini AI & Google Workspace

https://n8nworkflows.xyz/workflows/daily-email---calendar-summaries-to-slack-using-gemini-ai---google-workspace-8109


# Daily Email & Calendar Summaries to Slack using Gemini AI & Google Workspace

### 1. Workflow Overview

This workflow automates the process of generating daily summaries from a user's unread emails and calendar events and sends these summaries as a structured Slack message every morning. The primary use case is to provide a concise, AI-generated digest of important emails and upcoming calendar events, enhancing daily team communication and personal productivity.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Scheduling:** Triggers the workflow at a fixed time daily.
- **1.2 Data Retrieval:** Fetches unread emails from Gmail (past 7 days), retrieves filtering criteria from Google Sheets, and collects calendar events for the current day.
- **1.3 Data Filtering:** Filters emails against user-configured names, email addresses, and subjects from the Google Sheet.
- **1.4 AI Processing:** Uses Google Gemini AI to summarize emails and calendar events separately.
- **1.5 Data Restructuring:** Transforms AI summaries into Slack-compatible message blocks.
- **1.6 Message Assembly & Delivery:** Merges email and event summaries and sends the combined message to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Scheduling

- **Overview:**  
  This block initiates the workflow daily at 8 AM using a time-based trigger.

- **Nodes Involved:**  
  - Cron

- **Node Details:**

  - **Cron**  
    - Type: Time trigger node  
    - Configuration: Set to trigger once daily at 08:00 (8 AM).  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Get weekly unread emails" node  
    - Edge Cases: Potential issues if server time zone differs from user expectation; ensure n8n instance time zone matches intended schedule.

#### 2.2 Data Retrieval

- **Overview:**  
  Retrieves raw data required for summaries: unread emails from the last 7 days, filtering parameters from a Google Sheet, and today's calendar events.

- **Nodes Involved:**  
  - Get weekly unread emails (Gmail)  
  - Get Names, emails and subject (Google Sheets)  
  - Get many events in Google Calendar

- **Node Details:**

  - **Get weekly unread emails**  
    - Type: Gmail  
    - Configuration: Fetches up to 100 unread emails labeled INBOX, received within the last 7 days.  
    - Inputs: Triggered by Cron node  
    - Outputs: Connected to "Get Names, emails and subject"  
    - Edge Cases: Gmail API rate limits, authentication errors, or no unread emails found.

  - **Get Names, emails and subject**  
    - Type: Google Sheets  
    - Configuration: Reads a Google Sheet containing columns for Name, Email, and Subject used for filtering emails. Document ID and Sheet Name must be configured.  
    - Inputs: Triggered after "Get weekly unread emails"  
    - Outputs: Connects to "Restructure the data from spread sheet"  
    - Edge Cases: Errors if sheet or document ID is incorrect, or if sheet is empty.

  - **Get many events in Google Calendar**  
    - Type: Google Calendar Tool  
    - Configuration: Retrieves all calendar events for the current day (from 00:00 to 23:59) using dynamic dates. Calendar ID must be configured.  
    - Inputs: Triggered directly by AI Agent via ai_tool connection  
    - Outputs: Connects into AI processing block  
    - Edge Cases: API quota limits, authentication issues, or empty calendar events.

#### 2.3 Data Filtering

- **Overview:**  
  Filters the retrieved unread emails based on criteria extracted from the Google Sheet (Name, Email, Subject), allowing focus only on relevant emails.

- **Nodes Involved:**  
  - Restructure the data from spread sheet (Code)  
  - Filter the emails (Code)  

- **Node Details:**

  - **Restructure the data from spread sheet**  
    - Type: Code  
    - Configuration: Reads all rows from Google Sheets, extracts non-empty emails, names, and subjects into separate arrays for filtering.  
    - Inputs: From Google Sheets node  
    - Outputs: Connects to "Filter the emails"  
    - Edge Cases: Empty or malformed data rows; code assumes columns named exactly "Email", "Name", "Subject".

  - **Filter the emails**  
    - Type: Code  
    - Configuration: Filters the list of unread emails to only those matching any of the emails, names, or subjects from the spreadsheet lists.  
    - Inputs: Receives both raw emails (from "Get weekly unread emails") and filtered arrays from restructuring node.  
    - Outputs: Connects to "AI Agent1" for email summarization  
    - Edge Cases: If no emails match, downstream nodes may receive empty input; code assumes email sender info is in `from.value[0]`.

#### 2.4 AI Processing

- **Overview:**  
  Applies AI summarization separately to filtered emails and calendar events using Gemini AI and two LangChain AI Agent nodes.

- **Nodes Involved:**  
  - AI Agent1 (email summarization)  
  - AI Agent (calendar summarization)  
  - Google Gemini Chat Model (AI language model node)  

- **Node Details:**

  - **AI Agent1**  
    - Type: LangChain AI Agent  
    - Configuration: Summarizes each email in 3 sentences including sender name, subject, and brief content summary; outputs JSON-formatted summary suitable for parsing.  
    - Inputs: Filtered emails from "Filter the emails"  
    - Outputs: Connects to "Restructure the code from the AI agent" (email)  
    - Prompt includes detailed instructions and output formatting constraints.  
    - Edge Cases: AI model failures, malformed input data, or JSON parsing errors downstream.

  - **AI Agent**  
    - Type: LangChain AI Agent  
    - Configuration: Summarizes calendar events with a specified format (date/time, event title, notes), max 5 sentences, no markdown, with a heading prefix and suffix star.  
    - Inputs: Receives calendar events from "Get many events in Google Calendar" via AI tool connection  
    - Outputs: Connects to "Restructure the code from AI agent" (events)  
    - Edge Cases: No calendar events leads to empty or minimal output; AI model errors.

  - **Google Gemini Chat Model**  
    - Type: AI Language Model (Google Gemini)  
    - Configuration: Serves as a shared language model engine for both AI Agent and AI Agent1 nodes. Proper credentials and quota are required.  
    - Inputs: From both AI Agent nodes for processing prompts  
    - Outputs: Back to respective AI Agent nodes.  
    - Edge Cases: Authentication, API limits, or network errors.

#### 2.5 Data Restructuring

- **Overview:**  
  Transforms AI-generated summaries into Slack Block Kit message format to prepare for Slack delivery.

- **Nodes Involved:**  
  - Restructure the code from the AI agent (email summaries)  
  - Restructure the code from AI agent (calendar summaries)  
  - Append mails and events (merge)  
  - Restructure the code into slack block (final assembly)

- **Node Details:**

  - **Restructure the code from the AI agent (email summaries)**  
    - Type: Code  
    - Configuration: Parses AI output JSON strings, removes markdown backticks, builds Slack message blocks with sender, subject, and summary fields.  
    - Inputs: From AI Agent1  
    - Outputs: Connects to "Append mails and events " node (input 0)  
    - Edge Cases: JSON parsing errors if AI output malformed.

  - **Restructure the code from AI agent (calendar summaries)**  
    - Type: Code  
    - Configuration: Converts calendar AI output text into Slack blocks with dividers and sections.  
    - Inputs: From AI Agent  
    - Outputs: Connects to "Append mails and events " node (input 1)  
    - Edge Cases: Empty or null AI output.

  - **Append mails and events**  
    - Type: Merge  
    - Configuration: Merges incoming Slack blocks from emails and calendar events into a single stream for final processing.  
    - Inputs: Receives email blocks on input 0 and calendar blocks on input 1  
    - Outputs: Connects to "Restructure the code into slack block"  
    - Edge Cases: If one input is missing, output may be incomplete.

  - **Restructure the code into slack block**  
    - Type: Code  
    - Configuration: Flattens all Slack blocks from previous merge into one combined JSON object for Slack API consumption.  
    - Inputs: From "Append mails and events "  
    - Outputs: Connects to "Send a message"  
    - Edge Cases: Null or empty blocks array.

#### 2.6 Message Assembly & Delivery

- **Overview:**  
  Sends the assembled Slack message blocks to a configured Slack channel using OAuth2 authentication.

- **Nodes Involved:**  
  - Send a message (Slack)

- **Node Details:**

  - **Send a message**  
    - Type: Slack  
    - Configuration: Sends a block message to a Slack channel specified by channel ID. Uses OAuth2 credentials.  
    - Inputs: Slack blocks from "Restructure the code into slack block"  
    - Outputs: None (terminal node)  
    - Edge Cases: Slack API authentication errors, invalid channel ID, message formatting errors.

---

### 3. Summary Table

| Node Name                          | Node Type                              | Functional Role                                    | Input Node(s)                       | Output Node(s)                      | Sticky Note                                                                                                                |
|-----------------------------------|--------------------------------------|---------------------------------------------------|-----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Cron                              | Cron Trigger                         | Triggers workflow daily at 8 AM                    | None                              | Get weekly unread emails           |                                                                                                                            |
| Get weekly unread emails           | Gmail                               | Retrieves unread emails from last 7 days           | Cron                              | Get Names, emails and subject      | ## Get the unread emails for the past one week                                                                              |
| Get Names, emails and subject      | Google Sheets                       | Reads filtering criteria (Name, Email, Subject)    | Get weekly unread emails           | Restructure the data from spread sheet |                                                                                                                            |
| Restructure the data from spread sheet | Code                              | Extracts emails, names, subjects into arrays       | Get Names, emails and subject      | Filter the emails                 | ## Filter out emails based on Name, Email, Subject Users can configure the name, email and subject in the Excel sheet.         |
| Filter the emails                  | Code                               | Filters unread emails based on spreadsheet criteria | Restructure the data from spread sheet, Get weekly unread emails | AI Agent1                        |                                                                                                                            |
| AI Agent1                        | LangChain AI Agent                 | Summarizes filtered emails in 3 sentences JSON     | Filter the emails                  | Restructure the code from the AI agent | ## Summarizing emails Emails are summarized using an AI agent. The agent reads each email individually, generates a concise summary with Gemini AI, and restructures the output to seamlessly merge with event data. |
| Restructure the code from the AI agent | Code                              | Parses AI JSON email summaries into Slack blocks   | AI Agent1                        | Append mails and events           |                                                                                                                            |
| Get many events in Google Calendar | Google Calendar Tool               | Retrieves calendar events for current day           | AI Agent (ai_tool connection)     | AI Agent                         | ## Summarizing Events Read Google calendar and get events for the data and summarize it and restructure t                    |
| AI Agent                         | LangChain AI Agent                 | Summarizes calendar events                           | Get many events in Google Calendar | Restructure the code from AI agent |                                                                                                                            |
| Restructure the code from AI agent | Code                              | Parses AI calendar summaries into Slack blocks      | AI Agent                         | Append mails and events           |                                                                                                                            |
| Append mails and events           | Merge                              | Merges email and calendar Slack blocks              | Restructure the code from the AI agent (email), Restructure the code from AI agent (calendar) | Restructure the code into slack block | ## Send the event summary and Email summary to slack                                                                        |
| Restructure the code into slack block | Code                              | Flattens merged blocks into one Slack message object| Append mails and events           | Send a message                   |                                                                                                                            |
| Send a message                   | Slack                              | Sends the combined Slack message to configured channel | Restructure the code into slack block | None                            |                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron node**  
   - Type: Cron  
   - Set trigger time: 08:00 (daily)  
   - No credentials needed

2. **Add a Gmail node** ("Get weekly unread emails")  
   - Operation: Get All  
   - Filter: LabelIds = ["INBOX"], ReadStatus = "unread"  
   - ReceivedAfter: Expression `{{$today.minus({ days: 7 }).toISODate()}}`  
   - Limit: 100  
   - Connect input from Cron node  
   - Configure Gmail OAuth2 credentials

3. **Add a Google Sheets node** ("Get Names, emails and subject")  
   - Operation: Read Rows (List mode)  
   - Document ID: Set to your Google Sheets document containing Name, Email, Subject columns  
   - Sheet Name: Set accordingly  
   - Connect input from "Get weekly unread emails"  
   - Configure Google Sheets OAuth2 credentials

4. **Add a Code node** ("Restructure the data from spread sheet")  
   - Paste JS code to extract arrays of emails, names, and subjects from the sheet rows  
   - Connect input from Google Sheets node  
   - Set Execute Once: true

5. **Add a Code node** ("Filter the emails")  
   - Paste JS code that filters the unread emails from Gmail using the arrays from previous node  
   - Connect inputs from both "Restructure the data from spread sheet" and "Get weekly unread emails"  
   - Output filtered emails only

6. **Add a LangChain AI Agent node** ("AI Agent1") for email summarization  
   - Prompt: Instruct to summarize each email in 3 sentences including sender, subject, and brief summary  
   - Input: Connect from "Filter the emails"  
   - Configure to use Google Gemini Chat Model as language model  
   - Set prompt type: define

7. **Add a Code node** ("Restructure the code from the AI agent") for emails  
   - Paste JS code that parses AI JSON output and creates Slack blocks with sender, subject, and summary  
   - Connect input from AI Agent1

8. **Add a Google Calendar Tool node** ("Get many events in Google Calendar")  
   - Operation: getAll  
   - TimeMin: Expression `{{$today.toISO()}}`  
   - TimeMax: Expression `{{$today.plus({days: 1}).toISO()}}`  
   - Calendar ID: Set your calendar ID  
   - Connect input from AI Agent via ai_tool connection (or alternatively from Cron if no direct AI Agent trigger)  
   - Configure Google Calendar OAuth2 credentials

9. **Add a LangChain AI Agent node** ("AI Agent") for calendar summarization  
   - Prompt: Summarize calendar events with specified format and rules  
   - Connect input from Google Calendar node (ai_tool input)  
   - Use Google Gemini Chat Model as language model

10. **Add a Code node** ("Restructure the code from AI agent") for calendar events  
    - Paste JS code to transform AI calendar summary text into Slack blocks  
    - Connect input from AI Agent

11. **Add a Merge node** ("Append mails and events")  
    - Mode: Append or Merge (inputs 0 and 1)  
    - Connect email Slack blocks from "Restructure the code from the AI agent" (email) as input 0  
    - Connect calendar Slack blocks from "Restructure the code from AI agent" (calendar) as input 1

12. **Add a Code node** ("Restructure the code into slack block")  
    - Paste JS code to flatten merged Slack blocks into a single JSON object  
    - Connect input from "Append mails and events"

13. **Add a Slack node** ("Send a message")  
    - Message Type: Block  
    - Channel ID: Set the target Slack channel ID  
    - Blocks UI: `={{ '{ "blocks": ' + JSON.stringify($json.blocks) + ' }' }}`  
    - Authentication: OAuth2 with Slack API credentials  
    - Connect input from "Restructure the code into slack block"

14. **Connect all nodes following the logical order above.**

15. **Configure credentials:**  
    - Gmail OAuth2 for Gmail node  
    - Google Sheets OAuth2 for Sheets node  
    - Google Calendar OAuth2 for Calendar node  
    - Google Gemini AI credentials for LangChain nodes  
    - Slack OAuth2 app token for Slack node

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow reads unread emails from last 7 days, filters based on user-configured criteria, summarizes emails and calendar events using Gemini AI, and posts a combined summary to Slack daily at 8 AM. Requires setting up Google Sheets, Gmail, Google Calendar, Gemini AI, and Slack credentials.                                                        | Project description and usage instructions                                                       |
| Users must create a Google Sheet with columns **Name**, **Email**, **Subject** to specify filtering parameters. Only emails matching these criteria are summarized and sent.                                                                                                                                      | Filtering configuration                                                                           |
| Gemini AI (Google Gemini Chat Model) is used as the language model. Ensure valid API access and quota for n8n LangChain nodes.                                                                                                                                                                                  | AI integration details                                                                            |
| Slack messages use Block Kit JSON format for rich, structured notifications. The workflow parses AI JSON outputs into Slack blocks with dividers and sections.                                                                                                                                                   | Slack message formatting                                                                         |
| Video tutorial and detailed setup instructions can be found at [n8n blog post](https://n8n.io/blog/daily-email-calendar-summary-slack-gemini-ai-google-workspace) (example placeholder link).                                                                                                                    | External resources (replace with actual link if available)                                       |
| Make sure the n8n instance time zone matches the intended trigger time to avoid scheduling issues.                                                                                                                                                                                                              | Time zone consideration                                                                          |
| Potential failure points include API authentication errors, rate limits, AI response parsing errors, and empty data sets. Implement error handling and monitoring accordingly.                                                                                                                                   | Operational considerations                                                                       |

---

**Disclaimer:** The provided text originates solely from an n8n automated workflow. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.