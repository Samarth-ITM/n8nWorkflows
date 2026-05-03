Automated Reddit Engagement with GPT-4.1 - Monitor Posts & Generate Smart Replies

https://n8nworkflows.xyz/workflows/automated-reddit-engagement-with-gpt-4-1---monitor-posts---generate-smart-replies-5894


# Automated Reddit Engagement with GPT-4.1 - Monitor Posts & Generate Smart Replies

### 1. Workflow Overview

This workflow automates monitoring Reddit posts in selected subreddits for discussions related to photo editing and AI image editing apps, evaluates their relevance to a specific iOS photo editing app, and generates tailored AI-driven replies to engage with users. Key use cases include automating social media engagement, targeted marketing without overt sales language, and maintaining a log of interactions.

Logical blocks:

- **1.1 Input Reception and Triggering:** Manual and scheduled triggers initiate the workflow to fetch Reddit posts.
- **1.2 Data Collection:** Multiple Reddit nodes search posts across different subreddits with relevant keywords.
- **1.3 Filtering and Selection:** Posts are filtered by engagement metrics, content presence, and recency.
- **1.4 AI Content Analysis:** AI evaluates whether posts are relevant for the app promotion.
- **1.5 Decision Making:** Posts confirmed relevant are passed for AI reply generation.
- **1.6 AI Reply Generation:** GPT-4.1 crafts a conversational reply aligned with business goals.
- **1.7 Posting and Logging:** Replies are posted as Reddit comments and recorded in a Google Sheet for tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

- **Overview:** This block starts the workflow either manually (for testing) or automatically every 3 hours.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Trigger: Run every 3 hours (Schedule Trigger)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Configuration: No parameters; allows manual start  
    - Inputs: None  
    - Outputs: Connects to multiple Reddit fetching nodes  
    - Edge Cases: User must manually start; no error expected here

  - **Trigger: Run every 3 hours**  
    - Type: Schedule Trigger  
    - Configuration: Interval set to every 3 hours  
    - Inputs: None  
    - Outputs: Triggers multiple Reddit fetching nodes simultaneously  
    - Edge Cases: Possible timing conflicts if execution exceeds 3 hours, but unlikely

#### 2.2 Data Collection

- **Overview:** Fetches Reddit posts from five subreddits using the same search query to gather relevant posts about photo editing apps.
- **Nodes Involved:**  
  - Get Posts  
  - Get Posts1  
  - Get Posts2  
  - Get Posts3  
  - Get Posts4

- **Node Details:**

  Each node is a **Reddit** node configured to:

  - Type: Reddit node (OAuth2 authenticated)  
  - Parameters:  
    - Operation: Search  
    - Keyword: `photo editing app OR image editor OR "AI photo" OR "photo filters"`  
    - Subreddit: Varies per node (`smallbusiness`, `sidehustle`, `smallbusiness` again, `indiehackers`, `ios`)  
    - Limit: 50 posts  
    - Sort: hot  

  - Inputs: Trigger nodes  
  - Outputs: Feed into "Filter Posts By Features" node  
  - Version-specific: Uses OAuth2 credentials; ensure valid token and scopes  
  - Edge Cases: API rate limits, empty search results, auth token expiry, network timeouts

#### 2.3 Filtering and Selection

- **Overview:** Filters posts to retain only those with positive engagement (upvotes > 0), non-empty body content, and created within the last 180 days.
- **Nodes Involved:**  
  - Filter Posts By Features  
  - Select Key Fields

- **Node Details:**

  - **Filter Posts By Features** (If node)  
    - Type: If node with multiple conditions combined with AND  
    - Conditions:
      - `ups > 0` (numeric filter)  
      - `selftext` is not empty (string not empty)  
      - `created` date is within last 180 days (date comparison using DateTime from seconds)  
    - Inputs: Reddit nodes  
    - Outputs: Only passing posts go to "Select Key Fields"  
    - Edge Cases: Posts without `selftext`, missing fields, date conversion errors

  - **Select Key Fields** (Set node)  
    - Type: Set node  
    - Configuration: Extracts key fields for downstream processing, including:  
      - upvotes, subreddit_subscribers, postcontent (from selftext), url, ISO date, id, title, subreddit  
    - Inputs: Filter Posts By Features  
    - Outputs: To "Analysis Content By AI"  
    - Edge Cases: Missing or malformed data; expressions rely on valid JSON path

