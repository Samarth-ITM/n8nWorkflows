Weekly OKR Alignment Report with Gmail, Google Calendar, Notion, and GPT-4.1

https://n8nworkflows.xyz/workflows/weekly-okr-alignment-report-with-gmail--google-calendar--notion--and-gpt-4-1-8341


# Weekly OKR Alignment Report with Gmail, Google Calendar, Notion, and GPT-4.1

### 1. Workflow Overview

This workflow automates the generation of a **Weekly OKR Alignment Report** by aggregating data from Gmail emails, Google Calendar events, Notion databases (weekly plans and quarterly OKRs), and analyzing an image of a handwritten weekly plan with GPT-4.1. It systematically collects, filters, transforms, and merges these diverse inputs to produce an HTML report that checks alignment between OKRs and ongoing activities, assesses execution realism, identifies urgent email follow-ups, and suggests next steps.

The logical blocks are:

- **1.1 Scheduled Data Collection**: Triggered weekly, fetches emails, calendar events, and Notion databases (weekly plan and quarterly OKRs).
- **1.2 Data Filtering and Transformation**: Cleans and formats raw inputs (emails, calendar events, Notion entries) into concise, LLM-friendly strings.
- **1.3 Image Analysis of Weekly Plan**: Downloads and analyzes an image of the weekly plan notebook using GPT-4’s image understanding capabilities, structuring tasks by urgency/importance quadrants.
- **1.4 Data Aggregation and Parsing**: Parses the analyzed weekly plan JSON and prepares OKR entries for integration.
- **1.5 Alignment Analysis and Report Generation**: Feeds the aggregated data into GPT-4.1 to generate a comprehensive HTML report.
- **1.6 Report Delivery**: Sends the generated report via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Collection

**Overview:**  
Runs on a weekly schedule, triggering the entire workflow to gather relevant data from Gmail, Google Calendar, and Notion.

**Nodes Involved:**  
- Schedule Trigger  
- Get many messages (Gmail)  
- Get many events (Google Calendar)  
- Get weekly plan database (Notion)  
- Get quarterly OKR database (Notion)

**Node Details:**

- **Schedule Trigger**  
  - Type: Cron-based trigger node.  
  - Configuration: Set to execute at 20:30 (8:30 PM) every Saturday (Asia/Hong_Kong timezone implied).  
  - Input: None (starts workflow).  
  - Output: Triggers downstream nodes.  
  - Failures: Cron misconfiguration or time zone mismatch may cause missed runs.

- **Get many messages (Gmail)**  
  - Type: Gmail node, operation "getAll".  
  - Configuration: Fetch all emails received after 1 week ago from now (`{{$now.minus({ week: 1 })}}`).  
  - Credentials: Gmail OAuth2.  
  - Output: Raw email data including metadata and bodies.  
  - Potential issues: OAuth token expiration, API limits, empty inbox.

- **Get many events (Google Calendar)**  
  - Type: Google Calendar node, operation "getAll".  
  - Configuration: Fetch calendar events from 1 week ago up to 1 month in the future.  
  - Credentials: Google Calendar OAuth2.  
  - Output: Raw event data including summary, time, attendees.  
  - Failures: OAuth token expiration, calendar permission issues.

- **Get weekly plan database (Notion)**  
  - Type: Notion node, operation "getAll" on databasePage resource.  
  - Configuration: Notion database ID for weekly plans.  
  - Credentials: Notion API token.  
  - Output: Raw Notion page data for weekly plan entries.  
  - Failures: API rate limits, missing permissions, invalid database ID.

- **Get quarterly OKR database (Notion)**  
  - Type: Notion node, operation "getAll" on databasePage resource.  
  - Configuration: Notion database ID for quarterly OKRs.  
  - Credentials: Notion API token.  
  - Output: Raw Notion page data for OKRs.  
  - Failures: Same as weekly plan database node.

---

#### 1.2 Data Filtering and Transformation

**Overview:**  
Filters out irrelevant emails, cleans up email and calendar event data, and extracts structured information from Notion databases.

**Nodes Involved:**  
- Gmail Filter (Code)  
- Edit Fields (set)  
- Gmail Code (Code)  
- Edit Fields1 (set)  
- Calendar Code (Code)  
- Filter1 (Filter)  
- Filter2 (Filter)  
- Quarterly Code (Code)

