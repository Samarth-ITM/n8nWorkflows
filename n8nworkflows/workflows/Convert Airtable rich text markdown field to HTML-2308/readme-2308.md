Convert Airtable rich text markdown field to HTML

https://n8nworkflows.xyz/workflows/convert-airtable-rich-text-markdown-field-to-html-2308


# Convert Airtable rich text markdown field to HTML

### 1. Workflow Overview

This workflow converts Airtable rich text content written in Markdown format into HTML format. It supports two use cases: converting a single specified Airtable record or converting all records in the Airtable table in bulk. The converted HTML content is then updated back into the Airtable records, enabling easier integration with systems or platforms that require HTML instead of Markdown.

Logical blocks:

- **1.1 Input Reception**: Receives webhook requests containing parameters to trigger either single-record or bulk conversion.
- **1.2 Conditional Routing**: Determines whether to process a single record or all records based on input.
- **1.3 Single Record Processing**: Retrieves one Airtable record by ID, converts its Markdown field to HTML, and updates that record.
- **1.4 Bulk Records Processing**: Retrieves multiple Airtable records, converts each Markdown field to HTML, and updates all records.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Accepts HTTP webhook requests that trigger the workflow and provide parameters, specifically a recordId or the keyword "all" to select the processing mode.

- **Nodes Involved:**  
  - Airtable sync video description (Webhook)

- **Node Details:**

  - **Airtable sync video description**  
    - Type: Webhook  
    - Role: Entry point accepting HTTP requests to start the workflow.  
    - Configuration:  
      - Webhook path: `848644e5-6b1d-42b3-9259-5828c29780a8`  
      - No authentication or additional options configured (default webhook).  
    - Input: External HTTP call with query parameters (expects `recordId`).  
    - Output: JSON containing the query parameter `recordId` used downstream to decide processing mode.  
    - Possible failures:  
      - No or invalid query parameters could cause subsequent nodes to fail or take incorrect paths.  
      - Webhook timeout if downstream processing is delayed.  
    - Version: 2

---

#### 2.2 Conditional Routing

- **Overview:**  
  Checks whether the incoming request is for a single record conversion or for all records, routing the flow accordingly.

- **Nodes Involved:**  
  - Check if it's 1 record or all records - Airtable (IF)

- **Node Details:**

  - **Check if it's 1 record or all records - Airtable**  
    - Type: If  
    - Role: Conditional branching based on the `recordId` query parameter.  
    - Configuration:  
      - Condition: Checks if the expression `{{$json.query.recordId}}` equals the string `"all"`.  
      - If true: routes to "Get all records from airtable" (bulk processing).  
      - If false: routes to "Get single record from airtable" (single record processing).  
    - Input: JSON from webhook node containing `query.recordId`.  
    - Output: Two branches - True (bulk) and False (single).  
    - Edge cases:  
      - Missing `recordId` parameter causes false branch (single record) but the record ID may be empty, leading to errors in fetching.  
      - Case sensitivity is enabled - `"all"` must be lowercase exactly.  
    - Version: 2

---

#### 2.3 Single Record Processing

- **Overview:**  
  Retrieves a single record from Airtable by its ID, converts the Markdown field "📥 Video Description" to HTML, then updates the record with the converted HTML.

- **Nodes Involved:**  
  - Get single record from airtable (Airtable)  
  - Convert markdown to HTML1 (Markdown)  
  - Update single record in airtable (Airtable)

- **Node Details:**

  - **Get single record from airtable**  
    - Type: Airtable  
    - Role: Fetches one Airtable record by ID.  
    - Configuration:  
      - Operation: "get"  
      - Base: "360Creators" (Airtable base ID: `appk2iZaWQLO3Tqvx`)  
      - Table: "📺 Videos" (Table ID: `tblbYRGFgCo9u2VpB`)  
      - Record ID: dynamically set to `{{$json.query.recordId}}` from webhook input.  
    - Credentials: Uses stored Airtable API token named "🏰 360Creators".  
    - Output: Single record JSON including all fields.  
    - Edge cases:  
      - Invalid or non-existent record ID results in Airtable API error.  
      - API rate limits or authorization errors possible.  
    - Version: 2

  - **Convert markdown to HTML1**  
    - Type: Markdown  
    - Role: Converts the Markdown text in the field "📥 Video Description" to HTML string.  
    - Configuration:  
      - Mode: "markdownToHtml"  
      - Options: Simplified autolinks enabled (improves link detection)  
      - Markdown input: `{{$json["📥 Video Description"]}}`  
    - Input: Output JSON from previous node.  
    - Output: JSON with `data` property containing converted HTML.  
    - Edge cases:  
      - Empty or invalid Markdown input may result in empty or malformed HTML.  
      - Expression failure if field missing or null.  
    - Version: 1

  - **Update single record in airtable**  
    - Type: Airtable  
    - Role: Updates the original Airtable record with the converted HTML in the field "Video description HTML".  
    - Configuration:  
      - Operation: "update"  
      - Base and Table: Same as Get node  
      - Columns mapping:  
        - `id` mapped from `{{$json.id}}` (record identifier from Get node)  
        - `Video description HTML` set to `{{$json.data}}` (HTML from Markdown node)  
    - Credentials: Same Airtable token.  
    - Input: Output from Markdown conversion node.  
    - Possible failures:  
      - Update failure due to invalid record ID or API limits.  
    - Version: 2

