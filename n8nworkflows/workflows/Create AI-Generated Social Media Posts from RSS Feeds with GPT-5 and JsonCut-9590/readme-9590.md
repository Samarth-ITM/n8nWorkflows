Create AI-Generated Social Media Posts from RSS Feeds with GPT-5 and JsonCut

https://n8nworkflows.xyz/workflows/create-ai-generated-social-media-posts-from-rss-feeds-with-gpt-5-and-jsoncut-9590


# Create AI-Generated Social Media Posts from RSS Feeds with GPT-5 and JsonCut

---
### 1. Workflow Overview

This workflow automates the creation of AI-generated social media posts on Instagram by processing news articles from an RSS feed (BBC World News). It extracts news content, generates optimized Instagram post text and AI image prompts using GPT-5, creates composite images using the JsonCut API, uploads images and text to a social media management platform (Blotato), and finally publishes the post on Instagram.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Content Generation:** Triggered by new RSS feed items, generates Instagram-ready text and image prompts using OpenAI GPT-5.
- **1.2 AI Image Generation and Asset Download:** Generates background images via OpenAI's image API and downloads related resources including a logo.
- **1.3 Uploading Resources to JsonCut:** Uploads background images and logo files to JsonCut for image composition.
- **1.4 JsonCut Job Creation and Monitoring:** Creates a JsonCut job to compose the final image with text overlays and monitors the job status until completion.
- **1.5 Download Final Image and Social Media Posting:** Downloads the composed image, uploads media and generates Instagram post text, then publishes the post on Instagram via Blotato.
- **1.6 Error Handling and Control Flow:** Implements conditional checks and wait/retry logic for job processing and error states.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Content Generation

**Overview:**  
This block listens for new news items from the BBC World News RSS feed and uses OpenAI GPT-5 to generate optimized Instagram headlines and image prompts based on the news article content.

**Nodes Involved:**  
- RSS Feed Trigger  
- Generate content and Image Prompt

**Node Details:**

- **RSS Feed Trigger**  
  - Type: RSS Feed Read Trigger  
  - Role: Initiates the workflow on new RSS items every minute.  
  - Configuration: Feed URL set to `https://feeds.bbci.co.uk/news/world/rss.xml`; polling every minute.  
  - Connections: Output to "Generate content and Image Prompt".  
  - Edge cases: Feed downtime, no new items, malformed feed.

- **Generate content and Image Prompt**  
  - Type: OpenAI GPT-5 Mini (Langchain node)  
  - Role: Generates a JSON object with a rewritten headline and an AI image prompt for Instagram image background.  
  - Configuration: Model GPT-5 Mini; message instructs to rewrite headline (max 100 chars/12 words), create a descriptive image prompt without text. Output as JSON.  
  - Inputs: Receives article title and content from RSS.  
  - Outputs: JSON with "headline" and "image_prompt".  
  - Edge cases: API rate limits, invalid JSON output, AI hallucinations.

---

#### 2.2 AI Image Generation and Asset Download

**Overview:**  
Generates background images based on AI prompts, downloads the images and a logo for later composition.

**Nodes Involved:**  
- Generate Background Image  
- Download Background  
- Download Logo

**Node Details:**

- **Generate Background Image**  
  - Type: OpenAI Image Generation (Langchain node)  
  - Role: Generates a 1024x1024 image from the prompt created in the previous block.  
  - Configuration: Uses image_prompt from previous node; requests 1024x1024 size image URLs.  
  - Outputs: URLs of generated images.  
  - Edge cases: API failures, image generation delays, invalid URLs.

- **Download Background**  
  - Type: HTTP Request  
  - Role: Downloads the generated background image from the URL provided by OpenAI.  
  - Configuration: URL from the previous node's JSON (field `url`).  
  - Edge cases: HTTP errors, timeouts, invalid URLs.

- **Download Logo**  
  - Type: HTTP Request  
  - Role: Downloads a fixed logo image (from icons8).  
  - Configuration: Static URL for a black PNG icon.  
  - Edge cases: HTTP errors, network issues.

---

#### 2.3 Uploading Resources to JsonCut

**Overview:**  
Uploads the downloaded background image and logo to JsonCut service to prepare for image composition.

**Nodes Involved:**  
- Upload Background  
- Upload logo  
- Merge

**Node Details:**

- **Upload Background**  
  - Type: HTTP Request  
  - Role: Uploads the background image file to JsonCut.  
  - Configuration: POST multipart/form-data to JsonCut's upload endpoint with binary data; authenticated with JsonCut API key.  
  - Output: Upload response includes storage URLs.  
  - Edge cases: Authentication errors, upload failures.

