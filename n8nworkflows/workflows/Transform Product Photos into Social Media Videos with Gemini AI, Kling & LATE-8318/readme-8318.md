Transform Product Photos into Social Media Videos with Gemini AI, Kling & LATE

https://n8nworkflows.xyz/workflows/transform-product-photos-into-social-media-videos-with-gemini-ai--kling---late-8318


# Transform Product Photos into Social Media Videos with Gemini AI, Kling & LATE

---

# 1. Workflow Overview

This workflow automates the transformation of static product photos into sophisticated, cinematic social media videos using advanced AI services. It is designed primarily for e-commerce, brand marketing, and social media agencies aiming to generate premium vertical (9:16) brand animations from a single product image, incorporating human approval and automated multi-platform publishing.

The workflow consists of the following logical blocks:

- **1.1 Input Reception & Initialization:** Handles user input via Telegram, sets chat context variables, and prepares the image for processing.

- **1.2 Creative AI Direction:** Employs multi-agent AI (Google Gemini-powered) to generate refined visual edit prompts and cinematic motion prompts based on the product image.

- **1.3 Image Enhancement and Variation Generation:** Uses AI image editing services to create two enhanced visual concepts (A and B) from the original photo, uploads and prepares them for video generation.

- **1.4 Human Approval Gate:** Sends the edited image previews to the user via Telegram for approval before proceeding to resource-intensive video generation.

- **1.5 Cinematic Video Generation:** Utilizes Kling AI to generate two short cinematic video sequences matching the approved visual concepts, handling asynchronous processing and completion polling.

- **1.6 Video Post-Processing & Audio Synthesis:** Composes the two video clips into a single video, generates a matching ambient soundtrack, and optionally upscales the final output for premium quality.

- **1.7 Social Media Distribution:** Offers the user the option to share the final video automatically to multiple social media platforms (Instagram Reels, TikTok, YouTube Shorts) using Late.dev API.

- **1.8 Notifications & User Interaction:** Communicates status updates, approvals, rejections, and final video delivery to the user through Telegram messages.

---

# 2. Block-by-Block Analysis

---

## 2.1 Input Reception & Initialization

**Overview:**  
This block captures product photo uploads from users via Telegram, extracts the chat context, and prepares the original image for AI processing.

**Nodes Involved:**  
- Telegram Trigger  
- Request Photo  
- Set: Chat Vars  
- Merge Vars + Photo1  
- Merge API + Photo  
- Upload Original Image (imgbb)  
- Set: YOUR API KEY (ImgBB)  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger (Webhook)  
  - Role: Entry point, listens for any Telegram message from user, downloads attached files automatically.  
  - Configuration: listens for "message" update type with file download enabled.  
  - Inputs: Telegram user message (with photo).  
  - Outputs: JSON with message and photo data.  
  - Failures: Network issues, invalid bot token, file download errors.  
  - Notes: Requires Telegram Bot API credentials.

- **Request Photo**  
  - Type: Telegram node (sendAndWait)  
  - Role: Prompts the user to upload a product photo in supported formats (jpg, jpeg, webp).  
  - Configuration: Sends message with a custom form requesting a file upload.  
  - Inputs: Telegram chat ID from trigger.  
  - Outputs: Waits for user response with file.  
  - Failures: User does not respond or sends wrong file type.  
  - Notes: Interactive with Telegram users.

- **Set: Chat Vars**  
  - Type: Set node  
  - Role: Extracts and stores Telegram chat ID as a variable for downstream nodes.  
  - Configuration: Assigns `chat_id` from Telegram message JSON.  
  - Inputs: Telegram Trigger output.  
  - Outputs: JSON including `chat_id`.  
  - Failures: Missing chat ID in message.  

- **Merge Vars + Photo1**  
  - Type: Merge node (combine by position)  
  - Role: Combines chat variables and photo data for unified downstream processing.  
  - Inputs: Chat vars and photo upload response.  
  - Outputs: Combined JSON for further processing.

- **Set: YOUR API KEY (ImgBB)**  
  - Type: Set node  
  - Role: Stores the user’s ImgBB API key as a variable for image hosting uploads.  
  - Configuration: User must replace placeholder with real API key.  
  - Inputs: None (manual configuration).  
  - Outputs: JSON with `imgbb_api_key`.  
  - Notes: Essential for ImgBB requests to work.

- **Merge API + Photo**  
  - Type: Merge node  
  - Role: Combines ImgBB API key and photo data for the upload request.  
  - Inputs: ImgBB key and photo data.  
  - Outputs: Combined JSON.

- **Upload Original Image (imgbb)**  
  - Type: HTTP Request  
  - Role: Uploads the original product image to ImgBB to get a stable hosted URL for AI agents.  
  - Configuration: POST multipart form-data with image binary and API key.  
  - Inputs: Image binary data from Telegram, API key from Set node.  
  - Outputs: URL of hosted image for AI processing.  
  - Failures: Invalid API key, network timeout, file upload errors.  
  - Retry: Enabled.

---

## 2.2 Creative AI Direction

**Overview:**  
Coordinates AI agents to analyze the uploaded image and generate high-level visual and motion prompts for cinematic brand animations using Google Gemini and LangChain agents.

