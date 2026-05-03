Merge data for multiple executions

https://n8nworkflows.xyz/workflows/merge-data-for-multiple-executions-1160


# Merge data for multiple executions

### 1. Workflow Overview

This workflow demonstrates how to aggregate and merge data retrieved from multiple executions of an RSS feed reader node into a single consolidated data structure. It is designed for use cases where multiple RSS feed URLs need to be processed sequentially and their results combined for further unified processing or analysis.

The workflow is logically divided into the following blocks:

- **1.1 Input Preparation:** Generating a list of RSS feed URLs to process.
- **1.2 Batch Processing:** Iteratively processing each RSS feed URL one-by-one.
- **1.3 RSS Feed Reading:** Fetching the RSS feed data for each URL.
- **1.4 Completion Check:** Determining when all batches have been processed.
- **1.5 Data Merging:** Consolidating the results from all individual RSS feed reads into a single array.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Preparation

- **Overview:**  
  This block initializes the workflow by providing the list of RSS feed URLs to be processed. It outputs these URLs as JSON objects, one per item.

- **Nodes Involved:**  
  - Function

- **Node Details:**

  - **Function**  
    - *Type & Role:* Function node used to define the input data statically.  
    - *Configuration:* Returns an array of two JSON objects, each containing a `url` key with a feed URL (`https://medium.com/feed/n8n-io` and `https://dev.to/feed/n8n`).  
    - *Key Expressions:* Static JSON returned within `return` statement.  
    - *Input/Output:* No input, outputs multiple items (each item a feed URL).  
    - *Edge Cases:* None inherent, but if URLs are invalid, downstream nodes may fail.  
    - *Sub-workflow:* None.

#### 1.2 Batch Processing

- **Overview:**  
  This block processes the list of URLs one by one in batches of size 1, controlling the iteration flow through the workflow.

- **Nodes Involved:**  
  - SplitInBatches

- **Node Details:**

  - **SplitInBatches**  
    - *Type & Role:* Controls batch processing by emitting one item at a time.  
    - *Configuration:* `batchSize` set to 1 to process one URL per iteration.  
    - *Key Expressions:* None configured, default batch behavior.  
    - *Input/Output:* Receives multiple items (URLs) from Function node, outputs a single item per execution cycle.  
    - *Edge Cases:* If input is empty, no batches emitted.  
    - *Sub-workflow:* None.

#### 1.3 RSS Feed Reading

- **Overview:**  
  This block reads the RSS feed for the current URL passed from batch processing.

- **Nodes Involved:**  
  - RSS Feed Read

- **Node Details:**

  - **RSS Feed Read**  
    - *Type & Role:* Fetches and parses RSS feed data from the provided URL.  
    - *Configuration:* URL is dynamically set via expression `={{$json["url"]}}` to use the current batch item’s URL.  
    - *Key Expressions:* URL expression to dynamically inject feed URL.  
    - *Input/Output:* Receives single item with URL, outputs feed items as multiple items.  
    - *Edge Cases:* Possible errors include unreachable URLs, invalid feeds, or network timeouts.  
    - *Sub-workflow:* None.

#### 1.4 Completion Check

- **Overview:**  
  This block checks whether all URLs have been processed by evaluating the `noItemsLeft` context variable of the SplitInBatches node.

- **Nodes Involved:**  
  - IF

- **Node Details:**

  - **IF**  
    - *Type & Role:* Boolean condition node to branch workflow based on batch completion.  
    - *Configuration:* Checks if `{{$node["SplitInBatches"].context["noItemsLeft"]}}` is `true`.  
    - *Key Expressions:* Boolean condition comparing to `true`.  
    - *Input/Output:* Receives feed items from RSS Feed Read node.  
    - *Output:*  
      - True branch: proceeds to Merge Data node (all batches processed).  
      - False branch: loops back to SplitInBatches to process next batch.  
    - *Edge Cases:* If context variable is missing or undefined, condition evaluation might fail.  
    - *Sub-workflow:* None.

#### 1.5 Data Merging

- **Overview:**  
  This block collects and merges the data from all previous RSS feed read executions into a single consolidated array.

- **Nodes Involved:**  
  - Merge Data (Function node)

