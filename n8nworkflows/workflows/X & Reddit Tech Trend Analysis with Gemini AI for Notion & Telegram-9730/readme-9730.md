X & Reddit Tech Trend Analysis with Gemini AI for Notion & Telegram

https://n8nworkflows.xyz/workflows/x---reddit-tech-trend-analysis-with-gemini-ai-for-notion---telegram-9730


# X & Reddit Tech Trend Analysis with Gemini AI for Notion & Telegram

### 1. Workflow Overview

This workflow automates the process of analyzing technology trends from two major social media platforms: X (formerly Twitter) and Reddit. It leverages Google Gemini AI models via LangChain integration to process and summarize data, then outputs the insights to Notion and Telegram for documentation and notification purposes.

The workflow is structured into the following logical blocks:

- **1.1 Trigger and Initialization:** Periodic scheduling triggers the workflow, setting up initial data fields.
- **1.2 Data Collection:** Multiple Reddit nodes and an X (Twitter) node collect posts and tweets relevant to the technology trends.
- **1.3 AI Processing:** Google Gemini AI models process the collected data; an AI Agent orchestrates the flow, including parsing and structuring outputs.
- **1.4 Post-Processing and Messaging:** JavaScript nodes manipulate data, then the processed insights are sent as messages via Telegram.
- **1.5 Notion Integration:** The workflow creates and appends pages/blocks in Notion, archiving the analyzed trend information.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

- **Overview:**  
  This block initiates the workflow on a schedule and sets or edits necessary fields to prepare for data retrieval and processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Edit Fields2

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow at defined time intervals (default schedule parameters assumed).  
    - Config: Empty parameters, likely default schedule (e.g., daily or hourly).  
    - Inputs: None  
    - Outputs: To Edit Fields2  
    - Edge Cases: Missed trigger due to downtime or scheduling conflicts.  

  - **Edit Fields2**  
    - Type: Set Node  
    - Role: Initializes or modifies data fields for subsequent nodes.  
    - Config: No explicit parameters shown; likely sets initial variables or context for AI processing.  
    - Inputs: From Schedule Trigger  
    - Outputs: To AI Agent2  
    - Edge Cases: Failure if expected fields are missing or incorrectly set.

---

#### 2.2 Data Collection

- **Overview:**  
  This block collects raw data from social media platforms X (Twitter) and Reddit, pulling many posts/tweets relevant to tech trends.

- **Nodes Involved:**  
  - Search Tweets in X  
  - Get many posts in Reddit  
  - Get many posts in Reddit1  
  - Get many posts in Reddit2  
  - Get many posts in Reddit3

- **Node Details:**  

  - **Search Tweets in X**  
    - Type: Twitter Tool  
    - Role: Queries Twitter (X) for tweets matching certain search criteria relevant to tech trends.  
    - Config: Parameters likely include search keywords, date ranges, tweet count limits.  
    - Inputs: Passed as AI tools input to AI Agent2 (ai_tool connection).  
    - Outputs: Data sent to AI Agent2 for processing.  
    - Edge Cases: Twitter API rate limits, authorization errors, empty or malformed queries.

  - **Get many posts in Reddit**, **Get many posts in Reddit1**, **Get many posts in Reddit2**, **Get many posts in Reddit3**  
    - Type: Reddit Tool  
    - Role: Multiple nodes query Reddit for posts across different subreddits or topics related to technology trends.  
    - Config: Each likely configured with different subreddit names, keywords, or sorting methods (hot, new, top).  
    - Inputs: Passed as AI tools input to AI Agent2 (ai_tool connection).  
    - Outputs: Data sent to AI Agent2 for aggregation and analysis.  
    - Edge Cases: Reddit API limits, subreddit bans, empty results, connectivity issues.

---

#### 2.3 AI Processing

- **Overview:**  
  The core AI processing uses Google Gemini Chat models and LangChain's AI Agent to interpret, summarize, and structure the collected social media data.

- **Nodes Involved:**  
  - AI Agent2  
  - Google Gemini Chat Model4  
  - Google Gemini Chat Model5  
  - Structured Output Parser2

