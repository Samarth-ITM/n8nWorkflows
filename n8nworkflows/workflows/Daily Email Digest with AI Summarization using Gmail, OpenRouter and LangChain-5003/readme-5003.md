Daily Email Digest with AI Summarization using Gmail, OpenRouter and LangChain

https://n8nworkflows.xyz/workflows/daily-email-digest-with-ai-summarization-using-gmail--openrouter-and-langchain-5003


# Daily Email Digest with AI Summarization using Gmail, OpenRouter and LangChain

---
### 1. Workflow Overview

This workflow automates the daily generation and distribution of a summarized email digest for the inbox of `isb.quantana@quantana.in`. It targets teams needing a concise update of recent email communications, highlighting key topics, issues, actions, and follow-ups extracted from emails received in the past 24 hours.

The workflow is logically divided into five blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow every weekday at 8 AM IST (7 AM UTC).
- **1.2 Email Retrieval:** Fetches all emails received in the last 24 hours from the specified Gmail account.
- **1.3 Email Data Organization:** Aggregates and extracts relevant metadata from the email batch.
- **1.4 AI-Powered Summarization:** Uses LangChain with OpenRouter LLM to analyze email content, extracting summaries, issues, actions, and follow-ups.
- **1.5 Summary Distribution:** Sends a beautifully formatted HTML email digest to a predefined team mailing list.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow execution automatically every day at 8 AM IST (7 AM UTC), ensuring timely daily digests without manual intervention.

- **Nodes Involved:**  
  - Daily 8AM Trigger

- **Node Details:**

  - **Node Name:** Daily 8AM Trigger  
  - **Type:** Schedule Trigger  
  - **Configuration:**  
    - Trigger set to hourly interval with a fixed trigger time at 7 AM UTC (which corresponds to 8 AM IST).  
    - Runs every day (default weekdays implied by usage).  
  - **Input:** None (trigger node)  
  - **Output:** Triggers the next node, "Fetch Emails - Past 24 Hours"  
  - **Version Requirements:** n8n v0.132.0+ recommended for accurate scheduling features  
  - **Potential Failures:** Misconfigured time zones or daylight saving changes may cause timing drift  

#### 2.2 Email Retrieval

- **Overview:**  
  Fetches all emails received in the past 24 hours from the Gmail inbox of `isb.quantana@quantana.in` using Gmail API.

- **Nodes Involved:**  
  - Fetch Emails - Past 24 Hours

- **Node Details:**

  - **Node Name:** Fetch Emails - Past 24 Hours  
  - **Type:** Gmail (Get All Emails)  
  - **Configuration:**  
    - Uses Gmail API to query emails from the last 24 hours using a dynamic Gmail search query with `after:YYYY/MM/DD`.  
    - The query filters emails specifically for address `isb.quantana@quantana.in`.  
    - Returns all matching emails without pagination limits.  
  - **Key Expressions:**  
    - Dynamic date calculation with JavaScript to compute yesterday’s date and format it appropriately for Gmail search:  
      ```js
      (() => {
        const yesterday = new Date();
        yesterday.setDate(yesterday.getDate() - 1);
        return `isb.quantana@quantana.in after:${yesterday.getFullYear()}/${(yesterday.getMonth() + 1).toString().padStart(2, '0')}/${yesterday.getDate().toString().padStart(2, '0')}`;
      })()
      ```  
  - **Input:** Trigger from "Daily 8AM Trigger"  
  - **Output:** Email data array to "Organize Email Data - Morning"  
  - **Credentials:** OAuth2 credentials for Gmail API required (GMAIL_CREDS)  
  - **Potential Failures:**  
    - Authentication errors due to expired OAuth tokens  
    - Gmail API rate limits or quota exceeded  
    - Date calculation errors affecting query range  

#### 2.3 Email Data Organization

- **Overview:**  
  Aggregates and extracts essential metadata from all fetched emails, preparing data for AI summarization.

- **Nodes Involved:**  
  - Organize Email Data - Morning

- **Node Details:**

  - **Node Name:** Organize Email Data - Morning  
  - **Type:** Aggregate  
  - **Configuration:**  
    - Aggregates all input email items into a single combined data structure.  
    - Includes only specific fields: `id`, `From`, `To`, `CC`, and `snippet` (a short preview of the email body).  
  - **Input:** Emails from "Fetch Emails - Past 24 Hours"  
  - **Output:** Aggregated email metadata to "Email Summarizer"  
  - **Potential Failures:**  
    - If emails have missing fields, aggregation might produce incomplete outputs  
    - Large volumes of emails could affect performance  

#### 2.4 AI-Powered Summarization

- **Overview:**  
  Processes the aggregated email data using LangChain with OpenRouter LLM to extract a structured summary, including main topics, critical points, issues, action items, and follow-ups.

- **Nodes Involved:**  
  - Email Summarizer  
  - OpenRouter Chat Model  
  - Simple Memory

