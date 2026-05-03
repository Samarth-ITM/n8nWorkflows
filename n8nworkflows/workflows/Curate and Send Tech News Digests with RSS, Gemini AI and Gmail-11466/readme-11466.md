Curate and Send Tech News Digests with RSS, Gemini AI and Gmail

https://n8nworkflows.xyz/workflows/curate-and-send-tech-news-digests-with-rss--gemini-ai-and-gmail-11466


# Curate and Send Tech News Digests with RSS, Gemini AI and Gmail

### 1. Workflow Overview

This n8n workflow automates the daily curation and delivery of technology news digests, focusing on cybersecurity, artificial intelligence (AI), and related tech industry updates. Its core purpose is to aggregate multiple RSS news feeds, filter and sort recent articles, enrich them with AI-powered summarization, and send a curated newsletter email.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception (RSS Feed Collection):** Multiple RSS Feed Read nodes independently fetch the latest news articles from diverse sources categorized mainly as Cybersecurity, AI, and Nvidia-related tech news.

- **1.2 Data Merging and Aggregation:** Several Merge nodes combine inputs from individual RSS nodes into grouped streams by topic (Cybersecurity, AI, Nvidia) and then consolidates all topics into a single unified dataset.

- **1.3 Filtering and Sorting:** A Filter node excludes articles older than 24 hours to maintain freshness, followed by a Sort node ordering articles by date descending.

- **1.4 Data Transformation:** A Code (JavaScript) node compacts all articles into a single item with an array of normalized article objects, standardizing key fields.

- **1.5 AI Processing (Gemini AI Summarization):** A Google Gemini AI node receives the structured data and applies editorial rules to select, deduplicate, categorize, and summarize the top tech news into a JSON object containing subject and HTML content for the newsletter.

- **1.6 Final Email Preparation:** A Code node parses the AI output, constructs a clean, professional HTML email template, and passes it along.

- **1.7 Email Delivery:** A Gmail node sends the final curated news digest via email using OAuth2 credentials.

This modular, scalable design enables easy addition/removal of RSS sources and ensures a high-quality, timely tech news briefing delivered automatically to an inbox.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception (RSS Feed Collection)

- **Overview:**  
  Fetches news articles from multiple RSS feeds grouped by topic. Each feed is handled separately for stability and clarity.

- **Nodes Involved:**  
  - RSS_TheHackersNews  
  - RSS_GrahamCluley  
  - RSS_cybersecuritynews  
  - RSS_krebsonsecurity  
  - RSS_darkreading  
  - RSS_darkreading1  
  - RSS_Sans  
  - RSS_cve1  
  - RSS_cve2  
  - RSS_ilsole24ore Tech  
  - RSS_ilsole24ore Cyber  
  - RSS_cybersecurity360  
  - RSS_Read  
  - RSS_CiscoTalos  
  - RSS_artificialintelligence-news.  
  - RSS_GoogleResearch  
  - RSS_MIT  
  - RSS_OpenAI  
  - RSS_Nvidia1  
  - RSS_Nvidia2  
  - RSS_Nvidia3  
  - RSS_GoogleCloudBlog

- **Node Details (Representative examples):**  
  - **Type:** RSS Feed Read  
  - **Role:** Periodically fetches the latest items from a specific RSS feed URL.  
  - **Configuration:** Each node is configured with a unique RSS feed URL relevant to tech, AI, cybersecurity, or Nvidia news.  
  - **Inputs:** Triggered by Schedule Trigger node (starting the workflow run).  
  - **Outputs:** Emits array of RSS items with fields like title, link, content, isoDate.  
  - **Potential Failures:** Network issues, feed unavailability, malformed RSS content.  
  - **Sticky Notes:** "📌 RSS Nodes (Multiple Sources)" - explains isolated nodes for stability and debugging.

---

#### 2.2 Data Merging and Aggregation

- **Overview:**  
  Combines outputs from related RSS feeds into grouped streams by topic, then merges all topics into one dataset.

