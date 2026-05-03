Find relevant X tweets based on your profile and suggest responses

https://n8nworkflows.xyz/workflows/find-relevant-x-tweets-based-on-your-profile-and-suggest-responses-4851


# Find relevant X tweets based on your profile and suggest responses

### 1. Workflow Overview

This workflow, titled **"X Tweet Suggestions"**, is designed to help users on the social media platform X (formerly Twitter) by analyzing their recent tweets and profile to generate relevant tweet suggestions. It targets users who want to enhance their presence on X by providing strategic tweet replies and original tweets tailored to their profile style, goals, and trending topics in their niche.  

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** User submits their X username, goals, and optional info via a form trigger.
- **1.2 User Tweet Retrieval & Mapping:** Fetches the user's last tweets from X and structures them for analysis.
- **1.3 Profile Analysis:** Uses AI to analyze the user's tweet style and profile details.
- **1.4 Keyword Generation:** Generates keyword groups based on profile analysis and user goals to find relevant trending tweets.
- **1.5 Query Construction & Tweet Search:** Builds advanced search queries from keywords and fetches relevant trending tweets.
- **1.6 Aggregation & Tweet Writing:** Aggregates found tweets and uses AI to write personalized tweet suggestions (responses and originals).
- **1.7 Post-Processing & Display:** Cleans results (e.g., replacing em dashes) and prepares a detailed, interactive HTML display of suggestions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures user input from a web form to start the analysis process.
- **Nodes Involved:**  
  - `On form submission`
- **Node Details:**  
  - Type: Form Trigger  
  - Configuration: Presents a form titled "Perfect Niche Tweets" with fields for the user’s X username, multiple selection goals on X, and optional additional info.  
  - Input: User inputs at runtime.  
  - Output: JSON with user data.  
  - Edge Cases: Missing or invalid username may cause downstream API calls to fail. Multi-select goals could be empty, handled downstream.  
  - Sub-workflow: None.

#### 2.2 User Tweet Retrieval & Mapping

- **Overview:** Fetches last 20 tweets of the user from X API and extracts tweets array for analysis.
- **Nodes Involved:**  
  - `Get User Tweets`  
  - `Map Tweets`
- **Node Details:**  
  - `Get User Tweets`  
    - Type: HTTP Request  
    - Configured to call `https://api.twitterapi.io/twitter/user/last_tweets` with username from form and count=20.  
    - Authentication: Header Auth credential named "Header Auth account".  
    - Timeout: 30 seconds, continues on failure to avoid hard stops.  
    - Output: Raw API response containing tweets.  
    - Edge Cases: API auth failure, rate limits, user not found, empty tweet list.  
  - `Map Tweets`  
    - Type: Code node (JavaScript)  
    - Extracts `data.tweets` array from API response JSON into `{ tweets: [...] }`.  
    - Input: API response JSON.  
    - Output: JSON containing only tweets array for next steps.  
    - Edge Cases: Malformed data or missing tweets array.

#### 2.3 Profile Analysis

- **Overview:** Uses AI to analyze the user's tweet style and profile characteristics for personalized tweet generation.
- **Nodes Involved:**  
  - `User Profile Analyser`  
  - `profile schema` (structured output parser)  
- **Node Details:**  
  - `User Profile Analyser`  
    - Type: LangChain AI Agent (OpenAI-backed)  
    - Prompt: Provides detailed profile analysis from user's last tweets, focusing on style, punctuation, humor, personality, and more.  
    - Input: Tweets array JSON from `Map Tweets`.  
    - Output: AI-generated textual profile analysis JSON.  
    - Edge Cases: AI timeout, malformed input, unexpected output format.  
  - `profile schema`  
    - Type: LangChain Output Parser (Structured)  
    - Parses AI output into structured JSON fields: examplePrompt, punctuation, personalityType, humorType, expertise, recognizedPatterns.  
    - Required field: examplePrompt.  
    - Edge Cases: Parsing failures if AI output deviates.

#### 2.4 Keyword Generation

- **Overview:** Generates three groups of relevant keywords targeting different aspects of the user's interests and goals to find trending tweets.
- **Nodes Involved:**  
  - `Keyword Analyser`  
  - `keywords schema` (structured output parser)  
- **Node Details:**  
  - `Keyword Analyser`  
    - Type: LangChain AI Agent  
    - Prompt: Expert Twitter trend analyst generating three groups of 3-5 related keywords each, based on user profile analysis and optional additional info.  
    - Input: JSON profile analysis, user additional info from form.  
    - Output: JSON with keyword groups.  
    - Edge Cases: AI output format errors, keyword group size constraints.  
  - `keywords schema`  
    - Type: LangChain Output Parser (Structured)  
    - Parses AI output into exactly 3 keyword groups of 3-5 keywords each.  
    - Edge Cases: Parsing errors.

