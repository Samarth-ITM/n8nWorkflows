Transform Product Photos into 360° Videos with OpenAI & RunwayML

https://n8nworkflows.xyz/workflows/transform-product-photos-into-360--videos-with-openai---runwayml-3957


# Transform Product Photos into 360° Videos with OpenAI & RunwayML

### 1. Workflow Overview

This n8n workflow automates the transformation of simple product photos into high-quality 360° rotating videos using AI-powered tools. It is designed for eCommerce stores, marketing teams, and content creators to streamline product media generation with minimal manual effort.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures product photo, title, and description from a submitted form.
- **1.2 Upload Original Photo:** Uploads the submitted product photo to Google Drive for storage and further processing.
- **1.3 AI Prompt Generation:** Uses OpenAI via an AI Agent to generate a photorealistic prompt to enhance the product image.
- **1.4 Image Enhancement:** Enhances the original product photo using the generated AI prompt to create a refined image.
- **1.5 Video Generation Request:** Sends a request to RunwayML API to generate a 360° rotating video from the refined image.
- **1.6 Video Retrieval & Status Handling:** Waits and polls RunwayML for video generation status and retrieves the final video when ready.
- **1.7 Notification:** Optionally sends email notifications to users indicating success or failure of video generation, including the video URL if successful.

Each block is connected sequentially to form a robust end-to-end automation pipeline for product video creation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures product data via a form trigger to start the workflow.
- **Nodes Involved:**  
  - On form submission
- **Node Details:**

  **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point capturing product photo, title, and description submitted by users.  
  - Configuration: Uses a webhook ID to listen for form submissions.  
  - Inputs: External HTTP/form submission.  
  - Outputs: Sends captured data to "Upload Original Photo".  
  - Edge Cases: Missing or malformed form data; webhook misconfiguration.  
  - Version: 2.2  

---

#### 2.2 Upload Original Photo

- **Overview:** Uploads the submitted product photo file to Google Drive for persistent storage and further processing.  
- **Nodes Involved:**  
  - Upload Original Photo  
- **Node Details:**

  **Upload Original Photo**  
  - Type: Google Drive node  
  - Role: Upload the original product photo to a configured Google Drive folder.  
  - Configuration: Connects using Google Drive OAuth2 or Service Account credentials; target folder ID specified via placeholder `{{GDRIVE_FOLDER_ID}}`.  
  - Inputs: Triggered by form submission node, receives file binary data.  
  - Outputs: Passes file metadata and Google Drive file reference to "AI Agent".  
  - Edge Cases: Authentication failures, quota exceeded, invalid folder ID, file upload errors.  
  - Version: 3  

---

#### 2.3 AI Prompt Generation

- **Overview:** Generates a photorealistic AI prompt based on product title and description to enhance the image using OpenAI.  
- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
- **Node Details:**

  **AI Agent**  
  - Type: Langchain Agent  
  - Role: Orchestrates AI prompt generation logic using OpenAI Chat model.  
  - Configuration: Linked to OpenAI Chat Model as language model.  
  - Inputs: Receives Google Drive file info and product metadata.  
  - Outputs: Passes AI-generated prompt to "Download File".  
  - Edge Cases: API limits, invalid prompts, AI service downtime.  
  - Version: 1.8  

  **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides GPT-based chat completions for prompt generation.  
  - Configuration: Uses OpenAI API Key credentials; model and temperature parameters can be customized.  
  - Inputs: Receives prompt instructions from AI Agent.  
  - Outputs: Returns generated prompt back to AI Agent.  
  - Edge Cases: API key invalid, rate limits, network issues.  
  - Version: 1.2  

---

#### 2.4 Image Enhancement

- **Overview:** Downloads the original photo, enhances it using the AI-generated prompt, and prepares a refined image file.  
- **Nodes Involved:**  
  - Download File  
  - Create Refined Image  
  - Convert to File  
  - Get URL  
