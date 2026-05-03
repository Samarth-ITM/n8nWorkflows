Filter Breaking Geopolitical News with AI Scoring & Telegram Alerts

https://n8nworkflows.xyz/workflows/filter-breaking-geopolitical-news-with-ai-scoring---telegram-alerts-10248


# Filter Breaking Geopolitical News with AI Scoring & Telegram Alerts

---
### 1. Workflow Overview

This workflow, titled **"Filter Breaking Geopolitical News with AI Scoring & Telegram Alerts"**, is designed to monitor multiple international geopolitical news sources, filter relevant breaking news articles using keyword logic, score their urgency via AI, and then send consolidated alerts to a Telegram channel. It targets organizations and individuals who need real-time, cost-efficient monitoring of geopolitical events with minimal noise, focusing on critical alerts only.

The workflow’s logic is grouped into these main blocks:

- **1.1 Scheduling & Configuration Loading**  
  Periodic trigger and retrieval of dynamic configuration data (keywords, scoring rules, thresholds) from Google Drive.

- **1.2 Data Ingestion & Pre-Processing**  
  Fetching news articles from six global RSS feeds, merging them, cleaning and normalizing the data fields, then applying keyword-based filtering.

- **1.3 Deduplication & AI Prompt Preparation**  
  Checking if articles were already analyzed to avoid repetition, followed by generating dynamic AI prompts based on loaded config and article content.

- **1.4 AI-Based Breaking News Scoring**  
  Utilizing an AI agent with OpenAI GPT-4o-mini to analyze and score articles for urgency, using a structured output parser to enforce output consistency.

- **1.5 Post-Processing & Alerting**  
  Recording analyzed articles to a data table, cleaning old records, filtering articles passing the alert threshold, aggregating alerts into a single message, and sending the message to Telegram.

- **1.6 Documentation & Notes**  
  Sticky notes provide key contextual information, usage instructions, and links to documentation and configuration files.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Configuration Loading

- **Overview:**  
  Initiates workflow execution every 30 minutes (configurable) and loads external JSON configuration containing keywords, scoring criteria, thresholds, and Telegram chat IDs from Google Drive.

- **Nodes Involved:**  
  - Every 30 Minutes (Schedule Trigger)  
  - Load Config from Google Drive (HTTP Request)

- **Node Details:**  
  - **Every 30 Minutes**  
    - Type: Schedule Trigger  
    - Configured with a 30-minute interval schedule  
    - Starting point of the workflow  
    - Potential failures: scheduler misconfiguration, time zone issues  
  - **Load Config from Google Drive**  
    - Type: HTTP Request  
    - Fetches a publicly accessible JSON file containing filtering keywords and scoring rules  
    - Outputs JSON parsed automatically for downstream use  
    - Failures: network errors, invalid URL, permission issues

---

#### 1.2 Data Ingestion & Pre-Processing

- **Overview:**  
  Collects news articles from six RSS feeds, merges them, cleans and normalizes articles to essential fields, then applies a dynamic dual-keyword filter based on loaded config.

- **Nodes Involved:**  
  - NYT RSS Feed  
  - TOI RSS Feed  
  - Al Jazeera RSS Feed  
  - BBC RSS Feed  
  - SCMP RSS Feed  
  - NDTV RSS Feed  
  - Merge  
  - RSS_Cleanup_Node (Code)  
  - Dynamic Filter (Code)

- **Node Details:**  
  - **RSS Feed Nodes (NYT, TOI, Al Jazeera, BBC, SCMP, NDTV)**  
    - Type: RSS Feed Read  
    - Each fetches latest articles from a geopolitical news source RSS URL  
    - Outputs raw items with full RSS fields  
    - Failures: feed downtime, malformed XML, network errors  
  - **Merge**  
    - Type: Merge  
    - Combines six input streams into one unified stream  
    - Number of inputs set to 6  
    - Single output with combined items  
  - **RSS_Cleanup_Node**  
    - Type: Code (JavaScript)  
    - Strips HTML tags from content, normalizes fields (link, title, contentSnippet, pubDate, source) for consistent downstream processing  
    - Extracts source from multiple possible RSS fields with fallback to "Unknown"  
    - Handles missing contentSnippet by stripping HTML from content  
    - Failures: code errors if input format unexpected  
  - **Dynamic Filter**  
    - Type: Code (JavaScript)  
    - Implements dual-keyword filter: requires presence of at least one primary and one secondary keyword in article title or snippet  
    - Uses keywords loaded dynamically from the Config node  
    - Filters out non-relevant articles early to reduce AI processing load  
    - Edge cases: empty text fields, case sensitivity handled by lowercase matching

