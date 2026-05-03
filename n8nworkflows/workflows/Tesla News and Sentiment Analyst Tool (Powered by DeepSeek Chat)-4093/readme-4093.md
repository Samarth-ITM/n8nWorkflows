Tesla News and Sentiment Analyst Tool (Powered by DeepSeek Chat)

https://n8nworkflows.xyz/workflows/tesla-news-and-sentiment-analyst-tool--powered-by-deepseek-chat--4093


# Tesla News and Sentiment Analyst Tool (Powered by DeepSeek Chat)

---

### 1. Workflow Overview

This workflow, **Tesla News and Sentiment Analyst Tool (Powered by DeepSeek Chat)**, is an AI-powered sub-agent designed to perform real-time sentiment analysis on Tesla (TSLA) news. It aggregates headlines from five trusted RSS news sources, analyzes sentiment and key themes using DeepSeek Chat AI, and produces structured JSON output that supports trading decisions. This tool is a critical component within the broader **Tesla Quant Trading AI Agent** system and is intended to be triggered exclusively by that parent agent.

**Logical Blocks:**

- **1.1 Input Reception:**  
  The workflow is triggered externally by the parent workflow, receiving optional `message` and `sessionId` parameters for context.

- **1.2 News Aggregation:**  
  Five RSS feed nodes gather real-time Tesla-related headlines from diverse, trusted media sources to ensure comprehensive market coverage.

- **1.3 Short-Term Memory Context:**  
  A memory buffer node retains session-level context to maintain continuity and prevent redundant analysis within the same session.

- **1.4 AI Processing (DeepSeek Chat Model):**  
  The DeepSeek LLM parses aggregated news headlines, detecting tone, narrative themes, and extracting structured insights.

- **1.5 Sentiment Analysis & Formatting:**  
  A LangChain Agent node consolidates AI output, generates sentiment classification, a concise summary, and selects top headlines, outputting strict JSON for downstream use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block activates the workflow when triggered by another workflow, passing in contextual inputs (`message`, `sessionId`).

- **Nodes Involved:**  
  - `When Executed by Another Workflow`

- **Node Details:**

  - **When Executed by Another Workflow**  
    - *Type & Role:* Execute Workflow Trigger node; entry point activated externally.  
    - *Configuration:* Accepts workflow inputs named `message` (optional string) and `sessionId` (string) to maintain session context.  
    - *Inputs / Outputs:* No input connections; outputs trigger main flow to the Sentiment Analyst node.  
    - *Failure Types:* Missing inputs, improper triggering, or execution permission issues.  
    - *Sticky Note:* "Trigger from Parent Workflow — Executed only by main TSLA agent. Ensure proper inputs message, sessionId are passed."

#### 1.2 News Aggregation

- **Overview:**  
  Collects Tesla-related news headlines from five distinct RSS feeds, each representing a unique source and coverage angle.

- **Nodes Involved:**  
  - `RSS - Google News Tesla-specific`  
  - `RSS - Yahoo Finance TSLA`  
  - `RSS - Electrek Tesla`  
  - `RSS - CleanTechnica Tesla Archives`  
  - `RSS - TeslaNorth`

- **Node Details:**

  - **RSS - Google News Tesla-specific**  
    - Type: RSS Feed Read Tool  
    - Role: Fetches Tesla and TSLA stock news aggregated from multiple top-tier sources filtered by Google News search.  
    - Configuration: RSS URL filtered by “Tesla OR TSLA stock” scoped to US English.  
    - Potential Failures: RSS feed downtime, network issues, malformed feed data.  
    - Sticky Note: Describes this as primary real-time Tesla news source from major outlets.

  - **RSS - Yahoo Finance TSLA**  
    - Type: RSS Feed Read Tool  
    - Role: Captures TSLA stock-specific financial news, including earnings and analyst updates.  
    - Configuration: Yahoo Finance RSS feed for TSLA, US region, English language.  
    - Potential Failures: Feed unavailability, rate limiting, network timeouts.  
    - Sticky Note: Emphasizes financial news relevance for stock catalysts.

  - **RSS - Electrek Tesla**  
    - Type: RSS Feed Read Tool  
    - Role: Provides detailed Tesla tech, factory, and vehicle updates from Electrek.  
    - Configuration: Electrek Tesla RSS feed URL.  
    - Potential Failures: Feed format changes, network errors.  
    - Sticky Note: Notes Electrek as in-depth source for Tesla technology news.

  - **RSS - CleanTechnica Tesla Archives**  
    - Type: RSS Feed Read Tool  
    - Role: Pulls content focused on Tesla’s clean energy initiatives, battery tech, and policy news.  
    - Configuration: CleanTechnica Tesla tag RSS feed URL.  
    - Potential Failures: Feed unavailability or parsing errors.  
    - Sticky Note: Highlights environmental and innovation angle coverage.

  - **RSS - TeslaNorth**  
    - Type: RSS Feed Read Tool  
    - Role: Captures Tesla consumer product updates, app changes, pricing adjustments.  
    - Configuration: TeslaNorth RSS feed URL.  
    - Potential Failures: Feed downtime, network issues.  
    - Sticky Note: Describes TeslaNorth as fast, consumer-focused news source.

  - **General:** All five nodes output their news items as tools used by the Sentiment Analyst AI node.

