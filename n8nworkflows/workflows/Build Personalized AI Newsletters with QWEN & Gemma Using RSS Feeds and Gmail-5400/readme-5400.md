Build Personalized AI Newsletters with QWEN & Gemma Using RSS Feeds and Gmail

https://n8nworkflows.xyz/workflows/build-personalized-ai-newsletters-with-qwen---gemma-using-rss-feeds-and-gmail-5400


# Build Personalized AI Newsletters with QWEN & Gemma Using RSS Feeds and Gmail

### 1. Workflow Overview

This workflow automates the creation and delivery of a personalized AI-focused newsletter by leveraging RSS feeds, AI models, and Gmail. It targets users interested in receiving curated AI news summaries, rated and tagged by relevance to their specified interests. The workflow includes the following logical blocks:

- **1.1 Initialization & Data Ingestion**: Triggering the workflow, ensuring database and schema setup, and reading news items from an RSS feed.
- **1.2 Data Preparation & Enrichment**: Cleaning and reducing raw RSS data, counting word counts, and merging with existing database records.
- **1.3 AI-Powered Article Rating & Summarization**: Using two AI models (QWEN3 and Gemma3) to rate articles for relevance and generate summaries.
- **1.4 Filtering & Data Merge**: Filtering articles based on AI ratings (threshold ≥ 7), merging summaries with metadata, and preparing data for storage.
- **1.5 Database Storage**: Inserting rated articles into a Postgres database, separating relevant (≥7) and less relevant (<7) articles.
- **1.6 Newsletter Formatting & Delivery**: Formatting the curated articles into an HTML email and sending it via Gmail.
- **1.7 Configuration & User Input Areas**: Nodes and sticky notes for user-defined interests, credentials, and customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Data Ingestion

**Overview:**  
This block triggers the workflow manually or via schedule (disabled), ensures the Postgres schema and tables exist, and fetches news articles from an RSS feed.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Schedule Trigger (disabled)  
- Create DB and Schema if not exists  
- RSS Read  
- Postgres (fetch existing data)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution  
  - Configuration: Default manual trigger, no parameters  
  - Connections: Outputs to "Create DB and Schema if not exists"  
  - Edge Cases: None specific; manual trigger may cause no data if executed at wrong time

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Optional automated periodic trigger (disabled)  
  - Configuration: Every 3 days at 9 AM  
  - Connections: None (disabled)  
  - Edge Cases: Disabled, so no automatic trigger

- **Create DB and Schema if not exists**  
  - Type: Postgres (executeQuery)  
  - Role: Ensures database schema and table exist before ingestion  
  - Configuration: SQL to create schema `ai_references` and table `n8n_newsletter` with fields including title, word_count, score, relevant flag, timestamps  
  - Connections: Output to "RSS Read" and "Postgres" (fetch existing articles)  
  - Edge Cases: SQL execution errors (permissions, connectivity)

- **RSS Read**  
  - Type: RSS Feed Read  
  - Role: Fetches latest articles from specified RSS feed  
  - Configuration: URL = `https://www.all-ai.de/index.php?option=com_jmap&view=sitemap&format=rss`  
  - Connections: Output to "Count words"  
  - Edge Cases: RSS feed unavailable, malformed XML, network timeouts

- **Postgres (fetch existing data)**  
  - Type: Postgres (select)  
  - Role: Retrieves up to 500 existing newsletter records for comparison  
  - Configuration: Selects from table `ai_references.n8n_newsletter`  
  - Connections: Output to "Compare Datasets"  
  - Edge Cases: DB connectivity issues, query errors

---

#### 1.2 Data Preparation & Enrichment

**Overview:**  
This block processes RSS data to reduce fields, count word counts in articles, and compares new data with existing records using a dataset comparison node.

**Nodes Involved:**  
- Count words  
- Compare Datasets  
- Reduce informations  
- Set your Interests  
- Merge Data LLM+RSS  
- Merge  

**Node Details:**  

- **Count words**  
  - Type: Code  
  - Role: Counts number of words in article content and adds a `word_count` field  
  - Configuration: Removes HTML tags, splits text by whitespace, counts words  
  - Connections: Output to "Compare Datasets"  
  - Edge Cases: Empty or missing content, malformed HTML tags

