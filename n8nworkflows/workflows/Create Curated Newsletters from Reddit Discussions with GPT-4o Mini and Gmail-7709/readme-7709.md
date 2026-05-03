Create Curated Newsletters from Reddit Discussions with GPT-4o Mini and Gmail

https://n8nworkflows.xyz/workflows/create-curated-newsletters-from-reddit-discussions-with-gpt-4o-mini-and-gmail-7709


# Create Curated Newsletters from Reddit Discussions with GPT-4o Mini and Gmail

---

### 1. Workflow Overview

This workflow automates the creation of curated newsletters from Reddit discussions using AI-powered summarization and email delivery. It targets content creators, marketers, or community managers who want to generate engaging newsletters based on Reddit posts and comments relevant to a specific topic.

**Key logical blocks:**

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Data Retrieval:** Fetch many new posts from a subreddit, then retrieve comments for each post.
- **1.3 Filtering & Topic Assignment:** Set and apply a topic of interest filter to posts.
- **1.4 Batch Processing:** Loop over filtered posts to process them individually.
- **1.5 Comments Cleaning & Merging:** Flatten and merge Reddit comments for easier analysis.
- **1.6 AI Summarization:** Use OpenAI GPT-4o Mini to extract insights and summarize posts and comments.
- **1.7 Summaries Merging:** Combine individual summaries into a consolidated list.
- **1.8 Newsletter Generation:** Convert summaries into a styled HTML newsletter via AI.
- **1.9 Email Sending:** Send the generated newsletter via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow manually by user action.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’
- **Node Details:**

| Node Name                      | Type                     | Configuration & Role                                                                                                          | Inputs       | Outputs             | Edge Cases / Failures                         |
|-------------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------|--------------|---------------------|-----------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger           | Triggers the entire workflow manually when clicked. No parameters needed.                                                    | None         | To "Get many posts"  | User forgets to trigger workflow; no input.  |

---

#### 2.2 Data Retrieval

- **Overview:** Fetch multiple new posts from the "microsaas" subreddit; later fetch all comments of each filtered post.
- **Nodes Involved:**  
  - Get many posts  
  - Select Top 10 Post  
  - Get post comments
- **Node Details:**

| Node Name           | Type           | Configuration & Role                                                                                           | Inputs                  | Outputs                                               | Edge Cases / Failures                                           |
|---------------------|----------------|---------------------------------------------------------------------------------------------------------------|-------------------------|-------------------------------------------------------|----------------------------------------------------------------|
| Get many posts       | Reddit API     | Retrieves all new posts from subreddit "microsaas" using OAuth2 credentials. No inputs needed.                | Trigger from manual     | List of posts                                        | Reddit API rate limits; invalid OAuth2 credentials; no posts.  |
| Select Top 10 Post   | Code           | Sort posts by upvotes, comments, and creation date; selects top 10 posts; outputs minimal fields + pairing.  | Items from "Get many posts" | Top 10 posts with ranking and minimal fields          | Input data malformed; missing expected fields; sorting errors. |
| Get post comments    | Reddit API     | For each post, fetches all comments using subreddit and post ID (dynamic expressions).                        | Single post item from loop | All comments for that post                             | Reddit API limits; invalid subreddit or post ID; empty comments.|

---

#### 2.3 Filtering & Topic Assignment

- **Overview:** Defines the topic of interest and filters posts strictly based on whether they match this topic using AI.
- **Nodes Involved:**  
  - Set topic of interest  
  - filter topic of interest  
  - String to Json  
  - If topic of interest
- **Node Details:**

| Node Name           | Type                       | Configuration & Role                                                                                                          | Inputs                        | Outputs                     | Edge Cases / Failures                                    |
|---------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------------|-------------------------------|-----------------------------|---------------------------------------------------------|
| Set topic of interest| Set                        | Assigns fixed topic "Strategies and tactics to get new customer" and attaches full post JSON to output for filtering.        | Top 10 posts from previous node | Post + topic object          | Input post JSON malformed; topic mismatch.              |
| filter topic of interest | OpenAI GPT-4o Mini     | Examines each post JSON to determine if it matches the topic strictly; adds boolean field "topic_of_interest".               | Post + topic object            | Same post JSON + topic_of_interest flag | API errors; ambiguous or vague topic matching.          |
| String to Json       | Code                       | Parses OpenAI response string into robust JSON object to handle different JSON formatting styles from AI output.             | AI output string               | Parsed JSON object           | Parsing errors on malformed JSON or unexpected formats. |
| If topic of interest | If (conditional)            | Checks if `topic_of_interest` is true to pass post to next processing stages; otherwise discards post.                        | Parsed JSON object             | Pass or reject branch        | Missing or invalid `topic_of_interest` field.            |

---

#### 2.4 Batch Processing (Loop Over Items)

