Bright Data-Powered Competitive Price Lookup and Report Generator

https://n8nworkflows.xyz/workflows/bright-data-powered-competitive-price-lookup-and-report-generator-4348


# Bright Data-Powered Competitive Price Lookup and Report Generator

### 1. Workflow Overview

This workflow is designed to automate competitive price lookup and generate comprehensive reports using Bright Data’s marketplace dataset, Google Sheets as an input source, and AI-powered analysis via Google Gemini and LangChain. It targets users who want to monitor product prices from various sellers and receive ranked, human-readable reports of the best prices for specified items.

The workflow logically divides into these blocks:

- **1.1 Input Reception and Preparation**: Trigger and fetch product list from Google Sheets, splitting these into manageable batches.
- **1.2 Bright Data Snapshot Request and Polling**: For each product, request price data snapshots from Bright Data, then poll for snapshot completion and handle errors.
- **1.3 Snapshot Content Validation and Data Extraction**: Validate that snapshot data is ready and extract relevant pricing information.
- **1.4 AI-Powered Price Comparison and Report Generation**: Use Google Gemini and LangChain to analyze extracted data, generate a ranked list of lowest prices, and convert the result to HTML.
- **1.5 Report Delivery via Email**: Send the formatted report by email.
- **1.6 Error Handling and Notifications**: Detect snapshot failures and generate error messages for retries or alerts.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Preparation

**Overview:**  
Starts the workflow manually and retrieves a list of items from a Google Sheets document. It splits the item list into batches for sequential processing.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Google Sheets  
- Loop Over Items (SplitInBatches)  
- Sticky Note (Instructional Note)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start workflow manually for testing or execution  
  - Configuration: No parameters  
  - Input: None  
  - Output: Triggers next node (Google Sheets)  
  - Edge cases: None typical; manual trigger

- **Google Sheets**  
  - Type: Google Sheets node  
  - Role: Reads the input product list from a specified Google Sheet and tab  
  - Configuration: Reads from sheet `gid=0` in document with ID `1Jf8qNEg-KTh__aZ8L5YS5iMmBZLEEKCYipIszWARCmw`  
  - Credentials: Google Sheets OAuth2  
  - Input: Manual trigger output  
  - Output: List of product items (JSON)  
  - Edge cases: Google API auth errors, empty or malformed sheets, network issues

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Splits the input list into individual items or smaller batches for iterative processing  
  - Configuration: Default batch size (1 item per batch implied)  
  - Input: Google Sheets output  
  - Output: Single product item per batch sent downstream  
  - Edge cases: Empty input data, batch size misconfiguration

- **Sticky Note**  
  - Type: Sticky Note (Instructional)  
  - Contains a warning about case sensitivity of item names in the dataset to prevent zero-result searches  
  - Covers nodes related to input preparation  

---

#### 2.2 Bright Data Snapshot Request and Polling

**Overview:**  
For each item, requests a filtered dataset snapshot from Bright Data marketplace, waits 30 seconds, then polls snapshot status until ready or failed.

**Nodes Involved:**  
- Snapshot Request  
- Wait 30s - Polling Bright Data  
- Snapshot Progress  
- If - Checking status of Snapshot  
- If - Checking status for errors  
- Error message (replace with webhook/other notifier if needed)

**Node Details:**

- **Snapshot Request**  
  - Type: BrightData Marketplace Dataset filter  
  - Role: Requests a snapshot filtered for the current product title where `item_price` is not null  
  - Configuration: Uses dataset ID for "Google Shopping" (`gd_ltppk50q18kdw67omz`), filter includes case-sensitive `title includes {{$json.title}}` and price not null  
  - Credentials: Bright Data API  
  - Input: Single product item from Loop Over Items  
  - Output: Snapshot ID for polling  
  - Edge cases: Filter syntax errors, API auth failure, no matching items, case sensitivity issues

- **Wait 30s - Polling Bright Data**  
  - Type: Wait  
  - Role: Pauses workflow 30 seconds between snapshot status checks to avoid excessive API calls  
  - Configuration: Wait 30 seconds, repeats until snapshot ready or failed  
  - Input: Snapshot Request output  
  - Output: Triggers Snapshot Progress node  
  - Edge cases: Timeouts, premature exit if workflow interrupted

