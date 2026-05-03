Automated Daily Email Analysis & Summary with GPT-4o and Gmail

https://n8nworkflows.xyz/workflows/automated-daily-email-analysis---summary-with-gpt-4o-and-gmail-4893


# Automated Daily Email Analysis & Summary with GPT-4o and Gmail

### 1. Workflow Overview

This workflow, titled **Automated Daily Email Analysis & Summary with GPT-4o and Gmail**, is designed to run once daily and automatically process all emails received during the day. It performs a detailed analysis of the email content using an AI agent powered by GPT-4o and generates a structured, markdown-formatted summary called the **Daily Business Pulse**. This summary focuses on key actionable business insights such as pending follow-ups, tasks, leads, and strategies extracted strictly from the messages without assumptions or fabrications. Finally, it emails the formatted summary back to the user.

**Target Use Cases:**  
- Busy professionals or small business owners who want an automated, concise digest of daily email communications.  
- Teams seeking to quickly identify important tasks and leads from a large volume of messages.  
- Users wanting AI-assisted email triage and summary without manual review.

**Logical Blocks:**

- **1.1 Scheduled Trigger & Date Preparation:** Initiates the workflow daily at a set hour and calculates the date range for email retrieval.  
- **1.2 Email Retrieval:** Fetches all emails received during the specified day from Gmail.  
- **1.3 Data Aggregation & Cleanup:** Aggregates email fields and cleans the email bodies to prepare a text corpus for AI analysis.  
- **1.4 AI Content Analysis:** Uses an AI Agent with a GPT-4o model to analyze the cleaned email text and generate a structured summary.  
- **1.5 Formatting & Delivery:** Converts the AI-generated markdown summary into HTML and sends it via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Date Preparation

- **Overview:** This block triggers the workflow daily at 18:00 (6 PM) and calculates the current day's start (midnight) and next day's start to define the email retrieval window.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Date Transformer

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger (n8n core)  
    - *Role:* Starts workflow at fixed daily time  
    - *Configuration:* Triggers at hour 18 (6 PM UTC) every day  
    - *Inputs:* None  
    - *Outputs:* Triggers Date Transformer  
    - *Edge Cases:* Time zone differences may affect actual trigger time; n8n server time zone setting is important.

  - **Date Transformer**  
    - *Type:* Code (JavaScript) node  
    - *Role:* Computes `today` (midnight start of current day) and `tomorrow` (midnight start of next day) in ISO format with 'Z' suffix for UTC  
    - *Configuration:* Uses JS `Date` object to calculate dates; returns JSON with `today` and `tomorrow` keys  
    - *Expressions:* None, pure JS code  
    - *Inputs:* Trigger data (empty)  
    - *Outputs:* Date range JSON object for Gmail filters  
    - *Edge Cases:* Relies on server time zone; date boundary issues possible if server is not UTC.

---

#### 1.2 Email Retrieval

- **Overview:** Retrieves all emails received between `today` and `tomorrow` (i.e., during the current day) from the connected Gmail account.

- **Nodes Involved:**  
  - Gmail

- **Node Details:**

  - **Gmail**  
    - *Type:* Gmail node (OAuth2 authenticated)  
    - *Role:* Fetches emails matching date filters  
    - *Configuration:*  
      - Operation: `getAll` (fetch all matching emails)  
      - Filters: `receivedAfter` = `{{$json.today}}`, `receivedBefore` = `{{$json.tomorrow}}` (dynamic dates from Date Transformer)  
      - Simple: false (fetches full message data, not simplified)  
      - Credentials: OAuth2 Gmail account  
    - *Input:* Date range JSON from Date Transformer  
    - *Output:* List of email messages with headers and body text  
    - *Edge Cases:*  
      - Gmail API quota limits and rate limiting possible  
      - Messages without text content may appear empty  
      - Authentication token expiration can cause failures  

---

#### 1.3 Data Aggregation & Cleanup

- **Overview:** Aggregates important email fields for all messages, cleans HTML and promotional text from the email bodies, and combines them into a single text block for AI processing.

- **Nodes Involved:**  
  - Aggregate  
  - Email Cleanup (Code)

