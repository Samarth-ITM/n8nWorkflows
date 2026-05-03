Multi-Channel Feedback to Jira Pipeline with AI Analysis & Notion Reporting

https://n8nworkflows.xyz/workflows/multi-channel-feedback-to-jira-pipeline-with-ai-analysis---notion-reporting-10896


# Multi-Channel Feedback to Jira Pipeline with AI Analysis & Notion Reporting

---
### 1. Workflow Overview

This workflow automates the collection, analysis, prioritization, and reporting of multi-channel user feedback for product management. It consolidates feedback from three entry points: Telegram messages, Google Forms/Sheets, and Gmail emails. The workflow normalizes incoming data, enriches it with AI-powered categorization and scoring, automatically creates Jira tickets with calculated priorities, logs analytics, and generates monthly reports published on Notion and emailed to stakeholders.

**Logical Blocks:**

- **1.1 Input Reception:** Capture feedback from Telegram, Google Forms (Sheets), and Gmail triggers.
- **1.2 Data Normalization:** Unify heterogeneous feedback data into a standardized JSON format.
- **1.3 AI Processing & Enrichment:** Use OpenAI models to analyze feedback, categorize, tag, score urgency, and summarize.
- **1.4 Priority Calculation & Jira Ticket Creation:** Compute custom priority scores and create Jira issues accordingly.
- **1.5 User Notification and Logging:** Notify the user (especially Telegram), log data into Google Sheets, and send confirmation emails.
- **1.6 Notion Documentation:** Create detailed pages in Notion for each feedback/ticket.
- **1.7 Monthly Reporting:** Aggregate monthly feedback statistics, generate AI-driven strategic insights, publish reports to Notion, log in Google Sheets, and email stakeholders.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for new feedback from three sources: Telegram messages, Google Forms submissions, and Gmail emails labeled accordingly.

**Nodes Involved:**  
- Telegram Trigger  
- Google Form Trigger (Google Sheets Trigger configured for form responses)  
- Gmail Trigger  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for new Telegram messages (updates: "message")  
  - Configuration: Uses a webhook with a specific ID, triggers on incoming messages  
  - Inputs: External Telegram messages  
  - Outputs: Raw Telegram message JSON  
  - Edge Cases: Possible webhook misconfiguration, message format changes, Telegram API downtime

- **Google Form Trigger**  
  - Type: Google Sheets Trigger  
  - Role: Polls for new rows added to the Google Sheet linked to the form responses every minute  
  - Configuration: Poll mode every minute, sheet name and document ID linked to the form's response sheet  
  - Inputs: New Google Form submissions  
  - Outputs: Google Sheet row JSON data  
  - Edge Cases: Google API rate limits, sheet renaming, form structure changes

- **Gmail Trigger**  
  - Type: Gmail Trigger  
  - Role: Polls Gmail every hour for new emails from a specific sender and labeled with a custom label  
  - Configuration: Filters by sender email and label ID, hourly polling  
  - Inputs: Incoming emails matching filters  
  - Outputs: Email metadata and content JSON  
  - Edge Cases: Gmail API quota limits, email format variants, label misconfiguration

---

#### 2.2 Data Normalization

**Overview:**  
Converts diverse input formats from Telegram, Gmail, and Google Form into a unified JSON structure that standardizes user info, content, metadata, and timestamps.

**Nodes Involved:**  
- data normalizer (Code node)  

**Node Details:**

- **data normalizer**  
  - Type: Code  
  - Role: Determines the source channel by inspecting input JSON and normalizes fields accordingly  
  - Configuration: JavaScript code that checks for Telegram message structure, Gmail email fields, or Google Form response columns and creates a consistent JSON object with fields like id, source, user info, content, and metadata  
  - Key Expressions: Checks for presence of keys like `message`, `threadId`, or form fields; constructs normalized object with consistent keys  
  - Inputs: Raw data from triggers (Telegram, Gmail, Google Form)  
  - Outputs: Normalized standardized JSON for downstream nodes  
  - Edge Cases: Unknown or malformed input formats, missing fields, timing inaccuracies in timestamps

---

#### 2.3 AI Processing & Enrichment

**Overview:**  
Uses OpenAI models to analyze normalized feedback, categorize it, assign tags, assess sentiment, pain level, business impact, and implementation effort, and generate a concise summary and description.

**Nodes Involved:**  
- instant reply (Telegram message)  
- user enrichment (Code)  
- Message a model (OpenAI GPT-4o-mini)  
- parse json (Code)  

