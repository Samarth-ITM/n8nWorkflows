Transform Reddit Discussions into LinkedIn Post Ideas with GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/transform-reddit-discussions-into-linkedin-post-ideas-with-gpt-4o-and-google-sheets-6383


# Transform Reddit Discussions into LinkedIn Post Ideas with GPT-4o and Google Sheets

### 1. Workflow Overview

This workflow is designed to transform Reddit discussions into actionable LinkedIn post ideas using GPT-4o and Google Sheets for content strategists, marketers, founders, and professionals in B2B, SaaS, and tech domains. By leveraging Reddit data and advanced AI analysis, it extracts pain points and insights from posts and comments, then generates multiple LinkedIn post topics complete with headlines, hooks, and CTAs. The results are stored in a dynamically created Google Sheet for easy review and tracking.

**Logical Blocks:**

- **1.1 Input Reception:** Captures user input via a form trigger for subreddit and keyword.
- **1.2 Reddit Data Retrieval:** Searches Reddit for posts matching criteria and fetches comments.
- **1.3 Data Aggregation & Filtering:** Filters posts with comments, aggregates comment texts per post.
- **1.4 AI Insight Generation:** Uses OpenAI GPT-4o to analyze posts and comments, extracting themes, pains, and insights.
- **1.5 AI Post Idea Generation:** Generates 3–5 LinkedIn post ideas based on AI insights.
- **1.6 Output Preparation and Storage:** Merges all results, formats output fields, and appends the data to a Google Sheet created at runtime.
- **1.7 Documentation and Notes:** Sticky notes provide contextual explanation and user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures user input through a web form where users specify the target subreddit and keyword for Reddit search.
- **Nodes Involved:** 
  - On form submission
- **Node Details:**

  - **On form submission**
    - Type: Form Trigger
    - Role: Entry point; collects "Subreddit" and "What to search for in the subreddit" from the user.
    - Configuration: Two form fields, both required; triggers workflow on submission.
    - Input: HTTP request from form submission.
    - Output: JSON containing subreddit and keyword.
    - Edge cases: Missing or invalid input will prevent triggering.
    - Credentials: None required.
  
#### 2.2 Reddit Data Retrieval

- **Overview:** Searches Reddit for posts matching the user’s input and fetches comments for each post.
- **Nodes Involved:** 
  - Create spreadsheet
  - Get Posts
  - Filter only posts with comments
  - Loop Over Items
  - Get many comments in a post
  - Aggregate comments to single field
- **Node Details:**

  - **Create spreadsheet**
    - Type: Google Sheets (Spreadsheet creation)
    - Role: Dynamically creates a spreadsheet named with the subreddit, keyword, and timestamp to store results.
    - Configuration: Title uses expressions to include parameters and timestamp.
    - Input: Form submission JSON.
    - Output: Spreadsheet metadata including URL.
    - Credentials: Google Sheets OAuth2.
    - Edge cases: API quota limits, permission issues.

  - **Get Posts**
    - Type: Reddit node (search posts)
    - Role: Search posts in the specified subreddit matching keyword.
    - Configuration: Uses expressions from form data for subreddit and keyword.
    - Input: Spreadsheet creation output (via form inputs).
    - Output: List of Reddit posts JSON.
    - Credentials: Reddit OAuth2.
    - Edge cases: No posts found, API limits, authentication failure.

  - **Filter only posts with comments**
    - Type: Filter
    - Role: Filters posts to only those having more than 0 comments.
    - Configuration: Checks if `num_comments` > 0.
    - Input: Posts from Reddit.
    - Output: Filtered posts.
    - Edge cases: Posts without comments are discarded.

  - **Loop Over Items**
    - Type: SplitInBatches
    - Role: Iterates over each filtered post to fetch comments individually.
    - Configuration: Default batch options.
    - Input: Filtered posts.
    - Output: One post per iteration.
    - Edge cases: Large number of posts could slow workflow.

  - **Get many comments in a post**
    - Type: Reddit node (get all comments)
    - Role: Retrieves all comments for the current post.
    - Configuration: Uses post ID and subreddit from current post.
    - Input: Post from Loop Over Items.
    - Output: Comments array.
    - Credentials: Reddit OAuth2.
    - Edge cases: Posts with very large comment trees might cause timeouts or partial fetches.

  - **Aggregate comments to single field**
    - Type: Aggregate
    - Role: Aggregates all comment bodies into an array for analysis.
    - Configuration: Aggregates on field "body".
    - Input: Comments from previous node.
    - Output: Aggregated comment array.
    - Edge cases: No comments or empty bodies.

