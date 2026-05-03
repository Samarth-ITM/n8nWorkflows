Generate AI viral videos with NanoBanana & VEO3, shared on socials via Blotato

https://n8nworkflows.xyz/workflows/generate-ai-viral-videos-with-nanobanana---veo3--shared-on-socials-via-blotato-8270


# Generate AI viral videos with NanoBanana & VEO3, shared on socials via Blotato

### 1. Workflow Overview

This workflow automates the creation, enhancement, and multi-platform sharing of AI-generated viral videos by leveraging NanoBanana for image editing, VEO3 for video generation, and Blotato for social media distribution. It is designed for content creators and marketers aiming to produce engaging UGC-style video ads from Telegram-submitted ideas and images, streamline captioning, and publish seamlessly across multiple social networks.

The workflow is logically divided into five functional blocks:

- **1.1 Input Reception:** Collects user-submitted video ideas and images from Telegram.
- **1.2 Image Processing & Prompt Generation:** Analyzes the reference image, generates a refined image prompt, and enhances the image using NanoBanana.
- **1.3 Video Script & Generation:** Creates a structured video script prompt via OpenAI, formats it, and triggers video generation through VEO3.
- **1.4 Caption Refinement & Logging:** Rewrites video captions using GPT-4o and logs all relevant metadata and status updates into Google Sheets.
- **1.5 Multi-Platform Auto-Posting:** Uploads the generated video to Blotato and automatically posts it on various social platforms, followed by status updates and final Telegram notifications.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Receives video idea submissions from Telegram users, retrieves the associated image file, and uploads it to Google Drive while logging the initial data in Google Sheets.

- **Nodes Involved:**  
  - Telegram Trigger: Receive Video Idea  
  - Telegram: Get Image File  
  - Google Drive: Upload Image  
  - Google Sheets: Log Image & Caption  
  - Set: Bot Token (Placeholder)  
  - Telegram API: Get File URL

- **Node Details:**

  1. **Telegram Trigger: Receive Video Idea**  
     - *Type:* Telegram Trigger  
     - *Role:* Entry point; listens for incoming messages with video ideas and images.  
     - *Config:* Listens to all message updates.  
     - *Inputs:* Telegram messages with photo and caption.  
     - *Outputs:* Passes message JSON to downstream nodes.  
     - *Edge Cases:* Missing photo or caption; Telegram API downtime.

  2. **Telegram: Get Image File**  
     - *Type:* Telegram API node  
     - *Role:* Downloads the third-size photo file from Telegram message.  
     - *Config:* Uses photo[2].file_id from the trigger.  
     - *Inputs:* Photo file ID from Telegram trigger.  
     - *Outputs:* Raw file data for upload.  
     - *Edge Cases:* File not found or expired; API rate limits.

  3. **Google Drive: Upload Image**  
     - *Type:* Google Drive node  
     - *Role:* Uploads the downloaded image to a specified Google Drive folder.  
     - *Config:* Uses unique photo file ID as filename; public folder required.  
     - *Inputs:* Image file from Telegram node.  
     - *Outputs:* Provides publicly accessible webContentLink for image.  
     - *Edge Cases:* Permission errors; folder not public.

  4. **Google Sheets: Log Image & Caption**  
     - *Type:* Google Sheets (append/update)  
     - *Role:* Logs image metadata, caption, and initial status "EN COURS" in a Google Sheet.  
     - *Config:* Matches rows by "IMAGE NAME" (file unique ID).  
     - *Inputs:* Image URL, caption, image name from previous nodes.  
     - *Outputs:* Confirms log success.  
     - *Edge Cases:* Sheet permissions; data mismatch.

  5. **Set: Bot Token (Placeholder)**  
     - *Type:* Set node  
     - *Role:* Placeholder for Telegram bot token, used dynamically downstream.  
     - *Config:* Empty token string, must be set in production.  
     - *Inputs:* None from previous nodes, set manually.  
     - *Outputs:* Supplies token for HTTP request.  
     - *Edge Cases:* Missing or invalid bot token breaks Telegram API calls.

  6. **Telegram API: Get File URL**  
     - *Type:* HTTP Request  
     - *Role:* Retrieves the file path URL from Telegram API for the image (fourth photo size).  
     - *Config:* Uses bot token and photo[3].file_id.  
     - *Inputs:* Bot token and photo file ID from Set and Telegram trigger nodes.  
     - *Outputs:* File path URL used for image analysis.  
     - *Edge Cases:* Telegram API errors; invalid token.

