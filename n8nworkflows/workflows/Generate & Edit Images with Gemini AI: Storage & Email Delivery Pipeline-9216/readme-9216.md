Generate & Edit Images with Gemini AI: Storage & Email Delivery Pipeline

https://n8nworkflows.xyz/workflows/generate---edit-images-with-gemini-ai--storage---email-delivery-pipeline-9216


# Generate & Edit Images with Gemini AI: Storage & Email Delivery Pipeline

### 1. Workflow Overview

This workflow, titled **"Generate & Edit Images with Gemini AI: Storage & Email Delivery Pipeline"**, is designed to receive user input (including text prompts and optionally image files), generate images using the "Nano Banana" AI image generation API, process and store these images on Google Drive, and finally deliver the results via email. It supports two main input scenarios: with or without an uploaded image file, adapting the AI image generation prompt accordingly.

The workflow logic is structured into the following functional blocks:

- **1.1 Input Reception and Branching**: Receives incoming requests via a webhook and determines if an image file was uploaded to choose the appropriate AI prompt path.
- **1.2 Image Extraction and Formatting**: Extracts and formats uploaded images for AI processing when applicable.
- **1.3 AI Image Generation**: Sends requests to the Nano Banana AI API to generate images based on text prompts or combined text + image prompts.
- **1.4 Post-Processing and File Conversion**: Converts AI-generated responses into files suitable for storage.
- **1.5 Cloud Storage and Sharing**: Uploads and shares generated image files on Google Drive.
- **1.6 Email Delivery and Response**: Emails the generated image files to recipients and responds to the original webhook request.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Branching

- **Overview:**  
  This block receives the initial data from the user via a webhook, then uses a conditional check to branch processing depending on whether an image file was uploaded.

- **Nodes Involved:**  
  - Webhook  
  - If Image File Was Uploaded

- **Node Details:**

  - **Webhook**  
    - Type: Webhook (HTTP entry point)  
    - Configuration: Default webhook with no extra parameters; serves as the entry point for user data submission.  
    - Key variables: Request payload expected to contain text prompt and optionally an image file.  
    - Inputs: External HTTP request  
    - Outputs: Passes data to the "If Image File Was Uploaded" node  
    - Potential failures: Network issues, malformed input payloads, webhook misconfiguration.

  - **If Image File Was Uploaded**  
    - Type: If (conditional branching)  
    - Configuration: Checks if the input data contains an uploaded image file; likely based on presence of a file property or field.  
    - Inputs: Data from the Webhook node  
    - Outputs: Two branches:  
      - True branch: Passes to "Extract from File" (image file present)  
      - False branch: Passes to "Nano 🍌: Prompt Only" (no image file)  
    - Potential failures: Incorrect conditional expression, missing expected file fields.

---

#### 1.2 Image Extraction and Formatting

- **Overview:**  
  Extracts the uploaded file from the input data and formats it appropriately to be used as an AI prompt.

- **Nodes Involved:**  
  - Extract from File  
  - Code

- **Node Details:**

  - **Extract from File**  
    - Type: Extract From File  
    - Configuration: Extracts raw data or metadata from the uploaded image file to prepare it for further processing.  
    - Inputs: Image file from "If Image File Was Uploaded" (true branch)  
    - Outputs: Data to "Code" node  
    - Potential failures: File corruption, unsupported file types.

  - **Code**  
    - Type: Code (JavaScript)  
    - Configuration: Custom code to format the extracted image data into the correct format required by the AI API.  
    - Notes: Contains a note “format mage for AI” (likely a typo for “format image for AI”).  
    - Inputs: Extracted file data from "Extract from File"  
    - Outputs: Formatted data to "Nano 🍌" node  
    - Potential failures: Coding errors, data formatting issues.

---

#### 1.3 AI Image Generation

- **Overview:**  
  Sends HTTP requests to the Nano Banana AI API to generate images either from text-only prompts or a combination of text and image prompts.

- **Nodes Involved:**  
  - Nano 🍌  
  - Edit Fields1  
  - Convert to File  
  - Upload file  
  - Share file  
  - Edit Fields  
  - Send email  
  - Respond to Webhook

