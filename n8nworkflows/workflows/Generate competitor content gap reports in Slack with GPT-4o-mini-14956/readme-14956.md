Generate competitor content gap reports in Slack with GPT-4o-mini

https://n8nworkflows.xyz/workflows/generate-competitor-content-gap-reports-in-slack-with-gpt-4o-mini-14956


# Generate competitor content gap reports in Slack with GPT-4o-mini

# Technical Documentation: Competitor Content Gap Analyzer

## 1. Workflow Overview
The **Competitor Content Gap Analyzer** is an automated SEO tool designed to compare a user's webpage against a competitor's webpage for a specific target keyword. It scrapes the content of both pages, cleans the HTML to extract readable text, and utilizes GPT-4o-mini to generate a structured 6-section gap analysis. The final report is delivered directly to a Slack channel.

### Logical Blocks
- **1.1 Input Reception:** Captures URLs and SEO metadata via an n8n form.
- **1.2 Parallel Page Scraping and Cleaning:** Simultaneously fetches HTML from both URLs and strips non-essential elements (scripts/styles) to optimize AI token usage.
- **1.3 Data Merge:** Consolidates the cleaned text from both sources into a single data object.
- **1.4 AI Content Gap Analysis:** Employs a Large Language Model (LLM) to analyze the content and identify gaps.
- **1.5 Slack Report Delivery:** Formats the AI analysis and posts it to a specified Slack channel.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception
**Overview:** This block handles the initial user interaction and prepares the variables needed for the rest of the workflow.

- **Nodes Involved:** `1. Form — Submit Page URLs`, `2. Set — Extract Form Fields`.
- **Node Details:**
    - **1. Form — Submit Page URLs (Form Trigger):** 
        - **Role:** Entry point. Collects "Your Page URL", "Competitor Page URL", "Target Keyword", and "Your Business Name".
        - **Connection:** Triggers the `Set` node.
    - **2. Set — Extract Form Fields (Set):**
        - **Role:** Data mapping and normalization.
        - **Configuration:** Maps form fields to clean variable names (e.g., `yourUrl`, `targetKeyword`) and generates a `runDate` using the expression `{{ $now.toFormat('dd MMM yyyy HH:mm') }}`.
        - **Failure Potential:** Missing required fields if the form validation is bypassed.

### 2.2 Parallel Page Scraping and Cleaning
**Overview:** Fetches the raw HTML of both pages in parallel and cleans the content to ensure the AI receives only relevant text.

- **Nodes Involved:** `3. HTTP — Scrape Your Page`, `4. HTTP — Scrape Competitor Page`, `5. Code — Clean Your Page HTML`, `6. Code — Clean Competitor Page HTML`.
- **Node Details:**
    - **3. & 4. HTTP Request Nodes:**
        - **Role:** Fetch web content.
        - **Configuration:** Dynamic URLs based on form input; timeout set to 15,000ms.
        - **Edge Case:** Sites with bot protection may return a **403 Forbidden** error. (Solution: Add a `User-Agent` header).
    - **5. & 6. Code Nodes (JavaScript):**
        - **Role:** HTML Sanitization.
        - **Logic:** Uses Regex to remove `<script>` and `<style>` tags, strips all remaining HTML tags, collapses whitespace, and limits output to the first **8,000 characters** to prevent token overflow.
        - **Input:** Raw HTML from HTTP nodes.
        - **Output:** Cleaned text string + original metadata.

### 2.3 Data Merge
**Overview:** Synchronizes the two parallel paths back into a single stream of data.

- **Nodes Involved:** `7. Merge — Combine Both Pages`, `8. Code — Combine Page Data`.
- **Node Details:**
    - **7. Merge (Merge):**
        - **Role:** Joins the two parallel execution paths (Your Page vs Competitor Page).
        - **Mode:** Combine.
    - **8. Code — Combine Page Data (JavaScript):**
        - **Role:** Data safety and object consolidation.
        - **Logic:** Combines the arrays from the Merge node into one JSON object. Includes fallback text ("Could not scrape...") if one of the pages failed to load.
        - **Output:** A single object containing both `yourPageText` and `competitorPageText`.

### 2.4 AI Content Gap Analysis
**Overview:** The core intelligence block where the SEO analysis is performed.

- **Nodes Involved:** `9. AI Agent — Gap Analyzer`, `10. OpenAI — GPT-4o-mini Model`.
- **Node Details:**
    - **9. AI Agent (Agent):**
        - **Role:** Orchestrates the analysis based on a complex system prompt.
        - **Configuration:** The prompt mandates a 6-section report: Keyword Usage, Missing Topics, Your Advantages, Depth/Quality, 5 Priority Actions, and a Quick Verdict.
        - **Constraints:** Strictly requests plain text (no markdown/asterisks) for cleaner Slack formatting.
    - **10. OpenAI Model (ChatOpenAi):**
        - **Role:** The "brain" providing the analysis.
        - **Configuration:** Model: `gpt-4o-mini`, Temperature: `0.4` (for balanced creativity and consistency), Max Tokens: `1500`.

