Smart RSS Feed Monitoring with AI Filtering, Baserow Storage, and Slack Alerts

https://n8nworkflows.xyz/workflows/smart-rss-feed-monitoring-with-ai-filtering--baserow-storage--and-slack-alerts-6389


# Smart RSS Feed Monitoring with AI Filtering, Baserow Storage, and Slack Alerts

### 1. Workflow Overview

This workflow, titled **"Smart RSS Feed Monitoring with AI Filtering, Baserow Storage, and Slack Alerts"**, automates the monitoring of multiple RSS feeds, intelligently filters new articles using AI, stores seen articles in a Baserow database to avoid duplicates, and sends notifications of new articles to a Slack channel.

**Target Use Cases:**
- News agencies or content curators who want to monitor multiple RSS feeds for fresh content.
- Teams or individuals who want to automate alerting on new articles without manual checking.
- Scenarios requiring deduplication of articles by tracking already processed entries.

**Logical Blocks:**

- **1.1 Input Reception and RSS Feed URL Retrieval:** Starts the workflow manually and reads a list of RSS feed URLs from a Baserow database.
- **1.2 RSS Feed Download and Parsing:** Retrieves the raw RSS XML from each URL, converts XML to JSON, and prepares article data.
- **1.3 Seen Articles Retrieval:** Fetches a list of already processed article identifiers from Baserow.
- **1.4 AI-Powered Filtering:** Uses an OpenAI-based AI Agent to compare new articles against seen articles and returns only new, unprocessed articles.
- **1.5 Data Cleaning and Storage:** Parses AI output into usable JSON, saves newly identified articles to Baserow to mark them as seen.
- **1.6 Notification Dispatch:** Sends formatted messages with new articles to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and RSS Feed URL Retrieval

- **Overview:** Starts the workflow on manual trigger, then pulls all RSS feed URLs stored in a Baserow table.
- **Nodes Involved:** `Click to Start`, `Read Rss Link`, `Split Out`, `Sticky Note1`, `Sticky Note2`
- **Node Details:**

  - **Click to Start**
    - Type: Manual Trigger
    - Role: Initiates workflow execution manually.
    - Inputs: None
    - Outputs: Single trigger event to `Read Rss Link`.
    - Failure Modes: None inherent.
  
  - **Read Rss Link**
    - Type: Baserow node (read operation)
    - Role: Reads all rows from the Baserow table storing RSS feed URLs.
    - Configuration: Reads from `Database ID=243547`, `Table ID=579115`, returns all rows.
    - Inputs: Trigger from `Click to Start`.
    - Outputs: Array of rows containing RSS feed URLs.
    - Potential Failures: API key invalid, network failure, table or database ID mismatch.
  
  - **Split Out**
    - Type: Split Out
    - Role: Splits the array of rows into individual items to process each RSS feed URL independently.
    - Configuration: Splitting by the field `rssLink`.
    - Inputs: Output from `Read Rss Link`.
    - Outputs: Individual items, each containing a single RSS feed URL.
    - Potential Failures: If `rssLink` field missing or empty.
  
- **Sticky Notes:**
  - "Retrieves all RSS feed URLs from a Baserow table."
  - "Takes the output from \"Read Rss Link\" and splits each row into a separate item."
  
---

#### 1.2 RSS Feed Download and Parsing

- **Overview:** For each RSS feed URL, the workflow fetches the raw XML content, converts it into JSON, and prepares article data for further processing.
- **Nodes Involved:** `Fetch HTML`, `XML Converter`, `Sticky Note3`, `Sticky Note4`
- **Node Details:**

  - **Fetch HTML**
    - Type: HTTP Request
    - Role: Fetches raw XML content from the RSS feed URL.
    - Configuration: URL set dynamically from the `rssLink` property.
    - Inputs: Individual RSS feed URL from `Split Out`.
    - Outputs: Raw XML string in response body.
    - Potential Failures: Network issues, invalid URL, timeout, HTTP errors.
  
  - **XML Converter**
    - Type: XML
    - Role: Parses raw XML string into JSON object.
    - Configuration: Default options.
    - Inputs: Raw XML from `Fetch HTML`.
    - Outputs: JSON representation of RSS feed.
    - Potential Failures: Malformed XML, empty response.
  