- **Node Details:**

  **Download File**  
  - Type: Google Drive node  
  - Role: Downloads the original uploaded photo from Google Drive.  
  - Inputs: From AI Agent node.  
  - Outputs: Passes binary image data to "Create Refined Image".  
  - Edge Cases: File not found, permission errors.  
  - Version: 3  

  **Create Refined Image**  
  - Type: HTTP Request node  
  - Role: Calls an external API (likely RunwayML or other enhancement service) to enhance the image using the generated AI prompt.  
  - Configuration: HTTP POST with image binary and prompt as payload.  
  - Inputs: Original image file and AI prompt.  
  - Outputs: Enhanced image as response.  
  - Edge Cases: API failures, timeouts, invalid payload.  
  - Version: 4.2  

  **Convert to File**  
  - Type: Convert to File node  
  - Role: Converts enhanced image data into a file format compatible with downstream nodes.  
  - Inputs: Response from Create Refined Image.  
  - Outputs: Sends file to "Get URL".  
  - Edge Cases: Conversion errors, unsupported formats.  
  - Version: 1.1  

  **Get URL**  
  - Type: HTTP Request node  
  - Role: Generates or retrieves a publicly accessible URL for the refined image file to be used by video generation.  
  - Inputs: File from Convert to File node.  
  - Outputs: Passes URL to "Generate Video".  
  - Edge Cases: URL generation failure, access restrictions.  
  - Version: 4.2  

---

#### 2.5 Video Generation Request

- **Overview:** Initiates 360° rotating video generation on RunwayML using the refined image URL and configured parameters.  
- **Nodes Involved:**  
  - Generate Video  
  - Wait 30 sec  
- **Node Details:**

  **Generate Video**  
  - Type: HTTP Request node  
  - Role: Sends POST request to RunwayML API to start video generation with input image URL and other parameters (duration, aspect ratio, rotation speed).  
  - Inputs: Refined image URL.  
  - Outputs: Receives job ID or status response.  
  - Edge Cases: API authentication failures, invalid parameters, rate limits.  
  - Version: 4.2  

  **Wait 30 sec**  
  - Type: Wait node  
  - Role: Pauses workflow for 30 seconds to allow video processing on RunwayML.  
  - Inputs: From Generate Video node.  
  - Outputs: Triggers next step to poll video generation status.  
  - Edge Cases: Excessive wait time or premature polling.  
  - Version: 1.1  

---

#### 2.6 Video Retrieval & Status Handling

- **Overview:** Polls RunwayML API to check video generation status, retrieves the final video URL once ready, and routes workflow based on success or failure.  
- **Nodes Involved:**  
  - Get Video  
  - Route Actions Based on Status from Video Generation Tool (Switch)  
  - Wait 5 Sec  
- **Node Details:**

  **Get Video**  
  - Type: HTTP Request node  
  - Role: Queries RunwayML API for current job status and video URL if completed.  
  - Inputs: Triggered after wait nodes.  
  - Outputs: Passes status and video URL to Switch node.  
  - Edge Cases: API errors, job not found, timeout.  
  - Version: 4.2  

  **Route Actions Based on Status from Video Generation Tool (Switch)**  
  - Type: Switch node  
  - Role: Routes workflow depending on video generation status: success, failure, or pending.  
  - Inputs: Status response from Get Video.  
  - Outputs:  
    - Success → Video Generation Success Email node  
    - Failure → Video Generation Failure Email node  
    - Pending → Wait 5 Sec node (poll again)  
  - Edge Cases: Unrecognized status values, infinite polling loops.  
  - Version: 3.2  

  **Wait 5 Sec**  
  - Type: Wait node  
  - Role: Pauses for 5 seconds before re-polling the video status.  
  - Inputs: From Switch node on pending status.  
  - Outputs: Loops back to Get Video node.  
  - Edge Cases: Excessive API calls if video takes long to generate.  
  - Version: 1.1  

---

#### 2.7 Notification

- **Overview:** Sends email notifications to the user/admin informing about video generation success with video URL or failure.  
- **Nodes Involved:**  
  - Video Generation Success Email with Video URL  
  - Video Generation Failure Email  
- **Node Details:**

  **Video Generation Success Email with Video URL**  
  - Type: Email Send node  
  - Role: Sends email with link to the generated 360° product video.  
  - Configuration: SMTP credentials configured; email template customizable.  
  - Inputs: Receives video URL and metadata from Switch node on success.  
  - Outputs: Ends workflow or triggers further actions if configured.  
  - Edge Cases: SMTP errors, invalid email addresses.  
  - Version: 2.1  

  **Video Generation Failure Email**  
  - Type: Email Send node  
  - Role: Sends failure notification email to alert about video generation issues.  
  - Configuration: SMTP credentials configured; failure message template.  
  - Inputs: Triggered from Switch node on failure status.  
  - Outputs: Ends workflow or triggers retries if configured.  
  - Edge Cases: SMTP authentication errors.  
  - Version: 2.1  

---

### 3. Summary Table

