Automated Faceless YouTube Video Generator Using Leonardo AI and Creatomate

https://n8nworkflows.xyz/workflows/automated-faceless-youtube-video-generator-using-leonardo-ai-and-creatomate-2971


# Automated Faceless YouTube Video Generator Using Leonardo AI and Creatomate

### 1. Workflow Overview

This workflow automates the creation of faceless YouTube videos by integrating AI-driven scriptwriting, image generation, and video editing services. It is designed for content creators and marketers who want to scale video production without manual video shooting or editing.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & AI Script Generation:** Initiates the workflow on a schedule, uses an AI language model to generate a video script with multiple scenarios.
- **1.2 AI Image Generation (Leonardo AI):** For each scenario, generates images using Leonardo AI, waits for image processing, and retrieves image URLs.
- **1.3 Data Preparation:** Edits and formats the image and script data to prepare for video generation.
- **1.4 Video Generation (Creatomate):** Sends the prepared data to Creatomate to generate the final video.
- **1.5 Video Retrieval & Upload:** Retrieves the generated video from Creatomate and uploads it to YouTube.
- **1.6 Notifications:** Sends notifications via Telegram about the video generation status.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & AI Script Generation

- **Overview:** This block triggers the workflow on a schedule and generates a video script using an AI language model with structured output parsing.
- **Nodes Involved:** Schedule Trigger, AI Agent, OpenAI Chat Model, Structured Output Parser
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Starts the workflow automatically at defined intervals.
    - Configuration: Default schedule parameters (not explicitly detailed).
    - Inputs: None (trigger node).
    - Outputs: Triggers AI Agent node.
    - Edge Cases: Misconfigured schedule may cause no triggers; time zone issues.
  
  - **AI Agent**
    - Type: Langchain Agent
    - Role: Orchestrates AI model calls and output parsing.
    - Configuration: Uses OpenAI Chat Model as language model and Structured Output Parser for parsing.
    - Inputs: Trigger from Schedule Trigger.
    - Outputs: Initiates image generation nodes and merges results.
    - Edge Cases: AI model failures, rate limits, parsing errors.
  
  - **OpenAI Chat Model**
    - Type: Langchain OpenAI Chat Model
    - Role: Generates the video script based on system prompt and parameters.
    - Configuration: Uses DeepSeek V3 or other configured LLM.
    - Inputs: From AI Agent.
    - Outputs: Script text for parsing.
    - Edge Cases: Authentication errors, API limits, malformed prompts.
  
  - **Structured Output Parser**
    - Type: Langchain Structured Output Parser
    - Role: Parses AI-generated script into structured scenarios.
    - Inputs: Output from OpenAI Chat Model.
    - Outputs: Structured data passed back to AI Agent.
    - Edge Cases: Parsing failures if AI output format changes.

---

#### 1.2 AI Image Generation (Leonardo AI)

- **Overview:** Generates images for each scenario using Leonardo AI, waits for processing, and retrieves image URLs.
- **Nodes Involved:** Generate Image 1-6 - Leonardo AI, Wait nodes (Wait, Wait1, Wait4, Wait5, Wait7, Wait8), Get Image 1-6 URL - Leonardo AI
- **Node Details:**

  For each of the six images:

  - **Generate Image X - Leonardo AI**
    - Type: HTTP Request
    - Role: Sends image generation request to Leonardo AI API.
    - Configuration: Uses Leonardo AI credentials, model, prompt, and resolution parameters.
    - Inputs: From AI Agent node (script scenarios).
    - Outputs: Initiates Wait node.
    - Edge Cases: API authentication errors, request timeouts, invalid prompts.
  
  - **Wait / WaitX**
    - Type: Wait
    - Role: Pauses workflow to allow Leonardo AI to process image generation.
    - Configuration: Wait time configured to match Leonardo AI processing time.
    - Inputs: From Generate Image node.
    - Outputs: Triggers Get Image URL node.
    - Edge Cases: Insufficient wait time causing premature URL retrieval.
  
  - **Get Image X URL - Leonardo AI**
    - Type: HTTP Request
    - Role: Retrieves the generated image URL from Leonardo AI.
    - Configuration: Uses Leonardo AI API with job ID or similar identifier.
    - Inputs: From Wait node.
    - Outputs: Passes image URL to Edit Fields node.
    - Edge Cases: API errors, image not ready, invalid job ID.