---

#### 2.4 Bulk Records Processing

- **Overview:**  
  Retrieves multiple records from the Airtable table, converts the Markdown description field for each record to HTML, and updates all records with the converted HTML.

- **Nodes Involved:**  
  - Get all records from airtable (Airtable)  
  - Convert markdown to HTML2 (Markdown)  
  - Update all records in airtable (Airtable)

- **Node Details:**

  - **Get all records from airtable**  
    - Type: Airtable  
    - Role: Fetches multiple records from the Airtable table to process in bulk.  
    - Configuration:  
      - Operation: "search"  
      - Limit: 3 records (Note: limited to 3, possibly for testing or performance)  
      - Base and Table: Same as single record nodes  
      - ReturnAll: false (only limited number as above)  
    - Credentials: Airtable API token.  
    - Output: Array of record JSON objects.  
    - Edge cases:  
      - Large datasets require paging or returnAll=true for full sync.  
      - API rate limits.  
    - Version: 2

  - **Convert markdown to HTML2**  
    - Type: Markdown  
    - Role: Converts each record’s Markdown field "📥 Video Description" to HTML in batch.  
    - Configuration:  
      - Mode: "markdownToHtml"  
      - No special options enabled.  
      - Markdown input: `{{$json["📥 Video Description"]}}` (per record)  
    - Input: Array of records from previous node.  
    - Output: For each input item, returns the converted HTML in `data` property.  
    - Edge cases:  
      - Empty or null Markdown fields yield empty HTML.  
      - Expression failure if field missing.  
    - Version: 1

  - **Update all records in airtable**  
    - Type: Airtable  
    - Role: Updates multiple records with converted HTML content.  
    - Configuration:  
      - Operation: "update"  
      - Base and Table: Same as above  
      - Columns mapping:  
        - `id` from `{{$json.id}}`  
        - `Unpublished` explicitly set to false (resets or overrides this field)  
        - `Video description HTML` set to `{{$json.data}}` (converted HTML)  
    - Credentials: Airtable API token.  
    - Input: Output from Markdown conversion node (array).  
    - Edge cases:  
      - API update failures due to partial data or concurrent edits.  
      - Bulk update limits - batch size constraints by Airtable API (not shown here).  
    - Version: 2

---

### 3. Summary Table

| Node Name                          | Node Type     | Functional Role                                      | Input Node(s)                    | Output Node(s)                        | Sticky Note                                    |
|-----------------------------------|---------------|-----------------------------------------------------|---------------------------------|-------------------------------------|------------------------------------------------|
| Airtable sync video description   | Webhook       | Entry point to receive HTTP requests                 | None                            | Check if it's 1 record or all records - Airtable | # Tutorial [Youtube video](https://www.youtube.com/watch?v=PAoxZjICd7o) |
| Check if it's 1 record or all records - Airtable | If            | Routes processing based on recordId parameter        | Airtable sync video description | Get single record from airtable; Get all records from airtable |                                                |
| Get single record from airtable   | Airtable      | Retrieves one Airtable record by ID                   | Check if it's 1 record or all records - Airtable (false branch) | Convert markdown to HTML1                 |                                                |
| Convert markdown to HTML1          | Markdown      | Converts single record's Markdown to HTML             | Get single record from airtable  | Update single record in airtable    |                                                |
| Update single record in airtable  | Airtable      | Updates one Airtable record with converted HTML       | Convert markdown to HTML1        | None                              |                                                |
| Get all records from airtable     | Airtable      | Retrieves multiple Airtable records for bulk processing | Check if it's 1 record or all records - Airtable (true branch) | Convert markdown to HTML2                 |                                                |
| Convert markdown to HTML2          | Markdown      | Converts each record's Markdown field to HTML         | Get all records from airtable    | Update all records in airtable      |                                                |
| Update all records in airtable    | Airtable      | Bulk updates Airtable records with converted HTML     | Convert markdown to HTML2        | None                              |                                                |
| Sticky Note                      | Sticky Note   | Provides tutorial link for the workflow                | None                            | None                              | # Tutorial [Youtube video](https://www.youtube.com/watch?v=PAoxZjICd7o) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Airtable sync video description`  
   - Type: Webhook  
   - Webhook Path: `848644e5-6b1d-42b3-9259-5828c29780a8` (or any unique path)  
   - No authentication needed  
   - This node triggers the workflow with a query parameter `recordId`  

