Grow Reddit karma with DeepSeek, Google Sheets, Multilogin and Browser MCP

https://n8nworkflows.xyz/workflows/grow-reddit-karma-with-deepseek--google-sheets--multilogin-and-browser-mcp-13967


# Grow Reddit karma with DeepSeek, Google Sheets, Multilogin and Browser MCP

## 1. Workflow Overview

This workflow automates scheduled Reddit activity across multiple accounts managed in Google Sheets and launched through Multilogin. For each eligible account, it decides whether to attempt a new post or a comment, opens a browser session through Browser MCP, lets a DeepSeek-powered agent perform subreddit analysis and engagement, then writes activity results back to Google Sheets and closes the browser profile.

Primary use cases:

- Rotate through multiple Reddit accounts stored in a spreadsheet
- Avoid using shadowbanned or incomplete accounts
- Randomize activity timing and action type
- Launch isolated browser profiles via Multilogin
- Drive browser automation through an AI agent using Browser MCP
- Track posting/commenting metrics and update account state centrally

### 1.1 Scheduled Intake and Account Selection
The workflow starts on a schedule, reads account records from Google Sheets, filters valid records, removes shadowbanned accounts, and calculates which accounts should act in the current run.

### 1.2 Per-Account Delay and Proxy Validation
The selected accounts are processed in batches. Before launching browser automation, the workflow waits, obtains the active proxy exit IP, and checks whether it matches previously assigned IPs.

### 1.3 Browser Session Startup
If the proxy is acceptable, the workflow starts the account’s Multilogin profile, waits for browser initialization, derives the local Chrome debug endpoint, and retrieves the browser WebSocket URL.

### 1.4 AI-Driven Reddit Execution
The workflow prepares a compact input object and routes each item by action type:
- `post` → Reddit Post Agent
- `comment` → Reddit Comment Agent

Each agent uses:
- a DeepSeek chat model
- a Browser MCP tool connected to the running browser session

### 1.5 Result Parsing, Tracking, and Cleanup
The workflow parses the agent output, updates the Google Sheet with counters, timestamps, links, and karma data, obtains the active profile ID, closes the Multilogin profile, and moves to the next account.

---

## 2. Block-by-Block Analysis

## 2.1 Scheduled Intake and Account Selection

**Overview:**  
This block loads Reddit account rows from Google Sheets, keeps only usable accounts, strips the data down to fields needed later, removes shadowbanned rows, and decides which accounts will perform an action during the current run.

**Nodes Involved:**  
- Run workflow on schedule
- Read Reddit Accounts
- Filter Valid Accounts
- Extract Account Fields
- Remove Shadowbanned
- Calculate Actions
- Sort by Action Type
- Loop Through Accounts

### Node Details

#### Run workflow on schedule
- **Type and role:** Schedule Trigger; entry point for the workflow.
- **Configuration choices:** Uses a simple interval-based schedule rule.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Read Reddit Accounts**.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:** Misconfigured interval, inactive workflow, timezone assumptions.
- **Sub-workflow reference:** None.

#### Read Reddit Accounts
- **Type and role:** Google Sheets; reads the spreadsheet containing Reddit account records.
- **Configuration choices:**  
  - Auth via service account  
  - Reads from sheet **Reddit Accounts** in document **Boomerang + LeadAssist | Demand Generation | Reddit DM's**
  - Includes a filter for `shadowban? = No`
- **Key expressions or variables used:** Sheet/document selected via resource locator.
- **Input and output connections:** Input from **Run workflow on schedule**; output to **Filter Valid Accounts**.
- **Version-specific requirements:** Google Sheets node version 4.7.
- **Edge cases / failures:** Missing service account access, renamed columns, wrong sheet ID, API quota issues.
- **Sub-workflow reference:** None.

#### Filter Valid Accounts
- **Type and role:** Filter; keeps accounts that have the required Multilogin profile and expected proxy source.
- **Configuration choices:** Passes items only when:
  - `multilogin_profile_id` is not empty
  - `proxy_provider = GridPanel`
- **Key expressions or variables used:** `{{$json.multilogin_profile_id}}`, `{{$json.proxy_provider}}`
- **Input and output connections:** Input from **Read Reddit Accounts**; output to **Extract Account Fields**.
- **Version-specific requirements:** Filter node version 2.3.
- **Edge cases / failures:** Column name mismatch, blank profile IDs, inconsistent proxy provider labels.
- **Sub-workflow reference:** None.

#### Extract Account Fields
- **Type and role:** Set; normalizes and trims the account row to only downstream-relevant fields.
- **Configuration choices:** Creates fields such as:
  - `row_number`
  - `IP`
  - `UserID`
  - `Password`
  - `multilogin_profile_id`
  - `shadowban`
  - `comments_made_today`
  - `time_of_comment`
  - `posts_made_today`
  - `time_of_post`
  - `last_allocated_ip`
- **Key expressions or variables used:** Direct field mapping from the Google Sheets row, e.g. `{{$json['shadowban?']}}`.
- **Input and output connections:** Input from **Filter Valid Accounts**; output to **Remove Shadowbanned**.
- **Version-specific requirements:** Set node version 3.4.
- **Edge cases / failures:** Bad field names, null timestamps, numbers stored as strings.
- **Sub-workflow reference:** None.

#### Remove Shadowbanned
- **Type and role:** Filter; excludes rows where `shadowban` is not `No`.
- **Configuration choices:** Strict string equals check: `shadowban == No`.
- **Key expressions or variables used:** `{{$json.shadowban}}`
- **Input and output connections:** Input from **Extract Account Fields**; output to **Calculate Actions**.
- **Version-specific requirements:** Filter node version 2.3.
- **Edge cases / failures:** Different casing such as `no`, trailing spaces, duplicate filtering with the upstream sheet filter.
- **Sub-workflow reference:** None.

#### Calculate Actions
- **Type and role:** Code; determines which accounts should act now and whether they should post or comment.
- **Configuration choices:**  
  - Reads all account items
  - If current hour is `>= 12`, returns one fallback item with `action: "none"` and `wait_minutes: 5`
  - Randomly shuffles accounts
  - Picks a random subset of 3–8 accounts
  - Randomly assigns either `post` or `comment`
  - Enforces cooldowns:
    - posts: 60–180 minutes
    - comments: 30–120 minutes
  - If nothing qualifies, also returns `action: "none"`
- **Key expressions or variables used:** JavaScript with `time_of_post`, `time_of_comment`, `posts_made_today`, `comments_made_today`.
- **Input and output connections:** Input from **Remove Shadowbanned**; output to **Sort by Action Type**.
- **Version-specific requirements:** Code node version 2.
- **Edge cases / failures:**  
  - The comment says “ACTIVE HOURS 0 → 12” but logic actually stops activity at hour 12 or later
  - Returning `action: none` can still propagate into the loop even though downstream routing handles only `post` and `comment`
  - Invalid timestamps may produce `Invalid Date`
