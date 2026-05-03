Social Media Sentiment Analysis Dashboard with Custom AI for Twitter, Reddit & LinkedIn

https://n8nworkflows.xyz/workflows/social-media-sentiment-analysis-dashboard-with-custom-ai-for-twitter--reddit---linkedin-6430


# Social Media Sentiment Analysis Dashboard with Custom AI for Twitter, Reddit & LinkedIn

---

### 1. Workflow Overview

The **Social Media Sentiment Dashboard** workflow automates comprehensive brand monitoring and sentiment analysis across Twitter, Reddit, and LinkedIn. It is designed for brand managers, PR teams, and crisis response units to track real-time social media mentions, assess sentiment, identify influencers, and trigger alerts for critical issues or positive engagement opportunities.

The workflow is logically divided into four main blocks:

- **1.1 Social Media Monitoring Triggers**: Initiates data collection via scheduled automation and manual webhook calls.
- **1.2 Multi-Platform Social Scraping**: Extracts structured social media posts and mentions from Twitter, Reddit, and LinkedIn using AI-powered scraping.
- **1.3 Advanced Sentiment Analysis & Brand Intelligence**: Processes scraped posts with advanced sentiment scoring, brand mention detection, influencer tier classification, and priority scoring.
- **1.4 Smart Alerts & Dashboard Update**: Routes analyzed data to a Google Sheets dashboard, filters for crisis or priority alerts, and sends formatted notifications to Slack channels.

This modular structure ensures extensibility, real-time monitoring, and actionable insights for brand reputation management and crisis mitigation.

---

### 2. Block-by-Block Analysis

#### 2.1 Social Media Monitoring Triggers

**Overview:**  
This block sets up the workflow initiation through two parallel triggers: a scheduled trigger for automated periodic monitoring every 4 hours, and a webhook trigger for manual, on-demand sentiment checks.

**Nodes Involved:**  
- Social Media Monitor Trigger  
- Manual Sentiment Check Webhook  
- Sticky Note - Triggers

**Node Details:**

- **Social Media Monitor Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow every 4 hours to ensure continuous social media monitoring.  
  - Configuration: Interval set to 4 hours.  
  - Inputs: None (start node)  
  - Outputs: Connects to the Twitter, Reddit, and LinkedIn scraper nodes.  
  - Edge Cases: Scheduler downtime or misconfiguration might delay monitoring.  
  - Notes: Ensures no important mentions are missed via regular polling.

- **Manual Sentiment Check Webhook**  
  - Type: Webhook  
  - Role: Allows external systems or users to manually trigger sentiment analysis on demand, useful for crisis situations or immediate checks.  
  - Configuration: HTTP GET method, path `/sentiment-webhook`.  
  - Inputs: HTTP request.  
  - Outputs: Connects to Twitter, Reddit, and LinkedIn scraper nodes.  
  - Edge Cases: Unauthorized calls if webhook security not configured; no response body option set to false to return data.  
  - Notes: Provides flexibility for manual intervention.

- **Sticky Note - Triggers**  
  - Type: Sticky Note  
  - Role: Documentation and explanation of trigger mechanisms, usage, and benefits.  
  - Content Summary: Explains scheduled and webhook triggers, their purpose, frequency, and how they support real-time monitoring and crisis management.

---

#### 2.2 Multi-Platform Social Scraping

**Overview:**  
This block performs parallel AI-powered data extraction from three major social platforms: Twitter, Reddit, and LinkedIn, focusing on brand mentions and relevant social discussions using tailored prompt schemas for each platform.

**Nodes Involved:**  
- Twitter Brand Mentions Scraper  
- Reddit Brand Discussion Scraper  
- LinkedIn Professional Mentions Scraper  
- Sticky Note - Social Scraping

**Node Details:**

