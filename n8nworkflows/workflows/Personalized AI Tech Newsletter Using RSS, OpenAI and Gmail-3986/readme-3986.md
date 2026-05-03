Personalized AI Tech Newsletter Using RSS, OpenAI and Gmail

https://n8nworkflows.xyz/workflows/personalized-ai-tech-newsletter-using-rss--openai-and-gmail-3986


# Personalized AI Tech Newsletter Using RSS, OpenAI and Gmail

### 1. Workflow Overview

This workflow automates the creation and delivery of a personalized weekly technology news newsletter using RSS feeds, OpenAI’s embedding and language models, and Gmail for email delivery. It targets users who want to stay informed about technology topics without daily distractions by consolidating and summarizing news content.

The workflow is logically divided into two main blocks:

- **1.1 Daily News Collection and Storage**
  - Triggered daily, this block fetches tech news articles from multiple RSS feeds, normalizes and processes the content, converts it into vector embeddings with OpenAI, and stores it in an in-memory vector store for efficient semantic querying.

- **1.2 Weekly Newsletter Generation and Delivery**
  - Triggered weekly, this block retrieves relevant articles from the vector store based on user-defined interests, uses an AI agent to generate a concise summary of the most important news, converts the summary into an email-friendly HTML format, and sends it via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily News Collection and Storage

**Overview:**  
This block runs daily to ingest new articles from selected tech news RSS feeds, normalize the data for consistency, create semantic embeddings, and store the articles in an in-memory vector database.

**Nodes Involved:**  
- Get Articles Daily (Schedule Trigger)  
- Set Tech News RSS Feeds (Set)  
- Split Out (SplitOut)  
- Read RSS News Feeds (RSS Feed Read)  
- Normalize Fields (Set)  
- Recursive Character Text Splitter (Text Splitter)  
- Default Data Loader (Document Loader)  
- Embeddings OpenAI (OpenAI Embeddings)  
- Store News Articles (Vector Store In Memory)  

**Node Details:**

- **Get Articles Daily**  
  - Type: Schedule Trigger  
  - Role: Initiates the daily news fetching process at a set interval (daily).  
  - Configuration: Default daily interval with no specific time constraints.  
  - Inputs: None  
  - Outputs: Triggers “Set Tech News RSS Feeds”  
  - Edge Cases: Missed triggers if n8n instance is offline; no articles fetched if RSS feeds are down.

- **Set Tech News RSS Feeds**  
  - Type: Set  
  - Role: Defines the list of RSS feed URLs to fetch articles from.  
  - Configuration: Hardcoded array of six popular tech RSS feeds (Engadget, Ars Technica, The Verge, Wired, Technology Review, TechCrunch).  
  - Inputs: Trigger from “Get Articles Daily”  
  - Outputs: Passes the RSS feeds array to “Split Out”  
  - Edge Cases: Feed URLs must be valid and accessible; feeds returning empty or malformed data can cause downstream issues.

- **Split Out**  
  - Type: SplitOut  
  - Role: Iterates over each RSS feed URL to process feeds individually.  
  - Configuration: Splits the array field named “rss”.  
  - Inputs: List of RSS feed URLs  
  - Outputs: Each feed URL separately to “Read RSS News Feeds”  
  - Edge Cases: Empty feed list results in no processing; malformed field names cause errors.

- **Read RSS News Feeds**  
  - Type: RSS Feed Read  
  - Role: Fetches the articles from the individual RSS feeds.  
  - Configuration: Uses URL from input item. SSL verification enabled.  
  - Inputs: Single RSS feed URL  
  - Outputs: Raw article data array to “Normalize Fields”  
  - Edge Cases: Network errors, feed downtime, or invalid RSS format can cause failures or empty results.

- **Normalize Fields**  
  - Type: Set  
  - Role: Extracts and normalizes key fields from RSS feed items: title, content snippet, and publication date.  
  - Configuration:  
    - title: from `$json.title`  
    - content: from `$json['content:encodedSnippet']` or fallback to `$json.contentSnippet`  
    - date: from `$json.isoDate`  
  - Inputs: Raw feed item JSON array  
  - Outputs: Normalized articles to “Store News Articles”  
  - Edge Cases: Missing fields in RSS items may result in empty or undefined values.

- **Recursive Character Text Splitter**  
  - Type: Text Splitter  
  - Role: (Connected in flow but not active in main chain) This node is prepared to chunk long content into smaller pieces (max 3000 chars) for embedding, but currently not connected in the main workflow branch.  
  - Configuration: Chunk size 3000 characters.  
  - Edge Cases: Large content exceeding chunk size would be split if used.

