Sync Notion Content to Webflow CMS as Draft Items with Status Tracking

https://n8nworkflows.xyz/workflows/sync-notion-content-to-webflow-cms-as-draft-items-with-status-tracking-7935


# Sync Notion Content to Webflow CMS as Draft Items with Status Tracking

### 1. Workflow Overview

This workflow automates the synchronization of content from a Notion database to a Webflow CMS collection, creating or updating items as draft entries while tracking their status in Notion. It is designed for content teams managing publishing workflows, ensuring content marked as "Ready for publish" in Notion is reflected in Webflow with proper draft status and status updates in Notion.

The workflow consists of the following logical blocks:

- **1.1 Trigger and Input Gathering**: Manual trigger initiates the workflow, fetching Notion pages filtered by status and project.
- **1.2 Content Extraction and Merging**: Retrieves Notion content blocks for each page, merges them into structured JSON content.
- **1.3 Webflow Content Lookup and Decision**: Retrieves existing Webflow CMS items to determine if a page’s content should create a new item or update an existing one.
- **1.4 Webflow Item Creation or Update**: Creates new draft items or updates existing ones in Webflow, ensuring content fields are synchronized.
- **1.5 Notion Status Update and Logging**: Updates the Notion page’s status to "5. Done" or "On Hold" based on operation outcome and logs the submission.
- **1.6 Execution Control and Throttling**: Manages loop batching and waits to control execution flow and API rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Input Gathering

- **Overview:**  
Initiates the workflow manually and fetches all Notion pages from a specific database where the status is "Ready for publish" and linked to a specified project.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Get Notion Pages  
  - Loop Over Items  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Input: None  
    - Output: Triggers "Get Notion Pages"  
    - Failures: None (manual triggered)  

  - **Get Notion Pages**  
    - Type: Notion API (databasePage - getAll)  
    - Role: Retrieves all pages matching two filters:  
      - Status property equals "Ready for publish"  
      - Project relation contains a specified project ID  
    - Config: Returns all matching pages from database "Tech Content Tasks" (ID in parameters)  
    - Credentials: Notion API credentials configured  
    - Output: List of Notion pages for processing  
    - Failures: API errors, auth errors, empty result set (handled by next node)  

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes each Notion page individually in batches (default batch size)  
    - Config: Does not reset batch index, processes every item once  
    - Input: List of Notion pages  
    - Output: One page per iteration for further processing  
    - Failures: None, but empty input will cause no iterations

---

#### 1.2 Content Extraction and Merging

- **Overview:**  
For each Notion page, fetches all content blocks, extracts JSON content, and merges it into a single structured JSON object for Webflow.

- **Nodes Involved:**  
  - Code (returns first item)  
  - Get Notion Block  
  - Merge Content  

- **Node Details:**

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Passes the current item’s JSON forward directly (identity function)  
    - Input: Single Notion page JSON from batch  
    - Output: Same JSON for use in next node  
    - Failures: Expression errors if input is malformed  

  - **Get Notion Block**  
    - Type: Notion API (block - getAll)  
    - Role: Retrieves all content blocks for the current Notion page by block ID = page ID  
    - Config: Returns all blocks, no pagination needed  
    - Input: Page ID from previous node  
    - Output: List of content blocks  
    - Failures: API errors, invalid block ID, auth errors  

  - **Merge Content**  
    - Type: Code (JavaScript)  
    - Role: Joins all JSON content strings from the blocks into one JSON object representing merged content fields  
    - Key Expressions: Parses concatenated content strings into JSON, extracts name, id, url, sector from the first node named "Code"  
    - Input: All content blocks JSON  
    - Output: Single JSON with mergedContent object and related metadata  
    - Failures: JSON parse errors if content is malformed or incomplete

---

#### 1.3 Webflow Content Lookup and Decision

- **Overview:**  
Fetches all existing Webflow CMS items from the target collection to check if the current Notion content already exists in Webflow, deciding to create new or update existing items.

- **Nodes Involved:**  
  - Get Webflow Items  
  - Update or Create (Switch node)  

- **Node Details:**

  - **Get Webflow Items**  
    - Type: HTTP Request  
    - Role: Retrieves all items from the specified Webflow collection via Webflow API  
    - Config: GET request to Webflow API endpoint for collection items  
    - Authentication: HTTP Bearer with configured token  
    - Output: JSON array of items for matching  
    - Failures: API rate limits, auth token expiry, network errors  

  - **Update or Create**  
    - Type: Switch  
    - Role: Compares Notion content’s name to names of Webflow items to decide:  
      - If name not found in Webflow, proceed to create new item  
      - If name found, proceed to update existing item  
    - Conditions: Uses array containment checks on Webflow items’ names  
    - Input: Webflow items and merged content name  
    - Output: Two branches: create or update  
    - Failures: Expression errors if data missing or malformed

---

#### 1.4 Webflow Item Creation or Update

- **Overview:**  
Based on the decision, either creates a new draft item in Webflow or updates an existing draft item with merged content fields.

