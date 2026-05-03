Generate AI Video Sales Letters with Google VEO3, Creatomate Captions & Facebook Publishing

https://n8nworkflows.xyz/workflows/generate-ai-video-sales-letters-with-google-veo3--creatomate-captions---facebook-publishing-6449


# Generate AI Video Sales Letters with Google VEO3, Creatomate Captions & Facebook Publishing

---

### 1. Workflow Overview

This workflow automates the creation and publishing of AI-generated Video Sales Letters (VSL) optimized for Meta (Facebook) platforms. It integrates AI content generation, video creation, captioning, cloud storage, and social media publishing, structured into several logical blocks:

- **1.1 Input Reception:** Captures user input via a web form and prepares it for AI processing.
- **1.2 AI Content Generation:** Uses OpenAI models to generate UGC-style copy, product image descriptions, and video scripts.
- **1.3 Video Creation & Processing:** Authenticates with a video generation API, triggers video creation, polls for completion, converts video formats, and uploads the final video to Google Cloud Storage.
- **1.4 Captioning & Rendering:** Adds captions to the video via HTTP requests, waits for rendering completion, and verifies the captioning status.
- **1.5 Publishing:** Depending on captioning status, uploads the video post to Facebook or re-triggers the captioning/rendering process.
- **1.6 Control Flow & Decision Making:** Uses switch and wait nodes to manage asynchronous API responses and branching logic.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures form submissions and prepares initial data for AI content generation.
- **Nodes Involved:**  
  - On form submission  
  - Form Input  
  - UGC.Formulator  

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry-point webhook listening for user-submitted form data.  
    - Configuration: Uses a webhook ID to receive data asynchronously.  
    - Inputs: External HTTP form submissions.  
    - Outputs: Passes data to "Form Input".  
    - Edge Cases: Missing or malformed form data, webhook downtime.  

  - **Form Input**  
    - Type: Set  
    - Role: Prepares and formats data from form submission for downstream nodes.  
    - Configuration: Sets variables or adjusts data structure.  
    - Inputs: Output from "On form submission".  
    - Outputs: Passes to "UGC.Formulator".  
    - Edge Cases: Expression errors if expected fields are missing.  

  - **UGC.Formulator**  
    - Type: OpenAI (LangChain)  
    - Role: Generates user-generated-content style copy based on form input.  
    - Configuration: Calls OpenAI with prompt templates tailored for UGC content generation.  
    - Inputs: Data from "Form Input".  
    - Outputs: Passes generated content to "Product Image Describing".  
    - Edge Cases: API rate limits, malformed prompts, network errors.  

#### 1.2 AI Content Generation

- **Overview:** Generates a product image description and video script prompts to feed video generation APIs.  
- **Nodes Involved:**  
  - Product Image Describing  
  - VeoPrompt  
  - Content writer  
  - SET Credentials  

- **Node Details:**

  - **Product Image Describing**  
    - Type: OpenAI (LangChain)  
    - Role: Creates descriptive text for product images to enhance video content.  
    - Configuration: Uses AI language model with specific prompts for image description.  
    - Inputs: Output from "UGC.Formulator".  
    - Outputs: Data forwarded to "VeoPrompt".  
    - Edge Cases: API failures, unexpected inputs.  

  - **VeoPrompt**  
    - Type: OpenAI (LangChain)  
    - Role: Generates video script or prompts based on product description.  
    - Configuration: Uses prompt engineering to generate scripts suitable for video production.  
    - Inputs: From "Product Image Describing".  
    - Outputs: Feeds "Content writer".  
    - Edge Cases: AI output errors, timeout.  

  - **Content writer**  
    - Type: Chain LLM (LangChain)  
    - Role: Refines and structures the AI-generated script content.  
    - Configuration: Uses structured output parser for clean data extraction.  
    - Inputs: From "VeoPrompt" and "OpenAI Chat Model" (for language model integration).  
    - Outputs: Passes to "SET Credentials".  
    - Edge Cases: Parsing errors, API rate limits.  

  - **SET Credentials**  
    - Type: Set  
    - Role: Prepares authentication credentials for video generation API calls.  
    - Configuration: Sets necessary tokens or keys for JWT generation.  
    - Inputs: From "Content writer".  
    - Outputs: Passes to "JWT".  
    - Edge Cases: Missing or invalid credential data.  

#### 1.3 Video Creation & Processing

