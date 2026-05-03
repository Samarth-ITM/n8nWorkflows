Repurpose Reddit Posts into AI Tweets with Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/repurpose-reddit-posts-into-ai-tweets-with-gemini-and-google-sheets-9395


# Repurpose Reddit Posts into AI Tweets with Gemini and Google Sheets

---

### 1. Workflow Overview

This workflow automates the repurposing of Reddit posts into concise, edgy tweets using AI language models and logs them to Google Sheets to avoid duplication. It targets social media managers, content creators, and automation enthusiasts who want to generate fresh Twitter content derived from trending Reddit discussions, with minimal manual effort.

The workflow is organized into the following logical blocks:

- **1.1 Scheduled Trigger & Subreddit Selection:** Periodically triggers the process and randomly selects a subreddit or a promotional tweet type.
- **1.2 Reddit Post Fetching & Duplication Check:** Retrieves recent posts from the selected subreddit and cross-checks against logged posts in Google Sheets to avoid repetition.
- **1.3 AI Tweet Generation:** Uses Google Gemini Chat model and LangChain agent to craft unique, punchy tweets based on selected Reddit posts.
- **1.4 Post-Processing & Output Parsing:** Structures AI output into usable fields: tweet text, subreddit name, and post ID.
- **1.5 Tweet Posting & Logging:** Publishes the tweet on Twitter and logs the post ID with related info into Google Sheets for future duplication prevention.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Trigger & Subreddit Selection

**Overview:**  
This block periodically initiates the workflow every 2 hours and randomly chooses a subreddit or triggers a promotional tweet type to diversify content.

**Nodes Involved:**  
- Schedule Trigger1  
- Code1  
- Sticky Note (explanatory)

**Node Details:**

- **Schedule Trigger1**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts workflow execution every 2 hours.  
  - *Configuration:* Interval set to 2 hours (hoursInterval=2).  
  - *Connections:* Outputs to Code1.  
  - *Potential Failures:* Trigger misconfiguration; rare.

- **Code1**  
  - *Type:* Code (JavaScript)  
  - *Role:* Randomly selects a subreddit from a predefined list or decides to send a promotional ("advertise") tweet with 20% probability.  
  - *Configuration:*  
    - Uses a fixed array of subreddits: ["n8n", "microsaas", "SaaS", "automation", "n8n_ai_agents"].  
    - Avoids repeating the same output twice in a row by storing last choice in global context (`global.lastTweet`).  
    - Returns an object with keys `tweet` (subreddit or "advertise") and `ads` (boolean flag).  
  - *Key expressions:* Random selection with probability branching; global variable usage.  
  - *Connections:* Outputs to Tweet maker1.  
  - *Edge cases:* Global context reset on workflow restart may cause repeated tweets; randomness may bias selection.  
  - *Sticky Note:* Explains subreddit selection logic with example list.

- **Sticky Note** (positioned near Code1)  
  - *Content:* Describes the random subreddit selection logic with example subreddit list JSON.  

---

#### 2.2 Reddit Post Fetching & Duplication Check

**Overview:**  
Fetches recent trending posts from the selected subreddit and verifies against Google Sheets to ensure the same post hasn't been tweeted previously.

**Nodes Involved:**  
- Tweet maker1 (AI agent, part here for subreddit input)  
- Get many posts in Reddit1  
- read database2 (Google Sheets lookup)  

**Node Details:**

- **Get many posts in Reddit1**  
  - *Type:* Reddit Tool  
  - *Role:* Fetches up to 10 rising (trending) posts from the subreddit received from AI input (`$fromAI('subreddit')`).  
  - *Configuration:*  
    - Limit: 10 posts  
    - Filters: category = "rising"  
    - Subreddit dynamically assigned from AI output (string).  
  - *Credentials:* Reddit OAuth2 API.  
  - *Connections:* Output used by Tweet maker1 AI agent as a tool.  
  - *Edge cases:* API rate limits, subreddit errors, or empty subreddit name can cause failure.

