Generate Content Ideas with AI: Scrape Google & Facebook Posts to Slack

https://n8nworkflows.xyz/workflows/generate-content-ideas-with-ai--scrape-google---facebook-posts-to-slack-11152


# Generate Content Ideas with AI: Scrape Google & Facebook Posts to Slack

### 1. Workflow Overview

This workflow automates the generation of new content ideas by scraping trending posts and articles from Google News and Facebook, analyzing them with an AI language model, and then sending summarized content ideas to a Slack channel. It is designed for content marketers, social media managers, and strategists who want to streamline their research process for trending topics.

The workflow is logically divided into these main blocks:

- **1.1 Scheduled Trigger & Configuration:** Initiates the workflow on a weekly schedule and sets essential API tokens.
- **1.2 Data Scraping:** Uses Apify actors to scrape Google News articles and Facebook posts based on configured queries and URLs.
- **1.3 Data Extraction & Merging:** Extracts relevant data arrays from raw scraper outputs and merges them into a single dataset.
- **1.4 Data Limiting:** Restricts the dataset to a manageable number of items to optimize AI processing costs.
- **1.5 AI Content Analysis:** Uses an AI agent powered by OpenRouter to classify content themes and extract keywords.
- **1.6 Response Parsing & Message Formatting:** Parses AI output, formats it into Slack message payloads.
- **1.7 Slack Notification:** Sends the final content ideas as formatted messages to a configured Slack workspace.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Configuration

**Overview:**  
This block initiates the workflow every Monday at 09:00 JST and sets the necessary API tokens for Apify and OpenRouter services.

**Nodes Involved:**  
- Weekly Schedule Trigger  
- Workflow Configuration

**Node Details:**

- **Weekly Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow weekly on Mondays at 9 AM JST.  
  - Configuration: Set to trigger every week, Monday, hour=9.  
  - Connections: Outputs to Workflow Configuration node.  
  - Edge Cases: Timezone misconfiguration or n8n instance downtime could delay trigger.

- **Workflow Configuration**  
  - Type: Set  
  - Role: Stores user-defined API tokens required by Apify and OpenRouter nodes.  
  - Configuration: Two string variables `apifyToken` and `openRouterApiKey` must be set by the user before running.  
  - Connections: Outputs to Apify Facebook Scraper and Apify Google News Scraper nodes.  
  - Edge Cases: Missing or invalid tokens cause authentication failures downstream.

---

#### 2.2 Data Scraping

**Overview:**  
Scrapes content from Google News and Facebook using Apify actors with user-configured parameters.

**Nodes Involved:**  
- Apify Google News Scraper  
- Apify Facebook Scraper

**Node Details:**

- **Apify Google News Scraper**  
  - Type: Apify Node (Google Search Results Scraper)  
  - Role: Retrieves news articles based on query "Top News AI".  
  - Configuration:  
    - Queries: `"Top News AI"`  
    - Max results per query: 50  
    - Geographic location: US (`gl: "us"`)  
    - Search type: News (`tbm: "nws"`)  
  - Authentication: Uses Apify OAuth2 with token from Workflow Configuration.  
  - Connections: Outputs to Function: Extract Google Data node.  
  - Edge Cases: API timeouts, rate limits, or changes in Apify actor output structure.

- **Apify Facebook Scraper**  
  - Type: Apify Node (Facebook Posts Scraper)  
  - Role: Scrapes Facebook posts from a specific page URL.  
  - Configuration:  
    - Start URL: https://www.facebook.com/AIShift.Inc/  
    - Results limit: 50 posts  
    - Minimum likes: 50 (filters for popular posts)  
  - Authentication: Uses Apify OAuth2 with token from Workflow Configuration.  
  - Connections: Outputs to Function: Extract Facebook Data node.  
  - Edge Cases: Private or restricted pages, API limits, changes in Facebook page structure.

---

#### 2.3 Data Extraction & Merging

**Overview:**  
Extracts usable arrays of posts/articles from the raw Apify outputs and merges Google and Facebook datasets into one.

**Nodes Involved:**  
- Function: Extract Google Data  
- Function: Extract Facebook Data  
- Merge Data

**Node Details:**

