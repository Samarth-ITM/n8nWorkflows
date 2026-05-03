Automated Content Marketing Intelligence with OpenAI, Ahrefs & Multi-platform Integration

https://n8nworkflows.xyz/workflows/automated-content-marketing-intelligence-with-openai--ahrefs---multi-platform-integration-5979


# Automated Content Marketing Intelligence with OpenAI, Ahrefs & Multi-platform Integration

### 1. Workflow Overview

This workflow automates advanced content marketing intelligence by integrating multiple data sources—Ahrefs, SEMrush, BuzzSumo, Google Trends, AnswerThePublic, Reddit, and OpenAI—to generate actionable insights for content strategy. It targets digital marketing teams seeking to discover competitor data, trending keywords, audience pain points, and AI-generated content recommendations, and then disseminates results across Airtable, Notion, and Slack.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Configuration Setup:** Starts the workflow daily and loads static configuration parameters.
- **1.2 Data Collection:** Parallel API calls to fetch competitor data, keyword trends, audience insights from various platforms.
- **1.3 Data Processing:** Code nodes to synthesize raw data into structured intelligence on competitors, keywords, and audience.
- **1.4 Data Quality Validation:** Ensures essential competitor data is present before continuing.
- **1.5 AI Analysis:** Prepares a detailed prompt and queries OpenAI to generate strategic content recommendations.
- **1.6 Data Storage & Notification:** Saves results to Airtable and Notion databases, then sends a Slack alert summarizing key findings.
- **1.7 Data Merging:** Combines multiple data outputs to synchronize downstream storage and notifications.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Configuration Setup

- **Overview:** Initiates the workflow on a daily schedule and sets essential configuration parameters such as competitor domains, target regions, seed keywords, and timeframe.
- **Nodes Involved:**  
  - Daily Schedule Trigger  
  - 📋 Configuration Settings

- **Node Details:**

  - **Daily Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Triggers workflow execution once per day automatically.  
    - Configuration: Default daily interval (runs every 24 hours).  
    - Inputs: None  
    - Outputs: To Configuration Settings  
    - Edge cases: Trigger failure or scheduling misconfiguration could prevent execution.

  - **📋 Configuration Settings**  
    - Type: Set Node  
    - Role: Defines static input parameters to be used across the workflow.  
    - Configuration: Sets strings for competitor domains (`opofinance.com,etoro.com`), target regions (`US,UK,DE,FR,JP`), seed keywords (`forex trading,social trade,how to trade`), and timeframe in days (`30`).  
    - Inputs: From Daily Schedule Trigger  
    - Outputs: Feeds into all API data collection nodes  
    - Edge cases: Misconfigured values or empty strings could affect API queries.

#### 1.2 Data Collection

- **Overview:** Parallel HTTP requests retrieve competitor intelligence, keyword trends, questions, and audience insights from various external APIs.
- **Nodes Involved:**  
  - 🔍 Ahrefs Competitor Data  
  - 📊 SEMrush Competitor Keywords  
  - 📈 BuzzSumo Content Performance  
  - 📊 Google Trends Data  
  - ❓ AnswerThePublic Questions  
  - 💬 Reddit Audience Insights