- **Snapshot Progress**  
  - Type: BrightData getSnapshotMetadata  
  - Role: Retrieves current snapshot metadata including status and warnings  
  - Configuration: Uses snapshot_id from previous node  
  - Credentials: Bright Data API  
  - Input: Wait node output  
  - Output: Snapshot metadata JSON  
  - Edge cases: API errors, invalid snapshot ID

- **If - Checking status of Snapshot**  
  - Type: If (conditional)  
  - Role: Checks if snapshot status is not "running" to decide next step  
  - Input: Snapshot Progress output  
  - Output: If failed, goes to error handling; if ready, proceeds; else waits again  
  - Edge cases: Unexpected status values, missing status field

- **If - Checking status for errors**  
  - Type: If (conditional)  
  - Role: Detects if snapshot status is "failed" and triggers error message node or retries snapshot content fetch  
  - Input: Snapshot metadata  
  - Output: Routes to error notification or retry loop  
  - Edge cases: Non-failed error statuses

- **Error message (replace with webhook/other notifier if needed)**  
  - Type: Set  
  - Role: Constructs error message text including snapshot warning and product title for notification  
  - Configuration: Uses expressions to insert product title and snapshot warning message  
  - Input: From failed snapshot check  
  - Output: Loops back to Loop Over Items for next item processing  
  - Edge cases: Missing warning message, variable substitution fails

---

#### 2.3 Snapshot Content Validation and Data Extraction

**Overview:**  
Once the snapshot is ready, fetches its content, verifies it is an array, filters relevant entries, and extracts key fields for analysis.

**Nodes Involved:**  
- Snapshot Content  
- Code - Check If Snapshot is built  
- Check if snapshot ready  
- Code - Extract Necessary Data

**Node Details:**

- **Snapshot Content**  
  - Type: BrightData getSnapshotContent  
  - Role: Retrieves the actual data content from Bright Data snapshot  
  - Configuration: Uses snapshot ID from Snapshot Request node  
  - Credentials: Bright Data API  
  - Input: Output of "Check if snapshot ready" true branch or retry branch  
  - Output: Snapshot data JSON array or object  
  - Edge cases: API failures, empty data, invalid snapshot ID

- **Code - Check If Snapshot is built**  
  - Type: Code (JavaScript)  
  - Role: Checks if the snapshot data is an array and passes this info downstream  
  - Configuration: Returns JSON with `isArray` flag and raw data in `original`  
  - Input: Snapshot Content output  
  - Output: Passes to If node to decide next step  
  - Edge cases: Unexpected data formats, runtime errors

- **Check if snapshot ready**  
  - Type: If  
  - Role: Checks if snapshot content is an array; if false, triggers data extraction, else loops or ends  
  - Input: Output of Code - Check If Snapshot is built  
  - Output: True branch to Snapshot Content (retry?), False branch to data extraction  
  - Edge cases: Logic inversion or false positives

- **Code - Extract Necessary Data**  
  - Type: Code (JavaScript)  
  - Role: Filters snapshot items to those with valid `item_price`, extracts fields: price, seller_name, title, url  
  - Configuration: Maps relevant fields into a simplified list for AI processing  
  - Input: Snapshot Content data (array)  
  - Output: JSON with an array `items` containing extracted data  
  - Edge cases: Missing fields, null values, malformed data

---

#### 2.4 AI-Powered Price Comparison and Report Generation

**Overview:**  
Analyzes the extracted price data using Google Gemini and LangChain to find the top 20 lowest-priced sellers, formats the result in Markdown, then converts to styled HTML.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Compare Prices and Generate Report (LangChain chainLlm)  
- Markdown (Markdown to HTML)  
- Code - Build HTML

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini chat model node  
  - Role: Provides AI language model interaction backend  
  - Configuration: Uses model `models/gemini-2.0-flash`  
  - Credentials: Google PaLM API  
  - Input: Not direct from workflow but configured to feed Langchain node  
  - Output: AI-generated response text  
  - Edge cases: API limits, auth errors, latency

