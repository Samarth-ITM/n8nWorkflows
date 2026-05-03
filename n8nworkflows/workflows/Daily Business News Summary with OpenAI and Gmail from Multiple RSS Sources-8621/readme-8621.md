Daily Business News Summary with OpenAI and Gmail from Multiple RSS Sources

https://n8nworkflows.xyz/workflows/daily-business-news-summary-with-openai-and-gmail-from-multiple-rss-sources-8621


# Daily Business News Summary with OpenAI and Gmail from Multiple RSS Sources

### 1. Workflow Overview

This workflow automates the daily aggregation, summarization, and emailing of business news collected from multiple RSS feeds using OpenAI’s language model and Gmail. It is designed for users who want a concise daily digest of important business news without manually browsing multiple sources.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Scheduled trigger to start the workflow and RSS feed reading nodes to fetch news items from multiple sources.
- **1.2 Data Preparation:** Merge and deduplicate the aggregated RSS items to create a unique set of news articles.
- **1.3 Validation:** Conditional check to determine if there are new items to process or if the workflow should terminate early.
- **1.4 Limiting and Processing:** Limit the number of news items processed, transform data via custom code, and generate a summary using OpenAI.
- **1.5 Output:** Format the AI-generated summary and send it via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow on a schedule and reads news items from multiple RSS feeds.

- **Nodes Involved:**  
  - Schedule Trigger  
  - FT (RSS Feed Read)  
  - Yahoo Finance (RSS Feed Read)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger  
    - Role: Starts the workflow at scheduled intervals (e.g., daily)  
    - Configuration: Uses default or custom schedule (not explicitly detailed)  
    - Inputs: None  
    - Outputs: Triggers FT and Yahoo Finance nodes  
    - Edge cases: Misconfigured schedule could cause missed or too frequent runs

  - **FT (RSS Feed Read)**  
    - Type: RSS Feed Read  
    - Role: Fetches news items from the Financial Times RSS feed  
    - Configuration: RSS feed URL configured to the FT business news feed  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: News items passed to Merge node (output index 1)  
    - Edge cases: Network timeouts, invalid feed URLs, empty feeds

  - **Yahoo Finance (RSS Feed Read)**  
    - Type: RSS Feed Read  
    - Role: Fetches news items from Yahoo Finance RSS feed  
    - Configuration: RSS feed URL configured to Yahoo Finance business news  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: News items passed to Merge node (output index 0)  
    - Edge cases: Same as FT node

#### 2.2 Data Preparation

- **Overview:**  
  Aggregates multiple RSS feed outputs into a single stream and removes duplicate news items.

- **Nodes Involved:**  
  - Merge  
  - Remove Duplicates

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines RSS items from FT and Yahoo Finance into one list  
    - Configuration: Default merge mode (probably ‘append’)  
    - Inputs: FT (output 1), Yahoo Finance (output 0)  
    - Outputs: Combined list to Remove Duplicates node  
    - Edge cases: Empty inputs, large volume causing performance delays

  - **Remove Duplicates**  
    - Type: Remove Duplicates  
    - Role: Filters out duplicated news entries based on unique identifiers (e.g., link or title)  
    - Configuration: Default deduplication settings (likely based on item link or title)  
    - Inputs: Combined RSS feed items from Merge  
    - Outputs: Deduplicated news items to If node  
    - Edge cases: Poor deduplication criteria causing false positives/negatives

#### 2.3 Validation

- **Overview:**  
  Checks if there are any new news items to process; branches workflow accordingly.

- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - Type: If  
    - Role: Conditional check to determine if news items exist for further processing  
    - Configuration: Condition likely checks if the deduplicated list length > 0  
    - Inputs: Deduplicated news items  
    - Outputs:  
      - True: Proceed to Limit node  
      - False: Proceed to No New Items node  
    - Edge cases: Expression errors if input data format is unexpected

#### 2.4 Limiting and Processing

