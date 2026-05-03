Generate Videos from Text Prompts using GPT-5 and Google Veo-3

https://n8nworkflows.xyz/workflows/generate-videos-from-text-prompts-using-gpt-5-and-google-veo-3-7328


# Generate Videos from Text Prompts using GPT-5 and Google Veo-3

### 1. Workflow Overview

This n8n workflow automates the generation of videos from minimal text prompts using advanced AI models GPT-5 and Google Veo-3 image-to-video API. It is designed to start with a brief chat input—sometimes just one word—and transform it into a cinematic video prompt, then create a video through the Veo-3 API, poll for its completion, and finally retrieve the generated video file URL.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception & Minimal Prompt Setup**  
  Begins with a chat trigger that accepts user input, sets a default or provided image URL, and prepares the minimal data needed for video creation.

- **1.2 AI Prompt Expansion with GPT-5**  
  Uses the GPT-5 model to creatively expand the minimal input into a rich, cinematic prompt tailored for Veo-3 video generation.

- **1.3 Video Generation via Google Veo-3 API**  
  Sends the cinematic prompt along with an image URL to the Veo-3 API to start video generation.

- **1.4 Polling and Status Checking**  
  Repeatedly polls the Veo-3 API to check if the video generation has completed or encountered an error.

- **1.5 Video Retrieval and Error Handling**  
  Upon successful completion, downloads the generated video file. Handles errors by stopping the workflow and reporting the error message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Minimal Prompt Setup

- **Overview:**  
  Captures user input via chat, sets a default image URL for video generation, and prepares initial data.

- **Nodes Involved:**  
  - When chat message received  
  - Set image URL  
  - Sticky Note (STEP 1: CHAT INPUT → MINIMAL PROMPT)

- **Node Details:**

  - **When chat message received**  
    - Type: Chat Trigger (Langchain)  
    - Role: Entry point for receiving chat messages with user prompts.  
    - Config: Default options; expects flexible fields like `chatInput` (required), `image_url` (optional), `duration` (optional), `aspect_ratio` (optional).  
    - Input: Incoming webhook/chat message.  
    - Output: JSON with chat input fields.  
    - Edge Cases: Missing `chatInput` may cause downstream issues; no hard validation performed here.

  - **Set image URL**  
    - Type: Set Node  
    - Role: Assigns a default image URL value if none provided by the user, used as source for image-to-video.  
    - Config: Hardcoded URL `"https://s2-111386.kwimgs.com/bs2/mmu-aiplatform-temp/kling/20240620/1.jpeg"`.  
    - Input: Output of chat trigger node.  
    - Output: Adds `image_url` to JSON, passed downstream.  
    - Edge Cases: If user provides `image_url`, it is overwritten here; customization may be required.

  - **Sticky Note (STEP 1)**  
    - Type: Sticky Note  
    - Role: Provides contextual instructions describing expected input fields and workflow purpose.

---

#### 2.2 AI Prompt Expansion with GPT-5

- **Overview:**  
  Expands minimal user input into a rich, cinematic prompt suitable for Veo-3 video generation, enhancing visual and stylistic details.

- **Nodes Involved:**  
  - Cinematic Prompt (GPT-5)  
  - Sticky Note21 (GPT-5 usage instructions)

- **Node Details:**

  - **Cinematic Prompt (GPT-5)**  
    - Type: HTTP Request to AI/ML API (aimlApi node)  
    - Role: Sends a prompt to GPT-5 model to generate a visually rich, cinematic video prompt.  
    - Config:  
      - Model: `openai/gpt-5-mini-2025-08-07`  
      - Prompt: Custom template including input chat text.  
      - Max tokens: 5000  
      - Authentication: Uses predefined AI/ML API credential with bearer token.  
    - Key Expressions: Uses expression to inject chat input: `={{ $('When chat message received').item.json.chatInput }}`  
    - Input: Receives JSON with chat input and image URL.  
    - Output: Cinematic prompt text.  
    - Edge Cases: If GPT returns empty or invalid prompt, the workflow must fallback or fail gracefully. API errors or quota limits can cause failures.

  - **Sticky Note21**  
    - Provides instructions on GPT-5 usage for prompt expansion.

