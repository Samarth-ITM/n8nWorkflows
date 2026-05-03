Generate Twitter/X Captions from Google Drive Images using Cloudinary & GPT-4o-mini

https://n8nworkflows.xyz/workflows/generate-twitter-x-captions-from-google-drive-images-using-cloudinary---gpt-4o-mini-7278


# Generate Twitter/X Captions from Google Drive Images using Cloudinary & GPT-4o-mini

### 1. Workflow Overview

This workflow automates the generation and delivery of engaging Twitter/X captions based on images uploaded to a specific Google Drive folder. It is designed for social media content strategists and marketers seeking to streamline caption creation using AI, leveraging Google Drive, Cloudinary, and Azure OpenAI services.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Automatically detects new or updated image files in a designated Google Drive folder.
- **1.2 File Retrieval and Upload:** Downloads the detected files from Google Drive and uploads them to Cloudinary to obtain publicly accessible URLs.
- **1.3 AI Caption Generation:** Uses Azure OpenAI’s GPT-4o-mini model, integrated via LangChain, to analyze the image URL and generate a professional, multi-part Twitter caption and an HTML email body.
- **1.4 Email Delivery:** Sends the generated caption and email content to a specified recipient via SMTP.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Monitors a specific Google Drive folder for new or updated files, triggering the workflow automatically to begin processing.

- **Nodes Involved:**  
  - Get files from drive (Google Drive Trigger)

- **Node Details:**

  - **Get files from drive**  
    - Type: Google Drive Trigger  
    - Role: Watches a specific Google Drive folder for file updates every minute.  
    - Configuration:  
      - Event: `fileUpdated`  
      - Polling interval: Every minute  
      - Folder to watch: Folder ID `"1GDkSI9txB88oOcu_jr1tXEzkT0_FoGmC"` (named "Instagram")  
    - Input: None (trigger node)  
    - Output: File metadata including file ID, name, and URLs  
    - Credentials: Google OAuth2 with access to the folder  
    - Edge Cases:  
      - Folder access denied or revoked credentials causing trigger failure  
      - High frequency file changes potentially causing duplicate triggers  
      - Network or API rate limiting errors  
    - Sticky Note Reference: Sticky Note5 (explains trigger purpose and output)

#### 2.2 File Retrieval and Upload

- **Overview:**  
Downloads the triggered file from Google Drive and uploads it to Cloudinary to generate a public URL usable by AI services.

- **Nodes Involved:**  
  - Download the drive files (Google Drive)  
  - upload frames to cloudinary (HTTP Request)

- **Node Details:**

  - **Download the drive files**  
    - Type: Google Drive node  
    - Role: Downloads the actual file content using the file ID provided by the trigger node.  
    - Configuration:  
      - Operation: `download`  
      - File ID: Dynamically set from trigger output (`{{$json["id"]}}`)  
    - Input: File metadata from trigger  
    - Output: Binary file data ready for upload  
    - Credentials: Google OAuth2  
    - Edge Cases:  
      - File no longer exists or access revoked  
      - Large file size causing timeouts or memory issues  
    - Sticky Note Reference: Sticky Note4 (explains purpose: get file content)

  - **upload frames to cloudinary**  
    - Type: HTTP Request node  
    - Role: Uploads the downloaded image to Cloudinary to obtain a secure, publicly accessible URL.  
    - Configuration:  
      - URL: Cloudinary upload API endpoint  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body Parameters:  
        - `file`: Binary data from previous node  
        - `upload_preset`: "upload" (Cloudinary preset)  
      - Authentication: HTTP Basic with Cloudinary API key/secret credentials  
      - Retry on Fail: Enabled (max 5 retries)  
    - Input: Binary file from Google Drive download  
    - Output: JSON with `secure_url` for the uploaded image  
    - Credentials: Cloudinary HTTP Basic Auth  
    - Edge Cases:  
      - Authentication failure due to invalid credentials  
      - Upload preset misconfiguration causing rejection  
      - Network failures or API rate limits  
    - Sticky Note Reference: Sticky Note3 (explains purpose: upload and get URLs)

#### 2.3 AI Caption Generation

- **Overview:**  
Processes the Cloudinary image URL with an Azure OpenAI GPT-4o-mini model via LangChain to generate a professional Twitter caption and a styled HTML email body.

- **Nodes Involved:**  
  - Azure OpenAI Chat Model (Azure OpenAI node)  
  - Basic LLM Chain (LangChain Chain LLM node)

