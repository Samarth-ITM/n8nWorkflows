Generate Hollywood-Style Video Ads from Images with GPT-5 Mini and Fal.ai Sora-2

https://n8nworkflows.xyz/workflows/generate-hollywood-style-video-ads-from-images-with-gpt-5-mini-and-fal-ai-sora-2-11268


# Generate Hollywood-Style Video Ads from Images with GPT-5 Mini and Fal.ai Sora-2

### 1. Workflow Overview

This n8n workflow automates the generation of Hollywood-style video advertisements from user-submitted images and text using advanced AI models (GPT-5 Mini and Fal.ai Sora-2). The core functionality integrates image processing, AI prompt generation, video creation, and multi-channel delivery. The workflow is designed for marketing teams or content creators who want to rapidly produce high-quality video ads from simple inputs.

Logical blocks included:

- **1.1 Input Reception:** Receives ad text and image via webhook.
- **1.2 Image Processing:** Resizes and uploads the image for use in video generation.
- **1.3 AI Processing:** Uses OpenAI GPT-5 Mini and Fal.ai Sora-2 to generate video prompts and scripts.
- **1.4 Video Creation & Status Checking:** Sends prompts to video generation service, polls for completion.
- **1.5 Output and Notification:** Sends the completed video back to the requester and optionally delivers it via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures the initial user input — an advertisement text and an image — through a webhook to trigger the workflow.

- **Nodes Involved:**  
  - Get the Ad Text and Image (Webhook)

- **Node Details:**  
  - **Get the Ad Text and Image**  
    - Type: Webhook  
    - Role: Entry point to receive HTTP POST requests containing the advertisement text and image data.  
    - Configuration: Uses a unique webhook ID to listen for incoming requests.  
    - Inputs: External HTTP POST request with form data (likely image and text).  
    - Outputs: Passes the received data downstream for image processing and AI generation.  
    - Edge Cases: Missing or malformed payloads, unsupported image formats, or large files causing timeouts.

---

#### 1.2 Image Processing

- **Overview:**  
  This block resizes the input image to suitable dimensions and uploads it to a cloud storage service (Cloudinary) for accessibility in video generation.

- **Nodes Involved:**  
  - Image Resize  
  - Upload Image  
  - Edit Fields

- **Node Details:**  
  - **Image Resize**  
    - Type: Edit Image  
    - Role: Adjusts image size to meet video generation service requirements.  
    - Configuration: Preset or dynamic resizing parameters (not detailed).  
    - Inputs: Raw image from webhook.  
    - Outputs: Resized image.  
    - Edge Cases: Unsupported image formats, errors during resizing, file corruption.

  - **Upload Image**  
    - Type: Cloudinary  
    - Role: Uploads resized image to Cloudinary cloud storage and obtains a publicly accessible URL.  
    - Configuration: Requires Cloudinary credentials with proper access rights.  
    - Inputs: Resized image binary data.  
    - Outputs: Cloudinary URL and metadata.  
    - Edge Cases: Authentication errors, upload failures, network timeouts.

  - **Edit Fields**  
    - Type: Set  
    - Role: Prepares and formats data fields (likely adding or modifying metadata such as the Cloudinary URL) before merging with AI prompt data.  
    - Configuration: Custom field mapping or expression usage to combine image URL with other data.  
    - Inputs: Cloudinary upload result.  
    - Outputs: Formatted data for next merge step.

---

#### 1.3 AI Processing

- **Overview:**  
  This block uses GPT-5 Mini (OpenAI Chat Model) and Fal.ai Sora-2 agent to generate video prompt scripts based on the input text and image URL. The AI models collaborate to create a Hollywood-style video ad narrative.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Sora Video Prompt Generator  
  - Merge

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: langchain LM Chat OpenAI  
    - Role: Generates initial text-based input or enriches prompts for video generation.  
    - Configuration: Uses GPT-5 Mini model (or equivalent), no explicit parameters shown but likely includes temperature, max tokens, etc.  
    - Inputs: Raw text or structured data from previous nodes.  
    - Outputs: Text prompt for Sora agent.  
    - Edge Cases: API rate limits, authentication errors, invalid prompt formatting, model version mismatches.

  - **Sora Video Prompt Generator**  
    - Type: langchain Agent  
    - Role: Processes the OpenAI output and image data to formulate final video prompt commands tailored for the Fal.ai Sora-2 video generation engine.  
    - Configuration: Uses AI agent logic to combine inputs and generate complex prompt structures.  
    - Inputs: Output from OpenAI Chat Model and webhook inputs (concurrently).  
    - Outputs: Video prompt ready to send to video generation API.  
    - Edge Cases: Incomplete prompt generation, agent timeout, invalid prompt syntax.

  - **Merge**  
    - Type: Merge  
    - Role: Joins AI-generated prompts with the processed image metadata to prepare a complete payload for the video generation API.  
    - Configuration: Default merge mode (likely ‘append’).  
    - Inputs: Video prompt (Sora agent) and image URL/data fields (Edit Fields).  
    - Outputs: Combined payload for HTTP Request node.  
    - Edge Cases: Data mismatches, missing fields, merge conflicts.

