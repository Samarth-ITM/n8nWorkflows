Get all members of a Discord server with a specific role

https://n8nworkflows.xyz/workflows/get-all-members-of-a-discord-server-with-a-specific-role-2105


# Get all members of a Discord server with a specific role

### 1. Workflow Overview

This workflow is designed to retrieve all members from a specified Discord server (guild) who have a particular role. Because the Discord API limits member retrieval to batches of 100 users per request, this workflow implements a batching mechanism using Google Sheets as a persistent storage to track the last processed member ID. It sequentially fetches members in batches, filters them by role, and aggregates the results until all relevant members are retrieved.

The workflow contains the following main logical blocks:

- **1.1 Input Reception and Initialization**: Triggering the workflow manually or via webhook and setting up required parameters such as Discord server ID, role ID, and Google Sheets URL.
- **1.2 State Management Using Google Sheets**: Reading and updating the last processed member ID in Google Sheets to handle batch retrieval.
- **1.3 Discord Member Retrieval**: Fetching batches of up to 100 members from Discord, either starting fresh or continuing after the last saved member ID.
- **1.4 Member Filtering and Aggregation**: Filtering members to only those with the target role, merging results, and checking if there are more members to retrieve.
- **1.5 Cleanup and Final Response**: Deleting the last processed ID if all members have been fetched and sending the filtered results as response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview**: Receives a manual trigger or webhook call and sets up initial workflow parameters such as role ID, Google Sheets URL, and Discord server ID.
- **Nodes Involved**:
  - When clicking "Test workflow"
  - Webhook (disabled)
  - Setup: Edit this to get started
  - Get ID
  - Check if we have an ID

- **Node Details**:

  - **When clicking "Test workflow"**
    - Type: Manual Trigger
    - Role: Starts workflow execution manually for testing purposes.
    - Configuration: No parameters.
    - Input: None.
    - Output: Connects to Setup node.
    - Edge Cases: None expected.
  
  - **Webhook**
    - Type: Webhook
    - Role: Enables triggering the workflow via HTTP request (disabled by default).
    - Configuration: Path set to "discord-template", response mode set to respond with a node result.
    - Input: External HTTP request.
    - Output: Connects to Setup node.
    - Edge Cases: Requires activation and proper URL calling; disabled by default.
  
  - **Setup: Edit this to get started**
    - Type: Set Node
    - Role: Defines essential parameters for the workflow.
    - Configuration: Three string fields:
      - `Role ID` - Discord role ID to filter members.
      - `Google Sheets URL` - URL of the Google Sheets document used to store last processed member ID.
      - `Discord ID` - Guild/server ID.
    - Input: Trigger nodes.
    - Output: Connects to Get ID node.
    - Edge Cases: Requires user to enter valid IDs and URLs.
  
  - **Get ID**
    - Type: Google Sheets - Read Operation
    - Role: Reads the last stored member ID from Google Sheets to continue retrieval.
    - Configuration:
      - Document ID and Sheet Name ("gid=0") taken from Setup node.
      - Always outputs data even if sheet is empty.
    - Input: Setup node.
    - Output: Connects to "Check if we have an ID".
    - Edge Cases: Empty sheet means no last ID saved (first run).
    - Requires valid Google Sheets OAuth2 credentials.
  
  - **Check if we have an ID**
    - Type: If Node
    - Role: Determines whether to fetch members starting after the last ID or start from scratch.
    - Configuration: Checks if `ID` field from Get ID node exists and is non-empty.
    - Input: Get ID node.
    - Output:
      - True branch: Get next 100 Members after last ID.
      - False branch: Get First 100 Members.
    - Edge Cases: No ID means a fresh fetch; missing or malformed data could cause logic errors.

#### 2.2 State Management Using Google Sheets

- **Overview**: Tracks the last member ID fetched to paginate through Discord members in batches.
- **Nodes Involved**:
  - SaveID
  - Delete ID
  - Get ID (also in initialization block)

