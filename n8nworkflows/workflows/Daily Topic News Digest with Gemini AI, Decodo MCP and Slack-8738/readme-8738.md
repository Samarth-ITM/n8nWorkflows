Daily Topic News Digest with Gemini AI, Decodo MCP and Slack

https://n8nworkflows.xyz/workflows/daily-topic-news-digest-with-gemini-ai--decodo-mcp-and-slack-8738


# Daily Topic News Digest with Gemini AI, Decodo MCP and Slack

### 1. Workflow Overview

This workflow automates the daily collection, processing, and delivery of fresh news articles related to predefined technology topics using AI and web scraping tools. It is designed for teams or individuals who want an autonomous news digest focused on specific subjects such as generative AI, model context protocols, and web scraping. The digest is delivered daily via Slack in a well-formatted, deduplicated summary.

The workflow consists of the following logical blocks:

- **1.1 Daily Trigger & Topic Setup:** Schedules the workflow to run daily at 9 AM and defines the news topics to search.
- **1.2 AI-Powered News Discovery:** Employs a LangChain agent with Google Gemini AI and Decodo MCP tools to find fresh news articles from the last 48 hours for each topic.
- **1.3 Data Cleaning and Organization:** Formats the raw AI output, splits the article list, removes duplicates, and aggregates the results into a final list.
- **1.4 Slack Delivery:** Sends the curated news digest to a Slack channel with clickable links and summaries.
- **1.5 Documentation and Guidance:** Sticky notes provide detailed explanations, instructions, and links to resources.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Trigger & Topic Setup

- **Overview:** This block triggers the workflow every day at 9 AM and sets the predefined list of news topics.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Set Topics  
  - Sticky Note (Daily Trigger & Topics)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type & Role:* Built-in time-based trigger node to start the workflow daily.  
    - *Configuration:* Set to trigger daily at 9:00 AM (hour 9).  
    - *Inputs/Outputs:* No input; output triggers Set Topics node.  
    - *Failure Modes:* Possible scheduling misconfiguration or timezone mismatch.

  - **Set Topics**  
    - *Type & Role:* Sets static JSON containing the topics to search.  
    - *Configuration:* Four topics defined — “model context protocol (mcp)”, “generative AI”, “agentic AI”, “web scraping”.  
    - *Expressions:* None; static JSON input.  
    - *Inputs/Outputs:* Input from Schedule Trigger, output to AI agent node.  
    - *Failure Modes:* None significant unless manual edits cause invalid JSON.

  - **Sticky Note (Daily Trigger & Topics)**  
    - *Type & Role:* Informational.  
    - *Content:* Explains the daily schedule and topics.  
    - *Connections:* None (visual aid only).  

---

#### 2.2 AI-Powered News Discovery

- **Overview:** This block leverages the Google Gemini language model and Decodo MCP web scraping tools via a LangChain agent to discover and extract fresh news articles related to the defined topics.
- **Nodes Involved:**  
  - Gemini Chat Model  
  - Decodo MCP  
  - Agent: News Finder  
  - Sticky Note (Smart News Agent)

- **Node Details:**

  - **Gemini Chat Model**  
    - *Type & Role:* AI language model node using Google Gemini (PaLM) for natural language understanding and generation.  
    - *Configuration:* Standard setup with Google Palm API credentials.  
    - *Inputs/Outputs:* Feeds AI language model input to the Agent node.  
    - *Failure Modes:* API authentication errors, rate limits, or network timeouts.

  - **Decodo MCP**  
    - *Type & Role:* Provides web scraping and content extraction capabilities via MCP (Model Context Protocol) tools.  
    - *Configuration:* Endpoint URL includes placeholders for API key and profile ID (to be replaced with user credentials). Uses HTTP streamable transport.  
    - *Inputs/Outputs:* Exposes scraping tools to the Agent node.  
    - *Failure Modes:* Authentication failures, server unavailability, or malformed requests.

  - **Agent: News Finder**  
    - *Type & Role:* LangChain agent that orchestrates interaction between AI language model and Decodo MCP tools to collect news articles.  
    - *Configuration:*  
      - Text prompt instructs the agent to search for fresh news items per topic, using only Decodo MCP tools, avoiding duplicates, and providing structured JSON output with key fields (title, url, topic, source, date, summary).  
      - Max tries: 2 attempts on failure.  
      - Does not retry on fail by default.  
    - *Expressions:* Uses topics from Set Topics node.  
    - *Inputs/Outputs:* Receives topics and AI tool references; outputs raw JSON article lists to Format Results.  
    - *Failure Modes:* AI generation errors, tool call failures, invalid JSON output.

  - **Sticky Note (Smart News Agent)**  
    - *Type & Role:* Documentation describing the AI-powered news discovery process and integration with Decodo MCP tools.  
    - *Content:* Explains the purpose and functionality of the AI agent.  