**Node Details:**

- **instant reply**  
  - Type: Telegram  
  - Role: Sends immediate acknowledgment message to Telegram users confirming receipt and analysis start  
  - Configuration: Sends a fixed MarkdownV2 text to the chatId extracted from normalized data  
  - Inputs: Normalized data from data normalizer  
  - Outputs: Confirmation message sent  
  - Edge Cases: Invalid chatId, Telegram API errors, formatting issues in MarkdownV2

- **user enrichment**  
  - Type: Code  
  - Role: Enhances user data by mapping known emails to user tiers (enterprise, pro, free) and assigns weighted tier values for priority calculation  
  - Configuration: Email-to-tier mapping in code and default tiers; assigns tier weight for scoring  
  - Inputs: Normalized user info from instant reply output  
  - Outputs: Enhanced JSON with tier and tier_weight fields  
  - Edge Cases: Unknown emails not in mapping, missing email fields

- **Message a model**  
  - Type: OpenAI (LangChain node)  
  - Role: Analyzes feedback text and user metadata using GPT-4o-mini to produce structured JSON categorizing feedback and scoring attributes  
  - Configuration: Prompt instructs to return JSON with category, tags, sentiment, pain_level, business_impact, implementation_effort, summary, description, acceptance_criteria; context includes raw text, user tier, source, user hints  
  - Inputs: Enriched user and normalized feedback JSON  
  - Outputs: AI JSON analysis response embedded in JSON  
  - Edge Cases: API rate limits, JSON parse errors if model output malformed, network issues

- **parse json**  
  - Type: Code  
  - Role: Parses the raw AI text response from the model, stripping markdown code blocks and handling parsing errors gracefully (fallback to manual review structure)  
  - Configuration: JavaScript parsing with try/catch, fallback JSON with default values on error  
  - Inputs: Raw AI response text  
  - Outputs: Parsed structured JSON with AI analysis fields  
  - Edge Cases: Malformed JSON, empty or missing AI content

---

#### 2.4 Priority Calculation & Jira Ticket Creation

**Overview:**  
Calculates a custom priority score (RICE method weighted by user tier), assigns Jira priority names and IDs, and creates a Jira issue with the enriched feedback summary and description.

**Nodes Involved:**  
- priority calculator (Code)  
- Create an issue (Jira node)  
- post processing (Code)  
- If (conditional check for Telegram source)  

**Node Details:**

- **priority calculator**  
  - Type: Code  
  - Role: Computes RICE score based on AI pain, business impact, implementation effort, and user tier weight; maps score to priority level (P0-P3) and Jira priority name and ID  
  - Configuration: Uses weighted formulas and thresholds; returns detailed scoring object  
  - Inputs: JSON enriched with AI analysis and user tier weight  
  - Outputs: JSON with scoring results and Jira priority info  
  - Edge Cases: Missing AI scores, zero effort (division by zero risk), unknown tiers defaulting to free

- **Create an issue**  
  - Type: Jira  
  - Role: Creates a Jira Story issue in a specified project with priority, summary, and description from analysis  
  - Configuration: Uses project ID, issue type ID, summary with priority tag, Jira priority ID, and description from AI analysis  
  - Inputs: JSON from priority calculator and parsed AI JSON  
  - Outputs: Jira issue creation response including issue key and URL  
  - Edge Cases: Jira auth failure, invalid project or issue type, API rate limits

- **post processing**  
  - Type: Code  
  - Role: Combines Jira creation response with original feedback data, constructs Jira issue URL and identifiers for downstream use  
  - Configuration: Builds URLs using Jira key, adds Jira metadata to JSON  
  - Inputs: Jira issue creation response and original JSON  
  - Outputs: JSON enriched with Jira metadata  
  - Edge Cases: Missing Jira response, unexpected Jira API changes

- **If**  
  - Type: Conditional (If)  
  - Role: Checks if the feedback source is Telegram to decide whether to send user notification  
  - Configuration: Condition equals 'telegram' on normalized source  
  - Inputs: JSON from post processing  
  - Outputs: Routes to notification or logging branches  
  - Edge Cases: Misclassification of source

---

#### 2.5 User Notification and Logging

**Overview:**  
Sends final notification to Telegram users after Jira ticket creation, logs feedback and Jira data into a Google Sheet for analytics, and sends a confirmation email to product team.