#### 2.5 Query Construction & Tweet Search

- **Overview:** Builds advanced search queries using the keyword groups and user goals to fetch relevant trending tweets.
- **Nodes Involved:**  
  - `combine keywords`  
  - `Split Out`  
  - `Get relevant tweets`  
  - `Aggregate`
- **Node Details:**  
  - `combine keywords`  
    - Type: Code node  
    - Logic: For each keyword group, creates an OR-based search query string with filters on time (last 2 days), minimum favorites and retweets based on goals (e.g., viral or follower increase), language (English), and verified user filters.  
    - Input: Keyword groups JSON and user goals.  
    - Output: Array of search queries labeled `queries`.  
    - Edge Cases: Missing keyword groups or goals; empty queries.  
  - `Split Out`  
    - Type: Split Out node  
    - Splits the `queries` array into individual items to process each query separately.  
    - Input: Queries array.  
    - Output: Individual query items.  
  - `Get relevant tweets`  
    - Type: HTTP Request  
    - Calls `https://api.twitterapi.io/twitter/tweet/advanced_search` for each query, using header auth.  
    - Query parameters: `query` (search query string), `queryType=Top`.  
    - Output: Tweets matching each query.  
    - Edge Cases: API errors, rate limits, empty results.  
  - `Aggregate`  
    - Type: Aggregate node  
    - Aggregates all tweets from the multiple queries into a single collection.  
    - Input: Multiple individual tweet lists.  
    - Output: Combined tweets array.  
    - Edge Cases: Empty aggregation.

#### 2.6 Aggregation & Tweet Writing

- **Overview:** Uses AI to craft 10 tweet suggestions (mostly responses, plus up to 2 originals) based on user goals, profile, bio, and relevant tweets.
- **Nodes Involved:**  
  - `Tweet Writer`  
  - `tweet schema` (structured output parser)  
  - `Replace em dash`
- **Node Details:**  
  - `Tweet Writer`  
    - Type: LangChain AI Agent (OpenAI-backed, via LangChain agent node)  
    - Prompt: Expert on X conversions and brand building, instructed to write authentic, goal-aligned tweets engaging with authors of relevant tweets or creating originals.  
    - Input: Aggregated relevant tweets, user goals and profile, user bio (from first tweet author description), additional user info.  
    - Output: 10 tweet suggestions with priority, text, isAnswer flag, tweetUrl (if response), and reasoning.  
    - Constraints: Max 2 original tweets, no hashtags, limited exclamation marks, avoid certain AI cliché phrases, sarcasm allowed but tasteful.  
    - Edge Cases: AI response failure, malformed output, missing URLs.  
  - `tweet schema`  
    - Type: LangChain Output Parser (Structured)  
    - Parses AI output into a structured array of tweet objects with fields and validations.  
    - Edge Cases: Parsing errors.  
  - `Replace em dash`  
    - Type: Code node  
    - Replaces double hyphens "--" with spaced single dash " - " in tweets' text to improve readability.  
    - Input: Parsed tweets JSON.  
    - Output: Cleaned tweets JSON.  
    - Edge Cases: Tweets missing text property.

#### 2.7 Post-Processing & Display

- **Overview:** Prepares a rich HTML display of tweet suggestions, separating responses and originals, with interactive copy buttons and reasoning toggles.
- **Nodes Involved:**  
  - `Prepare Result`  
  - `Click to show Result`