- **Node Details:**

  - **Nano 🍌**  
    - Type: HTTP Request  
    - Configuration: Sends formatted image+text prompt data to the Nano Banana AI endpoint for image generation.  
    - Inputs: Output from "Code" node (formatted image and prompt)  
    - Outputs: Raw AI response to "Edit Fields1"  
    - Potential failures: API authentication, rate limits, network timeouts.

  - **Nano 🍌: Prompt Only**  
    - Type: HTTP Request  
    - Configuration: Sends text-only prompt to Nano Banana AI for image generation when no image file is uploaded.  
    - Inputs: False branch of "If Image File Was Uploaded"  
    - Outputs: Raw AI response to "Edit Fields2"  
    - Potential failures: Same as "Nano 🍌".

  - **Edit Fields1**  
    - Type: Set  
    - Configuration: Adjusts or sets fields on the AI response data to prepare for file conversion.  
    - Inputs: From "Nano 🍌"  
    - Outputs: To "Convert to File"

  - **Edit Fields2**  
    - Type: Set  
    - Configuration: Similar to Edit Fields1 but handles data from "Nano 🍌: Prompt Only"  
    - Inputs: From "Nano 🍌: Prompt Only"  
    - Outputs: To "Convert to File"

  - **Convert to File**  
    - Type: Convert To File  
    - Configuration: Converts the AI response data (likely base64 or URL) into a file format acceptable for Google Drive upload.  
    - Inputs: From either Edit Fields1 or Edit Fields2  
    - Outputs: To "Upload file"

  - **Upload file**  
    - Type: Google Drive  
    - Configuration: Uploads the generated image file to Google Drive.  
    - Inputs: From "Convert to File"  
    - Outputs: To "Share file"  
    - Credential: Google Drive OAuth2 credentials required.  
    - Potential failures: Auth errors, quota limits, network errors.

  - **Share file**  
    - Type: Google Drive  
    - Configuration: Adjusts sharing permissions on the uploaded file to make it accessible (e.g., public or link share).  
    - Inputs: From "Upload file"  
    - Outputs: To "Edit Fields"

  - **Edit Fields**  
    - Type: Set  
    - Configuration: Modifies the data to prepare for email sending, likely including file links and metadata.  
    - Inputs: From "Share file"  
    - Outputs: To "Send email"

  - **Send email**  
    - Type: Email Send  
    - Configuration: Sends an email containing the generated image file link or attachment to the user.  
    - Inputs: From "Edit Fields"  
    - Outputs: To "Respond to Webhook"  
    - Credential: Configured SMTP or email service credentials required.  
    - Potential failures: SMTP auth errors, email formatting issues.

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Configuration: Sends the final success response back to the original webhook caller, confirming completion and providing result data.  
    - Inputs: From "Send email"

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                   | Input Node(s)            | Output Node(s)           | Sticky Note                              |
|-------------------------|--------------------|---------------------------------|--------------------------|--------------------------|-----------------------------------------|
| Webhook                 | Webhook            | Receives user input              | External HTTP request    | If Image File Was Uploaded| Data from user. This can be replaced with a form trigger |
| If Image File Was Uploaded | If                 | Branches based on image presence | Webhook                  | Extract from File, Nano 🍌: Prompt Only |                                         |
| Extract from File       | Extract From File   | Extracts uploaded image file data| If Image File Was Uploaded| Code                     |                                         |
| Code                    | Code               | Formats image for AI             | Extract from File        | Nano 🍌                  | format mage for AI                      |
| Nano 🍌                 | HTTP Request       | Sends image+text prompt to AI   | Code                     | Edit Fields1             |                                         |
| Nano 🍌: Prompt Only     | HTTP Request       | Sends text-only prompt to AI    | If Image File Was Uploaded| Edit Fields2             |                                         |
| Edit Fields1            | Set                | Prepares AI response data       | Nano 🍌                   | Convert to File          |                                         |
| Edit Fields2            | Set                | Prepares AI response data       | Nano 🍌: Prompt Only      | Convert to File          |                                         |
| Convert to File         | Convert To File    | Converts AI response to file    | Edit Fields1, Edit Fields2| Upload file              |                                         |
| Upload file             | Google Drive       | Uploads file to Google Drive    | Convert to File           | Share file               |                                         |
| Share file              | Google Drive       | Shares uploaded file            | Upload file               | Edit Fields              |                                         |
| Edit Fields             | Set                | Prepares data for email sending | Share file                | Send email               |                                         |
| Send email              | Email Send         | Sends generated image via email | Edit Fields               | Respond to Webhook       |                                         |
| Respond to Webhook      | Respond to Webhook | Sends final response to caller  | Send email                |                          |                                         |
| Sticky Note1            | Sticky Note        |                                 |                          |                          |                                         |
| Sticky Note2            | Sticky Note        |                                 |                          |                          |                                         |
| Sticky Note3            | Sticky Note        |                                 |                          |                          |                                         |
| Sticky Note5            | Sticky Note        |                                 |                          |                          |                                         |
| Sticky Note10           | Sticky Note        |                                 |                          |                          |                                         |
| Sticky Note11           | Sticky Note        |                                 |                          |                          |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Name: `Webhook`  
   - Purpose: Entry point for receiving user data, including text prompts and optional image files.  
   - Parameters: Use default, no special config required.  
   - Save webhook URL for external calls.