- **Default Data Loader**  
  - Type: Document Loader  
  - Role: Formats normalized article content and adds metadata for embedding storage.  
  - Configuration:  
    - Metadata includes title, creation date (current time), and publish date.  
    - Data format: Markdown with title as header and content body.  
  - Inputs: Normalized article fields  
  - Outputs: Document format for vector storage  
  - Edge Cases: Metadata missing or improperly formatted could affect retrieval later.

- **Embeddings OpenAI**  
  - Type: OpenAI Embeddings  
  - Role: Generates vector embeddings for each article using OpenAI API.  
  - Configuration: Default embedding model (OpenAI API credentials required).  
  - Inputs: Document data from “Default Data Loader”  
  - Outputs: Embeddings to “Store News Articles”  
  - Edge Cases: API rate limits, credential errors, or network issues.

- **Store News Articles**  
  - Type: Vector Store In Memory  
  - Role: Stores embeddings and documents in an in-memory vector database keyed under “news_store_key” for fast semantic search.  
  - Configuration: Mode set to “insert” (adds new items).  
  - Inputs: Embeddings and document metadata  
  - Outputs: None (end of daily insertion chain)  
  - Edge Cases: Memory limitations if workflow runs long term; no persistence beyond n8n runtime.

---

#### 1.2 Weekly Newsletter Generation and Delivery

**Overview:**  
Runs once a week to query the stored news articles based on user interests, generate a personalized summary using AI, format the summary for email, and send it to the user’s inbox.

**Nodes Involved:**  
- Send Weekly Summary (Schedule Trigger)  
- Your topics of interest (Set)  
- Get News Articles (Vector Store In Memory Retrieval)  
- Embeddings OpenAI2 (OpenAI Embeddings)  
- News reader AI (LangChain Agent)  
- OpenAI Chat Model (Language Model)  
- Convert Response to an Email-Friendly Format (Markdown to HTML)  
- Send Newsletter (Gmail)  

**Node Details:**

- **Send Weekly Summary**  
  - Type: Schedule Trigger  
  - Role: Triggers the newsletter generation every Monday at 5 AM.  
  - Configuration: Weekly interval, triggers on Monday at 5:00.  
  - Inputs: None  
  - Outputs: Triggers “Your topics of interest” node.  
  - Edge Cases: Missed triggers if n8n instance is offline.

- **Your topics of interest**  
  - Type: Set  
  - Role: Defines user preferences for newsletter content.  
  - Configuration:  
    - Interests: “AI, games, gadgets”  
    - Number of news items to include: “15”  
  - Inputs: Trigger from “Send Weekly Summary”  
  - Outputs: Passes preferences to “News reader AI”  
  - Edge Cases: Empty or malformed interests could affect AI relevance.

- **Get News Articles**  
  - Type: Vector Store In Memory (Retrieve Mode)  
  - Role: Queries the vector store for the top 20 relevant articles based on AI embeddings.  
  - Configuration:  
    - Mode: “retrieve-as-tool”  
    - TopK: 20  
    - Tool name: “get_news”  
    - Memory key: “news_store_key”  
    - Tool description provided to AI agent  
  - Inputs: Embeddings from “Embeddings OpenAI2”  
  - Outputs: Retrieved article data to “News reader AI” as a tool input  
  - Edge Cases: Empty vector store results in no content for summary.

- **Embeddings OpenAI2**  
  - Type: OpenAI Embeddings  
  - Role: Embeds the query or input for retrieval.  
  - Configuration: Default OpenAI embedding model.  
  - Inputs: Query data from “Your topics of interest” node  
  - Outputs: Embeddings for use by “Get News Articles”  
  - Edge Cases: API errors or rate limits.

- **News reader AI**  
  - Type: LangChain Agent  
  - Role: Acts as an AI aggregator and summarizer for the fetched news.  
  - Configuration:  
    - Prompt: “Summarize last week's news.”  
    - System message instructs to focus on last week’s news, write in plain English, prioritize specified interests, fallback to most newsworthy tech events, and include specified number of news items.  
    - Uses “get_news” tool (vector store retrieval) and OpenAI Chat model.  
  - Inputs: User preferences, retrieved articles, and OpenAI chat model  
  - Outputs: Text summary to “Convert Response to an Email-Friendly Format”  
  - Edge Cases: AI hallucinations, prompt misinterpretation, or insufficient relevant data.

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Model (gpt-4o)  
  - Role: Provides natural language generation capabilities for the agent.  
  - Configuration: Model “gpt-4o” with caching enabled.  
  - Inputs: Prompt and context from “News reader AI”  
  - Outputs: Generated summary text  
  - Edge Cases: API errors, model quota limits, or slow response.

