Generate LinkedIn Posts with Gemini Content & Imagen Images for Instant Publishing

https://n8nworkflows.xyz/workflows/generate-linkedin-posts-with-gemini-content---imagen-images-for-instant-publishing-6772


# Generate LinkedIn Posts with Gemini Content & Imagen Images for Instant Publishing

---

### 1. Workflow Overview

This workflow automates the generation and instant publishing of professional LinkedIn posts based on user-submitted topics. It orchestrates AI-powered content creation, image generation, and LinkedIn API integration to fully automate LinkedIn post creation with accompanying images.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception:** Captures user input via a form submission.
- **1.2 Input Mapping:** Extracts and formats the user input for AI processing.
- **1.3 AI Content & Image Prompt Generation:** Uses Google Gemini AI to generate LinkedIn post text and an image prompt.
- **1.4 Output Normalization:** Formats and parses the AI output JSON into usable post content and image prompt.
- **1.5 Image Generation:** Sends the image prompt to Google’s Imagen API to generate a professional image.
- **1.6 Image Upload Preparation:** Registers and uploads the generated image to LinkedIn.
- **1.7 Content Curation:** Converts AI-generated Markdown text into LinkedIn-compatible formatting.
- **1.8 Post Publishing:** Publishes the post with image to LinkedIn.
- **1.9 Controlled Execution Timing:** Introduces wait time to ensure image upload completes before publishing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for user submissions of topics via a web form to trigger the workflow.
- **Nodes Involved:**  
  - On form submission  
  - Sticky Note1 (comment)

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point, captures user-submitted topic from a form titled "LinkedIn Post Generator".  
    - Config: Single required field named "Topic" with placeholder text.  
    - Input: HTTP form submission webhook.  
    - Output: JSON containing the submitted topic.  
    - Failure Modes: Missing required field, webhook errors, invalid input format.  
    - Notes: Starts the workflow on user submission.

---

#### 1.2 Input Mapping

- **Overview:** Maps the raw user input into a variable (`chatInput`) for AI processing.
- **Nodes Involved:**  
  - Mapper  
  - Sticky Note2

- **Node Details:**

  - **Mapper**  
    - Type: Set node  
    - Role: Extracts "Topic" from form JSON and assigns it to `chatInput` for downstream nodes.  
    - Config: Assigns `={{ $json.Topic }}` to `chatInput`.  
    - Input: Output from form submission node.  
    - Output: JSON with `chatInput`.  
    - Failure Modes: Missing or malformed input, expression evaluation errors.

---

#### 1.3 AI Content & Image Prompt Generation

- **Overview:** Uses a Google Gemini AI model to generate a professional LinkedIn post text and a detailed image prompt based on the topic.
- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model (credentialed for Google PaLM API)  
  - Sticky Note3

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent node with Google Gemini model integration  
    - Role: Sends system prompt and user input to generate structured JSON output containing post content and an image generation prompt.  
    - Config: Complex system message template instructing Gemini to produce:  
      - Engaging LinkedIn post text with hooks, insights, CTAs, and hashtags.  
      - A professional, minimalist image prompt aligned with LinkedIn’s corporate tone.  
    - Input: `chatInput` from Mapper node.  
    - Output: Raw AI-generated JSON string with fields: `post_content.text` and `image_prompt.description`.  
    - Credentials: Google Gemini API credentials (Google PaLM API).  
    - Failure Modes: API quota limits, model unavailability, malformed JSON output, network errors.  
    - Notes: Requires Google PaLM API access; Gemini model region restrictions may apply (see general notes).

---

#### 1.4 Output Normalization

- **Overview:** Parses and cleans the raw AI-generated JSON output into a structured JSON object with separate post text and image prompt fields.
- **Nodes Involved:**  
  - Normalizer  
  - Sticky Note4

- **Node Details:**

  - **Normalizer**  
    - Type: Code Node (JavaScript)  
    - Role: Removes markdown code block syntax from AI output, parses the JSON, and extracts `post_content.text` and `image_prompt.description`.  
    - Input: Raw JSON string from AI Agent node.  
    - Output: JSON with two fields: `post_content` (string) and `image_prompt` (string).  
    - Failure Modes: JSON parse errors if AI output malformed, missing fields, runtime JS errors.

---

#### 1.5 Image Generation

- **Overview:** Sends the extracted image prompt to Google’s Imagen API to generate a professional image for the LinkedIn post.
- **Nodes Involved:**  
  - Text to image1 (HTTP Request)  
  - Sticky Note5

