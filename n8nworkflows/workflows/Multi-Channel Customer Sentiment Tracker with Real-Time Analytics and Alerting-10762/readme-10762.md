Multi-Channel Customer Sentiment Tracker with Real-Time Analytics and Alerting

https://n8nworkflows.xyz/workflows/multi-channel-customer-sentiment-tracker-with-real-time-analytics-and-alerting-10762


# Multi-Channel Customer Sentiment Tracker with Real-Time Analytics and Alerting

### 1. Workflow Overview

This workflow is designed as a **Multi-Channel Customer Sentiment Tracker with Real-Time Analytics and Alerting**. It aggregates customer feedback from diverse digital sources such as social media, emails, support tickets, chat transcripts, and product reviews. The collected feedback undergoes cleaning, normalization, and sentiment analysis enhanced by AI models for nuanced emotion and entity extraction. The workflow computes sentiment trends and detects anomalies to trigger alerts. Finally, it stores processed feedback, syncs insights with CRM and marketing systems, updates dashboards, and generates comprehensive reports to inform decision-making.

Logical blocks in the workflow:

- **1.1 Scheduled Data Ingestion:** Periodic polling of multiple customer feedback sources.
- **1.2 Data Cleaning and Normalization:** Standardizing and sanitizing raw feedback text.
- **1.3 Data Unification:** Mapping disparate data into a consistent schema.
- **1.4 AI-Driven Sentiment and Theme Analysis:** Applying sentiment classification, emotion detection, and entity extraction using OpenAI and Azure OpenAI models.
- **1.5 Data Storage and Integration:** Persisting data in a database and syncing with CRM and marketing automation platforms.
- **1.6 Sentiment Trend Calculation and Aggregation:** Computing rolling averages, detecting spikes, and aggregating sentiment data by entity and time.
- **1.7 Alerting on Anomalies:** Identifying significant negative sentiment spikes or sudden shifts and sending alerts via Slack and email.
- **1.8 Dashboard Update and Reporting:** Updating Google Sheets dashboards and generating detailed insights reports for strategic action.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Ingestion

- **Overview:** This block triggers the workflow every 15 minutes to fetch fresh customer feedback from multiple channels via their respective APIs.
- **Nodes Involved:**  
  - Schedule Trigger - Poll Data Sources  
  - Workflow Configuration  
  - Fetch Social Media Feedback  
  - Fetch Email Feedback  
  - Fetch Support Tickets  
  - Fetch Chat Transcripts  
  - Fetch Product Reviews  
  - Merge All Feedback Sources  
- **Node Details:**

1. **Schedule Trigger - Poll Data Sources**  
   - Type: Schedule Trigger  
   - Role: Initiates the workflow on a fixed 15-minute interval.  
   - Configuration: Interval set to every 15 minutes.  
   - Inputs: None  
   - Outputs: Workflow Configuration node  
   - Edge Cases: Missed triggers if n8n instance is down; ensure system uptime.  

2. **Workflow Configuration**  
   - Type: Set  
   - Role: Holds URLs for all feedback APIs, thresholds for alerts, and integration endpoints.  
   - Configuration: Placeholder values for API URLs and thresholds (e.g., negativeSpikeThreshold=0.3).  
   - Inputs: Schedule Trigger  
   - Outputs: All Fetch nodes  
   - Edge Cases: Missing or invalid URLs can cause HTTP request failures.  

3. **Fetch Social Media Feedback**  
   - Type: HTTP Request  
   - Role: Retrieves social media feedback JSON data from configured API endpoint.  
   - Configuration: URL expression referencing Workflow Configuration.  
   - Inputs: Workflow Configuration  
   - Outputs: Merge All Feedback Sources (input 0)  
   - Edge Cases: Network errors, response format changes, API auth failures.  

4. **Fetch Email Feedback**  
   - Type: HTTP Request  
   - Role: Retrieves email feedback data.  
   - Configuration: URL from Workflow Configuration.  
   - Inputs: Workflow Configuration  
   - Outputs: Merge All Feedback Sources (input 1)  
   - Edge Cases: Same as above.  

