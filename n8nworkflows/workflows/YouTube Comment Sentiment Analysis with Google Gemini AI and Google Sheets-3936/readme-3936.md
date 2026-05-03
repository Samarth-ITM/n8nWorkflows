YouTube Comment Sentiment Analysis with Google Gemini AI and Google Sheets

https://n8nworkflows.xyz/workflows/youtube-comment-sentiment-analysis-with-google-gemini-ai-and-google-sheets-3936


# YouTube Comment Sentiment Analysis with Google Gemini AI and Google Sheets

### 1. Workflow Overview

This workflow automates the process of collecting YouTube video comments and analyzing their sentiment using Google Gemini AI or similar language models. It systematically retrieves all comments (including replies) from a specified video via the YouTube API, processes each comment individually, performs sentiment classification into Positive, Neutral, or Negative categories, and writes the results into a connected Google Sheet for reporting and visualization.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Initialization:** Manual start and initial data setup including video ID and previously saved comments.
- **1.2 Fetch & Prepare Comments:** Retrieve comments from YouTube API, parse and split them for batch processing, and save raw comments to Google Sheets.
- **1.3 Sentiment Analysis:** Analyze each comment’s sentiment, strength, and confidence using Google Gemini AI or other LLMs.
- **1.4 Update & Loop Handling:** Write back sentiment results to Google Sheets and handle pagination by looping through multiple API pages until all comments are processed.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger & Initialization

- **Overview:**  
  This block starts the workflow manually and prepares the input by retrieving the target YouTube video ID and loading any existing comments from Google Sheets.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Get comments (Google Sheets)  
  - ID Video (Set node)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Initiates workflow execution on user command.  
    - *Configuration:* Default manual trigger, no parameters.  
    - *Inputs:* None.  
    - *Outputs:* Triggers downstream nodes "Get comments" and "ID Video".  
    - *Failure Modes:* None typical, manual trigger requires user action.

  - **Get comments**  
    - *Type:* Google Sheets (Read)  
    - *Role:* Fetches existing comments data from the Google Sheet to avoid duplicates.  
    - *Configuration:* Reads from the designated spreadsheet and worksheet; expects columns including VIDEO_ID, COMMENTS, SENTIMENT, etc.  
    - *Inputs:* Triggered after manual start.  
    - *Outputs:* Provides existing data for batch processing.  
    - *Failure Modes:* Authentication errors with Google Sheets OAuth2; sheet not found; malformed data.

  - **ID Video**  
    - *Type:* Set node  
    - *Role:* Holds the YouTube video ID used to fetch comments.  
    - *Configuration:* Contains a fixed string parameter for the video ID (replaceable by user).  
    - *Inputs:* Triggered after manual start.  
    - *Outputs:* Provides the video ID to the "Get API Comments" node.  
    - *Failure Modes:* Empty or incorrect video ID leads to API errors downstream.

---

#### 2.2 Fetch & Prepare Comments

- **Overview:**  
  Retrieves comments from the YouTube Data API based on the video ID, parses the complex nested response to extract comments and replies, then splits them for individual processing. Raw comments are saved to Google Sheets for record keeping.

- **Nodes Involved:**  
  - Get API Comments (HTTP Request)  
  - Multipage? (If node)  
  - nextPageToken (Set node)  
  - Comments (Code node)  
  - Split comments (SplitOut node)  
  - Save comments (Google Sheets)

