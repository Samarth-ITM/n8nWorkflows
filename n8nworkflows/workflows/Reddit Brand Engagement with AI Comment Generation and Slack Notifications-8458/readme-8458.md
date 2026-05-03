Reddit Brand Engagement with AI Comment Generation and Slack Notifications

https://n8nworkflows.xyz/workflows/reddit-brand-engagement-with-ai-comment-generation-and-slack-notifications-8458


# Reddit Brand Engagement with AI Comment Generation and Slack Notifications

### 1. Workflow Overview

This workflow automates Reddit brand engagement by searching for recent posts in specific sales and AI-related subreddits, analyzing their content with an AI agent to decide whether to reply, generating authentic AI-crafted comment suggestions, and then forwarding these suggestions to a Slack channel for human review or posting. It is designed to assist sales development representatives (SDRs) and marketing teams by automating lead engagement and nurturing via Reddit, while maintaining authenticity and trust.

Logical blocks:

- **1.1 Input Reception & Scheduling:** Periodic triggering and Reddit search nodes fetch recent posts from targeted subreddits.
- **1.2 Data Aggregation & Filtering:** Merging results and filtering posts to only those from the last 24 hours.
- **1.3 AI Comment Generation:** Using an AI agent with a custom prompt to analyze posts and generate recommended comments.
- **1.4 Reply Decision Filtering:** Filtering AI outputs to decide which posts warrant a reply.
- **1.5 Notification Delivery:** Sending Slack messages containing comment ideas for approved posts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Scheduling

**Overview:**  
This block triggers the workflow daily at a set hour and queries multiple selected subreddits for recent posts matching specified keywords.

**Nodes Involved:**  
- Schedule Trigger  
- Automate (Reddit)  
- AI_Agents (Reddit)  
- salesdevelopment (Reddit)  
- SaaSSales (Reddit)  
- AskMarketing (Reddit)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow every day at 19:00 (7 PM).  
  - Configuration: Trigger at hour 19, no additional filters.  
  - Inputs: None (trigger node).  
  - Outputs: Connected to Reddit search nodes.  
  - Failure cases: Scheduling misconfiguration, missing system time zone setup.

- **Reddit Nodes (Automate, AI_Agents, salesdevelopment, SaaSSales, AskMarketing)**  
  - Type: Reddit  
  - Role: Search posts in specified subreddits with keyword filters.  
  - Configuration:  
    - Automate: Search “Automate” subreddit for keywords “AI Agent, AI, Automation, Sales”, limit 5, sorted by new.  
    - AI_Agents: Search “AI_Agents” subreddit for keywords “Sales, Sales Automation, Website Leads, Lead Qualification”, limit 5, sorted by new.  
    - salesdevelopment: Search “salesdevelopment” subreddit for “AI Agent, AI, Automation”, limit 5, sorted by new.  
    - SaaSSales: Search “Automate” subreddit (possible copy-paste error, though named SaaSSales) with keywords “AI, Automation, AI Agents”, limit 5, sorted by new.  
    - AskMarketing: Search “AskMarketing” subreddit for “AI, Automation, AI Agents”, limit 5, no sort specified (default).  
  - Credentials: Reddit OAuth2 API with valid authentication.  
  - Inputs: From Schedule Trigger.  
  - Outputs: Each outputs found posts to the Merge node.  
  - Edge cases: API rate limits, OAuth token expiration, keyword misalignment, subreddit restrictions.

#### 2.2 Data Aggregation & Filtering

**Overview:**  
Combines posts from all subreddit searches, then filters to only keep posts created in the last 24 hours.

**Nodes Involved:**  
- Merge  
- Filter last 24 hour posts (Code node)

**Node Details:**

- **Merge**  
  - Type: Merge  
  - Role: Combines multiple Reddit search results into one unified dataset.  
  - Configuration: Number of inputs set to 5 (one for each Reddit search node).  
  - Inputs: Reddit search nodes outputs.  
  - Outputs: To the filtering code node.  
  - Failure cases: Input data structure mismatch, empty inputs.

- **Filter last 24 hour posts**  
  - Type: Code (JavaScript)  
  - Role: Filters posts to only those created within the last 24 hours based on their `created_utc` timestamps.  
  - Configuration: Custom JS code that:  
    - Reads all input posts, accounting for multiple input styles.  
    - Calculates current UNIX timestamp and subtracts 24 hours.  
    - Filters posts by comparing `created_utc` to cutoff.  
    - Logs detailed debug info.  
  - Inputs: Merged posts.  
  - Outputs: Filtered recent posts to AI agent node.  
  - Edge cases: Missing or malformed `created_utc`, empty post arrays, code exceptions. Debug logs included for troubleshooting.

#### 2.3 AI Comment Generation

**Overview:**  
Passes filtered recent posts to an AI agent that analyzes each post and generates recommended comment content according to a detailed engagement strategy.

