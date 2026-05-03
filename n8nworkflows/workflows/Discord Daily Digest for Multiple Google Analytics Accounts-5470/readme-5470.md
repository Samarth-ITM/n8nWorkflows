Discord Daily Digest for Multiple Google Analytics Accounts

https://n8nworkflows.xyz/workflows/discord-daily-digest-for-multiple-google-analytics-accounts-5470


# Discord Daily Digest for Multiple Google Analytics Accounts

### 1. Workflow Overview

This workflow automates the daily delivery of Google Analytics data summaries to multiple Discord channels, each associated with a different Google Analytics property. It targets users managing multiple GA accounts and wanting timely analytics insights posted or updated in Discord channels as daily digests.

The workflow logically splits into these blocks:

- **1.1 Scheduled Triggers for Multiple Accounts**: Four scheduled triggers start the flow daily at staggered times, each corresponding to one Google Analytics property and its linked Discord channel.

- **1.2 Preparation of Input Data**: For each trigger, "Edit Fields" nodes prepare the identifiers (Google Analytics Property ID and Discord Channel ID) for the respective account.

- **1.3 Google Analytics Data Retrieval**: The workflow queries Google Analytics API for specific metrics of the day using the configured GA property ID.

- **1.4 Discord Messages Retrieval and Sorting**: Retrieves and sorts recent Discord messages from the target channel to identify existing summaries for updating.

- **1.5 Data Mapping and Comparison**: A custom Code node maps GA data against existing Discord messages by date, deciding whether to create new messages or update existing ones.

- **1.6 Message Posting and Updating**: Based on the action decided (create or update), the workflow sends new messages or updates existing ones on Discord using the official Discord API (via HTTP Request nodes). Batches and wait nodes ensure rate limits and orderly processing.

- **1.7 Auxiliary Documentation Nodes**: Sticky Notes provide instructions, references, and configuration tips for Discord channel IDs, Google Analytics IDs, OAuth setups, and API usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Triggers for Multiple Accounts

- **Overview:** Four Schedule Trigger nodes initiate the workflow at different times (21:50 to 21:53), each triggering data retrieval for a distinct Google Analytics property and Discord channel.

- **Nodes Involved:** Schedule Trigger, Schedule Trigger1, Schedule Trigger2, Schedule Trigger3

- **Node Details:**

  - **Type:** Schedule Trigger (n8n-nodes-base.scheduleTrigger)
  - **Configuration:** Each triggers once daily at a specific hour and minute between 21:50 and 21:53.
  - **Input/Output:** No input; output triggers downstream "Edit Fields" nodes.
  - **Edge Cases:** Missed triggers if n8n instance is down; timezone assumptions (21:xx is server time).
  - **Sticky Notes:** No direct sticky notes on these nodes but related documentation exists for input ID setup.

#### 2.2 Preparation of Input Data

- **Overview:** Sets the Google Analytics property ID and Discord channel ID for each account. Four "Edit Fields" nodes correspond to four accounts.

- **Nodes Involved:** Edit Fields, Edit Fields2, Edit Fields3, Edit Fields4

- **Node Details:**

  - **Type:** Set Node (n8n-nodes-base.set)
  - **Configuration:** Assigns two fields: `google_analytics_id` (Google Analytics Property ID) and `discord_channel_id` (Discord Channel ID).
  - **Values:** Values are filled per account; some nodes have placeholder or empty strings for channel or GA ID, requiring user input.
  - **Input/Output:** Input from Schedule Trigger nodes; output to "Code1".
  - **Edge Cases:** Incorrect or missing IDs cause API failures downstream.
  - **Sticky Notes:** 
    - Sticky Note, Sticky Note3, Sticky Note4, Sticky Note5: Identify which account corresponds to which GA property and Discord channel.
    - Sticky Note10 and Sticky Note11 provide instructions on how to find Discord channel IDs and Google Analytics IDs respectively.

#### 2.3 Google Analytics Data Retrieval

- **Overview:** Queries Google Analytics API v4 for a set of user engagement metrics for the specified property.

- **Nodes Involved:** Code1, Google Analytics, Sort

