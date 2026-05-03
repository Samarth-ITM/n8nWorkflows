Generate Content Strategy Insights from Reddit & X Using Gemini AI Analysis

https://n8nworkflows.xyz/workflows/generate-content-strategy-insights-from-reddit---x-using-gemini-ai-analysis-11698


# Generate Content Strategy Insights from Reddit & X Using Gemini AI Analysis

### 1. Workflow Overview

This workflow automates market trend research and content strategy generation based on a user-submitted topic. It leverages AI-powered analysis and data aggregation from social media platforms Reddit and X (formerly Twitter). The workflow is designed for digital marketers, content strategists, and researchers aiming to uncover trending subtopics and validate content ideas with social engagement metrics.

The logical structure of the workflow is divided into the following blocks:

- **1.1 Input Reception:** Captures a user’s topic from a web form.
- **1.2 Topic Expansion (AI Processing):** Uses an AI agent to expand the main topic into relevant subtopics and keywords.
- **1.3 Market Research Data Retrieval:** Searches Reddit and X for content related to the generated keywords.
- **1.4 Data Transformation & Aggregation:** Extracts, formats, and consolidates social media post data.
- **1.5 Content Item Analysis (AI Evaluation):** AI evaluates each retrieved post for trend potential, audience relevance, and content format suggestions.
- **1.6 Trend Synthesis & Strategy Recommendation:** A final AI agent synthesizes insights from analyzed posts to recommend top trends and strategic content ideas.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow on submission of a topic via a form interface, initiating the entire research and analysis process.

**Nodes Involved:**  
- On form submission

**Node Details:**

- **On form submission**  
  - *Type & Role:* Form Trigger — initiates workflow from user input.  
  - *Configuration:* Single text input field labeled "topic" with placeholder "one word is enough", form titled "topic" and described as "Auto-Research Assistant for Market Trends".  
  - *Expressions:* Captures the entered topic as `$json.topic`.  
  - *Connections:* Output to "AI Agent1" node for topic expansion.  
  - *Edge Cases:* Input validation is minimal; empty or ambiguous inputs may reduce AI output quality. Webhook must be publicly accessible for form triggering.

---

#### 1.2 Topic Expansion (AI Processing)

**Overview:**  
Expands the user’s topic into multiple subtopics and generates relevant SEO-friendly keywords using an AI language model.

**Nodes Involved:**  
- AI Agent1  
- Google Gemini Chat Model2  
- Structured Output Parser

**Node Details:**

- **AI Agent1**  
  - *Type & Role:* Langchain AI Agent — defines a custom prompt to generate 5 subtopics and related keywords from the input topic.  
  - *Configuration:* Prompt instructs the AI to output a JSON listing subtopics and 2 keywords each.  
  - *Expressions:* Uses `{{ $json.topic }}` to insert user input.  
  - *Connections:* Output to "Google Gemini Chat Model2".  
  - *Edge Cases:* AI output may deviate from expected JSON; hence, structured parsing follows.  

- **Google Gemini Chat Model2**  
  - *Type & Role:* Google Gemini AI chat model node — executes the AI prompt.  
  - *Configuration:* Linked to Google Palm API credentials.  
  - *Connections:* Output to "Structured Output Parser1".  
  - *Edge Cases:* API rate limits or authentication failures can occur.

- **Structured Output Parser1**  
  - *Type & Role:* Structured output parser — auto-fixes and formats AI response into JSON using schema example reflecting expected structure with “topic” and “subtopics”.  
  - *Input:* AI response text.  
  - *Output:* Clean JSON with subtopics and keywords.  
  - *Edge Cases:* Parsing errors if AI output strays too far from schema.

---

#### 1.3 Market Research Data Retrieval

**Overview:**  
Fetches relevant posts from Reddit and X platforms for each generated keyword.

**Nodes Involved:**  
- Code1  
- Loop Over Items  
- HTTP Request (Reddit Search)  
- Search Tweets (X search)  
- Edit Fields  
- Edit Fields1  
- Aggregate  
- Aggregate1  
- Merge  
- add source (NoOp node)  
- add more source (NoOp node)