**Nodes Involved:**  
- Comment AI Agent (LangChain Agent)  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**

- **Comment AI Agent**  
  - Type: LangChain Agent (AI agent)  
  - Role: Uses OpenAI GPT-4.1-mini model to analyze post data and produce JSON outputs with reply recommendation and comment text.  
  - Configuration:  
    - Input prompt includes post metadata (id, title, text, comments_count, upvote_ratio, subreddit, url).  
    - System message defines complex rules for engagement filters, tone, style, and output format.  
    - Output format JSON with keys: `post_id`, `should_reply` (boolean), `reason`, and `comment`.  
    - Uses OpenAI Chat Model node as language model.  
    - Has output parser configured.  
  - Inputs: Filtered posts.  
  - Outputs: AI-generated structured JSON with reply decision and comment.  
  - Edge cases: AI model errors, timeouts, malformed input data, API rate limiting, incomplete AI responses.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides GPT-4.1-mini model for the AI agent.  
  - Configuration: Model set to GPT-4.1-mini, no special options.  
  - Credentials: OpenAI API key.  
  - Inputs: From AI agent.  
  - Outputs: To Structured Output Parser.  
  - Edge cases: API key invalid/expired, network issues, model availability.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI agent raw output to structured JSON, auto-correcting minor format errors.  
  - Configuration: JSON schema example provided matching expected AI output.  
  - Inputs: From OpenAI Chat Model.  
  - Outputs: Back to Comment AI Agent for further processing.  
  - Edge cases: Parsing errors if AI output deviates significantly, possible silent failures.

#### 2.4 Reply Decision Filtering

**Overview:**  
Filters AI outputs to only allow posts where `should_reply` is true and post ID is not a specific excluded ID.

**Nodes Involved:**  
- Should Reply? (Filter)

**Node Details:**

- **Should Reply?**  
  - Type: Filter  
  - Role: Checks AI output's `should_reply` boolean and excludes a specific post ID `1mchxl3`.  
  - Configuration:  
    - Condition 1: `should_reply` must be true.  
    - Condition 2: `post_id` must not equal `1mchxl3`.  
    - Both conditions must be met (logical AND).  
  - Inputs: AI agent JSON output.  
  - Outputs: Approved posts proceed to Slack notification.  
  - Edge cases: Missing `should_reply` or `post_id` fields, false negatives due to strict filtering.

#### 2.5 Notification Delivery

**Overview:**  
Sends a Slack message to a specific channel with the Reddit post link and AI-generated comment idea for human review or posting.

**Nodes Involved:**  
- Send a message (Slack)

**Node Details:**

- **Send a message**  
  - Type: Slack node  
  - Role: Posts a formatted message to a Slack channel containing Reddit post URL and AI comment idea.  
  - Configuration:  
    - Text includes link to Reddit post (`https://www.reddit.com/r/CRM/comments/{{ post_id }}/`) and the comment content.  
    - Channel selected: Channel ID `C09DJJZ3DHQ` (named `astra-reddit-post-comment-ideas`).  
    - Authentication: OAuth2 with Slack credentials.  
    - Option set to exclude workflow link in message.  
  - Inputs: Filtered approved AI outputs.  
  - Outputs: None downstream (end node).  
  - Edge cases: Slack API token expiration, channel permission issues, message formatting errors.

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                                   | Input Node(s)                          | Output Node(s)               | Sticky Note                                                                                  |
|---------------------------|------------------------------------|-------------------------------------------------|--------------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger                   | Initiates workflow daily at 19:00                | None                                 | Automate, AI_Agents, salesdevelopment, SaaSSales, AskMarketing |                                                                                              |
| Automate                  | Reddit Search                     | Searches "Automate" subreddit with keywords      | Schedule Trigger                     | Merge                       |                                                                                              |
| AI_Agents                 | Reddit Search                     | Searches "AI_Agents" subreddit with keywords     | Schedule Trigger                     | Merge                       |                                                                                              |
| salesdevelopment          | Reddit Search                     | Searches "salesdevelopment" subreddit             | Schedule Trigger                     | Merge                       |                                                                                              |
| SaaSSales                 | Reddit Search                     | Searches "Automate" subreddit (likely typo)       | Schedule Trigger                     | Merge                       |                                                                                              |
| AskMarketing              | Reddit Search                     | Searches "AskMarketing" subreddit                  | Schedule Trigger                     | Merge                       |                                                                                              |
| Merge                     | Merge                             | Combines subreddit posts into one stream          | Automate, AI_Agents, salesdevelopment, SaaSSales, AskMarketing | Filter last 24 hour posts  |                                                                                              |
| Filter last 24 hour posts | Code                             | Filters posts to only those from last 24 hours    | Merge                               | Comment AI Agent            | Detailed debug logs included for troubleshooting filtering logic.                            |
| Comment AI Agent          | LangChain Agent                   | Generates AI comment ideas and reply decision     | Filter last 24 hour posts            | Should Reply?               | Complex prompt with engagement rules and authentic style for Reddit comments.                |
| OpenAI Chat Model         | LangChain OpenAI Chat Model       | Provides GPT-4.1-mini for AI agent                | Comment AI Agent                    | Structured Output Parser    |                                                                                              |
| Structured Output Parser  | LangChain Structured Output Parser| Parses AI output into JSON                         | OpenAI Chat Model                   | Comment AI Agent            | Auto-fixes minor output format errors.                                                      |
| Should Reply?             | Filter                           | Filters AI outputs to decide if reply should be sent | Comment AI Agent                    | Send a message              | Excludes a specific post ID from replies.                                                   |
| Send a message            | Slack                           | Posts AI comment suggestions to Slack channel     | Should Reply?                      | None                       | Posts to channel "astra-reddit-post-comment-ideas" with formatted Reddit post and comment.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 19:00 (7 PM) hour only.

