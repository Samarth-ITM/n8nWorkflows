Realtime Notion Todoist 2-way Sync with Redis

https://n8nworkflows.xyz/workflows/realtime-notion-todoist-2-way-sync-with-redis-2772


# Realtime Notion Todoist 2-way Sync with Redis

### 1. Workflow Overview

This workflow implements a **real-time, bi-directional synchronization** between Notion and Todoist task management systems, enhanced with Redis caching to prevent update loops. It is designed for users managing tasks across multiple Notion workspaces and Todoist projects, consolidating task and calendar event management.

The workflow is logically divided into these main blocks:

- **1.1 Setup and Configuration:** Tools to generate global configuration JSON, register OAuth credentials, and activate Todoist webhooks.
- **1.2 Notion to Todoist Sync (Diff Sync):** Triggered by Notion webhook, compares tasks, updates or creates corresponding Todoist tasks, handles deletions and status changes.
- **1.3 Todoist to Notion Sync (Diff Sync):** Triggered by Todoist webhook, compares tasks, updates or creates corresponding Notion pages, handles deletions and status changes.
- **1.4 Full Daily Sync:** Scheduled workflow that fetches all tasks from both systems, compares datasets, resolves inconsistencies, and sends a summary email report.
- **1.5 Redis Locking Mechanism:** Used throughout to prevent infinite update loops by locking task IDs for 15 seconds after updates.
- **1.6 Utility and Helper Nodes:** Includes nodes for mapping fields between Notion and Todoist, handling date formats, and managing execution data for debugging.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Configuration

**Overview:**  
This block assists in setting up the integration by generating global configuration JSON, managing OAuth authentication for Todoist, and registering webhooks.

**Nodes Involved:**  
- Notion-Todoist Sync Setup Helper (Form Trigger)  
- Get Notion Databases (Notion API)  
- Prep Dropdown (Code)  
- Choose Notion Database (Form)  
- Get projects (HTTP Request to Todoist)  
- Prep Dropdown1 (Code)  
- Choose Todoist Project (Form)  
- Get sections (HTTP Request to Todoist)  
- Generate config (Code)  
- Return config JSON (Form)  
- Todoist Webhook Setup Helper (Form Trigger)  
- Generate security token (Crypto)  
- Store variables (Code)  
- Redirect to Auth Page (Form)  
- OAuth redirect (Webhook)  
- Verify security token (If)  
- Exchange Tokens (HTTP Request)  
- Respond with success (Respond to Webhook)  
- Respond with error (Respond to Webhook)  

**Node Details:**

- **Notion-Todoist Sync Setup Helper:** Form trigger node to start the setup process. Presents a form to begin configuration.
- **Get Notion Databases:** Retrieves all Notion databases accessible with the provided credentials.
- **Prep Dropdown / Prep Dropdown1:** Prepare dropdown options for Notion databases and Todoist projects, excluding "Inbox".
- **Choose Notion Database / Choose Todoist Project:** Form nodes to select the target Notion database and Todoist project.
- **Get projects / Get sections:** HTTP requests to Todoist API to retrieve projects and sections for mapping.
- **Generate config:** Aggregates selected database and project IDs and sections into a JSON config for global use.
- **Return config JSON:** Outputs the generated config JSON for pasting into Globals nodes.
- **Todoist Webhook Setup Helper:** Form trigger to input Todoist Developer App Client ID and Secret.
- **Generate security token:** Generates a random state token for OAuth security.
- **Store variables:** Stores OAuth client ID, secret, and state in workflow static data.
- **Redirect to Auth Page:** Redirects user to Todoist OAuth authorization page with client ID and state.
- **OAuth redirect:** Receives OAuth callback with authorization code.
- **Verify security token:** Validates the returned state token matches the generated one.
- **Exchange Tokens:** Exchanges authorization code for access token.
- **Respond with success / error:** Responds to OAuth redirect with success or error message.

**Edge Cases / Failure Types:**  
- OAuth token exchange failure or invalid state token.  
- API rate limits or credential misconfiguration.  
- User cancels OAuth authorization.

---

#### 1.2 Notion to Todoist Sync (Diff Sync)

**Overview:**  
Triggered by Notion webhook, this block fetches updated Notion tasks, compares them with Todoist tasks, and updates Todoist accordingly. It handles task creation, updates, completion, reopening, and deletion (marked as "Obsolete" in Notion).

**Nodes Involved:**  
- Notion Webhook (Webhook)  
- Body is array? (If)  
- Split out Notion changes (SplitOut)  
- Extract IDs (Set)  
- Notion trigger reference (NoOp)  
- Globals2 (Set)  
- Execution Data5 (Execution Data)  
- Check if Notion ID is locked (Redis)  
- Only continue if not locked1 (Filter)  
- Get Notion task (Notion)  
- Map Notion to Todoist2 (Code)  
- Todoist ID exists? (If)  
- Set tries1 (Set)  
- Get Todoist Task2 (Todoist)  
- Differences exist (Filter)  
- Requires completion change (Filter)  
- Has been completed? (If)  
- Mark as Completed in Todoist1 (Todoist)  
- Mark as Incomplete in Todoist1 (Todoist)  
- Update task in Todoist1 (HTTP Request)  
- Create task in Todoist1 (HTTP Request)  
- Set creating flag1 (Redis)  
- Delete Task in Todoist2 (Todoist)  
- Store Todoist ID1 (Notion)  
- Lock Todoist ID4 (Redis)  
- Handle empty dates1 (Code)  
- Status changed but not empty (Filter)  
- Generate UUID2 (Crypto)  
- Update section (Sync API)2 (HTTP Request)  
- Get Backlog Section ID (Code)  
- Create task in Notion (Notion)  
- Mark as Focussed in Notion (Notion)  
- Mark as Done in Notion (Notion)  
- Mark as Obsolete in Notion (Notion)  
- Mark as In Progress in Notion (Notion)  
- Execution Data6 (Execution Data)  
- Catch known error1 (If)  
- Wait1 (Wait)  
- Retry limit reached1 (Stop and Error)  

