Instant Infographic Generator (LINE + Nano Banana Pro)

https://n8nworkflows.xyz/workflows/instant-infographic-generator--line---nano-banana-pro--11244


# Instant Infographic Generator (LINE + Nano Banana Pro)

### 1. Workflow Overview

This workflow, titled **Instant Infographic Generator (LINE + Nano Banana Pro)**, automates the creation and delivery of professional infographics based on user-submitted data via the LINE messaging platform. It is designed for scenarios where users want to transform complex data or topics into visually appealing, easy-to-understand infographics without manual design effort.

The workflow logically divides into three main blocks:

- **1.1 Data Processing**: Receives data input from LINE, extracts relevant information, and uses Google Gemini AI to analyze and transform the data into a structured prompt describing the infographic to be generated.
  
- **1.2 Infographic Rendering**: Submits the structured prompt to the Nano Banana Pro API, which generates the infographic image. This block includes a smart polling loop to check the job status until the graphic is ready.

- **1.3 Host & Deliver**: Downloads the generated infographic, uploads it to an AWS S3 bucket with public access, and replies to the original LINE user with the hosted infographic image.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Data Processing

**Overview:**  
This block handles incoming data from LINE, parses the message content, and uses Google Gemini AI to convert the raw topic or data points into a detailed and professional infographic prompt tailored for Nano Banana Pro.

**Nodes Involved:**  
- LINE Webhook  
- Extract Data  
- Optimize Prompt (Data Vis)  
- Parse Gemini Response

**Node Details:**

- **LINE Webhook**  
  - *Type:* Webhook  
  - *Role:* Entry point listening for HTTP POST requests from LINE messages.  
  - *Configuration:* Path set as `infographic-v7158`, method POST, no additional options.  
  - *Input/Output:* Receives raw LINE webhook JSON, outputs to Extract Data node.  
  - *Failure Modes:* Missing or malformed webhook requests; LINE signature validation not described here (may be external).  
  - *Notes:* Requires LINE channel webhook setup with the provided path.

- **Extract Data**  
  - *Type:* Code  
  - *Role:* Parses the LINE event JSON to extract the reply token, user ID, message text, and timestamp.  
  - *Configuration:* Custom JavaScript code that validates event presence and extracts key fields.  
  - *Input/Output:* Input from LINE Webhook; outputs simplified JSON with fields `replyToken`, `userId`, `message`, and `timestamp`.  
  - *Expressions:* Uses `items[0].json.body` to access raw webhook data.  
  - *Failure Modes:* Throws error if no events found or array invalid.  
  - *Edge Cases:* Incoming messages without text or with unexpected structure will cause errors.

- **Optimize Prompt (Data Vis)**  
  - *Type:* Google Gemini AI (via LangChain node)  
  - *Role:* Acts as a Data Visualization Specialist AI, transforming user message text into a rich prompt describing the infographic layout, style, and content.  
  - *Configuration:* Uses Gemini 2.0 Flash Lite model. System prompt instructs AI to analyze input data/topic and output a single English prompt describing infographic style and layout (3:4 aspect ratio).  
  - *Input:* Receives extracted user message as input.  
  - *Output:* JSON structured AI response containing candidates or direct content.  
  - *Failure Modes:* API call errors, model unavailability, or unexpected response formats.

- **Parse Gemini Response**  
  - *Type:* Code  
  - *Role:* Parses complex AI responses from Gemini to extract a clean prompt string for Nano Banana Pro.  
  - *Configuration:* JavaScript code handles multiple response formats, cleans text, strips quotes and newlines, converts to single-line prompt text.  
  - *Input:* Raw Gemini AI JSON response.  
  - *Output:* JSON with `prompt`, `userId`, and optional `taskId`.  
  - *Edge Cases:* Handles missing or malformed response parts gracefully; fallback to string conversion.  
  - *Failure Modes:* Parsing failures, unexpected AI output formats.

---

#### 2.2 Infographic Rendering

**Overview:**  
This block submits the prompt to the Nano Banana Pro API to generate the infographic image and uses a wait/poll loop to check for completion before proceeding.

**Nodes Involved:**  
- Submit to Nano Banana Pro  
- Wait  
- Check Job Status  
- Is Ready?  
- Parse Result