---

#### 1.2 Image Processing & Prompt Generation

- **Overview:**  
  Uses OpenAI Vision to analyze the uploaded image, updates the Google Sheet with descriptions, generates a natural UGC-style image prompt, and edits the image with NanoBanana AI.

- **Nodes Involved:**  
  - OpenAI Vision: Analyze Reference Image  
  - Google Sheets: Update Image Description  
  - Generate Image Prompt (Langchain Agent)  
  - NanoBanana: Create Image  
  - Wait for Image Edit  
  - Download Edited Image  
  - Google Sheets: Read Video Parameters (CONFIG)  
  - Set Master Prompt

- **Node Details:**

  1. **OpenAI Vision: Analyze Reference Image**  
     - *Type:* OpenAI Vision API via Langchain node  
     - *Role:* Analyzes the image and outputs YAML-formatted descriptive metadata for prompt generation.  
     - *Config:* Uses image URL from Telegram API getFile call.  
     - *Inputs:* Public image URL.  
     - *Outputs:* YAML description of image content (product/character).  
     - *Edge Cases:* Image URL invalid; API limits.

  2. **Google Sheets: Update Image Description**  
     - *Type:* Google Sheets append/update  
     - *Role:* Updates the image description field in the Google Sheet with the YAML output.  
     - *Config:* Matches by "IMAGE NAME".  
     - *Inputs:* YAML content from OpenAI Vision and IMAGE NAME.  
     - *Outputs:* Confirmation of update.  
     - *Edge Cases:* Sheet write access issues.

  3. **Generate Image Prompt**  
     - *Type:* Langchain Agent  
     - *Role:* Generates a concise, structured UGC-style image prompt JSON based on user caption and image description.  
     - *Config:* System message enforces styling rules (casual, realistic, handheld camera cues).  
     - *Inputs:* Caption and image description from Google Sheets.  
     - *Outputs:* JSON with single key `image_prompt`.  
     - *Edge Cases:* Output parsing failures; invalid JSON.

  4. **NanoBanana: Create Image**  
     - *Type:* HTTP Request to NanoBanana API  
     - *Role:* Sends the generated image prompt and original image URL to NanoBanana for AI-based image editing.  
     - *Config:* Uses HTTP header auth; prompt string escaped correctly.  
     - *Inputs:* Image prompt JSON and Google Drive image URL.  
     - *Outputs:* Response with editing task info including response_url.  
     - *Edge Cases:* API errors; authentication failures.

  5. **Wait for Image Edit**  
     - *Type:* Wait node  
     - *Role:* Pauses workflow for 20 seconds to allow NanoBanana to complete image editing.  
     - *Config:* Fixed 20-second wait.  
     - *Inputs:* Triggered from NanoBanana response.  
     - *Outputs:* Passes after waiting.  
     - *Edge Cases:* Insufficient wait time causing premature requests.

  6. **Download Edited Image**  
     - *Type:* HTTP Request  
     - *Role:* Downloads the edited image from NanoBanana after wait.  
     - *Config:* Uses response_url from NanoBanana's output.  
     - *Inputs:* URL from NanoBanana.  
     - *Outputs:* Edited image data including image URL(s).  
     - *Edge Cases:* URL expired; download failure.

  7. **Google Sheets: Read Video Parameters (CONFIG)**  
     - *Type:* Google Sheets read  
     - *Role:* Retrieves video generation parameters (e.g., model type) from configuration spreadsheet.  
     - *Config:* Reads fixed sheet and document IDs.  
     - *Inputs:* Triggered after edited image download.  
     - *Outputs:* JSON object containing video parameters.  
     - *Edge Cases:* Config sheet missing or inaccessible.

  8. **Set Master Prompt**  
     - *Type:* Set node  
     - *Role:* Defines the master structured schema for video prompt generation (JSON schema with detailed fields for scene, camera, lighting, etc.).  
     - *Config:* Contains a detailed JSON object as a string used by downstream AI agents.  
     - *Inputs:* Parameterized from Google Sheets output.  
     - *Outputs:* Provides standardized prompt template.  
     - *Edge Cases:* Schema syntax errors.

