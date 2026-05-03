Daily News Summarization with OpenAI & Perplexity AI Delivered via Zalo & Telegram

https://n8nworkflows.xyz/workflows/daily-news-summarization-with-openai---perplexity-ai-delivered-via-zalo---telegram-7627


# Daily News Summarization with OpenAI & Perplexity AI Delivered via Zalo & Telegram

---
### 1. Workflow Overview

This workflow automates the daily summarization of news articles gathered from multiple RSS feeds, leveraging both OpenAI and Perplexity AI for content analysis and summarization. The summarized news is then delivered to users via Zalo and Telegram messaging platforms.

The workflow is structured into the following logical blocks:

- **1.1 Trigger and RSS Feed Collection**: Activation by schedule or messaging platform triggers, followed by concurrent reading from four distinct RSS news feeds.

- **1.2 Data Preprocessing and Filtering**: Standardizing and selecting relevant fields from each RSS feed, merging them, filtering to retain only recent articles (published within the last day), and limiting the dataset to the 20 most recent items.

- **1.3 Aggregation and AI Summarization**: Aggregating filtered news items and sending them to OpenAI for summarization into a concise, Vietnamese-language summary. A Perplexity AI node also provides an alternative financial news summary.

- **1.4 Message Delivery**: Sending the AI-generated summaries to users via Zalo (both individual and group messages) and Telegram channels.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and RSS Feed Collection

**Overview:**  
This block initiates the workflow either on a scheduled interval or when a message is received via Telegram or Zalo. It then concurrently fetches news articles from four different RSS feeds.

**Nodes Involved:**  
- Schedule Trigger  
- Telegram Trigger  
- Zalo Trigger  
- My RSS 01  
- My RSS 02  
- My RSS 03  
- My RSS 04

**Node Details:**

- **Schedule Trigger**  
  - *Type*: Schedule Trigger  
  - *Role*: Automatically triggers the workflow at defined intervals (default is every minute as per configuration).  
  - *Config*: Default interval with no custom scheduling filters.  
  - *Input*: None  
  - *Output*: Triggers downstream RSS feed reads.  
  - *Edge Cases*: If n8n is down at scheduled time, trigger may be missed; no retries configured.

- **Telegram Trigger**  
  - *Type*: Telegram Trigger  
  - *Role*: Starts workflow upon receiving a Telegram message.  
  - *Config*: Listens for 'message' updates only. Requires Telegram API credentials.  
  - *Input*: Incoming Telegram messages.  
  - *Output*: Triggers RSS feed reading nodes.  
  - *Edge Cases*: Requires valid webhook setup and active Telegram bot.

- **Zalo Trigger**  
  - *Type*: Custom Zalo User Trigger  
  - *Role*: Triggers workflow on Zalo event type 1 (e.g., incoming message).  
  - *Config*: Uses OAuth2 credentials for Zalo User account.  
  - *Input*: Incoming Zalo messages/events.  
  - *Output*: Triggers RSS feed reading nodes.  
  - *Edge Cases*: OAuth token expiry or webhook issues may prevent triggering.

- **My RSS 01 to My RSS 04**  
  - *Type*: RSS Feed Read  
  - *Role*: Fetch news articles from specified RSS URLs.  
  - *Config*: Each node targets a specific RSS URL; some specify SSL validation ignoring.  
  - *Input*: Trigger from any of the trigger nodes.  
  - *Output*: Raw RSS items passed to subsequent ‘Edit Fields’ nodes.  
  - *Edge Cases*: Network errors, invalid feed URLs, or empty feeds handled by 'continueRegularOutput' to avoid workflow failure.

---

#### 2.2 Data Preprocessing and Filtering

**Overview:**  
Normalizes and extracts key fields from RSS articles, merges feeds, then filters to keep only articles published within the last day and limits to 20 recent items.

**Nodes Involved:**  
- Edit Fields3  
- Edit Fields  
- Edit Fields1  
- Edit Fields2  
- Merge  
- Filter  
- Limit  

**Node Details:**

- **Edit Fields3, Edit Fields, Edit Fields1, Edit Fields2**  
  - *Type*: Set  
  - *Role*: Standardizes article data fields for each RSS feed.  
  - *Config*: Extracts `title`, `pubDate`, and content fields (`content` or `content:encoded` or `contentSnippet`) into a consistent schema with the field name `conteudo` for content.  
  - *Input*: Raw RSS feed items from respective RSS nodes.  
  - *Output*: Normalized articles to ‘Merge’ node.  
  - *Edge Cases*: Missing or malformed fields may result in empty strings.