---

#### 2.3 Data Cleaning and Organization

- **Overview:** This block processes the raw output from the AI agent: it formats the JSON, splits the article list into individual items, removes duplicates based on URL and title, and aggregates the cleaned articles into a single combined list.
- **Nodes Involved:**  
  - Format Results  
  - Split  
  - Remove Duplicates  
  - Combine Results  
  - Sticky Note (Clean & Organize)

- **Node Details:**

  - **Format Results**  
    - *Type & Role:* Code node to parse and normalize the AI's raw JSON output.  
    - *Configuration:*  
      - Strips code fences (```json ... ```) if present.  
      - Extracts first JSON array from any extra text.  
      - Parses JSON safely, with fallback to fix trailing commas.  
      - Normalizes each article with fields: title, url, source, date (ISO string), topic, summary.  
      - Returns a clean array under `output.items`.  
    - *Inputs/Outputs:* Input from Agent node; output to Split node.  
    - *Failure Modes:* JSON parsing errors if AI output is malformed beyond recovery.

  - **Split**  
    - *Type & Role:* Splits the array of articles into individual items for deduplication.  
    - *Configuration:* Splits on the field `output.items`.  
    - *Inputs/Outputs:* Input from Format Results; output to Remove Duplicates.  
    - *Failure Modes:* Empty or missing array field may cause no output.

  - **Remove Duplicates**  
    - *Type & Role:* Removes duplicate news articles based on specified fields.  
    - *Configuration:* Compares `url` and `title` fields to identify duplicates.  
    - *Inputs/Outputs:* Input from Split; output to Combine Results.  
    - *Failure Modes:* Misconfiguration could cause false duplicates or miss duplicates.

  - **Combine Results**  
    - *Type & Role:* Aggregates all deduplicated items into a single array field named `articles`.  
    - *Configuration:* Aggregates all item data into one JSON object.  
    - *Inputs/Outputs:* Input from Remove Duplicates; output to Slack node.  
    - *Failure Modes:* Aggregation failures if no input items.

  - **Sticky Note (Clean & Organize)**  
    - *Type & Role:* Documentation describing the data cleaning workflow.  
    - *Content:* Explains each step from formatting to deduplication and aggregation.

---

#### 2.4 Slack Delivery

- **Overview:** This block formats and sends the final news digest to a designated Slack channel with clickable links, source attribution, and summaries.
- **Nodes Involved:**  
  - Slack  
  - Sticky Note (Send to Slack)

- **Node Details:**

  - **Slack**  
    - *Type & Role:* Slack node to post messages using Slack API and webhook.  
    - *Configuration:*  
      - Posts to the channel named `#random`.  
      - Message text is dynamically generated using a JavaScript expression that:  
        - Takes aggregated articles; falls back to current items if needed.  
        - Escapes text for Slack markdown safety.  
        - Sorts articles by date descending.  
        - Formats each article as a bullet with clickable title linking to URL, source domain, and summary.  
        - Provides fallback message "No fresh items today." if no articles.  
      - Uses OAuth2 Slack API credentials.  
    - *Inputs/Outputs:* Input from Combine Results; no output.  
    - *Failure Modes:* Slack API authentication errors, webhook misconfiguration, channel permission issues.

  - **Sticky Note (Send to Slack)**  
    - *Type & Role:* Documentation explaining the Slack delivery purpose.  
    - *Content:* Notes that the node delivers the digest with clickable links and summaries.

---

#### 2.5 Documentation and Guidance

- **Overview:** Several sticky notes provide detailed instructions, usage tips, and links to resources for the entire workflow.
- **Nodes Involved:**  
  - Sticky Note4 (Try It Out!)