- **Node Details:**

  - **Code1**
    - **Type:** Code node (JavaScript)
    - **Role:** Passes input data unchanged to Google Analytics node; effectively a passthrough.
    - **Input:** From "Edit FieldsX" nodes.
    - **Output:** To "Google Analytics".
    - **Edge Cases:** Minimal risk; mainly passes data.

  - **Google Analytics**
    - **Type:** Google Analytics node (n8n-nodes-base.googleAnalytics)
    - **Role:** Requests metrics: active1DayUsers, active7DayUsers, eventCount, screenPageViews, userEngagementDuration.
    - **Configuration:** Uses OAuth2 credentials for Google Analytics. Property ID dynamically set from input.
    - **Input:** From "Code1".
    - **Output:** To "Sort".
    - **Sticky Note:** Sticky Note6 explains setup and usage.
    - **Edge Cases:** Authentication errors, quota limits, invalid property ID.

  - **Sort**
    - **Type:** Sort node (n8n-nodes-base.sort)
    - **Role:** Sorts GA data by `date` ascending.
    - **Input:** From "Google Analytics".
    - **Output:** To "gaData".
    - **Edge Cases:** Sorting fails if `date` missing or malformed.

#### 2.4 Discord Messages Retrieval and Sorting

- **Overview:** Retrieves recent messages from the target Discord channel to detect existing daily digest messages for update purposes.

- **Nodes Involved:** gaData, Discord, Sort6, Code

- **Node Details:**

  - **gaData**
    - **Type:** Code node (JavaScript)
    - **Role:** Packages all GA data into a single JSON object property `gaData`.
    - **Input:** From "Sort".
    - **Output:** To "Discord".
    - **Edge Cases:** None significant.

  - **Discord**
    - **Type:** Discord node (n8n-nodes-base.discord)
    - **Role:** Fetches last 10 messages from the Discord channel specified by `discord_channel_id`.
    - **Authentication:** OAuth2 with Discord bot credentials.
    - **Input:** From "gaData".
    - **Output:** To "Sort6".
    - **Sticky Note:** Sticky Note7 explains Discord OAuth setup.
    - **Edge Cases:** Auth failures, rate limiting, channel permission errors.

  - **Sort6**
    - **Type:** Sort node
    - **Role:** Sorts Discord messages by the first embed field's value (assumed to be date).
    - **Input:** From "Discord".
    - **Output:** To "Code".
    - **Edge Cases:** Sorting errors if embed structure is inconsistent.

  - **Code**
    - **Type:** Code node (JavaScript)
    - **Role:** Core logic to compare GA data with Discord messages:
      - Extracts GA records and Discord messages.
      - Maps messages by date.
      - Compares fields to detect changes.
      - Determines whether to create new messages or update existing ones.
      - Formats message embeds with analytics data.
    - **Input:** From "Sort6" and GA data.
    - **Output:** Array of actions (create/update) with message payloads.
    - **Edge Cases:** Parsing errors if Discord messages lack expected embed fields; logic errors in comparison; date format mismatches.
    - **Sticky Note:** Sticky Note8 explains data mapping.

#### 2.5 Message Posting and Updating

- **Overview:** Based on the "create" or "update" action from the previous step, sends new messages or patches existing messages to Discord channels in batches with delays to respect rate limits.

- **Nodes Involved:** Switch, Loop Over Items4, HTTP Request, Wait4, Loop Over Items, HTTP Request8, Wait

- **Node Details:**

  - **Switch**
    - **Type:** Switch node
    - **Role:** Routes messages by action field: "create" or "update".
    - **Input:** From "Code".
    - **Output:** Two outputs, one for create, one for update paths.
    - **Edge Cases:** Action field missing or unexpected causes no routing.

  - **Loop Over Items4 (for create)**
    - **Type:** SplitInBatches node
    - **Role:** Processes messages to create in batches.
    - **Input:** From Switch (create output).
    - **Output:** To HTTP Request.
    - **Edge Cases:** Batch size defaults; no explicit batch size set.

  - **HTTP Request (Create)**
    - **Type:** HTTP Request node
    - **Role:** Posts new Discord messages via Discord API using Bot token auth.
    - **Configuration:** POST to `https://discord.com/api/v10/channels/{{channelId}}/messages`.
    - **Authentication:** HTTP Header Auth with Bot token.
    - **Input:** From Loop Over Items4.
    - **Output:** To Wait4.
    - **Sticky Note:** Sticky Note9 explains use of official Discord API for updates.
    - **Edge Cases:** HTTP errors, rate limits, invalid payloads.

  - **Wait4**
    - **Type:** Wait node
    - **Role:** Pauses 10 seconds between requests to manage rate limits.
    - **Input:** From HTTP Request (create).
    - **Output:** To Loop Over Items4.
    - **Edge Cases:** None significant.

  - **Loop Over Items (for update)**
    - **Type:** SplitInBatches node
    - **Role:** Processes update messages in batches.
    - **Input:** From Switch (update output).
    - **Output:** To HTTP Request8.
    - **Edge Cases:** Like Loop Over Items4.

  - **HTTP Request8 (Update)**
    - **Type:** HTTP Request node
    - **Role:** PATCH existing Discord messages via Discord API.
    - **Configuration:** PATCH to `https://discord.com/api/v10/channels/{{channelId}}/messages/{{messageId}}`.
    - **Authentication:** HTTP Header Auth with Bot token.
    - **Input:** From Loop Over Items.
    - **Output:** To Wait.
    - **Edge Cases:** Same as HTTP Request; additionally message ID errors.

  - **Wait**
    - **Type:** Wait node
    - **Role:** Pauses 10 seconds between update requests.
    - **Input:** From HTTP Request8.
    - **Output:** To Loop Over Items.
    - **Edge Cases:** None significant.

