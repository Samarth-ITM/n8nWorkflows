Track Daily Brand Mentions from Hacker News to Slack with GPT-4o-mini Sentiment Analysis

https://n8nworkflows.xyz/workflows/track-daily-brand-mentions-from-hacker-news-to-slack-with-gpt-4o-mini-sentiment-analysis-11080


# Track Daily Brand Mentions from Hacker News to Slack with GPT-4o-mini Sentiment Analysis

### 1. Workflow Overview

This workflow automates daily monitoring of online brand mentions on Hacker News, analyzes their sentiment using AI, and reports the findings to a Slack channel. It is designed to help brand teams track reputation, detect issues early, and summarize brand-related discussions efficiently.

Logical blocks grouped by functionality:

- **1.1 Trigger & Brand Setup**  
  Scheduled daily trigger and brand keyword configuration.

- **1.2 Fetch & Normalize Mentions**  
  Fetches recent Hacker News stories mentioning the brand and normalizes raw data.

- **1.3 AI Sentiment Classification**  
  Uses GPT-4o-mini to classify each mention’s sentiment, stance, topic, and urgency.

- **1.4 Format & Summarize Data**  
  Maps AI output into a clean format and generates a daily summary report for Slack.

- **1.5 Slack Notification**  
  Sends either the daily report or a “no mentions” message to Slack.

- **1.6 Error Handling**  
  Captures workflow errors and sends alerts to Slack for debugging.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Brand Setup

- **Overview:**  
  This block initiates the workflow daily at 09:00 and sets the brand and keyword used for searching mentions.

- **Nodes Involved:**  
  - Every Day at 09:00  
  - Brand Config

- **Node Details:**

  - **Every Day at 09:00**  
    - Type: Schedule Trigger  
    - Configuration: Cron expression `0 9 * * *` triggers workflow daily at 9 AM.  
    - Inputs: None (trigger node)  
    - Outputs: Brand Config  
    - Edge Cases: Cron misconfiguration, timezone issues.

  - **Brand Config**  
    - Type: Set  
    - Configuration: Defines two string variables: `brand` and `keywords` both set to "OpenAI". Used for filtering mentions.  
    - Inputs: Every Day at 09:00  
    - Outputs: Fetch Reddit Mentions  
    - Edge Cases: Empty or incorrect brand/keyword values could cause zero results.

---

#### 2.2 Fetch & Normalize Mentions

- **Overview:**  
  Fetches Hacker News stories matching the brand keyword and normalizes the raw API response into a consistent structure for downstream processing.

- **Nodes Involved:**  
  - Fetch Reddit Mentions  
  - Normalize Mentions  
  - Checking Score  
  - Mentions Exist?

- **Node Details:**

  - **Fetch Reddit Mentions**  
    - Type: HTTP Request  
    - Configuration: Queries Hacker News Algolia API with brand keywords, tagging stories only. Sends a custom User-Agent header "n8n-monitor-bot".  
    - Input: Brand Config  
    - Output: Normalize Mentions  
    - Edge Cases: HTTP errors (timeouts, 429 rate limits), empty or malformed responses.

  - **Normalize Mentions**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Extracts `hits` array from API response.  
      - Maps each mention to a normalized object with fields: platform, title, url, snippet, author, created_at (ISO 8601), brand, points, num_comments.  
      - Defaults for missing data: URL fallback to Hacker News item link, author "unknown".  
    - Input: Fetch Reddit Mentions  
    - Output: Checking Score  
    - Edge Cases: Empty hits array, missing fields, date parsing errors.

  - **Checking Score**  
    - Type: If  
    - Configuration: Checks if `num_comments` > 1000 for any mention (threshold for a special condition).  
    - Input: Normalize Mentions  
    - Output: Mentions Exist?  
    - Edge Cases: Missing `num_comments` field, type errors.

  - **Mentions Exist?**  
    - Type: If  
    - Configuration: Tests if any mentions exist (`length > 0`).  
    - Input: Checking Score  
    - Outputs:  
      - True: AI Agent - Classify Mention Sentiments  
      - False: No Mentions Message  
    - Edge Cases: Empty input, expression evaluation issues.