---

#### 1.4 Video Creation & Status Checking

- **Overview:**  
  This block sends the combined prompt and image data to a video creation API, waits for video generation completion by polling status, and handles branching logic based on completion state.

- **Nodes Involved:**  
  - HTTP Request  
  - Wait1  
  - Get Video Status  
  - If1  
  - Get Final Video1

- **Node Details:**  
  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Sends the combined prompt/image payload to the external video generation API to initiate video creation.  
    - Configuration: POST request with authentication, endpoint URL (not detailed).  
    - Inputs: Merged prompt and image data.  
    - Outputs: Job or video generation request ID.  
    - Edge Cases: Network errors, invalid API keys, request timeouts, API rate limits.

  - **Wait1**  
    - Type: Wait  
    - Role: Delays workflow execution to allow video processing time before status check.  
    - Configuration: Fixed or dynamic wait duration (not detailed).  
    - Inputs: Response from HTTP Request or Get Final Video1 (retry loop).  
    - Outputs: Triggers next status check.  
    - Edge Cases: Inadequate wait leading to premature status check, excessive wait causing delays.

  - **Get Video Status**  
    - Type: HTTP Request  
    - Role: Polls the video generation API for the current processing status of the video job.  
    - Configuration: GET request with job ID, authentication.  
    - Inputs: Video job ID from initial HTTP Request or retry loops.  
    - Outputs: Status response indicating if video is ready.  
    - Edge Cases: API downtime, invalid job ID, network errors.

  - **If1**  
    - Type: If  
    - Role: Branches workflow based on the video status response.  
    - Configuration: Condition checks if video is ready.  
    - Inputs: Status data from Get Video Status.  
    - Outputs:  
      - True branch: Get Final Video1 (retrieve video).  
      - False branch: Wait1 (continue polling).  
    - Edge Cases: Incorrect condition evaluation, infinite loops if status never changes.

  - **Get Final Video1**  
    - Type: HTTP Request  
    - Role: Retrieves the finalized video file or video URL once the video is ready.  
    - Configuration: GET request to video API with job ID.  
    - Inputs: Triggered by If1 on completion.  
    - Outputs: Video file URL or binary data for delivery.  
    - Edge Cases: Retrieval failure, expired URLs, partial data.

---

#### 1.5 Output and Notification

- **Overview:**  
  This block sends the completed video back to the original webhook requester and optionally forwards a notification via Telegram.

- **Nodes Involved:**  
  - Send the Video Back  
  - Send a text message

- **Node Details:**  
  - **Send the Video Back**  
    - Type: Respond to Webhook  
    - Role: Returns the generated video or video URL as an HTTP response to the original webhook call.  
    - Configuration: Uses n8n’s webhook response node to deliver data synchronously.  
    - Inputs: Video data from Get Final Video1.  
    - Outputs: Triggers Telegram notification.  
    - Edge Cases: Large video sizes causing timeout, response formatting errors.

  - **Send a text message**  
    - Type: Telegram  
    - Role: Sends a Telegram message (likely with video link or status update) to a predefined chat/user.  
    - Configuration: Requires valid Telegram bot credentials and chat ID.  
    - Inputs: Video URL or notification text from Send the Video Back.  
    - Outputs: None (terminal).  
    - Edge Cases: Telegram API limits, invalid chat ID, bot permissions.

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                      | Input Node(s)                      | Output Node(s)                  | Sticky Note                         |
|---------------------------|---------------------------------|------------------------------------|----------------------------------|--------------------------------|-----------------------------------|
| Get the Ad Text and Image | Webhook                         | Entry point for ad text and image  | -                                | Image Resize, Sora Video Prompt Generator |                                   |
| Image Resize              | Edit Image                      | Resize input image                  | Get the Ad Text and Image        | Upload Image                   |                                   |
| Upload Image              | Cloudinary                     | Upload resized image to cloud      | Image Resize                    | Edit Fields                   |                                   |
| Edit Fields               | Set                            | Format and augment data fields     | Upload Image                    | Merge                         |                                   |
| OpenAI Chat Model         | Langchain LM Chat OpenAI        | Generate AI text prompt             | -                              | Sora Video Prompt Generator   |                                   |
| Sora Video Prompt Generator | Langchain Agent               | Generate video prompt from AI      | OpenAI Chat Model, Get the Ad Text and Image | Merge                 |                                   |
| Merge                    | Merge                          | Combine AI prompt and image data   | Edit Fields, Sora Video Prompt Generator | HTTP Request                |                                   |
| HTTP Request             | HTTP Request                   | Send video generation request      | Merge                          | Wait1                        |                                   |
| Wait1                    | Wait                          | Delay for video processing         | HTTP Request, Get Final Video1  | Get Video Status             |                                   |
| Get Video Status          | HTTP Request                   | Poll video generation status       | Wait1                         | If1                         |                                   |
| If1                      | If                            | Branch based on video status       | Get Video Status               | Get Final Video1 (true), Wait1 (false) |                                   |
| Get Final Video1          | HTTP Request                   | Retrieve final video file           | If1 (true)                    | Send the Video Back          |                                   |
| Send the Video Back       | Respond to Webhook             | Return video to original requester | Get Final Video1               | Send a text message          |                                   |
| Send a text message       | Telegram                       | Notify via Telegram                 | Send the Video Back            | -                           |                                   |
| Sticky Note, Sticky Note1-10 | Sticky Note                 | Comments/annotations                | -                              | -                            | Various (content not provided)    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node: Get the Ad Text and Image**  
   - Type: Webhook  
   - Purpose: Accept POST requests containing ad text and image.  
   - Configure webhook URL and method (POST).  
   - Save and activate the workflow to generate webhook URL.