- **Node Details**:

  - **SaveID**
    - Type: Google Sheets - Append Operation
    - Role: Appends the latest member ID from the current batch to Google Sheets to remember progress.
    - Configuration:
      - Document and sheet specified via Setup node.
      - Appends ID from the last user in the merged batch.
      - Uses Google Sheets OAuth2 credentials.
    - Input: Merge node output.
    - Output: Connects back to Get ID to read updated ID.
    - Edge Cases: Append failure due to credentials or sheet access issues.
  
  - **Delete ID**
    - Type: Google Sheets - Delete Operation
    - Role: Deletes the stored member ID data once all members are fetched to reset state.
    - Configuration:
      - Document and sheet from Setup node.
      - Executes "delete" on all rows (assumed).
      - Executes once per run.
    - Input: "Check if we have more members left" false branch.
    - Output: Connects to SaveID to reset sheet state.
    - Edge Cases: Failure to delete due to permissions or sheet locks.

#### 2.3 Discord Member Retrieval

- **Overview**: Retrieves members from Discord in batches of 100, either starting at the beginning or after the last saved ID.
- **Nodes Involved**:
  - Get First 100 Members
  - Get next 100 Members after last ID

- **Node Details**:

  - **Get First 100 Members**
    - Type: Discord Node (Member Resource)
    - Role: Fetches the first 100 members of the specified Discord guild.
    - Configuration:
      - Guild ID from Setup node.
      - Simplify option enabled for easier data structure.
      - Uses Discord Bot API credentials.
    - Input: False branch of "Check if we have an ID".
    - Output: Connects to Merge node (output index 1).
    - Edge Cases: API rate limits, invalid guild ID, bot permissions.
  
  - **Get next 100 Members after last ID**
    - Type: Discord Node (Member Resource)
    - Role: Fetches next batch of 100 members after the ID stored in Google Sheets.
    - Configuration:
      - After parameter set to last saved ID.
      - Guild ID from Setup node.
      - Simplify option enabled.
      - Uses Discord Bot API credentials.
    - Input: True branch of "Check if we have an ID".
    - Output: Connects to Merge node (output index 0).
    - Edge Cases: Same as above plus invalid or expired last ID.

#### 2.4 Member Filtering and Aggregation

- **Overview**: Merges member batches, filters them to only include those with the specified role, and evaluates if more members remain to be fetched.
- **Nodes Involved**:
  - Merge
  - Filter to only include members with role
  - Check if we have more members left

- **Node Details**:

  - **Merge**
    - Type: Merge Node
    - Role: Combines member lists from either first fetch or subsequent fetches.
    - Configuration: Default merge (append).
    - Input: Both "Get First 100 Members" (index 1) and "Get next 100 Members after last ID" (index 0).
    - Output: Connects to "Check if we have more members left" and "Filter to only include members with role".
    - Edge Cases: Mismatched data structures could cause merge errors.
  
  - **Filter to only include members with role**
    - Type: Filter Node
    - Role: Filters merged members to retain only those who have the specified role.
    - Configuration: Checks if the member’s roles array includes the Role ID specified in Setup node.
    - Input: Merge node.
    - Output: Connects to Send Response node.
    - Edge Cases: Members without a roles property or empty arrays.
  
  - **Check if we have more members left**
    - Type: If Node
    - Role: Checks if the number of members fetched in the last batch is less than 100 to determine if retrieval is complete.
    - Configuration: Compares the length of the input array to 100, branches accordingly.
    - Input: Merge node.
    - Output:
      - True branch (less than 100): Connects to "We're done".
      - False branch: Connects to "Delete ID" to prepare for next batch.
    - Edge Cases: API returning fewer than 100 members could be an end signal; failure to process length could cause logic errors.

#### 2.5 Cleanup and Final Response

- **Overview**: Finalizes the workflow by either ending the process or preparing the sheet for next batch, and sends the filtered results as response.
- **Nodes Involved**:
  - We're done
  - Send Response

- **Node Details**:

  - **We're done**
    - Type: NoOp Node
    - Role: Marks completion of the workflow when no more members are left to fetch.
    - Input: True branch of "Check if we have more members left".
    - Output: None.
  
  - **Send Response**
    - Type: Respond to Webhook Node
    - Role: Sends the filtered members with the specified role back as a response to the webhook call.
    - Configuration: Responds with all incoming items.
    - Input: Filter node output.
    - Output: Terminal.
    - Edge Cases: Must be triggered via webhook to return data; otherwise no output.

---

### 3. Summary Table

