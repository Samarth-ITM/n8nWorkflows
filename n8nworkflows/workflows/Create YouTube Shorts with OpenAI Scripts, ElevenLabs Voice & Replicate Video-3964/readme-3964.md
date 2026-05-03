Create YouTube Shorts with OpenAI Scripts, ElevenLabs Voice & Replicate Video

https://n8nworkflows.xyz/workflows/create-youtube-shorts-with-openai-scripts--elevenlabs-voice---replicate-video-3964


# Create YouTube Shorts with OpenAI Scripts, ElevenLabs Voice & Replicate Video

### 1. Workflow Overview

This workflow automates the end-to-end creation of engaging YouTube Shorts videos using AI-generated scripts, AI voiceover, relevant imagery, and video assembly services. It addresses the needs of content creators, marketing agencies, and business owners by simplifying and accelerating the production of SEO-optimized short-form videos with professional narration and visuals.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception & Validation:** Captures user input via Telegram, validates presence of a video idea, checks API key availability.
- **1.2 Idea Discussion & Script Generation:** Uses AI to discuss and refine the video idea, generate a high-quality, SEO-optimized script.
- **1.3 Script Processing & Audio Creation:** Converts the script into audio narration using ElevenLabs or similar TTS API, segments and uploads audio.
- **1.4 Visuals Generation:** Creates image prompts from script segments, generates and fetches images, uploads them to Cloudinary.
- **1.5 Video Assembly:** Requests video clips, aggregates video and audio assets, sends data to Creatomate for video rendering.
- **1.6 Finalization & Distribution:** Retrieves the final video, converts and uploads it to YouTube, manages user approvals and notifications via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

**Overview:**  
This block receives user messages from Telegram, verifies if the message contains a video idea, and checks that all required API keys are set before proceeding.

**Nodes Involved:**  
- Telegram Trigger  
- If Message From User  
- If No Video Idea  
- Telegram: Conversational Response  
- Telegram: Approve Idea  
- Set API Keys  
- If All API Keys Set  
- Telegram: API Keys Missing  
- Missing API Keys (Stop and Error)

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Trigger node listening for Telegram messages.  
  - *Config:* Connected to a specific Telegram bot webhook.  
  - *Input/Output:* Input from Telegram user messages, output triggers conditional checks.  
  - *Failure Modes:* Telegram connectivity issues, webhook misconfiguration.

- **If Message From User**  
  - *Type:* Conditional node checking if incoming message is from a user.  
  - *Config:* Expression-based check for valid message content.  
  - *Input:* Telegram Trigger output.  
  - *Output:* Routes valid messages to idea discussion or ignores others.  
  - *Failure Modes:* Expression evaluation failure.

- **If No Video Idea**  
  - *Type:* Conditional node verifying presence of a video idea in message.  
  - *Config:* Checks if user input text is empty or missing.  
  - *Outputs:*  
    - No idea → sends a conversational response prompting for input.  
    - Idea present → sends approval prompt to user.  
  - *Failure Modes:* Edge cases if input format changes, empty messages.

- **Telegram: Conversational Response**  
  - *Type:* Telegram node sending message back to user.  
  - *Config:* Sends prompt asking for a video idea if missing.  
  - *Failure Modes:* Telegram send errors, rate limits.

- **Telegram: Approve Idea**  
  - *Type:* Telegram node requesting user approval for the entered idea.  
  - *Config:* Sends message with approval buttons (inline keyboard).  
  - *Output:* User response triggers approval conditional.

- **Set API Keys**  
  - *Type:* Set node that stores required API keys as workflow variables.  
  - *Notes:* Marked as "SET BEFORE STARTING".  
  - *Failure Modes:* Missing or invalid keys cause downstream failure.

- **If All API Keys Set**  
  - *Type:* Conditional node checking presence of all required API keys.  
  - *Outputs:*  
    - Keys present → proceed to input variables and script generation.  
    - Keys missing → notify user about missing keys.  

- **Telegram: API Keys Missing**  
  - *Type:* Telegram node alerting the user that required API keys are missing.  
  - *Failure Modes:* Telegram message errors.

- **Missing API Keys**  
  - *Type:* Stop and Error node halting workflow execution due to missing keys.  
  - *Purpose:* Prevents execution without necessary credentials.

---

#### 2.2 Idea Discussion & Script Generation

**Overview:**  
This block refines the user’s video idea through AI conversation, generates a structured script optimized for engagement and SEO, and tracks conversation memory context.

