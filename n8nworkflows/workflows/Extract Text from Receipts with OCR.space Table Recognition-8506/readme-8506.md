Extract Text from Receipts with OCR.space Table Recognition

https://n8nworkflows.xyz/workflows/extract-text-from-receipts-with-ocr-space-table-recognition-8506


# Extract Text from Receipts with OCR.space Table Recognition

### 1. Workflow Overview

This workflow enables users to upload an image of a receipt and extract its textual content using the OCR.space API with table recognition support. It is designed for scenarios where receipt images (up to 1MB) need to be converted into text, optionally recognizing tabular data within the receipt.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception:** A public form trigger collects the receipt image and a user indication of whether the image contains a table.
- **1.2 Input Normalization:** The user inputs are normalized into structured data, including a boolean flag for table detection.
- **1.3 OCR Processing:** The image and parameters are sent to the OCR.space API for text extraction.
- **1.4 Result Display:** The extracted text is presented back to the user in a styled, easy-to-read monospace card.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Collects the receipt image via a public form and asks whether the image contains a table. This initiates the workflow when a user submits the form.

- **Nodes Involved:**  
  - Trigger • Upload image for OCR

- **Node Details:**  
  - **Trigger • Upload image for OCR**  
    - Type: Form Trigger (Webhook-based)  
    - Role: Entry point; collects user input (image file and radio selection)  
    - Configuration:  
      - Form title: "Receipt OCR"  
      - Fields:  
        - File upload (single file, max 1MB, required)  
        - Radio button: "Is it a table" with options Yes/No (required)  
      - Form description: "Upload an image to extract text via OCR. Max file size: 1 MB"  
    - Input/Output:  
      - Input: HTTP form submission  
      - Output: JSON with file data and table selection  
    - Edge Cases/Potential Failures:  
      - File size exceeding 1MB (client-side restriction, but server may still receive)  
      - Missing required fields  
      - Invalid file formats (not explicitly validated here)  
    - Version: 2.3  

#### 2.2 Input Normalization

- **Overview:**  
Transforms the radio button input ("Yes"/"No") into a boolean flag `isTable`. Also ensures the uploaded file data is correctly referenced for downstream use.

- **Nodes Involved:**  
  - Prepare • Normalize inputs

- **Node Details:**  
  - **Prepare • Normalize inputs**  
    - Type: Set node  
    - Role: Data normalization and preparation for API call  
    - Configuration:  
      - Assign `isTable` as a boolean: `true` if input is "Yes", otherwise `false`  
      - Assign file object from the form field, handling possible naming variations  
      - Includes other fields unchanged  
    - Key Expressions:  
      - `={{ ($json["Is it a table"] ?? $json.Is_it_a_table) === "Yes" }}` to set `isTable`  
      - `={{ $json['File (Max 1MB)'] ?? $json.File__Max_1MB_ }}` to set the file object  
    - Input/Output:  
      - Input: JSON from form trigger  
      - Output: JSON with normalized `isTable` and file object  
    - Edge Cases/Potential Failures:  
      - Field name discrepancies due to different form field naming conventions  
      - Missing or malformed file object  
    - Version: 3.4  

#### 2.3 OCR Processing

- **Overview:**  
Sends the uploaded image and parameters to the OCR.space API to perform OCR with table recognition if requested.

- **Nodes Involved:**  
  - OCR.space • Parse image

