Monitor ecommerce reviews with MrScraper, GPT-4o-mini, Slack and Notion

https://n8nworkflows.xyz/workflows/monitor-ecommerce-reviews-with-mrscraper--gpt-4o-mini--slack-and-notion-13819


# Monitor ecommerce reviews with MrScraper, GPT-4o-mini, Slack and Notion

This document provides a technical analysis and reproduction guide for the **BrandPulse 360° — Ecommerce Review Intelligence** workflow.

---

### 1. Workflow Overview
The **BrandPulse 360°** workflow is an automated brand intelligence engine designed to monitor ecommerce product reviews across multiple platforms (Tokopedia, Shopee, Lazada, Amazon, etc.). It automates the cycle of data collection, sentiment analysis, competitive intelligence, and stakeholder reporting.

**Logical Blocks:**
*   **1.1 Trigger & Configuration:** Initiates the daily run and fetches the target product list from Google Sheets.
*   **1.2 Review Discovery:** Crawls product pages using MrScraper to identify specific review URLs or paginated sections.
*   **1.3 Data Extraction & Cleaning:** Scrapes individual reviews and applies heuristic filters (e.g., length vs. rating importance).
*   **1.4 AI Sentiment Analysis:** Uses GPT-4o-mini to categorize emotions, detect competitor mentions, and assess viral risk.
*   **1.5 Response & Storage:** Routes urgent alerts to Slack and archives structured data in Notion.
*   **1.6 Daily Health Reporting:** Aggregates all data from the run to calculate a proprietary "Brand Awareness Score" (BAS) for a daily executive digest.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Configuration
*   **Overview:** Sets the schedule and retrieves the list of SKUs to be monitored.
*   **Nodes Involved:** `Schedule Trigger (Daily 6AM)`, `Get Target Product List`, `Loop Each Product SKU`.
*   **Node Details:**
    *   **Schedule Trigger:** Fires every 24 hours at 06:00.
    *   **Get Target Product List (Google Sheets):** Reads a spreadsheet containing columns for `Platform`, `Product_URL`, `Brand_Name`, `SKU_Code`, and `Category`.
    *   **Loop Each Product SKU (Split in Batches):** Iterates through each product row to ensure independent processing.

#### 2.2 Review Discovery (Phase 2)
*   **Overview:** Targets the specific sections of ecommerce sites where reviews are hosted.
*   **Nodes Involved:** `MrScraper - Discover Review URLs`, `Extract Review URLs`.
*   **Node Details:**
    *   **MrScraper (Map Agent):** Navigates to the product URL to find sub-links (Review List Scraper).
    *   **Extract Review URLs (Code):** Uses Regex (`/review/i`, `/ulasan/i`, etc.) to filter discovered URLs, falling back to the main product page if no specific review links are found.

#### 2.3 Extraction & Filtering (Phase 3)
*   **Overview:** Pulls raw text and metadata from individual reviews and filters out noise.
*   **Nodes Involved:** `Loop Each Review`, `MrScraper - Extract Review Content`, `Filter & Enrich Review Data`, `Keep Only Valid Reviews`.
*   **Node Details:**
    *   **MrScraper (General Agent):** Extracts `review_text`, `rating`, `photo_count`, and `seller_reply`.
    *   **Filter & Enrich (Code):** 
        *   **Edge Case:** Skips reviews < 10 words *unless* the rating is ≤ 2 stars.
        *   **Deduplication:** Creates a base36 hash of the reviewer, date, and text.
        *   **Viral Pre-flag:** Flags reviews with 3+ photos or 10+ helpful votes as high-risk.
    *   **Keep Only Valid Reviews (If):** Drops items marked as `skip: true`.

#### 2.4 AI Intelligence (Phase 4)
*   **Overview:** Performs deep linguistic analysis of the review text.
*   **Nodes Involved:** `Brand Sentiment AI Agent`, `OpenAI Chat Model`, `Parse AI Response & Format Output`.
*   **Node Details:**
    *   **AI Agent (Chain):** Instructs GPT-4o-mini to return a JSON object with 15 specific fields (Sentiment score, Emotion tags, CX score, etc.).
    *   **OpenAI Chat Model:** Configured with `gpt-4o-mini` for cost-efficiency ($0.0001 per review).
    *   **Parse AI Response (Code):** Sanitizes the JSON output, adds Emojis based on sentiment, and builds the `slackUrgentMessage` string.

#### 2.5 Storage, Alerts & Reporting (Phase 5)
*   **Overview:** Finalizes the run by notifying humans and updating the database.
*   **Nodes Involved:** `Needs Urgent Alert?`, `Slack - Urgent Brand Alert`, `Notion - Save Review Analysis`, `Build Daily Brand Health Digest`, `Slack - Daily Brand Digest`.
*   **Node Details:**
    *   **Needs Urgent Alert? (If):** Checks `action_required`. True if urgency is "critical/high" or "viral_risk" is detected.
    *   **Notion Node:** Maps 27 fields to a Notion Database for long-term auditing.
    *   **Build Daily Digest (Code):** Aggregates all reviews to calculate the **BAS (Brand Awareness Score)**: `(Avg Rating) + (Positive Rate) + (Sentiment Score) + (Loyalty Signal) - (Viral Penalty)`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule | Temporal Trigger | None | Get Target Product List | Trigger fires daily at 6:00 AM. |