2. **Create If Node for Conditional Routing**  
   - Name: `Check if it's 1 record or all records - Airtable`  
   - Type: If  
   - Condition:  
     - Type: String  
     - Operation: Exists (check if the field exists)  
     - Left Value: Expression `{{$json.query.recordId}}`  
     - Check if it equals the string `"all"` (case sensitive)  
   - Connect `Airtable sync video description` output to this node input  

3. **Create Airtable Node to Get Single Record**  
   - Name: `Get single record from airtable`  
   - Type: Airtable  
   - Credentials: Use Airtable API token credential (create if needed)  
   - Operation: Get  
   - Base: Select base "360Creators" or enter base ID `appk2iZaWQLO3Tqvx`  
   - Table: Select table "📺 Videos" or enter table ID `tblbYRGFgCo9u2VpB`  
   - Record ID: Set to expression `{{$json.query.recordId}}`  
   - Connect the false branch of the If node to this node  

4. **Create Markdown Node to Convert Single Record Markdown to HTML**  
   - Name: `Convert markdown to HTML1`  
   - Type: Markdown  
   - Mode: markdownToHtml  
   - Options: Enable Simplified Auto Link  
   - Markdown Input: Expression `{{$json["📥 Video Description"]}}`  
   - Connect output of `Get single record from airtable` to this node  

5. **Create Airtable Node to Update Single Record**  
   - Name: `Update single record in airtable`  
   - Type: Airtable  
   - Credentials: Same Airtable token  
   - Operation: Update  
   - Base and Table: Same as above  
   - Columns to update:  
     - `id`: `{{$json.id}}` (record ID from Get node)  
     - `Video description HTML`: `{{$json.data}}` (converted HTML from Markdown node)  
   - Connect output of `Convert markdown to HTML1` to this node  

6. **Create Airtable Node to Get Multiple Records**  
   - Name: `Get all records from airtable`  
   - Type: Airtable  
   - Credentials: Airtable token  
   - Operation: Search  
   - Base and Table: Same as above  
   - Limit: 3 (adjust as needed or set Return All true for full sync)  
   - Connect true branch of If node to this node  

7. **Create Markdown Node to Convert Bulk Records Markdown to HTML**  
   - Name: `Convert markdown to HTML2`  
   - Type: Markdown  
   - Mode: markdownToHtml  
   - Options: None (default)  
   - Markdown Input: Expression `{{$json["📥 Video Description"]}}`  
   - Connect output of `Get all records from airtable` to this node  

8. **Create Airtable Node to Update All Records**  
   - Name: `Update all records in airtable`  
   - Type: Airtable  
   - Credentials: Airtable token  
   - Operation: Update  
   - Base and Table: Same as above  
   - Columns to update:  
     - `id`: `{{$json.id}}`  
     - `Unpublished`: `false` (boolean literal)  
     - `Video description HTML`: `{{$json.data}}`  
   - Connect output of `Convert markdown to HTML2` to this node  

9. **(Optional) Add Sticky Note**  
   - Content: `# Tutorial\n[Youtube video](https://www.youtube.com/watch?v=PAoxZjICd7o)`  
   - Place near webhook node for documentation  

10. **Save and Activate Workflow**  
    - Test by sending HTTP requests to webhook URL with `recordId` parameter set to a valid Airtable record ID for single processing or `all` for bulk processing.  

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                  |
|------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow converts Markdown rich text fields from Airtable to HTML to facilitate HTML usage.    | Workflow purpose                                                |
| Tutorial video explaining this workflow is available on YouTube.                               | https://www.youtube.com/watch?v=PAoxZjICd7o                    |
| Airtable API token credential "🏰 360Creators" must be set up with read/write access.           | Credential setup                                                |
| Bulk processing node limits (default 3 records) can be increased by adjusting the Airtable node parameters or enabling pagination/returnAll. | Bulk processing considerations                                 |
| Markdown node supports options like simplified autolink to improve link handling.               | Markdown conversion options                                     |

---

This documentation provides a complete, detailed explanation of the workflow logic, nodes, configuration, and reproduction instructions for both human developers and automation systems.