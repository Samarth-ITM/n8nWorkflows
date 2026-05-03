Linear Project to Productboard feature Sync

https://n8nworkflows.xyz/workflows/linear-project-to-productboard-feature-sync-2578


# Linear Project to Productboard feature Sync

### 1. Workflow Overview

This workflow synchronizes project status and target end date information from Linear projects to corresponding features in Productboard, ensuring alignment across teams. It listens for updates on specified Linear projects, maps Linear’s project statuses to Productboard’s feature statuses, updates the feature details in Productboard accordingly (including timeframe), and finally sends a Slack notification summarizing the changes.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception (Linear Webhook Triggers):** Captures project update events from Linear for configured teams.
- **1.2 Linear Project Data Extraction:** Parses incoming Linear project data to extract key details like project URL, ID, status, start and target dates.
- **1.3 Productboard Feature Identification:** Uses the Linear project URL to find the matching Productboard feature via a custom field.
- **1.4 Data Transformation (Status Mapping):** Converts Linear project statuses into corresponding Productboard feature statuses.
- **1.5 Productboard Feature Details Retrieval:** Fetches current feature details from Productboard to compare with new data.
- **1.6 Conditional Update Decision:** Checks if there is a meaningful difference between Linear and Productboard data to determine if an update is needed.
- **1.7 Productboard Feature Update:** If changes are detected, updates the Productboard feature’s status and timeframe.
- **1.8 Slack Notification:** Sends a formatted Slack message notifying about the update with feature details and a link.


---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception (Linear Webhook Triggers)
- **Overview:** Listens for project update events from Linear for multiple teams.
- **Nodes Involved:**  
  - Your Linear Project 1  
  - Your Linear Project 2

- **Node Details:**

  - **Your Linear Project 1 & 2**  
    - Type: `linearTrigger`  
    - Role: Webhook triggers that receive project update events from Linear API for specified teams.  
    - Configurations: Each node is configured with a Linear team ID and listens to the "project" resource.  
    - Credentials: Uses configured Linear API credentials.  
    - Inputs: External webhook calls from Linear when a project is updated.  
    - Outputs: Project update data JSON passed downstream.  
    - Edge Cases: Webhook misconfiguration, invalid credentials, or team ID mismatch can cause triggers to fail.  
    - Version: n8n’s linearTrigger node v1.

---

#### 1.2 Linear Project Data Extraction
- **Overview:** Extracts useful fields from the Linear project update payload, including project URL, ID, status, and dates.
- **Nodes Involved:**  
  - linear project id

- **Node Details:**

  - **linear project id**  
    - Type: `Set`  
    - Role: Parses the incoming JSON payload to create structured fields:  
      - `linear_project_url` (full URL from payload)  
      - `linear_project_id` (extracted from URL by splitting string)  
      - `linear_project_status` (project status name)  
      - `startDate` and `targetDate` (dates from project data)  
    - Configuration: Uses JavaScript expressions to extract and transform data from `$json`.  
    - Inputs: JSON from Linear webhook triggers.  
    - Outputs: Structured JSON with extracted project fields.  
    - Edge Cases: If URL format changes or fields are missing, extraction may fail. Expressions should be validated.  
    - Version: Set node v3.2.

---

#### 1.3 Productboard Feature Identification
- **Overview:** Searches Productboard for the feature that corresponds to the Linear project by matching a custom field value.
- **Nodes Involved:**  
  - get productboard feature id  
  - Split Out  
  - Edit Fields  
  - Merge1