**Node Details:**

- **Notion Webhook:** Receives real-time updates from Notion database.
- **Body is array?:** Handles different webhook payload formats (array or single object).
- **Split out Notion changes:** Splits multiple changes into individual items.
- **Extract IDs:** Extracts Notion page IDs from webhook payload.
- **Check if Notion ID is locked:** Checks Redis cache to avoid processing locked tasks.
- **Get Notion task:** Retrieves full Notion page data for the task.
- **Map Notion to Todoist2:** Maps Notion task properties to Todoist task fields, including priority, status, due date, and completion.
- **Todoist ID exists?:** Checks if the Notion task has an associated Todoist ID.
- **Get Todoist Task2:** Retrieves the corresponding Todoist task.
- **Differences exist:** Compares mapped Notion data with Todoist task to detect changes.
- **Requires completion change:** Checks if completion status differs.
- **Has been completed?:** Determines if the task was marked completed.
- **Mark as Completed/Incompleted in Todoist:** Updates Todoist task completion status.
- **Update task in Todoist1:** Updates existing Todoist task with Notion data.
- **Create task in Todoist1:** Creates new Todoist task if none exists.
- **Set creating flag1:** Sets Redis flag to prevent duplicate creation loops.
- **Delete Task in Todoist2:** Deletes Todoist task if marked obsolete in Notion.
- **Store Todoist ID1:** Updates Notion task with Todoist ID after creation.
- **Lock Todoist ID4:** Locks Todoist task ID in Redis to prevent loops.
- **Handle empty dates1:** Adjusts due date fields for Todoist API compatibility.
- **Update section (Sync API)2:** Updates Todoist task section using Sync API with batching.
- **Mark as Focussed/Done/Obsolete/In Progress in Notion:** Updates Notion task status accordingly.
- **Catch known error1 / Wait1 / Retry limit reached1:** Implements retry logic for Todoist API calls.

**Edge Cases / Failure Types:**  
- Redis lock misses causing loops.  
- Todoist API rate limits or transient errors.  
- Notion webhook retries causing duplicate processing.  
- Missing or invalid Todoist section mappings.  
- Tasks moved from Obsolete to Done or vice versa.

---

#### 1.3 Todoist to Notion Sync (Diff Sync)

**Overview:**  
Triggered by Todoist webhook, this block fetches updated Todoist tasks, compares them with Notion pages, and updates Notion accordingly. It handles task creation, updates, completion, reopening, and deletion (marked as "Obsolete" in Notion).

**Nodes Involved:**  
- Todoist Webhook (Webhook)  
- Switch by project (Switch)  
- Todoist trigger reference (NoOp)  
- Execution Data (Execution Data)  
- Globals1 (Set)  
- Check if Todoist ID is locked (Redis)  
- Only continue if not locked (Filter)  
- If event is not :deleted (If)  
- Set tries2 (Set)  
- Get Notion Task2 (Notion)  
- Notion Task found1 (Filter)  
- Map Todoist to Notion1 (Code)  
- Differences exist1 (Filter)  
- Update task in Notion (Notion)  
- Due date empty (Filter)  
- Mark as Focussed in Notion (Notion)  
- Mark as Done in Notion (Notion)  
- Mark as Obsolete in Notion (Notion)  
- Mark as In Progress in Notion (Notion)  
- Create task in Notion (Notion)  
- Update Description in Todoist1 (Todoist)  
- Description has changed in Todoist (Filter)  
- Update Description in Todoist (Todoist)  
- Generate UUID1 (Crypto)  
- Update section (Sync API)1 (HTTP Request)  
- Get Backlog Section ID (Code)  
- Check if creating flag exists1 (Redis)  
- Only continue if flag does not exist1 (Filter)  
- Set creating flag1 (Redis)  
- Execution Data3 (Execution Data)  
- Catch known error2 (If)  
- Wait2 (Wait)  
- Retry limit reached2 (Stop and Error)  

**Node Details:**

- **Todoist Webhook:** Receives real-time updates from Todoist.
- **Switch by project:** Routes webhook events by project ID (supports multi-project sync).
- **Check if Todoist ID is locked:** Checks Redis cache to avoid processing locked tasks.
- **If event is not :deleted:** Filters out deleted events for special handling.
- **Get Notion Task2:** Retrieves Notion page linked to Todoist task.
- **Notion Task found1:** Checks if Notion page exists.
- **Map Todoist to Notion1:** Maps Todoist task fields to Notion properties.
- **Differences exist1:** Compares Todoist data with Notion page to detect changes.
- **Update task in Notion:** Updates existing Notion page.
- **Create task in Notion:** Creates new Notion page if none exists.
- **Update Description in Todoist1:** Updates Todoist task description with Notion URL.
- **Description has changed in Todoist:** Detects if description needs update.
- **Generate UUID1 / Update section (Sync API)1:** Updates Todoist task section using Sync API.
- **Check if creating flag exists1 / Set creating flag1:** Redis flags to prevent duplicates.
- **Catch known error2 / Wait2 / Retry limit reached2:** Retry logic for Notion API calls.