- **Sub-workflow reference:** None.

#### Sort by Action Type
- **Type and role:** Sort; groups selected items by the `action` field.
- **Configuration choices:** Sorts ascending by `action`.
- **Key expressions or variables used:** Field name `action`.
- **Input and output connections:** Input from **Calculate Actions**; output to **Loop Through Accounts**.
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:** If `action` is absent or contains unexpected values, ordering is undefined.
- **Sub-workflow reference:** None.

#### Loop Through Accounts
- **Type and role:** Split In Batches; iterates over selected accounts one by one.
- **Configuration choices:** Uses default batching behavior.
- **Key expressions or variables used:** Downstream nodes refer back to `$('Loop Through Accounts').item.json...`
- **Input and output connections:**  
  - Input from **Sort by Action Type**
  - Main output continues batch item processing
  - Completion output flows to **Delay Between Accounts**
  - Receives continuation from **Close Multilogin Profile**
- **Version-specific requirements:** Split In Batches version 3.
- **Edge cases / failures:**  
  - If the input contains only `action: none`, downstream action routing may not produce useful work
  - Any missing fields in the current batch item will break later expressions
- **Sub-workflow reference:** None.

---

## 2.2 Per-Account Delay and Proxy Validation

**Overview:**  
This block introduces a pacing delay between accounts, checks the current proxy exit IP through httpbin, compares it with previous assigned IPs, and only proceeds if the IP is considered unique.

**Nodes Involved:**  
- Delay Between Accounts
- Get Proxy Exit IP
- Compare IP Addresses
- Check IP Uniqueness

### Node Details

#### Delay Between Accounts
- **Type and role:** Wait; introduces a delay before starting browser work for the next account.
- **Configuration choices:** Wait duration in minutes from `{{$json.wait_minutes}}`.
- **Key expressions or variables used:** `wait_minutes`
- **Input and output connections:**  
  - Receives from **Loop Through Accounts** completion branch
  - Also receives from **Check IP Uniqueness** false branch
  - Outputs to **Get Proxy Exit IP**
- **Version-specific requirements:** Wait node version 1.1.
- **Edge cases / failures:** Missing `wait_minutes`, resumed execution behavior after n8n restarts, long queues if many items accumulate.
- **Sub-workflow reference:** None.

#### Get Proxy Exit IP
- **Type and role:** HTTP Request; calls httpbin to determine the active public exit IP used by the proxy.
- **Configuration choices:**  
  - GET `https://httpbin.org/get`
  - Explicit proxy configured in node options: `http://YOUR_PROXY_USER:YOUR_PROXY_PASS@YOUR_PROXY_HOST:PORT`
  - `retryOnFail` enabled
  - `onError: continueErrorOutput`
- **Key expressions or variables used:** None dynamic in the request URL, but proxy credentials must be manually set.
- **Input and output connections:** Input from **Delay Between Accounts**; output to **Compare IP Addresses**.
- **Version-specific requirements:** HTTP Request version 4.3.
- **Edge cases / failures:** Bad proxy credentials, dead proxy, TLS/proxy handshake failures, httpbin downtime, malformed response.
- **Sub-workflow reference:** None.

#### Compare IP Addresses
- **Type and role:** Code; checks whether the current proxy exit IP matches any previously assigned IP in the extracted account list.
- **Configuration choices:**  
  - Reads `origin` from httpbin response
  - Reads all items from **Extract Account Fields**
  - Compares `last_allocated_ip` of every account against the current extracted IP
  - Returns a single object with:
    - `httpbin_ip`
    - `wait_minutes`
    - `ip_match`
- **Key expressions or variables used:** `$('Extract Account Fields').all()`, `$('Loop Through Accounts').first().json.wait_minutes`
- **Input and output connections:** Input from **Get Proxy Exit IP**; output to **Check IP Uniqueness**.
- **Version-specific requirements:** Code node version 2.
- **Edge cases / failures:**  
  - If `last_allocated_ip` is null/undefined, `.trim()` can throw
  - Uses `Loop Through Accounts`.first() instead of current item, which can produce the wrong `wait_minutes`
  - Compares against all extracted accounts, not just current account, so uniqueness is global to the run, not account-specific
- **Sub-workflow reference:** None.

#### Check IP Uniqueness
- **Type and role:** If; gates browser startup on the uniqueness result.
- **Configuration choices:** Continues only when `ip_match == false`.
- **Key expressions or variables used:** `{{$json.ip_match}}`
- **Input and output connections:**  
  - Input from **Compare IP Addresses**
  - True branch to **Open Multilogin Profile**
  - False branch to **Delay Between Accounts**
- **Version-specific requirements:** If node version 2.3.
- **Edge cases / failures:** Incorrect boolean typing if upstream data is malformed.
- **Sub-workflow reference:** None.

---

## 2.3 Browser Session Startup

**Overview:**  
This block starts a specific Multilogin profile, waits for the browser to become available, computes the local DevTools endpoint, and retrieves the WebSocket debugger URL needed by Browser MCP.

**Nodes Involved:**  
- Open Multilogin Profile
- Wait for Browser Init
- Build Debug URL
- Get Browser WebSocket URL
- Prepare Agent Input

### Node Details

#### Open Multilogin Profile
- **Type and role:** HTTP Request; launches a Multilogin browser profile configured for Playwright automation.
- **Configuration choices:**  
  - Calls `https://launcher.mlx.yt:45001/api/v2/profile/f/YOUR_FOLDER_ID/p/{multilogin_profile_id}/start`
  - Query parameters:
    - `automation_type=playwright`
    - `headless_mode=false`
  - Uses HTTP Bearer auth via generic credential
  - Accept header `application/json`
  - Allows unauthorized certificates
  - Timeout 30000 ms
  - `onError: continueErrorOutput`
- **Key expressions or variables used:** `{{$('Loop Through Accounts').item.json.multilogin_profile_id}}`
- **Input and output connections:**  
  - Input from **Check IP Uniqueness**
  - Main output to **Wait for Browser Init**
  - Also self-connected, creating a loop back to itself
- **Version-specific requirements:** HTTP Request version 4.3.
- **Edge cases / failures:**  
  - `YOUR_FOLDER_ID` must be replaced manually
  - Bad token or local launcher unavailable
  - Self-loop may cause repeated invocation if error outputs are not handled carefully
  - Port or response schema may differ by Multilogin version
- **Sub-workflow reference:** None.

#### Wait for Browser Init
- **Type and role:** Wait; pauses briefly to let the browser profile finish starting.
- **Configuration choices:** Waits 10 units; because `unit` is not explicitly shown, this should be verified in n8n UI after import.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Open Multilogin Profile**; output to **Build Debug URL**.
- **Version-specific requirements:** Wait node version 1.1.
- **Edge cases / failures:** Too short a delay may cause debug endpoint fetch to fail; imported unit ambiguity should be checked.
- **Sub-workflow reference:** None.

