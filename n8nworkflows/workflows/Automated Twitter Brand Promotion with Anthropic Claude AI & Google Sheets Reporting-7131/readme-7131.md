Automated Twitter Brand Promotion with Anthropic Claude AI & Google Sheets Reporting

https://n8nworkflows.xyz/workflows/automated-twitter-brand-promotion-with-anthropic-claude-ai---google-sheets-reporting-7131


# Automated Twitter Brand Promotion with Anthropic Claude AI & Google Sheets Reporting

### 1. Workflow Overview

This automated n8n workflow is designed for **Twitter brand promotion** using **Anthropic Claude AI** to generate authentic, strategic responses to relevant Twitter posts, coupled with **Google Sheets** reporting for tracking engagement. The workflow periodically searches Twitter for posts containing a specified keyword, generates personalized comments to promote a platform or brand subtly, posts these comments as Twitter replies, and logs all interactions into a dated Google Sheet report.

Logical blocks:

- **1.1 Trigger and Date Setup**: Scheduled and manual triggers initiate the workflow, setting the current date and preparing for Google Sheet interactions.

- **1.2 Google Sheet Verification and Setup**: Checks if a daily Google Sheet exists for logging; creates one with headers if absent.

- **1.3 Twitter Search and Tweet Extraction**: Searches Twitter for recent tweets matching the keyword, extracting a limited number of tweets for processing.

- **1.4 AI Comment Generation**: Uses Anthropic Claude AI to generate authentic, psychologically-informed Twitter replies promoting the brand.

- **1.5 Posting Replies on Twitter**: Posts generated comments as replies to the selected tweets.

- **1.6 Reporting to Google Sheets**: Appends the tweet URLs, generated responses, and keywords into the daily Google Sheet for record-keeping.

- **1.7 Controlled Execution Flow**: Includes batching, waiting, and merging nodes to manage API rate limits and orderly processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Date Setup

**Overview:**  
This block initiates the workflow either manually or on schedule every 4 hours and sets the current date formatted as `dd-MM-yyyy` for use in Google Sheets naming and data logging.

**Nodes Involved:**  
- Schedule Trigger (every 4 hours)  
- When clicking ‘Test workflow’ (manual trigger)  
- Automaticaly set todays Date  
- SET KEYWORD

**Node Details:**

- **Schedule Trigger (every 4 hours)**  
  - *Type:* ScheduleTrigger  
  - *Role:* Automatically triggers the workflow every 4 hours using a cron expression.  
  - *Config:* Cron expression set to `0 */4 * * *`.  
  - *Input/Output:* No input; outputs trigger signal to the date setting node.  
  - *Failures:* Cron misconfiguration unlikely; if triggered fails, the workflow won’t start.

- **When clicking ‘Test workflow’**  
  - *Type:* ManualTrigger  
  - *Role:* Allows manual initiation for testing or one-off runs.  
  - *Input/Output:* Outputs trigger signal to keyword setting.  

- **Automaticaly set todays Date**  
  - *Type:* Set  
  - *Role:* Assigns a `date` string with the current date in `dd-MM-yyyy` format for downstream Google Sheets usage.  
  - *Key Expression:* `={{ $now.format('dd-MM-yyyy') }}`  
  - *Output:* JSON object with `date` property.  

- **SET KEYWORD**  
  - *Type:* Set  
  - *Role:* Sets the `keyword` string used for Twitter search; user must replace placeholder `"=ENTER YOU KEYWORD HERE"` with actual search term.  
  - *Output:* JSON object with `keyword` property.  
  - *Edge Cases:* If keyword left as placeholder, Twitter search may return empty or irrelevant results.

---

#### 2.2 Google Sheet Verification and Setup

**Overview:**  
This block verifies if a Google Sheet exists for the current date. If not found, it creates a new sheet with defined headers to log Twitter interactions.