- **Convert Response to an Email-Friendly Format**  
  - Type: Markdown  
  - Role: Converts the AI-generated markdown summary into HTML for email.  
  - Configuration: Mode set to “markdownToHtml”.  
  - Inputs: AI summary markdown text  
  - Outputs: HTML-formatted email body to “Send Newsletter”  
  - Edge Cases: Malformed markdown could cause formatting issues.

- **Send Newsletter**  
  - Type: Gmail  
  - Role: Sends the formatted newsletter email to a fixed recipient.  
  - Configuration:  
    - Recipient: miha.ambroz@n8n.io (hardcoded)  
    - Subject: “Weekly tech newsletter”  
    - Message body: HTML from previous node  
  - Credentials: Gmail OAuth2 credentials required.  
  - Inputs: HTML email content  
  - Outputs: None (end of flow)  
  - Edge Cases: Authentication errors, quota limits, or email delivery failures.

---

### 3. Summary Table

| Node Name                      | Node Type                            | Functional Role                          | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                                                |
|--------------------------------|------------------------------------|----------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Get Articles Daily              | Schedule Trigger                   | Daily trigger to start fetching feeds  | None                             | Set Tech News RSS Feeds          |                                                                                                                            |
| Set Tech News RSS Feeds         | Set                               | Defines RSS feed URLs                   | Get Articles Daily               | Split Out                       |                                                                                                                            |
| Split Out                      | SplitOut                          | Iterates over each RSS feed             | Set Tech News RSS Feeds          | Read RSS News Feeds             |                                                                                                                            |
| Read RSS News Feeds            | RSS Feed Read                    | Fetches articles from RSS feeds         | Split Out                      | Normalize Fields                |                                                                                                                            |
| Normalize Fields               | Set                               | Extracts and normalizes article fields  | Read RSS News Feeds             | Store News Articles             |                                                                                                                            |
| Recursive Character Text Splitter | Text Splitter                     | Prepares to split long texts (not connected) | None                           | Default Data Loader (not connected) |                                                                                                                            |
| Default Data Loader            | Document Loader                   | Formats articles with metadata          | Recursive Character Text Splitter (not connected) | Store News Articles             |                                                                                                                            |
| Embeddings OpenAI             | OpenAI Embeddings                 | Generates vector embeddings              | Default Data Loader             | Store News Articles             |                                                                                                                            |
| Store News Articles            | Vector Store In Memory            | Stores articles embeddings in-memory    | Normalize Fields, Embeddings OpenAI | None                          | ## 1. Save news in a vector store (runs daily)                                                                             |
| Send Weekly Summary            | Schedule Trigger                 | Weekly trigger to start newsletter flow | None                           | Your topics of interest         |                                                                                                                            |
| Your topics of interest        | Set                               | Sets user interests and item count      | Send Weekly Summary             | News reader AI                 | ### Edit this:                                                                                                             |
| Embeddings OpenAI2            | OpenAI Embeddings                 | Embeds query for news retrieval          | Your topics of interest         | Get News Articles              |                                                                                                                            |
| Get News Articles             | Vector Store In Memory            | Retrieves top relevant articles          | Embeddings OpenAI2              | News reader AI                 |                                                                                                                            |
| News reader AI                | LangChain Agent                  | Summarizes news based on interests       | Your topics of interest, Get News Articles, OpenAI Chat Model | Convert Response to an Email-Friendly Format | ## 2. Send a summary (runs once a week)                                                                                  |
| OpenAI Chat Model             | OpenAI Chat Model                | Provides LLM for summarization            | News reader AI                 | News reader AI                 |                                                                                                                            |
| Convert Response to an Email-Friendly Format | Markdown                         | Converts markdown summary to HTML         | News reader AI                 | Send Newsletter               |                                                                                                                            |
| Send Newsletter              | Gmail                            | Sends the newsletter email                 | Convert Response to an Email-Friendly Format | None                          |                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Daily News Collection and Storage:**

   1. Add a **Schedule Trigger** node named `Get Articles Daily`. Set it to trigger daily (default interval).

   2. Add a **Set** node named `Set Tech News RSS Feeds`. Create a new field named `rss` of type Array with these RSS feed URLs:  
      ```
      https://www.engadget.com/rss.xml  
      https://feeds.arstechnica.com/arstechnica/index  
      https://www.theverge.com/rss/index.xml  
      https://www.wired.com/feed/rss  
      https://www.technologyreview.com/topnews.rss  
      https://techcrunch.com/feed/
      ```
      Connect `Get Articles Daily` → `Set Tech News RSS Feeds`.

   3. Add a **SplitOut** node named `Split Out` configured to split the field `rss`. Connect `Set Tech News RSS Feeds` → `Split Out`.

   4. Add an **RSS Feed Read** node named `Read RSS News Feeds`. Set URL to `={{ $json.rss }}` (expression). Connect `Split Out` → `Read RSS News Feeds`.

   5. Add a **Set** node named `Normalize Fields`. Assign fields:  
      - `title` = `{{$json.title}}`  
      - `content` = `{{$json['content:encodedSnippet'] ?? $json.contentSnippet}}`  
      - `date` = `{{$json.isoDate}}`  
      Connect `Read RSS News Feeds` → `Normalize Fields`.

   6. Add a **Default Data Loader** node named `Default Data Loader`.  
      - Set `jsonData` to an expression:  
        ```
        =# {{ $json.title }}
        {{ $json.content }}
        ```  
      - Set metadata:  
        - `title` = `{{$json.title}}`  
        - `=createDate` = `{{$now.toISO()}}`  
        - `publishDate` = `{{$json.date}}`  
      Connect `Normalize Fields` → `Default Data Loader`.

   7. Add an **Embeddings OpenAI** node named `Embeddings OpenAI`. Use your OpenAI credentials. Connect `Default Data Loader` → `Embeddings OpenAI`.

   8. Add a **Vector Store In Memory** node named `Store News Articles`. Set mode to `insert` and memory key to `news_store_key`. Connect both `Normalize Fields` (main output) and `Embeddings OpenAI` (ai_embedding output) to this node accordingly.