- **Twitter Brand Mentions Scraper**  
  - Type: ScrapeGraphAI Scraper Node  
  - Role: Scrapes live Twitter search results for posts mentioning the brand or company, extracting structured data per a defined schema including author, content, engagement metrics, and metadata.  
  - Configuration: Uses a custom user prompt requesting post extraction with brand mentions, with a Twitter search URL targeting brand keywords.  
  - Inputs: Trigger outputs (Schedule Trigger or Webhook).  
  - Outputs: To Advanced Sentiment Analysis node.  
  - Credentials: Requires ScrapeGraphAI API credentials.  
  - Edge Cases: API limits, rate limiting, changes in Twitter page structure, or failed scraping.  
  - Notes: Focus on real-time brand mentions and engagement.

- **Reddit Brand Discussion Scraper**  
  - Type: ScrapeGraphAI Scraper Node  
  - Role: Scrapes Reddit comments and posts mentioning the brand, structured extraction for posts, comments, subreddits, and engagement metrics.  
  - Configuration: Custom user prompt with schema for Reddit posts/comments, targeting Reddit search URL with brand keywords and sorted by new.  
  - Inputs: Trigger outputs.  
  - Outputs: To Advanced Sentiment Analysis node.  
  - Credentials: Requires ScrapeGraphAI API credentials.  
  - Edge Cases: Reddit API or scraping restrictions; rate limits; potential missing data due to new posts.  
  - Notes: Captures community discussions and reviews.

- **LinkedIn Professional Mentions Scraper**  
  - Type: ScrapeGraphAI Scraper Node  
  - Role: Extracts LinkedIn posts and professional discussions about the brand, focusing on business content and professional opinions with a specified schema.  
  - Configuration: Custom prompt with schema for LinkedIn posts including author titles, companies, likes, comments, shares, and industry tags.  
  - Inputs: Trigger outputs.  
  - Outputs: To Advanced Sentiment Analysis node.  
  - Credentials: Requires ScrapeGraphAI API credentials.  
  - Edge Cases: LinkedIn's dynamic content and privacy restrictions may affect scraping accuracy; possible API rate limits.  
  - Notes: Targets B2B and professional brand mentions.

- **Sticky Note - Social Scraping**  
  - Type: Sticky Note  
  - Role: Documentation explaining the multi-platform scraping strategy, AI-powered extraction, extensibility, and platform-specific optimization.  
  - Content Summary: Highlights AI-driven parsing, structured data extraction, and potential for adding other platforms.

---

#### 2.3 Advanced Sentiment Analysis & Brand Intelligence

**Overview:**  
This core block processes raw social media data with sophisticated sentiment analysis and brand intelligence algorithms, including multi-level sentiment scoring, brand and competitor mention detection, influencer classification, engagement calculation, and priority scoring for each post.

**Nodes Involved:**  
- Advanced Sentiment Analysis & Brand Intelligence (Code Node)  
- Sticky Note - Sentiment Analysis

**Node Details:**

- **Advanced Sentiment Analysis & Brand Intelligence**  
  - Type: Code Node (JavaScript)  
  - Role: Implements advanced logic to analyze each post from Twitter, Reddit, and LinkedIn for sentiment, brand mentions, influencer tier, engagement, priority, and crisis potential.  
  - Configuration:  
    - Uses embedded JavaScript code incorporating:  
      - Sentiment keyword dictionaries with strength levels (strong, moderate, mild).  
      - Emotional context markers (excitement, sarcasm, disappointment, questions).  
      - Brand and competitor keyword detection with context snippet extraction.  
      - Influencer tier classification based on followers or karma per platform.  
      - Engagement rate calculation tailored per platform engagement metrics.  
      - Priority scoring algorithm combining sentiment, brand mentions, influencer tier, and engagement rate to assign priority levels (Critical, High, Medium, Low).  
      - Flags for requiring response and crisis potential detection.  
  - Inputs: Scraped data from Twitter, Reddit, LinkedIn scraper nodes.  
  - Outputs:  
    - To Google Sheets Sentiment Dashboard (for data storage)  
    - To Crisis & Priority Alert Filter  
    - To Positive Sentiment Filter  
  - Edge Cases:  
    - Unexpected data formats or missing fields could cause errors.  
    - Sentiment misclassification due to slang or sarcasm not captured.  
    - Performance and execution time due to complex processing.  
  - Notes: Central processing node integrating multi-source data into actionable intelligence.

