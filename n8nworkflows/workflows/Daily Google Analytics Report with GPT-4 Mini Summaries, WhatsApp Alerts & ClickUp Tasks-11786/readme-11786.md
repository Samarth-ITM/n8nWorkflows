Daily Google Analytics Report with GPT-4 Mini Summaries, WhatsApp Alerts & ClickUp Tasks

https://n8nworkflows.xyz/workflows/daily-google-analytics-report-with-gpt-4-mini-summaries--whatsapp-alerts---clickup-tasks-11786


# Daily Google Analytics Report with GPT-4 Mini Summaries, WhatsApp Alerts & ClickUp Tasks

### 1. Workflow Overview

This workflow automates the generation and distribution of a daily web traffic report using Google Analytics data, enriched with AI-powered summaries, notifications via WhatsApp and email, logging in Google Sheets, and task creation in ClickUp. It is designed to support marketing and operations teams by providing clear insights on daily traffic trends, highlighting notable changes, and facilitating timely responses.

**Logical Blocks:**

- **1.1 Data Retrieval:** Fetch Google Analytics metrics for today and yesterday.
- **1.2 Data Normalization:** Convert raw GA data into uniform, typed, and human-readable formats.
- **1.3 Data Combination & Analysis:** Merge data, compute percentage changes, and add trend indicators.
- **1.4 AI Summary Generation:** Use GPT-4 Mini to create a concise daily summary of traffic trends.
- **1.5 Message Formatting & Notification:** Format AI output for WhatsApp and email, then send notifications.
- **1.6 Traffic Evaluation & Logging:** Evaluate traffic volume changes, log data into appropriate Google Sheets, and send conditional alerts.
- **1.7 Task Management:** Create ClickUp tasks to track necessary follow-up actions.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Retrieval

**Overview:**  
This block triggers the workflow daily and fetches Google Analytics data for "today" and "yesterday" with key metrics.

**Nodes Involved:**  
- Schedule Trigger  
- Today's Report  
- Yesterday's Report  

**Node Details:**

- **Schedule Trigger**  
  - *Type:* scheduleTrigger  
  - *Role:* Initiates workflow once per day (default daily interval).  
  - *Config:* Interval set to daily (every 24 hours).  
  - *Connections:* Outputs to "Today's Report" and "Yesterday's Report".  
  - *Edge Cases:* Workflow trigger failures due to scheduler issues or time zone misalignment.

- **Today's Report**  
  - *Type:* googleAnalytics (GA4)  
  - *Role:* Fetches metrics for the current day.  
  - *Config:*  
    - Date Range: today  
    - Metrics: userEngagementDuration, sessionsPerUser, sessions, screenPageViews  
    - Property ID: set to GA4 property "420608804"  
    - Credential: Google Analytics OAuth2  
  - *Connections:* Outputs to "Normalize & Convert Types" node.  
  - *Edge Cases:* OAuth token expiration, API rate limits, empty data if no traffic.

- **Yesterday's Report**  
  - *Type:* googleAnalytics (GA4)  
  - *Role:* Fetches metrics for the previous day.  
  - *Config:*  
    - Date Range: yesterday  
    - Metrics: screenPageViews, userEngagementDuration, sessions, sessionsPerUser  
    - Property ID: same as above  
    - Credential: Google Analytics OAuth2  
  - *Connections:* Outputs to "Normalize & Convert Types1" node.  
  - *Edge Cases:* Same as "Today's Report".

---

#### 1.2 Data Normalization

**Overview:**  
Converts raw GA data into a uniform format by parsing GA4 date strings into ISO date format and converting string metrics to numbers for downstream processing.

