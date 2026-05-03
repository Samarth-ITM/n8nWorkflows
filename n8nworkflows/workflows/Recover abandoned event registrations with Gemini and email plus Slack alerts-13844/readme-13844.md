Recover abandoned event registrations with Gemini and email plus Slack alerts

https://n8nworkflows.xyz/workflows/recover-abandoned-event-registrations-with-gemini-and-email-plus-slack-alerts-13844


# Recover abandoned event registrations with Gemini and email plus Slack alerts

This document provides a technical overview and implementation guide for the **Abandoned Cart Recovery for Event Registration** workflow.

---

### 1. Workflow Overview
The purpose of this workflow is to automatically identify users who started an event registration process but failed to complete it. It implements a multi-stage remarketing strategy (1-hour, 24-hour, and 72-hour windows) to increase conversion rates by sending personalized emails with increasing urgency.

**Logical Blocks:**
*   **1.1 Data Retrieval:** Periodic trigger and API call to fetch incomplete registration records.
*   **1.2 Logic Processing:** Calculation of time elapsed and determination of the specific recovery stage.
*   **1.3 Communication:** Routing of users to specific email templates based on their elapsed time.
*   **1.4 Logging:** Recording successful recovery attempts to a data table to prevent duplicate communications.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Retrieval & Input
*   **Overview:** Triggers the automation and fetches the raw data of users who have "started" but not "finished" registration.
*   **Nodes Involved:** `Run Every 4 Hours`, `Query Incomplete Registrations`.
*   **Node Details:**
    *   **Run Every 4 Hours (Schedule Trigger):** Fires every 4 hours. This frequency ensures that users in the 1h, 24h, or 72h windows are captured relatively close to their target times.
    *   **Query Incomplete Registrations (HTTP Request):** 
        *   **Role:** Calls the n8n Data Table API.
        *   **Config:** Method `GET`, URL with placeholder `PLACEHOLDER_REGISTRATIONS_TABLE_ID`.
        *   **Filters:** Uses query parameters `limit=100` and `filter=status=started`.
        *   **Failure Mode:** `Continue on Fail` is enabled to prevent the workflow from crashing if the API is temporarily unreachable.

#### 2.2 Logic & Stage Calculation
*   **Overview:** Filters out invalid records (e.g., no email) and maps users to specific "recovery stages" based on the age of their registration entry.
*   **Nodes Involved:** `Calculate Recovery Stage`, `Has Carts to Recover?`.
*   **Node Details:**
    *   **Calculate Recovery Stage (Code):**
        *   **Logic:** Compares `created_at` timestamp with current time.
        *   **Stages Defined:** 1h-6h (Stage: 1h), 24h-48h (Stage: 24h), 72h-96h (Stage: 72h).
        *   **Validation:** Skips entries without an email address or those already marked as "sent" in the `recovery_stages_sent` field.
    *   **Has Carts to Recover? (If):** Checks if the code node returned actual records or an "empty" message. Prevents downstream nodes from running unnecessarily.

#### 2.3 Communication Layer
*   **Overview:** Routes the user to the appropriate email content based on the stage determined in the previous block.
*   **Nodes Involved:** `Route by Recovery Stage`, `Send Nudge Email (1h)`, `Send Social Proof Email (24h)`, `Send Last Chance Email (72h)`.
*   **Node Details:**
    *   **Route by Recovery Stage (Switch):** Directs the flow into one of three branches based on the value of `{{ $json.recovery_stage }}`.
    *   **Email Nodes (EmailSend):**
        *   **Subject Lines:** Dynamic subjects such as "Almost done!", "Don't miss out", or "Last chance".
        *   **Configuration:** Requires SMTP/Mail credentials. Requires manual update of `PLACEHOLDER_REG_URL` in the body (if applicable).

