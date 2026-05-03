Create Daily Google Alerts Digest with Gemini AI Summarization and Gmail

https://n8nworkflows.xyz/workflows/create-daily-google-alerts-digest-with-gemini-ai-summarization-and-gmail-11452


# Create Daily Google Alerts Digest with Gemini AI Summarization and Gmail

---
### 1. Workflow Overview

This workflow automates the creation of a daily digest email summarizing Google Alerts using Google Gemini AI for summarization and Gmail for notifications. It is designed to help users efficiently monitor news or topics of interest by consolidating multiple alert emails into a single, easy-to-read briefing, thus avoiding inbox clutter.

**Target Use Cases:**  
- Daily executive briefings on competitors or industry trends  
- Efficient brand monitoring without sifting through individual alert emails  
- Creating summarized news digests for teams or personal use

**Logical Blocks:**  
- **1.1 Fetch Alerts & Extract Data:** Retrieve unread Google Alerts emails, aggregate their content, and extract article links and topics, cleaning tracking redirects.  
- **1.2 Scrape & Clean Content:** Visit each article URL, download the raw HTML content, and clean it by removing scripts, styles, and unnecessary tags to optimize AI input.  
- **1.3 AI Analysis & Summarization:** Use Google Gemini AI to generate concise summaries of the cleaned article content, enforcing structured JSON output.  
- **1.4 Compile & Format Report:** Combine all summaries with topics and links, aggregate into a single list, and generate a professional HTML email table sorted alphabetically by topic.  
- **1.5 Deliver Digest Email:** Send the compiled HTML digest via Gmail to the specified recipient.  
- **1.6 Clean Inbox:** Mark the original Google Alert emails as read to prevent reprocessing.

---

### 2. Block-by-Block Analysis

#### 2.1 Fetch Alerts & Extract Data

- **Overview:**  
  This block fetches unread Google Alerts emails, aggregates their HTML content, and extracts clean article links and topics by removing Google tracking redirects.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get Google Alerts (Gmail node)  
  - Aggregate Google Alerts Content (Aggregate node)  
  - Extract Links and Topics (Code node)  
  - Mark Alert as read (Gmail node)

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually for testing or ad hoc runs  
    - Config: No parameters; triggers workflow execution  
    - Input/Output: No input; output triggers Get Google Alerts  
    - Potential issues: None typical; manual trigger

  - **Get Google Alerts**  
    - Type: Gmail node (Get All Messages)  
    - Role: Retrieve all unread emails from sender `googlealerts-noreply@google.com`  
    - Config: Filters for unread, specific sender; returns all matching messages  
    - Credentials: Gmail OAuth2 (configured)  
    - Input: Trigger from manual node  
    - Output: List of email messages to aggregate node  
    - Edge cases: Gmail API quota limits, auth errors, no unread messages

  - **Aggregate Google Alerts Content**  
    - Type: Aggregate node  
    - Role: Combine all emails' HTML bodies into a single array for batch processing  
    - Config: Aggregates the "html" field from all items  
    - Input: Emails from Get Google Alerts  
    - Output: Single aggregated object with array of HTML content  
    - Edge cases: Empty input array, malformed HTML

  - **Extract Links and Topics**  
    - Type: Code node (JavaScript)  
    - Role: Parses aggregated HTML to extract article topics and clean URLs by removing Google redirect parameters  
    - Config: Uses regex to identify topic headers and href links, cleans tracking prefixes, decodes URLs, deduplicates results  
    - Key expressions: Regex pattern to find `<span>` with topic or `href` links, string manipulation to clean URLs  
    - Input: Aggregated HTML array  
    - Output: Array of unique objects with `{topic, link}`  
    - Edge cases: Emails without expected HTML structure, malformed URLs, missing topics  
    - Failure modes: Regex errors, empty or invalid data leading to empty output

  - **Mark Alert as read**  
    - Type: Gmail node (Mark as read)  
    - Role: Marks each processed Google Alert email as read to avoid duplication  
    - Config: Uses message ID from each email item  
    - Credentials: Gmail OAuth2  
    - Input: Emails from Get Google Alerts  
    - Output: Confirmation of marking messages as read  
    - Edge cases: API failures, race conditions if emails are deleted or changed  

---

#### 2.2 Scrape & Clean Content

- **Overview:**  
  This block visits each extracted article URL, downloads the raw HTML content, applies length trimming, and cleans HTML tags to produce text optimized for AI summarization.

