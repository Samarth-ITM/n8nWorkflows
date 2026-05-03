Generate Marketing Ad Banners with LINE, Gemini, and Nano Banana Pro

https://n8nworkflows.xyz/workflows/generate-marketing-ad-banners-with-line--gemini--and-nano-banana-pro-11205


# Generate Marketing Ad Banners with LINE, Gemini, and Nano Banana Pro

---

### 1. Workflow Overview

This workflow automates the creation of professional marketing ad banners from simple text descriptions received via LINE messaging. It targets marketers and businesses seeking quick, high-quality banner generation tailored for the Japanese market.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Prompt Optimization:** Receives user input from LINE webhook, extracts relevant data, and leverages Google Gemini for crafting a detailed, marketing-focused image prompt in English based on Japanese input.

- **1.2 Async Image Generation:** Submits the prompt to the Nano Banana Pro image generation service via Kie.ai API, then polls asynchronously until the image generation is complete.

- **1.3 Image Download and Hosting:** Downloads the generated image and uploads it to an AWS S3 bucket configured for public read access.

- **1.4 Delivery to User:** Sends the hosted image back to the user through LINE’s reply message API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Prompt Optimization

- **Overview:**  
  This block receives the incoming message from LINE via webhook, extracts the text and metadata, and uses Google Gemini to transform this input into a highly detailed English prompt optimized for marketing image generation.

- **Nodes Involved:**  
  - LINE Webhook1  
  - Extract Data1  
  - Optimize Prompt (Marketing)1  
  - Code1

- **Node Details:**

  - **LINE Webhook1**  
    - *Type:* Webhook  
    - *Role:* Entry point receiving POST requests from LINE platform.  
    - *Configuration:* Path set to `banner-v7158-2025`, HTTP method POST.  
    - *Inputs/Outputs:* No input, outputs raw LINE event JSON.  
    - *Failures:* Missing or malformed webhook requests; invalid HTTP methods.  

  - **Extract Data1**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses LINE webhook payload to extract replyToken, userId, message text, and timestamp.  
    - *Key Code:* Validates presence of `events` array, extracts first event details.  
    - *Inputs:* LINE webhook JSON.  
    - *Outputs:* Simplified JSON with `replyToken`, `userId`, `message`, `timestamp`.  
    - *Failures:* Throws error if no LINE events found.  

  - **Optimize Prompt (Marketing)1**  
    - *Type:* Google Gemini (PaLM) node  
    - *Role:* Uses Gemini to generate a detailed English prompt for image generation based on user’s Japanese input.  
    - *Configuration:* Model `models/gemini-2.0-flash-lite`, prompt structured with strict instructions for marketing-focused output under 500 characters.  
    - *Inputs:* Extracted message text.  
    - *Outputs:* Raw Gemini JSON response with prompt text.  
    - *Credentials:* Google Palm API credentials required.  
    - *Failures:* API auth failure, rate limits, malformed output.  

  - **Code1**  
    - *Type:* Code (JavaScript)  
    - *Role:* Cleans and extracts the actual prompt text from Gemini’s JSON response, removes unwanted quotes and escapes.  
    - *Inputs:* Gemini output JSON.  
    - *Outputs:* JSON containing cleaned prompt text, userId, and taskId (if present).  
    - *Failures:* Parsing errors if Gemini output is malformed or unexpected format.

---

#### 2.2 Async Image Generation

- **Overview:**  
  Submits the cleaned prompt to Nano Banana Pro via Kie.ai’s API, then waits and repeatedly checks job status until the image is ready.

- **Nodes Involved:**  
  - Submit to Nano Banana Pro1  
  - Wait1  
  - Check Job Status  
  - Wait for Generation1  
  - Parse Result1

- **Node Details:**

  - **Submit to Nano Banana Pro1**  
    - *Type:* HTTP Request  
    - *Role:* Sends POST request to Kie.ai API to create a new image generation task with prompt and parameters (aspect ratio, resolution, format).  
    - *Configuration:* JSON body includes model `nano-banana-pro`, aspect ratio `1:1`, resolution `2K`, output format `png`.  
    - *Authentication:* Generic HTTP header auth (Kie.ai credentials).  
    - *Inputs:* Prompt text from Code1.  
    - *Outputs:* JSON response with task info including taskId and recordId.  
    - *Failures:* API auth errors, network timeouts, invalid parameters.  

  - **Wait1**  
    - *Type:* Wait  
    - *Role:* Pauses workflow for 10 seconds before checking job status.  
    - *Inputs:* From Submit node.  
    - *Outputs:* Triggers Check Job Status.  
    - *Failures:* None typical, but long waits may cause timeouts in some execution environments.  

  - **Check Job Status**  
    - *Type:* HTTP Request  
    - *Role:* Queries the Kie.ai API with `taskId` and `recordId` to get current job status.  
    - *Configuration:* GET request with query parameters `taskId` and `recordId`.  
    - *Authentication:* Generic HTTP header auth (Kie.ai credentials).  
    - *Inputs:* From Wait1, includes task identification info.  
    - *Outputs:* Job status JSON.  
    - *Failures:* Auth errors, invalid IDs, API downtime.  

  - **Wait for Generation1**  
    - *Type:* Wait  
    - *Role:* Additional 10-second delay to avoid hitting API too frequently.  
    - *Inputs:* Check Job Status response indicating still processing.  
    - *Outputs:* Triggers Parse Result1.  

  - **Parse Result1**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses job status response; if successful extracts the generated image URL; if job failed throws an error; if still processing, triggers loop.  
    - *Failures:* JSON parsing errors, missing image URL, task failure states.