- **Overview:** Authenticates, requests video generation, polls status, converts to correct format, and uploads the video file to Google Cloud Storage.  
- **Nodes Involved:**  
  - JWT  
  - GET TOKEN  
  - Generate Video1  
  - Wait  
  - Fetch Status  
  - Switch  
  - Convert to File  
  - Upload to GCS (To be accessible via URL)  

- **Node Details:**

  - **JWT**  
    - Type: JWT  
    - Role: Creates JWT token for secure API authentication.  
    - Configuration: Uses credentials from "SET Credentials".  
    - Inputs: From "SET Credentials".  
    - Outputs: Passes JWT token to "GET TOKEN".  
    - Edge Cases: Token expiration, signing errors.  

  - **GET TOKEN**  
    - Type: HTTP Request  
    - Role: Requests an access token for video generation service using JWT.  
    - Configuration: HTTP POST with JWT in authorization header.  
    - Inputs: From "JWT".  
    - Outputs: Passes access token to "Generate Video1".  
    - Edge Cases: HTTP errors, invalid JWT, connectivity issues.  

  - **Generate Video1**  
    - Type: HTTP Request  
    - Role: Initiates video generation with the service using access token and prompts.  
    - Configuration: Sends video parameters and script prompts.  
    - Inputs: From "GET TOKEN".  
    - Outputs: Passes job ID or status to "Wait".  
    - Edge Cases: API rate limits, invalid parameters, service downtime.  

  - **Wait**  
    - Type: Wait  
    - Role: Delays workflow to allow video generation processing.  
    - Configuration: Waits for a fixed or dynamic period.  
    - Inputs: From "Generate Video1".  
    - Outputs: Passes control to "Fetch Status".  
    - Edge Cases: Timeout, premature continuation.  

  - **Fetch Status**  
    - Type: HTTP Request  
    - Role: Polls video generation API to check job status.  
    - Configuration: HTTP GET with job ID.  
    - Inputs: From "Wait".  
    - Outputs: Passes status to "Switch".  
    - Edge Cases: API errors, stale job IDs, network issues.  

  - **Switch**  
    - Type: Switch  
    - Role: Branches workflow based on status (e.g., success, failure, in progress).  
    - Configuration: Conditions check job status field.  
    - Inputs: From "Fetch Status".  
    - Outputs:  
      - On success: "Convert to File"  
      - On failure: "VeoPrompt" (to regenerate script)  
      - On pending: "Wait" (to poll again)  
    - Edge Cases: Unexpected status values, misconfigured conditions.  

  - **Convert to File**  
    - Type: Convert to File  
    - Role: Converts generated video to 9:16 aspect ratio format suitable for Meta platforms.  
    - Configuration: Uses video processing parameters for format conversion.  
    - Inputs: From "Switch" (success branch).  
    - Outputs: Passes to "Upload to GCS (To be accessible via URL)".  
    - Edge Cases: Conversion failures, unsupported formats.  

  - **Upload to GCS (To be accessible via URL)**  
    - Type: Google Cloud Storage  
    - Role: Uploads finalized video file to GCS for public access.  
    - Configuration: Stores video under configured bucket and path with retry enabled.  
    - Inputs: From "Convert to File".  
    - Outputs: Passes uploaded file URL to "Add Captions".  
    - Edge Cases: Authentication errors, network failures, quota limits.  

#### 1.4 Captioning & Rendering

- **Overview:** Adds captions via external API, waits for rendering, and verifies caption status before publishing.  
- **Nodes Involved:**  
  - Add Captions  
  - Rendering....  
  - Captions added?  
  - Switch1  

- **Node Details:**

  - **Add Captions**  
    - Type: HTTP Request  
    - Role: Sends video URL and caption data to captioning service.  
    - Configuration: HTTP POST with video URL.  
    - Inputs: From "Upload to GCS (To be accessible via URL)".  
    - Outputs: Passes job ID or status to "Rendering....".  
    - Edge Cases: API failures, invalid video URL.  

  - **Rendering....**  
    - Type: Wait  
    - Role: Delays workflow to allow caption rendering to complete.  
    - Configuration: Waits fixed period or until webhook callback.  
    - Inputs: From "Add Captions" and "Switch1" (multiple branches).  
    - Outputs: Passes control to "Captions added?".  
    - Edge Cases: Timeout or premature continuation.  

  - **Captions added?**  
    - Type: HTTP Request  
    - Role: Checks if captioning is complete and successful.  
    - Configuration: HTTP GET with job ID.  
    - Inputs: From "Rendering....".  
    - Outputs: Passes status to "Switch1".  
    - Edge Cases: API errors, invalid job ID.  

  - **Switch1**  
    - Type: Switch  
    - Role: Branches based on captioning status to decide next action.  
    - Configuration: Multiple conditions including:  
      - Upload Post (if captioning successful)  
      - Add Captions (retry)  
      - Rendering.... (wait again or error handling)  
    - Inputs: From "Captions added?".  
    - Outputs: Various nodes as above.  
    - Edge Cases: Infinite loops if captioning never succeeds, misconfigured conditions.  