**Edge Cases / Failure Types:**  
- Redis lock misses causing loops.  
- Todoist webhook events arriving out of order.  
- Notion API rate limits or transient errors.  
- Missing or invalid Notion database or property mappings.  
- Task deletions requiring marking as Obsolete instead of deletion.

---

#### 1.4 Full Daily Sync

**Overview:**  
Scheduled daily at 6 AM, this block performs a full synchronization by fetching all open tasks from Notion and Todoist, comparing datasets, resolving discrepancies, updating tasks, and sending a summary email report.

**Nodes Involved:**  
- Schedule Trigger (Schedule)  
- Notion (Notion API)  
- Todoist (Todoist API)  
- Globals (Set)  
- Compare Datasets (Compare Datasets)  
- Handle empty dates (Code)  
- Handle empty dates2 (Code)  
- Loop Over Items (SplitInBatches)  
- Exists/Completed in Notion (If)  
- Map Notion to Todoist1 (Code)  
- Delete Task in Todoist (Todoist)  
- Update Task in Todoist (HTTP Request)  
- Add project ID (Set)  
- If Todoist ID exists (If)  
- Create task in Todoist (HTTP Request)  
- Lock Todoist ID (Redis)  
- Prepare summary data (Set)  
- Merge summary (Merge)  
- Filter out status changes (Filter)  
- Map Todoist to Notion (Code)  
- Prepare summary data1 (Set)  
- Prepare summary data2 (Set)  
- Convert to HTML Table (HTML)  
- Generate email body (HTML)  
- Gmail (Gmail)  

**Node Details:**

- **Schedule Trigger:** Runs daily at 6 AM.
- **Notion / Todoist:** Fetch all open tasks from both systems.
- **Compare Datasets:** Compares tasks by ID to detect differences.
- **Handle empty dates / Handle empty dates2:** Adjusts date fields for API compatibility.
- **Loop Over Items:** Processes tasks in batches.
- **Exists/Completed in Notion:** Checks if task exists and is completed.
- **Map Notion to Todoist1:** Maps Notion task to Todoist format.
- **Delete Task in Todoist / Update Task in Todoist / Create task in Todoist:** Synchronizes Todoist tasks.
- **Lock Todoist ID:** Locks task ID in Redis.
- **Prepare summary data / Merge summary / Filter out status changes:** Prepares data for email report.
- **Map Todoist to Notion:** Maps Todoist task to Notion format.
- **Convert to HTML Table / Generate email body:** Formats summary as HTML email.
- **Gmail:** Sends the email report.

**Edge Cases / Failure Types:**  
- API rate limits due to large batch updates.  
- Email sending failures.  
- Data inconsistencies not resolved by diff sync.

---

#### 1.5 Redis Locking Mechanism

**Overview:**  
Redis nodes are used throughout the workflow to lock task IDs for 15 seconds after updates to prevent infinite update loops caused by webhook triggers firing on both sides.

**Nodes Involved:**  
- Lock Todoist ID (Redis)  
- Lock Todoist ID1 (Redis)  
- Lock Todoist ID2 (Redis)  
- Lock Todoist ID4 (Redis)  
- Lock Notion ID (Redis)  
- Check if Todoist ID is locked (Redis)  
- Check if Notion ID is locked (Redis)  
- Set creating flag (Redis)  
- Check if creating flag exists (Redis)  

**Node Details:**

- **Lock nodes:** Set a Redis key with TTL 15 seconds for task IDs.
- **Check nodes:** Retrieve Redis keys to check if task is locked.
- **Creating flag nodes:** Prevent duplicate task creation by setting flags keyed by task IDs.

**Edge Cases / Failure Types:**  
- Redis connectivity issues.  
- TTL expiration too short or too long causing missed updates or loops.

---

#### 1.6 Utility and Helper Nodes

**Overview:**  
These nodes handle field mapping, date formatting, execution data logging, markdown conversion, batching, and error handling.

**Nodes Involved:**  
- Map Notion to Todoist / Map Notion to Todoist1 / Map Notion to Todoist2 (Code)  
- Map Todoist to Notion / Map Todoist to Notion1 (Code)  
- Handle empty dates / Handle empty dates1 / Handle empty dates2 (Code)  
- Execution Data / Execution Data1 / Execution Data2 / Execution Data3 / Execution Data4 / Execution Data5 / Execution Data6 (Execution Data)  
- Prepare summary data / Prepare summary data1 / Prepare summary data2 (Set)  
- Merge summary (Merge)  
- Convert to HTML Table (HTML)  
- Turn Markdown into Notion Blocks (Code)  
- Handle each block separately (SplitOut)  
- Append Notion Block (Notion)  
- Pick Todoist Fields (Set)  
- Filter nodes for various conditions (Filter)  
- If nodes for branching logic (If)  
- Wait nodes for retry delays (Wait)  
- Stop and Error nodes for retry limits (StopAndError)  

**Node Details:**

- **Mapping nodes:** Convert between Notion and Todoist data models, including priority, status/section, due dates, and descriptions.
- **Handle empty dates:** Adjusts due date fields to comply with Todoist API requirements.
- **Execution Data nodes:** Store key data points for debugging and traceability.
- **Markdown conversion:** Converts Todoist markdown descriptions into Notion blocks for rich text.
- **Batching and merging:** Aggregate commands for Todoist Sync API with batching to respect rate limits.
- **Retry and error handling:** Implements retry logic with limits and waits for transient API errors.

