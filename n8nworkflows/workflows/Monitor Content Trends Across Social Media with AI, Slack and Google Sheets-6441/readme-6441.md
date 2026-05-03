Monitor Content Trends Across Social Media with AI, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/monitor-content-trends-across-social-media-with-ai--slack-and-google-sheets-6441


# Monitor Content Trends Across Social Media with AI, Slack and Google Sheets

### 1. Workflow Overview

This workflow is designed to **monitor content trends across multiple social media platforms and search engines using AI scraping and analysis, integrate the insights into a Google Sheets content calendar, and notify the content team via Slack**. Its primary use case is to enable marketing, content strategy, and social media teams to stay updated on emerging trends, viral content, and audience engagement patterns. This empowers data-driven content creation and timely publishing aligned with audience interests.

The workflow is logically divided into the following blocks:

- **1.1 Daily Trigger**: Initiates the workflow once a day at a fixed time to ensure regular updates.
- **1.2 Configuration Setup**: Defines the industries, platforms, content types, and generates all necessary search queries for trend monitoring.
- **1.3 Query Splitting and Processing**: Splits the large set of queries into manageable batches and prepares each query for scraping.
- **1.4 AI-Powered Data Collection**: Uses multiple AI scraper nodes to gather trend data from social platforms, Google Trends, BuzzSumo, and Reddit.
- **1.5 Data Merging and Analysis**: Combines the collected data and runs a comprehensive analysis to identify trending topics, content gaps, viral patterns, and actionable recommendations.
- **1.6 Outputs: Data Storage and Notifications**: Updates the Google Sheets content calendar with new content ideas and sends a summarized trend report to the Slack content team channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Trigger

- **Overview:** Triggers the workflow daily at 8 AM for fresh trend data collection.
- **Nodes Involved:**  
  - Daily Trend Monitor Trigger  
  - Sticky Note - Trigger
- **Node Details:**

  - **Daily Trend Monitor Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow every 24 hours at 8 AM local time.  
    - Configuration: Interval set to 24 hours.  
    - Input: None (trigger node).  
    - Output: Starts the next node, Trend Configuration Processor.  
    - Edge cases: Trigger failure due to server downtime or misconfigured timezone settings.  
    - Sticky Note: Explains daily trigger purpose and configuration options.

#### 2.2 Configuration Setup

- **Overview:** Sets up the content strategy parameters, including industries, content types, platforms, posting frequencies, and generates search queries for various trend types.
- **Nodes Involved:**  
  - Trend Configuration Processor  
  - Sticky Note - Config
- **Node Details:**

  - **Trend Configuration Processor**  
    - Type: Code Node (JavaScript)  
    - Role: Creates a comprehensive configuration object with industries, platforms, content calendar info, and generates multiple search queries for trend, keyword, and competitor analysis.  
    - Configuration: Hardcoded configuration for 10 industries, 10 content types, 8 platforms, and generates queries with URLs for scraping.  
    - Expressions: Uses dynamic date ('2025-07-25') and current timestamp for session id.  
    - Input: Trigger node output.  
    - Output: JSON object containing configuration and all queries.  
    - Edge cases: If configuration is manually edited incorrectly, queries may be malformed. The static date could cause stale data if not updated.  
    - Sticky Note: Describes the configuration setup logic and key features.

#### 2.3 Query Splitting and Processing

- **Overview:** Splits the large combined list of queries into batches for efficient processing and picks the current query for scraping.
- **Nodes Involved:**  
  - Split Trend Queries  
  - Query Processor
- **Node Details:**

  - **Split Trend Queries**  
    - Type: Split In Batches  
    - Role: Splits the large queries array into individual batches to process one query at a time.  
    - Configuration: Default batch size (1 by implication).  
    - Input: Output of Trend Configuration Processor.  
    - Output: Single batch with index.  
    - Edge cases: Large query lists could cause performance issues if batch size is too large.

  - **Query Processor**  
    - Type: Code Node  
    - Role: Combines all query types into a single array, selects the current query based on batch index, and sets metadata like query type.  
    - Configuration: Uses input from Split Trend Queries and original config for context.  
    - Input: Batch data and original config data.  
    - Output: JSON with the current selected query and metadata.  
    - Edge cases: Index out of range if batch data is corrupted or empty.

#### 2.4 AI-Powered Data Collection

