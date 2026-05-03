Automated YouTube Shorts With AI: Creation & Publishing Using OpenAI 

https://n8nworkflows.xyz/workflows/automated-youtube-shorts-with-ai--creation---publishing-using-openai--3977


# Automated YouTube Shorts With AI: Creation & Publishing Using OpenAI 

### 1. Workflow Overview

This workflow automates the end-to-end creation and publishing of YouTube Shorts using AI technologies. It targets digital marketers, content creators, and social media managers seeking to scale short-form video production with minimal manual effort while maintaining brand consistency and engagement. The workflow is designed to run primarily through Telegram interactions, enabling easy two-step human approvals: initial video idea approval and final video approval.

The workflow logic is divided into the following functional blocks:

- **1.1 Input Reception & Validation:** Receives user input via Telegram, validates API keys and user messages.
- **1.2 AI Idea Generation & Discussion:** Uses OpenAI to generate video ideas and scripts, with conversational memory to discuss and refine concepts.
- **1.3 Script Processing & Audio Creation:** Processes AI-generated scripts, chunks them, and converts text to natural-sounding voiceovers via ElevenLabs API.
- **1.4 Visual Content Creation:** Generates AI-based images and video clips matching the script and brand style using Replicate and related APIs.
- **1.5 Video Assembly & Editing:** Combines audio and video assets, applies automated editing effects, and prepares final render JSON for video production.
- **1.6 Publishing & Notifications:** Uploads the rendered video to YouTube, communicates status and approvals via Telegram.

Human checkpoints are embedded to ensure content quality and control.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

**Overview:**  
This block initiates the workflow by capturing user commands and messages via Telegram, checks for the presence of necessary API keys, and routes the flow accordingly.

**Nodes Involved:**  
- Telegram Trigger  
- If Message From User  
- If All API Keys Set  
- Telegram: API Keys Missing  
- Missing API Keys  
- Input Variables  
- Set API Keys  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point for user messages through Telegram bot webhook.  
  - Configuration: Listens on a specific webhook for incoming Telegram messages.  
  - Inputs: User Telegram message  
  - Outputs: Routes to message check.  
  - Failure Modes: Webhook misconfiguration, Telegram API downtime.

- **If Message From User**  
  - Type: If  
  - Role: Checks if the incoming Telegram message is valid for processing (non-empty, correct format).  
  - Inputs: Telegram Trigger output  
  - Outputs: Routes to AI discussion or discards irrelevant messages.  
  - Edge Cases: Empty or malformed messages.

- **If All API Keys Set**  
  - Type: If  
  - Role: Verifies that all required API keys (OpenAI, ElevenLabs, Replicate, Cloudinary, Creatomate, 0CodeKit, YouTube) are set before proceeding.  
  - Inputs: Set API Keys node output  
  - Outputs: Routes to main workflow or error notification.  
  - Edge Cases: Missing or expired API keys.

- **Telegram: API Keys Missing**  
  - Type: Telegram  
  - Role: Sends a message to the user indicating missing API keys, halting workflow.  
  - Inputs: If All API Keys Set (false branch)  
  - Outputs: None (ends flow)  
  - Edge Cases: Telegram API failures.

- **Missing API Keys**  
  - Type: Stop and Error  
  - Role: Stops workflow execution with an error state due to missing API keys.

- **Input Variables**  
  - Type: Set  
  - Role: Sets initial variables needed for AI generation based on user input and API keys.  
  - Inputs: If All API Keys Set (true branch)  
  - Outputs: Ideator node start.

- **Set API Keys**  
  - Type: Set  
  - Role: Central place to store and manage all API credentials for various services.  
  - Notes: Marked as "SET BEFORE STARTING" to remind user.

---

#### 2.2 AI Idea Generation & Discussion

**Overview:**  
This block uses OpenAI and Langchain agents to generate, discuss, and refine engaging YouTube Shorts video ideas and scripts, incorporating conversational memory and structured output parsing.

**Nodes Involved:**  
- Ideator 🧠 (OpenAI node)  
- Discuss Ideas 💡 (Langchain Agent)  
- Structure Model Output (Output Parser)  
- Track Conversation Memory (Memory Buffer)  
- If No Video Idea (If)  
- If Idea Approved (If)  
- Telegram: Conversational Response  
- Telegram: Approve Idea  
- Telegram: Processing Started  
- Idea Denied  