- **Node Details:**

  - **🔍 Ahrefs Competitor Data**  
    - Type: HTTP Request  
    - Role: Fetches competitor domain overview including traffic and backlinks from Ahrefs API.  
    - Configuration: Uses domain from first competitor domain in config, authenticated via HTTP Header Auth with Ahrefs API key. Timeout set to 60 seconds.  
    - Inputs: Configuration Settings  
    - Outputs: To Competitor Data Processing  
    - Edge cases: Timeout, API key invalid, rate limits.

  - **📊 SEMrush Competitor Keywords**  
    - Type: HTTP Request  
    - Role: Retrieves top organic keywords for competitor domain from SEMrush API.  
    - Configuration: Query parameters include API key, competitor domain, limit 50 keywords, specific export columns. Authentication via query parameter.  
    - Inputs: Configuration Settings  
    - Outputs: To Competitor Data Processing  
    - Edge cases: API quota exceeded, invalid domain.

  - **📈 BuzzSumo Content Performance**  
    - Type: HTTP Request  
    - Role: Fetches recent top-performing content based on seed keywords from BuzzSumo API.  
    - Configuration: Uses first seed keyword, filters results published within timeframe days, returns 20 articles, authenticated with HTTP Header Auth.  
    - Inputs: Configuration Settings  
    - Outputs: To Competitor Data Processing  
    - Edge cases: API limits, no results returned.

  - **📊 Google Trends Data**  
    - Type: HTTP Request  
    - Role: Retrieves trending data for seed keywords from Google Trends API.  
    - Configuration: Requests trend data for first seed keyword, geo set to US, time range last 3 months, language English.  
    - Inputs: Configuration Settings  
    - Outputs: To Keyword Trends Processing  
    - Edge cases: API structure changes or rate limiting.

  - **❓ AnswerThePublic Questions**  
    - Type: HTTP Request  
    - Role: Obtains popular user questions related to seed keywords from AnswerThePublic API.  
    - Configuration: Uses first seed keyword, country US, language English, authenticated with HTTP Header Auth.  
    - Inputs: Configuration Settings  
    - Outputs: To Keyword Trends Processing  
    - Edge cases: API downtime, missing data fields.

  - **💬 Reddit Audience Insights**  
    - Type: HTTP Request  
    - Role: Accesses Reddit posts from relevant subreddit matching seed keyword for audience sentiment and pain points.  
    - Configuration: OAuth2 authentication, fetches top 25 posts from the hot category for the month, subreddit named after seed keyword with spaces removed, e.g., "forextrading".  
    - Inputs: Configuration Settings  
    - Outputs: To Audience Insights Processing  
    - Edge cases: OAuth token expiration, subreddit not found.

#### 1.3 Data Processing

- **Overview:** Custom JavaScript nodes synthesize raw API data into structured intelligence on competitors, keyword trends, and audience pain points.
- **Nodes Involved:**  
  - 🔄 Process Competitor Data  
  - 📈 Process Keyword Trends  
  - 👥 Process Audience Insights

- **Node Details:**

  - **🔄 Process Competitor Data**  
    - Type: Code (JavaScript)  
    - Role: Aggregates Ahrefs, SEMrush, BuzzSumo data into a single competitor intelligence object, identifies content gaps, estimates traffic and publishing frequency.  
    - Key expressions: Uses input JSON from multiple sources, applies simple logic to extract top keywords, viral content, and sets static content gaps.  
    - Inputs: Ahrefs, SEMrush, BuzzSumo  
    - Outputs: To Data Quality Check and AI Prompt Preparation  
    - Edge cases: Missing or incomplete data may yield defaults and affect downstream logic.

  - **📈 Process Keyword Trends**  
    - Type: Code (JavaScript)  
    - Role: Combines Google Trends and AnswerThePublic data to identify trending keywords, long-tail questions, seasonal patterns, and content opportunities.  
    - Key expressions: Extracts timeline data from trends, maps questions with placeholder search volume and difficulty, generates sample content opportunities.  
    - Inputs: Google Trends, AnswerThePublic  
    - Outputs: To AI Prompt Preparation  
    - Edge cases: Random placeholder values may reduce accuracy; missing data leads to empty arrays.

  - **👥 Process Audience Insights**  
    - Type: Code (JavaScript)  
    - Role: Analyzes Reddit posts to identify top pain points, common questions, sentiment, engagement topics, and regional preferences.  
    - Key expressions: Filters posts by keywords like "problem" or "issue", calculates engagement scores, adds sample regional preferences.  
    - Inputs: Reddit Audience Insights  
    - Outputs: To AI Prompt Preparation  
    - Edge cases: Subreddit data missing or empty input leads to empty insights.

#### 1.4 Data Quality Validation

- **Overview:** Checks if core competitor data from the processing block is valid before continuing; stops workflow if data is insufficient.
- **Nodes Involved:**  
  - ✅ Data Quality Check  
  - Stop and Error

- **Node Details:**

  - **✅ Data Quality Check**  
    - Type: If Node  
    - Role: Validates that competitor domain data is not "N/A".  
    - Configuration: Condition compares `domain` field from processed competitor data to "N/A".  
    - Inputs: Process Competitor Data  
    - Outputs:  
      - True: Proceeds to save data and AI prompt preparation  
      - False: Routes to Stop and Error Node  
    - Edge cases: Incorrect data fields or node failure could cause false negatives.

  - **Stop and Error**  
    - Type: Stop and Error  
    - Role: Terminates workflow with error if data quality check fails.  
    - Inputs: From Data Quality Check (false path)  
    - Outputs: None

#### 1.5 AI Analysis