**Nodes Involved:**  
- Creative Director (Orchestrator) agent  
- Art Director  
- Motion designer  
- Google Gemini Chat Model  
- Google Gemini Chat Model15  
- Google Gemini Chat Model16  
- Structured Output Parser  
- Think  
- Merge Vars + Photo1  
- Set Storyboard Vars  
- start processing  

**Node Details:**

- **Creative Director (Orchestrator) agent**  
  - Type: LangChain Agent node  
  - Role: Master coordinator that sends image data to two sub-agents (Art Director and Motion Designer), merges their outputs into a single JSON object with refined cinematic prompts.  
  - Configuration: Uses system prompt describing luxury brand animation goals, cinematic standards, and strict JSON output format.  
  - Inputs: Uploaded image and merged vars.  
  - Outputs: JSON with keys: title, prompt1, prompt2, kling_prompt1, kling_prompt2, environment, sound.  
  - Failures: AI model API errors, output format errors, timeout.  
  - Requirements: Google Gemini (PaLM) API credentials.  
  - Notes: Uses passthrough of binary images.  

- **Art Director**  
  - Type: LangChain agent tool  
  - Role: Generates two sophisticated visual edit prompts and related metadata (title, environment, sound) based on the product image.  
  - Configuration: Invoked by Creative Director with luxury brand aesthetic instructions.  
  - Failure cases: Model errors, incomplete JSON output.  

- **Motion designer**  
  - Type: LangChain agent tool  
  - Role: Produces two cinematic motion prompts optimized for Kling AI video generation, including advanced camera choreography and lighting dynamics.  
  - Failures: Same as above.  

- **Google Gemini Chat Model / Chat Model15 / Chat Model16**  
  - Type: Language model nodes (Google Gemini)  
  - Role: Execute the underlying AI calls for the agents. Different nodes correspond to different agent roles or steps.  
  - Configuration: Model set to "models/gemini-2.5-pro".  
  - Failures: API auth errors, rate limits, invalid inputs.  

- **Structured Output Parser**  
  - Type: LangChain output parser  
  - Role: Ensures AI output conforms to expected JSON schema for downstream use.  
  - Input: Raw AI text output.  
  - Output: Parsed JSON with cinematic prompts.  
  - Failures: Parsing errors if AI output malformed.  

- **Think**  
  - Type: LangChain toolThink node  
  - Role: Allows AI agents to perform internal analysis or "thinking" steps during orchestration for quality control.  
  - Input/output: Internal use by agents.  

- **Merge Vars + Photo1**  
  - Type: Merge node to combine chat vars and image for agent input.

- **Set Storyboard Vars**  
  - Type: Set node  
  - Role: Stores the final cinematic creative outputs from the orchestrator agent into workflow variables for use in image editing and video generation.  
  - Keys set: title, prompt1, prompt2, kling_prompt1, kling_prompt2, sound.  

- **start processing**  
  - Type: Telegram node (send message)  
  - Role: Notifies the user that processing has started and shares the cinematic prompts for transparency.

---

## 2.3 Image Enhancement and Variation Generation

**Overview:**  
Generates two enhanced visual concepts (A and B) from the original product photo, leveraging Gemini Flash AI image editing, then uploads these edited images to ImgBB for stable hosting.

**Nodes Involved:**  
- Gen Image A  
- Split Image A  
- Download Image A  
- Rename -> photo A  
- Upload Edited A (imgbb)  
- Send Image A  
- Gen Image B  
- Split Image B  
- Download Image B  
- Rename -> photo B  
- Upload Edited B (imgbb)  
- Send Image B  
- Merge Edited URLs  
- Aggregate Edited URLs  

**Node Details:**

- **Gen Image A / Gen Image B**  
  - Type: HTTP Request  
  - Role: Call Gemini Flash image editing API with prompt1 and prompt2 respectively to generate enhanced image concepts.  
  - Configuration: POST JSON body includes prompt and original image URL from ImgBB, requesting 1 edited image.  
  - Inputs: Prompts and original image URL.  
  - Outputs: JSON containing edited image URLs.  
  - Failures: API errors, invalid prompt, network issues.  
  - Auth: Custom HTTP header auth (Veo3_FalAI).  

- **Split Image A / Split Image B**  
  - Type: SplitOut node  
  - Role: Extracts individual images from the array response for separate processing.  
  - Input: Edited images array.  
  - Output: Single image JSON item.  

- **Download Image A / Download Image B**  
  - Type: HTTP Request  
  - Role: Downloads the edited image file for Telegram sending and further uploads.  
  - Configuration: Requests binary file response.  
  - Failures: Download errors, invalid URLs.  

- **Rename -> photo A / Rename -> photo B**  
  - Type: Code node  
  - Role: Renames binary data property to `photo2` and `photo3` respectively for Telegram node compatibility.  
  - JS code: Returns binary data renamed accordingly.  

- **Upload Edited A (imgbb) / Upload Edited B (imgbb)**  
  - Type: HTTP Request  
  - Role: Uploads edited images to ImgBB to obtain stable URLs for video generation.  
  - Configuration: Multipart form upload with binary image and API key.  
  - Failures: Same as original upload.  