---

#### 2.3 AI Sentiment Classification

- **Overview:**  
  Uses GPT-4o-mini via LangChain agent to classify each mention’s sentiment, stance, topic, urgency, and provide a reason, outputting structured JSON.

- **Nodes Involved:**  
  - AI Agent - Classify Mention Sentiments  
  - OpenAI Chat Model - GPT-4o-mini  
  - Memory - Conversation Buffer  
  - Output Parser - Structured JSON  
  - Format Sentiment Data

- **Node Details:**

  - **AI Agent - Classify Mention Sentiments**  
    - Type: LangChain Agent  
    - Configuration:  
      - Input text includes brand, platform, title, snippet.  
      - System prompt instructs to classify mentions by sentiment (positive, negative, neutral), stance (supports, criticizes, questioning, just mentioning), topic (pricing, quality, support, features, bugs, performance, general, other), urgency (low, medium, high).  
      - Must output strict JSON only without markdown.  
    - Inputs: From Mentions Exist? node (true branch)  
    - Outputs: Format Sentiment Data  
    - Linked nodes: OpenAI model, memory buffer, output parser (integrated via LangChain)  
    - Edge Cases: AI rate limits, response parsing errors, incomplete or malformed JSON output.

  - **OpenAI Chat Model - GPT-4o-mini**  
    - Type: LangChain LLM Chat Model  
    - Configuration: Uses GPT-4o-mini model with OpenAI API credentials.  
    - Inputs: AI Agent - Classify Mention Sentiments  
    - Outputs: AI Agent - Classify Mention Sentiments  
    - Edge Cases: API authentication failures, quota limits, latency.

  - **Memory - Conversation Buffer**  
    - Type: LangChain Memory Buffer Window  
    - Configuration: Maintains conversation context with session key “GEO Classify”.  
    - Inputs: AI Agent - Classify Mention Sentiments  
    - Outputs: AI Agent - Classify Mention Sentiments  
    - Edge Cases: Memory overflow, session key conflicts.

  - **Output Parser - Structured JSON**  
    - Type: LangChain Output Parser  
    - Configuration: JSON schema example for sentiment, stance, topic, urgency, reason fields.  
    - Inputs: AI Agent - Classify Mention Sentiments  
    - Outputs: AI Agent - Classify Mention Sentiments  
    - Edge Cases: Parsing failures if AI output deviates from schema.

  - **Format Sentiment Data**  
    - Type: Set  
    - Configuration: Maps AI output fields (sentiment, stance, topic, urgency, reason) into clean JSON properties.  
    - Inputs: AI Agent - Classify Mention Sentiments  
    - Outputs: Build Daily Summary  
    - Edge Cases: Missing or undefined AI output properties.

---

#### 2.4 Format & Summarize Data

- **Overview:**  
  Builds a detailed daily summary report of brand mentions with sentiment counts and top 10 mentions formatted for Slack.

- **Nodes Involved:**  
  - Build Daily Summary

- **Node Details:**

  - **Build Daily Summary**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Aggregates total mentions and counts by sentiment (positive, negative, neutral).  
      - Selects top 10 mentions (based on input order).  
      - Constructs Slack-formatted markdown message including total counts and list of top mentions with emojis reflecting sentiment.  
      - Includes mention details: title, platform, topic, urgency, and URL.  
    - Inputs: Format Sentiment Data  
    - Outputs: Send Daily Report to Slack  
    - Edge Cases: Empty input, missing sentiment fields, message length limits in Slack.

---

#### 2.5 Slack Notification

- **Overview:**  
  Sends the generated daily summary or a “no mentions” message to a configured Slack channel.

- **Nodes Involved:**  
  - Send Daily Report to Slack  
  - No Mentions Message  
  - Send No Mentions to Slack

