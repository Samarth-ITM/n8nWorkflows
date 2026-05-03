Audit competitor SEO content with Decodo, Gemini, and Google Sheets

https://n8nworkflows.xyz/workflows/audit-competitor-seo-content-with-decodo--gemini--and-google-sheets-13817


# Audit competitor SEO content with Decodo, Gemini, and Google Sheets

# Reference Document: Audit Competitor SEO Content with Decodo, Gemini, and Google Sheets

This document provides a technical breakdown and reproduction guide for an automated n8n workflow designed to perform high-fidelity SEO audits of competitor content. By integrating **Decodo** for search and scraping, **Google Gemini** for AI analysis, and **Google Sheets** for data management, the workflow identifies content gaps between a specific brand identity and top-ranking search results.

---

### 1. Workflow Overview

The workflow automates the process of discovering competitors for specific keywords, extracting their website content, and analyzing that content against a defined "Brand Identity." The goal is to generate a structured action plan to outperform competitors based on unique brand USPs (Unique Selling Propositions).

#### Functional Blocks:
*   **1.1 Config & Brand Input:** Sets global search parameters and retrieves the "Gold Standard" brand guidelines.
*   **1.2 Market Discovery:** Sources target keywords and performs a live Google Search to identify top competitors.
*   **1.3 Intelligence Harvesting:** Splits search results into individual URLs and scrapes the full body content of each page.
*   **1.4 Strategic Audit:** Uses LLM (Gemini) and specialized SEO prompts to compare competitor content against the brand identity.
*   **1.5 Reporting Deck:** Appends the structured AI analysis and "Checkmate" instructions back into a master Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Config & Brand Input
*   **Overview:** Establishes the environment variables for the search (GEO/Locale) and loads the brand's core identity to use as the auditing benchmark.
*   **Nodes Involved:** `Manual Execution`, `Schedule Trigger`, `Config`, `Brand Info`.
*   **Node Details:**
    *   **Config (Set):** Defines `target_geo` (e.g., United States), `target_locale` (e.g., en-US), and `serp_results_amount` (number of competitors to analyze).
    *   **Brand Info (Google Sheets):** Reads from a specific sheet containing the brand's identity and value propositions. This serves as the "System Prompt" context later.

#### 2.2 Market Discovery
*   **Overview:** Identifies the current SEO landscape for the provided keywords.
*   **Nodes Involved:** `Data Sourcing`, `Google Search`.
*   **Node Details:**
    *   **Data Sourcing (Google Sheets):** Fetches the list of target keywords from the input spreadsheet.
    *   **Google Search (Decodo):** Executes a live search for each keyword using Decodo's API.
        *   *Key Parameters:* Uses expressions from the `Config` node for geo-targeting and limit settings.

#### 2.3 Intelligence Harvesting
*   **Overview:** Transforms raw search results into individual items and extracts the readable text from competitor pages.
*   **Nodes Involved:** `Split to Competitor URLs`, `Universal Scraping`, `Extract Body`.
*   **Node Details:**
    *   **Split to Competitor URLs (Code):** A JavaScript snippet that iterates through the search results, limits them based on the config, and flattens the data into individual URL items.
    *   **Universal Scraping (Decodo):** Navigates to each competitor URL and retrieves the HTML. It includes a retry logic (3 tries) to handle potential network issues.
    *   **Extract Body (HTML):** Uses a CSS selector (`body`) to isolate the main content while stripping out `img` and `link` tags to reduce token noise for the AI.

#### 2.4 Strategic Audit
*   **Overview:** The "Brain" of the workflow. It performs a Search Intent Audit using an AI Agent.
*   **Nodes Involved:** `AI Content Auditor`, `Google Gemini Chat Model`, `Structured Output Parser`.
*   **Node Details:**
    *   **AI Content Auditor (AI Agent):** Orchestrates the prompt. It receives the scraped content and the Brand Identity.
        *   *Prompt Logic:* Defines a "Content Gap" as a missing brand-specific angle rather than just missing text.
    *   **Google Gemini Chat Model:** The underlying LLM (PaLM/Gemini) used for the analysis.
    *   **Structured Output Parser:** Ensures the AI returns valid JSON with specific keys: `primary_h1`, `word_count`, `top_topics`, `winning_factor`, `content_gap`, and `action_plan`.