- **Send Image A / Send Image B**  
  - Type: Telegram node  
  - Role: Sends edited concept images to the user with captions for approval.  
  - Inputs: Binary image data renamed in previous node.  
  - Failures: Telegram API errors, chat ID missing.  

- **Merge Edited URLs**  
  - Type: Merge node  
  - Role: Combines URLs of both edited images from ImgBB for video generation.  

- **Aggregate Edited URLs**  
  - Type: Aggregate node  
  - Role: Aggregates merged URLs into a single array for subsequent nodes.

---

## 2.4 Human Approval Gate

**Overview:**  
Presents the two edited visual concepts to the user for approval via Telegram, then routes execution based on approval or rejection.

**Nodes Involved:**  
- Ask Approval (Images)  
- Approved?  
- Gate: Render Approved  
- Notify: Rejected  

**Node Details:**

- **Ask Approval (Images)**  
  - Type: Telegram node (sendAndWait)  
  - Role: Asks user to approve the image previews before video rendering. Implements double approval option.  
  - Inputs: Chat ID, message text, inline approval buttons.  
  - Failures: Telegram API errors, user timeout or no response.  

- **Approved?**  
  - Type: IF node  
  - Role: Checks user response JSON for approval flags (supports multiple possible JSON paths).  
  - Outputs: True (approved) or False (rejected).  
  - Failures: Missing or malformed approval data.  

- **Gate: Render Approved**  
  - Type: Merge node  
  - Role: Routes workflow forward only if approved, triggering video generation.  

- **Notify: Rejected**  
  - Type: Telegram node  
  - Role: Notifies user that images were rejected and no further processing will occur.  

---

## 2.5 Cinematic Video Generation

**Overview:**  
Generates two short cinematic videos based on the approved edited images and motion prompts using Kling AI video generation API. Includes polling for completion and downloading final videos.

**Nodes Involved:**  
- kling Queue A  
- Wait A  
- Kling Status A  
- Animation Completed? A  
- Kling Result A  
- Download A  
- kling Queue B  
- Wait B  
- Kling Status B  
- Animation Completed? B  
- Kling Result B  
- Download B  
- Merge Kling Results  
- Aggregate Results  
- FFmpeg Compose  
- Notify: Stitching  
- Wait Compose  
- Get Video  

**Node Details:**

- **kling Queue A / kling Queue B**  
  - Type: HTTP Request  
  - Role: Submit video generation requests to Kling API with respective edited image URLs and motion prompts.  
  - Config: POST with JSON body specifying image URL, prompt, aspect ratio (9:16), duration (5s).  
  - Failures: API errors, invalid URLs, authorization failures.  

- **Wait A / Wait B**  
  - Type: Wait node  
  - Role: Pause execution to allow asynchronous video rendering to proceed.  
  - Config: 40 seconds delay.  

- **Kling Status A / Kling Status B**  
  - Type: HTTP Request  
  - Role: Poll Kling API for video generation status using request_id from initial submission.  
  - Failures: Network errors, invalid request_id, timeout.  

- **Animation Completed? A / Animation Completed? B**  
  - Type: IF node  
  - Role: Checks if video status is "COMPLETED" to proceed or wait more.  
  - Loops back to Wait if not complete.  

- **Kling Result A / Kling Result B**  
  - Type: HTTP Request  
  - Role: Retrieve final video details from Kling API after completion.  
  - Failures: Same as status requests.  

- **Download A / Download B**  
  - Type: HTTP Request  
  - Role: Downloads generated video files for sending or composing.  
  - Configuration: Binary file response.  

- **Merge Kling Results**  
  - Type: Merge node (4 inputs)  
  - Role: Combines both Kling video outputs and possibly related metadata for composition.  

- **Aggregate Results**  
  - Type: Aggregate node  
  - Role: Aggregates merged Kling results for next steps.  

- **FFmpeg Compose**  
  - Type: HTTP Request  
  - Role: Uses Fal.AI FFmpeg API to stitch the two video clips sequentially into one continuous video.  
  - Inputs: Array of video URLs with timestamps and durations.  
  - Failures: API errors, invalid video URLs.  

- **Notify: Stitching**  
  - Type: Telegram node  
  - Role: Notifies user that video stitching is in progress.  

- **Wait Compose**  
  - Type: Wait node  
  - Role: Waits 30 seconds for stitching to complete.  

- **Get Video**  
  - Type: HTTP Request  
  - Role: Retrieves the composed final video from the FFmpeg compose API response URL.

---

## 2.6 Video Post-Processing & Audio Synthesis

**Overview:**  
Generates a matching ambient soundtrack for the composed video, optionally upscales the video, and downloads the final enhanced video.

**Nodes Involved:**  
- Create Soundtrack  
- Wait Soundtrack  
- Get Soundtrack  
- Combine  
- Upscale  
- Wait2  
- Get Result  
- Download Final Video (No Upscale)  
- Send Full Video (No Upscale)  
- Send Final Video Url  

**Node Details:**