- **Compare Datasets**  
  - Type: Compare Datasets  
  - Role: Compares new RSS articles with existing database records by `title` field  
  - Configuration: Matches on `title` field to identify new or updated articles  
  - Connections: Output to "Reduce informations"  
  - Edge Cases: Title mismatches, duplicate titles

- **Reduce informations**  
  - Type: Set  
  - Role: Selects and assigns key fields from RSS items: title, link, isoDate, content, word_count  
  - Configuration: Retains only essential fields for later processing  
  - Connections: Outputs to "Set your Interests", "Merge Data LLM+RSS", and "Merge"  
  - Edge Cases: Missing fields in RSS entries

- **Set your Interests**  
  - Type: Set  
  - Role: Defines user’s interests as a string for AI rating input  
  - Configuration: User-defined interests list as multiline string (e.g., AI Automation, MCP, etc.)  
  - Connections: Output to "Rating & Tagging Articles"  
  - Edge Cases: Improper formatting of interests may affect AI rating relevance

- **Merge Data LLM+RSS**  
  - Type: Merge  
  - Role: Combines AI-generated rating/tag data with RSS metadata by matching article title  
  - Configuration: Mode "combine" with field matching on `title`  
  - Connections: Outputs to "Summarize Article" and "Merge Data Summary + rest"  
  - Edge Cases: Missing matching titles can lead to partial merges

- **Merge**  
  - Type: Merge  
  - Role: Combines datasets, mainly for articles filtered by rating <7  
  - Configuration: Mode "combine", matching on `title`  
  - Connections: Outputs to "Insert records <7"  
  - Edge Cases: Same as above

---

#### 1.3 AI-Powered Article Rating & Summarization

**Overview:**  
This block uses two Ollama AI models (QWEN3 14B-q4 and Gemma3 4B) to rate articles by relevance and generate concise summaries.

**Nodes Involved:**  
- Model QWEN3 14B-q4  
- Rating & Tagging Articles  
- Parse Output to JSON  
- Rating 7+  
- Model Gemma3 4B  
- Summarize Article  
- Parse Output JSON  
- Merge Data Summary + rest

**Node Details:**  

- **Model QWEN3 14B-q4**  
  - Type: Langchain LM Chat Ollama  
  - Role: Provides article rating, tags, and matching interests using a large QWEN model  
  - Configuration: Model parameter "qwen3:14b-q4_K_M" with default options  
  - Credentials: Ollama API credentials required  
  - Connections: Output to "Rating & Tagging Articles"  
  - Edge Cases: API errors, model unavailability, rate limits

- **Rating & Tagging Articles**  
  - Type: Langchain Chain LLM  
  - Role: Generates JSON output with article rating (1-10), matching interests, and tags  
  - Configuration: Prompt includes article title, content, and user interests; outputs JSON with fields `title`, `rating`, `matching_interests`, `tags`  
  - Connections: Outputs to "Parse Output to JSON" (success) and "No Operation, do nothing1" (error)  
  - Edge Cases: Parsing errors from AI output, retries enabled (max 2 tries), possible malformed JSON

- **Parse Output to JSON**  
  - Type: Code  
  - Role: Extracts JSON object from AI text output and parses it safely  
  - Configuration: Uses regex to find JSON in text, throws errors if no JSON found or parse fails  
  - Connections: Output to "Rating 7+"  
  - Edge Cases: JSON not found or invalid, causes workflow error handling

- **Rating 7+**  
  - Type: If  
  - Role: Filters articles by rating ≥ 7  
  - Configuration: Condition `$json.rating >= 7`  
  - Connections: True branch to "Merge Data LLM+RSS", False branch to "Merge"  
  - Edge Cases: Missing rating field, invalid data types

- **Model Gemma3 4B**  
  - Type: Langchain LM Chat Ollama  
  - Role: Summarizes article content into a short summary (max 170 words)  
  - Configuration: Model "gemma3:latest", prompt instructs to summarize without translation, in source language  
  - Credentials: Ollama API credentials  
  - Connections: Output to "Summarize Article"  
  - Edge Cases: Model errors, API limits