- **Compare Prices and Generate Report**  
  - Type: LangChain chainLlm  
  - Role: Sends the extracted price items to AI for analysis and summary generation  
  - Configuration: Prompt requests top 20 lowest prices ranked with seller name, price, and URL; outputs Markdown text; avoids filler phrases like “Okay”  
  - Input: Extracted items JSON array  
  - Output: Markdown text with ranked seller list  
  - Edge cases: Insufficient data, ambiguous input, AI response errors

- **Markdown**  
  - Type: Markdown node  
  - Role: Converts AI-generated Markdown text into sanitized HTML  
  - Configuration: Mode set to convert Markdown to HTML, output key `html`  
  - Input: AI-generated Markdown text  
  - Output: HTML string  
  - Edge cases: Malformed Markdown, conversion errors

- **Code - Build HTML**  
  - Type: Code (JavaScript)  
  - Role: Wraps raw HTML in full HTML document structure with inline CSS styling for readability  
  - Configuration: Adds font family, colors, link styles, and margins for presentation  
  - Input: HTML from Markdown node  
  - Output: Complete HTML report for email  
  - Edge cases: Injection risks, malformed HTML

---

#### 2.5 Report Delivery via Email

**Overview:**  
Sends the final HTML report to a specified email address with a subject referencing the current item.

**Nodes Involved:**  
- Email Report

**Node Details:**

- **Email Report**  
  - Type: Email Send node  
  - Role: Delivers the generated price comparison report via SMTP email  
  - Configuration:  
    - Subject includes item title dynamically  
    - Recipient: jotunheim166@gmail.com  
    - Sender: n8n-mail@example.com  
    - HTML body from Code - Build HTML node  
  - Credentials: SMTP account  
  - Input: HTML report and batch item data  
  - Output: None (end node)  
  - Edge cases: SMTP auth failures, delivery errors, invalid emails

---

### 3. Summary Table

| Node Name                                   | Node Type                           | Functional Role                                | Input Node(s)                      | Output Node(s)                       | Sticky Note                                                                                                   |
|---------------------------------------------|-----------------------------------|------------------------------------------------|-----------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                | Manual Trigger                    | Entry point to start workflow manually          | None                              | Google Sheets                      |                                                                                                               |
| Google Sheets                                | Google Sheets                     | Reads product list from Google Sheets           | When clicking ‘Test workflow’     | Loop Over Items                   | ⚠️ Note: Item searching in the dataset is case-sensitive. Use correct casing for product names to avoid misses. |
| Loop Over Items                              | SplitInBatches                   | Splits product list into single items for processing | Google Sheets                    | Snapshot Request (2nd output)      |                                                                                                               |
| Sticky Note                                  | Sticky Note                      | Instructional note on input case sensitivity    | None                              | None                             | ⚠️ Note: Item searching in the dataset is case-sensitive. Use "Iphone" instead of "iphone", etc.               |
| Snapshot Request                            | BrightData Filter Dataset         | Requests filtered snapshot for current product  | Loop Over Items                  | Wait 30s - Polling Bright Data    |                                                                                                               |
| Wait 30s - Polling Bright Data              | Wait                             | Waits 30 seconds before polling snapshot status | Snapshot Request                | Snapshot Progress                |                                                                                                               |
| Snapshot Progress                           | BrightData Get Snapshot Metadata  | Retrieves snapshot status and warnings          | Wait 30s - Polling Bright Data   | If - Checking status of Snapshot  |                                                                                                               |
| If - Checking status of Snapshot             | If                               | Checks if snapshot status is no longer running  | Snapshot Progress                | If - Checking status for errors / Wait 30s |                                                                                                               |
| If - Checking status for errors               | If                               | Detects if snapshot status is failed             | If - Checking status of Snapshot | Error message / Snapshot Content   |                                                                                                               |
| Error message (replace with webhook/other notifier if needed) | Set                              | Sets error message for failed snapshot           | If - Checking status for errors  | Loop Over Items                  |                                                                                                               |
| Snapshot Content                            | BrightData Get Snapshot Content   | Retrieves snapshot content data                   | Check if snapshot ready (true)   | Code - Check If Snapshot is built |                                                                                                               |
| Code - Check If Snapshot is built            | Code                             | Checks if snapshot content is an array            | Snapshot Content                | Check if snapshot ready           |                                                                                                               |
| Check if snapshot ready                      | If                               | Validates snapshot content form                   | Code - Check If Snapshot is built | Snapshot Content / Code - Extract Necessary Data |                                                                                                               |
| Code - Extract Necessary Data                 | Code                             | Filters and extracts relevant product data       | Check if snapshot ready (false) | Compare Prices and Generate Report |                                                                                                               |
| Google Gemini Chat Model                      | LangChain Google Gemini Model     | AI language model backend                         | (Internal to LangChain node)     | Compare Prices and Generate Report |                                                                                                               |
| Compare Prices and Generate Report            | LangChain Chain LLM              | AI analysis to identify top 20 lowest prices     | Code - Extract Necessary Data    | Markdown                         |                                                                                                               |
| Markdown                                    | Markdown                          | Converts AI Markdown output to HTML               | Compare Prices and Generate Report | Code - Build HTML               |                                                                                                               |
| Code - Build HTML                            | Code                             | Wraps HTML with styling for email report          | Markdown                        | Email Report                    |                                                                                                               |
| Email Report                                | Email Send                       | Sends the final report via email                   | Code - Build HTML               | Loop Over Items (for next item)  |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "When clicking ‘Test workflow’" to start the workflow manually.

