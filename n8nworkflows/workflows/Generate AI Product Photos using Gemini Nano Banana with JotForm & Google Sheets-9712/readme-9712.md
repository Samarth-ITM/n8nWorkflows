Generate AI Product Photos using Gemini Nano Banana with JotForm & Google Sheets

https://n8nworkflows.xyz/workflows/generate-ai-product-photos-using-gemini-nano-banana-with-jotform---google-sheets-9712


# Generate AI Product Photos using Gemini Nano Banana with JotForm & Google Sheets

### 1. Workflow Overview

This workflow automates the generation of AI-enhanced product photos using the Gemini Nano Banana model from Google’s generative language API, integrating input data from JotForm submissions and Google Sheets. It is designed for product photographers, e-commerce managers, or marketing teams who want to create AI-generated product images based on detailed product descriptions and reference images.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Scheduling:** Triggers the workflow either on a schedule or when a JotForm is submitted, fetching pending product entries from Google Sheets.
- **1.2 Data Preparation:** Extracts and prepares product details and reference images from Google Sheets or JotForm input for processing.
- **1.3 AI Prompt Generation:** Sends product details to the Gemini language model to generate a text prompt for the image generation.
- **1.4 Image Generation with Gemini Nano Banana:** Uses the Gemini AI image model to generate product photos based on the prompt and reference images.
- **1.5 Image Processing and Storage:** Converts the generated AI image data into files, uploads them to Google Drive, and updates Google Sheets with image URLs and status.
- **1.6 Conditional Row Management:** Depending on whether the product row exists, the workflow updates the existing entry or adds a new row in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Scheduling

**Overview:**  
This block triggers the workflow either manually, on a schedule, or upon a JotForm submission, then retrieves pending product requests from Google Sheets.

**Nodes Involved:**  
- Schedule Trigger  
- JotForm Trigger  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get row(s) in sheet  
- Edit Fields

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts workflow every specified interval (hourly).  
  - Configuration: Interval set to every 1 hour.  
  - Input: None (time-based trigger).  
  - Output: Triggers downstream nodes.  
  - Failures: Trigger misconfiguration or time zone issues.

- **JotForm Trigger**  
  - Type: JotForm Trigger  
  - Role: Starts workflow upon submission of a specified JotForm form.  
  - Configuration: Linked to form ID "252856264643060".  
  - Input: Webhook from JotForm submission.  
  - Output: Product data from form submission.  
  - Failures: Webhook misconfiguration, form ID changes, network issues.  
  - Sticky Note: "Start execution whenever a jotform is submitted. Signup here on JotForm: https://www.jotform.com/?partner=zainurrehman"

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual start for testing or on-demand execution.  
  - Configuration: None.  
  - Input: Manual user action.  
  - Output: Triggers Google Sheets query.  

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Role: Fetches rows with Status = "Pending" from a specified sheet and document.  
  - Configuration: Filter on "Status" column equals "Pending". Sheet and document IDs are dynamically loaded.  
  - Input: Trigger node.  
  - Output: Rows of pending products.  
  - Failures: Authentication errors, missing sheet, API quota limits, empty results.

- **Edit Fields**  
  - Type: Set  
  - Role: Normalizes and prepares product data fields for downstream use.  
  - Configuration: Assigns product name, description, image URLs, requirements, references, and row number into structured JSON properties.  
  - Input: Rows from Google Sheets or JotForm data.  
  - Output: Prepared product data for AI prompt generation.

---

#### 1.2 Data Preparation (Images Downloading for AI)

**Overview:**  
Downloads the product and reference images from URLs to prepare them as inline data for the AI image generation request.

**Nodes Involved:**  
- Message a model  
- Product (HTTP Request)  
- Reference Image 1 (HTTP Request)  
- Reference Image 2 (HTTP Request)  
- Merge

**Node Details:**

- **Message a model**  
  - Type: Google Gemini (Langchain integration)  
  - Role: Generates a prompt for the AI image model based on product details.  
  - Configuration: Uses Gemini model "gemini-2.0-flash-lite"; message content dynamically builds prompt referencing product fields.  
  - Input: Edited fields with product info.  
  - Output: AI-generated prompt text for image creation.  
  - Failures: Authentication errors, model timeout, expression failures in prompt construction.