5. **Fetch Support Tickets**  
   - Type: HTTP Request  
   - Role: Retrieves support ticket data.  
   - Configuration: URL from Workflow Configuration.  
   - Inputs: Workflow Configuration  
   - Outputs: Merge All Feedback Sources (input 2)  
   - Edge Cases: Same as above.  

6. **Fetch Chat Transcripts**  
   - Type: HTTP Request  
   - Role: Retrieves chat transcript data.  
   - Configuration: URL from Workflow Configuration.  
   - Inputs: Workflow Configuration  
   - Outputs: Merge All Feedback Sources (input 3)  
   - Edge Cases: Same as above.  

7. **Fetch Product Reviews**  
   - Type: HTTP Request  
   - Role: Retrieves product review data.  
   - Configuration: URL from Workflow Configuration.  
   - Inputs: Workflow Configuration  
   - Outputs: Merge All Feedback Sources (input 4)  
   - Edge Cases: Same as above.  

8. **Merge All Feedback Sources**  
   - Type: Merge  
   - Role: Combines all feedback data streams into a single unified dataset for processing.  
   - Configuration: Mode set to “combine,” combining all inputs into one output array.  
   - Inputs: All fetch nodes  
   - Outputs: Clean and Normalize Text Data  
   - Edge Cases: Mismatched data formats; empty sources handled gracefully.

---

#### 1.2 Data Cleaning and Normalization

- **Overview:** Sanitizes and normalizes text inputs to remove HTML tags, special characters, and unify case, ensuring consistent data for analysis.
- **Nodes Involved:**  
  - Clean and Normalize Text Data  
  - Unify Data Schema  
- **Node Details:**

1. **Clean and Normalize Text Data**  
   - Type: Code (JavaScript)  
   - Role: For each feedback item, cleans the text content by removing HTML tags, normalizing Unicode, removing special characters, collapsing whitespace, and converting to lowercase.  
   - Configuration: Custom JS code running once per item.  
   - Inputs: Merged feedback from all sources  
   - Outputs: Unify Data Schema  
   - Edge Cases: Empty or missing text fields handled by fallback empty string; malformed HTML may cause partial cleaning errors.  

2. **Unify Data Schema**  
   - Type: Set  
   - Role: Maps various data fields from heterogeneous sources into a standard schema with fields like feedbackId, feedbackText, source, timestamp, customerId, and metadata (including raw data).  
   - Configuration: Generates unique feedbackId using timestamp and index; extracts primary text and source fields; stores raw JSON in metadata.  
   - Inputs: Clean and Normalize Text Data  
   - Outputs: Sentiment Analysis - Positive/Neutral/Negative  
   - Edge Cases: Missing fields default to empty or ‘unknown’ values; timestamp uses current time if missing.

---

#### 1.3 AI-Driven Sentiment and Theme Analysis

- **Overview:** Applies AI models to classify sentiment, detect emotions, and extract entities from the cleaned feedback text to enrich the dataset with semantic insights.
- **Nodes Involved:**  
  - Sentiment Analysis - Positive/Neutral/Negative  
  - OpenAI - Emotion Detection  
  - OpenAI - Entity Extraction  
  - Combine Analysis Results  
- **Node Details:**

1. **Sentiment Analysis - Positive/Neutral/Negative**  
   - Type: Langchain Sentiment Analysis  
   - Role: Classifies cleaned feedback text into sentiment categories and provides a sentiment score.  
   - Configuration: Input text from unified feedbackText field.  
   - Inputs: Unify Data Schema  
   - Outputs: OpenAI - Emotion Detection (main output)  
   - Edge Cases: Ambiguous or sarcastic text may reduce accuracy.  