**Node Details:**

- **Ideator 🧠**  
  - Type: OpenAI (Langchain)  
  - Role: Generates video ideas, SEO-optimized titles, and descriptions based on input variables.  
  - Configuration: Uses advanced prompt templates to align with YouTube marketing strategies.  
  - Inputs: Input Variables node  
  - Outputs: Script node with generated text.

- **Discuss Ideas 💡**  
  - Type: Langchain Agent  
  - Role: Interacts conversationally to refine ideas, handling user feedback and multiple iterations.  
  - Inputs: Telegram messages and Ideator outputs  
  - Outputs: Structured model output or routes to approval.

- **Structure Model Output**  
  - Type: Output Parser (Langchain)  
  - Role: Parses AI responses into structured data formats to extract titles, scripts, and metadata cleanly.  
  - Inputs: Discuss Ideas output  
  - Outputs: Refined structured data for next steps.

- **Track Conversation Memory**  
  - Type: Memory Buffer (Langchain)  
  - Role: Maintains conversational context for multi-turn interactions with the user.  
  - Inputs: Telegram messages  
  - Outputs: Context-aware input to Discuss Ideas node.

- **If No Video Idea**  
  - Type: If  
  - Role: Determines if the AI generated a valid video idea; if not, prompts user again or retries.  
  - Outputs: Telegram: Conversational Response (retry) or Telegram: Approve Idea (proceed).

- **If Idea Approved**  
  - Type: If  
  - Role: Checks user approval of the AI-generated idea before moving forward.  
  - Outputs: Starts processing or loops back to idea refinement.

- **Telegram: Conversational Response**  
  - Type: Telegram  
  - Role: Sends AI-generated content or prompts back to user for feedback.  
  - Inputs: If No Video Idea (true branch).

- **Telegram: Approve Idea**  
  - Type: Telegram  
  - Role: Sends approval request to user for AI-generated video idea.  
  - Inputs: If No Video Idea (false branch).

- **Telegram: Processing Started**  
  - Type: Telegram  
  - Role: Notifies user that content creation process has begun post-approval.

- **Idea Denied**  
  - Type: Set  
  - Role: Resets or adjusts variables when an idea is rejected.

---

#### 2.3 Script Processing & Audio Creation

**Overview:**  
Processes the approved AI-generated script, splits it into manageable chunks, and converts text into a natural-sounding voiceover using ElevenLabs API.

**Nodes Involved:**  
- Script (Set)  
- Convert Script to Audio (HTTP Request)  
- Chunk Script (HTTP Request)  
- Split Out (Split Out)  
- Upload to Cloudinary (HTTP Request)  

**Node Details:**

- **Script**  
  - Type: Set  
  - Role: Holds the finalized script text for audio conversion.  
  - Inputs: Ideator node output  
  - Outputs: Converts script to audio.

- **Convert Script to Audio**  
  - Type: HTTP Request  
  - Role: Calls ElevenLabs API to synthesize speech from script text.  
  - Configuration: POST request with script text, voice selection, and API key in headers.  
  - Inputs: Script node  
  - Outputs: Audio file or base64 data.  
  - Failure Modes: API key invalid, rate limits, network issues.

- **Chunk Script**  
  - Type: HTTP Request  
  - Role: Splits long audio or script into smaller segments for syncing with visuals.  
  - Inputs: Convert Script to Audio output  
  - Outputs: Segmented script/audio chunks.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the chunked script into individual items for parallel processing.  
  - Inputs: Chunk Script output  
  - Outputs: Feeds into image prompter for each chunk.

- **Upload to Cloudinary**  
  - Type: HTTP Request  
  - Role: Uploads generated audio files to Cloudinary for storage and CDN delivery.  
  - Inputs: Audio file outputs  
  - Outputs: URLs used in video assembly.  
  - Failure Modes: Cloudinary auth errors, file size limits.

---

#### 2.4 Visual Content Creation

**Overview:**  
Generates AI images and video clips tailored to the script and brand style using multiple AI models and APIs, aggregates and manages these media assets.