- **Node Details:**

  - **Azure OpenAI Chat Model**  
    - Type: LangChain Azure OpenAI LM Chat node  
    - Role: Provides AI model inference using GPT-4o-mini for language generation.  
    - Configuration:  
      - Model: `gpt-4o-mini`  
      - No special options configured  
    - Credentials: Azure OpenAI API key and endpoint  
    - Input: Prompt from Basic LLM Chain node's messages  
    - Output: AI-generated text response  
    - Edge Cases:  
      - API key or endpoint misconfiguration  
      - Model unavailability or throttling  
      - Unexpected response format  
    - Sticky Note Reference: Sticky Note (labeled "AzureOpenAI") describing backup AI service

  - **Basic LLM Chain**  
    - Type: LangChain Chain LLM node  
    - Role: Defines the AI system prompt and input prompt template to instruct the model on the desired output format and style.  
    - Configuration:  
      - Prompt text includes detailed instructions for generating:  
        - Subject line (≤60 chars)  
        - HTML email body with inline CSS  
        - Embedded image referencing `{{$json.secure_url}}`  
        - Twitter style post block with headline, tweets, hashtags, mentions, posting windows  
        - Accessibility and email rendering guidelines  
      - Uses expressions to insert the Cloudinary `secure_url` dynamically.  
    - Input: Receives JSON from Cloudinary upload (including `secure_url`)  
    - Output: Text output with subject and full HTML email content  
    - Edge Cases:  
      - Missing or invalid `secure_url` causing prompt failure  
      - AI generating incomplete or malformed responses not following format  
    - Sticky Note Reference: Sticky Note2 (explains AI caption generation purpose)

#### 2.4 Email Delivery

- **Overview:**  
Sends the generated email content with Twitter captions to a specified recipient using SMTP.

- **Nodes Involved:**  
  - Send email (Email Send node)

- **Node Details:**

  - **Send email**  
    - Type: Email Send node  
    - Role: Sends the AI-generated caption and email body as an HTML email.  
    - Configuration:  
      - Subject: Static text "Twitter Post" (could be enhanced to use generated subject)  
      - HTML content: Dynamic from `{{$json.text}}` (output of Basic LLM Chain)  
      - To email: Placeholder `<RECIEVER_EMAIL>` to be replaced with actual recipient  
      - From email: Placeholder `<YOUR_EMAIL>` to be replaced with sender address  
      - Options: Default (no special options)  
    - Credentials: SMTP account configured with username and password  
    - Input: Text output from Basic LLM Chain node  
    - Output: Success/failure execution status  
    - Edge Cases:  
      - SMTP authentication failure  
      - Email rejection due to invalid addresses or spam filters  
      - Large email content causing sending delays or truncation  
    - Sticky Note Reference: Sticky Note1 (explains email sending purpose)

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                          | Input Node(s)        | Output Node(s)        | Sticky Note                                                                                                                                                                    |
|----------------------------|----------------------------------|----------------------------------------|----------------------|-----------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Get files from drive        | Google Drive Trigger             | Detect new/updated files on Google Drive | None                 | Download the drive files | Type: n8n-nodes-base.googleDriveTrigger. Purpose: Monitors Google Drive for new files. Output: File metadata.                                                                  |
| Download the drive files    | Google Drive                    | Download file content from Google Drive | Get files from drive  | upload frames to cloudinary | Type: n8n-nodes-base.googleDrive. Purpose: Downloads file content for processing. Output: Binary file data.                                                                     |
| upload frames to cloudinary | HTTP Request                   | Upload downloaded file to Cloudinary    | Download the drive files | Basic LLM Chain        | Type: n8n-nodes-base.httpRequest. Purpose: Uploads files to Cloudinary storage. Output: Cloudinary URLs.                                                                        |
| Azure OpenAI Chat Model     | LangChain LM Chat (Azure OpenAI) | Runs GPT-4o-mini AI model               | Basic LLM Chain       | Basic LLM Chain         | Type: n8n-nodes-base.azureOpenAi. Purpose: Backup AI for caption generation. Output: AI text response.                                                                          |
| Basic LLM Chain             | LangChain Chain LLM             | Generates Twitter captions and email body | upload frames to cloudinary, Azure OpenAI Chat Model | Send email              | Type: n8n-nodes-base.langchain. Purpose: Generates Twitter captions using AI with detailed prompt instructions. Output: Generated caption text.                                |
| Send email                 | Email Send                     | Sends generated captions via SMTP email | Basic LLM Chain       | None                  | Type: n8n-nodes-base.emailSend. Purpose: Delivers generated captions via email. Output: Email notification with results.                                                      |
| Sticky Note1               | Sticky Note                   | Documentation note on Send email node   | None                 | None                  | "Type: n8n-nodes-base.emailSend. Purpose: Delivers generated captions via email. Includes file links and AI suggestions. Output: Email notification with results."             |
| Sticky Note2               | Sticky Note                   | Documentation note on Basic LLM Chain   | None                 | None                  | "Type: n8n-nodes-base.langchain. Purpose: Generates Twitter captions using AI. Analyzes uploaded content. Adds hashtags and optimizes for social media."                       |
| Sticky Note3               | Sticky Note                   | Documentation note on upload to Cloudinary | None                 | None                  | "Type: n8n-nodes-base.httpRequest. Purpose: Uploads files to Cloudinary storage. Takes file from Google Drive. Creates public URLs for AI processing."                        |
| Sticky Note4               | Sticky Note                   | Documentation note on Google Drive download | None                 | None                  | "Type: n8n-nodes-base.googleDrive. Purpose: Downloads file content from Google Drive. Output: File content ready for upload."                                                  |
| Sticky Note5               | Sticky Note                   | Documentation note on Google Drive trigger | None                 | None                  | "Type: n8n-nodes-base.googleDriveTrigger. Purpose: Monitors Google Drive for new files. Output: File info (name, URL, metadata)."                                             |
| Sticky Note                | Sticky Note                   | Documentation note on Azure OpenAI node | None                 | None                  | "Type: n8n-nodes-base.azureOpenAi. Purpose: Alternative AI for caption generation. Output: Additional caption suggestions."                                                  |
| Sticky Note6               | Sticky Note                   | Documentation note on preconditions and API requirements | None                 | None                  | "Pre-conditions & API Requirements: Google Drive OAuth2, Cloudinary API & preset, Azure OpenAI endpoint & key, SMTP credentials for email sending."                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node:**  
   - Type: `Google Drive Trigger`  
   - Configure event: `fileUpdated`  
   - Set folder to watch by ID (`1GDkSI9txB88oOcu_jr1tXEzkT0_FoGmC`)  
   - Poll every minute  
   - Connect Google Drive OAuth2 credentials with access to folder

