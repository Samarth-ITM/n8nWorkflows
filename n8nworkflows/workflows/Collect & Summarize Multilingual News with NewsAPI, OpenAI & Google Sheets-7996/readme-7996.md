Collect & Summarize Multilingual News with NewsAPI, OpenAI & Google Sheets

https://n8nworkflows.xyz/workflows/collect---summarize-multilingual-news-with-newsapi--openai---google-sheets-7996


# Collect & Summarize Multilingual News with NewsAPI, OpenAI & Google Sheets

### 1. Workflow Overview

This workflow automates the daily collection and summarization of multilingual news articles, specifically English and Japanese, using NewsAPI, OpenAI, and Google Sheets. It targets users who want to monitor news topics in both languages, summarizing gathered articles into concise Japanese sentences, and storing the results in a Google Sheet for easy tracking.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & Input Retrieval:** Daily trigger initiates the workflow, reading keywords and search flags from a Google Sheet.
- **1.2 Conditional Filtering:** Checks if news search is required per keyword.
- **1.3 News Collection (English & Japanese):** Parallel HTTP requests to NewsAPI for English and Japanese news articles.
- **1.4 Article Flattening & Merging:** Transforms raw API responses into individual article items and merges both language streams.
- **1.5 AI Summarization:** Sends each article to OpenAI to generate a concise Japanese summary (~50 characters).
- **1.6 Data Preparation & Storage:** Prepares output fields and appends summarized data back into a designated Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Input Retrieval

- **Overview:**  
Triggers the workflow daily at 13:00 and reads the list of keywords with a flag indicating if a search is required from a Google Sheet.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get rows from sheet

- **Node Details:**

1. **Schedule Trigger**  
   - Type: Schedule Trigger  
   - Role: Starts workflow every day at 13:00 (1 PM).  
   - Configuration: Hour-based interval set to 13:00.  
   - Inputs: None  
   - Outputs: Triggers "Get rows from sheet".  
   - Notes: Time can be adjusted by changing trigger hour.  
   - Failure modes: Misconfiguration or system downtime may prevent trigger execution.

2. **Get rows from sheet**  
   - Type: Google Sheets node  
   - Role: Reads all rows from a specified Google Sheet tab containing keywords and search flags.  
   - Configuration:  
     - Spreadsheet ID and sheet name must be specified (user’s own sheet).  
     - Required columns: "Keyword" and "SearchRequired" (values expected: "Yes" or "No").  
   - Inputs: Trigger from Schedule Trigger  
   - Outputs: JSON array of rows to "If Search Required".  
   - Credentials: Google OAuth2 account connected.  
   - Failure modes: Authentication errors, invalid sheet ID or name, missing required columns.

---

#### 2.2 Conditional Filtering

- **Overview:**  
Filters keywords to proceed only with those marked "Yes" in the "SearchRequired" column.

- **Nodes Involved:**  
  - If Search Required

- **Node Details:**

1. **If Search Required**  
   - Type: If node (conditional)  
   - Role: Checks if "SearchRequired" field equals "Yes" to continue the search.  
   - Configuration: String equality condition on `$json['SearchRequired'] === 'Yes'`.  
   - Inputs: Rows from "Get rows from sheet" (each containing Keyword and SearchRequired).  
   - Outputs: "Yes" path triggers news API requests; "No" path discards the item.  
   - Failure modes: Expression errors if property missing; ensures workflow efficiency by skipping unnecessary searches.

---

#### 2.3 News Collection (English & Japanese)

- **Overview:**  
Performs two parallel HTTP requests to NewsAPI: one fetching English news articles globally by keyword, the other fetching Japanese news articles filtered by specific domains.

- **Nodes Involved:**  
  - HTTP Request (EN)  
  - HTTP Request (JP)  
  - Sticky Note (collect both languages)

- **Node Details:**

1. **HTTP Request (EN)**  
   - Type: HTTP Request  
   - Role: Queries NewsAPI for English articles matching the keyword.  
   - Configuration:  
     - URL: `https://newsapi.org/v2/everything`  
     - Query params:  
       - `q`: keyword from input  
       - `language`: "en"  
       - `pageSize`: 5 (limit to 5 articles)  
       - `apiKey`: User’s NewsAPI key (must be replaced in the parameter)  
   - Inputs: "Yes" output from "If Search Required"  
   - Outputs: JSON response containing articles to "Split Articles (EN)".  
   - Failure modes: API key invalid/expired, rate limiting, network errors.