- **Nodes Involved:**  
  - Merge_Cyber1 (8 inputs)  
  - Merge_Cyber2 (2 inputs)  
  - Merge_Cyber3 (5 inputs)  
  - Merge_AI (4 inputs)  
  - Merge_Nvidia (3 inputs)  
  - Merge_All (5 inputs)

- **Node Details:**  
  - **Type:** Merge  
  - **Role:** Consolidates multiple incoming data streams into one unified output.  
  - **Configuration:** Configured with the correct number of inputs; no additional parameters.  
  - **Inputs and Outputs:**  
    - Cybersecurity merges inputs from multiple RSS nodes related to cybersecurity.  
    - AI merges from AI-related RSS feeds.  
    - Nvidia merges Nvidia-specific RSS feeds.  
    - Merge_All collects all topic merges into a single stream.  
  - **Potential Failures:** If input data format diverges, subsequent nodes may fail; no explicit deduplication here.  
  - **Sticky Notes:** "📌 Merge Nodes" - describes purpose of combining feeds.

---

#### 2.3 Filtering and Sorting

- **Overview:**  
  Filters out articles older than 24 hours to keep news fresh, then sorts remaining articles by their publication date descending.

- **Nodes Involved:**  
  - Filter  
  - Sort - Articles by Date

- **Node Details:**  
  - **Filter Node:**  
    - **Type:** Filter  
    - **Role:** Drops articles with isoDate older than current time minus 24 hours.  
    - **Key Expression:** Compares each item's isoDate to `DateTime.now().minus({ hours: 24 }).toISO()` using a dateTime operator.  
    - **Potential Failures:** Invalid date formats could cause filtering errors.  
  - **Sort Node:**  
    - **Type:** Sort  
    - **Role:** Orders articles by 'isoDate' in descending order.  
    - **Potential Failures:** Missing isoDate fields could affect sorting.

- **Sticky Notes:**  
  - Filter node: "📌 Filter" - explains filtering for freshness.  
  - Sort node: "📌Sort – Articles by Date" - ensures newest stories first.

---

#### 2.4 Data Transformation

- **Overview:**  
  Converts multiple article items into a single item with an array of article objects, normalizing key fields.

- **Nodes Involved:**  
  - Code in JavaScript

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Role:** Packs all articles into one JSON object with "articles" array.  
  - **Code Logic:** Extracts title, content or contentSnippet, link, and isoDate from each article.  
  - **Input:** Sorted and filtered article items.  
  - **Output:** Single item JSON with `articles` array containing normalized article info.  
  - **Potential Failures:** Missing fields in input items might cause undefined values.

- **Sticky Notes:**  
  - "📌Sort – Code in JavaScript" - describes packing articles array.

---

#### 2.5 AI Processing (Gemini AI Summarization)

- **Overview:**  
  Uses Google Gemini AI model to select, deduplicate, categorize, and summarize the most important tech news into a structured JSON output for the newsletter.

- **Nodes Involved:**  
  - LLM - News Summarizer

- **Node Details:**  
  - **Type:** Google Gemini AI (Langchain)  
  - **Role:** Applies editorial selection rules to input articles.  
  - **Model:** Gemini 2.5 Flash model chosen for summarization.  
  - **Prompt:** Detailed editorial instructions to choose max 8-10 relevant news, deduplicate, categorize into predefined topics, write concise, factual news blurbs with a mandatory formatted source link.  
  - **Input:** JSON object with array of articles.  
  - **Output:** JSON with `subject` (email subject) and `html` (newsletter content) fields.  
  - **Credentials:** Google Palm API credentials required.  
  - **Potential Failures:** API quota/timeout, malformed input, prompt misinterpretation, output not in strict JSON format.  
  - **Sticky Notes:** "📌 Gemini Node (AI Enrichment)" - describes AI summarization function.

---

#### 2.6 Final Email Preparation