- **Node Details:**

  - **Send Daily Report to Slack**  
    - Type: Slack  
    - Configuration:  
      - Sends `text` field from Build Daily Summary node to a Slack channel (configured by channel ID).  
      - Uses Slack API credentials (OAuth2).  
    - Inputs: Build Daily Summary  
    - Edge Cases: Slack API rate limits, invalid channel ID, credential expiry.

  - **No Mentions Message**  
    - Type: Set  
    - Configuration: Sets a message string like “No new AI engine mentions were found today for [brand].” using brand from Brand Config.  
    - Inputs: Mentions Exist? (false branch)  
    - Outputs: Send No Mentions to Slack  
    - Edge Cases: Missing brand variable.

  - **Send No Mentions to Slack**  
    - Type: Slack  
    - Configuration: Sends the no mentions message to Slack channel using the same credentials as above.  
    - Inputs: No Mentions Message  
    - Edge Cases: Same as Send Daily Report to Slack.

---

#### 2.6 Error Handling

- **Overview:**  
  Listens for workflow errors and sends an alert with error details to Slack for rapid troubleshooting.

- **Nodes Involved:**  
  - Error Handler Trigger  
  - Slack: Send Error Alert

- **Node Details:**

  - **Error Handler Trigger**  
    - Type: Error Trigger  
    - Configuration: Listens for any errors during workflow execution.  
    - Inputs: N/A (event-driven)  
    - Outputs: Slack: Send Error Alert  
    - Edge Cases: Failure to catch errors if node misconfigured.

  - **Slack: Send Error Alert**  
    - Type: Slack  
    - Configuration:  
      - Formats error details including node name, error message, and timestamp into Slack message.  
      - Uses OAuth2 Slack credentials.  
      - Sends to configured channel.  
    - Inputs: Error Handler Trigger  
    - Edge Cases: Slack API failures, missing error context.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                       | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                                                                                                                                                  |
|-------------------------------|----------------------------------|-------------------------------------|------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Every Day at 09:00             | Schedule Trigger                 | Daily trigger at 09:00               | -                            | Brand Config                    | ## Trigger & Brand Setup<br>Runs daily at 09:00 and loads the brand name + keyword filters.                                                                                                                                   |
| Brand Config                  | Set                             | Defines brand and keywords           | Every Day at 09:00            | Fetch Reddit Mentions           | ## Trigger & Brand Setup<br>Runs daily at 09:00 and loads the brand name + keyword filters.                                                                                                                                   |
| Fetch Reddit Mentions          | HTTP Request                    | Fetches Hacker News mentions        | Brand Config                 | Normalize Mentions              | ## Fetch Mentions<br>Queries Hacker News for new stories matching the brand keyword.                                                                                                                                         |
| Normalize Mentions             | Code                            | Normalizes raw API response          | Fetch Reddit Mentions         | Checking Score                 | ## Normalize Mentions<br>Converts raw API results into a clean structure for AI classification.                                                                                                                              |
| Checking Score                | If                              | Checks if num_comments > 1000        | Normalize Mentions            | Mentions Exist?                | ## Normalize Mentions<br>Converts raw API results into a clean structure for AI classification.                                                                                                                              |
| Mentions Exist?               | If                              | Checks if any mentions exist         | Checking Score                | AI Agent - Classify Mention Sentiments, No Mentions Message | ## Normalize Mentions<br>Converts raw API results into a clean structure for AI classification.                                                                                                                              |
| AI Agent - Classify Mention Sentiments | LangChain Agent           | AI classifies sentiment, stance, etc.| Mentions Exist?              | Format Sentiment Data           | ## AI Classification<br>AI evaluates sentiment, stance, topic, and urgency for each mention.                                                                                                                                 |
| OpenAI Chat Model - GPT-4o-mini | LangChain LLM Chat Model       | Provides GPT-4o-mini model for AI    | AI Agent - Classify Mention Sentiments | AI Agent - Classify Mention Sentiments | ## AI Classification<br>AI evaluates sentiment, stance, topic, and urgency for each mention.                                                                                                                                 |
| Memory - Conversation Buffer   | LangChain Memory Buffer Window  | Maintains AI conversation context    | AI Agent - Classify Mention Sentiments | AI Agent - Classify Mention Sentiments | ## AI Classification<br>AI evaluates sentiment, stance, topic, and urgency for each mention.                                                                                                                                 |
| Output Parser - Structured JSON | LangChain Output Parser         | Parses AI output into structured JSON| AI Agent - Classify Mention Sentiments | AI Agent - Classify Mention Sentiments | ## AI Classification<br>AI evaluates sentiment, stance, topic, and urgency for each mention.                                                                                                                                 |
| Format Sentiment Data          | Set                             | Maps AI output to clean JSON fields  | AI Agent - Classify Mention Sentiments | Build Daily Summary            | ## Format Sentiment Output and Summary<br>Maps AI-generated sentiment fields into clean JSON for reporting.Builds a daily Slack-friendly summary including top trending mentions.                                            |
| Build Daily Summary            | Code                            | Builds daily summary report text     | Format Sentiment Data         | Send Daily Report to Slack      | ## Format Sentiment Output and Summary<br>Maps AI-generated sentiment fields into clean JSON for reporting.Builds a daily Slack-friendly summary including top trending mentions.                                            |
| Send Daily Report to Slack     | Slack                           | Sends daily summary report to Slack | Build Daily Summary           | -                             | ## Slack Notifications<br>Sends either a full daily report or a “no mentions today” message.                                                                                                                                 |
| No Mentions Message            | Set                             | Sets "no mentions" message text      | Mentions Exist? (false)       | Send No Mentions to Slack       | ## Slack Notifications<br>Sends either a full daily report or a “no mentions today” message.                                                                                                                                 |
| Send No Mentions to Slack      | Slack                           | Sends no mentions message to Slack   | No Mentions Message           | -                             | ## Slack Notifications<br>Sends either a full daily report or a “no mentions today” message.                                                                                                                                 |
| Error Handler Trigger          | Error Trigger                   | Catches workflow errors              | -                            | Slack: Send Error Alert         | ## Error Handling<br>Captures workflow failures and sends details to Slack for quick debugging.                                                                                                                              |
| Slack: Send Error Alert        | Slack                           | Sends error details to Slack         | Error Handler Trigger         | -                             | ## Error Handling<br>Captures workflow failures and sends details to Slack for quick debugging.                                                                                                                              |
| Sticky Notes (various)         | Sticky Note                     | Documentation and explanations       | -                            | -                             | Multiple, see respective node rows for content, including overview, block explanations, and error handling notes.                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Add a **Schedule Trigger** node named "Every Day at 09:00".  
   - Set Cron expression: `0 9 * * *` (runs daily at 09:00).  

