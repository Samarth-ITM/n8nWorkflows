Review GitHub pull requests with AI and log results to PostgreSQL and Slack

https://n8nworkflows.xyz/workflows/review-github-pull-requests-with-ai-and-log-results-to-postgresql-and-slack-13652


# Review GitHub pull requests with AI and log results to PostgreSQL and Slack

### 1. Workflow Overview

This workflow automates the software quality assurance process by intercepting GitHub Pull Requests (PRs), analyzing the code changes (diffs), and providing automated feedback. It aims to reduce manual review time by scoring PRs based on size and complexity, posting results directly back to GitHub, archiving data in a PostgreSQL database, and notifying the team via Slack.

The logic is organized into four main functional blocks:
1.  **Input & Data Retrieval:** Captures the webhook and fetches specific file changes.
2.  **Data Extraction & Analysis:** Transforms raw API data into structured metrics and calculates a preliminary review score.
3.  **Routing & Integration:** Determines the severity of issues and pushes the review results to GitHub and the database.
4.  **Notification & Logging:** Summarizes the process for the team and provides an execution log.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Acquisition
This block initiates the workflow and gathers all necessary context from GitHub.
*   **Nodes Involved:** `GitHub Webhook - PR Events`, `Fetch Changed Files in PR`, `Merge PR Details and Files`.
*   **Node Details:**
    *   **GitHub Webhook - PR Events:** Listens for `pull_request` events. Requires GitHub credentials.
    *   **Fetch Changed Files in PR:** A `GET` request to the GitHub API (`/pulls/{number}/files`) to retrieve the actual code diffs which are not fully present in the initial webhook.
    *   **Merge PR Details and Files:** Uses "Combine" mode to join the initial PR metadata (Author, Title) with the list of changed files.

#### 2.2 Data Preparation & Scoring
This block processes the raw data into a format suitable for reporting and applies scoring logic.
*   **Nodes Involved:** `Extract Code Diffs`, `Score Review & Categorize Issues`.
*   **Node Details:**
    *   **Extract Code Diffs (Code Node):** Iterates through files to calculate total additions, deletions, and changes. It creates a unique `reviewId` and flattens the file structure for easier consumption.
    *   **Score Review & Categorize Issues (Code Node):** Evaluates PR health. It penalizes scores for large PRs (e.g., >500 additions) and initializes issue categories (Critical, Major, Minor). 
    *   **Note:** This is the logical entry point for integrating LLMs (like OpenAI or Anthropic) to perform actual semantic code analysis.

#### 2.3 Routing & Action Execution
Handles the conditional logic and external writes.
*   **Nodes Involved:** `Route by Review Severity`, `Post Review to GitHub PR`, `Store Review Results in PostgreSQL`.
*   **Node Details:**
    *   **Route by Review Severity (Switch):** Filters PRs based on whether `issueCount.critical` is greater than 0.
    *   **Post Review to GitHub PR:** Uses a `POST` request to the GitHub reviews endpoint. It submits the final approval/comment status.
    *   **Store Review Results in PostgreSQL:** Inserts the `reviewId`, score, and metadata into the `code_reviews` table. Requires a predefined schema in Postgres.

#### 2.4 Notification & Finalization
Provides visibility into the automated process.
*   **Nodes Involved:** `Send Summary to Slack`, `Log Review Completion`.
*   **Node Details:**
    *   **Send Summary to Slack:** Sends a formatted Markdown block containing the PR title, author, and AI-calculated score to a Slack Webhook.
    *   **Log Review Completion:** A final script that outputs a success message to the n8n console for debugging and auditing purposes.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| GitHub Webhook - PR Events | GitHub Trigger | Trigger | None | Fetch Changed Files in PR | 1. Trigger PR Detection |
| Fetch Changed Files in PR | HTTP Request | Data Retrieval | GitHub Webhook | Merge PR Details and Files | 1. Trigger PR Detection |
| Merge PR Details and Files | Merge | Data Union | Fetch Changed Files, GitHub Webhook | Extract Code Diffs | 2. Fetch & Analyze Code |
| Extract Code Diffs | Code | Data Parsing | Merge PR Details and Files | Score Review & Categorize Issues | 2. Fetch & Analyze Code |
| Score Review & Categorize Issues | Code | Logic/Scoring | Extract Code Diffs | Route by Review Severity | 3. AI Review & Score |
| Route by Review Severity | Switch | Routing | Score Review & Categorize Issues | Post Review to GitHub PR, Store Review Results | 3. AI Review & Score |
| Post Review to GitHub PR | HTTP Request | Integration | Route by Review Severity | Send Summary to Slack | 4. Post Review & Notify |
| Store Review Results in PostgreSQL | Postgres | Storage | Route by Review Severity | Send Summary to Slack | 4. Post Review & Notify |
| Send Summary to Slack | HTTP Request | Notification | Post Review, Store Review | Log Review Completion | 4. Post Review & Notify |
| Log Review Completion | Code | Logging | Send Summary to Slack | None | 4. Post Review & Notify |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Create a **GitHub Trigger** node. Select `pull_request` as the event. Connect your GitHub OAuth2 credentials.
2.  **Fetch Files:**
    *   Add an **HTTP Request** node. Method: `GET`. 
    *   URL: `https://api.github.com/repos/{{ $json.repository.owner.login }}/{{ $json.repository.name }}/pulls/{{ $json.pull_request.number }}/files`.
3.  **Data Merging:**
    *   Add a **Merge** node. Mode: `Combine`. Connect Input 1 from the Trigger and Input 2 from the HTTP Request.
4.  **Transformation (Code):**
    *   Add a **Code** node ("Extract Code Diffs"). Use JavaScript to map over the `changedFiles` array and calculate `totalAdditions` and `totalDeletions`.
5.  **Scoring Logic:**
    *   Add a second **Code** node ("Score Review"). Implement logic to set a base score (e.g., 100) and subtract points based on the number of files or lines changed.
6.  **Conditional Routing:**
    *   Add a **Switch** node. Rule 1: If `issueCount.critical` > 0 -> Output 1. Rule 2: If `issueCount.critical` <= 0 -> Output 2.
7.  **GitHub Feedback:**
    *   Add an **HTTP Request** node. Method: `POST`. 
    *   URL: `https://api.github.com/repos/{{ $json.repoOwner }}/{{ $json.repoName }}/pulls/{{ $json.prNumber }}/reviews`.
8.  **Database Logging:**
    *   Add a **PostgreSQL** node. Action: `Insert`. Table: `code_reviews`. Map the columns to the properties created in the "Score Review" node.
9.  **Slack Notification:**
    *   Add an **HTTP Request** node. Method: `POST`. URL: Your Slack Webhook URL. 
    *   Body: Create a JSON object with a "text" field summarizing the PR and the score.
10. **Final Console Log:**
    *   Add a final **Code** node. Use `console.log()` to print the `reviewId` and `prNumber`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Setup Step: PostgreSQL** | Ensure the table `code_reviews` exists with columns matching the Review ID, Score, and PR Number. |
| **Setup Step: AI Integration** | To upgrade this to full AI analysis, replace the "Score Review" code with an OpenAI/Claude node. |
| **Webhook Configuration** | The GitHub Webhook must be set to `application/json` in the GitHub repository settings. |
| **Performance Note** | For very large PRs, the GitHub API may paginate file results; this workflow assumes small to medium PRs. |