- **Overview:**  
  Parses the AI output JSON, validates it, removes code fences, and builds a styled HTML email template embedding the subject and HTML content.

- **Nodes Involved:**  
  - Build Final Newsletter HTML (Code)

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Role:**  
    - Extracts JSON from AI output (removes triple backticks and any text before/after JSON).  
    - Parses JSON safely, validates presence of `subject` and `html`.  
    - Builds a responsive, visually clean HTML email template incorporating subject as title and body.  
    - Returns single item JSON with `subject` and `html` for email sending.  
  - **Potential Failures:** Malformed AI output, JSON parsing errors, missing fields.  
  - **Sticky Notes:** "📌 Build Final Newsletter HTML (Code)"

---

#### 2.7 Email Delivery

- **Overview:**  
  Sends the curated newsletter email using Gmail with OAuth2 authentication.

- **Nodes Involved:**  
  - Send Final Digest Email

- **Node Details:**  
  - **Type:** Gmail Node  
  - **Role:** Sends email to configured recipient with subject and HTML body from previous node.  
  - **Parameters:**  
    - Recipient: user@example.com (should be replaced with actual target).  
    - Sender name: "n8n News"  
    - Subject: Fixed prefix "News_Tech | n8n RSS" (note: subject from code node is used in email body construction).  
  - **Credentials:** Gmail OAuth2 credentials required.  
  - **Potential Failures:** Authentication errors, rate limits, invalid email addresses.  
  - **Sticky Notes:** "📌 Final Output Node (Email / Notion / API / etc.)"

---

#### 2.8 Workflow Trigger