**Node Details:**

- **Submit to Nano Banana Pro**  
  - *Type:* HTTP Request  
  - *Role:* Sends the finalized prompt to Nano Banana Pro’s job creation API to start infographic rendering.  
  - *Configuration:* POST request to `https://api.kie.ai/api/v1/jobs/createTask` with JSON body including model `nano-banana-pro`, input prompt, aspect ratio `3:4`, resolution `2K`, and output format `png`.  
  - *Authentication:* Uses generic HTTP header authentication (likely API key).  
  - *Input:* Receives JSON with `prompt` from Parse Gemini Response.  
  - *Output:* Response containing `taskId` and other job details.  
  - *Failure Modes:* HTTP errors, authentication failures, invalid prompt format.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Pauses the workflow to avoid rapid polling. Triggered after job submission and when job is not ready.  
  - *Configuration:* Default wait time (assumed several seconds, exact duration not specified here).  
  - *Input/Output:* Loops between Wait and Check Job Status nodes.  
  - *Failure Modes:* Unintended infinite loops if job never completes.

- **Check Job Status**  
  - *Type:* HTTP Request  
  - *Role:* Polls Nano Banana Pro API for job status to confirm if rendering is complete.  
  - *Configuration:* GET request to `https://api.kie.ai/api/v1/jobs/recordInfo` with query parameters `taskId` and `recordId` extracted from previous job response.  
  - *Authentication:* Same generic HTTP header auth as submission.  
  - *Input:* Receives task and record IDs from Wait node.  
  - *Output:* Job status JSON including `state` (e.g., `success`).  
  - *Failure Modes:* HTTP errors, task not found, expired job IDs.

- **Is Ready?**  
  - *Type:* If  
  - *Role:* Checks if the job status `state` equals `"success"`.  
  - *Configuration:* Condition comparing `{{$json.data.state}} === "success"`.  
  - *Input:* Job status JSON.  
  - *Output:* On true – proceeds to Parse Result; on false – loops back to Wait node.  
  - *Failure Modes:* Missing or unexpected job state values.

- **Parse Result**  
  - *Type:* Code  
  - *Role:* Parses the successful job JSON to extract the generated infographic image URL.  
  - *Configuration:* JavaScript code parses `resultJson` field, extracts first URL in `resultUrls` array.  
  - *Input:* Job success JSON.  
  - *Output:* JSON with `imageUrl`, `taskId`, and status `completed`.  
  - *Failure Modes:* JSON parse errors, missing URLs, empty results.

---

#### 2.3 Host & Deliver

**Overview:**  
This block downloads the generated infographic image, uploads it to an AWS S3 bucket with public access, and replies back to the LINE user with the hosted image URL.

**Nodes Involved:**  
- Download Image  
- Upload to S3  
- Send Image to LINE

**Node Details:**

- **Download Image**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the infographic PNG file from the image URL provided by Nano Banana Pro.  
  - *Configuration:* GET request with response format set to file download.  
  - *Input:* Receives `imageUrl` from Parse Result node.  
  - *Output:* Binary file data representing the PNG image.  
  - *Failure Modes:* HTTP errors, file not found, network issues.

- **Upload to S3**  
  - *Type:* AWS S3  
  - *Role:* Uploads the downloaded PNG image to an S3 bucket for public access.  
  - *Configuration:* Upload operation to bucket `banners-bot-v7158`, filename generated dynamically as `infographic-<timestamp>.png`.  
  - *Input:* Receives binary image file from Download Image node.  
  - *Output:* JSON including S3 `Location` URL of the uploaded file.  
  - *Failure Modes:* AWS credential errors, bucket permissions, upload failures.

- **Send Image to LINE**  
  - *Type:* HTTP Request  
  - *Role:* Sends a reply message to the LINE user with the uploaded infographic image.  
  - *Configuration:* POST request to `https://api.line.me/v2/bot/message/reply` with JSON body containing the `replyToken` from LINE webhook and an image message referencing the S3 public URL for both original and preview images.  
  - *Authentication:* Generic HTTP Header Auth (likely LINE channel access token).  
  - *Input:* Receives reply token from initial webhook and image URL from S3 upload.  
  - *Failure Modes:* LINE API errors, invalid tokens, message formatting errors.

