Find TikTok Video Questions from Keywords Using Dumpling AI + GPT-4

https://n8nworkflows.xyz/workflows/find-tiktok-video-questions-from-keywords-using-dumpling-ai---gpt-4-9249


# Find TikTok Video Questions from Keywords Using Dumpling AI + GPT-4

### 1. Workflow Overview

This workflow automates the process of discovering popular and frequently asked questions from TikTok video comments, based on user-provided keywords. It is designed for content strategists, marketers, or creators aiming to identify engaging questions or themes that viewers commonly ask on TikTok videos, thereby generating insights for content creation or audience engagement.

The workflow is logically divided into two main blocks:

- **1.1 TikTok User and Video Retrieval:**  
  Starting from keyword input, this block searches for TikTok users related to the keyword via Dumpling AI, limits the results, fetches each user’s recent videos, and prepares these videos for comment analysis.

- **1.2 Comment Analysis and Question Extraction:**  
  For each video retrieved, this block gathers all viewer comments using Dumpling AI, cleans and structures the comment data via a Python code node, then leverages GPT-4 to identify and rank the top recurring viewer questions. The final extracted questions, along with video URLs and keywords, are saved to a DataTable for review or further use.

---

### 2. Block-by-Block Analysis

#### 2.1 TikTok User and Video Retrieval

**Overview:**  
This block handles user input, searches TikTok for users matching the keyword using Dumpling AI, filters and limits users, fetches their profile videos, and splits the videos for downstream processing.

**Nodes Involved:**  
- Receive Keyword Input  
- Search TikTok Users (Dumpling AI)  
- Split Search Results into Users  
- Limit to 3 Users (Optional)  
- Loop Through TikTok Users  
- Wait to Respect Rate Limits  
- Get TikTok Profile Videos (Dumpling AI)  
- Split Videos from Profile  
- Loop Through Videos  
- Sticky Note (Branch 1 description)

**Node Details:**

- **Receive Keyword Input**  
  - Type: Form Trigger  
  - Role: Entry point; receives keywords submitted via a web form titled "Tik Tok Search" with a single field "Keywords".  
  - Config: Webhook enabled; form field labeled "Keywords".  
  - Input: External HTTP form submission  
  - Output: JSON with keyword(s) under `Keywords`  
  - Failure Modes: Webhook unreachable, malformed input, missing keyword field.

- **Search TikTok Users (Dumpling AI)**  
  - Type: HTTP Request  
  - Role: Calls Dumpling AI API to search TikTok users by keyword.  
  - Config: POST to Dumpling AI’s `/search-tiktok-users` endpoint, sending JSON body `{"query": "<keywords>"}`. Uses HTTP Header Authentication with Dumpling AI credentials.  
  - Input: JSON keyword from previous node  
  - Output: JSON response containing `userList` array with user data  
  - Failure Modes: API authentication failure, network timeout, empty or malformed response.

- **Split Search Results into Users**  
  - Type: Split Out  
  - Role: Splits the `userList` array into individual user items for batch processing.  
  - Config: Field to split out: `userList`  
  - Input: Dumpling AI user list JSON  
  - Output: Individual user JSON objects  
  - Edge Cases: Empty user list results in no further processing.

- **Limit to 3 Users (Optional)**  
  - Type: Limit  
  - Role: Restricts the number of users processed to 3 to control workflow load.  
  - Config: Default limit to 3 items (implicit)  
  - Input: Individual user items  
  - Output: Up to 3 users  
  - Edge Cases: Less than 3 users available; no effect.

- **Loop Through TikTok Users**  
  - Type: Split In Batches  
  - Role: Processes users one at a time to enable sequential API calls and rate limit management.  
  - Config: Default batch size 1  
  - Input: Up to 3 users  
  - Output: Single user JSON per iteration  
  - Edge Cases: Batch processing failure, infinite loop if not properly configured.

- **Wait to Respect Rate Limits**  
  - Type: Wait  
  - Role: Adds a 15-second pause between API calls to avoid hitting Dumpling AI rate limits.  
  - Config: Wait time = 15 seconds  
  - Input: User item from loop  
  - Output: Delayed user item  
  - Edge Cases: Long wait times may slow down workflow; no direct failure.

