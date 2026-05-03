Create AI-Curated Tech News Digests with GPT-4.1 Mini and Notion Database

https://n8nworkflows.xyz/workflows/create-ai-curated-tech-news-digests-with-gpt-4-1-mini-and-notion-database-8665


# Create AI-Curated Tech News Digests with GPT-4.1 Mini and Notion Database

### 1. Workflow Overview

This workflow automates the creation of a **daily curated technology and startup news digest** by fetching articles from a Notion database, filtering and classifying them using AI, summarizing the content with GPT-4.1 Mini, formatting the summary into a Notion-compatible page structure, and finally publishing the digest back into Notion.

**Target Use Cases:**
- Tech editors or content curators who want to automate daily news digest creation.
- Teams or individuals tracking technology and startup news from a centralized Notion feed.
- Anyone looking to leverage AI for content summarization and publication in Notion.

**Logical Blocks:**

- **1.1 Input Reception**
  - Trigger the workflow manually or via schedule.
  - Fetch recent articles from a Notion database filtered by date.

- **1.2 Initial Filtering**
  - Filter articles based on keyword presence related to tech/startup topics.

- **1.3 AI Classification**
  - Use an AI text classifier (GPT-4.1 Mini) to further categorize articles as "Tech/Startup" or "Other".

- **1.4 Aggregation**
  - Combine filtered and classified articles into a single structured array.

- **1.5 AI Digest Generation**
  - Use an AI agent with GPT-4.1 Mini to generate a professional daily digest in Markdown.

- **1.6 Markdown to Notion Format Conversion**
  - Convert the AI-generated Markdown digest into Notion blocks JSON structure.

- **1.7 Notion Page Creation**
  - Create a new Notion page containing the digest under a specific parent page.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  The workflow starts either via manual trigger or schedule to initiate daily news digest creation. It then fetches recent articles from a specified Notion database filtered by date to get only fresh content.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Schedule Trigger (disabled)  
  - Get many database pages (Notion)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation for testing or on-demand execution.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connected to "Get many database pages" node.  
    - Edge Cases: Manual trigger requires user interaction; no auto-run.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Allows automated daily triggering (disabled in this workflow).  
    - Configuration: Set to trigger daily at 20:00 hours (8 PM).  
    - Inputs: None  
    - Outputs: Connected to "Get many database pages" node.  
    - Edge Cases: Disabled by default; needs enabling and credential setup for scheduled runs.

  - **Get many database pages**  
    - Type: Notion node, operation "getAll" on databasePage  
    - Role: Fetches all pages from a specific Notion database ("Tech & Startups rss feed") filtered for pages created or modified after yesterday.  
    - Configuration:  
      - DatabaseId: Hardcoded to the Notion database for tech/startup articles.  
      - Filter: Date property "Date" greater than yesterday (ISO string at runtime).  
      - ReturnAll: True (fetch all matching pages).  
    - Credentials: Notion API with access to database.  
    - Inputs: From manual or schedule trigger.  
    - Outputs: Passes fetched articles to "Code in JavaScript" for filtering.  
    - Edge Cases: Potential auth errors if token invalid; empty results if no articles found.

---

#### 1.2 Initial Filtering

- **Overview:**  
  Filters the fetched articles using a keyword-based approach to retain only those relevant to technology and startups.

- **Nodes Involved:**  
  - Code in JavaScript

- **Node Details:**

  - **Code in JavaScript**  
    - Type: Code node (JavaScript)  
    - Role: Filters articles by checking if their title, summary, or full article text contain any keywords from a predefined tech/startup list.  
    - Configuration:  
      - Keywords include "tech", "startup", "ai", "google", "microsoft", "apple", "vc", "funding", "ipo", "tiktok", "meta", etc.  
      - Filter logic is case-insensitive.  
    - Input: Articles from Notion database query.  
    - Output: Only articles matching any keyword continue.  
    - Edge Cases: May exclude relevant articles if keywords missing; false positives possible if keywords appear out of context.

---

#### 1.3 AI Classification

