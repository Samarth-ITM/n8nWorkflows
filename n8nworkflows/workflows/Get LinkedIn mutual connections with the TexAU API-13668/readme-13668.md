Get LinkedIn mutual connections with the TexAU API

https://n8nworkflows.xyz/workflows/get-linkedin-mutual-connections-with-the-texau-api-13668


# Get LinkedIn mutual connections with the TexAU API

This document provides a detailed technical analysis and reproduction guide for the **LinkedIn Mutual Connections** workflow.

### 1. Workflow Overview
The purpose of this workflow is to automate the retrieval of shared (mutual) connections between the user and a target LinkedIn profile using the **TexAU API**. This is used for lead qualification, relationship mapping, and social selling by identifying common ground with prospects.

The workflow is organized into four logical phases:
1.  **Automation Initiation:** Sends a request to TexAU to start the "LinkedIn Mutual Connections" automation.
2.  **Execution Tracking:** Queries the specific execution ID to confirm the task is registered.
3.  **Processing Delay:** Pauses the workflow to allow TexAU's cloud browsers to scrape the data.
4.  **Data Retrieval:** Fetches the final structured results containing names, headlines, and profile URLs of mutual connections.

---

### 2. Block-by-Block Analysis

#### 2.1 Automation Initiation
This block starts the external process on TexAU's servers.

*   **Nodes Involved:** `When clicking ‘Test workflow’`, `LinkedIn_Mutual_Connections`.
*   **Node Details:**
    *   **Manual Trigger:** Entry point for testing.
    *   **LinkedIn_Mutual_Connections (HTTP Request):**
        *   **Method:** POST
        *   **URL:** `https://api.texau.com/api/v1/public/run/682e2fbb9b7201d753f179bf`
        *   **Configuration:** Sends a JSON body containing the `workflowId`, `connectedAccountId` (LinkedIn cookie session), and the `liProfileUrl`. 
        *   **Key Expression:** `{{ $json.Linkedin_url }}` is expected as input (though the manual trigger should be configured with this field or preceded by a 'Set' node).
        *   **Headers:** Requires `Authorization` (Bearer token) and `X-TexAu-Context` (Org and Workspace IDs).
        *   **Failure Modes:** 401 Unauthorized (Expired TexAU token), 400 Bad Request (Invalid LinkedIn URL).

#### 2.2 Execution & Wait Logic
This block ensures the workflow tracks the specific job and waits for completion.

*   **Nodes Involved:** `Get Results`, `Wait`.
*   **Node Details:**
    *   **Get Results (HTTP Request):**
        *   **Method:** GET
        *   **URL:** `https://api.texau.com/api/v1/public/executions/{{ $json.data.id }}`
        *   **Role:** Verifies that the execution started in the previous step is active and retrieves metadata.
    *   **Wait (Wait Node):**
        *   **Duration:** 60 Seconds.
        *   **Role:** Since scraping LinkedIn is a synchronous-simulated browser task, a delay is necessary to prevent the final request from returning an empty result.

#### 2.3 Data Retrieval
The final phase extracts the actual list of mutual connections.

*   **Nodes Involved:** `Mutual_Connections_Results`.
*   **Node Details:**
    *   **Mutual_Connections_Results (HTTP Request):**
        *   **Method:** GET
        *   **URL:** `https://api.texau.com/api/v1/public/results/{{ $json.data.id }}`
        *   **Role:** Final data extraction.
        *   **Output:** Returns an array of objects including `fullName`, `headline`, `liMemberUrn`, `liProfilePublicUrl`, and `mutualConnectionsCount`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Test workflow’ | Manual Trigger | Entry Point | None | LinkedIn_Mutual_Connections | |
| LinkedIn_Mutual_Connections | HTTP Request | Run TexAU Task | Manual Trigger | Get Results | ## LinkedIn Mutual Connections via TexAU... (See logic description) |
| Get Results | HTTP Request | Check Execution | LinkedIn_Mutual_Connections | Wait | ## LinkedIn Mutual Connections via TexAU... |
| Wait | Wait | Processing Delay | Get Results | Mutual_Connections_Results | ## LinkedIn Mutual Connections via TexAU... |
| Mutual_Connections_Results | HTTP Request | Fetch Final Data | Wait | None | ## LinkedIn Mutual Connections via TexAU... |
| Mutual_Connections_Results1 | HTTP Request | Orphaned Node | None | None | |
| Sticky Note7 | Sticky Note | Documentation | None | None | Detailed setup steps and how it works. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** 
    *   Add a **Manual Trigger** node. 
    *   *Optional:* Add an input field named `Linkedin_url` to the trigger for ease of testing.
2.  **Start TexAU Automation:**
    *   Add an **HTTP Request** node named `LinkedIn_Mutual_Connections`.
    *   Set Method to `POST`.
    *   Set URL to: `https://api.texau.com/api/v1/public/run/682e2fbb9b7201d753f179bf`.
    *   In the **Body Parameters**, select `JSON` and paste the structure identifying your `workflowId` and `connectedAccountId`. Use `{{ $json.Linkedin_url }}` for the profile URL.
    *   In **Headers**, add `Authorization: Bearer [Your_Token]` and `X-TexAu-Context`.
3.  **Execution Tracking:**
    *   Add an **HTTP Request** node named `Get Results`.
    *   Set Method to `GET`.
    *   Set URL to: `https://api.texau.com/api/v1/public/executions/{{ $json.data.id }}`.
    *   Copy the same Headers from the previous step.
4.  **Timing:**
    *   Add a **Wait** node. Set the amount to `60` and the unit to `Seconds`.
5.  **Final Extraction:**
    *   Add an **HTTP Request** node named `Mutual_Connections_Results`.
    *   Set Method to `GET`.
    *   Set URL to: `https://api.texau.com/api/v1/public/results/{{ $json.data.id }}`.
    *   Ensure Headers (Auth and Context) are present.
6.  **Connection:**
    *   Connect the nodes in linear order: Trigger → Run → Get Status → Wait → Fetch Results.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **TexAU API Documentation** | Used for finding `workflowId` and `automationId` |
| **LinkedIn Cookie Policy** | TexAU requires a valid LinkedIn session cookie (`li_at`) via the `connectedAccountId`. |
| **Execution Time** | If the list of mutual connections is >100, increase the **Wait** node duration to 120 seconds. |
| **Rate Limiting** | Ensure your TexAU plan allows for the frequency of requests triggered by this workflow. |