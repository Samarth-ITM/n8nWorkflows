Summarize your emails with A.I. (via Openrouter) and send to Line messenger

https://n8nworkflows.xyz/workflows/summarize-your-emails-with-a-i---via-openrouter--and-send-to-line-messenger-2515


# Summarize your emails with A.I. (via Openrouter) and send to Line messenger

### 1. Workflow Overview

This workflow automates the process of reading emails, summarizing their content using an AI model hosted on Openrouter.ai, and sending the summarized messages to a Line messenger account. It is designed primarily for users overwhelmed by email volume—such as busy parents or executives—who want a concise digest of important emails, highlighting action items and deadlines.

The workflow is logically divided into three main blocks:

- **1.1 Email Retrieval:** Connects to an email account via IMAP to fetch new emails.
- **1.2 AI Summarization:** Sends each email’s content to an AI model for summarization and importance analysis.
- **1.3 Messaging Delivery:** Forwards the AI-generated summaries to a Line messenger user via the Line Messaging API.

Supporting the main logic, several sticky notes provide setup guidance, credential instructions, and usage tips for better customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Email Retrieval

- **Overview:**  
  This block retrieves emails from the user’s inbox using IMAP protocol. It acts as the data source feeding the workflow.

- **Nodes Involved:**  
  - Read emails (IMAP)

- **Node Details:**  

  - **Read emails (IMAP)**  
    - *Type & Role:* Email Read node (IMAP), version 2; connects to an email server and retrieves emails.  
    - *Configuration:*  
      - Uses IMAP credentials (configured externally) to access the email inbox.  
      - No post-processing action is performed on the retrieved emails (set to "nothing").  
      - Default options are used for email fetch (e.g., no specific folder or filter set).  
    - *Expressions / Variables:* None.  
    - *Input / Output:* No input nodes; outputs each email as JSON data.  
    - *Version Requirements:* n8n version supporting IMAP node v2.  
    - *Potential Failures:* Authentication errors due to incorrect IMAP credentials, network timeouts, or IMAP server unavailability.  
    - *Sub-workflows:* None.

#### 1.2 AI Summarization

- **Overview:**  
  This block sends the retrieved email data to Openrouter.ai’s chat completion API for summarization. The AI model analyzes the email’s importance, extracts deadlines and action items, and formats a concise summary.

- **Nodes Involved:**  
  - Send email to A.I. to summarize

- **Node Details:**  

  - **Send email to A.I. to summarize**  
    - *Type & Role:* HTTP Request node, version 4.2; performs a POST request to Openrouter.ai API endpoint to generate AI-driven summaries.  
    - *Configuration:*  
      - URL: `https://openrouter.ai/api/v1/chat/completions`  
      - Method: POST  
      - Body Type: JSON, dynamically constructed using expressions.  
      - The JSON body specifies:  
        - Model: `meta-llama/llama-3.1-70b-instruct:free` (a free AI model)  
        - Messages: A single user message containing a detailed prompt instructing the AI on how to summarize the email.  
        - Prompt includes dynamic insertion of email fields — sender email, subject, and HTML content — URL-encoded to avoid formatting issues.  
      - Authentication: Header Auth with a bearer token for Openrouter API access.  
    - *Expressions / Variables:*  
      - `{{ encodeURIComponent($json.from) }}` for sender email.  
      - `{{ encodeURIComponent($json.subject) }}` for the email subject.  
      - `{{ encodeURIComponent($json.textHtml) }}` for the email content in HTML.  
    - *Input / Output:*  
      - Input: JSON data from Read emails (IMAP).  
      - Output: AI response containing summary text.  
    - *Version Requirements:* HTTP Request node v4.2 or later recommended for better JSON handling.  
    - *Potential Failures:*  
      - API authentication errors if the bearer token is invalid.  
      - Network issues/timeouts contacting Openrouter.ai.  
      - Unexpected response formats or API changes.  
      - Expression errors if email fields are missing or malformed.  
    - *Sub-workflows:* None.

#### 1.3 Messaging Delivery

- **Overview:**  
  This block sends the summarized email content to the user’s Line messenger account via Line’s Messaging API, enabling quick review on mobile or desktop.

- **Nodes Involved:**  
  - Send summarized content to messenger

- **Node Details:**  

  - **Send summarized content to messenger**  
    - *Type & Role:* HTTP Request node, version 4.2; sends a POST request to Line Messaging API to push messages.  
    - *Configuration:*  
      - URL: `https://api.line.me/v2/bot/message/push`  
      - Method: POST  
      - Body Type: JSON, dynamically constructed to include:  
        - `to`: The Line user ID (hardcoded as `U3ec262c49811f30cdc2d2f2b0a0df99a`) — must be replaced with the user’s own Line ID.  
        - `messages`: An array with a single text message containing the AI summary.  
      - The summary text is extracted and sanitized by replacing newline characters with escaped newlines for JSON compliance.  
      - Authentication: Header Auth using a bearer token for Line channel access.  
    - *Expressions / Variables:*  
      - `{{ $json.choices[0].message.content.replace(/\n/g, "\\n") }}` to extract and format the AI summary text.  
    - *Input / Output:*  
      - Input: AI summary JSON from the previous node.  
      - Output: None (final step).  
    - *Version Requirements:* HTTP Request node v4.2 or later recommended.  
    - *Potential Failures:*  
      - Authentication errors due to invalid Line channel access token.  
      - Incorrect or expired tokens causing message sending failure.  
      - Invalid/missing `to` user ID.  
      - API rate limits or network errors.  
    - *Sub-workflows:* None.

---

### 3. Summary Table