#### 2.6 Auxiliary Documentation Nodes

- **Overview:** Sticky Notes provide essential guidance on setup, IDs, OAuth, and API usage.

- **Nodes Involved:** Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note5, Sticky Note6, Sticky Note7, Sticky Note8, Sticky Note9, Sticky Note10, Sticky Note11

- **Details:**

  - Instructions on how to find Google Analytics Property IDs and Discord Channel IDs.
  - Documentation links for setting up Google Analytics and Discord OAuth credentials.
  - Explanation of why Discord API is used directly for message updating.
  - Visual aids and links to external articles.
  - Important context for users modifying or deploying this workflow.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                     | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                              |
|---------------------|---------------------|-----------------------------------|--------------------------|-------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger    | Trigger workflow for Account 2    | -                        | Edit Fields3            |                                                                                                        |
| Schedule Trigger1    | Schedule Trigger    | Trigger workflow for Account 1    | -                        | Edit Fields2            |                                                                                                        |
| Schedule Trigger2    | Schedule Trigger    | Trigger workflow for Account 4    | -                        | Edit Fields4            |                                                                                                        |
| Schedule Trigger3    | Schedule Trigger    | Trigger workflow for Account 3    | -                        | Edit Fields             |                                                                                                        |
| Edit Fields          | Set                 | Set GA Property and Discord ID 3  | Schedule Trigger3         | Code1                   | Sticky Note4 (GA Property 3 / Discord Channel 3)                                                       |
| Edit Fields2         | Set                 | Set GA Property and Discord ID 1  | Schedule Trigger1         | Code1                   | Sticky Note (GA Property 1 / Discord Channel 1)                                                        |
| Edit Fields3         | Set                 | Set GA Property and Discord ID 2  | Schedule Trigger           | Code1                   | Sticky Note3 (GA Property 2 / Discord Channel 2)                                                       |
| Edit Fields4         | Set                 | Set GA Property and Discord ID 4  | Schedule Trigger2         | Code1                   | Sticky Note5 (GA Property 4 / Discord Channel 4)                                                       |
| Code1               | Code                 | Pass-through input data            | Edit Fields, Edit Fields2, Edit Fields3, Edit Fields4 | Google Analytics          |                                                                                                        |
| Google Analytics     | Google Analytics     | Retrieve GA metrics                | Code1                    | Sort                    | Sticky Note6 (GA setup guide)                                                                          |
| Sort                 | Sort                 | Sort GA data by date               | Google Analytics         | gaData                  |                                                                                                        |
| gaData               | Code                 | Package GA data                    | Sort                     | Discord                 |                                                                                                        |
| Discord              | Discord              | Get last 10 messages from channel | gaData                   | Sort6                   | Sticky Note7 (Discord OAuth setup)                                                                     |
| Sort6                | Sort                 | Sort Discord messages by date      | Discord                  | Code                    |                                                                                                        |
| Code                 | Code                 | Compare GA data with Discord msgs | Sort6, gaData            | Switch                  | Sticky Note8 (Mapping data using Date field)                                                          |
| Switch               | Switch               | Route create or update actions     | Code                     | Loop Over Items4, Loop Over Items |                                                                                             |
| Loop Over Items4     | SplitInBatches       | Batch process new message creation | Switch (create)          | HTTP Request             |                                                                                                        |
| HTTP Request         | HTTP Request         | Post new message to Discord        | Loop Over Items4         | Wait4                   | Sticky Note9 (Use official Discord API for updates, Bot Token Auth)                                   |
| Wait4                | Wait                 | Delay between creates (10s)        | HTTP Request             | Loop Over Items4         |                                                                                                        |
| Loop Over Items      | SplitInBatches       | Batch process message updates      | Switch (update)          | HTTP Request8            |                                                                                                        |
| HTTP Request8        | HTTP Request         | Patch existing Discord message     | Loop Over Items          | Wait                    | Sticky Note9 (Use official Discord API for updates, Bot Token Auth)                                   |
| Wait                 | Wait                 | Delay between updates (10s)        | HTTP Request8            | Loop Over Items          |                                                                                                        |
| Sticky Note          | Sticky Note          | Docs: GA Property 1 / Discord Ch1 | -                        | -                       | "## Google Analytics Property 1\n## Discord Channel 1"                                                |
| Sticky Note1         | Sticky Note          | Docs: New Discord message rationale | -                        | -                       | "## send a new discord message\n\nif its a new day, then we send a new discord message..."             |
| Sticky Note2         | Sticky Note          | Docs: Update existing message rationale | -                        | -                       | "## update an existing discord message\n\nGoogle analytics data in not real time..."                   |
| Sticky Note3         | Sticky Note          | Docs: GA Property 2 / Discord Ch2 | -                        | -                       | "## Google Analytics Property 2\n## Discord Channel 2"                                                |
| Sticky Note4         | Sticky Note          | Docs: GA Property 3 / Discord Ch3 | -                        | -                       | "## Google Analytics Property 3\n## Discord Channel 3"                                                |
| Sticky Note5         | Sticky Note          | Docs: GA Property 4 / Discord Ch4 | -                        | -                       | "## Google Analytics Property 4\n## Discord Channel 4"                                                |
| Sticky Note6         | Sticky Note          | Docs: GA node setup guide          | -                        | -                       | "### Fetches and sorts google analytics data\n\nEnsure that your google analytics data is setup properly..." |
| Sticky Note7         | Sticky Note          | Docs: Discord OAuth setup guide    | -                        | -                       | "### Fetches and sorts discord messages from the channel\n\nEnsure that your discord OAuth is setup properly..." |
| Sticky Note8         | Sticky Note          | Docs: Data mapping explanation     | -                        | -                       | "### Maps the data\nusing the Date field"                                                             |
| Sticky Note9         | Sticky Note          | Docs: Use Discord API for message update | -                        | -                       | "### Use the discord api to update messages\nHere, we use the official discord api rather than the discord node..." |
| Sticky Note10        | Sticky Note          | Docs: How to get Discord channel ID| -                        | -                       | "## Get your discord channel id\n1. Get your discord channel id by sending a text on your discord channel..." |
| Sticky Note11        | Sticky Note          | Docs: How to get GA property ID    | -                        | -                       | "## Get your google analytics id\n1. Find your google analytics id by going to google analytics dashboard..." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create four Schedule Trigger nodes**:
   - Schedule Trigger1: Daily at 21:50
   - Schedule Trigger: Daily at 21:51
   - Schedule Trigger3: Daily at 21:52
   - Schedule Trigger2: Daily at 21:53