2. **OpenAI - Emotion Detection**  
   - Type: OpenAI (Chat Completion)  
   - Role: Detects emotional tones from feedback text for nuanced understanding beyond sentiment polarity.  
   - Configuration: Uses OpenAI API with message operation and OpenAI credentials.  
   - Inputs: Sentiment Analysis node (main output)  
   - Outputs: OpenAI - Entity Extraction  
   - Edge Cases: API rate limits, model errors, or unexpected response formats.  

3. **OpenAI - Entity Extraction**  
   - Type: OpenAI (Chat Completion)  
   - Role: Extracts named entities, topics, or themes mentioned in the feedback to identify focus areas.  
   - Configuration: Same as emotion detection but targeting entities extraction.  
   - Inputs: OpenAI - Emotion Detection  
   - Outputs: Combine Analysis Results  
   - Edge Cases: Similar to above; unrecognized entities may be missed.  

4. **Combine Analysis Results**  
   - Type: Set  
   - Role: Aggregates sentiment, emotions, entities, and sentiment score into a single enriched JSON object with current timestamp.  
   - Configuration: Sets fields for sentiment, emotions, entities, sentimentScore, analysisTimestamp.  
   - Inputs: OpenAI - Entity Extraction  
   - Outputs: Store Feedback in Database, Sync to CRM System, Sync to Marketing Automation  
   - Edge Cases: Missing fields if previous nodes fail; timestamp always current.

---

#### 1.4 Data Storage and Integration

- **Overview:** Stores enriched feedback into a centralized PostgreSQL database and synchronizes key sentiment data with CRM and marketing automation systems.
- **Nodes Involved:**  
  - Store Feedback in Database  
  - Sync to CRM System  
  - Sync to Marketing Automation  
- **Node Details:**

1. **Store Feedback in Database**  
   - Type: Postgres  
   - Role: Inserts or updates customer_feedback table with comprehensive enriched feedback data including sentiment and metadata.  
   - Configuration: Maps JSON fields to DB columns; uses public schema and customer_feedback table.  
   - Inputs: Combine Analysis Results  
   - Outputs: Calculate Sentiment Trends  
   - Edge Cases: DB connection errors, data type mismatches, constraint violations.  

2. **Sync to CRM System**  
   - Type: HTTP Request  
   - Role: Posts sentiment summary and customer ID to the CRM API endpoint for customer profile enrichment.  
   - Configuration: URL and API key from Workflow Configuration; JSON body with customer_id, sentiment, feedback_summary.  
   - Inputs: Combine Analysis Results  
   - Outputs: None  
   - Edge Cases: API authentication errors, payload format errors.  

3. **Sync to Marketing Automation**  
   - Type: HTTP Request  
   - Role: Sends customer sentiment and entities data to marketing system for personalized campaigns.  
   - Configuration: URL and bearer token from Workflow Configuration; JSON body with customer_id, sentiment, entities.  
   - Inputs: Combine Analysis Results  
   - Outputs: None  
   - Edge Cases: Same as CRM sync.

---

#### 1.5 Sentiment Trend Calculation and Aggregation

- **Overview:** Processes stored feedback data to compute sentiment distributions, detect anomalies, and aggregate sentiment metrics by entity and time period.
- **Nodes Involved:**  
  - Calculate Sentiment Trends  
  - Aggregate by Entity and Time Period  
- **Node Details:**

1. **Calculate Sentiment Trends**  
   - Type: Code (JavaScript)  
   - Role: Processes all input items to calculate sentiment distribution counts, rolling averages (7-day window), spike detection, and trend summaries by source and time period.  
   - Configuration: Custom JS code with detailed logic to compute various metrics.  
   - Inputs: Store Feedback in Database (all items)  
   - Outputs: Aggregate by Entity and Time Period  
   - Edge Cases: Empty datasets, malformed timestamp formats, outlier data.  

