Forex News & Sentiment Telegram Alerts

https://n8nworkflows.xyz/workflows/forex-news---sentiment-telegram-alerts-4659


# Forex News & Sentiment Telegram Alerts

### 1. Workflow Overview

This workflow, titled **"Forex News & Sentiment Telegram Alerts"**, is designed to aggregate, analyze, and deliver forex market news and sentiment updates automatically via Telegram. It targets forex traders and analysts who want consolidated, AI-enhanced insights from multiple financial news RSS feeds. The workflow consists of the following logical blocks:

- **1.1 Scheduled News Aggregation:** Periodically fetches forex-related news items from a variety of RSS feeds.
- **1.2 Data Merging:** Combines the multiple incoming RSS feed data streams into a unified dataset.
- **1.3 Filtering and Preprocessing:** Applies filtering logic and custom code processing to prepare data for analysis.
- **1.4 AI-Based Sentiment Analysis:** Uses Google Gemini Chat and a Langchain agent to analyze news sentiment, focusing on EURUSD currency pair.
- **1.5 Telegram Notification Delivery:** Sends the analyzed news and sentiment alerts to Telegram subscribers.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled News Aggregation

- **Overview:**  
  This block triggers the workflow on a schedule and reads multiple forex news RSS feeds from different providers.

- **Nodes Involved:**  
  - Schedule Trigger  
  - RSS Read - GOOGLE  
  - RSS Read - fxstreet news  
  - RSS Read - FXSTREET  
  - RSS Read - DAILYFX technical  
  - RSS Read - DAILYFX fundamental  
  - RSS Read - DAILYFX forexnews  
  - RSS Read - DAILYFX forex articles  
  - RSS Read - FOREX LIVE technicals  
  - RSS Read - FOREX LIVE central banks  
  - RSS Read - FOREX LIVE forexorders  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow based on a predefined cron-like schedule (set for June 4, 2025, 00:00:22, Europe/London timezone).  
    - Config: Runs once daily at midnight with timezone awareness.  
    - Input/Output: No input; outputs to all RSS Read nodes simultaneously.  
    - Edge cases: If n8n instance is down at scheduled time, execution may be missed; timezone misconfiguration could cause timing issues.

  - **RSS Read Nodes (all 10 nodes)**  
    - Type: RSS Feed Read  
    - Role: Each reads news items from a specific RSS feed URL related to forex news and analysis.  
    - Config: Each node is configured with different RSS URLs (not explicitly shown in JSON, assumed set).  
    - Input: Triggered by Schedule Trigger.  
    - Output: Emits an array of news items from respective RSS feeds.  
    - Edge cases: Feed unavailability, malformed RSS data, network timeouts, or rate limiting by feed providers.

#### 2.2 Data Merging

- **Overview:**  
  Consolidates all RSS feed outputs into a single stream for unified processing.

- **Nodes Involved:**  
  - Merge

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines multiple inputs from all RSS Read nodes into one output stream.  
    - Config: Uses default merge mode (assumed to be “Merge By Index” or “Merge By Concatenation”).  
    - Input: Receives input from all RSS Read nodes.  
    - Output: Single combined array of news items.  
    - Edge cases: Mismatched data lengths from feeds might cause empty merges or missing data if merge mode is index-based.

#### 2.3 Filtering and Preprocessing

- **Overview:**  
  Applies a filter to refine news items and runs custom JavaScript code for additional processing before AI analysis.

- **Nodes Involved:**  
  - Filter  
  - Code

- **Node Details:**

  - **Filter**  
    - Type: Filter  
    - Role: Applies conditional logic to select relevant news items from the merged dataset (exact filter criteria not specified).  
    - Input: Receives combined news from Merge node.  
    - Output: Passes filtered news to Code node.  
    - Edge cases: Incorrect filter conditions could exclude all data or pass irrelevant data; expression errors can occur if expected data fields are missing.

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Executes custom JavaScript logic for data transformation or enrichment on filtered news.  
    - Input: Incoming filtered news items.  
    - Output: Processed news data ready for AI analysis.  
    - Edge cases: Syntax or runtime errors in code can halt workflow; improper handling of empty or malformed input data.