#### 1.3 Short-Term Memory Context

- **Overview:**  
  A memory buffer node preserves session-specific context to maintain continuity in analysis and prevent duplicate headline processing.

- **Nodes Involved:**  
  - `Simple Memory`

- **Node Details:**

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains short-term memory of Tesla-related prompts and analysis context across workflow executions.  
    - Configuration: Default memory buffer settings to hold recent session context; no custom parameters specified.  
    - Inputs / Outputs: Connected as AI memory source to the Sentiment Analyst node.  
    - Potential Failures: Memory overflow if session context grows excessively; potential mismatches with `sessionId` if incorrectly passed.  
    - Sticky Note: Explains short-term memory role in avoiding duplicate analysis and aligning headlines.

#### 1.4 AI Processing (DeepSeek Chat Model)

- **Overview:**  
  Uses DeepSeek’s large language model to parse all collected Tesla headlines, detect sentiment tone, and identify key narrative themes.

- **Nodes Involved:**  
  - `DeepSeek Chat Model`

- **Node Details:**

  - **DeepSeek Chat Model**  
    - Type: LangChain DeepSeek LLM Chat Node  
    - Role: Processes raw headline data, performing sentiment detection, theme extraction, and structured summary generation.  
    - Configuration: Uses stored DeepSeek API credentials under `DeepSeek account`; no additional parameters configured in node.  
    - Inputs / Outputs: Connected as AI language model used by the Sentiment Analyst node.  
    - Potential Failures: API key invalid or expired; network errors; DeepSeek API rate limiting or downtime.  
    - Sticky Note: Emphasizes this node outputs raw AI insight (not final decision), including tone and signal pattern detection.

#### 1.5 Sentiment Analysis & Formatting

- **Overview:**  
  This is the core agent node that consolidates inputs from all news feeds, memory, and the DeepSeek LLM, producing the final structured sentiment analysis JSON.

- **Nodes Involved:**  
  - `Tesla News and Sentiment Analyst`

- **Node Details:**

  - **Tesla News and Sentiment Analyst**  
    - Type: LangChain Agent Node  
    - Role: Orchestrates the entire sentiment analysis logic by calling all five RSS feed tools, the memory buffer, and the DeepSeek language model. It synthesizes the AI insights into a strict JSON output containing sentiment classification, a concise summary, and top 3–5 key headlines.  
    - Configuration:  
      - Input text taken from `message` workflow input.  
      - System message prompt defines role, data sources, analysis steps, output format (strict JSON).  
      - Uses all 5 RSS nodes as AI tools, `Simple Memory` as AI memory, and `DeepSeek Chat Model` as AI language model.  
    - Inputs / Outputs: Triggered by the initial workflow trigger node; outputs JSON object with keys: `sentiment`, `summary`, `topHeadlines`.  
    - Potential Failures: Expression evaluation errors if inputs missing; AI model failures; inconsistent RSS data; JSON parsing errors in output.  
    - Sticky Note: Describes responsibilities to combine LLM output, generate JSON with sentiment, summary, and top headlines.  
    - Sub-workflow Reference: This node acts as the main AI agent within this sub-workflow and is called by the parent Tesla Quant Trading AI Agent.

---

### 3. Summary Table

| Node Name                          | Node Type                         | Functional Role                                    | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                      |
|-----------------------------------|----------------------------------|---------------------------------------------------|-------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger          | Entry trigger from parent workflow                 | None                          | Tesla News and Sentiment Analyst| Trigger from Parent Workflow — Executed only by main TSLA agent. Ensure proper inputs message, sessionId are passed. |
| RSS - Google News Tesla-specific  | RSS Feed Read Tool                | Fetch Tesla news from Google News filtered feed   | None                          | Tesla News and Sentiment Analyst| News Feeds Overview: Lists and explains this as primary real-time Tesla news source.           |
| RSS - Yahoo Finance TSLA           | RSS Feed Read Tool                | Fetch TSLA financial news from Yahoo Finance      | None                          | Tesla News and Sentiment Analyst| News Feeds Overview: Captures TSLA specific financial news.                                    |
| RSS - Electrek Tesla               | RSS Feed Read Tool                | Fetch Tesla-related EV news from Electrek         | None                          | Tesla News and Sentiment Analyst| News Feeds Overview: Electrek is in-depth source for Tesla technology and operations.          |
| RSS - CleanTechnica Tesla Archives | RSS Feed Read Tool                | Fetch Tesla sustainability and clean energy news | None                          | Tesla News and Sentiment Analyst| News Feeds Overview: Offers clean energy and sustainability insights.                           |
| RSS - TeslaNorth                  | RSS Feed Read Tool                | Fetch Tesla product and consumer updates          | None                          | Tesla News and Sentiment Analyst| News Feeds Overview: Fast technical and consumer updates source.                              |
| Simple Memory                    | LangChain Memory Buffer Window    | Maintain short-term session context                | None                          | Tesla News and Sentiment Analyst| Short-Term Memory Module: Holds session-specific context to avoid duplicate analysis.          |
| DeepSeek Chat Model              | LangChain DeepSeek Chat LLM       | Parse headlines, detect tone and extract insights | None                          | Tesla News and Sentiment Analyst| DeepSeek Notes: Outputs raw insight (tone, narrative themes, signal patterns).                 |
| Tesla News and Sentiment Analyst | LangChain Agent                   | Core AI agent processing all inputs, outputs JSON | When Executed by Another Workflow; RSS feeds; Memory; DeepSeek Chat Model | None                         | Sentiment Analyst Agent: Combines LLM outputs, generates final JSON sentiment, summary, headlines.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create `When Executed by Another Workflow` node**  
   - Type: Execute Workflow Trigger  
   - Configure inputs: Add two parameters named `message` (optional) and `sessionId` (required) to receive context from the parent workflow.

