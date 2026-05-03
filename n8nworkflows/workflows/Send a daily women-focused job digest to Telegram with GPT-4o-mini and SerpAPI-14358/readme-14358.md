Send a daily women-focused job digest to Telegram with GPT-4o-mini and SerpAPI

https://n8nworkflows.xyz/workflows/send-a-daily-women-focused-job-digest-to-telegram-with-gpt-4o-mini-and-serpapi-14358


# Send a daily women-focused job digest to Telegram with GPT-4o-mini and SerpAPI

# 1. Workflow Overview

This workflow builds and sends a daily Telegram digest of women-focused job opportunities in India. Every day at 9:00 AM, it queries Google Jobs through SerpAPI using four targeted search phrases, merges the results, extracts job entries, filters them by relevant keywords, ranks them by recency, validates and formats the shortlist with GPT-4o-mini, and sends the final digest to Telegram.

Its primary use cases are:
- daily job discovery for women-oriented opportunities
- diversity and returnship opportunity monitoring
- automated curation of job leads into a human-readable Telegram message

## 1.1 Scheduled Input Reception
The workflow starts from a daily schedule trigger configured to run at 9:00 AM server time.

## 1.2 Parallel Job Search Retrieval
Four HTTP Request nodes run in parallel against SerpAPI’s Google Jobs engine, each using a different women-focused search query scoped to India.

## 1.3 Result Consolidation and Job Extraction
The fetched API responses are merged and flattened into individual job items with normalized fields such as title, company, location, description, and apply link.

## 1.4 Keyword Filtering and Recency Ranking
Jobs are first filtered by a regex-based relevance check on the description, then sorted by posting age and reduced to the top 3 most recent entries.

## 1.5 AI Validation and Message Formatting
Each shortlisted job is passed to an AI Agent using the OpenAI Chat Model `gpt-4o-mini`. The model determines whether the listing is truly relevant and formats accepted jobs for display. Irrelevant jobs are expected to return `IGNORE`.

## 1.6 Telegram Digest Delivery
All AI outputs are combined into one digest message and sent to a Telegram chat using HTML parse mode.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception

**Overview:**  
This block defines the workflow’s entry point. It launches the workflow once per day and fans out into the four job-search requests.

**Nodes Involved:**  
- Daily 9AM Trigger

### Node Details

#### Daily 9AM Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based workflow entry node.
- **Configuration choices:**  
  Configured with cron expression `0 9 * * *`, which means every day at 09:00.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Outputs to:
    - Fetch: Women Returnship Jobs
    - Fetch: Diversity Hiring Jobs
    - Fetch: Remote Jobs for Women
    - Fetch: Female Hiring Initiatives
- **Version-specific requirements:**  
  Uses node type version `1.1`.
- **Edge cases or potential failure types:**  
  - Trigger time uses the n8n server timezone, not necessarily the local business timezone.
  - If the workflow is inactive, it will not run automatically.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Parallel Job Search Retrieval

**Overview:**  
This block performs four Google Jobs searches through SerpAPI in parallel. Each node uses a different women-related query to broaden coverage of relevant opportunities in India.

**Nodes Involved:**  
- Fetch: Women Returnship Jobs
- Fetch: Diversity Hiring Jobs
- Fetch: Remote Jobs for Women
- Fetch: Female Hiring Initiatives

### Node Details

#### Fetch: Women Returnship Jobs
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends an HTTP GET-style request with query parameters to SerpAPI.
- **Configuration choices:**  
  - URL: `https://serpapi.com/search.json`
  - Query params:
    - `engine=google_jobs`
    - `q=women returnship jobs`
    - `location=India`
    - `api_key=YOUR_SERPAPI_KEY`
- **Key expressions or variables used:**  
  No expressions; hardcoded placeholder API key.
- **Input and output connections:**  
  - Input: Daily 9AM Trigger
  - Output: Merge All Results
- **Version-specific requirements:**  
  Uses node type version `4.1`.
