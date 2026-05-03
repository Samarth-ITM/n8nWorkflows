Daily Business News Briefings with NewsAPI & GPT-4 Insights to Slack

https://n8nworkflows.xyz/workflows/daily-business-news-briefings-with-newsapi---gpt-4-insights-to-slack-6273


# Daily Business News Briefings with NewsAPI & GPT-4 Insights to Slack

### 1. Workflow Overview

This workflow automates the delivery of daily business news summaries and insights directly into Slack channels. It leverages the NewsAPI to fetch the latest headlines filtered by country, category, and keyword query, and then uses GPT-4 to analyze, summarize, and prioritize the top 10 most impactful articles. Each article is tagged with sentiment indicators (Opportunity, Risk, Neutral), and an overall market or trend summary is generated. Finally, the enriched insights are posted automatically to a configured Slack user or channel.

The workflow is organized into the following logical blocks:

- **1.1 Schedule Trigger:** Defines the automation frequency.
- **1.2 User Configuration Input:** Sets the news query parameters (country, category, keyword).
- **1.3 Fetch News Articles:** Retrieves news data from NewsAPI.
- **1.4 Data Preparation and Merging:** Combines fetched news with user configuration for context.
- **1.5 (Optional) Limit to Top 10 Articles:** Filters the articles count before GPT processing (disabled by default).
- **1.6 GPT Business Insights Generation:** Summarizes and analyzes news articles using GPT-4.
- **1.7 Post to Slack:** Sends the final insights message to Slack with OAuth2 authentication.
- **1.8 Documentation and Guidance:** Sticky notes providing explanations, setup instructions, and helpful links.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:** Initiates the workflow execution on a recurring schedule (default: daily).
- **Nodes Involved:** `Schedule Trigger`
- **Node Details:**
  - Type: Schedule Trigger
  - Configuration: Runs based on an interval rule (default triggers daily; users can customize).
  - Inputs: None (trigger node).
  - Outputs: Sends trigger signal to `Set User Config (Country, Category, Query)`.
  - Edge Cases: Misconfigured schedule may cause missed or excessive runs.
  - Notes: Controls overall workflow periodicity.

#### 1.2 User Configuration Input

- **Overview:** Sets the parameters for news retrieval — country, category, and keyword query.
- **Nodes Involved:** `Set User Config (Country, Category, Query)`
- **Node Details:**
  - Type: Set Node (Data Initialization)
  - Configuration: Defines JSON with properties:
    - `country`: "us"
    - `category`: "technology"
    - `query`: "openai"
  - Inputs: Trigger from `Schedule Trigger`.
  - Outputs: Passes config to `Fetch News Articles`.
  - Edge Cases: Invalid country or category codes may yield empty or error responses from NewsAPI.
  - Notes: This node controls the data scope for the entire workflow.

#### 1.3 Fetch News Articles

- **Overview:** Calls NewsAPI to retrieve top headlines matching the configured filters.
- **Nodes Involved:** `Fetch News Articles`
- **Node Details:**
  - Type: HTTP Request
  - Configuration:
    - Method: GET
    - URL templated with expressions to insert country, category, and query.
    - Header: `X-Api-Key` with NewsAPI key (must be set in node).
  - Inputs: Receives config JSON.
  - Outputs: JSON response containing news articles array.
  - Edge Cases:
    - API key invalid or expired -> authentication failure.
    - Rate limiting by NewsAPI.
    - No articles found for query.
  - Notes: Requires valid NewsAPI key; max 100 articles per request.

#### 1.4 Data Preparation and Merging

- **Overview:** Combines user config data with fetched articles to provide GPT with full context.
- **Nodes Involved:** `Inject Config Data`, `Merge Config with Articles`
- **Node Details:**
  - `Inject Config Data`
    - Type: Set Node
    - Purpose: Provides a fixed config object (country: "us", category: "technology", query: "AI") for merging.
    - Note: This node uses slightly different query ("AI") than initial config; likely a fallback or example.
  - `Merge Config with Articles`
    - Type: Merge Node
    - Mode: Combine (combines two inputs into one JSON object).
    - Inputs: One from `Fetch News Articles`, one from `Inject Config Data`.
    - Outputs: Combined JSON for GPT input.
  - Edge Cases:
    - Mismatched or inconsistent config data may confuse GPT prompt.
    - Disabled Limit node affects data volume here.
  - Notes: Provides GPT with both raw news and query metadata.

#### 1.5 (Optional) Limit to Top 10 Articles (Disabled)

- **Overview:** Restricts articles to top 10 before passing to GPT for analysis.
- **Nodes Involved:** `Limit to Top 10 Trends` (disabled)
- **Node Details:**
  - Type: Code Node (JavaScript)
  - Configuration: Returns only first 10 items.
  - Inputs: Receives merged articles.
  - Outputs: Passes limited array to `Inject Config Data`.
  - Edge Cases: Disabled by default; if enabled, ensures GPT only processes a manageable number of articles.
  - Notes: Users can enable or remove this node to control data volume and GPT cost.