**Edge Cases / Failure Types:**  
- Mapping errors due to missing or unexpected data.  
- Markdown parsing edge cases.  
- API rate limits and transient failures.  
- Data type mismatches causing expression errors.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                                         | Input Node(s)                            | Output Node(s)                          | Sticky Note                                                                                         |
|--------------------------------|-------------------------|---------------------------------------------------------|-----------------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------|
| Get projects                   | HTTP Request            | Retrieve Todoist projects                               | -                                       | Prep Dropdown1                        |                                                                                                   |
| Get sections                  | HTTP Request            | Retrieve Todoist sections                               | -                                       | Generate config                      |                                                                                                   |
| Get Notion Databases          | Notion                  | Retrieve all Notion databases                           | -                                       | Prep Dropdown                       |                                                                                                   |
| Prep Dropdown                 | Code                    | Prepare dropdown options for Notion databases          | Get Notion Databases                    | Choose Notion Database               |                                                                                                   |
| Prep Dropdown1                | Code                    | Prepare dropdown options for Todoist projects          | Get projects                           | Choose Todoist Project               |                                                                                                   |
| Generate config               | Code                    | Generate global config JSON                             | Get sections                          | Return config JSON                  |                                                                                                   |
| Choose Notion Database        | Form                    | User selects Notion database                            | Prep Dropdown                        | Get Notion Database ID              |                                                                                                   |
| Choose Todoist Project        | Form                    | User selects Todoist project                            | Prep Dropdown1                       | Get Todoist Project ID              |                                                                                                   |
| Get Notion Database ID        | Code                    | Extract selected Notion database ID                     | Choose Notion Database                | Generate config                    |                                                                                                   |
| Get Todoist Project ID        | Code                    | Extract selected Todoist project ID                     | Choose Todoist Project                | Generate config                    |                                                                                                   |
| Return config JSON            | Form                    | Output generated config JSON                            | Generate config                      | -                                  |                                                                                                   |
| Todoist Webhook Setup Helper  | Form Trigger            | Input Client ID and Secret for Todoist OAuth           | -                                   | Generate security token            |                                                                                                   |
| Generate security token       | Crypto                  | Generate OAuth state token                              | Todoist Webhook Setup Helper          | Store variables                   |                                                                                                   |
| Store variables               | Code                    | Store OAuth credentials and state                       | Generate security token               | Redirect to Auth Page             |                                                                                                   |
| Redirect to Auth Page         | Form                    | Redirect user to Todoist OAuth authorization           | Store variables                      | OAuth redirect                   |                                                                                                   |
| OAuth redirect               | Webhook                 | Receive OAuth callback                                  | Redirect to Auth Page                | Verify security token            |                                                                                                   |
| Verify security token         | If                      | Validate OAuth state token                              | OAuth redirect                      | Exchange Tokens / Respond with error |                                                                                                   |
| Exchange Tokens              | HTTP Request            | Exchange authorization code for access token           | Verify security token                | Respond with success / Respond with error |                                                                                                   |
| Respond with success          | Respond to Webhook      | Confirm OAuth success                                   | Exchange Tokens                    | -                                  |                                                                                                   |
| Respond with error            | Respond to Webhook      | Confirm OAuth failure                                   | Exchange Tokens                    | -                                  |                                                                                                   |
| Notion Webhook               | Webhook                 | Receive Notion webhook events                           | -                                   | Body is array?                   |                                                                                                   |
| Body is array?               | If                      | Check if webhook payload is array                       | Notion Webhook                     | Split out Notion changes / Notion trigger reference |                                                                                                   |
| Split out Notion changes     | SplitOut                | Split multiple Notion webhook events                    | Body is array?                     | Extract IDs                     |                                                                                                   |
| Extract IDs                  | Set                     | Extract Notion page IDs                                 | Split out Notion changes           | Notion trigger reference         |                                                                                                   |
| Notion trigger reference     | NoOp                    | Reference node for Notion trigger                       | Extract IDs                       | Globals2 / Execution Data5       |                                                                                                   |
| Globals2                     | Set                     | Global config for Notion to Todoist sync                | Notion trigger reference          | Execution Data5                 |                                                                                                   |
| Execution Data5              | Execution Data           | Store execution info for debugging                      | Globals2                         | Check if Notion ID is locked    |                                                                                                   |
| Check if Notion ID is locked | Redis                   | Check Redis lock for Notion task                        | Execution Data5                  | Only continue if not locked1    |                                                                                                   |
| Only continue if not locked1 | Filter                  | Continue only if task is not locked                     | Check if Notion ID is locked      | Get Notion task                |                                                                                                   |
| Get Notion task              | Notion                  | Retrieve Notion task details                            | Only continue if not locked1      | Map Notion to Todoist2          |                                                                                                   |
| Map Notion to Todoist2       | Code                    | Map Notion task properties to Todoist fields           | Get Notion task                  | Todoist ID exists?              | Keep in sync: if you update the mapping, make sure to change it in the other node as well!         |
| Todoist ID exists?           | If                      | Check if Todoist ID exists in Notion task               | Map Notion to Todoist2           | Set tries1 / Create task in Todoist1 |                                                                                                   |
| Set tries1                  | Set                     | Initialize retry counter                                | Todoist ID exists?               | Get Todoist Task2             |                                                                                                   |
| Get Todoist Task2           | Todoist                 | Retrieve Todoist task                                   | Set tries1                      | Differences exist / Catch known error1 |                                                                                                   |
| Differences exist           | Filter                  | Check if differences exist between Notion and Todoist  | Get Todoist Task2               | Requires completion change / Update task in Todoist1 |                                                                                                   |
| Requires completion change  | Filter                  | Check if completion status changed                      | Differences exist               | Has been completed?            |                                                                                                   |
| Has been completed?         | If                      | Check if task is completed                              | Requires completion change      | Mark as Completed in Todoist1 / Mark as Incomplete in Todoist1 |                                                                                                   |
| Mark as Completed in Todoist1 | Todoist               | Mark Todoist task as completed                          | Has been completed?             | Update task in Todoist1        |                                                                                                   |
| Mark as Incomplete in Todoist1 | Todoist              | Mark Todoist task as incomplete                         | Has not been completed?         | Update task in Todoist1        |                                                                                                   |
| Update task in Todoist1     | HTTP Request            | Update Todoist task with Notion data                    | Differences exist / Mark as Completed/Incompleted | -                          |                                                                                                   |
| Create task in Todoist1     | HTTP Request            | Create new Todoist task                                 | Todoist ID exists? (false)      | Store Todoist ID1             |                                                                                                   |
| Store Todoist ID1           | Notion                  | Store Todoist ID in Notion task                         | Create task in Todoist1         | Lock Todoist ID4              |                                                                                                   |
| Lock Todoist ID4            | Redis                   | Lock Todoist task ID in Redis                           | Store Todoist ID1               | -                            |                                                                                                   |
| Handle empty dates1         | Code                    | Adjust due date fields for Todoist API                  | Map Notion to Todoist1          | Update task in Todoist1        |                                                                                                   |
| Status changed but not empty | Filter                  | Check if task section changed and not empty             | Map Notion to Todoist1          | Generate UUID2                |                                                                                                   |
| Generate UUID2              | Crypto                  | Generate UUID for Todoist Sync API commands             | Status changed but not empty    | Update section (Sync API)2     |                                                                                                   |
| Update section (Sync API)2  | HTTP Request            | Update Todoist task section using Sync API              | Generate UUID2                 | -                            | Update section for each task in Todoist using the Sync API in full sync mode. Use batching and split into multiple batches, if needed, since there is a request limit of 100 full syncs within a 15 minute period. |
| Get Backlog Section ID      | Code                    | Get Todoist section ID for status                       | Set creating flag1              | Generate UUID1 / Create task in Notion |                                                                                                   |
| Create task in Notion       | Notion                  | Create new Notion page                                  | Get Backlog Section ID          | Execution Data2 / Turn Markdown into Notion Blocks |                                                                                                   |
| Mark as Focussed in Notion  | Notion                  | Set Focus checkbox in Notion task                       | Due date empty / Neither focussed nor planned1 | Lock Notion ID              |                                                                                                   |
| Mark as Done in Notion      | Notion                  | Set Notion task status to Done                          | Has been completed?             | Lock Notion ID               |                                                                                                   |
| Mark as Obsolete in Notion  | Notion                  | Set Notion task status to Obsolete                      | Differences exist / Status is Obsolete? | Lock Notion ID               | Instead of deleting pages in Notion, it is a safer practice to mark them as obsolete instead.      |
| Mark as In Progress in Notion | Notion                | Set Notion task status to In Progress                   | Has not been completed?         | Lock Notion ID               |                                                                                                   |
| Execution Data6             | Execution Data           | Store execution info for debugging                      | Notion Task found1              | -                            |                                                                                                   |
| Catch known error1          | If                      | Catch known Todoist API errors                          | Get Todoist Task2 / Update task in Todoist1 | Create task in Todoist / Wait1 |                                                                                                   |
| Wait1                      | Wait                    | Wait before retrying Todoist API call                   | Catch known error1              | Update tries1                |                                                                                                   |
| Retry limit reached1        | Stop and Error          | Stop workflow after max retries                         | If tries left1 (false)          | -                            |                                                                                                   |
| Todoist Webhook             | Webhook                 | Receive Todoist webhook events                          | -                             | Switch by project            |                                                                                                   |
| Switch by project           | Switch                  | Route Todoist events by project ID                      | Todoist Webhook                | Todoist trigger reference    |                                                                                                   |
| Todoist trigger reference   | NoOp                    | Reference node for Todoist webhook                      | Switch by project              | Execution Data / Globals1    |                                                                                                   |
| Execution Data              | Execution Data           | Store execution info for debugging                      | Todoist trigger reference      | Check if Todoist ID is locked |                                                                                                   |
| Globals1                    | Set                     | Global config for Todoist to Notion sync                | Todoist trigger reference      | -                            |                                                                                                   |
| Check if Todoist ID is locked | Redis                 | Check Redis lock for Todoist task                       | Execution Data                 | Only continue if not locked  |                                                                                                   |
| Only continue if not locked | Filter                  | Continue only if task is not locked                     | Check if Todoist ID is locked  | If event is not :deleted     |                                                                                                   |
| If event is not :deleted    | If                      | Filter out deleted events                               | Only continue if not locked    | Set tries2 / Get Notion Task2 |                                                                                                   |
| Set tries2                  | Set                     | Initialize retry counter                                | If event is not :deleted       | Get Notion Task2             |                                                                                                   |
| Get Notion Task2            | Notion                  | Retrieve Notion page linked to Todoist task            | Set tries2                    | Notion Task found1           |                                                                                                   |
| Notion Task found1          | Filter                  | Check if Notion page exists                             | Get Notion Task2              | Map Todoist to Notion1 / Check if creating flag exists1 |                                                                                                   |
| Map Todoist to Notion1      | Code                    | Map Todoist task properties to Notion fields           | Notion Task found1            | Differences exist1           | Keep in sync: if you update the mapping, make sure to change it in the other node as well!         |
| Differences exist1          | Filter                  | Check if differences exist between Todoist and Notion  | Map Todoist to Notion1        | Update task in Notion / Due date empty |                                                                                                   |
| Update task in Notion       | Notion                  | Update existing Notion page                             | Differences exist1            | Lock Notion ID              |                                                                                                   |
| Due date empty              | Filter                  | Check if due date is empty                              | Update task in Notion         | Mark as Focussed in Notion  |                                                                                                   |
| Mark as Focussed in Notion  | Notion                  | Set Focus checkbox in Notion task                       | Due date empty                | Lock Notion ID              |                                                                                                   |
| Mark as Done in Notion      | Notion                  | Set Notion task status to Done                          | Differences exist1            | Lock Notion ID              |                                                                                                   |
| Mark as Obsolete in Notion  | Notion                  | Set Notion task status to Obsolete                      | Differences exist1            | Lock Notion ID              |                                                                                                   |
| Mark as In Progress in Notion | Notion                | Set Notion task status to In Progress                   | Differences exist1            | Lock Notion ID              |                                                                                                   |
| Create task in Notion       | Notion                  | Create new Notion page                                 | Check if creating flag exists1 | Execution Data2 / Turn Markdown into Notion Blocks |                                                                                                   |
| Turn Markdown into Notion Blocks | Code                | Convert Todoist markdown description to Notion blocks  | Create task in Notion         | Handle each block separately |                                                                                                   |
| Handle each block separately | SplitOut                | Split Notion blocks for appending                       | Turn Markdown into Notion Blocks | Append Notion Block         |                                                                                                   |
| Append Notion Block         | Notion                  | Append blocks to Notion page                            | Handle each block separately  | Lock Notion ID              |                                                                                                   |
| Update Description in Todoist1 | Todoist               | Update Todoist task description with Notion URL        | Description has changed in Todoist | -                          |                                                                                                   |
| Description has changed in Todoist | Filter             | Detect if Todoist description differs from Notion URL  | Get Todoist Task1            | Update Description in Todoist1 |                                                                                                   |
| Generate UUID1              | Crypto                  | Generate UUID for Todoist Sync API commands             | Get Backlog Section ID        | Update section (Sync API)1   |                                                                                                   |
| Update section (Sync API)1  | HTTP Request            | Update Todoist task section using Sync API              | Generate UUID1               | -                            | Update section for in Todoist using the Sync API in full sync mode (**maximum of 100 requests within a 15 minute period!**) |
| Get Backlog Section ID      | Code                    | Get Todoist section ID for status                       | Map Todoist to Notion1        | Generate UUID1 / Create task in Notion |                                                                                                   |
| Check if creating flag exists1 | Redis                 | Check Redis flag to prevent duplicate Notion creation  | Notion Task found1            | Only continue if flag does not exist1 |                                                                                                   |
| Only continue if flag does not exist1 | Filter           | Continue only if no duplicate creation flag            | Check if creating flag exists1 | Set creating flag1           |                                                                                                   |
| Set creating flag1          | Redis                   | Set Redis flag to prevent duplicate Notion creation    | Only continue if flag does not exist1 | Create task in Notion       |                                                                                                   |
| Execution Data3             | Execution Data           | Store execution info for debugging                      | Notion Task not found         | -                            |                                                                                                   |
| Catch known error2          | If                      | Catch known Notion API errors                           | Get Notion Task2 / Differences exist1 | Wait2 / End here            |                                                                                                   |
| Wait2                      | Wait                    | Wait before retrying Notion API call                    | Catch known error2            | Update tries2                |                                                                                                   |
| Retry limit reached2        | Stop and Error          | Stop workflow after max retries                         | If tries left2 (false)         | -                            |                                                                                                   |
| End here                   | NoOp                    | End of retry loop for Notion API                        | Catch known error2            | -                            |                                                                                                   |
| Schedule Trigger            | Schedule Trigger        | Trigger full daily sync                                 | -                            | Notion / Todoist / Globals   | Set the preferred period. Usually once a day, likely outside working hours should be a solid option. |
| Notion                     | Notion                  | Fetch all open Notion tasks                             | Schedule Trigger              | Compare Datasets             |                                                                                                   |
| Todoist                    | Todoist                 | Fetch all open Todoist tasks                           | Schedule Trigger              | Compare Datasets             |                                                                                                   |
| Globals                    | Set                     | Global config for full sync                             | Schedule Trigger              | -                            |                                                                                                   |
| Compare Datasets           | Compare Datasets        | Compare Notion and Todoist tasks                        | Notion / Todoist              | Handle empty dates / Loop Over Items |                                                                                                   |
| Handle empty dates          | Code                    | Adjust due date fields for Todoist API                  | Compare Datasets              | Update Task in Todoist       |                                                                                                   |
| Handle empty dates2         | Code                    | Adjust due date fields for Todoist API                  | Compare Datasets              | Add project ID               |                                                                                                   |
| Loop Over Items            | SplitInBatches          | Process tasks in batches                                | Compare Datasets              | Exists/Completed in Notion   |                                                                                                   |
| Exists/Completed in Notion | If                      | Check if task exists and is completed                   | Loop Over Items               | Map Notion to Todoist1 / Delete Task in Todoist |                                                                                                   |
| Map Notion to Todoist1      | Code                    | Map Notion task to Todoist fields                       | Exists/Completed in Notion    | Delete Task in Todoist / Update Task in Todoist |                                                                                                   |
| Delete Task in Todoist      | Todoist                 | Delete Todoist task                                    | Exists/Completed in Notion    | Prepare summary data2        |                                                                                                   |
| Update Task in Todoist      | HTTP Request            | Update Todoist task                                    | Exists/Completed in Notion    | Status changed / Lock Todoist ID2 / Prepare summary data |                                                                                                   |
| Add project ID             | Set                     | Add project ID to Todoist task data                     | Handle empty dates2           | If Todoist ID exists         |                                                                                                   |
| If Todoist ID exists       | If                      | Check if Todoist ID exists                              | Add project ID               | Set tries / Create task in Todoist |                                                                                                   |
| Create task in Todoist      | HTTP Request            | Create new Todoist task                                | If Todoist ID exists (false)  | Store Todoist ID             |                                                                                                   |
| Lock Todoist ID            | Redis                   | Lock Todoist task ID in Redis                          | Update Task in Todoist        | -                            |                                                                                                   |
| Prepare summary data       | Set                     | Prepare data for email summary                         | Update Task in Todoist        | Merge summary               |                                                                                                   |
| Prepare summary data1      | Set                     | Prepare data for email summary                         | Mark as Completed in Todoist  | Merge summary               |                                                                                                   |
| Prepare summary data2      | Set                     | Prepare data for email summary                         | Delete Task in Todoist        | Merge summary               |                                                                                                   |
| Merge summary             | Merge                   | Merge all summary data                                 | Prepare summary data / Prepare summary data1 / Prepare summary data2 | Filter out status changes |                                                                                                   |
| Filter out status changes | Filter                  | Filter out status-only changes                         | Merge summary                | Map Todoist to Notion        |                                                                                                   |
| Map Todoist to Notion       | Code                    | Map Todoist task to Notion fields                      | Filter out status changes     | Map summary fields           |                                                                                                   |
| Map summary fields          | Set                     | Prepare final summary fields for email                 | Map Todoist to Notion         | Convert to HTML Table        |                                                                                                   |
| Convert to HTML Table       | HTML                    | Convert summary data to HTML table                      | Map summary fields           | Generate email body          |                                                                                                   |
| Generate email body         | HTML                    | Generate styled HTML email body                         | Convert to HTML Table         | Gmail                      |                                                                                                   |
| Gmail                      | Gmail                   | Send email report                                     | Generate email body           | -                            |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Setup Credentials:**
   - Create and configure credentials in n8n for:
     - Notion (access token with database read/write)
     - Todoist (access token with task and project scopes)
     - Redis (connection to Redis Cloud or self-hosted instance)
     - Gmail OAuth2 (for sending email reports)