---

#### 2.3 Image Download and Hosting

- **Overview:**  
  Downloads the generated image from the URL and uploads it to a configured AWS S3 bucket with public read access.

- **Nodes Involved:**  
  - Download Image1  
  - Upload to S

- **Node Details:**

  - **Download Image1**  
    - *Type:* HTTP Request  
    - *Role:* Fetches the generated banner image file from external URL.  
    - *Configuration:* Uses URL from Parse Result1, response expected as a file stream.  
    - *Inputs:* Image URL JSON.  
    - *Outputs:* Binary image data.  
    - *Failures:* Network issues, URL expiry, 404 not found.  

  - **Upload to S**  
    - *Type:* AWS S3  
    - *Role:* Uploads the image file to AWS S3 bucket named `banners-bot-v7158`.  
    - *Configuration:* Upload operation, filename dynamically generated as `banner-<timestamp>.png`.  
    - *Credentials:* AWS credentials required with write and public-read permissions on the bucket.  
    - *Inputs:* Binary image data from Download Image1.  
    - *Outputs:* JSON containing S3 object location URL.  
    - *Failures:* Permission denied, bucket not public, upload failures.

---

#### 2.4 Delivery to User

- **Overview:**  
  Sends the hosted image URL back to the original LINE user as a reply message containing the image.

- **Nodes Involved:**  
  - HTTP Request2

- **Node Details:**

  - **HTTP Request2**  
    - *Type:* HTTP Request  
    - *Role:* Sends a reply message via LINE Messaging API with the image URL as both original and preview image.  
    - *Configuration:* POST to `https://api.line.me/v2/bot/message/reply`, JSON body includes `replyToken` and image message.  
    - *Authentication:* Bearer token in header (LINE channel access token).  
    - *Inputs:* Uses `replyToken` from LINE webhook and image URL from S3 upload.  
    - *Failures:* Invalid token, message quota exceeded, network errors.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                          | Input Node(s)      | Output Node(s)        | Sticky Note                                                                                         |
|-------------------------|--------------------------------|----------------------------------------|--------------------|-----------------------|---------------------------------------------------------------------------------------------------|
| Main Sticky Note         | Sticky Note                    | Overview and instructions               | -                  | -                     | # Instant Ad Banner Generator...Workflow overview and setup instructions.                          |
| LINE Webhook1            | Webhook                       | Receives input from LINE                | -                  | Extract Data1         | ## 1. Receive & Refine block description.                                                         |
| Extract Data1            | Code                          | Extracts LINE event data                 | LINE Webhook1       | Optimize Prompt (Marketing)1 | ## 1. Receive & Refine block description.                                                         |
| Optimize Prompt (Marketing)1 | Google Gemini (PaLM)         | Creates detailed marketing prompt       | Extract Data1       | Code1                 | ## 1. Receive & Refine block description.                                                         |
| Code1                   | Code                          | Cleans Gemini output to extract prompt  | Optimize Prompt (Marketing)1 | Submit to Nano Banana Pro1 | ## 1. Receive & Refine block description.                                                         |
| Submit to Nano Banana Pro1 | HTTP Request                 | Submits image generation task            | Code1               | Wait1                 | ## 2. Async Generation (Nano Banana) block description.                                           |
| Wait1                    | Wait                          | Waits 10 seconds before status check    | Submit to Nano Banana Pro1 | Check Job Status    | ## 2. Async Generation (Nano Banana) block description.                                           |
| Check Job Status         | HTTP Request                  | Checks job status from Kie API           | Wait1                | Wait for Generation1  | ## 2. Async Generation (Nano Banana) block description.                                           |
| Wait for Generation1     | Wait                          | Additional 10-second wait loop           | Check Job Status     | Parse Result1         | ## 2. Async Generation (Nano Banana) block description.                                           |
| Parse Result1            | Code                          | Parses status, extracts image URL        | Wait for Generation1  | Download Image1       | ## 2. Async Generation (Nano Banana) block description.                                           |
| Download Image1          | HTTP Request                  | Downloads generated image as file        | Parse Result1       | Upload to S           | ## 3. Host & Deliver block description.                                                           |
| Upload to S              | AWS S3                        | Uploads image to public S3 bucket        | Download Image1     | HTTP Request2         | ## 3. Host & Deliver block description.                                                           |
| HTTP Request2            | HTTP Request                  | Sends image back to user via LINE reply | Upload to S         | -                     | ## 3. Host & Deliver block description.                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create LINE Webhook1:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `banner-v7158-2025`  
   - No authentication needed.  

