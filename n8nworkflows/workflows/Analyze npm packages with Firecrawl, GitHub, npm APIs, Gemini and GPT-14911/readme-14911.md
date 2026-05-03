Analyze npm packages with Firecrawl, GitHub, npm APIs, Gemini and GPT

https://n8nworkflows.xyz/workflows/analyze-npm-packages-with-firecrawl--github--npm-apis--gemini-and-gpt-14911


# Analyze npm packages with Firecrawl, GitHub, npm APIs, Gemini and GPT

# Technical Reference: AI-Powered NPM Package Intelligence Agent

## 1. Workflow Overview
The **AI-Powered NPM Package Intelligence Agent** is a sophisticated automation designed to evaluate the health, reliability, and risk level of any given npm package. Instead of relying on manual checks, the workflow uses a combination of dynamic web discovery, real-time API data, and AI reasoning to provide a production-ready recommendation.

The logic is divided into the following functional blocks:
- **1.1 Input Reception & Discovery:** Captures the package name and uses Firecrawl to dynamically find the official npm and GitHub URLs.
- **1.2 URL Cleaning & Validation:** Processes search results to ensure only clean, direct repository and registry links are used.
- **1.3 Data Collection (API Layer):** Fetches real-time metrics from GitHub (stars, issues, commits) and npm (download counts).
- **1.4 Metrics Computation & Validation:** Calculates health ratios (issue-to-star) and determines if the package is active or stale.
- **1.5 AI Analysis & Decision Engine:** An AI Agent processes the data, applies a risk framework, and generates a structured report and a human-readable Slack message.
- **1.6 Error Handling & Output:** Manages "Package Not Found" scenarios and delivers the final report via Slack.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Discovery
**Overview:** This block captures the user's request and discovers the digital footprint of the package.
- **Nodes Involved:** `Enter Package Name`, `Normalize Input`, `Search NPM URL`, `Search GitHub Repo`, `Merge Search Results`.
- **Node Details:**
    - **Enter Package Name (Form Trigger):** Provides a UI for the user to enter the package name.
    - **Normalize Input (Set):** Standardizes the package name into a variable `package_name`.
    - **Search NPM URL (Firecrawl):** Performs a targeted search (`site:npmjs.com/package`) to find the official registry page.
    - **Search GitHub Repo (Firecrawl):** Performs a search (`site:github.com`) to locate the source code repository.
    - **Merge Search Results (Merge):** Combines the findings from both Firecrawl searches into a single data object.

### 2.2 URL Cleaning & Validation
**Overview:** Extracts high-confidence URLs from search results to prevent the workflow from visiting irrelevant pages (e.g., issue trackers or PRs).
- **Nodes Involved:** `Extract Clean URLs`.
- **Node Details:**
    - **Extract Clean URLs (Code):** 
        - **Logic:** Uses JavaScript to filter out URLs containing `blob`, `issues`, `pull`, or `commit`.
        - **Fallback:** If no npm URL is found, it programmatically constructs one using `https://www.npmjs.com/package/{package_name}`.
        - **Confidence Scoring:** Assigns "high", "medium", or "low" confidence based on whether both GitHub and npm URLs were discovered.

### 2.3 Data Collection (API Layer)
**Overview:** Gathers quantitative data from official APIs to avoid the instability of web scraping.
- **Nodes Involved:** `Fetch GitHub Stats`, `Fetch Last Commit`, `Commit deatils`, `Fetch NPM Downloads`.
- **Node Details:**
    - **Fetch GitHub Stats (GitHub):** Retrieves `stargazers_count`, `open_issues_count`, and `license`.
    - **Fetch Last Commit (HTTP Request):** Calls the GitHub API `/commits` endpoint to get the most recent activity.
    - **Commit deatils (Set):** Extracts and truncates the commit date, message, and author.
    - **Fetch NPM Downloads (HTTP Request):** Calls the npm API to get the total downloads for the last week.