2. **Aggregate by Entity and Time Period**  
   - Type: Aggregate  
   - Role: Aggregates all item data into a single object field named aggregatedData for downstream analysis.  
   - Configuration: Aggregate all items into one field.  
   - Inputs: Calculate Sentiment Trends  
   - Outputs: Check for Negative Spike, Check for Sudden Sentiment Shift, Update Dashboard - Google Sheets, Generate Insights Report  
   - Edge Cases: Large data volumes may impact performance.

---

#### 1.6 Alerting on Anomalies

- **Overview:** Checks aggregated sentiment data for threshold breaches indicating negative sentiment spikes or sudden shifts and triggers alerts via Slack and email.
- **Nodes Involved:**  
  - Check for Negative Spike  
  - Check for Sudden Sentiment Shift  
  - Send Alert to Slack  
  - Send Alert Email  
- **Node Details:**

1. **Check for Negative Spike**  
   - Type: If  
   - Role: Evaluates if negative sentiment percentage exceeds configured threshold.  
   - Configuration: Left value from aggregated negativeSentimentPercentage; threshold from Workflow Configuration.  
   - Inputs: Aggregate by Entity and Time Period  
   - Outputs: Send Alert to Slack, Send Alert Email (if true)  
   - Edge Cases: Missing or zero values may cause false negatives.  

2. **Check for Sudden Sentiment Shift**  
   - Type: If  
   - Role: Evaluates if sentiment shift metric exceeds configured threshold.  
   - Configuration: Left value from aggregated sentimentShift; threshold from Workflow Configuration.  
   - Inputs: Aggregate by Entity and Time Period  
   - Outputs: Send Alert to Slack, Send Alert Email (if true)  
   - Edge Cases: Same as above.  

3. **Send Alert to Slack**  
   - Type: Slack  
   - Role: Posts alert message to specified Slack channel using OAuth2 authentication.  
   - Configuration: Message includes alert type, entity, sentiment score, time period, and threshold exceeded.  
   - Inputs: Both If nodes (on true)  
   - Outputs: None  
   - Edge Cases: Slack API rate limits, invalid channel ID, OAuth token expiration.  

4. **Send Alert Email**  
   - Type: Email Send  
   - Role: Sends detailed alert email with sentiment metrics and affected entities to configured recipients.  
   - Configuration: HTML formatted content, subject includes alert type, sender and recipient emails configured.  
   - Inputs: Both If nodes (on true)  
   - Outputs: None  
   - Edge Cases: SMTP errors, invalid email addresses.

---

#### 1.7 Dashboard Update and Reporting

- **Overview:** Updates a Google Sheets dashboard with aggregated sentiment data and generates a comprehensive insights report for strategic review.
- **Nodes Involved:**  
  - Update Dashboard - Google Sheets  
  - Generate Insights Report  
- **Node Details:**

1. **Update Dashboard - Google Sheets**  
   - Type: Google Sheets  
   - Role: Appends or updates rows in a specific sheet with sentiment metrics for entities and time periods.  
   - Configuration: Uses OAuth2 credentials; matches rows by entity; writes fields like timestamp, feedback_count, sentiment_score, trend_direction, sentiment_distribution.  
   - Inputs: Aggregate by Entity and Time Period  
   - Outputs: None  
   - Edge Cases: Google API quota limits, OAuth token expiration, sheet permission issues.  

2. **Generate Insights Report**  
   - Type: Code (JavaScript)  
   - Role: Processes all input feedback data to produce an actionable insights report including sentiment distribution, top positive/negative entities, top emotions, trends over time, and recommendations.  
   - Configuration: Custom JS code handling empty data gracefully and compiling structured summary with recommendations based on thresholds.  
   - Inputs: Aggregate by Entity and Time Period  
   - Outputs: None (can be extended to save or send report)  
   - Edge Cases: Empty data input; numerical precision; complex logic correctness.

---

### 3. Summary Table

