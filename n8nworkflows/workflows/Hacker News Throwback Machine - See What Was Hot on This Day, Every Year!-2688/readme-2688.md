Hacker News Throwback Machine - See What Was Hot on This Day, Every Year!

https://n8nworkflows.xyz/workflows/hacker-news-throwback-machine---see-what-was-hot-on-this-day--every-year--2688


# Hacker News Throwback Machine - See What Was Hot on This Day, Every Year!

### 1. Workflow Overview

This workflow, titled **"Hacker News Throwback Machine - See What Was Hot on This Day, Every Year!"**, is designed to provide a nostalgic and insightful look at Hacker News front-page headlines for the current date across multiple years starting from 2007. It runs daily and performs the following logical blocks:

- **1.1 Scheduled Trigger & Date Preparation:** Initiates the workflow daily at a set hour, calculates the list of dates (same day/month) from the current year back to 2007, with special handling for 2007’s start date.
- **1.2 Data Retrieval & Extraction:** For each date, fetches the Hacker News front page HTML, extracts headlines and associated dates.
- **1.3 Data Aggregation & Formatting:** Combines headlines and dates into a structured JSON array representing multiple years’ data.
- **1.4 AI Processing & Categorization:** Sends the aggregated data to Google Gemini AI to categorize headlines into themes and generate a Markdown summary.
- **1.5 Notification Delivery:** Sends the AI-generated Markdown summary to a Telegram channel.

This modular structure ensures clear separation of concerns: scheduling and date logic, data fetching and parsing, AI-driven analysis, and final user notification.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Date Preparation

- **Overview:**  
  This block triggers the workflow daily at 21:00 (9 PM) and generates a list of dates representing the current day/month for every year from the current year down to 2007. It includes logic to exclude dates before February 19, 2007, for that year.

- **Nodes Involved:**  
  - Schedule Trigger  
  - CreateYearsList (Code)  
  - CleanUpYearList (Set)  
  - SplitOutYearList (SplitOut)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow daily at 21:00 hours.  
    - Configuration: Interval set to trigger at hour 21 daily.  
    - Inputs: None  
    - Outputs: Connects to CreateYearsList  
    - Edge Cases: Timezone considerations may affect trigger time; ensure server timezone matches expected time.

  - **CreateYearsList**  
    - Type: Code (JavaScript)  
    - Role: Generates an array of dates (YYYY-MM-DD) for the current day/month across years from current year down to 2007.  
    - Configuration:  
      - Extracts current date from input timestamp.  
      - Loops backward from current year to 2007.  
      - For 2007, only includes dates from Feb 19 onwards.  
      - Outputs array `datesToFetch` attached to each item.  
    - Inputs: From Schedule Trigger (timestamp)  
    - Outputs: Passes data with `datesToFetch` array to CleanUpYearList  
    - Edge Cases:  
      - If current date is before Feb 19 and year is 2007, skips that year.  
      - Assumes input timestamp is valid ISO date string.  
      - Date formatting strictly ISO (YYYY-MM-DD).  
    - Version: Requires n8n supporting JavaScript code node v2.

  - **CleanUpYearList**  
    - Type: Set  
    - Role: Ensures the `datesToFetch` array is properly assigned for downstream processing.  
    - Configuration: Sets `datesToFetch` field from previous node’s JSON.  
    - Inputs: From CreateYearsList  
    - Outputs: To SplitOutYearList  
    - Edge Cases: None significant.

  - **SplitOutYearList**  
    - Type: SplitOut  
    - Role: Splits the array `datesToFetch` into individual items to process each date separately.  
    - Configuration: Splits on field `datesToFetch`.  
    - Inputs: From CleanUpYearList  
    - Outputs: Each date item sent individually to GetFrontPage  
    - Edge Cases: Empty array would result in no further processing.

---

#### 2.2 Data Retrieval & Extraction

- **Overview:**  
  For each date generated, this block fetches the Hacker News front page HTML and extracts the headlines and the page date.