- **Overview:** Runs four separate AI scraper nodes in parallel to collect detailed trend data from social media, Google Trends, BuzzSumo, and Reddit.
- **Nodes Involved:**  
  - AI Social Trend Scraper  
  - AI Google Trends Scraper  
  - AI Viral Content Analyzer  
  - AI Reddit Insights Scraper  
  - Sticky Note - Scrapers
- **Node Details:**

  - **AI Social Trend Scraper**  
    - Type: ScrapeGraphAI Node (Custom AI Scraper)  
    - Role: Scrapes social media platforms for trending topics, hashtags, engagement, influencers, sentiment, and demographics.  
    - Configuration: Uses a detailed user prompt requesting structured JSON output; URL dynamically taken from current query's search_url.  
    - Input: Query Processor output.  
    - Output: Parsed AI results for social trends.  
    - Edge cases: API rate limits, prompt misinterpretation, invalid URLs.

  - **AI Google Trends Scraper**  
    - Type: ScrapeGraphAI Node  
    - Role: Extracts search volume, related queries, breakout terms, geographic data, seasonality, and content suggestions from Google Trends.  
    - Configuration: Fixed URL to Google Trends explore page; prompt requests structured JSON.  
    - Input: Query Processor output (fixed URL).  
    - Output: Google Trends data.  
    - Edge cases: Changes to Google Trends UI or data structure.

  - **AI Viral Content Analyzer**  
    - Type: ScrapeGraphAI Node  
    - Role: Analyzes viral content and engagement patterns on BuzzSumo for content inspiration.  
    - Configuration: Fixed URL to BuzzSumo trending page; prompt requests viral content patterns, hashtags, optimal posting times.  
    - Input: Query Processor output (fixed URL).  
    - Output: Viral content insights.  
    - Edge cases: Requires valid BuzzSumo access; possible scraping restrictions.

  - **AI Reddit Insights Scraper**  
    - Type: ScrapeGraphAI Node  
    - Role: Analyzes Reddit discussions to find emerging topics, pain points, and content gaps.  
    - Configuration: URL dynamically formed based on current industry query for specific Reddit subreddit hot posts; prompt requests community insights.  
    - Input: Query Processor output.  
    - Output: Reddit discussion analysis.  
    - Edge cases: Subreddit may not exist or be private; Reddit API limits.

  - Sticky Note: Describes the AI scrapers’ roles and extracted data types.

#### 2.5 Data Merging and Analysis

- **Overview:** Merges all AI data streams and runs a comprehensive analysis to synthesize insights, score trends, identify gaps, and generate actionable recommendations.
- **Nodes Involved:**  
  - Merge Trend Data  
  - Trend Analysis Processor  
  - Sticky Note - Analysis
- **Node Details:**

  - **Merge Trend Data**  
    - Type: Merge Node  
    - Role: Combines outputs from the four AI scraper nodes into a single dataset for analysis.  
    - Configuration: Mode set to "combine" to gather all inputs.  
    - Input: Outputs from AI scrapers.  
    - Output: Combined data for analysis.  
    - Edge cases: Missing inputs if any scraper fails; data format inconsistencies.

  - **Trend Analysis Processor**  
    - Type: Code Node  
    - Role: Analyzes merged data, calculates trend momentum scores, identifies content gaps, generates content suggestions, and recommends platforms and priorities.  
    - Configuration: Complex JavaScript logic with multiple helper functions to evaluate engagement, urgency, and suggest content types.  
    - Input: Merged data from AI scrapers and configuration.  
    - Output: Detailed trend analysis including trending topics, viral patterns, community insights, content opportunities, gaps, recommended actions, and overall trend health score.  
    - Edge cases: Parsing errors, missing fields in AI data, division by zero risks if no trends found.  
    - Sticky Note: Explains analysis features and outputs.

#### 2.6 Outputs: Data Storage and Notifications

- **Overview:** Updates the Google Sheets content calendar with new content ideas and sends a Slack notification summarizing the trend report.
- **Nodes Involved:**  
  - Content Calendar Updater  
  - Team Notification Sender  
  - Sticky Note - Outputs