- **Create Soundtrack**  
  - Type: HTTP Request  
  - Role: Calls MMAudio v2 API to generate premium ambient soundtrack matching the cinematic sound prompt.  
  - Inputs: Sound description from storyboard vars, video URL.  
  - Config: 10 seconds duration, 25 steps, no voice.  
  - Failures: API errors, network issues.  

- **Wait Soundtrack**  
  - Type: Wait node  
  - Role: Waits 100 seconds for soundtrack generation.  

- **Get Soundtrack**  
  - Type: HTTP Request  
  - Role: Retrieves generated soundtrack file or URL.  

- **Combine**  
  - Type: Merge node  
  - Role: Combines video and soundtrack data for upscaling.  

- **Upscale**  
  - Type: HTTP Request  
  - Role: Calls Topaz AI upscale API to enhance video resolution with 2x upscale factor.  
  - Inputs: Video URL.  
  - Failures: API limits, invalid video URL.  

- **Wait2**  
  - Type: Wait node  
  - Role: Waits 3 minutes for upscale processing.  

- **Get Result**  
  - Type: HTTP Request  
  - Role: Retrieves final upscaled video from Topaz API.  

- **Download Final Video (No Upscale)**  
  - Type: HTTP Request  
  - Role: Downloads the final video binary for sending.  

- **Send Full Video (No Upscale)**  
  - Type: Telegram node  
  - Role: Sends the full un-upscaled video to the user.  

- **Send Final Video Url**  
  - Type: Telegram node  
  - Role: Sends the final upscaled video URL to the user as a Markdown-formatted clickable link.

---

## 2.7 Social Media Distribution

**Overview:**  
Upon user approval, automatically publishes the final video to configured social media accounts on Instagram, TikTok, and YouTube Shorts using Late.dev API.

**Nodes Involved:**  
- Send message and wait for response  
- Get All Social Media Accounts  
- Share to Instagram  
- Share to TikTok  
- Share to Youtube Shorts  

**Node Details:**

- **Send message and wait for response**  
  - Type: Telegram node (sendAndWait)  
  - Role: Asks user if they want to share the final video on social media.  
  - Failures: User denial or no response.  

- **Get All Social Media Accounts**  
  - Type: HTTP Request  
  - Role: Retrieves all connected social media accounts from Late.dev for the user’s profile.  
  - Inputs: None, uses Late.dev API credential.  
  - Failures: API auth errors, network failures.  

- **Share to Instagram / TikTok / Youtube Shorts**  
  - Type: HTTP Request  
  - Role: Posts the final video URL to respective platform with platform-specific metadata and privacy settings.  
  - Inputs: Final video URL, platform account IDs, content captions including storyboard title.  
  - Failures: API errors, wrong account IDs, invalid video URL.  
  - Notes: TikTok post includes detailed privacy and content metadata.

---

## 2.8 Notifications & User Interaction

**Overview:**  
Manages communication with the user through Telegram about workflow progress, approvals, rejections, and final deliverables.

**Nodes Involved:**  
- start processing  
- Notify: Rejected  
- Notify: Stitching  
- Send Image A  
- Send Image B  
- Send video Concept A  
- Send video Concept B  
- Send Full Video (No Upscale)  
- Send Final Video Url  

**Node Details:**

- Each node is a Telegram node configured to send messages or media (photos/videos) to the user.  
- Messages contain status updates, prompts, or final media with captions.  
- Failures: Telegram API limits, invalid chat ID, network issues.

---