- **Overview:**  
  Uses an AI text classifier to further categorize filtered articles as "Tech/Startup" or "Other", allowing more precise content curation.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Text Classifier  
  - Code in JavaScript1

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Language Model Chat (OpenAI GPT-4.1 Mini)  
    - Role: Interfaces with GPT-4.1 Mini to classify article text.  
    - Configuration: Model set to "gpt-4.1-mini".  
    - Credentials: OpenAI API key.  
    - Input/Output: Receives article text from "Code in JavaScript" via "Text Classifier".  
    - Edge Cases: API rate limits, timeouts, or invalid API key may cause failures.

  - **Text Classifier**  
    - Type: Langchain textClassifier node  
    - Role: Prompts AI to classify each article into either "Tech/Startup" or "Other" based on title, summary, and full article.  
    - Configuration:  
      - Input template asks for single-word classification.  
      - Categories defined explicitly.  
    - Input: Article fields from previous node.  
    - Output: Classified articles forwarded only if "Tech/Startup".  
    - Edge Cases: Misclassification possible; AI hallucinations or ambiguous content.

  - **Code in JavaScript1**  
    - Type: Code node (JavaScript)  
    - Role: Aggregates all classified articles into a single JSON array to prepare for AI digest generation.  
    - Input: Classified articles from "Text Classifier".  
    - Output: One item with array of all articles under `articles` property.  
    - Edge Cases: Empty input results in empty array, which downstream nodes must handle.

---

#### 1.4 AI Digest Generation

- **Overview:**  
  An AI agent generates a polished daily digest summary in Markdown format, grouping articles by relevant categories and producing editorial-style bullet points.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model1

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain Agent node  
    - Role: Receives all articles and generates a professional daily digest following detailed instructions.  
    - Configuration:  
      - System message instructs to create a daily digest with intro, grouped categories, bullet points per article, and closing note.  
      - Formatting rules specify Markdown headings and clickable links, no raw JSON or AI meta-commentary.  
    - Input: Array of articles from "Code in JavaScript1".  
    - Output: Markdown digest text under `output`.  
    - Edge Cases: AI could fail to follow formatting precisely; long inputs could cause token limit issues.

  - **OpenAI Chat Model1**  
    - Type: Language Model Chat (OpenAI GPT-4.1 Mini)  
    - Role: Executes the AI agent prompt, generating the digest.  
    - Configuration: Model set to "gpt-4.1-mini".  
    - Credentials: OpenAI API key.  
    - Input: Digest prompt from "AI Agent".  
    - Output: Markdown formatted digest.  
    - Edge Cases: API limits or invalid credentials.

---

#### 1.5 Markdown to Notion Format Conversion

- **Overview:**  
  Converts the AI-generated Markdown digest into a JSON structure compatible with the Notion API to create rich content blocks.

- **Nodes Involved:**  
  - Code in JavaScript2