- **Edge cases or potential failure types:**  
  - Invalid or missing SerpAPI key
  - SerpAPI quota exhaustion
  - HTTP/network errors
  - Empty result set
- **Sub-workflow reference:**  
  None.

#### Fetch: Diversity Hiring Jobs
- **Type and technical role:** `n8n-nodes-base.httpRequest`
- **Configuration choices:**  
  Same endpoint and engine, with:
  - `q=diversity hiring jobs`
  - `location=India`
- **Key expressions or variables used:**  
  No expressions.
- **Input and output connections:**  
  - Input: Daily 9AM Trigger
  - Output: Merge All Results
- **Version-specific requirements:**  
  Type version `4.1`.
- **Edge cases or potential failure types:**  
  Same as above; also possible broad/unfocused jobs depending on search quality.
- **Sub-workflow reference:**  
  None.

#### Fetch: Remote Jobs for Women
- **Type and technical role:** `n8n-nodes-base.httpRequest`
- **Configuration choices:**  
  Uses:
  - `q=remote jobs for women`
  - `location=India`
- **Key expressions or variables used:**  
  No expressions.
- **Input and output connections:**  
  - Input: Daily 9AM Trigger
  - Output: Merge All Results
- **Version-specific requirements:**  
  Type version `4.1`.
- **Edge cases or potential failure types:**  
  Same SerpAPI/network concerns; search phrase may return generic remote jobs not specifically targeted to women.
- **Sub-workflow reference:**  
  None.

#### Fetch: Female Hiring Initiatives
- **Type and technical role:** `n8n-nodes-base.httpRequest`
- **Configuration choices:**  
  Uses:
  - `q=female hiring initiatives`
  - `location=India`
- **Key expressions or variables used:**  
  No expressions.
- **Input and output connections:**  
  - Input: Daily 9AM Trigger
  - Output: Merge All Results
- **Version-specific requirements:**  
  Type version `4.1`.
- **Edge cases or potential failure types:**  
  Same as the other HTTP nodes.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Result Consolidation and Job Extraction

**Overview:**  
This block merges the four API response streams and converts each SerpAPI response into individual normalized job items. It is the bridge between raw search responses and downstream filtering logic.

**Nodes Involved:**  
- Merge All Results
- Extract Job Items

### Node Details

#### Merge All Results
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines multiple incoming branches into one stream.
- **Configuration choices:**  
  No custom parameters are set. It relies on default merge behavior.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Inputs:
    - Fetch: Women Returnship Jobs
    - Fetch: Diversity Hiring Jobs
    - Fetch: Remote Jobs for Women
    - Fetch: Female Hiring Initiatives
  - Output:
    - Extract Job Items
- **Version-specific requirements:**  
  Type version `2.1`.
- **Edge cases or potential failure types:**  
  - Merge behavior depends on node defaults in the installed n8n version.
  - If one branch fails and error handling is not enabled, the workflow may stop.
  - If response timing differs, merged output behavior should be verified during testing.
- **Sub-workflow reference:**  
  None.

#### Extract Job Items
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that flattens `jobs_results` arrays into one item per job.
- **Configuration choices:**  
  The code:
  - loops over incoming items
  - reads `item.json.jobs_results || []`
  - creates one output item per job
  - maps fields:
    - `title` from `job.title`
    - `company` from `job.company_name`
    - `location` from `job.location`
    - `description` from `job.description`
    - `apply_link` from `job.apply_options?.[0]?.link || ""`
    - `source` from `job.via`
- **Key expressions or variables used:**  
  Internal JS variables:
  - `jobs`
  - `results`
  - `job`
- **Input and output connections:**  
  - Input: Merge All Results
  - Output: Keyword Filter
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - If SerpAPI response shape changes, `jobs_results` may be missing or renamed.
  - `apply_options` may be empty, producing blank apply links.
  - Important fields like posting date are not preserved, which affects downstream ranking.
  - No deduplication is implemented despite the sticky note claiming deduplication.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Keyword Filtering and Recency Ranking

