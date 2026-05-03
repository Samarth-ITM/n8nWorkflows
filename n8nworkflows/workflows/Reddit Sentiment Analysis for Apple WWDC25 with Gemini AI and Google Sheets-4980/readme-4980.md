Reddit Sentiment Analysis for Apple WWDC25 with Gemini AI and Google Sheets

https://n8nworkflows.xyz/workflows/reddit-sentiment-analysis-for-apple-wwdc25-with-gemini-ai-and-google-sheets-4980


# Reddit Sentiment Analysis for Apple WWDC25 with Gemini AI and Google Sheets

### 1. Workflow Overview

This workflow automates the process of scraping Reddit posts related to Apple’s WWDC25 event, analyzing the sentiment of posts and comments using Google Gemini AI, and storing the enriched data in a Google Sheets spreadsheet for further analysis or reporting. It is designed for social media analysts, marketing teams, or AI-driven sentiment research focusing on event-specific public discussions.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Scraping Trigger:** Manual start and initiation of Reddit data scraping via BrightData API.
- **1.2 Scraping Status Monitoring:** Polling the scraping job status to know when data is ready.
- **1.3 Data Retrieval and Preprocessing:** Fetching the scraped Reddit data and preparing it for analysis.
- **1.4 Post Classification:** Using Google Gemini AI to classify posts as WWDC-related or not.
- **1.5 Comment Sentiment Analysis:** Looping through post comments to analyze sentiment with Google Gemini AI.
- **1.6 Data Formatting and Filtering:** Structuring sentiment data, filtering unclear results.
- **1.7 Data Storage:** Appending or updating the analyzed data into a Google Sheets document.
- **1.8 Control and Looping:** Handling iteration over posts and comments, including reclassification loops.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Scraping Trigger

- **Overview:** Starts the workflow manually and triggers Reddit scraping for WWDC25 posts using BrightData API.
- **Nodes Involved:** When clicking ‘Execute workflow’, scrap reddit
- **Node Details:**

  - **When clicking ‘Execute workflow’**
    - Type: Manual Trigger
    - Role: Entry point for manual execution.
    - Config: No parameters.
    - Inputs: None.
    - Outputs: Triggers "scrap reddit".
    - Edge cases: None (manual trigger).

  - **scrap reddit**
    - Type: HTTP Request
    - Role: Initiates a BrightData dataset job to scrape Reddit posts with keyword "WWDC25" from the past month, limiting to 100 posts sorted by newest.
    - Config: POST to https://api.brightdata.com/datasets/v3/trigger with JSON body specifying keywords, date range, and number of posts.
    - Headers: Authorization Bearer token (placeholder `{ your api key}`).
    - Query Parameters: dataset_id, include_errors, type, discover_by.
    - Inputs: From manual trigger.
    - Outputs: Passes snapshot_id for status check.
    - Edge cases: API auth failure, invalid API key, rate limits, malformed JSON body.
    - Sticky Note: "Scrap Linkedin for all relevant posts" (likely a mislabel, correct context is Reddit).

#### 2.2 Scraping Status Monitoring

- **Overview:** Polls BrightData API to check if scraping is complete; waits if still running.
- **Nodes Involved:** Get status, Switch, Wait
- **Node Details:**

  - **Get status**
    - Type: HTTP Request
    - Role: Fetches scraping job progress using snapshot_id from the previous step.
    - Config: GET to https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}.
    - Headers: Authorization Bearer token.
    - Inputs: From "scrap reddit" node.
    - Outputs: JSON with status field.
    - Edge cases: API errors, invalid snapshot_id, authorization failure.

  - **Switch**
    - Type: Switch (conditional branching)
    - Role: Routes workflow based on scraping status ("ready" or "running").
    - Config: Two cases - "ready" and "running" matching the status field.
    - Inputs: From "Get status".
    - Outputs: If "ready", proceed to "Get data"; if "running", route to "Wait".
    - Edge cases: Unexpected status values or missing status field.

  - **Wait**
    - Type: Wait
    - Role: Pauses workflow for 15 seconds before rechecking status.
    - Config: Waits 15 seconds.
    - Inputs: From "Switch" if status is "running".
    - Outputs: Loops back to "Get status".
    - Edge cases: Long wait times if scraping takes too long.

- Sticky Note: "Get the status of the scrap, IF running, we wait for a few seconds until the results are returned."

#### 2.3 Data Retrieval and Preprocessing