2. **Create Setup Workflow:**
   - Add a **Form Trigger** node named "Notion-Todoist Sync Setup Helper" with a simple form to start.
   - Add **Notion** node to get all databases.
   - Add **Code** node to prepare dropdown options excluding "Inbox".
   - Add **Form** node "Choose Notion Database" with dropdown populated from previous node.
   - Add **HTTP Request** node to Todoist API to get projects.
   - Add **Code** node to prepare dropdown options excluding "Inbox".
   - Add **Form** node "Choose Todoist Project" with dropdown populated from previous node.
   - Add **HTTP Request** node to Todoist API to get sections filtered by project ID.
   - Add **Code** node "Generate config" to assemble JSON with database ID, project ID, and sections.
   - Add **Form** node "Return config JSON" to output the config.
   - Add **Form Trigger** node "Todoist Webhook Setup Helper" to input Client ID and Secret.
   - Add **Crypto** node to generate OAuth state token.
   - Add **Code** node to store OAuth variables in static data.
   - Add **Form** node "Redirect to Auth Page" to redirect user to Todoist OAuth.
   - Add **Webhook** node "OAuth redirect" to receive OAuth callback.
   - Add **If** node to verify OAuth state token.
   - Add **HTTP Request** node to exchange OAuth code for access token.
   - Add **Respond to Webhook** nodes for success and error responses.