#### 2.4 AI-Based Sentiment Analysis

- **Overview:**  
  Performs advanced sentiment and news analysis on the processed news data, focusing on EURUSD market impact using AI models.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - EURUSD News and Sentiment Analyst (Langchain agent)

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: Langchain Google Gemini Chat Model  
    - Role: Provides an AI language model backend for generating text-based insights from news data.  
    - Input: Acts as language model for the Langchain agent node.  
    - Output: AI-generated responses fed into the agent.  
    - Credential: Requires Google Gemini API credentials.  
    - Edge cases: API rate limits, authentication failures, or model timeouts.

  - **EURUSD News and Sentiment Analyst**  
    - Type: Langchain Agent  
    - Role: Implements a chain of prompts and logic to analyze EURUSD forex news sentiment using the AI model.  
    - Input: Receives processed news and AI model output.  
    - Output: Generates sentiment analysis summaries and actionable insights.  
    - Edge cases: Model misinterpretation, incomplete data, or prompt failures.  
    - Output connected to Telegram node.

#### 2.5 Telegram Notification Delivery

- **Overview:**  
  Sends the AI-analyzed forex news and sentiment summaries as alerts to Telegram users.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - Type: Telegram  
    - Role: Sends messages to a Telegram chat or channel.  
    - Input: Receives final analyzed messages from the Langchain agent.  
    - Credential: Requires Telegram Bot API token with chat permissions.  
    - Edge cases: Network failures, invalid bot token, or blocked chat/channel.  
    - Webhook ID present, indicating possible webhook-based response capability.

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                       | Input Node(s)                                   | Output Node(s)              | Sticky Note |
|-------------------------------|---------------------------|-------------------------------------|------------------------------------------------|-----------------------------|-------------|
| Schedule Trigger              | Schedule Trigger          | Starts workflow on schedule          | -                                              | RSS Read - GOOGLE, fxstreet news, FXSTREET, DAILYFX technical, DAILYFX fundamental, DAILYFX forexnews, DAILYFX forex articles, FOREX LIVE technicals, FOREX LIVE central banks, FOREX LIVE forexorders |             |
| RSS Read - GOOGLE            | RSS Feed Read             | Reads Google forex news RSS feed     | Schedule Trigger                                | Merge                       |             |
| RSS Read -  fxstreet news    | RSS Feed Read             | Reads fxstreet news RSS feed         | Schedule Trigger                                | Merge                       |             |
| RSS Read - FXSTREET          | RSS Feed Read             | Reads FXSTREET forex news RSS feed   | Schedule Trigger                                | Merge                       |             |
| RSS Read - DAILYFX technical | RSS Feed Read             | Reads DAILYFX technical news RSS feed| Schedule Trigger                                | Merge                       |             |
| RSS Read - DAILYFX fundamental| RSS Feed Read            | Reads DAILYFX fundamental news RSS   | Schedule Trigger                                | Merge                       |             |
| RSS Read - DAILYFX forexnews | RSS Feed Read             | Reads DAILYFX forex news RSS feed    | Schedule Trigger                                | Merge                       |             |
| RSS Read - DAILYFX forex articles| RSS Feed Read         | Reads DAILYFX forex articles RSS feed| Schedule Trigger                                | Merge                       |             |
| RSS Read - FOREX LIVE technicals| RSS Feed Read          | Reads FOREX LIVE technicals RSS feed | Schedule Trigger                                | Merge                       |             |
| RSS Read - FOREX LIVE central banks| RSS Feed Read         | Reads FOREX LIVE central banks RSS   | Schedule Trigger                                | Merge                       |             |
| RSS Read - FOREX LIVE forexorders| RSS Feed Read          | Reads FOREX LIVE forex orders RSS    | Schedule Trigger                                | Merge                       |             |
| Merge                        | Merge                     | Consolidates all RSS feed data       | All RSS Read nodes                              | Filter                      |             |
| Filter                       | Filter                    | Filters merged news items             | Merge                                           | Code                        |             |
| Code                         | Code                      | Custom JS preprocessing               | Filter                                           | EURUSD News and Sentiment Analyst |             |
| Google Gemini Chat Model     | Langchain AI Model        | Provides AI language model backend   | Connected as AI model to Langchain Agent       | EURUSD News and Sentiment Analyst |             |
| EURUSD News and Sentiment Analyst | Langchain Agent       | Performs AI sentiment analysis       | Code (main), Google Gemini Chat Model (AI)     | Telegram                    |             |
| Telegram                     | Telegram                  | Sends final alerts to Telegram users | EURUSD News and Sentiment Analyst               | -                           |             |
| Sticky Note                  | Sticky Note               | Visual annotation                    | -                                                | -                           |             |
| Sticky Note1                 | Sticky Note               | Visual annotation                    | -                                                | -                           |             |
| Sticky Note3                 | Sticky Note               | Visual annotation                    | -                                                | -                           |             |
| Sticky Note5                 | Sticky Note               | Visual annotation                    | -                                                | -                           |             |
| Sticky Note6                 | Sticky Note               | Visual annotation                    | -                                                | -                           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Configure to run once daily at 00:00:22 am (Europe/London timezone).  
   - No input connections.  