---

#### 1.3 Deduplication & AI Prompt Preparation

- **Overview:**  
  Prevents repeated analysis of the same article by checking against a Data Table, then generates a dynamic system prompt for AI scoring based on config and article content.

- **Nodes Involved:**  
  - Check for Duplicates (Data Table)  
  - Dynamic AI Prompt Generator (Code)

- **Node Details:**  
  - **Check for Duplicates**  
    - Type: Data Table (rowNotExists operation)  
    - Checks if article link exists in "analyzed_articles" data table  
    - Only passes articles not previously analyzed  
    - Edge cases: data table access issues, match failures  
  - **Dynamic AI Prompt Generator**  
    - Type: Code (JavaScript)  
    - Composes a system prompt string dynamically using scoring criteria from config (categories, score ranges, descriptions)  
    - Embeds instructions for AI to score urgency, assign categories, and determine alert flags  
    - Adds systemPrompt and config fields to each item for AI agent consumption  
    - Potential failure: malformed config JSON impacting prompt formatting

---

#### 1.4 AI-Based Breaking News Scoring

- **Overview:**  
  Uses an AI Agent node powered by OpenAI GPT-4o-mini to analyze each article according to the dynamic prompt and outputs a structured JSON with score, category, impact, and alert decision.

- **Nodes Involved:**  
  - Breaking News Analyzer (LangChain Agent)  
  - OpenAI Chat Model (LM)  
  - Structured Output Parser (LangChain Output Parser)

- **Node Details:**  
  - **Breaking News Analyzer**  
    - Type: LangChain Agent node  
    - Receives article data and dynamic system prompt  
    - Sends prompt to OpenAI Chat Model for AI scoring  
    - Uses "define" prompt type with output parser for structured response  
    - Outputs JSON with fields: score, category, alert_topic, alert_description, impact, source, timestamp, link, should_alert  
    - Failures: API rate limits, response parsing errors, model downtime  
  - **OpenAI Chat Model**  
    - Type: LangChain LM Chat OpenAI  
    - Model: gpt-4o-mini (temperature 0.3 for low randomness)  
    - Credentialed with OpenAI API key  
    - Provides AI completions for analysis  
  - **Structured Output Parser**  
    - Type: LangChain Output Parser Structured  
    - Enforces JSON schema on AI output to ensure consistent fields and types  
    - Prevents malformed AI responses from breaking downstream logic

---

#### 1.5 Post-Processing & Alerting

- **Overview:**  
  Stores analyzed article metadata to avoid duplicate future processing, cleans up old entries, filters articles passing alert threshold, aggregates alerts into a single message, and sends the consolidated alert message via Telegram.

- **Nodes Involved:**  
  - Re-attach Config (Code)  
  - If (Conditional)  
  - Record Analyzed Article (Data Table Insert)  
  - Cleanup Old Records (Data Table Delete)  
  - Aggregate Alerts (Code)  
  - Send Breaking News Alert (Telegram)

- **Node Details:**  
  - **Re-attach Config**  
    - Type: Code (JavaScript)  
    - Re-injects config JSON back into article items after AI processing (which removes it)  
    - Ensures alert threshold and Telegram chat ID accessible for filtering and messaging  
  - **If**  
    - Type: Conditional  
    - Checks if `should_alert` is true (score ≥ threshold, typically 6)  
    - Routes true items to alert aggregation; false items are discarded  
  - **Record Analyzed Article**  
    - Type: Data Table Insert  
    - Inserts article metadata (score, title, analyzed_at timestamp, article_link) into "analyzed_articles" data table  
    - Prevents repeated analysis of the same article  
  - **Cleanup Old Records**  
    - Type: Data Table Delete Rows  
    - Deletes records older than 7 days from the data table to keep it manageable  
  - **Aggregate Alerts**  
    - Type: Code (JavaScript)  
    - Consolidates all alert items into a single formatted text message with article details, impact, links, and scores  
    - Prepares message for Telegram dispatch  
  - **Send Breaking News Alert**  
    - Type: Telegram node  
    - Sends the aggregated alert message to configured Telegram chat ID with HTML parse mode enabled  
    - Credentialed with Telegram API token  
    - Failures: Telegram API errors, invalid chat ID, network issues