- **Upload logo**  
  - Type: HTTP Request  
  - Role: Uploads the logo image file to JsonCut similarly.  
  - Configuration: Same as Upload Background but with logo file data.  
  - Edge cases: Same as Upload Background.

- **Merge**  
  - Type: Merge node  
  - Role: Combines upload results of background and logo into a single data structure for subsequent steps.  
  - Configuration: Merges main outputs from Upload Background and Upload logo into one array.  
  - Edge cases: Mismatched data, incomplete merges.

---

#### 2.4 JsonCut Job Creation and Monitoring

**Overview:**  
Creates a JsonCut job to compose the Instagram post image with multiple layers (background, logo, text) and polls for job completion.

**Nodes Involved:**  
- Aggregate  
- Create JsonCut Job  
- Wait  
- Check JsonCut job Status  
- If Success  
- If Error  
- Download Image  
- Error Stop

**Node Details:**

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Prepares data fields (`storageUrl`) collected from previous uploads for JsonCut job creation.  
  - Edge cases: Missing fields.

- **Create JsonCut Job**  
  - Type: HTTP Request  
  - Role: Posts a job creation request to JsonCut API with a detailed config specifying how to compose the image.  
  - Configuration: Uses uploaded storage URLs for images; configures layers (blurred background, rectangles, logo, headline text with Roboto font, source text); output format PNG.  
  - Authenticated with JsonCut API key.  
  - Edge cases: API errors, malformed config JSON.

- **Wait**  
  - Type: Wait node  
  - Role: Pauses the workflow 3 seconds before polling job status to allow processing time.  
  - Edge cases: Too short/long wait times may cause premature or delayed polling.

- **Check JsonCut job Status**  
  - Type: HTTP Request  
  - Role: Queries JsonCut for current job status using jobId from creation response.  
  - Edge cases: Network errors, job not found.

- **If Success**  
  - Type: If node  
  - Role: Checks if job status is "COMPLETED" to proceed with image download.  
  - Edge cases: False positives/negatives if status field missing or changed format.

- **If Error**  
  - Type: If node  
  - Role: Checks if job status is "FAILED" or "CANCELLED" to stop workflow or retry.  
  - Edge cases: Handling unknown statuses.

- **Download Image**  
  - Type: HTTP Request  
  - Role: Downloads the final composed image from JsonCut after job completion.  
  - Edge cases: Download failures.

- **Error Stop**  
  - Type: Stop and Error node  
  - Role: Halts execution with an error message if image generation fails.  
  - Edge cases: Used only on failure path.

---

#### 2.5 Download Final Image and Social Media Posting

**Overview:**  
Uploads the final image to Blotato, generates Instagram post text with GPT-5, and publishes the post on Instagram.

**Nodes Involved:**  
- Upload media  
- Create Instagram Text  
- Create Instagram post

**Node Details:**

- **Upload media**  
  - Type: Blotato media upload node  
  - Role: Uploads the final image binary to Blotato media storage.  
  - Inputs: Binary data from "Download Image".  
  - Edge cases: API rate limits, upload failures.

- **Create Instagram Text**  
  - Type: OpenAI GPT-5 Mini (Langchain node)  
  - Role: Generates a compelling Instagram post caption based on the original headline and short description from the RSS feed.  
  - Configuration: Instruction to create engaging, rephrased text with hook, explanation, and call to action. Output as JSON with one field "post_text".  
  - Edge cases: API errors, invalid JSON.

- **Create Instagram post**  
  - Type: Blotato post creation node  
  - Role: Creates an Instagram post on the connected account using the generated text and uploaded media URL.  
  - Configuration: Account ID is fixed; post text and media URL come from previous nodes.  
  - Edge cases: Authentication, posting limits.

---

#### 2.6 Error Handling and Control Flow

**Overview:**  
Manages workflow execution flow including wait/retry for JsonCut job status, branching on success or failure, and stopping on errors.

**Nodes Involved:**  
- Wait  
- If Success  
- If Error  
- Error Stop

**Node Details:**

- **Wait**  
  - Used to delay status polling to avoid premature checks.

- **If Success & If Error**  
  - Condition checks on job status to direct flow.

- **Error Stop**  
  - Stops workflow with error message on failures.