- **Nodes Involved:**  
  - GetFrontPage (HTTP Request)  
  - ExtractDetails (HTML Extract)  
  - GetHeadlines (Set)  
  - GetDate (Set)

- **Node Details:**

  - **GetFrontPage**  
    - Type: HTTP Request  
    - Role: Fetches Hacker News front page HTML for the specified date.  
    - Configuration:  
      - URL: `https://news.ycombinator.com/front`  
      - Query Parameter: `day` set dynamically to current date from `datesToFetch`.  
      - Batching: Batch size 1 with 3-second interval to avoid rate limits.  
    - Inputs: From SplitOutYearList (single date)  
    - Outputs: Raw HTML response to ExtractDetails  
    - Edge Cases:  
      - HTTP errors (e.g., 404 if no front page for date).  
      - Network timeouts.  
      - Rate limiting by Hacker News.  
      - Date format must be correct ISO string.

  - **ExtractDetails**  
    - Type: HTML Extract  
    - Role: Parses HTML to extract headlines and page date.  
    - Configuration:  
      - Extracts array of headlines using CSS selector `.titleline` (excluding nested spans).  
      - Extracts date string from `.pagetop > font`.  
    - Inputs: From GetFrontPage (HTML)  
    - Outputs: JSON with `headlines` array and `date` string to GetHeadlines and GetDate nodes.  
    - Edge Cases:  
      - Changes in Hacker News HTML structure may break extraction.  
      - Empty or malformed HTML responses.  
      - Headlines without URLs or malformed links.

  - **GetHeadlines**  
    - Type: Set  
    - Role: Assigns extracted headlines array to a dedicated field `headlines`.  
    - Configuration: Sets `headlines` field from extracted data.  
    - Inputs: From ExtractDetails  
    - Outputs: To MergeHeadlinesDate (input 0)  
    - Edge Cases: None significant.

  - **GetDate**  
    - Type: Set  
    - Role: Assigns extracted date string to a dedicated field `date`.  
    - Configuration: Sets `date` field from extracted data.  
    - Inputs: From ExtractDetails  
    - Outputs: To MergeHeadlinesDate (input 1)  
    - Edge Cases: Date string format may vary; downstream nodes expect ISO date.

---

#### 2.3 Data Aggregation & Formatting

- **Overview:**  
  This block merges the headlines and date fields into combined JSON objects, aggregates all such objects into a single JSON array representing all years’ data.

- **Nodes Involved:**  
  - MergeHeadlinesDate (Merge)  
  - SingleJson (Aggregate)

- **Node Details:**

  - **MergeHeadlinesDate**  
    - Type: Merge  
    - Role: Combines headlines and date from two separate streams by position.  
    - Configuration: Mode set to "combine" by position index.  
    - Inputs:  
      - Input 0: From GetHeadlines (headlines array)  
      - Input 1: From GetDate (date string)  
    - Outputs: Combined JSON object with both `headlines` and `date` fields.  
    - Edge Cases: Mismatched input counts could cause data misalignment.

  - **SingleJson**  
    - Type: Aggregate  
    - Role: Aggregates all combined JSON objects into a single array.  
    - Configuration: Aggregates all items into one JSON array under `data`.  
    - Inputs: From MergeHeadlinesDate  
    - Outputs: Single JSON object with `data` array to Basic LLM Chain  
    - Edge Cases: Large data arrays may impact performance.

---

#### 2.4 AI Processing & Categorization

- **Overview:**  
  This block sends the aggregated Hacker News headlines data to Google Gemini AI to categorize headlines into themes, identify trends, and generate a Markdown summary.

