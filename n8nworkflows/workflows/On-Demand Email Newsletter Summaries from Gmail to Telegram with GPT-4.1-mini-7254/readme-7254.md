On-Demand Email Newsletter Summaries from Gmail to Telegram with GPT-4.1-mini

https://n8nworkflows.xyz/workflows/on-demand-email-newsletter-summaries-from-gmail-to-telegram-with-gpt-4-1-mini-7254


# On-Demand Email Newsletter Summaries from Gmail to Telegram with GPT-4.1-mini

### 1. Workflow Overview

This n8n workflow automates the creation and delivery of on-demand email newsletter summaries from Gmail to Telegram, leveraging GPT-4.1-mini for concise topic extraction. Its primary use case is to receive a numeric input via Telegram indicating how many days of emails to summarize, fetch relevant newsletters from Gmail, process each email to extract meaningful topics using AI, merge these topics into a single digest message, and then send the formatted summary back to the user on Telegram.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Receives a Telegram message with the number of days to look back for emails.
- **1.2 Gmail Email Retrieval:** Queries Gmail for emails from specific senders since the specified date.
- **1.3 Email Processing & Extraction:** Iterates over each fetched email, retrieves full message content, and extracts key fields (HTML, subject, sender, date).
- **1.4 AI Summarization:** Sends each email's content to the GPT-4.1-mini model to generate a JSON-formatted summary of newsletter topics.
- **1.5 Topics Aggregation:** Merges all summarized topics from emails into a consolidated list.
- **1.6 Message Formatting:** Creates a formatted Telegram message listing all topics, sanitizes the text for Telegram HTML mode, and splits it into message chunks if too long.
- **1.7 Telegram Delivery:** Sends the final message chunks to the user’s Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures user input from Telegram to determine how many past days of emails to summarize.
- **Nodes Involved:**  
  - Telegram Trigger  
  - Get days (Code node)
- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Trigger node listening for Telegram messages.  
    - Configured to receive "message" updates only and limited to a specific chat ID.  
    - Credentials: Telegram API OAuth2.  
    - Input: Telegram message (expected a number as text).  
    - Output: Passes message text downstream.  
    - Edge cases: Invalid input (non-numeric), unauthorized chat IDs.  
  - **Get days**  
    - Type: Code node, executes JavaScript.  
    - Parses the Telegram message text as integer daysAgo.  
    - Calculates the date string `YYYY/MM/DD` for the Gmail query filter.  
    - Returns an object with `dateString` for further use.  
    - Input: Telegram Trigger output.  
    - Output: dateString parameter.  
    - Edge cases: Non-integer input, negative values.

#### 2.2 Gmail Email Retrieval

- **Overview:** Queries Gmail for all emails from specified senders received after the calculated date.
- **Nodes Involved:**  
  - Get many messages (Gmail node)  
  - Loop Over Items (SplitInBatches node)
- **Node Details:**  
  - **Get many messages**  
    - Type: Gmail node (OAuth2).  
    - Operation: getAll messages matching filter query combining multiple 'from:' senders and after the dateString.  
    - Returns all matching messages.  
    - Input: dateString from Get days.  
    - Output: Array of message summaries (IDs).  
    - Edge cases: Gmail API rate limits, OAuth token expiry, malformed query.  
  - **Loop Over Items**  
    - Type: SplitInBatches node.  
    - Splits the list of Gmail messages into batches of one (default) for sequential processing.  
    - Input: Gmail messages array.  
    - Output: Single message item per iteration.  
    - Edge cases: Empty message list, batch processing errors.

#### 2.3 Email Processing & Extraction

- **Overview:** Fetches full content of each email and extracts relevant data fields.
- **Nodes Involved:**  
  - Get a message (Gmail node)  
  - Get message data (Code node)
- **Node Details:**  
  - **Get a message**  
    - Type: Gmail node, OAuth2.  
    - Operation: get full message by messageId from Loop Over Items.  
    - Returns raw email content including payload.  
    - Input: messageId from Loop Over Items.  
    - Output: Full message JSON.  
    - Edge cases: Message deleted or inaccessible, Gmail API errors.  
  - **Get message data**  
    - Type: Code node.  
    - Parses HTML content from raw Gmail payload, handling nested MIME parts.  
    - Extracts sender’s name (cleans email format), subject, and converts date to DD.MM.YYYY.  
    - Output: Object containing html, subject, from (sender), date (formatted).  
    - Edge cases: Missing HTML part, malformed email content.

#### 2.4 AI Summarization

- **Overview:** Sends each cleaned email content to GPT-4.1-mini for generating JSON summaries of newsletter topics.
- **Nodes Involved:**  
  - Clean (Code node)  
  - Message a model (OpenAI node)