3. **Create Globals Nodes:**
   - Add **Set** nodes named "Globals", "Globals1", and "Globals2" with the generated JSON config pasted from setup.

4. **Build Notion to Todoist Diff Sync:**
   - Add **Webhook** node "Notion Webhook" to receive Notion webhook events.
   - Add **If** node "Body is array?" to handle payload format.
   - Add **SplitOut** node to split multiple changes.
   - Add **Set** node to extract Notion page IDs.
   - Add **NoOp** node "Notion trigger reference".
   - Add **Set** node "Globals2" with global config.
   - Add **Execution Data** node for debugging.
   - Add **Redis** node to check if Notion ID is locked.
   - Add **Filter** node to continue only if not locked.
   - Add **Notion** node to get full Notion task.
   - Add **Code** node to map Notion task to Todoist fields.
   - Add **If** node to check if Todoist ID exists.
   - Add **Set** node to initialize retry counter.
   - Add **Todoist** node to get Todoist task.
   - Add **Filter** node to detect differences.
   - Add **Filter** node to check if completion status changed.
   - Add **If** node to check if task is completed.
   - Add **Todoist** nodes to mark task completed or reopened.
   - Add **HTTP Request** node to update Todoist task.
   - Add **HTTP Request** node to create Todoist task if missing.
   - Add **Redis** node to set creating flag.
   - Add **Todoist** node to delete task if obsolete.
   - Add **Notion** node to store Todoist ID in Notion task.
   - Add **Redis** node to lock Todoist ID.
   - Add **Code** node to handle empty dates.
   - Add **Filter** node to detect section changes.
   - Add **Crypto** node to generate UUID for Sync API.
   - Add **HTTP Request** node to update Todoist section using Sync API.
   - Add **Code** node to get backlog section ID.
   - Add **Notion** node to create Notion task if needed.
   - Add **Notion** nodes to mark task as Focussed, Done, Obsolete, or In Progress.
   - Add retry logic with **If**, **Wait**, and **Stop and Error** nodes.