#### 2.4 AI Content Analysis

- **Overview:** Uses GPT-4.1 to analyze if a post is relevant for promoting the iOS photo editing app based on content keywords and context.
- **Nodes Involved:**  
  - Analysis Content By AI  
  - OpenAI Chat Model (used as language model provider for the agent)

- **Node Details:**

  - **Analysis Content By AI** (Langchain agent)  
    - Type: Langchain Agent node configured with OpenAI GPT-4.1 model  
    - Prompt: Checks if the post content suggests the user might benefit from the app (mentions of photo editing, Picsart, image apps, or mobile editing)  
    - Output: "Yes" or "No"  
    - Inputs: Select Key Fields  
    - Outputs: Merge Input node  
    - Edge Cases: API errors, prompt failures, unexpected output format, rate limits

  - **OpenAI Chat Model** (Langchain LM Chat OpenAI)  
    - Type: OpenAI GPT-4.1 model node providing language model capability to the agent  
    - Inputs: Connected to Analysis Content By AI node  
    - Outputs: Provides AI evaluation

#### 2.5 Decision Making

- **Overview:** Combines results from AI analysis and filters posts to continue only those marked "Yes" for relevance.
- **Nodes Involved:**  
  - Merge Input  
  - Filter Posts By Content

- **Node Details:**

  - **Merge Input** (Merge node)  
    - Type: Merge node in “combine” mode including unpaired items  
    - Purpose: Combines multiple inputs presumably from AI analysis or different sources  
    - Inputs: Output of Analysis Content By AI  
    - Outputs: Filter Posts By Content  
    - Edge Cases: Unpaired data handling, mismatched input lengths

  - **Filter Posts By Content** (If node)  
    - Type: If node  
    - Condition: Output equals "Yes" (string equality)  
    - Inputs: Merge Input  
    - Outputs: To "Message a model" for reply generation  
    - Edge Cases: Output format mismatch, "Yes" string case sensitivity

#### 2.6 AI Reply Generation

- **Overview:** Generates a thoughtful, helpful reply to the Reddit post with natural mention of the iOS photo editing app if relevant.
- **Nodes Involved:**  
  - Message a model

- **Node Details:**

  - **Message a model** (Langchain OpenAI node)  
    - Type: Langchain OpenAI GPT-4.1 chat completion  
    - Prompt:  
      - Instructions to write a reply that acknowledges the problem, offers value, mentions the app only if relevant, asks follow-up questions, uses conversational tone, max 120 words  
      - Includes Reddit post content and title in prompt  
      - Rules to avoid marketing language, sound helpful  
    - Inputs: Filter Posts By Content (posts filtered as relevant)  
    - Outputs: To "Create a comment in a post" and "Append row in sheet"  
    - Edge Cases: Rate limits, prompt failures, irrelevant or incomplete replies

#### 2.7 Posting and Logging

- **Overview:** Posts the AI-generated reply as a comment on Reddit and logs all relevant post and response data in a Google Sheet for tracking.
- **Nodes Involved:**  
  - Create a comment in a post  
  - Append row in sheet

