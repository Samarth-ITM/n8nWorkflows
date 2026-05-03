Analyze Google Forms feedback with Gemini AI and send weekly reports

https://n8nworkflows.xyz/workflows/analyze-google-forms-feedback-with-gemini-ai-and-send-weekly-reports-13772


# Analyze Google Forms feedback with Gemini AI and send weekly reports

This document provides a technical breakdown of the **Customer Feedback Analyzer** workflow, designed to automate the sentiment analysis and reporting of Google Forms responses using Gemini AI.

---

### 1. Workflow Overview

The workflow automates the transition from raw customer feedback to structured business intelligence. It monitors a Google Sheet (linked to a Google Form), applies generative AI to categorize and score the text, and distributes the findings across project management tools and email.

**Logical Blocks:**
*   **1.1 Data Collection:** Triggered on a schedule to fetch and filter new, unanalyzed responses.
*   **1.2 AI Intelligence:** Uses Google Gemini to perform multi-dimensional analysis (sentiment, theme, and recommendations).
*   **1.3 Data Persistence & Reporting:** Updates the source spreadsheet, logs high-level metrics in Notion, and sends a formatted HTML report via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Collection
**Overview:** This block acts as the entry point, ensuring only new data is processed to manage API costs and prevent duplicate analysis.
*   **Nodes Involved:** `Daily feedback check`, `Read feedback responses`, `Filter unprocessed responses`.
*   **Node Details:**
    *   **Daily feedback check (Schedule Trigger):** Configured to trigger every day at 9:00 AM.
    *   **Read feedback responses (Google Sheets):** Connects to the specific Document ID and Sheet Name where Google Form responses are stored.
    *   **Filter unprocessed responses (Code):** 
        *   **Logic:** Iterates through rows. If the "Processed" column is empty or "no", it adds the row to an array.
        *   **Output:** It aggregates all individual responses into a single string (`feedbackText`) to allow the AI to process the entire batch in one call.
        *   **Edge Case:** If no new responses are found, it returns a `skip: true` flag to prevent downstream errors.

#### 2.2 AI Intelligence
**Overview:** Transforms unstructured text into structured JSON data.
*   **Nodes Involved:** `Analyze feedback with Gemini`, `Gemini Chat Model`.
*   **Node Details:**
    *   **Analyze feedback with Gemini (Basic LLM Chain):** 
        *   **Configuration:** Uses a system prompt requiring the AI to return a specific JSON schema containing `sentiment`, `score` (1-10), `category`, and `theme`.
        *   **Input:** Uses the `{{ $json.feedbackText }}` variable from the previous block.
    *   **Gemini Chat Model (Chat Google Gemini):**
        *   **Model:** `gemini-2.0-flash`.
        *   **Settings:** Temperature is set to `0.3` for consistent, deterministic outputs.
        *   **Failure Type:** API Authentication errors or context window limits if the feedback batch is excessively large.

#### 2.3 Reporting & Distribution
**Overview:** Parses the AI's response and updates external systems.
*   **Nodes Involved:** `Parse results and build report`, `Update feedback sheet with results`, `Log insights to Notion`, `Email report to team`.
*   **Node Details:**
    *   **Parse results and build report (Code):** 
        *   **Technical Role:** Uses Regex (`/\{[\s\S]*\}/`) to extract the JSON object from the AI's text response. 
        *   **Logic:** Calculates the `positiveRate` and `negativeRate` and constructs a multi-section HTML string for the email body.
    *   **Update feedback sheet with results (Google Sheets):** 
        *   **Operation:** `Update`. Maps the AI results back to the original rows using the `rowIndex`.
    *   **Log insights to Notion (Notion):** 
        *   **Action:** Creates a database page. 
        *   **Properties:** Maps `Total Responses`, `Positive Rate`, `Top Themes`, and `Recommendation`.
    *   **Email report to team (Gmail):** 
        *   **Configuration:** Sends the generated `emailBody` as an HTML message.
        *   **Subject:** Dynamic subject line including the current ISO date.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily feedback check | Schedule Trigger | Workflow Entry | (None) | Read feedback responses | Data collection |
| Read feedback responses | Google Sheets | Data Retrieval | Daily feedback check | Filter unprocessed responses | Data collection |
| Filter unprocessed responses | Code | Data Filtering | Read feedback responses | Analyze feedback with Gemini | Data collection |
| Analyze feedback with Gemini | Basic LLM Chain | AI Processing | Filter unprocessed responses | Parse results and build report | AI analysis |
| Gemini Chat Model | Google Gemini | Language Model | (None) | Analyze feedback with Gemini | AI analysis |
| Parse results and build report | Code | Data Formatting | Analyze feedback with Gemini | Update feedback sheet, Log to Notion | Reporting |
| Update feedback sheet with results | Google Sheets | Data Persistence | Parse results and build report | Email report to team | Reporting |
| Log insights to Notion | Notion | Logging | Parse results and build report | (None) | Reporting |
| Email report to team | Gmail | Communication | Update feedback sheet | (None) | Reporting |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** 
    *   Create a Google Sheet with headers: `Timestamp`, `Feedback`, `Processed`, `Sentiment`, `Category`, `Score`, `Theme`.
    *   Create a Notion Database with properties: `Date` (Title), `Total Responses` (Number), `Positive Rate` (Number), `Top Themes` (Text), `Recommendation` (Text).
2.  **Trigger Setup:** Add a **Schedule Trigger** node set to "Daily".
3.  **Data Fetching:** Add a **Google Sheets** node. Set the action to "Get Many" and point it to your feedback sheet.
4.  **Batch Processing Logic:** Add a **Code** node. Paste logic to check if `row.Processed` is falsy. Create an object that combines the row index and the text response.
5.  **AI Integration:** 
    *   Add a **Basic LLM Chain** node. 
    *   Connect a **Google Gemini Chat Model** node to its AI Model input. Select `gemini-2.0-flash`.
    *   In the Chain node, write a prompt asking for JSON output including: sentiment, score, and a summary recommendation.
6.  **Data Parsing:** Add a **Code** node. Use `JSON.parse` on the AI output. Construct an HTML string using standard tags (`<h2>`, `<ul>`, `<li>`) for the email body.
7.  **Update Source:** Add another **Google Sheets** node. Set the action to "Update". Use the `rowIndex` from the AI output to write the sentiment and category back to the specific sheet row. Set "Processed" to "yes".
8.  **Logging:** Add a **Notion** node. Set action to "Create Database Page" and map the summary fields.
9.  **Notification:** Add a **Gmail** node. Set the "Message" field to the `emailBody` variable and enable the "Is HTML" option.
10. **Credentials:** Ensure OAuth2 credentials are provided for Google (Sheets/Gmail) and API keys for Notion and Google Gemini.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisite:** Create a Google Form and link it to a Google Sheet first. | Implementation Requirement |
| **Model Choice:** Gemini 2.0 Flash is recommended for speed and cost-efficiency in text analysis. | Technical Optimization |
| **Scaling:** For high-volume forms, consider increasing the memory limit of the n8n instance. | Infrastructure Note |