---

#### 1.6 Documentation & Notes

- **Overview:**  
  Sticky notes provide detailed documentation, usage instructions, configuration links, and architectural explanations for users and maintainers.

- **Nodes Involved:**  
  - Multiple Sticky Note nodes dispersed throughout the workflow.

- **Content Highlights:**  
  - Overview of workflow purpose and AI filtering efficiency  
  - Scheduling details  
  - Configuration file explanations and links  
  - Pre-processing and filtering logic  
  - AI scoring customization and usage  
  - Data table usage for deduplication and cleanup  
  - Alert aggregation and Telegram notification setup  
  - Links to GitHub repo, architecture docs, setup guides, and use case explanations

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                             | Input Node(s)                                     | Output Node(s)                   | Sticky Note                                                                                   |
|---------------------------|------------------------------------|--------------------------------------------|--------------------------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------|
| Every 30 Minutes           | Schedule Trigger                   | Trigger workflow every 30 minutes          | -                                                | Load Config from Google Drive    | ## Scheduling Node<br>currently set at 30 mins interval but change it as per your requirements |
| Load Config from Google Drive | HTTP Request                     | Load dynamic config JSON from Google Drive | Every 30 Minutes                                 | NYT RSS Feed, TOI RSS Feed, Al Jazeera RSS Feed, BBC RSS Feed, SCMP RSS Feed, NDTV RSS Feed | ## [Loading Config Info.](https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/blob/master/ARCHITECTURE.md)<br>[1) Link to different config json files](https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/tree/master/configs)<br>[2) Explanation about different config use cases](https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/blob/master/USE_CASES.md) |
| NYT RSS Feed               | RSS Feed Read                     | Fetch New York Times World news RSS        | Load Config from Google Drive                     | Merge                           | ## Uses 6 RSS News Feeds                                                                        |
| TOI RSS Feed               | RSS Feed Read                     | Fetch Times of India top stories RSS       | Load Config from Google Drive                     | Merge                           | ## Uses 6 RSS News Feeds                                                                        |
| Al Jazeera RSS Feed        | RSS Feed Read                     | Fetch Al Jazeera all news RSS               | Load Config from Google Drive                     | Merge                           | ## Uses 6 RSS News Feeds                                                                        |
| BBC RSS Feed               | RSS Feed Read                     | Fetch BBC World news RSS                     | Load Config from Google Drive                     | Merge                           | ## Uses 6 RSS News Feeds                                                                        |
| SCMP RSS Feed              | RSS Feed Read                     | Fetch South China Morning Post RSS          | Load Config from Google Drive                     | Merge                           | ## Uses 6 RSS News Feeds                                                                        |
| NDTV RSS Feed              | RSS Feed Read                     | Fetch NDTV top stories RSS                   | Load Config from Google Drive                     | Merge                           | ## Uses 6 RSS News Feeds                                                                        |
| Merge                     | Merge                             | Combine RSS feeds into one stream            | NYT RSS Feed, TOI RSS Feed, Al Jazeera RSS Feed, BBC RSS Feed, SCMP RSS Feed, NDTV RSS Feed | RSS_Cleanup_Node               | ## [Pre-Processing Nodes](https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/blob/master/ARCHITECTURE.md)<br>1. Merge Data coming from various RSS Feeds |
| RSS_Cleanup_Node           | Code                              | Clean and normalize RSS item fields          | Merge                                             | Dynamic Filter                 | ## [Pre-Processing Nodes](https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/blob/master/ARCHITECTURE.md)<br>2. Clean up info. not required for processing |
| Dynamic Filter             | Code                              | Filter articles by primary & secondary keywords | RSS_Cleanup_Node                                  | Check for Duplicates           | ## [Pre-Processing Nodes](https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/blob/master/ARCHITECTURE.md)<br>3. Use the Primary and Secondary Filters from the Load Config File |
| Check for Duplicates       | Data Table (rowNotExists)          | Skip articles already analyzed                | Dynamic Filter                                    | Dynamic AI Prompt Generator    | ## [Pre-Processing Nodes](https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/blob/master/ARCHITECTURE.md)<br>4. Check for Duplicate Articles (i.e articles already triggered in previous runs) |
| Dynamic AI Prompt Generator | Code                             | Generate AI system prompt dynamically         | Check for Duplicates                              | Breaking News Analyzer         | ## [Pre-Processing Nodes](https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/blob/master/ARCHITECTURE.md)<br>5. Create the Dynamic Prompt Generator for use in Breaking News Analyzer Node |
| Breaking News Analyzer     | LangChain Agent (AI)               | AI scoring of article urgency & classification | Dynamic AI Prompt Generator + OpenAI Chat Model + Structured Output Parser | Re-attach Config, Record Analyzed Article | ## Breaking News Analyzer<br>1. Used to [Score Articles](https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/blob/master/USE_CASES.md#customizing-scoring-mechanism) for different Alert Levels via the Dynamic System Prompt Generator using Scoring Rules coming from Config Files<br>2. Uses AI Agent with Open AI Chat Model running gpt-4o-mini (with temperature set at 0.3 for consistent results) |
| OpenAI Chat Model          | LangChain LM Chat OpenAI          | AI language model for text analysis           | Breaking News Analyzer (AI Language Model input) | Breaking News Analyzer (AI Output Parser) | ## Breaking News Analyzer<br>2. Uses AI Agent with Open AI Chat Model running gpt-4o-mini (with temperature set at 0.3 for consistent results) |
| Structured Output Parser   | LangChain Output Parser Structured | Enforce and parse structured AI JSON output   | OpenAI Chat Model (AI Output Parser output)      | Breaking News Analyzer (AI Output Parser) | ## Breaking News Analyzer<br>2. Uses AI Agent with Open AI Chat Model running gpt-4o-mini (with temperature set at 0.3 for consistent results) |
| Re-attach Config          | Code                              | Reattach config JSON for threshold and chat ID | Breaking News Analyzer                            | If                            | ## Post-Processing Nodes<br>1. Re-attach Config - To bring the "alert_threshold" (Scoring Threshold) value from the upstream Config Files |
| If                        | Conditional                      | Filter articles where should_alert = true     | Re-attach Config                                  | Aggregate Alerts              | ## Post-Processing Nodes<br>2. If Node - Only pass the Articles >6 score to the True Branch (alert_threshold is set to 6) |
| Record Analyzed Article    | Data Table Insert                | Store analyzed article metadata to avoid duplicates | Breaking News Analyzer                            | Cleanup Old Records           | ## Data Insertion and Clean Up Nodes<br>1. Inserts all articles into the "analyzed_articles" n8n Data Table so that duplicate article checks can be done on next runs |
| Cleanup Old Records        | Data Table Delete Rows          | Delete analyzed articles older than 7 days     | Record Analyzed Article                            | -                            | ## Data Insertion and Clean Up Nodes<br>2. Clean Up Node - removes articles from "analyzed_articles" N8N Data Table for articles which are more than 7 days old |
| Aggregate Alerts           | Code                              | Aggregate alert articles into one Telegram message | If                                                | Send Breaking News Alert     | ## Post-Processing Nodes<br>3. Aggregate Alerts- Aggregates All Alerts passing the threshold |
| Send Breaking News Alert   | Telegram                         | Send consolidated alerts to Telegram chat      | Aggregate Alerts                                  | -                            | ## Post-Processing Nodes<br>4. Telegram Message Node - Send the aggregated messages to Telegram Chat Node using the Telegram Chat Id again referenced from the Config Files |
| Sticky Note                | Sticky Note                     | Documentation and context                      | -                                                | -                            | # Geopolitics Breaking News Alert System (Full description and links to Github repo and docs) |
| Sticky Note1               | Sticky Note                     | Documentation: Scheduling details              | -                                                | -                            | ## Scheduling Node<br>currently set at 30 mins interval but change it as per your requirements |
| Sticky Note2               | Sticky Note                     | Documentation: Config loading info and links    | -                                                | -                            | ## [Loading Config Info.](https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/blob/master/ARCHITECTURE.md) |
| Sticky Note3               | Sticky Note                     | Documentation: Pre-processing node explanation  | -                                                | -                            | ## [Pre-Processing Nodes](https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/blob/master/ARCHITECTURE.md) |
| Sticky Note4               | Sticky Note                     | Documentation: Breaking News Analyzer explanation | -                                                | -                            | ## Breaking News Analyzer<br>1. Scoring mechanism and AI model details |
| Sticky Note5               | Sticky Note                     | Documentation: RSS feeds used                    | -                                                | -                            | ## Uses 6 RSS News Feeds |
| Sticky Note6               | Sticky Note                     | Documentation: Data insertion and cleanup nodes | -                                                | -                            | ## Data Insertion and Clean Up Nodes |
| Sticky Note7               | Sticky Note                     | Documentation: Post-processing nodes and alerting | -                                                | -                            | ## Post-Processing Nodes |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: "Every 30 Minutes"  
   - Type: Schedule Trigger  
   - Configure to run every 30 minutes (adjustable interval)

