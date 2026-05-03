Generate weekly Reddit startup opportunity reports with Groq AI

https://n8nworkflows.xyz/workflows/generate-weekly-reddit-startup-opportunity-reports-with-groq-ai-15006


# Generate weekly Reddit startup opportunity reports with Groq AI

Now I'll write the comprehensive document. 1. Workflow Overview

**Purpose:** This workflow automatically generates a weekly startup opportunity report by mining Reddit posts for frustration and unmet-need signals. Every Monday at 8 AM (or on manual execution), it scans 10 entrepreneur-focused subreddits, fetches the 100 newest posts from each, filters them against 27 pain-point keywords, aggregates the matches, sends them to Groq's LLM for structured analysis, and delivers a formatted HTML email report to an inbox — all in under 60 seconds.

**Target Use Cases:**
- Entrepreneurs and product builders looking for validated pain points to build solutions around
- Startup ideation: discover underserved niches and quick-win MVP opportunities
- Ongoing competitive intelligence: monitor recurring complaints across communities weekly

**Logical Blocks:**

| Block | Name | Nodes | Purpose |
|-------|------|-------|---------|
| 0 | **Trigger & Entry** | Schedule Trigger, Manual Trigger | Initiates the workflow either on schedule or manually |
| 1 | **Subreddit Definition** | Define Subreddits (Code) | Outputs the list of 10 target subreddits as individual items |
| 2 | **Reddit Data Collection** | Fetch Reddit Posts (HTTP Request) | Retrieves the 100 newest posts from each subreddit via Reddit's public JSON API |
| 3 | **Pain-Point Filtering** | Filter Pain Points (Code) | Scans post titles and bodies for 27 frustration-signal keywords; drops non-matching posts |
| 4 | **Aggregation** | Aggregate All Posts | Merges all filtered posts from all subreddits into a single array |
| 5 | **AI Prompt Construction** | Build AI Prompt (Code) | Constructs a system + user prompt embedding all posts and structured report instructions |
| 6 | **AI Analysis** | HTTP Request @Groq | Sends the prompt to Groq's `llama-3.3-70b-versatile` model and receives a structured opportunity report |
| 7 | **Email Formatting & Delivery** | Format Email (Code), Gmail Send Email Report | Wraps the AI output in a styled HTML email template and delivers it via Gmail |

---

## 2. Block-by-Block Analysis

### Block 0: Trigger & Entry

**Overview:** Two entry points exist — a scheduled trigger that fires every Monday at 8 AM, and a manual trigger for on-demand execution. Both converge into the same processing pipeline starting at Block 1.

**Nodes Involved:** Every Monday 8AM, When clicking 'Execute workflow'

---

**Node: Every Monday 8AM**
- **Type:** `n8n-nodes-base.scheduleTrigger` (v1.2)
- **Technical Role:** Cron-like trigger that starts the workflow on a weekly schedule.
- **Configuration:** Fires every week on Monday (`triggerAtDay: [1]`) at hour 8 (`triggerAtHour: 8`).
- **Key Expressions/Variables:** None.
- **Input Connections:** None (entry point).
- **Output Connections:** → `1. Define Subreddits`
- **Version-Specific Requirements:** v1.2 supports the `weeks` interval field with day/hour granularity.
- **Edge Cases / Failure Types:**
  - If the n8n instance is down at the scheduled time, the trigger will not fire (no catch-up mechanism by default).
  - Timezone depends on the n8n server's configured timezone, not the user's local time.

---

**Node: When clicking 'Execute workflow'**
- **Type:** `n8n-nodes-base.manualTrigger` (v1)
- **Technical Role:** Allows manual execution from the n8n UI for testing or ad-hoc runs.
- **Configuration:** No parameters.
- **Key Expressions/Variables:** None.
- **Input Connections:** None (entry point).
- **Output Connections:** → `1. Define Subreddits`
- **Version-Specific Requirements:** None.
- **Edge Cases / Failure Types:** No automated failures; only activated by user interaction.

---

### Block 1: Subreddit Definition

**Overview:** A Code node outputs 10 subreddit names as separate items, enabling the downstream HTTP Request node to execute once per subreddit and fetch posts from each.

**Nodes Involved:** 1. Define Subreddits

---