- **Node Details:**

  - **Node Name:** Email Summarizer  
  - **Type:** LangChain Agent  
  - **Configuration:**  
    - Uses a custom system message prompt defining a two-step process:  
      1. Extract key details (topics, data points, requests, problems).  
      2. Organize output into summary, issues, actions, follow-ups with clear formatting rules (conciseness, priority flags, ownership).  
    - The prompt contains explicit instructions, examples, and formatting guidelines for clarity.  
  - **Input:** Aggregated email data from "Organize Email Data - Morning"  
  - **Output:** Structured AI-generated message to "Send Summary - Morning"  
  - **Potential Failures:**  
    - OpenRouter API errors (authentication, rate limits, network issues)  
    - Prompt parsing or LLM response format deviations  
    - Memory overflow if previous context is too large (managed by "Simple Memory")  

  - **Node Name:** OpenRouter Chat Model  
  - **Type:** LangChain OpenRouter Language Model Node  
  - **Configuration:**  
    - Connects to OpenRouter API using stored credentials (`OPENROUTER_KEY`).  
    - Serves as the LLM backend for the "Email Summarizer".  
  - **Input:** Language model input from "Email Summarizer"  
  - **Output:** LLM response to "Email Summarizer"  
  - **Potential Failures:** API key invalidation, network timeouts, quota exceeded  

  - **Node Name:** Simple Memory  
  - **Type:** LangChain Buffer Window Memory  
  - **Configuration:**  
    - Maintains context window for conversation history.  
    - Connected as memory input for "Email Summarizer" to preserve state or context if needed.  
  - **Input:** AI memory output from "Email Summarizer"  
  - **Output:** Memory context to "Email Summarizer"  
  - **Potential Failures:** Memory size limits, data corruption  

#### 2.5 Summary Distribution

- **Overview:**  
  Sends the AI-generated email summary as a formatted HTML message via Gmail API to a team mailing list, including CC recipients and styled layout.

- **Nodes Involved:**  
  - Send Summary - Morning

- **Node Details:**

  - **Node Name:** Send Summary - Morning  
  - **Type:** Gmail Send Email  
  - **Configuration:**  
    - Sends to `team-email@example.com` with CC to `cc-list@example.com`.  
    - Subject dynamically generated to reflect the date range of the summarized emails (previous day 00:00 to current day 07:00 AM).  
    - Body is a fully styled HTML email embedding AI-generated content using expressions to map over summary and action arrays.  
    - Attribution lines and reply options configured for professional appearance.  
  - **Key Expressions:**  
    - Subject line formatted using JavaScript date functions for locale-specific formatting.  
    - Email body uses template expressions to loop through AI summary JSON arrays:  
      ```plaintext
      {{ $json.message.content.summary_of_emails.map(email => `<li>${email}</li>`).join('') }}
      {{ $json.message.content.actions.map(action => `<li><span class="action-item">${action.name}:</span><span class="action-detail">${action.action}</span></li>`).join('') }}
      ```  
  - **Input:** AI summary output from "Email Summarizer"  
  - **Output:** Email sent confirmation (end of workflow)  
  - **Credentials:** OAuth2 credentials for Gmail API required (GMAIL_CREDS)  
  - **Potential Failures:**  
    - Email sending errors due to API limits or invalid addresses  
    - HTML rendering issues if AI content is malformed  
    - CC list misconfiguration causing delivery failures  

---

### 3. Summary Table

| Node Name                    | Node Type                      | Functional Role                   | Input Node(s)                  | Output Node(s)                     | Sticky Note                                                                                   |
|------------------------------|--------------------------------|---------------------------------|-------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Daily 8AM Trigger             | Schedule Trigger               | Workflow starter (daily trigger) | None                          | Fetch Emails - Past 24 Hours      | 📧 Email Auto-Summary Flow (n8n) - Purpose: Daily digest of `isb.quantana@quantana.in` emails, runs 8AM IST weekdays. |
| Fetch Emails - Past 24 Hours | Gmail                         | Retrieve last 24h emails         | Daily 8AM Trigger             | Organize Email Data - Morning     | See above sticky note                                                                         |
| Organize Email Data - Morning | Aggregate                    | Extract and combine email metadata| Fetch Emails - Past 24 Hours  | Email Summarizer                  | See above sticky note                                                                         |
| Email Summarizer              | LangChain Agent               | AI summarization of emails       | Organize Email Data - Morning | Send Summary - Morning            | See above sticky note                                                                         |
| OpenRouter Chat Model         | LangChain OpenRouter LLM Node | LLM backend for AI summarizer    | Email Summarizer              | Email Summarizer                  | See above sticky note                                                                         |
| Simple Memory                 | LangChain Memory Buffer       | Maintains AI conversation context| Email Summarizer (ai_memory)  | Email Summarizer                  | See above sticky note                                                                         |
| Send Summary - Morning        | Gmail                         | Send formatted summary email     | Email Summarizer              | None                            | See above sticky note                                                                         |
| Sticky Note                  | Sticky Note                   | Documentation and overview       | None                         | None                            | Contains detailed workflow summary and instructions                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node:**  
   - Name: `Daily 8AM Trigger`  
   - Type: Schedule Trigger  
   - Set interval to daily, trigger at 07:00 UTC (corresponding to 08:00 IST).  
   - Save.