**Nodes Involved:**  
- send result to user (Telegram)  
- log to analytics (Google Sheets)  
- end notification (Gmail)  

**Node Details:**

- **send result to user**  
  - Type: Telegram  
  - Role: Notifies Telegram user with ticket info, AI analysis summary, priority, and tags  
  - Configuration: Sends Markdown text to chatId with dynamic Jira URL and analysis details  
  - Inputs: JSON with Jira info and priority calculation  
  - Outputs: Telegram message sent  
  - Edge Cases: Invalid chatId, API errors

- **log to analytics**  
  - Type: Google Sheets  
  - Role: Appends feedback and Jira metadata as a new row in an analytics Google Sheet for reporting  
  - Configuration: Maps various fields like tags, source, category, Jira URL, priority, sentiment, timestamp, user info, scoring components  
  - Inputs: JSON from previous steps with Jira and scoring data  
  - Outputs: Append confirmation  
  - Edge Cases: Google Sheets API quota, schema mismatch, document or sheet renaming

- **end notification**  
  - Type: Gmail  
  - Role: Sends a thank-you email to a configured address confirming feedback processing completion  
  - Configuration: Fixed subject and message body, recipient email configured  
  - Inputs: JSON from log to analytics  
  - Outputs: Email sent  
  - Edge Cases: Gmail API quota, invalid email, sending failure

---

#### 2.6 Notion Documentation

**Overview:**  
Creates a Notion page for each processed feedback containing AI analysis, Jira ticket link, and detailed description.

**Nodes Involved:**  
- Create a page (Notion)  

**Node Details:**

- **Create a page**  
  - Type: Notion  
  - Role: Creates a new page in a specified Notion workspace with blocks containing the AI message content, Jira ticket link, categories, tags, summary, and acceptance criteria  
  - Configuration: Uses Notion integration token and page ID; builds content blocks dynamically from workflow data  
  - Inputs: JSON including AI analysis and Jira metadata  
  - Outputs: Confirmation and URL of created Notion page  
  - Edge Cases: Notion API limits, invalid token or page ID, content size limits

---

#### 2.7 Monthly Reporting

**Overview:**  
Runs on a monthly schedule to query all feedback data, aggregates statistics, generates AI-driven strategic insights, publishes a Notion report page, logs summary data into Google Sheets, and sends a formatted email report to stakeholders.

**Nodes Involved:**  
- 1 month Trigger (Schedule Trigger)  
- query google sheet (Google Sheets)  
- aggregate monthly stats (Code)  
- Message a model1 (OpenAI GPT-5)  
- parse response (Code)  
- monthly report (Notion)  
- montly report log (Google Sheets)  
- Email reporting (Gmail)  

**Node Details:**

- **1 month Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow every 30 days at a specified hour  
  - Configuration: Interval of 30 days, trigger at 17:00 hours  
  - Inputs: Time-based trigger  
  - Outputs: Initiates monthly report processing

- **query google sheet**  
  - Type: Google Sheets  
  - Role: Retrieves all feedback data rows from the analytics Google Sheet for the reporting period  
  - Configuration: Range A1:S1000, document and sheet ID configured  
  - Inputs: Trigger from schedule  
  - Outputs: Array of feedback records JSON  
  - Edge Cases: Large dataset truncation, API limits

- **aggregate monthly stats**  
  - Type: Code  
  - Role: Filters feedbacks for current month, aggregates counts by category, sentiment, priority, source, user tier; calculates averages and top pain points; generates summary text for Notion  
  - Configuration: JavaScript date filtering and aggregation logic  
  - Inputs: All feedback records from Google Sheets  
  - Outputs: Aggregated statistics object with summary text  
  - Edge Cases: No data for month, malformed timestamps

- **Message a model1**  
  - Type: OpenAI (LangChain node)  
  - Role: Analyzes monthly aggregated stats to generate executive summary, key insights, trends, strategic recommendations, and risk areas as JSON  
  - Configuration: Uses GPT-5 model with a detailed prompt including stats overview and category breakdown  
  - Inputs: Aggregated stats JSON  
  - Outputs: AI-generated strategic report JSON  
  - Edge Cases: API rate limits, JSON parse errors

- **parse response**  
  - Type: Code  
  - Role: Parses AI response JSON from the model for use in Notion reporting and sheet logging  
  - Configuration: Strips markdown code blocks, error handling for JSON parsing  
  - Inputs: Raw AI text response  
  - Outputs: Parsed AI insights JSON  
  - Edge Cases: Malformed AI output