#### 2.3 Data Aggregation & Filtering

- **Overview:** Prepares combined data structures per post for AI analysis.
- **Nodes Involved:** 
  - Construct post with comment object
- **Node Details:**

  - **Construct post with comment object**
    - Type: Set
    - Role: Packages post content, post ID, and aggregated comment texts into a single JSON object.
    - Configuration: Assigns `commentText` (array), `postId` (string), and `postContent` (string).
    - Input: Aggregated comments and current post data.
    - Output: Structured object for AI input.
    - Edge cases: Missing post content or empty comments can reduce AI insight quality.

#### 2.4 AI Insight Generation

- **Overview:** Uses GPT-4o to analyze Reddit post content and comments, extracting a central theme, five pain points, a narrative insight synthesis, and LinkedIn post angles.
- **Nodes Involved:** 
  - Generate insights
  - Merge insights with post content and comments
- **Node Details:**

  - **Generate insights**
    - Type: Langchain Agent (OpenAI GPT-4o)
    - Role: AI content analyst that processes post content and comments to extract structured insights.
    - Configuration: Custom prompt instructing extraction of central theme, 5 pains with quotes, synthesis paragraph, LinkedIn opportunity (headline, bullets, CTA), and relevance statement. Output in strict Markdown.
    - Input: JSON containing `postContent` and `commentText` array.
    - Output: Markdown string with analysis.
    - Credentials: OpenAI API key.
    - Edge cases: API latency, prompt failures, token limits.

  - **Merge insights with post content and comments**
    - Type: Merge
    - Role: Combines the AI-generated insights with original post and comment data for downstream use.
    - Configuration: Combine by position (index).
    - Input: AI insights and original post/comment JSON.
    - Output: Merged JSON object.
    - Edge cases: Mismatched array lengths could cause data misalignment.

#### 2.5 AI Post Idea Generation

- **Overview:** Based on insights, generates 3–5 LinkedIn post topics with headlines, key points, hooks, and CTAs.
- **Nodes Involved:** 
  - OpenAI Chat Model
  - Suggest post topics
  - Construct post ideas
  - Merge insights and post suggestions
- **Node Details:**

  - **OpenAI Chat Model**
    - Type: Langchain OpenAI Chat Model (GPT-4o-latest)
    - Role: Invokes GPT-4o conversational model to generate LinkedIn post ideas from AI-generated insights.
    - Configuration: Model set to "chatgpt-4o-latest" with no additional options.
    - Input: Output from "Generate insights".
    - Output: Message content with LinkedIn post topic suggestions in Markdown.
    - Credentials: OpenAI API.
    - Edge cases: Same as other OpenAI nodes (rate limits, token limits).

  - **Suggest post topics**
    - Type: OpenAI Node
    - Role: Another AI step prompting structured generation of 3–5 LinkedIn post topics, headlines, bullets, hooks, and CTAs.
    - Configuration: Custom prompt emphasizing B2B, SaaS, and tech relevance.
    - Input: Merged insights data.
    - Output: Markdown string with suggested topics.
    - Credentials: OpenAI API.
    - Edge cases: Prompt misinterpretation.

  - **Construct post ideas**
    - Type: Set
    - Role: Extracts AI message content to `postIdeas` string for downstream merging.
    - Configuration: Assigns from `$json.message.content`.
    - Input: AI post topic generation output.
    - Output: Cleaned post ideas string.
    - Edge cases: Null or malformed AI output.

  - **Merge insights and post suggestions**
    - Type: Merge
    - Role: Combines insights and LinkedIn post ideas into one object for final output.
    - Configuration: Combine by position.
    - Input: Constructed insights and post ideas.
    - Output: Combined data with full analysis and creative suggestions.
    - Edge cases: Position mismatch.

#### 2.6 Output Preparation and Storage

- **Overview:** Formats and appends all collected data (post content, comments, insights, LinkedIn ideas) to the Google Sheet created earlier.
- **Nodes Involved:** 
  - Prepare output columns
  - Output The Results