- **Get TikTok Profile Videos (Dumpling AI)**  
  - Type: HTTP Request  
  - Role: Retrieves a user’s TikTok videos from Dumpling AI by providing the user handle.  
  - Config: POST to `/get-tiktok-profile-videos` with body `{"handle": "<user_handle>"}`, authenticated via HTTP Header.  
  - Input: User object with `search_user_name` field  
  - Output: JSON with `aweme_list` containing videos  
  - Failure Modes: API failure, invalid user handle, empty video list.

- **Split Videos from Profile**  
  - Type: Split Out  
  - Role: Splits the list of videos (`aweme_list`) into individual video items for processing.  
  - Config: Field to split out: `aweme_list`  
  - Input: JSON with video list  
  - Output: Individual video JSON objects  
  - Edge Cases: No videos available; stops downstream processing.

- **Loop Through Videos**  
  - Type: Split In Batches  
  - Role: Processes videos one by one for comment extraction.  
  - Config: Default batch size 1  
  - Input: Individual video items  
  - Output: Single video JSON per iteration  
  - Edge Cases: Similar to previous loop node.

- **Sticky Note (Branch 1 description)**  
  - Content summarizes this block as the first branch handling search, user filtering, and video retrieval using Dumpling AI.

---

#### 2.2 Comment Analysis and Question Extraction

**Overview:**  
This block extracts comments from videos, cleans and consolidates comment text, uses GPT-4 to identify top viewer questions, and stores results in a DataTable.

**Nodes Involved:**  
- Loop Through Videos (entry into this branch)  
- Get Comments for Each Video  
- Extract Clean Comments (Python)  
- Find Top Viewer Questions (GPT-4)  
- Insert Result into DataTable  
- No Operation, do nothing (used for branching control)  
- Sticky Note1 (Branch 2 description)

**Node Details:**

- **Get Comments for Each Video**  
  - Type: HTTP Request  
  - Role: Calls Dumpling AI API to fetch comments for a given TikTok video URL.  
  - Config: POST to `/get-tiktok-video-comments` with body `{"url": "<video_share_url>"}`, authenticated with HTTP Header.  
  - Input: Video object with `share_info.share_url` field  
  - Output: JSON with nested comments data  
  - Failure Modes: API failures, empty comments, malformed video URL.

- **Extract Clean Comments (Python)**  
  - Type: Code (Python)  
  - Role: Recursively extracts comment text from raw nested JSON comments, including replies, consolidating into a flat list of texts.  
  - Config: Custom Python code that traverses comment objects under `comments` arrays, producing a list of `{text: comment_text}` objects under key `Comment`.  
  - Input: Raw comments JSON  
  - Output: JSON with `Comment` array containing cleaned comment texts  
  - Edge Cases: Unexpected JSON structures, missing fields, empty comments list.

- **Find Top Viewer Questions (GPT-4)**  
  - Type: OpenAI GPT-4 via Langchain node  
  - Role: Processes cleaned comments to identify the most frequently asked questions exactly as viewers wrote them, filtering out spam and unrelated chatter.  
  - Config: System prompt instructs to group similar questions, rank by frequency, and exclude non-question content; uses GPT-4.1 model.  
  - Input: JSON stringified cleaned comments array under `Comment`  
  - Output: Ranked list of top viewer questions (text output)  
  - Credential: Requires OpenAI API key with GPT-4 access  
  - Failure Modes: API rate limits, invalid API key, prompt formatting errors.

- **Insert Result into DataTable**  
  - Type: DataTable  
  - Role: Stores the final questions alongside the relevant video URLs and keywords for further review or reporting.  
  - Config: Maps columns: "Videos" from the current video's `share_info.share_url`, "Keywords" from the original keyword input. Uses a specific DataTable ID for storage.  
  - Input: JSON with extracted questions and context  
  - Output: Confirmation of data insertion  
  - Edge Cases: DataTable API errors, schema mismatches.

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Used to manage workflow paths, ensures proper branching and execution order; does not modify data.  
  - Input/Output: Pass-through  
  - Edge Cases: None.

- **Sticky Note1 (Branch 2 description)**  
  - Summarizes this block as the second branch that extracts comments, cleans data, uses AI to find top viewer questions, and stores results, ideal for content ideation.

---

### 3. Summary Table