- **Node Details:**  
  - **Clean**  
    - Type: Code node.  
    - Converts date from DD.MM.YYYY to MM.DD format for AI prompt.  
    - Passes html, subject, from, date fields unchanged otherwise.  
    - Filters out emails missing html or date.  
    - Input: Get message data output.  
    - Output: Cleaned JSON for AI consumption.  
  - **Message a model**  
    - Type: OpenAI LangChain node, model GPT-4.1-mini.  
    - Prompt includes instructions to create a JSON with an array of topics: title, descr, subject, from, date.  
    - Includes conditional logic for email source filtering and language.  
    - Input: Clean node output.  
    - Output: JSON with extracted topics per email.  
    - Credentials: OpenAI API key.  
    - Edge cases: API rate limits, malformed response, connection timeouts.

#### 2.5 Topics Aggregation

- **Overview:** Merges all topic arrays from each AI-generated summary into a single consolidated list.
- **Nodes Involved:**  
  - Loop Over Items (continuation)  
  - Merge (Code node)
- **Node Details:**  
  - **Loop Over Items** connects to **Merge** after receiving all AI summaries.  
  - **Merge**  
    - Type: Code node.  
    - Uses flatMap to extract and concatenate all `topics` arrays safely.  
    - Returns a single JSON object with combined `topics` array.  
    - Input: Array of AI-generated summaries.  
    - Output: Single merged topics list.  
    - Edge cases: Missing or empty topics arrays.

#### 2.6 Message Formatting

- **Overview:** Formats the merged topics into a Telegram-compatible message, sanitizes it for HTML parse mode, and splits long messages into chunks.
- **Nodes Involved:**  
  - Create TG message (Code node)  
  - Split (Code node)  
  - Sanitize (Code node)
- **Node Details:**  
  - **Create TG message**  
    - Type: Code node.  
    - Maps each topic to a formatted string with index, bolded title, description, subject, sender, and date.  
    - Joins all topics with double newlines.  
    - Output: Single long string message.  
  - **Split**  
    - Type: Code node.  
    - Splits the long text into chunks of max 3500 characters to comply with Telegram message size limits.  
    - Output: Array of smaller text parts.  
  - **Sanitize**  
    - Type: Code node.  
    - Fixes unbalanced Markdown-like formatting (* and _) by appending missing closing symbols.  
    - Converts *text* to <b>text</b> and _text_ to <i>text</i>.  
    - Escapes HTML entities (&, <, >) to avoid Telegram parse errors.  
    - Ensures the message is safe for Telegram HTML parse mode.  
    - Input: split message parts.  
    - Output: sanitized message parts ready for sending.

#### 2.7 Telegram Delivery

- **Overview:** Sends each sanitized message chunk to the user’s Telegram chat.
- **Nodes Involved:**  
  - Send a message (Telegram node)
- **Node Details:**  
  - **Send a message**  
    - Type: Telegram node.  
    - Sends text messages to the configured chat ID.  
    - Uses HTML parse mode.  
    - Disables web page preview and attribution.  
    - Input: sanitized text chunks.  
    - Output: Confirmation of sent message.  
    - Credentials: Telegram API OAuth2.  
    - Edge cases: Chat ID invalid, message too long despite splitting.

---

### 3. Summary Table

| Node Name          | Node Type                     | Functional Role                      | Input Node(s)         | Output Node(s)         | Sticky Note                                                                                                                      |
|--------------------|-------------------------------|------------------------------------|-----------------------|-----------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger    | Telegram Trigger               | Receives user input from Telegram  | —                     | Get days              | ## Try this out! Send a number to your Telegram bot (e.g., 2) and get a neatly formatted digest of all Gmail newsletters since that date.|
| Get days           | Code                          | Calculates date string from input  | Telegram Trigger      | Get many messages      |                                                                                                                                  |
| Get many messages   | Gmail                         | Retrieves emails from Gmail        | Get days              | Loop Over Items        |                                                                                                                                  |
| Loop Over Items     | SplitInBatches                | Iterates over each email message   | Get many messages      | Get a message, Merge   | ## Iterates over each message                                                                                                   |
| Get a message       | Gmail                         | Gets full email content            | Loop Over Items        | Get message data       | ## Iterates over each message                                                                                                   |
| Get message data    | Code                          | Extracts HTML, subject, sender, date| Get a message         | Clean                 | ## Iterates over each message                                                                                                   |
| Clean              | Code                          | Prepares data for AI model         | Get message data       | Message a model        | ## Clean up the text and forms the final message                                                                                 |
| Message a model     | OpenAI LangChain (GPT-4.1-mini)| Generates JSON summary of topics   | Clean                  | Loop Over Items        | ## Iterates over each message                                                                                                   |
| Merge              | Code                          | Merges all topics arrays into one  | Loop Over Items        | Create TG message      | ## Clean up the text and forms the final message                                                                                 |
| Create TG message   | Code                          | Formats merged topics as Telegram text| Merge              | Split                 | ## Clean up the text and forms the final message                                                                                 |
| Split              | Code                          | Splits long message into chunks    | Create TG message      | Sanitize               | ## Clean up the text and forms the final message                                                                                 |
| Sanitize           | Code                          | Sanitizes message for Telegram HTML| Split                  | Send a message         | ## Clean up the text and forms the final message                                                                                 |
| Send a message     | Telegram                      | Sends formatted messages to Telegram| Sanitize               | —                      | ## Try this out! Send a number to your Telegram bot (e.g., 2) and get a neatly formatted digest of all Gmail newsletters since that date.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates only  
   - Additional Fields: Set `chatIds` to your Telegram user/chat ID (string)  
   - Credentials: Configure Telegram OAuth2 API credentials  
   - Connect output to next node.

