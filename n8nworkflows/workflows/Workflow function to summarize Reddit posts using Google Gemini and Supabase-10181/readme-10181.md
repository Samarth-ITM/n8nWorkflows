Workflow function to summarize Reddit posts using Google Gemini and Supabase

https://n8nworkflows.xyz/workflows/workflow-function-to-summarize-reddit-posts-using-google-gemini-and-supabase-10181


# Workflow function to summarize Reddit posts using Google Gemini and Supabase

### 1. Workflow Overview

This workflow automates the process of summarizing Reddit posts using Google Gemini (PaLM) language models combined with Supabase for data storage. It is designed to:

- Periodically fetch the latest saved Reddit posts from a user's profile.
- Filter posts based on subreddit criteria and eliminate duplicates already stored in Supabase.
- Evaluate each post against a custom condition using an LLM.
- For posts meeting the condition, retrieve all associated comments.
- Extract and aggregate post and comment content into a single textual representation.
- Use Google Gemini to generate a concise summary and assign relevant tags.
- Store the summarized data back into Supabase.
- Include rate limiting pauses to avoid API quota issues.

The workflow is logically divided into these blocks:

- **1.1 Post Retrieval and Initial Filtering**
- **1.2 Post Evaluation with LLM Condition**
- **1.3 Comments Retrieval and Aggregation**
- **1.4 Post Summarization and Data Preparation**
- **1.5 Storage and Rate Limiting**

---

### 2. Block-by-Block Analysis

#### 1.1 Post Retrieval and Initial Filtering

**Overview:**  
This block obtains saved Reddit posts from the user’s profile, filters by subreddit and duplicates, and prepares posts for further processing.

**Nodes Involved:**  
- manual trigger when testing  
- check once a day if new post are available  
- Get post data from supabase  
- and extract their reddit id  
- get saved post from reddit profile  
- extract relevant attribut and filter posts based on subreddit  
- further filtering existing posts wrt existing posts in database  
- Loop Over Every Posts

**Node Details:**

- **manual trigger when testing**  
  - Type: Manual Trigger  
  - Role: Manual start point for testing the workflow.  
  - Config: No parameters.  
  - Inputs: None  
  - Outputs: Next node is "Get post data from supabase"

- **check once a day if new post are available**  
  - Type: Schedule Trigger  
  - Role: Automatically trigger workflow daily to check for new posts.  
  - Config: Interval set to once per day (default daily trigger).  
  - Inputs: None  
  - Outputs: Next node is "Get post data from supabase"

- **Get post data from supabase**  
  - Type: Supabase node (getAll)  
  - Role: Retrieves all existing Reddit posts from the Supabase database to identify duplicates.  
  - Config: Table: "reddit_posts", operation: getAll, returnAll: true, no filter.  
  - Credentials: Supabase API account.  
  - Inputs: Trigger nodes  
  - Outputs: Passes to "and extract their reddit id"

- **and extract their reddit id**  
  - Type: Code node  
  - Role: Extracts the "reddit_id" fields from Supabase result to create a list of saved post IDs.  
  - Config: JavaScript extracting reddit_id from all input items into an array.  
  - Inputs: Supabase data  
  - Outputs: Passes to "get saved post from reddit profile"

- **get saved post from reddit profile**  
  - Type: Reddit node  
  - Role: Fetches the last 10 saved posts from the Reddit profile.  
  - Config: Resource: Profile, Details: saved, Limit: 10.  
  - Credentials: Reddit OAuth2 account.  
  - Inputs: List of saved reddit IDs (from previous node).  
  - Outputs: Passes to "extract relevant attribut and filter posts based on subreddit"

- **extract relevant attribut and filter posts based on subreddit**  
  - Type: Code node  
  - Role: Extracts relevant fields from Reddit posts and filters posts by configured subreddits or keywords.  
  - Config:  
    - Accepted subreddits and keyword lists are empty arrays by default (customizable).  
    - Outputs only posts matching subreddit criteria with fields: id, title, description, subreddit, url, upvotes, num_comments, post_date (ISO string).  
  - Inputs: Reddit saved posts  
  - Outputs: Passes to "further filtering existing posts wrt existing posts in database"