**Nodes Involved:**  
- Discuss Ideas 💡 (Langchain Agent)  
- OpenAI Chat Model  
- Structure Model Output (Output Parser)  
- Track Conversation Memory (Memory Buffer Window)  
- If Idea Approved  
- Idea Denied  
- Telegram: Processing Started  
- Telegram: Approve Idea  
- Telegram: Conversational Response (fallback)

**Node Details:**

- **Discuss Ideas 💡**  
  - *Type:* Langchain agent node facilitating AI-driven conversation to refine ideas.  
  - *Config:* Uses OpenAI Chat Model as language model, integrates memory buffer and output parser.  
  - *Input:* User message, conversation history.  
  - *Output:* Structured script proposal, approval prompt.  
  - *Failure Modes:* API call failures, memory overflow, parsing errors.

- **OpenAI Chat Model**  
  - *Type:* Language model node (OpenAI GPT variant) for conversational AI.  
  - *Config:* Chat completion with parameters for style and engagement.  
  - *Failure Modes:* API rate limits, authentication errors, timeout.

- **Structure Model Output**  
  - *Type:* Output parser node that extracts structured data (script segments).  
  - *Purpose:* Ensures the AI output conforms to expected schema for downstream processing.

- **Track Conversation Memory**  
  - *Type:* Memory buffer node storing recent conversation context.  
  - *Purpose:* Maintains dialogue continuity for better AI responses.

- **If Idea Approved**  
  - *Type:* Conditional node checking user approval response from Telegram.  
  - *Outputs:*  
    - Approved → proceed to script generation.  
    - Denied → loop back or deny idea.

- **Idea Denied**  
  - *Type:* Set node triggered when idea is denied.  
  - *Purpose:* Resets or prompts for new idea generation.

- **Telegram: Processing Started**  
  - *Type:* Telegram node notifying user that processing has started.  
  - *Purpose:* UX feedback.

---

#### 2.3 Script Processing & Audio Creation

**Overview:**  
Processes the generated script by chunking it for audio conversion, converts script to voiceover audio, and uploads audio files to Cloudinary for use in video assembly.

**Nodes Involved:**  
- Ideator 🧠 (OpenAI script generation)  
- Script (Set node)  
- Convert Script to Audio (HTTP Request to TTS API)  
- Chunk Script (HTTP Request)  
- Split Out (Split Out node for chunked script)  
- Upload to Cloudinary (HTTP Request)  

**Node Details:**

- **Ideator 🧠**  
  - *Type:* Langchain OpenAI node generating the script based on idea and input variables.  
  - *Config:* Uses prompt templates for SEO optimized script creation.  
  - *Output:* Raw script text.

- **Script**  
  - *Type:* Set node storing the script text for further processing.

- **Convert Script to Audio**  
  - *Type:* HTTP Request node calling ElevenLabs or similar TTS service.  
  - *Config:* Sends script text, receives audio file or URL.  
  - *Failure Modes:* API authentication errors, rate limits, invalid text input, timeouts.

- **Chunk Script**  
  - *Type:* HTTP Request node splitting the long script into smaller segments suitable for chunked audio processing.  
  - *Purpose:* Ensures manageable audio segment sizes.

- **Split Out**  
  - *Type:* Split Out node that separates chunked script pieces for further processing.

- **Upload to Cloudinary**  
  - *Type:* HTTP Request node uploading audio chunks to Cloudinary cloud storage.  
  - *Failure Modes:* Network errors, authentication errors, upload size limits.

---

#### 2.4 Visuals Generation

**Overview:**  
Generates image prompts based on script segments, requests images from an AI image generation API, aggregates these images, and prepares them for video integration.

**Nodes Involved:**  
- Image Prompter 📷 (OpenAI prompt generation)  
- Aggregate Prompts (Aggregate node)  
- Request Images (HTTP Request)  
- Generating Images (Wait node)  
- Get Images (HTTP Request)  

**Node Details:**

- **Image Prompter 📷**  
  - *Type:* OpenAI node generating image prompts linked to script content segments.  
  - *Config:* Uses script text chunks to create relevant image descriptions.  
  - *Failure Modes:* API call errors, malformed prompts.

- **Aggregate Prompts**  
  - *Type:* Aggregate node collecting multiple image prompts into a batch for bulk image request.