2. **Create an HTTP Request node to load config**  
   - Name: "Load Config from Google Drive"  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://drive.usercontent.google.com/download?id=15tO_RUnFTDkANVocx0es7k9PZ-VxFLgc&export=download`  
   - Response Format: JSON  
   - Connect "Every 30 Minutes" → "Load Config from Google Drive"

3. **Add six RSS Feed Read nodes**  
   - Names & URLs:  
     - "NYT RSS Feed": https://rss.nytimes.com/services/xml/rss/nyt/World.xml  
     - "TOI RSS Feed": https://timesofindia.indiatimes.com/rssfeedstopstories.cms  
     - "Al Jazeera RSS Feed": https://www.aljazeera.com/xml/rss/all.xml  
     - "BBC RSS Feed": http://feeds.bbci.co.uk/news/world/rss.xml  
     - "SCMP RSS Feed": https://www.scmp.com/rss/3/feed  
     - "NDTV RSS Feed": https://feeds.feedburner.com/ndtvnews-top-stories  
   - Connect "Load Config from Google Drive" output to each RSS Feed node

4. **Create a Merge node**  
   - Name: "Merge"  
   - Type: Merge  
   - Set "Number of Inputs" to 6  
   - Connect all RSS Feed nodes to this Merge node

5. **Create a Code node to clean RSS data**  
   - Name: "RSS_Cleanup_Node"  
   - Type: Code  
   - Paste the provided JavaScript to normalize fields (link, title, contentSnippet, pubDate, source) and strip HTML  
   - Connect "Merge" → "RSS_Cleanup_Node"