| Node Name                         | Node Type                | Functional Role                                   | Input Node(s)                     | Output Node(s)                          | Sticky Note                                                                                                            |
|----------------------------------|--------------------------|-------------------------------------------------|----------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Receive Keyword Input             | Form Trigger             | Entry point: receives keyword input via form     | (Webhook trigger)                | Search TikTok Users (Dumpling AI)     | Branch 1 description: Trigger runs when a keyword is submitted via form                                                  |
| Search TikTok Users (Dumpling AI)| HTTP Request             | Searches TikTok users via Dumpling AI API        | Receive Keyword Input            | Split Search Results into Users       | Uses Dumpling AI to fetch TikTok profiles and their video content                                                       |
| Split Search Results into Users   | Split Out                | Splits user list array into individual users     | Search TikTok Users              | Limit to 3 Users (Optional)            |                                                                                                                        |
| Limit to 3 Users (Optional)       | Limit                    | Limits number of users processed to 3             | Split Search Results into Users  | Loop Through TikTok Users              |                                                                                                                        |
| Loop Through TikTok Users         | Split In Batches         | Processes users sequentially                       | Limit to 3 Users                | Split Videos from Profile, Wait to Respect Rate Limits |                                                                                                                        |
| Wait to Respect Rate Limits       | Wait                     | Adds delay to avoid API rate limits                | Loop Through TikTok Users        | Get TikTok Profile Videos (Dumpling AI)|                                                                                                                        |
| Get TikTok Profile Videos (Dumpling AI) | HTTP Request      | Fetches videos for each TikTok user               | Wait to Respect Rate Limits      | Loop Through TikTok Users              |                                                                                                                        |
| Split Videos from Profile         | Split Out                | Splits videos array into individual videos        | Loop Through TikTok Users        | Loop Through Videos                    |                                                                                                                        |
| Loop Through Videos               | Split In Batches         | Processes videos sequentially                      | Split Videos from Profile        | No Operation, do nothing; Get Comments for Each Video | Branch 1 description: Splits video list for individual processing                                                       |
| No Operation, do nothing          | NoOp                     | Workflow control node, no data modification       | Loop Through Videos              | (No further output)                    |                                                                                                                        |
| Get Comments for Each Video       | HTTP Request             | Retrieves comments for each video via Dumpling AI| Loop Through Videos              | Extract Clean Comments (Python)        |                                                                                                                        |
| Extract Clean Comments (Python)   | Code (Python)            | Cleans and extracts comment texts recursively     | Get Comments for Each Video      | Find Top Viewer Questions (GPT-4)     |                                                                                                                        |
| Find Top Viewer Questions (GPT-4) | OpenAI (GPT-4)           | Extracts top viewer questions from cleaned comments| Extract Clean Comments          | Insert Result into DataTable           | Branch 2 description: Uses GPT-4 to process clean text and extract top viewer questions                                 |
| Insert Result into DataTable      | DataTable                | Stores final questions and related data           | Find Top Viewer Questions        | Loop Through Videos                    | Branch 2 description: Final result is saved to DataTable                                                               |
| Sticky Note                      | Sticky Note              | Branch 1 descriptive note                          |                                  |                                       | Describes Branch 1: User search and video retrieval                                                                     |
| Sticky Note1                     | Sticky Note              | Branch 2 descriptive note                          |                                  |                                       | Describes Branch 2: Comment extraction and AI question analysis                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: `Form Trigger`  
   - Name: `Receive Keyword Input`  
   - Configure: Create a webhook form titled "Tik Tok Search" with a single text field named "Keywords".  
   - Save and activate webhook.

2. **Add HTTP Request Node to Search TikTok Users**  
   - Type: `HTTP Request`  
   - Name: `Search TikTok Users (Dumpling AI)`  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/search-tiktok-users`  
   - Authentication: HTTP Header Auth using Dumpling AI API credentials  
   - Body Parameters: JSON with `query` set to expression `={{ $json.Keywords }}`  
   - Connect output of `Receive Keyword Input` to this node.

3. **Add Split Out Node to Split User List**  
   - Type: `Split Out`  
   - Name: `Split Search Results into Users`  
   - Field to split out: `userList`  
   - Connect output of `Search TikTok Users` to this node.

4. **Add Limit Node to Limit Users to 3**  
   - Type: `Limit`  
   - Name: `Limit to 3 Users (Optional)`  
   - Default limit: 3 items  
   - Connect output of `Split Search Results into Users` to this node.

5. **Add Split In Batches Node to Loop Through Users**  
   - Type: `Split In Batches`  
   - Name: `Loop Through TikTok Users`  
   - Batch size: 1  
   - Connect output of `Limit to 3 Users` to this node.

6. **Add Wait Node to Respect Rate Limits**  
   - Type: `Wait`  
   - Name: `Wait to Respect Rate Limits`  
   - Wait time: 15 seconds  
   - Connect output of `Loop Through TikTok Users` to this node.

7. **Add HTTP Request Node to Get TikTok Profile Videos**  
   - Type: `HTTP Request`  
   - Name: `Get TikTok Profile Videos (Dumpling AI)`  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/get-tiktok-profile-videos`  
   - Authentication: HTTP Header Auth with Dumpling AI credentials  
   - Body Parameters: JSON with `handle` set to expression `={{ $json.user.search_user_name }}`  
   - Connect output of `Wait to Respect Rate Limits` to this node.