- **Request Images**  
  - *Type:* HTTP Request node sending batch image prompts to an image generation API (e.g., Replicate).  
  - *Failure Modes:* API limits, request timeouts.

- **Generating Images**  
  - *Type:* Wait node holding workflow until images are ready.  
  - *Purpose:* Allows asynchronous image generation process to complete.

- **Get Images**  
  - *Type:* HTTP Request node fetching generated images after processing delay.  
  - *Failure Modes:* Resource not ready, API errors.

---

#### 2.5 Video Assembly

**Overview:**  
Requests short video clips, aggregates video clips with audio, composes the video JSON configuration, sends the project to Creatomate for rendering, waits for rendering completion, and retrieves the final video.

**Nodes Involved:**  
- Request Videos (HTTP Request)  
- Generating Videos (Wait node)  
- Get Videos (HTTP Request)  
- Aggregate Videos (Aggregate node)  
- Merge Videos and Audio (Merge node)  
- Generate Render JSON (HTTP Request)  
- Set JSON Variable (Set node)  
- Send to Creatomate (HTTP Request)  
- Generating Final Video (Wait node)  
- Get Final Video (HTTP Request)  
- Merge Video Variables (Merge node)  

**Node Details:**

- **Request Videos**  
  - *Type:* HTTP Request node requesting video clips matching the script or theme.  
  - *Failure Modes:* API errors, no clips found.

- **Generating Videos**  
  - *Type:* Wait node ensuring video clips are generated and ready.

- **Get Videos**  
  - *Type:* HTTP Request node retrieving processed video clips.

- **Aggregate Videos**  
  - *Type:* Aggregate node combining multiple video clip data into a single collection.

- **Merge Videos and Audio**  
  - *Type:* Merge node synchronizing video clips with audio narration segments.

- **Generate Render JSON**  
  - *Type:* HTTP Request node creating JSON payload formatted for Creatomate video rendering API.

- **Set JSON Variable**  
  - *Type:* Set node storing the render configuration JSON.

- **Send to Creatomate**  
  - *Type:* HTTP Request node sending the video project to Creatomate API for rendering.

- **Generating Final Video**  
  - *Type:* Wait node holding execution until video rendering completes.

- **Get Final Video**  
  - *Type:* HTTP Request node fetching the final rendered video link or file.

- **Merge Video Variables**  
  - *Type:* Merge node consolidating final video metadata and user variables.

---

#### 2.6 Finalization & Distribution

**Overview:**  
Manages user approval of the final video, converts the video to base64, decodes it to a file, uploads the final video to YouTube, and sends Telegram notifications about upload status.

**Nodes Involved:**  
- Telegram: Approve Final Video  
- If Final Video Approved  
- Convert Video to Base64 (HTTP Request)  
- Decode Base64 to File (ConvertToFile)  
- Upload to YouTube (YouTube node)  
- Telegram: Video Uploaded  
- Telegram: Video Declined  

**Node Details:**

- **Telegram: Approve Final Video**  
  - *Type:* Telegram node prompting user to approve the rendered video.  
  - *Failure Modes:* User inactivity, Telegram message errors.

- **If Final Video Approved**  
  - *Type:* Conditional node branching on user approval.  
  - *Outputs:*  
    - Approved → process video upload.  
    - Declined → notify user and stop.

- **Convert Video to Base64**  
  - *Type:* HTTP Request node downloading the video and converting to base64 string.

- **Decode Base64 to File**  
  - *Type:* ConvertToFile node converting base64 string to a file object for upload.

- **Upload to YouTube**  
  - *Type:* YouTube node uploading video file to the user’s YouTube channel.  
  - *Config:* Requires OAuth2 credentials and video metadata.  
  - *Failure Modes:* OAuth token expiration, upload quota exceeded.

- **Telegram: Video Uploaded**  
  - *Type:* Telegram node notifying user of successful upload.