**Node Details:**

- **Gmail Filter**  
  - Type: Code node.  
  - Role: Removes noise emails by excluding promotional, social, spam, trash labels, and receipts.  
  - Key Logic: Checks labels and subject keywords; returns dummy item if no relevant emails found.  
  - Input: Raw emails from "Get many messages".  
  - Output: Filtered emails.  
  - Edge Cases: Empty filtered result handled gracefully; expression errors if labels missing.

- **Edit Fields (Emails)**  
  - Type: Set node.  
  - Role: Simplifies emails to essential fields: sender display name/email, subject, date, body snippet.  
  - Input: Filtered emails.  
  - Output: Cleaned email fields.  
  - Expressions: Uses fallback logic for sender name/email and snippet.  
  - Failures: Missing fields fallback to empty strings.

- **Gmail Code**  
  - Type: Code node.  
  - Role: Converts emails to a joined string with max 2000 characters per email, formatted for LLM input.  
  - Input: Edited email fields.  
  - Output: Single JSON string combining all emails.  
  - Edge Cases: Handles missing body content gracefully.

- **Edit Fields1 (Calendar Events)**  
  - Type: Set node.  
  - Role: Extracts summary, concatenated start/end times, attendees' first email, description.  
  - Input: Raw calendar events from "Get many events".  
  - Output: Structured calendar event fields.  
  - Expressions: Extracts first attendee email or empty.  
  - Failures: Missing attendees or description handled.

- **Calendar Code**  
  - Type: Code node.  
  - Role: Builds a concise LLM-friendly string summary per calendar event with date/time, title, attendees, and brief description (600 char max).  
  - Input: Edited calendar events.  
  - Output: Joined calendar event string.  
  - Edge Cases: Handles missing times or attendees.

- **Filter1 (Weekly Plan)**  
  - Type: Filter node.  
  - Role: Selects weekly plan database pages where end date equals current date in Asia/Hong_Kong timezone.  
  - Input: Raw weekly plan Notion data.  
  - Output: Filtered weekly plan pages.  
  - Edge Cases: Timezone correctness critical; empty results possible.

- **Filter2 (Quarterly OKRs)**  
  - Type: Filter node.  
  - Role: Filters quarterly OKRs whose start/end dates include current date (Asia/Hong_Kong timezone).  
  - Input: Raw quarterly OKR Notion data.  
  - Output: Relevant quarterly OKR pages.  
  - Edge Cases: Date parsing errors; timezone alignment important.

- **Quarterly Code**  
  - Type: Code node.  
  - Role: Formats filtered OKRs into numbered string blocks listing objective, key result, and status.  
  - Input: Filtered quarterly OKRs from Filter2.  
  - Output: Joined OKR string for LLM input.  
  - Edge Cases: Missing OKR properties replaced with placeholders.

---

#### 1.3 Image Analysis of Weekly Plan

**Overview:**  
Downloads the image of the weekly plan, sends it to GPT-4 for quadrant-based task extraction, and parses the returned JSON.

**Nodes Involved:**  
- Filter1 (output to HTTP Request)  
- HTTP Request  
- Analyze image (OpenAI Langchain node)  
- Weekly Code (Code)  
- Merge1 (merge with Quarterly Code output)

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request node.  
  - Role: Downloads the media file (image) of the weekly plan from Notion page property.  
  - Configuration: Uses file response format to get binary data for analysis.  
  - Input: Filtered weekly plan page from Filter1.  
  - Output: Binary image file.  
  - Failures: URL invalid, file not found, network issues.

- **Analyze image**  
  - Type: OpenAI Langchain node (specialized for images).  
  - Role: Sends base64 image to GPT-4 mini model with a prompt to extract tasks by urgency/importance quadrants.  
  - Input: Base64 encoded image from HTTP Request.  
  - Output: JSON object with quadrants containing tasks marked finished or not.  
  - Edge Cases: Image quality affecting OCR, ambiguous quadrants handled with best guess.

- **Weekly Code**  
  - Type: Code node.  
  - Role: Parses the JSON from the GPT output, safely handling various JSON formatting quirks; formats tasks into markdown-like string blocks per quadrant.  
  - Input: JSON content from Analyze image node.  
  - Output: Joined weekly plan task string for LLM input.  
  - Edge Cases: Invalid or unparsable JSON gracefully handled with fallback.