**Node Details:**

- **Code1**  
  - *Type & Role:* JavaScript code node — extracts all keywords from the AI-generated subtopics JSON and flattens them into individual items.  
  - *Input:* Structured JSON of subtopics/keywords.  
  - *Output:* List of keywords as individual messages.  
  - *Connections:* Output to "Loop Over Items".

- **Loop Over Items**  
  - *Type & Role:* Split in batches — iterates over each keyword to fetch social data.  
  - *Connections:* For each keyword, parallel requests to Reddit and X are initiated.  
  - *Output:* Merges results from both platforms.

- **HTTP Request**  
  - *Type & Role:* HTTP GET request — searches Reddit posts matching the keyword, limiting results to 5.  
  - *Configuration:* URL dynamically built as `https://www.reddit.com/search.json?q={{ $json.keyword }}`.  
  - *Edge Cases:* Reddit API rate limiting, network failures, or empty results.

- **Search Tweets**  
  - *Type & Role:* Twitter (X) node — searches tweets related to the keyword, limited to 10, sorted by recency.  
  - *Credentials:* Uses OAuth2 for X API access.  
  - *Error Handling:* Configured to continue workflow on errors (e.g., API limits).  
  - *Edge Cases:* Authentication failures, empty search results.

- **Code**  
  - *Type & Role:* JavaScript code — extracts and formats Reddit posts data into a consistent structure (title, url, subreddit, score, comments, etc.).  
  - *Input:* Raw Reddit API response.  
  - *Output:* Clean array of Reddit posts.

- **Edit Fields**  
  - *Type & Role:* Set node — adds metadata fields like `source = "reddit"` and `keyword` to each Reddit post item.  
  - *Input:* From "Code".  
  - *Output:* Annotated Reddit post items.

- **Edit Fields1**  
  - *Type & Role:* Set node — adds metadata for X posts including `source = "x (formally twitter)"`, and copies `text` and `keyword`.  
  - *Input:* From "Search Tweets".  
  - *Output:* Annotated X post items.

- **Aggregate & Aggregate1**  
  - *Type & Role:* Aggregation nodes — consolidate all Reddit and X posts respectively into single output arrays.

- **Merge**  
  - *Type & Role:* Merge node — combines aggregated Reddit and X data streams into one unified dataset.

- **add source / add more source**  
  - *Type & Role:* NoOp nodes — placeholders for future platform integrations.  
  - *Sticky Note:* Encourages adding more social platforms.

---

#### 1.4 Data Transformation & Aggregation

**Overview:**  
Splits merged social data and prepares it for individual AI analysis by looping over each content item.

**Nodes Involved:**  
- Split Out  
- Loop Over Items1

**Node Details:**

- **Split Out**  
  - *Type & Role:* Splits array of aggregated posts into individual items for separate processing.  
  - *Input:* Combined Reddit and X posts.  
  - *Output:* Single post items.

- **Loop Over Items1**  
  - *Type & Role:* Processes each content item individually in batches for scalable AI analysis.  
  - *Connections:* Splits the input into batches and feeds into "AI Agent3" and "AI Agent" nodes.

---

#### 1.5 Content Item Analysis (AI Evaluation)

**Overview:**  
Each social media post is analyzed by AI to extract insights like content summary, audience relevance, trend potential, and content format recommendations.

**Nodes Involved:**  
- AI Agent  
- Google Gemini Chat Model  
- Structured Output Parser  
- AI Agent3  
- Google Gemini Chat Model5  
- Code2

**Node Details:**

- **AI Agent**  
  - *Type & Role:* Langchain AI Agent — analyzes individual posts based on title, text, URL, subreddit, upvotes, comments, source, and keyword.  
  - *Prompt:* Requests 7 insights including summary, usefulness, audience, trend score, format suggestion, categories, and keyword.  
  - *Connections:* Output to "Loop Over Items1" (parallel to AI Agent3).  
  - *Edge Cases:* Posts with zero upvotes/comments are ignored for those metrics; textual fields may be missing.