- **Nodes Involved:**  
  - Create Webflow Item  
  - Update Webflow Item (Draft)1  

- **Node Details:**

  - **Create Webflow Item**  
    - Type: HTTP Request (POST)  
    - Role: Creates a new item in Webflow as draft with all fields from merged content  
    - Config: JSON body sets "isDraft": true, "isArchived": false, and fills fieldData with merged content fields  
    - Authentication: HTTP Bearer with valid Webflow token  
    - Output: Newly created Webflow item JSON  
    - Failures: Validation errors (missing fields), API limits, auth errors  

  - **Update Webflow Item (Draft)1**  
    - Type: HTTP Request (PATCH)  
    - Role: Updates existing Webflow item matching by name with merged content fields, ensures draft and unarchived status  
    - Config: PATCH to item ID found by filtering items by name, JSON body similar to create node  
    - Authentication: HTTP Bearer  
    - On Error: Continues on error to avoid halting workflow  
    - Failures: Item not found (filter returns empty), API errors, partial update failures

---

#### 1.5 Notion Status Update and Logging

- **Overview:**  
After Webflow synchronization, updates the Notion page status to "5. Done" on success or "On Hold" on partial failure; logs the submission details for auditing.

- **Nodes Involved:**  
  - Done (Notion page update)  
  - hold (Notion page update)  
  - Log Submission1  

- **Node Details:**

  - **Done**  
    - Type: Notion API (databasePage - update)  
    - Role: Sets Notion page status to "5. Done" for successful Webflow sync  
    - Config: Updates "Status|status" property  
    - Input: Page ID from merged content  
    - Failures: API errors, auth errors  

  - **hold**  
    - Type: Notion API (databasePage - update)  
    - Role: Sets Notion page status to "On Hold" for error cases or retries  
    - Failures: Same as above  

  - **Log Submission1**  
    - Type: Code  
    - Role: Logs summary details including Notion page ID, Webflow item ID, timestamp, and action ("submitted for review")  
    - Can be extended for notifications (Slack, email, etc.)  
    - Output: Log object for debugging or audit  
    - Failures: Script errors, logging system failures

---

#### 1.6 Execution Control and Throttling

- **Overview:**  
Manages execution pacing and batch processing to handle API limits and orderly processing.

- **Nodes Involved:**  
  - Wait  
  - Loop Over Items (continued)  

- **Node Details:**

  - **Wait**  
    - Type: Wait node  
    - Role: Pauses workflow execution for 1 second between iterations or after updates  
    - Webhook ID configured (for execution tracking)  
    - Failures: None generally, but long waits may affect throughput  

  - **Loop Over Items**  
    - Continues looping over Notion pages after delays and updates, controlling batch size and reset behavior.

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                               | Input Node(s)                 | Output Node(s)                             | Sticky Note                           |
|-----------------------------|-----------------------------|-----------------------------------------------|------------------------------|--------------------------------------------|-------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger              | Starts workflow manually                        |                              | Get Notion Pages                           |                                     |
| Get Notion Pages             | Notion (databasePage getAll)| Fetches Notion pages filtered by status/project| When clicking ‘Execute workflow’ | Loop Over Items                            |                                     |
| Loop Over Items              | SplitInBatches              | Processes each Notion page individually        | Get Notion Pages             | Code (on batch), Loop continuation         |                                     |
| Code                        | Code                        | Passes current Notion page JSON                 | Loop Over Items (batch)      | Get Notion Block                           |                                     |
| Get Notion Block             | Notion (block getAll)       | Fetches all content blocks for page            | Code                        | Merge Content                             |                                     |
| Merge Content               | Code                        | Merges Notion blocks content into JSON object  | Get Notion Block             | Get Webflow Items                         |                                     |
| Get Webflow Items            | HTTP Request                | Retrieves all current Webflow CMS items         | Merge Content               | Update or Create                          |                                     |
| Update or Create            | Switch                      | Decides to update or create Webflow item       | Get Webflow Items            | Create Webflow Item / Update Webflow Item (Draft)1 |                                     |
| Create Webflow Item          | HTTP Request (POST)         | Creates new draft item in Webflow               | Update or Create (create branch) | Done / hold                              |                                     |
| Update Webflow Item (Draft)1 | HTTP Request (PATCH)        | Updates existing draft item in Webflow          | Update or Create (update branch) | Done / hold                              |                                     |
| Done                        | Notion (databasePage update)| Marks Notion page status as "5. Done"           | Create Webflow Item / Update Webflow Item (Draft)1 | Log Submission1                        |                                     |
| hold                        | Notion (databasePage update)| Marks Notion page status as "On Hold"           | Create Webflow Item / Update Webflow Item (Draft)1 | Wait                                   |                                     |
| Log Submission1             | Code                        | Logs submission details                          | Done                        | Wait                                      | Optional: can integrate notifications such as Slack, email |
| Wait                        | Wait                        | Pauses execution 1 second between operations    | hold / Log Submission1       | Loop Over Items                            |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Type: Manual Trigger  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters needed