- **Function: Extract Google Data**  
  - Type: Code (JavaScript)  
  - Role: Parses the Apify Google News Scraper output to extract an array of news articles.  
  - Logic: Checks multiple possible keys (`organicResults`, `results`, `items`, etc.) to find the data array.  
  - Connections: Outputs extracted array to Merge Data node.  
  - Edge Cases: Unexpected API output structures, empty results.

- **Function: Extract Facebook Data**  
  - Type: Code (JavaScript)  
  - Role: Parses the Apify Facebook Scraper output to extract an array of posts.  
  - Logic: Similar to the Google Data extraction, handles various array locations in JSON.  
  - Connections: Outputs extracted array to Merge Data node.  
  - Edge Cases: Empty datasets, malformed responses.

- **Merge Data**  
  - Type: Merge  
  - Role: Combines Google and Facebook extracted arrays into a single dataset.  
  - Connections: Outputs merged data to Function: Limit to 5 Items node.  
  - Edge Cases: Unequal lengths, empty inputs from one or both sources.

---

#### 2.4 Data Limiting

**Overview:**  
Restricts the merged dataset to the first 5 items to reduce downstream AI processing costs.

**Nodes Involved:**  
- Function: Limit to 5 Items

**Node Details:**

- **Function: Limit to 5 Items**  
  - Type: Code (JavaScript)  
  - Role: Takes all merged items, returns only the first 5.  
  - Logic: Uses array slicing on all input items.  
  - Connections: Outputs limited array to AI Agent node.  
  - Edge Cases: Less than 5 items available (returns all), empty input.

---

#### 2.5 AI Content Analysis

**Overview:**  
Uses an AI agent powered by OpenRouter to classify each content piece into a theme and extract up to 3 keywords.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model

**Node Details:**

- **OpenRouter Chat Model**  
  - Type: Language Model Chat (OpenRouter)  
  - Role: Provides AI language model capabilities for the AI Agent node.  
  - Configuration: Uses API key from Workflow Configuration implicitly.  
  - Connections: Outputs responses to AI Agent node’s language model input.  
  - Edge Cases: API key invalid, rate limits, model downtime.

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Processes each content item by analyzing its title, summary, author, and URL, then outputs a JSON with `theme` and `keywords`.  
  - Configuration:  
    - Prompt includes instructions to select a theme from a fixed list and extract keywords.  
    - Output strictly formatted as JSON.  
  - Inputs: Receives up to 5 items from the limiting function.  
  - Outputs: Sends AI-generated JSON to Function: Parse LLM Response.  
  - Edge Cases: AI output not valid JSON, timeout, prompt misconfiguration.

---

#### 2.6 Response Parsing & Message Formatting

**Overview:**  
Parses AI JSON responses and formats each analyzed content item into a Slack message payload.

**Nodes Involved:**  
- Function: Parse LLM Response  
- Function: Create Slack Message

**Node Details:**

- **Function: Parse LLM Response**  
  - Type: Code (JavaScript)  
  - Role: Parses AI JSON output to extract theme and keywords, handles errors gracefully.  
  - Logic: Attempts JSON.parse, falls back to defaults if parsing fails.  
  - Adds `llmTheme` and `keywordsArray` to original item data.  
  - Connections: Outputs enriched data to Function: Create Slack Message.  
  - Edge Cases: Parsing errors, missing fields, unexpected AI outputs.

- **Function: Create Slack Message**  
  - Type: Code (JavaScript)  
  - Role: Formats each item into a Slack message block with theme, keywords, source URL, and summary.  
  - Logic: Builds markdown-formatted message string.  
  - Connections: Sends message JSON to Slack Post Content Ideas node.  
  - Edge Cases: Missing URLs or summaries handled with fallback text.

---

#### 2.7 Slack Notification

**Overview:**  
Sends the formatted content ideas as Slack messages to a configured Slack workspace.

**Nodes Involved:**  
- Slack Post Content Ideas

**Node Details:**

- **Slack Post Content Ideas**  
  - Type: Slack node  
  - Role: Posts messages to Slack channel or user.  
  - Configuration:  
    - Message text sourced from previous node’s `message` field.  
    - Uses OAuth2 Slack credentials.  
    - Markdown enabled.  
  - Connections: Terminal node.  
  - Edge Cases: Slack API rate limits, invalid credentials, message formatting issues.

---

### 3. Summary Table