- **Merge1**  
  - Type: Merge node (chooseBranch mode).  
  - Role: Combines weekly plan analysis output and quarterly OKR string for downstream processing.  
  - Input: Outputs from Weekly Code and Quarterly Code.  
  - Output: Single merged data stream.  
  - Failures: Misaligned input order or missing input.

---

#### 1.4 Data Aggregation and Parsing

**Overview:**  
Merges cleaned and formatted email and calendar data, plus combined plan and OKR data, to prepare inputs for final GPT prompt.

**Nodes Involved:**  
- Merge (combines Gmail Code and Calendar Code outputs)  
- Merge1 (combines Weekly Code and Quarterly Code outputs)  
- Merge2 (combines outputs of Merge and Merge1)

**Node Details:**

- **Merge**  
  - Type: Merge node (chooseBranch mode).  
  - Role: Combines emails joined string and calendar joined string into one data stream.  
  - Inputs: Gmail Code and Calendar Code outputs.  
  - Output: Merged data for report generation.

- **Merge1** (previously described)  
  - Merges weekly plan and quarterly OKR strings.

- **Merge2**  
  - Type: Merge node (chooseBranch mode).  
  - Role: Combines the two merged sets above (emails+calendar and weekly+OKRs) into a single input for the final AI model.  
  - Inputs: Merge and Merge1 outputs.  
  - Output: Unified data for report generation GPT node.

---

#### 1.5 Alignment Analysis and Report Generation

**Overview:**  
Uses GPT-4.1 to analyze all aggregated data and generate a comprehensive weekly OKR alignment report in HTML format.

**Nodes Involved:**  
- Message a model1 (OpenAI Langchain node)

**Node Details:**

- **Message a model1**  
  - Type: OpenAI Langchain node with GPT-4.1-mini model.  
  - Role: Receives joined strings of OKRs, calendar events, weekly plan tasks, and emails; applies a detailed prompt instructing the model to:  
    - Check alignment among data sources treating OKRs as the North Star.  
    - Assess execution realism.  
    - Extract urgent email follow-ups.  
    - Suggest next steps.  
  - Input: Outputs from Merge2 node containing all aggregated data.  
  - Output: HTML-formatted weekly OKR alignment report.  
  - Edge Cases: Model response errors, token length limits, malformed inputs.

---

#### 1.6 Report Delivery

**Overview:**  
Sends the generated HTML report to a specified email address.

**Nodes Involved:**  
- Send a message (Gmail)

**Node Details:**