- **Node Details:**  

  - **AI Agent2**  
    - Type: LangChain Agent  
    - Role: Orchestrates AI tasks, feeding data to language models and parsing results. It receives inputs from Reddit and Twitter data nodes and uses AI language models to process them.  
    - Config: Uses connected AI language models and output parsers via specialized input/output connections.  
    - Inputs: Multiple ai_tool inputs (from Reddit and Twitter nodes), ai_languageModel input (Google Gemini Chat Model4), ai_outputParser input (Structured Output Parser2).  
    - Outputs: Main output to "Code in JavaScript" node for further processing.  
    - Edge Cases: AI model API errors, parsing failures, data format mismatches, rate limits.

  - **Google Gemini Chat Model4**  
    - Type: Google Gemini Chat Language Model  
    - Role: Provides conversational AI capabilities to process raw data chunks.  
    - Config: Presumably set up with credentials for Google Gemini API and configured for chat tasks.  
    - Inputs: Connected as ai_languageModel input to AI Agent2.  
    - Outputs: Responses fed back to AI Agent2.  
    - Edge Cases: API quota limits, network errors, unexpected model output.

  - **Google Gemini Chat Model5**  
    - Type: Google Gemini Chat Language Model  
    - Role: Used for final AI output generation before structured parsing.  
    - Config: Similar to Model4 but probably with different prompt or parameters to produce structured summaries.  
    - Inputs: Output connected to Structured Output Parser2.  
    - Outputs: Passes AI-generated structured text to parser.  
    - Edge Cases: Same as Model4.

  - **Structured Output Parser2**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses AI-generated text into structured data formats suitable for downstream use.  
    - Config: Uses predefined parsing templates or JSON schemas.  
    - Inputs: From Google Gemini Chat Model5.  
    - Outputs: Back into AI Agent2 as structured data.  
    - Edge Cases: Parsing errors if AI output deviates from expected format.

---

#### 2.4 Post-Processing and Messaging

- **Overview:**  
  JavaScript code nodes transform the AI output for messaging; Telegram node sends summarized insights as text messages.

- **Nodes Involved:**  
  - Code in JavaScript  
  - Send a text message1  
  - Code in JavaScript1

- **Node Details:**  

  - **Code in JavaScript**  
    - Type: Code Node (JavaScript)  
    - Role: Processes or formats AI Agent outputs before sending messages.  
    - Config: Custom JavaScript scripts to prepare text content, e.g., formatting or filtering.  
    - Inputs: From AI Agent2 main output.  
    - Outputs: To Send a text message1.  
    - Edge Cases: Runtime errors, unexpected data structure.

  - **Send a text message1**  
    - Type: Telegram Node  
    - Role: Sends text messages containing trend analysis summaries to configured Telegram channels or users.  
    - Config: Connected Telegram bot credentials and target chat/channel IDs.  
    - Inputs: From Code in JavaScript.  
    - Outputs: To Code in JavaScript1 for further processing.  
    - Edge Cases: Telegram API rate limits, invalid chat IDs, bot permission errors.

  - **Code in JavaScript1**  
    - Type: Code Node (JavaScript)  
    - Role: Additional processing of messages or preparation for Notion integration.  
    - Config: Likely formats or extracts data for Notion pages.  
    - Inputs: From Send a text message1.  
    - Outputs: To Create a page (Notion).  
    - Edge Cases: Script errors, empty input data.

---

#### 2.5 Notion Integration

- **Overview:**  
  Creates and appends pages/blocks in Notion to store the analyzed trend data for record-keeping and further reference.

- **Nodes Involved:**  
  - Create a page  
  - Append a block

- **Node Details:**  

  - **Create a page**  
    - Type: Notion Node  
    - Role: Creates a new page in a specified Notion database or workspace, representing a summarized tech trend report.  
    - Config: Requires Notion API credentials, target database or parent page ID, and properties setup.  
    - Inputs: From Code in JavaScript1.  
    - Outputs: To Append a block.  
    - Edge Cases: Notion API rate limits, permission errors, invalid page properties.

  - **Append a block**  
    - Type: Notion Node  
    - Role: Adds content blocks (text, lists, etc.) to the newly created Notion page with detailed insights.  
    - Config: Uses page ID from previous node and block content formatted in expected Notion structures.  
    - Inputs: From Create a page.  
    - Outputs: None (end of workflow).  
    - Edge Cases: Block content format errors, API timeouts.