---

#### 1.3 Video Script & Generation

- **Overview:**  
  Generates a video script prompt using OpenAI GPT-4.1-mini and Langchain, formats the prompt, triggers video generation with VEO3, waits for rendering, and downloads the final video.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - AI Agent: Generate Video Script  
  - Format Prompt (Code node)  
  - Generate Video with VEO3 (HTTP Request)  
  - Wait for VEO3 Rendering  
  - Download Video from VEO3

- **Node Details:**

  1. **OpenAI Chat Model**  
     - *Type:* Langchain OpenAI Chat  
     - *Role:* Runs GPT-4.1-mini model to assist in creating AI video script prompts.  
     - *Config:* Uses system message with master prompt JSON from Set Master Prompt.  
     - *Inputs:* Master prompt schema and user inputs.  
     - *Outputs:* Raw AI output for parsing.  
     - *Edge Cases:* API quota exhaustion; model unavailability.

  2. **Structured Output Parser**  
     - *Type:* Langchain Output Parser  
     - *Role:* Parses raw AI output to structured JSON (e.g., title and full prompt).  
     - *Config:* Uses example JSON schema with `title` and `final_prompt` keys.  
     - *Inputs:* AI chat output.  
     - *Outputs:* Parsed JSON for further use.  
     - *Edge Cases:* Parsing failures due to malformed AI response.

  3. **AI Agent: Generate Video Script**  
     - *Type:* Langchain Agent  
     - *Role:* Generates a user-generated content style video prompt JSON object for VEO3 video generation.  
     - *Config:* System message enforces strict JSON output, casual style, one object with `title` and `final_prompt`.  
     - *Inputs:* Caption and image analysis, master prompt schema.  
     - *Outputs:* JSON with video prompt details.  
     - *Edge Cases:* Inconsistent output format breaking downstream nodes.

  4. **Format Prompt**  
     - *Type:* Code node (JavaScript)  
     - *Role:* Stringifies and escapes the final prompt JSON for VEO3 API compatibility.  
     - *Config:* Wraps final prompt as stringified JSON with model and aspectRatio parameters.  
     - *Inputs:* Parsed final_prompt JSON.  
     - *Outputs:* Request body JSON for VEO3 API.  
     - *Edge Cases:* JSON stringification errors.

  5. **Generate Video with VEO3**  
     - *Type:* HTTP Request  
     - *Role:* Calls VEO3 API to generate video based on prompt and edited image.  
     - *Config:* Uses API key via HTTP header auth; POST with prompt, model, aspect ratio, and image URLs.  
     - *Inputs:* Formatted prompt and video parameters.  
     - *Outputs:* Task ID for video rendering.  
     - *Edge Cases:* API rate limits; invalid parameters.

  6. **Wait for VEO3 Rendering**  
     - *Type:* Wait node  
     - *Role:* Waits 20 seconds for VEO3 to complete video rendering.  
     - *Config:* Fixed 20-second delay.  
     - *Inputs:* Task ID from video generation.  
     - *Outputs:* Passes to video download node.  
     - *Edge Cases:* Rendering delay longer than wait time.

  7. **Download Video from VEO3**  
     - *Type:* HTTP Request  
     - *Role:* Retrieves the rendered video URL(s) from VEO3 using task ID.  
     - *Config:* GET request with taskId query.  
     - *Inputs:* Task ID from previous node.  
     - *Outputs:* Final video URLs for distribution.  
     - *Edge Cases:* Task not completed; invalid taskId.