2. **Create Google Drive Node (Download):**  
   - Type: `Google Drive`  
   - Operation: `download`  
   - File ID: Set to `{{$json["id"]}}` from trigger output  
   - Connect to same Google Drive OAuth2 credentials  
   - Connect input from Google Drive Trigger node

3. **Create HTTP Request Node (Upload to Cloudinary):**  
   - Type: `HTTP Request`  
   - Method: `POST`  
   - URL: `https://api.cloudinary.com/v1_1/dpuvigpmt/image/upload`  
   - Content-Type: `multipart/form-data`  
   - Body parameters:  
     - `file`: Binary data from Google Drive download node  
     - `upload_preset`: `"upload"` (ensure this preset exists in Cloudinary account)  
   - Authentication: HTTP Basic Auth using Cloudinary API key and secret credentials  
   - Enable retry on failure with max 5 attempts  
   - Connect input from Google Drive download node

4. **Create Azure OpenAI Chat Model Node:**  
   - Type: `LangChain LM Chat Azure OpenAI`  
   - Model: `gpt-4o-mini`  
   - Connect Azure OpenAI API credentials (endpoint & key)  
   - Connect input from Basic LLM Chain node (see next step)

5. **Create Basic LLM Chain Node:**  
   - Type: `LangChain Chain LLM`  
   - Define system prompt with detailed instructions for caption generation (as described in overview)  
   - Use expression `{{$json.secure_url}}` to insert Cloudinary image URL dynamically  
   - Connect input from Cloudinary HTTP Request node  
   - Connect AI language model input to Azure OpenAI Chat Model node

6. **Create Email Send Node:**  
   - Type: `Email Send`  
   - Subject: `"Twitter Post"` (can be parameterized with AI output for improvement)  
   - HTML: Use expression `{{$json.text}}` from Basic LLM Chain output  
   - To email: Set to intended recipient’s email address  
   - From email: Set sender’s email address  
   - Connect SMTP credentials with valid email account  
   - Connect input from Basic LLM Chain node

7. **Connect Nodes Sequentially:**  
   - Google Drive Trigger → Google Drive Download → Cloudinary Upload → Basic LLM Chain → Email Send  
   - Azure OpenAI Chat Model node connected as AI language model provider to Basic LLM Chain node

8. **Test Workflow:**  
   - Upload or update an image in the target Google Drive folder  
   - Confirm the workflow triggers, uploads image to Cloudinary, generates caption, and sends email successfully

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Pre-conditions: Ensure Google Drive OAuth2 credentials have access to monitored folder. Cloudinary requires API keys and configured upload preset. Azure OpenAI endpoint and key must be valid. SMTP credentials needed for email sending. | Sticky Note6                                                                                   |
| The AI prompt enforces strict format rules for the generated caption and email, including character limits, style, and accessibility compliance. | Node: Basic LLM Chain (LangChain)                                                             |
| Cloudinary upload preset named `"upload"` must exist and allow unsigned uploads or be configured for authenticated uploads. | Node: upload frames to cloudinary (HTTP Request)                                              |
| SMTP email node placeholders `<RECIEVER_EMAIL>` and `<YOUR_EMAIL>` must be replaced with actual email addresses before deployment. | Node: Send email                                                                               |
| Azure OpenAI model used is `gpt-4o-mini`, a compact but capable GPT-4 variant optimized for cost and latency.            | Node: Azure OpenAI Chat Model                                                                 |
| For email compatibility, the prompt specifies inline CSS only and Outlook/Gmail-friendly markup (tables allowed).        | AI prompt in Basic LLM Chain                                                                   |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated n8n workflow. It complies strictly with content policies and contains no unlawful, offensive, or protected elements. All handled data is lawful and public.