- **Overview:** Processes each filtered post individually in batches to avoid overwhelming API limits or memory.
- **Nodes Involved:**  
  - Loop Over Items  
  - post
- **Node Details:**

| Node Name       | Type              | Configuration & Role                                                                                          | Inputs              | Outputs             | Edge Cases / Failures                        |
|-----------------|-------------------|--------------------------------------------------------------------------------------------------------------|---------------------|---------------------|---------------------------------------------|
| Loop Over Items | SplitInBatches    | Splits array of posts into individual items for sequential processing downstream. No batch size specified. | Filtered posts      | Single post per run  | Empty input array; batch size defaults.     |
| post            | Set               | Stores the entire JSON object of the current post for downstream use; copies input JSON as output.           | Single post item     | Same post JSON       | Input missing or malformed JSON.             |

---

#### 2.5 Comments Cleaning & Merging

- **Overview:** Processes Reddit comments to flatten nested replies, extract key metadata, then merges all comments of a post into a single list.
- **Nodes Involved:**  
  - Get post comments  
  - clean comments  
  - merge comments  
  - Merge
- **Node Details:**

| Node Name      | Type        | Configuration & Role                                                                                                        | Inputs                | Outputs               | Edge Cases / Failures                                 |
|----------------|-------------|----------------------------------------------------------------------------------------------------------------------------|-----------------------|-----------------------|------------------------------------------------------|
| Get post comments | Reddit API | Fetches all comments for a post with dynamic subreddit and post ID inputs.                                                   | Single post JSON       | Raw comments array     | API limits; empty or missing comments array.          |
| clean comments | Code        | JavaScript flattens nested Reddit comments to a simple list with fields like id, author, body, ups, permalink, depth, etc.  | Raw comments           | Flattened comment list | Comments missing expected fields; unexpected JSON.    |
| merge comments | Code        | Merges array of comment items into a single JSON object with a "list" field containing all comments for the post.           | Flattened comment items| Single item with list  | Input format errors; empty comment arrays.             |
| Merge          | Merge       | Combines two input streams by position: post data and merged comments, to unify data for summarization.                     | Post JSON + merged comments | Combined JSON object | Mismatched input lengths; data sync issues.            |

---

#### 2.6 AI Summarization

- **Overview:** Uses OpenAI GPT-4o Mini to analyze combined post and comments, extracting main insights, key learnings, and summaries focused on the topic of interest.
- **Nodes Involved:**  
  - Summarize post + comments  
  - merge summaries
- **Node Details:**

| Node Name             | Type                 | Configuration & Role                                                                                                           | Inputs                       | Outputs                    | Edge Cases / Failures                                      |
|-----------------------|----------------------|-------------------------------------------------------------------------------------------------------------------------------|------------------------------|----------------------------|-----------------------------------------------------------|
| Summarize post + comments | OpenAI GPT-4o Mini  | Takes combined post and comments JSON, analyzes and outputs structured summary JSON with fields: main_post_summary, insights, key_learnings | Combined post + comments JSON | Summary JSON               | API errors; incomplete input data; ambiguous topic focus. |
| merge summaries       | Code                 | Aggregates multiple summary JSON objects into a single object with a "list" array field containing all summaries.               | Array of summary JSON items   | Single combined summary list | Input format errors; empty inputs.                          |

---

#### 2.7 Newsletter Generation

- **Overview:** Converts the combined summaries list into a warm, engaging, and email-friendly HTML newsletter using GPT-4o Mini.
- **Nodes Involved:**  
  - create newsletter
- **Node Details:**

| Node Name       | Type                     | Configuration & Role                                                                                                    | Inputs                    | Outputs                 | Edge Cases / Failures                                  |
|-----------------|--------------------------|------------------------------------------------------------------------------------------------------------------------|---------------------------|-------------------------|-------------------------------------------------------|
| create newsletter | OpenAI GPT-4o Mini       | Receives JSON array of summaries, outputs an HTML formatted newsletter with structure, tone, emojis, and links per spec. | Combined summary list JSON | Newsletter HTML content | API errors; malformed input JSON; formatting issues.  |

---

#### 2.8 Email Sending

- **Overview:** Sends the generated newsletter HTML content via Gmail to the specified recipient.
- **Nodes Involved:**  
  - Send a message
- **Node Details:**

| Node Name     | Type          | Configuration & Role                                                                                          | Inputs                | Outputs          | Edge Cases / Failures                                   |
|---------------|---------------|--------------------------------------------------------------------------------------------------------------|-----------------------|------------------|--------------------------------------------------------|
| Send a message | Gmail node    | Sends email to `{{YOUR_EMAIL}}` with subject "Reddit Digest" and HTML body from newsletter generator node.  | Newsletter HTML       | Email sent status | Gmail OAuth2 auth failure; invalid email address; API limits.|

