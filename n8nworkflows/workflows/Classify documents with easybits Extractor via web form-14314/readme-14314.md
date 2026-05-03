Classify documents with easybits Extractor via web form

https://n8nworkflows.xyz/workflows/classify-documents-with-easybits-extractor-via-web-form-14314


# Classify documents with easybits Extractor via web form

# 1. Workflow Overview

This workflow receives a document through an n8n-hosted web form, converts the uploaded file into a base64 data URI, and sends it to the easybits Extractor API for document classification. Its primary purpose is to classify uploaded documents into predefined categories such as invoice types or any other labels configured in an easybits Extractor pipeline.

Typical use cases include:

- Classifying invoices by type
- Sorting uploaded business documents before storage
- Triggering downstream routing based on document category
- Detecting unsupported or uncertain documents via a `null` classification result

The workflow is organized into the following logical blocks:

## 1.1 Form-Based Input Reception

This block exposes a web form where a user uploads a file. It is the workflow entry point.

## 1.2 File Content Extraction

This block converts the uploaded binary file into a string property that can be reused in later steps.

## 1.3 Data URI Construction

This block rebuilds the extracted file content into a proper `data:` URI by combining the original MIME type with the base64 payload.

## 1.4 Classification via easybits Extractor

This block sends the generated data URI to the easybits Extractor API using a POST request authenticated with Bearer token credentials.

---

# 2. Block-by-Block Analysis

## 2.1 Form-Based Input Reception

**Overview:**  
This block collects a file upload from the user through an n8n form trigger. It starts the workflow and provides the uploaded document as binary data.

**Nodes Involved:**

- On form submission

### Node Details

#### On form submission