5. **Build Todoist to Notion Diff Sync:**
   - Add **Webhook** node "Todoist Webhook" to receive Todoist webhook events.
   - Add **Switch** node to route by project ID.
   - Add **NoOp** node "Todoist trigger reference".
   - Add **Execution Data** node.
   - Add **Set** node "Globals1" with global config.
   - Add **Redis** node to check if Todoist ID is locked.
   - Add **Filter** node to continue only if not locked.
   - Add **If** node to filter out deleted events.
   - Add **Set** node to initialize retry counter.
   - Add **Notion** node to get Notion task by Todoist ID.
   - Add **Filter** node to check if Notion task found.
   - Add **Code** node to map Todoist task to Notion fields.
   - Add **Filter** node to detect differences.
   - Add **Notion** node to update Notion task.
   - Add **Filter** node to check empty due date.
   - Add **Notion** node to mark task as Focussed.
   - Add **Notion** nodes to mark task as Done, Obsolete, or In Progress.
   - Add **Notion** node to create task if not found.
   - Add **Code** node to convert Todoist markdown description to Notion blocks.
   - Add **SplitOut** node to handle blocks separately.
   - Add **Notion** node to append blocks.
   - Add **Filter** node to detect description changes.
   - Add **Todoist** node to update description.
   - Add **Crypto** node to generate UUID.
   - Add **HTTP Request** node to update Todoist section using Sync API.
   - Add **Redis** nodes to manage creation flags.
   - Add retry logic with **If**, **Wait**, and **Stop and Error** nodes.