#### Build Debug URL
- **Type and role:** Set; constructs a local Chrome DevTools version endpoint.
- **Configuration choices:** Creates:
  - `localhost = http://127.0.0.1:{data.port}/json/version`
- **Key expressions or variables used:** `{{$json.data.port}}`
- **Input and output connections:** Input from **Wait for Browser Init**; output to **Get Browser WebSocket URL**.
- **Version-specific requirements:** Set version 3.4.
- **Edge cases / failures:** `data.port` missing in Multilogin response, invalid port, profile not fully initialized.
- **Sub-workflow reference:** None.

#### Get Browser WebSocket URL
- **Type and role:** HTTP Request; reads the browser debug endpoint and extracts the WebSocket debugger URL.
- **Configuration choices:** GET request to `{{$json.localhost}}`.
- **Key expressions or variables used:** `localhost`
- **Input and output connections:** Input from **Build Debug URL**; output to **Prepare Agent Input**.
- **Version-specific requirements:** HTTP Request version 4.3.
- **Edge cases / failures:** Endpoint unavailable, browser not ready, malformed JSON, local firewall issues.
- **Sub-workflow reference:** None.

#### Prepare Agent Input
- **Type and role:** Set; builds the compact payload used by the AI agents and browser tools.
- **Configuration choices:** Creates:
  - `url = {{$('Get Browser WebSocket URL').first().json.webSocketDebuggerUrl}}`
  - `target = https://old.reddit.com/r/AskReddit/new/`
  - `action` from current loop item
  - `row_number`, `comments_made_today`, `posts_made_today`, `last_allocated_ip`, `creation_ip`
- **Key expressions or variables used:** References both current loop item and prior node outputs.
- **Input and output connections:** Input from **Get Browser WebSocket URL**; output to **Route Post or Comment**.
- **Version-specific requirements:** Set version 3.4.
- **Edge cases / failures:**  
  - Hard-coded subreddit target; not dynamic per account
  - Missing `webSocketDebuggerUrl`
  - Using `.first()` can be risky if parallelized later
- **Sub-workflow reference:** None.

---

## 2.4 AI-Driven Reddit Execution

**Overview:**  
This block routes each prepared account item by action type and invokes the corresponding autonomous AI agent. Each agent uses a DeepSeek language model plus a Browser MCP tool wired to the active browser session.

**Nodes Involved:**  
- Route Post or Comment
- Reddit Post Agent
- DeepSeek Model (Post)
- Browser MCP Tool (Post)
- Reddit Comment Agent
- DeepSeek Model (Comment)
- Browser MCP Tool (Comment)

### Node Details

#### Route Post or Comment
- **Type and role:** Switch; dispatches items based on `action`.
- **Configuration choices:** Two named outputs:
  - `Post` when `action == post`
  - `Comment` when `action == comment`
- **Key expressions or variables used:** `{{$json.action}}`
- **Input and output connections:** Input from **Prepare Agent Input**; outputs to **Reddit Post Agent** and **Reddit Comment Agent**.
- **Version-specific requirements:** Switch version 3.4.
- **Edge cases / failures:** `action: none` has no matching branch, so those items effectively stop here.
- **Sub-workflow reference:** None.

#### Reddit Post Agent
- **Type and role:** LangChain Agent; autonomous browser-driven Reddit post creation.
- **Configuration choices:**  
  - User text defines target subreddit and “NEW POST ONLY” mode
  - Max iterations: 20
  - Large system prompt instructs the agent to:
    - read subreddit rules
    - analyze recent post titles
    - avoid duplicates
    - identify content gaps
    - create one new text post only
    - never comment
    - return permalink or skip reason
- **Key expressions or variables used:** `{{$json.target}}`
- **Input and output connections:**  
  - Main input from **Route Post or Comment**
  - AI language model input from **DeepSeek Model (Post)**
  - AI tool input from **Browser MCP Tool (Post)**
  - Main output to **Parse Post Result**
- **Version-specific requirements:** Agent node version 3.1.
- **Edge cases / failures:**  
  - LLM may return unstructured text instead of predictable success text
  - Browser tool failures can stop reasoning
  - Reddit UI changes can invalidate browser actions
  - Prompt assumes visible rule sections and old Reddit layout
- **Sub-workflow reference:** None.

#### DeepSeek Model (Post)
- **Type and role:** DeepSeek chat model; language model backend for the post agent.
- **Configuration choices:** Default options only.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connects to **Reddit Post Agent** as `ai_languageModel`.
- **Version-specific requirements:** `@n8n/n8n-nodes-langchain.lmChatDeepSeek` version 1; requires DeepSeek credentials in n8n.
- **Edge cases / failures:** Auth errors, model availability, rate limiting, token limits.
- **Sub-workflow reference:** None.

#### Browser MCP Tool (Post)
- **Type and role:** Browser MCP tool; lets the post agent execute browser actions against the active browser.
- **Configuration choices:**  
  - Base URL: `http://localhost:8931/mcp`
  - Operation: `callTool`
  - `driverUrl = {{$json.url}}`
  - `toolName` and `toolArguments` are provided dynamically by the AI using `$fromAI(...)`
- **Key expressions or variables used:** `$fromAI('Tool_Name', ...)`, `$fromAI('Tool_Arguments', ...)`, `{{$json.url}}`
- **Input and output connections:** Connects to **Reddit Post Agent** as `ai_tool`.
- **Version-specific requirements:** Browser MCP node version 1; Browser MCP server must be running locally.
- **Edge cases / failures:** MCP server unreachable, invalid driver URL, malformed AI-generated arguments, unsupported tool calls.
- **Sub-workflow reference:** None.

#### Reddit Comment Agent
- **Type and role:** LangChain Agent; autonomous subreddit analysis that may skip, comment, or create a new text post.
- **Configuration choices:**  
  - `onError: continueRegularOutput`
  - `retryOnFail: true`
  - Max iterations: 20
  - System prompt instructs agent to:
    - read subreddit rules
    - scan recent posts
    - choose one post
    - decide skip/comment/create post
    - act naturally
    - retrieve user karma afterward
    - return only a JSON array
- **Key expressions or variables used:** `{{$json.target}}`
- **Input and output connections:**  
  - Main input from **Route Post or Comment**
  - AI language model input from **DeepSeek Model (Comment)**
  - AI tool input from **Browser MCP Tool (Comment)**
  - Main output to **Parse Comment Result**
- **Version-specific requirements:** Agent node version 3.1.
- **Edge cases / failures:**  
  - Prompt says it may post or comment, despite the node being labeled as comment agent
  - Prompt contains a JSON example with a missing comma after `"comment_post":"string"`
  - Prompt asks to replace `YOUR_USERNAME_HERE`, but no username variable is provided
  - AI may produce malformed JSON; the downstream parser already assumes this can happen