- **read database2**  
  - *Type:* Google Sheets Tool (Read)  
  - *Role:* Looks up the Google Sheet for entries matching the subreddit and post_id to check if the post was used before.  
  - *Configuration:*  
    - Spreadsheet ID and sheet "posts" specified.  
    - Filters applied on "subreddit" and "post_id" columns using AI extracted values.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Connections:* Used as an AI tool input by Tweet maker1 for duplication check.  
  - *Edge cases:* Credential expiration, sheet access issues, or data inconsistency.

- **Tweet maker1 (AI agent)**  
  - *Role in this block:* Receives subreddit info and uses Reddit posts and Google Sheets data as tools to select an appropriate Reddit post that hasn’t been tweeted yet.  
  - *Connections:* Receives tools input from Reddit and Google Sheets nodes.

---

#### 2.3 AI Tweet Generation

**Overview:**  
Uses Google Gemini Chat Model with LangChain agent to generate a unique, edgy tweet referring to a Reddit post, ensuring no repetition of post IDs.

**Nodes Involved:**  
- Google Gemini Chat Model1  
- Tweet maker1  
- Structured Output Parser2  

**Node Details:**

- **Google Gemini Chat Model1**  
  - *Type:* LangChain Google Gemini LM Chat  
  - *Role:* Provides the language model backend for AI agent to generate tweets.  
  - *Credentials:* Google Palm API (Gemini).  
  - *Connections:* Connected as AI language model input to Tweet maker1.  
  - *Edge cases:* API quota limits, network errors, model downtime.

- **Tweet maker1**  
  - *Type:* LangChain AI Agent  
  - *Role:*  
    - Acts as a ghostwriter generating tweets from Reddit posts.  
    - Uses system message prompt specifying style rules: edgy, first-person, concise (≤ 200 chars), no hashtags, minimal emojis, no profanity.  
    - Utilizes three tools: Reddit posts, past tweets database, and logging mechanism to avoid duplicates.  
  - *Parameters:*  
    - Input text dynamically set from Code1 output (`={{ $json.tweet }}`).  
    - Custom system message guiding tone and style.  
    - Output is parsed by Structured Output Parser2.  
  - *Connections:*  
    - Receives AI language model input from Google Gemini Chat Model1.  
    - Receives AI tools input from Reddit and Google Sheets nodes.  
    - Outputs to Edit Fields1 for data extraction.  
  - *Edge cases:* Expression failures, AI hallucination, API failures.

- **Structured Output Parser2**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses AI output into structured JSON with fields: `tweet`, `subreddit`, and `id` (post_id).  
  - *Configuration:*  
    - Manual JSON schema specifying required fields and descriptions.  
  - *Connections:* Outputs parsed data to Tweet maker1 node's main output.  
  - *Edge cases:* Parsing errors if AI output deviates from schema.

---

#### 2.4 Post-Processing & Output Preparation

**Overview:**  
Assigns parsed AI output fields into workflow variables preparing for posting and logging.

**Nodes Involved:**  
- Edit Fields1  

**Node Details:**

- **Edit Fields1**  
  - *Type:* Set Node  
  - *Role:* Extracts and assigns the AI-generated tweet text, subreddit name, and post ID into new JSON fields for downstream nodes.  
  - *Configuration:*  
    - Assignments:  
      - `tweet` ← `$json.output.tweet`  
      - `subreddit` ← `$json.output.subreddit || null`  
      - `post_id` ← `$json.output.id || null`  
  - *Connections:* Outputs to Creates the tweet1 node (Twitter posting).  
  - *Edge cases:* Null or missing AI output fields.

---

#### 2.5 Tweet Posting & Logging

**Overview:**  
Posts the generated tweet to Twitter and records the Reddit post ID and tweet in Google Sheets to prevent future duplication.

**Nodes Involved:**  
- Creates the tweet1 (Twitter node)  
- Append row in sheet1 (Google Sheets append)  
- Sticky Notes (explanations)

**Node Details:**

- **Creates the tweet1**  
  - *Type:* Twitter Node (OAuth2)  
  - *Role:* Publishes the tweet text to Twitter (currently branded as X).  
  - *Configuration:*  
    - Text set from `{{$json.tweet}}`.  
    - Optional image attachments from `{{$json.image_id}}` if present.  
  - *Credentials:* Twitter OAuth2 API.  
  - *Connections:* Output to Append row in sheet1.  
  - *Edge cases:* Auth errors, Twitter API limits, content rejection.

