Sync new data between two apps

https://n8nworkflows.xyz/workflows/sync-new-data-between-two-apps-1930


# Sync new data between two apps

### 1. Workflow Overview

This workflow automates the synchronization of new qualified lead data from a **Postgres database** to a **Google Sheets** spreadsheet. It is designed to:

- Detect new or updated user entries in a Postgres table.
- Filter out unwanted records (specifically excluding emails containing `@n8n.io`).
- Append or update the qualified leads in a Google Sheets document.

The logical flow is divided into three main blocks:

- **1.1 Trigger Block:** Listens for new or updated records in the Postgres `users` table.
- **1.2 Filtering and Data Preparation Block:** Filters out internal emails and prepares user data.
- **1.3 Data Synchronization Block:** Saves the filtered user data to a Google Sheets file, appending new rows or updating existing ones based on a unique identifier.

Additional nodes facilitate manual testing and provide explanatory notes to guide users in customizing the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

- **Overview:**  
  Listens for changes in the Postgres `users` table to detect new or updated user records that should be synchronized.

- **Nodes Involved:**  
  - Postgres Trigger  
  - (Optional testing nodes: Manual Trigger, Code)

- **Node Details:**

  - **Postgres Trigger**  
    - Type: `postgresTrigger`  
    - Role: Watches for UPDATE events on the `users` table in a Postgres database.  
    - Configuration:  
      - Trigger event: `UPDATE` on the `users` table.  
      - Disabled by default (requires enabling for production use).  
      - Uses Postgres credentials named "Postgres Product Analytics".  
    - Input/Output: No input; outputs new/updated user records when triggered.  
    - Edge Cases:  
      - Disabled status means no automatic triggering unless enabled.  
      - Database connection/authentication errors possible.  
      - Possible missed events if Postgres notifications are not configured correctly.  
    - Notes: This node represents the live trigger for syncing data.

  - **Manual Trigger ("On clicking \"Execute Node\"")**  
    - Type: `manualTrigger`  
    - Role: Allows manual execution of the workflow for testing.  
    - Configuration: No special parameters.  
    - Input/Output: No input; triggers downstream nodes when manually executed.  
    - Edge Cases: None typical; used only in testing context.

  - **Code (Mock Data)**  
    - Type: `code`  
    - Role: Provides sample user data for manual testing.  
    - Configuration:  
      - JavaScript code returns a single user object with fields: id, username, email, company_size, role, users.  
    - Input/Output: No input; outputs mock user data for downstream nodes.  
    - Edge Cases: Mock data may not reflect real data schema perfectly.

#### 1.2 Filtering and Data Preparation Block

- **Overview:**  
  Filters the user data to exclude those with emails containing `@n8n.io`, ensuring only external qualified leads proceed for syncing.

- **Nodes Involved:**  
  - Filter

- **Node Details:**

  - **Filter**  
    - Type: `filter`  
    - Role: Filters out user records with emails containing the substring `@n8n.io`.  
    - Configuration:  
      - Condition: `$json.email` does **not** contain `n8n.io`.  
    - Input: Receives user data from either Postgres Trigger or Code node.  
    - Output: Passes filtered user data to the Google Sheets node.  
    - Edge Cases:  
      - If input data lacks the `email` field, expression may fail or exclude the record unintentionally.  
      - Case sensitivity in the filter is not specified; assumes case-sensitive match.  
      - No fallback output is configured for filtered-out records.

#### 1.3 Data Synchronization Block

- **Overview:**  
  Appends new or updates existing qualified lead records in a Google Sheets document under the sheet "Users to contact".

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**  
    - Type: `googleSheets`  
    - Role: Writes filtered user data to a Google Sheets spreadsheet, appending or updating rows based on the `id` field.  
    - Configuration:  
      - Operation: `appendOrUpdate`  
      - Document ID: `1gVfyernVtgYXD-oPboxOSJYQ-HEfAguEryZ7gTtK0V8`  
      - Sheet Name: `gid=0` (default first sheet)  
      - Columns mapped: `id`, `email`, `username` from incoming JSON data fields.  
      - Cell format: `USER_ENTERED` (allows Google Sheets to interpret data formats).  
      - Matching column for update: `id` (unique identifier to avoid duplicate rows).  
      - Uses `Google Sheets OAuth2` credentials named "Google Sheets account".  
    - Input: Accepts filtered user data from the Filter node.  
    - Output: Data output is standard success or error response from Google Sheets API.  
    - Edge Cases:  
      - Authentication failures if OAuth token expires or is invalid.  
      - API rate limits or quota exceeded errors.  
      - Mismatched document ID or sheet name causing write failures.  
      - Data format mismatches or unexpected field values could cause insertion errors.

---

### 3. Summary Table