- **Node Details:**

  - **Content Calendar Updater**  
    - Type: Google Sheets Node  
    - Role: Appends a new row to a specified Google Sheets document to log content calendar updates based on analysis results.  
    - Configuration:  
      - Document ID and Sheet Name hardcoded (placeholders used).  
      - Columns mapped with dynamic expressions pulling from trend analysis results (e.g., trend score, content type, hashtags, session id).  
      - Operation: Append row.  
    - Input: Trend Analysis Processor output.  
    - Output: Confirmation of row appended.  
    - Edge cases: Google API quota limits, credential expiration, invalid sheet ID.

  - **Team Notification Sender**  
    - Type: Slack Node  
    - Role: Sends formatted message to Slack channel 'content-team' summarizing daily trend health, top topics, high-priority actions, and a link to the content calendar.  
    - Configuration:  
      - Channel: content-team  
      - Message text: Uses expressions to extract data from Trend Analysis Processor output.  
      - Emoji icon set to :chart_with_upwards_trend:  
    - Input: Trend Analysis Processor output.  
    - Output: Slack message confirmation.  
    - Edge cases: Slack API auth issues, invalid channel, rate limits.  
    - Sticky Note: Describes storage and reporting outputs.

---

### 3. Summary Table

| Node Name                   | Node Type                     | Functional Role                                        | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                   |
|-----------------------------|-------------------------------|-------------------------------------------------------|-------------------------------|-------------------------------|---------------------------------------------------------------|
| Daily Trend Monitor Trigger  | Schedule Trigger              | Triggers daily workflow start                          | None                          | Trend Configuration Processor  | Step 1: Daily Trigger ⏰ - Explains daily trigger purpose     |
| Trend Configuration Processor| Code Node                    | Creates configuration and generates search queries    | Daily Trend Monitor Trigger    | Split Trend Queries            | Step 2: Configuration Setup ⚙️ - Describes config setup      |
| Split Trend Queries          | Split In Batches             | Splits queries into manageable batches                 | Trend Configuration Processor | Query Processor               |                                                               |
| Query Processor             | Code Node                    | Selects current query for scraping                     | Split Trend Queries            | AI Social Trend Scraper, AI Google Trends Scraper, AI Viral Content Analyzer, AI Reddit Insights Scraper |                                                               |
| AI Social Trend Scraper      | ScrapeGraphAI                | Scrapes social media trends and engagement data       | Query Processor               | Merge Trend Data              | Step 3: Data Collection 🤖 - Describes all AI scrapers        |
| AI Google Trends Scraper     | ScrapeGraphAI                | Extracts Google Trends data                             | Query Processor               | Merge Trend Data              | Step 3: Data Collection 🤖                                    |
| AI Viral Content Analyzer    | ScrapeGraphAI                | Analyzes viral content patterns on BuzzSumo           | Query Processor               | Merge Trend Data              | Step 3: Data Collection 🤖                                    |
| AI Reddit Insights Scraper   | ScrapeGraphAI                | Analyzes Reddit discussions for insights               | Query Processor               | Merge Trend Data              | Step 3: Data Collection 🤖                                    |
| Merge Trend Data             | Merge Node                  | Combines all scraped data                               | AI Social Trend Scraper, AI Google Trends Scraper, AI Viral Content Analyzer, AI Reddit Insights Scraper | Trend Analysis Processor     | Step 4: Trend Analysis 📊 - Explains analysis features        |
| Trend Analysis Processor     | Code Node                    | Analyzes merged data; identifies trends and gaps       | Merge Trend Data              | Content Calendar Updater, Team Notification Sender | Step 4: Trend Analysis 📊                                    |
| Content Calendar Updater     | Google Sheets Node           | Appends new content ideas to the content calendar      | Trend Analysis Processor      | None                          | Step 5: Data Storage & Reporting 📈 - Details storage options |
| Team Notification Sender     | Slack Node                  | Sends summarized trend report message to Slack channel | Trend Analysis Processor      | None                          | Step 5: Data Storage & Reporting 📈                           |
| Sticky Note - Trigger        | Sticky Note                  | Explains daily trigger                                  | None                          | None                          | Step 1: Daily Trigger ⏰                                       |
| Sticky Note - Config         | Sticky Note                  | Explains configuration setup                           | None                          | None                          | Step 2: Configuration Setup ⚙️                               |
| Sticky Note - Scrapers       | Sticky Note                  | Explains AI scraping nodes                              | None                          | None                          | Step 3: Data Collection 🤖                                    |
| Sticky Note - Analysis       | Sticky Note                  | Explains trend analysis node                            | None                          | None                          | Step 4: Trend Analysis 📊                                    |
| Sticky Note - Outputs        | Sticky Note                  | Explains outputs and reporting nodes                   | None                          | None                          | Step 5: Data Storage & Reporting 📈                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add the "Daily Trend Monitor Trigger" node:**
   - Type: Schedule Trigger  
   - Set to trigger every 24 hours at 8 AM local time.  
   - No credentials needed.