2. **Add a Google Sheets node** configured as:  
   - Operation: Read rows from a Google Sheet  
   - Document ID: `1Jf8qNEg-KTh__aZ8L5YS5iMmBZLEEKCYipIszWARCmw`  
   - Sheet Name: `gid=0` (default sheet)  
   - Credentials: Google Sheets OAuth2 account  
   - Connect the Manual Trigger output to this node.

3. **Add a SplitInBatches node** named "Loop Over Items":  
   - Default batch size (1) to process items one by one  
   - Connect Google Sheets output to this node.

4. **Add a Bright Data node named "Snapshot Request"**:  
   - Resource: marketplaceDataset  
   - Operation: filterDataset  
   - dataset_id: select "Google Shopping" (`gd_ltppk50q18kdw67omz`)  
   - Filter type: filters_group  
   - Filters group JSON:  
     ```json
     {
       "operator": "and",
       "filters": [
         { "name": "title", "operator": "includes", "value": "{{$json.title}}" },
         { "name": "item_price", "operator": "is_not_null" }
       ]
     }
     ```  
   - Credentials: Bright Data API  
   - Connect "Loop Over Items" second output (single item) to this node.

5. **Add a Wait node "Wait 30s - Polling Bright Data"**:  
   - Amount: 30 seconds  
   - Connect Snapshot Request output to this node.

6. **Add Bright Data node "Snapshot Progress"** to get snapshot metadata:  
   - Operation: getSnapshotMetadata  
   - snapshot_id: `{{$json.snapshot_id}}` (from previous node)  
   - Credentials: Bright Data API  
   - Connect Wait node output to this node.

7. **Add an If node "If - Checking status of Snapshot"**:  
   - Condition: `$json.status` not equals "running"  
   - Connect Snapshot Progress output to this node.

8. **Add an If node "If - Checking status for errors"** connected from true branch of above If:  
   - Condition: `$json.status` equals "failed"  
   - True branch connects to "Error message" node  
   - False branch connects to "Snapshot Content" node (next step)

9. **Add a Set node "Error message (replace with webhook/other notifier if needed)"**:  
   - Assign string variable `message` to:  
     `Bright Data snapshot error for item "{{$json.title}}": \n{{$json.warning}}` (using expression referencing "Loop Over Items" title and Snapshot Progress warning)  
   - Connect true branch of "If - Checking status for errors" to this node.  
   - Connect this node back to "Loop Over Items" to continue with next item.

10. **Add Bright Data node "Snapshot Content"**:  
    - Operation: getSnapshotContent  
    - snapshot_id: `={{ $('Snapshot Request').item.json.snapshot_id }}`  
    - Credentials: Bright Data API  
    - Connect false branch of "If - Checking status for errors" to this node.

11. **Add a Code node "Code - Check If Snapshot is built"** with JavaScript:  
    ```js
    const value = $json.items;
    const isArray = Array.isArray(value);
    return [{ json: { isArray, original: value } }];
    ```  
    - Connect Snapshot Content output to this node.