---

### 3. Summary Table

| Node Name                  | Node Type                     | Functional Role                                   | Input Node(s)                         | Output Node(s)                      | Sticky Note                                                                                                                           |
|----------------------------|-------------------------------|-------------------------------------------------|-------------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger              | Starts the workflow manually                     | None                                | Get many posts                    |                                                                                                                                     |
| Get many posts             | Reddit API                    | Retrieves new posts from subreddit "microsaas"  | When clicking ‘Execute workflow’    | Select Top 10 Post                | ## 📌 Get many posts: Retrieves multiple new posts from the "microsaas" subreddit; requires Reddit OAuth2 credentials.                |
| Select Top 10 Post         | Code                         | Selects top 10 posts by popularity and date      | Get many posts                     | Set topic of interest             | ## 📝 Select Top 10 Post: Sorts posts by ups, comments, and creation date; outputs minimal fields plus ranking.                      |
| Set topic of interest      | Set                          | Assigns fixed topic and post object               | Select Top 10 Post                 | filter topic of interest          | ## 📌 Set topic of interest: Sets the topic "Strategies and tactics to get new customer" for filtering and analysis.                   |
| filter topic of interest   | OpenAI GPT-4o Mini           | Filters posts to those matching the topic strictly | Set topic of interest              | String to Json                   | ## 🗂️ Filter Topic of Interest: AI node to add field `topic_of_interest` true/false based on strict relevance.                        |
| String to Json             | Code                         | Parses AI output string robustly into JSON        | filter topic of interest           | If topic of interest              | ## 🧑‍💻 String to Json: Parses various JSON formats from AI responses to robust JSON objects.                                           |
| If topic of interest       | If condition                 | Checks if post matches topic for further processing | String to Json                  | Loop Over Items                  | ## ⚖️ If topic of interest: Conditional branch based on `topic_of_interest` boolean to continue processing only relevant posts.       |
| Loop Over Items            | SplitInBatches               | Iterates over filtered posts individually         | If topic of interest              | merge summaries, post            | ## 🔄 Loop Over Items: Processes each post individually for detailed processing.                                                       |
| post                       | Set                          | Stores full JSON of current post                   | Loop Over Items (index 1)          | Get post comments                | ## 📌 Post: Stores the entire JSON object for the current post for further use.                                                       |
| Get post comments          | Reddit API                   | Fetches all comments for the current post          | post                             | clean comments                  | ## 📌 Get post comments: Retrieves comments for a post using dynamic subreddit and post ID; requires Reddit OAuth2 credentials.       |
| clean comments             | Code                         | Flattens nested comments to a simplified array     | Get post comments                 | merge comments                  | ## 🧹 Clean Comments: Flattens and extracts key data from nested Reddit comments and replies.                                         |
| merge comments             | Code                         | Combines all comment items into a single array     | clean comments                   | Merge                          | ## 📌 merge comments: Merges multiple comment items into a single array for easier downstream processing.                             |
| Merge                      | Merge                        | Combines post and merged comments JSON objects     | post, merge comments             | Summarize post + comments       | ## 📌 Merge: Combines post data and comments into one object for summarization.                                                       |
| Summarize post + comments  | OpenAI GPT-4o Mini           | Extracts insights and key learnings from post + comments | Merge                          | Loop Over Items (index 0)         | ## 📝 Summarize post + comments: AI node summarizing discussions into structured insights for newsletters.                          |
| merge summaries            | Code                         | Aggregates individual summaries into a list        | Summarize post + comments        | create newsletter              | ## 📌 Merge Summaries: Combines multiple summary objects into a single structured list for newsletter generation.                     |
| create newsletter          | OpenAI GPT-4o Mini           | Generates HTML newsletter from combined summaries  | merge summaries                 | Send a message                 | ## 📧 Create Newsletter: AI node generating warm, engaging HTML newsletter suitable for email clients.                               |
| Send a message             | Gmail                        | Sends the newsletter via Gmail to configured email | create newsletter              | None                          | ## 📩 Send a message: Sends email with newsletter content; requires Gmail OAuth2 credentials and correct email address.              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow on user command.

2. **Create Reddit Node to Get Many Posts**  
   - Type: Reddit API (Get All Posts)  
   - Parameters:  
     - Subreddit: microsaas  
     - Filters: category = new  
   - Credentials: Reddit OAuth2  
   - Connect: Manual Trigger → Get many posts

3. **Create Code Node to Select Top 10 Posts**  
   - Type: Code  
   - JS: Sort posts by ups, comments, created_utc; output top 10 with rank and minimal fields.  
   - Connect: Get many posts → Select Top 10 Post

