Create a Lookio RAG assistant from a CSV text corpus

https://n8nworkflows.xyz/workflows/create-a-lookio-rag-assistant-from-a-csv-text-corpus-11877


# Create a Lookio RAG assistant from a CSV text corpus

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow creates a new **Lookio RAG Assistant** from a **CSV text corpus**. It collects a CSV and assistant configuration via an n8n Form, converts each CSV row into a separate text file, uploads each file to Lookio as a resource, then creates an assistant whose allowed knowledge base is restricted to the uploaded resources.

**Target use cases:**
- Quickly building a scoped assistant for a project (RFPs, HR policies, internal docs) from a CSV export.
- Turning tabular datasets (title + content) into retrievable “documents” inside Lookio.

**Logical blocks (based on dependencies):**
1. **1.1 User Input & CSV Intake**: Form collects CSV + assistant settings + column mapping.
2. **1.2 Parse CSV → Items**: Extract CSV rows into JSON items.
3. **1.3 Row → Text File Conversion**: Turn each row’s text content into a `.txt` binary file named from the title column.
4. **1.4 Batch Loop & Resource Upload**: Iterate items and upload each generated file to Lookio.
5. **1.5 Collect Resource IDs & Provision Assistant**: Aggregate upload responses, extract Resource IDs, create Lookio assistant limited to those IDs.
6. **1.6 User Completion Page**: Display success message with assistant ID and uploaded resource count.

---

## 2. Block-by-Block Analysis

### 1.1 User Input & CSV Intake

**Overview:**  
Collects the CSV file and the assistant configuration (name/context/guidelines) plus the CSV header names to map title and content columns.

**Nodes involved:**
- **Upload a CSV** (Form Trigger)
- **Sticky Note**, **Sticky Note2** (documentation)

#### Node: Upload a CSV
- **Type / role:** `Form Trigger` — entry point; hosts a form and starts the workflow on submission.
- **Configuration (interpreted):**
  - Form title: **“CSV”**
  - Fields (all required):
    - File upload field labeled **csv** accepting `.csv`
    - Assistant name
    - Assistant context
    - Assistant output guidelines
    - Name of the column that contains the resource text (e.g., `content`)
    - Name of the column that contains the resource title (e.g., `title`)
- **Key variables/outputs:**
  - Outputs binary file in **`$binary.csv`**
  - Outputs form fields in **`$json[...]`** (e.g., `$json["Assistant name"]`)
- **Connections:**
  - Output → **Convert CSV to JSON**
- **Edge cases / failures:**
  - Missing/incorrect column header names will break later expressions (title/text lookup).
  - CSV formatting issues (delimiter, quoting, encoding) may prevent correct parsing.
  - Large CSVs can lead to many items and longer run time.
- **Version notes:** Form Trigger `typeVersion 2.3` (behavior/field options may vary across n8n versions).

#### Node: Sticky Note
- **Type / role:** `Sticky Note` — explains the end-to-end purpose and requirements.
- **Notable content (kept):**
  - Link: https://www.lookio.app
  - Mentions replacing `<YOUR-API-KEY>` and `<YOUR-WORKSPACE-ID>` in two HTTP nodes.
  - CSV must have title and content columns; form maps headers.

#### Node: Sticky Note2
- **Type / role:** `Sticky Note` — guidance on form mapping and assistant settings.
- **Edge-case guidance:** stresses exact CSV header names.

---

### 1.2 Parse CSV → Items

**Overview:**  
Reads the uploaded CSV binary and converts it to JSON items (one item per row).

**Nodes involved:**
- **Convert CSV to JSON**

#### Node: Convert CSV to JSON
- **Type / role:** `Extract From File` — parses a file from binary into structured data.
- **Configuration:**
  - Reads from binary property **`csv`** (the uploaded file field name).
  - Operation implied: CSV extraction to JSON items.
- **Connections:**
  - Input: **Upload a CSV**
  - Output → **Convert to txt**
- **Edge cases / failures:**
  - If the file field name changes from `csv`, this node will not find the binary.
  - Parsing may fail for malformed CSV, unexpected delimiters, or inconsistent row widths.
- **Version notes:** `typeVersion 1.1`.

---

### 1.3 Row → Text File Conversion

**Overview:**  
For each CSV row, creates a text binary file containing the resource text, and names the file using the resource title column (as specified in the form).

**Nodes involved:**
- **Convert to txt**