| Node Name                    | Node Type                  | Functional Role                          | Input Node(s)                  | Output Node(s)                         | Sticky Note                                                                                                                         |
|------------------------------|----------------------------|----------------------------------------|-------------------------------|--------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Weekly Schedule Trigger       | Schedule Trigger           | Triggers workflow weekly                | -                             | Workflow Configuration                | Triggers every Monday at 09:00 JST                                                                                                 |
| Workflow Configuration        | Set                        | Stores API tokens                       | Weekly Schedule Trigger        | Apify Facebook Scraper, Apify Google News Scraper | REQUIRED: Set your Apify API Token and OpenRouter API Key here.                                                                     |
| Apify Google News Scraper     | Apify                      | Scrapes Google News                     | Workflow Configuration         | Function: Extract Google Data         |                                                                                                                                    |
| Apify Facebook Scraper        | Apify                      | Scrapes Facebook posts                  | Workflow Configuration         | Function: Extract Facebook Data       |                                                                                                                                    |
| Function: Extract Google Data | Code                       | Extracts articles array from Google output | Apify Google News Scraper      | Merge Data                          |                                                                                                                                    |
| Function: Extract Facebook Data | Code                     | Extracts posts array from Facebook output | Apify Facebook Scraper         | Merge Data                          |                                                                                                                                    |
| Merge Data                   | Merge                      | Combines Google and Facebook data      | Function: Extract Google Data, Function: Extract Facebook Data | Function: Limit to 5 Items           | Merges results from Google and Facebook.                                                                                           |
| Function: Limit to 5 Items    | Code                       | Limits dataset size to 5                | Merge Data                    | AI Agent                            | Limits the dataset to 5 items to save AI tokens.                                                                                   |
| OpenRouter Chat Model         | Language Model Chat         | Provides AI model for analysis          | -                            | AI Agent                            |                                                                                                                                    |
| AI Agent                     | LangChain Agent            | Classifies content theme, extracts keywords | Function: Limit to 5 Items, OpenRouter Chat Model | Function: Parse LLM Response          |                                                                                                                                    |
| Function: Parse LLM Response  | Code                       | Parses AI JSON output                   | AI Agent                      | Function: Create Slack Message       | Parses the JSON response from the LLM.                                                                                            |
| Function: Create Slack Message | Code                      | Formats Slack message payload           | Function: Parse LLM Response   | Slack Post Content Ideas             | Format the analysis results into a Slack message.                                                                                  |
| Slack Post Content Ideas      | Slack                      | Sends message to Slack                  | Function: Create Slack Message | -                                  |                                                                                                                                    |
| Sticky Note                  | Sticky Note                 | Workflow description and requirements  | -                             | -                                  | ![Banner Image](https://example.com/replace-with-your-image.png) This workflow automates finding new content ideas by scraping trending news and social media posts, analyzing with AI, and delivering a summary to Slack. Perfect for marketers and strategists. Requires n8n self-hosted, Apify account, OpenRouter API key, and Slack account. |
| Sticky Note1                 | Sticky Note                 | Setup instructions                     | -                             | -                                  | How to set up: Configure credentials in Workflow Configuration; connect Slack; adjust Apify search queries and Facebook URLs; customize AI prompt if needed.                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add "Weekly Schedule Trigger" node:**  
   - Set to trigger every Monday at 09:00 JST (set `weeks` interval, trigger at day 1, hour 9).  
   - Connect its output to the next node.

3. **Add "Workflow Configuration" (Set) node:**  
   - Create two string fields:  
     - `apifyToken` (set to your Apify API token)  
     - `openRouterApiKey` (set to your OpenRouter API key)  
   - Connect input from the Schedule Trigger node.  
   - Connect outputs to both Apify scraper nodes.

4. **Add "Apify Facebook Scraper" node:**  
   - Select Apify Facebook Posts Scraper actor (ID: `KoJrdxJCTtpon81KY`).  
   - Set authentication to use Apify OAuth2 credentials with your token.  
   - Configure `customBody` JSON with your target Facebook page URL(s), e.g.:  
     ```json
     {
       "startUrls": [{"url": "https://www.facebook.com/AIShift.Inc/"}],
       "resultsLimit": 50,
       "minimumLikes": 50
     }
     ```  
   - Connect input from Workflow Configuration.

5. **Add "Apify Google News Scraper" node:**  
   - Select Apify Google Search Results Scraper actor (ID: `nFJndFXA5zjCTuudP`).  
   - Use same Apify OAuth2 credentials.  
   - Configure `customBody` JSON with your search parameters, e.g.:  
     ```json
     {
       "queries": "Top News AI",
       "maxResultsPerQuery": 50,
       "gl": "us",
       "tbm": "nws"
     }
     ```  
   - Connect input from Workflow Configuration.

6. **Add "Function: Extract Google Data" node (Code):**  
   - JavaScript code to parse Apify Google output and extract article array.  
   - Connect input from Apify Google News Scraper.  
   - Connect output to Merge Data node.

7. **Add "Function: Extract Facebook Data" node (Code):**  
   - JavaScript code to parse Apify Facebook output and extract posts array.  
   - Connect input from Apify Facebook Scraper.  
   - Connect output to Merge Data node.

8. **Add "Merge Data" node:**  
   - Merge mode: default (combine inputs as array).  
   - Connect inputs from both extraction functions.  
   - Connect output to Function: Limit to 5 Items.

9. **Add "Function: Limit to 5 Items" node (Code):**  
   - JavaScript code to slice first 5 items from merged array.  
   - Connect input from Merge Data node.  
   - Connect output to AI Agent node.

10. **Add "OpenRouter Chat Model" node:**  
    - Select OpenRouter LM Chat node.  
    - Ensure API key is configured in credentials.  
    - Connect output to AI Agent node’s language model input.

11. **Add "AI Agent" node (LangChain Agent):**  
    - Configure prompt to analyze content items: classify theme and extract keywords.  
    - Use input expressions to populate article details (title, summary, author, URL).  
    - Output should be JSON with `theme` and `keywords`.  
    - Connect input from Function: Limit to 5 Items and OpenRouter Chat Model node.  
    - Connect output to Function: Parse LLM Response.

12. **Add "Function: Parse LLM Response" node (Code):**  
    - JavaScript to parse AI JSON output safely and add `llmTheme` and `keywordsArray` fields to items.  
    - Connect input from AI Agent.  
    - Connect output to Function: Create Slack Message.

13. **Add "Function: Create Slack Message" node (Code):**  
    - JavaScript to format each item’s data into a Slack markdown message string.  
    - Connect input from Parse LLM Response.  
    - Connect output to Slack Post Content Ideas node.

14. **Add "Slack Post Content Ideas" node:**  
    - Configure with Slack OAuth2 credentials.  
    - Set the message text to be `{{ $json.message }}` with markdown enabled.  
    - Connect input from Function: Create Slack Message.  
    - This node sends the final content idea messages to the Slack workspace.

15. **Add Sticky Notes as desired to document the workflow inline.**

16. **Verify all credentials: Apify OAuth2, OpenRouter API Key, Slack OAuth2.**

17. **Test the workflow manually before relying on scheduled triggers.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow requires n8n to be self-hosted because of the `@apify/n8n-nodes-apify` node dependency.                             | n8n documentation on self-hosted nodes: https://docs.n8n.io/integrations/hosted-vs-self-hosted/                      |
| OpenRouter API key is used for AI processing but can be replaced with other supported LLM providers like OpenAI or Anthropic.   | OpenRouter docs: https://docs.openrouter.ai/                                                                        |
| Apify actor IDs used: Google Search Results Scraper (`nFJndFXA5zjCTuudP`), Facebook Posts Scraper (`KoJrdxJCTtpon81KY`).         | Apify actor marketplace: https://apify.com/actors                                                                   |
| Slack OAuth2 credentials must have permissions to post messages in the target Slack workspace/channel.                            | Slack API docs: https://api.slack.com/authentication/oauth-v2                                                      |
| Adjust Apify scraper queries and URLs to suit your target content topics and sources.                                            |                                                                                                                    |
| AI prompt defines six fixed themes: Market Trends/Stats, Tech/Product, Legal/Regulation, Strategy, Marketing/PR, Society/Culture.| Customize them to your use case by editing the AI Agent prompt.                                                     |
| To reduce AI token usage, the dataset is limited to 5 items per run; adjust if you need more or fewer items analyzed.            |                                                                                                                    |

---

**Disclaimer:**  
The provided text is sourced exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.