- **Sub-workflow reference:** None.

#### DeepSeek Model (Comment)
- **Type and role:** DeepSeek chat model; language model backend for the comment agent.
- **Configuration choices:** Default options.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connects to **Reddit Comment Agent** as `ai_languageModel`.
- **Version-specific requirements:** Version 1; requires DeepSeek credentials.
- **Edge cases / failures:** Same as post model node.
- **Sub-workflow reference:** None.

#### Browser MCP Tool (Comment)
- **Type and role:** Browser MCP tool; browser automation interface for the comment agent.
- **Configuration choices:** Same pattern as the post tool:
  - Base URL `http://localhost:8931/mcp`
  - `driverUrl = {{$json.url}}`
  - AI-defined tool name and arguments
- **Key expressions or variables used:** `$fromAI(...)`, `{{$json.url}}`
- **Input and output connections:** Connects to **Reddit Comment Agent** as `ai_tool`.
- **Version-specific requirements:** Browser MCP version 1.
- **Edge cases / failures:** Same as the post tool node.
- **Sub-workflow reference:** None.

---

## 2.5 Result Parsing, Tracking, and Cleanup

**Overview:**  
This block interprets AI outputs, updates spreadsheet fields for either posts or comments, resolves the current profile ID for cleanup, stops the Multilogin profile, and resumes the account loop.

**Nodes Involved:**  
- Parse Post Result
- Update Sheet - Posts
- Parse Comment Result
- Update Sheet - Comments
- Get Profile ID for Cleanup
- Close Multilogin Profile

### Node Details

#### Parse Post Result
- **Type and role:** Code; interprets post-agent output and updates in-memory account state.
- **Configuration choices:**  
  - Reads original account from `Route Post or Comment`
  - Reads `profile_id` from **Open Multilogin Profile**
  - Reads current proxy IP from **Get Proxy Exit IP**
  - Checks whether agent output contains the string `successfully`
  - On success:
    - increments `posts_made_today`
    - writes `time_of_post`
    - sets `last_action_status = post_success`
  - On failure:
    - sets `last_action_status = post_failed`
- **Key expressions or variables used:** `$('Route Post or Comment').all()[0].json`, `$('Open Multilogin Profile').first().json.data.id`, `$input.all()[0].json.output`
- **Input and output connections:** Input from **Reddit Post Agent**; output to **Update Sheet - Posts**.
- **Version-specific requirements:** Code node version 2.
- **Edge cases / failures:**  
  - Success detection depends on the substring `successfully`, which may never appear
  - No permalink extraction despite agent being asked to return one
  - Uses `.all()[0]` and `.first()` assumptions that can become fragile
- **Sub-workflow reference:** None.

#### Update Sheet - Posts
- **Type and role:** Google Sheets update; writes post activity data back to the account row.
- **Configuration choices:** Updates matched row by `row_number` with:
  - `row_number`
  - `time_of_post`
  - `posts_made_today`
  - `last_allocated_ip`
- **Key expressions or variables used:** `{{$json.row_number}}`, `{{$json.time_of_post}}`, `{{$json.posts_made_today}}`, `{{$json.last_allocated_ip}}`
- **Input and output connections:** Input from **Parse Post Result**; output to **Get Profile ID for Cleanup**.
- **Version-specific requirements:** Google Sheets version 4.7; service account auth.
- **Edge cases / failures:** Row number mismatch, protected sheet ranges, schema drift, string-vs-number conversion issues.
- **Sub-workflow reference:** None.

#### Parse Comment Result
- **Type and role:** Code; extracts structured values from potentially malformed agent output using regex.
- **Configuration choices:**  
  - Reads original account from **Route Post or Comment**
  - Reads `origin` from **Get Proxy Exit IP**
  - Reads `profile_id` from **Open Multilogin Profile**
  - Parses from `output` string:
    - `username`
    - `comment_content`
    - `comment_post` → stored as `posts_links`
    - `coments_Permalink` → stored as `comments_links`
    - `post_karma`
    - `comment_karma`
    - `total_karma`
    - `status`
  - Increments `comments_made_today` only when `status == comment_success`
  - Sets `time_of_comment`
- **Key expressions or variables used:** Regex extraction from `$input.first().json.output`.
- **Input and output connections:** Input from **Reddit Comment Agent**; output to **Update Sheet - Comments**.
- **Version-specific requirements:** Code node version 2.
- **Edge cases / failures:**  
  - Very brittle regex parsing
  - Prompt typo `coments_Permalink` is mirrored in parser
  - Escaped quotes/newlines may break matches
  - If output is absent, parser can fail or return partial data
  - This parser assumes string output rather than valid JSON array
- **Sub-workflow reference:** None.

#### Update Sheet - Comments
- **Type and role:** Google Sheets update; writes comment-related tracking information back to the sheet.
- **Configuration choices:** Updates matched row by `row_number` with:
  - `karma`
  - `row_number`
  - `posts_links`
  - `comments_links`
  - `time_of_comment`
  - `last_allocated_ip`
  - `comments_made_today`
- **Key expressions or variables used:** Values from parser output, notably `{{$json.total_karma}}`.
- **Input and output connections:** Input from **Parse Comment Result**; output to **Get Profile ID for Cleanup**.
- **Version-specific requirements:** Google Sheets version 4.7.
- **Edge cases / failures:** Same Google Sheets constraints as post update; parser omissions can write blanks.
- **Sub-workflow reference:** None.

#### Get Profile ID for Cleanup
- **Type and role:** Code; normalizes the cleanup payload regardless of whether the path was post or comment.
- **Configuration choices:**  
  - Tries to return all items from **Parse Post Result**
  - If that fails, returns all items from **Parse Comment Result**
- **Key expressions or variables used:** `$('Parse Post Result').all()`, `$('Parse Comment Result').all()`
- **Input and output connections:** Inputs from both **Update Sheet - Posts** and **Update Sheet - Comments**; output to **Close Multilogin Profile**.
- **Version-specific requirements:** Code node version 2.
- **Edge cases / failures:**  
  - Uses try/catch on node access rather than explicit branching
  - If both upstream states are unavailable, cleanup fails
- **Sub-workflow reference:** None.

#### Close Multilogin Profile
- **Type and role:** HTTP Request; stops the active Multilogin profile.
- **Configuration choices:**  
  - Calls `https://launcher.mlx.yt:45001/api/v1/profile/stop/p/{profile_id}`
  - Uses Bearer auth
  - Accept header `application/json`
  - Allows unauthorized certs