- **Overview:** Prepares a detailed prompt combining all processed data and sends it to OpenAI’s Chat Completion API to generate tailored content recommendations.
- **Nodes Involved:**  
  - 📝 Prepare AI Prompt  
  - 🔧 OpenAI HTTP Request Alternative

- **Node Details:**

  - **📝 Prepare AI Prompt**  
    - Type: Set Node  
    - Role: Constructs a multiline prompt string embedding competitor intelligence, keyword opportunities, and audience insights with instructions for JSON-formatted recommendations.  
    - Inputs: Outputs from competitor data, keyword trends, and audience insights processing  
    - Outputs: To OpenAI HTTP Request Alternative  
    - Edge cases: Expression errors if input JSON missing or malformed.

  - **🔧 OpenAI HTTP Request Alternative**  
    - Type: HTTP Request  
    - Role: Sends prompt to OpenAI API endpoint `/v1/chat/completions` using POST method with required headers and authentication.  
    - Configuration: HTTP Header Auth with OpenAI API key; Content-Type `application/json`; body parameters dynamically set from prompt node.  
    - Inputs: AI Prompt  
    - Outputs: To data storage and notification nodes  
    - Edge cases: API rate limits, invalid API key, malformed prompt.

#### 1.6 Data Storage & Notification

- **Overview:** Saves processed intelligence and AI recommendations to Airtable and Notion, then sends a Slack alert with key highlights.
- **Nodes Involved:**  
  - 💾 Save to Airtable - Competitors  
  - 💾 Save to Airtable - Keywords  
  - 📝 Save to Notion  
  - 📢 Send Slack Alert

- **Node Details:**

  - **💾 Save to Airtable - Competitors**  
    - Type: Airtable  
    - Role: Creates a record in the "competitor-intelligence" table with processed competitor data fields (domain, backlinks, traffic, content gaps, publishing frequency, timestamp).  
    - Inputs: Merged data after quality check and AI completion  
    - Outputs: To Data Merging Node  
    - Edge cases: Airtable API token invalid, rate limits, schema mismatch.

  - **💾 Save to Airtable - Keywords**  
    - Type: Airtable  
    - Role: Creates a record in the "keyword-opportunities" table with keyword trends and long-tail questions extracted.  
    - Inputs: AI completion and keyword trends processing output  
    - Outputs: To Data Merging Node  
    - Edge cases: Same as above.

  - **📝 Save to Notion**  
    - Type: Notion  
    - Role: Saves combined insights and AI-generated recommendations into a Notion database (content-research-database).  
    - Configuration: Page ID to be set by user.  
    - Inputs: Merged data from AI and processed nodes  
    - Outputs: To Data Merging Node  
    - Edge cases: Missing page ID, API authorization errors.

  - **📢 Send Slack Alert**  
    - Type: Slack  
    - Role: Sends a formatted alert message to a configured Slack channel summarizing competitor content gaps, trending keywords, and AI recommendations snippet.  
    - Configuration: OAuth2 authentication, channel "content-research-alerts", markdown enabled.  
    - Inputs: Merged data including AI completion, competitor data, and keyword trends  
    - Outputs: None  
    - Edge cases: Slack token expired, channel not found.

#### 1.7 Data Merging

- **Overview:** Combines outputs from competitor data saving and keyword saving nodes to synchronize downstream processes.
- **Nodes Involved:**  
  - 🔗 Merge All Data