- **Sticky Note - Sentiment Analysis**  
  - Type: Sticky Note  
  - Role: Documentation of the advanced sentiment analysis approach: multi-level sentiment detection, confidence scoring, emotional context, brand intelligence, influencer classification, and crisis detection.  
  - Content Summary: Details AI-powered sentiment detection, brand and competitor tracking, influencer tiering, and engagement scoring.

---

#### 2.4 Smart Alerts & Dashboard Update

**Overview:**  
This block routes processed data to a live Google Sheets dashboard and filters posts to send targeted Slack alerts based on crisis potential, priority, or positive sentiment, enabling real-time notifications and stakeholder reporting.

**Nodes Involved:**  
- Google Sheets Sentiment Dashboard  
- Crisis & Priority Alert Filter  
- Slack Crisis & Priority Alert  
- Positive Sentiment Filter  
- Slack Positive Mention Alert  
- Sticky Note - Dashboard & Alerts

**Node Details:**

- **Google Sheets Sentiment Dashboard**  
  - Type: Google Sheets Node  
  - Role: Appends or updates sentiment analysis results into a Google Sheets document for live dashboard visualization and historical tracking.  
  - Configuration:  
    - Document and sheet specified by IDs (placeholders masked).  
    - Auto-mapping input data to columns like post_id, platform, author, sentiment, confidence, mentions, priority, engagement, influencer tier, crisis potential, timestamp, and post URL.  
    - Operation: Append or update by matching on post_id.  
    - Authentication: Service Account OAuth2.  
  - Inputs: From Advanced Sentiment Analysis node.  
  - Outputs: None.  
  - Edge Cases:  
    - Google API quota limits.  
    - Authentication failures.  
    - Data mismatches or schema changes causing failed inserts.

- **Crisis & Priority Alert Filter**  
  - Type: If Node  
  - Role: Filters posts flagged as crisis potential, critical priority, or negative brand mentions to trigger urgent alerts.  
  - Configuration:  
    - Conditions combined with “any” (logical OR):  
      - `crisis_potential == true`  
      - `priority_level == "Critical"`  
      - Brand mention with negative or very negative sentiment  
  - Inputs: From Advanced Sentiment Analysis node.  
  - Outputs:  
    - True branch: To Slack Crisis & Priority Alert node  
    - False branch: No output.  
  - Edge Cases: Incorrect flagging or missing data could cause missed alerts.

- **Slack Crisis & Priority Alert**  
  - Type: Slack Node  
  - Role: Sends formatted, rich Slack messages to a dedicated crisis alert channel with detailed post info and urgency indicators.  
  - Configuration:  
    - Message template includes sentiment, author, influencer tier, content preview, engagement metrics, brand intelligence, and priority.  
    - Channel identified by name (masked).  
    - OAuth2 authentication.  
    - Link previews enabled.  
  - Inputs: From Crisis & Priority Alert Filter (true branch).  
  - Outputs: None.  
  - Edge Cases: Slack API rate limits, authentication expiration, or channel misconfiguration.

- **Positive Sentiment Filter**  
  - Type: If Node  
  - Role: Filters posts with positive or very positive sentiment to trigger engagement notifications.  
  - Configuration:  
    - Conditions combined with “any”: sentiment equals “positive” or “very_positive”.  
  - Inputs: From Advanced Sentiment Analysis node.  
  - Outputs:  
    - True branch: To Slack Positive Mention Alert node  
    - False branch: No output.  
  - Edge Cases: Sentiment misclassification may affect notifications.