- **Node Details:**

  - **Aggregate**  
    - *Type:* Aggregate node (n8n core)  
    - *Role:* Collates email `headers.from`, `headers.subject`, and `text` fields from all messages into arrays  
    - *Configuration:* Aggregates three fields into arrays for further processing  
    - *Input:* Emails array from Gmail node  
    - *Output:* Single aggregated JSON with arrays of from, subject, and text  
    - *Edge Cases:* Missing headers or empty fields possible if email malformed

  - **Email Cleanup**  
    - *Type:* Code (JavaScript) node  
    - *Role:* Cleans and formats each email’s body text by removing HTML tags, promotional unsubscribe/view triggers, excess newlines, and trims whitespace  
    - *Configuration:*  
      - Processes each email item  
      - Constructs a formatted string including sender, subject, and cleaned preview  
      - Joins all email previews into one combined text block under `combinedText` key  
    - *Input:* Aggregated email data  
    - *Output:* JSON with combinedText string for AI input  
    - *Edge Cases:*  
      - Emails without `header` or `text` fields handled with default placeholders  
      - Regex replacements may miss some HTML edge cases or unusual unsubscribe formats

---

#### 1.4 AI Content Analysis

- **Overview:** Uses a LangChain AI Agent node, backed by OpenAI GPT-4o-mini, to analyze the combined daily email text and generate a markdown-formatted summary that highlights prioritized business insights.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain Agent node (n8n community)  
    - *Role:* Processes input text using AI to produce structured summary according to a strict prompt  
    - *Configuration:*  
      - System message defines role as "AI Chief of Staff"  
      - Prompt instructs to generate Daily Business Pulse with sections for open loops, next steps, leads, dead conversations, strategy notes, and top tasks  
      - Uses dynamic input `{{ $json.combinedText }}` from previous node  
      - Does not fabricate information; omits empty sections  
    - *Input:* Cleaned combined email text  
    - *Output:* Markdown summary text in `output` field  
    - *Edge Cases:*  
      - AI model availability and latency  
      - Token limits of GPT-4o-mini may truncate very large email sets  
      - Prompt adherence depends on model behavior

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat node  
    - *Role:* Provides GPT-4o-mini language model backend to AI Agent  
    - *Configuration:*  
      - Model: `gpt-4o-mini`  
      - No additional options set  
      - Credentials: OpenAI API key  
    - *Input:* AI Agent requests  
    - *Output:* AI-generated completion for Agent  
    - *Edge Cases:*  
      - API key limits and rate limits  
      - Model version and changes may affect output quality

---

#### 1.5 Formatting & Delivery

- **Overview:** Converts the AI-generated markdown summary into HTML format suitable for email and sends it to a predefined recipient using Gmail.

- **Nodes Involved:**  
  - Format HTML (Code)  
  - Send Message (Gmail)

- **Node Details:**

  - **Format HTML**  
    - *Type:* Code (JavaScript) node  
    - *Role:* Transforms markdown syntax in AI output to basic HTML tags (bold, italics, strikethrough, code, links, images, line breaks)  
    - *Configuration:*  
      - Maps all input items’ `output` markdown  
      - Applies regex replacements for markdown to HTML  
      - Returns single `html` string array for email content  
    - *Input:* Markdown text from AI Agent  
    - *Output:* HTML string for email body  
    - *Edge Cases:*  
      - Limited markdown support; complex markdown features not handled  
      - Inline images and links rely on markdown format correctness

  - **Send Message**  
    - *Type:* Gmail node (OAuth2 authenticated)  
    - *Role:* Sends the final formatted email summary to a fixed recipient email address  
    - *Configuration:*  
      - Send To: `zawagner@gmail.com` (hardcoded recipient)  
      - Subject: `"Here's Your Daily Pulse"`  
      - Message: Uses HTML content from Format HTML node  
      - Credentials: Same Gmail OAuth2 account as before  
    - *Input:* HTML email content  
    - *Output:* Email sent confirmation  
    - *Edge Cases:*  
      - Email sending failures due to connectivity, quota, or auth issues  
      - Hardcoded recipient limits reuse without modification

---

### 3. Summary Table