---

### 3. Summary Table

| Node Name              | Node Type                      | Functional Role                        | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                 |
|------------------------|--------------------------------|-------------------------------------|---------------------------|--------------------------|---------------------------------------------------------------------------------------------|
| **LINE Webhook**         | Webhook                        | Entry point for LINE messages       | -                         | Extract Data             |                                                                                             |
| **Extract Data**          | Code                           | Parses LINE event JSON               | LINE Webhook              | Optimize Prompt (Data Vis) | Sticky Note 1: 1. Extract data/topic from LINE. 2. Gemini acts as Data Analyst. 3. Structures data into visual description. |
| **Optimize Prompt (Data Vis)** | Google Gemini AI (LangChain) | Converts message into infographic prompt | Extract Data              | Parse Gemini Response     |                                                                                             |
| **Parse Gemini Response** | Code                           | Cleans and extracts prompt text     | Optimize Prompt (Data Vis) | Submit to Nano Banana Pro  |                                                                                             |
| **Submit to Nano Banana Pro** | HTTP Request                  | Sends prompt to Nano Banana Pro API | Parse Gemini Response     | Wait                     | Sticky Note 2: 1. Submit to Nano Banana Pro. 2. Configured for 3:4 aspect ratio. 3. Smart loop checks every 5s. |
| **Wait**                  | Wait                           | Pauses between polling attempts     | Submit to Nano Banana Pro, Is Ready? | Check Job Status         |                                                                                             |
| **Check Job Status**      | HTTP Request                   | Polls job status from API            | Wait                      | Is Ready?                 |                                                                                             |
| **Is Ready?**             | If                             | Checks if job status is "success"   | Check Job Status           | Parse Result (true), Wait (false) |                                                                                             |
| **Parse Result**          | Code                           | Extracts image URL from job response | Is Ready?                 | Download Image            | Sticky Note 3: 1. Download image. 2. Upload to S3. 3. Send visual report back to LINE.       |
| **Download Image**        | HTTP Request                   | Downloads infographic PNG            | Parse Result               | Upload to S3              |                                                                                             |
| **Upload to S3**          | AWS S3                         | Uploads image to public S3 bucket    | Download Image             | Send Image to LINE        |                                                                                             |
| **Send Image to LINE**    | HTTP Request                   | Sends infographic back to LINE user | Upload to S3               | -                        |                                                                                             |
| **Sticky Note 1**         | Sticky Note                    | Visual note for Data Processing block | -                         | -                        | See Extract Data above                                                                      |
| **Sticky Note 2**         | Sticky Note                    | Visual note for Infographic Rendering | -                         | -                        | See Submit to Nano Banana Pro above                                                        |
| **Sticky Note 3**         | Sticky Note                    | Visual note for Host & Deliver block | -                         | -                        | See Parse Result above                                                                     |
| **Main Description**      | Sticky Note                    | High-level workflow description      | -                         | -                        |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (LINE Webhook)**  
   - Type: Webhook  
   - Set HTTP Method: POST  
   - Set Path: `infographic-v7158`  
   - No authentication or additional options needed here (handle externally if required).  

2. **Create Code Node (Extract Data)**  
   - Connect input from LINE Webhook.  
   - Paste JavaScript code to parse LINE webhook JSON, extract `replyToken`, `userId`, `message.text`, and `timestamp`.  
   - Configure error handling to throw if no events found.

3. **Create Google Gemini AI Node (Optimize Prompt (Data Vis))**  
   - Connect input from Extract Data node.  
   - Select model: `models/gemini-2.0-flash-lite`.  
   - Set system message to instruct AI as Data Visualization Expert, requesting a prompt describing infographic layout/style for Nano Banana Pro in English with 3:4 aspect ratio.  
   - Use user message text from Extract Data as AI input.  
   - Enable JSON output.

4. **Create Code Node (Parse Gemini Response)**  
   - Connect input from Optimize Prompt node.  
   - Paste JavaScript code to parse AI response, extract and clean the prompt string, and output JSON with `prompt`, `userId`, and optional `taskId`.