| Node Name                                  | Node Type                 | Functional Role                         | Input Node(s)                  | Output Node(s)                                 | Sticky Note                                      |
|--------------------------------------------|---------------------------|---------------------------------------|-------------------------------|------------------------------------------------|-------------------------------------------------|
| On form submission                         | Form Trigger              | Entry point capturing form data       | -                             | Upload Original Photo                           |                                                 |
| Upload Original Photo                      | Google Drive              | Upload product photo to Drive          | On form submission             | AI Agent                                       |                                                 |
| AI Agent                                  | Langchain Agent           | Generate AI prompt for image enhancement | Upload Original Photo          | Download File                                  |                                                 |
| OpenAI Chat Model                         | Langchain OpenAI Chat Model | Provides GPT-based prompt generation  | AI Agent (ai_languageModel)    | AI Agent (ai_languageModel)                     |                                                 |
| Download File                             | Google Drive              | Download original photo from Drive    | AI Agent                      | Create Refined Image                            |                                                 |
| Create Refined Image                      | HTTP Request              | Enhance image using AI prompt         | Download File                 | Convert to File                                |                                                 |
| Convert to File                          | Convert to File           | Prepare enhanced image file           | Create Refined Image           | Get URL                                        |                                                 |
| Get URL                                  | HTTP Request              | Generate accessible URL for refined image | Convert to File               | Generate Video                                 |                                                 |
| Generate Video                           | HTTP Request              | Request 360° video generation         | Get URL                       | Wait 30 sec                                    |                                                 |
| Wait 30 sec                             | Wait                      | Pause to allow video processing       | Generate Video                | Get Video                                      |                                                 |
| Get Video                               | HTTP Request              | Poll video generation status          | Wait 30 sec, Wait 5 Sec        | Route Actions Based on Status from Video Generation Tool |                                                 |
| Route Actions Based on Status from Video Generation Tool | Switch                    | Route workflow by video generation status | Get Video                    | Video Generation Failure Email, Video Generation Success Email with Video URL, Wait 5 Sec (poll) |                                                 |
| Wait 5 Sec                              | Wait                      | Pause before re-polling status        | Route Actions Based on Status  | Get Video                                      |                                                 |
| Video Generation Failure Email           | Email Send                | Notify failure of video generation    | Route Actions Based on Status  | -                                              |                                                 |
| Video Generation Success Email with Video URL | Email Send                | Notify success with generated video link | Route Actions Based on Status | -                                              |                                                 |
| Sticky Note                              | Sticky Note               | Comment/Annotation                    | -                             | -                                              |                                                 |
| Sticky Note1                             | Sticky Note               | Comment/Annotation                    | -                             | -                                              |                                                 |
| Sticky Note2                             | Sticky Note               | Comment/Annotation                    | -                             | -                                              |                                                 |
| Sticky Note3                             | Sticky Note               | Comment/Annotation                    | -                             | -                                              |                                                 |
| Sticky Note4                             | Sticky Note               | Comment/Annotation                    | -                             | -                                              |                                                 |
| Sticky Note5                             | Sticky Note               | Comment/Annotation                    | -                             | -                                              |                                                 |
| Sticky Note6                             | Sticky Note               | Comment/Annotation                    | -                             | -                                              |                                                 |
| Sticky Note7                             | Sticky Note               | Comment/Annotation                    | -                             | -                                              |                                                 |
| Sticky Note8                             | Sticky Note               | Comment/Annotation                    | -                             | -                                              |                                                 |
| Sticky Note9                             | Sticky Note               | Comment/Annotation                    | -                             | -                                              |                                                 |
| Sticky Note10                            | Sticky Note               | Comment/Annotation                    | -                             | -                                              |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger  
   - Configure webhook to receive product photo, title, and description.  
   - Position: Start node.

2. **Create "Upload Original Photo" node**  
   - Type: Google Drive  
   - Connect credentials for Google Drive OAuth2 or Service Account.  
   - Configure to upload the file received from the form trigger to folder `{{GDRIVE_FOLDER_ID}}`.  
   - Connect output of form trigger to this node.

3. **Create "AI Agent" node**  
   - Type: Langchain Agent  
   - Link to "OpenAI Chat Model" as its language model.  
   - Configure to accept information including the file link, product title, and description.  
   - Connect output of Google Drive upload node to this node.

4. **Create "OpenAI Chat Model" node**  
   - Type: Langchain OpenAI Chat Model  
   - Configure with OpenAI API key credentials.  
   - Set model parameters (e.g., GPT-4 or GPT-3.5 turbo), temperature as required.  
   - Connect as AI language model input of AI Agent.