---

#### 1.4 Caption Refinement & Logging

- **Overview:**  
  Refines the video caption using GPT-4o, saves final metadata and status in Google Sheets, and sends the video URL back to the Telegram user.

- **Nodes Involved:**  
  - Rewrite Caption with GPT-4o  
  - Save Caption Video to Google Sheets  
  - Send Video URL via Telegram  
  - Send Final Video Preview  
  - Update Status to "DONE"  
  - Telegram: Send notification

- **Node Details:**

  1. **Rewrite Caption with GPT-4o**  
     - *Type:* Langchain OpenAI node  
     - *Role:* Rewrites the TikTok-style caption to be under 200 characters, following strict output rules.  
     - *Config:* Uses GPT-4o model; prompt includes video idea caption and video title.  
     - *Inputs:* Caption and title from Telegram and AI Agent nodes.  
     - *Outputs:* Concise caption text.  
     - *Edge Cases:* Caption exceeding length; API errors.

  2. **Save Caption Video to Google Sheets**  
     - *Type:* Google Sheets append/update  
     - *Role:* Logs finalized video metadata including title, caption, video URL, and status "CREATE".  
     - *Config:* Matches row by "IMAGE NAME".  
     - *Inputs:* Caption text, video title, and final video URL.  
     - *Outputs:* Confirmation of save.  
     - *Edge Cases:* Sheet access errors.

  3. **Send Video URL via Telegram**  
     - *Type:* Telegram node  
     - *Role:* Sends the generated video URL back to the user's Telegram chat.  
     - *Config:* Text message containing video URL.  
     - *Inputs:* Final video URL and chat ID from Telegram trigger.  
     - *Outputs:* Confirmation of send.  
     - *Edge Cases:* Telegram API errors or invalid chat ID.

  4. **Send Final Video Preview**  
     - *Type:* Telegram node  
     - *Role:* Sends the final video as a video file preview to Telegram chat.  
     - *Config:* Uses video URL from Google Sheets.  
     - *Inputs:* Chat ID and video URL.  
     - *Outputs:* Confirmation of video sent.  
     - *Edge Cases:* Video URL inaccessible; Telegram file size limits.

  5. **Update Status to "DONE"**  
     - *Type:* Google Sheets append/update  
     - *Role:* Updates the status field to "Published" for the processed video.  
     - *Config:* Matches by "IMAGE NAME" (unique file ID).  
     - *Inputs:* Status update data.  
     - *Outputs:* Confirmation.  
     - *Edge Cases:* Concurrent updates; sheet conflicts.

  6. **Telegram: Send notification**  
     - *Type:* Telegram node  
     - *Role:* Sends a simple "Published" notification message to the Telegram user chat.  
     - *Config:* Text message with static "Published".  
     - *Inputs:* Chat ID from Telegram trigger.  
     - *Outputs:* Confirmation.  
     - *Edge Cases:* Telegram downtime.

---

#### 1.5 Multi-Platform Auto-Posting

- **Overview:**  
  Uploads the final video to Blotato and posts simultaneously across multiple social media platforms using Blotato nodes.

- **Nodes Involved:**  
  - Upload Video to BLOTATO  
  - Tiktok  
  - Linkedin  
  - Facebook  
  - Instagram  
  - Twitter (X)  
  - Youtube  
  - Threads  
  - Bluesky  
  - Pinterest  
  - Merge