- **Node Details:**

  - **🔗 Merge All Data**  
    - Type: Merge  
    - Role: Combines multiple input streams from Airtable saving nodes into one output for Notion and Slack notifications.  
    - Configuration: Multiplex mode (combines all inputs maintaining separate streams).  
    - Inputs: From Airtable competitor and keyword saving nodes  
    - Outputs: To Notion save and Slack alert nodes  
    - Edge cases: Data mismatch, missing inputs.

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                                 | Input Node(s)                                         | Output Node(s)                                             | Sticky Note                                                                                                                                        |
|-------------------------------|-----------------------|------------------------------------------------|-------------------------------------------------------|------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Schedule Trigger         | Schedule Trigger      | Starts workflow daily                           | None                                                  | 📋 Configuration Settings                                  |                                                                                                                                                    |
| 📋 Configuration Settings      | Set                   | Defines static config parameters                | Daily Schedule Trigger                                | All API data collection nodes                              |                                                                                                                                                    |
| 🔍 Ahrefs Competitor Data       | HTTP Request          | Fetches competitor overview from Ahrefs         | 📋 Configuration Settings                            | 🔄 Process Competitor Data                                  |                                                                                                                                                    |
| 📊 SEMrush Competitor Keywords  | HTTP Request          | Retrieves competitor keywords from SEMrush      | 📋 Configuration Settings                            | 🔄 Process Competitor Data                                  |                                                                                                                                                    |
| 📈 BuzzSumo Content Performance | HTTP Request          | Fetches recent viral content                     | 📋 Configuration Settings                            | 🔄 Process Competitor Data                                  |                                                                                                                                                    |
| 📊 Google Trends Data           | HTTP Request          | Retrieves keyword trend data                      | 📋 Configuration Settings                            | 📈 Process Keyword Trends                                  |                                                                                                                                                    |
| ❓ AnswerThePublic Questions    | HTTP Request          | Gets popular questions for seed keywords         | 📋 Configuration Settings                            | 📈 Process Keyword Trends                                  |                                                                                                                                                    |
| 💬 Reddit Audience Insights     | HTTP Request          | Gathers Reddit posts for audience insights       | 📋 Configuration Settings                            | 👥 Process Audience Insights                               |                                                                                                                                                    |
| 🔄 Process Competitor Data      | Code                  | Synthesizes competitor intelligence               | Ahrefs, SEMrush, BuzzSumo                            | ✅ Data Quality Check, 📝 Prepare AI Prompt                |                                                                                                                                                    |
| 📈 Process Keyword Trends       | Code                  | Processes keyword trends and questions            | Google Trends, AnswerThePublic                        | 📝 Prepare AI Prompt                                       |                                                                                                                                                    |
| 👥 Process Audience Insights    | Code                  | Extracts audience pain points and sentiment       | Reddit Audience Insights                             | 📝 Prepare AI Prompt                                       |                                                                                                                                                    |
| ✅ Data Quality Check           | If                    | Validates presence of competitor data             | 🔄 Process Competitor Data                            | 💾 Save to Airtable - Competitors (true), Stop and Error (false) |                                                                                                                                                    |
| Stop and Error                 | Stop and Error         | Stops workflow if data is invalid                  | ✅ Data Quality Check (false path)                    | None                                                      |                                                                                                                                                    |
| 📝 Prepare AI Prompt            | Set                   | Constructs prompt for OpenAI                        | Competitor, Keyword, Audience processing nodes       | 🔧 OpenAI HTTP Request Alternative                         |                                                                                                                                                    |
| 🔧 OpenAI HTTP Request Alternative | HTTP Request          | Sends prompt to OpenAI and receives recommendations | 📝 Prepare AI Prompt                                  | 💾 Save to Airtable - Competitors, 💾 Save to Airtable - Keywords, 📝 Save to Notion, 📢 Send Slack Alert |                                                                                                                                                    |
| 💾 Save to Airtable - Competitors | Airtable               | Stores competitor intelligence                     | ✅ Data Quality Check (true), 🔧 OpenAI HTTP Request Alternative | 🔗 Merge All Data                                         |                                                                                                                                                    |
| 💾 Save to Airtable - Keywords   | Airtable               | Stores keyword opportunities                        | 🔧 OpenAI HTTP Request Alternative                    | 🔗 Merge All Data                                         |                                                                                                                                                    |
| 📝 Save to Notion               | Notion                 | Saves combined data and AI insights                 | 🔧 OpenAI HTTP Request Alternative                    | 🔗 Merge All Data                                         |                                                                                                                                                    |
| 📢 Send Slack Alert            | Slack                  | Sends alert with key insights and AI recommendations | 🔧 OpenAI HTTP Request Alternative                    | None                                                      |                                                                                                                                                    |
| 🔗 Merge All Data              | Merge                  | Combines Airtable save outputs for downstream use | 💾 Save to Airtable - Competitors, 💾 Save to Airtable - Keywords | 📝 Save to Notion, 📢 Send Slack Alert                   |                                                                                                                                                    |
| 📖 Setup Instructions          | Sticky Note             | Provides setup instructions and credential requirements | None                                                  | None                                                      | ## 🧠 Advanced Content Research Automation\n\n### 📋 **Configuration Required:**\n- API Credentials for Ahrefs, SEMrush, BuzzSumo, AnswerThePublic, OpenAI, Reddit, Airtable, Notion, Slack\n- Database setup for Airtable and Notion\n- Slack channel setup\n- Customization guidance\n\n### 🚀 **Workflow Features:**\n- Competitor Intelligence, Keyword Discovery, Audience Extraction, AI Recommendations\n\n### 📊 **Outputs:** Airtable, Notion, Slack\n\n### ⚙️ **Execution:** Daily automated run with retry and validation.\n\n**Ready to activate? Configure your credentials and hit Execute!** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Schedule Trigger` node:**
   - Name: `Daily Schedule Trigger`
   - Set to trigger daily (default interval).
   - Connect its output to next node.