2. **Create four Set nodes ("Edit Fields" nodes)**, each connected from one Schedule Trigger:
   - Assign two string fields:
     - `google_analytics_id`: Set your GA Property ID for that account.
     - `discord_channel_id`: Set the Discord Channel ID where messages will be posted.
   - Connect each Set node to a single Code node ("Code1").

3. **Create a Code node ("Code1")**:
   - JavaScript code: `return $input.all();` (simply passes data).
   - Connect all four "Edit Fields" nodes to this node (fan-in).

4. **Add a Google Analytics node**:
   - Credentials: Google Analytics OAuth2 with valid user token.
   - Property ID: Set dynamically with expression `={{ $json.google_analytics_id }}`.
   - Metrics to fetch (GA4): active1DayUsers, active7DayUsers, eventCount, screenPageViews, userEngagementDuration.
   - Connect "Code1" node output to this node.

5. **Add a Sort node** to sort GA data by `date` ascending.
   - Connect from Google Analytics node.

6. **Add a Code node ("gaData")**:
   - JavaScript code: `return {"gaData": $input.all()};` (wraps data).
   - Connect from Sort node.

7. **Add a Discord node**:
   - OAuth2 credentials for your Discord Bot.
   - Operation: Get All Messages.
   - Limit: 10 messages.
   - Guild ID: Set your Discord server ID.
   - Channel ID: Dynamic from `discord_channel_id` field, expression e.g. `={{ $json.discord_channel_id }}`.
   - Connect from "gaData" node.