- **Node Details:**

  1. **Upload Video to BLOTATO**  
     - *Type:* Blotato media upload node  
     - *Role:* Uploads video file URL to Blotato media library for further posting.  
     - *Config:* Uses video URL from VEO3 download node.  
     - *Inputs:* Video URL from "Download Video from VEO3".  
     - *Outputs:* Media URL for social posting nodes.  
     - *Credentials:* Blotato API credentials required.  
     - *Edge Cases:* Upload failure; API limits.

  2. **Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube, Threads, Bluesky, Pinterest**  
     - *Type:* Blotato social post nodes  
     - *Role:* Each node posts the video with the caption text on its respective platform using Blotato.  
     - *Config:* Platforms configured with respective account IDs; captions sourced dynamically from Google Sheets ("CAPTION VIDEO"); media URLs from Blotato upload.  
     - *Inputs:* Media URL and caption text.  
     - *Outputs:* Post success/failure responses.  
     - *Credentials:* Shared Blotato API credentials.  
     - *Edge Cases:* Platform-specific posting restrictions; quota limits; media format issues.

  3. **Merge**  
     - *Type:* Merge node (chooseBranch mode)  
     - *Role:* Collects outputs from all platform posting nodes to proceed to status update.  
     - *Config:* Waits for any one branch to finish among nine inputs.  
     - *Inputs:* All social post nodes as inputs.  
     - *Outputs:* Single merged output to update status.  
     - *Edge Cases:* Partial failures; node timeout.

---

### 3. Summary Table

