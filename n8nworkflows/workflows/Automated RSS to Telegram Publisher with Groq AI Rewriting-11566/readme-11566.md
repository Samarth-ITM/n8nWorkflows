Automated RSS to Telegram Publisher with Groq AI Rewriting

https://n8nworkflows.xyz/workflows/automated-rss-to-telegram-publisher-with-groq-ai-rewriting-11566


# Automated RSS to Telegram Publisher with Groq AI Rewriting

### 1. Workflow Overview

This workflow automates the process of fetching new articles from multiple RSS feeds, refining the content using a Groq AI language model, and sending polished news posts to a private Telegram chat. The main use case is to provide consistent, visually appealing Telegram updates without manual intervention.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & RSS Feed Retrieval:** Periodic fetching of RSS feed URLs from a data table, iterating over each feed to retrieve new posts.
- **1.2 Latest Post Filtering & Deduplication:** Extracting only the most recent article from each feed and checking whether it has already been processed, using a second data table to track posted links.
- **1.3 AI-Powered Content Rewriting:** Using Groq's free-tier AI model to rewrite and polish the article text, including shortening if necessary and adding Telegram-specific formatting.
- **1.4 Publishing to Telegram:** Sending the AI-enhanced post to a specified Telegram user via a Telegram bot.
- **1.5 Data Management:** Adding newly processed links to the tracking data table to avoid reposting.
  
The workflow also includes descriptive sticky notes explaining each logical block and operational instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & RSS Feed Retrieval

**Overview:**  
This block initiates the workflow every 30 minutes, retrieves the list of RSS feed URLs from the `rss_list` data table, and iterates over each feed to fetch its posts.

**Nodes Involved:**  
- Every 30 minutes  
- Get RSS list  
- For each RSS  
- Read RSS links  

**Node Details:**

- **Every 30 minutes**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow execution at 30-minute intervals.  
  - Configuration: Interval set to 30 minutes.  
  - Input: None (trigger)  
  - Output: Triggers "Get RSS list".  
  - Edge cases: Workflow may miss executions if n8n instance is down.  

- **Get RSS list**  
  - Type: Data Table (Get)  
  - Role: Retrieves all RSS feed URLs from the `rss_list` table.  
  - Configuration: Operation "get", returns all rows.  
  - Input: Trigger from Schedule.  
  - Output: Provides list data to "For each RSS".  
  - Edge cases: Empty or missing table results in no feeds processed.  

- **For each RSS**  
  - Type: SplitInBatches  
  - Role: Iterates over each RSS feed URL individually to process them sequentially.  
  - Configuration: Default batch options (default batch size 1).  
  - Input: Array of RSS feeds from "Get RSS list".  
  - Output: Each feed URL passed to "Read RSS links".  
  - Edge cases: Large number of feeds could slow processing.  

- **Read RSS links**  
  - Type: RSS Feed Read  
  - Role: Reads the RSS feed URL passed to it and fetches all current posts.  
  - Configuration: URL dynamically set from current batch item (`{{$json.rss}}`).  
  - Input: Single RSS feed URL from "For each RSS".  
  - Output: List of posts from the feed to "Get only latest link".  
  - Edge cases: Feed may be unreachable or malformed; no posts returned.  

---

#### 1.2 Latest Post Filtering & Deduplication

**Overview:**  
From the fetched posts, only the latest article is selected. The workflow checks if this article’s link has already been processed by looking it up in the `post_links` data table to avoid reposting duplicates.

**Nodes Involved:**  
- Get only latest link  
- If link not in table  

**Node Details:**

- **Get only latest link**  
  - Type: Limit  
  - Role: Selects only the first (latest) post from the RSS feed items.  
  - Configuration: Default limit 1 (no offset).  
  - Input: List of posts from "Read RSS links".  
  - Output: Single post item to "If link not in table".  
  - Edge cases: Empty input results in no output, skipping further processing.  

- **If link not in table**  
  - Type: Data Table (Row Not Exists)  
  - Role: Checks if the post’s link is absent from the `post_links` table to determine if it is new.  
  - Configuration: Filter condition where `link == {{$json.link}}`, operation "rowNotExists".  
  - Input: Single latest post from "Get only latest link".  
  - Output: If new (not existing), proceeds to "Add link".  
  - Edge cases: Table access errors or malformed link data may cause issues.  

---

#### 1.3 AI-Powered Content Rewriting

**Overview:**  
New posts are sent to a Groq AI chat model that rewrites and formats the content to be visually appealing and Telegram-ready. The text is shortened if over 700 characters and enhanced with formatting and emojis.

**Nodes Involved:**  
- Add link  
- Groq Chat Model  
- Rewrite Post  

**Node Details:**

- **Add link**  
  - Type: Data Table (Append)  
  - Role: Adds the new post’s link to the `post_links` table to mark it as processed.  
  - Configuration: Appends the field `link` with `{{$json.link}}`.  
  - Input: New post data from "If link not in table".  
  - Output: Passes to "Rewrite Post".  
  - Edge cases: Data table write failures or duplicate insertion.  