2. **HTTP Request (JP)**  
   - Type: HTTP Request  
   - Role: Queries NewsAPI for Japanese articles from specified Japanese news domains.  
   - Configuration:  
     - URL: `https://newsapi.org/v2/everything`  
     - Query params:  
       - `q`: keyword  
       - `domains`: Japanese news websites (nhk.or.jp, asahi.com, etc.)  
       - `sortBy`: "publishedAt" (latest first)  
       - `pageSize`: 5  
       - `apiKey`: User’s NewsAPI key (same as above)  
   - Inputs: Same as EN request  
   - Outputs: JSON response to "Split Articles (JP)".  
   - Failure modes: Same as EN request; domain filtering may reduce results.

3. **Sticky Note (Collect both English and Japanese news articles)**  
   - Provides contextual guidance on collecting news from both languages.

---

#### 2.4 Article Flattening & Merging

- **Overview:**  
Transforms arrays of articles from each HTTP response into individual items, then merges both English and Japanese article streams for unified processing.

- **Nodes Involved:**  
  - Split Articles (EN)  
  - Split Articles (JP)  
  - Merge Articles  
  - Sticky Note (“1 item = 1 article”)

- **Node Details:**

1. **Split Articles (EN)**  
   - Type: Code node  
   - Role: Extracts the `articles` array from NewsAPI response, outputs each article as a separate item for further processing.  
   - Code: Maps `articles` array to individual JSON items.  
   - Inputs: HTTP Request (EN) output  
   - Outputs: Single article items to "Merge Articles" (first input).  
   - Failure modes: If `articles` field missing or empty, outputs empty array.

2. **Split Articles (JP)**  
   - Type: Code node  
   - Role: Same function as above but for Japanese articles.  
   - Inputs: HTTP Request (JP) output  
   - Outputs: Individual article items to "Merge Articles" (second input).  
   - Failure modes: Similar to EN split node.

3. **Merge Articles**  
   - Type: Merge node  
   - Role: Combines English and Japanese article streams into one unified stream for summarization.  
   - Inputs: Two inputs from "Split Articles (EN)" and "Split Articles (JP)".  
   - Outputs: Merged combined stream to "Summarize with OpenAI".  
   - Failure modes: Mismatched or missing inputs could cause empty output.

4. **Sticky Note (“1 item = 1 article”)**  
   - Clarifies that each workflow item corresponds to one news article for processing.

---

#### 2.5 AI Summarization

- **Overview:**  
Uses OpenAI GPT (model gpt-4.1-mini) to generate a concise summary (~50 characters) in Japanese for each article, strictly factual without opinions or emojis.

- **Nodes Involved:**  
  - Summarize with OpenAI  
  - Sticky Note (OpenAI credentials)

- **Node Details:**

1. **Summarize with OpenAI**  
   - Type: OpenAI (LangChain integration)  
   - Role: Sends the article title and content (description or content field) to GPT-4 for concise Japanese summary.  
   - Configuration:  
     - Model: gpt-4.1-mini  
     - Messages:  
       - System prompt: Editor summarizing news into concise Japanese (~50 chars), no opinions or emojis.  
       - User prompt: Includes article title and content.  
   - Inputs: Merged article items  
   - Outputs: Summary text in `$json.message.content` to "Prepare fields".  
   - Credentials: User’s OpenAI API key required.  
   - Failure modes: API errors, rate limits, malformed inputs, or connectivity issues.

2. **Sticky Note (Connect your own OpenAI account here...)**  
   - Reminder to link a valid OpenAI account or disable node if not used.

---

#### 2.6 Data Preparation & Storage

- **Overview:**  
Prepares output fields including URL, Date, Keyword, and Summary, then appends this data to a specified Google Sheet.

- **Nodes Involved:**  
  - Prepare fields  
  - Append rows to sheet  
  - Sticky Notes (Google Sheets setup and output instructions)

- **Node Details:**

1. **Prepare fields**  
   - Type: Set node  
   - Role: Maps AI summary and merged article URL into fields for Google Sheets.  
   - Configuration:  
     - Sets `summary` from OpenAI result content.  
     - Sets `link` from merged article’s URL field.  
     - Passes through other fields like Date and Keyword.  
   - Inputs: OpenAI summarization output  
   - Outputs: Formatted JSON for Google Sheets appending.  
   - Failure modes: Missing expected fields could cause blank entries.

2. **Append rows to sheet**  
   - Type: Google Sheets node  
   - Role: Appends new rows with Date, Keyword, Summary, URL to output sheet tab.  
   - Configuration:  
     - Spreadsheet ID and output sheet name (user’s own).  
     - Columns mapped explicitly: Date, Keyword, Summary, URL.  
   - Inputs: Prepared fields from previous node  
   - Credentials: Same Google OAuth2 as input sheet node.  
   - Failure modes: Authentication, sheet access issues, schema mismatch.