- **Node Details:**

  - **Sticky Note4 (Try It Out!)**  
    - *Type & Role:* Comprehensive documentation covering:  
      - Workflow use cases (automated news monitoring, competitive intelligence, etc.).  
      - Step-by-step explanation of how the workflow functions.  
      - Setup requirements (Decodo MCP credentials, Gemini API key, Slack workspace).  
      - Links to external resources and support (Decodo MCP setup guide, Gemini API key page, Slack docs, Discord and support email).  
    - *Connections:* None (informational only).

---

### 3. Summary Table

| Node Name          | Node Type                                   | Functional Role                       | Input Node(s)           | Output Node(s)          | Sticky Note                                         |
|--------------------|---------------------------------------------|------------------------------------|------------------------|-------------------------|-----------------------------------------------------|
| Schedule Trigger    | Schedule Trigger                            | Daily workflow trigger             | —                      | Set Topics              | ## ⏰ DAILY TRIGGER & TOPICS                        |
| Set Topics         | Set                                         | Defines news topics                | Schedule Trigger        | Agent: News Finder      | ## ⏰ DAILY TRIGGER & TOPICS                        |
| Gemini Chat Model  | LM Chat Google Gemini                        | AI language model for agent        | —                      | Agent: News Finder (lm) | ## 🤖 SMART NEWS AGENT                              |
| Decodo MCP         | Decodo MCP Client Tool                       | Web scraping tool                  | —                      | Agent: News Finder (tool)| ## 🤖 SMART NEWS AGENT                              |
| Agent: News Finder | LangChain Agent                              | AI agent orchestrating news search| Set Topics, Gemini Chat Model, Decodo MCP | Format Results         | ## 🤖 SMART NEWS AGENT                              |
| Format Results     | Code                                         | Cleans and normalizes AI output    | Agent: News Finder      | Split                   | ## 🔧 CLEAN & ORGANIZE                              |
| Split              | Split Out                                    | Splits articles array into items   | Format Results          | Remove Duplicates       | ## 🔧 CLEAN & ORGANIZE                              |
| Remove Duplicates  | Remove Duplicates                            | Removes duplicate articles         | Split                   | Combine Results         | ## 🔧 CLEAN & ORGANIZE                              |
| Combine Results    | Aggregate                                    | Aggregates cleaned articles        | Remove Duplicates       | Slack                   | ## 🔧 CLEAN & ORGANIZE                              |
| Slack              | Slack                                        | Sends news digest to Slack channel | Combine Results         | —                       | ## 📱 SEND TO SLACK                                |
| Sticky Note        | Sticky Note                                  | Documentation                     | —                      | —                       | ## ⏰ DAILY TRIGGER & TOPICS                        |
| Sticky Note1       | Sticky Note                                  | Documentation                     | —                      | —                       | ## 🤖 SMART NEWS AGENT                              |
| Sticky Note2       | Sticky Note                                  | Documentation                     | —                      | —                       | ## 🔧 CLEAN & ORGANIZE                              |
| Sticky Note3       | Sticky Note                                  | Documentation                     | —                      | —                       | ## 📱 SEND TO SLACK                                |
| Sticky Note4       | Sticky Note                                  | Comprehensive usage instructions  | —                      | —                       | ## Try It Out! Full workflow documentation         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**
   - Type: Schedule Trigger  
   - Set to run daily at 9:00 AM (triggerAtHour: 9).  
   - No credentials required.  

2. **Create Set Topics node:**
   - Type: Set  
   - Mode: Raw JSON  
   - JSON Output:  
     ```json
     {
       "topics": [
         "model context protocol (mcp)",
         "generative AI",
         "agentic AI",
         "web scraping"
       ]
     }
     ```  
   - Connect Schedule Trigger output to Set Topics input.

3. **Create Gemini Chat Model node:**
   - Type: LM Chat Google Gemini  
   - Configure with Google Palm API credentials (set up OAuth2 or API key as per Google).  
   - No additional parameters needed.  

4. **Create Decodo MCP node:**
   - Type: Decodo MCP Client Tool (LangChain)  
   - Set Endpoint URL:  
     ```
     https://server.smithery.ai/@Decodo/decodo-mcp-server/mcp?api_key=YOUR_DECODO_API_KEY&profile=YOUR_DECODO_PROFILE_ID
     ```  
   - Replace placeholders with your Decodo MCP API key and profile ID.  
   - Set serverTransport to "httpStreamable".  

