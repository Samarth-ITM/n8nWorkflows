Generate AI Images with APImage and Upload to Google Drive

https://n8nworkflows.xyz/workflows/generate-ai-images-with-apimage-and-upload-to-google-drive-6398


# Generate AI Images with APImage and Upload to Google Drive

---

### 1. Workflow Overview

This workflow automates the generation of AI images using the APImage API and subsequently uploads the resulting images to Google Drive. It is designed for users who want to quickly create AI-generated images via a web form and store them securely in their Google Drive storage. The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Collects user input via a form to specify image generation parameters.
- **1.2 AI Image Generation:** Sends a request to the APImage API to generate an image based on user input.
- **1.3 Image Handling and Upload:** Downloads the generated image file from APImage and uploads it to Google Drive.

Each block is connected in sequence to provide a streamlined process from user input to final image storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block presents a form interface to the user where they can describe the desired image, select dimensions, and choose an AI model. It acts as the workflow’s entry point.

- **Nodes Involved:**  
  - Generate Image

- **Node Details:**  

  - **Generate Image**  
    - Type: Form Trigger  
    - Role: Captures user input via a web form.  
    - Configuration:  
      - Form titled "APImage AI Image Generator".  
      - Fields:  
        - "Describe the image you want" (text input, required)  
        - "Dimensions" (dropdown: Square, Landscape, Portrait, required)  
        - "AI Model" (dropdown: Basic, Premium, required)  
      - Webhook ID configured for external HTTP POST requests.  
    - Expressions/Variables:  
      - Outputs user-submitted JSON with keys:  
        - `Describe the image you want`  
        - `Dimensions`  
        - `AI Model`  
    - Input/Output:  
      - Input: HTTP POST from form submission  
      - Output: JSON containing form data  
    - Potential Failures:  
      - Invalid or missing required fields (form validation).  
      - Webhook connectivity issues.  
    - Version: 2.2

---

#### 2.2 AI Image Generation

- **Overview:**  
  This block sends a POST request to the APImage API to generate an AI image based on the user’s inputs collected in the previous block.

- **Nodes Involved:**  
  - APImage API

- **Node Details:**  

  - **APImage API**  
    - Type: HTTP Request  
    - Role: Sends image generation request to APImage API endpoint.  
    - Configuration:  
      - URL: https://apimage.org/api/ai-image-generate  
      - Method: POST  
      - Body Type: JSON  
      - Body Parameters:  
        - `prompt`: Text describing the image, mapped from form input `Describe the image you want`  
        - `dimensions`: Chosen dimension value (Square, Landscape, Portrait)  
        - `model`: AI model selection (Basic or Premium)  
      - Headers: Authorization with Bearer token placeholder (`Bearer YOUR_API_KEY`) — user must replace with actual API key.  
    - Expressions/Variables:  
      - Uses expressions to map form data into JSON body fields.  
    - Input/Output:  
      - Input: JSON from "Generate Image" node.  
      - Output: JSON containing URL(s) of generated image(s).  
    - Potential Failures:  
      - Authentication errors (invalid/missing API key).  
      - API rate limiting or quota exceeded.  
      - Network timeouts or 5xx server errors from APImage.  
      - Malformed request due to expression errors.  
    - Version: 4.2  
    - Sticky Note Reference: Instructions to replace API key and links to API documentation.

---

#### 2.3 Image Handling and Upload

- **Overview:**  
  This block downloads the image file from the URL returned by the APImage API and uploads it to the user’s Google Drive.

- **Nodes Involved:**  
  - Download Image  
  - Upload file

- **Node Details:**  

  - **Download Image**  
    - Type: HTTP Request  
    - Role: Downloads the generated image file from the provided URL.  
    - Configuration:  
      - URL set dynamically from the first image URL in the JSON response (`{{$json.images[0]}}`).  
      - Response format: File, saved to output property `generated_image`.  
    - Input/Output:  
      - Input: JSON output from "APImage API" node.  
      - Output: Binary file data under `generated_image`.  
    - Potential Failures:  
      - Invalid or missing image URL.  
      - Network errors or timeouts.  
      - Large file sizes causing timeouts (see debug sticky note).  
    - Version: 4.2

  - **Upload file**  
    - Type: Google Drive (App Node)  
    - Role: Uploads the downloaded image file to Google Drive.  
    - Configuration:  
      - File name: `generated_image` (uses binary data from previous node).  
      - Drive: "My Drive" (default Google Drive root).  
      - Folder ID: Root folder (`/ (Root folder)`).  
      - Input data field name: `generated_image` (binary data field).  
      - Requires Google Drive OAuth2 credentials configured in n8n.  
    - Input/Output:  
      - Input: Binary image data from "Download Image".  
      - Output: Metadata of uploaded file in Google Drive.  
    - Potential Failures:  
      - Authentication errors with Google Drive credentials.  
      - Insufficient permissions or quota in Google Drive.  
      - File size or type restrictions.  
    - Version: 3  
    - Sticky Note Reference: Notes about replacing this node to upload to other services like Dropbox or Notion.