3. **Add "Trend Configuration Processor" node:**
   - Type: Code Node (JavaScript)  
   - Paste the provided JS code that defines industries, content types, platforms, content calendar, and generates search queries with URLs.  
   - Connect output of Daily Trend Monitor Trigger to this node.

4. **Add "Split Trend Queries" node:**
   - Type: Split In Batches  
   - Connect output of Trend Configuration Processor to this node.  
   - Default batch size (1) is suitable for processing queries one by one.

5. **Add "Query Processor" node:**
   - Type: Code Node  
   - Paste the JS code that extracts the current query based on batch index and merges all query types.  
   - Connect output of Split Trend Queries to this node.

6. **Add four AI Scraper nodes, connect output of Query Processor to each:**

   - **AI Social Trend Scraper**  
     - Type: ScrapeGraphAI Node  
     - Set userPrompt to analyze social platform trends with structured JSON output.  
     - Set websiteUrl to `={{ $json.currentQuery.search_url }}`.  
     - Provide appropriate API credentials if required.

   - **AI Google Trends Scraper**  
     - Type: ScrapeGraphAI Node  
     - Set userPrompt to extract Google Trends data with structured JSON.  
     - Set websiteUrl to `https://trends.google.com/trends/explore?date=now%207-d&geo=US`.  
     - Provide API credentials.

   - **AI Viral Content Analyzer**  
     - Type: ScrapeGraphAI Node  
     - Set userPrompt to analyze viral content on BuzzSumo.  
     - Set websiteUrl to `https://buzzsumo.com/trending-now`.  
     - Provide API credentials.

   - **AI Reddit Insights Scraper**  
     - Type: ScrapeGraphAI Node  
     - Set userPrompt to analyze Reddit discussions.  
     - Set websiteUrl to `={{ 'https://www.reddit.com/r/' + $json.currentQuery.industry.replace(/\s+/g, '') + '/hot/' }}`.  
     - Provide API credentials or ensure public access.

7. **Add "Merge Trend Data" node:**
   - Type: Merge Node  
   - Set mode to "Combine".  
   - Connect outputs of all four AI Scraper nodes to this node.

8. **Add "Trend Analysis Processor" node:**
   - Type: Code Node  
   - Paste the JS code that performs data analysis, scoring, content gap identification, and recommendation generation.  
   - Connect output of Merge Trend Data to this node.

9. **Add "Content Calendar Updater" node:**
   - Type: Google Sheets Node  
   - Operation: Append  
   - Specify Google Sheets document ID and sheet name (e.g., "Content_Calendar_2025").  
   - Map columns dynamically from Trend Analysis Processor output (date, status, platform, priority, session id, trend score, content type, posting time, hashtags, description, expected engagement).  
   - Configure Google Sheets OAuth2 credentials.

10. **Add "Team Notification Sender" node:**
    - Type: Slack Node  
    - Channel: content-team  
    - Message: Compose using expressions referencing Trend Analysis Processor outputs (trend score, top trending topic, high-priority actions, total opportunities).  
    - Set emoji icon to :chart_with_upwards_trend:.  
    - Configure Slack OAuth2 credentials.

11. **Connect Trend Analysis Processor output to both Content Calendar Updater and Team Notification Sender.**

12. **Optionally, add sticky notes to document each step and logic block as per the workflow.**

13. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow tags: "Trend Monitoring", "Content Strategy" highlighting its focus on marketing and trend analysis automation. | For categorization and search in n8n automation environments.                                      |
| Sticky notes provide an excellent stepwise explanation and best practice tips for each workflow section.                  | Use for onboarding new users or team members reviewing the workflow logic.                          |
| External tools used include ScrapeGraphAI for AI scraping and Slack & Google Sheets integrations for reporting.           | Ensure proper API credentials and rate limits are managed for these external services.              |
| The workflow uses a fixed date ("2025-07-25") in configuration code; update this accordingly for live deployments.        | To maintain relevance, modify date handling for dynamic current date in production.                 |
| Slack message contains a link to Google Sheets document for easy access to full reports by the content team.              | Verify link correctness and sharing permissions in Google Sheets to avoid access issues.            |

---

This comprehensive reference document enables understanding, modification, and reliable reproduction of the "Monitor Content Trends Across Social Media with AI, Slack and Google Sheets" workflow without requiring the original JSON export.