2. **Create Brand Config Node**  
   - Add a **Set** node named "Brand Config".  
   - Set two string variables:  
     - `brand`: `"OpenAI"`  
     - `keywords`: `"OpenAI"`  
   - Connect "Every Day at 09:00" → "Brand Config".

3. **Create Fetch Mentions Node**  
   - Add an **HTTP Request** node named "Fetch Reddit Mentions".  
   - Configure URL: `http://hn.algolia.com/api/v1/search?query={{$json["keywords"]}}&tags=story` (expression mode).  
   - Set HTTP header: `User-Agent: n8n-monitor-bot`.  
   - Connect "Brand Config" → "Fetch Reddit Mentions".

4. **Create Normalize Mentions Node**  
   - Add a **Code** node named "Normalize Mentions".  
   - Paste provided JavaScript to extract and normalize `hits` from response, mapping fields platform, title, url, snippet, author, created_at, brand, points, num_comments.  
   - Connect "Fetch Reddit Mentions" → "Normalize Mentions".

5. **Create Checking Score Node**  
   - Add an **If** node named "Checking Score".  
   - Condition: Check if `$json.num_comments > 1000`.  
   - Connect "Normalize Mentions" → "Checking Score".

6. **Create Mentions Exist? Node**  
   - Add an **If** node named "Mentions Exist?".  
   - Condition: Check if `$items().length > 0`.  
   - Connect "Checking Score" (main output) → "Mentions Exist?".