---

### 3. Summary Table

| Node Name      | Node Type           | Functional Role                     | Input Node(s)        | Output Node(s)    | Sticky Note                                                                                                                                            |
|----------------|---------------------|-----------------------------------|----------------------|-------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Generate Image | Form Trigger        | Collects user input via web form  | (Webhook trigger)    | APImage API       | "### 🧩 Choose Your Input" - Guidance on form params and API docs link.                                                                                |
| APImage API    | HTTP Request        | Sends image generation request    | Generate Image       | Download Image    | "## ✨ How To Get Started" - API key setup instructions and dashboard link.                                                                            |
| Download Image | HTTP Request        | Downloads generated image file    | APImage API          | Upload file       |                                                                                                                                                        |
| Upload file    | Google Drive        | Uploads image to Google Drive     | Download Image       |                   | "### 📤 Choose Your Destination" - Alternative upload service options & link to Google Drive credential setup.                                         |
| Sticky Note    | Sticky Note         | Instruction to get started        |                      |                   | "## ✨ How To Get Started" (covers APImage API node).                                                                                                |
| Sticky Note2   | Sticky Note         | Notes on upload destination       |                      |                   | "### 📤 Choose Your Destination" (covers Upload file node).                                                                                           |
| Sticky Note3   | Sticky Note         | Notes on input configuration      |                      |                   | "### 🧩 Choose Your Input" (covers Generate Image node).                                                                                            |
| Sticky Note1   | Sticky Note         | Debug info for timeout errors     |                      |                   | "### 🐞 Debug Support" - Timeout handling and support contact info.                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `Generate Image`  
   - Type: Form Trigger  
   - Configure a new webhook with a unique ID.  
   - Form Title: "APImage AI Image Generator"  
   - Add Form Fields:  
     - Text field labeled "Describe the image you want" (Required)  
     - Dropdown field labeled "Dimensions" with options: Square, Landscape, Portrait (Required)  
     - Dropdown field labeled "AI Model" with options: Basic, Premium (Required)  
   - Save the node.

2. **Add HTTP Request Node to Call APImage API**  
   - Name: `APImage API`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: https://apimage.org/api/ai-image-generate  
   - Body Content Type: JSON  
   - Set JSON Body using expressions:  
     ```json
     {
       "prompt": "{{ $json['Describe the image you want'] }}",
       "dimensions": "{{ $json['Dimensions'] }}",
       "model": "{{ $json['AI Model'] }}"
     }
     ```  
   - Add HTTP Header:  
     - Key: `Authorization`  
     - Value: `Bearer YOUR_API_KEY` (replace `YOUR_API_KEY` with your actual APImage API key, including the "Bearer" prefix)  
   - Connect "Generate Image" node’s output to this node’s input.

3. **Add HTTP Request Node to Download Image**  
   - Name: `Download Image`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: Use expression to get the first image URL from previous node: `{{$json["images"][0]}}`  
   - Under "Options", configure response format: File  
   - Set output property name to `generated_image` to hold the binary file data.  
   - Connect "APImage API" node’s output to this node.

4. **Add Google Drive Node to Upload File**  
   - Name: `Upload file`  
   - Type: Google Drive  
   - Authentication: Configure OAuth2 Google Drive credentials in n8n beforehand.  
   - Drive: Select "My Drive" (default)  
   - Folder ID: Use root folder (`/ (Root folder)`) or specify another folder if desired.  
   - File Name: Use `generated_image` (binary data from "Download Image")  
   - Input Data Field Name: `generated_image`  
   - Connect "Download Image" node’s output to this node.

5. **(Optional) Add Sticky Notes for Documentation**  
   - Add sticky notes near relevant nodes with the content as detailed in Section 3 for user guidance and debugging tips.

6. **Finalizing and Testing**  
   - Ensure all nodes are properly connected in sequence:  
     Generate Image → APImage API → Download Image → Upload file  
   - Replace all placeholder API keys with valid credentials.  
   - Test the workflow by submitting the form and verifying the image is generated and appears in Google Drive.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                        | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| You can find your API Key inside the Dashboard of your APImage account.                                                                                           | https://apimage.org/dashboard                                                                              |
| The APImage API documentation details parameters and usage limits.                                                                                                | https://apimage.org/docs                                                                                   |
| Google Drive credentials setup instructions for n8n.                                                                                                             | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.googleDrive |
| For 504 Gateway Timeout errors, increase the workflow execution timeout settings, especially on cloud-hosted n8n instances.                                      | See Sticky Note1 content                                                                                    |
| APImage Support Email for API or service issues: ask@support@apimage.org                                                                                           | mailto:ask@support@apimage.org                                                                              |
| The "Upload file" node can be replaced with other destination nodes like Dropbox, Notion, WordPress, or Shopify, but ensure file naming consistency.              | See Sticky Note2 content                                                                                    |
| The "Generate Image" input form can be replaced with other data sources as long as the required parameters (prompt, dimensions, model) are provided correctly.    | See Sticky Note3 content                                                                                    |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---