#### Node: Convert to txt
- **Type / role:** `Convert to File` — transforms a JSON string into a binary text file.
- **Configuration:**
  - Operation: **toText**
  - **Source property expression** (text body):
    - Reads from the current CSV row using a *dynamic column name* provided in the form:
      - `sourceProperty = {{ $('Upload a CSV').item.json["Name of the column that contains the resource text"] }}`
    - That expression returns the header name; the node uses it to read the corresponding property from the current row.
  - **File name expression**:
    - `fileName = {{ $json[ $('Upload a CSV').item.json["Name of the column that contains the resource title"] ] }}`
    - This resolves to: “take this row’s value from the title column and use it as filename”.
  - Output binary property name: **`data`**
- **Connections:**
  - Input: **Convert CSV to JSON**
  - Output → **Loop Over Items**
- **Edge cases / failures:**
  - If the user typed a column name that doesn’t exist, `$json[...]` resolves to `undefined`, leading to empty fileName or empty text.
  - Titles with illegal filename characters can produce unexpected filenames.
  - Very large text fields can create large binaries and slow uploads.
- **Version notes:** `typeVersion 1.1`.

---

### 1.4 Batch Loop & Resource Upload

**Overview:**  
Iterates over all generated text files and uploads each as a Lookio resource. Uses a batching loop to manage sequential processing.

**Nodes involved:**
- **Loop Over Items**
- **Import resource to Lookio**
- **Sticky Note1**, **Sticky Note3** (credentials reminder)

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — controls iteration over items (batch processing loop).
- **Configuration:**
  - Default options (no explicit batch size shown; n8n defaults apply unless configured in UI).
- **Connections:**
  - Input: **Convert to txt**
  - Outputs:
    - Main output 1 → **Import resource to Lookio** (process each item)
    - Main output 2 → **Aggregate success messages** (runs when batches complete / no items left)
  - Additionally receives loop-back input from **Import resource to Lookio** to fetch the next batch.
- **Edge cases / failures:**
  - If batch size is too large, Lookio API may rate-limit; too small can slow workflow.
  - If an upload fails and stops execution, aggregation and assistant creation never happen.
- **Version notes:** `typeVersion 3`.

#### Node: Import resource to Lookio
- **Type / role:** `HTTP Request` — uploads each text file to Lookio as a resource via multipart form-data.
- **Configuration:**
  - Method: **POST**
  - URL: `https://api.lookio.app/webhook/add-resource`
  - Content-Type: **multipart-form-data**
  - Headers:
    - `api_key: <YOUR-API-KEY>` (**must be replaced**)
  - Body fields:
    - `workspace_id: <YOUR-WORKSPACE-ID>` (**must be replaced**)
    - `method: file`
    - `source_title: {{ $binary.data.fileName }}` (uses filename from conversion step)
    - `file`: binary upload from input field **`data`**
- **Connections:**
  - Input: **Loop Over Items**
  - Output → **Loop Over Items** (loop continuation)
- **Expected output:**
  - A response per uploaded file that includes a **Resource ID** field (used later). The workflow assumes the property name is exactly `"Resource ID"`.
- **Edge cases / failures:**
  - 401/403 if API key invalid.
  - 400 if workspace_id missing/invalid, or multipart payload incorrect.
  - If Lookio response schema changes (e.g., `"resource_id"` instead of `"Resource ID"`), later mapping will fail.
  - Timeouts for large files or network issues.
- **Version notes:** HTTP Request `typeVersion 4.3` (options/parameter UI differs by n8n version).

#### Sticky Note1 / Sticky Note3
- **Type / role:** `Sticky Note` — both remind to set Lookio API key and workspace ID (placed near relevant nodes).
- **Content:** “Make sure to set your Lookio API key and workspace ID in here.”

---

### 1.5 Collect Resource IDs & Provision Assistant

**Overview:**  
After all uploads finish, aggregates all upload responses, extracts the Resource IDs into an array, and creates a Lookio assistant limited to those resources.

**Nodes involved:**
- **Aggregate success messages**
- **Convert IDs into an Array**
- **Create Lookio assistant**

#### Node: Aggregate success messages
- **Type / role:** `Aggregate` — collects data from all processed items into a single item.
- **Configuration:**
  - Mode: **aggregateAllItemData** (aggregates all items’ JSON into one structure).
- **Connections:**
  - Input: “done” output of **Loop Over Items**
  - Output → **Convert IDs into an Array**
- **Edge cases / failures:**
  - If any upload items never produced output (failed mid-run), the aggregated set will be incomplete.
- **Version notes:** `typeVersion 1`.

#### Node: Convert IDs into an Array
- **Type / role:** `Set` — transforms aggregated data into the array needed by the assistant creation API.
- **Configuration:**
  - Creates field **`IDs`** (type: array) with expression:
    - `{{ $json.data.map(item => item["Resource ID"]) }}`
  - Assumes aggregated payload is in `$json.data` and each item has `"Resource ID"`.