**Nodes Involved:**  
- Image Prompter 📷 (OpenAI)  
- Aggregate Prompts (Aggregate)  
- Request Images (HTTP Request)  
- Generating Images (Wait)  
- Get Images (HTTP Request)  
- Request Videos (HTTP Request)  
- Generating Videos (Wait)  
- Get Videos (HTTP Request)  
- Aggregate Videos (Aggregate)  

**Node Details:**

- **Image Prompter 📷**  
  - Type: OpenAI (Langchain)  
  - Role: Creates descriptive prompts for AI image generation based on script chunks.  
  - Inputs: Split Out node output  
  - Outputs: Prompts aggregated for image generation.

- **Aggregate Prompts**  
  - Type: Aggregate  
  - Role: Collects all image prompts into a single array to batch request generation.  
  - Inputs: Image Prompter output  
  - Outputs: Request Images node.

- **Request Images**  
  - Type: HTTP Request  
  - Role: Sends batch prompts to AI image generation APIs (e.g., Replicate) for producing visuals.  
  - Inputs: Aggregate Prompts output  
  - Outputs: Initiates image generation process.

- **Generating Images**  
  - Type: Wait  
  - Role: Waits asynchronously for image generation to complete.  
  - Inputs: Request Images output  
  - Outputs: Triggers Get Images.

- **Get Images**  
  - Type: HTTP Request  
  - Role: Fetches completed generated images from the AI service.  
  - Inputs: Generating Images output  
  - Outputs: Starts video requests.

- **Request Videos**  
  - Type: HTTP Request  
  - Role: Requests AI-generated video clips corresponding to visual themes or animations.  
  - Inputs: Get Images output  
  - Outputs: Initiates video generation.

- **Generating Videos**  
  - Type: Wait  
  - Role: Waits for video generation to finish asynchronously.  
  - Inputs: Request Videos output  
  - Outputs: Triggers Get Videos.

- **Get Videos**  
  - Type: HTTP Request  
  - Role: Retrieves generated videos from the AI service.  
  - Inputs: Generating Videos output  
  - Outputs: Aggregates video files.

- **Aggregate Videos**  
  - Type: Aggregate  
  - Role: Collects video clips into a single dataset for final assembly.  
  - Inputs: Get Videos output  
  - Outputs: Feeds into merge node for video/audio.

---

#### 2.5 Video Assembly & Editing

**Overview:**  
Merges audio and video assets, generates JSON instructions for video rendering, sends the job to Creatomate for AI-powered video editing, and monitors final video generation.

**Nodes Involved:**  
- Merge Videos and Audio (Merge)  
- Generate Render JSON (HTTP Request)  
- Set JSON Variable (Set)  
- Send to Creatomate (HTTP Request)  
- Generating Final Video (Wait)  
- Get Final Video (HTTP Request)  
- Merge Video Variables (Merge)  

**Node Details:**

- **Merge Videos and Audio**  
  - Type: Merge  
  - Role: Combines audio files and video clips into one unified data structure for rendering.  
  - Inputs: Aggregate Videos and Upload to Cloudinary outputs  
  - Outputs: Render JSON generation.

- **Generate Render JSON**  
  - Type: HTTP Request  
  - Role: Creates a structured JSON payload defining video edit instructions, transitions, overlays, and timing for Creatomate.  
  - Inputs: Merge Videos and Audio output  
  - Outputs: Set JSON Variable node.

- **Set JSON Variable**  
  - Type: Set  
  - Role: Prepares the final JSON object with all rendering parameters for Creatomate API.  
  - Inputs: Generate Render JSON output  
  - Outputs: Send to Creatomate node.

- **Send to Creatomate**  
  - Type: HTTP Request  
  - Role: Submits the video rendering job to Creatomate API.  
  - Inputs: Set JSON Variable output  
  - Outputs: Initiates video render wait.

- **Generating Final Video**  
  - Type: Wait  
  - Role: Waits asynchronously until video rendering is completed on Creatomate.  
  - Inputs: Send to Creatomate output  
  - Outputs: Get Final Video node.

- **Get Final Video**  
  - Type: HTTP Request  
  - Role: Retrieves the finished video file URL or data from Creatomate.  
  - Inputs: Generating Final Video output  
  - Outputs: Merge Video Variables.

- **Merge Video Variables**  
  - Type: Merge  
  - Role: Consolidates final video data with metadata for user approval.  
  - Inputs: Get Final Video and Aggregate Prompts outputs  
  - Outputs: Sends Telegram approval request.