- **Merge**  
  - *Type*: Merge  
  - *Role*: Combines the four normalized RSS feed streams into a single collection for unified processing.  
  - *Config*: Set to accept 4 inputs, merging all items.  
  - *Input*: Outputs from all four ‘Edit Fields’ nodes.  
  - *Output*: Combined article list to ‘Filter’ node.  
  - *Edge Cases*: Uneven data arrival times could delay merging.

- **Filter**  
  - *Type*: Filter  
  - *Role*: Filters articles to include only those published within the last 1 day (24 hours).  
  - *Config*: Uses an expression comparing `pubDate` of each article to current date minus 1 day. Case sensitive, strict type validation.  
  - *Input*: Merged articles from ‘Merge’.  
  - *Output*: Filtered articles to ‘Limit’.  
  - *Edge Cases*: Articles without valid `pubDate` will be excluded.

- **Limit**  
  - *Type*: Limit  
  - *Role*: Restricts the dataset to the 20 most recent articles after filtering.  
  - *Config*: Maximum items set to 20.  
  - *Input*: Filtered articles.  
  - *Output*: Limited article list to ‘Aggregate’.  
  - *Sticky Note*:  
    - “## About Limit  
      limits the analysis to the 20 most recent articles”  
  - *Edge Cases*: If fewer than 20 articles available, all are passed.

---

#### 2.3 Aggregation and AI Summarization

**Overview:**  
Aggregates selected article fields, sends them to OpenAI for a detailed summary in Vietnamese, and also leverages Perplexity AI for a financial news summary.

**Nodes Involved:**  
- Aggregate  
- Message an assistant (OpenAI)  
- Message a model in Perplexity1  
- Simple Memory1  

**Node Details:**

- **Aggregate**  
  - *Type*: Aggregate  
  - *Role*: Combines all article items into a single data structure containing specified fields: `title`, `pubDate`, and `conteudo`.  
  - *Config*: Aggregates all incoming items, includes only the stated fields.  
  - *Input*: Limited articles from ‘Limit’.  
  - *Output*: Aggregated array of articles to ‘Message an assistant’.  
  - *Edge Cases*: Empty input results in empty aggregation, possibly causing downstream issues.

- **Message an assistant (OpenAI)**  
  - *Type*: OpenAI node (Langchain integration)  
  - *Role*: Sends aggregated news data to OpenAI for summarization into a 400-word summary focused on the top 15 to 19 highlights, translated into Vietnamese and cleaned of asterisks.  
  - *Config*:  
    - Text parameter dynamically builds a numbered list of article titles and contents from aggregated data (indices 0 to 12).  
    - Prompt set to “define” (likely a predefined prompt template in Langchain).  
    - Uses OpenAI API credentials.  
  - *Input*: Aggregated article data from ‘Aggregate’.  
  - *Output*: Summarized text output to message sending nodes.  
  - *Edge Cases*:  
    - If fewer than 13 articles, expressions referencing missing indices may fail or produce empty entries.  
    - API rate limits, authentication failures, or network errors may cause execution errors.

- **Message a model in Perplexity1**  
  - *Type*: Perplexity AI Tool node  
  - *Role*: Sends a static prompt to Perplexity AI asking for a 400-word summary of economic news from a financial journalist perspective.  
  - *Config*:  
    - Model set to “sonar-reasoning”.  
    - Message content in Vietnamese instructs to generate a comprehensive economic news summary.  
    - Uses Perplexity API credentials.  
  - *Input*: Triggered downstream from ‘Simple Memory1’.  
  - *Output*: Output sent to ‘Message an assistant’ as AI tool response.  
  - *Edge Cases*: API errors, credential issues, or response timeouts.

- **Simple Memory1**  
  - *Type*: Langchain Memory Buffer Window  
  - *Role*: Maintains conversational context or memory using a custom session key “data”.  
  - *Config*: Session ID type is custom key.  
  - *Input*: Receives output from ‘Message a model in Perplexity1’ via AI tool channel.  
  - *Output*: Forwards to ‘Message an assistant’ via AI memory channel.  
  - *Edge Cases*: Memory overflow or session key conflicts could cause unexpected behavior.

---

#### 2.4 Message Delivery

**Overview:**  
Delivers the AI-generated summaries to end users via Zalo (personal and group messages) and Telegram.

**Nodes Involved:**  
- Gửi tin nhắn cá nhân & nhóm1 (Send personal & group messages on Zalo)  
- Send a text message (Telegram)

**Node Details:**

- **Gửi tin nhắn cá nhân & nhóm1**  
  - *Type*: Custom Zalo User Node  
  - *Role*: Sends the summarized news text to a specified Zalo friend and group.  
  - *Config*:  
    - Uses OAuth2 credentials for Zalo User account.  
    - Sends message content from OpenAI output (`$json.output`).  
    - Target friend ID: 5689866405750973941  
    - Group type set to 1 (specific group or chat type).  
  - *Input*: Receives summary output from ‘Message an assistant’.  
  - *Output*: None (end node).  
  - *Edge Cases*: OAuth token expiration, invalid friend ID, or message quota limits.