- **Node Details:**  
  - `Prepare Result`  
    - Type: Code node  
    - Logic:  
      - Maps each tweet suggestion to its original tweet data (author, text) collected from aggregated tweets.  
      - Builds HTML blocks for response tweets and original tweets with badges, clickable text (to copy), and toggleable strategic reasoning sections.  
      - Includes debug info on missing or skipped tweets.  
      - Provides user-friendly styling and interaction scripts for copying and toggling reasoning panels.  
    - Input: Cleaned tweet suggestions and aggregated original tweets data.  
    - Output: Full HTML content with tweet suggestions.  
    - Edge Cases: Missing original tweet data, unknown users, empty suggestions.  
  - `Click to show Result`  
    - Type: HTML node  
    - Displays the generated HTML content in the n8n UI.  
    - Input: HTML string from `Prepare Result`.  
    - Output: Rendered UI display.  
    - Edge Cases: Rendering issues if HTML malformed.

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                         | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                                   |
|---------------------|----------------------------------|---------------------------------------|--------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger                     | Captures user inputs                   | -                        | Get User Tweets            | # 🚀 X Tweet Suggestions Workflow                                                                             |
| Get User Tweets      | HTTP Request                    | Fetches user's recent tweets           | On form submission       | Map Tweets                 | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| Map Tweets           | Code                            | Extracts tweets array from API response | Get User Tweets          | User Profile Analyser      | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| User Profile Analyser| LangChain AI Agent (OpenAI)     | Analyzes user tweet style & personality | Map Tweets               | profile schema, Keyword Analyser | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| profile schema       | LangChain Output Parser (Structured) | Parses user profile AI output          | User Profile Analyser    | Keyword Analyser           | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| Keyword Analyser     | LangChain AI Agent (OpenAI)     | Generates keyword groups for tweet search | profile schema           | keywords schema, combine keywords | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| keywords schema      | LangChain Output Parser (Structured) | Parses keyword groups AI output         | Keyword Analyser         | combine keywords           | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| combine keywords     | Code                            | Builds advanced search queries           | keywords schema          | Split Out                  | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| Split Out            | Split Out                       | Splits queries array into single queries | combine keywords         | Get relevant tweets        | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| Get relevant tweets  | HTTP Request                    | Fetches trending tweets for each query  | Split Out                | Aggregate                  | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| Aggregate            | Aggregate                      | Aggregates tweets from all queries       | Get relevant tweets      | Tweet Writer               | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| Tweet Writer         | LangChain AI Agent (OpenAI)     | Generates personalized tweet suggestions | Aggregate                | Replace em dash            | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| tweet schema         | LangChain Output Parser (Structured) | Parses generated tweets AI output       | Tweet Writer             | Replace em dash            | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| Replace em dash      | Code                            | Cleans tweets text for readability       | Tweet Writer             | Prepare Result             | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| Prepare Result       | Code                            | Builds interactive HTML UI of tweet suggestions | Replace em dash          | Click to show Result       | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| Click to show Result | HTML                            | Displays the final tweet suggestions UI   | Prepare Result           | -                          | Add your OpenAI credentials and Twitter API key from https://twitterapi.io; see setup instructions.           |
| Sticky Note          | Sticky Note                    | Setup instructions and workflow overview | -                        | -                          | Contains detailed setup and usage instructions, including API key setup and workflow purpose explanation.    |
| Sticky Note1         | Sticky Note                    | Workflow title banner                     | -                        | -                          | # 🚀 X Tweet Suggestions Workflow                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named `On form submission` with:  
   - Form title: "Perfect Niche Tweets"  
   - Fields:  
     - Text field: "What's your name on X (it's within your url)" (single-line)  
     - Dropdown multi-select: "What are your current goals on X" with options:  
       - Increase followers  
       - Boost impressions or reach  
       - Boost engagement  
       - Write a viral tweet  
       - Just be helpful and motivating  
     - Text field (optional): "Additional Infos (Optional)"  
   - No special credentials needed.

2. **Add an HTTP Request node** named `Get User Tweets`:  
   - Method: GET  
   - URL: `https://api.twitterapi.io/twitter/user/last_tweets`  
   - Query parameters:  
     - `userName`: Expression referencing form input: `={{ $json["What's your name on X (it's within your url)"] }}`  
     - `count`: 20  
   - Authentication: Use HTTP Header Auth credential named "Header Auth account" with your TwitterAPI key.  
   - Timeout: 30 seconds.  
   - Enable "Continue On Fail" to avoid halting workflow if API fails.  
   - Connect output of `On form submission` to this node.

3. **Add a Code node** named `Map Tweets`:  
   - JavaScript code:  
     ```
     return { tweets: $input.first().json.data.tweets };
     ```  
   - Input: Output of `Get User Tweets`.  
   - Output: Simplified JSON with only tweets array.  

4. **Add a LangChain AI Agent node** named `User Profile Analyser`:  
   - Model: OpenAI (e.g., o3-mini) with your OpenAI credentials.  
   - Prompt: Detailed user profile analysis instructions using:  
     ```
     You will provide a detailed user profile of an X user based on their recent tweets. Your output will serve to create new tweets for this user...  
     here are the user's last tweets: {{ $json.tweets.toJsonString() }}
     ```  
   - Connect input from `Map Tweets`.

5. **Add a LangChain Output Parser node** named `profile schema`:  
   - Schema type: Manual  
   - Define JSON schema with fields for examplePrompt, punctuation, personalityType, humorType, fieldOfExpertise, recognizedPatterns (see workflow).  
   - Connect AI output from `User Profile Analyser`.

