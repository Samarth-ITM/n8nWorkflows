Publish daily gaming guides from Reddit using Google Gemini to your web app

https://n8nworkflows.xyz/workflows/publish-daily-gaming-guides-from-reddit-using-google-gemini-to-your-web-app-14854


# Publish daily gaming guides from Reddit using Google Gemini to your web app

# Workflow Documentation: Reddit $\rightarrow$ AI Gaming Guide Publisher

### 1. Workflow Overview

This workflow automates the creation and publication of SEO-optimized gaming guides by monitoring trending discussions on Reddit. It identifies high-value questions or problems shared by gamers, uses Google Gemini AI to draft a professional guide, and pushes the final content to a web application via an HTTP POST request.

The logic is divided into four functional blocks:
- **1.1 Input & Scheduling:** Triggers the process daily and defines the target subreddits.
- **1.2 Data Acquisition & Extraction:** Fetches raw RSS feeds from Reddit and parses XML into structured JSON.
- **1.3 AI Content Generation:** Analyzes trending posts to select the best topic and generates a detailed Markdown guide.
- **1.4 Publication & Loop Control:** Sends the content to the target web app and manages the iteration cycle to prevent rate limiting.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Scheduling
**Overview:** Initiates the workflow on a fixed schedule and provides the list of subreddits to be scanned.
- **Nodes Involved:** `Schedule Trigger1`, `List of Subreddits1`.
- **Node Details:**
    - **Schedule Trigger1** (Schedule Trigger): Configured to fire daily at 08:00.
    - **List of Subreddits1** (Code): A JavaScript node returning a static array of subreddit names (e.g., `PokemonChampions`, `PokemonWindsWaves`, `gamingnews`). 
    - **Connections:** `Schedule Trigger1` $\rightarrow$ `List of Subreddits1`.

#### 2.2 Data Acquisition & Extraction
**Overview:** Iterates through the subreddit list, fetches the latest "hot" posts via RSS, and cleans the raw XML data.
- **Nodes Involved:** `Loop Over Items1`, `Get Reddit JSON1`, `Code in JavaScript`.
- **Node Details:**
    - **Loop Over Items1** (Split in Batches): Ensures each subreddit is processed individually.
    - **Get Reddit JSON1** (HTTP Request): Performs a GET request to `https://www.reddit.com/r/{{ $json.name }}/hot.rss?limit=10`.
    - **Code in JavaScript** (Code): A custom parser that:
        - Splits the raw XML by `<entry>` tags.
        - Extracts `title`, `author`, `link`, and `content`.
        - Strips HTML tags and entities using Regular Expressions.
        - Limits the content snippet to 600 characters.
    - **Connections:** `Loop Over Items1` $\rightarrow$ `Get Reddit JSON1` $\rightarrow$ `Code in JavaScript`.

#### 2.3 AI Content Generation
**Overview:** Uses Large Language Model (LLM) logic to transform a list of posts into a single, high-quality professional article.
- **Nodes Involved:** `Message a model`, `Code in JavaScript1`.
- **Node Details:**
    - **Message a model** (Google Gemini): 
        - **Model:** `gemini-3-flash-preview`.
        - **Prompt:** Acts as a "World-class Gaming Journalist." It filters out memes/rants, selects one post involving a game mechanic or quest struggle, and generates a Markdown guide including an Introduction, Requirements, Step-by-Step walkthrough, and a Data Table.
        - **Constraint:** Forced to output a strict JSON object with `headline` and `article_body` keys.
    - **Code in JavaScript1** (Code): Parses the raw text string returned by Gemini to extract the `headline` and `article_body` into top-level JSON fields.
    - **Connections:** `Code in JavaScript` $\rightarrow$ `Message a model` $\rightarrow$ `Code in JavaScript1`.