3. **Add a `Set` node:**
   - Name: `📋 Configuration Settings`
   - Add string fields:
     - `competitor_domains` = `"opofinance.com,etoro.com"`
     - `target_regions` = `"US,UK,DE,FR,JP"`
     - `seed_keywords` = `"forex trading,social trade,how to trade"`
     - `timeframe_days` = `"30"`
   - Connect input from `Daily Schedule Trigger`.

4. **Add HTTP Request nodes to fetch data:**

   - **Ahrefs Competitor Data:**
     - Name: `🔍 Ahrefs Competitor Data`
     - Method: GET
     - URL: `https://api.ahrefs.com/v3/site-explorer/overview`
     - Query Parameters:
       - `target`: `={{ $json.competitor_domains.split(',')[0] }}`
       - `mode`: `domain`
       - `output`: `json`
     - Authentication: HTTP Header Auth
       - Credential: Ahrefs API key
     - Set timeout to 60000ms
     - Connect input from `📋 Configuration Settings`.

   - **SEMrush Competitor Keywords:**
     - Name: `📊 SEMrush Competitor Keywords`
     - Method: GET
     - URL: `https://api.semrush.com/`
     - Query Parameters:
       - `type`: `domain_organic`
       - `key`: `={{ $credentials.semrush.api_key }}`
       - `domain`: `={{ $json.competitor_domains.split(',')[0] }}`
       - `display_limit`: `50`
       - `export_columns`: `Ph,Po,Pp,Pd,Nq,Cp,Ur,Tr,Tc,Co,Nr,Td`
     - Authentication: Query Auth with SEMrush API key
     - Connect input from `📋 Configuration Settings`.

   - **BuzzSumo Content Performance:**
     - Name: `📈 BuzzSumo Content Performance`
     - Method: GET
     - URL: `https://api.buzzsumo.com/search/articles.json`
     - Query Parameters:
       - `q`: `={{ $json.seed_keywords.split(',')[0] }}`
       - `num_results`: `20`
       - `published_after`: `={{ $now.minus({ days: parseInt($json.timeframe_days) }).toFormat('yyyy-MM-dd') }}`
     - Authentication: HTTP Header Auth (BuzzSumo API Key)
     - Connect input from `📋 Configuration Settings`.

   - **Google Trends Data:**
     - Name: `📊 Google Trends Data`
     - Method: GET
     - URL: `https://trends.google.com/trends/api/explore`
     - Query Parameters:
       - `hl`: `en-US`
       - `tz`: `360`
       - `req`: `={"comparisonItem":[{"keyword":"{{ $json.seed_keywords.split(',')[0] }}","geo":"US","time":"today 3-m"}],"category":0,"property":""}`
     - Connect input from `📋 Configuration Settings`.

   - **AnswerThePublic Questions:**
     - Name: `❓ AnswerThePublic Questions`
     - Method: GET
     - URL: `https://api.answerthepublic.com/api/v1/questions`
     - Query Parameters:
       - `keyword`: `={{ $json.seed_keywords.split(',')[0] }}`
       - `country`: `us`
       - `language`: `en`
     - Authentication: HTTP Header Auth (AnswerThePublic API Key)
     - Connect input from `📋 Configuration Settings`.

   - **Reddit Audience Insights:**
     - Name: `💬 Reddit Audience Insights`
     - Method: GET
     - URL: `=https://oauth.reddit.com/r/{{ $json.seed_keywords.split(',')[0].replace(' ', '') }}/hot.json`
     - Query Parameters:
       - `limit`: `25`
       - `t`: `month`
     - Authentication: OAuth2 (Reddit)
     - Connect input from `📋 Configuration Settings`.