- **Node Details:**

  - **Get API Comments**  
    - *Type:* HTTP Request  
    - *Role:* Calls YouTube Data API's commentThreads endpoint to fetch comments for the specified video.  
    - *Configuration:* Uses HTTP GET with query parameters including video ID from ID Video node, API key in authentication, and pagination token if present.  
    - *Inputs:* Receives video ID and optionally nextPageToken.  
    - *Outputs:* Returns JSON containing comment threads, replies, nextPageToken.  
    - *Failure Modes:* API key invalid, quota exceeded, network timeout, invalid video ID, malformed response.

  - **Multipage?**  
    - *Type:* If node  
    - *Role:* Checks if the API response contains a nextPageToken to determine if more comment pages need fetching.  
    - *Configuration:* Condition evaluates presence of nextPageToken in previous node output.  
    - *Inputs:* Receives API response from "Get API Comments".  
    - *Outputs:* If true, proceeds to set nextPageToken and loop; else stops pagination loop.  
    - *Failure Modes:* Missing or unexpected token format.

  - **nextPageToken**  
    - *Type:* Set node  
    - *Role:* Stores the nextPageToken for use in the subsequent API request to fetch additional comment pages.  
    - *Configuration:* Extracts and saves nextPageToken from previous API response.  
    - *Inputs:* From "Multipage?" node true branch.  
    - *Outputs:* Passes token back to "Get API Comments" for next page fetch.  
    - *Failure Modes:* Token format invalid or expired leads to API errors.

  - **Comments**  
    - *Type:* Code node  
    - *Role:* Parses the nested API response to extract individual comments and replies into a flat list suitable for further processing.  
    - *Configuration:* Custom JavaScript code parses JSON structure, extracts text, author, timestamps, etc.  
    - *Inputs:* Receives API response JSON from "Get API Comments".  
    - *Outputs:* Array of comment objects.  
    - *Failure Modes:* Parsing errors if API response structure changes or malformed JSON.

  - **Split comments**  
    - *Type:* SplitOut node  
    - *Role:* Splits the array of comments into individual items to process separately downstream.  
    - *Configuration:* Default split by item.  
    - *Inputs:* Receives comments array from "Comments" node.  
    - *Outputs:* Individual comment objects emitted one by one.  
    - *Failure Modes:* Empty input array results in no output.

  - **Save comments**  
    - *Type:* Google Sheets (Write)  
    - *Role:* Saves raw comments to the Google Sheet, appending new entries.  
    - *Configuration:* Appends rows to the sheet with columns VIDEO_ID, COMMENTS, and others as needed.  
    - *Inputs:* Single comment item from "Split comments".  
    - *Outputs:* Confirms data saved.  
    - *Failure Modes:* Google Sheets auth failure, quota exceeded, invalid sheet structure.

---

#### 2.3 Sentiment Analysis

- **Overview:**  
  Each individual comment is analyzed by the AI model to classify sentiment and extract sentiment strength and confidence. This block uses Google Gemini AI integrated via Langchain nodes.

- **Nodes Involved:**  
  - Loop Over Comments (SplitInBatches node)  
  - Sentiment Analysis (Langchain sentimentAnalysis node)  
  - Google Gemini (Langchain lmChatGoogleGemini node)

- **Node Details:**

  - **Loop Over Comments**  
    - *Type:* SplitInBatches node  
    - *Role:* Processes comments in batches to avoid overloading AI API or hitting rate limits.  
    - *Configuration:* Batch size configured (default or set by user).  
    - *Inputs:* Receives all comments from Google Sheets read earlier.  
    - *Outputs:* Sends batches to Sentiment Analysis node; also outputs unprocessed batch for update.  
    - *Failure Modes:* Batch size too large causing timeouts; empty input batch.

  - **Sentiment Analysis**  
    - *Type:* Langchain sentimentAnalysis node  
    - *Role:* Classifies sentiment using AI; expected outputs: Positive, Neutral, Negative plus metrics.  
    - *Configuration:* Uses AI language model passed from "Google Gemini" node; may include prompt templates or custom instructions.  
    - *Inputs:* Receives individual comment text from "Loop Over Comments".  
    - *Outputs:* Sentiment classification results.  
    - *Failure Modes:* AI API errors, timeout, malformed input, quota limits.

  - **Google Gemini**  
    - *Type:* Langchain lmChatGoogleGemini node  
    - *Role:* Connects to Google Gemini API as the underlying language model for sentiment analysis.  
    - *Configuration:* Requires configured Google Gemini API credentials; handles chat-based prompts.  
    - *Inputs:* Receives prompt and comment data from Sentiment Analysis node.  
    - *Outputs:* Returns sentiment analysis results.  
    - *Failure Modes:* Credential errors, API limits, network issues.

---

#### 2.4 Update & Loop Handling

- **Overview:**  
  Updates Google Sheets with sentiment results for each comment and manages workflow looping for multi-page comment retrieval.