| Node Name                         | Node Type                              | Functional Role                         | Input Node(s)                              | Output Node(s)                                | Sticky Note                                                                                      |
|----------------------------------|--------------------------------------|---------------------------------------|-------------------------------------------|-----------------------------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger: Receive Video Idea | Telegram Trigger                    | Entry - receive video idea from user  | -                                         | Telegram: Get Image File, Set: Bot Token (Placeholder) | # 📑 STEP 1 — Collect Idea & Image                                                            |
| Telegram: Get Image File          | Telegram                             | Downloads image from Telegram          | Telegram Trigger: Receive Video Idea       | Google Drive: Upload Image                     | # 📑 STEP 1 — Collect Idea & Image                                                            |
| Google Drive: Upload Image        | Google Drive                         | Uploads image to Google Drive          | Telegram: Get Image File                    | Google Sheets: Log Image & Caption             | # 📑 STEP 1 — Collect Idea & Image                                                            |
| Google Sheets: Log Image & Caption| Google Sheets                       | Logs image and caption metadata        | Google Drive: Upload Image                  | Set: Bot Token (Placeholder)                    | # 📑 STEP 1 — Collect Idea & Image                                                            |
| Set: Bot Token (Placeholder)      | Set                                 | Sets Telegram bot token placeholder    | Google Sheets: Log Image & Caption, Telegram Trigger: Receive Video Idea | Telegram API: Get File URL                      | # 📑 STEP 1 — Collect Idea & Image                                                            |
| Telegram API: Get File URL        | HTTP Request                        | Gets Telegram file URL for image       | Set: Bot Token (Placeholder)                | OpenAI Vision: Analyze Reference Image         | # 📑 STEP 1 — Collect Idea & Image                                                            |
| OpenAI Vision: Analyze Reference Image | Langchain OpenAI                  | Analyzes image and outputs YAML        | Telegram API: Get File URL                   | Google Sheets: Update Image Description         | # 📑 STEP 2 — Create Image with NanoBanana                                                    |
| Google Sheets: Update Image Description | Google Sheets                   | Updates image description in sheet     | OpenAI Vision: Analyze Reference Image      | Generate Image Prompt                            | # 📑 STEP 2 — Create Image with NanoBanana                                                    |
| Generate Image Prompt             | Langchain Agent                     | Creates UGC-style image prompt JSON    | Google Sheets: Update Image Description     | NanoBanana: Create Image                         | # 📑 STEP 2 — Create Image with NanoBanana                                                    |
| NanoBanana: Create Image          | HTTP Request (Fal.ai API)           | Sends prompt and image for AI editing  | Generate Image Prompt                        | Wait for Image Edit                              | # 📑 STEP 2 — Create Image with NanoBanana                                                    |
| Wait for Image Edit               | Wait                               | Waits 20 seconds for image editing     | NanoBanana: Create Image                     | Download Edited Image                            | # 📑 STEP 2 — Create Image with NanoBanana                                                    |
| Download Edited Image             | HTTP Request                       | Downloads edited image                  | Wait for Image Edit                          | Google Sheets: Read Video Parameters (CONFIG)  | # 📑 STEP 2 — Create Image with NanoBanana                                                    |
| Google Sheets: Read Video Parameters (CONFIG) | Google Sheets               | Reads video generation config           | Download Edited Image                        | Set Master Prompt                               | # 📑 STEP 3 — Generate Video Ad Script                                                       |
| Set Master Prompt                | Set                                | Defines master schema for video prompt | Google Sheets: Read Video Parameters (CONFIG) | AI Agent: Generate Video Script                  | # 📑 STEP 3 — Generate Video Ad Script                                                       |
| OpenAI Chat Model                | Langchain OpenAI Chat               | Base LLM model for video script prompt | Set Master Prompt                            | Structured Output Parser                         | # 📑 STEP 3 — Generate Video Ad Script                                                       |
| Structured Output Parser         | Langchain Output Parser             | Parses AI chat output into JSON        | OpenAI Chat Model                            | AI Agent: Generate Video Script                  | # 📑 STEP 3 — Generate Video Ad Script                                                       |
| AI Agent: Generate Video Script  | Langchain Agent                    | Generates structured video prompt JSON | Structured Output Parser, Think              | Format Prompt                                   | # 📑 STEP 3 — Generate Video Ad Script                                                       |
| Format Prompt                   | Code                              | Prepares JSON string prompt for API    | AI Agent: Generate Video Script              | Generate Video with VEO3                        | # 📑 STEP 4 — Generate Video with VEO3                                                       |
| Generate Video with VEO3          | HTTP Request                      | Calls VEO3 API to generate video       | Format Prompt                                | Wait for VEO3 Rendering                         | # 📑 STEP 4 — Generate Video with VEO3                                                       |
| Wait for VEO3 Rendering          | Wait                             | Waits 20 seconds for video rendering   | Generate Video with VEO3                      | Download Video from VEO3                        | # 📑 STEP 4 — Generate Video with VEO3                                                       |
| Download Video from VEO3          | HTTP Request                      | Downloads final video URLs              | Wait for VEO3 Rendering                       | Rewrite Caption with GPT-4o                     | # 📑 STEP 4 — Generate Video with VEO3                                                       |
| Rewrite Caption with GPT-4o       | Langchain OpenAI                  | Rewrites caption under 200 chars       | Download Video from VEO3                      | Save Caption Video to Google Sheets             | # 📑 STEP 4 — Generate Video with VEO3                                                       |
| Save Caption Video to Google Sheets| Google Sheets                   | Logs final caption, title, video URL   | Rewrite Caption with GPT-4o                   | Send Video URL via Telegram                      | # 📑 STEP 4 — Generate Video with VEO3                                                       |
| Send Video URL via Telegram       | Telegram                         | Sends video URL text to Telegram chat  | Save Caption Video to Google Sheets           | Send Final Video Preview                         |                                                                                              |
| Send Final Video Preview          | Telegram                         | Sends video file preview to Telegram   | Send Video URL via Telegram                    | Upload Video to BLOTATO                          |                                                                                              |
| Upload Video to BLOTATO           | Blotato                         | Uploads video to Blotato media library  | Send Final Video Preview                       | Tiktok, Linkedin, Facebook, Instagram, etc.     | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Tiktok                          | Blotato                         | Posts video on TikTok                   | Upload Video to BLOTATO                        | Merge                                           | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Linkedin                        | Blotato                         | Posts video on LinkedIn                 | Upload Video to BLOTATO                        | Merge                                           | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Facebook                       | Blotato                         | Posts video on Facebook                 | Upload Video to BLOTATO                        | Merge                                           | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Instagram                      | Blotato                         | Posts video on Instagram                | Upload Video to BLOTATO                        | Merge                                           | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Twitter (X)                   | Blotato                         | Posts video on Twitter (X)              | Upload Video to BLOTATO                        | Merge                                           | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Youtube                       | Blotato                         | Posts video on YouTube (private)        | Upload Video to BLOTATO                        | Merge                                           | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Threads                       | Blotato                         | Posts video on Threads                   | Upload Video to BLOTATO                        | Merge                                           | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Bluesky                       | Blotato                         | Posts video on Bluesky                   | Upload Video to BLOTATO                        | Merge                                           | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Pinterest                    | Blotato                         | Posts video on Pinterest board           | Upload Video to BLOTATO                        | Merge                                           | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Merge                         | Merge                          | Aggregates all social post outputs      | Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube, Threads, Bluesky, Pinterest | Update Status to "DONE"                          | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Update Status to "DONE"          | Google Sheets                  | Updates status to “Published”            | Merge                                         | Telegram: Send notification                      | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Telegram: Send notification      | Telegram                       | Sends "Published" notification          | Update Status to "DONE"                        | -                                               | # 📑 STEP 5 — Auto-Post to All Platforms                                                    |
| Think                         | Langchain Tool Think            | Tool helper for AI agent                 | OpenAI Chat Model                             | AI Agent: Generate Video Script                  |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger: Receive Video Idea**  
   - Type: Telegram Trigger  
   - Credentials: Connect your Telegram bot.  
   - Parameters: Listen for message updates.  