6. **Create a Code node for dynamic filtering**  
   - Name: "Dynamic Filter"  
   - Type: Code  
   - Use provided JavaScript that loads primary and secondary keywords from config and filters articles accordingly  
   - Connect "RSS_Cleanup_Node" → "Dynamic Filter"

7. **Create a Data Table node to check duplicates**  
   - Name: "Check for Duplicates"  
   - Type: Data Table  
   - Configure to use "analyzed_articles" data table  
   - Operation: rowNotExists  
   - Condition: `article_link == {{ $json.link }}`  
   - Connect "Dynamic Filter" → "Check for Duplicates"

8. **Create a Code node to generate dynamic AI prompts**  
   - Name: "Dynamic AI Prompt Generator"  
   - Type: Code  
   - Use provided JavaScript to build system prompts with scoring criteria and instructions from config  
   - Connect "Check for Duplicates" → "Dynamic AI Prompt Generator"

9. **Add LangChain OpenAI Chat Model node**  
   - Name: "OpenAI Chat Model"  
   - Type: LangChain LM Chat OpenAI  
   - Model: gpt-4o-mini  
   - Temperature: 0.3  
   - Set OpenAI API credentials (OpenAI API key)  
   - Connect "Dynamic AI Prompt Generator" → LangChain Agent node input