- **further filtering existing posts wrt existing posts in database**  
  - Type: Code node  
  - Role: Filters out posts that are already in Supabase (by comparing reddit_ids).  
  - Config: Uses redditIds extracted earlier to exclude duplicates.  
  - Inputs: Newly fetched posts  
  - Outputs: Passes unique new posts to "Loop Over Every Posts"

- **Loop Over Every Posts**  
  - Type: Split In Batches  
  - Role: Processes each post individually in batch mode to handle posts one by one.  
  - Config: Default batch size 1 (implied).  
  - Inputs: Filtered posts  
  - Outputs: Triggers LLM1 for each post.

**Potential Failures:**  
- Supabase connection/authentication errors.  
- Reddit OAuth2 token expiration or rate limits.  
- Empty subreddit filter lists may exclude all posts unintentionally.  
- Large number of posts may require pagination or batch size tuning.

---

#### 1.2 Post Evaluation with LLM Condition

**Overview:**  
This block evaluates if each Reddit post meets a user-defined condition using an LLM, deciding if it should proceed for comment aggregation and summarization.

**Nodes Involved:**  
- LLM1 (Google Gemini Chat Model1)  
- If the condition is satisfied

**Node Details:**

- **LLM1 (Google Gemini Chat Model1)**  
  - Type: Langchain Google Gemini Chat Model  
  - Role: Asks a custom yes/no question about the Reddit post to determine relevance.  
  - Config: Model: "models/gemini-2.0-flash".  
  - Prompt: Checks if the Reddit post satisfies a user-defined condition (placeholder text `{YOUR CONDITION}`).  
  - Credentials: Google Gemini API.  
  - Inputs: Single Reddit post (batch).  
  - Outputs: To "If the condition is satisfied" node.

- **If the condition is satisfied**  
  - Type: If node  
  - Role: Branches workflow depending on LLM1 output - expects answer "YES" to continue with comment fetching, otherwise skips to next post.  
  - Config: Checks if LLM1 output text equals "YES" (case sensitive, strict validation).  
  - Inputs: LLM1 output.  
  - Outputs:  
    - True branch: "get all comments for the current post"  
    - False branch: Loops to next post in "Loop Over Every Posts"

**Potential Failures:**  
- LLM response may not be exactly "YES" or "NO", causing logic to fail (needs strict output parsing).  
- Google Gemini API rate limits or errors.  
- Misconfiguration of prompt could yield ambiguous answers.

---

#### 1.3 Comments Retrieval and Aggregation

**Overview:**  
For posts passing the condition, this block fetches all comments, extracts relevant attributes recursively including nested replies, and aggregates the entire content into a single text blob for summarization.

**Nodes Involved:**  
- get all comments for the current post  
- extract relevant attribut from comments  
- aggregate post and comment into a single text

**Node Details:**

- **get all comments for the current post**  
  - Type: Reddit node  
  - Role: Retrieves all comments for the current post using Reddit API.  
  - Config: Resource: postComment, Operation: getAll, ReturnAll: true, PostId and Subreddit dynamically from current post.  
  - Credentials: Reddit OAuth2 account.  
  - Inputs: From "If the condition is satisfied" true branch.  
  - Outputs: To "extract relevant attribut from comments"

- **extract relevant attribut from comments**  
  - Type: Code node  
  - Role: Recursively extracts comment body, author, upvotes, creation date, and nested replies into a structured JSON object.  
  - Config: Recursive JavaScript function handles infinite nested replies.  
  - Inputs: Raw Reddit comment data.  
  - Outputs: Structured comment data with nested replies.

- **aggregate post and comment into a single text**  
  - Type: Code node  
  - Role: Converts the post title, description, and all nested comments into a single formatted string suitable for input to the LLM summarizer.  
  - Config:  
    - Concatenates comments with indentation based on replies.  
    - Adds post title and description at the start.  
    - Includes counts and separation lines for readability.  
  - Inputs: Structured comments data and post data from "Loop Over Every Posts".  
  - Outputs: Single aggregated text string in JSON field `aggregated_text`.

**Potential Failures:**  
- Recursive comment extraction may hit stack limits on very deep nesting.  
- Large comment volumes could exceed LLM input token limits.  
- Reddit API downtime or rate limits may cause comment fetching failures.

---

#### 1.4 Post Summarization and Data Preparation