**Node: 1. Define Subreddits**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Generates a list of subreddit names as individual output items.
- **Configuration:** Runs a JavaScript snippet that maps an array of 10 subreddit strings into n8n item format.
- **Key Expressions/Variables:**
  - Hardcoded array: `["entrepreneur", "smallbusiness", "freelance", "SaaS", "startups", "productivity", "webdev", "marketing", "consulting", "Accounting"]`
  - Each output item: `{ json: { subreddit: s } }`
- **Input Connections:** ← `Every Monday 8AM` OR ← `When clicking 'Execute workflow'`
- **Output Connections:** → `2. Fetch Reddit Posts`
- **Version-Specific Requirements:** Code node v2 uses the `$input` model; here only `return` is used.
- **Edge Cases / Failure Types:**
  - If a subreddit name is misspelled, the downstream HTTP request will receive a 404 from Reddit but the workflow will not halt (the error handling depends on the HTTP Request node's settings).
  - Adding/removing subreddits requires editing the hardcoded array.

---

### Block 2: Reddit Data Collection

**Overview:** Makes one HTTP GET request per subreddit item to Reddit's public JSON API, retrieving up to 100 newest posts. No Reddit authentication is required; only a User-Agent header is sent.

**Nodes Involved:** 2. Fetch Reddit Posts

---

**Node: 2. Fetch Reddit Posts**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Fetches recent Reddit posts for each subreddit.
- **Configuration:**
  - **URL:** `https://www.reddit.com/r/{{ $json.subreddit }}/new.json` (dynamically constructed from each input item's `subreddit` field)
  - **Method:** GET
  - **Query Parameters:** `limit=100`
  - **Headers:** `User-Agent: n8n-pain-miner/1.0 (automated research tool)`
  - **Authentication:** None (public API)
- **Key Expressions/Variables:** `$json.subreddit` interpolated in URL.
- **Input Connections:** ← `1. Define Subreddits`
- **Output Connections:** → `3. Filter Pain Points`
- **Version-Specific Requirements:** v4.2 supports the `specifyBody` and `sendQuery`/`sendHeaders` parameter structure.
- **Edge Cases / Failure Types:**
  - **Rate limiting:** Reddit may return HTTP 429 if too many requests are made in rapid succession. The 10 sequential requests are usually fine, but repeated manual executions could trigger this.
  - **Subreddit not found (404):** Invalid subreddit name returns a 404; this will cause the node to error unless error handling is configured.
  - **Empty response:** Private or archived subreddits may return empty data.
  - **Network timeout:** Default timeout applies; Reddit JSON endpoints are generally fast.

---

### Block 3: Pain-Point Filtering

**Overview:** A Code node processes each subreddit's posts, scanning the combined title and body text against 27 frustration-signal keywords. Matching posts are extracted with key metadata; non-matching posts are silently dropped.

**Nodes Involved:** 3. Filter Pain Points

---

**Node: 3. Filter Pain Points**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Filters Reddit posts to only those expressing frustration, unmet needs, or wishes for tools.
- **Configuration:** Runs a JavaScript snippet.
- **Key Expressions/Variables:**
  - Reads input from `$json.data.children` (Reddit API response structure).
  - 27 keywords (case-insensitive match): `"i hate"`, `"why doesn't"`, `"why isn't"`, `"there's no tool"`, `"there is no tool"`, `"wish there was"`, `"no app for"`, `"why is there no"`, `"someone should build"`, `"why can't"`, `"so frustrated"`, `"sick of manually"`, `"tired of doing"`, `"doesn't exist"`, `"pain point"`, `"annoying that"`, `"is it just me"`, `"can't find anything"`, `"i need a way to"`, `"such a waste of time"`, `"why do i have to"`, `"manual process"`, `"no solution"`, `"nobody has built"`, `"looking for a tool"`, `"does anyone know of a"`, `"is there a way to automate"`, `"still no way to"`
  - Each matching post outputs: `{ subreddit, title, text (truncated to 400 chars), url, score, comments }`
  - If no posts match, returns an empty array `[]`.
- **Input Connections:** ← `2. Fetch Reddit Posts`
- **Output Connections:** → `4. Aggregate All Posts`
- **Version-Specific Requirements:** None.
- **Edge Cases / Failure Types:**
  - **Zero matches:** Returns an empty array; the Aggregate node receives no items. The downstream Build AI Prompt node handles this by generating a "no pain points found" prompt.
  - **Keyword false positives:** Broad phrases like `"is it just me"` could match posts that aren't actually expressing pain points. Sensitivity tuning is recommended.
  - **Reddit API structure change:** If Reddit changes its JSON structure (e.g., `data.children` path), this node will fail silently (returning empty).

---

### Block 4: Aggregation

**Overview:** Merges all filtered post items from all subreddit iterations into a single array field called `posts`, enabling the AI prompt builder to work with the entire dataset as one item.

**Nodes Involved:** 4. Aggregate All Posts

---

**Node: 4. Aggregate All Posts**
- **Type:** `n8n-nodes-base.aggregate` (v1)
- **Technical Role:** Collects all items into one item with an array field.
- **Configuration:**
  - **Aggregate mode:** `aggregateAllItemData`
  - **Destination field name:** `posts`
  - **Options:** default (none specified)
- **Key Expressions/Variables:** None.
- **Input Connections:** ← `3. Filter Pain Points`
- **Output Connections:** → `5. Build AI Prompt`
- **Version-Specific Requirements:** v1 of the Aggregate node.
- **Edge Cases / Failure Types:**
  - **Empty input:** If all posts were filtered out, the aggregate receives 0 items and outputs a single item with `posts: []`. The downstream node handles this gracefully.
  - **Large input:** If keyword matching is too loose, the `posts` array could be very large, potentially hitting memory or token limits downstream.

---

### Block 5: AI Prompt Construction

**Overview:** Builds a complete chat prompt (system + user messages) embedding all aggregated posts with detailed instructions for the AI to produce a structured opportunity report with four sections: Top Pain Point Clusters, Best Opportunities, Underserved Niches, and Quick Wins.

**Nodes Involved:** 5. Build AI Prompt

---

**Node: 5. Build AI Prompt**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Constructs the full AI request body and stores the post count for downstream use.
- **Configuration:** Runs a JavaScript snippet.
- **Key Expressions/Variables:**
  - Reads `$json.posts` (the aggregated array from the previous node).
  - If `posts` is empty, generates a fallback prompt: *"No pain points were found this week across the monitored subreddits."* with model `gpt-4o`, max_tokens 500.
  - If posts exist, constructs:
    - **System prompt:** Instructs the AI to act as a "sharp, no-fluff startup opportunity analyst" who "thinks like a product builder."
    - **User prompt:** Embeds all posts in a numbered format with metadata (subreddit, title, text, score, comments, URL), then requests output in exactly four sections: 🔥 Top Pain Point Clusters, 💡 Best Opportunities, 🎯 Underserved Niches, ⚡ Quick Wins.
  - Output fields: `requestBody` (object with model, max_tokens, messages), `postCount` (integer).
  - The `requestBody.model` is set to `"gpt-4o"` and `max_tokens` to 2500, but the actual Groq call overrides the model (see Block 6).
- **Input Connections:** ← `4. Aggregate All Posts`
- **Output Connections:** → `6. HTTP Request @Groq`
- **Version-Specific Requirements:** None.
- **Edge Cases / Failure Types:**
  - **Token overflow:** If the combined post text exceeds the context window of `llama-3.3-70b-versatile` (~128K tokens), the Groq API call may fail or truncate. In practice, 100 posts × 400 chars each = ~40K characters, well within limits.
  - **Empty posts array:** Handled gracefully with a fallback prompt.
  - **Selftext truncation:** Post bodies are truncated to 400 characters during filtering, which may cut off context. This is a deliberate trade-off for token efficiency.

---

### Block 6: AI Analysis

**Overview:** Sends the constructed prompt to Groq's `llama-3.3-70b-versatile` model via the OpenAI-compatible chat completions endpoint. The model returns a structured analysis of pain points, opportunities, niches, and quick wins.

**Nodes Involved:** 6. HTTP Request @Groq

---

**Node: 6. HTTP Request @Groq**
- **Type:** `n8n-nodes-base.httpRequest` (v4.3)
- **Technical Role:** Calls Groq's chat completions API with the AI prompt.
- **Configuration:**
  - **URL:** `https://api.groq.com/openai/v1/chat/completions`
  - **Method:** POST
  - **Body type:** JSON (specified via `specifyBody: "json"`)
  - **JSON Body expression:**
    ```json
    {
      "model": "llama-3.3-70b-versatile",
      "max_tokens": 2500,
      "messages": {{ JSON.stringify($('5. Build AI Prompt').first().json.requestBody.messages) }}
    }
    ```
  - **Authentication:** `predefinedCredentialType` — references a `groqApi` credential (HTTP Header Auth with `Authorization: Bearer <key>`).
- **Key Expressions/Variables:**
  - `$('5. Build AI Prompt').first().json.requestBody.messages` — pulls the messages array from the Build AI Prompt node's output.
  - Model is hardcoded to `llama-3.3-70b-versatile` regardless of what the prompt builder sets.
  - `max_tokens` is set to 2500.
- **Input Connections:** ← `5. Build AI Prompt`
- **Output Connections:** → `7. Format Email`
- **Version-Specific Requirements:** v4.3 of the HTTP Request node.
- **Edge Cases / Failure Types:**
  - **Authentication failure (401):** Invalid or expired Groq API key. Re-create the credential at [console.groq.com](https://console.groq.com).
  - **Rate limiting (429):** Groq free tier has rate limits. If hit, consider adding a retry or delay.
  - **Model unavailable:** If `llama-3.3-70b-versatile` is deprecated or renamed, the API returns an error. Update the model name in the JSON body.
  - **Max tokens too low:** At 2500 tokens, longer reports may be truncated. Increase if needed.
  - **Credential not configured:** The `groqApi` credential ID is empty in the template — must be configured before first run.

---

### Block 7: Email Formatting & Delivery

**Overview:** Extracts the AI response content, formats it into a polished HTML email with a dark header, metadata bar, and styled body, then sends it via Gmail.

**Nodes Involved:** 7. Format Email, 8. Gmail. Send Email Report

---

**Node: 7. Format Email**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Transforms the Groq API response into an HTML email with subject line.
- **Configuration:** Runs a JavaScript snippet.
- **Key Expressions/Variables:**
  - Extracts AI content: `$json.choices?.[0]?.message?.content` — follows OpenAI-compatible response format from Groq. Falls back to `"AI analysis failed."` if the path is missing.
  - Retrieves post count: `$('5. Build AI Prompt').first().json.postCount` — references the Build AI Prompt node for the count.
  - Generates `today` as a formatted date string (e.g., "Monday, June 16, 2025").
  - Constructs HTML with:
    - Dark header (`#0f0f0f`) with title "🔍 Reddit Pain Mining Report" and date
    - Orange-accented metadata bar showing post count and subreddit count
    - White content area with `<pre>` styling for the AI report (preserving formatting)
    - Footer with attribution and subreddit list
  - Output fields: `subject` (e.g., `🔍 Reddit Pain Mining — 42 Opportunities Found (Monday, June 16, 2025)`), `htmlBody` (full HTML string).
- **Input Connections:** ← `6. HTTP Request @Groq`
- **Output Connections:** → `8. Gmail. Send Email Report`
- **Version-Specific Requirements:** None.
- **Edge Cases / Failure Types:**
  - **AI response structure change:** If Groq changes the response format away from the OpenAI-compatible `choices[0].message.content` structure, the fallback "AI analysis failed." message will be sent.
  - **HTML injection:** The AI content is inserted raw into `<pre>` tags. If the AI output contains HTML, it could render unexpectedly. The `<pre>` tag with `white-space: pre-wrap` mitigates most XSS risks but doesn't sanitize.

---

**Node: 8. Gmail. Send Email Report**
- **Type:** `n8n-nodes-base.gmail` (v2.1)
- **Technical Role:** Sends the formatted HTML email report.
- **Configuration:**
  - **To:** `your@email.com` (placeholder — must be changed to the recipient's email)
  - **Subject:** `={{ $json.subject }}`
  - **Message (HTML body):** `={{ $json.htmlBody }}`
  - **Options:** default (none specified)
  - **Authentication:** Gmail OAuth2 credential (`gmailOAuth2`) — must be configured.
- **Key Expressions/Variables:** `$json.subject`, `$json.htmlBody`.
- **Input Connections:** ← `7. Format Email`
- **Output Connections:** None (terminal node).
- **Version-Specific Requirements:** Gmail node v2.1 requires OAuth2 credential setup through Google.
- **Edge Cases / Failure Types:**
  - **OAuth2 token expired:** Gmail OAuth tokens expire; n8n handles refresh automatically if the credential was set up correctly, but expired/revoked tokens will cause send failure.
  - **Recipient not changed:** The default `your@email.com` will cause a delivery failure. Must be updated.
  - **Gmail sending limits:** Google accounts have daily sending limits (~500 for regular Gmail). One email per week is well within limits.
  - **HTML rendering:** Some email clients may not render the HTML perfectly; the `<pre>` block and inline styles ensure decent cross-client compatibility.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Monday 8AM | Schedule Trigger | Weekly Monday 8AM trigger | — | 1. Define Subreddits | Reddit Pain Mining — Weekly Opportunity Report: Automatically scans 10 subreddits every Monday, filters ~1000 posts for genuine frustration signals, runs AI analysis, and delivers a structured startup opportunity report to your inbox — all in under 60 seconds.; Workflow diagram: Schedule Trigger → fires every Monday 8AM |
| When clicking 'Execute workflow' | Manual Trigger | On-demand manual execution | — | 1. Define Subreddits | Reddit Pain Mining — Weekly Opportunity Report: Automatically scans 10 subreddits every Monday, filters ~1000 posts for genuine frustration signals, runs AI analysis, and delivers a structured startup opportunity report to your inbox — all in under 60 seconds.; Workflow diagram: (same as above, alternative entry) |
| 1. Define Subreddits | Code | Outputs 10 subreddit names as individual items | Every Monday 8AM, When clicking 'Execute workflow' | 2. Fetch Reddit Posts | Reddit Pain Mining — Weekly Opportunity Report: Edit the subreddit list in the Define Subreddits node to target your specific niche; Workflow diagram: Define Subreddits → outputs 10 items, one per subreddit |
| 2. Fetch Reddit Posts | HTTP Request | Fetches 100 newest posts per subreddit via Reddit JSON API | 1. Define Subreddits | 3. Filter Pain Points | Reddit Pain Mining — Weekly Opportunity Report: No Reddit auth needed — uses public JSON API with a User-Agent header; Workflow diagram: Fetch Reddit Posts → HTTP GET to reddit.com/r/{subreddit}/new.json — runs 10 times |
| 3. Filter Pain Points | Code | Filters posts against 27 frustration-signal keywords | 2. Fetch Reddit Posts | 4. Aggregate All Posts | Reddit Pain Mining — Weekly Opportunity Report: Add or remove keywords in the Filter Pain Points node to tune sensitivity; Workflow diagram: Filter Pain Points → Code node scanning title + body for 27 keywords; Match → passes forward; No match → item dropped silently |
| 4. Aggregate All Posts | Aggregate | Merges all filtered posts into a single array | 3. Filter Pain Points | 5. Build AI Prompt | Reddit Pain Mining — Weekly Opportunity Report: Each run analyzes ~1000 posts and produces actionable startup/product opportunities in under 60 seconds; Workflow diagram: Aggregate All Posts → merges all 10 subreddit results into one array |
| 5. Build AI Prompt | Code | Constructs system + user prompt with all posts embedded | 4. Aggregate All Posts | 6. HTTP Request @Groq | Workflow diagram: Build AI Prompt → Code node constructing full system + user prompt with all posts embedded |
| 6. HTTP Request @Groq | HTTP Request | Calls Groq chat completions API with LLM prompt | 5. Build AI Prompt | 7. Format Email | Reddit Pain Mining — Weekly Opportunity Report: Groq account — free at https://console.groq.com, create an API key, add as HTTP Header Auth credential; Replace Groq with OpenAI GPT-4o by swapping the HTTP Request URL and auth header; Workflow diagram: HTTP Request → Groq → POST to api.groq.com/openai/v1/chat/completions; Model: llama-3.3-70b-versatile; Returns: clustered pain points, best opportunities, underserved niches, quick wins |
| 7. Format Email | Code | Wraps AI output in styled HTML email template | 6. HTTP Request @Groq | 8. Gmail. Send Email Report | Workflow diagram: Format Email → Code node wrapping AI output in styled HTML template |
| 8. Gmail. Send Email Report | Gmail | Sends final HTML email report | 7. Format Email | — | Reddit Pain Mining — Weekly Opportunity Report: Gmail OAuth2 — connect your Google account via n8n's built-in Gmail credential; Open the Gmail node and change the To field to your own email address; Swap the Gmail node for Slack, Telegram, or Outlook if preferred; Workflow diagram: Gmail → sends final report to your inbox; Success → report delivered; Gmail auth failure → reconnect OAuth credential; Example output sections: The weekly email report is structured into four sections: 🔥 Top Pain Point Clusters, 💡 Best Opportunities, 🎯 Underserved Niches, ⚡ Quick Wins |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your n8n instance.

2. **Add a Schedule Trigger node.**
   - Type: `Schedule Trigger`
   - Version: 1.2
   - Configuration: Interval = `weeks`, trigger at Day = `Monday` (value 1), Hour = `8`
   - This node fires the workflow every Monday at 8:00 AM.

3. **Add a Manual Trigger node.**
   - Type: `Manual Trigger`
   - Version: 1
   - Configuration: No parameters needed.
   - This provides an on-demand execution option.

4. **Connect both trigger nodes** to the next node (Step 5). Each trigger outputs to the same destination.

5. **Add a Code node: "1. Define Subreddits".**
   - Type: `Code`
   - Version: 2
   - JavaScript code: Return an array of 10 subreddit names, each as `{ json: { subreddit: "name" } }`. The subreddits are: `entrepreneur`, `smallbusiness`, `freelance`, `SaaS`, `startups`, `productivity`, `webdev`, `marketing`, `consulting`, `Accounting`.
   - Connect inputs from both trigger nodes.

6. **Add an HTTP Request node: "2. Fetch Reddit Posts".**
   - Type: `HTTP Request`
   - Version: 4.2
   - Method: `GET`
   - URL: `https://www.reddit.com/r/{{ $json.subreddit }}/new.json` (expression)
   - Query Parameters: `limit` = `100` (enable "Send Query Parameters")
   - Headers: `User-Agent` = `n8n-pain-miner/1.0 (automated research tool)` (enable "Send Headers")
   - Authentication: None
   - Options: default
   - Connect input from "1. Define Subreddits"

7. **Add a Code node: "3. Filter Pain Points".**
   - Type: `Code`
   - Version: 2
   - JavaScript code:
     - Extract `posts` from `$json.data.children`, mapping each to `p.data`.
     - Define the 27 keywords array (see Block 3 details above).
     - Filter posts where the combined lowercase title + body text contains any keyword.
     - If no matches, return empty array.
     - For each match, output: `{ subreddit: "r/{name}", title, text (truncated 400 chars), url: "https://reddit.com{permalink}", score, comments }`.
   - Connect input from "2. Fetch Reddit Posts"

8. **Add an Aggregate node: "4. Aggregate All Posts".**
   - Type: `Aggregate`
   - Version: 1
   - Mode: `aggregateAllItemData`
   - Destination field: `posts`
   - Options: default
   - Connect input from "3. Filter Pain Points"

9. **Add a Code node: "5. Build AI Prompt".**
   - Type: `Code`
   - Version: 2
   - JavaScript code:
     - Read `$json.posts`. If empty, return a fallback requestBody with a single user message saying "No pain points were found this week across the monitored subreddits." (model: `gpt-4o`, max_tokens: 500, postCount: 0).
     - If posts exist, format each post as a numbered entry with subreddit, title, text, score, comments, and URL, separated by `---` dividers.
     - Define a system prompt instructing the AI to act as a "sharp, no-fluff startup opportunity analyst" who "thinks like a product builder."
     - Define a user prompt requesting exactly four sections: 🔥 Top Pain Point Clusters, 💡 Best Opportunities, 🎯 Underserved Niches, ⚡ Quick Wins — with specific sub-requirements for each.
     - Output: `{ requestBody: { model: "gpt-4o", max_tokens: 2500, messages: [{role: "system", content: systemPrompt}, {role: "user", content: userPrompt}] }, postCount: posts.length }`
   - Connect input from "4. Aggregate All Posts"

10. **Add an HTTP Request node: "6. HTTP Request @Groq".**
    - Type: `HTTP Request`
    - Version: 4.3
    - Method: `POST`
    - URL: `https://api.groq.com/openai/v1/chat/completions`
    - Body specification: `JSON`
    - JSON Body (expression):
      ```
      {
        "model": "llama-3.3-70b-versatile",
        "max_tokens": 2500,
        "messages": {{ JSON.stringify($('5. Build AI Prompt').first().json.requestBody.messages) }}
      }
      ```
    - Authentication: `Predefined Credential Type` → `Groq API` (HTTP Header Auth)
    - **Credential setup:** Go to [console.groq.com](https://console.groq.com), create a free account, generate an API key, then in n8n create a credential of type "Header Auth" with name `Authorization` and value `Bearer <your-groq-api-key>`. Assign it as `groqApi` credential on this node.
    - Connect input from "5. Build AI Prompt"

11. **Add a Code node: "7. Format Email".**
    - Type: `Code`
    - Version: 2
    - JavaScript code:
      - Extract AI content: `$json.choices?.[0]?.message?.content` (fallback: `"AI analysis failed."`)
      - Retrieve post count: `$('5. Build AI Prompt').first().json.postCount`
      - Generate formatted date string for today.
      - Build HTML email with:
        - Dark header (`background: #0f0f0f`) with title "🔍 Reddit Pain Mining Report" and date
        - Orange-bordered metadata bar with post count and "10 subreddits"
        - White content area with `<pre style="white-space: pre-wrap">` containing the AI report
        - Centered footer with attribution and subreddit list
      - Output: `{ subject: "🔍 Reddit Pain Mining — {count} Opportunities Found ({date})", htmlBody: "<html...>" }`
    - Connect input from "6. HTTP Request @Groq"

12. **Add a Gmail node: "8. Gmail. Send Email Report".**
    - Type: `Gmail`
    - Version: 2.1
    - **To:** Change from `your@email.com` to your actual recipient email address.
    - **Subject:** `={{ $json.subject }}`
    - **Message:** `={{ $json.htmlBody }}`
    - **Options:** default
    - **Credential setup:** In n8n, create a Gmail OAuth2 credential. You'll be redirected to Google to authorize n8n to send emails on your behalf. Assign it as the `gmailOAuth2` credential on this node.
    - Connect input from "7. Format Email"

13. **Verify all connections** match the following flow:
    - Every Monday 8AM → 1. Define Subreddits
    - When clicking 'Execute workflow' → 1. Define Subreddits
    - 1. Define Subreddits → 2. Fetch Reddit Posts
    - 2. Fetch Reddit Posts → 3. Filter Pain Points
    - 3. Filter Pain Points → 4. Aggregate All Posts
    - 4. Aggregate All Posts → 5. Build AI Prompt
    - 5. Build AI Prompt → 6. HTTP Request @Groq
    - 6. HTTP Request @Groq → 7. Format Email
    - 7. Format Email → 8. Gmail. Send Email Report

14. **Test the workflow** by clicking "Execute workflow" (manual trigger) before relying on the schedule. Verify:
    - Reddit API returns data (check HTTP Request node output)
    - Pain-point filtering produces matches (check Code node output)
    - Groq API returns a response (check HTTP Request node output)
    - Email is received with formatted content

15. **Activate the workflow** by toggling it to "Active" so the schedule trigger runs automatically.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Groq console — create a free API key for LLM access | [https://console.groq.com](https://console.groq.com) |
| Groq credential must be configured as HTTP Header Auth with `Authorization: Bearer <key>` | Applies to node "6. HTTP Request @Groq" |
| Gmail OAuth2 credential must be connected via n8n's built-in Google OAuth flow | Applies to node "8. Gmail. Send Email Report" |
| No Reddit authentication is required — the workflow uses Reddit's public JSON API with a custom User-Agent header | Applies to node "2. Fetch Reddit Posts" |
| The recipient email address `your@email.com` in the Gmail node is a placeholder and must be replaced | Must be changed before first run |
| The workflow processes approximately 1,000 posts per run (10 subreddits × 100 posts each) | Typical weekly execution |
| To increase frequency, change the Schedule Trigger from weekly to daily | Edit node "Every Monday 8AM" interval settings |
| To change the target niche, edit the subreddit array in node "1. Define Subreddits" | Direct array modification in the Code node |
| To tune sensitivity, add or remove keywords in node "3. Filter Pain Points" | Currently 27 keywords defined |
| To switch from Groq to OpenAI GPT-4o, change the URL to `https://api.openai.com/v1/chat/completions` and update the authentication credential | Alternative LLM provider |
| To switch delivery from Gmail to Slack, Telegram, or Outlook, replace the Gmail node with the appropriate node type | Output channel flexibility |
| The AI model `llama-3.3-70b-versatile` is specified in the HTTP Request body, overriding the `gpt-4o` model name set in the prompt builder | The `model` field in Build AI Prompt is not used by the Groq call |
| Max tokens for the Groq completion is set to 2500; increase if reports are getting truncated | Adjust in both the JSON body of node "6. HTTP Request @Groq" and the `max_tokens` in node "5. Build AI Prompt" |
| The `<pre>` tag in the email template preserves AI formatting (markdown-style headings, bullet lists) but does not render actual markdown | Consider adding a markdown-to-HTML converter if richer formatting is needed |