5. **Create "Download File" node**  
   - Type: Google Drive  
   - Configure to download the uploaded original product photo binary using the file ID from AI Agent output.  
   - Connect AI Agent output to this node.

6. **Create "Create Refined Image" node**  
   - Type: HTTP Request  
   - Configure POST request to image enhancement API (RunwayML or equivalent).  
   - Payload includes original image binary and AI-generated prompt text.  
   - Connect output of Download File node.

7. **Create "Convert to File" node**  
   - Type: Convert to File  
   - Configure to convert response from image enhancement API into a file format.  
   - Connect from Create Refined Image node.

8. **Create "Get URL" node**  
   - Type: HTTP Request  
   - Configure to generate a public URL for the refined image file to be used by RunwayML for video generation.  
   - Connect from Convert to File node.

9. **Create "Generate Video" node**  
   - Type: HTTP Request  
   - Configure POST request to RunwayML API to start video generation, passing refined image URL and parameters like video duration, aspect ratio, rotation speed.  
   - Connect from Get URL node.

10. **Create "Wait 30 sec" node**  
    - Type: Wait  
    - Set duration to 30 seconds.  
    - Connect from Generate Video node.

11. **Create "Get Video" node**  
    - Type: HTTP Request  
    - Configure GET or POST request to RunwayML API to poll video generation status and fetch video URL if ready.  
    - Connect from Wait 30 sec node.

12. **Create "Route Actions Based on Status from Video Generation Tool" node**  
    - Type: Switch  
    - Configure conditions to route based on video generation status (e.g., success, failure, pending).  
    - Connect from Get Video node.

13. **Create "Video Generation Success Email with Video URL" node**  
    - Type: Email Send  
    - Configure SMTP/email credentials.  
    - Customize email template to include video URL and success message.  
    - Connect success branch from Switch node.

14. **Create "Video Generation Failure Email" node**  
    - Type: Email Send  
    - Configure SMTP/email credentials.  
    - Customize email template for failure notification.  
    - Connect failure branch from Switch node.

15. **Create "Wait 5 Sec" node**  
    - Type: Wait  
    - Set duration to 5 seconds.  
    - Connect pending branch from Switch node.  
    - Loop output back to "Get Video" node to re-poll status.

16. **Set up credentials**  
    - Google Drive OAuth2 / Service Account for file upload and download.  
    - OpenAI API key for AI prompt generation.  
    - RunwayML API key for video generation.  
    - SMTP/email credentials for notification emails.

17. **Replace placeholders**  
    - Replace `{{GDRIVE_FOLDER_ID}}` with your Google Drive folder ID.  
    - Set sender email `From` address in email nodes (e.g., `admin@example.com`).

18. **Test workflow**  
    - Submit product photo, title, and description via the configured form/webhook.  
    - Monitor logs and outputs for successful video generation and notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Fully automated end-to-end product videography pipeline using AI-powered tools OpenAI and RunwayML.            | Workflow description overview.                                                                               |
| Secure credential management with no hardcoded API keys, ensuring best practices for security.                 | Security best practices for credential configuration.                                                        |
| Modular node architecture allows easy scaling and customization for different video formats and styles.        | Workflow design philosophy.                                                                                   |
| Ideal for Shopify, WooCommerce, marketing teams, and content creators looking to streamline product media.     | Use cases section.                                                                                            |
| Setup instructions require Google Drive, OpenAI, RunwayML API credentials, and optionally SMTP for email alerts.| Setup prerequisites and deployment notes.                                                                    |
| Replace placeholders `{{GDRIVE_FOLDER_ID}}` and email addresses before production deployment.                   | Critical configuration reminders.                                                                             |
| Video generation parameters such as duration, aspect ratio, and rotation speed can be customized in HTTP nodes. | Customization options for video output.                                                                       |
| Refer to n8n documentation for setting up OAuth2 credentials and Webhook triggers.                              | https://docs.n8n.io                                                                                           |
| RunwayML API documentation for details on video generation endpoints and parameters.                           | https://docs.runwayml.com/api                                                                               |
| OpenAI API documentation for prompt engineering and chat completions.                                          | https://platform.openai.com/docs/api-reference/chat                                                          |

---

This completes the comprehensive analysis and documentation of the "Transform Product Photos into 360° Videos with OpenAI & RunwayML" workflow. It is designed for maintainability, ease of reproduction, and extensibility by both human developers and AI automation agents.