---

#### 2.6 Publishing & Notifications

**Overview:**  
Handles final user approval for the rendered video, converts video data to appropriate formats, uploads to YouTube, and notifies the user of completion or rejection.

**Nodes Involved:**  
- Telegram: Approve Final Video  
- If Final Video Approved  
- Convert Video to Base64 (HTTP Request)  
- Decode Base64 to File (Convert to File)  
- Upload to YouTube (YouTube)  
- Telegram: Video Uploaded  
- Telegram: Video Declined  

**Node Details:**

- **Telegram: Approve Final Video**  
  - Type: Telegram  
  - Role: Sends the final rendered video preview for user approval via Telegram.  
  - Inputs: Merge Video Variables output  
  - Outputs: If Final Video Approved node.

- **If Final Video Approved**  
  - Type: If  
  - Role: Branches flow based on user approval or rejection of final video.  
  - Outputs: Proceeds to upload or sends decline message.

- **Convert Video to Base64**  
  - Type: HTTP Request  
  - Role: Converts video file to Base64 encoding for n8n file handling.  
  - Inputs: If Final Video Approved (true branch)  
  - Outputs: Decode Base64 to File.

- **Decode Base64 to File**  
  - Type: Convert To File  
  - Role: Converts Base64 string back into a file object suitable for upload.  
  - Inputs: Convert Video to Base64 output  
  - Outputs: Upload to YouTube.

- **Upload to YouTube**  
  - Type: YouTube node  
  - Role: Uses YouTube API to upload the final video with metadata and publish it on user's channel.  
  - Inputs: Decode Base64 to File output  
  - Outputs: Telegram: Video Uploaded.

- **Telegram: Video Uploaded**  
  - Type: Telegram  
  - Role: Notifies the user that video upload succeeded.  
  - Inputs: Upload to YouTube output.