- **Key expressions or variables used:** `{{$json.profile_id}}`
- **Input and output connections:** Input from **Get Profile ID for Cleanup**; output back to **Loop Through Accounts** to continue the batch.
- **Version-specific requirements:** HTTP Request version 4.3.
- **Edge cases / failures:** Invalid profile ID, local launcher unavailable, auth failures, profile already closed.
- **Sub-workflow reference:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run workflow on schedule | n8n-nodes-base.scheduleTrigger | Scheduled entry point |  | Read Reddit Accounts | ## How it works\nThis workflow runs on a schedule, reads Reddit account rows from Google Sheets, and keeps only valid non-shadowbanned records. It then selects a small random batch, checks proxy exit IP uniqueness, and starts each account in Multilogin.\nThis workflow runs on a schedule, reads Reddit account rows from Google Sheets, and keeps only valid non-shadowbanned records. It then selects a small random batch, checks proxy exit IP uniqueness, and starts each account in Multilogin.\nFor each account, the flow opens a Browser MCP session and routes to either the post or comment AI agent based on the calculated action. The result is parsed, activity counters and timestamps are updated in Google Sheets, the profile is closed, and the loop continues with the next account.\n## Setup steps\n1. Connect Google Sheets credentials and confirm the required columns exist in the account sheet.\n2. Configure Multilogin API access, folder ID, and profile IDs.\n3. Set your proxy credentials in `Get Proxy Exit IP`.\n4. Confirm Browser MCP is reachable at `http://localhost:8931/mcp`.\n5. Configure DeepSeek/model credentials and run one manual test.\n6. Set the schedule trigger interval and activate the workflow. |
| Read Reddit Accounts | n8n-nodes-base.googleSheets | Load account rows from sheet | Run workflow on schedule | Filter Valid Accounts | ## How it works\nThis workflow runs on a schedule, reads Reddit account rows from Google Sheets, and keeps only valid non-shadowbanned records. It then selects a small random batch, checks proxy exit IP uniqueness, and starts each account in Multilogin.\nFor each account, the flow opens a Browser MCP session and routes to either the post or comment AI agent based on the calculated action. The result is parsed, activity counters and timestamps are updated in Google Sheets, the profile is closed, and the loop continues with the next account.\n## Setup steps\n1. Connect Google Sheets credentials and confirm the required columns exist in the account sheet.\n2. Configure Multilogin API access, folder ID, and profile IDs.\n3. Set your proxy credentials in `Get Proxy Exit IP`.\n4. Confirm Browser MCP is reachable at `http://localhost:8931/mcp`.\n5. Configure DeepSeek/model credentials and run one manual test.\n6. Set the schedule trigger interval and activate the workflow. |
| Filter Valid Accounts | n8n-nodes-base.filter | Keep accounts with profile ID and expected proxy source | Read Reddit Accounts | Extract Account Fields | ## Account filtering\nReads account rows, extracts the fields used by downstream nodes, and keeps only records with valid profile/proxy data. Shadowbanned or incomplete rows are removed before batching. |
| Extract Account Fields | n8n-nodes-base.set | Normalize account row fields for downstream use | Filter Valid Accounts | Remove Shadowbanned | ## Account filtering\nReads account rows, extracts the fields used by downstream nodes, and keeps only records with valid profile/proxy data. Shadowbanned or incomplete rows are removed before batching. |
| Remove Shadowbanned | n8n-nodes-base.filter | Exclude shadowbanned accounts | Extract Account Fields | Calculate Actions | ## Account filtering\nReads account rows, extracts the fields used by downstream nodes, and keeps only records with valid profile/proxy data. Shadowbanned or incomplete rows are removed before batching. |
| Calculate Actions | n8n-nodes-base.code | Randomly choose accounts and assign post/comment actions with cooldowns | Remove Shadowbanned | Sort by Action Type | ## Action timing\nThis step picks a small random set of eligible accounts (3–8), assigns either post or comment, and enforces cooldowns based on each account’s last activity. If nothing is eligible, it returns `action: none` and retries later. |
| Sort by Action Type | n8n-nodes-base.sort | Sort selected accounts by action | Calculate Actions | Loop Through Accounts |  |
| Loop Through Accounts | n8n-nodes-base.splitInBatches | Iterate over selected accounts | Sort by Action Type, Close Multilogin Profile | Delay Between Accounts |  |
| Delay Between Accounts | n8n-nodes-base.wait | Pause between account attempts | Loop Through Accounts, Check IP Uniqueness | Get Proxy Exit IP |  |
| Get Proxy Exit IP | n8n-nodes-base.httpRequest | Determine active proxy exit IP via httpbin | Delay Between Accounts | Compare IP Addresses | ## Proxy validation\nCalls httpbin to get the active exit IP, compares it with previously assigned IPs, and only continues when the IP is unique for this run. Non-unique IPs are skipped. |
| Compare IP Addresses | n8n-nodes-base.code | Compare current proxy IP against previously assigned IPs | Get Proxy Exit IP | Check IP Uniqueness | ## Proxy validation\nCalls httpbin to get the active exit IP, compares it with previously assigned IPs, and only continues when the IP is unique for this run. Non-unique IPs are skipped. |
| Check IP Uniqueness | n8n-nodes-base.if | Gate browser startup based on unique IP | Compare IP Addresses | Open Multilogin Profile, Delay Between Accounts | ## Proxy validation\nCalls httpbin to get the active exit IP, compares it with previously assigned IPs, and only continues when the IP is unique for this run. Non-unique IPs are skipped. |
| Open Multilogin Profile | n8n-nodes-base.httpRequest | Start browser profile in Multilogin | Check IP Uniqueness | Wait for Browser Init, Open Multilogin Profile | ## Browser session setup\nStarts the Multilogin profile, waits for browser initialization, fetches the debug endpoint, and prepares the WebSocket URL used by Browser MCP tools. |
| Wait for Browser Init | n8n-nodes-base.wait | Allow browser startup time | Open Multilogin Profile | Build Debug URL | ## Browser session setup\nStarts the Multilogin profile, waits for browser initialization, fetches the debug endpoint, and prepares the WebSocket URL used by Browser MCP tools. |
| Build Debug URL | n8n-nodes-base.set | Build local DevTools endpoint URL | Wait for Browser Init | Get Browser WebSocket URL | ## Browser session setup\nStarts the Multilogin profile, waits for browser initialization, fetches the debug endpoint, and prepares the WebSocket URL used by Browser MCP tools. |
| Get Browser WebSocket URL | n8n-nodes-base.httpRequest | Read DevTools metadata including WebSocket debugger URL | Build Debug URL | Prepare Agent Input | ## Browser session setup\nStarts the Multilogin profile, waits for browser initialization, fetches the debug endpoint, and prepares the WebSocket URL used by Browser MCP tools. |
| Prepare Agent Input | n8n-nodes-base.set | Assemble AI/browser session payload | Get Browser WebSocket URL | Route Post or Comment |  |
| Route Post or Comment | n8n-nodes-base.switch | Route item to post or comment path | Prepare Agent Input | Reddit Post Agent, Reddit Comment Agent | ## AI execution\nRoutes each account by action (`post` or `comment`) to the matching agent. Both agents use the same Browser MCP tool and return structured output for parsing. |
| Reddit Post Agent | @n8n/n8n-nodes-langchain.agent | Autonomous post-only Reddit engagement agent | Route Post or Comment, DeepSeek Model (Post), Browser MCP Tool (Post) | Parse Post Result | ## AI execution\nRoutes each account by action (`post` or `comment`) to the matching agent. Both agents use the same Browser MCP tool and return structured output for parsing. |
| DeepSeek Model (Post) | @n8n/n8n-nodes-langchain.lmChatDeepSeek | LLM backend for post agent |  | Reddit Post Agent | ## AI execution\nRoutes each account by action (`post` or `comment`) to the matching agent. Both agents use the same Browser MCP tool and return structured output for parsing. |
| Browser MCP Tool (Post) | n8n-nodes-browser-mcp.browserToolTool | Browser tool bridge for post agent |  | Reddit Post Agent | ## AI execution\nRoutes each account by action (`post` or `comment`) to the matching agent. Both agents use the same Browser MCP tool and return structured output for parsing. |
| Reddit Comment Agent | @n8n/n8n-nodes-langchain.agent | Autonomous comment-oriented Reddit engagement agent | Route Post or Comment, DeepSeek Model (Comment), Browser MCP Tool (Comment) | Parse Comment Result | ## AI execution\nRoutes each account by action (`post` or `comment`) to the matching agent. Both agents use the same Browser MCP tool and return structured output for parsing. |
| DeepSeek Model (Comment) | @n8n/n8n-nodes-langchain.lmChatDeepSeek | LLM backend for comment agent |  | Reddit Comment Agent | ## AI execution\nRoutes each account by action (`post` or `comment`) to the matching agent. Both agents use the same Browser MCP tool and return structured output for parsing. |
| Browser MCP Tool (Comment) | n8n-nodes-browser-mcp.browserToolTool | Browser tool bridge for comment agent |  | Reddit Comment Agent | ## AI execution\nRoutes each account by action (`post` or `comment`) to the matching agent. Both agents use the same Browser MCP tool and return structured output for parsing. |
| Parse Post Result | n8n-nodes-base.code | Interpret post-agent result and update account state | Reddit Post Agent | Update Sheet - Posts | ## Tracking and cleanup\nParses agent output, updates Google Sheets counters/timestamps/links, closes the active Multilogin profile, and continues to the next account in the batch. |
| Update Sheet - Posts | n8n-nodes-base.googleSheets | Persist post counters and timestamp | Parse Post Result | Get Profile ID for Cleanup | ## Tracking and cleanup\nParses agent output, updates Google Sheets counters/timestamps/links, closes the active Multilogin profile, and continues to the next account in the batch. |
| Parse Comment Result | n8n-nodes-base.code | Regex-parse comment agent output and karma details | Reddit Comment Agent | Update Sheet - Comments | ## Tracking and cleanup\nParses agent output, updates Google Sheets counters/timestamps/links, closes the active Multilogin profile, and continues to the next account in the batch. |
| Update Sheet - Comments | n8n-nodes-base.googleSheets | Persist comment counters, links, timestamp, and karma | Parse Comment Result | Get Profile ID for Cleanup | ## Tracking and cleanup\nParses agent output, updates Google Sheets counters/timestamps/links, closes the active Multilogin profile, and continues to the next account in the batch. |
| Get Profile ID for Cleanup | n8n-nodes-base.code | Normalize profile ID payload for cleanup | Update Sheet - Posts, Update Sheet - Comments | Close Multilogin Profile | ## Tracking and cleanup\nParses agent output, updates Google Sheets counters/timestamps/links, closes the active Multilogin profile, and continues to the next account in the batch. |
| Close Multilogin Profile | n8n-nodes-base.httpRequest | Stop active Multilogin browser profile | Get Profile ID for Cleanup | Loop Through Accounts | ## Tracking and cleanup\nParses agent output, updates Google Sheets counters/timestamps/links, closes the active Multilogin profile, and continues to the next account in the batch. |
| Sticky Note Overview | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note Filtering | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note Proxy | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note Multilogin | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note AI Agents | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note Tracking | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note Action timing | n8n-nodes-base.stickyNote | Documentation note |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger** node named **Run workflow on schedule**.
   - Choose an interval schedule appropriate for your runtime frequency.
   - Activate later after testing.