2. **Add Extract Data1 (Code Node):**  
   - Parses LINE webhook payload to extract `replyToken`, `userId`, `message`, and `timestamp`.  
   - JavaScript code as per node description.  
   - Connect LINE Webhook1 output to this node.

3. **Add Optimize Prompt (Marketing)1 (Google Gemini node):**  
   - Model: `models/gemini-2.0-flash-lite`  
   - Input message: Pass extracted message text from Extract Data1.  
   - Prompt instructions: Marketing creative director role with strict formatting rules for English prompt generation.  
   - Configure Google Palm API credentials.  
   - Connect Extract Data1 output to this node.

4. **Add Code1 (Code Node):**  
   - Cleans Gemini response to extract prompt text.  
   - Use provided JS code for parsing and cleaning.  
   - Connect Optimize Prompt output to this node.

5. **Add Submit to Nano Banana Pro1 (HTTP Request):**  
   - Method: POST  
   - URL: `https://api.kie.ai/api/v1/jobs/createTask`  
   - Body (JSON):  
     ```json
     {
       "model": "nano-banana-pro",
       "input": {
         "prompt": "{{ $json.prompt }}",
         "aspect_ratio": "1:1",
         "resolution": "2K",
         "output_format": "png"
       }
     }
     ```  
   - Headers: `Content-Type: application/json`  
   - Authentication: Generic HTTP Header Auth with Kie.ai credentials.  
   - Connect Code1 output to this node.

6. **Add Wait1 (Wait Node):**  
   - Wait duration: 10 seconds.  
   - Connect Submit to Nano Banana Pro1 output to Wait1.

7. **Add Check Job Status (HTTP Request):**  
   - Method: GET  
   - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`  
   - Query parameters:  
     - `taskId` = `={{ $json.data.taskId }}`  
     - `recordId` = `={{ $json.data.recordId }}`  
   - Authentication: Same Kie.ai header auth credentials.  
   - Connect Wait1 output to this node.

8. **Add Wait for Generation1 (Wait Node):**  
   - Wait 10 seconds.  
   - Connect Check Job Status output to this node.

9. **Add Parse Result1 (Code Node):**  
   - Parses job status response JSON, extracts image URL if ready.  
   - Throws error if job failed.  
   - Returns status 'processing' to loop back if incomplete.  
   - Connect Wait for Generation1 output to this node.  

10. **Loop Connections:**  
    - Connect Parse Result1 “processing” output back to Wait1 or Check Job Status to keep polling.  

11. **Add Download Image1 (HTTP Request):**  
    - Method: GET  
    - URL: `={{ $json.imageUrl }}`  
    - Response format: File (binary).  
    - Connect Parse Result1 output to this node.

12. **Add Upload to S (AWS S3 Node):**  
    - Operation: Upload  
    - Bucket name: `banners-bot-v7158`  
    - File name: `banner-{{Date.now()}}.png`  
    - Credentials: AWS account with S3 write and public read permission.  
    - Connect Download Image1 output to this node.

13. **Add HTTP Request2 (HTTP Request):**  
    - Method: POST  
    - URL: `https://api.line.me/v2/bot/message/reply`  
    - Headers:  
      - Authorization: `Bearer YOUR_LINE_CHANNEL_ACCESS_TOKEN`  
      - Content-Type: `application/json`  
    - Body (JSON):  
      ```json
      {
        "replyToken": "{{ $('LINE Webhook1').item.json.body.events[0].replyToken }}",
        "messages": [
          {
            "type": "image",
            "originalContentUrl": "{{ $('Upload to S').item.json.Location }}",
            "previewImageUrl": "{{ $('Upload to S').item.json.Location }}"
          }
        ]
      }
      ```  
    - Connect Upload to S output to this node.

14. **Test the workflow:**  
    - Deploy webhook URL to LINE Developers console.  
    - Ensure all credentials are configured and valid: LINE, Google Gemini, Kie.ai, AWS S3.  
    - Ensure S3 bucket is public read accessible.  
    - Trigger workflow by sending a message to LINE bot.  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Ensure AWS S3 bucket has public read access to allow LINE to fetch images directly from S3 URLs.         | AWS S3 Bucket Policy configuration.                                                                |
| Set up credentials for LINE Messaging API with channel access token for sending replies.                  | https://developers.line.biz/en/docs/messaging-api/getting-started/                                 |
| Google Gemini (PaLM) API requires an active Google Cloud project and API key with PaLM model enabled.    | https://developers.generativeai.google/tutorials/getting-started                                   |
| Kie.ai API credentials must be configured with HTTP header authentication for Nano Banana Pro image tasks.| Kie.ai API documentation: https://docs.kie.ai/                                                     |
| The workflow is designed to handle only the first event in LINE webhook payloads for simplicity.          | For multiple events, the code should be adapted accordingly.                                       |
| The marketing prompt instructions are tailored for Japanese market and require careful text prompt handling.| Sticky Note with prompt instructions explicitly forbids double quotes and requests focused output. |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---