| Node Name          | Node Type                  | Functional Role                               | Input Node(s)       | Output Node(s)     | Sticky Note                |
|--------------------|----------------------------|----------------------------------------------|---------------------|--------------------|----------------------------|
| Schedule Trigger    | Schedule Trigger            | Initiates workflow daily at 18:00             | None                | Date Transformer    |                            |
| Date Transformer    | Code                       | Calculates today and tomorrow ISO date strings | Schedule Trigger    | Gmail               |                            |
| Gmail              | Gmail                      | Fetches all emails received within date range | Date Transformer    | Aggregate            |                            |
| Aggregate           | Aggregate                  | Aggregates sender, subject, and text fields   | Gmail               | Email Cleanup       |                            |
| Email Cleanup       | Code                       | Cleans and combines email content into text   | Aggregate           | AI Agent            |                            |
| AI Agent            | LangChain Agent            | Analyzes combined text and generates summary  | Email Cleanup       | Format HTML          |                            |
| OpenAI Chat Model   | LangChain OpenAI Chat      | Provides GPT-4o-mini model backend             | AI Agent (lang model) | AI Agent             |                            |
| Format HTML         | Code                       | Converts markdown summary to HTML              | AI Agent            | Send Message         |                            |
| Send Message        | Gmail                      | Sends the HTML email summary to recipient     | Format HTML         | None                 |                            |
| Sticky Note         | Sticky Note                | No content                                     | None                | None                 |                            |
| Sticky Note1        | Sticky Note                | No content                                     | None                | None                 |                            |
| Sticky Note2        | Sticky Note                | No content                                     | None                | None                 |                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger` node:**  
   - Type: Schedule Trigger  
   - Set trigger time: 18:00 daily (UTC or adjust for your timezone)

2. **Create `Date Transformer` node (Code):**  
   - Connect output of Schedule Trigger to this node  
   - Use JavaScript code to calculate `today` and `tomorrow` ISO date strings representing the current day range (midnight to midnight next day)  
   - Return JSON with keys `today` and `tomorrow`

3. **Create `Gmail` node:**  
   - Connect output of Date Transformer to Gmail node  
   - Operation: `getAll`  
   - Filters:  
     - `receivedAfter` = `={{ $json.today }}`  
     - `receivedBefore` = `={{ $json.tomorrow }}`  
   - Credentials: Configure OAuth2 for your Gmail account  
   - Ensure full message data is fetched (`simple = false`)

4. **Create `Aggregate` node:**  
   - Connect Gmail output to Aggregate  
   - Aggregate fields: `headers.from`, `headers.subject`, `text`  
   - Result: arrays of these fields for all messages

5. **Create `Email Cleanup` node (Code):**  
   - Connect Aggregate output here  
   - JavaScript code to:  
     - Loop over messages  
     - Extract sender, subject, and text (default fallback if missing)  
     - Remove HTML tags and unsubscribe/promotional content with regex  
     - Collapse excess newlines  
     - Format each message preview string with sender, subject, and cleaned preview  
     - Join all previews into one string under `combinedText` key

6. **Create `AI Agent` node (LangChain Agent):**  
   - Connect Email Cleanup output here  
   - Set system prompt defining role as AI Chief of Staff  
   - Input prompt includes `{{ $json.combinedText }}` to pass cleaned email text  
   - Define output format: markdown with specific sections (open loops, next steps, leads, etc.)  
   - Do not fabricate content; omit empty sections

7. **Create `OpenAI Chat Model` node:**  
   - Set model to `gpt-4o-mini`  
   - Connect this node as AI language model backend for AI Agent node  
   - Configure OpenAI API credentials with valid key

8. **Create `Format HTML` node (Code):**  
   - Connect AI Agent output here  
   - JavaScript code to convert markdown to HTML:  
     - Bold, italics, strikethrough, inline code, links, images, line breaks  
   - Output single `html` string array for email body

9. **Create `Send Message` node (Gmail):**  
   - Connect Format HTML output here  
   - Configure:  
     - Recipient email (e.g., `zawagner@gmail.com`)  
     - Subject: `"Here's Your Daily Pulse"`  
     - Message body: `={{ $json.html[0] }}` (HTML content)  
   - Use same Gmail OAuth2 credentials as before

10. **Activate the workflow** and test by running manually or waiting for scheduled trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                               |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow requires valid OAuth2 credentials for Gmail and an OpenAI API key for GPT-4o-mini. | Credential setup in n8n for Gmail and OpenAI |
| GPT-4o-mini model is used due to efficiency and cost considerations while maintaining quality.  | OpenAI model documentation                    |
| The AI prompt restricts the agent from inventing information, ensuring factual summaries only.  | Prompt design best practices                   |
| Ensure n8n server timezone matches expected schedule or adjust trigger time accordingly.        | n8n scheduling documentation                   |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.