- **Node Details:**

  - **Prepare output columns**
    - Type: Set
    - Role: Maps merged data into specific named columns for Google Sheets.
    - Configuration: Assigns fields: "Post Content", "Comments" (joined with separators), "Insights", and "Linkedin post ideas".
    - Input: Merged insights and post suggestions.
    - Output: Formatted JSON for Google Sheets.
    - Edge cases: Large comment arrays may cause cell overflow.

  - **Output The Results**
    - Type: Google Sheets Append
    - Role: Appends the formatted data as new rows into the spreadsheet "Sheet1".
    - Configuration: Auto-mapping columns, appending data.
    - Input: Prepared columns from previous node.
    - Output: Sheet append response.
    - Credentials: Google Sheets OAuth2.
    - Edge cases: API quota, sheet write permissions, invalid data types.

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                          | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                           |
|--------------------------------|-----------------------------------|----------------------------------------|-------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------|
| On form submission              | Form Trigger                      | User input collection                   | —                             | Create spreadsheet             |                                                                                                     |
| Create spreadsheet              | Google Sheets (Spreadsheet)       | Creates spreadsheet for results        | On form submission            | Get Posts                     |                                                                                                     |
| Get Posts                      | Reddit node                      | Searches Reddit posts                   | Create spreadsheet            | Filter only posts with comments|                                                                                                     |
| Filter only posts with comments | Filter                          | Filters posts with >0 comments          | Get Posts                    | Loop Over Items               | # Fetch Posts and Comments                                                                           |
| Loop Over Items                | SplitInBatches                   | Iterates over posts to process one by one | Filter only posts with comments| Get many comments in a post   | # Fetch Posts and Comments                                                                           |
| Get many comments in a post    | Reddit node                      | Retrieves all comments for a post       | Loop Over Items              | Aggregate comments to single field| # Fetch Posts and Comments                                                                           |
| Aggregate comments to single field | Aggregate                      | Aggregates comment bodies into array    | Get many comments in a post  | Construct post with comment object| # Fetch Posts and Comments                                                                           |
| Construct post with comment object | Set                            | Packages post content and comments      | Aggregate comments to single field, Loop Over Items | Generate insights, Merge insights with post content and comments | # Fetch Posts and Comments                                                                           |
| Generate insights              | Langchain Agent (OpenAI GPT-4o) | Analyzes content, extracts pains & insights | Construct post with comment object | Merge insights with post content and comments | # Generate Insights                                                                                   |
| Merge insights with post content and comments | Merge                | Combines AI insights with original data | Generate insights, Construct post with comment object | Suggest post topics, Construct insights | # Generate Insights                                                                                   |
| OpenAI Chat Model              | Langchain OpenAI Chat Model       | Generates LinkedIn post ideas from insights | Merge insights with post content and comments | Generate insights             |                                                                                                     |
| Suggest post topics            | OpenAI Node                     | AI generates 3–5 LinkedIn post topics  | Merge insights with post content and comments | Construct post ideas           | # Generate Post ideas                                                                                 |
| Construct post ideas           | Set                             | Extracts AI post ideas text             | Suggest post topics          | Merge insights and post suggestions | # Generate Post ideas                                                                                 |
| Merge insights and post suggestions | Merge                        | Combines insights and LinkedIn ideas   | Construct insights, Construct post ideas | Prepare output columns        | # Generate Post ideas                                                                                 |
| Construct insights             | Set                             | Packages analysis output fields         | Merge insights with post content and comments | Merge insights and post suggestions | # Generate Insights                                                                                   |
| Prepare output columns         | Set                             | Formats data for Google Sheets          | Merge insights and post suggestions | Output The Results            |                                                                                                     |
| Output The Results             | Google Sheets Append             | Appends data to Google Sheet            | Prepare output columns       | Loop Over Items               |                                                                                                     |
| Sticky Note                   | Sticky Note                     | Explanatory note                        | —                           | —                           | # Generate Post ideas                                                                                 |
| Sticky Note1                  | Sticky Note                     | Explanatory note                        | —                           | —                           | # Generate Insights                                                                                   |
| Sticky Note2                  | Sticky Note                     | Explanatory note                        | —                           | —                           | # Fetch Posts and Comments                                                                           |
| Sticky Note3                  | Sticky Note                     | Workflow overview and instructions     | —                           | —                           | # Find trendy Linkedin post ideas from Reddit posts<br>Who’s it for, How it works, Setup, Customize |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure two required fields: "Subreddit" (text), "What to search for in the subreddit" (text).  
   - Use this as the workflow entry point.

2. **Create Google Sheets Node to Create Spreadsheet**  
   - Type: Google Sheets  
   - Operation: Create Spreadsheet  
   - Title Expression: `post_ideas_{{$json.Subreddit}}_{{$json["What to search for in the subreddit"]}}_{{$now.format("yyyy_MM_dd_HH_mm_ss")}}`  
   - Connect input from Form Trigger node.  
   - Use Google Sheets OAuth2 credentials.

3. **Create Reddit Node to Search Posts**  
   - Type: Reddit  
   - Operation: Search  
   - Subreddit: Use expression from form input.  
   - Keyword: Use expression from form input.  
   - Connect input from Google Sheets node (for passing form data).  
   - Use Reddit OAuth2 credentials.