7. **Set up AI Classification Nodes**  
   - Add **LangChain Agent** node named "AI Agent - Classify Mention Sentiments".  
     - Configure input text with brand, platform, title, snippet.  
     - System message to instruct classification by sentiment, stance, topic, urgency, strict JSON output as per schema.  
   - Add **LangChain OpenAI Chat Model** node named "OpenAI Chat Model - GPT-4o-mini".  
     - Set model to "gpt-4o-mini" with OpenAI credentials.  
   - Add **LangChain Memory Buffer** node named "Memory - Conversation Buffer".  
     - Use session key `"GEO Classify"`.  
   - Add **LangChain Output Parser** node named "Output Parser - Structured JSON".  
     - Provide JSON schema example for classification fields.  
   - Wire these LangChain nodes accordingly:  
     - "OpenAI Chat Model - GPT-4o-mini" → AI Agent (llm)  
     - "Memory - Conversation Buffer" → AI Agent (memory)  
     - "Output Parser - Structured JSON" → AI Agent (output parser)  
   - Connect "Mentions Exist?" (true) → "AI Agent - Classify Mention Sentiments".

8. **Create Format Sentiment Data Node**  
   - Add a **Set** node named "Format Sentiment Data".  
   - Assign variables from AI Agent output: sentiment, stance, topic, urgency, reason.  
   - Connect "AI Agent - Classify Mention Sentiments" → "Format Sentiment Data".

9. **Create Build Daily Summary Node**  
   - Add a **Code** node named "Build Daily Summary".  
   - Paste provided JavaScript to aggregate sentiment counts, prepare top 10 mentions, and build Slack message text with markdown formatting.  
   - Connect "Format Sentiment Data" → "Build Daily Summary".

10. **Create Send Daily Report to Slack Node**  
    - Add a **Slack** node named "Send Daily Report to Slack".  
    - Configure Slack API credentials (OAuth2 or token-based).  
    - Select the Slack channel to send messages to.  
    - Map message text from "Build Daily Summary".  
    - Connect "Build Daily Summary" → "Send Daily Report to Slack".

11. **Create No Mentions Message Node**  
    - Add a **Set** node named "No Mentions Message".  
    - Set string variable `text` to:  
      `"No new AI engine mentions were found today for {{$node["Brand Config"].json["brand"]}}."`  
    - Connect "Mentions Exist?" (false) → "No Mentions Message".

12. **Create Send No Mentions to Slack Node**  
    - Add a **Slack** node named "Send No Mentions to Slack".  
    - Configure Slack credentials and channel as above.  
    - Map message text from "No Mentions Message".  
    - Connect "No Mentions Message" → "Send No Mentions to Slack".

13. **Create Error Handling Nodes**  
    - Add an **Error Trigger** node named "Error Handler Trigger".  
    - Add a **Slack** node named "Slack: Send Error Alert".  
    - Configure Slack credentials and channel.  
    - Set message text using expressions to include error node name, message, and timestamp.  
    - Connect "Error Handler Trigger" → "Slack: Send Error Alert".

14. **Add Sticky Notes (Optional)**  
    - Add sticky notes with documentation content for clarity during development.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Daily AI Engine Mentions Tracker helps brands monitor Hacker News mentions and sentiment automatically.                                                                                                                           | Workflow overview sticky note.                                                                                   |
| Setup requires Slack API credentials with permission to post to a channel, and OpenAI API credentials for GPT-4o-mini model.                                                                                                       | Credentials setup instructions.                                                                                   |
| Hacker News API used is Algolia’s search API: https://hn.algolia.com/api                                                                                                                     | API info referenced in Fetch Reddit Mentions node.                                                               |
| Slack messages use markdown formatting with emojis for sentiment indicators (✅ positive, ❌ negative, ⚪ neutral).                                                                                                                | Formatting details in Build Daily Summary node.                                                                   |
| Error handling notifies Slack channel on workflow failures for prompt debugging.                                                                                                                                                   | Error Handling sticky note.                                                                                        |
| For AI classification, strict JSON output enforced with schema ensures reliable parsing downstream.                                                                                                                               | Output Parser - Structured JSON node details.                                                                     |
| Workflow is designed to be extensible to other platforms or brands by adjusting brand config and fetch node URL accordingly.                                                                                                       | General workflow adaptability note.                                                                               |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.