- **Append row in sheet1**  
  - *Type:* Google Sheets Append  
  - *Role:* Records the tweet data for future reference, including date, post_id, subreddit, and tweet text.  
  - *Configuration:*  
    - Document ID and sheet "posts" specified.  
    - Columns: Date (current day), post_id, subreddit, PAST TWEETS (tweet text).  
  - *Credentials:* Google Sheets OAuth2.  
  - *Connections:* Final node.  
  - *Edge cases:* Sheet quota limits, credential expiration.

- **Sticky Note10 & Sticky Note11**  
  - *Sticky Note10:* Explains how to obtain Twitter API credentials and shows a Twitter post example.  
  - *Sticky Note11:* Explains the purpose of updating Google Sheets to prevent duplicate tweeting of same Reddit posts.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                        | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                              |
|------------------------|----------------------------------|-------------------------------------|-----------------------|-----------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger1       | Schedule Trigger                 | Periodically starts workflow         | —                     | Code1                 |                                                                                                        |
| Code1                  | Code (JavaScript)                | Randomly selects subreddit or promo | Schedule Trigger1      | Tweet maker1          | Explains random subreddit selection with sample list                                                   |
| Tweet maker1           | LangChain AI Agent               | Generates tweet from Reddit post    | Code1, Get many posts in Reddit1, read database2, Google Gemini Chat Model1, Structured Output Parser2 | Edit Fields1           |                                                                                                        |
| Get many posts in Reddit1 | Reddit Tool                     | Fetches 10 trending Reddit posts    | —                     | Tweet maker1 (tool)   |                                                                                                        |
| read database2          | Google Sheets Tool (Read)        | Checks for duplicate Reddit post IDs| —                     | Tweet maker1 (tool)   |                                                                                                        |
| Google Gemini Chat Model1 | LangChain Google Gemini LM Chat | Provides AI language model           | —                     | Tweet maker1 (AI LM)  |                                                                                                        |
| Structured Output Parser2 | LangChain Output Parser Structured | Parses AI output into structured JSON| —                     | Tweet maker1 (parser) |                                                                                                        |
| Edit Fields1            | Set                             | Assigns tweet and metadata fields    | Tweet maker1           | Creates the tweet1     |                                                                                                        |
| Creates the tweet1      | Twitter Node                    | Posts tweet to Twitter               | Edit Fields1           | Append row in sheet1   | Explains how to get Twitter API credentials and shows example post                                     |
| Append row in sheet1    | Google Sheets Append            | Logs tweeted Reddit post and text   | Creates the tweet1     | —                     | Explains logging the Reddit post to prevent duplicates                                                |
| Sticky Note             | Sticky Note                     | Explains subreddit selection logic  | —                     | —                     | Explains the random subreddit selection logic with example subreddits                                 |
| Sticky Note9            | Sticky Note                     | Describes AI repurposing logic      | —                     | —                     | Describes AI agent steps to get posts, check duplicates, and repurpose for Twitter                    |
| Sticky Note10           | Sticky Note                     | Twitter posting instructions        | —                     | —                     | Provides instructions and visual for Twitter API credentials and posting                              |
| Sticky Note11           | Sticky Note                     | Google Sheets logging instructions  | —                     | —                     | Explains the importance of logging posts to avoid duplicate tweets                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to every 2 hours (hoursInterval=2).  
   - Connect output to next node.

2. **Create Code Node (Code1)**  
   - Type: Code (JavaScript)  
   - Paste JavaScript code that:  
     - Defines subreddit list: ["n8n", "microsaas", "SaaS", "automation", "n8n_ai_agents"]  
     - Sets 20% chance of promotional tweet ("advertise")  
     - Prevents repeating last chosen subreddit or promo by storing in global context  
     - Outputs `{ tweet, ads }` JSON object  
   - Connect Schedule Trigger1 main output to Code1 main input.

3. **Create Reddit Tool Node (Get many posts in Reddit1)**  
   - Type: Reddit Tool  
   - Operation: getAll  
   - Limit: 10  
   - Filters: category = "rising"  
   - Subreddit: Expression → `{{$fromAI('subreddit', 'name of the subreddit', 'string')}}`  
   - Configure Reddit OAuth2 credentials.  
   - No direct input connection; it serves as AI tool input.