- **Node Details:**

  - **get productboard feature id**  
    - Type: `HTTP Request`  
    - Role: Queries Productboard API’s custom fields endpoint to retrieve features matching the Linear project URL stored in a custom field.  
    - Configuration:  
      - Method: GET  
      - Authentication: HTTP Header Auth with Productboard API credentials  
      - Query: Filters by a custom field UUID (must be configured)  
      - Headers: Content-Type application/json, X-Version 1  
    - Inputs: Linear project ID and URL  
    - Outputs: JSON response of matching features  
    - Edge Cases: Invalid or missing Productboard credentials, wrong custom field UUID, API rate limits.  
    - Version: HTTP Request node v4.1

  - **Split Out**  
    - Type: `Split Out`  
    - Role: Splits the response array under `data` to individual items for processing.  
    - Inputs: JSON array from previous node  
    - Outputs: Individual feature JSON objects  
    - Edge Cases: Empty or malformed data array.

  - **Edit Fields**  
    - Type: `Set`  
    - Role: Extracts the Linear project URL substring from the custom field value and extracts Productboard feature ID from the response.  
    - Configuration: Uses regex match on `value` field to get URL, sets `feature_id` from `hierarchyEntity.id`.  
    - Inputs: Individual feature JSON from Split Out  
    - Outputs: Structured JSON with `linear_url_productboard` and `feature_id` fields.

  - **Merge1**  
    - Type: `Merge`  
    - Role: Combines data streams by matching on Linear project URL and Productboard URL fields.  
    - Config: Merge by fields `linear_project_url` and `linear_url_productboard`  
    - Inputs: From `linear project id` and `Edit Fields` nodes  
    - Outputs: Merged JSON containing both Linear and Productboard data for matching projects/features.

---

#### 1.4 Data Transformation (Status Mapping)
- **Overview:** Maps Linear project statuses to corresponding Productboard feature statuses for consistent status representation.
- **Nodes Involved:**  
  - map linear to productboard status  
  - mapping  
  - Merge

- **Node Details:**

  - **map linear to productboard status**  
    - Type: `Set`  
    - Role: Passes the `linear_project_status` field downstream as `linear_status`.  
    - Inputs: Merged JSON from Merge1  
    - Outputs: JSON with `linear_status` field set.

  - **mapping**  
    - Type: `Code` (JavaScript)  
    - Role: Converts Linear statuses to Productboard statuses using a switch-case mapping:  
      - "Backlog" → "Candidate"  
      - "Planned" or "Paused" → "Planned"  
      - "In Progress" → "In progress"  
      - "Completed" → "Released"  
      - "Canceled" → "Won't do"  
      - Default → "Candidate"  
    - Runs once per item to process each status.  
    - Outputs: JSON with `productboard_status` field.  
    - Edge Cases: Unknown or unexpected statuses default to "Candidate".

  - **Merge**  
    - Type: `Merge`  
    - Role: Combines mapped status output with original merged data by position (index).  
    - Inputs: From `mapping` and previous merged data streams.  
    - Outputs: Enriched JSON with both Linear and Productboard status data.

---

#### 1.5 Productboard Feature Details Retrieval
- **Overview:** Retrieves full current details of the Productboard feature to compare with new status and dates.
- **Nodes Involved:**  
  - get productboard feature details  
  - Merge2

- **Node Details:**

  - **get productboard feature details**  
    - Type: `HTTP Request`  
    - Role: GET request to Productboard API to fetch detailed feature data by `feature_id`.  
    - Config:  
      - Method: GET  
      - Authentication: Productboard API credentials  
      - Headers: Content-Type application/json, X-Version 1  
      - URL constructed dynamically by feature ID.  
    - Inputs: Feature IDs from merged data.  
    - Outputs: Full feature JSON data.  
    - Edge Cases: Feature ID invalid, API errors, network issues.

  - **Merge2**  
    - Type: `Merge`  
    - Role: Combines feature details with previously merged data, joining on `feature_id` and `data.id`.  
    - Inputs: From `get productboard feature details` and previous merged data.  
    - Outputs: Complete JSON containing Linear data, mapped status, and current Productboard feature details.

---

#### 1.6 Conditional Update Decision
- **Overview:** Checks if the mapped Productboard status or calculated end date differ from current Productboard data to decide if an update is necessary.
- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - Type: `If`  
    - Role: Evaluates two conditions combined with OR:  
      1. Productboard status from mapping differs from current feature status.  
      2. Calculated end date from Linear `targetDate` differs from Productboard feature’s `timeframe.endDate`.  
    - Logic:  
      - For dates, calculates last day of month from Linear `targetDate`.  
      - Performs string comparison.  
    - Inputs: Merged JSON with all necessary fields.  
    - Outputs:  
      - `true` path if update is required.  
      - `false` path if no update needed.  
    - Edge Cases: Missing dates, null/undefined values, string comparison mismatches.

---