| Node Name                      | Node Type                      | Functional Role                               | Input Node(s)                                     | Output Node(s)                                      | Sticky Note                                                                                         |
|--------------------------------|--------------------------------|-----------------------------------------------|---------------------------------------------------|-----------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger - Poll Data Sources | Schedule Trigger              | Initiate workflow on scheduled interval       | None                                              | Workflow Configuration                              | Schedule Feedback Cycle: Runs on a set cadence to pull feedback from all channels. Prevents gaps and surfaces issues early. |
| Workflow Configuration          | Set                            | Store API URLs, thresholds, integration info  | Schedule Trigger - Poll Data Sources               | Fetch Social Media Feedback, Fetch Email Feedback, Fetch Support Tickets, Fetch Chat Transcripts, Fetch Product Reviews | Setup Instructions: Connect APIs, configure AI, DB, alerts, dashboards.                             |
| Fetch Social Media Feedback     | HTTP Request                   | Retrieve social media feedback JSON            | Workflow Configuration                             | Merge All Feedback Sources                           | Fetch Social Posts: Collect mentions, reviews, comments across platforms. Captures real-time sentiment. |
| Fetch Email Feedback            | HTTP Request                   | Retrieve email feedback data                    | Workflow Configuration                             | Merge All Feedback Sources                           |                                                                                                   |
| Fetch Support Tickets           | HTTP Request                   | Retrieve support ticket data                    | Workflow Configuration                             | Merge All Feedback Sources                           | Aggregate Support Tickets: Reveals recurring pain points hidden in isolated cases.                 |
| Fetch Chat Transcripts          | HTTP Request                   | Retrieve chat transcript data                   | Workflow Configuration                             | Merge All Feedback Sources                           |                                                                                                   |
| Fetch Product Reviews           | HTTP Request                   | Retrieve product review data                    | Workflow Configuration                             | Merge All Feedback Sources                           | Parse Product Reviews: Public reviews influence perception and expose product gaps.               |
| Merge All Feedback Sources      | Merge                          | Combine all feedback into one data stream       | Fetch Social Media Feedback, Fetch Email Feedback, Fetch Support Tickets, Fetch Chat Transcripts, Fetch Product Reviews | Clean and Normalize Text Data                        |                                                                                                   |
| Clean and Normalize Text Data   | Code                           | Clean and normalize text for analysis           | Merge All Feedback Sources                          | Unify Data Schema                                   |                                                                                                   |
| Unify Data Schema               | Set                            | Map varying data fields into standard schema    | Clean and Normalize Text Data                       | Sentiment Analysis - Positive/Neutral/Negative      |                                                                                                   |
| Sentiment Analysis - Positive/Neutral/Negative | Langchain Sentiment Analysis  | Classify sentiment and score                     | Unify Data Schema                                  | OpenAI - Emotion Detection                          | Analyze Sentiment: Classifies tone with context-aware NLP. Normalized scoring avoids misreading sarcasm or subtle complaints. |
| OpenAI - Emotion Detection      | OpenAI Chat Model              | Detect emotions in feedback text                 | Sentiment Analysis - Positive/Neutral/Negative     | OpenAI - Entity Extraction                          | Extract Themes & Topics Sync: Identifies key subjects turning raw sentiment into actionable themes. |
| OpenAI - Entity Extraction      | OpenAI Chat Model              | Extract entities/themes from text                | OpenAI - Emotion Detection                         | Combine Analysis Results                            |                                                                                                   |
| Combine Analysis Results        | Set                            | Aggregate sentiment, emotions, entities, timestamp | OpenAI - Entity Extraction                         | Store Feedback in Database, Sync to CRM System, Sync to Marketing Automation | Intelligent Feedback Routing: Automatically directs feedback to the right team for critical issues. |
| Store Feedback in Database      | Postgres                      | Persist enriched feedback data                    | Combine Analysis Results                            | Calculate Sentiment Trends                          |                                                                                                   |
| Sync to CRM System              | HTTP Request                   | Update CRM with sentiment data                    | Combine Analysis Results                            | None                                               |                                                                                                   |
| Sync to Marketing Automation    | HTTP Request                   | Update marketing automation with sentiment data  | Combine Analysis Results                            | None                                               |                                                                                                   |
| Calculate Sentiment Trends      | Code                           | Compute sentiment distributions, rolling averages, spikes | Store Feedback in Database                         | Aggregate by Entity and Time Period                  | Identify Sentiment Trends: Tracks mood shifts over time to spot brewing issues.                   |
| Aggregate by Entity and Time Period | Aggregate                    | Aggregate sentiment data by entity/time period   | Calculate Sentiment Trends                          | Check for Negative Spike, Check for Sudden Sentiment Shift, Update Dashboard - Google Sheets, Generate Insights Report |                                                                                                   |
| Check for Negative Spike        | If                             | Detect if negative sentiment spike exceeds threshold | Aggregate by Entity and Time Period                | Send Alert to Slack, Send Alert Email               | Detect Emerging Issues: Flags sudden spikes or new themes as early warnings.                     |
| Check for Sudden Sentiment Shift | If                            | Detect if sentiment shift exceeds threshold       | Aggregate by Entity and Time Period                | Send Alert to Slack, Send Alert Email               |                                                                                                   |
| Send Alert to Slack             | Slack                         | Send alert message to Slack channel                | Check for Negative Spike, Check for Sudden Sentiment Shift | None                                               |                                                                                                   |
| Send Alert Email                | Email Send                    | Send detailed alert email to recipients            | Check for Negative Spike, Check for Sudden Sentiment Shift | None                                               |                                                                                                   |
| Update Dashboard - Google Sheets | Google Sheets                 | Append or update sentiment data on dashboard sheet | Aggregate by Entity and Time Period                | None                                               |                                                                                                   |
| Generate Insights Report        | Code                           | Generate comprehensive insights and recommendations | Aggregate by Entity and Time Period                | None                                               | Insights & Reporting: Centralized feedback intelligence to detect sentiment and support actions.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Name: Schedule Trigger - Poll Data Sources  
   - Type: Schedule Trigger  
   - Set to run every 15 minutes.