- **Telegram: Video Declined**  
  - *Type:* Telegram node notifying user that video was declined.

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                       | Input Node(s)                   | Output Node(s)                     | Sticky Note                              |
|----------------------------|------------------------------------|------------------------------------|--------------------------------|----------------------------------|-----------------------------------------|
| Telegram Trigger            | Telegram Trigger                   | Receives user messages              | -                              | If Message From User             |                                         |
| If Message From User        | If                                | Checks if message is from user      | Telegram Trigger               | Discuss Ideas 💡                 |                                         |
| Discuss Ideas 💡            | Langchain Agent                   | AI conversation, idea refinement   | If Message From User           | If No Video Idea                 |                                         |
| If No Video Idea            | If                                | Checks presence of video idea       | Discuss Ideas 💡               | Telegram: Conversational Response, Telegram: Approve Idea |                                         |
| Telegram: Conversational Response | Telegram                      | Asks user for video idea            | If No Video Idea               | -                                |                                         |
| Telegram: Approve Idea      | Telegram                          | Requests idea approval              | If No Video Idea               | If Idea Approved                |                                         |
| If Idea Approved            | If                                | Checks user approval                | Telegram: Approve Idea         | Telegram: Processing Started, Idea Denied |                                         |
| Idea Denied                | Set                               | Handles denied idea                 | If Idea Approved              | Discuss Ideas 💡                |                                         |
| Telegram: Processing Started | Telegram                         | Notifies processing start          | If Idea Approved              | Set API Keys                   |                                         |
| Set API Keys               | Set                               | Stores API keys                    | Telegram: Processing Started   | If All API Keys Set             | SET BEFORE STARTING                      |
| If All API Keys Set         | If                                | Validates API keys presence         | Set API Keys                  | Input Variables, Telegram: API Keys Missing |                                         |
| Telegram: API Keys Missing  | Telegram                          | Notifies missing API keys          | If All API Keys Set            | Missing API Keys                |                                         |
| Missing API Keys            | Stop and Error                    | Stops workflow on missing keys     | Telegram: API Keys Missing     | -                              |                                         |
| Input Variables            | Set                               | Sets input variables for generation | If All API Keys Set            | Ideator 🧠                    |                                         |
| Ideator 🧠                 | Langchain OpenAI                 | Generates SEO optimized script      | Input Variables               | Script                        |                                         |
| Script                    | Set                               | Holds generated script             | Ideator 🧠                   | Convert Script to Audio         |                                         |
| Convert Script to Audio     | HTTP Request                     | Converts script to audio            | Script                       | Chunk Script, Upload to Cloudinary |                                         |
| Chunk Script               | HTTP Request                     | Splits script into chunks           | Convert Script to Audio       | Split Out                     |                                         |
| Split Out                 | Split Out                        | Outputs chunked script parts        | Chunk Script                 | Image Prompter 📷              |                                         |
| Image Prompter 📷           | OpenAI                          | Generates image prompts             | Split Out                   | Aggregate Prompts, Request Images |                                         |
| Aggregate Prompts          | Aggregate                        | Aggregates image prompts            | Image Prompter 📷             | Merge Video Variables (branch 1) |                                         |
| Request Images             | HTTP Request                    | Sends image generation requests    | Image Prompter 📷             | Generating Images              |                                         |
| Generating Images          | Wait                            | Waits for image generation          | Request Images               | Get Images                    |                                         |
| Get Images                 | HTTP Request                    | Retrieves generated images          | Generating Images            | Request Videos                |                                         |
| Request Videos             | HTTP Request                    | Requests video clips                | Get Images                   | Generating Videos             |                                         |
| Generating Videos          | Wait                            | Waits for video clip generation     | Request Videos               | Get Videos                   |                                         |
| Get Videos                 | HTTP Request                    | Retrieves video clips               | Generating Videos            | Aggregate Videos             |                                         |
| Aggregate Videos           | Aggregate                       | Aggregates video clips              | Get Videos                   | Merge Videos and Audio        |                                         |
| Merge Videos and Audio     | Merge                          | Combines videos and audio           | Aggregate Videos, Upload to Cloudinary | Generate Render JSON          |                                         |
| Upload to Cloudinary       | HTTP Request                   | Uploads audio files                 | Convert Script to Audio       | Merge Videos and Audio (branch 2) |                                         |
| Generate Render JSON       | HTTP Request                   | Creates JSON for video rendering    | Merge Videos and Audio        | Set JSON Variable             |                                         |
| Set JSON Variable          | Set                            | Stores render JSON                  | Generate Render JSON          | Send to Creatomate            |                                         |
| Send to Creatomate         | HTTP Request                   | Sends project for rendering         | Set JSON Variable             | Generating Final Video        |                                         |
| Generating Final Video     | Wait                           | Waits for final video rendering     | Send to Creatomate            | Get Final Video              |                                         |
| Get Final Video            | HTTP Request                   | Retrieves final rendered video      | Generating Final Video        | Merge Video Variables        |                                         |
| Merge Video Variables      | Merge                          | Combines final video with metadata  | Get Final Video, Aggregate Prompts | Telegram: Approve Final Video |                                         |
| Telegram: Approve Final Video | Telegram                      | Requests user approval for final video | Merge Video Variables        | If Final Video Approved       |                                         |
| If Final Video Approved    | If                             | Checks if final video is approved   | Telegram: Approve Final Video | Convert Video to Base64, Telegram: Video Declined |                                         |
| Convert Video to Base64    | HTTP Request                   | Converts video to base64 string     | If Final Video Approved       | Decode Base64 to File         |                                         |
| Decode Base64 to File      | ConvertToFile                  | Converts base64 to file object      | Convert Video to Base64       | Upload to YouTube             |                                         |
| Upload to YouTube          | YouTube                       | Uploads video to YouTube            | Decode Base64 to File        | Telegram: Video Uploaded      |                                         |
| Telegram: Video Uploaded   | Telegram                      | Notifies user of successful upload | Upload to YouTube             | -                            |                                         |
| Telegram: Video Declined   | Telegram                      | Notifies user video was declined    | If Final Video Approved (No)  | -                            |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Configure with your Telegram bot webhook.  
   - This node listens for incoming user messages.