### 2.4 Metrics Computation & Validation
**Overview:** Transforms raw API numbers into "health signals" and validates if the package is real.
- **Nodes Involved:** `Merge All Metrics`, `Compute Health Metrics`, `Check Package Valid`.
- **Node Details:**
    - **Merge All Metrics (Merge):** Aggregates data from GitHub and npm into a single object.
    - **Compute Health Metrics (Set):** 
        - `issue_to_star_ratio`: Calculated as `open_issues / stars`.
        - `activity_status`: Labeled as "active" if the last commit was $\le$ 7 days ago, otherwise "stale".
    - **Check Package Valid (If):** A critical gate that checks if GitHub stars $> 0$. If false, the workflow redirects to the Error Handler.

### 2.5 AI Analysis & Decision Engine
**Overview:** The "brain" of the workflow. It interprets the metrics and generates a professional risk assessment.
- **Nodes Involved:** `AI Analysis Engine`, `OpenAI Chat Model`, `Google Gemini Chat Model`, `Google Gemini Chat Model1`, `Structured Output Parser`, `/search in Firecrawl`, `/scrape in Firecrawl`.
- **Node Details:**
    - **AI Analysis Engine (AI Agent):** Uses a comprehensive system prompt to separate "Observed Facts" from "Inferred Judgment".
    - **Models:** Employs a mix of OpenAI (GPT-4o mini) and Google Gemini for reasoning.
    - **Tools:** The agent can use Firecrawl's search and scrape tools to look for alternatives or deeper documentation if the API data is insufficient.
    - **Structured Output Parser:** Ensures the AI returns a valid JSON object matching the specific schema (metrics, risk level, recommendation).

### 2.6 Error Handling & Output
**Overview:** Ensures the user receives a response even if the package does not exist.
- **Nodes Involved:** `Package Not Found Handler`, `Send Slack Report`, `Send Slack Report_01`.
- **Node Details:**
    - **Package Not Found Handler (Set):** Creates a "High Risk" mock report for missing packages.
    - **Send Slack Report (Slack):** Posts the AI-generated formatted markdown report to a specific Slack user.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Enter Package Name | Form Trigger | User Input | - | Normalize Input | INPUT + URL DISCOVERY |
| Normalize Input | Set | Variable Setup | Enter Package Name | Search NPM URL | INPUT + URL DISCOVERY |
| Search NPM URL | Firecrawl | Dynamic Discovery | Normalize Input | Search GitHub Repo, Merge Search Results | INPUT + URL DISCOVERY |
| Search GitHub Repo | Firecrawl | Dynamic Discovery | Search NPM URL | Merge Search Results | INPUT + URL DISCOVERY |
| Merge Search Results | Merge | Data Aggregation | Search NPM URL, Search GitHub Repo | Extract Clean URLs | INPUT + URL DISCOVERY |
| Extract Clean URLs | Code | URL Filtering | Merge Search Results | Fetch GitHub Stats | URL CLEANING + VALIDATION |
| Fetch GitHub Stats | GitHub | Metrics Retrieval | Extract Clean URLs | Github details | DATA COLLECTION (APIs Layer) |
| Github details | Set | Data Extraction | Fetch GitHub Stats | Fetch Last Commit, Merge All Metrics | DATA COLLECTION (APIs Layer) |
| Fetch Last Commit | HTTP Request | Activity Retrieval | Github details | Commit deatils | DATA COLLECTION (APIs Layer) |
| Commit deatils | Set | Data Extraction | Fetch Last Commit | Fetch NPM Downloads, Merge All Metrics | DATA COLLECTION (APIs Layer) |
| Fetch NPM Downloads | HTTP Request | Popularity Retrieval | Commit deatils | Merge All Metrics | DATA COLLECTION (APIs Layer) |
| Merge All Metrics | Merge | Data Aggregation | Fetch NPM Downloads, Commit deatils, Github details | Compute Health Metrics | METRICS + VALIDATION LOGIC |
| Compute Health Metrics | Set | Signal Calculation | Merge All Metrics | Check Package Valid | METRICS + VALIDATION LOGIC |
| Check Package Valid | If | Quality Gate | Compute Health Metrics | AI Analysis Engine, Package Not Found Handler | METRICS + VALIDATION LOGIC |
| AI Analysis Engine | AI Agent | Decision Making | Check Package Valid | Send Slack Report | AI ANALYSIS + OUTPUT |
| OpenAI Chat Model | LLM | Reasoning | - | AI Analysis Engine | AI ANALYSIS + OUTPUT |
| Google Gemini Chat Model | LLM | Reasoning | - | Structured Output Parser | AI ANALYSIS + OUTPUT |
| Structured Output Parser | Parser | Schema Validation | Google Gemini Chat Model | AI Analysis Engine | AI ANALYSIS + OUTPUT |
| Google Gemini Chat Model1 | LLM | Reasoning | - | AI Analysis Engine | AI ANALYSIS + OUTPUT |
| /search in Firecrawl | Tool | Context Enrichment | - | AI Analysis Engine | AI ANALYSIS + OUTPUT |
| /scrape in Firecrawl | Tool | Context Enrichment | - | AI Analysis Engine | AI ANALYSIS + OUTPUT |
| Package Not Found Handler | Set | Fallback Logic | Check Package Valid | Send Slack Report_01 | ERROR HANDLING |
| Send Slack Report | Slack | User Notification | AI Analysis Engine | - | AI ANALYSIS + OUTPUT |
| Send Slack Report_01 | Slack | User Notification | Package Not Found Handler | - | ERROR HANDLING |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Input & Discovery Setup
1. Create a **Form Trigger** named "Enter Package Name" with a required text field "Package Name".
2. Add a **Set** node "Normalize Input" to assign `package_name` = `{{ $json['Package Name'] }}`.
3. Add two **Firecrawl** nodes:
    - **Search NPM URL**: Set operation to `Search`, query to `{{ $json.package_name }} site:npmjs.com/package`.
    - **Search GitHub Repo**: Set operation to `Search`, query to `{{ $json.package_name }} site:github.com`.