5. **Create Agent: News Finder node:**
   - Type: LangChain Agent  
   - Set “text” parameter to the following prompt (replace {{$json.topics}} with expression referencing Set Topics output):  
     ```
     You are a news scout. Topics: {{$json.topics}}.
     Goal: For each topic, use the Decodo MCP tools available to you to find fresh, high-quality news from the last 48 hours, extract essentials, and list concise candidates.

     Rules:
     - Use only tools exposed via Decodo MCP.
     - Prefer official/reputable sources; avoid spam and duplicates (same URL or same title).
     - Up to 5 items per topic, newest first.
     - Provide for each item: title, url, topic, (optional) source domain, publish date/time if available, 2–3 line plain-text summary.
     - If nothing relevant is found for a topic, skip it.

     Output format for THIS STEP ONLY:
     Return a JSON array of candidate objects, one per item (not wrapped in any parent object), e.g.:
     [
       {"title":"...", "url":"https://...", "topic":"...", "source":"example.com", "date":"2025-09-15T09:00:00Z", "summary":"..."}
     ]

     Do not include any markdown or commentary—just the JSON array.
     ```  
   - Set Max Tries to 2; do not enable retry on fail.  
   - Connect:  
     - AI language model input to Gemini Chat Model node output.  
     - AI tool input to Decodo MCP node output.  
     - Text input to Set Topics output.  

6. **Create Format Results node (Code):**
   - Type: Code  
   - Paste the provided JavaScript code that:  
     - Strips code fences from AI output.  
     - Extracts the first JSON array.  
     - Parses and normalizes each article with fields (title, url, source, date, topic, summary).  
   - Connect Agent: News Finder output to Format Results input.

7. **Create Split node:**
   - Type: Split Out  
   - Field to split: `output.items` (reference the array from Format Results).  
   - Connect Format Results output to Split input.

8. **Create Remove Duplicates node:**
   - Type: Remove Duplicates  
   - Compare by fields: `url`, `title`.  
   - Connect Split output to Remove Duplicates input.

9. **Create Combine Results node:**
   - Type: Aggregate  
   - Aggregate all item data into field `articles`.  
   - Connect Remove Duplicates output to Combine Results input.

10. **Create Slack node:**
    - Type: Slack  
    - Authenticate with Slack API OAuth2 credentials (workspace with permissions to post).  
    - Set Channel to `#random` or desired channel name.  
    - Set Text parameter to the JavaScript expression that:  
      - Aggregates articles from input or current items.  
      - Escapes Markdown special characters.  
      - Sorts articles by descending date.  
      - Formats each article as:  
        ```
        *<URL|Title>* (Source)
        Summary
        ```  
      - Fallback message if no articles: "No fresh items today."  
    - Connect Combine Results output to Slack input.

11. **Arrange nodes visually and optionally add Sticky Notes** with content from the documentation to improve understandability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This n8n template demonstrates how to build an autonomous AI news agent using Decodo MCP that automatically finds, scrapes, and delivers fresh industry news to your team via Slack. Use cases include automated news monitoring, competitive intelligence, startup tracking, regulatory updates, research automation, or daily briefings.                                     | Sticky Note4 full content                                                                                          |
| Decodo MCP provides web scraping and content extraction tools with geo-restriction and anti-bot bypass capabilities.                                                                                                                                                                                                                                                      | https://github.com/Decodo/mcp-web-scraper and https://decodo.com/blog/how-to-set-up-mcp-server                      |
| Gemini API key (Google PaLM) is required for AI processing; sign up and manage keys at https://aistudio.google.com/app/apikey.                                                                                                                                                                                                                                            | https://aistudio.google.com/app/apikey                                                                             |
| Slack workspace and API integration are required for message delivery. Configure Slack OAuth2 credentials in n8n accordingly.                                                                                                                                                                                                                                             | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/                                           |
| For support, join the Decodo Discord community or contact support@decodo.com.                                                                                                                                                                                                                                                                                              | https://discord.com/invite/gvJhWJPaB4 and mailto:support@decodo.com                                               |
| The workflow runs daily at 9 AM India Standard Time by default (adjust Schedule Trigger as needed).                                                                                                                                                                                                                                                                        | Schedule Trigger node configuration                                                                                |
| The AI agent output formatting code robustly handles malformed JSON and various output quirks to minimize errors.                                                                                                                                                                                                                                                         | Format Results node code                                                                                           |

---

This completes the comprehensive analysis and documentation of the "Daily Topic News Digest with Gemini AI, Decodo MCP and Slack" n8n workflow.