- **Send a text message (Telegram)**  
  - *Type*: Telegram node  
  - *Role*: Sends the OpenAI summary text to Telegram users.  
  - *Config*:  
    - Uses Telegram API credentials.  
    - Sends message text from OpenAI output (`$json.output`).  
  - *Input*: Receives summary output from ‘Message an assistant’.  
  - *Output*: None (end node).  
  - *Edge Cases*: Telegram bot token revocation, chat permissions, or network errors.

---

### 3. Summary Table

| Node Name                         | Node Type                     | Functional Role                            | Input Node(s)                                  | Output Node(s)                            | Sticky Note                                                     |
|----------------------------------|-------------------------------|--------------------------------------------|------------------------------------------------|------------------------------------------|----------------------------------------------------------------|
| Schedule Trigger                 | Schedule Trigger              | Periodic workflow trigger                  | None                                           | My RSS 01, My RSS 02, My RSS 03, My RSS 04 |                                                                |
| Telegram Trigger                | Telegram Trigger             | Trigger on Telegram message                 | None                                           | My RSS 01, My RSS 02, My RSS 03, My RSS 04 |                                                                |
| Zalo Trigger                   | Custom Zalo User Trigger     | Trigger on Zalo event                       | None                                           | My RSS 01, My RSS 02, My RSS 03, My RSS 04 |                                                                |
| My RSS 01                     | RSS Feed Read                | Fetch RSS feed 1                           | Schedule Trigger, Telegram Trigger, Zalo Trigger | Edit Fields3                             |                                                                |
| My RSS 02                     | RSS Feed Read                | Fetch RSS feed 2                           | Schedule Trigger, Telegram Trigger, Zalo Trigger | Edit Fields                              |                                                                |
| My RSS 03                     | RSS Feed Read                | Fetch RSS feed 3                           | Schedule Trigger, Telegram Trigger, Zalo Trigger | Edit Fields1                             |                                                                |
| My RSS 04                     | RSS Feed Read                | Fetch RSS feed 4                           | Schedule Trigger, Telegram Trigger, Zalo Trigger | Edit Fields2                             |                                                                |
| Edit Fields3                  | Set                          | Normalize RSS 1 article fields             | My RSS 01                                       | Merge                                    |                                                                |
| Edit Fields                   | Set                          | Normalize RSS 2 article fields             | My RSS 02                                       | Merge                                    |                                                                |
| Edit Fields1                  | Set                          | Normalize RSS 3 article fields             | My RSS 03                                       | Merge                                    |                                                                |
| Edit Fields2                  | Set                          | Normalize RSS 4 article fields             | My RSS 04                                       | Merge                                    |                                                                |
| Merge                        | Merge                        | Combine all normalized articles             | Edit Fields3, Edit Fields, Edit Fields1, Edit Fields2 | Filter                                   |                                                                |
| Filter                       | Filter                       | Keep only articles published in last day   | Merge                                           | Limit                                    |                                                                |
| Limit                        | Limit                        | Limit to 20 most recent articles            | Filter                                          | Aggregate                                | ## About Limit limits the analysis to the 20 most recent articles |
| Aggregate                    | Aggregate                    | Aggregate selected article fields           | Limit                                           | Message an assistant                     |                                                                |
| Message an assistant          | OpenAI (Langchain)           | Summarize articles via OpenAI               | Aggregate, Simple Memory1                        | Gửi tin nhắn cá nhân & nhóm1, Send a text message |                                                                |
| Message a model in Perplexity1 | Perplexity AI Tool           | Generate financial news summary              | Simple Memory1                                  | Simple Memory1, Message an assistant     |                                                                |
| Simple Memory1               | Langchain Memory Buffer      | Maintain session memory                      | Message a model in Perplexity1                   | Message an assistant                     |                                                                |
| Gửi tin nhắn cá nhân & nhóm1 | Custom Zalo User             | Send summary message via Zalo                | Message an assistant                            | None                                     |                                                                |
| Send a text message           | Telegram                     | Send summary message via Telegram            | Message an assistant                            | None                                     |                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**
   - Add a **Schedule Trigger** node with default interval to trigger periodically.
   - Add a **Telegram Trigger** node configured to listen for incoming messages (updates: message). Connect Telegram API credentials.
   - Add a **Custom Zalo User Trigger** node configured to listen for event type 1. Connect Zalo OAuth2 credentials.