- **Node Details:**  
  - **OCR.space • Parse image**  
    - Type: HTTP Request node  
    - Role: Integration with OCR.space API for optical character recognition  
    - Configuration:  
      - URL: `https://api.ocr.space/parse/image`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body parameters include:  
        - `language`: set to Polish (`pol`)  
        - `file`: binary data from the uploaded file  
        - `OCREngine`: set to 2 (OCR engine version)  
        - `isTable`: boolean flag from previous node  
      - Authentication: HTTP Header with API key (credential required)  
    - Key Expressions:  
      - `={{ $json.isTable }}` to pass the table flag dynamically  
    - Input/Output:  
      - Input: JSON with file and `isTable` flag  
      - Output: JSON response from OCR.space with parsed text results  
    - Edge Cases/Potential Failures:  
      - API authentication failure (invalid or missing API key)  
      - Network timeouts or HTTP errors  
      - File size or format rejected by API  
      - OCR errors or empty results  
    - Version: 4.2  
    - Credentials: Requires configured HTTP Header Auth named `ocr.space` with valid API key  

#### 2.4 Result Display

- **Overview:**  
Presents the extracted OCR text to the user in a styled, monospace card format for easy viewing and copying.

- **Nodes Involved:**  
  - Display • Show OCR text

- **Node Details:**  
  - **Display • Show OCR text**  
    - Type: Form node (completion form)  
    - Role: Final user interface for displaying OCR results  
    - Configuration:  
      - Form title: "OCR Result"  
      - Custom CSS styles for a card-like appearance with monospace font, responsive design  
      - Completion message set dynamically to the OCR extracted text from the response path `ParsedResults[0].ParsedText`  
    - Input/Output:  
      - Input: JSON from OCR.space node  
      - Output: HTTP response displaying the form with text  
    - Edge Cases/Potential Failures:  
      - Missing or malformed OCR text in response  
      - Large text causing display issues (no explicit truncation)  
    - Version: 2.3  

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                     |
|-----------------------------|---------------------|-------------------------------|-------------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Trigger • Upload image for OCR | Form Trigger        | Collects user input (image + table flag) | -                             | Prepare • Normalize inputs   | Public form that collects the image (≤1 MB) and asks if the content is a table.                |
| Prepare • Normalize inputs   | Set                 | Normalizes inputs for API call | Trigger • Upload image for OCR | OCR.space • Parse image       | Converts the radio field (“Yes/No”) into a true/false `isTable` flag. Keeps the uploaded file attached. |
| OCR.space • Parse image      | HTTP Request        | Sends image and parameters to OCR.space API | Prepare • Normalize inputs     | Display • Show OCR text       | Sends file + options to OCR.space API. Language = `pol`, Engine = `2`, passes `isTable`. ⚠️ Needs API key (header auth). |
| Display • Show OCR text      | Form                | Displays parsed OCR text to user | OCR.space • Parse image         | -                           | Shows parsed text from `ParsedResults[0].ParsedText` inside a clean, monospace card for easy copy-paste. |
| Sticky Note                 | Sticky Note          | Informational                  | -                             | -                           | 📌 What this flow does: Upload a receipt image → select if it’s a table → OCR.space parses it → result displayed in a styled card. |
| Sticky Note1                | Sticky Note          | Informational                  | -                             | -                           | 🔹 Trigger • Upload image for OCR: Public form that collects the image (≤1 MB) and asks if the content is a table. |
| Sticky Note2                | Sticky Note          | Informational                  | -                             | -                           | 🔹 Prepare • Normalize inputs: Converts the radio field (“Yes/No”) into a true/false `isTable` flag. Keeps the uploaded file attached. |
| Sticky Note3                | Sticky Note          | Informational                  | -                             | -                           | 🔹 OCR.space • Parse image: Sends file + options to OCR.space API. Language = `pol`, Engine = `2`, passes `isTable`. ⚠️ Needs API key (header auth). |
| Sticky Note4                | Sticky Note          | Informational                  | -                             | -                           | 🔹 Display • Show OCR text: Shows parsed text from `ParsedResults[0].ParsedText` inside a clean, monospace card for easy copy-paste. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Node Type: **Form Trigger**  
   - Name: `Trigger • Upload image for OCR`  
   - Configuration:  
     - Form Title: "Receipt OCR"  
     - Form Description: "Upload an image to extract text via OCR. Max file size: 1 MB"  
     - Add Fields:  
       - File upload field: Label "File (Max 1MB)", single file, required  
       - Radio field: Label "Is it a table", options "Yes" and "No", required  
   - Save and note the webhook URL generated for form submission  