5. **Create HTTP Request Node (Submit to Nano Banana Pro)**  
   - Connect input from Parse Gemini Response.  
   - Method: POST  
   - URL: `https://api.kie.ai/api/v1/jobs/createTask`  
   - Authentication: Generic HTTP Header Auth (configure with Nano Banana Pro API key).  
   - Headers: `Content-Type: application/json`  
   - Body (JSON):  
     ```json
     {
       "model": "nano-banana-pro",
       "input": {
         "prompt": "{{ $json.prompt }}",
         "aspect_ratio": "3:4",
         "resolution": "2K",
         "output_format": "png"
       }
     }
     ```  
   - Enable sending body and headers.

6. **Create Wait Node (Wait)**  
   - Connect input from Submit to Nano Banana Pro node.  
   - Use default wait time (~5 seconds or configurable).  

7. **Create HTTP Request Node (Check Job Status)**  
   - Connect input from Wait node.  
   - Method: GET  
   - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`  
   - Authentication: Same as Submit node.  
   - Query Parameters:  
     - `taskId` = `={{ $json.data.taskId }}`  
     - `recordId` = `={{ $json.data.recordId }}`  
   - Leave headers empty or as needed.

8. **Create If Node (Is Ready?)**  
   - Connect input from Check Job Status node.  
   - Condition: Check if `{{$json.data.state}}` equals `"success"`.  
   - True branch connects to Parse Result node.  
   - False branch connects back to Wait node (to continue polling).

9. **Create Code Node (Parse Result)**  
   - Connect input from Is Ready? node’s true branch.  
   - JavaScript code to parse `resultJson` from API response, extract first image URL from `resultUrls`.  
   - Output JSON with `imageUrl`, `taskId`, and status.

10. **Create HTTP Request Node (Download Image)**  
    - Connect input from Parse Result node.  
    - Method: GET  
    - URL: `={{ $json.imageUrl }}`  
    - Set response format to download file (binary).  

11. **Create AWS S3 Node (Upload to S3)**  
    - Connect input from Download Image node.  
    - Operation: Upload  
    - Bucket Name: `banners-bot-v7158`  
    - File Name: `={{ 'infographic-' + Date.now() + '.png' }}`  
    - Configure AWS Credentials with appropriate permissions for upload and public access.  

12. **Create HTTP Request Node (Send Image to LINE)**  
    - Connect input from Upload to S3 node.  
    - Method: POST  
    - URL: `https://api.line.me/v2/bot/message/reply`  
    - Authentication: Generic HTTP Header Auth (LINE channel access token).  
    - Headers: `Content-Type: application/json`  
    - Body (JSON):  
      ```json
      {
        "replyToken": "{{ $('LINE Webhook').item.json.replyToken }}",
        "messages": [
          {
            "type": "image",
            "originalContentUrl": "{{ $('Upload to S3').item.json.Location }}",
            "previewImageUrl": "{{ $('Upload to S3').item.json.Location }}"
          }
        ]
      }
      ```  

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                                                |
|------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| The infographic format is fixed to a 3:4 vertical aspect ratio suitable for posters. | Sticky Note 2 details this configuration for Nano Banana Pro rendering.                                                       |
| The AI role "Gemini" acts as a Data Visualization Specialist creating detailed prompts. | Sticky Note 1 explains this role and the data extraction process from LINE messages.                                           |
| The workflow uses a smart polling loop with a 5-second wait interval to check job completion. | Sticky Note 2 and node connections describe this loop between Wait, Check Job Status, and Is Ready? nodes.                      |
| The final infographic is publicly hosted on AWS S3 for easy sharing back to LINE users. | Sticky Note 3 and Upload to S3 node configuration confirm public bucket usage and sharing mechanism.                           |
| To use the Google Gemini node, proper API credentials and LangChain integration must be configured in n8n. | Google Gemini integration requires setup in n8n environment variables or credentials.                                          |
| LINE webhook requires proper channel setup and security validation external to this workflow. | LINE API documentation: https://developers.line.biz/en/docs/messaging-api/overview/                                            |
| Nano Banana Pro API requires API key authentication and adherence to rate limits. | API docs at https://kie.ai (assumed from URLs) should be referenced for credentials and usage policies.                         |

---

**Disclaimer:**  
The text provided stems exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.