2. **Create Set node for Workflow Configuration**  
   - Name: Workflow Configuration  
   - Add string fields for each API endpoint URL: socialMediaApiUrl, emailApiUrl, supportTicketsApiUrl, chatApiUrl, reviewsApiUrl, crmApiUrl, marketingApiUrl.  
   - Add number fields for negativeSpikeThreshold (default 0.3) and sentimentShiftThreshold (default 0.2).  
   - Connect Schedule Trigger output to this node.

3. **Create HTTP Request nodes for each feedback source**  
   - Names: Fetch Social Media Feedback, Fetch Email Feedback, Fetch Support Tickets, Fetch Chat Transcripts, Fetch Product Reviews  
   - Configure each with URL expression referencing respective Workflow Configuration fields.  
   - Set response format to JSON.  
   - Connect Workflow Configuration output to all these nodes.

4. **Create Merge node**  
   - Name: Merge All Feedback Sources  
   - Mode: Combine all inputs  
   - Connect outputs of all Fetch nodes to this Merge node.

5. **Create Code node**  
   - Name: Clean and Normalize Text Data  
   - Set mode to run once per item.  
   - Paste provided JavaScript code to clean text fields (remove HTML, normalize unicode, remove special characters, lowercase).  
   - Connect Merge node output to this node.

6. **Create Set node**  
   - Name: Unify Data Schema  
   - Assign standardized fields: feedbackId (timestamp + index), feedbackText (main text field), source, timestamp (current ISO), customerId, metadata (JSON stringified original data).  
   - Connect Clean and Normalize Text Data output to this node.

7. **Create Langchain Sentiment Analysis node**  
   - Name: Sentiment Analysis - Positive/Neutral/Negative  
   - Set inputText to unified feedbackText field.  
   - Connect Unify Data Schema output.

8. **Create OpenAI Chat Model node for Emotion Detection**  
   - Name: OpenAI - Emotion Detection  
   - Operation: message  
   - Use valid OpenAI credentials.  
   - Connect Sentiment Analysis output.