---

#### 2.3 Video Generation via Google Veo-3 API

- **Overview:**  
  Sends the cinematic prompt and image URL to the Veo-3 image-to-video generation API to create a new video.

- **Nodes Involved:**  
  - Create Video with AI/ML API1  
  - Sticky Note18 (block title and instructions)  
  - Sticky Note7 (API key reminder)

- **Node Details:**

  - **Create Video with AI/ML API1**  
    - Type: HTTP Request  
    - Role: POST request to start video generation on Veo-3 API.  
    - Config:  
      - URL: `https://api.aimlapi.com/v2/generate/video/google/generation`  
      - Method: POST  
      - Body Parameters:  
        - `prompt`: cinematic prompt or fallback to chat input  
        - `model`: `"google/veo-3.0-i2v-fast"`  
        - `image_url`: from Set image URL node  
      - Headers: Content-Type `application/json`  
      - Authentication: Uses AI/ML API credentials with Bearer token.  
    - Input: Uses expanded prompt and image URL.  
    - Output: JSON including generation ID for polling.  
    - Edge Cases: Auth errors, malformed body, network issues, API quota limits.

  - **Sticky Note7**  
    - Reminder to set API key in credentials.

  - **Sticky Note18**  
    - Describes this block as the main video generation step.

---

#### 2.4 Polling and Status Checking

- **Overview:**  
  Polls the Veo-3 API status endpoint every 30 seconds to check if video generation is complete or errored.

- **Nodes Involved:**  
  - Wait 30 sec.1  
  - Get status1  
  - Completed?  
  - Error?  
  - Sticky Note19 (main flow instructions)

- **Node Details:**

  - **Wait 30 sec.1**  
    - Type: Wait Node  
    - Role: Pauses workflow for 30 seconds between polls.  
    - Config: Fixed wait time of 30 seconds.  
    - Input: After video creation request.  
    - Output: Triggers status check node.

  - **Get status1**  
    - Type: HTTP Request  
    - Role: GET request to Veo-3 API status endpoint to check generation status.  
    - Config:  
      - URL: `https://api.aimlapi.com/v2/generate/video/google/generation`  
      - Query Parameter: `generation_id` from previous step.  
      - Authentication: AI/ML API credentials.  
    - Input: Wait node output with generation ID.  
    - Output: JSON with status field (`completed`, `error`, or others).  
    - Edge Cases: Network errors, invalid generation ID, auth issues.

  - **Completed?**  
    - Type: If Node  
    - Role: Checks if status equals `completed`.  
    - Input: Status JSON.  
    - True Output: Proceed to get video file.  
    - False Output: Pass to Error? node.

  - **Error?**  
    - Type: If Node  
    - Role: Checks if status equals `error`.  
    - Input: Status JSON.  
    - True Output: Stop and Error node.  
    - False Output: Loop back to Wait 30 sec.1 for another poll.

  - **Sticky Note19**  
    - Explains the main workflow and polling logic.

---

#### 2.5 Video Retrieval and Error Handling

- **Overview:**  
  Downloads the completed video file and handles errors by stopping the workflow and reporting error messages.

- **Nodes Involved:**  
  - Get Video File  
  - Stop and Error  
  - Sticky Note9 (download instructions)

