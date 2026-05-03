Automated Disqus Forum Reports with Gmail HTML Tables

https://n8nworkflows.xyz/workflows/automated-disqus-forum-reports-with-gmail-html-tables-4531


# Automated Disqus Forum Reports with Gmail HTML Tables

### 1. Workflow Overview

This workflow automates the generation and emailing of Disqus forum activity reports using Gmail, displaying the data in an HTML table format. It targets users or moderators who want to receive periodic summaries of forum performance and content engagement. The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow on demand.
- **1.2 Data Retrieval from Disqus:** Four nodes concurrently fetch forum metadata, threads, categories, and posts from Disqus.
- **1.3 Data Merging:** Separate merging of related Disqus data sets (Threads + Forum, Categories + Posts), followed by a final merge combining all data streams.
- **1.4 Data Processing:** Code nodes to combine and format the merged data into an HTML table.
- **1.5 Notification Delivery:** Sending the formatted report via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Initiates the report generation process manually for testing or on-demand runs.
- **Nodes Involved:**  
  - `When clicking ‘Test workflow’`
- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - **Type & Role:** Manual Trigger node, serves as the entry point.  
    - **Configuration:** No parameters; triggers workflow execution when clicked.  
    - **Inputs:** None (trigger).  
    - **Outputs:** Triggers four Disqus data retrieval nodes simultaneously.  
    - **Edge Cases:** No failure expected; user-dependent trigger.  
    - **Version:** Compatible with n8n v1.x+.  

#### 2.2 Data Retrieval from Disqus

- **Overview:** Four nodes independently fetch different data components from the Disqus platform to assemble a full report dataset.  
- **Nodes Involved:**  
  - `Get Forum`  
  - `Get Threads Forum`  
  - `Get Categories Forum`  
  - `Get Posts Forum`  
- **Node Details:**

  - **Get Forum**  
    - **Type & Role:** Disqus node; retrieves general forum information.  
    - **Configuration:** Default Disqus credentials; no filters or query parameters specified, likely fetches default or all forums associated with credentials.  
    - **Inputs:** Trigger from manual node.  
    - **Outputs:** Connected to `Merge Threads + Forum` node (input index 1).  
    - **Edge Cases:** API rate limits, authentication failures, empty forum list.  

  - **Get Threads Forum**  
    - **Type & Role:** Disqus node; fetches forum threads (discussion topics).  
    - **Configuration:** Default credentials; no additional parameters shown.  
    - **Inputs:** Trigger from manual node.  
    - **Outputs:** Connected to `Merge Threads + Forum` node (input index 0).  
    - **Edge Cases:** Timeout with large datasets, API failures, permission issues.  

  - **Get Categories Forum**  
    - **Type & Role:** Disqus node; retrieves forum categories.  
    - **Configuration:** Default Disqus credentials; no filters.  
    - **Inputs:** Trigger from manual node.  
    - **Outputs:** Connected to `Merge Categories + Posts` node (input index 0).  
    - **Edge Cases:** Empty categories, API quota exceeded.  

  - **Get Posts Forum**  
    - **Type & Role:** Disqus node; fetches posts (comments or replies) within the forum.  
    - **Configuration:** Standard Disqus credentials; no parameters.  
    - **Inputs:** Trigger from manual node.  
    - **Outputs:** Connected to `Merge Categories + Posts` node (input index 1).  
    - **Edge Cases:** Large post volumes causing timeout, auth failures.  

#### 2.3 Data Merging

- **Overview:** Combines the fetched data sets into a cohesive data structure for further processing.  
- **Nodes Involved:**  
  - `Merge Threads + Forum`  
  - `Merge Categories + Posts`  
  - `Merge All`  
