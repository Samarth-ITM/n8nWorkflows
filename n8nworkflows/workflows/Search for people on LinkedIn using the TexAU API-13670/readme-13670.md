Search for people on LinkedIn using the TexAU API

https://n8nworkflows.xyz/workflows/search-for-people-on-linkedin-using-the-texau-api-13670


# Search for people on LinkedIn using the TexAU API

This document provides a comprehensive technical analysis of the **Linkedin People Search** n8n workflow, which automates LinkedIn lead discovery using the TexAU API.

---

### 1. Workflow Overview

The primary purpose of this workflow is to automate the extraction of LinkedIn profile data based on a search URL. It acts as a bridge between an input trigger (like an AI agent or a CRM) and the TexAU automation platform.

**Logic Blocks:**
*   **1.1 Input Reception:** Receives the LinkedIn search parameters via a sub-workflow trigger.
*   **1.2 Automation Execution:** Sends a request to TexAU to start a "LinkedIn People Search" job.
*   **1.3 Job Management:** Tracks the execution status and introduces a cooling period for data processing.
*   **1.4 Data Retrieval:** Fetches the final structured results (profiles) from the TexAU cloud.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
*   **Overview:** This block prepares the workflow to be used as a tool or sub-process by other workflows, accepting a search description or URL.
*   **Nodes Involved:** `When Executed by Another Workflow`.
*   **Node Details:**
    *   **Type:** Execute Workflow Trigger.
    *   **Configuration:** Defines a required input variable named `Description`.
    *   **Input/Output:** Receives external data; outputs a JSON object containing the `Description` string.

#### 2.2 Automation Execution
*   **Overview:** Triggers the specific TexAU automation.
*   **Nodes Involved:** `LinkedIn_People_Search`.
*   **Node Details:**
    *   **Type:** HTTP Request (POST).
    *   **Technical Role:** API Dispatcher.
    *   **Configuration:** Calls `https://api.texau.com/api/v1/public/run`. It passes the `automationId` for LinkedIn People Search and a `connectedAccountId`.
    *   **Key Expressions:** `liPeopleSearchUrl: {{ $json.Description }}`.
    *   **Edge Cases:** Invalid LinkedIn URL, expired TexAU session, or reached API rate limits.

#### 2.3 Job Management
*   **Overview:** Handles the asynchronous nature of TexAU by checking the job ID and waiting for completion.
*   **Nodes Involved:** `Get Results`, `Wait`.
*   **Node Details:**
    *   **Get Results (HTTP Request):** Performs a GET request to `/executions/{{ $json.data.id }}` to confirm the job has been registered in the TexAU system.
    *   **Wait:** A 60-second pause.
    *   **Purpose:** Ensures TexAU has sufficient time to scrape LinkedIn before the workflow attempts to download the result file.

#### 2.4 Data Retrieval
*   **Overview:** Fetches the final dataset produced by the automation.
*   **Nodes Involved:** `People_Search_Results`.
*   **Node Details:**
    *   **Type:** HTTP Request (GET).
    *   **Configuration:** Calls `https://api.texau.com/api/v1/public/results/{{ $json.data.id }}`.
    *   **Output:** A structured list of LinkedIn profiles (Name, Title, URL, etc.).
    *   **Potential Failures:** If the wait time was too short, this node might return an empty set or a "processing" status instead of the final data.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Executed by Another Workflow | Execute Workflow Trigger | Sub-workflow Entry | (None) | LinkedIn_People_Search | (Describes general workflow logic and setup steps) |
| LinkedIn_People_Search | HTTP Request | Job Trigger | When Executed by Another Workflow | Get Results | (Describes general workflow logic and setup steps) |
| Get Results | HTTP Request | Status Check | LinkedIn_People_Search | Wait | (Describes general workflow logic and setup steps) |
| Wait | Wait | Delay (60s) | Get Results | People_Search_Results | (Describes general workflow logic and setup steps) |
| People_Search_Results | HTTP Request | Data Extraction | Wait | (None) | (Describes general workflow logic and setup steps) |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to recreate the LinkedIn Search automation:

1.  **Trigger Setup:**
    *   Create a **Execute Workflow Trigger** node.
    *   Add a variable named `Description` (which will hold the LinkedIn Search URL).

2.  **API Authorization:**
    *   All HTTP Request nodes in this workflow require the following headers:
        *   `Authorization`: `Bearer [Your_TexAU_API_Key]`
        *   `X-TexAu-Context`: A JSON string containing your `orgUserId` and `workspaceId`.

3.  **Trigger the Search (HTTP Node):**
    *   Add an **HTTP Request** node named `LinkedIn_People_Search`.
    *   **Method:** `POST`.
    *   **URL:** `https://api.texau.com/api/v1/public/run`.
    *   **Body (JSON):** Set `automationId` to `63f5eaad7022e05c1180244a`. Map `liPeopleSearchUrl` to `{{ $json.Description }}`.

4.  **Acknowledge Execution (HTTP Node):**
    *   Add an **HTTP Request** node named `Get Results`.
    *   **Method:** `GET`.
    *   **URL:** `https://api.texau.com/api/v1/public/executions/{{ $json.data.id }}`.

5.  **Cooling Period:**
    *   Add a **Wait** node.
    *   Set the wait time to **60 seconds**. (Adjust higher for searches involving 10+ profiles).

6.  **Final Fetch (HTTP Node):**
    *   Add an **HTTP Request** node named `People_Search_Results`.
    *   **Method:** `GET`.
    *   **URL:** `https://api.texau.com/api/v1/public/results/{{ $json.data.id }}`.

7.  **Connectivity:**
    *   Connect the nodes linearly: Trigger -> Start Search -> Get Execution -> Wait -> Final Results.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **TexAU Documentation** | Refer to [TexAU API Docs](https://texau.com/api) for updating `automationId` if the LinkedIn Search template changes. |
| **Rate Limiting** | LinkedIn scraping is sensitive; ensure the `connectedAccountId` in the first HTTP node refers to a TexAU proxy/account with valid cookies. |
| **Max Count** | The current configuration is hardcoded to return 5 results (`"maxCountPeopleSearch": 5`). |