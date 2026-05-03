Analyze weekly Fireflies sales calls with GPT-4o-mini, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/analyze-weekly-fireflies-sales-calls-with-gpt-4o-mini--google-sheets-and-slack-15013


# Analyze weekly Fireflies sales calls with GPT-4o-mini, Google Sheets and Slack

# Technical Analysis: Weekly Sales Call Intelligence Workflow

## 1. Workflow Overview
This workflow is designed for sales managers and revenue teams to automate the analysis of sales call transcripts. Every Monday morning, it retrieves all transcripts from the past seven days via the Fireflies.ai API, processes them using GPT-4o-mini to extract critical sales intelligence, logs the detailed results into a Google Sheet, and sends a high-level executive summary to a Slack channel.

### Logical Blocks
- **1.1 Schedule and Configuration:** Triggers the weekly run and defines global variables (API keys, IDs, and keywords).
- **1.2 Transcript Retrieval & Filtering:** Fetches recent calls and filters for those occurring within the last 7 days.
- **1.3 Full Data Extraction & AI Analysis:** For every qualified call, it retrieves the full transcript and uses an AI Agent to extract pricing, objections, signals, and competitors.
- **1.4 Results Assembly & Distribution:** Merges AI insights with metadata to populate a Google Sheet and aggregates all calls into a single Slack report.

---

## 2. Block-by-Block Analysis

### 2.1 Schedule and Configuration
**Overview:** Initializes the workflow and centralizes all environment variables to avoid hard-coding values across multiple nodes.
- **Nodes Involved:** `1. Schedule — Every Monday 9AM`, `2. Set — Config Values`.
- **Node Details:**
    - **1. Schedule — Every Monday 9AM** (Schedule Trigger): Uses a cron expression `0 9 * * 1` to trigger every Monday at 09:00.
    - **2. Set — Config Values** (Set): Defines key-value pairs:
        - `firefliesApiKey`: Authentication token for Fireflies.
        - `sheetId` / `sheetName`: Destination Google Sheet details.
        - `slackChannel`: Target channel for the report.
        - `salesKeywords`: Comma-separated list used for filtering relevant calls.
        - `weekStart` / `weekEnd` / `sevenDaysAgoMs`: Dynamic dates calculated via expressions to define the 7-day window.

### 2.2 Transcript Retrieval & Filtering
**Overview:** Connects to Fireflies.ai to identify which calls need to be analyzed.
- **Nodes Involved:** `3. HTTP — Fetch Recent Transcripts`, `4. Code — Filter Last 7 Days`, `5. IF — Any Transcripts Found?`, `6. Set — No Calls This Week`.
- **Node Details:**
    - **3. HTTP — Fetch Recent Transcripts** (HTTP Request): Performs a GraphQL POST request to `/graphql` to get the 50 most recent transcripts. Uses Bearer authentication.
    - **4. Code — Filter Last 7 Days** (Code): 
        - **Logic:** Filters the array of transcripts based on the `sevenDaysAgoMs` timestamp.
        - **Output:** Returns one item per qualifying transcript, effectively creating a loop for the subsequent nodes.
    - **5. IF — Any Transcripts Found?** (IF): Checks the `hasTranscripts` boolean.
        - *True:* Proceeds to detailed analysis.
        - *False:* Diverts to the "No Calls" set node to end the workflow cleanly.
    - **6. Set — No Calls This Week** (Set): Provides a final status message if no data was found.

### 2.3 Full Data Extraction & AI Analysis
**Overview:** Deep-dives into each specific call to extract raw text and structured intelligence.
- **Nodes Involved:** `7. HTTP — Fetch Full Transcript`, `8. Code — Extract Transcript Data`, `9. AI Agent — Analyze Sales Call`, `10. OpenAI — GPT-4o-mini Model`, `11. Parser — Structured Call Analysis`.
- **Node Details:**
    - **7. HTTP — Fetch Full Transcript** (HTTP Request): GraphQL query requesting specific details: `sentences`, `summary`, `analytics` (sentiments), and `transcript_url` using the `transcriptId`.
    - **8. Code — Extract Transcript Data** (Code): 
        - Cleans and truncates the transcript text (max 4000 chars) to fit LLM context windows.
        - Aggregates sentiment percentages into a human-readable "Overall Sentiment" label.
        - Extracts specific sentences flagged by Fireflies as "pricing", "questions", or "tasks".
    - **9. AI Agent — Analyze Sales Call** (AI Agent): 
        - **Role:** Sales Intelligence Analyst.
        - **Prompt:** Directs the AI to analyze the provided excerpt and Fireflies-detected markers.
        - **Requirements:** Specifically asks for 6 fields: pricing, objections, buying signals, competitors, key takeaway, and recommended action.
    - **10. OpenAI — GPT-4o-mini Model** (Chat Model): Configured with `temperature: 0.3` for consistency and `gpt-4o-mini` for cost-efficiency.
    - **11. Parser — Structured Call Analysis** (Structured Output Parser): Enforces a strict JSON schema for the AI output, ensuring the workflow doesn't break during the data merge phase.

### 2.4 Results Assembly & Output
**Overview:** Consolidates the AI's findings and distributes them to the team.
- **Nodes Involved:** `12. Code — Combine Analysis Results`, `13. Google Sheets — Log Call Analysis`, `14. Code — Build Weekly Slack Summary`, `15. Slack — Send Weekly Report`.
- **Node Details:**
    - **12. Code — Combine Analysis Results** (Code): Maps the AI's JSON output and the transcript metadata into a single flat object suitable for a spreadsheet row.
    - **13. Google Sheets — Log Call Analysis** (Google Sheets): Appends the result to the specified sheet. Maps 13 columns including "Call Date", "Objections", and "Recommended Action".
    - **14. Code — Build Weekly Slack Summary** (Code): 
        - **Logic:** This node aggregates all items from the previous analysis step.
        - **Formatting:** Iterates through all calls to count total volume, identify top objections, and list unique competitors.
        - **Output:** A formatted Markdown string for Slack.
    - **15. Slack — Send Weekly Report** (Slack): Sends the aggregated message to the configured channel using OAuth2.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Schedule — Every Monday 9AM | Schedule Trigger | Workflow Entry | - | 2. Set — Config Values | Schedule and Config |