- **Node Details:**

  - **Text to image1**  
    - Type: HTTP Request  
    - Role: Calls Google Imagen API with prompt extracted from Normalizer node.  
    - Config: POST request with JSON body containing prompt, aspect ratio 1:1, sample count 1, watermark enabled, and content safety settings.  
    - Input: `image_prompt` from Normalizer node.  
    - Output: Response includes base64-encoded image bytes.  
    - Credentials: Google Cloud authentication (OAuth2 or API token).  
    - Failure Modes: API quota, authentication failure, invalid prompt, latency, region restrictions.

---

#### 1.6 Image Upload Preparation

- **Overview:** Registers the image upload with LinkedIn, converts the image to binary buffer, then uploads it to LinkedIn’s media server.
- **Nodes Involved:**  
  - Register Image Upload (HTTP Request)  
  - Convert to binary buffer (Code Node)  
  - Upload Image to LinkedIn1 (HTTP Request)  
  - Sticky Notes8, 9, 11

- **Node Details:**

  - **Register Image Upload**  
    - Type: HTTP Request  
    - Role: Registers an image upload session with LinkedIn API, requests upload URL and asset ID.  
    - Config: POST JSON specifying owner urn, upload recipe, and upload mechanism.  
    - Input: None (triggered after image generation).  
    - Output: Upload URL and asset URN for image.  
    - Credentials: LinkedIn OAuth2 and Bearer Token.  
    - Failure Modes: Auth errors, invalid owner URN, API rate limits.

  - **Convert to binary buffer**  
    - Type: Code Node  
    - Role: Decodes base64 image from Imagen API response into binary buffer for upload; attaches upload URL and asset URN for next node.  
    - Input: Base64 image from Imagen, upload details from Register Image Upload.  
    - Output: Binary buffer data with upload metadata.  
    - Failure Modes: Missing or malformed base64, buffer conversion errors.

  - **Upload Image to LinkedIn1**  
    - Type: HTTP Request (PUT)  
    - Role: Uploads binary image data to the LinkedIn upload URL.  
    - Config: PUT request with binary data, content-type image/png, authorization header.  
    - Input: Binary image buffer from previous node.  
    - Output: HTTP response confirming upload success.  
    - Credentials: LinkedIn OAuth2 and Bearer Token.  
    - Failure Modes: Upload failure, token expiration, network errors.

---

#### 1.7 Content Curation

- **Overview:** Converts the AI-generated Markdown LinkedIn post text into LinkedIn-compatible plain text format, handling bold, italics, lists, line breaks, headers, links, and hashtags.
- **Nodes Involved:**  
  - Code (Markdown to LinkedIn format)  
  - Sticky Note10

- **Node Details:**

  - **Code**  
    - Type: Code Node (JavaScript)  
    - Role: Processes the Markdown post content to remove unsupported Markdown syntax and transform it into clean text with LinkedIn formatting conventions, including:  
      - Removing Markdown bold/italic syntax  
      - Converting numbered/bullet lists to bullet points  
      - Handling line breaks and headers  
      - Formatting hashtags to appear at the end, separated by line breaks  
      - Removing code blocks syntax  
    - Input: `post_content` from Convert to binary buffer node.  
    - Output: JSON with `linkedin_content` field containing LinkedIn-ready text.  
    - Failure Modes: Input missing or malformed, regex parsing errors.

---

#### 1.8 Controlled Execution Timing

- **Overview:** Introduces a deliberate wait of 60 seconds to ensure the image upload is fully processed by LinkedIn before publishing the post.
- **Nodes Involved:**  
  - Wait  
  - Sticky Note11

- **Node Details:**

  - **Wait**  
    - Type: Wait node  
    - Role: Pauses workflow execution for 60 seconds.  
    - Config: Fixed 60 seconds delay.  
    - Input: Triggered after image upload.  
    - Output: Passes through after wait.  
    - Failure Modes: Timeout or halt if workflow interrupted.

---

#### 1.9 Post Publishing

- **Overview:** Publishes the curated LinkedIn post with the uploaded image to the user’s LinkedIn feed.
- **Nodes Involved:**  
  - HTTP Request (LinkedIn post)  
  - Sticky Note7

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request (POST)  
    - Role: Calls LinkedIn's UGC Posts API to create a public post with text content and attached image asset.  
    - Config:  
      - POST to https://api.linkedin.com/v2/ugcPosts  
      - Body includes author URN, lifecycle state, share content with text and media (image asset URN), visibility set to public.  
      - Headers: Content-Type application/json, X-Restli-Protocol-Version 2.0.0, Bearer token auth.  
    - Input: `linkedin_content` from Code node and `asset` URN from upload nodes.  
    - Credentials: LinkedIn OAuth2 and Bearer Token.  
    - Failure Modes: Auth failure, invalid asset URN, API rate limits, network errors.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                   | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                              |
