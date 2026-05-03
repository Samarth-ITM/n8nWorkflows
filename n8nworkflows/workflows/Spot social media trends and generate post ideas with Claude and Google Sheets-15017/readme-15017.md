Spot social media trends and generate post ideas with Claude and Google Sheets

https://n8nworkflows.xyz/workflows/spot-social-media-trends-and-generate-post-ideas-with-claude-and-google-sheets-15017


# Spot social media trends and generate post ideas with Claude and Google Sheets

# Workflow Documentation: Social Media Trend Spotter & Idea Generator

## 1. Workflow Overview
This AI-powered workflow is designed to automate the discovery of trending topics across major social platforms and transform them into actionable content strategies. It monitors Twitter, Reddit, and Google Trends, filters these topics based on a specific brand niche, and uses Claude AI to generate platform-optimized post ideas. Finally, it calculates optimal posting times based on historical engagement patterns and logs everything into Google Sheets for content calendar management.

### Logical Blocks
- **1.1 Trigger & Configuration:** Handles the incoming request and establishes the brand profile, target audience, and operational constraints.
- **1.2 Trend Acquisition & Aggregation:** Simultaneously fetches real-time trending data from three distinct external APIs and merges them.
- **1.3 Analysis & AI Generation:** Filters trends for niche relevance and invokes Claude AI to create a detailed content matrix (captions, visuals, and hooks).
- **1.4 Timing Optimization & Output:** Calculates the best dates/times to post based on platform-specific patterns, archives the data in Google Sheets, and returns a final JSON response.

---

## 2. Block-by-Block Analysis

### Block 1: Trigger & Configuration
**Overview:** Initializes the process by receiving a request and setting up the global variables and brand identity used by subsequent nodes.

- **Nodes Involved:** `Receive Trend Request`, `Validate Config & Build Parameters`.
- **Node Details:**
    - **Receive Trend Request (Webhook):** 
        - *Role:* Entry point for the workflow.
        - *Configuration:* HTTP POST method at path `/social-trends`.
        - *Output:* Sends the request body to the configuration node.
    - **Validate Config & Build Parameters (Code):** 
        - *Role:* Normalizes input and defines constants.
        - *Logic:* Sets defaults for `platforms`, `niche`, `brandVoice`, and `targetAudience`. It also defines `engagementPatterns` (peak hours/days) for Twitter, Instagram, and LinkedIn.
        - *Edge Cases:* If the webhook payload is empty, it falls back to default "Technology & Innovation" settings.

### Block 2: Fetch Trends & Wait for Aggregation
**Overview:** Gathers raw trend data from multiple sources and ensures all asynchronous requests are completed before processing.

- **Nodes Involved:** `Fetch Twitter Trending Topics`, `Fetch Reddit Hot Topics`, `Fetch Google Trends`, `Merge All Trend Sources`, `Wait for Trend Aggregation`.
- **Node Details:**
    - **Fetch Twitter Trending Topics (HTTP Request):** 
        - *Role:* Retrieves trending topics via Twitter API v2.
        - *Auth:* Twitter OAuth2.
        - *Failure Handling:* Set to "Continue on Fail" to prevent the whole workflow from crashing if one API is down.
    - **Fetch Reddit Hot Topics (HTTP Request):** 
        - *Role:* Pulls "hot" posts from `/r/all`.
        - *Auth:* Reddit OAuth2.
    - **Fetch Google Trends (HTTP Request):** 
        - *Role:* Fetches daily search trends.
        - *Auth:* None (Public API).
    - **Merge All Trend Sources (Merge):** 
        - *Role:* Combines the arrays from the three HTTP requests into a single data stream.
    - **Wait for Trend Aggregation (Wait):** 
        - *Role:* Acts as a buffer to ensure data stability before the heavy parsing logic starts.

### Block 3: Parse, Filter & AI Generation
**Overview:** The "intelligence" core. It cleans raw API data, scores it based on niche relevance, and generates creative content using LLMs.