5. **Add Code nodes for data processing:**

   - **Process Competitor Data:**
     - Name: `🔄 Process Competitor Data`
     - Use JavaScript to combine Ahrefs, SEMrush, BuzzSumo data.
     - Extract and structure key metrics: traffic, backlinks, top keywords, viral content, content gaps.
     - Connect inputs from Ahrefs, SEMrush, BuzzSumo nodes.

   - **Process Keyword Trends:**
     - Name: `📈 Process Keyword Trends`
     - Use JavaScript to process Google Trends and AnswerThePublic data.
     - Extract trending keywords, long-tail questions, content opportunities.
     - Connect inputs from Google Trends and AnswerThePublic nodes.

   - **Process Audience Insights:**
     - Name: `👥 Process Audience Insights`
     - Use JavaScript to analyze Reddit posts.
     - Extract pain points, common questions, engagement, regional preferences.
     - Connect input from Reddit Audience Insights node.

6. **Add an If node for data validation:**

   - Name: `✅ Data Quality Check`
   - Condition: Check that competitor domain from processed data is not "N/A".
   - Connect input from `🔄 Process Competitor Data`.
   - True output: proceed to saving and AI prompt.
   - False output: connect to Stop and Error node.

7. **Add a Stop and Error node:**

   - Name: `Stop and Error`
   - Connect input from `✅ Data Quality Check` false output.

8. **Add a Set node to prepare AI prompt:**

   - Name: `📝 Prepare AI Prompt`
   - Prepare a multiline string embedding JSON data from competitor intelligence, keyword trends, and audience insights.
   - Include instructions requesting JSON-formatted content recommendations.
   - Connect inputs from all three processing nodes (`🔄 Process Competitor Data`, `📈 Process Keyword Trends`, `👥 Process Audience Insights`).

9. **Add HTTP Request node for OpenAI:**

   - Name: `🔧 OpenAI HTTP Request Alternative`
   - Method: POST
   - URL: `https://api.openai.com/v1/chat/completions`
   - Headers: `Content-Type: application/json`
   - Authentication: HTTP Header Auth with OpenAI API key
   - Body: JSON including the prompt from `📝 Prepare AI Prompt`
   - Connect input from `📝 Prepare AI Prompt`.

10. **Add Airtable nodes to save data:**

    - **Save Competitor Data:**
      - Name: `💾 Save to Airtable - Competitors`
      - Base: `content-research-base`
      - Table: `competitor-intelligence`
      - Map fields from `🔄 Process Competitor Data` output.
      - Connect input from `✅ Data Quality Check` true output and from OpenAI node.

    - **Save Keyword Data:**
      - Name: `💾 Save to Airtable - Keywords`
      - Base: `content-research-base`
      - Table: `keyword-opportunities`
      - Map fields from `📈 Process Keyword Trends` output.
      - Connect input from OpenAI node.

11. **Add a Merge node:**

    - Name: `🔗 Merge All Data`
    - Mode: Multiplex
    - Connect inputs from both Airtable save nodes.

12. **Add Notion node to save combined data:**

    - Name: `📝 Save to Notion`
    - Configure with Notion API key and target database/page ID.
    - Connect input from `🔗 Merge All Data`.

13. **Add Slack node to send alert:**

    - Name: `📢 Send Slack Alert`
    - Use OAuth2 credentials
    - Target channel: `content-research-alerts`
    - Message text includes a summary of content gaps, trending keywords, and AI recommendations preview.
    - Connect input from `🔗 Merge All Data`.

14. **Add a Sticky Note with setup instructions:**

    - Include API credentials needed, database/table setup, channel names, and customization tips.

15. **Validate all API credentials and input parameters before execution.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| This workflow requires valid API credentials for Ahrefs, SEMrush, BuzzSumo, AnswerThePublic, OpenAI, Reddit OAuth, Airtable, Notion, and Slack. Proper configuration of Airtable bases and Notion databases is necessary.           | Setup instructions sticky note                  |
| Slack alerts leverage rich markdown formatting to provide immediate actionable insights for marketing teams.                                                                                                                  | Slack node configuration                         |
| The AI prompt is designed to produce JSON-formatted recommendations, facilitating automated parsing or direct usage by AI agents or dashboards.                                                                               | AI prompt preparation node                       |
| The workflow runs daily and includes a data quality check to prevent propagation of incomplete or invalid data, ensuring reliability in operational environments.                                                              | Data Quality Check node                           |
| The workflow demonstrates integration of multiple marketing intelligence platforms and highlights modular design for easy extension or customization with additional data sources or output channels.                            | Overall workflow design                           |

---

**Disclaimer:** The provided text is sourced exclusively from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal, offensive, or protected elements. All data processed is legal and public.