2. **Create Set Node**  
   - Node Type: **Set**  
   - Name: `Prepare • Normalize inputs`  
   - Configuration:  
     - Add assignment `isTable` as string with expression:  
       `={{ ($json["Is it a table"] ?? $json.Is_it_a_table) === "Yes" }}`  
     - Add assignment for file object named `"File (Max 1MB)"` as object, expression:  
       `={{ $json['File (Max 1MB)'] ?? $json.File__Max_1MB_ }}`  
     - Enable "Include Other Fields" to keep existing data  
   - Connect output of trigger node to this node  

3. **Create HTTP Request Node**  
   - Node Type: **HTTP Request**  
   - Name: `OCR.space • Parse image`  
   - Configuration:  
     - HTTP Method: POST  
     - URL: `https://api.ocr.space/parse/image`  
     - Content Type: `multipart/form-data`  
     - Body Parameters:  
       - `language`: fixed value `"pol"`  
       - `file`: select "Form Binary Data" type, input data field name matching the file field from previous node (e.g. `File__Max_1MB_`)  
       - `OCREngine`: fixed value `"2"`  
       - `isTable`: expression `={{ $json.isTable }}`  
     - Authentication: HTTP Header Auth  
       - Create credential named `ocr.space` with the API key for OCR.space placed in the header  
   - Connect output of Set node to this node  

4. **Create Form Node for Display**  
   - Node Type: **Form**  
   - Name: `Display • Show OCR text`  
   - Configuration:  
     - Form Title: "OCR Result"  
     - Operation: Completion  
     - Completion Title: "OCR Result"  
     - Completion Message: expression `={{ $json.ParsedResults[0].ParsedText }}`  
     - Custom CSS:  
       ```css
       .card {
         position: relative;
         max-width: 500px;
         margin: 20px auto;
         padding: 20px;
         background: #fefefe !important;
         border-radius: 12px;
         box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
         font-family: 'Courier New', monospace;
         color: #1f2937;
         font-size: 16px;
         line-height: 1.6;
         white-space: pre-wrap;
       }
       .header p { text-align: left; }
       @media (max-width: 768px) {
         .card { margin: 10px !important; padding: 15px !important; font-size: 14px; }
       }
       ```  
   - Connect output of HTTP Request node to this node  

5. **Credentials Setup**  
   - Set up an HTTP Header Auth credential named `ocr.space` with your OCR.space API key as the header value. This is mandatory for the HTTP Request node to authenticate.

6. **Final Connections**  
   - Connect nodes in the following sequence:  
     `Trigger • Upload image for OCR` → `Prepare • Normalize inputs` → `OCR.space • Parse image` → `Display • Show OCR text`

7. **Test Workflow**  
   - Activate the workflow  
   - Access the public form URL from the trigger node  
   - Upload an image file (≤1MB) and specify whether it contains a table  
   - Submit and observe the OCR text result displayed in a styled card  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow supports OCR in Polish language (`language` = `pol`). Change this parameter as needed. | Modify the `language` parameter in the HTTP Request node accordingly for other languages.       |
| OCR.space API requires an API key that must be added via HTTP Header Auth credential in n8n.         | https://ocr.space/ocrapi                                                                          |
| The styled result card uses monospace font for better readability of extracted text and tables.     | Custom CSS is embedded in the Display node for a clean UI experience.                            |
| File size restriction is client-side only; server or OCR.space API may reject larger files.          | Ensure file size is within 1MB to prevent upload or processing errors.                           |
| The `isTable` flag enables OCR.space table recognition engine, enhancing accuracy for receipts with tables. | Set via radio button on the form for end-user control.                                           |

---

**Disclaimer:** The provided text exclusively originates from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.