4. **Create Filter Node to Keep Posts with Comments**  
   - Type: Filter  
   - Condition: `$json.num_comments > 0`  
   - Connect input from Reddit search node.

5. **Create SplitInBatches Node to Loop Over Posts**  
   - Type: SplitInBatches  
   - Default batch size.  
   - Connect input from Filter node.

6. **Create Reddit Node to Get All Comments of a Post**  
   - Type: Reddit  
   - Operation: Get All Comments for post  
   - Use expressions to get `postId` and `subreddit` from current post data.  
   - Connect input from SplitInBatches node.  
   - Use Reddit OAuth2 credentials.

7. **Create Aggregate Node to Collect Comment Bodies**  
   - Type: Aggregate  
   - Aggregate field: `body`  
   - Connect input from Get All Comments node.

8. **Create Set Node to Construct Post-Comment Object**  
   - Assign:  
     - `commentText` = aggregated array of comment bodies  
     - `postId` = current post’s ID  
     - `postContent` = current post’s text content (`selftext`)  
   - Connect input from Aggregate node and SplitInBatches node (for post details).

9. **Create Langchain Agent Node to Generate Insights**  
   - Type: Langchain Agent (GPT-4o)  
   - Prompt: Customized detailed prompt to extract central theme, 5 pains with quotes, synthesis, LinkedIn angle, relevance (as per original prompt).  
   - Input: JSON with `postContent` and `commentText`.  
   - Connect input from Set node.

10. **Create Merge Node to Combine Insights with Post Data**  
    - Mode: Combine by position  
    - Connect inputs from Langchain agent and Set node.

11. **Create OpenAI Chat Model Node to Generate LinkedIn Post Ideas**  
    - Model: `chatgpt-4o-latest`  
    - Input: Merged insights and post content.  
    - Connect input from Merge node.

12. **Create OpenAI Node to Suggest Post Topics**  
    - Prompt: Detailed instruction to generate 3–5 LinkedIn post topics with headlines, bullets, hooks, CTAs.  
    - Connect input from Merge node.

13. **Create Set Node to Extract Post Ideas**  
    - Assign `postIdeas` = AI message content containing LinkedIn post ideas.  
    - Connect input from Suggest Post Topics node.

14. **Create Merge Node to Combine Insights and Post Suggestions**  
    - Mode: Combine by position  
    - Connect inputs from Construct Insights node (see next) and Construct Post Ideas node.

15. **Create Set Node to Construct Insights**  
    - Assign fields:  
      - `contentAnalysis` = AI insights output  
      - `commentText` = comment array  
      - `postId`  
      - `postContent`  
    - Connect input from Merge insights with post content.

16. **Create Set Node to Prepare Output Columns for Google Sheets**  
    - Assign columns:  
      - "Post Content" from `postContent`  
      - "Comments" joined with separator `\n\n\n\n--------\n\n\n`  
      - "Insights" from `contentAnalysis`  
      - "Linkedin post ideas" from `postIdeas`  
    - Connect input from Merge insights and post suggestions node.

17. **Create Google Sheets Node to Append Data**  
    - Operation: Append  
    - Sheet Name: "Sheet1"  
    - Document ID: Use expression referencing spreadsheet URL from "Create spreadsheet" node.  
    - Configure auto-mapping columns.  
    - Connect input from Prepare Output Columns node.  
    - Use Google Sheets OAuth2 credentials.

18. **Connect Output The Results node back to Loop Over Items**  
    - To continue batch processing of posts.

19. **Add Sticky Notes for Documentation**  
    - Add explanatory notes at logical sections: Input, Fetching Posts and Comments, Generating Insights, Generating Post Ideas, Workflow Overview.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Built for practical content marketing automation targeting B2B, SaaS, and tech professionals.                                                       | Sticky Note3 content                                                                                  |
| Workflow relies on n8n credentials manager for Reddit, OpenAI, and Google Sheets integration; no hardcoded keys.                                    | Sticky Note3 content                                                                                  |
| Customize AI prompts to change tone, depth, or style of insights and post ideas.                                                                    | Sticky Note3 content                                                                                  |
| Potential enhancements: add filters for high-engagement posts, integrate Slack or Email notifications for insights.                               | Sticky Note3 content                                                                                  |
| Workflow transforms Reddit discussions into LinkedIn post ideas with detailed AI analysis extracting pain points and actionable content angles.    | Workflow Overview and Sticky Notes summary                                                          |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.