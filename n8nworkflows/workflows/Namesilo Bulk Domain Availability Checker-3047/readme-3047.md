Namesilo Bulk Domain Availability Checker

https://n8nworkflows.xyz/workflows/namesilo-bulk-domain-availability-checker-3047


# Namesilo Bulk Domain Availability Checker

### 1. Workflow Overview

The **Namesilo Bulk Domain Availability Checker** workflow automates the process of checking domain name availability in bulk using the Namesilo API. It is designed to handle large lists of domains efficiently by batching requests (up to 200 domains per batch), respecting API rate limits with enforced wait times, and compiling results into a structured Excel spreadsheet. This workflow is ideal for domain investors, developers, marketers, and IT professionals who need to quickly assess domain availability without manual, repetitive checks.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Setup:** Collects user input including the domain list and Namesilo API key.
- **1.2 Domain Preparation and Batching:** Processes the domain list into batches of up to 200 domains.
- **1.3 Batch Loop and API Requests:** Iterates over each batch, sending requests to Namesilo API.
- **1.4 Response Parsing and Data Extraction:** Parses the API response to identify available and unavailable domains.
- **1.5 Rate Limiting Enforcement:** Implements a 5-minute wait between batches to comply with API limits.
- **1.6 Results Aggregation and Output:** Merges all batch results and exports them into an Excel file.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Setup

- **Overview:** This block initializes the workflow by receiving the domain list and Namesilo API key from the user.
- **Nodes Involved:**  
  - Set Data  
  - Start (Manual Trigger)  
  - Sticky Note (Instructional)

- **Node Details:**

  - **Start**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Set Data"  
    - Edge cases: None

  - **Set Data**  
    - Type: Set  
    - Role: Stores user inputs: the list of domains (one per line) and the Namesilo API key.  
    - Configuration:  
      - Domains: Multiline string with domains separated by new lines (e.g., domain1.com, domain2.com)  
      - Namesilo API Key: User’s API key string  
    - Expressions: None (static values expected to be replaced by user)  
    - Inputs: From "Start"  
    - Outputs: To "Convert & Split Domains"  
    - Edge cases: Empty or malformed domain list; missing or invalid API key may cause downstream failures.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides user instructions and API key acquisition link.  
    - Content Highlights:  
      - Link to obtain free Namesilo API key: https://www.namesilo.com/account/api-manager  
      - Notes on batch size (200 domains) and 5-minute wait due to rate limits.

---

#### 2.2 Domain Preparation and Batching

- **Overview:** Converts the multiline domain string into batches of up to 200 domains, formatted as comma-separated strings for API requests.
- **Nodes Involved:**  
  - Convert & Split Domains  
  - Loop Over Domains (SplitInBatches)

- **Node Details:**

  - **Convert & Split Domains**  
    - Type: Code  
    - Role: Parses the domain string, trims whitespace, filters out empty lines, and splits the domains into batches of 200.  
    - Configuration:  
      - JavaScript code splits input domains by newline, trims each, filters empty entries, then groups them into batches of 200 domains joined by commas.  
    - Expressions/Variables: Reads `$json.Domains` from "Set Data" node.  
    - Inputs: From "Set Data"  
    - Outputs: Array of objects with property `batchedDomains` containing comma-separated domain strings.  
    - Edge cases: Empty domain list results in zero batches; malformed input lines are filtered out.  
    - Version: Uses n8n Code node v2.

  - **Loop Over Domains**  
    - Type: SplitInBatches  
    - Role: Iterates over each batch of domains, processing one batch per loop cycle.  
    - Configuration: Default batch processing; no additional parameters.  
    - Inputs: From "Convert & Split Domains"  
    - Outputs: Two outputs:  
      - Main output 0: To "Merge Results" (for aggregated data)  
      - Main output 1: To "Namesilo Requests" (to process current batch)  
    - Edge cases: Handles empty batches gracefully; batch size controlled upstream.

---

#### 2.3 Batch Loop and API Requests

- **Overview:** For each batch, sends an HTTP request to Namesilo’s batch domain availability API endpoint.
- **Nodes Involved:**  
  - Namesilo Requests

- **Node Details:**

  - **Namesilo Requests**  
    - Type: HTTP Request  
    - Role: Calls Namesilo API to check availability of the current batch of domains.  
    - Configuration:  
      - HTTP Method: GET  
      - URL: Constructed dynamically using expressions:  
        `https://www.namesilo.com/apibatch/checkRegisterAvailability?version=1&type=json&key={{ $('Set Data').item.json['Namesilo API Key'] }}&domains={{ $json.batchedDomains }}`  
      - Retry on fail: Enabled with 5 seconds wait between retries.  
    - Inputs: From "Loop Over Domains" (batch data)  
    - Outputs: To "Parse Data"  
    - Edge cases:  
      - API key invalid or expired → authentication errors  
      - Network timeouts or API downtime → retry mechanism handles transient failures  
      - Exceeding rate limits → handled by enforced wait node downstream  
    - Version: HTTP Request node v4.2

---

