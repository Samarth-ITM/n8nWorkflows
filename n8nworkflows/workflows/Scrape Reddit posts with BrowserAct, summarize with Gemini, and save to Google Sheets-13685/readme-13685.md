Scrape Reddit posts with BrowserAct, summarize with Gemini, and save to Google Sheets

https://n8nworkflows.xyz/workflows/scrape-reddit-posts-with-browseract--summarize-with-gemini--and-save-to-google-sheets-13685


# Scrape Reddit posts with BrowserAct, summarize with Gemini, and save to Google Sheets

# Reddit Intelligence Monitor: AI-Powered Scraping Reference

This document provides a technical breakdown of the n8n workflow designed to automate market research by monitoring Reddit keywords and competitor subreddits, analyzing content via Google Gemini, and archiving insights into Google Sheets.

---

### 1. Workflow Overview

The workflow acts as a daily intelligence gatherer. It retrieves search terms and subreddit names from a configuration sheet, uses the **BrowserAct** stealth scraper to bypass bot detection, processes the top 3 posts per topic using AI, and saves the summarized results.

#### Logical Blocks:
*   **1.1 Configuration & Trigger:** Scheduled daily execution and retrieval of monitoring targets from Google Sheets.
*   **1.2 Routing & Looping:** Separating targets into "Keyword Search" and "Competitor Subreddit" paths.
*   **1.3 Stealth Scraping:** Utilizing BrowserAct to navigate Reddit and extract raw post data.
*   **1.4 Data Refinement:** Cleaning raw JSON/HTML data and batching posts to optimize AI token usage.
*   **1.5 AI Analysis:** Generating executive summaries for each topic using Google Gemini.
*   **1.6 Archiving:** Appending the finalized intelligence reports back to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Configuration & Trigger
*   **Overview:** Sets the automation frequency and loads the "What to monitor" list.
*   **Nodes Involved:** `Schedule Trigger`, `GSheets: Read Config`.
*   **Node Details:**
    *   **Schedule Trigger:** Triggers daily (default set to 10:00 AM).
    *   **GSheets: Read Config:** Connects via Service Account to read the `Config` tab. It expects columns named `keywords` and `competitor`.
    *   **Edge Cases:** If the Google Sheet is empty or the service account lacks "Editor" permissions, the workflow will fail at the start.

#### 1.2 Routing & Looping
*   **Overview:** Distinguishes between keyword-based monitoring and specific subreddit monitoring.
*   **Nodes Involved:** `If: Is Keyword?`, `Is Competitor?`, `Loop: Keyword List`, `Loop: Competitor List`.
*   **Node Details:**
    *   **If Nodes:** Check if the current row from the config sheet contains a keyword or a competitor name.
    *   **Loop Nodes (Split in Batches):** Processes each entry one by one to ensure the scraper and AI nodes handle one target at a time, preventing timeouts.

#### 1.3 Stealth Scraping (BrowserAct)
*   **Overview:** High-level web scraping that mimics human behavior to extract Reddit posts.
*   **Nodes Involved:** `BrowserAct: Scrape Keywords`, `BrowserAct: Scrape Subreddit`.
*   **Node Details:**
    *   **Type:** BrowserAct Integration.
    *   **Configuration:** Uses specific Workflow IDs (pre-configured in BrowserAct) to target Reddit Search or Subreddit feeds.
    *   **Input:** Passes `{{ $json.keywords }}` or `{{ $json.competitor }}` to the BrowserAct task.
    *   **Potential Failures:** Requires a valid BrowserAct API key and active Task IDs. If Reddit changes its UI significantly, the BrowserAct template may need updating.

#### 1.4 Data Refinement
*   **Overview:** Transforms messy scraper output into a clean, batched text block for the AI.
*   **Nodes Involved:** `Clean Keyword Data`, `Clean Competitor Data`, `Format Keyword Digest`, `Format Competitor Digest`.
*   **Node Details:**
    *   **Clean Nodes (Code):** JavaScript logic that parses the BrowserAct output. It extracts the top 3 posts, identifying titles, content (selftext), and URLs while filtering out image links (jpg/png).
    *   **Format Nodes (Code):** Aggregates the 3 posts into a single string formatted with separators (e.g., "========== Post 1 ==========").
    *   **Key Logic:** If no data is found, it returns a "No Data" placeholder to prevent the AI node from crashing.

#### 1.5 AI Analysis
*   **Overview:** Uses LLMs to provide a professional summary of the gathered posts.
*   **Nodes Involved:** `Keyword Analyst`, `Competitor Analyst`, `Google Gemini Chat Model`.
*   **Node Details:**
    *   **Type:** AI Chain with LangChain.
    *   **Prompt:** Instructs the AI to act as an "Intelligence Analyst," summarize in 2-3 sentences, and identify bug reports or feature requests.
    *   **Model:** Google Gemini (PaLM) Api.
    *   **Input Connection:** Receives the `batch_content` from the Format nodes.