- **Product**  
  - Type: HTTP Request  
  - Role: Fetches the main product image as binary data from the product image URL.  
  - Configuration: URL dynamically set from "Edit Fields" node's "Product Image".  
  - Input: Triggered after prompt generation.  
  - Output: Binary data of the product image.  
  - Failures: HTTP errors, invalid URLs, timeouts.

- **Reference Image 1**  
  - Type: HTTP Request  
  - Role: Fetches first reference image for AI input.  
  - Configuration: URL from "Reference Image 1" field.  
  - Input: Same as Product node.  
  - Output: Binary data of reference image 1.  
  - Failures: Same as Product node.

- **Reference Image 2**  
  - Type: HTTP Request  
  - Role: Fetches second reference image similarly.  
  - Configuration: URL from "Reference Image 2" field.  
  - Input: Same as above.  
  - Output: Binary data of reference image 2.  
  - Failures: Same as above.

- **Merge**  
  - Type: Merge  
  - Role: Combines the three image HTTP requests into one output for downstream processing.  
  - Configuration: Number of inputs set to 3 (product + 2 references).  
  - Input: Product, Reference Image 1, Reference Image 2 nodes.  
  - Output: Combined data array with all images.  
  - Failures: Mismatched input counts, missing images.

- Sticky Note: "Images Downloader - For AI"

---

#### 1.3 AI Prompt and Image Generation

**Overview:**  
Sends the generated prompt and the downloaded reference images to the Gemini Nano Banana model to generate a new AI product photo.

**Nodes Involved:**  
- Aggregate  
- Gemini Nano Banana  
- Get Image Contents  
- Convert to File

**Node Details:**

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates all image data items into an array to send in a single request.  
  - Configuration: Aggregate all item data together.  
  - Input: Extracted merged images.  
  - Output: Combined JSON array of image data.  
  - Failures: Empty inputs, aggregation errors.

- **Gemini Nano Banana**  
  - Type: HTTP Request  
  - Role: Calls Google’s Gemini 2.5 Flash Image generation API with prompt and inline image data.  
  - Configuration: POST request to "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent" with JSON body including prompt text and three inline images in base64. Uses OAuth2 credentials for Google Palm API.  
  - Input: Aggregated prompt and images.  
  - Output: Response containing generated text and image data.  
  - Failures: Authentication (OAuth2) errors, API quota exceeded, malformed request, timeout.

- **Get Image Contents**  
  - Type: Set  
  - Role: Extracts the generated image data and MIME type from Gemini’s response.  
  - Configuration: Finds the first inlineData part in response JSON and assigns the base64 image data and mime type to variables.  
  - Input: Gemini Nano Banana node output.  
  - Output: Clean JSON with image data and mimeType.  
  - Failures: Missing or malformed response data, parsing errors.

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts the base64 image data string into a binary file object to be uploaded.  
  - Configuration: Uses mimeType extracted before to set file type. Source property is the "image" string.  
  - Input: Extracted image content JSON.  
  - Output: Binary file data for upload.  
  - Failures: Conversion errors, invalid base64 data.

- Sticky Note: "Photograph Generation"

---

#### 1.4 Image Storage and Sheet Update

**Overview:**  
Uploads the generated image file to Google Drive, then updates or appends the product row in Google Sheets with the generated image link and status.

**Nodes Involved:**  
- Upload to Drive  
- If  
- Update row in sheet  
- Add row in sheet

**Node Details:**

- **Upload to Drive**  
  - Type: Google Drive  
  - Role: Uploads the generated image binary file to Google Drive folder.  
  - Configuration: Filename constructed from product name and image extension derived from mimeType. Drive ID set to "My Drive", folder set to root.  
  - Input: Binary file from Convert to File.  
  - Output: Google Drive file metadata including webViewLink.  
  - Failures: Authentication errors, insufficient permissions, quota exceeded.

- **If**  
  - Type: If  
  - Role: Checks if the product row exists in Google Sheets to decide whether to update or add a new row.  
  - Configuration: Checks that the "Get row(s) in sheet" node returned data (not empty).  
  - Input: Output from Upload to Drive and prior sheet fetch.  
  - Output: Routes to "Update row in sheet" if exists, else to "Add row in sheet".  
  - Failures: Expression errors, empty input.