- **Connections:**
  - Input: **Aggregate success messages**
  - Output → **Create Lookio assistant**
- **Edge cases / failures:**
  - If `"Resource ID"` is missing/renamed, resulting array contains `undefined` values.
  - If `$json.data` is not an array (aggregate node output differs), expression fails.
- **Version notes:** Set `typeVersion 3.4`.

#### Node: Create Lookio assistant
- **Type / role:** `HTTP Request` — provisions a Lookio assistant and scopes it to the uploaded resources.
- **Configuration:**
  - Method: **POST**
  - URL: `https://api.lookio.app/webhook/create-assistant`
  - Content-Type: **form-urlencoded**
  - Headers:
    - `api_key: <YOUR-API-KEY>` (**must be replaced**)
  - Body parameters:
    - `workspace_id: <YOUR-WORKSPACE-ID>` (**must be replaced**)
    - `assistant_name: {{ $('Upload a CSV').item.json['Assistant name'] }}`
    - `context: {{ $('Upload a CSV').item.json['Assistant context'] }}`
    - `output_guidelines: {{ $('Upload a CSV').item.json['Assistant output guidelines'] }}`
    - `resources_access_type: Limited selection`
    - `allowed_resources: {{ $json.IDs }}` (array of Resource IDs)
- **Connections:**
  - Input: **Convert IDs into an Array**
  - Output → **Success message form ending**
- **Expected output:**
  - Response should contain **Assistant ID** field used by the completion page (assumed property name: `"Assistant ID"`).
- **Edge cases / failures:**
  - Auth/workspace errors as above.
  - If API expects JSON array formatting but receives form-urlencoded stringified array, creation may fail depending on Lookio endpoint parsing.
  - If assistant response field names differ, completion message may show blank assistant ID.
- **Version notes:** HTTP Request `typeVersion 4.3`.

---

### 1.6 User Completion Page

**Overview:**  
Shows a completion screen confirming the assistant creation and indicating how many resources were uploaded.

**Nodes involved:**
- **Success message form ending**

#### Node: Success message form ending
- **Type / role:** `Form` (operation: completion) — displays an end screen to the form submitter.
- **Configuration:**
  - Completion title: **Successful upload!**
  - Completion message (dynamic):
    - Uses assistant name: `{{ $json['Assistant name'] }}`
    - Uses count of aggregated items: `{{ $('Aggregate success messages').item.json.data.length }}`
    - Shows assistant ID: `{{ $json['Assistant ID'] }}`
- **Connections:**
  - Input: **Create Lookio assistant**
  - Output: none (end)
- **Edge cases / failures:**
  - If assistant creation response does not include `Assistant name` or `Assistant ID` in the current item JSON, placeholders render empty.
  - If aggregation didn’t run (earlier failure), resource count expression may fail.
- **Version notes:** Form `typeVersion 2.3`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / overview & setup requirements | — | — | # Create a Lookio RAG Assistant from a CSV Text Corpus… Link: https://www.lookio.app … replace `<YOUR-API-KEY>` and `<YOUR-WORKSPACE-ID>` in `Create Lookio assistant` and `Import resource to Lookio`… |
| Sticky Note1 | Sticky Note | Credentials reminder near upload section | — | — | ## Action required \n\nMake sure to set your Lookio API key and workspace ID in here. |
| Convert to txt | Convert to File | Convert each CSV row text into binary .txt file | Convert CSV to JSON | Loop Over Items |  |
| Loop Over Items | Split In Batches | Iterate over items and control looping | Convert to txt; Import resource to Lookio | Import resource to Lookio; Aggregate success messages |  |
| Convert CSV to JSON | Extract From File | Parse uploaded CSV into JSON items | Upload a CSV | Convert to txt |  |
| Upload a CSV | Form Trigger | Collect CSV + assistant settings + column mapping | — | Convert CSV to JSON | ## Form Input Helper … (from Sticky Note2; applies to the form area) |
| Aggregate success messages | Aggregate | Aggregate all upload responses into one item | Loop Over Items | Convert IDs into an Array |  |
| Convert IDs into an Array | Set | Map aggregated results to array of Resource IDs | Aggregate success messages | Create Lookio assistant |  |
| Success message form ending | Form (completion) | Show completion screen with assistant details | Create Lookio assistant | — |  |
| Create Lookio assistant | HTTP Request | Create assistant limited to uploaded resources | Convert IDs into an Array | Success message form ending | ## Action required \n\nMake sure to set your Lookio API key and workspace ID in here. |
| Import resource to Lookio | HTTP Request | Upload each .txt as a Lookio resource | Loop Over Items | Loop Over Items | ## Action required \n\nMake sure to set your Lookio API key and workspace ID in here. |
| Sticky Note2 | Sticky Note | Documentation for form mapping and assistant settings | — | — | ## Form Input Helper … CSV Column Mapping … must specify exact CSV header names. |
| Sticky Note3 | Sticky Note | Credentials reminder near assistant creation section | — | — | ## Action required \n\nMake sure to set your Lookio API key and workspace ID in here. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add node: “Upload a CSV”**
   - Type: **Form Trigger**
   - Configure form fields:
     1) File field: label **csv**, required, accept `.csv`  
     2) Text fields (required): **Assistant name**, **Assistant context**, **Assistant output guidelines**  
     3) Text fields (required): **Name of the column that contains the resource text**, **Name of the column that contains the resource title**
   - This node is the **entry point**.