2. **Add Telegram Get Image File node**  
   - Type: Telegram  
   - Operation: Get file using photo[2].file_id from trigger.  
   - Connect from Telegram Trigger.  

3. **Add Google Drive Upload node**  
   - Type: Google Drive  
   - Upload the downloaded image file.  
   - Use photo[2].file_unique_id as file name.  
   - Configure target folder (must be public).  
   - Connect from Telegram Get Image File.  

4. **Add Google Sheets Log Image & Caption node**  
   - Type: Google Sheets Append/Update  
   - Log IMAGE NAME, IMAGE URL (Google Drive webContentLink), CAPTION, STATUS="EN COURS".  
   - Match rows by IMAGE NAME.  
   - Connect from Google Drive Upload node.  

5. **Add Set node for Bot Token (Placeholder)**  
   - Set a variable YOUR_BOT_TOKEN (empty string placeholder).  
   - Connect from Google Sheets Log node and Telegram Trigger node (parallel).  

6. **Add HTTP Request node to get Telegram file path**  
   - URL: `https://api.telegram.org/bot{{YOUR_BOT_TOKEN}}/getFile?file_id={{photo[3].file_id}}`  
   - Connect from Set Bot Token node.  

7. **Add OpenAI Vision node**  
   - Type: Langchain OpenAI (image analysis)  
   - Input image URL: Constructed from Telegram file path result.  
   - Outputs YAML description.  
   - Connect from Telegram API Get File URL node.  

8. **Add Google Sheets Update Image Description node**  
   - Append/update IMAGE DESCRIPTION with YAML from OpenAI Vision.  
   - Match by IMAGE NAME.  
   - Connect from OpenAI Vision node.  

9. **Add Langchain Agent node Generate Image Prompt**  
   - Input: CAPTION and IMAGE DESCRIPTION.  
   - System prompt: UGC Image Prompt Builder (as per detailed system message).  
   - Output: Single JSON key `image_prompt`.  
   - Connect from Google Sheets Update Image Description node.  

10. **Add HTTP Request node NanoBanana: Create Image**  
    - POST to NanoBanana API with JSON body containing prompt and Google Drive image URL.  
    - Use HTTP Header Auth (Fal.ai credentials).  
    - Connect from Generate Image Prompt node.  

11. **Add Wait node (20s) for image editing**  
    - Connect from NanoBanana Create Image node.  

12. **Add HTTP Request node Download Edited Image**  
    - GET request to NanoBanana response_url.  
    - Connect from Wait node.  

13. **Add Google Sheets Read Video Parameters (CONFIG) node**  
    - Reads video generation config parameters.  
    - Connect from Download Edited Image node.  