- **Update row in sheet**  
  - Type: Google Sheets  
  - Role: Updates existing product row with status "Completed" and generated image URL.  
  - Configuration: Matches rows by row_number, updates Status and Generated Image columns. Sheet and document IDs dynamically loaded.  
  - Input: Conditional path from If node when row exists.  
  - Output: Confirmation of update.  
  - Failures: Row mismatch, authentication, API limits.

- **Add row in sheet**  
  - Type: Google Sheets  
  - Role: Adds a new product row with all details and generated image URL.  
  - Configuration: Appends a new row with all product data fields and status "Completed".  
  - Input: Conditional path from If node when row does not exist.  
  - Output: Confirmation of append.  
  - Failures: Authentication, quota, malformed data.

- Sticky Note: "Logging everything in google sheets"

---

### 3. Summary Table

| Node Name                 | Node Type                   | Functional Role                                            | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                          |
|---------------------------|-----------------------------|------------------------------------------------------------|---------------------------------|-------------------------------|----------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger             | Periodically triggers workflow to check for pending tasks | None                            | Get row(s) in sheet            |                                                                      |
| JotForm Trigger           | JotForm Trigger              | Triggers workflow on JotForm submission                    | None                            | Edit Fields                   | Start execution whenever a jotform is submitted. Signup here on JotForm: https://www.jotform.com/?partner=zainurrehman |
| When clicking ‘Execute workflow’ | Manual Trigger              | Manual start for on-demand execution                        | None                            | Get row(s) in sheet            |                                                                      |
| Get row(s) in sheet       | Google Sheets                | Fetches pending rows with Status = "Pending"               | Schedule Trigger, Manual Trigger | Edit Fields                   |                                                                      |
| Edit Fields               | Set                         | Normalizes product data fields                              | Get row(s) in sheet, JotForm Trigger | Message a model              |                                                                      |
| Message a model           | Google Gemini (Langchain)    | Generates AI prompt text for image generation              | Edit Fields                    | Product, Reference Image 1, Reference Image 2 |                                                                      |
| Product                  | HTTP Request                 | Downloads main product image                                | Message a model                | Merge                        |                                                                      |
| Reference Image 1         | HTTP Request                 | Downloads first reference image                            | Message a model                | Merge                        |                                                                      |
| Reference Image 2         | HTTP Request                 | Downloads second reference image                           | Message a model                | Merge                        |                                                                      |
| Merge                    | Merge                        | Combines three image downloads                              | Product, Reference Image 1, Reference Image 2 | Extract from File          | Images Downloader - For AI                                           |
| Extract from File         | Extract from File            | Converts merged binary inputs to JSON                       | Merge                         | Aggregate                    |                                                                      |
| Aggregate                | Aggregate                   | Aggregates image data into an array                         | Extract from File              | Gemini Nano Banana           |                                                                      |
| Gemini Nano Banana        | HTTP Request                 | Calls AI model to generate product photo                    | Aggregate                     | Get Image Contents           | Photograph Generation                                                |
| Get Image Contents        | Set                         | Extracts base64 image and mime type from AI response       | Gemini Nano Banana             | Convert to File              |                                                                      |
| Convert to File           | Convert to File              | Converts base64 string to binary file                       | Get Image Contents             | Upload to Drive              |                                                                      |
| Upload to Drive           | Google Drive                 | Uploads generated image file                                | Convert to File                | If                          |                                                                      |
| If                       | If                          | Checks if product row exists to decide update vs append    | Upload to Drive, Get row(s) in sheet | Update row in sheet, Add row in sheet | Logging everything in google sheets                                 |
| Update row in sheet       | Google Sheets                | Updates existing product row                                | If (true branch)              |                               |                                                                      |
| Add row in sheet          | Google Sheets                | Appends new product row                                    | If (false branch)             |                               |                                                                      |
| Sticky Note               | Sticky Note                  | Visual annotations                                         | None                          | None                        | See section 2 for content                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**
   - Add a **Schedule Trigger** node.
     - Set interval to every 1 hour.
   - Add a **JotForm Trigger** node.
     - Configure with form ID "252856264643060".
     - Ensure webhook is active.
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’".