- **Telegram: Video Declined**  
  - Type: Telegram  
  - Role: Notifies user that the video was rejected and terminates or loops back.  
  - Inputs: If Final Video Approved (false branch).

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                         | Input Node(s)                     | Output Node(s)                      | Sticky Note                                |
|----------------------------|--------------------------------|---------------------------------------|----------------------------------|-----------------------------------|--------------------------------------------|
| Telegram Trigger            | Telegram Trigger               | Entry point for Telegram user messages| -                                | If Message From User               |                                            |
| If Message From User        | If                            | Validates Telegram message             | Telegram Trigger                 | Discuss Ideas 💡                   |                                            |
| If All API Keys Set         | If                            | Checks if all required API keys exist | Set API Keys                    | Input Variables / Telegram: API Keys Missing |                                            |
| Telegram: API Keys Missing  | Telegram                      | Notifies missing API keys              | If All API Keys Set             | Missing API Keys                  |                                            |
| Missing API Keys            | Stop and Error                | Stops workflow due to missing keys    | Telegram: API Keys Missing      | -                                 |                                            |
| Set API Keys               | Set                           | Stores all API credentials             | Telegram: Processing Started    | If All API Keys Set               | SET BEFORE STARTING                         |
| Input Variables             | Set                           | Sets initial variables for AI          | If All API Keys Set             | Ideator 🧠                       |                                            |
| Ideator 🧠                  | OpenAI (Langchain)            | Generates video ideas and scripts      | Input Variables                | Script                          |                                            |
| Discuss Ideas 💡             | Langchain Agent               | Conversational AI for idea refinement  | If Message From User / Idea Denied | Structure Model Output          |                                            |
| Structure Model Output      | Output Parser (Langchain)     | Parses AI output into structured data  | Discuss Ideas 💡               | Discuss Ideas 💡                 |                                            |
| Track Conversation Memory   | Memory Buffer (Langchain)      | Maintains context over multiple turns  | Telegram messages              | Discuss Ideas 💡                 |                                            |
| If No Video Idea            | If                            | Checks if a valid idea was generated   | Discuss Ideas 💡               | Telegram: Conversational Response / Telegram: Approve Idea |                                            |
| Telegram: Conversational Response | Telegram               | Sends AI-generated messages to user   | If No Video Idea               | -                               |                                            |
| Telegram: Approve Idea      | Telegram                      | Requests user approval on idea         | If No Video Idea               | If Idea Approved                |                                            |
| If Idea Approved            | If                            | Checks user approval on video idea     | Telegram: Approve Idea          | Telegram: Processing Started / Idea Denied |                                            |
| Telegram: Processing Started| Telegram                      | Notifies user that processing started  | If Idea Approved               | Set API Keys                    |                                            |
| Idea Denied                 | Set                           | Handles idea rejection                  | If Idea Approved               | Discuss Ideas 💡                |                                            |
| Script                     | Set                           | Holds script text for audio conversion | Ideator 🧠                    | Convert Script to Audio         |                                            |
| Convert Script to Audio     | HTTP Request                  | Calls ElevenLabs to generate voiceover | Script                      | Chunk Script / Upload to Cloudinary |                                            |
| Chunk Script               | HTTP Request                  | Splits script/audio into smaller chunks| Convert Script to Audio        | Split Out                      |                                            |
| Split Out                  | Split Out                     | Splits chunked script into individual parts | Chunk Script               | Image Prompter 📷              |                                            |
| Image Prompter 📷           | OpenAI (Langchain)            | Generates image prompts from script chunks | Split Out                | Aggregate Prompts / Request Images |                                            |
| Aggregate Prompts          | Aggregate                     | Aggregates all image prompts            | Image Prompter 📷             | Request Images                 |                                            |
| Request Images             | HTTP Request                  | Sends prompts to AI image generation API | Aggregate Prompts           | Generating Images              |                                            |
| Generating Images          | Wait                         | Waits for image generation completion   | Request Images              | Get Images                    |                                            |
| Get Images                 | HTTP Request                  | Fetches generated images                | Generating Images            | Request Videos                |                                            |
| Request Videos             | HTTP Request                  | Requests AI-generated video clips       | Get Images                  | Generating Videos             |                                            |
| Generating Videos          | Wait                         | Waits for video generation completion   | Request Videos             | Get Videos                   |                                            |
| Get Videos                 | HTTP Request                  | Retrieves generated videos               | Generating Videos           | Aggregate Videos             |                                            |
| Aggregate Videos           | Aggregate                     | Aggregates all video clips               | Get Videos                  | Merge Videos and Audio       |                                            |
| Upload to Cloudinary       | HTTP Request                  | Uploads audio to Cloudinary storage      | Convert Script to Audio      | Merge Videos and Audio       |                                            |
| Merge Videos and Audio     | Merge                        | Combines videos and audio assets         | Aggregate Videos / Upload to Cloudinary | Generate Render JSON       |                                            |
| Generate Render JSON       | HTTP Request                  | Creates JSON instructions for video rendering | Merge Videos and Audio    | Set JSON Variable            |                                            |
| Set JSON Variable          | Set                          | Prepares rendering JSON payload          | Generate Render JSON        | Send to Creatomate           |                                            |
| Send to Creatomate         | HTTP Request                 | Submits job to Creatomate video editor   | Set JSON Variable           | Generating Final Video       |                                            |
| Generating Final Video     | Wait                         | Waits for final video rendering          | Send to Creatomate          | Get Final Video              |                                            |
| Get Final Video            | HTTP Request                 | Retrieves rendered video                  | Generating Final Video      | Merge Video Variables        |                                            |
| Merge Video Variables      | Merge                        | Combines final video data and metadata    | Get Final Video / Aggregate Prompts | Telegram: Approve Final Video |                                            |
| Telegram: Approve Final Video | Telegram                   | Sends video for user approval             | Merge Video Variables       | If Final Video Approved      |                                            |
| If Final Video Approved    | If                           | Checks final video approval                | Telegram: Approve Final Video | Convert Video to Base64 / Telegram: Video Declined |                                            |
| Convert Video to Base64    | HTTP Request                 | Converts video file to Base64 encoding    | If Final Video Approved     | Decode Base64 to File        |                                            |
| Decode Base64 to File      | Convert To File              | Converts Base64 string to file object     | Convert Video to Base64     | Upload to YouTube            |                                            |
| Upload to YouTube          | YouTube                      | Uploads final video to YouTube channel    | Decode Base64 to File       | Telegram: Video Uploaded     |                                            |
| Telegram: Video Uploaded   | Telegram                     | Notifies user of successful upload        | Upload to YouTube           | -                           |                                            |
| Telegram: Video Declined   | Telegram                     | Notifies user of video rejection           | If Final Video Approved (false) | -                        |                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure webhook with your Telegram bot token.  
   - Position as entry node.