- **Nodes Involved:** `Parse & Filter Trending Topics`, `Generate Content Ideas with Claude AI`, `Claude AI Model`, `Parse Claude Response`.
- **Node Details:**
    - **Parse & Filter Trending Topics (Code):** 
        - *Role:* Data normalization and scoring.
        - *Logic:* Maps different API responses to a common format (`topic`, `volume`, `source`). It calculates a `finalScore` by combining volume and a "niche match" bonus.
        - *Filter:* Removes prohibited topics and excludes trends below the `minTrendScore`.
    - **Generate Content Ideas with Claude AI (AI Agent):** 
        - *Role:* Content creation.
        - *Configuration:* Uses a sophisticated prompt instructing the AI to act as a "creative social media strategist." It requests a specific JSON output including angles, captions, hashtags, and visual concepts.
    - **Claude AI Model (LLM Chain):** 
        - *Role:* The underlying model.
        - *Configuration:* Claude Sonnet (v 20250514), Temperature 0.7.
    - **Parse Claude Response (Code):** 
        - *Role:* Sanitizes the AI output.
        - *Logic:* Removes markdown code blocks (```json) and parses the string into a usable JSON object.

### Block 4: Timing Analysis & Output
**Overview:** Optimizes the delivery of the generated content and records it for the user.

- **Nodes Involved:** `Wait Before Timing Analysis`, `Calculate Optimal Posting Times`, `Log Ideas to Google Sheets`, `Format Final Response`, `Send Response to Webhook`.
- **Node Details:**
    - **Calculate Optimal Posting Times (Code):** 
        - *Role:* Scheduling logic.
        - *Logic:* Compares the current date/time against the `engagementPatterns` defined in Block 1. It identifies the next available peak window within 7 days and calculates an "Expected Engagement Rate" based on trend score and timing sensitivity.
    - **Log Ideas to Google Sheets (Google Sheets):** 
        - *Role:* Permanent storage/Content Calendar.
        - *Operation:* Appends rows to a specified spreadsheet.
    - **Format Final Response (Code):** 
        - *Role:* Data aggregation.
        - *Logic:* Summarizes the total ideas generated across all trends into a single cohesive report.
    - **Send Response to Webhook (Respond to Webhook):** 
        - *Role:* Final delivery.
        - *Output:* Sends the final JSON report back to the original requester.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Trend Request | Webhook | Entry Point | - | Validate Config... | 1. Trigger & Configuration |
| Validate Config... | Code | Param Initialization | Receive Trend Request | Fetch Twitter, Reddit, Google | 1. Trigger & Configuration |
| Fetch Twitter Trending | HTTP Request | Data Acquisition | Validate Config... | Merge All... | 2. Fetch Trends & Wait... |
| Fetch Reddit Hot Topics| HTTP Request | Data Acquisition | Validate Config... | Merge All... | 2. Fetch Trends & Wait... |
| Fetch Google Trends | HTTP Request | Data Acquisition | Validate Config... | Merge All... | 2. Fetch Trends & Wait... |
| Merge All Trend Sources | Merge | Data Integration | Twitter, Reddit, Google | Wait for Trend... | 2. Fetch Trends & Wait... |
| Wait for Trend... | Wait | Sync Buffer | Merge All... | Parse & Filter... | 2. Fetch Trends & Wait... |
| Parse & Filter... | Code | Scoring & Cleaning | Wait for Trend... | Gen. Content Ideas... | 3. Parse, Filter & AI Gen. |
| Gen. Content Ideas... | AI Agent | Creative Generation | Parse & Filter... | Parse Claude Resp. | 3. Parse, Filter & AI Gen. |
| Claude AI Model | LLM Model | Intelligence Core | - | Gen. Content Ideas... | 3. Parse, Filter & AI Gen. |
| Parse Claude Response | Code | JSON Sanitization | Gen. Content Ideas... | Wait Before Timing... | 3. Parse, Filter & AI Gen. |
| Wait Before Timing... | Wait | Sync Buffer | Parse Claude Resp. | Calculate Optimal... | 4. Timing Analysis & Output |
| Calculate Optimal... | Code | Scheduling Logic | Wait Before Timing... | Log Ideas to GSheets | 4. Timing Analysis & Output |
| Log Ideas to GSheets | Google Sheets | Archiving | Calculate Optimal... | Format Final Resp. | 4. Timing Analysis & Output |
| Format Final Response | Code | Data Aggregation | Log Ideas... | Send Resp. to Webhook | 4. Timing Analysis & Output |
| Send Resp. to Webhook | Respond to Webhook | Final Delivery | Format Final Resp. | - | 4. Timing Analysis & Output |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Setup Trigger and Config
1. Create a **Webhook** node: Path `social-trends`, Method `POST`, Response Mode `Response Node`.
2. Create a **Code** node (`Validate Config & Build Parameters`): Paste the JS logic to define `config`, `brandProfile`, and `engagementPatterns`.

### Step 2: Build the Data Acquisition Layer
1. Create three **HTTP Request** nodes:
    - **Twitter:** URL `https://api.twitter.com/2/trends/place`, Query Param `id=1`, Twitter OAuth2 credentials.
    - **Reddit:** URL `https://oauth.reddit.com/r/all/hot`, Query Params `limit=25, t=day`, Reddit OAuth2 credentials.
    - **Google:** URL `https://trends.google.com/trends/api/dailytrends`, Query Params `hl=en-US, geo=IN`.
2. Connect all three to a **Merge** node (Mode: Combine).
3. Connect the Merge node to a **Wait** node (Resume: After Execution).

### Step 3: Implement AI Processing
1. Create a **Code** node (`Parse & Filter Trending Topics`): Implement the scoring logic (Relevance = Volume + Niche Match). Ensure it returns an array of objects for parallel processing.
2. Create an **AI Agent** node:
    - Set prompt to include `{{$json.trend.topic}}`, `{{$json.brandProfile.name}}`, and the required JSON structure.
    - Attach a **Claude AI Model** node (Model: `claude-sonnet-4-20250514`, Temperature: `0.7`, Anthropic API Credentials).
3. Create a **Code** node (`Parse Claude Response`): Implement the regex/JSON.parse logic to clean AI markdown.

### Step 4: Timing and Final Output
1. Add a **Wait** node (Resume: After Execution) after the AI parser.
2. Create a **Code** node (`Calculate Optimal Posting Times`): Implement the JS loop that checks `engagementPatterns` against the current date and the target timezone.
3. Create a **Google Sheets** node: Operation `Append`, select your Document ID and Sheet Name.
4. Create a **Code** node (`Format Final Response`): Map the processed items into a summary object (Total Ideas, Urgent Posts, etc.).
5. Create a **Respond to Webhook** node: Set response body to `{{ JSON.stringify($json, null, 2) }}`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Multi-platform trend monitoring | Twitter, Reddit, and Google Trends integration |
| Performance Tracking | Handled via Google Sheets logging |
| Timezone Management | User-definable via webhook payload (defaults to Asia/Kolkata) |
| Content Guardrails | "Prohibited Topics" filter implemented in the parsing code |