**Overview:**  
This block uses Google Gemini to summarize the aggregated post and comments text, extracts structured summary and tags, and prepares the data for Supabase insertion.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Structured Output Parser  
- LLM2 (chainLlm with output parser)  
- prepare data to be inserted in supabase

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini Chat Model  
  - Role: Summarizes the aggregated text into a short summary and assigns tags.  
  - Config: Model: "models/gemini-2.0-flash".  
  - Credentials: Google Gemini API.  
  - Inputs: Aggregated text from previous node.  
  - Outputs: Passes to "Structured Output Parser".

- **Structured Output Parser**  
  - Type: Langchain structured output parser  
  - Role: Parses Google Gemini output into JSON with fields `summary` and `tags` (comma separated).  
  - Config: JSON schema example provided for validation.  
  - Inputs: Raw LLM output.  
  - Outputs: Parsed structured data to "LLM2"

- **LLM2**  
  - Type: Langchain chainLlm  
  - Role: Passes parsed summary and tags further down the chain.  
  - Config: Custom system prompt and tagging options placeholders.  
  - Inputs: Parsed summary and tags.  
  - Outputs: To "prepare data to be inserted in supabase"

- **prepare data to be inserted in supabase**  
  - Type: Code node  
  - Role: Combines original post info with LLM-generated summary and tags, formatting tags as an array, ready for database insertion.  
  - Config: Extracts first item from batch, splits tags string into array, defaults summary and tags to empty if missing.  
  - Inputs: Post data and LLM2 output.  
  - Outputs: To "insert new reddit post"

**Potential Failures:**  
- Mismatch between LLM output and structured parser schema may cause parsing errors.  
- Tags missing or malformed could cause insertion issues.  
- Google Gemini API or network errors.

---

#### 1.5 Storage and Rate Limiting

**Overview:**  
This final block inserts the summarized Reddit post data into Supabase and includes a wait node to respect rate limiting policies before processing the next post.

**Nodes Involved:**  
- insert new reddit post  
- Wait (rate limiting)

**Node Details:**

- **insert new reddit post**  
  - Type: Supabase node (insert)  
  - Role: Inserts or updates the summarized Reddit post data into the "reddit_posts" table.  
  - Config: Table: "reddit_posts", data sent mapped automatically from input JSON.  
  - Credentials: Supabase API account.  
  - Inputs: Prepared post summary data.  
  - Outputs: To "Wait (rate limiting)"

- **Wait (rate limiting)**  
  - Type: Wait node  
  - Role: Pauses execution briefly to prevent hitting API rate limits.  
  - Config: Default wait time (not explicitly configured, can be customized).  
  - Inputs: After database insertion.  
  - Outputs: Loops back to "Loop Over Every Posts" to process next post.

**Potential Failures:**  
- Database insertion errors (e.g., constraint violations).  
- Wait node misconfiguration leading to too short or too long pauses.  
- Infinite loops if exit conditions are not carefully managed.

---

### 3. Summary Table

