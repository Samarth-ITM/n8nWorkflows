Generate trending X, Facebook, and Threads posts with Apify, Gemini, and Buffer

https://n8nworkflows.xyz/workflows/generate-trending-x--facebook--and-threads-posts-with-apify--gemini--and-buffer-14961


# Generate trending X, Facebook, and Threads posts with Apify, Gemini, and Buffer

# Workflow Analysis: Auto-Generate Trending X, Facebook & Thread Posts

## 1. Workflow Overview
This workflow automates the process of identifying trending topics in Nigeria and publishing AI-generated reactions to them across multiple social media platforms. It is designed to run on a schedule, ensuring content remains fresh and relevant while avoiding repetition of recently discussed topics.

The logic is organized into three primary functional blocks:
- **1.1 Trend Fetching:** A multi-layered acquisition strategy that uses Apify to scrape X (Twitter) trends with a fallback to Google Gemini AI search if scraping fails.
- **1.2 Deduplication & Selection:** A filtering system using an internal n8n Data Table to ensure that any trend used within the last 24 hours is excluded before picking the top candidates for posting.
- **1.3 Content Generation & Distribution:** An AI loop that transforms a trend into a casual, pidgin-English post and distributes it via Buffer MCP to X, Facebook, and Threads.

---

## 2. Block-by-Block Analysis

### 2.1 Trend Fetching
**Overview:** This block gathers a list of current trending phrases specifically for the Nigerian market. It employs a "fail-safe" mechanism where Apify is the primary source, and Gemini is the secondary.

- **Nodes Involved:** `Schedule Trigger1`, `Run an Actor and get dataset2`, `Extract Trends3`, `If3`, `Run an Actor and get dataset1`, `Extract Trends5`, `If4`, `Get Daily x trends1`, `Gemini Success1`, `Generate Tweet (HTTP Fallback)1`, `Extract Trends4`, `Split Trend Apify`, `Split Trend`, `Split Trend1`.
- **Node Details:**
    - **Schedule Trigger1:** Triggers the workflow at 08:00, 12:00, and 16:00 daily via cron expression.
    - **Apify Nodes (`Run an Actor...`):** Uses the "Twitter Trending Topics Scraper" actor. Configured for `country: nigeria`.
    - **Extract Trends (3, 4, 5):** JavaScript nodes that parse raw Apify arrays or Gemini text strings into a standardized list of trends.
    - **If Nodes (3, 4):** Validate if the previous node successfully returned a "success" flag.
    - **Get Daily x trends1 (Gemini):** Uses `gemini-2.5-flash` with Google Search enabled to manually find trends if Apify fails.
    - **Generate Tweet (HTTP Fallback)1:** A REST API call to Gemini for trend retrieval when the LangChain node fails.
    - **Split Trend Nodes:** Normalize the various data sources into a consistent format (one item per trend) for the merge node (`Split Trend1`).
- **Edge Cases:** If both Apify and Gemini fail to return valid trends, the workflow will stop at the merge point as no data will be passed forward.

### 2.2 Deduplication & Selection
**Overview:** Prevents the AI from posting about the same trend repeatedly by checking a history table and filtering based on a 24-hour window.

- **Nodes Involved:** `Get Used Trends`, `Code in JavaScript`, `Filter Last 24h`, `Remove Used Trends`, `Pick Trend`.
- **Node Details:**
    - **Get Used Trends:** Retrieves all historical records from the `trend_table` Data Table.
    - **Filter Last 24h:** A JS node that calculates the time difference between "now" and the `date` stored in the table, returning only trends used in the last 24 hours.
    - **Remove Used Trends:** Compares current trends against the "recent" list and removes matches.
    - **Pick Trend:** Slices the resulting array to select the number of trends to process (default is set to 1).
- **Failure Types:** Expression failures in `Filter Last 24h` if the `date` column in the Data Table is empty or improperly formatted.

### 2.3 Content Generation & Distribution
**Overview:** Transforms the selected trend into a creative post and publishes it to multiple social channels.

- **Nodes Involved:** `Loop Over Items`, `trend`, `Gemini Success`, `Generate Tweet (HTTP Fallback)`, `Extract Tweet Text`, `Post only tweets`, `facebook post`, `Thread post`, `Code in JavaScript1`, `Insert row`.
- **Node Details:**
    - **Loop Over Items:** Iterates through each picked trend.
    - **trend (Gemini):** Generates a post based on the trend. Prompt specifies: *Pidgin + casual tone, live events context, max 275 characters*.
    - **Extract Tweet Text:** A robust JS node that handles different Gemini response structures (Direct text, Tool calls, or Candidate arrays).
    - **Buffer MCP Nodes:** Sends the final text to three different Channel IDs (X, Facebook, Threads) using the `create_post` tool.
    - **Insert row:** Logs the trend and the current timestamp back into `trend_table` to mark it as used.