- **Node Details:**

  - **Get Video File**  
    - Type: HTTP Request  
    - Role: Downloads the video file from the `video.url` provided in the completed status response.  
    - Config:  
      - URL: Dynamic, from `{{$json.video.url}}`  
      - No special authentication (assumed public or token handled upstream).  
    - Input: Completed? node true branch output.  
    - Output: Binary video file (downloaded content).  
    - Edge Cases: Invalid or expired URL, network errors.

  - **Stop and Error**  
    - Type: Stop and Error Node  
    - Role: Stops workflow execution and reports error message from API status.  
    - Config: Error message extracted from `{{$json.error.message}}`.  
    - Input: Error? node true branch output.

  - **Sticky Note9**  
    - Describes how to get and download the video.

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                                  | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                 |
|-------------------------------|---------------------------|-------------------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------|
| When chat message received     | Chat Trigger (Langchain)  | Entry point; receives user chat input           | —                           | Set image URL                |                                                                                             |
| Set image URL                 | Set Node                  | Assigns default image URL                         | When chat message received   | Cinematic Prompt (GPT-5)     |                                                                                             |
| Cinematic Prompt (GPT-5)       | HTTP Request (AI/ML API)  | Expands user input into cinematic video prompt  | Set image URL                | Create Video with AI/ML API1 | Use GPT‑5 turbo to expand the description into a Veo‑3‑friendly prompt (camera, movement, mood, composition). |
| Create Video with AI/ML API1   | HTTP Request              | Initiates video generation on Veo-3 API          | Cinematic Prompt (GPT-5)     | Wait 30 sec.1                | Set API Key created in Step 2                                                               |
| Wait 30 sec.1                 | Wait Node                 | Waits 30 seconds before polling status           | Create Video with AI/ML API1 | Get status1                  |                                                                                             |
| Get status1                  | HTTP Request              | Polls Veo-3 API for video generation status      | Wait 30 sec.1               | Completed?                   |                                                                                             |
| Completed?                   | If Node                   | Checks if video generation is completed          | Get status1                 | Get Video File / Error?      |                                                                                             |
| Error?                       | If Node                   | Checks if video generation encountered an error | Completed?                  | Stop and Error / Wait 30 sec.1 |                                                                                             |
| Stop and Error               | Stop and Error Node       | Stops workflow and reports error                  | Error?                      | —                           |                                                                                             |
| Get Video File               | HTTP Request              | Downloads the completed video file                | Completed?                  | —                           | ## Get and download your video                                                             |
| Sticky Note7                 | Sticky Note               | Instruction: Set API Key Reminder                  | —                           | —                           | Set API Key created in Step 2                                                               |
| Sticky Note9                 | Sticky Note               | Instruction: Get and download your video           | —                           | —                           | ## Get and download your video                                                             |
| Sticky Note10                | Sticky Note               | Block title: Start and Prompt Enhancing            | —                           | —                           | ### Start and Prompt Enhancing                                                             |
| Sticky Note18                | Sticky Note               | Block title: Generate Video via VEO-3               | —                           | —                           | # Generate Video via VEO-3                                                                   |
| Sticky Note19                | Sticky Note               | Main flow explanation and polling overview          | —                           | —                           | ## STEP 4: MAIN FLOW (END‑TO‑END)                                                           |
| Sticky Note20                | Sticky Note               | Credentials setup instructions                      | —                           | —                           | ## STEP 2: AI/ML API CREDENTIALS                                                           |
| Sticky Note21                | Sticky Note               | GPT-5 prompt expansion instructions                 | —                           | —                           | Use GPT‑5 turbo to expand the description into a Veo‑3‑friendly prompt (camera, movement, mood, composition). |
| Sticky Note22                | Sticky Note               | Optional save & publish instructions                | —                           | —                           | ## STEP 3 (OPTIONAL): SAVE & PUBLISH                                                       |
| Sticky Note23                | Sticky Note               | Workflow description and robustness summary         | —                           | —                           | **Veo‑3 Image‑to‑Video with GPT‑5 (Chat‑First)**<br>**What this workflow does**<br>...     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Chat Trigger Node**  
   - Type: Langchain Chat Trigger (`@n8n/n8n-nodes-langchain.chatTrigger`)  
   - Purpose: Start the workflow upon receiving a chat message.  
   - Configuration: Use default settings. Expect JSON input with at least `chatInput` field.

2. **Add a Set Node ("Set image URL")**  
   - Assign a default image URL variable named `image_url` with value:  
     `https://s2-111386.kwimgs.com/bs2/mmu-aiplatform-temp/kling/20240620/1.jpeg`  
   - Connect output of Chat Trigger to this Set node.