- Edge cases: Infinite loops if retry logic not limited; unhandled status codes.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                  | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                          |
|-----------------------------|----------------------------------|-------------------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| RSS Feed Trigger            | RSS Feed Read Trigger             | Triggers workflow on new BBC news RSS items     | -                                | Generate content and Image Prompt| ### Create Job with Jsoncut API and wait for the result. Alternatively, the JsonCut Community node can also be used (https://github.com/jsoncut/n8n-nodes-jsoncut). |
| Generate content and Image Prompt | OpenAI GPT-5 Mini (Langchain)      | Generates Instagram headline and image prompt   | RSS Feed Trigger                 | Generate Background Image        | ### Generate Text and Background with AI                                                           |
| Generate Background Image   | OpenAI Image Generation (Langchain) | Creates AI-generated background image            | Generate content and Image Prompt| Download Background, Download Logo|                                                                                                    |
| Download Background         | HTTP Request                     | Downloads background image from OpenAI URL      | Generate Background Image        | Upload Background               | ### Download ressources                                                                             |
| Download Logo               | HTTP Request                     | Downloads logo image                             | Generate Background Image        | Upload logo                    | ### Download ressources                                                                             |
| Upload Background           | HTTP Request                     | Uploads background image to JsonCut             | Download Background              | Merge                         | ### Upload Files to JsonCut API                                                                     |
| Upload logo                 | HTTP Request                     | Uploads logo image to JsonCut                    | Download Logo                   | Merge                         | ### Upload Files to JsonCut API                                                                     |
| Merge                      | Merge                           | Combines upload responses for JsonCut job config| Upload Background, Upload logo  | Aggregate                     |                                                                                                    |
| Aggregate                  | Aggregate                      | Prepares storage URLs array for job creation    | Merge                          | Create JsonCut Job             |                                                                                                    |
| Create JsonCut Job          | HTTP Request                     | Creates image composition job on JsonCut API    | Aggregate                      | Wait                          |                                                                                                    |
| Wait                       | Wait                           | Waits 3 seconds before polling job status       | Create JsonCut Job             | Check JsonCut job Status        |                                                                                                    |
| Check JsonCut job Status    | HTTP Request                     | Polls JsonCut API for job status                 | Wait                          | If Success                    |                                                                                                    |
| If Success                 | If                             | Checks if job status is COMPLETED                | Check JsonCut job Status       | Download Image, If Error         |                                                                                                    |
| If Error                   | If                             | Checks if job status is FAILED or CANCELLED      | If Success                    | Error Stop, Wait                |                                                                                                    |
| Download Image             | HTTP Request                     | Downloads final composed image                    | If Success                    | Upload media                   |                                                                                                    |
| Error Stop                 | Stop and Error                 | Stops workflow with error message on failure     | If Error                     | -                             |                                                                                                    |
| Upload media               | Blotato media upload            | Uploads final image to Blotato media storage     | Download Image                | Create Instagram Text          |                                                                                                    |
| Create Instagram Text      | OpenAI GPT-5 Mini (Langchain)   | Generates Instagram post caption text            | Upload media                 | Create Instagram post          |                                                                                                    |
| Create Instagram post      | Blotato post creation           | Creates Instagram post with media and caption   | Create Instagram Text         | -                             | ### Post on Instagram Additional social media channels can be added here.                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger Node**  
   - Type: RSS Feed Read Trigger  
   - Parameters: Feed URL `https://feeds.bbci.co.uk/news/world/rss.xml`  
   - Poll interval: every minute  
   - Connect output to "Generate content and Image Prompt".

2. **Create OpenAI GPT-5 Node "Generate content and Image Prompt"**  
   - Model: GPT-5 Mini  
   - Input: JSON from RSS feed (title, content)  
   - Prompt: Instruct to rewrite headline (max 100 chars/12 words), generate descriptive image prompt without text, output JSON with fields "headline" and "image_prompt".  
   - Connect output to "Generate Background Image".

3. **Create OpenAI Image Generation Node "Generate Background Image"**  
   - Input prompt: from previous node's JSON field `message.content.image_prompt`  
   - Image size: 1024x1024  
   - Output: image URLs  
   - Connect to "Download Background" and "Download Logo".

4. **Create HTTP Request "Download Background"**  
   - Method: GET  
   - URL: from `url` field of previous node's output  
   - Download binary data of the background image  
   - Connect output to "Upload Background".

5. **Create HTTP Request "Download Logo"**  
   - Method: GET  
   - URL: `https://img.icons8.com/?size=100&id=tn7B1mGNMgGC&format=png&color=000000`  
   - Download binary data of the logo image  
   - Connect output to "Upload logo".

