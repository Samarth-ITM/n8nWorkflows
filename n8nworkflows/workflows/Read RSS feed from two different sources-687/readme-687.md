Read RSS feed from two different sources

https://n8nworkflows.xyz/workflows/read-rss-feed-from-two-different-sources-687


# Read RSS feed from two different sources

### 1. Workflow Overview

This workflow is designed to read RSS feeds from two different sources sequentially and process their items. It is triggered manually and includes logic to iterate over multiple feed URLs, fetching and handling each feed one by one. The workflow’s primary use case is to aggregate content from distinct RSS sources, enabling further processing or integration downstream.

Logical blocks:

- **1.1 Input Reception:** Manual trigger that initiates the workflow.
- **1.2 Feed Source Preparation:** Generates the list of RSS feed URLs to be processed.
- **1.3 Iterative Processing:** Loops over each RSS feed URL, fetching feed data for each in turn.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow execution manually by user action.

- **Nodes Involved:**  
  - When clicking "Execute Workflow"

- **Node Details:**  

  - **Node Name:** When clicking "Execute Workflow"  
  - **Type:** Manual Trigger  
  - **Configuration:** No parameters set; triggers workflow on manual execution.  
  - **Expressions/Variables:** None.  
  - **Input / Output:** No input; output connects to the "Code" node.  
  - **Version Requirements:** Compatible with n8n core versions supporting manual trigger.  
  - **Potential Failure Modes:** None typical; manual trigger is reliable unless workflow is inactive.  
  - **Sub-workflow:** None.

#### 1.2 Feed Source Preparation

- **Overview:**  
  This block defines the RSS feed URLs to be processed by returning a static array of feed URLs, one for Medium and one for dev.to.

- **Nodes Involved:**  
  - Code

- **Node Details:**  

  - **Node Name:** Code  
  - **Type:** Code (JavaScript)  
  - **Configuration:** Returns an array of JSON objects, each with a `url` property containing an RSS feed URL. Specifically:  
    ```js
    return [
      { json: { url: 'https://medium.com/feed/n8n-io' } },
      { json: { url: 'https://dev.to/feed/n8n' } }
    ];
    ```  
  - **Expressions/Variables:** Hardcoded URLs inside the JS code.  
  - **Input / Output:** Input from manual trigger; output is an array of RSS feed URLs, which feeds into the batch splitter.  
  - **Version Requirements:** Requires n8n version supporting the Code node with JavaScript support.  
  - **Potential Failure Modes:** If feed URLs change or become invalid, downstream nodes may fail to fetch data.  
  - **Sub-workflow:** None.

#### 1.3 Iterative Processing

- **Overview:**  
  This block processes each RSS feed URL separately by splitting the list into batches of one and reading each RSS feed sequentially.

- **Nodes Involved:**  
  - Loop Over Items  
  - RSS Feed Read

- **Node Details:**  

  - **Node Name:** Loop Over Items  
  - **Type:** SplitInBatches  
  - **Configuration:** Default options (batch size defaults to 1) to process one feed URL per iteration. No additional parameters set.  
  - **Expressions/Variables:** Receives feed list from "Code" node; outputs one feed URL per batch.  
  - **Input / Output:** Input from "Code" node; output has two connections:  
    - First output (empty) - unused here.  
    - Second output connects to "RSS Feed Read" node to process the current batch feed URL.  
  - **Version Requirements:** Requires SplitInBatches node version 3 or higher for the dual output behavior.  
  - **Potential Failure Modes:** If batching parameters or input data are malformed, may cause processing issues.  
  - **Sub-workflow:** None.

  - **Node Name:** RSS Feed Read  
  - **Type:** RSS Feed Read  
  - **Configuration:**  
    - URL is dynamically set using the expression `{{$json.url}}` to fetch the current feed URL from incoming data.  
    - No additional options provided.  
  - **Expressions/Variables:** The feed URL is dynamically injected from the batch item JSON.  
  - **Input / Output:** Input from "Loop Over Items" node; output not connected further in this workflow (could be extended to process or store feed items).  
  - **Version Requirements:** Compatible with n8n versions supporting RSS Feed Read node version 1.  
  - **Potential Failure Modes:**  
    - Network errors or unreachable feed URL.  
    - Invalid or malformed RSS feed data.  
    - Timeout or rate limiting from feed hosts.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role             | Input Node(s)             | Output Node(s)          | Sticky Note                         |
|---------------------------|-----------------------|----------------------------|---------------------------|-------------------------|-----------------------------------|
| When clicking "Execute Workflow" | Manual Trigger       | Entry point for workflow    | —                         | Code                    |                                   |
| Code                      | Code (JavaScript)     | Produces list of RSS feed URLs | When clicking "Execute Workflow" | Loop Over Items          |                                   |
| Loop Over Items           | SplitInBatches        | Iterates over feed URLs     | Code                      | RSS Feed Read           |                                   |
| RSS Feed Read             | RSS Feed Read         | Reads RSS feed from URL     | Loop Over Items           | —                       |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: `When clicking "Execute Workflow"`  
   - Leave default settings (no parameters).  
   - This node will serve as the workflow entry point.

2. **Add a Code node:**  
   - Name: `Code`  
   - Set mode to JavaScript.  
   - Paste the following code to return the two RSS feed URLs:  
     ```js
     return [
       { json: { url: 'https://medium.com/feed/n8n-io' } },
       { json: { url: 'https://dev.to/feed/n8n' } }
     ];
     ```  
   - Connect output of the Manual Trigger node to this Code node.

3. **Add a SplitInBatches node:**  
   - Name: `Loop Over Items`  
   - No special options need to be configured; default batch size is 1.  
   - Connect output of the Code node to this node.

4. **Add an RSS Feed Read node:**  
   - Name: `RSS Feed Read`  
   - In the URL parameter, click the gears icon and select "Add Expression".  
   - Use the expression editor to set the URL to `{{$json.url}}` (this dynamically reads the URL from the batch item).  
   - Connect the second output (index 1) of the SplitInBatches node (`Loop Over Items`) to this RSS Feed Read node.  
   - No credentials are required for RSS feeds unless the feed is protected.

5. **Finalizing:**  
   - Ensure the workflow is active and save it.  
   - Test the workflow by clicking "Execute Workflow" on the manual trigger.  
   - The workflow will fetch and process each RSS feed sequentially.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------|
| RSS Feed Read node uses dynamic URL expressions to allow flexible feed processing.                   | n8n RSS Feed Read documentation                  |
| The workflow processes feeds sequentially via SplitInBatches, enabling scalable extension.           | n8n SplitInBatches node documentation             |
| For production use, consider adding error handling for network failures and invalid RSS feeds.       | Best practices in n8n error handling              |
| The feed URLs used are examples; update them to your desired feeds as needed.                        | Medium: https://medium.com/feed/n8n-io            |
| Dev.to feed URL: https://dev.to/feed/n8n                                                            |                                                  |

---

This documentation provides a comprehensive and structured view of the "Read RSS feed from two different sources" workflow to facilitate understanding, modification, and reproduction.