**Overview:**  
This block narrows the raw job list to likely relevant jobs using regex filtering, then attempts to rank by recency and keep only the top three. It is the main shortlist-generation step before AI review.

**Nodes Involved:**  
- Keyword Filter
- Rank by Recency & Take Top 3

### Node Details

#### Keyword Filter
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional filter node based on regex matching.
- **Configuration choices:**  
  Checks whether `{{$json.description}}` matches the regex:
  `women|diversity|returnship|female|remote`
  using OR logic.
- **Key expressions or variables used:**  
  - Left value: `={{ $json.description }}`
- **Input and output connections:**  
  - Input: Extract Job Items
  - True output: Rank by Recency & Take Top 3
  - False output: not connected
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Case sensitive mode is enabled, so uppercase or capitalized variants may fail to match.
  - If `description` is null or missing, matching may fail or exclude valid jobs.
  - Filtering only on description may miss relevant jobs whose title indicates relevance.
- **Sub-workflow reference:**  
  None.

#### Rank by Recency & Take Top 3
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript sorting and trimming node.
- **Configuration choices:**  
  The code:
  - defines `parsePostedTime(text)`
  - converts values like “2 days ago”, “3 weeks ago”, etc. into numeric scores
  - attempts to read:
    - `item.json.detected_extensions?.posted_at`
    - or `item.json.posted_at`
  - sorts ascending by recency score
  - slices the array to top 3
  - returns only `job.json`
- **Key expressions or variables used:**  
  Internal JS variables:
  - `parsePostedTime`
  - `posted`
  - `jobs`
  - `score`
- **Input and output connections:**  
  - Input: Keyword Filter
  - Output: AI Agent: Validate & Format Job
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - The upstream Extract Job Items node does not preserve `posted_at` or `detected_extensions`, so most items will likely have no posting timestamp here.
  - As a result, all jobs may default to score `9999`, making the “recency” ranking ineffective.
  - If text formats differ from expected phrases like “2 days ago”, parsing will degrade.
  - No deduplication occurs before top-3 selection.
- **Sub-workflow reference:**  
  None.

---

## 2.5 AI Validation and Message Formatting

**Overview:**  
This block uses GPT-4o-mini to perform semantic validation beyond keyword matching. It determines whether the shortlisted jobs are genuinely relevant and formats each accepted job into a digest-ready text block.

**Nodes Involved:**  
- AI Agent: Validate & Format Job
- OpenAI Chat Model

### Node Details

#### AI Agent: Validate & Format Job
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain agent node that sends a prompt and structured task to a connected language model.
- **Configuration choices:**  
  - Prompt is fully defined in the node.
  - The model is instructed to:
    - validate relevance to women/diversity/returnship/remote-friendly roles
    - reject generic or vague listings
    - output a formatted listing if relevant
    - output only `IGNORE` if irrelevant
  - Prompt includes interpolated job data:
    - `{{ $json.title }}`
    - `{{ $json.company }}`
    - `{{ $json.location }}`
    - `{{ $json.description }}`
    - `{{ $json.apply_link }}`
- **Key expressions or variables used:**  
  The prompt references current item fields listed above.
- **Input and output connections:**  
  - Main input: Rank by Recency & Take Top 3
  - AI language model input: OpenAI Chat Model
  - Output: Build Telegram Digest Message
- **Version-specific requirements:**  
  Type version `3`. Requires compatible n8n LangChain/AI nodes.
- **Edge cases or potential failure types:**  
  - If the OpenAI credential is missing or invalid, execution fails.
  - If model output does not follow expected format, downstream digest may include malformed text.
  - There is no explicit post-filter removing `IGNORE`, so those may appear in the final Telegram message.
  - Long descriptions may increase token usage.
- **Sub-workflow reference:**  
  None.

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the language model backend to the AI Agent.
- **Configuration choices:**  
  - Model selected: `gpt-4o-mini`
  - No built-in tools configured
  - Standard OpenAI credential attached
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output via `ai_languageModel` to AI Agent: Validate & Format Job
- **Version-specific requirements:**  
  Type version `1.3`. Requires an n8n version with LangChain AI nodes installed and supported.