| Get Target Product List | Google Sheets | Data Sourcing | Schedule Trigger | Loop Each Product SKU | Google Sheets provides the product list. |
| Loop Each Product SKU | Split In Batches | Iteration | Get Target Product List | MrScraper - Discover Review URLs | Each row = one product SKU to monitor. |
| MrScraper - Discover | MrScraper | URL Discovery | Loop Each Product SKU | Extract Review URLs | Map Agent crawls the product page. |
| Extract Review URLs | Code | URL Filtering | MrScraper - Discover | Loop Each Review | Filters to only review-pattern URLs. |
| Loop Each Review | Split In Batches | Iteration | Extract Review URLs | MrScraper - Extract | URL passed into the review extraction loop. |
| MrScraper - Extract | MrScraper | Content Scraper | Loop Each Review | Filter & Enrich | General Agent extracts full review data. |
| Filter & Enrich | Code | Data Sanitization | MrScraper - Extract | Keep Only Valid Reviews | Skips reviews under 10 words unless rating ≤ 2. |
| Keep Only Valid Reviews| If | Gatekeeper | Filter & Enrich | AI Agent | Passes only items where skip = false. |
| Brand Sentiment AI Agent| AI Chain | Analysis | Keep Only Valid Reviews | Parse AI Response | GPT-4o-mini analyzes each review. |
| OpenAI Chat Model | OpenAI | LLM Engine | None | Brand Sentiment AI Agent | gpt-4o-mini ≈ $0.0001 per review. |
| Parse AI Response | Code | Post-processing | Brand Sentiment AI Agent | Needs Urgent Alert? | Sanitizes output and prepares Slack message. |
| Needs Urgent Alert? | If | Logic Branch | Parse AI Response | Slack Alert / Notion | Routes reviews where action_required = true. |
| Slack - Urgent Alert | Slack | Instant Notification| Needs Urgent Alert? | Notion | Fires instantly to #brand-alerts. |
| Notion - Save Review | Notion | Database Storage | Needs Urgent Alert / Slack | Loop Each Review | Stores every processed review (27 fields). |
| Build Daily Digest | Code | Aggregation | Loop Each Product SKU | Slack - Daily Digest | Calculates Brand Awareness Score (BAS). |
| Slack - Daily Digest | Slack | Reporting | Build Daily Digest | None | Posts full report to #brand-monitoring. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   **Google Sheets:** Create a sheet with headers: `Platform`, `Product_URL`, `Brand_Name`, `SKU_Code`, `Category`, `Active`.
    *   **MrScraper:** Create two agents: (1) a **Map Agent** to find URLs and (2) a **General Agent** to extract text, stars, and user data.
    *   **Notion:** Create a Database with properties matching the "Notion - Save Review Analysis" node (Sentiment, CX Score, Viral Risk, etc.).
2.  **Trigger Setup:**
    *   Add a **Schedule Trigger** (6 AM).
    *   Add a **Google Sheets** node; select your file and the "Read" operation.
3.  **Discovery Loop:**
    *   Add **Split in Batches** (Loop Each Product SKU).
    *   Add **MrScraper (Map Agent)**; map the `Product_URL` from Sheets.
    *   Add a **Code Node** (Extract Review URLs) to filter the results using `reviewPatterns`.
4.  **Extraction Loop:**
    *   Add a second **Split in Batches** (Loop Each Review).
    *   Add **MrScraper (General Agent)** to get the actual review content.
    *   Add a **Code Node** (Filter & Enrich) to calculate the `reviewHash` and word counts.
5.  **Intelligence Setup:**
    *   Add a **Chain LLM** node (AI Agent). Paste the system prompt defining the 15-field JSON output.
    *   Attach an **OpenAI Chat Model** (gpt-4o-mini).
    *   Add a **Code Node** (Parse AI Response) to handle `JSON.parse` and emoji mapping.
6.  **Action Routing:**
    *   Add an **If Node** (Needs Urgent Alert?) checking `{{ $json.action_required }}`.
    *   Connect the "True" branch to a **Slack Node** (Urgent Brand Alert).
    *   Connect both "True" (post-Slack) and "False" branches to a **Notion Node** (Database Page: Create).
7.  **Closure:**
    *   Loop the Notion node back to the **Loop Each Review** node.
    *   Loop the **Loop Each Review** "Done" branch back to the **Loop Each Product SKU**.
    *   Connect the "Done" branch of the first loop to a **Code Node** (Build Daily Digest) to aggregate `$input.all()`.
    *   Connect to a final **Slack Node** (Daily Brand Digest).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **MrScraper API Setup** | Required to enable "AI Scraper API access" in settings for dynamic content. |
| **Brand Awareness Score (BAS)** | Proprietary calculation: Max 100 points. Rating(30) + Positive%(25) + Sentiment(20) + Loyalty(10) + Benchmark(5) - ViralPenalty(10). |
| **Cost Optimization** | Using `gpt-4o-mini` instead of `gpt-4o` reduces costs by ~90% while maintaining analysis accuracy. |
| **Deduplication Logic** | The Code node `reviewHash` prevents double-counting if the scraper re-reads the same review on different days. |