---

#### 1.3 Data Preparation

- **Overview:** Edits and formats the image URLs and associated data for video generation.
- **Nodes Involved:** Edit Fields1, Edit Fields2, Edit Fields3, Edit Fields, Edit Fields4, Edit Fields6, Merge
- **Node Details:**

  For each image URL:

  - **Edit FieldsX**
    - Type: Set
    - Role: Adjusts fields such as image URL, scenario text, or metadata to prepare for video assembly.
    - Configuration: Sets or modifies key-value pairs for downstream nodes.
    - Inputs: From corresponding Get Image URL node.
    - Outputs: Connects to Merge node.
    - Edge Cases: Missing or malformed data causing incorrect field edits.
  
  - **Merge**
    - Type: Merge
    - Role: Combines all edited scenario/image data into a single data set for video generation.
    - Configuration: Default merge mode, always outputs data.
    - Inputs: From all Edit Fields nodes.
    - Outputs: To Creatomate video generation node.
    - Edge Cases: Data mismatch or missing inputs causing incomplete merges.

---

#### 1.4 Video Generation (Creatomate)

- **Overview:** Sends the merged script and image data to Creatomate to generate the final video.
- **Nodes Involved:** Cre - Generate Video1, Wait6, Cre - Get Video
- **Node Details:**

  - **Cre - Generate Video1**
    - Type: HTTP Request
    - Role: Calls Creatomate API to start video generation using a pre-created template.
    - Configuration: Uses Creatomate credentials, template cURL, and merged data payload.
    - Inputs: From Merge node.
    - Outputs: Triggers Wait6 node.
    - Edge Cases: API authentication errors, invalid template, payload errors.
  
  - **Wait6**
    - Type: Wait
    - Role: Waits for Creatomate to finish video processing.
    - Configuration: Wait time set to expected video generation duration.
    - Inputs: From Cre - Generate Video1.
    - Outputs: Triggers Cre - Get Video node.
    - Edge Cases: Insufficient wait time causing premature video retrieval.
  
  - **Cre - Get Video**
    - Type: HTTP Request
    - Role: Retrieves the generated video URL or file from Creatomate.
    - Configuration: Uses Creatomate API and job ID.
    - Inputs: From Wait6.
    - Outputs: To YouTube upload node.
    - Edge Cases: API errors, video not ready, invalid job ID.

---

#### 1.5 Video Retrieval & Upload

- **Overview:** Uploads the generated video to YouTube.
- **Nodes Involved:** YouTube
- **Node Details:**

  - **YouTube**
    - Type: YouTube Node
    - Role: Uploads the final video to a configured YouTube channel.
    - Configuration: Requires OAuth2 credentials for YouTube, video metadata (title, description).
    - Inputs: From Cre - Get Video node.
    - Outputs: None (end node).
    - Edge Cases: Authentication errors, upload failures, quota limits.

---

#### 1.6 Notifications

- **Overview:** Sends notifications about the video generation status via Telegram.
- **Nodes Involved:** Telegram
- **Node Details:**

  - **Telegram**
    - Type: Telegram Node
    - Role: Sends messages to a Telegram chat or channel.
    - Configuration: Requires Telegram Bot credentials and chat ID.
    - Inputs: Connected downstream of video generation or upload nodes (position suggests notification after video generation).
    - Outputs: None.
    - Edge Cases: Bot token invalid, chat ID incorrect, message send failures.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                      | Input Node(s)                          | Output Node(s)                        | Sticky Note                          |