- **Overview:**  
  Limits the number of news items processed, formats them via code nodes, sends them to OpenAI for summarization, and prepares the final message.

- **Nodes Involved:**  
  - Limit  
  - Code1  
  - Message a model (OpenAI)  
  - Code2

- **Node Details:**

  - **Limit**  
    - Type: Limit  
    - Role: Restricts the number of news items processed to a manageable number (e.g., top 10)  
    - Configuration: Limit count set (exact count not specified)  
    - Inputs: News items from If node (True branch)  
    - Outputs: Limited items to Code1 node  
    - Edge cases: Limit set too low or too high affecting summary quality

  - **Code1**  
    - Type: Code  
    - Role: Transforms or formats limited news items into a prompt suitable for OpenAI  
    - Configuration: Custom JavaScript code (not shown) that likely creates a textual summary prompt  
    - Inputs: Limited news items  
    - Outputs: Formatted text to Message a model node  
    - Edge cases: Code errors, empty input array

  - **Message a model**  
    - Type: OpenAI (LangChain) node  
    - Role: Sends the formatted prompt to OpenAI to generate a summary of the news  
    - Configuration: Uses OpenAI API credentials; prompt constructed from Code1 output  
    - Inputs: Text prompt from Code1  
    - Outputs: AI-generated summary to Code2 node  
    - Version: Requires LangChain OpenAI node version 1.8 or compatible  
    - Edge cases: API rate limits, authentication errors, latency, malformed prompts

  - **Code2**  
    - Type: Code  
    - Role: Finalizes formatting of the AI-generated summary into an email message body  
    - Configuration: Custom JavaScript code (not shown)  
    - Inputs: AI summary from Message a model  
    - Outputs: Final message to Send a message node  
    - Edge cases: Code execution errors, empty AI response

#### 2.5 Output

- **Overview:**  
  Sends the formatted news summary email to the intended recipient via Gmail.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**

  - **Send a message**  
    - Type: Gmail  
    - Role: Sends the email containing the daily business news summary  
    - Configuration: Uses Gmail OAuth2 credentials; email parameters (to, subject, body) set dynamically based on Code2 output  
    - Inputs: Final email message from Code2  
    - Outputs: None (end of workflow)  
    - Edge cases: Authentication errors, quota limits, invalid recipient address

#### 2.6 No New Items Handling

- **Overview:**  
  Handles the case when no new news items are found, preventing unnecessary processing or emails.

- **Nodes Involved:**  
  - No New Items (Code)

- **Node Details:**

  - **No New Items**  
    - Type: Code  
    - Role: Possibly logs or creates a message indicating no new news was found; terminates workflow gracefully  
    - Configuration: Simple JavaScript code (not shown)  
    - Inputs: From If node (False branch)  
    - Outputs: None or workflow termination  
    - Edge cases: None significant; code should safely handle empty input

---

### 3. Summary Table

| Node Name        | Node Type               | Functional Role                         | Input Node(s)                | Output Node(s)            | Sticky Note                          |
|------------------|-------------------------|---------------------------------------|-----------------------------|---------------------------|------------------------------------|
| Schedule Trigger | Schedule Trigger        | Starts workflow on schedule            | None                        | FT, Yahoo Finance          |                                    |
| FT               | RSS Feed Read           | Reads RSS from Financial Times         | Schedule Trigger             | Merge (output 1)           |                                    |
| Yahoo Finance    | RSS Feed Read           | Reads RSS from Yahoo Finance            | Schedule Trigger             | Merge (output 0)           |                                    |
| Merge            | Merge                   | Merges multiple RSS feed outputs        | FT (1), Yahoo Finance (0)   | Remove Duplicates          |                                    |
| Remove Duplicates| Remove Duplicates       | Removes duplicated news items           | Merge                       | If                        |                                    |
| If               | If                      | Checks if new items exist               | Remove Duplicates            | Limit (True), No New Items (False) |                          |
| Limit            | Limit                   | Limits items to process                  | If (True)                   | Code1                     |                                    |
| Code1            | Code                    | Formats news items for AI prompt        | Limit                       | Message a model            |                                    |
| Message a model  | OpenAI (LangChain)      | Generates summary from AI                | Code1                       | Code2                     |                                    |
| Code2            | Code                    | Formats AI summary into email content   | Message a model             | Send a message            |                                    |
| Send a message   | Gmail                   | Sends email with news summary            | Code2                       | None                      |                                    |
| No New Items     | Code                    | Handles case of no new news               | If (False)                  | None                      |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Set the schedule to run daily (or preferred interval).
   - This node will trigger the workflow start.