3. **Add HTTP Request Node for GPT-5 Prompt Expansion ("Cinematic Prompt (GPT-5)")**  
   - Type: HTTP Request (AI/ML API)  
   - URL: Use the AI/ML API endpoint for GPT-5 model (`openai/gpt-5-mini-2025-08-07`)  
   - Method: POST  
   - Body: JSON with a prompt template that includes the chat input expression:  
     ```
     You are a creative video prompt engineer.

     Turn the idea below into a short, visually rich description for Veo‑3 image‑to‑video.
     Keep it flexible and cinematic.

     Guidelines (use loosely):
     - mention camera movement / lens hints
     - time of day, mood, color palette
     - pacing / motion style
     - framing and composition

     Keep it to 2–4 sentences (under 120 words). If the input is just one word, expand it sensibly.

     Idea:
     {{chatInput}}
     ```
   - Max tokens: 5000  
   - Authentication: Use AI/ML API Bearer token credential (`AI/ML account`)  
   - Connect Set node output to this node.

4. **Add HTTP Request Node to Create Video ("Create Video with AI/ML API1")**  
   - Type: HTTP Request  
   - URL: `https://api.aimlapi.com/v2/generate/video/google/generation`  
   - Method: POST  
   - Headers: Content-Type: `application/json`  
   - Body Parameters:  
     - `prompt`: Use output from GPT-5 node or fallback to original chat input if empty  
     - `model`: `"google/veo-3.0-i2v-fast"`  
     - `image_url`: Use the value from Set image URL node  
   - Authentication: Use AI/ML API Bearer token credential  
   - Connect GPT-5 node output to this node.

5. **Add Wait Node ("Wait 30 sec.1")**  
   - Type: Wait  
   - Duration: 30 seconds  
   - Connect Create Video node output to Wait node.

6. **Add HTTP Request Node to Poll Status ("Get status1")**  
   - Type: HTTP Request  
   - URL: `https://api.aimlapi.com/v2/generate/video/google/generation`  
   - Method: GET  
   - Query Parameter: `generation_id` from Create Video node output  
   - Authentication: Use AI/ML API Bearer token credential  
   - Connect Wait node output to this node.

7. **Add If Node ("Completed?")**  
   - Condition: Check if `$json.status == "completed"`  
   - True branch: Proceed to download video  
   - False branch: Connect to next If node for error checking  
   - Connect Get Status node output to this node.

8. **Add If Node ("Error?")**  
   - Condition: Check if `$json.status == "error"`  
   - True branch: Connect to Stop and Error node  
   - False branch: Loop back to Wait node to poll again  
   - Connect Completed? node false branch to Error? node.

9. **Add Stop and Error Node ("Stop and Error")**  
   - Configuration:  
     - Error message set to `$json.error.message` from status response  
   - Connect Error? node true branch here.

10. **Add HTTP Request Node to Download Video ("Get Video File")**  
    - Type: HTTP Request  
    - URL: Extract dynamic URL from status response JSON: `{{$json.video.url}}`  
    - Connect Completed? node true branch here.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Get your AI/ML API key from https://aimlapi.com/app/keys and configure it in n8n credentials as "AI/ML account" with Bearer token authentication. Avoid hardcoding keys in nodes for security.                                                                                                                                                                   | Credential Setup Instructions                                   |
| This workflow supports minimal input (even a single word) and uses GPT-5 to expand it into a cinematic prompt optimized for Google Veo-3 video generation.                                                                                                                                                                                                   | Workflow Design Principle                                       |
| Optional extensions include saving generated videos to Google Drive, generating SEO-friendly titles with GPT-5, publishing videos to YouTube, or logging analytics to sheets/databases. These are not part of the core workflow.                                                                                                                                 | Suggested Enhancements                                         |
| Polling interval is set to 30 seconds but can be adjusted based on SLA, API quota, or responsiveness requirements.                                                                                                                                                                                                                                            | Polling Configuration Advice                                   |
| See sticky notes within the workflow for detailed step explanations, usage instructions, and best practices (e.g., prompt crafting, error handling).                                                                                                                                                                                                          | In-workflow Documentation                                      |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.