2. **Add RSS Feed Read Nodes:**
   - Create four **RSS Feed Read** nodes, each configured with one of the following URLs:
     - RSS 01: `https://rss.app/feeds/txgQZnXzgMDnuhVX.xml` (SSL validation enabled)
     - RSS 02: `https://rss.app/feeds/tcEnk7rveNrAqN6Y.xml`
     - RSS 03: `https://news.google.com/rss/search?q=kinh+tế&hl=vi&gl=VN&ceid=VN:vi`
     - RSS 04: `https://rss.app/feeds/tqLgD0PBHuJhlfLA.xml` (SSL validation disabled)
   - Enable "continue on error" to maintain workflow execution despite individual feed failures.
   - Connect outputs of all three triggers (Schedule, Telegram, Zalo) to each RSS feed node to allow triggering from any source.

3. **Add Set Nodes to Normalize Data:**
   - Create four **Set** nodes named Edit Fields3, Edit Fields, Edit Fields1, Edit Fields2.
   - Configure each to retain and rename fields:
     - `title` (string) from `$json.title`
     - `pubDate` (string) from `$json.pubDate`
     - `conteudo` (string) mapped from the feed-specific content field:
       - Edit Fields3: `$json.content`
       - Edit Fields: `$json['content:encoded']`
       - Edit Fields1: `$json.contentSnippet`
       - Edit Fields2: `$json.contentSnippet`
   - Connect each RSS feed node to its corresponding Set node.

4. **Merge and Filter:**
   - Add a **Merge** node configured with 4 inputs to combine all normalized articles.
   - Connect all four Set nodes to the Merge node.
   - Add a **Filter** node with condition:
     - `pubDate` is after `{{$today.minus({ days: 1 })}}`
     - Case sensitive, strict validation.
   - Connect Merge output to Filter node.

5. **Limit and Aggregate:**
   - Add a **Limit** node restricting to max 20 items.
   - Connect Filter output to Limit node.
   - Add an **Aggregate** node configured to aggregate all item data including only fields: `title`, `pubDate`, `conteudo`.
   - Connect Limit output to Aggregate node.

6. **OpenAI Summarization:**
   - Add an **OpenAI** node (Langchain integration).
   - Configure prompt as “define” (preset).
   - Build dynamic text input as a numbered list with entries for indices 0 to 12 of aggregated data’s title and conteudo fields, concatenated and translated into Vietnamese, removing asterisks.
   - Connect Aggregate output to OpenAI node.
   - Set OpenAI API credentials.

7. **Perplexity AI Summarization:**
   - Add a **Perplexity AI Tool** node.
   - Set model to “sonar-reasoning”.
   - Message content: `"Bạn là một nhà báo tài chính, kinh tế, chính trị chuyên sâu, hãy viết bản tin kinh tế tổng hợp 400 chữ."`
   - Connect Perplexity output to a **Langchain Memory Buffer Window** node.
   - Configure memory node with session key “data” and custom session ID type.
   - Connect Memory output to the OpenAI node as AI memory input.

8. **Message Delivery:**
   - Add a **Custom Zalo User** node for sending messages.
   - Configure to send message content from OpenAI output to friend ID `5689866405750973941` and group type 1.
   - Connect OpenAI node output to this Zalo node.
   - Add a **Telegram** node for sending messages.
   - Configure to send message text from OpenAI output.
   - Connect OpenAI node output to Telegram node.

9. **Final Connections:**
   - Ensure triggers connect to all RSS nodes.
   - RSS nodes connect to their respective Set nodes.
   - Set nodes connect to Merge.
   - Merge to Filter.
   - Filter to Limit.
   - Limit to Aggregate.
   - Aggregate to OpenAI.
   - Perplexity node chain feeds into OpenAI via memory.
   - OpenAI output to Zalo and Telegram message nodes.

10. **Credentials Setup:**
    - Configure OpenAI API credentials with proper API key.
    - Configure Telegram API credentials with bot token and webhook URL.
    - Configure Zalo OAuth2 credentials with valid tokens and scopes.
    - Configure Perplexity API credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The Limit node restricts processing to the 20 most recent articles to optimize AI summarization workload. | Sticky note attached near Limit node.                                                             |
| Workflow integrates multiple messaging platforms (Zalo and Telegram) for broader user reach.               | Useful for multi-channel news delivery setups.                                                    |
| OpenAI prompt uses dynamic data referencing aggregated article fields for tailored summarization.          | Ensure enough articles exist to avoid index errors in prompt expressions.                          |
| Perplexity AI node uses “sonar-reasoning” model specialized for reasoning and summarization tasks.          | Credential required from Perplexity AI account.                                                   |
| Zalo custom nodes require OAuth2 user account with messaging permissions and webhooks correctly configured. | See Zalo developer documentation for setup details.                                              |
| Telegram trigger requires webhook setup and bot token with necessary permissions.                          | Refer to Telegram Bot API documentation for webhook configuration.                                |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created using n8n, an integration and automation tool. This process strictly complies with existing content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.