4. **Create Google Sheets Read Node (read database2)**  
   - Type: Google Sheets Tool (Read)  
   - Document ID: Use your Google Sheet ID (example given in original workflow).  
   - Sheet Name: "posts" (gid=0)  
   - Filters:  
     - subreddit = `{{$fromAI('subreddit', 'subreddit', 'string')}}`  
     - post_id = `{{$fromAI('id', 'id of the post', 'string')}}`  
   - Configure Google Sheets OAuth2 credentials.  
   - No direct input connection; serves as AI tool input.

5. **Create Google Gemini Chat Model Node (Google Gemini Chat Model1)**  
   - Type: LangChain LM Chat Google Gemini  
   - Configure Google Palm API (Gemini) credentials.  
   - No input connection; used as AI language model input.

6. **Create LangChain AI Agent Node (Tweet maker1)**  
   - Type: LangChain Agent  
   - Text input: Expression → `={{ $json.tweet }}` (from Code1 output)  
   - System message: Set detailed prompt with instructions on tweet style, length, tools usage, and constraints (as per original workflow).  
   - Link AI language model input to Google Gemini Chat Model1 output.  
   - Link AI tools input to Get many posts in Reddit1 and read database2 outputs.  
   - Link AI output parser input to Structured Output Parser2 output.  
   - Connect Code1 output to Tweet maker1 main input.

7. **Create Structured Output Parser Node (Structured Output Parser2)**  
   - Type: LangChain Output Parser Structured  
   - Input Schema (manual):  
     ```json
     {
       "type": "object",
       "properties": {
         "tweet": { "type": "string" },
         "subreddit": { "type": "string" },
         "id": { "type": "string", "description": "id of the post on reddit" }
       },
       "required": ["tweet"]
     }
     ```  
   - Connect output to Tweet maker1 AI output parser input.

8. **Create Set Node (Edit Fields1)**  
   - Type: Set  
   - Assignments:  
     - tweet = `{{$json.output.tweet}}`  
     - subreddit = `{{$json.output.subreddit || null}}`  
     - post_id = `{{$json.output.id || null}}`  
   - Connect Tweet maker1 main output to Edit Fields1 input.

9. **Create Twitter Node (Creates the tweet1)**  
   - Type: Twitter (OAuth2)  
   - Text: Expression → `{{$json.tweet}}`  
   - Optional attachments: `{{$json.image_id || null}}` if images are used.  
   - Configure Twitter OAuth2 API credentials from https://developer.x.com  
   - Connect Edit Fields1 output to Creates the tweet1 input.

10. **Create Google Sheets Append Node (Append row in sheet1)**  
    - Type: Google Sheets Append  
    - Document ID: Same as read database2  
    - Sheet Name: "posts" (gid=0)  
    - Columns mapping:  
      - Date = `{{$now.format("dd/MM/yyyy")}}`  
      - post_id = `{{$json.post_id}}` (from Edit Fields1)  
      - subreddit = `{{$json.subreddit}}`  
      - PAST TWEETS = `{{$json.tweet}}`  
    - Configure Google Sheets OAuth2 credentials.  
    - Connect Creates the tweet1 output to Append row in sheet1 input.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| For Twitter API credentials, visit: https://developer.x.com                                                                                                  | Sticky Note10                                                          |
| Google Sheets used for logging posts to avoid duplicate tweets. Spreadsheet ID and sheet "posts" must be accessible and shared with workflow credentials.   | Sticky Note11                                                          |
| The AI agent uses a strict prompt to generate edgy, punchy tweets without profanity or hashtags, ensuring modern Twitter style and first-person voice.       | Prompt inside Tweet maker1 node parameters                            |
| Example subreddits used for random selection include "n8n", "microsaas", "SaaS", "automation", and "n8n_ai_agents".                                          | Sticky Note near Code1 node                                            |
| Screenshot references for Twitter and Google Sheets steps are included in Sticky Notes for visual guidance.                                                  | Sticky Note10 and Sticky Note11                                        |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.

---