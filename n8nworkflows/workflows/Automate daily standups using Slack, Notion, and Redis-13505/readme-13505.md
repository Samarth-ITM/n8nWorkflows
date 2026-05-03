Automate daily standups using Slack, Notion, and Redis

https://n8nworkflows.xyz/workflows/automate-daily-standups-using-slack--notion--and-redis-13505


# Automate daily standups using Slack, Notion, and Redis

This document provides a technical analysis of the **Automated Daily Standup System**, an n8n workflow designed to manage team communication via Slack, persist data in Notion, and track real-time session states using Redis.

---

### 1. Workflow Overview

The workflow automates two daily touchpoints—a morning standup and an evening check-in—for distributed teams. It manages the full lifecycle of these interactions: identifying active members, initiating private conversations, tracking response progress, and generating aggregated reports for management.

#### Logical Blocks:
*   **1.1 Morning Initiator:** Triggers at 9:40 AM IST. Iterates through active members to start a 4-question standup session.
*   **1.2 Morning Summary:** Triggers at 10:10 AM IST. Compiles morning responses and blockers into a Slack report.
*   **1.3 Evening Initiator:** Triggers at 6:00 PM IST. Starts a 3-question completion check-in.
*   **1.4 Evening Summary:** Triggers at 6:30 PM IST. Compiles task completion stats and help requests for the end of the day.

---

### 2. Block-by-Block Analysis

#### 2.1 Morning & Evening Initiators (Logic Flow)
**Overview:** These blocks fetch active team members from Notion and initiate a 1-on-1 Slack DM conversation. They use Redis to ensure a user isn't prompted multiple times if a session is already active.

*   **Nodes Involved:** `Schedule Trigger`, `Notion (Fetch Members)`, `Split In Batches`, `Code (Validation)`, `Redis (GET)`, `Notion (Create Session)`, `Slack (Send Message)`.
*   **Node Details:**
    *   **Notion (Get active members):** Filters the `Team_Members` database for `Active_Status = Active` and `Daily_Standups = true`.
    *   **Code (Extract & Validate):** Normalizes Notion properties (handling various naming conventions like `property_full_name`). It checks for the existence of `slack_user_id` and `developer_id`.
    *   **Redis (Check session):** Uses keys like `morning:standup:session:{slack_user_id}`. If the key exists, the flow for that user is skipped.
    *   **Notion (Create Session):** Creates a record in `Conversation_State` (Morning) or `Conversation_State_Afternoon` (Evening) with `Status = In Progress`.
    *   **Slack (Send message):** Sends the first question of the standup. 
    *   **Edge Cases:** If Redis is down, `continueOnFail` is enabled on several nodes to prevent the entire loop from breaking. Missing Slack IDs result in a silent skip for that specific user.

#### 2.2 Morning & Evening Summaries (Reporting Flow)
**Overview:** These blocks aggregate data from the standup response databases and post a formatted report to a designated admin channel.

*   **Nodes Involved:** `Schedule Trigger`, `Notion (Fetch Members)`, `Notion (Fetch Standups)`, `Merge`, `Code (Build Message)`, `Slack (Send)`.
*   **Node Details:**
    *   **Notion (Fetch Standups):** Queries for records where `Date` is today and `Responded = true`.
    *   **Merge:** Combines the list of "All Active Members" with "Today's Responses" to identify who is missing (Not Attended).
    *   **Code (Build Summary):** A complex JavaScript node that calculates:
        *   Attendance percentage.
        *   Identification of "Blockers" using keyword filtering (e.g., filtering out "none", "n/a", "no").
        *   Task completion counts (Yes vs. No/Pending).
    *   **Slack (Send to Admin):** Posts a heavily formatted message using Slack's Block Kit or Markdown to channel `C07PSV73SK0`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Morning initiator trigger | ScheduleTrigger | Cron (9:40 AM IST) | None | Get active team members | Fires Mon–Sat at 9:40 AM IST. |
| Get active team members | Notion | Fetch Data | Morning initiator trigger | Check if members found | Fetches active members from `Team_Members`. |
| Developer loop | SplitInBatches | Iterator | Check if members found | Extract and validate | Iterates one member at a time. |
| Check existing Redis session | Redis | Session Check | Is valid member | Parse session state | GETs existing session key. |
| Create Notion session | Notion | Persistence | Needs new session | Prepare session data | Creates a `Conversation_State` record. |
| Send standup message | Slack | Communication | Save session to Redis | Update Slack channel ID | Sends Question 1 via Slack DM. |
| Evening schedule trigger | ScheduleTrigger | Cron (6:30 PM IST) | None | Fetch Active Members | Fires Mon–Sat at 6:30 PM IST. |
| Build evening summary | Code | Data Logic | Merge Data | Send to Admin Channel | Merges datasets, calculates response rate. |
| Fetch Evening Standups | Notion | Fetch Data | Evening schedule trigger | Merge Data | Fetches today's evening standups. |
| Store evening session | Redis | Caching | Evening session ready | Send evening message | SETs key with 24h TTL. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Notion Setup
*   Create databases: `Team_Members`, `Daily_Standups`, `Daily_Standups_Afternoon`, `Conversation_State`, `Conversation_State_Afternoon`.
*   **Team_Members** must have: `Full_Name` (Title), `Slack_User_ID` (Text), `Developer_ID` (Text), `Active_Status` (Status), `Daily_Standups` (Checkbox).

#### Step 2: Redis Setup
*   Ensure a Redis instance is available.
*   Configure the Redis Node with host, port, and password.
*   **TTL Strategy:** Use 86400 (24 hours) for morning/evening keys to prevent duplicate daily prompts.

#### Step 3: Slack Bot Setup
*   Create a Slack App with `chat:write`, `im:write`, and `users:read` scopes.
*   Invite the Bot to the admin channel (e.g., `C07PSV73SK0`).

#### Step 4: Logic Configuration
1.  **Triggers:** Set Schedule Triggers to "Cron" with your team's local timezone (the JSON uses `Asia/Kolkata`).
2.  **Loops:** Use **Split In Batches** (batch size 1) to process users. This avoids hitting Notion/Slack rate limits.
3.  **Data Normalization:** Use a **Code Node** after fetching Notion data. Notion JSON can be deeply nested; map them to flat objects (e.g., `item.json.property_slack_user_id` -> `slack_user_id`).
4.  **Convergence:** Ensure all paths (including "Skip" or "False" branches) lead back to the **Split In Batches** node to continue the loop.

#### Step 5: Sub-workflow logic (Response Handling)
*   *Note: This specific JSON covers the Initiators and Summaries. A separate Slack Webhook workflow is required to handle the "incoming message" events from Slack and update the Notion/Redis state as users answer questions.*

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Timezone Alignment | The workflow is strictly configured for **Asia/Kolkata (IST)**. Cron expressions should be adjusted if used in other regions. |
| Session Caching | Morning TTL: 8h / Evening TTL: 3h as per logic description, though Redis nodes in JSON show 86400s (24h). |
| Data Persistence | Redis is used for fast state lookups; Notion is the permanent record for analytics. |
| Error Handling | Most Slack and Redis nodes have `continueOnFail: true` to prevent a single user's error from stopping the daily standup for the whole team. |