8. **Add a Sort node ("Sort6")**:
   - Sort messages by first embed field value (assumed date).
   - Connect from Discord node.

9. **Add a Code node ("Code")**:
   - Implement the logic to:
     - Extract GA data and Discord messages.
     - Map Discord messages by date using embed fields.
     - Compare GA data with existing messages.
     - Build an array of objects with actions "create" or "update" and payloads for Discord embeds.
   - Connect from "Sort6" node.

10. **Add a Switch node**:
    - Field to switch on: `action`.
    - Outputs: Two outputs — one for "create", one for "update" (string equality).

11. **For "create" output**:
    - Add a SplitInBatches node ("Loop Over Items4") to batch process creations.
    - Connect to HTTP Request node:
      - Method: POST
      - URL: `https://discord.com/api/v10/channels/{{ $json.channelId }}/messages`
      - Authentication: HTTP Header Auth (Bot token)
      - Headers: Content-Type: application/json
      - Body: JSON from `$json.payload`
    - Connect to a Wait node with 10 seconds delay.
    - Loop back to SplitInBatches node for batch processing.

12. **For "update" output**:
    - Add a SplitInBatches node ("Loop Over Items") to batch process updates.
    - Connect to HTTP Request node:
      - Method: PATCH
      - URL: `https://discord.com/api/v10/channels/{{ $json.channelId }}/messages/{{ $json.messageId }}`
      - Authentication: HTTP Header Auth (Bot token)
      - Headers: Content-Type: application/json
      - Body: JSON from `$json.payload`
    - Connect to a Wait node with 10 seconds delay.
    - Loop back to SplitInBatches node.

13. **Set up Credentials**:
    - Google Analytics OAuth2: with scopes to read GA4 data.
    - Discord OAuth2 API: For reading Discord messages (bot with required permissions).
    - HTTP Header Auth Credential: Bot token for Discord API calls (POST/PATCH).

14. **Add Sticky Notes for Documentation**:
    - Include instructions for:
      - Finding Discord channel ID.
      - Finding Google Analytics property ID.
      - Setting up Google Analytics OAuth2.
      - Setting up Discord OAuth2.
      - Using Discord API for message updates.
      - Explanation of data refresh timing and update logic.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                       | Context or Link                                                                                                                           |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Use the official Discord API endpoints to update messages as the Discord node in n8n does not support message edits yet. Follow the Discord API documentation for authentication and endpoints: https://discord.com/developers/docs/reference#authentication                                                                             | Sticky Note9                                                                                                                             |
| Instructions to find your Discord channel ID by sending a message and copying the channel ID from the message link: https://articles.emp0.com/wp-content/uploads/2025/06/12345678-1.png                                                                                                | Sticky Note10                                                                                                                            |
| Instructions to find your Google Analytics property ID from the GA dashboard: https://articles.emp0.com/wp-content/uploads/2025/06/12345678.png                                                                                                                                          | Sticky Note11                                                                                                                            |
| Google Analytics node setup guide and metrics reference: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleanalytics/                                                                                                                                           | Sticky Note6                                                                                                                             |
| Discord OAuth2 setup guide for n8n: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.discord/                                                                                                                                                                       | Sticky Note7                                                                                                                             |
| Google Analytics data is not real-time and may change for up to 7 days. The workflow updates previous Discord messages to reflect final traffic values.                                                                                                                               | Sticky Note2                                                                                                                             |
| New daily Discord messages are sent at UTC midnight refresh time to show the latest Google Analytics data for the day.                                                                                                                                                                | Sticky Note1                                                                                                                             |

---

This detailed analysis and reconstruction guide provide all necessary information to understand, modify, or reproduce the "Discord Daily Digest for Multiple Google Analytics Accounts" workflow in n8n. The workflow integrates scheduled triggers, data retrieval, mapping, conditional logic, and API calls to automate daily analytics reporting in Discord.