- **Node Details:**

  - **Merge Threads + Forum**  
    - **Type & Role:** Merge node; joins Threads (input 0) and Forum info (input 1).  
    - **Configuration:** Default merge settings, presumably combining arrays or objects by index or key.  
    - **Inputs:** Threads Forum (0), Forum (1).  
    - **Outputs:** Connected to `Merge All` (input 0).  
    - **Edge Cases:** Mismatched array lengths, null data.  

  - **Merge Categories + Posts**  
    - **Type & Role:** Merge node; joins Categories (input 0) and Posts (input 1).  
    - **Configuration:** Default merge logic.  
    - **Inputs:** Categories Forum (0), Posts Forum (1).  
    - **Outputs:** Connected to `Merge All` (input 1).  
    - **Edge Cases:** Disparate data sets, missing keys.  

  - **Merge All**  
    - **Type & Role:** Merge node; combines outputs of previous merges for full data assembly.  
    - **Configuration:** Standard merge combining two merged data sets.  
    - **Inputs:** Merge Threads + Forum (0), Merge Categories + Posts (1).  
    - **Outputs:** Connected to `Combine Data`.  
    - **Edge Cases:** Large payloads causing memory or timeout issues.  

#### 2.4 Data Processing

- **Overview:** Processes and formats merged data into an HTML table suitable for embedding in an email body.  
- **Nodes Involved:**  
  - `Combine Data` (Code node)  
  - `Format Table` (Code node)  
- **Node Details:**

  - **Combine Data**  
    - **Type & Role:** Code node (JavaScript); combines merged data into a structured format.  
    - **Configuration:** Custom JavaScript code likely iterates over merged data to unify and prepare for formatting.  
    - **Inputs:** Data from `Merge All`.  
    - **Outputs:** Passes combined data to `Format Table`.  
    - **Edge Cases:** Code errors due to unexpected data structure, undefined/null values.  
    - **Version:** Uses code node v2, compatible with n8n 0.150+.  

  - **Format Table**  
    - **Type & Role:** Code node (JavaScript); generates an HTML table from combined data.  
    - **Configuration:** Custom code outputs HTML string with table tags and inline styles for Gmail compatibility.  
    - **Inputs:** Combined data from `Combine Data`.  
    - **Outputs:** Sends formatted HTML to Gmail node.  
    - **Edge Cases:** HTML formatting errors, character encoding issues.  
    - **Version:** Code node v2.  

#### 2.5 Notification Delivery

- **Overview:** Sends the generated HTML report via Gmail to predefined recipients.  
- **Nodes Involved:**  
  - `Gmail Report Notifications`  
- **Node Details:**

  - **Gmail Report Notifications**  
    - **Type & Role:** Gmail node; sends an email with HTML content.  
    - **Configuration:**  
      - Uses OAuth2 credentials configured for Gmail.  
      - Email content type set to HTML.  
      - Subject and recipient fields configured in parameters (not visible in JSON, assumed configured).  
    - **Inputs:** Receives formatted HTML body from `Format Table`.  
    - **Outputs:** None (end node).  
    - **Edge Cases:** Authentication errors, quota limits, invalid recipient address, email content rejected by spam filters.  
    - **Version:** Gmail node v2.1, requires OAuth2 Gmail credentials.  

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                 | Input Node(s)                      | Output Node(s)               | Sticky Note                                   |
|---------------------------|--------------------|--------------------------------|----------------------------------|-----------------------------|-----------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Initiates workflow              | None                             | Get Forum, Get Posts Forum, Get Threads Forum, Get Categories Forum |                                               |
| Get Forum                 | Disqus             | Fetch forum info                | When clicking ‘Test workflow’    | Merge Threads + Forum (input 1) |                                               |
| Get Threads Forum         | Disqus             | Fetch forum threads             | When clicking ‘Test workflow’    | Merge Threads + Forum (input 0) |                                               |
| Get Categories Forum      | Disqus             | Fetch forum categories          | When clicking ‘Test workflow’    | Merge Categories + Posts (input 0) |                                               |
| Get Posts Forum           | Disqus             | Fetch forum posts               | When clicking ‘Test workflow’    | Merge Categories + Posts (input 1) |                                               |
| Merge Threads + Forum     | Merge              | Combine threads and forum info | Get Threads Forum, Get Forum     | Merge All (input 0)           |                                               |
| Merge Categories + Posts  | Merge              | Combine categories and posts   | Get Categories Forum, Get Posts Forum | Merge All (input 1)           |                                               |
| Merge All                 | Merge              | Combine all merged data         | Merge Threads + Forum, Merge Categories + Posts | Combine Data                 |                                               |
| Combine Data              | Code               | Combine merged data structures | Merge All                       | Format Table                 |                                               |
| Format Table              | Code               | Format combined data as HTML table | Combine Data                    | Gmail Report Notifications   |                                               |
| Gmail Report Notifications | Gmail              | Send HTML report via email      | Format Table                    | None                        | Requires Gmail OAuth2 credentials for sending emails |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: `When clicking ‘Test workflow’`  
   - No special parameters needed.