2. **Create a Gmail Node to Fetch Emails:**  
   - Name: `Fetch Emails - Past 24 Hours`  
   - Type: Gmail (Get All Emails)  
   - Operation: `getAll`  
   - Return All: `true`  
   - Under Filters, set Query (`q`) to a JavaScript expression that dynamically calculates yesterday’s date and formats the Gmail search query for `isb.quantana@quantana.in` emails received after that date:  
     ```js
     (() => {
       const yesterday = new Date();
       yesterday.setDate(yesterday.getDate() - 1);
       return `isb.quantana@quantana.in after:${yesterday.getFullYear()}/${(yesterday.getMonth() + 1).toString().padStart(2,'0')}/${yesterday.getDate().toString().padStart(2,'0')}`;
     })()
     ```  
   - Connect input from `Daily 8AM Trigger`.  
   - Assign Gmail OAuth2 credentials (e.g., `GMAIL_CREDS`).

3. **Create an Aggregate Node to Organize Email Data:**  
   - Name: `Organize Email Data - Morning`  
   - Type: Aggregate  
   - Set to aggregate all item data into one.  
   - Include only fields: `id`, `From`, `To`, `CC`, `snippet`.  
   - Connect input from `Fetch Emails - Past 24 Hours`.

4. **Create LangChain Nodes for AI Processing:**  
   - Create `OpenRouter Chat Model` Node:  
     - Type: LangChain OpenRouter LLM Node  
     - Configure with OpenRouter API credentials (`OPENROUTER_KEY`).  
   - Create `Simple Memory` Node:  
     - Type: LangChain Buffer Window Memory  
     - No special parameters needed; default buffer window.  
   - Create `Email Summarizer` Node:  
     - Type: LangChain Agent  
     - Set System Message prompt with detailed instructions to extract main topics, critical data, issues, actions, and follow-ups. Use the provided comprehensive prompt text ensuring clarity, conciseness, and formatting rules.  
     - Connect `Email Summarizer` to use `OpenRouter Chat Model` as its language model and `Simple Memory` as its memory.  
     - Connect input from `Organize Email Data - Morning`.

5. **Create Gmail Node to Send Summary Email:**  
   - Name: `Send Summary - Morning`  
   - Type: Gmail (Send Email)  
   - Configure:  
     - Send To: `team-email@example.com`  
     - CC List: `cc-list@example.com`  
     - Subject: Use an expression to format the subject line with previous day and current day date range in `en-GB` locale, e.g.,  
       ```js
       =`ESAgent - ${new Date(new Date().setDate(new Date().getDate() - 1)).toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' })}-00:00 to ${new Date().toLocaleDateString('en-GB', { day: '2-digit', month: 'short', year: 'numeric' })}-07:00AM`
       ```  
     - Message: Use a fully formed HTML template embedding expressions to map AI-generated summary and actions arrays into list items as shown in the original workflow.  
     - Disable "Append Attribution" and "Reply To Sender Only" options.  
   - Connect input from `Email Summarizer`.  
   - Assign Gmail OAuth2 credentials (same as fetch node or separate with necessary permissions).

6. **Connect the Nodes in the Following Order:**  
   - `Daily 8AM Trigger` → `Fetch Emails - Past 24 Hours` → `Organize Email Data - Morning` → `Email Summarizer` → `Send Summary - Morning`  
   - Connect `OpenRouter Chat Model` and `Simple Memory` as sub-nodes feeding into `Email Summarizer` as per LangChain agent requirements.

7. **Add a Sticky Note (Optional):**  
   - Use to document purpose, API requirements, and general notes for maintainers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow uses Gmail API with OAuth2 for both fetching and sending emails. Ensure correct scopes (readonly for fetch, send for email).      | Gmail API OAuth2 documentation: https://developers.google.com/gmail/api/auth/about-auth                |
| OpenRouter API key must be stored securely in n8n credentials and have sufficient quota for LLM calls.                                     | OpenRouter API docs: https://openrouter.ai/docs                                                        |
| The AI prompt in the LangChain Agent node is critical for accurate email summarization; modifying it requires understanding of prompt engineering and the expected JSON output format. | See LangChain documentation on prompt design: https://js.langchain.com/docs/modules/prompts            |
| The workflow assumes emails are primarily in English and structured for business communication; unusual email formats may reduce summarization quality. |                                                                                                       |
| HTML email template uses inline CSS and dynamic content injection; ensure output sanitization to avoid injection risks.                   |                                                                                                       |
| The workflow runs daily at 8 AM IST, processing emails from the prior day to early morning; adjust schedule or date logic for different time zones or intervals. |                                                                                                       |
| This workflow can be extended with Slack API nodes or other notification systems for alerting on errors or critical issues discovered via AI. |                                                                                                       |

---

**Disclaimer:** The provided text is derived solely from an n8n automated workflow and adheres strictly to current content policies with no illegal or offensive elements. All processed data is lawful and public.