2. **Add Image Resize Node**  
   - Type: Edit Image  
   - Connect input from webhook node.  
   - Configure resizing parameters (e.g., width, height) appropriate for video input.

3. **Add Cloudinary Upload Node: Upload Image**  
   - Type: Cloudinary  
   - Connect input from Image Resize node.  
   - Configure Cloudinary credentials (API key, secret, cloud name).  
   - Set upload options (folder, public ID if needed).

4. **Add Set Node: Edit Fields**  
   - Type: Set  
   - Connect input from Upload Image node.  
   - Add or modify fields to prepare final data format (e.g., add Cloudinary URL under a specific key).

5. **Add OpenAI Chat Model Node**  
   - Type: Langchain LM Chat OpenAI  
   - No direct input (or optionally from webhook text field).  
   - Configure OpenAI credentials (API key).  
   - Select GPT-5 Mini or equivalent model.  
   - Set parameters like temperature, max tokens as needed.

6. **Add Sora Video Prompt Generator Node**  
   - Type: Langchain Agent  
   - Configure to receive inputs from OpenAI Chat Model and webhook node concurrently.  
   - This node should generate the video prompt script using combined AI outputs and user input.

7. **Add Merge Node**  
   - Type: Merge  
   - Connect inputs from Edit Fields node and Sora Video Prompt Generator node.  
   - Default merge mode (append).

8. **Add HTTP Request Node**  
   - Type: HTTP Request  
   - Connect input from Merge node.  
   - Configure POST request to video generation API endpoint.  
   - Include authentication headers or tokens.  
   - Payload includes merged prompt and image URLs.

9. **Add Wait Node (Wait1)**  
   - Type: Wait  
   - Connect input from HTTP Request node.  
   - Configure delay duration (e.g., 30 seconds or as needed).

10. **Add HTTP Request Node (Get Video Status)**  
    - Type: HTTP Request  
    - Connect input from Wait node.  
    - Configure GET request to video API status endpoint with job ID.

11. **Add If Node (If1)**  
    - Type: If  
    - Connect input from Get Video Status node.  
    - Condition to check if video generation status is "ready" or equivalent.

12. **Add HTTP Request Node (Get Final Video1)**  
    - Type: HTTP Request  
    - Connect input from If1 node on true branch.  
    - Configure GET request to retrieve final video file or URL.

13. **Add Respond to Webhook Node (Send the Video Back)**  
    - Type: Respond to Webhook  
    - Connect input from Get Final Video1 node.  
    - Configure to send video data or URL as response to initial webhook call.

14. **Add Telegram Node (Send a text message)**  
    - Type: Telegram  
    - Connect input from Respond to Webhook node.  
    - Configure Telegram bot credentials and chat ID.  
    - Set message content, e.g., video link or notification text.

15. **Connect If1 false branch back to Wait1** to implement polling loop until video is ready.

16. **Test the entire workflow with sample POST requests containing image and ad text.**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow uses GPT-5 Mini and Fal.ai Sora-2, ensure API keys and credentials are valid and active. | AI model integration                             |
| Cloudinary is used for image hosting; ensure Cloudinary account and credentials are properly set. | Image upload and hosting                          |
| Telegram notifications require a Telegram Bot token and chat ID; set up a bot via BotFather.     | Telegram integration                             |
| Video generation API endpoints and authentication must be configured as per provider documentation. | External API integration                          |
| Consider adding error handling nodes or workflows for timeouts, API failures, or invalid inputs. | Workflow robustness and reliability              |

---

_This document is based exclusively on the n8n workflow JSON provided. All data processed is legal and publicly accessible._