- **Sticky Notes:**
  - "Fetches the raw XML content from each RSS feed URL."
  - "Parses the raw XML content into a structured JSON object."

---

#### 1.3 Seen Articles Retrieval and Data Preparation

- **Overview:** Retrieves the list of previously processed articles from Baserow and prepares the data structure combining newly fetched articles and seen articles for AI filtering.
- **Nodes Involved:** `Get Seen Products`, `Edit data structure`, `Sticky Note5`, `Sticky Note6`
- **Node Details:**

  - **Get Seen Products**
    - Type: Baserow node (read operation)
    - Role: Fetches all entries from the Baserow table holding links of already seen articles.
    - Configuration: Reads from `Database ID=243547`, `Table ID=578089`, returns all rows.
    - Inputs: JSON from `XML Converter` (though connection is linear, execution dependencies ensure correct order).
    - Outputs: Array of "seen" article links (field `Nom`).
    - Potential Failures: API key issues, network errors, table/database misconfiguration.
  
  - **Edit data structure**
    - Type: Code node (JavaScript)
    - Role: Transforms raw RSS feed items into a simplified article array and gathers already processed GUIDs.
    - Configuration: Custom JS code:
      - Extracts `title`, `link`, `content` from parsed XML JSON.
      - Collects `alreadyProcessedGuids` from `Get Seen Products` output.
      - Returns a combined JSON object with `input` (articles) and `alreadyProcessedGuids`.
    - Inputs: Output from `Get Seen Products` and `XML Converter`.
    - Outputs: Structured object with new articles and seen GUIDs.
    - Potential Failures: Parsing errors, empty or malformed inputs.
  
- **Sticky Notes:**
  - "Retrieves all previously seen article links from a Baserow table."
  - "Prepares data for the AI Agent by structuring new articles and previously seen article GUIDs."

---

#### 1.4 AI-Powered Filtering

- **Overview:** Uses an AI Agent powered by OpenAI GPT-4o-mini model to filter out articles that have already been processed, returning only new articles.
- **Nodes Involved:** `AI Agent`, `OpenAI Chat Model`, `Simple Memory`, `Structured Output Parser`, `Sticky Note7`, `Sticky Note8`, `Sticky Note9`, `Sticky Note13`
- **Node Details:**

  - **AI Agent**
    - Type: Langchain AI Agent node
    - Role: Receives articles and already processed GUIDs, loops over the input array, and outputs only new articles.
    - Configuration:
      - Uses a detailed prompt instructing the AI to return a JSON array of new articles not present in `alreadyProcessedGuids`.
      - Output must be raw JSON formatted strictly as array.
      - Connected to `OpenAI Chat Model` for language model and to `Simple Memory` for session context keyed by `alreadyProcessedGuids`.
    - Inputs: Data from `Edit data structure`.
    - Outputs: Raw JSON string of filtered new articles.
    - Potential Failures: AI API quota exceeded, invalid response format, timeout, prompt interpretation errors.
  
  - **OpenAI Chat Model**
    - Type: Langchain OpenAI LM Chat node
    - Role: Provides GPT-4o-mini model for AI Agent.
    - Configuration: Model set to `gpt-4o-mini`.
    - Inputs: From AI Agent node.
    - Outputs: AI-generated text response.
    - Potential Failures: Authentication errors, API limits, service downtime.
  
  - **Simple Memory**
    - Type: Langchain Memory Buffer Window
    - Role: Maintains conversation context using `alreadyProcessedGuids` as session key.
    - Configuration: Context window length 50 messages.
    - Inputs: Feeds context to AI Agent.
    - Outputs: Context-aware AI interaction.
    - Potential Failures: Memory overflow, state loss.
  
  - **Structured Output Parser**
    - Type: Langchain Output Parser Structured
    - Role: Parses AI output JSON string into structured JSON objects.
    - Configuration: JSON schema example with `title`, `link`, `guid`, `content`.
    - Inputs: AI Agent output.
    - Outputs: Structured JSON array usable downstream.
    - Potential Failures: Parsing errors if AI output malformed.
  