- **Slack Positive Mention Alert**  
  - Type: Slack Node  
  - Role: Sends Slack notifications highlighting positive brand mentions for engagement opportunities.  
  - Configuration:  
    - Message template includes author, platform, sentiment level, content excerpt, engagement, influencer tier, and link.  
    - Channel identified by name (masked).  
    - OAuth2 authentication.  
    - Link previews enabled.  
  - Inputs: From Positive Sentiment Filter (true branch).  
  - Outputs: None.  
  - Edge Cases: Same as Slack Crisis alert node.

- **Sticky Note - Dashboard & Alerts**  
  - Type: Sticky Note  
  - Role: Documentation explaining the alerting system, priority filtering, crisis management, notifications, and Google Sheets dashboard features.  
  - Content Summary: Covers real-time notifications, historical data tracking, visualization, and dual alert channels for crisis and positive engagement.

---

### 3. Summary Table

| Node Name                         | Node Type                       | Functional Role                          | Input Node(s)                          | Output Node(s)                                | Sticky Note                                                                                           |
|----------------------------------|--------------------------------|----------------------------------------|--------------------------------------|-----------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Social Media Monitor Trigger      | Schedule Trigger                | Automated periodic workflow trigger    | None                                 | Twitter Brand Mentions Scraper, Reddit Brand Discussion Scraper, LinkedIn Professional Mentions Scraper | Sticky Note - Triggers: Explains automated scheduling for continuous monitoring                      |
| Manual Sentiment Check Webhook    | Webhook                        | Manual on-demand workflow trigger      | HTTP Requests                       | Twitter Brand Mentions Scraper, Reddit Brand Discussion Scraper, LinkedIn Professional Mentions Scraper | Sticky Note - Triggers: Explains manual webhook for crisis or immediate analysis                      |
| Twitter Brand Mentions Scraper    | ScrapeGraphAI Scraper           | Scrapes Twitter for brand mentions     | Social Media Monitor Trigger, Manual Sentiment Check Webhook | Advanced Sentiment Analysis & Brand Intelligence | Sticky Note - Social Scraping: Details multi-platform AI scraping strategy                            |
| Reddit Brand Discussion Scraper   | ScrapeGraphAI Scraper           | Scrapes Reddit for brand discussions   | Social Media Monitor Trigger, Manual Sentiment Check Webhook | Advanced Sentiment Analysis & Brand Intelligence | Sticky Note - Social Scraping: Details multi-platform AI scraping strategy                            |
| LinkedIn Professional Mentions Scraper | ScrapeGraphAI Scraper       | Scrapes LinkedIn for professional mentions | Social Media Monitor Trigger, Manual Sentiment Check Webhook | Advanced Sentiment Analysis & Brand Intelligence | Sticky Note - Social Scraping: Details multi-platform AI scraping strategy                            |
| Advanced Sentiment Analysis & Brand Intelligence | Code Node                 | Performs sentiment analysis & brand intelligence | Twitter, Reddit, LinkedIn Scrapers  | Google Sheets Sentiment Dashboard, Crisis & Priority Alert Filter, Positive Sentiment Filter           | Sticky Note - Sentiment Analysis: Explains AI-powered sentiment and brand intelligence processing     |
| Google Sheets Sentiment Dashboard | Google Sheets Node              | Stores analyzed data in live dashboard | Advanced Sentiment Analysis & Brand Intelligence | None                                          | Sticky Note - Dashboard & Alerts: Describes dashboard update for live tracking and reporting         |
| Crisis & Priority Alert Filter    | If Node                       | Filters crisis or critical priority alerts | Advanced Sentiment Analysis & Brand Intelligence | Slack Crisis & Priority Alert                  | Sticky Note - Dashboard & Alerts: Explains alert filtering for crisis and priority notifications      |
| Slack Crisis & Priority Alert     | Slack Node                    | Sends crisis/high priority alerts to Slack | Crisis & Priority Alert Filter       | None                                          | Sticky Note - Dashboard & Alerts: Describes real-time slack alerts for crisis management             |
| Positive Sentiment Filter         | If Node                       | Filters positive sentiment mentions    | Advanced Sentiment Analysis & Brand Intelligence | Slack Positive Mention Alert                   | Sticky Note - Dashboard & Alerts: Explains positive sentiment alerting for engagement opportunities  |
| Slack Positive Mention Alert      | Slack Node                    | Sends positive mention alerts to Slack | Positive Sentiment Filter            | None                                          | Sticky Note - Dashboard & Alerts: Describes slack alerts for positive mentions                        |
| Sticky Note - Triggers            | Sticky Note                   | Documentation of trigger mechanisms    | None                                 | None                                          |                                                                                                     |
| Sticky Note - Social Scraping     | Sticky Note                   | Documentation of social scraping       | None                                 | None                                          |                                                                                                     |
| Sticky Note - Sentiment Analysis  | Sticky Note                   | Documentation of sentiment analysis    | None                                 | None                                          |                                                                                                     |
| Sticky Note - Dashboard & Alerts  | Sticky Note                   | Documentation of alerts and dashboard  | None                                 | None                                          |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   1.1. Add a **Schedule Trigger** node named `Social Media Monitor Trigger`.  
   - Set interval: repeat every 4 hours exactly.  
   - No credentials needed.

   1.2. Add a **Webhook** node named `Manual Sentiment Check Webhook`.  
   - HTTP Method: GET  
   - Path: `sentiment-webhook`  
   - Option: `No Response Body` set to false (respond with output)  
   - No authentication configured (consider securing webhook in production).