6. **Add another LangChain AI Agent node** named `Keyword Analyser`:  
   - Model: OpenAI (e.g., 4.1-mini) with same credentials.  
   - Prompt: Twitter trend analyst generating 3 keyword groups, using:  
     ```
     User Profile Analysis: {{ $json.output.toJsonString() }}  
     User Additional Input: {{ $('On form submission').item.json["Additional Infos (Optional)"] || "none" }}  
     Your Mission: Generate 3 groups of keywords...  
     ```  
   - Connect input from `profile schema`.

7. **Add a LangChain Output Parser node** named `keywords schema`:  
   - Manual JSON schema expecting 3 arrays of 3-5 keywords each.  
   - Connect AI output from `Keyword Analyser`.

8. **Add a Code node** named `combine keywords`:  
   - Logic to create advanced search queries per keyword group, adding filters for time, favorites, retweets, language, and verified users based on goals.  
   - Connect input from `keywords schema` and reference user goals from form.  

9. **Add a Split Out node** named `Split Out`:  
   - Field to split out: `queries` (array of search query strings).  
   - Connect input from `combine keywords`.

10. **Add an HTTP Request node** named `Get relevant tweets`:  
    - Method: GET  
    - URL: `https://api.twitterapi.io/twitter/tweet/advanced_search`  
    - Query parameters:  
      - `query`: `={{ $json["queries"] }}` (from split output)  
      - `queryType`: "Top"  
    - Authentication: Use same HTTP Header Auth "Header Auth account".  
    - Connect input from `Split Out`.

11. **Add an Aggregate node** named `Aggregate`:  
    - Aggregate all `tweets` fields from results of `Get relevant tweets` into a single array.  
    - Connect input from `Get relevant tweets`.

12. **Add a LangChain AI Agent node** named `Tweet Writer`:  
    - Model: OpenAI (e.g., o3-mini) with credentials.  
    - Prompt: Expert tweet creator with detailed instructions to write 10 tweets (mostly replies), using user goals, profile, bio, and trending tweets.  
    - Input: Output from `Aggregate`.  
    - Connect input from `Aggregate`.

13. **Add a LangChain Output Parser node** named `tweet schema`:  
    - Manual JSON schema for array of tweets with priority, text, isAnswer, tweetUrl, reasoning fields.  
    - Connect AI output from `Tweet Writer`.

14. **Add a Code node** named `Replace em dash`:  
    - JavaScript code replacing all `--` in tweet texts with ` - ` for readability.  
    - Connect input from `Tweet Writer` (parsed).

15. **Add a Code node** named `Prepare Result`:  
    - Logic to:  
      - Map each suggested tweet’s URL to original tweet data (author, text) from aggregated data.  
      - Build HTML with badges indicating response or original tweets.  
      - Include interactive features: copy-on-click tweet text, toggle strategic reasoning display.  
      - Add debug info on missing or unknown tweets.  
    - Connect input from `Replace em dash`.

16. **Add an HTML node** named `Click to show Result`:  
    - Display the HTML generated by `Prepare Result`.  
    - Connect input from `Prepare Result`.

17. **Credentials Setup:**  
    - Create HTTP Header Auth credential named "Header Auth account" with TwitterAPI key from https://twitterapi.io.  
    - Create OpenAI API credential and assign to all LangChain/OpenAI nodes.

18. **Workflow Execution:**  
    - Activate or execute workflow.  
    - User fills and submits form.  
    - Workflow processes sequentially and outputs an interactive HTML interface with tweet suggestions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Setup instructions: Add OpenAI credentials and TwitterAPI key from https://twitterapi.io?ref=1PROMPT (:pray:). Workflow fetches user tweets, analyzes profile, generates keywords, searches tweets, and crafts personalized tweets.               | Sticky Note content in the workflow                 |
| How to use: Execute workflow, fill the form with X username and goals, submit and wait 2-3 minutes, then view results by double-clicking "Click to show Result" node.                                                                           | Sticky Note content in the workflow                 |
| The workflow avoids hashtags in tweets, limits exclamation marks, and bans cliché AI phrases to maintain authenticity and engagement.                                                                                                         | Prompt instructions in Tweet Writer node            |
| Tweets that are crypto-related or political are excluded from responses. Only respond to one tweet per user among relevant tweets.                                                                                                            | Prompt instructions in Tweet Writer node            |
| Follow https://x.com/OnePromptMagic for more free n8n templates and updates.                                                                                                                                                                  | Sticky Note content in the workflow                 |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow created with n8n, adhering strictly to content policies and legal public data usage.