2. **Add If node named "If Message From User"**  
   - Condition: Check if incoming message text exists and matches expected commands or input format.  
   - Connect Telegram Trigger output to this node.

3. **Add Set node "Set API Keys"**  
   - Store all required API keys as workflow variables (OpenAI, ElevenLabs, Replicate, Cloudinary, Creatomate, 0CodeKit, YouTube).  
   - Connect from Telegram: Processing Started node (to be added later).

4. **Add If node "If All API Keys Set"**  
   - Validate presence of all API keys from "Set API Keys".  
   - True branch leads to "Input Variables" node, false branch to "Telegram: API Keys Missing".

5. **Add Telegram node "Telegram: API Keys Missing"**  
   - Sends a message to user informing missing API keys.  
   - Connect false branch of "If All API Keys Set" to this node.

6. **Add StopAndError node "Missing API Keys"**  
   - Stops workflow execution with error message.  
   - Connect from "Telegram: API Keys Missing".

7. **Add Set node "Input Variables"**  
   - Prepares initial variables for AI prompts based on user input and API keys.  
   - Connect true branch of "If All API Keys Set" to this node.

8. **Add OpenAI node "Ideator 🧠"**  
   - Configure with OpenAI API key.  
   - Create prompt to generate YouTube Shorts video ideas, titles, and descriptions optimized for SEO.  
   - Connect from "Input Variables".

9. **Add Set node "Script"**  
   - Store AI-generated script text for processing.  
   - Connect from "Ideator 🧠".

10. **Add HTTP Request node "Convert Script to Audio"**  
    - Configure POST request to ElevenLabs API for text-to-speech conversion.  
    - Use script text from "Script" node.  
    - Include API key in headers.  
    - Connect from "Script".

11. **Add HTTP Request node "Chunk Script"**  
    - Split audio/script into smaller chunks for syncing.  
    - Connect from "Convert Script to Audio".

12. **Add Split Out node "Split Out"**  
    - Split chunked script into individual items.  
    - Connect from "Chunk Script".

13. **Add OpenAI node "Image Prompter 📷"**  
    - Generate image prompts from each script chunk.  
    - Connect from "Split Out".

14. **Add Aggregate node "Aggregate Prompts"**  
    - Collect all image prompts into an array.  
    - Connect from "Image Prompter 📷".

15. **Add HTTP Request node "Request Images"**  
    - Send batch image prompts to AI image generation API (e.g., Replicate).  
    - Connect from "Aggregate Prompts".

16. **Add Wait node "Generating Images"**  
    - Wait for image generation completion.  
    - Connect from "Request Images".

17. **Add HTTP Request node "Get Images"**  
    - Retrieve generated images from AI service.  
    - Connect from "Generating Images".

18. **Add HTTP Request node "Request Videos"**  
    - Request AI-generated video clips.  
    - Connect from "Get Images".

19. **Add Wait node "Generating Videos"**  
    - Wait for video generation completion.  
    - Connect from "Request Videos".

20. **Add HTTP Request node "Get Videos"**  
    - Retrieve generated video clips.  
    - Connect from "Generating Videos".

21. **Add Aggregate node "Aggregate Videos"**  
    - Aggregate all video clips into a list.  
    - Connect from "Get Videos".

22. **Add HTTP Request node "Upload to Cloudinary"**  
    - Upload audio files produced by ElevenLabs for CDN and storage.  
    - Connect from "Convert Script to Audio".

23. **Add Merge node "Merge Videos and Audio"**  
    - Merge aggregated videos and uploaded audio URLs for final assembly.  
    - Connect inputs from "Aggregate Videos" and "Upload to Cloudinary".

24. **Add HTTP Request node "Generate Render JSON"**  
    - Prepare JSON configuration for Creatomate video rendering.  
    - Connect from "Merge Videos and Audio".

25. **Add Set node "Set JSON Variable"**  
    - Set final JSON payload for Creatomate API.  
    - Connect from "Generate Render JSON".

26. **Add HTTP Request node "Send to Creatomate"**  
    - Submit video rendering job to Creatomate API.  
    - Connect from "Set JSON Variable".