3. **Add node: “Convert CSV to JSON”**
   - Type: **Extract From File**
   - Set **Binary Property Name** = `csv`
   - Connect: **Upload a CSV → Convert CSV to JSON**

4. **Add node: “Convert to txt”**
   - Type: **Convert to File**
   - Operation: **To Text**
   - Source property (expression):  
     - Use the column name supplied by the form to select the row field that contains the body text.
   - Binary property name: `data`
   - Filename (expression):  
     - Use the column name supplied by the form to select the row field that contains the title; set that as the file name.
   - Connect: **Convert CSV to JSON → Convert to txt**

5. **Add node: “Loop Over Items”**
   - Type: **Split In Batches**
   - Leave defaults or set a batch size (e.g., 1–10 depending on API limits).
   - Connect: **Convert to txt → Loop Over Items**

6. **Add node: “Import resource to Lookio”**
   - Type: **HTTP Request**
   - Method: **POST**
   - URL: `https://api.lookio.app/webhook/add-resource`
   - Content type: **Multipart Form-Data**
   - Headers:
     - `api_key` = your Lookio API key
   - Body parameters:
     - `workspace_id` = your Lookio workspace ID
     - `method` = `file`
     - `source_title` = expression referencing the binary filename (from `data`)
     - `file` = **binary upload**, input field name `data`
   - Connect: **Loop Over Items (main) → Import resource to Lookio**
   - Connect loop-back: **Import resource to Lookio → Loop Over Items** (so it continues to the next batch/item)

7. **Add node: “Aggregate success messages”**
   - Type: **Aggregate**
   - Mode: **Aggregate All Item Data**
   - Connect: **Loop Over Items (done/no items output) → Aggregate success messages**

8. **Add node: “Convert IDs into an Array”**
   - Type: **Set**
   - Add field:
     - Name: `IDs`
     - Type: array
     - Value: expression mapping aggregated results to extract `"Resource ID"` from each upload response item.
   - Connect: **Aggregate success messages → Convert IDs into an Array**

9. **Add node: “Create Lookio assistant”**
   - Type: **HTTP Request**
   - Method: **POST**
   - URL: `https://api.lookio.app/webhook/create-assistant`
   - Content type: **Form-URL Encoded**
   - Headers:
     - `api_key` = your Lookio API key
   - Body parameters:
     - `workspace_id` = your Lookio workspace ID
     - `assistant_name` = from the form submission
     - `context` = from the form submission
     - `output_guidelines` = from the form submission
     - `resources_access_type` = `Limited selection`
     - `allowed_resources` = the `IDs` array
   - Connect: **Convert IDs into an Array → Create Lookio assistant**

10. **Add node: “Success message form ending”**
    - Type: **Form**
    - Operation: **Completion**
    - Set:
      - Completion title: “Successful upload!”
      - Completion message: include assistant name, count of uploaded resources (from the aggregate output), and assistant ID from the create-assistant response.
    - Connect: **Create Lookio assistant → Success message form ending**

11. **Credentials / constants to set**
    - Replace values in both HTTP nodes:
      - `api_key` header
      - `workspace_id` body parameter
    - No n8n credential object is used here; keys are embedded as header/body values. (You may optionally move them to environment variables or n8n credentials for safer handling.)

12. **Test run**
    - Run the form, upload a CSV, and ensure the two mapping fields exactly match your CSV headers (title + content).
    - Verify the upload responses contain `"Resource ID"` and assistant creation returns `"Assistant ID"`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Lookio product site | https://www.lookio.app |
| Setup requirement: replace `<YOUR-API-KEY>` and `<YOUR-WORKSPACE-ID>` in both HTTP Request nodes (`Import resource to Lookio`, `Create Lookio assistant`) | Mentioned in workflow notes |
| CSV must include at least Title + Content columns; you must enter the exact header names in the form | Form mapping requirement |
| Credit: “A workflow by Guillaume Duvernay” | Included in the workflow notes |