- **Overview:** Fetches scraped Reddit data and loops through posts to process them individually.
- **Nodes Involved:** Get data, Loop Over Items, Edit Fields, Split Out
- **Node Details:**

  - **Get data**
    - Type: HTTP Request
    - Role: Retrieves scraped Reddit posts in JSON format.
    - Config: GET https://api.brightdata.com/datasets/v3/snapshot/s_mbtsa7xhm9v2pdx3l (snapshot_id hardcoded here).
    - Query Parameter: format=json.
    - Headers: Authorization Bearer token.
    - Inputs: From "Switch" on "ready".
    - Outputs: Raw Reddit post data list.
    - Edge cases: API errors, snapshot expired, auth failure.
    - Sticky Note: "Fetch the response from the API call. This includes a list of all the reddit posts we have."

  - **Loop Over Items**
    - Type: SplitInBatches
    - Role: Iterates over each post in the data to process individually.
    - Config: Default batch size (not specified).
    - Inputs: From "Get data".
    - Outputs: Single post per iteration.
    - Edge cases: Empty dataset, batch size too large.

  - **Edit Fields**
    - Type: Set
    - Role: Extracts and normalizes key fields from each Reddit post for downstream processing.
    - Config: Sets fields like post_id, url, user_posted, title, description, num_comments, comments, community_description, community_name.
    - Inputs: From "Text Classifier".
    - Outputs: Passes normalized post data including comments array.
    - Edge cases: Missing fields in original data.

  - **Split Out**
    - Type: SplitOut
    - Role: Splits the comments array to process each comment separately.
    - Config: Field to split out is "comments".
    - Inputs: From "Edit Fields".
    - Outputs: Each comment as individual item.
    - Edge cases: Posts with no comments (empty array).

  - Sticky Note (Loop Over Items): "Loop items so as not to overwhelm the AI with multiple searched."

#### 2.4 Post Classification

- **Overview:** Classifies Reddit posts as related to WWDC events or other using Google Gemini AI; non-related posts re-queued for classification.
- **Nodes Involved:** Text Classifier, No category, Google Gemini Chat Model
- **Node Details:**

  - **Google Gemini Chat Model**
    - Type: Language Model Chat (Google Gemini)
    - Role: Performs AI-based classification of post text.
    - Config: Model "models/gemini-2.0-flash".
    - Credentials: Google Palm API credentials.
    - Inputs: Post title and description as input text.
    - Outputs: Classification result to Text Classifier.

  - **Text Classifier**
    - Type: Text Classifier (Langchain)
    - Role: Categorizes posts into "WWDC events" or "other".
    - Config: Categories defined with descriptions to distinguish Apple WWDC-related content.
    - InputText Expression: Combines post title and description.
    - Inputs: From Google Gemini Chat Model.
    - Outputs: On category "WWDC events," passes to Edit Fields; on "other" category, loops back to No category.

  - **No category**
    - Type: Set
    - Role: Sets a response string "No category" to re-enter loop.
    - Inputs: From Text Classifier when category is not WWDC.
    - Outputs: Loops back to "Loop Over Items".
    - Edge cases: Possible infinite loop if many posts classified as "other".
    - Sticky Note: "Text classification for only WWDC related events. We loop No category back to maintain the flow."

#### 2.5 Comment Sentiment Analysis

- **Overview:** Analyzes sentiment of each comment related to a WWDC post using AI, producing nuanced categories.
- **Nodes Involved:** edit for Sentiment analysis, Sentiment Analysis per comment, Google Gemini Chat Model1
- **Node Details:**

  - **edit for Sentiment analysis**
    - Type: Set
    - Role: Prepares comment data and post context (title, description) for sentiment analysis.
    - Config: Extracts comment text, user commenting, URL, post description, and title.
    - Inputs: From Split Out node.
    - Outputs: Prepared comment data.

  - **Google Gemini Chat Model1**
    - Type: Language Model Chat (Google Gemini)
    - Role: Performs sentiment analysis on comments.
    - Config: Model "models/gemini-2.0-flash-lite".
    - Credentials: Google Palm API credentials.
    - Inputs: From edit for Sentiment analysis.
    - Outputs: Sentiment category for each comment.

  - **Sentiment Analysis per comment**
    - Type: Sentiment Analysis (Langchain)
    - Role: Categorizes comments into detailed sentiments: Excited, Disappointed, Impressed, Indifferent, skeptical, Not clear.
    - Config: System prompt instructs AI to produce JSON output only and relate comments to post context.
    - Inputs: From Google Gemini Chat Model1.
    - Outputs: Sentiment results with multiple outputs merged later.
    - Edge cases: Ambiguous comments, AI output parsing errors.
    - Sticky Note: "Run Our Sentiment analysis via AI to understand what the users were feeling. This can be broken down into nuanced emotions to capture more details."