- **Google Gemini Chat Model**  
  - *Type & Role:* Executes the AI prompt for "AI Agent".  
  - *Credentials:* Google Palm API.  

- **Structured Output Parser**  
  - *Type & Role:* Parses AI's JSON response for "AI Agent" into structured format.

- **AI Agent3**  
  - *Type & Role:* Langchain AI Agent — synthesizes batch of analyzed content items to select top 3-5 trends, group them by themes, and suggest content strategies.  
  - *Prompt:* Instructed to review all items, select top trends by score and uniqueness, group them, and provide strategy recommendations.  
  - *Connections:* Output to "Code2" for JSON cleanup.

- **Google Gemini Chat Model5**  
  - *Type & Role:* Executes AI prompt for "AI Agent3".  
  - *Credentials:* Google Palm API.

- **Code2**  
  - *Type & Role:* Cleans AI responses by removing markdown code blocks around JSON, parses it, and outputs clean JSON for downstream use or display.

---

#### 1.6 Trend Synthesis & Strategy Recommendation

**Overview:**  
Finalizes the trend analysis by cleaning and structuring the AI-generated strategic insights for output or further use.

**Nodes Involved:**  
- Code2 (see above)

**Node Details:**

- **Code2**  
  - *Role:* Post-processing of AI Agent3 output, ensuring clean JSON is returned for final use.  
  - *Edge Cases:* JSON parse errors if AI output format changes unexpectedly.

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                               | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                                               |
|-------------------------|--------------------------------|-----------------------------------------------|--------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger                   | Entry point: receives user topic input        | —                        | AI Agent1               | ## 1. get topic from user to split into more sub topics                                                                                   |
| AI Agent1               | Langchain AI Agent             | Expands topic into subtopics & keywords       | On form submission        | Google Gemini Chat Model2|                                                                                                                                           |
| Google Gemini Chat Model2| Google Gemini Chat Model       | Runs AI prompt for topic expansion             | AI Agent1                | Structured Output Parser1|                                                                                                                                           |
| Structured Output Parser1| Output Parser Structured      | Parses AI output JSON for subtopics/keywords  | Google Gemini Chat Model2 | Code1                   |                                                                                                                                           |
| Code1                   | Code                           | Extracts keywords from AI output               | Structured Output Parser1 | Loop Over Items          |                                                                                                                                           |
| Loop Over Items         | Split In Batches                | Iterates over keywords to collect social data | Code1                    | HTTP Request, Search Tweets, add source, add more source | ## 2. do market research from different platform                                                                                           |
| HTTP Request            | HTTP Request                   | Search Reddit posts for keywords               | Loop Over Items           | Code                    |                                                                                                                                           |
| Code                    | Code                           | Extracts and formats Reddit post data          | HTTP Request              | Edit Fields              |                                                                                                                                           |
| Edit Fields             | Set                            | Adds Reddit metadata fields                     | Code                     | Aggregate                |                                                                                                                                           |
| Search Tweets           | Twitter (X)                    | Searches X for posts by keyword                 | Loop Over Items           | Edit Fields1             |                                                                                                                                           |
| Edit Fields1            | Set                            | Adds X metadata fields                          | Search Tweets             | Aggregate1               |                                                                                                                                           |
| Aggregate               | Aggregate                      | Aggregates all Reddit posts                     | Edit Fields               | Merge                    |                                                                                                                                           |
| Aggregate1              | Aggregate                      | Aggregates all X posts                          | Edit Fields1              | Merge                    |                                                                                                                                           |
| Merge                   | Merge                          | Combines Reddit and X datasets                  | Aggregate, Aggregate1     | Split Out                | ## add more platforms                                                                                                                      |
| add source              | NoOp                           | Placeholder for additional platform integration| Loop Over Items           | —                        | ## add more platforms                                                                                                                      |
| add more source         | NoOp                           | Placeholder for additional platform integration| Loop Over Items           | —                        | ## add more platforms                                                                                                                      |
| Split Out               | Split Out                      | Splits combined posts into individual items    | Merge                     | Loop Over Items1          | ## 3. summarize posts                                                                                                                      |
| Loop Over Items1        | Split In Batches               | Processes each post individually for AI analysis| Split Out                | AI Agent3, AI Agent       | ## 4. analyze the trend                                                                                                                    |
| AI Agent                | Langchain AI Agent             | Analyzes individual posts for trend potential | Loop Over Items1          | Google Gemini Chat Model  | ## 4. analyze the trend                                                                                                                    |
| Google Gemini Chat Model| Google Gemini Chat Model       | Runs AI prompt for individual post analysis    | AI Agent                  | Structured Output Parser  | ## 4. analyze the trend                                                                                                                    |
| Structured Output Parser| Output Parser Structured       | Parses AI analysis output JSON                   | Google Gemini Chat Model  | AI Agent                 |                                                                                                                                           |
| AI Agent3               | Langchain AI Agent             | Synthesizes analyzed posts into final trends   | Loop Over Items1          | Google Gemini Chat Model5 | ## 4. analyze the trend                                                                                                                    |
| Google Gemini Chat Model5| Google Gemini Chat Model      | Runs AI prompt for trend synthesis & strategy  | AI Agent3                 | Code2                    | ## 4. analyze the trend                                                                                                                    |
| Code2                   | Code                           | Cleans and parses final AI JSON output          | Google Gemini Chat Model5 | —                        |                                                                                                                                           |
| Sticky Note             | Sticky Note                   | Content note on adding more platforms           | —                        | —                        | ## add more platforms                                                                                                                      |
| Sticky Note1            | Sticky Note                   | Overview of workflow operation & setup          | —                        | —                        | ## Trend Analyzer & Content Strategist\n\n### How it works\n1. Submit a topic...\n\n### Setup\n- Connect Google Gemini API\n- Connect X account |
| Sticky Note2            | Sticky Note                   | Block 1.1 user input description                 | —                        | —                        | ## 1. get topic from user to split into more sub topics                                                                                   |
| Sticky Note3            | Sticky Note                   | Block 1.3 market research description            | —                        | —                        | ## 2. do market research from different platform                                                                                           |
| Sticky Note4            | Sticky Note                   | Block 1.5 post summarization                      | —                        | —                        | ## 3. summarize posts                                                                                                                      |
| Sticky Note5            | Sticky Note                   | Block 1.6 trend analysis & strategy               | —                        | —                        | ## 4. analyze the trend                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node (Form Trigger):**  
   - Title: "topic"  
   - Description: "Auto-Research Assistant for Market Trends"  
   - Add one text field: label "topic", placeholder "one word is enough".  
   - This node triggers the workflow on user input.