- **Type and technical role:**  
  `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that creates a hosted form and starts the workflow when the form is submitted.

- **Configuration choices:**  
  - Form title is set to **Image Upload**
  - One form field is defined:
    - Type: **file**
    - Label: **image**
  - No extra form options are configured

- **Key expressions or variables used:**  
  None inside the node itself, but the uploaded binary is later referenced as:
  - `$('On form submission').first().binary.image.mimeType`

- **Input and output connections:**  
  - Input: none, this is the trigger
  - Output: connected to **Extract from File**

- **Version-specific requirements:**  
  - Uses **typeVersion 2.5**
  - Requires a version of n8n that supports the Form Trigger node and hosted form functionality

- **Edge cases or potential failure types:**  
  - Form may receive unsupported file types unless validated elsewhere
  - If no file is uploaded, downstream nodes will fail because `binary.image` will not exist
  - Large files may hit instance upload size or memory limits
  - If the form URL is not publicly accessible, external users cannot submit files

- **Sub-workflow reference:**  
  None

---

## 2.2 File Content Extraction

**Overview:**  
This block transforms the uploaded binary file into a property on the JSON payload. The resulting content is then used to construct the data URI sent to easybits.

**Nodes Involved:**

- Extract from File

### Node Details

#### Extract from File

- **Type and technical role:**  
  `n8n-nodes-base.extractFromFile`  
  Utility node that reads binary file content and converts it into a normal JSON property.

- **Configuration choices:**  
  - Operation: **binaryToPropery**
  - Binary property name: **image**
  - No additional options set

- **Key expressions or variables used:**  
  - Reads binary input from `binary.image`
  - Produces JSON output in a property named `data`

- **Input and output connections:**  
  - Input: **On form submission**
  - Output: connected to **Edit Fields**

- **Version-specific requirements:**  
  - Uses **typeVersion 1.1**
  - Behavior depends on the Form Trigger output preserving the uploaded file in `binary.image`

- **Edge cases or potential failure types:**  
  - If the form field name changes from `image`, this node will fail unless updated
  - If no binary input exists, conversion fails
  - Non-standard file encodings or malformed binary payloads may produce unusable output
  - Very large files may increase memory usage during conversion

- **Sub-workflow reference:**  
  None

---

## 2.3 Data URI Construction

**Overview:**  
This block creates a valid base64 data URI by combining the uploaded file’s MIME type with the extracted file content. This is required because the easybits API expects the file in data URI format.

**Nodes Involved:**

- Edit Fields

### Node Details

#### Edit Fields

- **Type and technical role:**  
  `n8n-nodes-base.set`  
  Adds or overwrites fields in the item JSON.

- **Configuration choices:**  
  - Creates a field named `data`
  - Type: string
  - Value is dynamically built from the original MIME type and the extracted file payload

- **Key expressions or variables used:**  
  The assigned value is:
  - `data:{{ $('On form submission').first().binary.image.mimeType }};base64,{{ $json.data }}`
  
  This expression:
  - fetches MIME type from the original trigger node
  - appends the base64 content produced by **Extract from File**
  - produces a string such as:
    - `data:application/pdf;base64,...`
    - `data:image/png;base64,...`
    - `data:image/jpeg;base64,...`

- **Input and output connections:**  
  - Input: **Extract from File**
  - Output: connected to **easybits Extractor for Classification**

- **Version-specific requirements:**  
  - Uses **typeVersion 3.4**
  - Depends on expression support for cross-node access via `$('On form submission').first()`

- **Edge cases or potential failure types:**  
  - If `binary.image.mimeType` is missing, the data URI will be malformed
  - If `$json.data` is empty or not base64 content, the API request will likely fail
  - If the trigger node returns multiple items in future extensions, `.first()` may use the wrong item
  - If the field name changes from `image`, the MIME type lookup expression must also change

- **Sub-workflow reference:**  
  None

---

## 2.4 Classification via easybits Extractor

**Overview:**  
This block sends the prepared document to the easybits Extractor API and receives the classification result. It is the final operational step of the workflow.

**Nodes Involved:**

- easybits Extractor for Classification

### Node Details

#### easybits Extractor for Classification

- **Type and technical role:**  
  `n8n-nodes-base.httpRequest`  
  Makes an authenticated HTTP POST request to the easybits Extractor API.

- **Configuration choices:**  
  - Method: **POST**
  - URL:  
    `https://extractor.easybits.tech/api/pipelines/YOUR_PIPELINE_ID`
  - Body type: **JSON**
  - Authentication: **predefined credential type**
  - Credential type: **httpBearerAuth**
  - Request body:
    ```json
    {
      "files": [
        "{{ $json.data }}"
      ]
    }
    ```

- **Key expressions or variables used:**  
  - `{{ $json.data }}`  
    Sends the constructed data URI as the first element of the `files` array

- **Input and output connections:**  
  - Input: **Edit Fields**
  - Output: none

- **Version-specific requirements:**  
  - Uses **typeVersion 4.3**
  - Requires an n8n version supporting this HTTP Request node configuration format and predefined Bearer Auth credentials

- **Edge cases or potential failure types:**  
  - If `YOUR_PIPELINE_ID` is not replaced, the request will fail
  - Invalid or missing Bearer token causes authentication errors
  - Network issues, API downtime, or timeout can interrupt execution
  - If the easybits pipeline is not configured with a `document_class` field, output may not match expectations
  - If the API expects a stricter file format than provided, request validation may fail
  - If classification is uncertain, the API may return `null`, which downstream logic must handle

- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Receives uploaded file via hosted web form and starts the workflow |  | Extract from File | ## 📋 Form Upload<br>Accepts a file upload via a **web form**. Supports **PDF, PNG, and JPEG**. |
| Extract from File | Extract From File | Converts uploaded binary file into a JSON/base64-compatible property | On form submission | Edit Fields | ## 📄 Extract to Base64<br>Converts the uploaded **binary file** into a base64-encoded string stored in `data`. |
| Edit Fields | Set | Builds the final data URI from MIME type + base64 payload | Extract from File | easybits Extractor for Classification | ## 🔗 Build Data URI<br>Dynamically reads the **MIME type** from the uploaded file and prepends it as a base64 data URI. |
| easybits Extractor for Classification | HTTP Request | Sends the document to easybits Extractor API for classification | Edit Fields |  | ## 🚀 Send to easybits<br>POSTs the data URI to the **easybits Extractor API** for classifcation. |
| Sticky Note | Sticky Note | Visual documentation for upload block |  |  | ## 📋 Form Upload<br>Accepts a file upload via a **web form**. Supports **PDF, PNG, and JPEG**. |
| Sticky Note1 | Sticky Note | Visual documentation for base64 extraction block |  |  | ## 📄 Extract to Base64<br>Converts the uploaded **binary file** into a base64-encoded string stored in `data`. |
| Sticky Note2 | Sticky Note | Visual documentation for data URI construction block |  |  | ## 🔗 Build Data URI<br>Dynamically reads the **MIME type** from the uploaded file and prepends it as a base64 data URI. |
| Sticky Note3 | Sticky Note | Visual documentation for API submission block |  |  | ## 🚀 Send to easybits<br>POSTs the data URI to the **easybits Extractor API** for classifcation. |
| Sticky Note4 | Sticky Note | Global workflow documentation and setup guidance |  |  | # 📄 easybits' Document Classification<br><br>## What This Workflow Does<br>Upload a document (PDF, PNG, or JPEG) via a hosted web form and let **easybits Extractor** classify it into one of your defined categories. This workflow handles the upload, file conversion, and API call – you just define the categories.<br><br>## How It Works<br>1. **Form Upload** – A user uploads a file through the n8n web form<br>2. **Base64 Conversion** – The binary file is extracted and converted to a base64 string<br>3. **Build Data URI** – The correct MIME type is read from the original upload and prepended to create a proper data URI<br>4. **Classification via easybits** – The data URI is POSTed to the easybits Extractor API, which analyzes the document and returns a `document_class` label (e.g. `medical_invoice`, `hotel_invoice`, or `null` if uncertain)<br><br>From here, you can extend the workflow however you like – route files to Google Drive folders, send Slack alerts for unrecognized documents, log results to a spreadsheet, etc.<br><br>---<br><br>## Setup Guide<br><br>### 1. Create Your easybits Extractor Pipeline<br>1. Go to **extractor.easybits.tech** and create a new pipeline<br>2. Add a single field to the mapping called **`document_class`**<br>3. In that field's description, paste a classification prompt that tells the model which categories exist and how to identify each one (see the "📊 Document Classification" sticky note in this workflow for a full example prompt you can copy and adapt)<br>4. The prompt should instruct the model to return **exactly one category label** – or `null` if the document doesn't match any category. No explanations, no extra text.<br>5. Adjust the categories to fit your use case. The example uses `medical_invoice`, `restaurant_invoice`, and `hotel_invoice` – but you can define whatever classes you need.<br>6. Copy your **Pipeline ID** and **API Key**<br><br>### 2. Connect the Nodes in n8n<br>1. In the **easybits Extractor (Classification)** HTTP node, replace the pipeline URL with your own: `https://extractor.easybits.tech/api/pipelines/YOUR_PIPELINE_ID`<br>2. Create a **Bearer Auth** credential using your easybits API Key and assign it to that node<br><br>### 3. Activate & Test<br>1. Click **Active** in the top-right corner of n8n<br>2. Open the form URL and upload a test document<br>3. Check the execution output – you should see your `document_class` label in the response |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: **easybits' Extractor Classification Workflow**
   - Leave it inactive until configuration is complete

2. **Add the trigger node**
   - Create a **Form Trigger** node
   - Name it: **On form submission**
   - Set the form title to: **Image Upload**
   - Add one form field:
     - Field type: **File**
     - Field label/name: **image**
   - Keep other options at default unless you need custom validation or branding

3. **Add the file extraction node**
   - Create an **Extract From File** node
   - Name it: **Extract from File**
   - Connect **On form submission → Extract from File**
   - Configure:
     - Operation: **Binary to Property**
     - Binary Property Name: **image**
   - This converts the uploaded file from `binary.image` into a regular JSON property named `data`

