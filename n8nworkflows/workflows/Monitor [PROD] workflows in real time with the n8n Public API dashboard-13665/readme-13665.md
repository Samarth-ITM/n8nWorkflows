Monitor [PROD] workflows in real time with the n8n Public API dashboard

https://n8nworkflows.xyz/workflows/monitor--prod--workflows-in-real-time-with-the-n8n-public-api-dashboard-13665


# Monitor [PROD] workflows in real time with the n8n Public API dashboard

# Workflow Reference: n8n Resiliency Dashboard

### 1. Workflow Overview
This workflow generates a real-time, auto-refreshing HTML dashboard to monitor the health of specific production workflows. It identifies workflows tagged with `[PROD]`, retrieves their recent execution history (successes vs. errors) via the n8n Public API, calculates performance metrics, and serves the data through a professional web interface.

**Logical Blocks:**
- **1.1 Input Reception:** Triggered via a Webhook to serve the dashboard on demand.
- **1.2 Configuration & Discovery:** Sets the instance URL and fetches all workflows, filtering for those with the `[PROD]` tag.
- **1.3 Execution Data Retrieval:** Queries the n8n API for the last 50 successful and 50 failed executions for each identified workflow.
- **1.4 Data Processing & Visualization:** Merges the data, calculates success rates, generates a 7-day error trend, and constructs the final HTML/CSS/JS dashboard.
- **1.5 Response Delivery:** Sends the rendered HTML back to the browser.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration & Discovery
This block identifies which workflows need monitoring based on your tagging system.
- **Nodes Involved:** `Webhook`, `Config`, `API: List Workflows`, `Filter: [PROD] Tag`.
- **Node Details:**
    - **Webhook:** Entry point (`GET` method). Set to "Respond to Webhook" mode to wait for the final HTML.
    - **Config (Set):** Defines the `base_url` variable (e.g., `https://your-n8n-instance.com`). This is used in all subsequent API calls.
    - **API: List Workflows (HTTP Request):** Calls `/api/v1/workflows` using n8n API credentials.
    - **Filter: [PROD] Tag (Code):** A JavaScript snippet that iterates through the list of workflows and only passes those containing a tag named exactly `[PROD]`.

#### 2.2 Execution Data Retrieval
Fetches the raw data needed to calculate health statistics.
- **Nodes Involved:** `GET success`, `GET error`.
- **Node Details:**
    - **GET success (HTTP Request):** Queries `/api/v1/executions` with query parameters `status=success`, `limit=50`, and the specific `workflowId`.
    - **GET error (HTTP Request):** Similar to the success node but filters for `status=error`.
    - **Connections:** These run in parallel for every workflow passed from the previous block.
    - **Failure handling:** Standard API timeouts or credential issues will prevent data from loading.

#### 2.3 Data Processing & Visualization
Aggregates the execution data and builds the UI.
- **Nodes Involved:** `Merge`, `Compare`, `HTML`.
- **Node Details:**
    - **Merge:** Combines the success and error execution arrays by position.
    - **Compare (Code):** Calculates the success rate percentage, identifies the "Healthy" status (no errors in the last 50 runs), and groups errors by date for the 7-day chart.
    - **HTML (Code):** Generates a complete HTML document including:
        - **CSS:** A dark-themed, modern UI using "Syne" and "DM Sans" fonts.
        - **Chart.js:** Logic to render the error bar chart.
        - **Meta Refresh:** Set to 30 seconds to keep the dashboard live.

#### 2.4 Response Delivery
- **Nodes Involved:** `Respond to Webhook`.
- **Node Details:**
    - **Configuration:** Set to `Content-Type: text/html; charset=utf-8`.
    - **Output:** Returns the `html` string generated in the previous step to the requester's browser.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Webhook** | Webhook | Trigger | (None) | Config | (None) |
| **Config** | Set | Global Vars | Webhook | API: List Workflows | Step 1 — Config: Set your n8n base URL here. |
| **API: List Workflows** | HTTP Request | Data Fetching | Config | Filter: [PROD] Tag | Step 2 — Fetch & Filter: Fetches all workflows via n8n API. |
| **Filter: [PROD] Tag** | Code | Data Filtering | API: List Workflows | GET error, GET success | Step 2 — Fetch & Filter: Filters only those tagged [PROD]. |
| **GET success** | HTTP Request | Data Fetching | Filter: [PROD] Tag | Merge | Step 3 — Fetch Executions: Fetches last 50 successes. |
| **GET error** | HTTP Request | Data Fetching | Filter: [PROD] Tag | Merge | Step 3 — Fetch Executions: Fetches last 50 errors. |
| **Merge** | Merge | Data Joining | GET success, GET error | Compare | Step 4 — Compare & Render: Merges success + error executions. |
| **Compare** | Code | Data Processing | Merge | HTML | Step 4 — Compare & Render: Computes stats and 7-day chart. |
| **HTML** | Code | UI Rendering | Compare | Respond to Webhook | Step 4 — Compare & Render: Renders the full HTML dashboard. |
| **Respond to Webhook**| Respond to Webhook | Output | HTML | (None) | (None) |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Go to **Settings > n8n API** and generate an API Key.
    *   Create a new Credential in n8n of type **n8n API** and paste your key.
    *   Tag at least one workflow with the tag `[PROD]`.

2.  **Trigger & Config:**
    *   Add a **Webhook** node. Set Method to `GET` and Path to `resiliency-dashboard`. Set HTTP Response to "When Last Node Finishes".
    *   Add a **Set** node ("Config"). Create a string variable `base_url` containing your n8n instance URL (e.g., `https://n8n.example.com`).

3.  **Fetch Workflows:**
    *   Add an **HTTP Request** node ("API: List Workflows"). 
    *   URL: `{{ $json.base_url }}/api/v1/workflows`. 
    *   Authentication: `Predefined Credential Type` > `n8nApi`.
    *   Add a **Code** node ("Filter: [PROD] Tag"). Use JS to filter the `data` array for workflows where `tags` contains an object with `name: "[PROD]"`.

4.  **Fetch Executions:**
    *   Add an **HTTP Request** node ("GET success"). URL: `{{ $nodes.Config.json.base_url }}/api/v1/executions?status=success&limit=50&workflowId={{ $json.id }}`.
    *   Add an **HTTP Request** node ("GET error"). URL: `{{ $nodes.Config.json.base_url }}/api/v1/executions?status=error&limit=50&workflowId={{ $json.id }}`.
    *   Connect the Filter node to **both** request nodes.

5.  **Analytics & UI:**
    *   Add a **Merge** node. Mode: `Combine`, Property: `Position`.
    *   Add a **Code** node ("Compare"). Write JS to calculate: `totalSuccess / (totalSuccess + totalErrors)` for each ID and aggregate errors per day for the last 7 days.
    *   Add a **Code** node ("HTML"). Create a template literal containing the HTML/CSS and inject the variables from the "Compare" node. Use `<meta http-equiv="refresh" content="30">` in the head for auto-update.

6.  **Response:**
    *   Add a **Respond to Webhook** node. 
    *   Response Body: `{{ $json.html }}`. 
    *   Header: `Content-Type: text/html`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Live Update** | The dashboard refreshes every 30 seconds automatically via HTML meta tags. |
| **API Documentation** | Uses the n8n Public API: [https://docs.n8n.io/api/](https://docs.n8n.io/api/) |
| **Tagging Requirement** | Only workflows with the exact tag `[PROD]` will appear in the list. |
| **Security Note** | The Webhook URL is public; consider adding a basic auth header if sensitive data is displayed. |