2. **Create RSS Feed Read Nodes (10 total):**  
   - Types: RSS Feed Read  
   - Configure each node with its respective forex news RSS URL (from Google, FXStreet, DailyFX categories, Forex Live).  
   - Connect each node’s input from the Schedule Trigger’s output (parallel connections).  

3. **Create Merge Node:**  
   - Type: Merge  
   - Connect all RSS Feed Read nodes to the Merge node’s inputs.  
   - Use default merge mode to concatenate all inputs into one output array.  

4. **Create Filter Node:**  
   - Type: Filter  
   - Connect Merge node output to Filter node input.  
   - Define filter rules to select relevant news (e.g., keywords, date filters).  

5. **Create Code Node:**  
   - Type: Code  
   - Connect Filter node output to Code node input.  
   - Write JavaScript code to clean, enrich, or reformat the filtered news items.  

6. **Create Google Gemini Chat Model Node:**  
   - Type: Langchain Google Gemini Chat Model  
   - Configure with Google Gemini API credentials.  
   - No direct input; this node serves as an AI model provider for the Langchain agent.  

7. **Create Langchain Agent Node (EURUSD News and Sentiment Analyst):**  
   - Type: Langchain Agent  
   - Connect Code node output to this node’s main input.  
   - Set Google Gemini Chat Model node as the AI language model input.  
   - Configure prompts and chains to analyze EURUSD forex sentiment and news.  

8. **Create Telegram Node:**  
   - Type: Telegram  
   - Connect Langchain Agent output to Telegram node input.  
   - Configure with Telegram Bot API token and specify chat/channel ID.  
   - Set message content to the Langchain agent’s output.  

9. **Test Workflow:**  
   - Run manually or wait for scheduled trigger.  
   - Verify data flows through all nodes and Telegram receives messages.  

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                   |
|----------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow integrates multiple forex news sources and AI tools for enhanced trading insights. | Workflow description                              |
| Google Gemini API credentials and Telegram Bot token must be securely configured.             | Credential setup                                  |
| For more on Langchain agents and Google Gemini integration, see n8n documentation and Langchain resources. | https://docs.n8n.io/nodes/ai/                      |
| Telegram Bot setup guide: https://core.telegram.org/bots                                      | Telegram integration details                      |

---

**Disclaimer:** The text above is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.