8. **Add Split Out Node to Split Videos**  
   - Type: `Split Out`  
   - Name: `Split Videos from Profile`  
   - Field to split out: `aweme_list`  
   - Connect output of `Loop Through TikTok Users` to this node (parallel path)  
   - Connect output of `Get TikTok Profile Videos` to `Loop Through TikTok Users` (to close loop).

9. **Add Split In Batches Node to Loop Through Videos**  
   - Type: `Split In Batches`  
   - Name: `Loop Through Videos`  
   - Batch size: 1  
   - Connect output of `Split Videos from Profile` to this node.

10. **Add HTTP Request Node to Get Comments for Each Video**  
    - Type: `HTTP Request`  
    - Name: `Get Comments for Each Video`  
    - Method: POST  
    - URL: `https://app.dumplingai.com/api/v1/get-tiktok-video-comments`  
    - Authentication: HTTP Header Auth with Dumpling AI credentials  
    - Body Parameters: JSON with `url` set to expression `={{ $json.share_info.share_url }}`  
    - Connect output of `Loop Through Videos` to this node.

11. **Add Python Code Node to Extract Clean Comments**  
    - Type: `Code` (Python)  
    - Name: `Extract Clean Comments (Python)`  
    - Paste the provided Python script that recursively extracts comment texts and replies into a flat list under key `Comment`.  
    - Connect output of `Get Comments for Each Video` to this node.

12. **Add OpenAI Node for GPT-4 Processing**  
    - Type: `OpenAI` (Langchain node)  
    - Name: `Find Top Viewer Questions (GPT-4)`  
    - Model: GPT-4.1  
    - Messages:  
      - System: Instructions to identify recurring viewer questions exactly as asked, group similar questions, rank by frequency, exclude spam, no answer generation.  
      - User prompt: Pass JSON-stringified comment data via expression `=JSON.stringify($json.Comment)`  
    - Credentials: Connect OpenAI API credentials with GPT-4 access.  
    - Connect output of `Extract Clean Comments (Python)` to this node.

13. **Add DataTable Node to Store Results**  
    - Type: `DataTable`  
    - Name: `Insert Result into DataTable`  
    - DataTable ID: Use or create a DataTable for "Tik Tok Keywords"  
    - Map columns:  
      - Videos: expression `={{ $('Loop Through Videos').item.json.share_info.share_url }}`  
      - Keywords: expression `={{ $json.message.content }}` (or from keyword input)  
    - Connect output of `Find Top Viewer Questions (GPT-4)` to this node.

14. **Add No Operation Node (Optional for Branch Control)**  
    - Type: `NoOp`  
    - Name: `No Operation, do nothing`  
    - Connect this node from `Loop Through Videos` as a parallel path if needed for workflow control.

15. **Add Sticky Notes for Documentation (Optional)**  
    - Add two sticky notes describing Branch 1 (user and video retrieval) and Branch 2 (comment extraction and AI analysis) for clarity in the editor.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Uses Dumpling AI API for TikTok data: user search, profile videos, video comments                    | Dumpling AI API documentation (external, not provided here)                                    |
| Uses OpenAI GPT-4 (via Langchain node) for natural language processing and question extraction       | Requires GPT-4 enabled OpenAI API subscription                                                  |
| Workflow is designed to respect Dumpling AI rate limits with built-in 15-second wait nodes           | Avoids API throttling or bans                                                                   |
| Python code node recursively extracts nested comments and replies, ensuring comprehensive comment data | Custom code provided in node configuration                                                     |
| Workflow input via web form trigger allows easy keyword submission for dynamic queries                | Webhook URL generated by n8n; can be embedded or called from external UIs                       |
| Results stored in n8n DataTable for easy visualization, filtering and export                          | DataTable ID must be created or existing before workflow use                                   |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.