#### 2.6 Data Formatting and Filtering

- **Overview:** Formats sentiment data for each comment, merges all sentiment results, and filters out unclear sentiment outputs.
- **Nodes Involved:** Merge, format sentiment, Filter
- **Node Details:**

  - **Merge**
    - Type: Merge
    - Role: Combines multiple outputs of sentiment analysis into one data stream.
    - Config: Number of inputs: 6 (matches outputs from sentiment analysis).
    - Inputs: From Sentiment Analysis per comment.
    - Outputs: Consolidated sentiment data.

  - **format sentiment**
    - Type: Set
    - Role: Extracts relevant fields and sentiment category from merged data; associates comment with post.
    - Config: Sets fields like title, sentimentAnalysis, post_id, url, description, community_name, category, comment.
    - Inputs: From Merge.
    - Outputs: Formatted sentiment data.

  - **Filter**
    - Type: Filter
    - Role: Removes comments with unclear sentiment ("Not clear") to keep only definitive results.
    - Config: Condition excludes items where sentimentAnalysis contains "Not clear".
    - Inputs: From format sentiment.
    - Outputs: Filtered data to be stored.
    - Sticky Note: "Filter only items that are of a clear sentiment and updates these to a table."

#### 2.7 Data Storage

- **Overview:** Appends or updates the analyzed Reddit comments with sentiment data into a Google Sheets document.
- **Nodes Involved:** Append Sentiments
- **Node Details:**

  - **Append Sentiments**
    - Type: Google Sheets
    - Role: Appends or updates sheet rows with enriched sentiment data.
    - Config:
      - Operation: appendOrUpdate.
      - Sheet name: "Reddit Comments".
      - Document ID: Google Sheets document ID for "APPLE #WWDC25 Sentiments".
      - Columns: comment, user_commenting, url, description, title, sentimentAnalysis, post_id, community_name.
      - Matching column: url (to avoid duplicates).
    - Credentials: Google Sheets OAuth2.
    - Inputs: From Filter.
    - Outputs: Loops back to "Loop Over Items" to process next batch.
    - Edge cases: Google API quota limits, permission errors.
    - Sticky Note: None.

#### 2.8 Control and Looping

- **Overview:** Controls workflow flow with loops for classification and sentiment analysis batches.
- **Nodes Involved:** Loop Over Items, No category
- **Node Details:**

  - **Loop Over Items**
    - Type: SplitInBatches
    - Role: Controls iteration over posts and comments to avoid AI overload.
    - Inputs/outputs: Multiple connections for looping.

  - **No category**
    - Loops posts classified as "other" back to "Loop Over Items" to maintain flow.

- Sticky Notes:
  - "Loop items so as not to overwhelm the AI with multiple searched."
  - "Text classification for only WWDC related events. We loop No category back to maintain the flow."
  - "Simple data transformation."
  - "Merge the paths and sentiments."

---

### 3. Summary Table