- **Sticky Notes:**
  - "Filters new articles using AI, returning only those not previously seen."
  - "Ensures the AI Agent's output conforms to a predefined JSON structure."
  - "Parses the AI Agent's JSON string output."
  - "Provides the AI model for the agent."

---

#### 1.5 Data Cleaning and Storage

- **Overview:** Converts the AI Agent's JSON output into discrete items and saves each new article's link in Baserow to mark them as seen.
- **Nodes Involved:** `Clean JSON`, `Save seen products`, `Sticky Note10`
- **Node Details:**

  - **Clean JSON**
    - Type: Code node (JavaScript)
    - Role: Parses the raw JSON string output from AI Agent and returns each article as a separate item.
    - Configuration: Uses `JSON.parse` on AI output, maps articles to individual JSON items.
    - Inputs: AI Agent output.
    - Outputs: Separate n8n JSON items, each representing one new article.
    - Potential Failures: Parsing failure if AI output is invalid JSON.
  
  - **Save seen products**
    - Type: Baserow node (create operation)
    - Role: Saves new article links into Baserow "seen products" table to avoid reprocessing.
    - Configuration: Writes to `Database ID=243547`, `Table ID=578089`, field `Nom` set to article `link`.
    - Inputs: Processed individual new articles from `Clean JSON`.
    - Outputs: Confirmation of storage.
    - Potential Failures: API errors, field missing, duplicates handling.
  
- **Sticky Notes:**
  - "Saves the links of newly processed articles to the Baserow 'seen products' table."

---

#### 1.6 Notification Dispatch

- **Overview:** Sends a detailed notification message to a Slack channel for each new article.
- **Nodes Involved:** `Slack`, `Sticky Note11`
- **Node Details:**

  - **Slack**
    - Type: Slack node
    - Role: Sends a message to a specified Slack channel with article details.
    - Configuration:
      - Channel set via ID `C091X7ZNW4V`.
      - Message type: Block message.
      - Text includes article title, content, and link from `Clean JSON` node.
      - Uses Slack API OAuth2 or Webhook credential.
    - Inputs: New article items from `Save seen products`.
    - Outputs: Slack API response.
    - Potential Failures: Invalid credentials, rate limits, channel ID errors, network issues.
  
- **Sticky Notes:**
  - "Sends a notification with new article details to a Slack channel."

---

### 3. Summary Table

| Node Name           | Node Type                        | Functional Role                                | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                   |
|---------------------|---------------------------------|-----------------------------------------------|-----------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Click to Start      | Manual Trigger                  | Starts workflow manually                       | None                  | Read Rss Link           | Triggers the workflow.                                                                       |
| Read Rss Link       | Baserow (read)                  | Retrieves all RSS feed URLs from Baserow      | Click to Start         | Split Out               | Retrieves all RSS feed URLs from a Baserow table.                                           |
| Split Out           | Split Out                      | Splits RSS URLs into individual items         | Read Rss Link          | Fetch HTML              | Takes the output from "Read Rss Link" and splits each row into a separate item.              |
| Fetch HTML          | HTTP Request                   | Fetches raw RSS XML from each URL              | Split Out              | XML Converter           | Fetches the raw XML content from each RSS feed URL.                                         |
| XML Converter       | XML                           | Parses raw XML into JSON                        | Fetch HTML             | Get Seen Products       | Parses the raw XML content into a structured JSON object.                                   |
| Get Seen Products   | Baserow (read)                  | Retrieves already processed article links      | XML Converter          | Edit data structure     | Retrieves all previously seen article links from a Baserow table.                           |
| Edit data structure | Code                           | Prepares combined input and seen articles data | Get Seen Products      | AI Agent                | Prepares data for the AI Agent by structuring new articles and previously seen article GUIDs.|
| AI Agent            | Langchain AI Agent             | Filters new articles by comparing with seen   | Edit data structure    | Clean JSON              | Filters new articles using AI, returning only those not previously seen.                     |
| OpenAI Chat Model   | Langchain OpenAI LM Chat       | Provides GPT-4o-mini model for AI Agent        | AI Agent (ai_languageModel) | AI Agent           | Provides the AI model for the agent.                                                        |
| Simple Memory       | Langchain Memory Buffer Window | Maintains session context for AI Agent         | AI Agent (ai_memory)   | AI Agent                | Provides the AI model for the agent.                                                        |
| Structured Output Parser | Langchain Output Parser Structured | Parses AI output JSON string                   | AI Agent               | -                       | Ensures the AI Agent's output conforms to a predefined JSON structure.                      |
| Clean JSON          | Code                           | Parses AI JSON output into separate items      | AI Agent               | Save seen products       | Parses the AI Agent's JSON string output.                                                   |
| Save seen products  | Baserow (create)               | Saves new article links as seen                 | Clean JSON             | Slack                   | Saves the links of newly processed articles to the Baserow 'seen products' table.            |
| Slack               | Slack                         | Sends new article notifications to Slack       | Save seen products     | -                       | Sends a notification with new article details to a Slack channel.                           |
| Sticky Note (various) | Sticky Note                   | Informational notes throughout workflow         | -                      | -                       | See individual sticky notes as detailed above.                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: `Click to Start`
   - Type: Manual Trigger
   - No special parameters.
   