- **Edge cases or potential failure types:**  
  - Invalid credential
  - Model access restrictions on the OpenAI account
  - API rate limits or temporary service errors
- **Sub-workflow reference:**  
  None.

---

## 2.6 Telegram Digest Delivery

**Overview:**  
This block combines the AI-generated job summaries into a single daily digest and sends it to Telegram. It is the final output stage of the workflow.

**Nodes Involved:**  
- Build Telegram Digest Message
- Send Daily Digest to Telegram

### Node Details

#### Build Telegram Digest Message
- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates all incoming AI outputs into one message string.
- **Configuration choices:**  
  The code:
  - starts the message with `🔥 Women Job Opportunities (Today)`
  - appends each item as numbered text using `item.json.output`
  - adds a footer: `✨ Stay consistent. Opportunities come daily!`
  - returns a single item with `json.message`
- **Key expressions or variables used:**  
  Internal variables:
  - `message`
  - `item`
  - `index`
- **Input and output connections:**  
  - Input: AI Agent: Validate & Format Job
  - Output: Send Daily Digest to Telegram
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Assumes the AI Agent returns output in `item.json.output`; this should be verified in your n8n version.
  - If AI returns `IGNORE`, it is still appended unless another filter is added.
  - If no items arrive, the node will still send a mostly empty digest unless guarded.
- **Sub-workflow reference:**  
  None.

#### Send Daily Digest to Telegram
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a text message to a Telegram chat using a bot credential.
- **Configuration choices:**  
  - Text: `={{ $json.message }}`
  - `chatId=YOUR_TELEGRAM_CHAT_ID`
  - Parse mode: `HTML`
- **Key expressions or variables used:**  
  - `={{ $json.message }}`