- **Node Details:**

  - **Create a comment in a post** (Reddit node)  
    - Type: Reddit node (OAuth2)  
    - Operation: Post a comment on the Reddit post using post ID and AI-generated reply text  
    - Inputs: Message a model  
    - Outputs: Append row in sheet  
    - Edge Cases: Posting errors, Reddit API limits, invalid post ID, comment text length limits

  - **Append row in sheet** (Google Sheets node)  
    - Type: Google Sheets node (OAuth2)  
    - Operation: Append a row with columns: Upvotes, Subreddit, Date_Found, Post_Title, Reddit_URL, Subscribers, Post_Content (trimmed 500 chars), Status Tracking, Posted_to_Reddit, AI_Generated_Reply  
    - Inputs: Message a model (after comment creation)  
    - Outputs: End of workflow  
    - Edge Cases: Sheet access errors, invalid data, API quota exceeded

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                      | Input Node(s)                            | Output Node(s)                    | Sticky Note                                                                                     |
|---------------------------|----------------------------------|------------------------------------|----------------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Manual start of workflow            | None                                   | Get Posts, Get Posts1, Get Posts2, Get Posts3, Get Posts4 |                                                                                                |
| Trigger: Run every 3 hours | Schedule Trigger                  | Scheduled start every 3 hours       | None                                   | Get Posts, Get Posts1, Get Posts2, Get Posts3, Get Posts4 |                                                                                                |
| Get Posts                 | Reddit                           | Fetch Reddit posts from subreddit  | When clicking ‘Test workflow’, Trigger: Run every 3 hours | Filter Posts By Features          |                                                                                                |
| Get Posts1                | Reddit                           | Fetch Reddit posts from subreddit  | When clicking ‘Test workflow’, Trigger: Run every 3 hours | Filter Posts By Features          |                                                                                                |
| Get Posts2                | Reddit                           | Fetch Reddit posts from subreddit  | When clicking ‘Test workflow’, Trigger: Run every 3 hours | Filter Posts By Features          |                                                                                                |
| Get Posts3                | Reddit                           | Fetch Reddit posts from subreddit  | When clicking ‘Test workflow’, Trigger: Run every 3 hours | Filter Posts By Features          |                                                                                                |
| Get Posts4                | Reddit                           | Fetch Reddit posts from subreddit  | When clicking ‘Test workflow’, Trigger: Run every 3 hours | Filter Posts By Features          |                                                                                                |
| Filter Posts By Features  | If                              | Filter posts by engagement, content, and recency | Get Posts, Get Posts1, Get Posts2, Get Posts3, Get Posts4 | Select Key Fields                 |                                                                                                |
| Select Key Fields         | Set                             | Extract key fields for AI analysis  | Filter Posts By Features                | Analysis Content By AI            |                                                                                                |
| Analysis Content By AI    | Langchain Agent                 | AI determines post relevance        | Select Key Fields                       | Merge Input                      |                                                                                                |
| OpenAI Chat Model         | Langchain LM Chat OpenAI         | Provides GPT-4.1 model for agent    | Connected to Analysis Content By AI    | Analysis Content By AI            |                                                                                                |
| Merge Input               | Merge                           | Combines AI results from multiple inputs | Analysis Content By AI                 | Filter Posts By Content           |                                                                                                |
| Filter Posts By Content   | If                              | Filter posts confirmed relevant     | Merge Input                            | Message a model                  |                                                                                                |
| Message a model           | Langchain OpenAI                 | Generate AI reply comment            | Filter Posts By Content                 | Create a comment in a post, Append row in sheet |                                                                                                |
| Create a comment in a post| Reddit                          | Post AI-generated comment on Reddit | Message a model                        | Append row in sheet               |                                                                                                |
| Append row in sheet       | Google Sheets                   | Log post and reply details          | Message a model, Create a comment in a post | None                           |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**

   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’` with default settings.
   - Add a **Schedule Trigger** node named `Trigger: Run every 3 hours` with an interval of 3 hours.

2. **Add Reddit Search Nodes**

   - For each subreddit (`smallbusiness`, `sidehustle`, `smallbusiness` again, `indiehackers`, `ios`), create a **Reddit** node named (`Get Posts`, `Get Posts1`, `Get Posts2`, `Get Posts3`, `Get Posts4`).
   - Configure each node:  
     - Operation: Search  
     - Keyword: `photo editing app OR image editor OR "AI photo" OR "photo filters"`  
     - Limit: 50  
     - Sort: hot  
     - Subreddit: respective value  
   - Connect both trigger nodes’ outputs to each of these Reddit nodes’ inputs.

3. **Add Filtering Nodes**

   - Add an **If** node named `Filter Posts By Features`.  
   - Set conditions:  
     - Number condition: `ups > 0`  
     - String condition: `selftext` not empty  
     - Date condition: `created` after (current date minus 180 days) (use DateTime expressions)  
   - Connect outputs of all Reddit nodes to this If node's inputs (fan-in).
   
   - Add a **Set** node named `Select Key Fields`.  
   - Assign fields:  
     - `upvotes`: from `ups`  
     - `subreddit_subscribers`  
     - `postcontent`: from `selftext`  
     - `url`  
     - `date`: convert `created` seconds to ISO string  
     - `id`  
     - `title`  
     - `subreddit`  
   - Connect If node’s "true" output to this Set node.

4. **Configure AI Analysis**

   - Add a **Langchain Agent** node named `Analysis Content By AI`.  
   - Use GPT-4.1 model via a connected **Langchain LM Chat OpenAI** node named `OpenAI Chat Model`.  
   - Prompt:  
     ```
     Reddit post: {{ $json.postcontent }}

     Is this post from someone who might benefit from an iOS image editing app? Consider if they mention photo editing, Picsart, image apps, or mobile editing.

     Output only yes or no
     ```
   - Connect `Select Key Fields` output to `Analysis Content By AI`.
   - Connect `OpenAI Chat Model` as the language model of the agent.

5. **Merge and Filter AI Responses**

   - Add a **Merge** node named `Merge Input` in combine mode with include unpaired enabled.  
   - Connect output of `Analysis Content By AI` to this node.
   
   - Add an **If** node named `Filter Posts By Content`.  
   - Condition: check if `output` property equals "Yes".  
   - Connect `Merge Input` output to this If node.

6. **Generate AI Reply**

   - Add a **Langchain OpenAI** node named `Message a model`.  
   - Model: GPT-4.1  
   - Prompt (single message):  
     ```
     Based on this Reddit post, write a helpful, conversational reply that:

     1. Acknowledges their specific problem
     2. Provides genuine value first
     3. Naturally mentions my iOS photo editing app https://apps.apple.com/app/prompt-pic/id6747992467 ONLY if relevant
     4. Asks a follow-up question
     5. Max 120 words, conversational tone

     Reddit Post: {{ $('Select Key Fields').item.json.postcontent }}
     Post Title: {{ $('Select Key Fields').item.json.title }}

     My app: Prompt Pic https://apps.apple.com/app/prompt-pic/id6747992467

     Rules:
     - Only mention app if relevant or inspiring.
     - No marketing language.
     - Sound helpful, not salesy.
     ```
   - Connect `Filter Posts By Content` true output to this node.

7. **Post Reply to Reddit**

   - Add a **Reddit** node named `Create a comment in a post`.  
   - Operation: Post comment on post.  
   - Parameters:  
     - `postId`: `={{ $('Select Key Fields').item.json.id }}`  
     - `commentText`: `={{ $json.message.content }}` (output from AI)  
   - Connect `Message a model` output to this node.

8. **Log Data to Google Sheets**

   - Add a **Google Sheets** node named `Append row in sheet`.  
   - Operation: Append row  
   - Document ID and Sheet Name: your Google Sheet ID and sheet gid=0  
   - Columns to append:  
     - Upvotes, Subreddit, Date_Found (current ISO datetime), Post_Title, Reddit_URL, Subscribers, Post_Content (truncate to 500 chars), Status Tracking ("New"), Posted_to_Reddit ("Yes"), AI_Generated_Reply (AI response)  
   - Connect `Create a comment in a post` output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow uses Reddit OAuth2 credentials; ensure your Reddit app has appropriate scopes for reading posts and posting comments. | Reddit API documentation: https://www.reddit.com/dev/api/                                           |
| Google Sheets OAuth2 credentials must have write access to the target spreadsheet.                                        | Google Sheets API: https://developers.google.com/sheets/api                                         |
| GPT-4.1 model utilized via Langchain nodes requires valid OpenAI API key with access to GPT-4.                            | OpenAI API docs: https://platform.openai.com/docs/models/gpt-4                                     |
| Prompt engineering is critical to avoid salesy or marketing tone in replies; adjust prompt as needed for tone refinement.|                                                                                                     |
| The workflow merges multiple subreddit searches allowing broad monitoring and aggregation.                               |                                                                                                |

---

**Disclaimer:**  
The provided description and analysis come exclusively from an automated n8n workflow. The workflow respects content policies and contains no illegal, offensive, or protected material. All processed data are legal and publicly accessible.