|----------------------------|----------------------------------|------------------------------------|--------------------------------------|-------------------------------------|------------------------------------|
| Schedule Trigger            | Schedule Trigger                 | Starts workflow on schedule        | None                                 | AI Agent                            |                                    |
| AI Agent                   | Langchain Agent                  | Orchestrates AI script generation  | Schedule Trigger, OpenAI Chat Model, Structured Output Parser | Generate Image nodes, Merge         |                                    |
| OpenAI Chat Model          | Langchain OpenAI Chat Model      | Generates video script             | AI Agent (ai_languageModel)           | Structured Output Parser            |                                    |
| Structured Output Parser   | Langchain Structured Output Parser| Parses AI script output            | OpenAI Chat Model (ai_outputParser)  | AI Agent                           |                                    |
| Generate Image 1 - LeonadoAI | HTTP Request                    | Requests image generation          | AI Agent                             | Wait                               |                                    |
| Wait                       | Wait                            | Waits for image processing         | Generate Image 1 - Leonardo AI       | Get Image 1 URL - Leonardo AI      |                                    |
| Get Image 1 URL - LeonadoAI | HTTP Request                    | Retrieves generated image URL      | Wait                                | Edit Fields1                      |                                    |
| Edit Fields1               | Set                             | Prepares image 1 data              | Get Image 1 URL - Leonardo AI        | Merge                             |                                    |
| Generate Image 2 - LeonadoAI | HTTP Request                    | Requests image generation          | AI Agent                             | Wait1                              |                                    |
| Wait1                      | Wait                            | Waits for image processing         | Generate Image 2 - Leonardo AI       | Get Image 2 URL - Leonardo AI      |                                    |
| Get Image 2 URL - LeonadoAI | HTTP Request                    | Retrieves generated image URL      | Wait1                               | Edit Fields2                      |                                    |
| Edit Fields2               | Set                             | Prepares image 2 data              | Get Image 2 URL - Leonardo AI        | Merge                             |                                    |
| Generate Image 3 - LeonadoAI | HTTP Request                    | Requests image generation          | AI Agent                             | Wait4                              |                                    |
| Wait4                      | Wait                            | Waits for image processing         | Generate Image 3 - Leonardo AI       | Get Image 3 URL - Leonardo AI      |                                    |
| Get Image 3 URL - LeonadoAI | HTTP Request                    | Retrieves generated image URL      | Wait4                               | Edit Fields3                      |                                    |
| Edit Fields3               | Set                             | Prepares image 3 data              | Get Image 3 URL - Leonardo AI        | Merge                             |                                    |
| Generate Image 4 - LeonadoAI | HTTP Request                    | Requests image generation          | AI Agent                             | Wait5                              |                                    |
| Wait5                      | Wait                            | Waits for image processing         | Generate Image 4 - Leonardo AI       | Get Image 4 URL - Leonardo AI      |                                    |
| Get Image 4 URL - LeonadoAI | HTTP Request                    | Retrieves generated image URL      | Wait5                               | Edit Fields                      |                                    |
| Edit Fields                | Set                             | Prepares image 4 data              | Get Image 4 URL - Leonardo AI        | Merge                             |                                    |
| Generate Image 5 - LeonadoAI | HTTP Request                    | Requests image generation          | AI Agent                             | Wait7                              |                                    |
| Wait7                      | Wait                            | Waits for image processing         | Generate Image 5 - Leonardo AI       | Get Image 5 URL - Leonardo AI      |                                    |
| Get Image 5 URL - LeonadoAI | HTTP Request                    | Retrieves generated image URL      | Wait7                               | Edit Fields4                     |                                    |
| Edit Fields4               | Set                             | Prepares image 5 data              | Get Image 5 URL - Leonardo AI        | Merge                             |                                    |
| Generate Image 6 - LeonadoAI | HTTP Request                    | Requests image generation          | AI Agent                             | Wait8                              |                                    |
| Wait8                      | Wait                            | Waits for image processing         | Generate Image 6 - Leonardo AI       | Get Image 6 URL - Leonardo AI      |                                    |
| Get Image 6 URL - LeonadoAI | HTTP Request                    | Retrieves generated image URL      | Wait8                               | Edit Fields6                     |                                    |
| Edit Fields6               | Set                             | Prepares image 6 data              | Get Image 6 URL - Leonardo AI        | Merge                             |                                    |
| Merge                      | Merge                           | Combines all image and script data | Edit Fields1-6                      | Cre - Generate Video1              |                                    |
| Cre - Generate Video1      | HTTP Request                    | Starts video generation in Creatomate | Merge                             | Wait6                             |                                    |
| Wait6                      | Wait                            | Waits for video generation         | Cre - Generate Video1                | Cre - Get Video                   |                                    |
| Cre - Get Video            | HTTP Request                    | Retrieves generated video          | Wait6                              | YouTube                          |                                    |
| YouTube                   | YouTube Node                    | Uploads video to YouTube           | Cre - Get Video                     | None                             |                                    |
| Telegram                  | Telegram Node                   | Sends notification messages        | Position suggests after video generation | None                             |                                    |
| Sticky Note (various)     | Sticky Note                     | Comments and notes                 | None                              | None                             | See notes below                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Configure trigger interval as desired (e.g., daily).
   - No credentials needed.