4. **Create Set Node to Define Topic of Interest**  
   - Type: Set  
   - Assignments:  
     - topic = "Strategies and tactics to get new customer"  
     - post = current JSON item (={{$json}})  
   - Connect: Select Top 10 Post → Set topic of interest

5. **Create OpenAI Node to Filter Topic of Interest**  
   - Type: OpenAI GPT-4o Mini  
   - Prompt: Check if post relates strictly to topic; add field `topic_of_interest` true/false.  
   - Credentials: OpenAI API  
   - Connect: Set topic of interest → filter topic of interest

6. **Create Code Node to Parse AI Output to JSON**  
   - Type: Code  
   - JS: Parse message.content handling JSON, fenced code, escaped strings.  
   - Connect: filter topic of interest → String to Json

7. **Create If Node to Check Topic Match**  
   - Type: If  
   - Condition: `{{$json.topic_of_interest}}` equals true (boolean strict)  
   - Connect: String to Json → If topic of interest

8. **Create Loop (SplitInBatches) Node to Process Posts Individually**  
   - Type: SplitInBatches  
   - Connect: If topic of interest → Loop Over Items

9. **Create Set Node to Save Current Post JSON**  
   - Type: Set  
   - Assignments: Assign `post` = current item JSON (={{$json}})  
   - Connect: Loop Over Items (output 1) → post

10. **Create Reddit Node to Get Post Comments**  
    - Type: Reddit API (Get All Post Comments)  
    - Parameters:  
      - subreddit = `{{$json.post.subreddit}}`  
      - postId = `{{$json.post.id}}`  
      - limit = `{{$json.post.num_comments}}`  
    - Credentials: Reddit OAuth2  
    - Connect: post → Get post comments

11. **Create Code Node to Clean Comments**  
    - Type: Code  
    - JS: Flatten nested comments and replies into a simple array with metadata fields like id, author, body, ups, permalink, depth.  
    - Connect: Get post comments → clean comments

12. **Create Code Node to Merge Comments**  
    - Type: Code  
    - JS: Combine all comment items into one JSON object with a `list` field array.  
    - Connect: clean comments → merge comments

13. **Create Merge Node to Combine Post and Comments**  
    - Type: Merge (combine by position)  
    - Inputs: post node output and merge comments output  
    - Connect:  
      - post → Merge (input 1)  
      - merge comments → Merge (input 2)

14. **Create OpenAI Node to Summarize Post + Comments**  
    - Type: OpenAI GPT-4o Mini  
    - Prompt: Analyze post + comments JSON, output JSON with keys: topic, main_post_summary, comments_insights, key_learnings.  
    - Credentials: OpenAI API  
    - Connect: Merge → Summarize post + comments

15. **Connect Summarize post + comments output to Loop Over Items (output 0)**  
    - Connect Summarize post + comments → Loop Over Items (output 0)

16. **Create Code Node to Merge Summaries**  
    - Type: Code  
    - JS: Aggregate all summary items into a single JSON object with a `list` array.  
    - Connect: Loop Over Items (output 0) → merge summaries

17. **Create OpenAI Node to Create Newsletter**  
    - Type: OpenAI GPT-4o Mini  
    - Prompt: Convert JSON array "list" into warm, engaging HTML newsletter with structure, emojis, and references.  
    - Credentials: OpenAI API  
    - Connect: merge summaries → create newsletter

18. **Create Gmail Node to Send Message**  
    - Type: Gmail (Send Email)  
    - Parameters:  
      - To: `{{YOUR_EMAIL}}` (replace with actual recipient)  
      - Subject: "Reddit Digest" (or customizable)  
      - Message: `={{ $json.message.content }}` (newsletter HTML)  
    - Credentials: Gmail OAuth2  
    - Connect: create newsletter → Send a message

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow curates Reddit content focused on a defined topic, summarizing discussions with AI and delivering a newsletter by email. | Overview and purpose of workflow |
| Credentials required: Reddit OAuth2, OpenAI API key, Gmail OAuth2. Configure before running. | Setup instructions in sticky notes and "Sticky Note" node |
| Topic of interest should be specific to improve AI filtering accuracy and output relevance. | Configuration tip in "Set topic of interest" node |
| Rate limits from Reddit and OpenAI may affect performance; consider batching and error handling. | Known limitations |
| Output newsletter is HTML formatted for email clients, includes emojis and structured sections for readability. | Newsletter generation details |
| For questions or collaboration, contact via email or Twitter: [@guanchehacker](http://www.x.com/GuancheHacker) | Support and contact info from sticky notes |
| Test email sending with your own Gmail before deploying broadly to avoid delivery issues. | Gmail node setup tip |

---

**Disclaimer:** This document is based exclusively on an automated n8n workflow designed for lawful content curation and dissemination. It contains no illegal, offensive, or protected material. All data processed is publicly available and legal.

---