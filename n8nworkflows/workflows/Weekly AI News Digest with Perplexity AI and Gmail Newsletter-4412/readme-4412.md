Weekly AI News Digest with Perplexity AI and Gmail Newsletter

https://n8nworkflows.xyz/workflows/weekly-ai-news-digest-with-perplexity-ai-and-gmail-newsletter-4412


# Weekly AI News Digest with Perplexity AI and Gmail Newsletter

### 1. Workflow Overview

This workflow, titled **"Weekly AI News Digest with Perplexity AI and Gmail Newsletter"**, automates the process of compiling and sending a curated weekly email digest summarizing the latest developments in artificial intelligence. It targets users interested in receiving a structured AI news summary that covers key categories such as new technology breakthroughs, AI security, top stories, and trending topics.

The workflow executes once per week (every Monday at 8 AM) and consists of the following logical blocks:

- **1.1 Scheduled Execution & Date Preparation**: Triggers workflow weekly and sets date variables for querying recent news.
- **1.2 AI News Retrieval via Perplexity API**: Sends a structured prompt to Perplexity AI to retrieve summarized news content from the last week.
- **1.3 Formatting News Content into HTML**: Processes the AI response, formats the news text into styled HTML including citations, and prepares the email subject.
- **1.4 Email Dispatch via Gmail**: Sends the formatted email digest to the configured recipients using Gmail’s OAuth2 authenticated SMTP.

Supporting these blocks are documentation and configuration sticky notes embedded for user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Execution & Date Preparation

- **Overview:**  
  This block initiates the workflow on a weekly schedule and prepares date-related variables for use in the API query.

- **Nodes Involved:**  
  - Daily Trigger  
  - Set Dates

- **Node Details:**

  - **Daily Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts workflow every Monday at 8 AM.  
    - *Configuration:* Runs on weeks interval, trigger day = Monday (1), trigger hour = 8.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Connects to "Set Dates".  
    - *Potential Failures:* Misconfigured time zone or schedule may cause unexpected trigger times.

  - **Set Dates**  
    - *Type:* Set  
    - *Role:* Assigns two string variables: `currentDate` (today’s date) and `lastweekDate` (date 7 days prior) in `yyyy-MM-dd` format.  
    - *Configuration:* Uses n8n expressions `$now.format('yyyy-MM-dd')` and `$now.minus({days: 7}).format('yyyy-MM-dd')` to dynamically generate dates at runtime.  
    - *Inputs:* From "Daily Trigger".  
    - *Outputs:* Connects to "Perplexity AI News Search".  
    - *Edge Cases:* Date calculation depends on system clock; daylight saving changes may affect accuracy if timezone not managed.

---

#### 2.2 AI News Retrieval via Perplexity API

- **Overview:**  
  This block queries Perplexity AI’s chat completion API with a prompt requesting a summarized compilation of the top AI news from the last week, segmented by categories.

- **Nodes Involved:**  
  - Perplexity AI News Search

- **Node Details:**

  - **Perplexity AI News Search**  
    - *Type:* HTTP Request  
    - *Role:* Sends a POST request to Perplexity AI’s chat completions endpoint to generate news summaries.  
    - *Configuration:*  
      - URL: `https://api.perplexity.ai/chat/completions`  
      - Method: POST  
      - Headers: Authorization with Bearer token (replace `XXX-XXXXX` with actual Perplexity API key).  
      - JSON Body: Contains the prompt requesting news summaries for categories "New Tech", "AI Security", "Top Stories", "Trending Topics" from the past week, with parameters tuned for low temperature and frequency penalty to reduce repetition.  
      - No images or related questions returned.  
    - *Inputs:* From "Set Dates" (triggered after date variables set).  
    - *Outputs:* Connects to "Format Email Content1".  
    - *Failure Modes:*  
      - Authentication errors (invalid or expired API key).  
      - Network timeouts or API rate limits.  
      - Unexpected API response structure causing downstream errors.

---

#### 2.3 Formatting News Content into HTML

- **Overview:**  
  This block processes the raw AI-generated news summary text, converting markdown-like formatting into styled HTML, adds a citations section if available, and constructs a full HTML email body and subject line.

- **Nodes Involved:**  
  - Format Email Content1

- **Node Details:**

  - **Format Email Content1**  
    - *Type:* Code (JavaScript)  
    - *Role:*  
      - Extracts the AI response (`choices[0].message.content`) and citations array from the Perplexity API response.  
      - Retrieves current date from "Set Dates" node.  
      - Converts markdown-like syntax (headings, bold, italics, lists, links) into corresponding HTML elements with inline CSS styling optimized for email clients.  
      - Builds a citations block section with clickable source links.  
      - Constructs a complete, visually appealing HTML email structure with header, main content, footer, and tip section.  
      - Sets the email subject including the current date.  
      - Implements error handling: if processing fails, generates a fallback error email with raw API response for troubleshooting.  
    - *Inputs:* From "Perplexity AI News Search".  
    - *Outputs:* Connects to "Gmail".  
    - *Edge Cases / Failures:*  
      - If API response format changes or is incomplete, parsing may fail.  
      - Date retrieval from "Set Dates" may fail if upstream node data missing.  
      - HTML rendering may vary by email client; inline CSS used to mitigate.  
      - Large content might exceed email size limits.  
    - *Expressions & Variables:* Uses `$('Set Dates').first().json.currentDate` to access date; `$input.first().json` for API data.

---

#### 2.4 Email Dispatch via Gmail

- **Overview:**  
  This block sends the formatted HTML email digest through Gmail to the intended recipients.

- **Nodes Involved:**  
  - Gmail