- **Nodes Involved:**  
  - Update sentiment (Google Sheets)  
  - nextPageToken (Set node) (also in Fetch block)  
  - Multipage? (If node) (also in Fetch block)

- **Node Details:**

  - **Update sentiment**  
    - *Type:* Google Sheets (Write)  
    - *Role:* Updates existing rows in Google Sheets with sentiment label, strength, confidence, and marks rows as processed.  
    - *Configuration:* Uses row IDs or unique identifiers to update correct rows.  
    - *Inputs:* Receives sentiment results from "Sentiment Analysis".  
    - *Outputs:* Confirmation of update.  
    - *Failure Modes:* Authentication errors, concurrency issues, row not found, quota limits.

  - **nextPageToken** and **Multipage?** nodes as detailed in block 2.2 handle continuing the loop until all comment pages are fetched and processed.

---

### 3. Summary Table

| Node Name             | Node Type                              | Functional Role                        | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                |
|-----------------------|--------------------------------------|-------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                     | Workflow manual start trigger        | None                          | Get comments, ID Video         |                                                            |
| Get comments          | Google Sheets                        | Reads existing comments from sheet   | When clicking ‘Test workflow’ | Loop Over Comments             |                                                            |
| ID Video              | Set                                 | Holds YouTube video ID                | When clicking ‘Test workflow’ | Get API Comments               |                                                            |
| Get API Comments      | HTTP Request                        | Fetches comments from YouTube API    | ID Video, nextPageToken        | Multipage?, Comments           |                                                            |
| Multipage?            | If                                 | Checks for nextPageToken for pagination | Get API Comments             | nextPageToken                  |                                                            |
| nextPageToken         | Set                                 | Stores nextPageToken for next API call | Multipage?                  | Get API Comments               |                                                            |
| Comments              | Code                                | Parses API response extracting comments | Get API Comments             | Split comments                |                                                            |
| Split comments        | SplitOut                            | Splits comments array into individual items | Comments                    | Save comments                 |                                                            |
| Save comments         | Google Sheets                       | Saves raw comments to Google Sheets  | Split comments                | None                          |                                                            |
| Loop Over Comments    | SplitInBatches                     | Processes comments in batches         | Get comments                  | Sentiment Analysis, Update sentiment (unprocessed) |                                                            |
| Sentiment Analysis    | Langchain sentimentAnalysis          | Classifies comment sentiment          | Loop Over Comments            | Update sentiment              |                                                            |
| Google Gemini         | Langchain lmChatGoogleGemini         | AI language model for sentiment analysis | Sentiment Analysis (ai_languageModel) | Sentiment Analysis          |                                                            |
| Update sentiment      | Google Sheets                       | Updates Google Sheet with sentiment results | Sentiment Analysis          | Loop Over Comments (for next batch) |                                                            |
| Sticky Note           | Sticky Note                        | (Empty content)                       | None                          | None                          |                                                            |
| QuickChart            | QuickChart                         | (Unused in this workflow)             | None                          | None                          |                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Add a Manual Trigger node named "When clicking ‘Test workflow’". No special configuration needed.

2. **Create Google Sheets node to read comments:**  
   - Add "Get comments" node of type Google Sheets (Read).  
   - Configure credentials for Google Sheets OAuth2.  
   - Set Spreadsheet ID to your YouTube Comments Sheet.  
   - Select worksheet with columns: VIDEO_ID, COMMENTS, SENTIMENT, STRENGTH, CONFIDENCE, DO.  
   - Configure to read all rows or filtered rows as needed.

3. **Create Set node for video ID:**  
   - Add "ID Video" Set node.  
   - Add a string parameter with your target YouTube video ID (replace default 'xxxxxxxx').

4. **Create HTTP Request node to fetch comments:**  
   - Add "Get API Comments" node of type HTTP Request (GET).  
   - Set URL to YouTube Data API commentThreads endpoint:  
     `https://www.googleapis.com/youtube/v3/commentThreads`  
   - Configure query parameters:  
     - `videoId` → Expression from "ID Video" node.  
     - `key` → Your YouTube API key (passed via Authentication or query param).  
     - `pageToken` → Expression from "nextPageToken" Set node (if exists).  
     - Other params: `part=snippet,replies`, `maxResults=100`.  
   - Use authentication with API key configured.