# 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                                   | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                          |
|---------------------------|----------------------------------|--------------------------------------------------|---------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Telegram Trigger           | Telegram Trigger                  | Entry point, receives user messages/photos       | -                               | Request Photo, Set: Chat Vars   | Part of 🎯 Command Center: Input & Coordination Hub.                                               |
| Request Photo             | Telegram                         | Requests product photo upload                      | Telegram Trigger                | Merge Vars + Photo1             | Part of 🎯 Command Center: Input & Coordination Hub.                                               |
| Set: Chat Vars             | Set                              | Stores Telegram chat ID                            | Telegram Trigger                | Merge Vars + Photo1             | Part of 🎯 Command Center: Input & Coordination Hub.                                               |
| Merge Vars + Photo1        | Merge                            | Combines chat vars and photo for AI agents       | Request Photo, Set: Chat Vars   | Creative Director (Orchestrator) | Part of 🎯 Command Center: Input & Coordination Hub.                                               |
| Set: YOUR API KEY (ImgBB)  | Set                              | Stores ImgBB API key                              | -                               | Merge API + Photo              | Part of 🎯 Command Center: Input & Coordination Hub. Essential for ImgBB API.                      |
| Merge API + Photo          | Merge                            | Combines ImgBB API key with photo data            | Set: YOUR API KEY, Merge Vars + Photo1 | Upload Original Image (imgbb) | Part of 🎯 Command Center: Input & Coordination Hub.                                               |
| Upload Original Image (imgbb) | HTTP Request                  | Uploads original image to ImgBB                    | Merge API + Photo               | Merge Vars + Photo              | Part of 🎨 Content Generation Engine: Autonomous Image Processing                                   |
| Creative Director (Orchestrator) agent | LangChain Agent      | Master AI agent coordinating sub-agents           | Merge Vars + Photo1             | Set Storyboard Vars, start processing | Part of 🧠 Vision Intelligence Network: Multi-Agent Creative Direction                             |
| Art Director              | LangChain Agent Tool             | Generates sophisticated visual prompts            | Creative Director               | Creative Director              | Part of 🧠 Vision Intelligence Network: Multi-Agent Creative Direction                             |
| Motion designer           | LangChain Agent Tool             | Generates cinematic motion prompts                 | Creative Director               | Creative Director              | Part of 🧠 Vision Intelligence Network: Multi-Agent Creative Direction                             |
| Google Gemini Chat Models | Language Model Nodes (Gemini)    | Executes AI language model calls                   | LangChain agents               | LangChain agents              | Part of 🧠 Vision Intelligence Network: Multi-Agent Creative Direction                             |
| Structured Output Parser   | LangChain Output Parser          | Parses AI JSON output                               | Creative Director               | Creative Director              | Part of 🧠 Vision Intelligence Network: Multi-Agent Creative Direction                             |
| Think                     | LangChain ToolThink              | Internal AI analysis step                           | Creative Director               | Creative Director              | Part of 🧠 Vision Intelligence Network: Multi-Agent Creative Direction                             |
| Set Storyboard Vars       | Set                              | Stores cinematic prompts and metadata              | Creative Director               | Merge Vars + Photo             | Part of 🧠 Vision Intelligence Network: Multi-Agent Creative Direction                             |
| start processing          | Telegram                        | Notifies user processing started                    | Set Storyboard Vars             | Merge Vars + Photo             | Part of 🧠 Vision Intelligence Network: Multi-Agent Creative Direction                             |
| Gen Image A               | HTTP Request                    | AI image editing for concept A                      | Merge Vars + Photo              | Split Image A                  | Part of 🎨 Content Generation Engine: Autonomous Image Processing                                   |
| Split Image A             | SplitOut                       | Extracts single image from array                     | Gen Image A                    | Download Image A               | Part of 🎨 Content Generation Engine: Autonomous Image Processing                                   |
| Download Image A          | HTTP Request                   | Downloads edited image A                             | Split Image A                  | Rename -> photo A              | Part of 🎨 Content Generation Engine: Autonomous Image Processing                                   |
| Rename -> photo A         | Code                           | Renames binary property for Telegram compatibility  | Download Image A               | Upload Edited A (imgbb), Send Image A | Part of 🎨 Content Generation Engine: Autonomous Image Processing                                   |
| Upload Edited A (imgbb)   | HTTP Request                   | Uploads edited image A to ImgBB                      | Rename -> photo A              | Merge Edited URLs             | Part of 🎨 Content Generation Engine: Autonomous Image Processing                                   |
| Send Image A              | Telegram                      | Sends edited concept A image to user                 | Rename -> photo A              | Ask Approval (Images)          | Part of 🛡️ Human Oversight Gate: Strategic Approval Checkpoint                                    |
| Gen Image B               | HTTP Request                   | AI image editing for concept B                      | Merge Vars + Photo              | Split Image B                  | Part of 🎨 Content Generation Engine: Autonomous Image Processing                                   |
| Split Image B             | SplitOut                       | Extracts single image from array                     | Gen Image B                    | Download Image B               | Part of 🎨 Content Generation Engine: Autonomous Image Processing                                   |
| Download Image B          | HTTP Request                   | Downloads edited image B                             | Split Image B                  | Rename -> photo B              | Part of 🎨 Content Generation Engine: Autonomous Image Processing                                   |
| Rename -> photo B         | Code                           | Renames binary property for Telegram compatibility  | Download Image B               | Upload Edited B (imgbb), Send Image B | Part of 🎨 Content Generation Engine: Autonomous Image Processing                                   |
| Upload Edited B (imgbb)   | HTTP Request                   | Uploads edited image B to ImgBB                      | Rename -> photo B              | Merge Edited URLs             | Part of 🎨 Content Generation Engine: Autonomous Image Processing                                   |
| Send Image B              | Telegram                      | Sends edited concept B image to user                 | Rename -> photo B              | Ask Approval (Images)          | Part of 🛡️ Human Oversight Gate: Strategic Approval Checkpoint                                    |
| Merge Edited URLs         | Merge                          | Combines ImgBB URLs of edited images                 | Upload Edited A (imgbb), Upload Edited B (imgbb) | Aggregate Edited URLs         | Part of 🛡️ Human Oversight Gate: Strategic Approval Checkpoint                                    |
| Aggregate Edited URLs     | Aggregate                      | Aggregates URLs into array                            | Merge Edited URLs              | Gate: Render Approved          | Part of 🛡️ Human Oversight Gate: Strategic Approval Checkpoint                                    |
| Ask Approval (Images)     | Telegram (sendAndWait)         | Requests user approval of edited images              | Send Image A, Send Image B     | Approved?                     | Part of 🛡️ Human Oversight Gate: Strategic Approval Checkpoint                                    |
| Approved?                 | IF                            | Checks user approval response                         | Ask Approval                  | Gate: Render Approved (true), Notify: Rejected (false) | Part of 🛡️ Human Oversight Gate: Strategic Approval Checkpoint                                    |
| Gate: Render Approved     | Merge                         | Passes forward only if images approved                | Approved?                     | kling Queue A, kling Queue B   | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Notify: Rejected          | Telegram                      | Notifies user of rejection and halts further processing | Approved?                     | -                            | Part of 🛡️ Human Oversight Gate: Strategic Approval Checkpoint                                    |
| kling Queue A             | HTTP Request                  | Submits video generation request A to Kling API      | Gate: Render Approved          | Wait A                        | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Wait A                   | Wait                          | Waits 40 seconds for video A generation               | kling Queue A                 | Kling Status A                | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Kling Status A            | HTTP Request                  | Polls Kling API for completion status of video A     | Wait A                        | Animation Completed? A        | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Animation Completed? A    | IF                            | Checks if video A is completed                         | Kling Status A                | Kling Result A (true), Wait A (false) | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Kling Result A            | HTTP Request                  | Gets final video A info                                | Animation Completed? A         | Download A                   | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Download A                | HTTP Request                  | Downloads video A file                                 | Kling Result A                | Send video Concept A          | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Send video Concept A      | Telegram                      | Sends video A to user                                 | Download A                   | -                            | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| kling Queue B             | HTTP Request                  | Submits video generation request B to Kling API      | Gate: Render Approved          | Wait B                        | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Wait B                   | Wait                          | Waits 40 seconds for video B generation               | kling Queue B                 | Kling Status B                | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Kling Status B            | HTTP Request                  | Polls Kling API for completion status of video B     | Wait B                        | Animation Completed? B        | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Animation Completed? B    | IF                            | Checks if video B is completed                         | Kling Status B                | Kling Result B (true), Wait B (false) | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Kling Result B            | HTTP Request                  | Gets final video B info                                | Animation Completed? B         | Download B                   | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Download B                | HTTP Request                  | Downloads video B file                                 | Kling Result B                | Send video Concept B          | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Send video Concept B      | Telegram                      | Sends video B to user                                 | Download B                   | -                            | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Merge Kling Results       | Merge (4 inputs)              | Combines both Kling video results and metadata        | Kling Result A, Kling Result B, Download A, Download B | Aggregate Results           | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| Aggregate Results         | Aggregate                    | Aggregates merged Kling results                        | Merge Kling Results           | FFmpeg Compose               | Part of 🎬 Cinematic Production Studio: Motion Agent Video Generation                              |
| FFmpeg Compose            | HTTP Request                  | Stitches video A and B into a single video             | Aggregate Results             | Notify: Stitching, Wait Compose | Part of 🎵 Audio Intelligence Lab: Soundtrack Generation & Composition                            |
| Notify: Stitching         | Telegram                      | Notifies user about video stitching                     | FFmpeg Compose               | Wait Compose                 | Part of 🎵 Audio Intelligence Lab: Soundtrack Generation & Composition                            |
| Wait Compose              | Wait                          | Waits 30 seconds for video stitching                    | Notify: Stitching             | Get Video                   | Part of 🎵 Audio Intelligence Lab: Soundtrack Generation & Composition                            |
| Get Video                 | HTTP Request                  | Retrieves composed video from FFmpeg API               | Wait Compose                 | Create Soundtrack            | Part of 🎵 Audio Intelligence Lab: Soundtrack Generation & Composition                            |
| Create Soundtrack         | HTTP Request                  | Generates ambient soundtrack matching video             | Get Video                   | Wait Soundtrack              | Part of 🎵 Audio Intelligence Lab: Soundtrack Generation & Composition                            |
| Wait Soundtrack           | Wait                          | Waits 100 seconds for soundtrack generation             | Create Soundtrack            | Get Soundtrack               | Part of 🎵 Audio Intelligence Lab: Soundtrack Generation & Composition                            |
| Get Soundtrack            | HTTP Request                  | Retrieves final soundtrack                               | Wait Soundtrack              | Combine                     | Part of 🎵 Audio Intelligence Lab: Soundtrack Generation & Composition                            |
| Combine                  | Merge                          | Combines final video and soundtrack for upscaling       | Get Soundtrack               | Upscale                     | Part of ⚡ Quality Enhancement Engine: Autonomous Upscaling System                               |
| Upscale                  | HTTP Request                   | Sends video for 2x upscale using Topaz AI               | Combine                     | Wait2                       | Part of ⚡ Quality Enhancement Engine: Autonomous Upscaling System                               |
| Wait2                    | Wait                           | Waits 3 minutes for upscale process                      | Upscale                     | Get Result                  | Part of ⚡ Quality Enhancement Engine: Autonomous Upscaling System                               |
| Get Result               | HTTP Request                   | Retrieves final upscaled video                            | Wait2                       | Download Final Video (No Upscale), Wait | Part of ⚡ Quality Enhancement Engine: Autonomous Upscaling System                               |
| Download Final Video (No Upscale) | HTTP Request            | Downloads final video                                     | Get Result                  | Send Full Video (No Upscale) | Part of ⚡ Quality Enhancement Engine: Autonomous Upscaling System                               |
| Send Full Video (No Upscale) | Telegram                  | Sends final video without upscale to user                | Download Final Video (No Upscale) | Combine                    | User notification with final video delivery                                                     |
| Send Final Video Url      | Telegram                      | Sends clickable URL of final upscaled video              | Get Result                  | -                           | User notification with final video URL                                                        |
| Send message and wait for response | Telegram              | Asks user if they want to share video on social media    | Send Final Video Url          | Get All Social Media Accounts | Part of 📱 Social Distribution Network: Multi-Platform Publishing Agent                         |
| Get All Social Media Accounts | HTTP Request              | Retrieves connected social media accounts from Late.dev  | Send message and wait for response | Share to Instagram, TikTok, Youtube Shorts | Part of 📱 Social Distribution Network: Multi-Platform Publishing Agent                         |
| Share to Instagram       | HTTP Request                   | Posts video to Instagram Reels                            | Get All Social Media Accounts  | -                           | Part of 📱 Social Distribution Network: Multi-Platform Publishing Agent                         |
| Share to TikTok          | HTTP Request                   | Posts video to TikTok with detailed metadata             | Get All Social Media Accounts  | -                           | Part of 📱 Social Distribution Network: Multi-Platform Publishing Agent                         |
| Share to Youtube Shorts  | HTTP Request                   | Posts video to YouTube Shorts                             | Get All Social Media Accounts  | -                           | Part of 📱 Social Distribution Network: Multi-Platform Publishing Agent                         |