- **Send a message**  
  - Type: Gmail node, operation "send".  
  - Configuration: Sends email to a configured address with subject "Weekly OKR Report" and message body containing the HTML from GPT output.  
  - Credentials: Gmail OAuth2.  
  - Input: HTML report from Message a model1.  
  - Failures: OAuth errors, invalid email address, sending limits.

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                                     | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                                        |
|-------------------------|---------------------------|----------------------------------------------------|--------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger           | Weekly execution trigger                            | None                           | Get many messages, Get many events, Get weekly plan database, Get quarterly okr database | "0 30 20 * * 6" cron expression for weekly run at 20:30 Saturday                                                                 |
| Get many messages       | Gmail                     | Fetch emails from last week                         | Schedule Trigger               | Gmail Filter                   |                                                                                                                                   |
| Gmail Filter            | Code                      | Filter out noise emails (promotions, receipts)    | Get many messages              | Edit Fields                   |                                                                                                                                   |
| Edit Fields             | Set                       | Simplify email fields (from, subject, date, body) | Gmail Filter                  | Gmail Code                    |                                                                                                                                   |
| Gmail Code              | Code                      | Join email info into LLM-friendly string           | Edit Fields                   | Merge                        |                                                                                                                                   |
| Get many events         | Google Calendar           | Fetch calendar events from last week to next month| Schedule Trigger              | Edit Fields1                  |                                                                                                                                   |
| Edit Fields1            | Set                       | Simplify calendar event fields                      | Get many events               | Calendar Code                 |                                                                                                                                   |
| Calendar Code           | Code                      | Join calendar events into LLM-friendly string      | Edit Fields1                  | Merge                        |                                                                                                                                   |
| Get weekly plan database| Notion                    | Fetch weekly plan pages from Notion                 | Schedule Trigger              | Filter1                      |                                                                                                                                   |
| Filter1                 | Filter                    | Filter weekly plan pages ending today               | Get weekly plan database      | HTTP Request                 | On the last day of each week, the system grabs the weekly plan of that week. {{ $now.setZone('Asia/Hong_Kong').toISODate() }}     |
| HTTP Request            | HTTP Request              | Download weekly plan image file                      | Filter1                      | Analyze image                |                                                                                                                                   |
| Analyze image           | OpenAI Langchain (Image)  | Analyze handwritten weekly plan image                | HTTP Request                 | Weekly Code                  |                                                                                                                                   |
| Weekly Code             | Code                      | Parse and format weekly plan JSON into task strings | Analyze image                | Merge1                      |                                                                                                                                   |
| Get quarterly okr database| Notion                  | Fetch quarterly OKR pages from Notion                | Schedule Trigger              | Filter2                      |                                                                                                                                   |
| Filter2                 | Filter                    | Filter quarterly OKRs active on current date         | Get quarterly okr database    | Quarterly Code               |                                                                                                                                   |
| Quarterly Code          | Code                      | Format quarterly OKRs into numbered string blocks    | Filter2                      | Merge1                      |                                                                                                                                   |
| Merge                   | Merge                     | Merge emails and calendar data                        | Gmail Code, Calendar Code     | Merge2                      |                                                                                                                                   |
| Merge1                  | Merge                     | Merge weekly plan and quarterly OKRs                 | Weekly Code, Quarterly Code   | Merge2                      |                                                                                                                                   |
| Merge2                  | Merge                     | Merge all data streams for GPT input                 | Merge, Merge1                | Message a model1             |                                                                                                                                   |
| Message a model1        | OpenAI Langchain (GPT-4.1)| Generate HTML weekly OKR alignment report            | Merge2                      | Send a message              | Instructions emphasize valid HTML output and treating OKRs as the North Star. See prompt for detailed analysis instructions.      |
| Send a message          | Gmail                     | Send the generated report via email                   | Message a model1             | None                         |                                                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" node:**  
   - Type: Schedule Trigger  
   - Cron expression: `0 30 20 * * 6` (every Saturday at 20:30)  
   - Timezone: Asia/Hong_Kong (ensure correct)  

2. **Create "Get many messages" Gmail node:**  
   - Operation: getAll  
   - Filters: receivedAfter = `={{ $now.minus({ week: 1 }) }}`  
   - Return All: true  
   - Credentials: configure Gmail OAuth2  

3. **Create "Gmail Filter" Code node:**  
   - Paste code filtering out emails by labels and subject (exclude promotions, social, spam, receipts)  
   - If empty, return dummy email with body "No human-relevant emails this week."  

4. **Create "Edit Fields" Set node:**  
   - Fields:  
     - from_display = `{{$json.From.name || $json.From.email || $json.From}}`  
     - subject = `{{$json.Subject}}`  
     - date = `{{$json.internalDate}}`  
     - body = `{{$json.textPlain || $json.snippet}}`  

5. **Create "Gmail Code" Code node:**  
   - Code combines emails into joined string with max 2000 chars per email, formatted with headers and body  

6. **Create "Get many events" Google Calendar node:**  
   - Operation: getAll  
   - timeMin = `={{ $now.minus({ week: 1 }) }}`  
   - timeMax = `={{ $now.plus({ month: 1 }) }}`  
   - Calendar: your email address  
   - Credentials: Google Calendar OAuth2  

7. **Create "Edit Fields1" Set node:**  
   - Fields:  
     - summary = `{{$json.summary}}`  
     - time = `{{$json.start.dateTime}}{{$json.end.dateTime}}`  
     - attendees = `{{$json.attendees[0].email}}`  
     - description = `{{$json.description}}`  

8. **Create "Calendar Code" Code node:**  
   - Code that joins calendar events into LLM-friendly formatted strings with date/time, title, attendees, and truncated description  

9. **Create "Get weekly plan database" Notion node:**  
   - Operation: getAll, resource: databasePage  
   - Database ID: your weekly plan Notion database ID  
   - Credentials: Notion API  