#### 1.6 Archiving
*   **Overview:** Finalizes the process by writing the AI's findings into the reporting sheet.
*   **Nodes Involved:** `Archive Keywords`, `Archive Competitors`.
*   **Node Details:**
    *   **Operation:** Append.
    *   **Mapping:** Maps `Date` (via expression), `Summary` (AI output), and the `Competitor/Keyword` name to the Google Sheet.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule Trigger | Workflow Start | - | GSheets: Read Config | |
| GSheets: Read Config | Google Sheets | Data Retrieval | Schedule Trigger | If Nodes | Prep Your Google Sheet: Config needs keywords, competitor. |
| If: Is Keyword? | If | Logic Router | GSheets: Read Config | Loop: Keyword List | |
| Loop: Keyword List | Split In Batches | Iterator | If: Is Keyword? | BrowserAct: Scrape | |
| BrowserAct: Scrape Keywords | BrowserAct | Web Scraper | Loop: Keyword List | Clean Keyword Data | Link BrowserAct: This node acts as your robot to browse Reddit. |
| Clean Keyword Data | Code | Data Cleaning | BrowserAct | Format Digest | |
| Format Keyword Digest | Code | Data Batching | Clean Keyword Data | Keyword Analyst | Smart AI Summary: We pack 3 posts together for the AI. |
| Keyword Analyst | Chain LLM | Content Analysis | Format Digest | Archive Keywords | Smart AI Summary: We pack 3 posts together for the AI. |
| Google Gemini Chat Model | Gemini Chat Model | AI Engine | Keyword/Comp Analyst | - | |
| Archive Keywords | Google Sheets | Data Archiving | Keyword Analyst | Loop: Keyword List | Prep Your Google Sheet: "Report" needs Date, Competitor, Summary. |
| Is Competitor? | If | Logic Router | GSheets: Read Config | Loop: Competitor List | |
| Loop: Competitor List | Split In Batches | Iterator | Is Competitor? | BrowserAct: Scrape | |
| BrowserAct: Scrape Subreddit| BrowserAct | Web Scraper | Loop: Competitor List | Clean Competitor Data | Link BrowserAct: This node acts as your robot to browse Reddit. |
| Clean Competitor Data | Code | Data Cleaning | BrowserAct | Format Digest | |
| Format Competitor Digest | Code | Data Batching | Clean Competitor Data | Competitor Analyst | |
| Competitor Analyst | Chain LLM | Content Analysis | Format Digest | Archive Competitors | |
| Archive Competitors | Google Sheets | Data Archiving | Competitor Analyst | Loop: Competitor List | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create a Google Sheet with a `Config` tab (Columns: `keywords`, `competitor`) and a `Report` tab (Columns: `Date`, `Competitor`, `Summary`).
    *   Obtain a **BrowserAct API Key** and create two tasks: one for Reddit Search and one for Subreddit scraping.
2.  **Trigger Setup:**
    *   Add a **Schedule Trigger** node set to your preferred daily time.
3.  **Data Ingestion:**
    *   Add a **Google Sheets** node (`Read Config`). Configure Service Account credentials. Set "Operation" to "Get Many" for the `Config` tab.
4.  **Routing Logic:**
    *   Add two **If** nodes. One checks if `keywords` is not empty; the other checks if `competitor` is not empty.
5.  **Looping Mechanism:**
    *   Add **Split in Batches** nodes after the If nodes to handle each search/subreddit item individually.
6.  **Scraping Configuration:**
    *   Add **BrowserAct** nodes. Input your Task IDs. Map the `keywords` or `competitor` field from the loop node to the BrowserAct input parameters.
7.  **Data Transformation (The "Cleaning" Nodes):**
    *   Add a **Code** node to parse the BrowserAct string. Use `JSON.parse()` to turn the output into an array. Use `.slice(0, 3)` to keep only the latest 3 posts.
    *   Add a second **Code** node to concatenate these 3 posts into a single `batch_content` string for the AI.
8.  **AI Integration:**
    *   Add a **Chain LLM** node. Connect a **Google Gemini Chat Model** node to it. 
    *   In the Prompt, use the `batch_content` variable and define the required English summary format.
9.  **Storage:**
    *   Add a **Google Sheets** node (`Archive`). Set operation to "Append". 
    *   Map the current date using `{{ new Date().toLocaleDateString() }}` and the AI's `{{ $json.text }}` to the columns.
10. **Closing the Loop:**
    *   Connect the output of the Archive node back to the input of the **Split in Batches** node to process the next item.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **BrowserAct Documentation** | Used for stealth scraping configuration. |
| **Google Gemini API** | Requires a valid API Key from Google AI Studio. |
| **Daily Digest Optimization** | Batching 3 posts per AI call significantly reduces API costs compared to summarizing every post individually. |