|-------------------------|----------------------------------|-------------------------------------------------|--------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger                     | Captures user topic input from form submission  | -                        | Mapper                     | **On Form Submission** Triggered when a user submits a topic through the form.                          |
| Mapper                  | Set                             | Maps input topic to variable `chatInput`        | On form submission       | AI Agent                   | **Mapper** Maps the user-submitted topic for AI processing.                                             |
| AI Agent                | LangChain Agent (Google Gemini) | Generates LinkedIn post text and image prompt   | Mapper                   | Normalizer                 | **AI Agent** Uses Google Gemini to generate professional content and image prompt.                      |
| Normalizer              | Code                            | Parses and cleans AI JSON output                  | AI Agent                 | Text to image1             | **Normalizer** Cleans & formats AI output into usable content and prompt.                              |
| Text to image1          | HTTP Request                    | Calls Google Imagen API to generate image        | Normalizer               | Register Image Upload      | **Text to Image** Sends prompt to Google Imagen for image generation.                                  |
| Register Image Upload   | HTTP Request                    | Registers image upload session with LinkedIn     | Text to image1           | Convert to binary buffer   | **Register Image on Linkedin** Registers image upload with LinkedIn API.                              |
| Convert to binary buffer| Code                            | Converts base64 image to binary buffer            | Register Image Upload    | Upload Image to LinkedIn1  | **Decoder** Decodes image to binary buffer for upload.                                                |
| Upload Image to LinkedIn1| HTTP Request                   | Uploads binary image to LinkedIn                   | Convert to binary buffer | Code                       | **Upload Binary buffer** Uploads image to LinkedIn.                                                   |
| Code                    | Code                            | Converts Markdown text to LinkedIn format         | Upload Image to LinkedIn1| Wait                       | **Curate content** Converts AI Markdown output to LinkedIn-compatible text format.                    |
| Wait                    | Wait                            | Waits 60 seconds before publishing post           | Code                     | HTTP Request (LinkedIn)    | **Wait for uploading the image** Wait 30-60 seconds between upload and post creation.                 |
| HTTP Request            | HTTP Request                    | Publishes LinkedIn post with text and image       | Wait                     | -                          | **LinkedIn** Publishes generated post and image to LinkedIn profile.                                  |
| Google Gemini Chat Model| Language Model                  | (Credential node for AI Agent)                     | -                        | AI Agent                   |                                                                                                        |
| Sticky Note (multiple)  | Sticky Note                    | Provides descriptive comments on workflow parts   | -                        | -                          | Various detailed comments with links to Google Imagen API and LinkedIn API docs.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: On form submission**  
   - Type: Form Trigger  
   - Form Title: "LinkedIn Post Generator"  
   - Form Fields: One required field labeled "Topic" with placeholder "Enter prompt here...."  
   - Position: (512,304)

2. **Create Mapper Node**  
   - Type: Set  
   - Assign variable `chatInput` with expression `={{ $json.Topic }}`  
   - Connect output of On form submission to Mapper input  
   - Position: (768,304)

3. **Create AI Agent Node**  
   - Type: LangChain Agent  
   - Set system message as provided, including detailed instructions to generate LinkedIn post content and image prompt in JSON format, using `{{ $json.chatInput }}` as topic.  
   - Connect Mapper output to AI Agent input  
   - Position: (1008,304)

4. **Configure Google Gemini Chat Model Credentials**  
   - Add Google PaLM API credentials under name "Google Gemini(PaLM) Api account"  
   - Link credentials to AI Agent node's language model setting

5. **Create Normalizer Node**  
   - Type: Code (JavaScript)  
   - Add code to remove markdown code block markers, parse JSON, and extract `post_content.text` and `image_prompt.description` into JSON output keys `post_content` and `image_prompt`  
   - Connect AI Agent output to Normalizer input  
   - Position: (1392,304)

6. **Create Text to image1 Node**  
   - Type: HTTP Request  
   - URL: `https://us-central1-aiplatform.googleapis.com/v1/projects/absolute-hub-464011-c1/locations/us-central1/publishers/google/models/imagen-4.0-generate-preview-06-06:predict`  
   - Method: POST  
   - Body: JSON with prompt as `{{ $json.image_prompt }}`, aspectRatio 1:1, sampleCount 1, watermark enabled, safetySetting block_few, etc.  
   - Authentication: Generic HTTP Bearer token (Google Cloud token)  
   - Connect Normalizer output to this node  
   - Position: (1664,304)