2. **Create Reddit Search nodes (5 nodes):**  
   - For each, set type: Reddit  
   - Configure OAuth2 Reddit credentials.  
   - Set parameters:  
     - Automate: Subreddit = "Automate", Keywords = "AI Agent, AI, Automation, Sales", Limit = 5, Sort = "new"  
     - AI_Agents: Subreddit = "AI_Agents", Keywords = "Sales, Sales Automation, Website Leads, Lead Qualification", Limit = 5, Sort = "new"  
     - salesdevelopment: Subreddit = "salesdevelopment", Keywords = "AI Agent, AI, Automation", Limit = 5, Sort = "new"  
     - SaaSSales: Subreddit = "Automate" (check if this is a typo, otherwise same as Automate node), Keywords = "AI, Automation, AI Agents", Limit = 5, Sort = "new"  
     - AskMarketing: Subreddit = "AskMarketing", Keywords = "AI, Automation, AI Agents", Limit = 5, default sort.  
   - Connect each node’s input from Schedule Trigger’s main output.

3. **Create Merge node:**  
   - Type: Merge  
   - Set number of inputs to 5.  
   - Connect all 5 Reddit search nodes to corresponding inputs.

4. **Create Code node "Filter last 24 hour posts":**  
   - Paste the provided JavaScript code that filters posts by `created_utc` timestamp within last 24 hours.  
   - Connect Merge node output to this Code node.

5. **Create LangChain OpenAI Chat Model node:**  
   - Type: LangChain OpenAI Chat Model  
   - Set model to "gpt-4.1-mini".  
   - Provide OpenAI API credentials.

6. **Create LangChain Agent node "Comment AI Agent":**  
   - Type: LangChain Agent  
   - Set prompt type to "define".  
   - Configure input template with post fields: id, title, selftext, num_comments, upvote_ratio, subreddit, url.  
   - Paste system message with detailed engagement instructions, tone, filters, and output JSON format.  
   - Use the LangChain OpenAI Chat Model node as its language model input.  
   - Enable output parser.  
   - Connect Code node output to this agent node input.

7. **Create LangChain Structured Output Parser node:**  
   - Type: LangChain Structured Output Parser  
   - Provide JSON schema example matching AI agent output format.  
   - Connect OpenAI Chat Model node output to this parser.  
   - Connect parser output back to the Comment AI Agent node input (for structured output integration).

8. **Create Filter node "Should Reply?":**  
   - Type: Filter  
   - Conditions:  
     - `should_reply` must be true (boolean).  
     - `post_id` must not equal "1mchxl3".  
   - Connect Comment AI Agent output to this filter node.

9. **Create Slack node "Send a message":**  
   - Type: Slack  
   - Set authentication with Slack OAuth2 credentials.  
   - Set channel to "C09DJJZ3DHQ" (channel name: astra-reddit-post-comment-ideas).  
   - Compose message text:  
     ```
     Post: https://www.reddit.com/r/CRM/comments/{{ $json.output.post_id }}/

     Comment Idea: 
     {{ $json.output.comment }}
     ```  
   - Connect filter node output to this Slack node.

10. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow is designed to produce authentic, peer-style Reddit comments to build trust, not aggressively promote the brand.           | System message in Comment AI Agent defines tone and engagement filters.                             |
| Slack channel "astra-reddit-post-comment-ideas" is used to collect comment ideas for manual posting or review.                           | Slack node configuration.                                                                           |
| The "Filter last 24 hour posts" node includes extensive debug logging to aid in troubleshooting data inputs and filtering logic.        | Code node JavaScript includes console.logs for monitoring.                                         |
| Reddit search in SaaSSales node uses the "Automate" subreddit which may be a configuration error or intentional duplicate.               | Review recommended to ensure correct subreddit targeting.                                          |
| OpenAI model used is a lightweight GPT-4.1-mini; ensure API quota and latency expectations align with workflow demands.                  | OpenAI Chat Model node configuration.                                                              |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.