- **Nodes Involved:**  
  - Extract Links and Topics (output from previous block)  
  - Get Website Content (HTTP Request node)  
  - Trim Text (Set node)  
  - Clean HTML Tags, newlines and trim text1 (Code node)

- **Node Details:**  

  - **Get Website Content**  
    - Type: HTTP Request  
    - Role: Fetch raw HTML from each article link  
    - Config: URL taken dynamically from `link` field; allows unauthorized certificates; set to never error to continue workflow on failures  
    - Input: Array of links from Extract Links and Topics  
    - Output: HTTP response data containing HTML content or error info  
    - Edge cases: Timeout, 404 or other HTTP errors, SSL issues, malformed URLs

  - **Trim Text**  
    - Type: Set node  
    - Role: Add a `trim_length` parameter to limit the length of content passed to cleaning node, controlling AI token usage  
    - Config: Sets `trim_length` to 300000 characters (max content length)  
    - Input: Website content  
    - Output: Same data with added `trim_length` field  
    - Edge cases: Trimming too much loses content; trimming too little risks high token cost

  - **Clean HTML Tags, newlines and trim text1**  
    - Type: Code node (JavaScript)  
    - Role: Cleans HTML content by removing scripts, styles, comments, all HTML tags, decodes HTML entities, and normalizes whitespace  
    - Config: Runs once per item; truncates cleaned text to `trim_length`  
    - Key expressions: Regex for scripts/styles/comments; string replacements for entities and whitespace  
    - Input: Raw HTML content with `trim_length`  
    - Output: Cleaned plain text suitable for AI input under `data` field  
    - Edge cases: Unexpected HTML structure, very large inputs causing performance issues

---

#### 2.3 AI Analysis & Summarization

- **Overview:**  
  This block sends the cleaned article content to Google Gemini AI to generate structured summaries, ensuring output is valid JSON.

- **Nodes Involved:**  
  - Clean HTML Tags, newlines and trim text1  
  - Google Gemini Chat Model (LangChain node)  
  - Structured Output Parser (LangChain Output Parser node)  
  - Create Summary (LangChain Agent node)

- **Node Details:**  

  - **Google Gemini Chat Model**  
    - Type: LangChain LM Chat node (Google Gemini)  
    - Role: Calls Google Gemini 2.5 Pro model to generate AI output  
    - Config: Model name set to `models/gemini-2.5-pro`; uses Google Palm API credentials  
    - Input: Text from cleaning node via Create Summary node (as AI input)  
    - Output: Raw AI-generated chat completion  
    - Edge cases: API quota limits, authentication errors, malformed input causing poor responses

  - **Structured Output Parser**  
    - Type: LangChain Output Parser (Structured)  
    - Role: Enforces strict JSON format on AI output, expecting key "Summary"  
    - Config: AutoFix enabled to correct minor format errors  
    - Input: Output from Google Gemini Chat Model  
    - Output: Parsed JSON object with summary text  
    - Edge cases: AI returns invalid JSON, parsing failures

  - **Create Summary**  
    - Type: LangChain Agent node  
    - Role: Orchestrates AI summarization using a custom prompt instructing the AI to analyze website content and return a short summary in a JSON object with key "Summary"  
    - Config: Prompt includes instructions to avoid markdown code blocks, output raw JSON only, and limit summary to 2-4 sentences  
    - Input: Cleaned text from cleaning node (field `data`)  
    - Output: AI summary in JSON format  
    - On error: Continues workflow ignoring individual failures  
    - Edge cases: AI failure to respond, improperly structured output

---

#### 2.4 Compile & Format Report

- **Overview:**  
  The block standardizes the AI summary with its topic and link, aggregates all summaries into one list, and generates a polished HTML email table sorted by topic.

- **Nodes Involved:**  
  - Create Summary  
  - Map summary, topic and link (Set node)  
  - put all entries into a single list (Aggregate node)  
  - Create HTML template and sort by topic (Code node)