- **Node Details:**

  - **Code in JavaScript2**  
    - Type: Code node (JavaScript)  
    - Role: Parses Markdown line-by-line, converting headings, bullet points, dividers, and paragraphs into corresponding Notion block objects.  
    - Configuration:  
      - Supports headings (#, ##, ###), bulleted lists with clickable Markdown links, horizontal rules, and paragraphs.  
      - Splits blocks into chunks of 100 blocks due to Notion API limits; only first chunk is sent immediately.  
      - Prepares a `pagePayload` JSON object including title, icon, cover image, and children blocks for Notion page creation.  
      - Hardcoded parent page ID for digest storage.  
    - Input: Markdown digest text from AI output.  
    - Output: JSON object for Notion page creation and extra chunks for potential appending.  
    - Edge Cases: Complex Markdown might not be fully supported; links malformed or missing; chunking logic must handle large content.

---

#### 1.6 Notion Page Creation

- **Overview:**  
  Creates a new page in Notion with the formatted daily digest content.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request node  
    - Role: Calls Notion API to create a new page with the digest content.  
    - Configuration:  
      - Method: POST  
      - URL: https://api.notion.com/v1/pages  
      - Headers include Authorization Bearer token and Notion-Version.  
      - JSON body taken from `pagePayload` prepared in "Code in JavaScript2".  
    - Credentials: Notion API key with page creation rights.  
    - Input: JSON formatted page data.  
    - Output: Response from Notion API (created page info).  
    - Edge Cases: Auth failures, API rate limits, invalid payload errors, parent page not found.

---

### 3. Summary Table

| Node Name                     | Node Type                                  | Functional Role                                  | Input Node(s)                       | Output Node(s)                   | Sticky Note                                                                                   |
|-------------------------------|--------------------------------------------|-------------------------------------------------|-----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                            | Manual workflow start trigger                    | None                              | Get many database pages          | manual activation is for testing                                                               |
| Schedule Trigger              | Schedule Trigger (disabled)                 | Scheduled workflow start (daily at 20:00)       | None                              | Get many database pages          | we can schedule it for daily                                                                   |
| Get many database pages       | Notion node (getAll database pages)        | Fetch recent articles from Notion database       | When clicking ‘Execute workflow’, Schedule Trigger | Code in JavaScript              | date filter can be modified for desired days or may be hours to fetch articles from notion database |
| Code in JavaScript            | Code node (JavaScript)                      | Initial keyword-based filtering of articles      | Get many database pages           | Text Classifier                  | this code does initial filtering for required articles for our these **Tech and startup**      |
| OpenAI Chat Model             | Langchain Chat Model (OpenAI GPT-4.1 Mini) | Runs AI classification prompt                     | Text Classifier                  | Text Classifier                  | more filtering for articles with text classifier model used is (gpt-4.1-mini)                  |
| Text Classifier               | Langchain Text Classifier                   | Classifies articles as "Tech/Startup" or "Other" | OpenAI Chat Model                | Code in JavaScript1              | more filtering for articles with text classifier model used is (gpt-4.1-mini)                  |
| Code in JavaScript1           | Code node (JavaScript)                      | Aggregates filtered articles into one array      | Text Classifier                  | AI Agent                       | combile articles in one object to pass into ai agent                                          |
| AI Agent                     | Langchain Agent                             | Generates daily digest summary in Markdown       | Code in JavaScript1              | Code in JavaScript2              | generate summaries of all articles into single article with raw notion page format             |
| OpenAI Chat Model1            | Langchain Chat Model (OpenAI GPT-4.1 Mini) | Runs AI agent prompt                              | AI Agent                       | AI Agent                       | generate summaries of all articles into single article with raw notion page format             |
| Code in JavaScript2           | Code node (JavaScript)                      | Converts Markdown digest into Notion page JSON   | AI Agent                       | HTTP Request                   | format into correct notion page json object to pass into http node                            |
| HTTP Request                 | HTTP Request node                           | Creates Notion page with daily digest             | Code in JavaScript2              | None                           | create notion page; here created for specific day                                             |
| Sticky Note1                 | Sticky Note                                | Comment: manual activation is for testing        | None                              | None                           | manual activation is for testing                                                               |
| Sticky Note2                 | Sticky Note                                | Comment: scheduling workflow daily                | None                              | None                           | we can schedule it for daily                                                                   |
| Sticky Note3                 | Sticky Note                                | Comment: date filter modifiable                    | None                              | None                           | date filter can be modified for desired days or may be hours to fetch articles from notion database |
| Sticky Note4                 | Sticky Note                                | Comment: initial keyword filtering                 | None                              | None                           | this code does initial filtering for required articles for our these **Tech and startup**      |
| Sticky Note5                 | Sticky Note                                | Comment: AI text classification filtering         | None                              | None                           | more filtering for articles with text classifier model used is (gpt-4.1-mini)                  |
| Sticky Note6                 | Sticky Note                                | Comment: aggregate articles into one object       | None                              | None                           | combile articles in one object to pass into ai agent                                          |
| Sticky Note7                 | Sticky Note                                | Comment: AI generates raw notion page format      | None                              | None                           | generate summaries of all articles into single article with raw notion page format             |
| Sticky Note8                 | Sticky Note                                | Comment: format into Notion page JSON              | None                              | None                           | format into correct notion page json object to pass into http node                            |
| Sticky Note9                 | Sticky Note                                | Comment: creates daily digest Notion page          | None                              | None                           | create notion page; here created for specific day                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Enable manual start/testing  
   - No parameters needed  
   - Position: Start of workflow

2. **(Optional) Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 20:00 (8 PM)  
   - Disable initially if manual testing is preferred

3. **Create Notion Node to Fetch Database Pages**  
   - Type: Notion  
   - Operation: getAll on resource databasePage  
   - Database ID: Set to your Notion database containing articles  
   - Filter: Date property "Date" greater than yesterday's date (use expression `{{$now.minus({days:1}).toISO()}}`)  
   - Return All: True  
   - Credentials: Connect to a Notion API credential with read access  
   - Connect both Manual and Schedule triggers to this node

4. **Create JavaScript Code Node for Keyword Filtering**  
   - Type: Code (JavaScript)  
   - Paste filtering code that checks article title, summary, and full article for keywords such as “tech”, “startup”, “ai”, etc.  
   - Input: From Notion node  
   - Output: Filtered articles only

5. **Create OpenAI Chat Model Node for Text Classification**  
   - Type: Langchain lmChatOpenAi  
   - Model: Select "gpt-4.1-mini"  
   - Credentials: OpenAI API key  
   - Input: Article text from previous node (via Text Classifier node)  
   - Connect output to Text Classifier node

6. **Create Text Classifier Node**  
   - Type: Langchain textClassifier  
   - Input Text Template: Prompt asking to classify into "Tech/Startup" or "Other" based on article fields  
   - Categories: "Tech/Startup" and "Other"  
   - Connect output to next JavaScript node

7. **Create JavaScript Node to Aggregate Classified Articles**  
   - Type: Code (JavaScript)  
   - Combine all classified articles into a single array under the property `articles`  
   - Input: From Text Classifier  
   - Output: Single item with `articles` array

8. **Create AI Agent Node for Digest Generation**  
   - Type: Langchain Agent  
   - System Message: Instructions to generate a professional daily digest with intro, grouped categories, bullet points, and closing note in Markdown  
   - Input: `articles` array from previous node  
   - Connect output to OpenAI Chat Model1 node

9. **Create OpenAI Chat Model Node 1**  
   - Type: Langchain lmChatOpenAi  
   - Model: "gpt-4.1-mini"  
   - Credentials: OpenAI API key  
   - Input: Text prompt from AI Agent  
   - Output: Markdown formatted digest

10. **Create JavaScript Node to Convert Markdown to Notion Blocks**  
    - Type: Code (JavaScript)  
    - Code parses Markdown lines into Notion block objects (headings, bullet lists, paragraphs, dividers)  
    - Chunk blocks into maximum 100 items (Notion API limit)  
    - Prepare `pagePayload` JSON with:  
      - Parent page ID (your Notion parent page)  
      - Icon and cover image settings  
      - Title: "Tech & Startup Daily Digest – YYYY-MM-DD" (today’s date)  
      - Children: first chunk of Notion blocks  
    - Output: JSON for Notion API call

11. **Create HTTP Request Node to Create Notion Page**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: https://api.notion.com/v1/pages  
    - Headers:  
      - Authorization: Bearer YOUR_NOTION_API_KEY  
      - Notion-Version: 2022-06-28  
    - Body: JSON, taken from previous node’s `pagePayload`  
    - Credentials: Notion API key with write access  
    - Input: JSON page payload

12. **Connect all nodes accordingly:**  
    - Manual Trigger -> Get many database pages  
    - Schedule Trigger (optional) -> Get many database pages  
    - Get many database pages -> Code in JavaScript (keyword filtering)  
    - Code in JavaScript -> OpenAI Chat Model -> Text Classifier  
    - Text Classifier -> Code in JavaScript1 (aggregate)  
    - Code in JavaScript1 -> AI Agent -> OpenAI Chat Model1  
    - OpenAI Chat Model1 -> Code in JavaScript2 (Markdown to Notion JSON)  
    - Code in JavaScript2 -> HTTP Request (create Notion page)

**Important Notes:**  
- Replace placeholders for API keys and Notion database/page IDs with your own.  
- Ensure OpenAI and Notion credentials are correctly set and authorized.  
- Adjust date filtering as needed for different time windows.  
- Monitor for API rate limits and errors during AI calls and Notion page creation.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Manual trigger node is intended primarily for testing the workflow before enabling scheduling.                | Sticky Note1                                                                                     |
| Scheduling can be enabled for daily automation at a fixed time (default 20:00).                                | Sticky Note2                                                                                     |
| Date filter in the Notion fetch node can be customized to fetch articles from a specific timeframe or hours.  | Sticky Note3                                                                                     |
| Initial JavaScript filtering focuses on keyword-based selection relevant to tech and startups.                 | Sticky Note4                                                                                     |
| AI text classification uses GPT-4.1 Mini model for more precise filtering of articles.                         | Sticky Note5                                                                                     |
| Articles are combined into a single object before passing to the AI digest generation agent.                   | Sticky Note6                                                                                     |
| AI agent creates the digest in Markdown format ready for conversion to Notion page content.                    | Sticky Note7                                                                                     |
| Markdown-to-Notion conversion code formats content into valid Notion API JSON block structures.                | Sticky Note8                                                                                     |
| Final HTTP Request node creates a new Notion page for the daily digest under a specified parent page.          | Sticky Note9                                                                                     |
| GPT-4.1 Mini is used for both classification and summarization, balancing quality and cost.                    | Model selection noted in multiple nodes                                                         |
| Parent page ID and Notion API keys must be replaced with user-specific values for production use.             | Specified in Code in JavaScript2 and HTTP Request nodes                                         |
| This workflow follows best practices for chunking large Notion content into API-compliant payloads.           | Code in JavaScript2 chunking logic                                                              |

---

*Disclaimer: The provided content is exclusively generated from an automated workflow designed with n8n. It complies with all relevant content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.*