- **Groq Chat Model**  
  - Type: Langchain AI Chat Model (Groq)  
  - Role: Provides AI language model service for rewriting content.  
  - Configuration: Model set to `groq/compound-mini`, no additional options.  
  - Credentials: Requires Groq API token.  
  - Input: Invoked by "Rewrite Post" as AI model node.  
  - Output: AI-generated rewritten text.  
  - Edge cases: API rate limits, token invalidity, model errors, or timeouts.  

- **Rewrite Post**  
  - Type: Langchain Agent  
  - Role: Sends the latest article content to the Groq Chat Model with a detailed system prompt instructing formatting and shortening rules.  
  - Configuration: System message includes instructions to produce a polished Telegram-ready news post with title and content, respecting character limits and formatting.  
  - Input: Data from "Add link", AI model reference "Groq Chat Model".  
  - Output: Rewritten post text passed to "Send post to user".  
  - Edge cases: Prompt rendering errors, empty inputs, or AI service failures.  

---

#### 1.4 Publishing to Telegram

**Overview:**  
The polished post text is sent to a specified Telegram user via a Telegram bot.

**Nodes Involved:**  
- Send post to user  

**Node Details:**

- **Send post to user**  
  - Type: Telegram  
  - Role: Sends the final formatted post to a Telegram chat.  
  - Configuration:  
    - Text is set dynamically from `{{$json.output}}` (AI output).  
    - Chat ID is fixed to a Telegram user ID (271293613).  
  - Credentials: Requires Telegram bot OAuth2 token.  
  - Input: Rewritten post text from "Rewrite Post".  
  - Output: Loops back to "For each RSS" to process next feed.  
  - Edge cases: Telegram API errors, invalid bot token, chat ID errors, message length limits.  

---

#### 1.5 Data Management & Workflow Finalization

**Overview:**  
This block manages data table updates and signals the end of processing for each batch/feed.

**Nodes Involved:**  
- Finish workflow  

**Node Details:**

- **Finish workflow**  
  - Type: No Operation (NoOp)  
  - Role: Marks the end of processing for each RSS feed batch.  
  - Configuration: Empty, just a terminator.  
  - Input: From "For each RSS" for consumed batches.  
  - Output: None.  
  - Edge cases: None.  

---

### 3. Summary Table

| Node Name           | Node Type                            | Functional Role                                  | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                           |
|---------------------|------------------------------------|-------------------------------------------------|-----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Every 30 minutes     | Schedule Trigger                   | Triggers workflow every 30 minutes               | -                           | Get RSS list               | **1. Get RSS posts** Every 30 minutes, fetch posts from each RSS feed stored in the rss_list table.                   |
| Get RSS list         | Data Table (Get)                  | Retrieves RSS feed URLs from rss_list table      | Every 30 minutes            | For each RSS               |                                                                                                                       |
| For each RSS         | SplitInBatches                   | Iterates over each RSS feed URL                   | Get RSS list                | Read RSS links, Finish workflow |                                                                                                                       |
| Read RSS links       | RSS Feed Read                    | Reads posts from the given RSS feed URL           | For each RSS                | Get only latest link       |                                                                                                                       |
| Get only latest link | Limit                           | Selects the latest post from RSS feed items       | Read RSS links              | If link not in table       | **2. Process latest links** Retrieve only the latest post from each RSS feed. Check whether the post’s link exists in the post_links table, and add it if it’s not found. |
| If link not in table | Data Table (Row Not Exists)       | Checks if post’s link is new (not in post_links) | Get only latest link        | Add link                   |                                                                                                                       |
| Add link             | Data Table (Append)               | Adds new post link to post_links table            | If link not in table        | Rewrite Post               |                                                                                                                       |
| Groq Chat Model      | Langchain AI Chat Model (Groq)    | Provides AI rewriting model                        | Invoked by Rewrite Post     | Rewrite Post (AI output)   | **3. Rewrite content** Free-tier Groq AI rewrites the post and shortens it when necessary.                            |
| Rewrite Post         | Langchain Agent                  | Sends post content to AI for rewriting            | Add link, Groq Chat Model   | Send post to user          |                                                                                                                       |
| Send post to user    | Telegram                        | Sends rewritten post to Telegram chat             | Rewrite Post                | For each RSS               | **4. Send to Telegram** Each enhanced RSS post is sent to your private Telegram messages.                             |
| Finish workflow      | No Operation (NoOp)              | Marks end of feed processing batch                 | For each RSS                | -                          |                                                                                                                       |
| Sticky Note          | Sticky Note                      | Descriptive note                                  | -                           | -                          | **Telegram RSS Autoposter** This workflow automatically pulls new articles from RSS feeds and sends AI-enhanced summaries to Telegram. Requires Groq API and Telegram bot. |
| Sticky Note1         | Sticky Note                      | Describes RSS fetching block                      | -                           | -                          | **1. Get RSS posts** Every 30 minutes, fetch posts from each RSS feed stored in the rss_list table.                   |
| Sticky Note2         | Sticky Note                      | Describes AI rewriting block                       | -                           | -                          | **3. Rewrite content** Free-tier Groq AI rewrites the post and shortens it when necessary.                            |
| Sticky Note3         | Sticky Note                      | Describes Telegram sending block                   | -                           | -                          | **4. Send to Telegram** Each enhanced RSS post is sent to your private Telegram messages.                             |
| Sticky Note4         | Sticky Note                      | Describes latest post filtering and deduplication block | -                     | -                          | **2. Process latest links** Retrieve only the latest post from each RSS feed. Check whether the post’s link exists in the post_links table, and add it if it’s not found. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Name: `Every 30 minutes`  
   - Set interval to 30 minutes.  