| Node Name                        | Node Type                  | Functional Role                          | Input Node(s)                         | Output Node(s)                                  | Sticky Note                                                                                                                      |
|---------------------------------|----------------------------|----------------------------------------|-------------------------------------|------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking "Test workflow"    | Manual Trigger             | Manual start of workflow for testing  | None                                | Setup: Edit this to get started                 |                                                                                                                                  |
| Webhook                         | Webhook                    | Webhook start (disabled by default)   | None                                | Setup: Edit this to get started                 |                                                                                                                                  |
| Setup: Edit this to get started | Set                        | Defines key parameters (Role ID, etc.)| When clicking "Test workflow", Webhook | Get ID                                      |                                                                                                                                  |
| Get ID                         | Google Sheets (Read)        | Reads last saved member ID             | Setup: Edit this to get started      | Check if we have an ID, SaveID                   |                                                                                                                                  |
| Check if we have an ID          | If                         | Decides to fetch first or next batch  | Get ID                             | Get next 100 Members after last ID, Get First 100 Members |                                                                                                                       |
| Get First 100 Members           | Discord                    | Fetches first 100 members from guild  | Check if we have an ID (false branch) | Merge                                        |                                                                                                                          |
| Get next 100 Members after last ID | Discord                | Fetches next batch after last ID       | Check if we have an ID (true branch) | Merge                                        |                                                                                                                          |
| Merge                          | Merge                      | Combines fetched member batches       | Get First 100 Members, Get next 100 Members after last ID | Check if we have more members left, Filter to only include members with role |                                                                                       |
| Check if we have more members left | If                     | Checks if batch less than 100 to end  | Merge                             | We're done (true), Delete ID (false)             |                                                                                                                          |
| Delete ID                      | Google Sheets (Delete)      | Clears last saved ID to reset state   | Check if we have more members left (false) | SaveID                                     |                                                                                                                                  |
| SaveID                        | Google Sheets (Append)      | Appends latest member ID to sheet     | Delete ID, Merge                   | Get ID                                            |                                                                                                                                  |
| Filter to only include members with role | Filter             | Filters members by role                | Merge                             | Send Response                                     |                                                                                                                                  |
| Send Response                  | Respond to Webhook          | Sends filtered members as response    | Filter to only include members with role | None                                        |                                                                                                                                  |
| We're done                    | NoOp                       | Marks workflow completion              | Check if we have more members left (true) | None                                        |                                                                                                                                  |
| Sticky Note                   | Sticky Note                | Setup instructions and usage notes    | None                               | None                                             | ## Setup 1. Add your Google Sheets and Discord credentials. 2. Create a Google Sheets document with ID column ... [link](https://www.pythondiscord.com/pages/guides/pydis-guides/contributing/obtaining-discord-ids/) |
| Sticky Note1                  | Sticky Note                | Suggests replacing response node for use cases | None                          | None                                             | You can replace this node according to your use case. In my case, I've sent a DM to all users                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: "When clicking \"Test workflow\""
   - Type: Manual Trigger
   - No parameters.
  
2. **Create Webhook Node (Optional)**
   - Name: "Webhook"
   - Type: Webhook
   - Parameters:
     - Path: "discord-template"
     - Response Mode: "responseNode"
   - Disabled by default.
  
3. **Create Set Node for Setup**
   - Name: "Setup: Edit this to get started"
   - Type: Set
   - Add three string fields:
     - Role ID: `<Enter your roleID here>`
     - Google Sheets URL: `<Enter your Sheets URL here>`
     - Discord ID: `<Enter your server/guild ID here>`
  
4. **Connect Trigger(s) to Setup Node**
   - Connect "When clicking \"Test workflow\"" and "Webhook" output to "Setup" input.
  
5. **Create Google Sheets Read Node**
   - Name: "Get ID"
   - Type: Google Sheets
   - Operation: Read (default)
   - Document ID: Expression referencing Setup node's "Google Sheets URL"
   - Sheet Name: "gid=0"
   - Credentials: Google Sheets OAuth2 (configure with your Google account)
   - Enable "Always Output Data" to handle empty sheets.
   - Connect Setup output to this node.
  
6. **Create If Node to Check for Existing ID**
   - Name: "Check if we have an ID"
   - Type: If
   - Condition: Check if `ID` field exists and is non-empty in input JSON.
   - Configure to execute once.
   - Connect "Get ID" output to this node.
  