5. **Add If node for pagination check:**  
   - Add "Multipage?" If node.  
   - Condition: Check if `nextPageToken` exists in API response JSON.

6. **Add Set node for nextPageToken:**  
   - Add "nextPageToken" Set node.  
   - Set value to the token extracted from previous "Get API Comments" response.

7. **Connect nextPageToken back to Get API Comments** for looping.

8. **Add Code node to parse comments:**  
   - Add "Comments" Code node (JavaScript).  
   - Write code to extract all comments and replies from the nested API response into flat array of comments.  
   - Each comment object should include text, author, timestamp, video ID.

9. **Add SplitOut node to split comments:**  
   - Add "Split comments" node.  
   - Configure to split the array from the "Comments" node into single comment items.

10. **Add Google Sheets node to save raw comments:**  
    - Add "Save comments" Google Sheets node.  
    - Configure to append rows with relevant columns (VIDEO_ID, COMMENTS, etc.).  
    - Set credentials and spreadsheet info.

11. **Add SplitInBatches node to loop over comments:**  
    - Add "Loop Over Comments" SplitInBatches node.  
    - Configure batch size (e.g., 10-20).  
    - Input from "Get comments" (existing comments to process).

12. **Add Langchain Google Gemini node:**  
    - Add "Google Gemini" node (lmChatGoogleGemini).  
    - Configure with Google Gemini API credentials.  
    - Input prompt should request sentiment classification with strength and confidence.

13. **Add Langchain Sentiment Analysis node:**  
    - Add "Sentiment Analysis" node.  
    - Connect AI language model input from "Google Gemini" node.  
    - Pass individual comment text as input.

14. **Add Google Sheets node to update sentiment:**  
    - Add "Update sentiment" Google Sheets node.  
    - Configure to update existing rows with sentiment results, strength, confidence, and mark as processed.  
    - Use unique identifiers or row IDs from original comments.

15. **Connect nodes accordingly:**  
    - Manual Trigger → Get comments and ID Video.  
    - ID Video → Get API Comments.  
    - Get API Comments → Multipage? and Comments.  
    - Multipage? true → nextPageToken → Get API Comments (loop).  
    - Comments → Split comments → Save comments.  
    - Get comments → Loop Over Comments → Sentiment Analysis → Update sentiment.  
    - Google Gemini → Sentiment Analysis (as AI model).

16. **Credentials setup:**  
    - Configure Google API credentials for Google Sheets OAuth2.  
    - Configure YouTube API key in HTTP Request node.  
    - Configure Google Gemini API credentials in Langchain node.

17. **Test workflow:**  
    - Run manual trigger to validate.  
    - Check Google Sheets for saved comments and sentiment analysis output.

---

### 5. General Notes & Resources

| Note Content                                                                                            | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Clone the template Google Sheet before use: ensure columns VIDEO_ID, COMMENTS, SENTIMENT, STRENGTH, CONFIDENCE, DO exist. | https://docs.google.com/spreadsheets/d/14lAY5CjoYHqqFODiLGg636Gqw4JwWqb2CX10E8ZS6Uo/edit                   |
| Get YouTube API key from Google Developers Console.                                                    | https://developers.google.com/youtube/registering_an_application                                           |
| Contact for consulting and support: info@n3w.it                                                       | mailto:info@n3w.it                                                                                          |
| LinkedIn profile of author                                                                            | https://www.linkedin.com/in/davideboizza/                                                                  |
| Workflow handles pagination automatically by looping on nextPageToken until all comments are fetched. | Important for videos with many comments to avoid missing data.                                             |
| AI model can be switched or customized easily with Langchain nodes.                                   | Supports GPT, Claude, Gemini, or other LLMs depending on credentials and configuration.                     |
| Multi-language sentiment analysis possible by configuring AI prompt and model parameters.             | Useful for non-English comments or multilingual videos.                                                    |

---

This structured documentation enables developers and AI agents to fully understand, reproduce, and extend the YouTube comment sentiment analysis workflow using n8n with Google Gemini AI and Google Sheets integration.