| Node Name             | Node Type                | Functional Role                          | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                                                                |
|-----------------------|--------------------------|----------------------------------------|---------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Postgres Trigger      | postgresTrigger          | Detect new or updated users in Postgres | (none)                    | Filter                  | ### 1. Trigger step listens for new events                                                                                                |
| Filter                | filter                   | Filter out emails containing `@n8n.io` | Postgres Trigger, Code    | Google Sheets           | ### 2. Filter and transform your data. Excludes `@n8n.io` emails. See notes for alternatives like Set, ItemList, Code nodes.               |
| Google Sheets         | googleSheets             | Append or update users in Google Sheets | Filter                    | (none)                  | ### 3. Save the user in a Google Sheet. Can be replaced by other services like Excel, HubSpot, Pipedrive, Zendesk, etc.                    |
| On clicking "Execute Node" | manualTrigger        | Manual workflow trigger for testing    | (none)                    | Code                    | For testing the workflow                                                                                                                   |
| Code                  | code                     | Provides mock user data for testing    | On clicking "Execute Node" | Filter                  | Mock Data                                                                                                                                  |
| Sticky Note1          | stickyNote               | Explanation and setup instructions     | (none)                    | (none)                  | ## 👋 How to use this template... (setup instructions and customization guide)                                                             |
| Sticky Note           | stickyNote               | Notes on trigger step                   | (none)                    | (none)                  | ### 1. Trigger step listens for new events                                                                                                |
| Sticky Note6          | stickyNote               | Notes on filtering and data transformation | (none)                 | (none)                  | ### 2. Filter and transform your data... explanations and links to other transformation nodes                                              |
| Sticky Note2          | stickyNote               | Notes on Google Sheets node function    | (none)                    | (none)                  | ### 3. Save the user in a Google Sheet... explanation and links to alternative integrations                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Postgres Trigger Node:**  
   - Add a `Postgres Trigger` node.  
   - Set credentials to your Postgres connection.  
   - Configure to trigger on `UPDATE` events for the `users` table.  
   - Ensure the trigger is enabled for live operation.

2. **Create Manual Trigger Node (optional, for testing):**  
   - Add a `Manual Trigger` node.  
   - This allows manual execution of the workflow during testing.

3. **Create Code Node (for testing mock data):**  
   - Add a `Code` node connected to the Manual Trigger.  
   - Insert JavaScript code returning an array with one user object containing keys: `id`, `username`, `email`, `company_size`, `role`, `users`.  
   - Example code snippet:  
     ```javascript
     return [
       {
         id: 1,
         username: "max_mustermann",
         email: "max_mustermann@acme.com",
         company_size: "500-999",
         role: "Sales",
         users: 50,
       }
     ];
     ```

4. **Create Filter Node:**  
   - Add a `Filter` node connected to both the Postgres Trigger (for live data) and the Code node (for testing).  
   - Configure the filter condition: exclude records where `email` contains `@n8n.io`.  
   - Use an expression like `{{$json.email}}` does not contain `n8n.io`.

5. **Create Google Sheets Node:**  
   - Add a `Google Sheets` node connected to the Filter node.  
   - Configure credentials with OAuth2 for Google Sheets API access.  
   - Set operation to `appendOrUpdate`.  
   - Set Document ID to your Google Sheets document (replace with your own).  
   - Specify the Sheet by either name or GID (e.g., `gid=0`).  
   - Define the column mappings: map incoming data fields `id`, `email`, `username` to sheet columns with matching headers.  
   - Specify `id` as the matching column to update existing rows.  
   - Set cell format to `USER_ENTERED` to allow Google Sheets to format cells appropriately.

6. **Add Sticky Notes (optional):**  
   - Add sticky notes with instructions and explanations next to relevant nodes for clarity.

7. **Connect Nodes:**  
   - Connect Postgres Trigger → Filter → Google Sheets.  
   - Connect Manual Trigger → Code → Filter (for testing).  

8. **Test the Workflow:**  
   - Enable the Postgres Trigger node for live syncing.  
   - Use manual trigger and code node to test with mock data.  
   - Run nodes individually or execute the workflow to verify data flows correctly and updates Google Sheets as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                                                                                                                                                            |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This template shows how to sync data from one service to another, specifically from Postgres to Google Sheets. To test, duplicate the provided Google Sheets template, set up credentials, and execute the workflow. Customization includes swapping triggers, modifying filters, and changing the destination node.                                                                                                                                                                                                                                               | [Google Sheets template link](https://docs.google.com/spreadsheets/d/1gVfyernVtgYXD-oPboxOSJYQ-HEfAguEryZ7gTtK0V8/edit?usp=sharing)                                                                                                                          |
| For filtering and data transformation, n8n supports various nodes beyond Filter, such as Set, ItemList, and Code. These can be used to customize data manipulation as needed.                                                                                                                                                                                                                                                                                                                                                                                     | [n8n Filter Node Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.filter)  
[n8n Set Node Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set)  
[n8n ItemList Node Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.itemlists)  
[n8n Code Node Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code) |
| Google Sheets node can be replaced with other CRM or spreadsheet services such as Microsoft Excel, HubSpot, Pipedrive, Zendesk, depending on the user's integration needs. Ensure proper credentials and API permissions are configured accordingly.                                                                                                                                                                                                                                                                                                         | [Google Sheets Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleSheets)  
[Microsoft Excel Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftexcel)  
[HubSpot Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.hubspot)  
[Pipedrive Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.pipedrive)  
[Zendesk Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.zendesk) |

---

This documentation is designed to support both human operators and automated agents in understanding, replicating, and modifying the workflow efficiently.