2. **Create Baserow Node to Read RSS URLs**
   - Name: `Read Rss Link`
   - Type: Baserow (Read)
   - Database ID: `243547`
   - Table ID: `579115`
   - Return All: True
   - Connect `Click to Start` → `Read Rss Link`
   
3. **Create Split Out Node**
   - Name: `Split Out`
   - Type: Split Out
   - Field to Split Out: `rssLink`
   - Connect `Read Rss Link` → `Split Out`
   
4. **Create HTTP Request Node**
   - Name: `Fetch HTML`
   - Type: HTTP Request
   - URL: Expression `{{$json["rssLink"]}}`
   - Connect `Split Out` → `Fetch HTML`
   
5. **Create XML Node**
   - Name: `XML Converter`
   - Type: XML
   - Default options
   - Connect `Fetch HTML` → `XML Converter`
   
6. **Create Baserow Node to Read Seen Articles**
   - Name: `Get Seen Products`
   - Type: Baserow (Read)
   - Database ID: `243547`
   - Table ID: `578089`
   - Return All: True
   - Execute Once: True (to avoid duplicate reads per item)
   - Connect `XML Converter` → `Get Seen Products`
   
7. **Create Code Node to Edit Data Structure**
   - Name: `Edit data structure`
   - Type: Code (JavaScript)
   - Code:
     ```javascript
     const articles = $items("XML Converter")[0].json.rss.channel.item.map(item => ({
       title: item.title,
       link: item.link,
       content: item.description
     }));
     
     const alreadyProcessedGuids = $input.all().map(item => item.json.Nom);
     
     return [
       {
         json: {
           input: articles,
           alreadyProcessedGuids: alreadyProcessedGuids
         }
       }
     ];
     ```
   - Connect `Get Seen Products` → `Edit data structure`
   
8. **Create Langchain OpenAI Chat Model Node**
   - Name: `OpenAI Chat Model`
   - Type: Langchain LM Chat OpenAI
   - Model: `gpt-4o-mini`
   - Connect as AI language model input to AI Agent (see next step)
   
9. **Create Langchain Simple Memory Node**
   - Name: `Simple Memory`
   - Type: Langchain Memory Buffer Window
   - Session Key: `=alreadyProcessedGuids`
   - Session ID Type: Custom Key
   - Context Window Length: 50
   - Connect as AI memory input to AI Agent
   