#### 1.5 Publishing

- **Overview:** Posts the final video content to Facebook via a dedicated upload node.  
- **Nodes Involved:**  
  - Upload Post  

- **Node Details:**

  - **Upload Post**  
    - Type: Upload Post (custom or third-party node)  
    - Role: Publishes the video post to Facebook using the prepared video URL and captions.  
    - Configuration: Uses Facebook OAuth2 credentials and post parameters.  
    - Inputs: From "Switch1" success branch.  
    - Outputs: Final output node, ends workflow.  
    - Edge Cases: Authentication failures, API limits, content rejection by Facebook.  

---

### 3. Summary Table

| Node Name                       | Node Type                          | Functional Role                          | Input Node(s)              | Output Node(s)                        | Sticky Note                               |
|--------------------------------|----------------------------------|----------------------------------------|----------------------------|-------------------------------------|-------------------------------------------|
| On form submission             | Form Trigger                     | Entry point for user input via form    | N/A                        | Form Input                          |                                           |
| Form Input                    | Set                              | Prepares form data for AI processing   | On form submission         | UGC.Formulator                     |                                           |
| UGC.Formulator                | OpenAI (LangChain)               | Generates UGC style copy                | Form Input                 | Product Image Describing            |                                           |
| Product Image Describing      | OpenAI (LangChain)               | Creates product image description      | UGC.Formulator             | VeoPrompt                         |                                           |
| VeoPrompt                    | OpenAI (LangChain)               | Generates video script prompt           | Product Image Describing   | Content writer                    |                                           |
| Content writer               | Chain LLM (LangChain)            | Refines AI-generated script             | VeoPrompt, OpenAI Chat Model | SET Credentials                  |                                           |
| SET Credentials              | Set                              | Prepares API credentials for video API | Content writer             | JWT                              |                                           |
| JWT                         | JWT                              | Generates JWT token for API auth        | SET Credentials            | GET TOKEN                        |                                           |
| GET TOKEN                   | HTTP Request                     | Retrieves access token for video API    | JWT                        | Generate Video1                  |                                           |
| Generate Video1             | HTTP Request                     | Initiates video generation job          | GET TOKEN                  | Wait                            |                                           |
| Wait                        | Wait                             | Waits for video generation job to process | Generate Video1            | Fetch Status                    |                                           |
| Fetch Status                | HTTP Request                     | Polls video generation job status       | Wait                       | Switch                         |                                           |
| Switch                      | Switch                          | Branches workflow by video job status   | Fetch Status               | Convert to File, VeoPrompt, Wait |                                           |
| Convert to File             | Convert to File                  | Converts video to 9:16 aspect ratio     | Switch (success branch)    | Upload to GCS (To be accessible via URL) |                                           |
| Upload to GCS (To be accessible via URL)| Google Cloud Storage           | Uploads video to cloud storage for access | Convert to File           | Add Captions                   |                                           |
| Add Captions                | HTTP Request                     | Sends video for captioning               | Upload to GCS              | Rendering....                  |                                           |
| Rendering....               | Wait                             | Waits for caption rendering to complete | Add Captions, Switch1      | Captions added?                |                                           |
| Captions added?             | HTTP Request                     | Checks captioning status                 | Rendering....              | Switch1                       |                                           |
| Switch1                    | Switch                          | Branches by captioning status            | Captions added?            | Upload Post, Add Captions, Rendering.... |                                           |
| Upload Post                 | Upload Post (custom node)        | Publishes video post to Facebook         | Switch1                    | N/A                          |                                           |
| OpenAI Chat Model           | LangChain LM Chat OpenAI         | Provides language model for script refinement | N/A                      | Content writer                |                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Point:**
   - Add a **Form Trigger** node named "On form submission".
   - Configure webhook to receive form data asynchronously.

2. **Prepare Form Data:**
   - Add a **Set** node "Form Input".
   - Connect "On form submission" to "Form Input".
   - Configure to map and format incoming form fields for AI usage.

3. **Generate UGC Style Copy:**
   - Add an **OpenAI (LangChain)** node "UGC.Formulator".
   - Connect "Form Input" to "UGC.Formulator".
   - Configure prompt templates for user-generated content style copy.
   - Set OpenAI credentials (API key).