6. **Build Full Daily Sync:**
   - Add **Schedule Trigger** node to run daily at 6 AM.
   - Add **Notion** node to fetch all open Notion tasks.
   - Add **Todoist** node to fetch all open Todoist tasks.
   - Add **Set** node "Globals" with global config.
   - Add **Compare Datasets** node to compare tasks.
   - Add **Code** nodes to handle empty dates.
   - Add **SplitInBatches** node to process tasks.
   - Add **If** node to check if task exists in Notion.
   - Add **Code** node to map Notion to Todoist.
   - Add **Todoist** nodes to delete or update tasks.
   - Add **Set** node to add project ID.
   - Add **If** node to check Todoist ID existence.
   - Add **HTTP Request** node to create tasks.
   - Add **Redis** node to lock Todoist ID.
   - Add **Set** nodes to prepare summary data.
   - Add **Merge** node to combine summaries.
   - Add **Filter** node to filter out status-only changes.
   - Add **Code** node to map Todoist to Notion.
   - Add **Set** node to prepare summary fields.
   - Add **HTML** nodes to convert to table and generate email body.
   - Add **Gmail** node to send email report.

7. **Add Redis Locking Nodes:**
   - Add Redis nodes to lock and check task IDs and creation flags throughout workflows.

8. **Add Utility Nodes:**
   - Add Code nodes for mapping, date handling, markdown conversion.
   - Add Execution Data nodes for debugging.
   - Add retry logic nodes (If, Wait, Stop and Error).

9. **Add Sticky Notes:**
   - Add sticky notes with setup instructions, mapping reminders, and usage notes as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Demo & Explanation video: [YouTube Demo](https://youtu.be/k66j6ZspjCg)                                                                                                                                                           | Demonstrates workflow functionality and setup                                                                  |
| Setup video: [YouTube Setup](https://youtu.be/73jhyU0t4c4)                                                                                                                                                                       | Step-by-step setup guide                                                                                        |
| Notion database template: [Notion Template](https://steadfast-banjo-d1f.notion.site/17682b476c848086b002de766879aa71)                                                                                                            | Pre-configured database with required properties                                                               |
| Redis free cloud instance: [Redis Cloud](https://redis.io/try-free/)                                                                                                                                                             | Recommended Redis hosting for locking mechanism                                                                |
| Todoist Developer App setup instructions included in sticky notes                                                                                                                                                               | Required for webhook activation and OAuth authentication                                                       |
| Mapping between Notion and Todoist fields is critical and must be kept in sync in both directions                                                                                                                              | See sticky notes "Keep in sync"                                                                                 |
| Limitations: No support for subtasks, recurring tasks, simultaneous edits within 15-20 seconds, or URLs in task names                                                                                                           | Important for user expectations                                                                                  |
| Use Redis locks to prevent infinite update loops caused by webhook triggers firing on both sides                                                                                                                               | Critical for stable operation                                                                                     |
| The workflow includes advanced retry logic for API calls to handle transient errors and rate limits                                                                                                                            | Ensures robustness                                                                                                |
| Recommended to split large flows into separate workflows for maintainability: Notion-Todoist Full Sync, Notion-Todoist Diff Sync, Todoist-Notion Diff Sync                                                                      | See sticky note "Finishing Touches"                                                                              |
| Email reports summarize daily sync changes and include direct links to tasks for manual fixes if needed                                                                                                                        | Improves transparency and manual intervention                                                                   |

---

This comprehensive reference document enables advanced users and AI agents to understand, reproduce, and maintain the "Realtime Notion Todoist 2-way Sync with Redis" workflow effectively.