#### 2.4 Publication & Loop Control
**Overview:** Delivers the final article to the end-point and manages the flow back to the loop.
- **Nodes Involved:** `Send to WebApp`, `Wait1`.
- **Node Details:**
    - **Send to WebApp** (HTTP Request): POSTs the `headline` (as `title`) and `article_body` (as `content`) to a specified URL.
    - **Wait1** (Wait): Pauses execution briefly. This is critical for avoiding Reddit RSS rate limits and ensuring a clean loop transition.
    - **Connections:** `Code in JavaScript1` $\rightarrow$ `Send to WebApp` $\rightarrow$ `Wait1` $\rightarrow$ `Loop Over Items1`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger1 | Schedule Trigger | Workflow Initiation | - | List of Subreddits1 | Fires once per day at 08:00. |
| List of Subreddits1 | Code | Subreddit Definition | Schedule Trigger1 | Loop Over Items1 | Returns a static array of subreddit names. |
| Loop Over Items1 | Split in Batches | Iteration Control | List of Subreddits1 / Wait1 | Get Reddit JSON1 | Processes ONE subreddit per iteration. |
| Get Reddit JSON1 | HTTP Request | Data Fetching | Loop Over Items1 | Code in JavaScript | Fetches Reddit RSS feed. |
| Code in JavaScript | Code | XML to JSON Parser | Get Reddit JSON1 | Message a model | Extracts title, author, link, and cleans HTML. |
| Message a model | Google Gemini | Content Generation | Code in JavaScript | Code in JavaScript1 | Generates SEO Markdown guide from top posts. |
| Code in JavaScript1 | Code | AI Response Parser | Message a model | Send to WebApp | Parses Gemini JSON string to mappable fields. |
| Send to WebApp | HTTP Request | Content Publication | Code in JavaScript1 | Wait1 | POSTs guide to web app endpoint. |
| Wait1 | Wait | Rate Limit Protection | Send to WebApp | Loop Over Items1 | Pauses execution before returning to loop. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger & Configuration:**
    - Create a **Schedule Trigger** set to trigger every day at 08:00.
    - Create a **Code Node** (`List of Subreddits1`) returning an array of objects: `return [{json:{name: "subreddit1"}}, {json:{name: "subreddit2"}}]`.

2.  **Iteration Logic:**
    - Add a **Split in Batches** node (`Loop Over Items1`) to handle the subreddits one by one.

3.  **Data Retrieval:**
    - Create an **HTTP Request** node (`Get Reddit JSON1`) using the GET method. URL: `https://www.reddit.com/r/{{ $json.name }}/hot.rss?limit=10`.
    - Create a **Code Node** (`Code in JavaScript`) to parse the RSS XML. Implement the `getTag` helper function to extract `<title>`, `<name>`, and `<content>` tags, then use `.replace()` to strip HTML tags.

4.  **AI Integration:**
    - Add a **Google Gemini** node (`Message a model`). 
    - **Credentials:** Connect your Google Palm/Gemini API key.
    - **Model:** Select `gemini-3-flash-preview`.
    - **Prompt:** Paste the detailed "Gaming Journalist" system prompt, ensuring the `{{ JSON.stringify($json.trending_posts) }}` expression is included at the end.
    - Set the node to **JSON Output** mode.
    - Add a **Code Node** (`Code in JavaScript1`) to extract the result: `JSON.parse(item.candidates[0].content.parts[0].text)`.

5.  **Publication & Closing:**
    - Create an **HTTP Request** node (`Send to WebApp`) using the POST method.
    - Map the body parameters: `title` $\rightarrow$ `{{ $json.headline }}` and `content` $\rightarrow$ `{{ $json.article_body }}`.
    - Add a **Wait Node** (`Wait1`) set to "immediately" or a few seconds.
    - Connect the output of the **Wait Node** back to the input of the **Split in Batches** node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Technical Stack | n8n, Reddit RSS, Gemini AI, Supabase Edge Functions |
| Rate Limiting | If Reddit returns 429 errors, increase the duration of the `Wait` node. |
| Authentication | The `Send to WebApp` node currently has no headers. If using Supabase, add an `Authorization: Bearer [KEY]` header. |
| Prompting | The prompt explicitly forbids "AI sounding" robotic fluff to maintain a professional blog tone. |