10. **Create Langchain AI Agent Node**
    - Name: `AI Agent`
    - Type: Langchain AI Agent
    - Prompt:
      ```
      You receive a JSON object with two properties:
      - input: an array of articles. Each article has the following properties: title, link, guid, and content.
      - alreadyProcessedGuids: an array containing the guids of the articles that have already been processed.
      
      Here is the object to process:
      {{ JSON.stringify($json) }}
      
      🎯 Your task:
      Loop through the input array and return only the articles whose guid is **not** included in alreadyProcessedGuids: 
      {{ JSON.stringify($json.alreadyProcessedGuids) }}
      📤 Output format:
      Return a **valid JSON array** (not a string) containing only the new, unprocessed articles, with only these properties: title, link, guid, and content.
      
      If all articles were already processed, return exactly: `[]`
      
      ❌ Do NOT return anything else: 
      No text, no explanation, no markdown, no key like `"output"`, no wrapping object.
      
      ⚠️ The output must be raw JSON and must end with `}]`, not `}]}`
      ```
    - Has Output Parser: True
    - Connect `Edit data structure` → `AI Agent`
    - Connect `OpenAI Chat Model` → `AI Agent` (ai_languageModel input)
    - Connect `Simple Memory` → `AI Agent` (ai_memory input)
    
11. **Create Langchain Structured Output Parser Node**
    - Name: `Structured Output Parser`
    - Type: Langchain Output Parser Structured
    - JSON Schema Example:
      ```json
      {
        "title": "",
        "link": "",
        "guid": "",
        "content": ""
      }
      ```
    - Connect AI Agent output to this parser (if used explicitly; in this workflow it is set inside the AI Agent node).
    
12. **Create Code Node to Clean JSON**
    - Name: `Clean JSON`
    - Type: Code (JavaScript)
    - Code:
      ```javascript
      const raw = $input.first().json.output;
      let articles = JSON.parse(raw);
      return articles.map(article => ({ json: article }));
      ```
    - Connect `AI Agent` → `Clean JSON`
    
13. **Create Baserow Node to Save Seen Articles**
    - Name: `Save seen products`
    - Type: Baserow (Create)
    - Database ID: `243547`
    - Table ID: `578089`
    - Operation: Create
    - Field `Nom`: Set to expression `={{ $json.link }}`
    - Connect `Clean JSON` → `Save seen products`
    
14. **Create Slack Node**
    - Name: `Slack`
    - Type: Slack
    - Channel ID: `C091X7ZNW4V` (use your channel’s ID)
    - Message Type: Block
    - Text:
      ```
      Title : {{ $('Clean JSON').item.json.title }}
      Content : {{ $('Clean JSON').item.json.content }}
      Link : {{ $('Clean JSON').item.json.link }}
      ```
    - Connect `Save seen products` → `Slack`
    - Configure Slack credentials with OAuth2 Bot Token or Webhook URL.
    
---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| This n8n workflow is your personal news scout! It automates monitoring RSS feeds, identifies new articles, saves seen links in Baserow, and sends Slack notifications.                                                                                                                                                                                                               | Workflow purpose and description                                                                                                 |
| **Credentials:** Baserow API Key (from Baserow settings API tokens), OpenAI API Key (from https://platform.openai.com/), Slack API OAuth2 or Webhook (https://api.slack.com/apps)                                                                                                                                                                                                    | Credential setup instructions                                                                                                   |
| **Baserow Tables:** Two tables required: One for RSS feed URLs (Database ID: 243547, Table ID: 579115, field: `rssLink`), and one for seen articles (Database ID: 243547, Table ID: 578089, field: `Nom`)                                                                                                                                                                             | Database setup                                                                                                                   |
| To run automatically, schedule this workflow with a time trigger or run manually with the trigger node.                                                                                                                                                                                                                                                                              | Execution options                                                                                                                |
| Slack channel ID `C091X7ZNW4V` is used in the example. Replace with your own channel ID for notifications.                                                                                                                                                                                                                                                                           | Slack channel configuration                                                                                                     |
| AI Agent is configured with a detailed prompt to ensure clean JSON output, avoiding malformed or wrapped responses which would break parsing.                                                                                                                                                                                                                                         | AI prompt design best practice                                                                                                 |
| The workflow handles deduplication by storing article links in Baserow and filtering via AI, improving accuracy and flexibility over simple string matching.                                                                                                                                                                                                                          | Deduplication strategy                                                                                                          |
| For troubleshooting, inspect API keys, network status, and ensure Baserow tables and Slack channels exist and are accessible with configured credentials.                                                                                                                                                                                                                            | Troubleshooting advice                                                                                                          |

---

This structured analysis aims to provide both advanced users and automation agents clear understanding to modify, extend, or reproduce the workflow effectively.