---

# 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials**  
   - Use @BotFather to create a Telegram bot and obtain token.  
   - Add Telegram API credentials in n8n with this token.

2. **Create HTTP Credentials**  
   - Add HTTP header authentication credentials for Veo3_FalAI (Fal.AI APIs).  
   - Add Google Palm API credentials for Google Gemini.  
   - Add Late.dev API credentials for social media publishing.

3. **Add Nodes for Input Reception:**  
   - Add **Telegram Trigger** node: listen for "message" updates, enable file download.  
   - Add **Request Photo** Telegram node: sendAndWait operation asking user to upload product photo (jpg/jpeg/webp) with custom form.  
   - Add **Set: Chat Vars** node: assign `chat_id` from Telegram message JSON.  
   - Connect Telegram Trigger → Request Photo → Set: Chat Vars.

4. **Configure ImgBB API Key:**  
   - Add **Set: YOUR API KEY (ImgBB)** node: store your ImgBB API key string.  
   - Connect Request Photo and Set: Chat Vars → Merge Vars + Photo1 → Set YOUR API KEY (ImgBB) → Merge API + Photo.

5. **Upload Original Image:**  
   - Add **Upload Original Image (imgbb)** HTTP Request node: POST multipart form-data to `https://api.imgbb.com/1/upload` with binary image from Telegram and API key from Set node.  
   - Connect Merge API + Photo → Upload Original Image (imgbb).