- **Node Details:**

  - **Merge Data**  
    - *Type & Role:* Function node that aggregates items from multiple executions of the RSS Feed Read node into a single array.  
    - *Configuration:*  
      - Uses a loop with a counter starting at 0.  
      - In each iteration, tries to fetch items from the `RSS Feed Read` node at execution index `counter` using `$items("RSS Feed Read", 0, counter)`.  
      - Pushes all fetched items’ JSON data into `allData` array.  
      - When no more executions exist (catch block), returns a single item with `allData` containing all merged data.  
    - *Key Expressions:* Uses `$items` with execution index to collect data across executions.  
    - *Input/Output:* Receives input from IF node’s true branch; outputs a single item containing merged data array.  
    - *Edge Cases:*  
      - If RSS Feed Read node has no previous executions or data, the loop stops gracefully.  
      - If an expression fails or unexpected data shape, may error or return incomplete data.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                 | Input Node(s)            | Output Node(s)       | Sticky Note                                                                                                     |
|---------------------|--------------------|--------------------------------|--------------------------|----------------------|----------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger     | Starts the workflow manually    |                          | Function             |                                                                                                                |
| Function            | Function           | Provides list of RSS feed URLs  | On clicking 'execute'     | SplitInBatches       |                                                                                                                |
| SplitInBatches      | Split In Batches   | Processes URLs one-by-one       | Function                  | RSS Feed Read        |                                                                                                                |
| RSS Feed Read       | RSS Feed Read      | Reads RSS feed for given URL    | SplitInBatches            | IF                   |                                                                                                                |
| IF                  | IF                 | Checks if all batches processed | RSS Feed Read             | Merge Data (true), SplitInBatches (false) |                                                                                                                |
| Merge Data          | Function           | Merges data from all executions | IF (true)                 |                      | The Merge Data Function node fetches the data from different executions of the RSS Feed Read node and merges them under a single object. **Note:** If you want to process the items that get merged, you will have to convert the single item into n8n understandable multiple items. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**
   - Name: `On clicking 'execute'`
   - Purpose: Manually starts the workflow.
   - No parameters.

2. **Create Function node:**
   - Name: `Function`
   - Connect input from `On clicking 'execute'`.
   - Set Function Code to:
     ```javascript
     return [
       { json: { url: 'https://medium.com/feed/n8n-io' } },
       { json: { url: 'https://dev.to/feed/n8n' } }
     ];
     ```
   - Purpose: Output an array of RSS feed URLs.

3. **Create SplitInBatches node:**
   - Name: `SplitInBatches`
   - Connect input from `Function`.
   - Set Batch Size to `1`.
   - Purpose: Process URLs one at a time.

4. **Create RSS Feed Read node:**
   - Name: `RSS Feed Read`
   - Connect input from `SplitInBatches`.
   - Set URL parameter to expression: `{{$json["url"]}}`.
   - Purpose: Read RSS feed data from the current batch URL.

5. **Create IF node:**
   - Name: `IF`
   - Connect input from `RSS Feed Read`.
   - Set condition:
     - Type: Boolean
     - Expression: `{{$node["SplitInBatches"].context["noItemsLeft"]}}`
     - Compare to: `true`
   - Purpose: Check if all batches have been processed.

6. **Connect IF node branches:**
   - True branch: Connect to `Merge Data` node.
   - False branch: Loop back to `SplitInBatches` node to process next item.

7. **Create Merge Data Function node:**
   - Name: `Merge Data`
   - Connect input from IF node’s true branch.
   - Set Function Code to:
     ```javascript
     const allData = [];
     let counter = 0;
     do {
       try {
         const items = $items("RSS Feed Read", 0, counter).map(item => item.json);
         allData.push(...items);
       } catch (error) {
         return [{ json: { allData } }];
       }
       counter++;
     } while (true);
     ```
   - Purpose: Aggregate all RSS feed items from previous executions into a single array.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The Merge Data Function node fetches the data from different executions of the RSS Feed Read node and merges them under a single object.                            | Workflow description and node comment.                                                         |
| **Note:** If you want to process the items that get merged, you will have to convert the single item into n8n understandable multiple items (e.g., using SplitInBatches). | Workflow description and node comment.                                                         |
| RSS Feed Read node URL parameter uses dynamic expressions to ingest URLs from input JSON.                                                                             | Important for dynamic feed input handling.                                                     |
| Loop in Merge Data relies on catching errors from non-existing execution indexes to terminate.                                                                        | Important to understand how the loop ends to avoid infinite loops.                             |

---

This document fully describes the logic, node configuration, and flow of the "Merge data for multiple executions" workflow, enabling users and automation agents to understand, reproduce, and modify it reliably.