#### 2.4 Response Parsing and Data Extraction

- **Overview:** Parses the JSON response from Namesilo API, extracting domain availability status into a structured format.
- **Nodes Involved:**  
  - Parse Data

- **Node Details:**

  - **Parse Data**  
    - Type: Code  
    - Role: Parses the nested JSON string inside the API response, extracts available and unavailable domains, and formats them into a uniform array of objects with `Domain` and `Availability` properties.  
    - Configuration:  
      - JavaScript code:  
        - Validates presence of `data` property in input JSON  
        - Parses JSON string in `data` field  
        - Extracts `available` and `unavailable` domains safely using optional chaining  
        - Outputs array with each domain and its availability status ("Available" or "Unavailable")  
    - Inputs: From "Namesilo Requests"  
    - Outputs: To "Wait" node  
    - Edge cases:  
      - Missing or malformed `data` property → throws error and stops workflow  
      - Unexpected JSON structure → error handling included  
      - Domains missing in either available or unavailable lists handled gracefully  
    - Version: Code node v2

---

#### 2.5 Rate Limiting Enforcement

- **Overview:** Enforces a 5-minute wait between processing each batch to comply with Namesilo API rate limits.
- **Nodes Involved:**  
  - Wait

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution for 5 minutes after processing each batch before continuing to the next batch.  
    - Configuration:  
      - Wait time: 5 minutes  
    - Inputs: From "Parse Data"  
    - Outputs: Back to "Loop Over Domains" to process next batch  
    - Edge cases: None significant; long wait times may affect workflow responsiveness  
    - Version: Wait node v1.1

---

#### 2.6 Results Aggregation and Output

- **Overview:** Aggregates domain availability results from all batches and converts the data into an Excel file for user download.
- **Nodes Involved:**  
  - Merge Results  
  - Convert to Excel