2. **Create "AI Agent1" (Langchain AI Agent):**  
   - Prompt: Instruct AI to expand input topic into 5 subtopics, each with 2 SEO-friendly keywords in a specific JSON format.  
   - Use expression to pass user input: `{{ $json.topic }}`.  
   - Connect output from "On form submission" to this node.

3. **Create "Google Gemini Chat Model2" node:**  
   - Credentials: Connect Google Palm API with valid account.  
   - Connect input from "AI Agent1" to this node.

4. **Create "Structured Output Parser1":**  
   - Enable auto-fix.  
   - Provide JSON schema example matching expected AI output (topic, subtopics with keywords).  
   - Connect input from "Google Gemini Chat Model2".

5. **Create "Code1" (Code node):**  
   - JavaScript code to extract keywords array from parsed subtopics and map them into separate items.  
   - Connect input from "Structured Output Parser1".

6. **Create "Loop Over Items" (SplitInBatches):**  
   - Connect input from "Code1".  
   - This node will iterate over each keyword to fetch social data.

7. **Create "HTTP Request":**  
   - Method: GET  
   - URL: `https://www.reddit.com/search.json?q={{ $json.keyword }}`  
   - Query parameter: limit=5  
   - Connect input from "Loop Over Items".

8. **Create "Code" node:**  
   - JavaScript code to parse Reddit JSON response and extract relevant fields (title, url, subreddit, score, comments, etc.).  
   - Connect input from "HTTP Request".