**Nodes Involved:**  
- Check if there's a Google Sheet for todays Date  
- Code (count posts)  
- Create new sheet  
- Define sheet headers  
- Add headers to the Sheet  
- Merge  
- Add info to the Report Table1

**Node Details:**

- **Check if there's a Google Sheet for todays Date**  
  - *Type:* GoogleSheets (read)  
  - *Role:* Attempts to read the sheet named by the current date.  
  - *Config:* Uses dynamic `sheetName` = current date; `documentId` must be set to user's report spreadsheet ID.  
  - *On Error:* Continue error output to handle sheet not found gracefully.  
  - *Edge Cases:* Sheet not found triggers downstream sheet creation.

- **Code (count posts)**  
  - *Type:* Code (JavaScript)  
  - *Role:* Returns JSON with `postsCount` equal to the number of tweets fetched; used for limiting tweets processed downstream.  
  - *Input:* Twitter search results array length.  
  - *Output:* JSON with `postsCount`.

- **Create new sheet**  
  - *Type:* GoogleSheets (create)  
  - *Role:* Creates a new sheet/tab named with the current date in the Google Spreadsheet if none exists.  
  - *Input:* Triggered when no sheet found.  
  - *Edge Cases:* API quota limits, permissions issues.

- **Define sheet headers**  
  - *Type:* Set  
  - *Role:* Defines header row values as an array: ["Post URL", "Our response", "Keyword"].  
  - *Output:* JSON with headers and date.

- **Add headers to the Sheet**  
  - *Type:* HTTP Request  
  - *Role:* Uses Google Sheets API directly to PUT header row to the new sheet.  
  - *Config:* Dynamic URL with spreadsheetId and sheet name; uses Google OAuth2 credentials.  
  - *Method:* PUT with raw value input.  
  - *Edge Cases:* API errors, invalid credentials.

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines data streams from sheet header setup and earlier data for further processing.

- **Add info to the Report Table1**  
  - *Type:* GoogleSheets (append)  
  - *Role:* Appends tweet URL, AI-generated response, and keyword to the sheet named by current date.  
  - *Config:* Uses dynamic inputs and spreadsheetId.  
  - *Edge Cases:* Append failures due to invalid sheet or permissions.

---

#### 2.3 Twitter Search and Tweet Extraction

**Overview:**  
Searches Twitter for recent tweets containing the specified keyword within the last 4 hours. Extracts a limited number of tweets (2 or 3) to avoid overloading downstream nodes.

**Nodes Involved:**  
- Twitter Search by Keyword  
- Split tweets  
- Code (postsCount)

**Node Details:**

- **Twitter Search by Keyword**  
  - *Type:* HTTP Request  
  - *Role:* Calls twitterapi.io advanced search endpoint with query parameter `query` set to keyword + `within_time:4h`.  
  - *Authentication:* Via HTTP Header Auth with twitterapi.io credentials.  
  - *Config:* Max redirects set to 5.  
  - *Edge Cases:* Auth failure, rate limiting, malformed query.

- **Split tweets**  
  - *Type:* Code  
  - *Role:* Processes response, extracts tweets array, limits output to 2 tweets if postsCount is 15, else 3 tweets.  
  - *Key Logic:* Uses JavaScript to iterate over tweets and push up to limit.  
  - *Output:* Array of tweet JSON objects for further processing.  
  - *Edge Cases:* Empty or missing tweets array, malformed data.

- **Code (postsCount)**  
  - (Already described in 2.2)

---

#### 2.4 AI Comment Generation

**Overview:**  
Generates authentic, psychologically-informed Twitter replies promoting the brand/platform using the Anthropic Claude AI model.

**Nodes Involved:**  
- Anthropic Chat Model  
- Compose Comment

**Node Details:**

- **Anthropic Chat Model**  
  - *Type:* Langchain Anthropic Node  
  - *Role:* Calls Anthropic Claude Sonnet 4 AI model to generate text based on input prompt.  
  - *Model:* `claude-sonnet-4-20250514` selected.  
  - *Options:* Thinking enabled for more context-aware responses.  
  - *Input:* Twitter post text from `Split tweets`.  
  - *Output:* AI-generated text response.  
  - *Edge Cases:* API quota, latency, auth failures.