- **Node Details:**

  - **Gmail**  
    - *Type:* Gmail  
    - *Role:* Sends an email using Gmail API with OAuth2 authentication.  
    - *Configuration:*  
      - `sendTo`: Recipient email address (must be updated with actual recipient).  
      - `subject`: Dynamic, from formatted email content node.  
      - `message`: HTML body from previous node (`emailHtml`).  
      - Credentials: Gmail OAuth2 credential selected (must be properly set up for sending).  
    - *Inputs:* From "Format Email Content1".  
    - *Outputs:* None (terminal node).  
    - *Failures:*  
      - Authentication failure if OAuth2 token expired or invalid.  
      - Recipient email misconfiguration.  
      - Gmail API limits or quota exceeded.  
      - Email rejected as spam or blocked by Gmail filters.

---

#### 2.5 Documentation and Setup Notes (Supporting)

- **Nodes Involved:**  
  - Workflow Documentation (sticky note)  
  - Setup Instructions (sticky note)

- **Details:**  
  - These sticky notes provide user-facing documentation on workflow purpose, setup requirements (Perplexity API key, SMTP/Gmail credentials, recipient email), scheduling, and customization hints.  
  - Positioned for visibility but do not affect runtime.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                       | Input Node(s)         | Output Node(s)        | Sticky Note                                                                                 |
|--------------------------|--------------------|------------------------------------|-----------------------|-----------------------|---------------------------------------------------------------------------------------------|
| Daily Trigger            | Schedule Trigger   | Weekly trigger to start workflow    | None                  | Set Dates             | ⚙️ **Configuration Needed:** Modify schedule here to change timing                          |
| Set Dates               | Set                | Assigns current and last week dates | Daily Trigger         | Perplexity AI News Search |                                                                                             |
| Perplexity AI News Search | HTTP Request       | Queries Perplexity AI for news      | Set Dates             | Format Email Content1  | ⚙️ **Configuration Needed:** Add your Perplexity API key in node headers                    |
| Format Email Content1    | Code               | Formats AI news into styled HTML    | Perplexity AI News Search | Gmail                |                                                                                             |
| Gmail                   | Gmail              | Sends formatted email via Gmail     | Format Email Content1  | None                  | ⚙️ **Configuration Needed:** Update recipient and sender emails; use Gmail OAuth2 credential |
| Workflow Documentation   | Sticky Note        | Describes workflow purpose and use  | None                  | None                  | See detailed workflow description in node content                                          |
| Setup Instructions       | Sticky Note        | Setup and configuration instructions | None                  | None                  | See detailed setup steps in node content                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Daily Trigger" node**  
   - Type: Schedule Trigger  
   - Parameters: Set interval to weekly, trigger at day = Monday (1), trigger hour = 8 AM (local time).  
   - No credentials needed.  

2. **Create "Set Dates" node**  
   - Type: Set  
   - Parameters: Add two string fields:  
     - `currentDate` = Expression: `{{$now.format('yyyy-MM-dd')}}`  
     - `lastweekDate` = Expression: `{{$now.minus({days: 7}).format('yyyy-MM-dd')}}`  
   - Connect output of "Daily Trigger" to this node's input.

3. **Create "Perplexity AI News Search" node**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.perplexity.ai/chat/completions`  
     - Method: POST  
     - Authentication: None (use header authorization)  
     - Headers: Add header `Authorization` with value `Bearer YOUR_PERPLEXITY_API_KEY` (replace with your key)  
     - Body Content Type: JSON  
     - Body (JSON): Use the provided JSON body with the prompt requesting AI news summaries by category, filtering for last week, with specified model and parameters.  
   - Connect output of "Set Dates" to this node's input.

4. **Create "Format Email Content1" node**  
   - Type: Code  
   - Parameters: Paste the JavaScript code that:  
     - Extracts news content and citations from previous node output  
     - Converts text formatting to HTML with inline styles  
     - Builds full HTML email template with header, main content, citations section, tip section, and footer  
     - Sets email subject including current date  
     - Includes error handling fallback with raw response display  
   - Connect output of "Perplexity AI News Search" to this node.

5. **Create "Gmail" node**  
   - Type: Gmail  
   - Parameters:  
     - `sendTo`: Enter recipient email address(es)  
     - `subject`: Use expression: `{{$json.emailSubject}}` (from previous node)  
     - `message`: Use expression: `{{$json.emailHtml}}`  
     - Options: Leave default unless specific changes needed  
   - Credentials: Set up Gmail OAuth2 credential with valid permissions to send emails.  
   - Connect output of "Format Email Content1" to this node.

6. **(Optional) Add Documentation Sticky Notes**  
   - Create two Sticky Note nodes with contents:  
     - Workflow Documentation: Describe workflow purpose, schedule, categories, setup, and customization.  
     - Setup Instructions: List required API keys, SMTP/Gmail setup, recipient email, and scheduling info.

7. **Verify and Activate Workflow**  
   - Test the workflow manually or wait for scheduled trigger.  
   - Confirm email delivery with formatted AI news content.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow runs weekly (Monday 8 AM) but schedule can be modified in "Daily Trigger" node.                         | Scheduling configuration                             |
| Requires valid Perplexity API key added as Bearer token in HTTP Request node header.                             | Perplexity API setup                                 |
| Gmail node requires OAuth2 credentials configured with appropriate scopes for sending emails.                   | Gmail OAuth2 setup                                   |
| Email HTML uses inline CSS optimized for broad email client compatibility.                                       | Email formatting best practices                       |
| Error handling implemented in code node to fallback in case of API or parsing failures, providing raw response. | Robustness and troubleshooting guidance             |
| Suggested AI news categories: New Tech, AI Security, Top Stories, Trending Topics; customizable in prompt.       | Content customization                                |
| Inline links in email direct users to original news sources for deeper reading.                                  | User engagement and source attribution               |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automation workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.