7. **Create Register Image Upload Node**  
   - Type: HTTP Request  
   - URL: `https://api.linkedin.com/v2/assets?action=registerUpload`  
   - Method: POST  
   - Body: JSON registering upload with owner urn `urn:li:person:<YourLinkedInPersonURN>`, recipe for feedshare-image, synchronous upload mechanism  
   - Authentication: LinkedIn OAuth2 and Bearer token  
   - Connect Text to image1 output to this node  
   - Position: (1936,304)

8. **Create Convert to binary buffer Node**  
   - Type: Code  
   - Add JS code to extract base64 image from Text to image1 response, convert to binary buffer, and attach upload URL and asset URN from Register Image Upload response  
   - Connect Register Image Upload output to this node  
   - Position: (2224,304)

9. **Create Upload Image to LinkedIn1 Node**  
   - Type: HTTP Request (PUT)  
   - URL: `={{ $('Register Image Upload').item.json.value.uploadMechanism['com.linkedin.digitalmedia.uploading.MediaUploadHttpRequest'].uploadUrl }}`  
   - Method: PUT  
   - Headers: Authorization Bearer `<LinkedIn Token>`, Content-Type image/png  
   - Send binary data from previous node as request body  
   - Connect Convert to binary buffer output to this node  
   - Position: (2496,304)

10. **Create Code Node for Content Curation**  
    - Type: Code  
    - Add JS code to convert Markdown LinkedIn post text into LinkedIn-compatible format, handling bold, italics, lists, line breaks, headers, links, and hashtags as described  
    - Connect Upload Image to LinkedIn1 output to this node  
    - Position: (2768,304)

11. **Create Wait Node**  
    - Type: Wait  
    - Duration: 60 seconds  
    - Connect Code node output to Wait node input  
    - Position: (3040,304)

12. **Create HTTP Request Node for Post Publishing**  
    - Type: HTTP Request (POST)  
    - URL: `https://api.linkedin.com/v2/ugcPosts`  
    - Body: JSON including:  
      - author: `urn:li:person:<YourLinkedInPersonURN>`  
      - lifecycleState: PUBLISHED  
      - specificContent with shareCommentary text from curated content, media array with registered asset URN, status READY  
      - visibility: public  
    - Headers: X-Restli-Protocol-Version 2.0.0, Content-Type application/json, Authorization Bearer `<LinkedIn Token>`  
    - Connect Wait node output to this node  
    - Position: (3328,304)

13. **Credential Setup:**  
    - Google PaLM API credential for Google Gemini model  
    - Google Cloud authentication token for Imagen API  
    - LinkedIn OAuth2 and Bearer Token credentials for API access

14. **Connect All Nodes Following the Logical Flow:**  
    On form submission → Mapper → AI Agent → Normalizer → Text to image1 → Register Image Upload → Convert to binary buffer → Upload Image to LinkedIn1 → Code (curate content) → Wait → HTTP Request (post to LinkedIn)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| **AI-Powered LinkedIn Post Automation:** Automates professional LinkedIn posts creation with AI-generated content and images, facilitating instant publishing.                                                                                                                                                                                                                                                                                                                               | Workflow overview sticky note at workflow start                                                                                                   |
| Google Gemini Image Generation model has geo restrictions; model availability may vary by country causing "model not found" errors.                                                                                                                                                                                                                                                                                                                                                              | Sticky Note12; important operational consideration                                                                                               |
| Google Imagen API for text-to-image generation is used with corporate tone and style guidelines to create professional LinkedIn post images.                                                                                                                                                                                                                                                                                                                                                    | See Sticky Note5 and [Google Imagen API Reference](https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/imagen-api?hl=en)        |
| LinkedIn API usage includes image registration, upload, and post publishing following LinkedIn's UGC Posts API specification.                                                                                                                                                                                                                                                                                                                                                                   | See Sticky Notes8, 9, 11 and [LinkedIn Images API documentation](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/images-api?view=li-lms-2025-07&tabs=http) |
| Markdown content from AI is converted to LinkedIn-compatible plain text due to LinkedIn's limited Markdown support; this includes handling bold, italics, lists, line breaks, headers, and hashtags properly.                                                                                                                                                                                                                                                                                    | See Code node comments for detailed processing logic                                                                                              |
| Wait node is essential to avoid race conditions between image upload completion and post publishing on LinkedIn.                                                                                                                                                                                                                                                                                                                                                                               | See Sticky Note11                                                                                                                                |

---

Disclaimer: The text provided derives exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.

---