---

### 3. Summary Table

| Node Name                | Node Type                               | Functional Role                           | Input Node(s)                     | Output Node(s)                  | Sticky Note |
|--------------------------|---------------------------------------|-----------------------------------------|----------------------------------|--------------------------------|-------------|
| Schedule Trigger          | Schedule Trigger                      | Initiate workflow on schedule            | None                             | Edit Fields2                   |             |
| Edit Fields2             | Set                                  | Initialize or edit input data fields     | Schedule Trigger                 | AI Agent2                     |             |
| AI Agent2                | LangChain Agent                      | Orchestrates AI processing and parsing   | Edit Fields2, Reddit, Twitter    | Code in JavaScript             |             |
| Search Tweets in X       | Twitter Tool                        | Collect tweets from X                     | None (input to AI Agent2)        | AI Agent2 (ai_tool)            |             |
| Get many posts in Reddit  | Reddit Tool                        | Fetch posts from Reddit                   | None (input to AI Agent2)        | AI Agent2 (ai_tool)            |             |
| Get many posts in Reddit1 | Reddit Tool                        | Fetch posts from Reddit (additional)     | None (input to AI Agent2)        | AI Agent2 (ai_tool)            |             |
| Get many posts in Reddit2 | Reddit Tool                        | Fetch posts from Reddit (additional)     | None (input to AI Agent2)        | AI Agent2 (ai_tool)            |             |
| Get many posts in Reddit3 | Reddit Tool                        | Fetch posts from Reddit (additional)     | None (input to AI Agent2)        | AI Agent2 (ai_tool)            |             |
| Google Gemini Chat Model4 | Google Gemini Chat Model             | AI language model for initial processing | Connected to AI Agent2           | AI Agent2 (ai_languageModel)  |             |
| Google Gemini Chat Model5 | Google Gemini Chat Model             | AI language model for final output        | None (feeds Structured Output Parser2) | Structured Output Parser2 |             |
| Structured Output Parser2 | Structured Output Parser             | Parses AI output into structured format  | Google Gemini Chat Model5        | AI Agent2 (ai_outputParser)   |             |
| Code in JavaScript       | Code (JavaScript)                   | Format/process AI output for messaging   | AI Agent2                       | Send a text message1          |             |
| Send a text message1     | Telegram                            | Send Telegram messages with insights     | Code in JavaScript              | Code in JavaScript1           |             |
| Code in JavaScript1      | Code (JavaScript)                   | Format/process message for Notion        | Send a text message1            | Create a page                |             |
| Create a page            | Notion                             | Create a new Notion page for insights    | Code in JavaScript1             | Append a block               |             |
| Append a block           | Notion                             | Append content blocks to Notion page     | Create a page                  | None                        |             |
| Sticky Note              | Sticky Note                        | Visual annotation                        | None                          | None                        |             |
| Sticky Note1             | Sticky Note                        | Visual annotation                        | None                          | None                        |             |
| Sticky Note2             | Sticky Note                        | Visual annotation                        | None                          | None                        |             |
| Sticky Note3             | Sticky Note                        | Visual annotation                        | None                          | None                        |             |
| Sticky Note4             | Sticky Note                        | Visual annotation                        | None                          | None                        |             |
| Sticky Note5             | Sticky Note                        | Visual annotation                        | None                          | None                        |             |
| Sticky Note6             | Sticky Note                        | Visual annotation                        | None                          | None                        |             |
| Sticky Note7             | Sticky Note                        | Visual annotation                        | None                          | None                        |             |
| Sticky Note8             | Sticky Note                        | Visual annotation                        | None                          | None                        |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Parameters: Use default or configure desired periodicity (e.g., daily at a specific time).  
   - Connect output to Edit Fields2.