#### 1.7 Productboard Feature Update
- **Overview:** Sends a PATCH request to Productboard API to update the feature’s status and timeframe based on Linear data.
- **Nodes Involved:**  
  - update productboard status & timeframe

- **Node Details:**

  - **update productboard status & timeframe**  
    - Type: `HTTP Request`  
    - Role: PATCH request to Productboard API to update feature details:  
      - Status name updated to mapped Productboard status.  
      - Timeframe updated with granularity and calculated start and end dates based on Linear `targetDate`.  
    - Request body uses dynamic expressions to format dates:  
      - If `targetDate` exists, startDate is first of month, endDate is last day of month.  
      - If no `targetDate`, timeframe granularity and dates are set to "none".  
    - Config:  
      - Batching enabled with batch size 1 and 2s interval to avoid rate limits.  
      - Authentication: Productboard API credentials.  
      - Headers: X-Version 1, Accept application/json.  
    - Inputs: Conditioned JSON after `If` node.  
    - Outputs: API response confirming update.  
    - Edge Cases: API errors, rate limits, invalid data formats, auth failures.

---

#### 1.8 Slack Notification
- **Overview:** Sends a Slack message to notify the team about the feature update with status and timeframe details and a link to Productboard.
- **Nodes Involved:**  
  - Slack

- **Node Details:**

  - **Slack**  
    - Type: `Slack` node  
    - Role: Posts a block message in a configured Slack channel summarizing the Productboard feature update.  
    - Message includes:  
      - Linear and Productboard emojis for visual context (:linear:, :productboard:)  
      - Feature name, updated status, and formatted timeframe end date (e.g., "December 2024")  
      - Button linking directly to the Productboard feature page.  
    - Configuration:  
      - Uses Slack credentials (OAuth2)  
      - Channel set to `#product-notifications` (configurable)  
      - Uses Slack Block Kit for rich formatting.  
    - Inputs: Data from updated Productboard feature node.  
    - Outputs: Slack API response.  
    - Edge Cases: Slack API failures, invalid channel, credential expiry.

---

### 3. Summary Table

| Node Name                          | Node Type         | Functional Role                                | Input Node(s)                         | Output Node(s)                                    | Sticky Note                                                                                     |
|-----------------------------------|-------------------|-----------------------------------------------|-------------------------------------|--------------------------------------------------|------------------------------------------------------------------------------------------------|
| Your Linear Project 1              | linearTrigger     | Receive Linear project updates (team 1)       | -                                   | get productboard feature id, linear project id    |                                                                                                |
| Your Linear Project 2              | linearTrigger     | Receive Linear project updates (team 2)       | -                                   | linear project id, get productboard feature id    |                                                                                                |
| linear project id                 | Set               | Extract project URL, ID, status, dates         | Your Linear Project 1/2              | Merge1                                            |                                                                                                |
| get productboard feature id       | HTTP Request      | Fetch Productboard feature(s) by custom field | Your Linear Project 1 / linear project id | Split Out                                         | Fetches the Productboard feature ID using a custom field value.                                 |
| Split Out                        | Split Out         | Split Productboard response array to items     | get productboard feature id          | Edit Fields                                       |                                                                                                |
| Edit Fields                     | Set               | Extract Linear URL substring & feature ID      | Split Out                          | Merge1                                            |                                                                                                |
| Merge1                          | Merge             | Merge Linear project data with Productboard feature data | linear project id, Edit Fields        | map linear to productboard status, get productboard feature details |                                                                                                |
| map linear to productboard status | Set               | Prepare Linear status for mapping               | Merge1                              | mapping                                           |                                                                                                |
| mapping                         | Code              | Map Linear project status to Productboard status | map linear to productboard status   | Merge                                            |                                                                                                |
| Merge                           | Merge             | Combine mapped status with merged data          | mapping, Merge1                    | get productboard feature details                   |                                                                                                |
| get productboard feature details  | HTTP Request      | Get full Productboard feature details           | Merge                             | Merge2                                            |                                                                                                |
| Merge2                          | Merge             | Merge feature details with previous data        | get productboard feature details, Merge | If                                                |                                                                                                |
| If                             | If                | Check if status or end date differs               | Merge2                            | update productboard status & timeframe (true), else none |                                                                                                |
| update productboard status & timeframe | HTTP Request      | Update Productboard feature status and timeframe | If (true)                        | Slack                                             |                                                                                                |
| Slack                          | Slack             | Notify team in Slack about the update             | update productboard status & timeframe | -                                                |                                                                                                |
| Sticky Note                    | Sticky Note       | Tips for configuration                           | -                                 | -                                                | ## Tips\n- Avoid copying and pasting the Linear node; instead, add a new one from the menu.\n- Remember to configure the custom Productboard field in the "Get Productboard Feature ID" node. |
| Sticky Note1                   | Sticky Note       | Preview of Slack message format                   | -                                 | -                                                | ## Preview Slack Message\n:linear: to :productboard: update\nMy awesome feature name\nStatus: Candidate\n:dart: date: Decembre 2024\nYou can view the update in Productboard using the link below:\n<link productboard feature> |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Linear Webhook Triggers**  
   - Add two `Linear Trigger` nodes named "Your Linear Project 1" and "Your Linear Project 2".  
   - Configure each with the respective Linear team ID to listen for project updates (`resources=["project"]`).  
   - Connect your Linear API credentials to both nodes.  
   - These nodes will receive webhook calls from Linear when projects are updated.