- **Input and output connections:**  
  - Input: Build Telegram Digest Message
  - Output: none
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Invalid Telegram bot credential
  - Wrong chat ID
  - Bot not added to the channel/group
  - Telegram HTML parse failures if generated text contains unsupported or unescaped markup
  - Message length limits if too many or overly long entries are included
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily 9AM Trigger | n8n-nodes-base.scheduleTrigger | Starts the workflow daily at 9:00 AM |  | Fetch: Women Returnship Jobs; Fetch: Diversity Hiring Jobs; Fetch: Remote Jobs for Women; Fetch: Female Hiring Initiatives | ⏰ Trigger — Fires once daily at 9:00 AM (server time). Kicks off all four job searches simultaneously in parallel. |
| Fetch: Women Returnship Jobs | n8n-nodes-base.httpRequest | Queries SerpAPI Google Jobs for women returnship roles in India | Daily 9AM Trigger | Merge All Results | 📡 Job Fetching — Four parallel Google Jobs searches via SerpAPI — returnship programs, diversity hiring, remote roles, and female-focused initiatives. All scoped to India. Results merge into a single stream before processing. |
| Fetch: Diversity Hiring Jobs | n8n-nodes-base.httpRequest | Queries SerpAPI Google Jobs for diversity hiring roles in India | Daily 9AM Trigger | Merge All Results | 📡 Job Fetching — Four parallel Google Jobs searches via SerpAPI — returnship programs, diversity hiring, remote roles, and female-focused initiatives. All scoped to India. Results merge into a single stream before processing. |
| Fetch: Remote Jobs for Women | n8n-nodes-base.httpRequest | Queries SerpAPI Google Jobs for remote jobs for women in India | Daily 9AM Trigger | Merge All Results | 📡 Job Fetching — Four parallel Google Jobs searches via SerpAPI — returnship programs, diversity hiring, remote roles, and female-focused initiatives. All scoped to India. Results merge into a single stream before processing. |
| Fetch: Female Hiring Initiatives | n8n-nodes-base.httpRequest | Queries SerpAPI Google Jobs for female hiring initiatives in India | Daily 9AM Trigger | Merge All Results | 📡 Job Fetching — Four parallel Google Jobs searches via SerpAPI — returnship programs, diversity hiring, remote roles, and female-focused initiatives. All scoped to India. Results merge into a single stream before processing. |
| Merge All Results | n8n-nodes-base.merge | Combines the four search result branches | Fetch: Women Returnship Jobs; Fetch: Diversity Hiring Jobs; Fetch: Remote Jobs for Women; Fetch: Female Hiring Initiatives | Extract Job Items | 📡 Job Fetching — Four parallel Google Jobs searches via SerpAPI — returnship programs, diversity hiring, remote roles, and female-focused initiatives. All scoped to India. Results merge into a single stream before processing. |
| Extract Job Items | n8n-nodes-base.code | Flattens SerpAPI jobs arrays into individual job records | Merge All Results | Keyword Filter |  |
| Keyword Filter | n8n-nodes-base.if | Filters jobs whose descriptions match target keywords | Extract Job Items | Rank by Recency & Take Top 3 | 🔍 Filter & Rank — Job descriptions are regex-filtered for relevance keywords (women, diversity, returnship, female, remote). Passing jobs are then sorted by posting recency and trimmed to the top 3. This keeps the daily digest short and high-signal. |
| Rank by Recency & Take Top 3 | n8n-nodes-base.code | Sorts shortlisted jobs by posting age and keeps top 3 | Keyword Filter | AI Agent: Validate & Format Job | 🔍 Filter & Rank — Job descriptions are regex-filtered for relevance keywords (women, diversity, returnship, female, remote). Passing jobs are then sorted by posting recency and trimmed to the top 3. This keeps the daily digest short and high-signal. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o-mini as the language model |  | AI Agent: Validate & Format Job | 🤖 AI Validation & Formatting — Each job passes through an AI agent (GPT-4o-mini) that checks genuine relevance — rejecting generic senior roles or vague listings — and formats accepted jobs with an emoji tag, plain-language relevance note, and apply link. Jobs that don't qualify return IGNORE and are dropped. |
| AI Agent: Validate & Format Job | @n8n/n8n-nodes-langchain.agent | Validates relevance and formats each shortlisted job | Rank by Recency & Take Top 3; OpenAI Chat Model | Build Telegram Digest Message | 🤖 AI Validation & Formatting — Each job passes through an AI agent (GPT-4o-mini) that checks genuine relevance — rejecting generic senior roles or vague listings — and formats accepted jobs with an emoji tag, plain-language relevance note, and apply link. Jobs that don't qualify return IGNORE and are dropped. |
| Build Telegram Digest Message | n8n-nodes-base.code | Builds one final Telegram digest message from AI outputs | AI Agent: Validate & Format Job | Send Daily Digest to Telegram | 📬 Telegram Dispatch — Formatted job listings are bundled into a single daily digest message and sent to a Telegram chat. HTML parse mode is enabled so emoji tags and line breaks render correctly. |
| Send Daily Digest to Telegram | n8n-nodes-base.telegram | Sends the digest message to a Telegram chat | Build Telegram Digest Message |  | 📬 Telegram Dispatch — Formatted job listings are bundled into a single daily digest message and sent to a Telegram chat. HTML parse mode is enabled so emoji tags and line breaks render correctly. |
| Sticky Note: Overview | n8n-nodes-base.stickyNote | Visual documentation of purpose and setup |  |  | ## 🤝 Women's Daily Job Digest Bot  / ### How it works / Every morning at 9 AM, this workflow searches Google Jobs for four specific job categories — returnship programs, diversity hiring, remote-friendly roles, and female hiring initiatives — all scoped to India. Results are merged, deduplicated, filtered by relevance keywords, ranked by recency, and then sent through an AI agent that validates each listing and formats it for human readability. Only the top 3 most relevant and recent jobs are sent to a Telegram channel each day. / ### Setup steps / 1. **SerpAPI** — Add your SerpAPI key inside each of the four HTTP Request nodes (replace the placeholder key). / 2. **OpenAI** — Connect your OpenAI credential to the `OpenAI Chat Model` node. The workflow uses `gpt-4o-mini` by default. / 3. **Telegram** — Connect a Telegram Bot credential and update the `chatId` in the `Send to Telegram` node with your channel or user ID. / 4. **Test** — Run a manual trigger test to confirm jobs are fetching, filtering correctly, and messages are arriving in Telegram. / 5. **Activate** — Enable the workflow. It will fire automatically every day at 9:00 AM. |
| Sticky Note: Trigger | n8n-nodes-base.stickyNote | Visual documentation for scheduling |  |  | ## ⏰ Trigger / Fires once daily at 9:00 AM (server time). Kicks off all four job searches simultaneously in parallel. |
| Sticky Note: Fetching | n8n-nodes-base.stickyNote | Visual documentation for fetch stage |  |  | ## 📡 Job Fetching / Four parallel Google Jobs searches via SerpAPI — returnship programs, diversity hiring, remote roles, and female-focused initiatives. All scoped to India. Results merge into a single stream before processing. |
| Sticky Note: Filter & Rank | n8n-nodes-base.stickyNote | Visual documentation for filtering stage |  |  | ## 🔍 Filter & Rank / Job descriptions are regex-filtered for relevance keywords (women, diversity, returnship, female, remote). Passing jobs are then sorted by posting recency and trimmed to the top 3. This keeps the daily digest short and high-signal. |
| Sticky Note: AI | n8n-nodes-base.stickyNote | Visual documentation for AI stage |  |  | ## 🤖 AI Validation & Formatting / Each job passes through an AI agent (GPT-4o-mini) that checks genuine relevance — rejecting generic senior roles or vague listings — and formats accepted jobs with an emoji tag, plain-language relevance note, and apply link. Jobs that don't qualify return IGNORE and are dropped. |
| Sticky Note: Telegram | n8n-nodes-base.stickyNote | Visual documentation for Telegram stage |  |  | ## 📬 Telegram Dispatch / Formatted job listings are bundled into a single daily digest message and sent to a Telegram chat. HTML parse mode is enabled so emoji tags and line breaks render correctly. |
| Sticky Note: Security | n8n-nodes-base.stickyNote | Visual documentation for credential handling |  |  | ## 🔐 Credentials & Security / Replace the SerpAPI key in all four HTTP nodes with your own key. Use named credentials for OpenAI and Telegram — never paste raw tokens into shared templates. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `AI-Powered Women Job Discovery & Daily Digest using SerpAPI, GPT-4o-mini, and Telegram`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Name: `Daily 9AM Trigger`
   - Configure a cron rule:
     - Expression: `0 9 * * *`
   - This runs every day at 9:00 AM in the server timezone.