2. **Add "If Message From User" node (If)**  
   - Check if the message is from a user and contains content.  
   - Connect Telegram Trigger to this node.

3. **Add "Discuss Ideas 💡" (Langchain Agent) node**  
   - Use OpenAI Chat Model as the language model.  
   - Configure with conversational prompts to refine video ideas.  
   - Connect "If Message From User" to this node.

4. **Add "If No Video Idea" (If) node**  
   - Checks if the user provided a video idea.  
   - Connect "Discuss Ideas 💡" output to this node.

5. **Add Telegram nodes:**  
   - "Telegram: Conversational Response" to prompt for idea if missing.  
   - "Telegram: Approve Idea" to request approval if idea present.  
   - Connect respective outputs of "If No Video Idea" accordingly.

6. **Add "If Idea Approved" (If) node**  
   - Connect "Telegram: Approve Idea" to this node.  
   - Split output to "Telegram: Processing Started" if approved, else "Idea Denied".

7. **Add "Idea Denied" (Set) node**  
   - Clears or resets the idea, loops back to "Discuss Ideas 💡".

8. **Add "Telegram: Processing Started" node**  
   - Notifies user that processing is underway.  
   - Connect to "Set API Keys".

9. **Add "Set API Keys" node**  
   - Define all required API keys as workflow variables here (OpenAI, ElevenLabs, Cloudinary, Replicate, Creatomate, YouTube OAuth2).  
   - Connect to "If All API Keys Set".

10. **Add "If All API Keys Set" (If) node**  
    - Verify all API keys are set.  
    - If not, connect to "Telegram: API Keys Missing" and then "Missing API Keys" (Stop and Error).

11. **Add "Input Variables" (Set) node**  
    - Prepare variables for script generation (e.g., video idea, style, length).  
    - Connect from "If All API Keys Set" (keys present branch).

12. **Add "Ideator 🧠" (Langchain OpenAI) node**  
    - Configure to generate an SEO optimized video script based on input variables.  
    - Connect "Input Variables" node to this.

13. **Add "Script" (Set) node**  
    - Store generated script text.  
    - Connect from "Ideator 🧠".

14. **Add "Convert Script to Audio" (HTTP Request) node**  
    - Configure call to ElevenLabs or equivalent TTS API sending the script text to generate voiceover audio.  
    - Connect from "Script".

15. **Add "Chunk Script" (HTTP Request) node**  
    - Split the script into smaller chunks for processing.  
    - Connect from "Convert Script to Audio".

16. **Add "Split Out" node**  
    - Split chunked script into individual parts.  
    - Connect from "Chunk Script".

17. **Add "Image Prompter 📷" (OpenAI) node**  
    - Generate image prompts from script chunks.  
    - Connect from "Split Out".

18. **Add "Aggregate Prompts" node**  
    - Aggregate all image prompts into a batch.  
    - Connect from "Image Prompter 📷".

19. **Add "Request Images" (HTTP Request) node**  
    - Send prompts to Replicate or similar image generation API.  
    - Connect from "Image Prompter 📷".

20. **Add "Generating Images" (Wait) node**  
    - Wait for asynchronous image generation to complete.  
    - Connect from "Request Images".