- **Compose Comment**  
  - *Type:* Langchain Agent Node  
  - *Role:* Contains a detailed prompt instructing the AI on how to craft strategic, authentic Twitter replies integrating brand mentions naturally.  
  - *Prompt Highlights:*  
    - Psychological engagement techniques (empathy, curiosity, social proof).  
    - Multiple response structures (Problem Agitation Solution, Social Proof Story, Curiosity Hook).  
    - Twitter vernacular and casual language encouraged.  
    - Absolute rules to always generate a response, even if post content is empty (fallback response included).  
  - *Input:* Tweet text JSON from `Split tweets`.  
  - *Output:* Generated comment text.  
  - *Config:* System message defines AI persona as helpful Twitter user sharing positive experiences.  
  - *Edge Cases:* Empty input, ambiguous tweets handled by fallback.

---

#### 2.5 Posting Replies on Twitter

**Overview:**  
Posts the AI-generated comments as replies to the selected tweets on Twitter, respecting concurrency and rate limits.

**Nodes Involved:**  
- Loop Over Items  
- Post on Twitter  
- Wait  
- Merge2

**Node Details:**

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes tweets/comments in batches to control rate and avoid API limits.  
  - *Config:* Default batch size (not explicitly set).  
  - *Input:* Comments from `Compose Comment`.  
  - *Outputs:* Two branches — one to append data to report, one to post on Twitter.

- **Post on Twitter**  
  - *Type:* Twitter Node  
  - *Role:* Posts each comment as a reply to the corresponding tweet (`inReplyToStatusId` set dynamically).  
  - *Config:*  
    - Text: `={{ $json.output }}` (AI comment)  
    - Reply ID: ID from `Split tweets` node.  
    - Retry enabled on failure (max 2 tries) with 5000ms wait between tries.  
    - Continues on error silently.  
  - *Edge Cases:* Auth errors, rate limits, transient Twitter API failures.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Waits 8 minutes after posting each tweet before proceeding, likely to manage Twitter rate limits or pacing.  
  - *Config:* 8 minutes delay.  
  - *Input:* From `Post on Twitter`.  
  - *Output:* Loops back to `Loop Over Items` for next batch.

- **Merge2**  
  - *Type:* Merge  
  - *Role:* Combines streams of AI comments and original tweets by position for synchronized processing.

---

#### 2.6 Reporting to Google Sheets

**Overview:**  
Logs each processed Twitter post URL, the AI-generated comment, and the search keyword into the daily Google Sheet for record-keeping.

**Nodes Involved:**  
- Add info to the Report Table  
- Add info to the Report Table1

**Node Details:**

- **Add info to the Report Table**  
  - *Type:* GoogleSheets (append)  
  - *Role:* Appends a row with keyword, post URL, and AI response to the day's Google Sheet.  
  - *Config:* Uses dynamic sheet name based on current date and user-provided spreadsheet ID.  
  - *On Error:* Continue silently to avoid workflow interruption.  

- **Add info to the Report Table1**  
  - *Type:* GoogleSheets (append)  
  - *Role:* Similar to above; receives merged data stream for appending.  
  - *Purpose:* Ensures all processed interactions are logged after sheet creation or header addition.  

- *Edge Cases:* API quota, permissions, sheet existence issues.

---

#### 2.7 Sticky Notes and Documentation

**Overview:**  
Sticky notes provide user instructions, configuration tips, and documentation inside the workflow UI.

**Nodes Involved:**  
- Sticky Note7  
- Sticky Note8  
- Sticky Note9  
- Sticky Note10  
- Sticky Note11  
- Sticky Note12

**Content Summary:**