- **monthly report**  
  - Type: Notion  
  - Role: Creates a Notion page summarizing the monthly report using AI executive summary  
  - Configuration: Title includes month and year, content block with executive summary  
  - Inputs: Parsed AI insights JSON  
  - Outputs: Notion page creation confirmation

- **montly report log**  
  - Type: Google Sheets  
  - Role: Appends the monthly report metrics and executive summary to a dedicated Google Sheet for historical record  
  - Configuration: Maps month, counts, scores, top category, and Notion report URL  
  - Inputs: Monthly stats and AI insights  
  - Outputs: Append confirmation

- **Email reporting**  
  - Type: Gmail  
  - Role: Sends a styled HTML email report to stakeholders with key metrics and a link to the Notion report  
  - Configuration: Email subject with month, recipient configured, HTML body with CSS styling and dynamic data insertion  
  - Inputs: Monthly report log data  
  - Outputs: Email sent  
  - Edge Cases: Email client rendering issues, Gmail API limits

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role                                     | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                  |
|---------------------|----------------------------|----------------------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| google form trigger  | Google Sheets Trigger      | Trigger on new Google Form submissions              | -                            | data normalizer             | ## 1) Trigger<br>3 possible triggers: Telegram, Google Sheet, Gmail                         |
| Gmail Trigger        | Gmail Trigger              | Trigger on new labeled Gmail emails                  | -                            | data normalizer             | ## 1) Trigger<br>3 possible triggers: Telegram, Google Sheet, Gmail                         |
| Telegram Trigger     | Telegram Trigger           | Trigger on new Telegram messages                     | -                            | data normalizer             | ## 1) Trigger<br>3 possible triggers: Telegram, Google Sheet, Gmail                         |
| data normalizer      | Code                       | Normalize input data into unified JSON               | google form trigger, Gmail Trigger, Telegram Trigger | instant reply             | ## 2) Requests treated and enriched<br>AI analysis and Telegram confirmation                |
| instant reply       | Telegram                   | Send immediate confirmation to Telegram user         | data normalizer             | user enrichment             | ## 2) Requests treated and enriched<br>AI analysis and Telegram confirmation                |
| user enrichment      | Code                       | Enrich user info with tier and weighted scoring      | instant reply               | Message a model             | ## 2) Requests treated and enriched<br>AI analysis and Telegram confirmation                |
| Message a model      | OpenAI (GPT-4o-mini)       | AI categorization, tagging, scoring, summarization  | user enrichment             | parse json                  | ## 2) Requests treated and enriched<br>AI analysis and Telegram confirmation                |
| parse json          | Code                       | Parse AI JSON output, handle errors                   | Message a model             | priority calculator         | ## 3) Priority calculation and Jira ticket creation                                         |
| priority calculator  | Code                       | Compute RICE priority score and Jira priority mapping | parse json                  | Create an issue             | ## 3) Priority calculation and Jira ticket creation                                         |
| Create an issue      | Jira                       | Create Jira ticket with priority and description     | priority calculator         | post processing            | ## 3) Priority calculation and Jira ticket creation                                         |
| post processing      | Code                       | Append Jira metadata to feedback JSON                 | Create an issue             | If                          | ## 3) Priority calculation and Jira ticket creation                                         |
| If                   | Conditional (If)           | Check if source is Telegram to notify user           | post processing             | send result to user, log to analytics | ## 3) Priority calculation and Jira ticket creation                                         |
| send result to user  | Telegram                   | Send final ticket and analysis notification          | If (true branch)            | log to analytics            | ## 3) Priority calculation and Jira ticket creation                                         |
| log to analytics     | Google Sheets              | Log feedback and Jira info for analytics              | send result to user, If (false branch) | end notification           | ## 3) Priority calculation and Jira ticket creation                                         |
| end notification     | Gmail                      | Send confirmation email to team                       | log to analytics            | Create a page               | ## 3) Priority calculation and Jira ticket creation                                         |
| Create a page       | Notion                     | Create detailed Notion page for feedback and ticket  | end notification            | -                           | ## 3) Priority calculation and Jira ticket creation                                         |
| 1 month Trigger      | Schedule Trigger           | Trigger monthly reporting                             | -                          | query google sheet          | ## 4) Monthly reporting<br>Monthly trigger triggers aggregation and reporting               |
| query google sheet   | Google Sheets              | Retrieve all feedback data for the month              | 1 month Trigger             | aggregate monthly stats     | ## 4) Monthly reporting<br>Monthly trigger triggers aggregation and reporting               |
| aggregate monthly stats | Code                    | Aggregate stats and prepare summary for AI analysis  | query google sheet          | Message a model1            | ## 4) Monthly reporting<br>Monthly trigger triggers aggregation and reporting               |
| Message a model1     | OpenAI (GPT-5)             | Generate strategic insights and recommendations       | aggregate monthly stats     | parse response              | ## 4) Monthly reporting<br>Monthly trigger triggers aggregation and reporting               |
| parse response       | Code                       | Parse AI report JSON output                            | Message a model1            | monthly report              | ## 4) Monthly reporting<br>Monthly trigger triggers aggregation and reporting               |
| monthly report       | Notion                     | Create monthly report page                            | parse response              | montly report log           | ## 4) Monthly reporting<br>Monthly trigger triggers aggregation and reporting               |
| montly report log    | Google Sheets              | Log monthly report metrics                             | monthly report              | Email reporting             | ## 4) Monthly reporting<br>Monthly trigger triggers aggregation and reporting               |
| Email reporting      | Gmail                      | Email formatted monthly report to stakeholders        | montly report log           | -                           | ## 4) Monthly reporting<br>Monthly trigger triggers aggregation and reporting               |