2. **Extract Linear Project Data**  
   - Add a `Set` node called "linear project id".  
   - Use expressions to extract and set fields from incoming JSON:  
     - `linear_project_url = {{$json.url}}`  
     - `linear_project_id = {{$json.url.split('https://linear.app/<your company>/project/')[1]}}`  
     - `linear_project_status = {{$json.data.status.name}}`  
     - `startDate = {{$json.data.startDate}}`  
     - `targetDate = {{$json.data.targetDate}}`  
   - Connect both Linear triggers to this node.

3. **Fetch Productboard Features by Custom Field**  
   - Add an `HTTP Request` node named "get productboard feature id".  
   - Set method to GET.  
   - URL: `https://api.productboard.com/hierarchy-entities/custom-fields-values`  
   - Add query parameter: `customField.id` set to the UUID of the custom field in Productboard that holds the Linear project URL.  
   - Set headers: `Content-Type: application/json` and `X-Version: 1`.  
   - Use HTTP Header Authentication with your Productboard API credentials.  
   - Connect "Your Linear Project 1" and "linear project id" nodes to this node as per the original workflow.

4. **Split Productboard Response**  
   - Add a `Split Out` node named "Split Out".  
   - Configure to split the `data` array in the response.  
   - Connect "get productboard feature id" to "Split Out".

5. **Extract Productboard Data Fields**  
   - Add a `Set` node called "Edit Fields".  
   - Extract the Linear project URL substring from the custom field value using regex:  
     - `linear_url_productboard = {{$json['value'].match('^(https:\\/\\/linear\\.app\\/[^\\/]+\\/project\\/[^\\/]+)')[0]}}`  
   - Also set `feature_id` from `hierarchyEntity.id` field.  
   - Connect "Split Out" to this node.

6. **Merge Linear and Productboard Data**  
   - Add a `Merge` node called "Merge1".  
   - Configure to merge by fields: `linear_project_url` (from linear project id node) and `linear_url_productboard` (from Edit Fields).  
   - Connect "linear project id" and "Edit Fields" to this node.

7. **Prepare for Status Mapping**  
   - Add a `Set` node named "map linear to productboard status".  
   - Set field `linear_status` to `{{$json.linear_project_status}}`.  
   - Connect "Merge1" to this node.

8. **Map Linear Status to Productboard Status**  
   - Add a `Code` node named "mapping".  
   - Use the following JavaScript code to map statuses:

   ```js
   const linearStatus = $json.linear_status;
   let productboardStatus;

   switch(linearStatus) {
     case 'Backlog':
       productboardStatus = 'Candidate';
       break;
     case 'Planned':
     case 'Paused':
       productboardStatus = 'Planned';
       break;
     case 'In Progress':
       productboardStatus = 'In progress';
       break;
     case 'Completed':
       productboardStatus = 'Released';
       break;
     case 'Canceled':
       productboardStatus = "Won't do";
       break;
     default:
       productboardStatus = 'Candidate';
   }

   return { productboard_status: productboardStatus };
   ```

   - Set mode to "runOnceForEachItem".  
   - Connect "map linear to productboard status" to this node.