#### 1.6 GPT Business Insights Generation

- **Overview:** Uses GPT-4 to analyze, summarize, and classify top news articles with business insights.
- **Nodes Involved:** `Generate Business Insights (GPT)`
- **Node Details:**
  - Type: LangChain OpenAI Node (Chat Completion)
  - Model: GPT-4.1-mini (custom variant of GPT-4)
  - Messages:
    - User prompt includes:
      - Business analyst persona.
      - Context of user query, category, country.
      - Instruction to select top 10 impactful articles.
      - Summarize each article in 2-3 lines.
      - Tag each article with risk/opportunity/neutral emoji.
      - Provide a final summary paragraph.
    - System prompt sets professional analyst tone.
  - Inputs: Combined JSON of articles and config.
  - Outputs: GPT-generated insights text.
  - Credentials: OpenAI API key required.
  - Edge Cases:
    - API rate limits or key issues.
    - Long inputs may exceed token limits.
    - GPT misinterpretation if input data malformed.
  - Notes: Customizable prompt controls output tone and depth.

#### 1.7 Post to Slack

- **Overview:** Sends generated news insights to a Slack user via OAuth2 authenticated Slack node.
- **Nodes Involved:** `Post to Slack`
- **Node Details:**
  - Type: Slack Node
  - Configuration:
    - Message text set by GPT output expression.
    - User recipient specified (user ID "U096VCG525P", cached as "james").
    - Authentication via OAuth2 Slack credential.
  - Inputs: GPT insights text.
  - Outputs: None (terminal node).
  - Edge Cases:
    - Slack API auth failures.
    - User ID invalid or user unreachable.
    - Network issues.
  - Notes: Message format can be customized; requires Slack OAuth2 app setup.

#### 1.8 Documentation and Guidance

- **Overview:** Collection of sticky notes providing workflow explanation, setup instructions, and external resource links.
- **Nodes Involved:** `Sticky Note`, `Sticky Note1` to `Sticky Note8`
- **Node Details:**
  - Provide detailed descriptions of each step.
  - Include setup instructions for API keys and credentials.
  - Link to relevant documentation pages and a demo video.
  - Offer community support links.
  - No input/output connections; purely informational.

---

### 3. Summary Table

| Node Name                         | Node Type                    | Functional Role                          | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                                          |
|----------------------------------|------------------------------|----------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                 | Schedule Trigger             | Initiates workflow on schedule         | None                             | Set User Config (Country, Category, Query) | Explains scheduling and frequency control                                                                              |
| Set User Config (Country, Category, Query) | Set                          | Sets news API parameters                | Schedule Trigger                 | Fetch News Articles              | Explains config of country, category, and keyword                                                                      |
| Fetch News Articles              | HTTP Request                | Retrieves news from NewsAPI             | Set User Config                  | Merge Config with Articles, Limit to Top 10 Trends | Explains HTTP request setup and API key requirements                                                                    |
| Limit to Top 10 Trends           | Code (disabled)             | Optional filter of articles count       | Fetch News Articles              | Inject Config Data               | Disabled by default; limits articles to top 10                                                                          |
| Inject Config Data               | Set                          | Provides static config data for GPT     | Limit to Top 10 Trends           | Merge Config with Articles       | Details config injection to aid GPT context                                                                             |
| Merge Config with Articles       | Merge                        | Combines news articles and config       | Fetch News Articles, Inject Config Data | Generate Business Insights (GPT) | Explains data merging before GPT processing                                                                              |
| Generate Business Insights (GPT) | LangChain OpenAI Chat Node  | Summarizes, prioritizes, and tags news | Merge Config with Articles       | Post to Slack                   | Describes GPT prompt design and analysis logic                                                                          |
| Post to Slack                   | Slack                       | Sends GPT insights to Slack user        | Generate Business Insights (GPT) | None                           | Details Slack message sending and OAuth2 setup                                                                          |
| Sticky Note                    | Sticky Note                 | Workflow overview and purpose           | None                             | None                           | General description of workflow purpose                                                                                 |
| Sticky Note1                   | Sticky Note                 | Step-by-step workflow breakdown         | None                             | None                           | Stepwise function explanation                                                                                           |
| Sticky Note2                   | Sticky Note                 | Explains config injection settings      | None                             | None                           | Configuration importance for news scope                                                                                 |
| Sticky Note3                   | Sticky Note                 | Details on fetching news via HTTP       | None                             | None                           | NewsAPI request explanation and API key instructions                                                                    |
| Sticky Note4                   | Sticky Note                 | Data preparation and merging context    | None                             | None                           | Merging articles with config explanation                                                                                |
| Sticky Note5                   | Sticky Note                 | GPT business summary details             | None                             | None                           | GPT prompt and output format explanation                                                                                 |
| Sticky Note6                   | Sticky Note                 | Slack post explanation                   | None                             | None                           | Slack node usage and credential notes                                                                                    |
| Sticky Note7                   | Sticky Note                 | Schedule trigger overview                | None                             | None                           | Scheduling control explanation                                                                                          |
| Sticky Note8                   | Sticky Note                 | Demo video link                          | None                             | None                           | Link to setup video guide                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" node**  
   - Type: Schedule Trigger  
   - Set desired interval (default daily).  
   - No inputs; connects output to next node.