3. **Add the first HTTP Request node**
   - Node type: `HTTP Request`
   - Name: `Fetch: Women Returnship Jobs`
   - Method can remain default for query-based request to URL.
   - URL: `https://serpapi.com/search.json`
   - Enable query parameters.
   - Add:
     - `engine` = `google_jobs`
     - `q` = `women returnship jobs`
     - `location` = `India`
     - `api_key` = your SerpAPI key
   - Connect `Daily 9AM Trigger` to this node.

4. **Add the second HTTP Request node**
   - Name: `Fetch: Diversity Hiring Jobs`
   - Same URL and structure
   - Query parameters:
     - `engine` = `google_jobs`
     - `q` = `diversity hiring jobs`
     - `location` = `India`
     - `api_key` = your SerpAPI key
   - Connect from `Daily 9AM Trigger`.

5. **Add the third HTTP Request node**
   - Name: `Fetch: Remote Jobs for Women`
   - Query parameters:
     - `engine` = `google_jobs`
     - `q` = `remote jobs for women`
     - `location` = `India`
     - `api_key` = your SerpAPI key
   - Connect from `Daily 9AM Trigger`.

6. **Add the fourth HTTP Request node**
   - Name: `Fetch: Female Hiring Initiatives`
   - Query parameters:
     - `engine` = `google_jobs`
     - `q` = `female hiring initiatives`
     - `location` = `India`
     - `api_key` = your SerpAPI key
   - Connect from `Daily 9AM Trigger`.