27. **Add Wait node "Generating Final Video"**  
    - Wait for Creatomate to finish rendering.  
    - Connect from "Send to Creatomate".

28. **Add HTTP Request node "Get Final Video"**  
    - Retrieve final rendered video file or URL.  
    - Connect from "Generating Final Video".

29. **Add Merge node "Merge Video Variables"**  
    - Combine final video with metadata for user approval.  
    - Connect from "Get Final Video" and "Aggregate Prompts".

30. **Add Telegram node "Telegram: Approve Final Video"**  
    - Send video preview to user for final approval.  
    - Connect from "Merge Video Variables".

31. **Add If node "If Final Video Approved"**  
    - Branch based on user approval.  
    - True branch proceeds to video upload; false branch sends decline message.

32. **Add HTTP Request node "Convert Video to Base64"**  
    - Convert approved video to Base64 for n8n file handling.  
    - Connect from "If Final Video Approved" (true).

33. **Add Convert To File node "Decode Base64 to File"**  
    - Convert Base64 string back to file object.  
    - Connect from "Convert Video to Base64".

34. **Add YouTube node "Upload to YouTube"**  
    - Upload video to YouTube channel using YouTube API credentials.  
    - Connect from "Decode Base64 to File".

35. **Add Telegram node "Telegram: Video Uploaded"**  
    - Notify user of successful upload.  
    - Connect from "Upload to YouTube".

36. **Add Telegram node "Telegram: Video Declined"**  
    - Notify user if video is rejected.  
    - Connect from "If Final Video Approved" (false).

37. **Add Langchain nodes for conversation:**  
    - "Discuss Ideas 💡" (agent), "Structure Model Output" (parser), and "Track Conversation Memory" (memory buffer).  
    - Connect appropriately to handle interactive idea generation and refinement.

38. **Add If nodes "If No Video Idea" and "If Idea Approved"**  
    - Manage flow based on AI idea validity and user approvals.

39. **Add Telegram nodes for "Telegram: Conversational Response" and "Telegram: Approve Idea"**  
    - Send prompts and requests back to user.

40. **Connect all nodes according to dependencies and logical flow as per above.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow requires API keys from OpenAI, ElevenLabs, Replicate, Cloudinary, Creatomate, 0CodeKit, and YouTube API access. Please ensure all keys are configured before running.                                             | See "Set API Keys" node instructions.                                                                                                             |
| The entire process is designed to be controlled and monitored via Telegram, with two approval steps to maintain content control and quality.                                                                                   | Telegram bot setup required.                                                                                                                       |
| For best results, customize AI prompts and voice/image/video model options to fit your brand voice and marketing style.                                                                                                        | Customization instructions provided in sticky notes within the workflow.                                                                          |
| The workflow automates complex video assembly with Creatomate API, a powerful AI video editing platform.                                                                                                                      | Creatomate API documentation: https://docs.creatomate.com                                                                                          |
| Video rendering steps include asynchronous waits and polling to handle processing times for AI generation services.                                                                                                           | Wait nodes "Generating Images", "Generating Videos", and "Generating Final Video" handle asynchronous waits.                                        |
| YouTube upload requires OAuth2 credentials with appropriate scopes for video publishing; ensure permission scopes are set correctly.                                                                                          | YouTube node credential setup required.                                                                                                           |
| The workflow integrates Langchain for advanced AI prompt management, conversational memory, and structured output parsing to improve interaction quality.                                                                     | Langchain nodes: Ideator 🧠, Discuss Ideas 💡, Structure Model Output, Track Conversation Memory.                                                   |
| Automation reduces the need for manual video editing or voiceover recording, saving time and resources for marketing teams.                                                                                                   | Ideal for scaling consistent short-form video content creation.                                                                                     |
| Sticky notes included in the workflow provide detailed setup instructions, API key placement, and customization tips.                                                                                                        | Review all sticky notes directly in the workflow editor for guidance.                                                                              |
| Telegram interaction flow ensures that the user can approve or deny at multiple checkpoints, preventing unwanted content publishing.                                                                                        | Two Telegram approval nodes: "Telegram: Approve Idea" and "Telegram: Approve Final Video".                                                         |

---

This detailed analysis and stepwise reproduction guide provide a comprehensive reference for understanding, maintaining, and customizing the Automated YouTube Shorts With AI workflow in n8n.