**Nodes Involved:**  
- Normalize & Convert Types (for today's data)  
- Normalize & Convert Types1 (for yesterday's data)  

**Node Details:**

- **Normalize & Convert Types / Normalize & Convert Types1**  
  - *Type:* code  
  - *Role:* JS code normalizes data for consistent date format and numeric types.  
  - *Config:*  
    - Parses GA4 date format (YYYYMMDD) to ISO (YYYY-MM-DD).  
    - Converts metrics: totalUsers, userEngagementDuration, sessionsPerUser, sessions, screenPageViews to numbers.  
  - *Input:* Output from respective GA report nodes.  
  - *Output:* Single JSON object with typed fields.  
  - *Connections:* Both connect to "Combine Today & Yesterday" node.  
  - *Edge Cases:* Missing or malformed date fields may yield null dates; non-numeric metric values may cause conversion errors.

---

#### 1.3 Data Combination & Analysis

**Overview:**  
Merges today's and yesterday's normalized data streams, calculates percentage changes for key metrics, and annotates trends with arrows and low-traffic flags.

**Nodes Involved:**  
- Combine Today & Yesterday  
- Calculate Percent Changes  
- Add Trend Arrows & Low-Traffic Handling  

**Node Details:**

- **Combine Today & Yesterday**  
  - *Type:* merge  
  - *Role:* Combines two input streams (today and yesterday) into a single array for comparative analysis.  
  - *Config:* Default append mode.  
  - *Input:* Normalized data nodes.  
  - *Output:* Combined data to "Calculate Percent Changes".  
  - *Edge Cases:* Mismatched or missing inputs may cause calculation errors downstream.

- **Calculate Percent Changes**  
  - *Type:* code  
  - *Role:* Calculates day-over-day percent change for users, sessions, and page views.  
  - *Config:*  
    - Handles zero or missing values gracefully to avoid division by zero.  
    - Returns percent changes rounded to 1 decimal place.  
  - *Input:* Combined data array.  
  - *Output:* JSON object with percent changes and raw user counts.  
  - *Connections:* Outputs to "Add Trend Arrows & Low-Traffic Handling".  
  - *Edge Cases:* Zero values in yesterday data handled by returning 0 or 100% change as appropriate.

- **Add Trend Arrows & Low-Traffic Handling**  
  - *Type:* code  
  - *Role:* Adds arrow emojis indicating direction of change and flags low traffic (today’s users < yesterday’s users).  
  - *Config:*  
    - Uses '⬆️' for positive change, '⬇️' for negative, '===' for no change.  
    - Adds boolean lowTraffic flag for conditional logic later.  
  - *Input:* Output of percent changes.  
  - *Output:* Enhanced JSON with formatted strings and flags.  
  - *Connections:* Outputs to "Generate AI Summary", conditional "If" node, and Google Sheets logging node.  
  - *Edge Cases:* Null or undefined changes default to neutral trend.

---

#### 1.4 AI Summary Generation

**Overview:**  
Generates a professional, concise natural language summary of daily analytics trends using GPT-4 Mini, tailored for communication via WhatsApp or email.

**Nodes Involved:**  
- Generate AI Summary  
- Format Message for WhatsApp / Email  

**Node Details:**

- **Generate AI Summary**  
  - *Type:* openAi (LangChain node)  
  - *Role:* Calls GPT-4.1 Mini model to produce a daily analytics report summary.  
  - *Config:*  
    - Prompt instructs the model to analyze traffic trends, highlight significant changes, provide recommendations, and handle low traffic explicitly.  
    - Input variables include users, session, pageView, todayUser, yesterdayUser, lowTraffic.  
    - Truncation enabled to limit output length.  
  - *Credentials:* OpenAI API key.  
  - *Input:* Enhanced trend data.  
  - *Output:* AI-generated text content.  
  - *Edge Cases:* API rate limits, incomplete input data, or model inference errors.

- **Format Message for WhatsApp / Email**  
  - *Type:* set  
  - *Role:* Extracts the AI-generated text from response arrays to a simple string field for downstream messaging.  
  - *Config:* Assigns `output[0].content[0].text` to a flat field `output[0].content[0].text`.  
  - *Input:* AI summary node output.  
  - *Output:* Formatted text string ready for messaging.  
  - *Edge Cases:* Unexpected AI response structure could cause assignment errors.

---

#### 1.5 Message Formatting & Notification

**Overview:**  
Sends the AI-generated summary via WhatsApp and emails to relevant recipients, based on traffic trends.

**Nodes Involved:**  
- Send message (WhatsApp)  
- Send email to dedicated person  
- Send email to marketing team  

**Node Details:**

- **Send message (WhatsApp)**  
  - *Type:* WhatsApp node (custom)  
  - *Role:* Sends the formatted summary as a WhatsApp message.  
  - *Config:* Uses WhatsApp API integration with webhook ID.  
  - *Input:* Formatted message text.  
  - *Edge Cases:* WhatsApp API rate limits, incorrect phone number or API credentials.

- **Send email to dedicated person**  
  - *Type:* emailSend  
  - *Role:* Sends the report via email to a specific recipient if low traffic detected.  
  - *Config:*  
    - Subject: "Daily Web Analytics Report"  
    - From and To emails set to the dedicated person's address.  
    - Uses SMTP credentials.  
    - Email body: AI-generated content.  
  - *Input:* Formatted message.  
  - *Edge Cases:* SMTP authentication failures, email delivery issues.

- **Send email to marketing team**  
  - *Type:* emailSend  
  - *Role:* Sends a custom HTML email when low traffic is detected, including specific recommendations.  
  - *Config:*  
    - Subject: "Need attention to review the report"  
    - To marketing team email address.  
    - HTML includes placeholders for today's users, sessions, page views, and low traffic warning.  
    - Uses SMTP credentials.  
  - *Input:* Triggered conditionally via "If" node (lowTraffic true).  
  - *Edge Cases:* Same as above.

---

#### 1.6 Traffic Evaluation & Logging

**Overview:**  
Determines if today's traffic is lower than yesterday's and logs the data accordingly into Google Sheets, maintaining separate sheets for high and low traffic. Also logs a daily summary record.

**Nodes Involved:**  
- If (traffic check)  
- Add log to Less Traffic  
- Add log to High Traffic Sheet  
- Append or update row in sheet  

**Node Details:**

- **If**  
  - *Type:* if  
  - *Role:* Checks if `lowTraffic` flag is true (today’s users less than yesterday’s).  
  - *Config:* Boolean condition on `lowTraffic` field.  
  - *Input:* From "Add Trend Arrows & Low-Traffic Handling" node.  
  - *Output:*  
    - True: sends to "Send email to marketing team" and "Add log to Less Traffic"  
    - False: sends to "Add log to High Traffic Sheet"  
  - *Edge Cases:* Missing or malformed `lowTraffic` flag could cause misrouting.

- **Add log to Less Traffic**  
  - *Type:* googleSheets  
  - *Role:* Appends or updates a row in the "Less Traffic Reports" Google Sheet.  
  - *Config:*  
    - Sheet ID and document URL set to specific Google Sheet for low traffic.  
    - Uses service account authentication.  
    - Auto-maps input data fields.  
  - *Input:* Data from "Send email to marketing team" path.  
  - *Edge Cases:* Google Sheets API limits, permission errors.

- **Add log to High Traffic Sheet**  
  - *Type:* googleSheets  
  - *Role:* Similar to above but logs to "High Traffic Reports" sheet.  
  - *Config:* Same as above but different sheet ID.  
  - *Input:* From "If" node false branch.  
  - *Edge Cases:* Same as above.

- **Append or update row in sheet**  
  - *Type:* googleSheets  
  - *Role:* Logs daily metrics into a "Daily Traffic Report" sheet for historical tracking.  
  - *Config:*  
    - Sheet ID and document URL set accordingly.  
    - Auto-maps fields including users, sessions, pageViews, todayUser, yesterdayUser, lowTraffic, todayDate.  
  - *Input:* From "Add Trend Arrows & Low-Traffic Handling" node.  
  - *Output:* Connects to "Create a task".  
  - *Edge Cases:* Same as above.

---

#### 1.7 Task Management

**Overview:**  
Creates a ClickUp task with relevant daily traffic metrics to notify the team for follow-up actions on the reported data.

**Nodes Involved:**  
- Create a task  

**Node Details:**

- **Create a task**  
  - *Type:* clickUp  
  - *Role:* Adds a new task in a specific ClickUp list with traffic metrics as content.  
  - *Config:*  
    - Team, space, list IDs set explicitly.  
    - Task name: "n8n_TestReport" (likely customizable).  
    - Content includes users, sessions, and page views from input data.  
    - Uses OAuth2 credentials for ClickUp API.  
  - *Input:* From Google Sheets logging node.  
  - *Edge Cases:* OAuth token expiration, API rate limits, incorrect list or team IDs.

---

### 3. Summary Table

| Node Name                     | Node Type                  | Functional Role                          | Input Node(s)                      | Output Node(s)                                   | Sticky Note                                                                                                                     |
|-------------------------------|----------------------------|----------------------------------------|----------------------------------|-------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | scheduleTrigger            | Triggers workflow daily                 | -                                | Today's Report, Yesterday's Report              |                                                                                                                                 |
| Today's Report                | googleAnalytics            | Fetch today's GA metrics                | Schedule Trigger                 | Normalize & Convert Types                         | Part of Analytics Report & Normalize block                                                                                      |
| Yesterday's Report            | googleAnalytics            | Fetch yesterday's GA metrics            | Schedule Trigger                 | Normalize & Convert Types1                        | Part of Analytics Report & Normalize block                                                                                      |
| Normalize & Convert Types     | code                      | Normalize today's data                   | Today's Report                  | Combine Today & Yesterday                         | Part of Analytics Report & Normalize block                                                                                      |
| Normalize & Convert Types1    | code                      | Normalize yesterday's data               | Yesterday's Report              | Combine Today & Yesterday                         | Part of Analytics Report & Normalize block                                                                                      |
| Combine Today & Yesterday     | merge                     | Combine normalized data                  | Normalize & Convert Types, Normalize & Convert Types1 | Calculate Percent Changes                   | Combine and Calculation block                                                                                                   |
| Calculate Percent Changes     | code                      | Calculate day-over-day percentage changes | Combine Today & Yesterday       | Add Trend Arrows & Low-Traffic Handling           | Combine and Calculation block                                                                                                   |
| Add Trend Arrows & Low-Traffic Handling | code              | Add trend arrows and low traffic flag    | Calculate Percent Changes       | Generate AI Summary, If, Append or update row in sheet | Combine and Calculation block                                                                                                   |
| Generate AI Summary           | openAi (LangChain)         | Generate AI summary report               | Add Trend Arrows & Low-Traffic Handling | Format Message for WhatsApp / Email             | Generate and format message block                                                                                               |
| Format Message for WhatsApp / Email | set                 | Format AI output for messaging           | Generate AI Summary             | Send email to dedicated person, Send message     | Generate and format message block                                                                                               |
| Send message                 | WhatsApp                   | Send summary via WhatsApp                 | Format Message for WhatsApp / Email | -                                              | Generate and format message block                                                                                               |
| If                          | if                         | Check if traffic is lower today           | Add Trend Arrows & Low-Traffic Handling | Send email to marketing team & Add log to Less Traffic (true branch), Add log to High Traffic Sheet (false branch) | Check report & notify users & logs block                                                                                        |
| Send email to dedicated person | emailSend                | Send email if low traffic detected        | Format Message for WhatsApp / Email | -                                              | Generate and format message block                                                                                               |
| Send email to marketing team | emailSend                  | Send alert email if low traffic            | If (true branch)               | Add log to Less Traffic                            | Check report & notify users & logs block                                                                                        |
| Add log to Less Traffic       | googleSheets              | Log low traffic data                       | Send email to marketing team    | -                                              | Check report & notify users & logs block                                                                                        |
| Add log to High Traffic Sheet | googleSheets              | Log high traffic data                      | If (false branch)              | -                                              | Check report & notify users & logs block                                                                                        |
| Append or update row in sheet | googleSheets              | Log daily metrics                           | Add Trend Arrows & Low-Traffic Handling | Create a task                                    | Create log & create task block                                                                                                  |
| Create a task                | clickUp                    | Create ClickUp task with daily metrics     | Append or update row in sheet   | -                                              | Create log & create task block                                                                                                  |
| Sticky Note                  | stickyNote                 | Workflow overview and description          | -                              | -                                              | Workflow overview, features, credentials                                                                                        |
| Sticky Note1                 | stickyNote                 | Analytics report & normalization notes      | -                              | -                                              | Analytics Report & Normalize block                                                                                              |
| Sticky Note2                 | stickyNote                 | Combine and calculation notes                | -                              | -                                              | Combine and Calculation block                                                                                                   |
| Sticky Note3                 | stickyNote                 | AI summary generation and formatting notes  | -                              | -                                              | Generate and format message block                                                                                               |
| Sticky Note4                 | stickyNote                 | Traffic checking and notification notes      | -                              | -                                              | Check report & notify users & logs block                                                                                        |
| Sticky Note5                 | stickyNote                 | Logging and task creation notes               | -                              | -                                              | Create log & create task block                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger` node:**  
   - Type: scheduleTrigger  
   - Set interval to daily (every 24 hours) to run workflow once a day.

2. **Create `Today's Report` (Google Analytics node):**  
   - Type: Google Analytics (GA4)  
   - Date Range: `today`  
   - Metrics: userEngagementDuration, sessionsPerUser, sessions, screenPageViews  
   - Property ID: set the GA4 property (e.g., "420608804")  
   - Credentials: connect Google Analytics OAuth2 credential.  
   - Connect output from Schedule Trigger.

3. **Create `Yesterday's Report` (Google Analytics node):**  
   - Type: Google Analytics (GA4)  
   - Date Range: `yesterday`  
   - Metrics: screenPageViews, userEngagementDuration, sessions, sessionsPerUser  
   - Property ID: same as above  
   - Credentials: same Google Analytics OAuth2.  
   - Connect output from Schedule Trigger.

4. **Create `Normalize & Convert Types` node (code):**  
   - Type: Code  
   - Paste JS to parse GA date string and convert metrics to numbers for today's data.  
   - Connect output from `Today's Report`.

5. **Create `Normalize & Convert Types1` node (code):**  
   - Same as above but for yesterday's data.  
   - Connect output from `Yesterday's Report`.

6. **Create `Combine Today & Yesterday` node:**  
   - Type: merge  
   - Mode: append inputs  
   - Connect outputs from both normalization nodes.

7. **Create `Calculate Percent Changes` node (code):**  
   - Type: Code  
   - Paste JS to calculate percent changes with safe division and rounding.  
   - Connect output from `Combine Today & Yesterday`.

8. **Create `Add Trend Arrows & Low-Traffic Handling` node (code):**  
   - Type: Code  
   - Paste JS to add trend arrows and boolean lowTraffic flag.  
   - Connect output from `Calculate Percent Changes`.

9. **Create `Generate AI Summary` node (OpenAI LangChain):**  
   - Type: LangChain OpenAI  
   - Model: GPT-4.1-mini  
   - Configure prompt to include instructions for traffic trend analysis and recommendations; inject data variables.  
   - Credentials: OpenAI API key.  
   - Connect output from `Add Trend Arrows & Low-Traffic Handling`.

10. **Create `Format Message for WhatsApp / Email` node (set):**  
    - Type: Set  
    - Assign `output[0].content[0].text` to flat field for messaging.  
    - Connect output from `Generate AI Summary`.

11. **Create `Send message` node (WhatsApp):**  
    - Type: WhatsApp node  
    - Configure WhatsApp API credentials and webhook.  
    - Text body: map from formatted message field.  
    - Connect output from `Format Message for WhatsApp / Email`.

12. **Create `If` node:**  
    - Type: If  
    - Condition: Check if `lowTraffic` field equals `true`.  
    - Connect output from `Add Trend Arrows & Low-Traffic Handling`.

13. **Create `Send email to marketing team` node:**  
    - Type: emailSend  
    - Set SMTP credentials.  
    - To: marketing email address.  
    - Subject: "Need attention to review the report"  
    - Body: custom HTML including today's users, session, page views, and low traffic advisory.  
    - Connect from `If` node True branch.

14. **Create `Send email to dedicated person` node:**  
    - Type: emailSend  
    - Set SMTP credentials.  
    - To: dedicated person’s email.  
    - Subject: "Daily Web Analytics Report"  
    - Body: AI summary text.  
    - Connect output from `Format Message for WhatsApp / Email`.

15. **Create `Add log to Less Traffic` node (Google Sheets):**  
    - Type: Google Sheets  
    - Operation: appendOrUpdate  
    - Sheet: "Less Traffic Reports" (sheet ID and doc URL)  
    - Auth: Google service account credentials.  
    - Connect from `Send email to marketing team`.

16. **Create `Add log to High Traffic Sheet` node (Google Sheets):**  
    - Type: Google Sheets  
    - Operation: appendOrUpdate  
    - Sheet: "High Traffic Reports" (sheet ID and doc URL)  
    - Auth: Google service account.  
    - Connect from `If` node False branch.

17. **Create `Append or update row in sheet` node (Google Sheets):**  
    - Type: Google Sheets  
    - Operation: appendOrUpdate  
    - Sheet: "Daily Traffic Report" (sheet ID and doc URL)  
    - Auth: Google service account.  
    - Connect output from `Add Trend Arrows & Low-Traffic Handling`.

18. **Create `Create a task` node (ClickUp):**  
    - Type: ClickUp  
    - Configure OAuth2 credentials.  
    - Set team, space, and list IDs.  
    - Name task (e.g., "n8n_TestReport")  
    - Content: insert users, sessions, page views.  
    - Connect from `Append or update row in sheet`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                            | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automates daily web analytics reporting using Google Analytics, AI summarization, WhatsApp/email notifications, Google Sheets logging, and ClickUp task creation.                                                                 | Workflow description (Sticky Note)                                                                 |
| Analytics data normalization includes date format conversion (YYYYMMDD to YYYY-MM-DD) and type coercion for safe calculations.                                                                                                         | Sticky Note1                                                                                        |
| Combines data from two days, calculates percentage changes, adds trend arrows, and flags low-traffic cases to adapt alerting logic accordingly.                                                                                        | Sticky Note2                                                                                        |
| AI generates human-friendly summaries from metrics; formatted outputs are distributed via email and WhatsApp.                                                                                                                           | Sticky Note3                                                                                        |
| Traffic evaluation triggers conditional notifications and logs: low traffic triggers alerts and logs to "Less Traffic Reports"; otherwise logs to "High Traffic Reports."                                                                 | Sticky Note4                                                                                        |
| Logs all daily metrics into "Daily Traffic Report" Google Sheet and creates ClickUp tasks for professional follow-up.                                                                                                                  | Sticky Note5                                                                                        |
| Ensure all API credentials (Google Analytics OAuth2, Google Sheets service account, SMTP, OpenAI API, WhatsApp API, and ClickUp OAuth2) are properly configured and have required permissions.                                            | Credentials summary                                                                                |
| WhatsApp API integration requires valid Meta WhatsApp API credentials and webhook setup for message delivery.                                                                                                                          | See WhatsApp node notes                                                                             |
| Google Sheets nodes require proper sheet URLs and sheet IDs; ensure sheets have appropriate headers matching mapped fields.                                                                                                            | Google Sheets node configuration                                                                    |
| GPT-4.1-mini OpenAI model usage requires an OpenAI account with API access; prompt is structured to avoid over-analysis during very low traffic days.                                                                                   | AI node configuration                                                                             |
| For troubleshooting, watch out for API rate limits, expired tokens, and malformed inputs in code nodes.                                                                                                                                | General error considerations                                                                       |

---

*Disclaimer:* The text above is derived exclusively from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.