2. **Create Social Media Scraper Nodes:**

   2.1. Add a **ScrapeGraphAI** node named `Twitter Brand Mentions Scraper`.  
   - Configure user prompt to extract Twitter posts with brand mentions, using the provided JSON schema for Twitter posts.  
   - Set target URL to Twitter search for `"YourBrand" OR "YourCompany"` sorted by live.  
   - Connect credentials for ScrapeGraphAI API.  
   - Connect input from both triggers.

   2.2. Add a **ScrapeGraphAI** node named `Reddit Brand Discussion Scraper`.  
   - Configure user prompt to extract Reddit posts/comments mentioning the brand, using provided Reddit schema.  
   - Set target URL to Reddit search for `"YourBrand"` comments sorted by new.  
   - Connect ScrapeGraphAI credentials.  
   - Connect input from both triggers.

   2.3. Add a **ScrapeGraphAI** node named `LinkedIn Professional Mentions Scraper`.  
   - Configure user prompt to extract LinkedIn posts and discussions about the brand, with LinkedIn schema.  
   - Set target URL to LinkedIn content search for `"YourBrand"`.  
   - Connect ScrapeGraphAI credentials.  
   - Connect input from both triggers.

3. **Create Sentiment Analysis Node:**

   3.1. Add a **Code** node named `Advanced Sentiment Analysis & Brand Intelligence`.  
   - Paste the provided JavaScript code for advanced sentiment processing.  
   - Inputs: connect from the three scraper nodes’ outputs.  
   - Outputs: will connect to Google Sheets and alert filters.

4. **Create Output Nodes:**

   4.1. Add a **Google Sheets** node named `Google Sheets Sentiment Dashboard`.  
   - Configure operation: Append or update rows using `post_id` as matching key.  
   - Map columns according to the sentiment analysis output fields (post_id, platform, author, sentiment, confidence, mentions, priority, engagement, influencer tier, crisis potential, timestamp, post_url).  
   - Set Spreadsheet document ID and sheet name (replace placeholders with your actual Google sheet IDs).  
   - Authenticate using Google Service Account OAuth2.  
   - Connect input from sentiment analysis node.