14. **Add Set Master Prompt node**  
    - Set the detailed JSON schema for video prompt.  
    - Connect from Google Sheets Read Video Parameters node.  

15. **Add Langchain OpenAI Chat Model node**  
    - Model: GPT-4.1-mini  
    - Connect from Set Master Prompt node.  

16. **Add Structured Output Parser node**  
    - Parses AI chat output to structured JSON with `title` and `final_prompt`.  
    - Connect from OpenAI Chat Model node.  

17. **Add Langchain Agent AI Agent: Generate Video Script node**  
    - System prompt: Structured Video Ad Prompt Generator (as defined).  
    - Inputs: caption, image description, master prompt schema.  
    - Connect from Structured Output Parser.  

18. **Add Code node Format Prompt**  
    - Stringify and escape final_prompt JSON to prepare for API.  
    - Connect from AI Agent Generate Video Script node.  

19. **Add HTTP Request Generate Video with VEO3 node**  
    - POST to VEO3 API with prompt, model, aspectRatio, image URLs.  
    - Use HTTP Header Auth (Kie AI credentials).  
    - Connect from Format Prompt node.  

20. **Add Wait node (20s) for VEO3 rendering**  
    - Connect from Generate Video with VEO3 node.  

21. **Add HTTP Request Download Video from VEO3 node**  
    - GET with taskId from video generation node.  
    - Connect from Wait for VEO3 Rendering node.  

22. **Add Langchain OpenAI Rewrite Caption with GPT-4o node**  
    - Rewrites caption text to ≤200 chars.  
    - Connect from Download Video from VEO3 node.  

23. **Add Google Sheets Save Caption Video node**  
    - Append/update with STATUS="CREATE", video title, caption, final video URL.  
    - Match by IMAGE NAME.  
    - Connect from Rewrite Caption node.  

24. **Add Telegram Send Video URL node**  
    - Sends text message with video URL to Telegram chat.  
    - Connect from Save Caption Video node.  

25. **Add Telegram Send Final Video Preview node**  
    - Sends video file preview to Telegram chat.  
    - Connect from Telegram Send Video URL node.  

26. **Add Blotato Upload Video node**  
    - Upload final video URL to Blotato media library.  
    - Connect from Telegram Send Final Video Preview node.  

27. **Add Blotato social posting nodes for Tiktok, Linkedin, Facebook, Instagram, Twitter (X), Youtube, Threads, Bluesky, Pinterest**  
    - Configure each with correct account IDs and platform.  
    - Use caption from Google Sheets and media URL from Blotato upload.  
    - Connect all from Upload Video to BLOTATO node.  

28. **Add Merge node (chooseBranch mode, 9 inputs)**  
    - Aggregates outputs from all social posting nodes.  
    - Connect all social nodes to Merge node.  

29. **Add Google Sheets Update Status to "DONE" node**  
    - Updates status to "Published" for the image/video entry.  
    - Connect from Merge node.  

30. **Add Telegram Send Notification node**  
    - Sends "Published" message to Telegram user.  
    - Connect from Update Status to "DONE" node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                      |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Full tutorial video and branding image provided by Dr. Firas.                                                 | [YouTube Tutorial](https://youtu.be/nlwpbXQqNQ4)                                                                    |
| Full documentation with setup instructions, API configs, and customization tips available on Notion.          | [Notion Documentation](https://automatisation.notion.site/NonoBanan-2643d6550fd98041aef5dcbe8ab0f7a1?source=copy_link) |
| Requirements include a Blotato Pro plan, API key generation, enabling verified community nodes in n8n, and public Google Drive folder. | [Blotato](https://blotato.com/?ref=firas), Google Sheets template shared in workflow notes.                          |
| Workflow uses verified community nodes from Blotato and OpenAI credentials; ensure proper credential setup.   | Credential management recommended in n8n docs.                                                                       |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It strictly follows content policies and contains no illegal or protected elements. All data processed is legal and public.