- **Node Details:**  

  - **Map summary, topic and link**  
    - Type: Set node  
    - Role: Creates uniform objects with fields `summary`, `topic`, and `link` by merging AI summary output with extracted topic and link from earlier steps  
    - Config: Assigns `summary` from AI field `output.Summary`, `topic` and `link` from Extract Links and Topics node  
    - Input: Output from Create Summary node and Extract Links and Topics node  
    - Output: Structured summary-topic-link objects  
    - Edge cases: Missing fields causing nulls

  - **put all entries into a single list**  
    - Type: Aggregate node  
    - Role: Combines all individual article summary objects into one array under field `output` for final report generation  
    - Config: Aggregates all item data  
    - Input: Mapped summary objects  
    - Output: Single aggregated object with all summaries  
    - Edge cases: Empty input lists

  - **Create HTML template and sort by topic**  
    - Type: Code node (JavaScript)  
    - Role: Generates a responsive HTML table for email, sorting all entries alphabetically by topic; includes columns for Summary, Topic, and Link with clickable URLs; applies styling for clarity  
    - Config: Uses inline styles for email compatibility, cleans strings, formats links, handles empty data gracefully  
    - Input: Aggregated list of summaries  
    - Output: JSON object with `htmlEmailBody` containing the full HTML string  
    - Edge cases: Empty or malformed data, invalid URLs

---

#### 2.5 Deliver Digest Email

- **Overview:**  
  Sends the compiled HTML summary email to the configured recipient.

- **Nodes Involved:**  
  - Create HTML template and sort by topic  
  - Send Google Alert Summary (Gmail node)

- **Node Details:**  

  - **Send Google Alert Summary**  
    - Type: Gmail node (Send Email)  
    - Role: Sends the compiled HTML digest email  
    - Config: Recipient email hardcoded (e.g., `youremail@yourdomain.com`), subject "Google Alert Summary", message body set to `htmlEmailBody` from previous node  
    - Credentials: Gmail OAuth2  
    - Input: HTML content from Create HTML template node  
    - Output: Confirmation of sent message  
    - Edge cases: Gmail API errors, invalid recipient address

---

#### 2.6 Clean Inbox

- **Overview:**  
  Marks all processed Google Alert emails as read to prevent repeated processing.

- **Nodes Involved:**  
  - Get Google Alerts  
  - Mark Alert as read (Gmail node)

- **Node Details:**  

  - **Mark Alert as read**  
    - (Already detailed in 2.1)  
    - Connected directly after Get Google Alerts, marks each message read by message ID

---

### 3. Summary Table

| Node Name                         | Node Type                              | Functional Role                                 | Input Node(s)                  | Output Node(s)                         | Sticky Note                                                                                           |
|----------------------------------|--------------------------------------|------------------------------------------------|-------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                       | Starts workflow manually                        | None                          | Get Google Alerts                     |                                                                                                     |
| Get Google Alerts                | Gmail (Get All Messages)             | Retrieves unread Google Alerts emails          | When clicking ‘Execute workflow’ | Aggregate Google Alerts Content, Mark Alert as read | # 1. Fetch Alerts & Extract Data: Retrieves all unread emails from Google Alerts sender.           |
| Aggregate Google Alerts Content | Aggregate                           | Aggregates all email HTML bodies                | Get Google Alerts             | Extract Links and Topics              | # 1. Fetch Alerts & Extract Data: Combines all email HTML bodies for batch processing.              |
| Extract Links and Topics         | Code                               | Parses aggregated HTML to extract topics/links | Aggregate Google Alerts Content | Get Website Content                   | # 1. Fetch Alerts & Extract Data: Extracts article topics and cleans Google tracking redirects.     |
| Mark Alert as read               | Gmail (Mark as read)                | Marks processed alerts as read                   | Get Google Alerts             | None                                | # 6. Clean inbox: Marks original Google Alert emails as read to avoid reprocessing.                  |
| Get Website Content              | HTTP Request                       | Downloads raw HTML content from article URLs    | Extract Links and Topics      | Trim Text                           | # 2. Scrape & Clean Content: Visits article URLs and downloads raw HTML content.                    |
| Trim Text                      | Set                                | Limits content length for AI                      | Get Website Content           | Clean HTML Tags, newlines and trim text1 | # 2. Scrape & Clean Content: Sets max character limit to control AI token usage.                    |
| Clean HTML Tags, newlines and trim text1 | Code                      | Cleans HTML tags, scripts, comments, trims text | Trim Text                    | Create Summary                      | # 2. Scrape & Clean Content: Produces clean plain text optimized for AI input.                      |
| Google Gemini Chat Model         | LangChain LM Chat (Google Gemini)   | Calls Google Gemini AI for summarization         | Create Summary (ai_languageModel input) | Create Summary, Structured Output Parser | # 3. AI Analysis & Summarization: Uses Google Gemini to generate concise summaries.                 |
| Structured Output Parser         | LangChain Output Parser (Structured) | Enforces JSON output format from AI               | Google Gemini Chat Model (ai_outputParser output) | Create Summary (ai_outputParser input) | # 3. AI Analysis & Summarization: Ensures AI response is valid JSON with key "Summary".             |
| Create Summary                  | LangChain Agent                    | Orchestrates AI summarization with prompt         | Clean HTML Tags, newlines and trim text1 | Google Gemini Chat Model             | # 3. AI Analysis & Summarization: Prompts AI to generate short JSON summaries of article content.   |
| Map summary, topic and link      | Set                                | Combines AI summary with topic and link          | Create Summary                | put all entries into a single list   | # 4. Compile & Format Report: Standardizes data object with summary, topic, and link.               |
| put all entries into a single list | Aggregate                       | Aggregates all article summaries into one list   | Map summary, topic and link   | Create HTML template and sort by topic | # 4. Compile & Format Report: Aggregates all summaries for final report.                            |
| Create HTML template and sort by topic | Code                        | Generates responsive HTML table for email digest | put all entries into a single list | Send Google Alert Summary            | # 4. Compile & Format Report: Creates professional, sorted HTML table for email.                    |
| Send Google Alert Summary        | Gmail (Send Email)                 | Sends the compiled summary digest via email      | Create HTML template and sort by topic | None                                | # 5. Delivering: Sends the compiled HTML digest email to recipient.                                 |
| Sticky Note9                    | Sticky Note                       | Documentation and overview of workflow           | None                         | None                                | ## Automated AI News Digest from Google Alerts with detailed usage and setup instructions           |
| Sticky Note10                   | Sticky Note                       | Explains Fetch Alerts & Extract Data block       | None                         | None                                | # 1. Fetch Alerts & Extract Data explanation                                                         |
| Sticky Note11                   | Sticky Note                       | Explains Scrape & Clean Content block            | None                         | None                                | # 2. Scrape & Clean Content explanation                                                             |
| Sticky Note12                   | Sticky Note                       | Explains AI Analysis & Summarization block       | None                         | None                                | # 3. AI Analysis & Summarization explanation                                                        |
| Sticky Note13                   | Sticky Note                       | Explains Compile & Format Report block            | None                         | None                                | # 4. Compile & Format Report explanation                                                            |
| Sticky Note14                   | Sticky Note                       | Explains Delivering block                          | None                         | None                                | # 5. Delivering explanation                                                                          |
| Sticky Note15                   | Sticky Note                       | Explains Clean inbox block                         | None                         | None                                | # 6. Clean inbox explanation                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a Manual Trigger node named "When clicking ‘Execute workflow’". This will start the workflow manually for testing.