4. **Generate Product Image Description:**
   - Add an **OpenAI (LangChain)** node "Product Image Describing".
   - Connect "UGC.Formulator" to "Product Image Describing".
   - Configure prompt for image description generation.

5. **Generate Video Script Prompt:**
   - Add an **OpenAI (LangChain)** node "VeoPrompt".
   - Connect "Product Image Describing" to "VeoPrompt".
   - Configure prompt for video script generation.

6. **Refine Script Content:**
   - Add a **Chain LLM (LangChain)** node "Content writer".
   - Connect "VeoPrompt" and an **OpenAI Chat Model** node to "Content writer".
   - Configure output parser for structured output.
   - Set OpenAI API credentials.

7. **Set API Credentials:**
   - Add a **Set** node "SET Credentials".
   - Connect "Content writer" to "SET Credentials".
   - Configure to set necessary credentials/tokens for JWT creation.

8. **Create JWT Token:**
   - Add a **JWT** node named "JWT".
   - Connect "SET Credentials" to "JWT".
   - Configure JWT signing parameters (private key, algorithm).

9. **Get Access Token:**
   - Add an **HTTP Request** node "GET TOKEN".
   - Connect "JWT" to "GET TOKEN".
   - Configure POST request to video generation API token endpoint with JWT in headers.

10. **Trigger Video Generation:**
    - Add an **HTTP Request** node "Generate Video1".
    - Connect "GET TOKEN" to "Generate Video1".
    - Configure request body with video script and parameters.

11. **Wait for Processing:**
    - Add a **Wait** node "Wait".
    - Connect "Generate Video1" to "Wait".
    - Configure fixed or dynamic wait time.

12. **Poll Video Generation Status:**
    - Add an **HTTP Request** node "Fetch Status".
    - Connect "Wait" to "Fetch Status".
    - Configure GET request with job ID to check status.

13. **Branch by Status:**
    - Add a **Switch** node "Switch".
    - Connect "Fetch Status" to "Switch".
    - Configure conditions:
      - On success: connect to "Convert to File"
      - On failure: connect to "VeoPrompt" (to regenerate script)
      - On pending: connect back to "Wait"

14. **Convert Video Format:**
    - Add a **Convert to File** node "Convert to File".
    - Connect "Switch" success branch to this node.
    - Configure to convert video to 9:16 aspect ratio.

15. **Upload Video to Google Cloud Storage:**
    - Add a **Google Cloud Storage** node "Upload to GCS (To be accessible via URL)".
    - Connect "Convert to File" to this node.
    - Configure bucket, path, authentication credentials.
    - Enable retry on failure.

16. **Add Captions:**
    - Add an **HTTP Request** node "Add Captions".
    - Connect "Upload to GCS" to "Add Captions".
    - Configure POST request to captioning service with video URL.

17. **Wait for Caption Rendering:**
    - Add a **Wait** node "Rendering....".
    - Connect "Add Captions" and some branches from "Switch1" to "Rendering....".
    - Configure appropriate wait time or webhook callback.

18. **Check Caption Status:**
    - Add an **HTTP Request** node "Captions added?".
    - Connect "Rendering...." to "Captions added?".
    - Configure GET request with caption job ID.

19. **Branch by Caption Status:**
    - Add a **Switch** node "Switch1".
    - Connect "Captions added?" to "Switch1".
    - Configure multiple conditions:
      - If captions ready, connect to "Upload Post"
      - If not, loop to "Add Captions" or "Rendering...." for retry

20. **Publish Video Post:**
    - Add a **Upload Post** node (custom Facebook node).
    - Connect "Switch1" success branch to this node.
    - Configure Facebook OAuth2 credentials.
    - Map video URL and caption data for post content.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                             |
|----------------------------------------------------------------------------------------------------|---------------------------------------------|
| The video should be generated and ready to convert to 9:16 aspect ratio before the "Convert to File" node. | Sticky note near "Convert to File" node.    |
| Enable retry on fail with a 2-second wait between retries for Google Cloud Storage uploads.         | Configured in "Upload to GCS (To be accessible via URL)". |
| Use webhook-based triggers for asynchronous waits in "Wait" and "Rendering...." nodes.             | Webhook IDs configured in respective wait nodes. |
| Facebook publishing requires OAuth2 authentication configured in the "Upload Post" node.            | Ensure valid Facebook API credentials.      |
| API rate limits and network errors must be handled gracefully, especially for OpenAI and HTTP requests. | General integration best practice.           |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects the applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---