| 2. Set — Config Values | Set | Env Var Definition | 1. Schedule | 3. HTTP — Fetch... | Schedule and Config |
| 3. HTTP — Fetch Recent Transcripts | HTTP Request | Initial Data Fetch | 2. Set — Config Values | 4. Code — Filter... | Transcript Fetch... |
| 4. Code — Filter Last 7 Days | Code | Date Filtering | 3. HTTP — Fetch... | 5. IF — Any... | Transcript Fetch... |
| 5. IF — Any Transcripts Found? | IF | Execution Gate | 4. Code — Filter... | 7. HTTP — Fetch... / 6. Set | Transcript Fetch... |
| 6. Set — No Calls This Week | Set | Workflow Termination | 5. IF — Any... | - | Transcript Fetch... |
| 7. HTTP — Fetch Full Transcript | HTTP Request | Detailed Data Fetch | 5. IF — Any... | 8. Code — Extract... | Full Transcript Fetch... |
| 8. Code — Extract Transcript Data | Code | Text Pre-processing | 7. HTTP — Fetch... | 9. AI Agent... | Full Transcript Fetch... |
| 9. AI Agent — Analyze Sales Call | AI Agent | Intelligence Extraction | 8. Code — Extract... | 12. Code — Combine... | Full Transcript Fetch... |
| 10. OpenAI — GPT-4o-mini Model | Chat Model | LLM Engine | - | 9. AI Agent... | Full Transcript Fetch... |
| 11. Parser — Structured Call Analysis | Output Parser | Schema Enforcement | - | 9. AI Agent... | Full Transcript Fetch... |
| 12. Code — Combine Analysis Results | Code | Data Merging | 9. AI Agent... | 13. Google Sheets / 14. Code | Results Assembly... |
| 13. Google Sheets — Log Call Analysis | Google Sheets | Permanent Logging | 12. Code — Combine... | - | Results Assembly... |
| 14. Code — Build Weekly Slack Summary | Code | Report Aggregation | 12. Code — Combine... | 15. Slack — Send... | Results Assembly... |
| 15. Slack — Send Weekly Report | Slack | Notification Delivery | 14. Code — Build... | - | Results Assembly... |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Trigger and Global Config
1. Create a **Schedule Trigger** node. Set the interval to "Cron Expression" and use `0 9 * * 1`.
2. Create a **Set** node. Add the following string variables: `firefliesApiKey`, `sheetId`, `sheetName`, `slackChannel`, `companyName`, and `salesKeywords`. 
3. Add three dynamic variables for dates using the `$now` expression: `weekStart`, `weekEnd`, and `sevenDaysAgoMs` (converted to Milliseconds).

### Step 2: Fetching and Filtering
1. Create an **HTTP Request** node. Set method to `POST`, URL to `https://api.fireflies.ai/graphql`. In the body, use the GraphQL query `query GetRecentTranscripts { transcripts(limit: 50) { id title date duration participants } }`. Add a header `Authorization: Bearer {{ firefliesApiKey }}`.
2. Create a **Code** node. Write JavaScript to filter the `transcripts` array where `date >= sevenDaysAgoMs`. The node must return one item per transcript to ensure the workflow loops.
3. Create an **IF** node. Set the condition to check if `hasTranscripts` is `true`.
4. For the `false` path, connect a **Set** node with a "No calls found" message.

### Step 3: Analysis Loop
1. Create an **HTTP Request** node. Use a GraphQL query to fetch full details (`sentences`, `summary`, `analytics`) for the specific `transcriptId`.
2. Create a **Code** node to:
    - Combine `speaker_name` and `text` from the `sentences` array.
    - Truncate the result to 4000 characters.
    - Calculate the "Overall Sentiment" based on positive/negative percentages.
3. Create an **AI Agent** node. 
    - Connect an **OpenAI Chat Model** node (select `gpt-4o-mini`, temperature `0.3`).
    - Connect a **Structured Output Parser** node. Define the JSON schema with the 6 required fields (pricingMentions, objections, buyingSignals, competitorMentions, keyTakeaway, recommendedAction).
    - Set the system prompt to act as a "Sales Intelligence Analyst" and provide the truncated transcript and metadata as input.

### Step 4: Final Distribution
1. Create a **Code** node to merge the AI output with metadata (Date, Title, URL) from the previous extraction step.
2. Create a **Google Sheets** node. Set operation to `Append`. Map the merged data to the corresponding columns of your "Sales Call Analysis" sheet.
3. Create a **Code** node to aggregate all analyzed items. Loop through all items to create a summary of total calls, top 3 objections, and top 3 signals.
4. Create a **Slack** node. Use the "Post Message" operation and pass the aggregated Markdown text to the `slackChannel` variable.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Required Sheet Columns:** Call Date, Meeting Title, Duration (min), Participants, Overall Sentiment, Pricing Mentions, Objections, Buying Signals, Competitor Mentions, Key Takeaway, Recommended Action, Transcript URL, Logged At | Google Sheets Setup |
| **API Access:** Fireflies API key is found under Settings $\rightarrow$ Integrations. | Fireflies.ai |
| **Credential Needs:** OpenAI API Key, Google OAuth2, Slack OAuth2 | Authentication |