9. **Merge Mapped Status with Data**  
   - Add a `Merge` node named "Merge".  
   - Configure to merge by position (combine).  
   - Connect "mapping" and "Merge1" to this node.

10. **Fetch Productboard Feature Details**  
    - Add an `HTTP Request` node called "get productboard feature details".  
    - Method: GET  
    - URL: `https://api.productboard.com/features/{{$json.feature_id}}`  
    - Headers: `Content-Type: application/json`, `X-Version: 1`  
    - Authentication: HTTP Header Auth with Productboard API credentials.  
    - Connect "Merge" to this node.

11. **Merge Feature Details**  
    - Add a `Merge` node named "Merge2".  
    - Configure to merge by fields: `feature_id` (from previous data) and `data.id` (from feature details).  
    - Connect "get productboard feature details" and "Merge" to this node.

12. **Check If Update Is Needed**  
    - Add an `If` node named "If".  
    - Configure two OR conditions:  
      - Check if mapped `productboard_status` differs from current `data.status.name`.  
      - Check if calculated end date from Linear `targetDate` differs from `data.timeframe.endDate`.  
      - Date calculation: extract year and month from `targetDate`, get last day of month, format as `YYYY-MM-DD`.  
    - Connect "Merge2" to this node.

13. **Update Productboard Feature**  
    - Add an `HTTP Request` node named "update productboard status & timeframe".  
    - Method: PATCH  
    - URL: `https://api.productboard.com/features/{{$json.feature_id}}`  
    - Headers: `X-Version: 1`, `accept: application/json`  
    - Authentication: HTTP Header Auth with Productboard credentials.  
    - Body (JSON) includes:  
      ```json
      {
        "data": {
          "status": {
            "name": "{{$json.productboard_status}}"
          },
          "timeframe": {
            "granularity": "{{$json.targetDate ? 'month' : 'none'}}",
            "startDate": "{{$json.targetDate ? $json.targetDate.substring(0,7) + '-01' : 'none'}}",
            "endDate": "{{$json.targetDate ? (() => { const d=new Date($json.targetDate); const y=d.getFullYear(); const m=d.getMonth()+1; const lastDay=new Date(y,m,0).getDate(); return `${y}-${m.toString().padStart(2,'0')}-${lastDay}`; })() : 'none'}}"
          }
        }
      }
      ```
    - Enable batching with size 1 and 2000ms interval.  
    - Connect `If` node’s true output to this node.

14. **Send Slack Notification**  
    - Add a `Slack` node named "Slack".  
    - Configure with Slack credentials (OAuth2).  
    - Channel: Set to your desired channel (e.g., `#product-notifications`).  
    - Message Type: Block  
    - Message content uses Slack Block Kit JSON with dynamic fields:  
      - Feature name, status, formatted timeframe end date, and a button linking to the Productboard feature URL.  
    - Connect "update productboard status & timeframe" to this node.

15. **Activate the Workflow**  
    - Verify all credentials are properly set (Linear API, Productboard API, Slack API).  
    - Test each node step by step.  
    - Activate the workflow to enable automatic syncing on Linear project updates.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Tips: Avoid copying and pasting the Linear trigger node; add a new one from the menu instead to ensure proper webhook registration. Remember to configure the Productboard custom field UUID in the "get productboard feature id" node. | Sticky Note near Linear and Productboard feature ID nodes.                                                                            |
| Preview Slack Message format: Shows how Slack notification will look including emojis, status, date, and link.                                                                                                                  | Sticky Note near Slack node.                                                                                                          |
| Productboard API documentation: https://developer.productboard.com/reference/                                                                                                                                                | Useful for customizing API requests and understanding field schemas.                                                                 |
| Linear API documentation: https://developers.linear.app/docs/graphql/                                                                                                                                                | Helpful for configuring triggers and understanding project payloads.                                                                  |
| Slack Block Kit Builder: https://app.slack.com/block-kit-builder/                                                                                                                                                              | Useful for customizing Slack message formatting.                                                                                       |

---

This document provides a detailed, stepwise reference to understand, modify, or reproduce the entire Linear to Productboard synchronization workflow in n8n, including error considerations and integration points.