2. **Fetch Pending Rows:**
   - Add a **Google Sheets** node named "Get row(s) in sheet".
     - Operation: Read rows.
     - Filter rows where "Status" equals "Pending".
     - Configure with correct Google Sheets credentials.
     - Set Sheet Name and Document ID dynamically or fixed depending on your environment.

3. **Data Preparation:**
   - Add a **Set** node named "Edit Fields".
     - Map fields: Product Name, Product Description, Product Image, Requirement, Reference Image 1, Reference Image 2, row_number.
     - Use expressions to assign values from previous node JSON.

4. **Generate AI Prompt:**
   - Add a **Google Gemini (Langchain)** node named "Message a model".
     - Model ID: "models/gemini-2.0-flash-lite".
     - Message content template:
       ```
       You are a professional photographer who does Product photography. Below are the details of a product and what type of photographic image he needs.

       Product: {{$json['Product Name']}}, {{$json['Product Description']}}

       Product image: {{$json['Product Image']}}

       Requirement: {{$json.Requirement}}

       Note: Just provide the prompt directly, no extra text.
       ```
     - Authenticate with Google Gemini API credentials.

5. **Download Images:**
   - Add three **HTTP Request** nodes: "Product", "Reference Image 1", "Reference Image 2".
     - Each configured with the corresponding URL fields from "Edit Fields".
     - Method: GET.
     - Response format: Binary.

6. **Merge Images:**
   - Add a **Merge** node.
     - Number of inputs: 3.
     - Connect "Product", "Reference Image 1", and "Reference Image 2" to it.

7. **Extract Binary Data:**
   - Add an **Extract from File** node.
     - Operation: binaryToProperty.
     - Input from "Merge".

8. **Aggregate Image Data:**
   - Add an **Aggregate** node.
     - Aggregate all item data for single input.

9. **Generate AI Image:**
   - Add an **HTTP Request** node named "Gemini Nano Banana".
     - POST to "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent".
     - Body (JSON) includes:
       - Prompt text from "Message a model" node.
       - Inline image data from aggregated images.
       - Generation config with responseModalities: ["TEXT", "IMAGE"].
     - Authentication: Google Palm API OAuth2 credentials.

10. **Extract Generated Image:**
    - Add a **Set** node named "Get Image Contents".
      - Extract base64 image data and mimeType from Gemini response JSON.
      - Assign to "image" and "mimeType" fields.

11. **Convert to File:**
    - Add a **Convert to File** node.
      - Source property: "image".
      - MIME type: from "mimeType".

12. **Upload to Google Drive:**
    - Add a **Google Drive** node named "Upload to Drive".
      - Filename: product name + extension from mimeType (e.g., png).
      - Drive: "My Drive".
      - Folder: Root or specified folder.
      - Use Google Drive OAuth2 credentials.

13. **Conditional Update or Append:**
    - Add an **If** node.
      - Condition: Check if "Get row(s) in sheet" returned data (row exists).
    - Connect "true" branch to **Google Sheets** node "Update row in sheet".
      - Update existing row by row_number.
      - Update "Status" to "Completed" and "Generated Image" with Drive link.
    - Connect "false" branch to **Google Sheets** node "Add row in sheet".
      - Append new row with all product info and generated image link.

14. **Connect triggers:**
    - Connect Schedule Trigger, Manual Trigger, and JotForm Trigger all to "Get row(s) in sheet" and follow the flow.

15. **Credential Setup:**
    - Configure Google Sheets, Google Drive, Google Gemini, and JotForm credentials with proper OAuth2 or API keys.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Start execution whenever a jotform is submitted. Signup here on JotForm                                   | https://www.jotform.com/?partner=zainurrehman                                                          |
| Images Downloader - For AI                                                                                  | Block covering the image download HTTP requests and merge operations                                   |
| Photograph Generation                                                                                       | Refers to the Gemini Nano Banana image generation and conversion nodes                                 |
| Logging everything in google sheets                                                                         | Refers to update or append operations on Google Sheets to keep track of workflow progress and results |

---

This documentation covers the full logic and configuration of the "Generate AI Product Photos using Gemini Nano Banana with JotForm & Google Sheets" workflow, enabling advanced users or automation agents to understand, reproduce, or modify it confidently.  
All nodes and their connections are accounted for without raw JSON dumps.