2. **Create Code Node "Get days"**
   - Type: Code  
   - JavaScript: Parse `$json.message.text` as integer daysAgo  
   - Compute date `daysAgo` days before today, output as `dateString` in `YYYY/MM/DD`  
   - Connect input from Telegram Trigger.

3. **Create Gmail Node "Get many messages"**
   - Type: Gmail  
   - Operation: getAll  
   - Filters: Use query combining multiple senders with `from:`, exclude some with `-""`, and filter after `{{ $json.dateString }}`  
   - Return all messages  
   - Credentials: Gmail OAuth2 credentials  
   - Connect input from "Get days".

4. **Create SplitInBatches Node "Loop Over Items"**
   - Type: SplitInBatches  
   - Use default batch size (1) for iterating messages one by one  
   - Connect input from "Get many messages".

5. **Create Gmail Node "Get a message"**
   - Type: Gmail  
   - Operation: get by messageId  
   - Use expression to get message ID from current item: `={{ $json.id }}`  
   - Credentials: Gmail OAuth2  
   - Connect input from "Loop Over Items".

6. **Create Code Node "Get message data"**
   - Type: Code  
   - JavaScript: Extract full HTML from Gmail payload, handle nested MIME parts  
   - Extract sender's name cleaned from email format, subject, and format date to DD.MM.YYYY  
   - Connect input from "Get a message".

7. **Create Code Node "Clean"**
   - Type: Code  
   - JavaScript: Convert date from DD.MM.YYYY to MM.DD format, filter out entries missing html or date  
   - Pass through html, subject, from, date fields  
   - Connect input from "Get message data".

8. **Create OpenAI Node "Message a model"**
   - Type: @n8n/n8n-nodes-langchain.openAi  
   - Model: gpt-4.1-mini  
   - Prompt: Provide detailed instructions to analyze newsletters and output JSON with topics array including title, descr, subject, from, date  
   - Use dynamic variables for subject, from, date, html from input  
   - Credentials: OpenAI API key  
   - Connect input from "Clean".

9. **Connect "Message a model" output back to "Loop Over Items"**
   - This closes the batch iteration loop to accumulate all AI responses.

10. **Create Code Node "Merge"**
    - Type: Code  
    - JavaScript: Merge all incoming items' `json.message.content.topics` arrays into one flat array `topics`  
    - Connect input from "Loop Over Items" (after all iterations complete).

11. **Create Code Node "Create TG message"**
    - Type: Code  
    - JavaScript: Map each topic to a formatted string with index, bold title, description, subject, from, date, joined by double newlines  
    - Connect input from "Merge".

12. **Create Code Node "Split"**
    - Type: Code  
    - JavaScript: Split long text into chunks of 3500 characters max to respect Telegram message limits  
    - Connect input from "Create TG message".

13. **Create Code Node "Sanitize"**
    - Type: Code  
    - JavaScript:  
      - Fix unbalanced * and _ by appending missing closing symbols  
      - Convert *text* to `<b>text</b>`, _text_ to `<i>text</i>` with multi-line support  
      - Escape &, <, > characters  
    - Connect input from "Split".

14. **Create Telegram Node "Send a message"**
    - Type: Telegram  
    - Parameters:  
      - Text: `={{ $json.text }}`  
      - Chat ID: your Telegram user/chat ID  
      - Parse Mode: HTML  
      - Disable Web Page Preview: true  
      - Disable Attribution: true  
    - Credentials: Telegram API OAuth2  
    - Connect input from "Sanitize".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Try this out! Send a number to your Telegram bot (e.g., 2) and get a neatly formatted digest of all Gmail newsletters received since that date. Each email is summarized by an LLM into concise topics, merged, chunked, and formatted safely for Telegram HTML mode. | Sticky note on "Telegram Trigger" and "Send a message" nodes.                                             |
| Iterates over each message to process them sequentially in batch mode.                                                                                           | Sticky note on "Loop Over Items", "Get a message", "Get message data", "Message a model" nodes.            |
| Cleans up the text and forms the final message to be sent to Telegram, including merging topics and formatting.                                                  | Sticky note on "Clean", "Merge", "Create TG message", "Split", "Sanitize" nodes.                           |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.