9. **Create "Edit Fields":**  
   - Add static field `source` = "reddit".  
   - Add field `keyword` with value from `$('Code1').item.json.keyword`.  
   - Include other fields from input.  
   - Connect input from "Code".

10. **Create "Search Tweets":**  
    - Operation: Search  
    - Limit: 10  
    - Search text: `={{ $json.keyword }}`  
    - Sort order: recency  
    - Credentials: Connect OAuth2 for Twitter (X) account.  
    - Connect input from "Loop Over Items".

11. **Create "Edit Fields1":**  
    - Add static field `source` = "x (formally twitter)".  
    - Add field `text` from tweet text.  
    - Add field `keyword` from `$('Code1').item.json.keyword`.  
    - Connect input from "Search Tweets".

12. **Create "Aggregate" and "Aggregate1":**  
    - Aggregate all Reddit items from "Edit Fields".  
    - Aggregate all X items from "Edit Fields1".

13. **Create "Merge":**  
    - Merge the outputs from "Aggregate" and "Aggregate1".  
    - Connect both aggregate nodes as inputs.

14. **Create "Split Out":**  
    - Field to split out: `data[0].keyword` (to split combined array of posts).  
    - Connect input from "Merge".

15. **Create "Loop Over Items1":**  
    - Split in batches to process each post separately.  
    - Connect input from "Split Out".

16. **Create "AI Agent" (for post analysis):**  
    - Prompt AI to analyze each post's content, engagement metrics, and suggest summary, audience, trend score (0-100), format suggestion, categories, and keyword.  
    - Connect input from "Loop Over Items1".

17. **Create "Google Gemini Chat Model":**  
    - Credentials: Google Palm API.  
    - Connect input from "AI Agent".

18. **Create "Structured Output Parser":**  
    - Auto-fix enabled with JSON schema matching expected AI analysis output.  
    - Connect input from "Google Gemini Chat Model".

19. **Create "AI Agent3" (Trend synthesis and strategy):**  
    - Prompt AI to review all analyzed posts, select top 3-5 trends by trend_score and relevance, group related trends, and suggest content strategies.  
    - Connect input from "Loop Over Items1" (parallel to AI Agent).

20. **Create "Google Gemini Chat Model5":**  
    - Credentials: Google Palm API.  
    - Connect input from "AI Agent3".

21. **Create "Code2":**  
    - JavaScript code to clean AI output by removing markdown code block markers and parsing JSON.  
    - Connect input from "Google Gemini Chat Model5".

22. **Create "NoOp" nodes "add source" and "add more source":**  
    - Placeholders for future platform support.  
    - Connect as branches from "Loop Over Items" node.

23. **Add Sticky Notes as per positions to document workflow sections and reminders:**

- "On form submission" node area: "## 1. get topic from user to split into more sub topics"  
- Around market research nodes: "## 2. do market research from different platform"  
- Near merge and split out nodes: "## add more platforms"  
- Near post summarization and analysis: "## 3. summarize posts" and "## 4. analyze the trend"  
- Workflow overview note: "## Trend Analyzer & Content Strategist\n\n### How it works\n1. Submit a topic...\n\n### Setup\n- Connect your Google Gemini API account.\n- Connect your X account..."

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                            |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Workflow uses Google Gemini (PaLM) API for all AI language model interactions.                                   | Requires Google Palm API credentials setup in n8n.         |
| X (formerly Twitter) API accessed via OAuth2 credentials; ensure proper API keys and permissions are granted.   | Twitter/X API documentation for OAuth2 setup.              |
| Workflow is designed modularly to allow adding more social platforms (placeholders included).                    | "add source" and "add more source" NoOp nodes in workflow. |
| Sticky note includes detailed operation and setup instructions for user reference.                              | Located near "On form submission" node.                     |
| AI prompt templates embedded in Langchain agent nodes; customizable for targeted research or analysis needs.  | Prompts include JSON output formatting for parsing ease.   |

---

This structured documentation provides a complete understanding of the workflow's design, stepwise node roles, and instructions for recreation and modification, enabling advanced users and AI agents to work with the process effectively.