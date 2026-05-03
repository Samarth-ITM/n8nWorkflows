Generate Social Media Hashtags from Twitter & YouTube Trends with Mistral AI

https://n8nworkflows.xyz/workflows/generate-social-media-hashtags-from-twitter---youtube-trends-with-mistral-ai-6203


# Generate Social Media Hashtags from Twitter & YouTube Trends with Mistral AI

### 1. Workflow Overview

This workflow automates the generation of social media hashtags inspired by current Twitter and YouTube trending topics, leveraging Mistral AI language models for creative hashtag synthesis. It is designed for social media managers, marketers, or content creators aiming to stay relevant and innovative with hashtag strategies based on live trends. The workflow is composed of the following logical blocks:

- **1.1 Input Reception (Scheduled Trigger):** Initiates the workflow daily to fetch fresh trend data.
- **1.2 Data Extraction (Web Scraping):** Scrapes trending hashtags and keywords from Twitter and YouTube trends websites.
- **1.3 Data Filtering and Preparation:** Extracts relevant HTML content from the scrapes and limits the dataset to the top 100 trends.
- **1.4 Trend Merging:** Combines the filtered Twitter and YouTube trends into a single dataset.
- **1.5 Hashtag Generation (AI Processing):** Uses Mistral AI to generate a new, smart hashtag based on the merged trends and a specified keyword.
- **1.6 Output Handling:** Parses the AI response and stores the results in Google Sheets for record-keeping and further use.
- **1.7 Iterative Processing:** Processes generated hashtags in batches before appending them to the sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block triggers the entire workflow once per day, ensuring that the hashtag generation is based on fresh, daily trends.
- **Nodes Involved:** `daily trigger`
- **Node Details:**
  - **Node:** daily trigger
  - **Type:** Schedule Trigger
  - **Configuration:** Set to trigger daily (interval with default settings).
  - **Inputs:** None (start node)
  - **Outputs:** Triggers both Twitter and YouTube trend extraction nodes.
  - **Failure Modes:** Misconfiguration of schedule; workflow not triggering; time zone considerations.
  - **Version:** 1.2

#### 2.2 Data Extraction (Web Scraping)

- **Overview:** Extracts raw HTML data from two trend websites: Twitter trends from `trends24.in` and YouTube trends from `youtube.trends24.in`.
- **Nodes Involved:** `extract twitter trends`, `extract YouTube trends`
- **Node Details:**
  - **Node:** extract twitter trends
    - **Type:** Crawlee Node (Community Node for web scraping)
    - **Configuration:** URL set to https://trends24.in/, operation: extractHtml
    - **Inputs:** Trigger from `daily trigger`
    - **Outputs:** Raw HTML data passed to `filter twitter trends`
    - **Failure Modes:** Network errors, site structure changes, scraping blocks by website.
    - **Version:** 1
  - **Node:** extract YouTube trends
    - **Type:** Crawlee Node
    - **Configuration:** URL set to https://youtube.trends24.in/, operation: extractHtml
    - **Inputs:** Trigger from `daily trigger`
    - **Outputs:** Raw HTML data passed to `filter YouTube trends`
    - **Failure Modes:** Same as above.
    - **Version:** 1

#### 2.3 Data Filtering and Preparation

- **Overview:** Extracts relevant trend items from the raw HTML for both Twitter and YouTube, then limits Twitter trends to the top 100.
- **Nodes Involved:** `filter twitter trends`, `filter YouTube trends`, `get only top 100 trends`
- **Node Details:**
  - **Node:** filter twitter trends
    - **Type:** HTML Extract
    - **Configuration:** Extracts HTML content from `.trend-card__list li a` elements, cleaning up text.
    - **Inputs:** From `extract twitter trends`
    - **Outputs:** Passes extracted trends to `get only top 100 trends`
    - **Failure Modes:** CSS selector may break if website changes; empty extraction.
    - **Version:** 1.2
  - **Node:** filter YouTube trends
    - **Type:** HTML Extract
    - **Configuration:** Extracts HTML content from `.keywords-list li` elements, cleaning up text.
    - **Inputs:** From `extract YouTube trends`
    - **Outputs:** Passes extracted trends to `Merge` node (index 1)
    - **Failure Modes:** Same as above.
    - **Version:** 1.2
  - **Node:** get only top 100 trends
    - **Type:** Code (JavaScript)
    - **Configuration:** Slices the array of Twitter trends to keep only first 100 items.
    - **Inputs:** From `filter twitter trends`
    - **Outputs:** Passes trimmed trends to `Merge` node (index 0)
    - **Failure Modes:** Input trends may have fewer than 100 entries; code errors if input missing.
    - **Version:** 2
    - **Code:** `return items.map(item => { item.json.trends = item.json.trends.slice(0, 100); return item; });`

#### 2.4 Trend Merging