3. **Sticky Note (Set your own Google Sheet...)**  
   - Instructions for spreadsheet setup:  
     - Two tabs: 01_Input (with Keyword, SearchRequired columns) and 02_Output (Date, Keyword, Summary, URL columns).  
     - Output sheet must initially be empty to append.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                              | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                           |
|-------------------------|----------------------------------|----------------------------------------------|----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger                  | Daily workflow trigger at 13:00               | -                          | Get rows from sheet          | Runs daily at 13:00 (change in Schedule Trigger)                                                    |
| Get rows from sheet      | Google Sheets                    | Reads keywords and search flags                | Schedule Trigger            | If Search Required           | Read keywords from your Google Sheet. Required columns: Keyword, SearchRequired (values: Yes / No). |
| If Search Required       | If                              | Filters keywords where SearchRequired == Yes  | Get rows from sheet         | HTTP Request (EN), HTTP Request (JP) | Continue only if SearchRequired === Yes.                                                        |
| HTTP Request (EN)        | HTTP Request                    | Fetches English news articles from NewsAPI    | If Search Required          | Split Articles (EN)          | Insert your own NewsAPI key here                                                                    |
| HTTP Request (JP)        | HTTP Request                    | Fetches Japanese news articles from NewsAPI   | If Search Required          | Split Articles (JP)          | Insert your own NewsAPI key here                                                                    |
| Sticky Note              | Sticky Note                     | Collect both English and Japanese news articles | -                          | -                           | ## Collect both English and Japanese news articles                                                  |
| Split Articles (EN)      | Code                           | Flattens English articles array into items    | HTTP Request (EN)           | Merge Articles (input 1)     | ## 1 item = 1 article                                                                               |
| Split Articles (JP)      | Code                           | Flattens Japanese articles array into items   | HTTP Request (JP)           | Merge Articles (input 2)     | ## 1 item = 1 article                                                                               |
| Merge Articles           | Merge                          | Combines English and Japanese article streams | Split Articles (EN), Split Articles (JP) | Summarize with OpenAI | Merge both article streams before summarization.                                                   |
| Summarize with OpenAI    | OpenAI (LangChain)             | Summarizes articles into concise Japanese text | Merge Articles              | Prepare fields              | Connect your own OpenAI account here (or deactivate this node if not needed)                        |
| Prepare fields           | Set                            | Prepares fields for Google Sheets appending   | Summarize with OpenAI       | Append rows to sheet         |                                                                                                     |
| Append rows to sheet     | Google Sheets                  | Appends summarized articles to output sheet   | Prepare fields              | -                           | Output sheet must have columns: Date, Keyword, Summary, URL (empty at first)                        |
| Sticky Note1             | Sticky Note                    | -                                              | -                          | -                           | ## Runs daily at 13:00 (change in Schedule Trigger)                                                |
| Sticky Note2             | Sticky Note                    | -                                              | -                          | -                           | ## 1 item = 1 article                                                                               |
| Sticky Note3             | Sticky Note                    | -                                              | -                          | -                           | ## Set your own Google Sheet (enter Spreadsheet ID & Sheet name)                                  |
| Sticky Note4             | Sticky Note                    | -                                              | -                          | -                           | Prepare your own Google Spreadsheet with the following structure...                                |
| Sticky Note5             | Sticky Note                    | -                                              | -                          | -                           | ## Insert your NewsAPI key here                                                                    |
| Sticky Note6             | Sticky Note                    | -                                              | -                          | -                           | ## Connect your own OpenAI account here (or deactivate this node if not needed)                    |
| Sticky Note7             | Sticky Note                    | -                                              | -                          | -                           | ## Output sheet must have columns: Date, Keyword, Summary, URL (empty at first)                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger daily at 13:00 (hour = 13).  
   - No input required.

2. **Create a Google Sheets node ("Get rows from sheet")**  
   - Operation: Read rows.  
   - Configure with your Google Sheets OAuth2 credentials.  
   - Set Spreadsheet ID and Input Sheet Name/Tab (e.g., "01_Input").  
   - Required columns in sheet: "Keyword", "SearchRequired".  
   - Connect Schedule Trigger output to this node.

3. **Create an If node ("If Search Required")**  
   - Condition: `$json["SearchRequired"] === "Yes"`.  
   - Connect "Get rows from sheet" output to this node.