- **Summarize Article**  
  - Type: Langchain Chain LLM  
  - Role: Generates JSON with `title` and `summary` for each article  
  - Configuration: Prompt enforces summary length and language rules  
  - Connections: Output to "Parse Output JSON"  
  - Edge Cases: Parsing errors, handled with continue on error and retry enabled

- **Parse Output JSON**  
  - Type: Code  
  - Role: Extracts and cleans JSON from summarization output, returns only title and summary fields  
  - Configuration: Robust cleaning to fix escape sequences and line breaks  
  - Connections: Output to "Merge Data Summary + rest" (success) and "No Operation, do nothing2" (error)  
  - Edge Cases: Parsing failures, malformed JSON

- **Merge Data Summary + rest**  
  - Type: Merge  
  - Role: Combines summaries with previously merged AI rating and RSS data by matching `title`  
  - Configuration: Mode "combine" matching on `title`  
  - Connections: Outputs to "Format HTML EMail" and "Insert records >=7"  
  - Edge Cases: Missing matches, data inconsistencies

---

#### 1.4 Filtering & Data Merge

**Overview:**  
This block filters articles by rating threshold, merges datasets accordingly, and routes articles to appropriate storage nodes.

**Nodes Involved:**  
- Rating 7+ (If node, see above)  
- Merge Data LLM+RSS  
- Merge  
- Insert records <7  
- Insert records >=7  

**Node Details:**  

- **Insert records <7**  
  - Type: Postgres (insert)  
  - Role: Inserts articles with rating below 7 into `n8n_newsletter` table as not relevant  
  - Configuration: Fields mapped include score, title, publication date (`pup_date`), relevant = false, word_count  
  - Connections: None (end node)  
  - Edge Cases: DB write errors, duplicate entries

- **Insert records >=7**  
  - Type: Postgres (insert)  
  - Role: Inserts highly rated articles (≥7) marked as relevant into database  
  - Configuration: Same fields as above with relevant = true  
  - Connections: None (end node)  
  - Edge Cases: Same as above

---

#### 1.5 Newsletter Formatting & Delivery

**Overview:**  
This block formats the filtered and summarized articles into an HTML newsletter and sends it via Gmail.

**Nodes Involved:**  
- Format HTML EMail  
- Send Email Newsletter  

**Node Details:**  

- **Format HTML EMail**  
  - Type: Code  
  - Role: Constructs an HTML email body with a headline and article sections, including title, tags, date, summary, and link  
  - Configuration: Loops over all items, builds HTML string with headings, paragraphs, and horizontal rules  
  - Connections: Outputs to "Send Email Newsletter"  
  - Edge Cases: Empty item list results in minimal email, HTML encoding issues

- **Send Email Newsletter**  
  - Type: Gmail  
  - Role: Sends the constructed newsletter email to a recipient  
  - Configuration: Subject "AI News", message body includes greeting and embedded newsletter HTML, uses Gmail OAuth2 credentials  
  - Credentials: Gmail OAuth2 (configured)  
  - Edge Cases: Authentication failures, email sending limits, invalid recipient address

---

#### 1.6 Configuration & User Input Areas

**Overview:**  
These nodes and sticky notes provide customization areas for users to configure interests, credentials, and workflow behavior.

**Nodes Involved:**  
- Set your Interests  
- Sticky Notes (multiple)

**Sticky Notes Content Summary:**  

- Sticky Note (near data reduction): Explains comparing datasets and mentions word count code is for dashboards.  
- Sticky Note1 (near AI rating nodes): Describes rating and summarizing articles, instructs to download Ollama models or use local ones.  
- Sticky Note2 (near DB insert nodes): Notes saving both low and high rated articles in the database.  
- Sticky Note3 (near send email node): Reminds user to update email address and Google OAuth2 credentials.  
- Sticky Note4 (near interests node): Instructs user to change their personal interests here.  
- Sticky Note5 (near DB and RSS feed nodes): Reminds user to configure Postgres credentials and RSS feed URL.  
- Sticky Note6 (near triggers): Notes options to trigger manually or schedule (currently disabled).  
- Sticky Note7 (near formatting node): Points to customization of newsletter style and headline.

---