- Step-by-step connection instructions for Google Sheets, Twitter, twitterapi.io, Anthropic accounts.  
- Instruction to replace placeholders in the "Compose Comment" node prompt before running.  
- Descriptions for logical blocks: Twitter search, comment generation, posting, and reporting.  
- Tutorial video links for key service integrations:  
  - Google Sheets: https://www.youtube.com/watch?v=pWGXlZBGu4k  
  - Twitter: https://www.youtube.com/watch?v=jxKKwYZQF7Q  
  - twitterapi.io: https://www.youtube.com/watch?v=lEo7IAgj0UY  
  - Anthropic AI: https://www.youtube.com/watch?v=1jl_vBoVvq0  

---

### 3. Summary Table

| Node Name                         | Node Type                         | Functional Role                                 | Input Node(s)                       | Output Node(s)                            | Sticky Note                                                                                                 |
|----------------------------------|----------------------------------|------------------------------------------------|-----------------------------------|------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | ManualTrigger                    | Manual workflow start                           | —                                 | SET KEYWORD                              | See Sticky Note7 for setup instructions                                                                    |
| Schedule Trigger (every 4 hours)  | ScheduleTrigger                 | Scheduled workflow start                        | —                                 | Automaticaly set todays Date             | See Sticky Note7 for setup instructions                                                                    |
| Automaticaly set todays Date      | Set                             | Sets current date string                        | Schedule Trigger, ManualTrigger    | Check if there's a Google Sheet for todays Date |                                                                                                             |
| Check if there's a Google Sheet for todays Date | GoogleSheets (read)              | Checks for existence of daily sheet             | Automaticaly set todays Date       | Code (postsCount), Code (postsCount)     |                                                                                                             |
| Code (postsCount)                 | Code                            | Counts number of tweets fetched                 | Check if there's a Google Sheet    | SET KEYWORD (via connection), Split tweets |                                                                                                             |
| SET KEYWORD                      | Set                             | Sets Twitter search keyword                      | When clicking ‘Test workflow’, Code | Twitter Search by Keyword                 | See Sticky Note7: Replace placeholder keyword                                                                |
| Twitter Search by Keyword         | HTTP Request                   | Searches Twitter API for tweets                  | SET KEYWORD                      | Split tweets                             | Requires twitterapi.io credentials (see Sticky Note7)                                                       |
| Split tweets                    | Code                            | Extracts and limits tweets for processing       | Twitter Search by Keyword          | Compose Comment, Merge2                   |                                                                                                             |
| Anthropic Chat Model             | Langchain Anthropic             | Calls AI to generate comment                     | Compose Comment                   | Compose Comment                           | Requires Anthropic AI credentials (see Sticky Note7)                                                        |
| Compose Comment                 | Langchain Agent                 | Generates strategic Twitter reply                | Split tweets, Anthropic Chat Model | Loop Over Items                         | See Sticky Note7: Customize prompt placeholders                                                             |
| Loop Over Items                 | SplitInBatches                 | Processes tweets/comments in manageable batches | Compose Comment                   | Merge2, Add info to the Report Table, Post on Twitter |                                                                                                             |
| Merge2                          | Merge                          | Combines tweet and comment streams               | Split tweets, Loop Over Items      | Merge, Add info to the Report Table1      |                                                                                                             |
| Merge                           | Merge                          | Combines data streams for reporting              | Define sheet headers, Add headers to the Sheet | Add info to the Report Table1              |                                                                                                             |
| Add info to the Report Table    | GoogleSheets (append)          | Appends report data to daily Google Sheet        | Loop Over Items                   | Create new sheet (on error)                |                                                                                                             |
| Add info to the Report Table1   | GoogleSheets (append)          | Appends report data                               | Merge                            | —                                        |                                                                                                             |
| Create new sheet                | GoogleSheets (create)          | Creates new daily sheet if missing                | Add info to the Report Table       | Define sheet headers                      |                                                                                                             |
| Define sheet headers            | Set                             | Defines header row values                          | Create new sheet                 | Add headers to the Sheet                  |                                                                                                             |
| Add headers to the Sheet        | HTTP Request                   | Adds headers as first row in the new sheet        | Define sheet headers             | Merge                                     |                                                                                                             |
| Post on Twitter                | Twitter Node                  | Posts AI-generated comment as Twitter reply       | Loop Over Items                   | Wait                                      | Requires Twitter account connection (see Sticky Note7)                                                     |
| Wait                          | Wait                            | Waits 8 minutes between posts                      | Post on Twitter                  | Loop Over Items                            |                                                                                                             |
| Code (count posts)             | Code                            | (Duplicate) Counts tweets for limit logic         | Check if there's a Google Sheet   | SET KEYWORD                              |                                                                                                             |
| Sticky Note7                  | StickyNote                     | Configuration and startup instructions             | —                               | —                                        | https://www.youtube.com/watch?v=pWGXlZBGu4k, https://www.youtube.com/watch?v=jxKKwYZQF7Q, https://www.youtube.com/watch?v=lEo7IAgj0UY, https://www.youtube.com/watch?v=1jl_vBoVvq0 |
| Sticky Note8                  | StickyNote                     | Twitter search block description                    | —                               | —                                        |                                                                                                             |
| Sticky Note9                  | StickyNote                     | AI comment generation block description             | —                               | —                                        |                                                                                                             |
| Sticky Note10                 | StickyNote                     | Posting comments block description                    | —                               | —                                        |                                                                                                             |
| Sticky Note11                 | StickyNote                     | Google Sheets reporting block description            | —                               | —                                        |                                                                                                             |
| Sticky Note12                 | StickyNote                     | Alternative Google Sheets reporting block description | —                               | —                                        |                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create triggers:**

   - Add a **Schedule Trigger** node:
     - Set cron expression to `0 */4 * * *` for every 4 hours.
   - Add a **Manual Trigger** node for testing.