- **Overview:**  
  Automates workflow execution at defined intervals.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Role:** Starts workflow automatically based on configured time interval (default is daily run).  
  - **Potential Failures:** None typical.  
  - **Sticky Notes:** "📌 Schedule Trigger"

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                             | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                  |
|----------------------------|--------------------------------|---------------------------------------------|------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger               | Starts workflow at defined intervals         |                              | RSS_TheHackersNews, ... (all RSS nodes) | 📌 Schedule Trigger                                                                                           |
| RSS_TheHackersNews         | RSS Feed Read                 | Fetches Hacker News RSS feed                   | Schedule Trigger              | Merge_Cyber1                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_GrahamCluley           | RSS Feed Read                 | Fetches Graham Cluley RSS feed                 | Schedule Trigger              | Merge_Cyber1                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_cybersecuritynews      | RSS Feed Read                 | Fetches Cybersecurity News RSS                  | Schedule Trigger              | Merge_Cyber1                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_krebsonsecurity        | RSS Feed Read                 | Fetches Krebs on Security RSS                   | Schedule Trigger              | Merge_Cyber1                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_darkreading            | RSS Feed Read                 | Fetches Dark Reading RSS                        | Schedule Trigger              | Merge_Cyber1                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_darkreading1           | RSS Feed Read                 | Fetches Checkpoint Research RSS                 | Schedule Trigger              | Merge_Cyber1                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_Sans                  | RSS Feed Read                 | Fetches SANS ISC RSS                            | Schedule Trigger              | Merge_Cyber1                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_cve1                  | RSS Feed Read                 | Fetches NVD CVE RSS                             | Schedule Trigger              | Merge_Cyber2                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_cve2                  | RSS Feed Read                 | Fetches CVE Feed RSS                            | Schedule Trigger              | Merge_Cyber2                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_ilsole24ore Tech       | RSS Feed Read                 | Fetches Il Sole 24 Ore Tech RSS                 | Schedule Trigger              | Merge_Cyber3                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_ilsole24ore Cyber      | RSS Feed Read                 | Fetches Il Sole 24 Ore Cybersecurity RSS       | Schedule Trigger              | Merge_Cyber3                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_cybersecurity360       | RSS Feed Read                 | Fetches Cybersecurity360 RSS                    | Schedule Trigger              | Merge_Cyber3                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_Read                   | RSS Feed Read                 | Fetches ESET Blog RSS                           | Schedule Trigger              | Merge_Cyber3                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_CiscoTalos             | RSS Feed Read                 | Fetches Cisco Talos Blog RSS                    | Schedule Trigger              | Merge_Cyber3                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_artificialintelligence-news. | RSS Feed Read           | Fetches Artificial Intelligence News RSS       | Schedule Trigger              | Merge_AI                    | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_GoogleResearch         | RSS Feed Read                 | Fetches Google Research Blog RSS                | Schedule Trigger              | Merge_AI                    | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_MIT                   | RSS Feed Read                 | Fetches MIT AI News RSS                         | Schedule Trigger              | Merge_AI                    | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_OpenAI                | RSS Feed Read                 | Fetches OpenAI News RSS                         | Schedule Trigger              | Merge_AI                    | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_GoogleCloudBlog        | RSS Feed Read                 | Fetches Google Cloud Threat Intelligence RSS   | Schedule Trigger              | Merge_Cyber1                 | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_Nvidia1               | RSS Feed Read                 | Fetches Nvidia News Releases RSS                | Schedule Trigger              | Merge_Nvidia                | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_Nvidia2               | RSS Feed Read                 | Fetches Nvidia Developer Blog RSS               | Schedule Trigger              | Merge_Nvidia                | 📌 RSS Nodes (Multiple Sources)                                                                               |
| RSS_Nvidia3               | RSS Feed Read                 | Fetches Nvidia Blog Feed RSS                     | Schedule Trigger              | Merge_Nvidia                | 📌 RSS Nodes (Multiple Sources)                                                                               |
| Merge_Cyber1              | Merge                        | Merges multiple cybersecurity RSS feeds        | Multiple cybersecurity RSS nodes | Merge_All                 | 📌 Merge Nodes                                                                                                |
| Merge_Cyber2              | Merge                        | Merges CVE related RSS feeds                     | RSS_cve1, RSS_cve2           | Merge_All                    | 📌 Merge Nodes                                                                                                |
| Merge_Cyber3              | Merge                        | Merges other cybersecurity feeds                 | Multiple RSS nodes           | Merge_All                    | 📌 Merge Nodes                                                                                                |
| Merge_AI                  | Merge                        | Merges AI-related RSS feeds                       | Multiple AI RSS nodes        | Merge_All                    | 📌 Merge Nodes                                                                                                |
| Merge_Nvidia              | Merge                        | Merges Nvidia-specific RSS feeds                  | RSS_Nvidia1,2,3             | Merge_All                    | 📌 Merge Nodes                                                                                                |
| Merge_All                 | Merge                        | Combines all topic merges into one stream        | Merge_Cyber1,2,3, AI, Nvidia | Filter                      | 📌 Merge Nodes                                                                                                |
| Filter                    | Filter                       | Filters articles newer than 24 hours             | Merge_All                   | Sort - Articles by Date      | 📌 Filter                                                                                                    |
| Sort - Articles by Date   | Sort                         | Sorts articles by isoDate descending              | Filter                      | Code in JavaScript           | 📌Sort – Articles by Date                                                                                     |
| Code in JavaScript        | Code (JavaScript)            | Packs articles into a single JSON object with array | Sort - Articles by Date      | LLM - News Summarizer        | 📌Sort – Code in JavaScript                                                                                    |
| LLM - News Summarizer     | Google Gemini AI             | Applies editorial AI summarization & selection   | Code in JavaScript           | Build Final Newsletter HTML  | 📌 Gemini Node (AI Enrichment)                                                                                 |
| Build Final Newsletter HTML| Code (JavaScript)            | Parses AI output, builds final HTML email template| LLM - News Summarizer        | Send Final Digest Email      | 📌 Build Final Newsletter HTML (Code)                                                                         |
| Send Final Digest Email   | Gmail                        | Sends final curated newsletter via email          | Build Final Newsletter HTML  |                              | 📌 Final Output Node (Email / Notion / API / etc.)                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Configuration: Set interval to daily or desired frequency to run the workflow automatically.