| Node Name                  | Node Type                                | Functional Role                            | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                  |
|----------------------------|-----------------------------------------|-------------------------------------------|---------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                         | Manual start                              | None                            | scrap reddit                   |                                                                                                              |
| scrap reddit               | HTTP Request                            | Initiate Reddit scraping                   | When clicking ‘Execute workflow’ | Get status                    | Scrap Linkedin for all relevant posts (likely Reddit)                                                        |
| Get status                 | HTTP Request                            | Check scraping job status                  | scrap reddit                   | Switch                        | Get the status of the scrap, IF running, we wait for a few seconds until the results are returned.           |
| Switch                    | Switch                                  | Branch depending on scraping status       | Get status                    | Get data, Wait                |                                                                                                              |
| Wait                      | Wait                                    | Pause before rechecking scraping status   | Switch                        | Get status                   |                                                                                                              |
| Get data                  | HTTP Request                            | Fetch scraped Reddit posts                 | Switch                        | Loop Over Items              | Fetch the response from the API call. This includes a list of all the reddit posts we have                    |
| Loop Over Items           | SplitInBatches                          | Iterate over posts to process individually | Get data, No category, Append Sentiments | Text Classifier, (empty output), Text Classifier | Loop items so as not to overwhelm the AI with multiple searched                                               |
| Text Classifier           | Langchain Text Classifier               | Classify posts as WWDC related or other   | Loop Over Items               | Edit Fields, No category      | Text classification for only WWDC related events. We loop No category back to maintain the flow               |
| No category               | Set                                     | Handle posts not classified as WWDC       | Text Classifier               | Loop Over Items              | Text classification for only WWDC related events. We loop No category back to maintain the flow               |
| Edit Fields               | Set                                     | Normalize post data fields                 | Text Classifier               | Split Out                    | Simple data transformation                                                                                    |
| Split Out                 | SplitOut                                | Split comments array into individual items | Edit Fields                  | edit for Sentiment analysis   |                                                                                                              |
| edit for Sentiment analysis | Set                                    | Prepare comment and post context for AI   | Split Out                    | Sentiment Analysis per comment |                                                                                                              |
| Google Gemini Chat Model  | Langchain LM Chat (Google Gemini)       | AI model for post classification          | Text Classifier (ai_languageModel) | Text Classifier               |                                                                                                              |
| Google Gemini Chat Model1 | Langchain LM Chat (Google Gemini)       | AI model for comment sentiment analysis   | edit for Sentiment analysis (ai_languageModel) | Sentiment Analysis per comment |                                                                                                              |
| Sentiment Analysis per comment | Langchain Sentiment Analysis          | AI sentiment categorization of comments   | edit for Sentiment analysis, Google Gemini Chat Model1 | Merge (6 outputs)             | Run Our Sentiment analysis via AI to understand what the users were feeling. This can be broken down into nuanced emotions |
| Merge                     | Merge                                   | Combine multiple sentiment analysis outputs | Sentiment Analysis per comment | format sentiment             | Merge the paths and sentiments                                                                               |
| format sentiment          | Set                                     | Format sentiment data for storage          | Merge                        | Filter                       |                                                                                                              |
| Filter                    | Filter                                  | Exclude unclear sentiment results          | format sentiment             | Append Sentiments            | Filter only items that are of a clear sentiment and updates these to a table                                 |
| Append Sentiments         | Google Sheets                           | Append or update analyzed data in spreadsheet | Filter                       | Loop Over Items              |                                                                                                              |
| Sticky Note               | Sticky Note                            | Various contextual notes                    | None                         | None                        | Multiple notes scattered across workflow included above                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**
   - Name: When clicking ‘Execute workflow’
   - No parameters.

2. **Add an HTTP Request node for scraping trigger**
   - Name: scrap reddit
   - Method: POST
   - URL: https://api.brightdata.com/datasets/v3/trigger
   - Headers: Authorization: Bearer `{ your api key}`
   - Query Parameters:
     - dataset_id: gd_lvz8ah06191smkebj4
     - include_errors: true
     - type: discover_new
     - discover_by: keyword
   - Body Type: JSON
   - Body:
     ```
     [
       {
         "keyword": "WWDC25",
         "date": "Past month",
         "num_of_posts": 100,
         "sort_by": "New"
       }
     ]
     ```
   - Connect "When clicking ‘Execute workflow’" → "scrap reddit".

3. **Add HTTP Request node to get status**
   - Name: Get status
   - Method: GET
   - URL: https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}
   - Headers: Authorization: Bearer `{ your api key}`
   - Connect "scrap reddit" → "Get status"

4. **Add a Switch node**
   - Name: Switch
   - Property to check: `status` field in JSON
   - Rules:
     - If status equals "ready", route to "Get data"
     - If status equals "running", route to "Wait"
   - Connect "Get status" → "Switch"

5. **Add a Wait node**
   - Name: Wait
   - Duration: 15 seconds
   - Connect "Switch" (running) → "Wait"
   - Connect "Wait" → "Get status" (loop back for polling)

6. **Add HTTP Request node to get scraped data**
   - Name: Get data
   - Method: GET
   - URL: https://api.brightdata.com/datasets/v3/snapshot/{{snapshot_id}}
   - Query Parameters: format=json
   - Headers: Authorization Bearer `{ your api key}`
   - Connect "Switch" (ready) → "Get data"

7. **Add SplitInBatches node**
   - Name: Loop Over Items
   - Default batch size or configure as needed
   - Connect "Get data" → "Loop Over Items"

8. **Add Google Gemini Chat Model node (AI classification)**
   - Name: Google Gemini Chat Model
   - Model: models/gemini-2.0-flash
   - Credentials: Google Palm API (set up OAuth2 or API key)
   - Input Text: Combine post title and description
   - Connect "Loop Over Items" → "Google Gemini Chat Model"