2. **Create a Data Table node to fetch RSS list:**  
   - Type: Data Table (Get)  
   - Name: `Get RSS list`  
   - Set operation to "get" and enable "return all".  
   - Select the data table with RSS feeds (`rss_list`).  

3. **Create a SplitInBatches node:**  
   - Type: SplitInBatches  
   - Name: `For each RSS`  
   - Connect "Get RSS list" output to this node input.  
   - Use default batch size (1).  

4. **Create an RSS Feed Read node:**  
   - Type: RSS Feed Read  
   - Name: `Read RSS links`  
   - Configure URL as an expression: `{{$json.rss}}` (current batch item RSS feed URL).  
   - Connect "For each RSS" output to this node input.  

5. **Create a Limit node:**  
   - Type: Limit  
   - Name: `Get only latest link`  
   - Set limit to 1 to keep only the latest post.  
   - Connect "Read RSS links" output to this node input.  

6. **Create a Data Table node to check for existing links:**  
   - Type: Data Table (Row Not Exists)  
   - Name: `If link not in table`  
   - Set filter condition: key `link` equals expression `{{$json.link}}`  
   - Operation: "rowNotExists"  
   - Select the data table `post_links`.  
   - Connect "Get only latest link" output to this node input.  

7. **Create a Data Table node to add new links:**  
   - Type: Data Table (Append)  
   - Name: `Add link`  
   - Map the field `link` to value `{{$json.link}}`.  
   - Select the data table `post_links`.  
   - Connect "If link not in table" output to this node input.  

8. **Create a Langchain AI Chat Model node:**  
   - Type: Langchain AI Chat Model (Groq)  
   - Name: `Groq Chat Model`  
   - Model: `groq/compound-mini`  
   - Credentials: Link your Groq API token here.  

9. **Create a Langchain Agent node for rewriting:**  
   - Type: Langchain Agent  
   - Name: `Rewrite Post`  
   - Set prompt type to "define".  
   - Configure the system message with instructions:  
     ```
     You handle news formatting for this Telegram channel.
     Your role is to make each news post look polished and visually appealing, without altering the actual information too much.
     If a text is longer than 700 characters, shorten it carefully so the essential meaning stays intact and the final version fits the limit.
     Feel free to use Telegram’s formatting tools — bold, italic, and fitting emojis — to enhance readability.
     Every result you produce should be a clean, publication-ready news post.
     Create a beautifully formatted version of the news titled:
     "{{ $('Get only latest link').item.json.title }}"
     ---
     {{ $('Get only latest link').item.json.content }}
     ---
     The response should be a ready-to-publish text.
     ```
   - Set the text input to empty `[empty]` or as required by the node.  
   - Connect the AI model reference to `Groq Chat Model`.  
   - Connect "Add link" output to this node input.  

10. **Create a Telegram node:**  
    - Type: Telegram  
    - Name: `Send post to user`  
    - Text: Expression `{{$json.output}}` (AI rewritten post)  
    - Chat ID: Set your Telegram user ID (e.g., `271293613`).  
    - Credentials: Use Telegram bot credentials (OAuth2 token).  
    - Connect "Rewrite Post" output to this node input.  

11. **Connect the Telegram node back to "For each RSS":**  
    - This completes the loop to process all feeds.  

12. **Create a No Operation node:**  
    - Type: NoOp  
    - Name: `Finish workflow`  
    - Connect one output of "For each RSS" to this node, representing end of batch processing.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                            | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Workflow requires two n8n data tables: `rss_list` with a column `rss` containing RSS feed URLs, and `post_links` with a column `link` to track processed articles.                                                                                      | Data Tables setup in n8n                                                                                         |
| Groq API token is needed for AI rewriting; it uses the free-tier model `groq/compound-mini`.                                                                                                                                                              | Groq API documentation                                                                                           |
| Telegram bot token and your Telegram user ID are required for sending messages.                                                                                                                                                                           | Telegram Bot API documentation                                                                                   |
| The AI prompt instructs rewriting with Telegram markdown formatting and emoji usage, with a strict character limit of 700 characters to ensure posts fit Telegram message constraints.                                                                      | Prompt embedded in `Rewrite Post` node                                                                           |
| Sticky notes within the workflow provide helpful explanations of each block and overall workflow purpose, useful for onboarding new users or maintenance.                                                                                                | Visible inside the workflow editor                                                                               |
| This workflow is ideal for users wanting automated, polished Telegram news updates aggregated from multiple RSS feeds without manual intervention.                                                                                                       | Use case summary                                                                                                |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.