9. **Create OpenAI Chat Model node for Entity Extraction**  
   - Name: OpenAI - Entity Extraction  
   - Operation: message  
   - Use same OpenAI credentials.  
   - Connect OpenAI - Emotion Detection output.

10. **Create Set node**  
    - Name: Combine Analysis Results  
    - Assign sentiment, emotions, entities, sentimentScore, analysisTimestamp from previous nodes.  
    - Connect OpenAI - Entity Extraction output.

11. **Create Postgres node**  
    - Name: Store Feedback in Database  
    - Configure connection to your PostgreSQL database.  
    - Map fields to columns in customer_feedback table.  
    - Connect Combine Analysis Results output.

12. **Create two HTTP Request nodes for CRM and Marketing sync**  
    - Names: Sync to CRM System, Sync to Marketing Automation  
    - Set method to POST, content-type to application/json.  
    - Set URLs and authorization headers from Workflow Configuration.  
    - Include customer_id, sentiment, and other relevant fields in body parameters.  
    - Connect Combine Analysis Results output.

13. **Create Code node for sentiment trends**  
    - Name: Calculate Sentiment Trends  
    - Paste provided JS code for rolling averages, spike detection, distribution.  
    - Connect Store Feedback in Database output.

14. **Create Aggregate node**  
    - Name: Aggregate by Entity and Time Period  
    - Configure to aggregate all items to a single field aggregatedData.  
    - Connect Calculate Sentiment Trends output.

15. **Create two If nodes**  
    - Names: Check for Negative Spike, Check for Sudden Sentiment Shift  
    - Conditions comparing respective aggregated values against thresholds from Workflow Configuration.  
    - Connect Aggregate by Entity and Time Period output to both.  

16. **Create Slack node**  
    - Name: Send Alert to Slack  
    - Configure OAuth2 Slack credentials and target channel ID.  
    - Compose alert message text with dynamic expressions.  
    - Connect both If nodes’ true outputs to this node.

17. **Create Email Send node**  
    - Name: Send Alert Email  
    - Configure SMTP or email service credentials.  
    - Set recipient, sender, subject, and HTML body with dynamic content.  
    - Connect both If nodes’ true outputs.

18. **Create Google Sheets node**  
    - Name: Update Dashboard - Google Sheets  
    - Connect OAuth2 credentials.  
    - Configure document ID and sheet name ("Sentiment Dashboard").  
    - Set fields to append or update rows keyed by entity.  
    - Connect Aggregate by Entity and Time Period output.

19. **Create Code node**  
    - Name: Generate Insights Report  
    - Paste comprehensive JS code generating sentiment summaries, top entities, emotions, trends, and recommendations.  
    - Connect Aggregate by Entity and Time Period output.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Prerequisites: Social media/survey API credentials; OpenAI API key; database access; CRM credentials; email setup | Sticky Note on workflow prerequisites section                                                  |
| Use Cases: Customer sentiment tracking, product feedback aggregation, support ticket prioritization, brand monitoring, trend identification | Sticky Note on use cases                                                                        |
| Setup Instructions: Connect APIs, configure AI analysis, set up database, enable alerts, link dashboards       | Sticky Note with setup instructions                                                             |
| Workflow Benefits: Reduces analysis time by ~85%, captures actionable insights, enables rapid response          | Sticky Note on customization and benefits                                                      |
| How It Works: Scheduled polling, sentiment classification, AI theme extraction, storage, alerting, dashboards  | Sticky Note on workflow operational overview                                                   |
| Helpful for teams needing real-time multi-source sentiment tracking and alerts to proactively manage brand reputation and customer experience | General context                                                                                 |
| Slack channel ID and email addresses are placeholders that must be configured to valid values for alerts        | Configuration caution                                                                            |
| OpenAI and Azure OpenAI credentials must be valid and have sufficient quota                                   | Integration setup note                                                                         |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow built with n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and public.