2. **Add Disqus Nodes for Data Retrieval:**  
   - Four Disqus nodes with default credentials (ensure Disqus API credentials are set up):  
     - `Get Forum` (no parameters, fetches forum info)  
     - `Get Threads Forum` (fetches threads)  
     - `Get Categories Forum` (fetches categories)  
     - `Get Posts Forum` (fetches posts)  
   - Connect all four nodes' inputs to the Manual Trigger node output.

3. **Add Merge Nodes for Partial Data Combining:**  
   - `Merge Threads + Forum` node:  
     - Inputs: connect `Get Threads Forum` to input 0, `Get Forum` to input 1.  
     - Use default merge settings.  
   - `Merge Categories + Posts` node:  
     - Inputs: connect `Get Categories Forum` to input 0, `Get Posts Forum` to input 1.  
     - Use default merge settings.

4. **Add Final Merge Node:**  
   - `Merge All` node:  
     - Inputs: connect `Merge Threads + Forum` to input 0, `Merge Categories + Posts` to input 1.  
     - Default merge settings.

5. **Add Code Node to Combine Data:**  
   - `Combine Data` node (Code v2):  
     - Use JavaScript to access merged input data, combine into a unified structure (e.g., an object or array that consolidates threads, posts, categories, and forum information).  
     - Connect input from `Merge All`.

6. **Add Code Node to Format Table:**  
   - `Format Table` node (Code v2):  
     - Write JavaScript code to generate an HTML table string from the combined data. Include inline styles for compatibility with Gmail’s HTML rendering.  
     - Connect input from `Combine Data`.

7. **Add Gmail Node to Send Email:**  
   - `Gmail Report Notifications` node (Gmail v2.1):  
     - Configure OAuth2 credentials for Gmail account.  
     - Set `To`, `Subject`, and `HTML Body` fields; bind the HTML body to the output of `Format Table`.  
     - Connect input from `Format Table`.  

8. **Final Checks:**  
   - Verify all credentials (Disqus and Gmail) are configured and authorized.  
   - Test the workflow by clicking the Manual Trigger node.  
   - Monitor for API limits or errors in nodes and adjust parameters or add error handling as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                      |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Gmail node requires OAuth2 credentials setup in n8n credentials manager for sending emails.     | Gmail OAuth2 documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/ |
| Disqus node configuration requires API keys and proper forum access credentials.                | Disqus API docs: https://disqus.com/api/docs/       |
| HTML tables in Gmail should avoid complex CSS; inline styles are recommended for compatibility. | Gmail HTML email guidelines: https://developers.google.com/gmail/design/reference/ |
| This workflow is designed for manual run but can be scheduled via Cron Trigger for automation.  | n8n Cron Trigger node docs: https://docs.n8n.io/nodes/n8n-nodes-base.cron/ |

---

**Disclaimer:** The provided content and data are generated exclusively from an n8n workflow automation. All data handled is public and legal, complying with content policies.