| Node Name                      | Node Type                 | Functional Role               | Input Node(s)               | Output Node(s)                    | Sticky Note                                                                                                                              |
|-------------------------------|---------------------------|------------------------------|-----------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Read emails (IMAP)             | EmailReadImap             | Fetch emails from IMAP server | None                        | Send email to A.I. to summarize  | Find your email server's IMAP Settings. - Link for [gmail](https://www.getmailspring.com/setup/access-gmail-via-imap-smtp)               |
| Send email to A.I. to summarize | HttpRequest               | Send emails to Openrouter AI for summarization | Read emails (IMAP)            | Send summarized content to messenger | For the A.I. you can use Openrouter.ai. - Set up a free account - The A.I. model selected is FREE to use. Credentials: Header Auth with Bearer token. |
| Send summarized content to messenger | HttpRequest               | Send AI summary to Line messenger | Send email to A.I. to summarize | None                             | Don't use the official Line node. It's outdated. Credentials: Header Auth with Bearer token. Find token at [Line API console](https://developers.line.biz/console/).|
| Sticky Note                   | StickyNote                | Informational note            | None                        | None                             | ## Summarize emails with A.I. You can find out more about the [use case](https://rumjahn.com/how-a-i-saved-my-kids-school-life-and-my-marriage/) |
| Sticky Note1                  | StickyNote                | Setup tip for IMAP settings   | None                        | None                             | Find your email server's IMAP Settings. - Link for [gmail](https://www.getmailspring.com/setup/access-gmail-via-imap-smtp)               |
| Sticky Note2                  | StickyNote                | Setup tip for Openrouter AI credentials | None                        | None                             | For the A.I. you can use Openrouter.ai. - Set up a free account - The A.I. model selected is FREE to use. Credentials: Header Auth with Bearer token. |
| Sticky Note3                  | StickyNote                | Setup tip for Line messaging credentials | None                        | None                             | Don't use the official Line node. It's outdated. Credentials: Header Auth with Bearer token. Find token at [Line API console](https://developers.line.biz/console/).|

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Read emails (IMAP)" Node**  
   - Type: EmailReadImap  
   - Configure IMAP credentials with your email server (host, port, username, password, SSL as required).  
   - Set "Post Process Action" to "Nothing."  
   - Position node to start the workflow.  

2. **Create "Send email to A.I. to summarize" Node**  
   - Type: HTTP Request  
   - Set Method to POST.  
   - URL: `https://openrouter.ai/api/v1/chat/completions`  
   - Authentication: Header Auth with:  
     - Username: `Authorization`  
     - Password: `Bearer {Your_Openrouter_API_Key}` (replace with your actual API key)  
   - Body Content-Type: JSON  
   - JSON Body:  
     ```json
     {
       "model": "meta-llama/llama-3.1-70b-instruct:free",
       "messages": [
         {
           "role": "user",
           "content": "I want you to read and summarize all the emails. If it's not rimportant, just give me a short summary with less than 10 words. Highlight as important if it is, add an emoji to indicate it is urgent:\nFor the relevant content, find any action items and deadlines. Sometimes I need to sign up before a certain date or pay before a certain date, please highlight that in the summary for me.\nPut the deadline in BOLD at the top. If the email is not important, keep the summary short to 1 sentence only.\nHere's the email content for you to read:\nSender email address: {{ encodeURIComponent($json.from) }}\nSubject: {{ encodeURIComponent($json.subject) }}\n{{ encodeURIComponent($json.textHtml) }}"
         }
       ]
     }
     ```  
   - Use expressions to dynamically insert email data for `from`, `subject`, and `textHtml`.  
   - Connect "Read emails (IMAP)" node's output to this node's input.

3. **Create "Send summarized content to messenger" Node**  
   - Type: HTTP Request  
   - Set Method to POST.  
   - URL: `https://api.line.me/v2/bot/message/push`  
   - Authentication: Header Auth with:  
     - Username: `Authorization`  
     - Password: `Bearer {Your_Line_Channel_Access_Token}` (replace with your actual Line channel access token).  
   - Body Content-Type: JSON  
   - JSON Body:  
     ```json
     {
       "to": "U3ec262c49811f30cdc2d2f2b0a0df99a",
       "messages": [
         {
           "type": "text",
           "text": "{{ $json.choices[0].message.content.replace(/\\n/g, \"\\\\n\") }}"
         }
       ]
     }
     ```  
   - Replace the `"to"` field with your Line user ID (found in Line Developer Console under Messaging API).  
   - Connect the "Send email to A.I. to summarize" node's output to this node's input.

4. **Setup Credentials:**  
   - IMAP credentials for your email provider (e.g., Gmail IMAP settings).  
   - Header Auth credentials for Openrouter API with your API key prefixed by "Bearer ".  
   - Header Auth credentials for Line Messaging API with your channel access token prefixed by "Bearer ".  

5. **Optional: Add Sticky Notes**  
   - Add notes to document setup instructions and links for future reference.

6. **Test the Workflow:**  
   - Execute the workflow manually or set a trigger to run periodically.  
   - Verify emails are fetched, summarized, and pushed to your Line messenger.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| You can find out more about the use case and inspiration behind this workflow.                  | https://rumjahn.com/how-a-i-saved-my-kids-school-life-and-my-marriage/                                    |
| For Gmail IMAP setup, refer to this guide to find your server settings and enable access.      | https://www.getmailspring.com/setup/access-gmail-via-imap-smtp                                            |
| Openrouter.ai credentials require creating a free account and using header auth with Bearer token. | Openrouter.ai official site (https://openrouter.ai/)                                                       |
| Do not use the official Line node in n8n; it is outdated. Use HTTP Request node with header auth instead. | Line Developer Console: https://developers.line.biz/console/ (Messaging API section for channel access token) |

---

This documentation enables users and AI agents to fully understand, reproduce, and modify the workflow while being aware of potential failure points and setup requirements.