- **Edge Cases:** Gemini might produce text exceeding 275 characters despite instructions; the Buffer API may reject posts if the `channelId` is incorrect.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger1 | Schedule Trigger | Entry point (Timed) | - | Run Actor 2 | |
| Run an Actor... (2) | Apify | Scrape X Trends | Schedule Trigger1 | Extract Trends3 | Fetch Nigeria Trends |
| Extract Trends3 | Code | Parse Apify Data | Run Actor 2 | If3 | |
| If3 | If | Validate Apify 2 | Extract Trends3 | Split Trend Apify / Run Actor 1 | |
| Run an Actor... (1) | Apify | Scrape X Trends (Backup) | If3 | Extract Trends5 | |
| Extract Trends5 | Code | Parse Apify Data | Run Actor 1 | If4 | |
| If4 | If | Validate Apify 1 | Extract Trends5 | Split Trend Apify / Get Daily Trends | |
| Get Daily x trends1 | Google Gemini | AI Search Trends | If4 | Gemini Success1 | |
| Gemini Success1 | If | Validate AI Trends | Get Daily Trends | Extract Trends4 / HTTP Fallback 1 | |
| Generate Tweet (HTTP Fallback)1 | HTTP Request | API Search Trends | Gemini Success1 | Extract Trends4 | |
| Extract Trends4 | Code | Parse AI Text | Gemini Success1 / HTTP Fallback 1 | Split Trend | |
| Split Trend Apify | Code | Normalize Apify Data | If3 / If4 | Split Trend1 | |
| Split Trend | Code | Normalize AI Data | Extract Trends4 | Split Trend1 | |
| Split Trend1 | Merge | Consolidate Trends | Split Trend / Split Trend Apify | Create Data Table / Get Used Trends | |
| Get Used Trends | Data Table | Read History | Split Trend1 | Code in JavaScript | Deduplication |
| Code in JavaScript | Code | Debugging/Logging | Get Used Trends | Filter Last 24h | |
| Filter Last 24h | Code | 24h Time Filter | Code in JS | Remove Used Trends | |
| Remove Used Trends | Code | Deduplicate List | Filter Last 24h | Pick Trend | |
| Pick Trend | Code | Limit Trend Count | Remove Used Trends | Loop Over Items | Setup number of tweets per cycle |
| Loop Over Items | Split in Batches | Iteration | Pick Trend | trend / Replace Me | |
| trend | Google Gemini | Content Creation | Loop Over Items | Gemini Success | Gemini Credential Required |
| Gemini Success | If | Validate Post Text | trend | Extract Tweet Text / HTTP Fallback | |
| Generate Tweet (HTTP Fallback) | HTTP Request | API Post Creation | Gemini Success | Extract Tweet Text | |
| Extract Tweet Text | Code | Clean AI Output | Gemini Success / HTTP Fallback | Buffer Nodes | |
| Post only tweets | MCP Client (Buffer) | Publish to X | Extract Tweet Text | Code in JS1 | Buffer Channel ID Setup |
| facebook post | MCP Client (Buffer) | Publish to FB | Extract Tweet Text | Code in JS1 | Buffer Channel ID Setup |
| Thread post | MCP Client (Buffer) | Publish to Threads | Extract Tweet Text | Code in JS1 | Buffer Channel ID Setup |
| Code in JavaScript1 | Code | Prepare Log Data | Buffer Nodes | Insert row | |
| Insert row | Data Table | Update History | Code in JS1 | - | |
| Create a data table | Data Table | Initial Setup | Split Trend1 | Get Used Trends | Run manually once, then disable |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Infrastructure Setup
1. **Data Table:** Create a Data Table named `trend_table` with two columns: `trend` (String) and `date` (String).
2. **Credentials:**
    - **Apify:** API Token.
    - **Google Gemini:** Google PaLM API key.
    - **Buffer:** Bearer Token for the Buffer API.

### Step 2: Trend Acquisition Logic
1. Create a **Schedule Trigger** with cron `0 8,12,16 * * *`.
2. Add an **Apify Node** (Actor: `easyapi/twitter-trending-topics-scraper`) configured for `country: nigeria`.
3. Add a **Code Node** (`Extract Trends3`) to map `record.title` into a JSON array.
4. Add an **If Node** to check if `success` is true.
5. Branch the "False" output to a second **Apify Node** or a **Gemini Node** (`gemini-2.5-flash`) with a prompt to search for Nigerian trends via Google Search.
6. Use a **Merge Node** (`Split Trend1`) to collect all parsed trends from either the Apify or Gemini paths.

### Step 3: Deduplication Logic
1. Add a **Data Table Node** (`Get Used Trends`) to fetch all rows from `trend_table`.
2. Create a **Code Node** (`Filter Last 24h`) to compare `new Date()` with the `date` column and return items where the difference is $< 86,400,000$ ms.
3. Create a **Code Node** (`Remove Used Trends`) to filter out these recent trends from the current list.
4. Create a **Code Node** (`Pick Trend`) using `items.slice(0, X)` where X is the number of posts desired per cycle.

### Step 4: AI Generation and Distribution
1. Add a **Split in Batches Node** (`Loop Over Items`) to process trends one by one.
2. Add a **Gemini Node** with the specific prompt for pidgin-English reactions.
3. Add an **If Node** to validate that the AI returned a response; if not, use an **HTTP Request Node** as a fallback to the Gemini API.
4. Add a **Code Node** (`Extract Tweet Text`) to isolate the text from the Gemini response object.
5. Create three **MCP Client Nodes** (Buffer) using the `create_post` tool. Set unique `channelId`s for X, Facebook, and Threads.
6. Finally, add a **Code Node** to format the trend and current ISO date, followed by a **Data Table Node** (`Insert row`) to save the used trend to `trend_table`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Gemini 2.5 Flash is used for high-speed, low-cost content generation with search capabilities. | AI Model Selection |
| The `trend_table` must be initialized manually once via the `Create a data table` node. | Data Table Setup |
| The `Pick Trend` node is the primary control for volume; adjust `.slice(0, 1)` to change the number of posts. | Volume Control |
| Buffer MCP allows multi-channel distribution through a single API integration. | Distribution Strategy |