| Node Name                             | Node Type                                      | Functional Role                                            | Input Node(s)                                    | Output Node(s)                                  | Sticky Note                                                                                                     |
|-------------------------------------|------------------------------------------------|------------------------------------------------------------|-------------------------------------------------|------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| manual trigger when testing          | Manual Trigger                                 | Manual start for testing workflow                           | None                                            | Get post data from supabase                     | ## Get current reddit post from database                                                                       |
| check once a day if new post are available | Schedule Trigger                              | Daily automatic trigger                                    | None                                            | Get post data from supabase                     | ## Get the last 10 saved post (can be changed) - keep only relevant information about posts - filter subreddits |
| Get post data from supabase          | Supabase node                                  | Fetch all existing Reddit posts from DB                    | manual trigger, schedule trigger                 | and extract their reddit id                      |                                                                                                                |
| and extract their reddit id          | Code                                           | Extract reddit IDs from Supabase results                   | Get post data from supabase                      | get saved post from reddit profile               |                                                                                                                |
| get saved post from reddit profile   | Reddit node                                    | Fetch last 10 saved Reddit posts from user profile        | and extract their reddit id                       | extract relevant attribut and filter posts based on subreddit |                                                                                                                |
| extract relevant attribut and filter posts based on subreddit | Code                                           | Filter posts by subreddit and extract relevant fields     | get saved post from reddit profile               | further filtering existing posts wrt existing posts in database |                                                                                                                |
| further filtering existing posts wrt existing posts in database | Code                                           | Remove posts already saved in DB                            | extract relevant attribut and filter posts based on subreddit | Loop Over Every Posts                            |                                                                                                                |
| Loop Over Every Posts                | Split In Batches                               | Process posts sequentially one by one                       | further filtering existing posts wrt existing posts in database | LLM1, loop back on false condition               | ## Process each post one by one                                                                                  |
| LLM1                               | Google Gemini Chat Model                        | Check if post meets custom condition                        | Loop Over Every Posts                            | If the condition is satisfied                    | ## Check if a post satisfy a condition you want (can be changed)                                               |
| If the condition is satisfied        | If Node                                        | Branch based on LLM1 yes/no answer                          | LLM1                                             | get all comments for the current post, Loop Over Every Posts (false branch) |                                                                                                                |
| get all comments for the current post | Reddit node                                    | Retrieve all comments for the current post                  | If the condition is satisfied (true branch)     | extract relevant attribut from comments          | ## Fetch all comments for a post and aggregate them with post title/body                                        |
| extract relevant attribut from comments | Code                                           | Recursively extract comment data including nested replies  | get all comments for the current post            | aggregate post and comment into a single text    |                                                                                                                |
| aggregate post and comment into a single text | Code                                           | Concatenate post & comments into a formatted text          | extract relevant attribut from comments, Loop Over Every Posts | LLM2                                            |                                                                                                                |
| Google Gemini Chat Model             | Google Gemini Chat Model                        | Summarize aggregated text and assign tags                   | aggregate post and comment into a single text    | Structured Output Parser                          | ## Summarize the reddit post and its comments and prepare data to be inserted in supabase (can be changed)     |
| Structured Output Parser             | Langchain structured output parser             | Parse LLM summary output into JSON                           | Google Gemini Chat Model                         | LLM2                                            |                                                                                                                |
| LLM2                               | Langchain chainLlm                             | Pass parsed summary and tags                                 | Structured Output Parser                         | prepare data to be inserted in supabase           |                                                                                                                |
| prepare data to be inserted in supabase | Code                                           | Format post info with summary and tags for DB insertion    | LLM2, Loop Over Every Posts                      | insert new reddit post                            |                                                                                                                |
| insert new reddit post               | Supabase node                                  | Insert summarized post data into Supabase                   | prepare data to be inserted in supabase            | Wait (rate limiting)                             | ## Save in supabase and wait a bit to avoid rate limiting                                                    |
| Wait (rate limiting)                 | Wait node                                      | Pause to respect API rate limits                            | insert new reddit post                           | Loop Over Every Posts                            |                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create trigger nodes:**  
   - Add a **Schedule Trigger** node ("check once a day if new post are available") with daily interval to run automatically once per day.  
   - Add a **Manual Trigger** node ("manual trigger when testing") for manual workflow execution during development.

2. **Retrieve existing posts from Supabase:**  
   - Add a **Supabase** node ("Get post data from supabase") configured to:  
     - Operation: getAll  
     - Table: "reddit_posts"  
     - ReturnAll: true  
   - Connect both trigger nodes to this Supabase node.

3. **Extract reddit IDs from Supabase data:**  
   - Add a **Code** node ("and extract their reddit id") with this JavaScript snippet:  
     ```js
     const items = $input.all();
     const redditIds = items.map(item => item.json.reddit_id);
     return { redditIds };
     ```  
   - Connect from Supabase node.

4. **Fetch saved posts from Reddit profile:**  
   - Add a **Reddit** node ("get saved post from reddit profile") configured as:  
     - Resource: profile  
     - Details: saved  
     - Limit: 10  
   - Use OAuth2 Reddit credentials.  
   - Connect from "and extract their reddit id".

5. **Filter posts by subreddit and extract relevant fields:**  
   - Add a **Code** node ("extract relevant attribut and filter posts based on subreddit") with custom JS filtering posts by subreddit and extracting fields (id, title, description, subreddit, url, upvotes, num_comments, post_date).  
   - Initialize `acceptedSubReddits` and `subredditKeywords` arrays as per your requirements.  
   - Connect from Reddit node.