3. **Add a Google Sheets node** named **Read Reddit Accounts**.
   - Operation: read rows
   - Authentication: **Service Account**
   - Connect Google credentials with access to the spreadsheet.
   - Select the spreadsheet:
     - Document: `Boomerang + LeadAssist | Demand Generation | Reddit DM's`
     - Sheet: `Reddit Accounts`
   - Add a filter:
     - Column: `shadowban?`
     - Value: `No`

4. **Connect** `Run workflow on schedule -> Read Reddit Accounts`.

5. **Add a Filter node** named **Filter Valid Accounts**.
   - Keep rows where:
     - `multilogin_profile_id` is not empty
     - `proxy_provider` equals `GridPanel`

6. **Connect** `Read Reddit Accounts -> Filter Valid Accounts`.

7. **Add a Set node** named **Extract Account Fields**.
   - Keep/create these fields:
     - `row_number = {{$json.row_number}}`
     - `IP = {{$json.creation_ip}}`
     - `UserID = {{$json.account_id}}`
     - `Password = {{$json.account_password}}`
     - `multilogin_profile_id = {{$json.multilogin_profile_id}}`
     - `shadowban = {{$json['shadowban?']}}`
     - `comments_made_today = {{$json.comments_made_today}}`
     - `time_of_comment = {{$json.time_of_comment}}`
     - `posts_made_today = {{$json.posts_made_today}}`
     - `time_of_post = {{$json.time_of_post}}`
     - `last_allocated_ip = {{$json.last_allocated_ip}}`

8. **Connect** `Filter Valid Accounts -> Extract Account Fields`.

9. **Add a Filter node** named **Remove Shadowbanned**.
   - Condition:
     - `{{$json.shadowban}}` equals `No`

10. **Connect** `Extract Account Fields -> Remove Shadowbanned`.

11. **Add a Code node** named **Calculate Actions** with the provided action-selection logic.
   - It should:
     - read all account items
     - stop activity after noon by returning `action: none`
     - shuffle accounts
     - choose 3–8 accounts
     - randomly assign `post` or `comment`
     - enforce randomized cooldowns based on `time_of_post` / `time_of_comment`
     - return `wait_minutes: 5`
   - If reproducing exactly, copy the logic behavior, including the `action: none` fallback.