2. **Add an If node**  
   - Name: `If Image File Was Uploaded`  
   - Connect input from `Webhook`.  
   - Condition: Check if the incoming data contains an uploaded image file (e.g., check if `items[0].json.file` exists or equivalent).  
   - True branch: Image file present.  
   - False branch: No image file.

3. **Create Extract From File node**  
   - Name: `Extract from File`  
   - Connect input from True branch of `If Image File Was Uploaded`.  
   - Configure to extract the image file data or metadata needed for AI API.

4. **Create Code node**  
   - Name: `Code`  
   - Connect input from `Extract from File`.  
   - Add JavaScript code to format the extracted image data into the Nano Banana AI API expected format (e.g., base64 encoding or multipart upload format).  
   - Output should be the formatted data ready for HTTP request.

5. **Create HTTP Request node**  
   - Name: `Nano 🍌`  
   - Connect input from `Code`.  
   - Configure to POST the formatted data to Nano Banana AI image generation API for combined image+text prompt.  
   - Set authentication and headers as needed.

6. **Create HTTP Request node for text-only prompts**  
   - Name: `Nano 🍌: Prompt Only`  
   - Connect input from False branch of `If Image File Was Uploaded`.  
   - Configure to POST text prompt only to the same Nano Banana AI API endpoint.

7. **Add two Set nodes**  
   - Names: `Edit Fields1` and `Edit Fields2`  
   - Connect `Edit Fields1` input from `Nano 🍌` output.  
   - Connect `Edit Fields2` input from `Nano 🍌: Prompt Only` output.  
   - Configure each to set or modify fields required for file conversion (e.g., extract image URL or base64 data).

8. **Add a Convert To File node**  
   - Name: `Convert to File`  
   - Connect inputs from both `Edit Fields1` and `Edit Fields2` (merge via parallel connections).  
   - Configure to convert the AI response data into file format (image/png or jpg).

9. **Add Google Drive Upload node**  
   - Name: `Upload file`  
   - Connect input from `Convert to File`.  
   - Configure OAuth2 credentials for Google Drive.  
   - Set target folder or drive location.

10. **Add Google Drive Share node**  
    - Name: `Share file`  
    - Connect input from `Upload file`.  
    - Configure to adjust sharing permissions (e.g., anyone with the link can view).

11. **Add a Set node**  
    - Name: `Edit Fields`  
    - Connect input from `Share file`.  
    - Configure to prepare data for email sending, including file links and recipient data.

12. **Add Email Send node**  
    - Name: `Send email`  
    - Connect input from `Edit Fields`.  
    - Configure SMTP or email service credentials.  
    - Setup email template with attachment or link to the generated image.

13. **Add Respond to Webhook node**  
    - Name: `Respond to Webhook`  
    - Connect input from `Send email`.  
    - Configure to send success response and any relevant data back to the original caller.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                            |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------|
| The webhook node can be replaced with other triggers such as form submissions or API calls.         | Workflow Overview / Webhook node           |
| Google Drive OAuth2 credentials must be configured in n8n prior to uploading and sharing files.     | Google Drive nodes                         |
| SMTP credentials are required to send emails via the "Send email" node.                             | Email Send node                            |
| The Nano Banana AI API endpoint and authentication details are required and must be set in HTTP nodes.| AI Image Generation nodes                  |
| The workflow supports both text-only and combined text+image prompt inputs for flexible usage.     | Branching logic (If node)                   |

---

**Disclaimer:** This documentation is derived exclusively from the provided n8n workflow export. It adheres strictly to content policies and contains only legal, public data.