6. **Create AI Agents for Creative Direction:**  
   - Add **Creative Director (Orchestrator) agent** LangChain node configured with system prompt for orchestrating Art Director and Motion Designer sub-agents.  
   - Add **Art Director** and **Motion designer** LangChain agent tool nodes with their respective system prompts for visual and motion prompt generation.  
   - Add **Google Gemini Chat Model** nodes linked to respective agents, using "models/gemini-2.5-pro".  
   - Add **Structured Output Parser** node to parse JSON output.  
   - Connect Upload Original Image (imgbb) → Merge Vars + Photo1 → Creative Director (Orchestrator) agent → Set Storyboard Vars → start processing (Telegram node to notify user).

7. **Generate Edited Images:**  
   - Add **Gen Image A** and **Gen Image B** HTTP Request nodes calling Gemini Flash image editing API with prompt1 and prompt2 respectively.  
   - Add **Split Image A/B** nodes to split out images.  
   - Add **Download Image A/B** HTTP Request nodes configured to download images as binary.  
   - Add **Rename -> photo A/B** code nodes to rename binary properties.  
   - Add **Upload Edited A/B (imgbb)** HTTP Request nodes to upload edited images to ImgBB.  
   - Add **Send Image A/B** Telegram nodes to send images with captions "Concept A" and "Concept B".  
   - Connect Set Storyboard Vars → Merge Vars + Photo → Gen Image A/B → Split → Download → Rename → Upload Edited → Send Image.

8. **Implement Approval Gate:**  
   - Add **Ask Approval (Images)** Telegram node to request user approval with double approval buttons.  
   - Add **Approved?** IF node to check approval response.  
   - Add **Gate: Render Approved** merge node to continue only if approved.  
   - Add **Notify: Rejected** Telegram node to inform user if rejected.  
   - Connect Send Image A/B → Ask Approval → Approved? → Gate: Render Approved (true) or Notify: Rejected (false).