9. **Add Text Classifier node**
   - Name: Text Classifier
   - Categories:
     - WWDC events (description: Apple WWDC related events)
     - other (description: non-Apple/WWDC posts)
   - Input text: Post title and description
   - Connect "Google Gemini Chat Model" (ai_languageModel) → "Text Classifier"

10. **Add Set node for posts classified as WWDC**
    - Name: Edit Fields
    - Extract and assign relevant fields from post JSON: post_id, url, user_posted, title, description, num_comments, comments, community_description, community_name.
    - Connect "Text Classifier" (WWDC events output) → "Edit Fields"

11. **Add SplitOut node**
    - Name: Split Out
    - Field to split out: comments
    - Connect "Edit Fields" → "Split Out"

12. **Add Set node to prepare comment data**
    - Name: edit for Sentiment analysis
    - Assign fields: comment, user_commenting, url, description (from Edit Fields), title (from Edit Fields)
    - Connect "Split Out" → "edit for Sentiment analysis"

13. **Add Google Gemini Chat Model node for sentiment**
    - Name: Google Gemini Chat Model1
    - Model: models/gemini-2.0-flash-lite
    - Credentials: Google Palm API
    - Connect "edit for Sentiment analysis" (ai_languageModel) → "Sentiment Analysis per comment"

14. **Add Sentiment Analysis node**
    - Name: Sentiment Analysis per comment
    - Categories: Excited, Disappointed, Impressed, Indifferent, skeptical, Not clear
    - System prompt: Instruct AI to output JSON only, relate comment to post context.
    - Inputs: From "edit for Sentiment analysis" and "Google Gemini Chat Model1"
    - Outputs: 6 outputs merged later.
    - Connect "Google Gemini Chat Model1" → "Sentiment Analysis per comment"

15. **Add Merge node**
    - Name: Merge
    - Number of inputs: 6 (match outputs from sentiment analysis)
    - Connect all outputs of "Sentiment Analysis per comment" to "Merge"

16. **Add Set node to format sentiment data**
    - Name: format sentiment
    - Assign fields from merged data: title, sentimentAnalysis, post_id, url, description, community_name, category, comment.
    - Connect "Merge" → "format sentiment"

17. **Add Filter node**
    - Name: Filter
    - Condition: sentimentAnalysis does NOT contain "Not clear"
    - Connect "format sentiment" → "Filter"

18. **Add Google Sheets node**
    - Name: Append Sentiments
    - Operation: appendOrUpdate
    - Spreadsheet ID: Your Google Sheets ID (e.g., "192dM77Cyn5Gei9GNLTRvQjmATLmKrVX3vyN03E4mW6Q")
    - Sheet name: "Reddit Comments"
    - Columns: comment, user_commenting, url, description, title, sentimentAnalysis, post_id, community_name
    - Matching columns: url
    - Credentials: Google Sheets OAuth2
    - Connect "Filter" → "Append Sentiments"
    - Connect "Append Sentiments" → "Loop Over Items" (to process next batch)

19. **Add Set node for "No category" loop**
    - Name: No category
    - Assign response: "No category"
    - Connect "Text Classifier" (other output) → "No category"
    - Connect "No category" → "Loop Over Items" (to continue processing posts)

20. **Final checks:**
    - Ensure all API keys and credentials are configured properly.
    - Snapshot_id usage in "Get data" should be dynamic (use data from scraping trigger).
    - Test workflow with smaller batches before full run.
    - Monitor for rate limits and handle errors gracefully.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The workflow is designed to analyze Reddit data for Apple WWDC25 event using AI-driven sentiment analysis and classification. | Workflow purpose overview.                                                                                |
| Beware of potential infinite loops if many posts are classified as "other" and re-queued repeatedly.                      | Relevant in No category looping logic.                                                                   |
| Google Gemini AI credentials require valid Google PaLM API access.                                                        | Credential setup in Google Gemini Chat Model nodes.                                                      |
| BrightData API keys must be valid and have permission to run dataset scraping jobs.                                        | API usage in scraping and status nodes.                                                                  |
| Google Sheets integration requires OAuth2 credentials with edit permissions on the target spreadsheet.                    | Append Sentiments node credential context.                                                               |
| Sticky notes in the workflow provide contextual descriptions for each functional block.                                    | See nodes named "Sticky Note" for inline documentation.                                                  |
| For more on n8n Langchain nodes and Google Gemini integration, consult official n8n documentation and Google PaLM API docs. | External resources for advanced customization and troubleshooting.                                       |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.