21. **Add "Get Images" (HTTP Request) node**  
    - Retrieve generated images.  
    - Connect from "Generating Images".

22. **Add "Request Videos" (HTTP Request) node**  
    - Request video clips matching script or theme.  
    - Connect from "Get Images".

23. **Add "Generating Videos" (Wait) node**  
    - Wait for video clips generation.  
    - Connect from "Request Videos".

24. **Add "Get Videos" (HTTP Request) node**  
    - Fetch generated video clips.  
    - Connect from "Generating Videos".

25. **Add "Aggregate Videos" node**  
    - Aggregate all video clips.  
    - Connect from "Get Videos".

26. **Add "Upload to Cloudinary" (HTTP Request) node**  
    - Upload audio chunks to Cloudinary.  
    - Connect from "Convert Script to Audio".

27. **Add "Merge Videos and Audio" (Merge) node**  
    - Merge video clips and audio files into a combined dataset.  
    - Connect from "Aggregate Videos" and "Upload to Cloudinary".

28. **Add "Generate Render JSON" (HTTP Request) node**  
    - Create a JSON configuration for Creatomate video assembly API.  
    - Connect from "Merge Videos and Audio".

29. **Add "Set JSON Variable" (Set) node**  
    - Store the JSON payload for rendering.  
    - Connect from "Generate Render JSON".

30. **Add "Send to Creatomate" (HTTP Request) node**  
    - Send render JSON to Creatomate API to start video rendering.  
    - Connect from "Set JSON Variable".

31. **Add "Generating Final Video" (Wait) node**  
    - Wait for video rendering completion.  
    - Connect from "Send to Creatomate".

32. **Add "Get Final Video" (HTTP Request) node**  
    - Retrieve final rendered video URL or file.  
    - Connect from "Generating Final Video".

33. **Add "Merge Video Variables" (Merge) node**  
    - Merge final video data with user variables.  
    - Connect from "Get Final Video" and "Aggregate Prompts" (branch 2).

34. **Add "Telegram: Approve Final Video" node**  
    - Request user approval for the rendered video.  
    - Connect from "Merge Video Variables".

35. **Add "If Final Video Approved" (If) node**  
    - Branch based on user approval.  
    - If approved, connect to "Convert Video to Base64".  
    - If declined, connect to "Telegram: Video Declined".

36. **Add "Convert Video to Base64" (HTTP Request) node**  
    - Download and convert video to base64.  
    - Connect from "If Final Video Approved" (approved branch).

37. **Add "Decode Base64 to File" (ConvertToFile) node**  
    - Convert base64 string to file object.  
    - Connect from "Convert Video to Base64".

38. **Add "Upload to YouTube" (YouTube) node**  
    - Upload video file to YouTube channel.  
    - Configure with OAuth2 credentials and video metadata.  
    - Connect from "Decode Base64 to File".

39. **Add "Telegram: Video Uploaded" node**  
    - Notify user of successful video upload.  
    - Connect from "Upload to YouTube".

40. **Add "Telegram: Video Declined" node**  
    - Notify user about video decline.  
    - Connect from "If Final Video Approved" (declined branch).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Workflow enables rapid YouTube Shorts creation with AI-generated scripts, voiceovers, images, and video assembly. | Workflow description and purpose |
| Requires API keys for OpenAI, ElevenLabs, Cloudinary, Replicate, Creatomate, and YouTube OAuth2. | Setup instructions |
| Telegram bot used for user interaction, idea input, and approval workflow. | User interface |
| Video assembly handled by Creatomate API, which requires JSON configuration of assets. | Video rendering service |
| Audio voiceover created using ElevenLabs or equivalent TTS service for realistic narration. | Audio generation |
| Image generation uses Replicate or similar AI image APIs to provide relevant visuals. | Visual content generation |
| YouTube upload requires OAuth2 credentials with proper scopes for video upload. | Distribution |
| Wait nodes used to handle asynchronous API processing and ensure resource readiness. | Workflow synchronization |
| Sticky notes are present in the workflow for internal comments and reminders, e.g., "SET BEFORE STARTING" on API keys. | Workflow annotations |
| See official n8n docs for node-specific details: https://docs.n8n.io/ | Additional reference |

---

This documentation enables advanced users and AI agents to fully understand, reproduce, and extend the YouTube Shorts automation workflow, including detailed node configurations, error considerations, and integration specifics.