### 3. Summary Table

| Node Name                      | Node Type                       | Functional Role                          | Input Node(s)                                | Output Node(s)                              | Sticky Note                                                                                     |
|--------------------------------|--------------------------------|----------------------------------------|----------------------------------------------|--------------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger                 | Workflow entry point                    | None                                         | Create DB and Schema if not exists          | Change here: Trigger manually or schedule (deactivated)                                       |
| Schedule Trigger               | Schedule Trigger (disabled)    | Optional periodic trigger               | None                                         | None                                       | Change here: Trigger manually or schedule (deactivated)                                       |
| Create DB and Schema if not exists | Postgres (executeQuery)        | Ensures DB schema/table exist           | When clicking ‘Execute workflow’              | RSS Read, Postgres                          | Change here: Postgres Creds, RSS Feed                                                        |
| RSS Read                      | RSS Feed Read                  | Reads latest news from RSS feed         | Create DB and Schema if not exists            | Count words                                | Change here: Postgres Creds, RSS Feed                                                        |
| Count words                   | Code                          | Counts words in article content         | RSS Read                                      | Compare Datasets                           | Compare Datasets and reduce data - Word_count Code node is just for future Dashboards          |
| Compare Datasets              | Compare Datasets               | Compares new RSS data with DB records   | Count words, Postgres                         | Reduce informations                        | Compare Datasets and reduce data - Word_count Code node is just for future Dashboards          |
| Reduce informations           | Set                           | Selects key article fields               | Compare Datasets                              | Set your Interests, Merge Data LLM+RSS, Merge | Compare Datasets and reduce data - Word_count Code node is just for future Dashboards          |
| Set your Interests            | Set                           | Defines user interests for AI rating    | Reduce informations                           | Rating & Tagging Articles                  | Change here: Your Interests                                                                   |
| Model QWEN3 14B-q4            | Langchain LM Chat Ollama      | Rates articles for relevance and tags   | Rating & Tagging Articles (implicit)          | Rating & Tagging Articles                   | Rate Articles, summarize. Download Ollama models or choose local ones                          |
| Rating & Tagging Articles     | Langchain Chain LLM           | Generates rating, tags and matching interests | Reduce informations, Set your Interests       | Parse Output to JSON, No Operation, do nothing1 | Rate Articles, summarize. Download Ollama models or choose local ones                          |
| Parse Output to JSON          | Code                          | Parses AI rating JSON output             | Rating & Tagging Articles                      | Rating 7+                                  | Rate Articles, summarize. Download Ollama models or choose local ones                          |
| Rating 7+                    | If                            | Filters articles rating >=7               | Parse Output to JSON                          | Merge Data LLM+RSS (true), Merge (false)  | Rate Articles, summarize. Download Ollama models or choose local ones                          |
| Merge Data LLM+RSS           | Merge                         | Combines AI rating and RSS metadata      | Rating 7+, Reduce informations                | Model Gemma3 4B, Merge Data Summary + rest | Compare Datasets and reduce data - Word_count Code node is just for future Dashboards          |
| Model Gemma3 4B              | Langchain LM Chat Ollama      | Summarizes article content               | Merge Data LLM+RSS                            | Summarize Article                          | Rate Articles, summarize. Download Ollama models or choose local ones                          |
| Summarize Article            | Langchain Chain LLM           | Creates article summaries                 | Model Gemma3 4B                              | Parse Output JSON                          | Rate Articles, summarize. Download Ollama models or choose local ones                          |
| Parse Output JSON            | Code                          | Parses AI summary JSON output             | Summarize Article                            | Merge Data Summary + rest, No Operation, do nothing2 | Rate Articles, summarize. Download Ollama models or choose local ones                          |
| Merge Data Summary + rest    | Merge                         | Combines summaries with article data      | Parse Output JSON, Merge Data LLM+RSS        | Format HTML EMail, Insert records >=7     | Save data - save the low rated and high rated articles into the table                         |
| Merge                       | Merge                         | Combines articles with rating <7          | Rating 7+ (false), Reduce informations        | Insert records <7                         | Save data - save the low rated and high rated articles into the table                         |
| Insert records <7            | Postgres (insert)             | Inserts low-rated articles into DB        | Merge                                        | None                                      | Save data - save the low rated and high rated articles into the table                         |
| Insert records >=7           | Postgres (insert)             | Inserts highly rated articles into DB     | Merge Data Summary + rest                     | None                                      | Save data - save the low rated and high rated articles into the table                         |
| Format HTML EMail            | Code                          | Formats newsletter HTML content           | Merge Data Summary + rest                     | Send Email Newsletter                     | Change here - Style and headline of your Newsletter                                          |
| Send Email Newsletter        | Gmail                         | Sends formatted newsletter via email     | Format HTML EMail                             | None                                      | Change here - Your Email and Google OAuth2 credentials                                       |
| No Operation, do nothing1    | NoOp                          | Gracefully handles errors during rating   | Rating & Tagging Articles (error path)        | None                                      |                                                                                              |
| No Operation, do nothing2    | NoOp                          | Gracefully handles errors during summarizing | Parse Output JSON (error path)                | None                                      |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters

2. **Create Postgres Node to Initialize DB and Schema**  
   - Type: Postgres (Execute Query)  
   - SQL Query:  
     ```
     CREATE SCHEMA IF NOT EXISTS ai_references;

     CREATE TABLE IF NOT EXISTS ai_references.n8n_newsletter (
         id SERIAL PRIMARY KEY,
         title VARCHAR(255) NOT NULL,
         word_count INTEGER,
         import_date TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
         pup_date TIMESTAMPTZ,
         relevant BOOLEAN NOT NULL DEFAULT false,
         score INTEGER NOT NULL DEFAULT 1 CHECK (score BETWEEN 1 AND 10)
     );
     ```  
   - Credentials: Configure Postgres connection (e.g., Supabase)  
   - Connect Manual Trigger output to this node

3. **Create RSS Feed Read Node**  
   - Type: RSS Feed Read  
   - URL: `https://www.all-ai.de/index.php?option=com_jmap&view=sitemap&format=rss`  
   - Connect previous Postgres node output to this

4. **Create Code Node "Count words"**  
   - Type: Code  
   - JavaScript: Count words in content field, remove HTML tags, add `word_count` field  
   - Run once for each item  
   - Connect RSS Read output to this node

5. **Create Postgres Node to Fetch Existing Articles**  
   - Type: Postgres (Select)  
   - Operation: Select  
   - Schema: `ai_references`  
   - Table: `n8n_newsletter`  
   - Limit: 500  
   - Credentials: same Postgres credentials  
   - Connect Postgres initialization node output to this node

6. **Create Compare Datasets Node**  
   - Type: Compare Datasets  
   - Merge by Fields: `title` (field1 and field2)  
   - Connect "Count words" output to input 1  
   - Connect Postgres select node output to input 2

7. **Create Set Node "Reduce informations"**  
   - Type: Set  
   - Assign fields: `title`, `link`, `isoDate`, `content`, `word_count` (from incoming JSON)  
   - Connect Compare Datasets output to this node

8. **Create Set Node "Set your Interests"**  
   - Type: Set  
   - Assign field: `my_interests` with multiline string describing user interests (e.g., AI Automation, MCP, RAG, European AI Act, No Code AI Tools)  
   - Connect "Reduce informations" output to this node

9. **Create Langchain LM Chat Ollama Node "Model QWEN3 14B-q4"**  
   - Type: Langchain LM Chat Ollama  
   - Model: `qwen3:14b-q4_K_M`  
   - Credentials: Ollama API  
   - Connect "Set your Interests" output to this node

10. **Create Langchain Chain LLM Node "Rating & Tagging Articles"**  
    - Type: Langchain Chain LLM  
    - Prompt includes article title, content, and user interests  
    - Output JSON with fields: title, rating (1-10), matching_interests, tags  
    - Retry on fail: enabled, max tries 2, 3s wait between tries  
    - Connect "Model QWEN3 14B-q4" output to this node

11. **Create Code Node "Parse Output to JSON"**  
    - Type: Code  
    - Extract JSON object from AI output using regex, parse safely  
    - Throws error if no JSON or parse fails  
    - Connect "Rating & Tagging Articles" success output to this node