- **Nodes Involved:**  
  - Basic LLM Chain (LangChain Chain LLM)  
  - Google Gemini Chat Model (LangChain LM Chat Google Gemini)

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: LangChain Chain LLM  
    - Role: Defines the prompt and processing logic for the AI model.  
    - Configuration:  
      - Prompt instructs the AI to analyze JSON data containing headlines and dates across years for the same day.  
      - Task: Identify top 10-15 headlines, categorize into themes, add markdown hyperlinks with year prefix, and output in Markdown format with a specific header and optional trends section.  
      - Input: JSON stringified data from SingleJson node.  
      - Output: Markdown formatted text.  
    - Inputs: From SingleJson (aggregated data)  
    - Outputs: To Telegram node  
    - Edge Cases:  
      - AI model may fail or timeout.  
      - Prompt formatting errors.  
      - Large input data may exceed token limits.

  - **Google Gemini Chat Model**  
    - Type: LangChain LM Chat Google Gemini  
    - Role: Provides the AI language model backend for the Basic LLM Chain node.  
    - Configuration:  
      - Model name: `models/gemini-1.5-pro`  
      - Credentials: Google Palm API key configured.  
    - Inputs: Connected as AI language model for Basic LLM Chain.  
    - Outputs: Passes AI response back to Basic LLM Chain.  
    - Edge Cases:  
      - API authentication errors.  
      - Rate limits or quota exceeded.  
      - Network issues.

---

#### 2.5 Notification Delivery

- **Overview:**  
  Sends the AI-generated Markdown summary to a Telegram channel for subscribers to view.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - Type: Telegram  
    - Role: Sends a message to a Telegram chat/channel.  
    - Configuration:  
      - Text: Markdown formatted summary from Basic LLM Chain output.  
      - Chat ID: `@OnThisDayHN` (Telegram channel or group).  
      - Parse mode: Markdown enabled.  
      - Append attribution: Disabled.  
      - Credentials: Telegram Bot API token configured.  
    - Inputs: From Basic LLM Chain  
    - Outputs: None (end node)  
    - Edge Cases:  
      - Invalid bot token or chat ID.  
      - Telegram API rate limits.  
      - Markdown formatting errors causing message rejection.

---

### 3. Summary Table

| Node Name            | Node Type                         | Functional Role                        | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                          |
|----------------------|----------------------------------|-------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger                 | Initiates workflow daily at 21:00  | None                          | CreateYearsList                |                                                                                                                      |
| CreateYearsList       | Code                            | Generates list of dates to fetch    | Schedule Trigger              | CleanUpYearList                |                                                                                                                      |
| CleanUpYearList       | Set                             | Assigns datesToFetch array          | CreateYearsList               | SplitOutYearList               |                                                                                                                      |
| SplitOutYearList      | SplitOut                        | Splits dates array into single items| CleanUpYearList               | GetFrontPage                  |                                                                                                                      |
| GetFrontPage          | HTTP Request                    | Fetches Hacker News front page HTML| SplitOutYearList              | ExtractDetails                |                                                                                                                      |
| ExtractDetails        | HTML Extract                    | Extracts headlines and date from HTML| GetFrontPage                 | GetHeadlines, GetDate         |                                                                                                                      |
| GetHeadlines          | Set                             | Sets headlines field                | ExtractDetails                | MergeHeadlinesDate (input 0)  |                                                                                                                      |
| GetDate               | Set                             | Sets date field                    | ExtractDetails                | MergeHeadlinesDate (input 1)  |                                                                                                                      |
| MergeHeadlinesDate    | Merge                           | Combines headlines and date fields | GetHeadlines, GetDate         | SingleJson                    |                                                                                                                      |
| SingleJson            | Aggregate                       | Aggregates all items into one JSON | MergeHeadlinesDate            | Basic LLM Chain               |                                                                                                                      |
| Basic LLM Chain       | LangChain Chain LLM             | Defines AI prompt and processes data| SingleJson                   | Telegram                     |                                                                                                                      |
| Google Gemini Chat Model | LangChain LM Chat Google Gemini | AI model backend for Basic LLM Chain| Connected internally to Basic LLM Chain | Basic LLM Chain (AI response) | Requires Google Gemini API key credential.                                                                           |
| Telegram              | Telegram                        | Sends Markdown summary to Telegram | Basic LLM Chain               | None                         | Requires Telegram bot token and chat ID. Sends to @OnThisDayHN channel with Markdown parse mode enabled.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 21:00 (9 PM).