5. **Create Alert Filtering and Notification Nodes:**

   5.1. Add an **If** node named `Crisis & Priority Alert Filter`.  
   - Conditions:  
     - `crisis_potential` boolean is true  
     - OR `priority_level` equals "Critical"  
     - OR (`mentions_your_brand` is true AND `sentiment` is "negative" or "very_negative")  
   - Connect input from sentiment analysis node.

   5.2. Add a **Slack** node named `Slack Crisis & Priority Alert`.  
   - Configure Slack OAuth2 authentication.  
   - Set channel by name or ID (replace with your Slack crisis alerts channel).  
   - Paste the formatted alert message template including crisis indicators, sentiment, engagement, author, influencer tier, and links.  
   - Connect input from `Crisis & Priority Alert Filter` true branch.

   5.3. Add an **If** node named `Positive Sentiment Filter`.  
   - Condition: sentiment equals "positive" OR "very_positive".  
   - Connect input from sentiment analysis node.

   5.4. Add a **Slack** node named `Slack Positive Mention Alert`.  
   - Configure Slack OAuth2 authentication.  
   - Set channel for positive mentions (replace with your Slack channel).  
   - Paste the positive mention alert message template with author, platform, sentiment, content, engagement, influencer tier, and link.  
   - Connect input from `Positive Sentiment Filter` true branch.

6. **Connect Nodes Accordingly:**

   - Connect both trigger nodes (`Social Media Monitor Trigger` and `Manual Sentiment Check Webhook`) to all three scraper nodes in parallel.  
   - Connect outputs of the three scraper nodes to the `Advanced Sentiment Analysis & Brand Intelligence` node.  
   - Connect `Advanced Sentiment Analysis & Brand Intelligence` output to:  
     - `Google Sheets Sentiment Dashboard`  
     - `Crisis & Priority Alert Filter`  
     - `Positive Sentiment Filter`  
   - Connect filters to respective Slack alert nodes.

7. **Add Documentation Sticky Notes (Optional):**

   - Add sticky notes with content describing each block for clarity and maintenance ease.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Step 1: Social Media Monitoring Triggers - Explains dual trigger system for automated scheduling and manual webhook initiation for comprehensive real-time brand monitoring and crisis management.                                                                                                                                                                                                 | Sticky Note - Triggers node documentation                                                                                                                                        |
| Step 2: Multi-Platform Social Scraping - Describes AI-powered scraping from Twitter, Reddit, and LinkedIn with structured data extraction, extensibility for additional platforms, and custom keyword/hashtag monitoring capabilities.                                                                                                                                                           | Sticky Note - Social Scraping node documentation                                                                                                                                 |
| Step 3: Advanced Sentiment Analysis - Details multi-level sentiment detection, confidence scoring, emotional context, brand and competitor tracking, influencer classification, engagement scoring, and crisis detection logic.                                                                                                                                                                  | Sticky Note - Sentiment Analysis node documentation                                                                                                                              |
| Step 4: Smart Alerts & Dashboard - Covers priority filtering, crisis detection, Slack notifications, Google Sheets dashboard live updates, and dual alert channels for crisis and positive mentions.                                                                                                                                                                                              | Sticky Note - Dashboard & Alerts node documentation                                                                                                                              |
| ScrapeGraphAI API is required for social media scraping nodes; ensure API keys are securely stored and usage limits are monitored.                                                                                                                                                                                                                                                             | ScrapeGraphAI official documentation: https://scrapegraph.com/docs                                                                                                                                                     |
| Google Sheets OAuth2 credentials require appropriate API access with service account permissions for sheet editing.                                                                                                                                                                                                                                                                             | Google Sheets API documentation: https://developers.google.com/sheets/api                                                                                                              |
| Slack OAuth2 credentials must have permission to post messages to designated channels; set up Slack app with required scopes.                                                                                                                                                                                                                                                                   | Slack API documentation: https://api.slack.com/authentication/oauth-v2                                                                                                               |
| Carefully manage rate limits and error handling for external APIs (Twitter, Reddit, LinkedIn scraping, Google Sheets, Slack) to avoid workflow failures or data loss.                                                                                                                                                                                                                              | Best practices for API rate limiting and error retries                                                                                                                             |
| The sentiment analysis code is complex and may require periodic updates to keyword lists and logic to adapt to evolving language and social media trends.                                                                                                                                                                                                                                       | Consider integrating ML-based NLP models in future iterations for improved accuracy                                                                                                  |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.