4. **Create two HTTP Request nodes:**

   - **"HTTP Request (EN)"**  
     - URL: `https://newsapi.org/v2/everything`  
     - Query Parameters:  
       - `q`: Expression `{{$json["Keyword"]}}`  
       - `language`: "en"  
       - `pageSize`: "5"  
       - `apiKey`: Your NewsAPI key  
     - Connect "If Search Required" true output to this node.

   - **"HTTP Request (JP)"**  
     - URL: `https://newsapi.org/v2/everything`  
     - Query Parameters:  
       - `q`: Expression `{{$json["Keyword"]}}`  
       - `domains`: "nhk.or.jp,asahi.com,nikkei.com,news.yahoo.co.jp,itmedia.co.jp,impress.co.jp,ascii.jp,prtimes.jp"  
       - `sortBy`: "publishedAt"  
       - `pageSize`: "5"  
       - `apiKey`: Your NewsAPI key  
     - Connect "If Search Required" true output (parallel) to this node.

5. **Create two Code nodes to split articles into individual items:**

   - **"Split Articles (EN)"**  
     - JavaScript code:  
       ```javascript
       const arr = $json.articles ?? [];
       return arr.map(a => ({ json: a }));
       ```  
     - Connect "HTTP Request (EN)" output to this node.

   - **"Split Articles (JP)"**  
     - Same JavaScript code as above.  
     - Connect "HTTP Request (JP)" output to this node.

6. **Create a Merge node ("Merge Articles")**  
   - Mode: Merge by Append (default).  
   - Connect "Split Articles (EN)" output to first input.  
   - Connect "Split Articles (JP)" output to second input.

7. **Create an OpenAI node ("Summarize with OpenAI")**  
   - Use LangChain OpenAI node or native OpenAI node.  
   - Connect your OpenAI credentials (API key).  
   - Model: gpt-4.1-mini (or equivalent available).  
   - Messages:  
     - System role: "You are an editor summarizing news articles into concise Japanese sentences (~50 characters). No opinions, no emojis."  
     - User role (prompt):  
       ```
       Summarize the following article (~50 characters in Japanese).
       Title: {{$json["title"]}}
       Content: {{$json["description"] || $json["content"] || "No content"}}
       ```  
   - Connect "Merge Articles" output to this node.

8. **Create a Set node ("Prepare fields")**  
   - Assignments:  
     - `summary`: Expression: `{{$json["message"]?.["content"]}}` (summary from OpenAI)  
     - `link`: Expression: `{{$node["Merge Articles"].json["url"]}}` (article URL)  
   - Include other fields (Date, Keyword).  
   - Connect "Summarize with OpenAI" output to this node.

9. **Create a Google Sheets node ("Append rows to sheet")**  
   - Operation: Append row(s).  
   - Configure with Google Sheets OAuth2 credentials.  
   - Set Spreadsheet ID and Output Sheet Name/Tab (e.g., "02_Output").  
   - Map columns explicitly:  
     - Date: e.g., from article publishedAt or other field  
     - Keyword  
     - Summary  
     - URL  
   - Connect "Prepare fields" output to this node.

10. **Add Sticky Notes at key points for documentation purposes** (optional but recommended):  
    - At trigger: "Runs daily at 13:00 (change in Schedule Trigger)"  
    - Near input sheet node: "Set your own Google Sheet (enter Spreadsheet ID & Sheet name)"  
    - Near NewsAPI calls: "Insert your NewsAPI key here"  
    - Near OpenAI node: "Connect your own OpenAI account here (or deactivate this node if not needed)"  
    - Near output sheet node: "Output sheet must have columns: Date, Keyword, Summary, URL (empty at first)"  
    - At article splitting: "1 item = 1 article"  
    - Near news fetch nodes: "Collect both English and Japanese news articles"  
    - Instructions for Google Sheet structure:  
      - Input tab "01_Input" with "Keyword" and "SearchRequired" columns  
      - Output tab "02_Output" with "Date", "Keyword", "Summary", "URL" columns initially empty

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Prepare your own Google Spreadsheet with two tabs: "01_Input" (Keyword, SearchRequired) and "02_Output" (Date, Keyword, Summary, URL) | Sticky Note4 content in workflow                     |
| Workflow requires valid NewsAPI key for news fetching.                                                  | Sticky Note5                                          |
| OpenAI usage incurs charges on your account; connect your own API key or disable summarization if unwanted. | Sticky Note6                                          |
| The summarization prompt instructs no opinions or emojis to maintain factual tone and conciseness.      | Summarize with OpenAI node prompt                     |
| Workflow is designed for daily automated news monitoring and summarization in Japanese.                 | Workflow overview and sticky notes                    |

---

*Disclaimer:* The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.