- **Overview:** Combines Twitter and YouTube filtered trends into a single data stream for hashtag generation.
- **Nodes Involved:** `Merge`
- **Node Details:**
  - **Node:** Merge
  - **Type:** Merge
  - **Configuration:** Default merge mode; merges two input streams: Twitter trends (index 0) and YouTube trends (index 1)
  - **Inputs:** From `get only top 100 trends` and `filter YouTube trends`
  - **Outputs:** Passes merged trends to `hashtag generator`
  - **Failure Modes:** Mismatched input sizes; data structure inconsistencies.
  - **Version:** 3.1

#### 2.5 Hashtag Generation (AI Processing)

- **Overview:** Uses a LangChain Mistral Cloud Chat model and an Agent node to generate a unique, brand-safe hashtag inspired by the merged trends and a given keyword.
- **Nodes Involved:** `Mistral Cloud Chat Model1`, `Structured Output Parser1`, `hashtag generator`
- **Node Details:**
  - **Node:** Mistral Cloud Chat Model1
    - **Type:** LangChain Language Model (Mistral Cloud)
    - **Configuration:** Model set to `mistral-small-latest` with default options.
    - **Inputs:** Receives prompt from `hashtag generator` AI input.
    - **Outputs:** AI-generated chat response sent back to `hashtag generator`
    - **Credentials:** Requires valid Mistral Cloud API credentials.
    - **Failure Modes:** API connection failures, rate limits, invalid credentials, model downtime.
    - **Version:** 1
  - **Node:** Structured Output Parser1
    - **Type:** LangChain Output Parser (Structured)
    - **Configuration:** Expects JSON output with a key "hashtag" containing an array of strings.
    - **Inputs:** Receives AI output from `hashtag generator`
    - **Outputs:** Parsed structured output back to `hashtag generator`
    - **Failure Modes:** Malformed JSON from AI, parsing errors.
    - **Version:** 1.2
  - **Node:** hashtag generator
    - **Type:** LangChain Agent
    - **Configuration:** Receives merged trends as input, with a system message instructing the generation of one short, lowercase, catchy hashtag related to a given keyword and trends. The prompt explicitly requests a JSON object with key `new_hashtag`.
    - **Inputs:** Receives trends from `Merge` node; AI model and output parser assigned here.
    - **Outputs:** Sends generated hashtag to the batch processing node `Loop Over Items2`.
    - **Failure Modes:** Expression evaluation errors, AI errors, prompt misinterpretation.
    - **Version:** 2

#### 2.6 Output Handling and Iterative Processing

- **Overview:** Processes the AI-generated hashtag(s) in batches and appends them to a Google Sheet for storage.
- **Nodes Involved:** `Loop Over Items2`, `Google Sheets2`
- **Node Details:**
  - **Node:** Loop Over Items2
    - **Type:** Split In Batches
    - **Configuration:** Default batch settings, processing batches without reset.
    - **Inputs:** From `hashtag generator`
    - **Outputs:** Passes each batch item to `Google Sheets2`
    - **Failure Modes:** Batch size misconfiguration, memory or timeout issues with large batches.
    - **Version:** 3
  - **Node:** Google Sheets2
    - **Type:** Google Sheets
    - **Configuration:** Appends a row to the sheet named "Sheet1" in document ID `1A9EHzMz88dlmBZm1ooQasWb4sU3OTv2tuMtuu1yBAyw`. The column "hastag " is mapped to `output.hashtag` from the AI output.
    - **Inputs:** From `Loop Over Items2`
    - **Outputs:** None (terminal node)
    - **Credentials:** Requires authenticated Google Sheets OAuth2 credentials.
    - **Failure Modes:** Authentication failure, quota limits, sheet permission issues, data mapping errors.
    - **Version:** 4.6

---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                             | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                   |
|-------------------------|-----------------------------------------|---------------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| daily trigger           | Schedule Trigger                        | Starts the workflow daily                    | None                             | extract twitter trends, extract YouTube trends |                                                                                              |
| extract twitter trends  | Crawlee Node (Web Scraper)              | Scrapes Twitter trending data                | daily trigger                   | filter twitter trends            | extract twitter and youtube trends from https://trends24.in/  by   using crawl and scrape community node scrape the websites |
| extract YouTube trends  | Crawlee Node (Web Scraper)              | Scrapes YouTube trending data                 | daily trigger                   | filter YouTube trends            | extract twitter and youtube trends from https://trends24.in/  by   using crawl and scrape community node scrape the websites |
| filter twitter trends   | HTML Extract                           | Extracts Twitter trend hashtags from HTML    | extract twitter trends          | get only top 100 trends          | filters hashtags out of scraper output                                                       |
| filter YouTube trends   | HTML Extract                           | Extracts YouTube trend hashtags from HTML    | extract YouTube trends          | Merge                          | filters hashtags out of scraper output                                                       |
| get only top 100 trends | Code (JavaScript)                      | Limits Twitter trends to top 100              | filter twitter trends           | Merge                          | filters hashtags out of scraper output                                                       |
| Merge                   | Merge                                 | Combines Twitter and YouTube trends           | get only top 100 trends, filter YouTube trends | hashtag generator             |                                                                                              |
| hashtag generator       | LangChain Agent                       | Generates new hashtag based on trends and keyword | Merge                          | Loop Over Items2                | generate hashtag related to topic we gave based on current trends                            |
| Mistral Cloud Chat Model1 | LangChain Language Model (Mistral)    | AI model generating hashtag suggestions       | hashtag generator (ai_languageModel) | hashtag generator (ai_languageModel) |                                                                                              |
| Structured Output Parser1 | LangChain Output Parser (Structured)   | Parses AI output into structured JSON          | hashtag generator (ai_outputParser) | hashtag generator (ai_outputParser) |                                                                                              |
| Loop Over Items2        | Split In Batches                      | Processes generated hashtags in batches       | hashtag generator               | Google Sheets2                 |                                                                                              |
| Google Sheets2          | Google Sheets                         | Appends generated hashtags to Google Sheets  | Loop Over Items2                | None                          |                                                                                              |
| Sticky Note             | Sticky Note                          | Notes on extraction step                      | None                           | None                          | extract twitter and youtube trends from https://trends24.in/  by   using crawl and scrape community node scrape the websites |
| Sticky Note1            | Sticky Note                          | Notes on filtering step                        | None                           | None                          | filters hashtags out of scraper output                                                       |
| Sticky Note2            | Sticky Note                          | Notes on AI hashtag generation step            | None                           | None                          | generate hashtag related to topic we gave based on current trends                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Scheduled Trigger node:**
   - Type: Schedule Trigger
   - Set it to trigger daily with default interval settings.