7. **Add a Merge node**
   - Node type: `Merge`
   - Name: `Merge All Results`
   - Keep default settings unless your n8n version requires explicit append behavior.
   - Connect all four HTTP Request nodes into this Merge node.
   - Important: verify in your n8n version that the merge mode actually collects all four branches as intended.

8. **Add a Code node to flatten the jobs**
   - Node type: `Code`
   - Name: `Extract Job Items`
   - Paste this logic conceptually:
     - Iterate over all incoming items.
     - Read `jobs_results` from each SerpAPI response.
     - Create one output item per job.
     - Map these fields:
       - `title`
       - `company`
       - `location`
       - `description`
       - `apply_link`
       - `source`
   - Use the following code:

   ```javascript
   let jobs = [];

   for (const item of items) {
     const results = item.json.jobs_results || [];
     
     for (const job of results) {
       jobs.push({
         json: {
           title: job.title,
           company: job.company_name,
           location: job.location,
           description: job.description,
           apply_link: job.apply_options?.[0]?.link || "",
           source: job.via
         }
       });
     }
   }

   return jobs;
   ```

   - Connect `Merge All Results` to `Extract Job Items`.

9. **Add an IF node for keyword filtering**
   - Node type: `IF`
   - Name: `Keyword Filter`
   - Create a string regex condition on:
     - Left value: `{{ $json.description }}`
     - Operation: `regex`
     - Right value: `women|diversity|returnship|female|remote`
   - Use OR combinator if building with multiple conditions, though here there is only one regex condition.
   - Connect `Extract Job Items` to `Keyword Filter`.
   - Use only the `true` branch for the next step.

10. **Add a Code node for ranking**
    - Node type: `Code`
    - Name: `Rank by Recency & Take Top 3`
    - Connect the `true` output of `Keyword Filter` to this node.
    - Add code equivalent to:

   ```javascript
   function parsePostedTime(text) {
     if (!text) return 9999;

     text = text.toLowerCase();

     if (text.includes("hour")) {
       return parseInt(text) / 24;
     }
     if (text.includes("day")) {
       return parseInt(text);
     }
     if (text.includes("week")) {
       return parseInt(text) * 7;
     }
     if (text.includes("month")) {
       return parseInt(text) * 30;
     }

     return 9999;
   }

   let jobs = items.map(item => {
     const posted =
       item.json.detected_extensions?.posted_at ||
       item.json.posted_at ||
       "";

     return {
       ...item,
       score: parsePostedTime(posted)
     };
   });

   jobs.sort((a, b) => a.score - b.score);
   jobs = jobs.slice(0, 3);

   return jobs.map(job => ({ json: job.json }));
   ```

   - Important implementation note: as written, this code expects posting metadata that is not preserved by `Extract Job Items`. If you want ranking to work properly, also carry forward `posted_at` or `detected_extensions` from the SerpAPI response in the previous node.

11. **Add the OpenAI Chat Model node**
    - Node type: `OpenAI Chat Model`
    - Name: `OpenAI Chat Model`
    - Select model: `gpt-4o-mini`
    - Attach an OpenAI credential.
    - Credential setup:
      - Create or select an OpenAI API credential in n8n.
      - Ensure the account has access to the selected model.

12. **Add the AI Agent node**
    - Node type: `AI Agent`
    - Name: `AI Agent: Validate & Format Job`
    - Set prompt mode to define text directly.
    - Paste the full instruction prompt used in the workflow, including the output format and `IGNORE` rule.
    - Connect:
      - Main input from `Rank by Recency & Take Top 3`
      - AI language model connection from `OpenAI Chat Model`