12. **Connect** `Remove Shadowbanned -> Calculate Actions`.

13. **Add a Sort node** named **Sort by Action Type**.
   - Sort field: `action`

14. **Connect** `Calculate Actions -> Sort by Action Type`.

15. **Add a Split In Batches node** named **Loop Through Accounts**.
   - Default settings are acceptable for one-item iteration.

16. **Connect** `Sort by Action Type -> Loop Through Accounts`.

17. **Add a Wait node** named **Delay Between Accounts**.
   - Unit: minutes
   - Amount: `{{$json.wait_minutes}}`

18. **Connect**:
   - `Loop Through Accounts` completion/continue path -> `Delay Between Accounts`

19. **Add an HTTP Request node** named **Get Proxy Exit IP**.
   - Method: GET
   - URL: `https://httpbin.org/get`
   - In request options, set proxy:
     - `http://YOUR_PROXY_USER:YOUR_PROXY_PASS@YOUR_PROXY_HOST:PORT`
   - Enable retry on fail.
   - Optional but recommended: continue on error as in the source workflow.

20. **Connect** `Delay Between Accounts -> Get Proxy Exit IP`.

21. **Add a Code node** named **Compare IP Addresses**.
   - Logic:
     - Read `origin` from current input
     - Read all rows from **Extract Account Fields**
     - Compare `last_allocated_ip` values to the current exit IP
     - Output:
       - `httpbin_ip`
       - `wait_minutes`
       - `ip_match`
   - For a safer rebuild, guard against null `last_allocated_ip` before calling `.trim()`.

22. **Connect** `Get Proxy Exit IP -> Compare IP Addresses`.

23. **Add an If node** named **Check IP Uniqueness**.
   - Condition:
     - `{{$json.ip_match}}` equals `false`

24. **Connect** `Compare IP Addresses -> Check IP Uniqueness`.

25. **Connect the false branch** of **Check IP Uniqueness** back to **Delay Between Accounts**.

26. **Add an HTTP Request node** named **Open Multilogin Profile**.
   - Method: GET
   - URL:
     - `https://launcher.mlx.yt:45001/api/v2/profile/f/YOUR_FOLDER_ID/p/{{$('Loop Through Accounts').item.json.multilogin_profile_id}}/start`
   - Replace `YOUR_FOLDER_ID` with your real Multilogin folder ID.
   - Query params:
     - `automation_type = playwright`
     - `headless_mode = false`
   - Authentication:
     - Generic credential type
     - HTTP Bearer Auth
   - Header:
     - `Accept: application/json`
   - Options:
     - timeout 30000
     - allow unauthorized certificates = true
   - Optional: continue on error, matching the source workflow.

27. **Connect the true branch** of **Check IP Uniqueness** to **Open Multilogin Profile**.

28. **Review the imported self-loop behavior** on **Open Multilogin Profile**.
   - The source workflow has `Open Multilogin Profile -> Open Multilogin Profile`.
   - This is likely unintended or used as a retry artifact.
   - For a clean rebuild, omit this self-loop unless you intentionally implement explicit retry logic.

29. **Add a Wait node** named **Wait for Browser Init**.
   - Set a short fixed wait, such as 10 seconds.
   - Verify the unit explicitly in the UI; the source JSON leaves this ambiguous.

30. **Connect** `Open Multilogin Profile -> Wait for Browser Init`.

31. **Add a Set node** named **Build Debug URL**.
   - Create:
     - `localhost = http://127.0.0.1:{{$json.data.port}}/json/version`

32. **Connect** `Wait for Browser Init -> Build Debug URL`.

33. **Add an HTTP Request node** named **Get Browser WebSocket URL**.
   - Method: GET
   - URL: `{{$json.localhost}}`

34. **Connect** `Build Debug URL -> Get Browser WebSocket URL`.

35. **Add a Set node** named **Prepare Agent Input**.
   - Create:
     - `url = {{$('Get Browser WebSocket URL').first().json.webSocketDebuggerUrl}}`
     - `target = https://old.reddit.com/r/AskReddit/new/`
     - `action = {{$('Loop Through Accounts').item.json.action}}`
     - `row_number = {{$('Loop Through Accounts').item.json.row_number}}`
     - `comments_made_today = {{$('Loop Through Accounts').item.json.comments_made_today}}`
     - `posts_made_today = {{$('Loop Through Accounts').item.json.posts_made_today}}`
     - `last_allocated_ip = {{$('Loop Through Accounts').item.json.last_allocated_ip}}`
     - `creation_ip = {{$('Loop Through Accounts').item.json.creation_ip}}`
   - Note: `target` is hard-coded. If you want flexibility, replace it with a sheet column or variable.

36. **Connect** `Get Browser WebSocket URL -> Prepare Agent Input`.

37. **Add a Switch node** named **Route Post or Comment**.
   - Rule 1:
     - If `{{$json.action}} == post` → output `Post`
   - Rule 2:
     - If `{{$json.action}} == comment` → output `Comment`

38. **Connect** `Prepare Agent Input -> Route Post or Comment`.

39. **Add a DeepSeek chat model node** named **DeepSeek Model (Post)**.
   - Connect valid DeepSeek credentials.
   - Keep default model options unless you need specific model settings.

40. **Add a Browser MCP Tool node** named **Browser MCP Tool (Post)**.
   - Base URL: `http://localhost:8931/mcp`
   - Operation: `callTool`
   - Driver URL: `{{$json.url}}`
   - Tool name: AI-provided via `$fromAI(...)`
   - Tool arguments: AI-provided via `$fromAI(...)`
   - Ensure Browser MCP server is running locally and reachable.

41. **Add a LangChain Agent node** named **Reddit Post Agent**.
   - Prompt type: define
   - Input text:
     - Use the target field and specify post-only mode
   - System message:
     - Add the long autonomous post-only policy shown in the workflow
   - Max iterations: 20

42. **Wire the post branch**:
   - `Route Post or Comment (Post) -> Reddit Post Agent`
   - `DeepSeek Model (Post) -> Reddit Post Agent` as AI language model
   - `Browser MCP Tool (Post) -> Reddit Post Agent` as AI tool

43. **Add a DeepSeek chat model node** named **DeepSeek Model (Comment)**.
   - Connect DeepSeek credentials.

44. **Add a Browser MCP Tool node** named **Browser MCP Tool (Comment)**.
   - Base URL: `http://localhost:8931/mcp`
   - Operation: `callTool`
   - Driver URL: `{{$json.url}}`
   - Tool name and arguments generated via `$fromAI(...)`

45. **Add a LangChain Agent node** named **Reddit Comment Agent**.
   - Prompt type: define
   - Input text:
     - Use the target subreddit
   - System message:
     - Add the long autonomous comment-oriented policy from the workflow
   - Max iterations: 20
   - Enable streaming
   - Enable retry on fail
   - Set error behavior to continue regular output if you want to match the source