2. **Set current date:**

   - Add a **Set** node named “Automaticaly set todays Date”.
   - Add assignment: `date` = `={{ $now.format('dd-MM-yyyy') }}`.
   - Connect both triggers to this node.

3. **Check or create daily Google Sheet tab:**

   - Add a **Google Sheets** node "Check if there's a Google Sheet for todays Date":
     - Operation: Read
     - Sheet name: `={{ $json.date }}`
     - Document ID: your Google Spreadsheet ID (connect Google Sheets OAuth2).
     - On error: continue error output.
   - Connect date node output to this.

4. **Count fetched posts (for tweet limit):**

   - Add a **Code** node named "Code (postsCount)":
     - JavaScript:  
       ```javascript
       return [{ json: { postsCount: items.length } }];
       ```
   - Connect from Google Sheets node.

5. **Set Twitter keyword:**

   - Add a **Set** node named "SET KEYWORD":
     - Assignment: `keyword` = `"YOUR_SEARCH_KEYWORD"`.
   - Connect from Code node.

6. **Search Twitter by keyword:**

   - Add an **HTTP Request** node "Twitter Search by Keyword":
     - URL: `https://api.twitterapi.io/twitter/tweet/advanced_search`
     - Method: GET
     - Query parameter: `query` = `={{ $json.keyword }} within_time:4h`
     - Authentication: HTTP Header Auth with twitterapi.io token.
   - Connect from SET KEYWORD.

7. **Extract and limit tweets:**

   - Add a **Code** node "Split tweets":
     - JavaScript to extract tweets array and limit to 2 or 3 tweets based on `postsCount`.
   - Connect from Twitter search node.

8. **Set up Anthropic Claude AI node:**

   - Add **Langchain Anthropic Chat Model** node:
     - Model: `claude-sonnet-4-20250514`
     - Enable "thinking" option.
     - Connect input from "Compose Comment" node (next step).