6. **Create HTTP Request "Upload Background"**  
   - Method: POST  
   - URL: `https://api.jsoncut.com/api/v1/files/upload`  
   - Content-Type: multipart/form-data  
   - Binary data field: `data` (from Download Background)  
   - Header: Accept: application/json  
   - Auth: HTTP Header Auth (JsonCut API Key)  
   - Connect output to "Merge" (input 0).

7. **Create HTTP Request "Upload logo"**  
   - Same as Upload Background but binary data from Download Logo  
   - Connect output to "Merge" (input 1).

8. **Create Merge Node**  
   - Merge mode: Combine inputs into single array  
   - Inputs: Upload Background and Upload logo outputs  
   - Connect output to "Aggregate".

9. **Create Aggregate Node**  
   - Aggregate field `data.storageUrl` into an array for job config  
   - Connect output to "Create JsonCut Job".

10. **Create HTTP Request "Create JsonCut Job"**  
    - Method: POST  
    - URL: `https://api.jsoncut.com/api/v1/jobs`  
    - Body: JSON specifying image composition layers:
      - Background image (blurred and cover fit) from first storageUrl  
      - Two decorative rectangles in green (#6be3a2)  
      - Logo image (second storageUrl)  
      - Headline text from "Generate content and Image Prompt" node's JSON field  
      - Source text "source: bbc"  
    - Headers: Content-Type: application/json, Accept: application/json  
    - Auth: HTTP Header Auth (JsonCut API Key)  
    - Connect output to "Wait".

11. **Create Wait Node**  
    - Duration: 3 seconds  
    - Connect output to "Check JsonCut job Status".

12. **Create HTTP Request "Check JsonCut job Status"**  
    - Method: GET  
    - URL: `https://api.jsoncut.com/api/v1/jobs/{{ $json.data.jobId }}`  
    - Header: Accept: application/json  
    - Auth: HTTP Header Auth (JsonCut API Key)  
    - Connect output to "If Success".

13. **Create If Node "If Success"**  
    - Condition: `$json.data.status == "COMPLETED"`  
    - True output to "Download Image"  
    - False output to "If Error".

14. **Create If Node "If Error"**  
    - Condition: `$json.data.status == "FAILED"` OR `$json.data.status == "CANCELLED"`  
    - True output to "Error Stop"  
    - False output to "Wait" (retry polling).

15. **Create HTTP Request "Download Image"**  
    - Method: GET  
    - URL: `https://api.jsoncut.com/api/v1/files/{{ $json.data.outputFileId }}/download`  
    - Response format: file binary  
    - Header: Accept: application/octet-stream  
    - Auth: HTTP Header Auth (JsonCut API Key)  
    - Connect output to "Upload media".

16. **Create Blotato Media Upload Node "Upload media"**  
    - Resource: media  
    - Use binary data from "Download Image"  
    - Auth: Blotato API credentials  
    - Connect output to "Create Instagram Text".

17. **Create OpenAI GPT-5 Node "Create Instagram Text"**  
    - Model: GPT-5 Mini  
    - Input: original RSS feed title and content  
    - Prompt: Create engaging Instagram caption different from headline, with hook, context, call to action; output JSON with "post_text" field only.  
    - Connect output to "Create Instagram post".

18. **Create Blotato Post Creation Node "Create Instagram post"**  
    - Account ID: fixed (e.g., "1234")  
    - Post content text: from "Create Instagram Text" output `message.content`  
    - Post media URL: from "Upload media" response `data[0].url`  
    - Auth: Blotato API credentials

19. **Create Stop and Error Node "Error Stop"**  
    - Error message: "Failed to generate image"  
    - Connect from If Error node when job fails.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| JsonCut Community node can be used as an alternative to manual API calls for job creation/status | https://github.com/jsoncut/n8n-nodes-jsoncut                                                   |
| This workflow uses the GPT-5 Mini model via Langchain nodes for text and image generation         | Requires valid OpenAI API credentials configured in n8n                                          |
| Post publishing is done via Blotato social media management nodes                                | Requires Blotato API credentials and configured Instagram account                               |
| Logo image source: icons8                                                                        | https://img.icons8.com/?size=100&id=tn7B1mGNMgGC&format=png&color=000000                         |
| JsonCut API documentation for job and file endpoints                                            | https://api.jsoncut.com/docs                                                                     |

---

**Disclaimer:** All data processed in this workflow are publicly available news articles. The workflow complies with all content policies and does not contain illegal or offensive content.