2. **Create Code Node "CreateYearsList":**  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript code that:  
     - Reads current date from input.  
     - Generates an array `datesToFetch` for the same day/month from current year down to 2007.  
     - Skips dates before Feb 19, 2007.  
   - Connect Schedule Trigger output to this node.

3. **Create Set Node "CleanUpYearList":**  
   - Type: Set  
   - Assign field `datesToFetch` from input JSON.  
   - Connect CreateYearsList output to this node.

4. **Create SplitOut Node "SplitOutYearList":**  
   - Type: SplitOut  
   - Configure to split on field `datesToFetch`.  
   - Connect CleanUpYearList output to this node.

5. **Create HTTP Request Node "GetFrontPage":**  
   - Type: HTTP Request  
   - URL: `https://news.ycombinator.com/front`  
   - Add query parameter `day` set to `={{ $json.datesToFetch }}`  
   - Enable batching: batch size 1, batch interval 3000 ms (3 seconds).  
   - Connect SplitOutYearList output to this node.

6. **Create HTML Extract Node "ExtractDetails":**  
   - Type: HTML Extract  
   - Extraction values:  
     - Key: `headlines`, CSS selector `.titleline`, return array, skip `span` elements.  
     - Key: `date`, CSS selector `.pagetop > font`.  
   - Connect GetFrontPage output to this node.

7. **Create Set Node "GetHeadlines":**  
   - Type: Set  
   - Assign field `headlines` from extracted data.  
   - Connect ExtractDetails output to this node.

8. **Create Set Node "GetDate":**  
   - Type: Set  
   - Assign field `date` from extracted data.  
   - Connect ExtractDetails output to this node.

9. **Create Merge Node "MergeHeadlinesDate":**  
   - Type: Merge  
   - Mode: Combine by position.  
   - Connect GetHeadlines output to input 0.  
   - Connect GetDate output to input 1.

10. **Create Aggregate Node "SingleJson":**  
    - Type: Aggregate  
    - Aggregate all items into one JSON array under `data`.  
    - Connect MergeHeadlinesDate output to this node.

11. **Create LangChain Chain LLM Node "Basic LLM Chain":**  
    - Type: LangChain Chain LLM  
    - Paste the prompt text that instructs the AI to analyze JSON data of headlines/dates, categorize into themes, and output Markdown summary.  
    - Connect SingleJson output to this node.

12. **Create LangChain LM Chat Google Gemini Node "Google Gemini Chat Model":**  
    - Type: LangChain LM Chat Google Gemini  
    - Model name: `models/gemini-1.5-pro`  
    - Add Google Palm API credentials (Google Gemini API key).  
    - Connect this node as the AI language model for Basic LLM Chain (via AI languageModel connection).

13. **Create Telegram Node "Telegram":**  
    - Type: Telegram  
    - Text: `={{ $json.text }}` (Markdown summary from Basic LLM Chain)  
    - Chat ID: `@OnThisDayHN`  
    - Additional fields: parse mode set to Markdown, disable append attribution.  
    - Add Telegram Bot credentials (bot token).  
    - Connect Basic LLM Chain output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow is part of the [#100DaysOfAgenticAi](https://github.com/ibrhdotme/100DaysOfAgenticAi/) project.       | Project GitHub repository for related workflows and AI automation experiments.                   |
| Telegram channel used is `@OnThisDayHN` — ensure your bot has permissions to post there.                        | Telegram channel for publishing daily Hacker News summaries.                                    |
| Google Gemini API key and Telegram Bot token must be securely stored and configured in n8n credentials.       | Credential setup instructions for API access.                                                   |
| The workflow uses Google Gemini (PaLM) model `models/gemini-1.5-pro` for AI processing.                         | Requires access to Google PaLM API with appropriate quota.                                      |
| The Hacker News front page URL and HTML structure may change, requiring updates to the extraction CSS selectors.| Monitor Hacker News site for changes to `.titleline` and `.pagetop > font` selectors.            |

---

This documentation provides a detailed, stepwise understanding of the workflow’s structure, node configurations, data flow, and integration points, enabling advanced users and AI agents to reproduce, modify, or troubleshoot the workflow effectively.