9. **Compose strategic comment:**

   - Add **Langchain Agent** node "Compose Comment":
     - Paste the detailed prompt text about social media strategy, brand promotion, fallback response, etc.
     - Configure system message: "You are a helpful twitter user that wants to share his good experience in using mentioned platform".
     - Connect input from "Split tweets" and AI node.

10. **Batch processing of tweets:**

    - Add **SplitInBatches** node "Loop Over Items":
      - Connect from "Compose Comment".
      - This node manages batch size and processing.

11. **Post replies on Twitter:**

    - Add **Twitter** node "Post on Twitter":
      - Text: `={{ $json.output }}`
      - Reply to tweet ID: `={{ $('Split tweets').item.json.id }}`
      - Twitter OAuth2 credentials.
      - Enable retry on fail with 2 max attempts, 5000ms wait.
      - Continue on error.
    - Connect from batch node.

12. **Wait between posts for rate limiting:**

    - Add **Wait** node "Wait":
      - Wait 8 minutes.
    - Connect from Twitter post node.
    - Connect back to batch node to process next items.

13. **Google Sheets reporting:**

    - Add **Google Sheets** node "Add info to the Report Table":
      - Operation: Append
      - Document ID and Sheet name as date.
      - Map columns: Keyword, Post URL, Our response.
      - Connect from batch processing node (parallel branch).

14. **If daily sheet missing, create and initialize:**

    - Add **Google Sheets** node "Create new sheet":
      - Operation: Create
      - Title: `={{ $now.format('dd-MM-yyyy') }}`
    - Connect from "Add info to the Report Table" error branch.

    - Add **Set** node "Define sheet headers":
      - Assign headers array: ["Post URL", "Our response", "Keyword"]
      - Assign date string.
    - Connect from "Create new sheet".

    - Add **HTTP Request** node "Add headers to the Sheet":
      - Method: PUT
      - URL: `https://sheets.googleapis.com/v4/spreadsheets/{{ spreadsheetId }}/values/{{ date }}!A:Z`
      - Body: Put headers array as raw values.
      - Authentication: Google Sheets OAuth2.
    - Connect from "Define sheet headers".

    - Add **Merge** node to combine header setup with append operations.
    - Connect outputs accordingly to ensure data appends after sheet creation.

15. **Add sticky notes for documentation:**

    - Add sticky notes summarizing configuration instructions, tutorial links, and block descriptions for user clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Connect your Google Account to all Google Sheets nodes. Tutorial: https://www.youtube.com/watch?v=pWGXlZBGu4k                                                                                                                                   | Sticky Note7                                                                                                    |
| Connect your Twitter account to 'Post on Twitter' node. Tutorial: https://www.youtube.com/watch?v=jxKKwYZQF7Q                                                                                                                                   | Sticky Note7                                                                                                    |
| Connect your twitterapi.io account to 'Twitter Search by Keyword' node. Tutorial: https://www.youtube.com/watch?v=lEo7IAgj0UY                                                                                                                  | Sticky Note7                                                                                                    |
| Connect your Anthropic account. Tutorial: https://www.youtube.com/watch?v=1jl_vBoVvq0                                                                                                                                                            | Sticky Note7                                                                                                    |
| Replace all [PLACEHOLDERS] in 'Compose Comment' node prompt with your company/brand/platform information before running.                                                                                                                        | Sticky Note7                                                                                                    |
| After all services connected, enter your keyword in 'SET KEYWORD' node and activate the workflow.                                                                                                                                               | Sticky Note7                                                                                                    |
| The best marketing doesn't feel like marketing; comments generated should feel like helpful advice with strategic imperfections and natural Twitter tone.                                                                                       | Prompt inside 'Compose Comment' node                                                                             |
| The workflow includes waits and retries to handle rate limits and transient errors gracefully.                                                                                                                                                   | General design                                                                                                  |
| Google Sheets API is used directly to add headers due to limitations in the default node for creating headers dynamically.                                                                                                                      | Block 2.2                                                                                                       |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.