3. **Weekly Newsletter Generation and Delivery:**

   1. Add a **Schedule Trigger** node named `Send Weekly Summary`. Configure it to run weekly on Mondays at 5:00 AM.

   2. Add a **Set** node named `Your topics of interest`. Assign:  
      - `Interests` = `AI, games, gadgets`  
      - `Number of news items to include` = `15`  
      Connect `Send Weekly Summary` → `Your topics of interest`.

   3. Add an **Embeddings OpenAI** node named `Embeddings OpenAI2` with your OpenAI credentials. Connect `Your topics of interest` → `Embeddings OpenAI2`.

   4. Add a **Vector Store In Memory** node named `Get News Articles`.  
      - Set mode to `retrieve-as-tool`  
      - TopK to `20`  
      - Tool name: `get_news`  
      - Memory key: `news_store_key`  
      - Tool description: `Call this tool to get the latest news articles.`  
      Connect `Embeddings OpenAI2` → `Get News Articles`.

   5. Add an **OpenAI Chat Model** node named `OpenAI Chat Model`.  
      - Model: `gpt-4o`  
      - Use your OpenAI credentials.  
      Connect this node as the `ai_languageModel` input into the next node.

   6. Add a **LangChain Agent** node named `News reader AI`.  
      - Text prompt: `=Summarize last week's news.`  
      - System message (prompt options):  
        ```
        Only get last week's news. Act as a tech news aggregator and write in plain, easy-to-understand English. Prioritize news related to the following topics: {{ $json.Interests }}.
        If none of those topics are mentioned in the news, use your best judgment to highlight the most newsworthy, frequently mentioned and relevant events in technology.

        Provide a total of {{ $json['Number of news items to include'] }} news items.
        ```  
      - Connect `Your topics of interest` → `News reader AI` (main input)  
      - Connect `Get News Articles` → `News reader AI` (ai_tool input)  
      - Connect `OpenAI Chat Model` → `News reader AI` (ai_languageModel input)  

   7. Add a **Markdown** node named `Convert Response to an Email-Friendly Format`. Set mode to `markdownToHtml`. Connect `News reader AI` → `Convert Response to an Email-Friendly Format`.

   8. Add a **Gmail** node named `Send Newsletter`.  
      - Set `Send To` to your email (replace the example `miha.ambroz@n8n.io`)  
      - Subject: `Weekly tech newsletter`  
      - Message: `={{ $json.data }}` (HTML from previous node)  
      - Configure OAuth2 credentials for Gmail.  
      Connect `Convert Response to an Email-Friendly Format` → `Send Newsletter`.

4. **Activate the workflow and ensure your OpenAI and Gmail credentials are set up properly.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                     | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow fetches tech news via [RSS feeds](https://en.wikipedia.org/wiki/RSS) from selected websites, stores them in a vector database, and uses an AI agent to send a weekly, personalized newsletter, reducing daily distractions.         | Workflow Description, Sticky Note (position: -680, -260)                                           |
| Runs daily to save news in a vector store and weekly to send a summarized newsletter.                                                                                                                                                           | Sticky Notes 1 and 2 in the workflow                                                              |
| Customization ideas: Replace RSS feeds, use Pinecone or Weaviate for persistent vector storage, adjust AI summarization style, or replace email with Telegram bot for chat delivery.                                                            | Sticky Note 6                                                                                      |

---

This structured reference document should enable advanced users and automation agents to fully comprehend, replicate, and adapt the “Personalized AI Tech Newsletter Using RSS, OpenAI and Gmail” workflow efficiently.