2. **Add Gmail Node to Get Google Alerts:**  
   - Node type: Gmail (Get All Messages)  
   - Name: "Get Google Alerts"  
   - Configure to fetch all unread emails with filter `sender:googlealerts-noreply@google.com`.  
   - Set `Return All` to true.  
   - Connect credentials with Gmail OAuth2 account.  
   - Connect output from Manual Trigger to this node.

3. **Add Aggregate Node to Combine Email Content:**  
   - Node type: Aggregate  
   - Name: "Aggregate Google Alerts Content"  
   - Configure to aggregate all items’ `html` field into a single array.  
   - Connect output from "Get Google Alerts" to this node.

4. **Add Code Node to Extract Links and Topics:**  
   - Node type: Code  
   - Name: "Extract Links and Topics"  
   - Paste the JavaScript code that loops through aggregated emails, extracts topics and cleans Google redirect links using regex and string methods.  
   - Connect output from "Aggregate Google Alerts Content" to this node.

5. **Add HTTP Request Node to Get Website Content:**  
   - Node type: HTTP Request  
   - Name: "Get Website Content"  
   - Set URL to be dynamic: `={{$json.link}}` to fetch each article URL.  
   - Enable `Allow Unauthorized Certificates`.  
   - Set response to never error to continue on failures.  
   - Connect output from "Extract Links and Topics" to this node.

6. **Add Set Node to Trim Text Length:**  
   - Node type: Set  
   - Name: "Trim Text"  
   - Add field `trim_length` as number with value 300000 (max characters).  
   - Connect output from "Get Website Content" to this node.

7. **Add Code Node to Clean HTML Tags and Text:**  
   - Node type: Code  
   - Name: "Clean HTML Tags, newlines and trim text1"  
   - Paste provided JavaScript code that removes scripts, styles, comments, decodes HTML entities, normalizes whitespace, and trims to `trim_length`.  
   - Connect output from "Trim Text" to this node.