#### 2.4 Recovery Logging
*   **Overview:** Formats the event data and saves it back to a tracking table.
*   **Nodes Involved:** `Log Recovery Event`, `Log to Cart Recovery Table`.
*   **Node Details:**
    *   **Log Recovery Event (Code):** Simplifies the object structure to prepare for table insertion, including `reg_id`, `stage`, `email`, and a `sent_at` timestamp.
    *   **Log to Cart Recovery Table (Data Table):** Appends the row to the tracking table (`PLACEHOLDER_CART_RECOVERY_TABLE_ID`) to ensure the user doesn't receive the same stage email twice in the next cycle.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Every 4 Hours | Schedule Trigger | Periodic Trigger | None | Query Incomplete Reg. | Detects incomplete registrations and sends a 3-stage recovery sequence. |
| Query Incomplete Reg. | HTTP Request | Data Fetching | Run Every 4 Hours | Calculate Recovery Stage | Detects incomplete registrations and sends a 3-stage recovery sequence. |
| Calculate Recovery Stage | Code | Business Logic | Query Incomplete Reg. | Has Carts to Recover? | Detects incomplete registrations and sends a 3-stage recovery sequence. |
| Has Carts to Recover? | If | Flow Control | Calculate Recovery Stage | Route by Recovery Stage | Detects incomplete registrations and sends a 3-stage recovery sequence. |
| Route by Recovery Stage | Switch | Routing | Has Carts to Recover? | Email Nodes (1h, 24h, 72h) | **1h — Nudge:** Friendly reminder... **24h — Social Proof:**... **72h — Last Chance:**... |
| Send Nudge Email (1h) | Email Send | Communication | Route by Stage | Log Recovery Event | **1h — Nudge:** Friendly reminder, takes 30 seconds to finish. |
| Send Social Proof Email (24h) | Email Send | Communication | Route by Stage | Log Recovery Event | **24h — Social Proof:** "47 more professionals registered since you left". |
| Send Last Chance Email (72h) | Email Send | Communication | Route by Stage | Log Recovery Event | **72h — Last Chance:** Price increase urgency, scarcity messaging. |
| Log Recovery Event | Code | Data Formatting | Email Nodes | Log to Table | Detects incomplete registrations and sends a 3-stage recovery sequence. |
| Log to Cart Recovery Table | Data Table | Persistence | Log Recovery Event | None | Detects incomplete registrations and sends a 3-stage recovery sequence. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Add a **Schedule Trigger** node. Set it to run every 4 hours.
2.  **Data Fetching:** Add an **HTTP Request** node.
    *   **URL:** Your n8n table API endpoint (`/api/v1/data-tables/[ID]/rows`).
    *   **Method:** GET.
    *   **Auth:** Header Auth (n8n API Key).
    *   **Query Params:** `filter`: `status=started`.
3.  **Elapsed Time Logic:** Add a **Code** node. Use JavaScript to:
    *   Get the current time.
    *   Calculate `(Now - created_at)` in hours.
    *   Assign stage labels: '1h', '24h', or '72h' based on defined ranges.
    *   Filter out items where the stage has already been sent (check `recovery_stages_sent`).
4.  **Empty Check:** Add an **If** node to check if the Code node returned any valid items. If "empty", stop.
5.  **Branching:** Add a **Switch** node. Set the "Data Type" to String. Create three rules for "1h", "24h", and "72h".
6.  **Templates:** Add three **Email Send** nodes. 
    *   Connect one to each Switch output.
    *   Configure SMTP/Gmail/Outlook credentials.
    *   Include a "Finish Registration" link in the body pointing to your event URL.
7.  **Log Preparation:** Add a **Code** node after the Email nodes. Map the relevant fields (`reg_id`, `email`, `stage`) into a flat JSON object.
8.  **Storage:** Add an **n8n Data Table** node. Select the "Append" operation and choose your "Recovery Logs" table ID.
9.  **Connections:** Ensure all nodes are linked as per the logic flow: Trigger -> HTTP -> Code -> If -> Switch -> Emails -> Code -> Data Table.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Strategy Source** | Based on marketing principles from Julius: "No urgency, no sale". |
| **Feedback & Consulting** | For custom builds or feedback: [Contact Milo Bravo](https://tally.so/r/EkKGgB) |
| **LinkedIn Profile** | Connect with the creator: [Milo Bravo on LinkedIn](https://linkedin.com/in/MiloBravo/) |
| **Setup Requirement** | Replace all `PLACEHOLDER` strings in node parameters before activation. |