12. **Add an If node "Check if snapshot ready"**:  
    - Condition: `$json.isArray` equals false (boolean false)  
    - True branch connects back to Snapshot Content (retry)  
    - False branch continues to data extraction

13. **Add a Code node "Code - Extract Necessary Data"** with JavaScript:  
    ```js
    const input = items[0].json.original;
    const posts = Array.isArray(input) ? input : [];
    const filteredPosts = posts.filter(post => post && post.item_price != null);
    const extracted = filteredPosts.map(post => ({
      price: post.item_price,
      seller_name: post.seller_name || null,
      title: post.title || null,
      url: post.url || null,
    }));
    return [{ json: { items: extracted } }];
    ```  
    - Connect false branch of "Check if snapshot ready" to this node.

14. **Add a LangChain Google Gemini Chat Model node "Google Gemini Chat Model"**:  
    - Model name: `models/gemini-2.0-flash`  
    - Credentials: Google PaLM API  
    - This node is referenced internally by next LangChain node.

15. **Add a LangChain Chain LLM node "Compare Prices and Generate Report"**:  
    - Text input: `={{ $json }}`  
    - Messages:  
      - `"Analyze these listings from Google Shopping and identify the top 20 sources (stores or websites) offering the lowest prices. Provide a ranked list including the seller name, product price, and link to the listing if available. Ensure all product names match or are highly relevant to {{ $('Loop Over Items').item }}. Don't start your answer with 'Okay'. Return all the text as Markdown."`  
    - Prompt type: define  
    - Connect "Code - Extract Necessary Data" output to this node  
    - Assign Google Gemini Chat Model node as its language model credential

16. **Add a Markdown node "Markdown"**:  
    - Mode: markdownToHtml  
    - Markdown source: `={{ $json.text }}` from previous node  
    - Destination key: `html`  
    - Connect "Compare Prices and Generate Report" output to this node.

17. **Add a Code node "Code - Build HTML"** with JavaScript:  
    ```js
    const rawHtml = $json.html;
    return [{
      json: {
        html: `
          <html>
            <head>
              <style>
                body { font-family: Arial, sans-serif; line-height: 1.5; font-size: 15px; }
                h1, h2, h3 { color: #333; }
                a { color: #1a73e8; text-decoration: none; }
                ul { padding-left: 20px; }
                li { margin-bottom: 6px; }
              </style>
            </head>
            <body>
              ${rawHtml}
            </body>
          </html>`
      }
    }];
    ```  
    - Connect Markdown output to this node.

18. **Add an Email Send node "Email Report"**:  
    - Subject: `Your N8N report about {{ $('Loop Over Items').item.json.title }}`  
    - To Email: `jotunheim166@gmail.com`  
    - From Email: `n8n-mail@example.com` (replace with valid sender email)  
    - HTML Body: `={{ $json.html }}` from Code - Build HTML node  
    - Credentials: SMTP account with valid configuration  
    - Connect Code - Build HTML output to this node.

19. **Connect Email Report output back to "Loop Over Items" to continue processing next item until all are done.**

20. **Add a Sticky Note near the input nodes** with the content:  
    ```
    ⚠️ Note
    Item searching in the dataset is case-sensitive.

    Make sure that the item names in your table are typed correctly.
    For example:
    Use "Iphone" instead of "iphone", "GeForce" instead of "Geforce", and so on.

    Otherwise, your search may return zero results.
    ```

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Item searching in Bright Data dataset is case-sensitive; ensure correct casing in Google Sheets input to avoid zero results. | Sticky note attached to input preparation nodes                                                       |
| Uses Bright Data marketplace dataset "Google Shopping" to filter product listings by title and price availability.         | Dataset ID `gd_ltppk50q18kdw67omz` in Bright Data node configuration                                   |
| AI analysis leverages Google's Gemini 2.0 model via LangChain for generating ranked price comparison reports.              | Google Gemini Chat Model node with model `models/gemini-2.0-flash` and Google PaLM API credentials      |
| Email reports are sent via SMTP; ensure SMTP credentials are valid and sender email is authorized to send emails.          | Email Report node configuration                                                                        |

---

**Disclaimer:**  
The provided text is directly derived from an automated workflow built with n8n, an integration and automation tool. All processing respects applicable content policies and contains no illegal or offensive material. All data processed is legal and publicly accessible.