6. **Filter out posts already in Supabase:**  
   - Add a **Code** node ("further filtering existing posts wrt existing posts in database") with JS that excludes posts whose IDs are in the saved redditIds array passed as a parameter.  
   - Connect from previous filter node.

7. **Split posts into individual batches:**  
   - Add a **Split In Batches** node ("Loop Over Every Posts") with default batch size 1.  
   - Connect from previous filtering node.

8. **Evaluate each post with LLM condition:**  
   - Add a **Google Gemini Chat Model** node ("LLM1") using the "models/gemini-2.0-flash" model and Google Palm API credentials.  
   - Configure prompt to ask a yes/no question about the post content (title + description).  
   - Connect from "Loop Over Every Posts".

9. **Add conditional branching:**  
   - Add an **If** node ("If the condition is satisfied") that checks if the response from LLM1 equals "YES".  
   - Connect from LLM1.

10. **Fetch comments for posts passing condition:**  
    - Add a **Reddit** node ("get all comments for the current post") configured to get all comments of the current post by dynamic PostId and Subreddit.  
    - Connect the true branch from "If the condition is satisfied".

11. **Extract relevant attributes from comments:**  
    - Add a **Code** node ("extract relevant attribut from comments") with recursive JS function to parse nested replies and extract fields (body, upvotes, author, created_utc, replies).  
    - Connect from Reddit comments node.

12. **Aggregate post and comment text:**  
    - Add a **Code** node ("aggregate post and comment into a single text") that concatenates post title, description, and recursively formatted comments text into a single string.  
    - Connect from "extract relevant attribut from comments" and also access current post data from "Loop Over Every Posts".

13. **Summarize aggregated text with Google Gemini:**  
    - Add a **Google Gemini Chat Model** node ("Google Gemini Chat Model") with the same model and credentials.  
    - Configure prompt to summarize in less than 300 words and assign tags from a predefined list.  
    - Connect from aggregation node.

14. **Parse structured summary output:**  
    - Add a **Structured Output Parser** node ("Structured Output Parser") with JSON example schema containing `summary` and `tags`.  
    - Connect from Google Gemini summarization node.

15. **Chain LLM output for further processing:**  
    - Add a **Langchain chainLlm** node ("LLM2") to process parsed output with custom system prompt.  
    - Connect from structured output parser.

16. **Prepare data for Supabase insertion:**  
    - Add a **Code** node ("prepare data to be inserted in supabase") that merges original post data with summary and tags (splitting tags string into array).  
    - Connect from LLM2 and "Loop Over Every Posts".

17. **Insert summarized post into Supabase:**  
    - Add a **Supabase** node ("insert new reddit post") configured to insert into "reddit_posts" table with auto mapping of input JSON.  
    - Connect from preparation code node.

18. **Add rate limiting wait:**  
    - Add a **Wait** node ("Wait (rate limiting)") to pause between insertions.  
    - Connect from Supabase insert node.

19. **Loop back to process next post:**  
    - Connect the output of Wait node back to "Loop Over Every Posts" node to continue batch processing.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                            |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| The workflow uses Google Gemini API (PaLM) for natural language processing tasks including conditional checks and summarization. | Google Gemini (PaLM) API credentials required.             |
| Reddit OAuth2 credentials are mandatory for accessing user profile saved posts and fetching comments.    | Reddit API OAuth2 setup required with appropriate scopes.  |
| Supabase serves as the persistent storage for posts and summaries, requiring proper table setup with fields such as reddit_id, title, url, summary, tags, post_date, upvotes, num_comments. | Supabase project and table configuration needed.           |
| Rate limiting is managed via a Wait node to avoid exceeding API quotas, adjust wait durations as needed. | Adjust wait times based on API usage patterns and limits.  |
| Custom prompts and conditions should be updated in LLM nodes to fit specific use cases and filtering needs. | Replace `{YOUR CONDITION}` and `{YOUR CUSTOM TAGS}` placeholders in prompts accordingly. |
| Recursive comment extraction handles deeply nested Reddit comment trees, but extremely large threads may require additional handling to avoid timeouts or memory issues. | Monitor and tune for performance with large Reddit posts.  |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.