2. **Create "Set User Config (Country, Category, Query)" node**  
   - Type: Set  
   - Mode: Raw JSON  
   - JSON content example:  
     ```json
     {
       "country": "us",
       "category": "technology",
       "query": "openai"
     }
     ```  
   - Connect input from Schedule Trigger.

3. **Create "Fetch News Articles" node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL:  
     ```  
     https://newsapi.org/v2/top-headlines?country={{ $json.country }}&category={{ $json.category }}&q={{ $json.query }}  
     ```  
   - Header Parameters:  
     - Name: `X-Api-Key`  
     - Value: Your NewsAPI key (set in node)  
   - Connect input from Set User Config.

4. **Create "Inject Config Data" node**  
   - Type: Set  
   - Mode: Raw JSON  
   - JSON example:  
     ```json
     {
       "country": "us",
       "category": "technology",
       "query": "AI"
     }
     ```  
   - Connect input from (optionally) Limit to Top 10 Trends or directly from Fetch News Articles.

5. **Create "Limit to Top 10 Trends" node (optional)**  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     const topTen = items.slice(0, 10);
     return topTen;
     ```  
   - Initially disabled; enable if limiting article count is desired.  
   - Connect input from Fetch News Articles; output to Inject Config Data.

6. **Create "Merge Config with Articles" node**  
   - Type: Merge  
   - Mode: Combine (combineAll)  
   - Connect two inputs:  
     - Main input from Fetch News Articles (or Limit node if used)  
     - Second input from Inject Config Data  
   - Output to GPT node.

7. **Create "Generate Business Insights (GPT)" node**  
   - Type: LangChain OpenAI Chat Completion  
   - Model ID: `gpt-4.1-mini` (or GPT-4 variant)  
   - Messages configuration:  
     - User message with prompt that includes:  
       - User query, category, country variables  
       - Instruction to pick top 10 articles based on impact, summarize, tag sentiment  
       - Final paragraph summary  
       - Embed JSON.stringify of merged input data  
     - System message: "You are a professional trend analyst summarizing news with actionable business insights."  
   - Credentials: OpenAI API key  
   - Connect input from Merge node.

8. **Create "Post to Slack" node**  
   - Type: Slack  
   - Authentication: OAuth2 (configure Slack OAuth2 credentials)  
   - User: Select user by ID (e.g., "U096VCG525P") or channel  
   - Text: Set expression to output of GPT node: `{{ $json.choices[0].message.content }}`  
   - Connect input from GPT node.

9. **Connect all nodes in order:**  
   Schedule Trigger → Set User Config → Fetch News Articles → (optional Limit) → Inject Config Data → Merge Config with Articles → Generate Business Insights (GPT) → Post to Slack.

10. **Add Sticky Notes for documentation (optional):**  
    - Provide descriptions at key points such as Schedule, Config, Fetch, GPT, Slack, using the content from the original sticky notes.

11. **Activate the workflow** and test.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow fetches and summarizes top business news daily, delivering actionable insights directly to Slack.     | Overview                                                                                        |
| Requires NewsAPI Key, OpenAI API Key, and Slack OAuth2 credentials properly configured.                         | Setup requirements                                                                              |
| Customize country, category, and query in the "Set User Config" node to tailor news focus.                      | Configuration customization                                                                     |
| Optional "Limit to Top 10 Trends" node can be enabled to cap article count before GPT analysis.                 | Performance and cost control                                                                    |
| GPT prompt can be edited to change tone, add recommendations, or adjust number of articles summarized.         | GPT customization                                                                              |
| Slack node sends message to specified user; can be modified for channels or different recipients.              | Slack integration                                                                              |
| Demo video guide available: [Loom Setup Guide](https://www.loom.com/share/970a3fba1ed44352a2194f1ef6a8dc45)     | Setup video                                                                                     |
| Community support offered on [Discord](https://discord.com/invite/XPKeKXeB7d) and [n8n Forum](https://community.n8n.io/) | Support channels                                                                               |
| NewsAPI HTTP Request docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/      | Node documentation                                                                             |
| OpenAI Chat Model docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.lmchatopenai/ | Node documentation                                                                            |
| Slack node docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/                        | Node documentation                                                                             |

---

This completes the comprehensive technical reference document for the "Daily Business News Briefings with NewsAPI & GPT-4 Insights to Slack" workflow. It is designed for both advanced users and AI agents to understand, reproduce, and customize the workflow effectively.