---

### 4. Reproducing the Workflow from Scratch

1. **Set up input triggers:**
   - Create a **Telegram Trigger** node configured with your Telegram bot token and webhook ID to listen for incoming messages.
   - Create a **Google Sheets Trigger** node linked to your Google Form response Sheet, configured to poll every minute for new rows.
   - Create a **Gmail Trigger** node filtering emails by sender and label, polling hourly.

2. **Normalize input data:**
   - Add a **Code** node named `data normalizer`.
   - Implement JavaScript to detect input source (Telegram, Gmail, Google Forms) and output a unified JSON object with fields: id, source, timestamp, user (email, name, tier), content (raw_text, subject), metadata.
   - Connect all three triggers to this node.

3. **Send instant confirmation to Telegram users:**
   - Add a **Telegram** node named `instant reply`.
   - Configure it to send a MarkdownV2 confirmation message like "✅ Feedback received...".
   - Use dynamic expression for `chatId` from normalized data.
   - Connect `data normalizer` to `instant reply`.

4. **User enrichment:**
   - Add a **Code** node named `user enrichment`.
   - Map known emails to tiers ('enterprise', 'pro', 'free') and assign tier weights (2.0, 1.5, 1.0).
   - Pass through existing user data, adding `tier` and `tier_weight`.
   - Connect `instant reply` to `user enrichment`.

5. **AI feedback analysis:**
   - Add an **OpenAI (LangChain)** node named `Message a model`.
   - Use GPT-4o-mini model.
   - Prompt instructs to return JSON with category, tags, sentiment, pain_level, business_impact, implementation_effort, summary, description, acceptance_criteria.
   - Include feedback raw_text, user tier, source, user hints in the prompt.
   - Connect `user enrichment` to `Message a model`.

6. **Parse AI JSON output:**
   - Add a **Code** node named `parse json`.
   - Parse AI text response, removing markdown code blocks.
   - Implement try/catch for JSON parsing; fallback to default object on error.
   - Connect `Message a model` to `parse json`.

7. **Calculate priority:**
   - Add a **Code** node named `priority calculator`.
   - Compute weighted RICE score based on AI pain, impact, effort, and tier_weight.
   - Assign priority P0-P3 based on thresholds and map to Jira priority IDs.
   - Connect `parse json` to `priority calculator`.

8. **Create Jira issue:**
   - Add a **Jira** node named `Create an issue`.
   - Configure with your Jira project ID, issue type ID (e.g., Story).
   - Use expressions for summary with priority tag and AI summary, description from AI analysis.
   - Set Jira priority using computed `jira_priority_id`.
   - Connect `priority calculator` to `Create an issue`.

9. **Post-process Jira response:**
   - Add a **Code** node named `post processing`.
   - Extract Jira key, id, URL, and add to JSON.
   - Connect `Create an issue` to `post processing`.

10. **Conditional notification routing:**
    - Add an **If** node.
    - Condition: if `source` equals "telegram".
    - Connect `post processing` to `If`.