2. **Create AI Agent Node**
   - Type: Langchain Agent
   - Connect Schedule Trigger output to AI Agent input.
   - Configure AI Agent to use:
     - Language Model: OpenAI Chat Model node (configured next).
     - Output Parser: Structured Output Parser node (configured next).

3. **Create OpenAI Chat Model Node**
   - Type: Langchain OpenAI Chat Model
   - Configure with OpenAI credentials.
   - Set model to DeepSeek V3 or preferred LLM.
   - Define prompt template to generate video script with multiple scenarios.
   - Connect AI Agent’s ai_languageModel input to this node.

4. **Create Structured Output Parser Node**
   - Type: Langchain Structured Output Parser
   - Configure to parse the AI script output into structured JSON scenarios.
   - Connect OpenAI Chat Model’s output to this node.
   - Connect AI Agent’s ai_outputParser input to this node.

5. **Create Leonardo AI Image Generation Nodes (6 sets)**
   For each image (1 to 6):
   - Create HTTP Request node named "Generate Image X - Leonardo AI"
     - Configure with Leonardo AI API credentials.
     - Set method and endpoint to request image generation.
     - Use scenario text and image style prompts from AI Agent output.
   - Create Wait node named "WaitX"
     - Configure wait time to allow image generation (e.g., 30-60 seconds).
   - Create HTTP Request node named "Get Image X URL - Leonardo AI"
     - Configure to retrieve generated image URL using job ID from previous node.
   - Connect nodes in order: Generate Image X → WaitX → Get Image X URL.

6. **Create Edit Fields Nodes (6)**
   For each image URL node:
   - Create Set node named "Edit FieldsX"
   - Configure to set or modify fields such as image URL, scenario text, or metadata.
   - Connect Get Image X URL node output to Edit FieldsX node.

7. **Create Merge Node**
   - Type: Merge
   - Configure to combine all Edit Fields nodes outputs.
   - Connect all Edit Fields nodes outputs to Merge node inputs.

8. **Create Creatomate Video Generation Nodes**
   - Create HTTP Request node "Cre - Generate Video1"
     - Configure with Creatomate API credentials.
     - Use pre-created "AI generated story template" cURL and payload from Merge node.
   - Create Wait node "Wait6"
     - Configure wait time for video generation (e.g., several minutes).
   - Create HTTP Request node "Cre - Get Video"
     - Configure to retrieve generated video URL or file.
   - Connect nodes: Merge → Cre - Generate Video1 → Wait6 → Cre - Get Video.

9. **Create YouTube Upload Node**
   - Type: YouTube
   - Configure with OAuth2 credentials for YouTube.
   - Set video metadata fields (title, description).
   - Connect Cre - Get Video output to YouTube node input.

10. **Create Telegram Notification Node**
    - Type: Telegram
    - Configure with Telegram Bot credentials and chat ID.
    - Connect downstream of video generation or upload nodes to send status messages.

11. **Connect all nodes as per the described flow.**

12. **Test the workflow end-to-end with sample inputs and credentials.**

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                             |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow uses a pre-created Creatomate template named “AI generated story template.”      | Creatomate template must be created and cURL copied for use in the HTTP Request node.                      |
| ElevenLabs voice integration is used within Creatomate for voiceover in videos.               | Requires linking ElevenLabs account in Creatomate.                                                        |
| Ideal for content creators on YouTube, Instagram, TikTok wanting faceless video content.      | Workflow automates scripting, image generation, and video editing.                                        |
| Adjust system prompt in AI Agent for script customization.                                    | Allows tailoring video topics, tone, and style.                                                           |
| Leonardo AI image style can be customized (e.g., cinematic, cartoon, realistic).               | Modify prompts in Generate Image nodes accordingly.                                                       |
| Telegram node sends notifications about video generation status.                              | Useful for monitoring automated workflow progress.                                                        |

---

This documentation provides a complete understanding of the workflow’s structure, node configurations, and integration points, enabling reproduction, modification, and troubleshooting.