- **Node Details:**

  - **Merge Results**  
    - Type: Code  
    - Role: Normalizes and prepares the merged data from each batch into a consistent JSON array with `Domain` and `Availability` fields.  
    - Configuration:  
      - JavaScript code maps input items to output items with only the two relevant fields.  
    - Inputs: From "Loop Over Domains" (main output 0)  
    - Outputs: To "Convert to Excel"  
    - Edge cases: Handles empty inputs gracefully.  
    - Version: Code node v2

  - **Convert to Excel**  
    - Type: ConvertToFile  
    - Role: Converts the merged JSON data into an Excel spreadsheet file named `domain_results.xlsx`.  
    - Configuration:  
      - Operation: xlsx  
      - Binary Property Name: Uses expression `={{ $json.MergedDomains }}` (note: this expression appears to be a placeholder; actual implementation expects JSON input)  
      - File name: `domain_results.xlsx`  
    - Inputs: From "Merge Results"  
    - Outputs: Final output file (binary) for download or further use  
    - Edge cases: Large datasets may affect performance; formatting is basic with two columns only.  
    - Version: ConvertToFile node v1.1

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                          | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                  |
|----------------------|---------------------|----------------------------------------|------------------------|---------------------------|----------------------------------------------------------------------------------------------|
| Start                | Manual Trigger      | Initiates workflow manually             | None                   | Set Data                  |                                                                                              |
| Set Data             | Set                 | Collects domain list and API key        | Start                  | Convert & Split Domains    |                                                                                              |
| Sticky Note          | Sticky Note         | Provides instructions and API key link | None                   | None                      | How-To: Claim free Namesilo API key at https://www.namesilo.com/account/api-manager. Workflow sends up to 200 domains per loop. Output is Excel. Wait 5min per loop due to rate limits. |
| Convert & Split Domains | Code              | Splits domains into batches of 200      | Set Data               | Loop Over Domains          |                                                                                              |
| Loop Over Domains    | SplitInBatches      | Iterates over domain batches             | Convert & Split Domains | Merge Results, Namesilo Requests |                                                                                              |
| Namesilo Requests    | HTTP Request        | Sends batch domain availability request | Loop Over Domains      | Parse Data                |                                                                                              |
| Parse Data           | Code                | Parses API response and extracts status | Namesilo Requests       | Wait                      |                                                                                              |
| Wait                 | Wait                | Enforces 5-minute delay between batches | Parse Data              | Loop Over Domains          |                                                                                              |
| Merge Results        | Code                | Aggregates all batch results             | Loop Over Domains       | Convert to Excel          |                                                                                              |
| Convert to Excel     | ConvertToFile       | Converts aggregated data to Excel file  | Merge Results           | None                      |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Start`  
   - Purpose: To start the workflow manually.

2. **Create Set Node**  
   - Name: `Set Data`  
   - Purpose: Store user inputs.  
   - Parameters:  
     - Add string field `Domains` with multiline input (e.g., `domain1.com\ndomain2.com\ndomain3.com`).  
     - Add string field `Namesilo API Key` with your API key (replace `YOUR_API_KEY`).  
   - Connect `Start` → `Set Data`.

3. **Create Code Node**  
   - Name: `Convert & Split Domains`  
   - Purpose: Split domain list into batches of 200.  
   - Parameters:  
     - Use JavaScript code:  
       ```js
       const domains = $json.Domains.split("\n").map(d => d.trim()).filter(Boolean);
       const batchSize = 200;
       let batches = [];
       for (let i = 0; i < domains.length; i += batchSize) {
         batches.push(domains.slice(i, i + batchSize).join(","));
       }
       return batches.map(batch => ({ batchedDomains: batch }));
       ```  
   - Connect `Set Data` → `Convert & Split Domains`.

4. **Create SplitInBatches Node**  
   - Name: `Loop Over Domains`  
   - Purpose: Process each batch individually.  
   - Parameters: Default.  
   - Connect `Convert & Split Domains` → `Loop Over Domains`.

5. **Create HTTP Request Node**  
   - Name: `Namesilo Requests`  
   - Purpose: Send batch availability request to Namesilo API.  
   - Parameters:  
     - HTTP Method: GET  
     - URL:  
       ```
       https://www.namesilo.com/apibatch/checkRegisterAvailability?version=1&type=json&key={{ $('Set Data').item.json['Namesilo API Key'] }}&domains={{ $json.batchedDomains }}
       ```  
     - Enable "Retry On Fail" with 5 seconds wait between retries.  
   - Connect `Loop Over Domains` (output 1) → `Namesilo Requests`.

6. **Create Code Node**  
   - Name: `Parse Data`  
   - Purpose: Parse API response and extract domain availability.  
   - Parameters:  
     - Use JavaScript code:  
       ```js
       if (!$json || !$json.data) throw new Error("Invalid input data format");
       let parsedData;
       try {
         parsedData = JSON.parse($json.data);
       } catch (error) {
         throw new Error("Error parsing JSON data: " + error.message);
       }
       const availableDomains = parsedData.reply?.available ? Object.values(parsedData.reply.available) : [];
       const unavailableDomains = parsedData.reply?.unavailable ? Object.values(parsedData.reply.unavailable) : [];
       const output = [];
       availableDomains.forEach(domainObj => {
         if (domainObj && domainObj.domain) {
           output.push({ Domain: domainObj.domain, Availability: "Available" });
         }
       });
       unavailableDomains.forEach(domain => {
         if (typeof domain === "string") {
           output.push({ Domain: domain, Availability: "Unavailable" });
         } else if (typeof domain === "object" && domain.domain) {
           output.push({ Domain: domain.domain, Availability: "Unavailable" });
         }
       });
       return output;
       ```  
   - Connect `Namesilo Requests` → `Parse Data`.

7. **Create Wait Node**  
   - Name: `Wait`  
   - Purpose: Enforce 5-minute delay between batches.  
   - Parameters:  
     - Unit: Minutes  
     - Time: 5  
   - Connect `Parse Data` → `Wait`.

8. **Connect Wait Node back to Loop Over Domains**  
   - Connect `Wait` → `Loop Over Domains` (to continue next batch).

9. **Create Code Node**  
   - Name: `Merge Results`  
   - Purpose: Normalize and aggregate domain availability results.  
   - Parameters:  
     - Use JavaScript code:  
       ```js
       const newItems = items.map(item => ({
         json: {
           Domain: item.json.Domain,
           Availability: item.json.Availability
         }
       }));
       return newItems;
       ```  
   - Connect `Loop Over Domains` (output 0) → `Merge Results`.

10. **Create ConvertToFile Node**  
    - Name: `Convert to Excel`  
    - Purpose: Export aggregated data to Excel file.  
    - Parameters:  
      - Operation: xlsx  
      - File Name: `domain_results.xlsx`  
      - Binary Property Name: Leave default or set to `data` (ensure input data is JSON array from previous node)  
    - Connect `Merge Results` → `Convert to Excel`.

11. **Add Sticky Note (Optional)**  
    - Content:  
      ```
      ## How-To
      1. Claim your free Namesilo API key here: https://www.namesilo.com/account/api-manager

      2. Set your API key and domains in "Set Data" node.

      The workflow sends up to 200 domains per loop until all domains are processed. The output is in Excel format.

      Enjoy!

      Note: Each loop waits 5 minutes. This is required due to Namesilo rate limits.
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Claim your free Namesilo API key at https://www.namesilo.com/account/api-manager                                | Namesilo API key acquisition                            |
| The workflow respects Namesilo API rate limits by waiting 5 minutes between batch requests                       | API rate limiting requirement                           |
| Batch size is set to 200 domains per request to comply with Namesilo API limitations                            | Batch size limitation                                   |
| Output Excel file contains two columns: Domain and Availability (Available/Unavailable)                         | Output format description                               |
| Users should have basic knowledge of n8n workflows and a Namesilo account                                       | User prerequisites                                     |
| Workflow can be customized to filter domains, change batch size, or integrate with domain registration services | Customization suggestions                               |

---

This document fully describes the **Namesilo Bulk Domain Availability Checker** workflow, enabling users and developers to understand, reproduce, and customize the workflow effectively.