4. **Add the field transformation node**
   - Create a **Set** node
   - Name it: **Edit Fields**
   - Connect **Extract from File → Edit Fields**
   - Add one assignment:
     - Field name: **data**
     - Type: **String**
     - Value:
       ```text
       =data:{{ $('On form submission').first().binary.image.mimeType }};base64,{{ $json.data }}
       ```
   - This creates a proper data URI using:
     - the MIME type from the uploaded file
     - the base64 content extracted in the previous node

5. **Add the HTTP request node**
   - Create an **HTTP Request** node
   - Name it: **easybits Extractor for Classification**
   - Connect **Edit Fields → easybits Extractor for Classification**
   - Configure:
     - Method: **POST**
     - URL:
       `https://extractor.easybits.tech/api/pipelines/YOUR_PIPELINE_ID`
     - Replace `YOUR_PIPELINE_ID` with the pipeline ID from easybits
     - Authentication: **Predefined Credential Type**
     - Credential Type: **HTTP Bearer Auth**
     - Send Body: **true**
     - Specify Body: **JSON**
     - JSON Body:
       ```json
       {
         "files": [
           "{{ $json.data }}"
         ]
       }
       ```

6. **Create the easybits Bearer Auth credential**
   - In n8n credentials, create a new **HTTP Bearer Auth** credential
   - Paste your easybits API key as the bearer token
   - Assign this credential to the **easybits Extractor for Classification** node

7. **Create the pipeline in easybits Extractor**
   - Open `https://extractor.easybits.tech`
   - Create a new pipeline
   - Add a single output field named:
     - `document_class`
   - In the field description, write a classification instruction telling the model:
     - what classes exist
     - how to decide among them
     - to return exactly one label
     - to return `null` if uncertain or not matching any class
   - Save the pipeline and copy:
     - Pipeline ID
     - API key

8. **Validate file handling assumptions**
   - Ensure the uploaded form field is still named `image`
   - Ensure expected MIME types match your input documents:
     - `application/pdf`
     - `image/png`
     - `image/jpeg`
   - If you rename the file field, update both:
     - the **Extract from File** node binary property
     - the MIME type expression in **Edit Fields**

9. **Optionally add visual notes**
   - Add Sticky Note nodes if you want the workflow to remain self-documented
   - Suggested note areas:
     - form upload
     - base64 extraction
     - data URI construction
     - easybits API submission
     - global setup instructions

10. **Test the workflow**
    - Save the workflow
    - Open the Form Trigger test or production URL
    - Upload a sample PDF, PNG, or JPEG
    - Verify:
      - the Form Trigger receives the file in `binary.image`
      - **Extract from File** produces `data`
      - **Edit Fields** produces a valid `data:<mime>;base64,...` string
      - the HTTP node returns a response containing the classification result

11. **Activate the workflow**
    - Once testing succeeds, activate it
    - Share the hosted form URL with users or embed it where needed

12. **Plan downstream handling**
    - If `document_class` can be `null`, add conditional logic after the HTTP node in future revisions
    - Common extensions include:
      - If node for routing by class
      - cloud storage by folder
      - notifications for unknown documents
      - spreadsheet/database logging

**Sub-workflow setup:**  
This workflow does not invoke any sub-workflows and does not require any called workflow configuration.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| easybits Extractor platform for creating the classification pipeline | https://extractor.easybits.tech |
| The workflow expects an easybits pipeline with a field named `document_class` | easybits pipeline configuration |
| The API endpoint pattern is `https://extractor.easybits.tech/api/pipelines/YOUR_PIPELINE_ID` | Replace with your real pipeline ID |
| Authentication uses n8n HTTP Bearer Auth credentials with the easybits API key | n8n credential setup |
| Example classifications mentioned in the workflow notes: `medical_invoice`, `restaurant_invoice`, `hotel_invoice`, or `null` | easybits classification design |
| Suggested extensions include routing files to Google Drive, sending Slack alerts, or logging results to spreadsheets | Post-classification automation ideas |