#### 2.5 Reporting Deck
*   **Overview:** Finalizes the process by recording the findings.
*   **Nodes Involved:** `Strategy Master Writer`, `Done`.
*   **Node Details:**
    *   **Strategy Master Writer (Google Sheets):** Appends the AI-generated audit (Rank, Keyword, The Gap, Action Plan, etc.) into a dedicated "Competitor Audit Feed" tab.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Manual Execution | Manual Trigger | Manual Start | None | Config | Config |
| Schedule Trigger | Schedule Trigger | Automated Start | None | Config | Config |
| Config | Set | Variable Setup | Manual/Schedule | Brand Info | Config |
| Brand Info | Google Sheets | Context Loading | Config | Data Sourcing | Config |
| Data Sourcing | Google Sheets | Input Retrieval | Brand Info | Google Search | Market Discovery |
| Google Search | Decodo | SERP Retrieval | Data Sourcing | Split to Competitor URLs | Market Discovery |
| Split to Competitor URLs | Code | Data Flattening | Google Search | Universal Scraping | Intelligence Harvesting |
| Universal Scraping | Decodo | Web Scraping | Split to Competitor URLs | Extract Body | Intelligence Harvesting |
| Extract Body | HTML | Cleaning Content | Universal Scraping | AI Content Auditor | Intelligence Harvesting |
| AI Content Auditor | AI Agent | SEO Audit | Extract Body | Strategy Master Writer | Strategic Audit |
| Google Gemini Chat Model | Gemini Chat | AI Reasoning | None | AI Content Auditor | Strategic Audit |
| Structured Output Parser | Output Parser | Data Formatting | None | AI Content Auditor | Strategic Audit |
| Strategy Master Writer | Google Sheets | Data Logging | AI Content Auditor | Done | Reporting Deck |
| Done | No-Op | End Point | Strategy Master Writer | None | Reporting Deck |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** Create a Google Sheet with three tabs: `Target Keywords`, `Brand Identity`, and `Competitor Audit Feed`.
2.  **Global Config:**
    *   Add a **Manual Trigger** and a **Schedule Trigger**.
    *   Connect them to a **Set Node** (`Config`). Define `target_geo` (String), `target_locale` (String), and `serp_results_amount` (Number).
3.  **Data Ingestion:**
    *   Add a **Google Sheets Node** (`Brand Info`) to download the brand profile.
    *   Add a **Google Sheets Node** (`Data Sourcing`) to fetch keywords.
4.  **Search & Filtering:**
    *   Add a **Decodo Node** (`Google Search`). Set the operation to `google_search` and map the query to the keyword and limits to the `Config` node.
    *   Add a **Code Node** (`Split to Competitor URLs`). Use the script to iterate through `results.organic` and `slice` the array by the `resultLimit`.
5.  **Extraction:**
    *   Add a **Decodo Node** (`Universal Scraping`). Map the URL from the previous node.
    *   Add an **HTML Node** (`Extract Body`). Set the operation to `Extract HTML Content`, using the CSS selector `body` and ignoring `img` tags.
6.  **AI Integration:**
    *   Add an **AI Agent Node** (`AI Content Auditor`).
    *   Connect a **Google Gemini Chat Model** and a **Structured Output Parser** to the Agent.
    *   In the Agent's System Message, reference the `Brand Info` node to define the "Gold Standard."
    *   In the Output Parser, define the JSON schema (H1, Word Count, Winning Factor, Gap, Action Plan).
7.  **Final Output:**
    *   Add a **Google Sheets Node** (`Strategy Master Writer`). Set the operation to `Append`. Map the AI's structured output keys to your spreadsheet columns.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Master Google Sheet Template** | [View Spreadsheet](https://docs.google.com/spreadsheets/d/1FRRxOZeNt7rEi-87Nlm_wKSP4Ue2FnMSeIKBNm6Onao/edit?gid=0#gid=0) |
| **Decodo API Credentials** | [Get Decodo API Key](https://visit.decodo.com/c/6679292/3071239/17480) |
| **Search Depth Tip** | Control token costs by lowering `serp_results_amount` in the Config node. |
| **Anti-Bot Handling** | Decodo's Universal Scraping is used to bypass common blocks that standard HTTP requests fail against. |