2. **Add two Crawlee Nodes for scraping:**
   - Node 1: Name it `extract twitter trends`
     - URL: `https://trends24.in/`
     - Operation: `extractHtml`
   - Node 2: Name it `extract YouTube trends`
     - URL: `https://youtube.trends24.in/`
     - Operation: `extractHtml`
   - Connect `daily trigger` output to both these nodes.

3. **Add HTML Extract nodes to parse scraped HTML:**
   - Node 1: `filter twitter trends`
     - Operation: `extractHtmlContent`
     - Data Property Name: `data.html`
     - Extraction Values: CSS selector `.trend-card__list li a`, return array as HTML.
     - Enable text cleanup.
     - Connect `extract twitter trends` output to this node.
   - Node 2: `filter YouTube trends`
     - Same operation as above.
     - CSS selector `.keywords-list li`
     - Connect `extract YouTube trends` output to this node.

4. **Add a Code node to limit Twitter trends to top 100:**
   - Node: `get only top 100 trends`
   - Language: JavaScript
   - Code:
     ```javascript
     return items.map(item => {
       item.json.trends = item.json.trends.slice(0, 100);
       return item;
     });
     ```
   - Connect `filter twitter trends` output to this node.

5. **Add a Merge node to combine Twitter and YouTube trends:**
   - Node: `Merge`
   - Connect output of `get only top 100 trends` to input 0.
   - Connect output of `filter YouTube trends` to input 1.

6. **Add LangChain Agent node to generate hashtags:**
   - Node: `hashtag generator`
   - Set prompt text to include merged trends and a keyword placeholder (`{{ $json.keywords }}`).
   - System message instructs to generate one short and catchy hashtag, unique and related.
   - Configure AI language model and output parser references here (see next steps).
   - Connect output of `Merge` node to `hashtag generator`.

7. **Add Mistral Cloud Chat Model node:**
   - Node: `Mistral Cloud Chat Model1`
   - Set model to `mistral-small-latest`.
   - Provide Mistral Cloud API credentials.
   - Connect as AI model input to `hashtag generator`.

8. **Add Structured Output Parser node:**
   - Node: `Structured Output Parser1`
   - Configure with JSON schema expecting a key `hashtag` with an array of strings.
   - Connect as AI output parser input to `hashtag generator`.

9. **Add Split In Batches node to process hashtags:**
   - Node: `Loop Over Items2`
   - Default split settings (no reset).
   - Connect `hashtag generator` output to this node.

10. **Add Google Sheets node to store hashtags:**
    - Node: `Google Sheets2`
    - Operation: Append rows
    - Document ID: `1A9EHzMz88dlmBZm1ooQasWb4sU3OTv2tuMtuu1yBAyw` (replace with your sheet ID)
    - Sheet name: `Sheet1`
    - Map column named "hastag " to `output.hashtag` from AI output.
    - Use Google Sheets OAuth2 credentials.
    - Connect output of `Loop Over Items2` node to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                           |
|-----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| The workflow uses community Crawlee Nodes for scraping trend data from `trends24.in` and `youtube.trends24.in`. | Scraping community nodes documentation: https://docs.n8n.io/integrations/community-nodes/crawlee/ |
| Mistral Cloud requires API credentials and access to `mistral-small-latest` model.                               | https://mistral.ai/                                                       |
| Google Sheets node requires OAuth2 credentials with proper spreadsheet access permissions.                       | https://docs.n8n.io/integrations/builtin/google-sheets/                   |
| Hashtag generation prompt enforces lowercase, no spaces, no special characters beyond `#`, and uniqueness.      | Important for social media branding and content safety                    |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created in n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.