2. **Add Notion node to get pages**  
   - Type: Notion (databasePage - getAll)  
   - Name: `Get Notion Pages`  
   - Configure:  
     - Database ID: Notion database ID for "Tech Content Tasks"  
     - Filters:  
       - Status|status equals "Ready for publish"  
       - Project|relation contains project ID "83054fca8faf4590aa1d9046c8942086"  
     - Return All: true  
   - Credentials: Link to valid Notion API credentials  
   - Connect trigger node output to this node

3. **Add SplitInBatches node**  
   - Type: SplitInBatches  
   - Name: `Loop Over Items`  
   - Default batch size (e.g., 1)  
   - Connect `Get Notion Pages` output to this node

4. **Add Code node (identity)**  
   - Type: Code  
   - Name: `Code`  
   - JavaScript code: `return $input.first().json;`  
   - Connect `Loop Over Items` output to this node (default output)

5. **Add Notion node to get blocks**  
   - Type: Notion (block - getAll)  
   - Name: `Get Notion Block`  
   - Parameters:  
     - Block ID: Expression `{{$json.id}}` from Code node  
     - Return All: true  
   - Credentials: Same Notion credentials  
   - Connect `Code` output to this node

6. **Add Code node to merge content**  
   - Type: Code  
   - Name: `Merge Content`  
   - JavaScript code:  
     ```javascript
     const mergedString = $input.all().map(item => item.json.content || '').join('');
     const mergedJson = JSON.parse(mergedString);
     return [{
       json: {
         name: $('Code').first().json.name,
         id: $('Code').first().json.id,
         url: $('Code').first().json.url,
         sector: $('Code').first().json.property_sector,
         mergedContent: mergedJson
       }
     }];
     ```  
   - Connect `Get Notion Block` output to this node

7. **Add HTTP Request node to get Webflow items**  
   - Type: HTTP Request  
   - Name: `Get Webflow Items`  
   - Method: GET  
   - URL: `https://api.webflow.com/v2/collections/{collection_id}/items` (replace with actual collection ID)  
   - Authentication: HTTP Bearer with Webflow API token  
   - Connect `Merge Content` output to this node

8. **Add Switch node to decide update or create**  
   - Type: Switch  
   - Name: `Update or Create`  
   - Rules:  
     - Condition 1 (Create): Webflow items’ names do NOT contain current merged content name  
     - Condition 2 (Update): Webflow items’ names contain current merged content name  
   - Connect `Get Webflow Items` output to this node

9. **Add HTTP Request node to create Webflow item**  
   - Type: HTTP Request (POST)  
   - Name: `Create Webflow Item`  
   - URL: `https://api.webflow.com/v2/collections/{collection_id}/items`  
   - Body: JSON with `"isDraft": true`, `"isArchived": false`, and `fieldData` populated with mergedContent fields (use expressions for each)  
   - Authentication: HTTP Bearer with Webflow token  
   - Connect "Create" branch of switch to this node  
   - On error: Continue (optional)  

10. **Add HTTP Request node to update Webflow item**  
    - Type: HTTP Request (PATCH)  
    - Name: `Update Webflow Item (Draft)1`  
    - URL: Construct with existing item ID filtered by name  
    - Body: Similar to create node, with draft and archived flags  
    - Authentication: HTTP Bearer  
    - Connect "Update" branch of switch to this node  
    - On error: Continue  

11. **Add Notion node to mark done**  
    - Type: Notion (databasePage - update)  
    - Name: `Done`  
    - Update property "Status|status" to "5. Done"  
    - Page ID: Expression from merged content (`{{$json.id}}`)  
    - Credentials: Notion API  
    - Connect outputs of both create and update Webflow nodes to this node

12. **Add Notion node to mark on hold**  
    - Type: Notion (databasePage - update)  
    - Name: `hold`  
    - Update property "Status|status" to "On Hold"  
    - Connect outputs of create and update nodes as alternate path (e.g., on failure or manual routing) to this node  

13. **Add Code node to log submission**  
    - Type: Code  
    - Name: `Log Submission1`  
    - JavaScript code logs page ID, item ID, timestamp, and action for audit  
    - Connect `Done` output to this node

14. **Add Wait node**  
    - Type: Wait  
    - Name: `Wait`  
    - Configure delay: 1 second  
    - Connect `hold` and `Log Submission1` outputs to this node

15. **Connect Wait node output back to Loop Over Items for next batch**

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow is tagged "Nected" and named "Content update"                                          | Internal project or client-specific naming                                                                         |
| Webflow API documentation for collections and items: https://developers.webflow.com/#collections-items | Useful for understanding API endpoints and data structure                                                         |
| Notion API documentation: https://developers.notion.com/reference/intro                                | For understanding database and block operations                                                                  |
| Optional: The "Log Submission1" node can be extended to integrate Slack, email, or other notification channels | Enhances workflow monitoring and alerting                                                                          |

---

Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.