8. **Add LangChain Agent Node for AI Summarization:**  
   - Node type: LangChain Agent  
   - Name: "Create Summary"  
   - Configure prompt instructing AI to analyze website content and output a JSON object with a "Summary" key containing a 2-4 sentence summary.  
   - Set input text parameter to `{{$json.data}}`.  
   - Connect output from "Clean HTML Tags, newlines and trim text1" to this node.

9. **Add Google Gemini Chat Model Node:**  
   - Node type: LangChain LM Chat (Google Gemini)  
   - Name: "Google Gemini Chat Model"  
   - Model: `models/gemini-2.5-pro`  
   - Connect LangChain Google Palm API credentials.  
   - Connect `Create Summary` node's AI language model input to this node.

10. **Add Structured Output Parser Node:**  
    - Node type: LangChain Output Parser (Structured)  
    - Name: "Structured Output Parser"  
    - Configure JSON schema expecting key `"Summary"`.  
    - Enable AutoFix.  
    - Connect AI output parser from "Google Gemini Chat Model" to this node.  
    - Connect this node's output back to `Create Summary` node's AI output parser input.

11. **Add Set Node to Map Summary, Topic, and Link:**  
    - Node type: Set  
    - Name: "Map summary, topic and link"  
    - Assign fields:  
      - `summary` = `={{ $json.output.Summary }}` (from AI output)  
      - `topic` = `={{$('Extract Links and Topics').item.json.topic}}`  
      - `link` = `={{$('Extract Links and Topics').item.json.link}}`  
    - Connect output from "Create Summary" to this node.

12. **Add Aggregate Node to Combine Summaries:**  
    - Node type: Aggregate  
    - Name: "put all entries into a single list"  
    - Configure to aggregate all item data into a single field `output`.  
    - Connect output from "Map summary, topic and link" to this node.

13. **Add Code Node to Create HTML Template:**  
    - Node type: Code  
    - Name: "Create HTML template and sort by topic"  
    - Paste given JavaScript code that builds a styled HTML table with columns Summary, Topic, Link, sorting entries alphabetically by topic.  
    - Connect output from "put all entries into a single list" to this node.

14. **Add Gmail Node to Send Digest Email:**  
    - Node type: Gmail (Send Email)  
    - Name: "Send Google Alert Summary"  
    - Configure recipient email (e.g., your email address).  
    - Subject: "Google Alert Summary"  
    - Message body: `={{ $json.htmlEmailBody }}` from HTML template node.  
    - Connect Gmail OAuth2 credentials.  
    - Connect output from "Create HTML template and sort by topic" to this node.

15. **Add Gmail Node to Mark Original Emails as Read:**  
    - Node type: Gmail (Mark as read)  
    - Name: "Mark Alert as read"  
    - Set parameter Message ID: `={{ $('Get Google Alerts').item.json.id }}`  
    - Connect Gmail OAuth2 credentials.  
    - Connect output from "Get Google Alerts" to this node.

16. **Connect all nodes as per above and save workflow.**  
    - Confirm the execution order matches the described flow.  
    - Test with manual trigger; replace manual trigger with Schedule Trigger for production use.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Automated AI News Digest from Google Alerts: Stop drowning in individual notification emails. This workflow collects your Google Alerts, reads the articles, and compiles a single, AI-summarized briefing. Use cases include daily briefings on competitors, industry trends, or brand mentions. Setup requires Google Gemini API and Gmail OAuth2. Contact berni@zindel.digital for help. | Inline workflow sticky note with overview and usage instructions                                  |
| For production, replace the Manual Trigger node with a Schedule or Cron Trigger to automate daily runs (e.g., every morning at 7 AM).                                                                                                                                                                                                                                                            | See Sticky Note9 and Sticky Note10                                                                |
| Customize AI prompt in the "Create Summary" node to tailor summaries for specific industries or languages, e.g., add questions or extra fields in the output parser for real estate or other domains.                                                                                                                                                                                              | See Sticky Note12                                                                                  |
| Gmail API quotas and OAuth2 credentials must be correctly configured for both reading alerts and sending emails.                                                                                                                                                                                                                                                                                  | General requirement                                                                                 |
| The HTML email template uses inline CSS for maximum compatibility across email clients and sorts news items alphabetically by topic for readability.                                                                                                                                                                                                                                            | See "Create HTML template and sort by topic" node code                                           |
| To avoid repeated processing, the workflow marks the original Google Alert emails as read after fetching them.                                                                                                                                                                                                                                                                                   | Sticky Note15                                                                                     |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected material. All data handled is legal and public.