2. **Add Individual RSS Feed Read Nodes:**  
   - For each RSS feed (e.g., TheHackersNews, GrahamCluley, cybersecuritynews, etc.), add one RSS Feed Read node.  
   - Configure each node with its unique RSS feed URL.  
   - Connect the Schedule Trigger main output to each RSS node input (parallel branches).

3. **Add Merge Nodes for Topic Grouping:**  
   - Create Merge nodes to group feeds by topic:  
     - Merge_Cyber1 (8 inputs) for main cybersecurity feeds  
     - Merge_Cyber2 (2 inputs) for CVE feeds  
     - Merge_Cyber3 (5 inputs) for other cybersecurity feeds  
     - Merge_AI (4 inputs) for AI-related feeds  
     - Merge_Nvidia (3 inputs) for Nvidia-related feeds  
   - Connect appropriate RSS feed outputs to these Merge nodes.

4. **Add Merge_All Node:**  
   - Create a Merge node with 5 inputs.  
   - Connect outputs of all topic Merge nodes (Cyber1, Cyber2, Cyber3, AI, Nvidia) to Merge_All inputs.

5. **Add Filter Node:**  
   - Add a Filter node configured to retain only items where `isoDate` is within the last 24 hours.  
   - Use a dateTime condition comparing article isoDate to `now minus 24 hours`.

6. **Add Sort Node:**  
   - Add a Sort node after Filter.  
   - Configure sort field as `isoDate` descending.

7. **Add Code (JavaScript) Node for Transformation:**  
   - Create a Code node to consolidate all articles into a single item with an `articles` array.  
   - Use code to map title, content/contentSnippet, link, and isoDate from each incoming article.

8. **Add Google Gemini AI Node:**  
   - Add a Google Gemini AI (Langchain) node.  
   - Configure with model "models/gemini-2.5-flash".  
   - Set prompt with detailed editorial rules for selecting and summarizing top 8-10 tech news.  
   - Provide Google Palm API credentials.  
   - Connect from Code node output to this AI node input.

9. **Add Code Node to Build Final Newsletter HTML:**  
   - Add a Code node to parse AI output JSON, clean formatting, validate, and build a responsive HTML email template embedding subject and body.  
   - Connect output of Gemini AI node here.

10. **Add Gmail Node for Email Sending:**  
    - Add Gmail node configured with OAuth2 credentials.  
    - Set recipient email (e.g., user@example.com).  
    - Set subject and HTML body from previous node output.  
    - Connect from the final Code node output.

11. **Add Sticky Notes for Documentation:**  
    - Add sticky notes as per functional blocks for clarity and maintenance.

12. **Test Workflow:**  
    - Run manually or wait for scheduled trigger.  
    - Monitor for errors such as API quota issues or malformed inputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| This setup automates collection, normalization, and publication of tech news from diverse RSS feeds, ideal for newsletters or knowledge management.                                                                             | Sticky Note3 content                                                       |
| Full setup guide available at https://paoloronco.it/n8n-template-rss-tech-news-to-your-inbox/                                                                                                                                 | Sticky Note3 link                                                          |
| YouTube video explaining the workflow: https://www.youtube.com/watch?v=Gck8nmvx1UA                                                                                                                                             | Sticky Note3 YouTube                                                        |
| Editorial rules for AI summarization ensure factual, concise, and relevant news curation, avoiding hype or opinion.                                                                                                            | LLM node prompt content                                                   |
| Gmail node uses OAuth2 credentials; ensure proper credentials setup for email sending.                                                                                                                                           | Gmail node details                                                         |
| RSS feeds include multiple languages and sources; normalization is key for consistent processing.                                                                                                                              | General workflow design                                                   |
| The workflow is inactive by default (`active: false`); activate in your environment.                                                                                                                                           | Workflow metadata                                                          |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow. This process complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.