2. **Create a Set node named "Edit Fields2"**  
   - Purpose: Initialize any required variables or fields for AI processing.  
   - Configure to set key-value pairs needed downstream (e.g., search terms, date ranges).  
   - Connect output to AI Agent2.

3. **Create Reddit Tool nodes (four instances):**  
   - Names: "Get many posts in Reddit", "Get many posts in Reddit1", "Get many posts in Reddit2", "Get many posts in Reddit3".  
   - Configure each with appropriate subreddit names, search queries, and post limits.  
   - Ensure Reddit API credentials are set in n8n credentials manager.  
   - No direct input connections; these will connect via ai_tool input to AI Agent2.

4. **Create a Twitter Tool node named "Search Tweets in X"**  
   - Configure search parameters (keywords, date range, tweet count).  
   - Set up Twitter API credentials (OAuth2).  
   - No direct input connection; connect as ai_tool input to AI Agent2.

5. **Create Google Gemini Chat Model nodes (two instances):**  
   - Names: "Google Gemini Chat Model4" and "Google Gemini Chat Model5".  
   - Configure Google Gemini API credentials.  
   - Model4 connects as ai_languageModel input to AI Agent2.  
   - Model5 feeds output to Structured Output Parser2.

6. **Create Structured Output Parser node named "Structured Output Parser2"**  
   - Configure parsing templates or schemas suitable for the expected AI output format.  
   - Connect input from Google Gemini Chat Model5.  
   - Connect output to AI Agent2 as ai_outputParser.

7. **Create LangChain Agent node named "AI Agent2"**  
   - Configure agent with references to ai_tool inputs (Reddit and Twitter nodes), ai_languageModel input (Google Gemini Chat Model4), and ai_outputParser (Structured Output Parser2).  
   - Connect input from Edit Fields2.  
   - Connect main output to "Code in JavaScript".

8. **Create JavaScript Code node named "Code in JavaScript"**  
   - Write scripts to format or filter AI Agent output for messaging.  
   - Input: AI Agent2 main output.  
   - Output: Connect to "Send a text message1".

9. **Create Telegram node named "Send a text message1"**  
   - Configure Telegram bot credentials and target chat ID(s).  
   - Input: From "Code in JavaScript".  
   - Output: Connect to "Code in JavaScript1".

10. **Create JavaScript Code node named "Code in JavaScript1"**  
    - Further process message data for Notion integration.  
    - Input: From "Send a text message1".  
    - Output: Connect to "Create a page".

11. **Create Notion node named "Create a page"**  
    - Configure Notion API credentials.  
    - Set target database or parent page ID.  
    - Map incoming data fields to Notion page properties.  
    - Input: From "Code in JavaScript1".  
    - Output: Connect to "Append a block".

12. **Create Notion node named "Append a block"**  
    - Configure to append text or content blocks to the newly created page.  
    - Input: From "Create a page".  
    - Output: None (end of workflow).

13. **Optional: Add Sticky Notes**  
    - Use sticky notes for documentation and visual grouping within the n8n editor.

---

### 5. General Notes & Resources

| Note Content                                                    | Context or Link                                      |
|----------------------------------------------------------------|-----------------------------------------------------|
| Workflow integrates Google Gemini AI models via LangChain for advanced AI processing. | Useful for workflows requiring powerful AI summarization and structured output. |
| Telegram node requires bot token and chat ID for message delivery. | Telegram Bot API documentation: https://core.telegram.org/bots/api |
| Notion integration requires API key and database/page IDs with proper permissions. | Notion API docs: https://developers.notion.com/ |
| Reddit and Twitter API credentials must be configured with appropriate scopes and rate limits considered. | Twitter API v2 documentation: https://developer.twitter.com/en/docs/twitter-api |
| Scheduling node should be adjusted to avoid API rate limits and workflow overlaps. | n8n Scheduling docs: https://docs.n8n.io/nodes/n8n-nodes-base.schedu letrigger/ |

---

This comprehensive document enables understanding, reproduction, and modification of the "X & Reddit Tech Trend Analysis with Gemini AI for Notion & Telegram" workflow, covering each node’s role, configuration, and integration dependencies.