10. **Add LangChain Agent node for breaking news analysis**  
    - Name: "Breaking News Analyzer"  
    - Type: LangChain Agent  
    - Parameters:  
      - Text: Concatenate article fields (title, contentSnippet, link, source, pubDate)  
      - System Message: Use dynamic prompt field (`systemPrompt`) from input JSON  
      - Prompt Type: Define  
      - Output Parser: Enabled  
    - Connect "OpenAI Chat Model" output → "Breaking News Analyzer" AI output parser  
    - Connect "Dynamic AI Prompt Generator" → "Breaking News Analyzer" main input

11. **Add Structured Output Parser node**  
    - Name: "Structured Output Parser"  
    - Type: LangChain Output Parser Structured  
    - Provide JSON schema example as given (score, category, alert_topic, alert_description, impact, source, timestamp, link, should_alert)  
    - Connect "OpenAI Chat Model" AI output → "Structured Output Parser" → "Breaking News Analyzer" AI output parser input

12. **Create a Code node to reattach config**  
    - Name: "Re-attach Config"  
    - Type: Code  
    - Use provided JS to add config JSON back to output items  
    - Connect "Breaking News Analyzer" → "Re-attach Config"

13. **Create an If node to filter alerts**  
    - Name: "If"  
    - Type: Conditional  
    - Condition: `{{ $json.output.should_alert }} === true`  
    - Connect "Re-attach Config" → "If"

14. **Create Data Table insert node for analyzed articles**  
    - Name: "Record Analyzed Article"  
    - Type: Data Table Insert  
    - Target Table: "analyzed_articles"  
    - Insert columns: score, title (alert_topic), analyzed_at (current ISO timestamp), article_link  
    - Connect "Breaking News Analyzer" → "Record Analyzed Article"

15. **Create Data Table delete node for cleanup**  
    - Name: "Cleanup Old Records"  
    - Type: Data Table Delete Rows  
    - Condition: `analyzed_at < now minus 7 days`  
    - Connect "Record Analyzed Article" → "Cleanup Old Records"

16. **Create a Code node to aggregate alerts**  
    - Name: "Aggregate Alerts"  
    - Type: Code  
    - Use provided JS that consolidates all alerts passing threshold into one formatted message string  
    - Connect "If" true output → "Aggregate Alerts"

17. **Add Telegram node to send alerts**  
    - Name: "Send Breaking News Alert"  
    - Type: Telegram  
    - Chat ID: `{{ $json.config.telegram_chat_id }}` (from config)  
    - Text: `=={{ $json.message }}`  
    - Parse mode: HTML  
    - Set Telegram credentials with API token  
    - Connect "Aggregate Alerts" → "Send Breaking News Alert"

18. **Create sticky notes for documentation**  
    - Add sticky note nodes describing each major block and linking to GitHub repo and docs as per content

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Smart filtering + AI scoring reduces AI costs by 80-90%, focusing on critical geopolitical news alerts only.                               | Workflow overview sticky note                                                                                                     |
| Scheduling node runs every 30 minutes by default; configurable to suit user needs.                                                          | Sticky Note1                                                                                                                      |
| External JSON config files loaded from Google Drive contain keywords, scoring rules, alert thresholds, and Telegram chat IDs.               | Sticky Note2 + GitHub config repo: https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/tree/master/configs             |
| Pre-processing includes merging 6 RSS feeds, cleaning, filtering by keywords, and deduplication against a data table.                      | Sticky Note3                                                                                                                      |
| AI scoring uses LangChain Agent with OpenAI GPT-4o-mini, temperature 0.3 for consistent results, dynamic prompts based on scoring criteria. | Sticky Note4 + AI scoring customization: https://github.com/devdutta/n8n-geopolitics-breaking-news-alert/blob/master/USE_CASES.md#customizing-scoring-mechanism |
| Data table "analyzed_articles" stores processed articles to prevent duplicates and cleans records older than 7 days.                       | Sticky Note6                                                                                                                      |
| Alerts are aggregated into a single Telegram message using configured chat ID from the config file.                                         | Sticky Note7                                                                                                                      |
| Full project documentation, architecture, setup guide, and use cases available in the GitHub repository.                                    | https://github.com/devdutta/n8n-geopolitics-breaking-news-alert                                                                   |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.