11. **Send final notification to Telegram user:**
    - Add a **Telegram** node named `send result to user`.
    - Send message with ticket link, AI analysis details.
    - Connect If (true) branch to `send result to user`.

12. **Log feedback to Google Sheets:**
    - Add a **Google Sheets** node named `log to analytics`.
    - Configure to append new row with multiple columns: tags, source, category, Jira URL, priority, sentiment, timestamp, user info, scores, feedback ID, Jira ticket key.
    - Connect If (false) branch and `send result to user` to `log to analytics`.

13. **Send confirmation email to team:**
    - Add a **Gmail** node named `end notification`.
    - Configure recipient email, subject, and thank-you message.
    - Connect `log to analytics` to `end notification`.

14. **Create Notion page for feedback:**
    - Add a **Notion** node named `Create a page`.
    - Configure with Notion integration token and parent page ID.
    - Populate content blocks with AI message, Jira ticket info, category, tags, summary, description, acceptance criteria.
    - Connect `end notification` to `Create a page`.

15. **Monthly reporting trigger:**
    - Add a **Schedule Trigger** node named `1 month Trigger` set to trigger every 30 days at 17:00.

16. **Query feedback data from Google Sheets:**
    - Add a **Google Sheets** node named `query google sheet`.
    - Configure to read all rows from analytics sheet.
    - Connect `1 month Trigger` to `query google sheet`.

17. **Aggregate monthly stats:**
    - Add a **Code** node named `aggregate monthly stats`.
    - Filter feedback by current month, aggregate counts by category, sentiment, priority, source, tier.
    - Calculate averages for RICE, pain, impact; find top pain points and feature requests.
    - Generate summary text for Notion.
    - Connect `query google sheet` to `aggregate monthly stats`.

18. **Generate strategic insights via AI:**
    - Add an **OpenAI (LangChain)** node named `Message a model1`.
    - Use GPT-5 model.
    - Prompt instructs to analyze monthly stats and generate JSON with executive summary, key insights, trends, recommendations, and risks.
    - Connect `aggregate monthly stats` to `Message a model1`.

19. **Parse AI insights:**
    - Add a **Code** node named `parse response`.
    - Parse and clean AI JSON report.
    - Connect `Message a model1` to `parse response`.

20. **Create monthly report Notion page:**
    - Add a **Notion** node named `monthly report`.
    - Title with month/year, content block with executive summary.
    - Connect `parse response` to `monthly report`.

21. **Log monthly report to Google Sheets:**
    - Add a **Google Sheets** node named `montly report log`.
    - Append metrics and executive summary to monthly reports sheet.
    - Connect `monthly report` to `montly report log`.

22. **Send monthly report email:**
    - Add a **Gmail** node named `Email reporting`.
    - Configure HTML email with styled metrics summary and link to Notion report.
    - Connect `montly report log` to `Email reporting`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow consolidates product feedback from Telegram, Google Forms/Sheets, and Gmail, normalizes and enriches it with AI, automatically creates Jira tickets, and generates monthly reports in Notion and via email.      | Overall workflow description                                                                         |
| Setup steps include configuring webhooks for Google Forms and Telegram, adding Google and Gmail credentials, Notion integration token and database ID, Telegram bot token, AI provider API key, and Jira credentials.           | Setup instructions                                                                                   |
| The workflow uses OpenAI GPT-4o-mini for feedback categorization and GPT-5 for monthly strategic insights generation.                                                                                                       | AI model usage                                                                                       |
| Video walkthrough and additional instructions available here: @[youtube](q0Is11oU18Y)                                                                                                                                         | Video walkthrough link                                                                               |
| Sticky notes in the workflow provide high-level summaries of each block and instructions for setup and usage.                                                                                                               | Workflow sticky notes                                                                                |
| Jira URLs and Notion pages are dynamically constructed using API responses and workflow variables to maintain traceability between feedback, tickets, and documentation.                                                      | Integration best practices                                                                           |
| The workflow handles errors gracefully in AI JSON parsing nodes, using fallback JSON to avoid crashes and enable manual review when AI outputs malformed JSON.                                                              | Error handling best practices                                                                        |
| Google Sheets are used both for real-time logging of feedback and monthly analytics aggregation, enabling transparency and offline review of data.                                                                          | Analytics data storage                                                                                |
| The monthly report email uses custom HTML and CSS styling to ensure professional presentation and clear data communication to stakeholders.                                                                                 | Email reporting design                                                                               |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.