46. **Wire the comment branch**:
   - `Route Post or Comment (Comment) -> Reddit Comment Agent`
   - `DeepSeek Model (Comment) -> Reddit Comment Agent` as AI language model
   - `Browser MCP Tool (Comment) -> Reddit Comment Agent` as AI tool

47. **Add a Code node** named **Parse Post Result**.
   - Logic should:
     - read original account data from **Route Post or Comment**
     - read profile ID from **Open Multilogin Profile**
     - read current exit IP from **Get Proxy Exit IP**
     - detect post success
     - increment `posts_made_today`
     - set `time_of_post`
     - set `last_action_status`
   - If rebuilding safely, consider parsing permalink explicitly instead of checking only for `successfully`.

48. **Connect** `Reddit Post Agent -> Parse Post Result`.

49. **Add a Google Sheets node** named **Update Sheet - Posts**.
   - Operation: update row
   - Match column: `row_number`
   - Authentication: same service account
   - Update fields:
     - `row_number`
     - `time_of_post`
     - `posts_made_today`
     - `last_allocated_ip`

50. **Connect** `Parse Post Result -> Update Sheet - Posts`.

51. **Add a Code node** named **Parse Comment Result**.
   - Logic should:
     - read original account
     - read exit IP and profile ID
     - regex-parse fields from agent output:
       - username
       - comment_content
       - comment_post
       - coments_Permalink
       - post_karma
       - comment_karma
       - total_karma
       - status
     - increment `comments_made_today` if `status == comment_success`
     - set `time_of_comment`
   - If rebuilding safely, parse valid JSON instead of regex whenever possible.

52. **Connect** `Reddit Comment Agent -> Parse Comment Result`.

53. **Add a Google Sheets node** named **Update Sheet - Comments**.
   - Operation: update row
   - Match column: `row_number`
   - Update fields:
     - `karma`
     - `row_number`
     - `posts_links`
     - `comments_links`
     - `time_of_comment`
     - `last_allocated_ip`
     - `comments_made_today`

54. **Connect** `Parse Comment Result -> Update Sheet - Comments`.

55. **Add a Code node** named **Get Profile ID for Cleanup**.
   - Logic:
     - if post path exists, return **Parse Post Result**
     - else return **Parse Comment Result**
   - A cleaner rebuild could use separate cleanup branches merged by a Merge node.

56. **Connect**:
   - `Update Sheet - Posts -> Get Profile ID for Cleanup`
   - `Update Sheet - Comments -> Get Profile ID for Cleanup`

57. **Add an HTTP Request node** named **Close Multilogin Profile**.
   - Method: GET
   - URL:
     - `https://launcher.mlx.yt:45001/api/v1/profile/stop/p/{{$json.profile_id}}`
   - Authentication:
     - Generic credential type
     - HTTP Bearer Auth
   - Header:
     - `Accept: application/json`
   - Options:
     - allow unauthorized certificates = true

58. **Connect** `Get Profile ID for Cleanup -> Close Multilogin Profile`.

59. **Connect** `Close Multilogin Profile -> Loop Through Accounts` to continue batch processing.

60. **Create Google credentials**
   - Use a service account in n8n.
   - Share the spreadsheet with the service account email.
   - Verify that the sheet contains all columns referenced by the workflow:
     - `proxy_provider`
     - `creation_ip`
     - `last_allocated_ip`
     - `multilogin_profile_id`
     - `shadowban?`
     - `karma`
     - `posts_made_today`
     - `time_of_post`
     - `comments_made_today`
     - `time_of_comment`
     - `posts_links`
     - `comments_links`
     - `row_number`
     - plus any account identity columns used upstream

61. **Create Multilogin credentials**
   - In n8n, create HTTP Bearer Auth credentials for the Multilogin launcher API.
   - Ensure the launcher is reachable at `https://launcher.mlx.yt:45001`.
   - Replace `YOUR_FOLDER_ID` in the start URL.
   - Ensure each spreadsheet row has a valid `multilogin_profile_id`.

62. **Prepare proxy access**
   - Replace placeholder proxy credentials in **Get Proxy Exit IP**.
   - If the real browsing proxy is managed inside Multilogin rather than in this node, keep in mind the IP check may not validate the same network path.

63. **Prepare Browser MCP**
   - Start Browser MCP locally so `http://localhost:8931/mcp` is reachable.
   - Ensure it supports the browser actions referenced in the prompts:
     - navigate
     - snapshot
     - click
     - type
     - wait
     - evaluate

64. **Prepare DeepSeek credentials**
   - Create DeepSeek credentials in n8n for both model nodes.
   - Confirm the LangChain DeepSeek node is installed in your n8n environment.

65. **Test the flow manually with one account**
   - Temporarily reduce the sheet to one known-safe account
   - Confirm:
     - Multilogin starts
     - debug URL resolves
     - Browser MCP can control the browser
     - agent returns usable output
     - Google Sheets updates correctly
     - profile closes correctly

66. **Validate known weak points before production**
   - Replace `YOUR_FOLDER_ID`
   - Replace proxy placeholders
   - Consider making the target subreddit dynamic
   - Consider fixing malformed JSON expectations in the comment agent
   - Consider removing the self-loop on **Open Multilogin Profile**
   - Consider handling `action: none` explicitly
   - Consider parsing agent output as real JSON rather than regex/string checks

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow runs on a schedule, reads Reddit account rows from Google Sheets, filters valid non-shadowbanned accounts, checks proxy IP uniqueness, launches browser profiles in Multilogin, routes to post/comment AI agents, updates tracking fields in Google Sheets, closes the profile, and loops to the next account. | Workflow overview sticky note |
| Setup steps listed in the workflow: connect Google Sheets credentials, configure Multilogin API and folder/profile IDs, set proxy credentials in `Get Proxy Exit IP`, confirm Browser MCP at `http://localhost:8931/mcp`, configure DeepSeek credentials, then set the schedule interval and activate the workflow. | Workflow overview sticky note / `http://localhost:8931/mcp` |
| Account filtering note: records are reduced to the fields used downstream, and incomplete or shadowbanned rows are removed before batching. | Filtering sticky note |
| Proxy validation note: the workflow calls httpbin, extracts the active exit IP, and skips non-unique IPs. | Proxy sticky note |
| Browser session setup note: Multilogin is started first, then the workflow fetches the debug endpoint and WebSocket URL for Browser MCP. | Multilogin sticky note |
| AI execution note: routing is based on `action` (`post` or `comment`), and both branches use Browser MCP tools and return structured output for parsing. | AI sticky note |
| Tracking and cleanup note: agent results are parsed, counters/timestamps/links are written to Google Sheets, the Multilogin profile is closed, and the loop continues. | Tracking sticky note |