7. **Create Discord Node to Get First 100 Members**
   - Name: "Get First 100 Members"
   - Type: Discord
   - Resource: Member
   - Operation: List members
   - Guild ID: Expression from Setup node's "Discord ID"
   - Options: Simplify enabled
   - Credentials: Discord Bot API (configure with your bot credentials)
   - Connect "Check if we have an ID" false branch here.
  
8. **Create Discord Node to Get Next 100 Members after Last ID**
   - Name: "Get next 100 Members after last ID"
   - Type: Discord
   - Resource: Member
   - Operation: List members
   - Guild ID: Expression from Setup node's "Discord ID"
   - After: Expression referencing `ID` from "Get ID" node
   - Options: Simplify enabled
   - Credentials: Discord Bot API
   - Connect "Check if we have an ID" true branch here.
  
9. **Create Merge Node**
   - Name: "Merge"
   - Type: Merge
   - Mode: Append (default)
   - Connect outputs from "Get First 100 Members" (index 1) and "Get next 100 Members after last ID" (index 0) to this node.
  
10. **Create If Node to Check if More Members Exist**
    - Name: "Check if we have more members left"
    - Type: If
    - Condition: Check if length of input array is less than 100.
    - Connect "Merge" output to this node.
  
11. **Create NoOp Node for Completion**
    - Name: "We're done"
    - Type: NoOp
    - Connect true branch of "Check if we have more members left" here.
  
12. **Create Google Sheets Delete Node**
    - Name: "Delete ID"
    - Type: Google Sheets
    - Operation: Delete
    - Document ID and Sheet Name from Setup node (expression)
    - Credentials: Google Sheets OAuth2
    - Connect false branch of "Check if we have more members left" here.
  
13. **Create Google Sheets Append Node**
    - Name: "SaveID"
    - Type: Google Sheets
    - Operation: Append
    - Document ID and Sheet Name from Setup node (expression)
    - Columns: Map "ID" to `user.id` of last member from "Merge" node (use expression)
    - Credentials: Google Sheets OAuth2
    - Connect output of "Delete ID" here.
  
14. **Connect "SaveID" output back to "Get ID"**
    - For iterative loop to continue fetching batches.
  
15. **Create Filter Node to Include Only Members with Specific Role**
    - Name: "Filter to only include members with role"
    - Type: Filter
    - Condition: Check if member's roles array contains the Role ID from Setup node.
    - Connect "Merge" output to this node.
  
16. **Create Respond to Webhook Node**
    - Name: "Send Response"
    - Type: Respond to Webhook
    - Respond with all incoming items.
    - Connect "Filter to only include members with role" output here.
  
17. **Optional: Add Sticky Notes**
    - Add sticky notes with setup instructions and usage notes for clarity.
  
18. **Credential Setup**
    - Configure Google Sheets OAuth2 credentials with access to your Google Sheets document.
    - Configure Discord Bot API credentials with bot token having necessary permissions on the target Discord server.
  
19. **Activate Workflow**
    - Enable the webhook node if you intend to trigger via webhook.
    - Otherwise, use manual trigger node for testing.
  
20. **Execution**
    - Trigger the workflow manually or via webhook to start retrieving members.
    - Monitor Google Sheets for the stored last ID.
    - Workflow loops fetching batches until all members with the specified role are retrieved.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                                                                |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| Setup instructions emphasize creating a Google Sheets document with an "ID" column to track member pagination progress.                               | Sticky Note in workflow                                                                                                                       |
| To obtain Discord IDs (Role ID, Guild ID), refer to: https://www.pythondiscord.com/pages/guides/pydis-guides/contributing/obtaining-discord-ids/        | Setup Sticky Note                                                                                                                             |
| Minimum n8n version required: 1.28.0                                                                                                                   | Workflow metadata and requirements                                                                                                           |
| Potential use cases include messaging all members of a role, analyzing user growth, and storing member data regularly.                                 | Workflow description                                                                                                                          |
| Replace "Send Response" node with custom logic as needed, e.g., sending direct messages to filtered members.                                           | Sticky Note1                                                                                                                                  |
| Google Sheets operations assume proper OAuth2 credentials and sheet permissions; misconfiguration may cause failures.                                  | General operational note                                                                                                                      |
| Discord Bot API requires bot token with admin rights or sufficient permissions for member list retrieval and role inspection.                         | General operational note                                                                                                                      |

---

This detailed reference document enables both human users and AI agents to fully comprehend, reproduce, and adapt the workflow to various scenarios involving Discord member management and automation.