2. **Create two RSS Feed Read nodes:**
   - Name one "FT":
     - Set RSS URL to Financial Times business news feed.
   - Name the other "Yahoo Finance":
     - Set RSS URL to Yahoo Finance business news feed.
   - Connect the Schedule Trigger output to both these nodes.

3. **Create a Merge node:**
   - Connect "Yahoo Finance" node output (index 0) to Merge input 0.
   - Connect "FT" node output (index 1) to Merge input 1.
   - Set merge mode to "Append" (default).

4. **Create a Remove Duplicates node:**
   - Connect Merge output to Remove Duplicates input.
   - Configure deduplication criteria (usually based on news item link or title).

5. **Create an If node:**
   - Connect Remove Duplicates output to If input.
   - Configure condition to check if the number of items > 0.
     - Expression example: `{{ $items("Remove Duplicates").length > 0 }}`

6. **Create a Limit node:**
   - Connect If node’s ‘True’ output to Limit input.
   - Set limit count (e.g., 10) to restrict the number of news items.

7. **Create a Code node ("Code1"):**
   - Connect Limit output to Code1 input.
   - In Code1, write JavaScript to transform news items into a prompt text for AI summarization.
     - Use fields like title, summary, and link.
     - Concatenate into a single string prompt.

8. **Create an OpenAI node ("Message a model"):**
   - Connect Code1 output to this node.
   - Configure OpenAI credentials.
   - Set model (e.g., gpt-4 or gpt-3.5-turbo).
   - Use the prompt generated by Code1 as the input.
   - Adjust parameters like temperature, max tokens as needed.

9. **Create a second Code node ("Code2"):**
   - Connect OpenAI output to Code2.
   - Format the AI response into an email body (HTML or plain text).

10. **Create a Gmail node ("Send a message"):**
    - Connect Code2 output to Gmail node.
    - Configure Gmail OAuth2 credentials.
    - Set recipient email address, subject (e.g., "Daily Business News Summary"), and body from Code2 output.

11. **Create a Code node ("No New Items"):**
    - Connect If node’s ‘False’ output to this node.
    - Optionally include code to log or notify that no new news was available.
    - No outputs needed; this ends the workflow for that branch.

12. **Final connections and testing:**
    - Confirm all nodes are connected as per above.
    - Test the workflow manually to verify RSS fetching, summarization, and email sending.
    - Adjust parameters or credentials as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                               |
|--------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow uses OpenAI’s GPT models via LangChain integration in n8n for summarization tasks. | OpenAI API documentation: https://platform.openai.com/docs  |
| Gmail node requires OAuth2 credentials properly configured for sending emails from your account. | Gmail API and OAuth setup: https://developers.google.com/gmail/api |
| RSS feed URLs must be verified and kept up-to-date to ensure reliable news fetching.             | Example FT feed: https://www.ft.com/?format=rss               |
| Consider rate limits on OpenAI usage and Gmail sending quotas to avoid failures.                 | OpenAI rate limits: https://platform.openai.com/docs/guides/rate-limits |
| Custom code nodes depend on JavaScript; errors here can break data flow, so validate carefully. | n8n JavaScript documentation: https://docs.n8n.io/nodes/expressions/ |
| Testing with smaller limit counts helps debug before scaling up the number of processed items.   |                                                               |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.