10. **Create "Filter1" Filter node:**  
    - Condition: property_date.end equals current date in Asia/Hong_Kong timezone `={{ $now.setZone('Asia/Hong_Kong').toISODate() }}`  

11. **Create "HTTP Request" node:**  
    - URL: `={{ $json.property_files_media[0] }}` (media file URL from filtered weekly plan)  
    - Response format: file (binary)  

12. **Create "Analyze image" OpenAI Langchain node:**  
    - Input type: base64 (from binary)  
    - Model: GPT-4 mini (or appropriate image analysis capable model)  
    - Prompt: Provided detailed quadrant analysis JSON prompt  
    - Credentials: OpenAI API  

13. **Create "Weekly Code" Code node:**  
    - Parses JSON from Analyze image output (handles code fences, double encoded strings, errors)  
    - Formats tasks into markdown-style strings by quadrant  

14. **Create "Get quarterly okr database" Notion node:**  
    - Operation: getAll, resource: databasePage  
    - Database ID: your quarterly OKR Notion database ID  
    - Credentials: Notion API  

15. **Create "Filter2" Filter node:**  
    - Conditions: property_date.start ≤ today ≤ property_date.end (Asia/Hong_Kong timezone)  

16. **Create "Quarterly Code" Code node:**  
    - Formats quarterly OKRs into numbered blocks with Objective, Key Result, and Status  

17. **Create "Merge" node:**  
    - Input 1: Gmail Code output  
    - Input 2: Calendar Code output  
    - Mode: chooseBranch (output empty)  

18. **Create "Merge1" node:**  
    - Input 1: Weekly Code output  
    - Input 2: Quarterly Code output  
    - Mode: chooseBranch (output empty)  

19. **Create "Merge2" node:**  
    - Input 1: Merge output (emails + calendar)  
    - Input 2: Merge1 output (weekly plan + OKRs)  
    - Mode: chooseBranch (output empty)  

20. **Create "Message a model1" OpenAI Langchain node:**  
    - Model: GPT-4.1-mini  
    - Messages: System prompt and user prompt formatting inputs from merged sources, instructing report generation in HTML  
    - Credentials: OpenAI API  

21. **Create "Send a message" Gmail node:**  
    - Send To: your email address  
    - Subject: "Weekly OKR Report"  
    - Message: `={{$json["message"]["content"]}}` (HTML report from GPT)  
    - Credentials: Gmail OAuth2  

22. **Connect nodes as per the connections:**  
    - Schedule Trigger -> Get many messages, Get many events, Get weekly plan database, Get quarterly okr database  
    - Get many messages -> Gmail Filter -> Edit Fields -> Gmail Code -> Merge (input 1)  
    - Get many events -> Edit Fields1 -> Calendar Code -> Merge (input 2)  
    - Get weekly plan database -> Filter1 -> HTTP Request -> Analyze image -> Weekly Code -> Merge1 (input 1)  
    - Get quarterly okr database -> Filter2 -> Quarterly Code -> Merge1 (input 2)  
    - Merge, Merge1 -> Merge2  
    - Merge2 -> Message a model1 -> Send a message  

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow uses Asia/Hong_Kong timezone for date filtering in Notion databases to align weekly data.       | Important for correct weekly plan and OKR filtering.                                              |
| The image analysis prompt uses quadrant-based task prioritization (urgent/important matrix).                  | Ensures structured extraction of handwritten weekly plan tasks.                                  |
| Final GPT prompt instructs output strictly in valid HTML, avoiding markdown.                                  | Ensures email clients render the report correctly.                                               |
| OpenAI API credentials require access to GPT-4 and GPT-4 mini models, including image input capability.      | Ensure account permissions and quota for multimodal GPT usage.                                   |
| Gmail and Google Calendar nodes use OAuth2 credentials; token refresh and permissions must be maintained.    | Prevents workflow failure due to authentication errors.                                         |
| Notion API access requires database read permissions for both weekly plan and quarterly OKR databases.       | Database IDs must be accurate and up to date.                                                    |
| Code nodes include robust JSON parsing and fallback logic to handle inconsistent LLM outputs gracefully.     | Improves stability and error handling of the workflow.                                          |

---

This documentation enables understanding, reproduction, and modification of the "Weekly OKR Alignment Report with Gmail, Google Calendar, Notion, and GPT-4.1" workflow in n8n.