9. **Generate Videos with Kling AI:**  
   - Add **kling Queue A/B** HTTP Request nodes to submit video generation jobs with edited image URLs and kling prompts, aspect 9:16, duration 5s.  
   - Add **Wait A/B** nodes with 40-second delays to wait for processing.  
   - Add **Kling Status A/B** HTTP Request nodes to poll job status.  
   - Add **Animation Completed? A/B** IF nodes to check if status is "COMPLETED". Loop back to wait if not.  
   - Add **Kling Result A/B** HTTP Request nodes to get final video info.  
   - Add **Download A/B** HTTP Request nodes to download video binaries.  
   - Add **Send video Concept A/B** Telegram nodes to send video previews.  
   - Connect Gate: Render Approved → kling Queue A/B → Wait A/B → Kling Status A/B → Animation Completed? → Kling Result A/B → Download A/B → Send video Concept A/B.

10. **Compose Final Video:**  
    - Add **Merge Kling Results** and **Aggregate Results** nodes to combine video data.  
    - Add **FFmpeg Compose** HTTP Request node to stitch clips sequentially.  
    - Add **Notify: Stitching** Telegram node to update user.  
    - Add **Wait Compose** node (30s delay).  
    - Add **Get Video** HTTP Request node to retrieve composed video.  
    - Connect Download A/B → Merge Kling Results → Aggregate Results → FFmpeg Compose → Notify: Stitching → Wait Compose → Get Video.

11. **Generate Soundtrack and Upscale:**  
    - Add **Create Soundtrack** HTTP Request node to generate ambient audio with MMAudio API using storyboard sound prompt and composed video URL.  
    - Add **Wait Soundtrack** node (100s delay).  
    - Add **Get Soundtrack** HTTP Request node to retrieve audio.  
    - Add **Combine** merge node to merge video and audio data.  
    - Add **Upscale** HTTP Request node to upscale video 2x with Topaz AI API.  
    - Add **Wait2** node (3 minutes delay).  
    - Add **Get Result** HTTP Request node to get upscaled video.  
    - Add **Download Final Video (No Upscale)** HTTP Request node to download video.  
    - Add **Send Full Video (No Upscale)** Telegram node to send video to user.  
    - Add **Send Final Video Url** Telegram node to send video URL link.  
    - Connect Get Video → Create Soundtrack → Wait Soundtrack → Get Soundtrack → Combine → Upscale → Wait2 → Get Result → Download Final Video → Send Full Video (No Upscale) & Send Final Video Url.

12. **Social Media Sharing:**  
    - Add **Send message and wait for response** Telegram node to ask user if they want to share on socials.  
    - Add **Get All Social Media Accounts** HTTP Request node to retrieve accounts from Late.dev API.  
    - Add **Share to Instagram**, **Share to TikTok**, **Share to Youtube Shorts** HTTP Request nodes to post video.  
    - Connect Send Final Video Url → Send message and wait for response → Get All Social Media Accounts → Share to Instagram/TikTok/Youtube Shorts.

13. **Final Setup:**  
    - Configure all credentials properly (Telegram API, Google Palm API, Veo3_FalAI API, Late.dev API, ImgBB API).  
    - Replace all placeholder API keys with real values.  
    - Test each API node individually before running workflow end-to-end.

---

# 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| 🎯 Command Center hub manages user interaction through Telegram and initializes AI agents with product photos. Requires ImgBB API key setup for image hosting.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note1 in workflow                             |
| 🧠 Vision Intelligence Network uses Google Gemini 2.5 Pro multi-agent AI to generate cinematic visual and motion prompts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note2 in workflow                             |
| 🎨 Content Generation Engine creates two enhanced visual concepts through Gemini Flash AI image editing, preserving product identity and luxury brand aesthetics.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note3 in workflow                             |
| 🛡️ Human Oversight Gate enforces strategic approval checkpoint via Telegram to avoid unnecessary video rendering costs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note4 in workflow                             |
| 🎬 Cinematic Production Studio handles autonomous video creation with Kling AI, including job submission, polling, and downloading of final videos.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note6 in workflow                             |
| 🎵 Audio Intelligence Lab generates matching ambient soundtrack with MMAudio v2 and synchronizes audio with video composition.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note7 in workflow                             |
| ⚡ Quality Enhancement Engine performs autonomous 2x upscaling of videos using Topaz AI for premium output.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note8 in workflow                             |
| 📱 Social Distribution Network automates multi-platform publishing using Late.dev API for Instagram, TikTok, and YouTube Shorts with platform-specific metadata and privacy settings.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note9 in workflow, includes links: http://late.dev/, https://getlate.dev/docs |
| Bonus Quick Guide for Late.dev usage outlines API key retrieval, profile and account management, media upload options, posting examples, and troubleshooting tips.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note5 in workflow, https://getlate.dev/      |
| Installation instructions detail importing workflow, configuring credentials, setting up Telegram Bot, testing services, connecting Late.dev, and starting the workflow. Support contact emails provided for technical assistance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note11 in workflow                            |
| Workflow created by https://shortibrand.bilsimaging.com/ by bilsimaging.com with integrated AI services and Telegram interaction for premium product animation generation and social media automation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Branding and credits in Sticky Note                  |

---

**Disclaimer:** The provided content is exclusively from an automated n8n workflow. All data and processes comply with content policies and involve legal, public information only. No raw JSON included beyond necessary references.