2. **Add five RSS Feed Read Tool nodes** for news aggregation:

   - **RSS - Google News Tesla-specific**  
     - URL: `https://news.google.com/rss/search?q=Tesla+OR+TSLA+stock&hl=en-US&gl=US&ceid=US:en`  
     - No additional options needed.

   - **RSS - Yahoo Finance TSLA**  
     - URL: `https://feeds.finance.yahoo.com/rss/2.0/headline?s=TSLA&region=US&lang=en-US`  
     - No additional options needed.

   - **RSS - Electrek Tesla**  
     - URL: `https://electrek.co/guides/tesla/feed/`  
     - No additional options needed.

   - **RSS - CleanTechnica Tesla Archives**  
     - URL: `https://cleantechnica.com/tag/tesla/feed/`  
     - No additional options needed.

   - **RSS - TeslaNorth**  
     - URL: `https://teslanorth.com/feed/`  
     - No additional options needed.

3. **Add `Simple Memory` node**  
   - Type: LangChain Memory Buffer Window  
   - Use default settings with no custom parameters.

4. **Add `DeepSeek Chat Model` node**  
   - Type: LangChain DeepSeek Chat LLM  
   - Credentials: Configure DeepSeek API credentials (see next step).  
   - No additional parameters needed.

5. **Add `Tesla News and Sentiment Analyst` node**  
   - Type: LangChain Agent  
   - Input Text: Map from workflow input `message`.  
   - System Message: Configure with detailed prompt describing role, data sources (all 5 RSS feeds), analysis steps, and strict JSON output format as in overview.  
   - Prompt Type: Define.  
   - Link all five RSS nodes as AI tools (ai_tool connections).  
   - Connect `Simple Memory` as AI memory source.  
   - Connect `DeepSeek Chat Model` as AI language model.  

6. **Connect nodes:**

   - Connect `When Executed by Another Workflow` output to `Tesla News and Sentiment Analyst` main input.  
   - Connect outputs of all five RSS nodes to `Tesla News and Sentiment Analyst` as AI tools.  
   - Connect `Simple Memory` to `Tesla News and Sentiment Analyst` as AI memory.  
   - Connect `DeepSeek Chat Model` to `Tesla News and Sentiment Analyst` as AI language model.

7. **Set up credentials:**

   - Add DeepSeek API credentials in n8n credentials section. Name it exactly `DeepSeek account`.  
   - No OAuth or other credentials required for RSS feeds.

8. **Triggering and Inputs:**

   - This workflow must be triggered from the parent Tesla Quant Trading AI Agent using `Execute Workflow` node.  
   - Required inputs:  
     - `message`: Optional string containing context or prompt.  
     - `sessionId`: String to maintain session context.

9. **Final output:**

   - The `Tesla News and Sentiment Analyst` node outputs a strict JSON object with keys:  
     - `sentiment`: One of `"bullish"`, `"bearish"`, or `"neutral"`  
     - `summary`: 2–3 sentence summary describing market sentiment drivers  
     - `topHeadlines`: Array of 3 to 5 key headline strings relevant for trading decisions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is not standalone and must be triggered by the Tesla Quant Trading AI Agent with appropriate inputs.                 | See: [Tesla Quant Trading AI Agent workflow](https://n8n.io/workflows/4092-tesla-quant-trading-ai-agent-using-telegram-gpt-41-main-interface/) |
| DeepSeek API key credentials required for operation; ensure valid and active key is configured in n8n credentials.                 | Credentials: DeepSeek API → `DeepSeek account`                                                     |
| RSS feeds require live internet access; best run in cloud-hosted n8n or locally with tunnel to access external feeds.              |                                                                                                    |
| Creator and licensing: Don Jayamaha (LinkedIn: https://linkedin.com/in/donjayamahajr), © 2025 Treasurium Capital Limited Company.   | Templates and more info: https://n8n.io/creators/don-the-gem-dealer/                                |
| Sticky notes in workflow provide detailed explanations on trigger, memory, news feeds, DeepSeek model, and sentiment analyst roles.| Review sticky notes in node editor for in-context guidance                                         |

---

**Disclaimer:** The provided text and workflow strictly comply with content policies and handle only legal, public data sources. No illegal or offensive content is included.

---