12. **Create If Node "Rating 7+"**  
    - Type: If  
    - Condition: `$json.rating >= 7` (number comparison)  
    - Connect "Parse Output to JSON" output to this node

13. **Create Merge Node "Merge Data LLM+RSS"**  
    - Type: Merge  
    - Mode: Combine  
    - Merge by field: `title`  
    - Connect "Rating 7+" true output to this node  
    - Also connect "Reduce informations" output as second input

14. **Create Merge Node "Merge"**  
    - Type: Merge  
    - Mode: Combine  
    - Merge by field: `title`  
    - Connect "Rating 7+" false output to this node  
    - Also connect "Reduce informations" output as second input

15. **Create Langchain LM Chat Ollama Node "Model Gemma3 4B"**  
    - Type: Langchain LM Chat Ollama  
    - Model: `gemma3:latest`  
    - Prompt: summarize article content, max 170 words, no translation, keep source language  
    - Credentials: Ollama API  
    - Connect "Merge Data LLM+RSS" output to this node

16. **Create Langchain Chain LLM Node "Summarize Article"**  
    - Type: Langchain Chain LLM  
    - Same summarization prompt as above  
    - Retry on fail enabled  
    - Connect "Model Gemma3 4B" output to this node

17. **Create Code Node "Parse Output JSON"**  
    - Type: Code  
    - Extracts JSON from summarization output, cleans escape sequences, returns only title and summary  
    - Continue on error enabled  
    - Connect "Summarize Article" output to this node

18. **Create Merge Node "Merge Data Summary + rest"**  
    - Type: Merge  
    - Mode: Combine  
    - Merge by field: `title`  
    - Connect "Parse Output JSON" output and "Merge Data LLM+RSS" output as inputs

19. **Create Postgres Insert Node "Insert records >=7"**  
    - Type: Postgres (Insert)  
    - Schema: `ai_references`  
    - Table: `n8n_newsletter`  
    - Map fields: title, word_count, score (rating), pup_date, relevant=true  
    - Credentials: Postgres  
    - Connect "Merge Data Summary + rest" output to this node

20. **Create Postgres Insert Node "Insert records <7"**  
    - Type: Postgres (Insert)  
    - Same configuration as above, but relevant=false  
    - Connect "Merge" output to this node

21. **Create Code Node "Format HTML EMail"**  
    - Type: Code  
    - Builds HTML newsletter from article data: title, tags, date, summary, link  
    - Connect "Merge Data Summary + rest" output to this node

22. **Create Gmail Node "Send Email Newsletter"**  
    - Type: Gmail  
    - Subject: "AI News"  
    - Message body includes greeting and HTML newsletter content  
    - Credentials: Gmail OAuth2 credentials  
    - Connect "Format HTML EMail" output to this node

23. **Create NoOp Nodes for graceful error handling**  
    - "No Operation, do nothing1" connected to "Rating & Tagging Articles" error output  
    - "No Operation, do nothing2" connected to "Parse Output JSON" error output

24. **Add Sticky Notes with instructions and areas for user customization**  
    - For triggers, Postgres credentials, RSS feed URL, user interests, newsletter style, and email credentials  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                        |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| The workflow supports manual or scheduled execution (currently schedule trigger is disabled). | Trigger node and Sticky Note6                                          |
| Ollama AI models QWEN3 14B-q4 and Gemma3 4B are used; users must download or configure them.  | Sticky Note1 and node configurations                                   |
| User must supply Gmail OAuth2 credentials to send emails successfully.                        | Send Email Newsletter node and Sticky Note3                            |
| Postgres credentials and RSS feed URLs must be configured according to user environment.      | Create DB node, Postgres nodes, and Sticky Note5                       |
| User interests defined in a Set node guide the AI rating relevance and tagging process.       | Set your Interests node and Sticky Note4                               |
| Word count node is primarily for dashboarding and analytics, not required for core functionality. | Sticky Note near Count words node                                      |
| Newsletter formatting is customizable in the "Format HTML EMail" code node.                   | Sticky Note7                                                          |
| The workflow gracefully handles parsing errors in AI outputs by continuing execution silently.| No Operation nodes and error handling parameters                       |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly follows content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.