### 2.5 Slack Report Delivery
**Overview:** Final delivery of the analysis to the end-user.

- **Nodes Involved:** `11. Set — Prepare Slack Message`, `12. Slack — Send Gap Report`.
- **Node Details:**
    - **11. Set — Prepare Slack Message (Set):**
        - **Role:** Aggregates the AI output and original URLs for the final message.
        - **Variable:** `gapReport` captures the AI output.
    - **12. Slack — Send Gap Report (Slack):**
        - **Role:** Sends the formatted message to a channel.
        - **Configuration:** Uses Slack's `mrkdwn` for bolding headers.
        - **Auth:** Requires OAuth2 credentials.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Form — Submit Page URLs | Form Trigger | User Input Capture | N/A | 2. Set | Form Input and Field Extraction |
| 2. Set — Extract Form Fields | Set | Data Normalization | 1. Form | 3. HTTP, 4. HTTP | Form Input and Field Extraction |
| 3. HTTP — Scrape Your Page | HTTP Request | Fetch User Page | 2. Set | 5. Code | Parallel Page Scraping and Cleaning |
| 4. HTTP — Scrape Competitor Page | HTTP Request | Fetch Competitor Page | 2. Set | 6. Code | Parallel Page Scraping and Cleaning |
| 5. Code — Clean Your Page HTML | Code | HTML Stripping | 3. HTTP | 7. Merge | Parallel Page Scraping and Cleaning |
| 6. Code — Clean Competitor Page HTML | Code | HTML Stripping | 4. HTTP | 7. Merge | Parallel Page Scraping and Cleaning |
| 7. Merge — Combine Both Pages | Merge | Sync Parallel Paths | 5. Code, 6. Code | 8. Code | Data Merge |
| 8. Code — Combine Page Data | Code | Object Consolidation | 7. Merge | 9. AI Agent | Data Merge |
| 9. AI Agent — Gap Analyzer | AI Agent | SEO Analysis | 8. Code | 11. Set | AI Content Gap Analysis |
| 10. OpenAI — GPT-4o-mini Model | ChatOpenAi | LLM Provider | N/A | 9. AI Agent | AI Content Gap Analysis |
| 11. Set — Prepare Slack Message | Set | Report Formatting | 9. AI Agent | 12. Slack | Slack Report Delivery |
| 12. Slack — Send Gap Report | Slack | Notification Delivery | 11. Set | N/A | Slack Report Delivery |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Input & Mapping
1. Create a **Form Trigger** node. Add four required fields: `Your Page URL`, `Competitor Page URL`, `Target Keyword`, and `Your Business Name`.
2. Create a **Set** node connected to the form. Map the four form fields to variables: `yourUrl`, `competitorUrl`, `targetKeyword`, and `businessName`. Add a variable `runDate` using the expression `{{ $now.toFormat('dd MMM yyyy HH:mm') }}`.

### Step 2: Parallel Scraping
3. Create two **HTTP Request** nodes.
    - Node A: Set URL to `{{ $json.yourUrl }}`.
    - Node B: Set URL to `{{ $('2. Set — Extract Form Fields').item.json.competitorUrl }}`.
    - Set timeout to 15,000ms for both.
4. Create two **Code** nodes (JavaScript) to clean the HTML. Use a function to:
    - Remove `<script>` and `<style>` tags using Regex.
    - Remove all remaining HTML tags.
    - Trim whitespace and slice the string to 8,000 characters.
    - Ensure the output object contains the cleaned text and the original metadata (URL, Keyword, etc.).

### Step 3: Data Integration
5. Create a **Merge** node. Set mode to "Combine" and connect both Code nodes to it.
6. Create a **Code** node to safely combine the two inputs. Ensure the output is a single item with keys `yourPageText` and `competitorPageText`.

### Step 4: AI Analysis
7. Create an **AI Agent** node.
    - Use a "Define" prompt type.
    - Paste the detailed SEO strategist prompt (including the 6 required sections).
    - Pass the `businessName`, `targetKeyword`, `yourUrl`, `competitorUrl`, `yourPageText`, and `competitorPageText` into the prompt via expressions.
8. Create an **OpenAI Chat Model** node and connect it to the AI Agent.
    - Model: `gpt-4o-mini`.
    - Temperature: `0.4`.
    - Max Tokens: `1500`.

### Step 5: Final Delivery
9. Create a **Set** node to prepare the Slack message. Create a variable `gapReport` using `{{ $json.output }}`.
10. Create a **Slack** node.
    - Authentication: OAuth2.
    - Text: Use a template to include the Business Name, Keyword, URLs, and the `gapReport` content.
    - Enable `mrkdwn` in options.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Bot Protection Handling** | If nodes 3 or 4 return a 403 error, add a Header: `User-Agent` = `Mozilla/5.0 (compatible; n8n-bot/1.0)` |
| **Token Optimization** | The Code nodes specifically limit text to 8,000 characters to ensure the prompt fits within the `gpt-4o-mini` context window. |
| **Formatting Constraint** | The AI is explicitly told to avoid markdown in the response to prevent conflicting with Slack's specific block formatting. |