13. **Use this prompt in the AI Agent**
   ```text
   You are an assistant for a Women Job Opportunity Bot.

   Your job is to:
   1. Check if the job is truly relevant for:
      - women hiring
      - diversity hiring
      - returnship programs
      - female-only roles
      - remote/flexible jobs suitable for women

   2. Reject low-quality or irrelevant jobs like:
      - generic senior roles (Director, VP without diversity focus)
      - unrelated roles
      - spam or vague listings

   3. If relevant → format cleanly.

   OUTPUT FORMAT:

   [Optional Tag]
   Choose ONE if applicable:
   👩 Women Focused
   🌍 Diversity Hiring
   🏠 Remote Friendly
   🔁 Returnship Program

   Role: [Job Title]
   Company: [Company Name]
   Location: [Location]

   Why it is relevant:
   [1 short simple sentence]

   Apply:
   [Apply Link]

   ---

   If NOT relevant → return ONLY:
   IGNORE

   ---

   Job Data:

   Title: {{ $json.title }}
   Company: {{ $json.company }}
   Location: {{ $json.location }}
   Description: {{ $json.description }}
   Link: {{ $json.apply_link }}
   ```

14. **Add a Code node to build the final Telegram message**
    - Node type: `Code`
    - Name: `Build Telegram Digest Message`
    - Connect from `AI Agent: Validate & Format Job`
    - Use code like:

   ```javascript
   let message = "🔥 Women Job Opportunities (Today)\n\n";

   items.forEach((item, index) => {
     message += `${index + 1}. ${item.json.output}\n\n`;
   });

   message += "✨ Stay consistent. Opportunities come daily!\n";

   return [
     {
       json: {
         message
       }
     }
   ];
   ```

   - Verify that your AI Agent outputs the formatted text under `json.output`. If not, adapt the property path.

15. **Add the Telegram node**
    - Node type: `Telegram`
    - Name: `Send Daily Digest to Telegram`
    - Operation: send message/text
    - Text field: `{{ $json.message }}`
    - Chat ID: your Telegram user, group, or channel ID
    - Additional field:
      - `parse_mode` = `HTML`
    - Connect from `Build Telegram Digest Message`.

16. **Configure Telegram credentials**
    - Create a Telegram Bot with BotFather.
    - Copy the bot token into an n8n Telegram credential.
    - If sending to a channel:
      - Add the bot to the channel
      - Give it permission to post
      - Use the correct channel chat ID or username format supported by your setup

17. **Test the workflow manually**
    - Run the workflow manually.
    - Confirm:
      - each HTTP node returns SerpAPI data
      - `jobs_results` exists
      - `Extract Job Items` produces one item per job
      - `Keyword Filter` passes expected items
      - ranking logic behaves as expected
      - AI outputs are readable
      - Telegram receives one digest message

18. **Correct known implementation gaps before production**
    - Recommended fixes:
      - preserve posting-time metadata in `Extract Job Items`
      - add a node after the AI Agent to filter out items where output equals `IGNORE`
      - consider deduplication before ranking
      - consider making keyword matching case-insensitive
      - add fallback logic if no jobs are found

19. **Activate the workflow**
    - After testing, enable the workflow so the schedule trigger runs automatically every day.

20. **Optional sub-workflow setup**
    - This workflow does **not** invoke any sub-workflow nodes.
    - No sub-workflow parameters are required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is currently inactive (`active: false`) and must be enabled before scheduled execution will occur. | Workflow status |
| The overview note claims results are deduplicated, but no deduplication node or deduplication logic exists in the actual workflow. | Logic discrepancy |
| The recency-ranking logic expects posting metadata that is not retained by the extraction code, so ranking may not work as intended without modification. | Logic discrepancy |
| The AI note says jobs returning `IGNORE` are dropped, but no node currently removes them after AI processing. | Logic discrepancy |
| Replace placeholder values `YOUR_SERPAPI_KEY` and `YOUR_TELEGRAM_CHAT_ID` before use. | Configuration |
| Use named credentials for OpenAI and Telegram instead of embedding tokens in nodes. | Security |
| Trigger timing is based on server time, not necessarily India Standard Time unless the n8n instance timezone is configured that way. | Scheduling |