4. Connect both to a **Merge** node (Combine by position).

### Step 2: URL Logic
1. Add a **Code** node "Extract Clean URLs". Paste the JavaScript logic to filter URLs and implement the npm fallback URL construction.

### Step 3: API Integration
1. Add a **GitHub** node "Fetch GitHub Stats" (Repository $\rightarrow$ Get). Use expressions to extract the owner and repo name from the cleaned `github_url`.
2. Add an **HTTP Request** node "Fetch Last Commit" calling `https://api.github.com/repos/{owner}/{repo}/commits?per_page=1`.
3. Add a **Set** node "Commit details" to parse the date and message.
4. Add an **HTTP Request** node "Fetch NPM Downloads" calling `https://api.npmjs.org/downloads/point/last-week/{package_name}`.
5. Use a **Merge** node to combine the outputs of the GitHub Stats, Commit details, and NPM downloads.

### Step 4: Metrics & Validation
1. Add a **Set** node "Compute Health Metrics" to calculate `issue_to_star_ratio` and `activity_status` (Date difference $\le$ 7 days).
2. Add an **If** node "Check Package Valid" to verify if `Github_Stars` is greater than 0.

### Step 5: AI Agent Configuration
1. Create an **AI Agent** node.
2. Connect an **OpenAI Chat Model** (gpt-4o-mini) and **Google Gemini Chat Model**.
3. Add a **Structured Output Parser** connected to the agent with the detailed JSON schema (including `package_info`, `risk`, `decision`, and `slack_report`).
4. Add two **Firecrawl Tools** (`Search` and `Scrape`) to the agent.
5. Paste the comprehensive system prompt into the Agent's "System Message" field, emphasizing the separation of **Observed Facts** vs **Inferred Judgment**.

### Step 6: Output & Fallbacks
1. Connect the **True** path of the If node to the AI Agent, and the AI Agent to a **Slack** node.
2. Connect the **False** path of the If node to a **Set** node "Package Not Found Handler" containing the predefined "Avoid" report.
3. Connect the handler to a second **Slack** node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires Firecrawl API Key for dynamic URL discovery | Core Component |
| Requires GitHub OAuth2 for repository metrics | API Integration |
| AI Agent uses a multi-model approach (OpenAI + Gemini) for higher accuracy | AI Architecture |
| The "Observed Facts" vs "Inferred Judgment" split prevents AI hallucination | Prompt Engineering |