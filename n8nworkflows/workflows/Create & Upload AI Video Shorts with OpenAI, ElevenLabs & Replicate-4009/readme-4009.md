Create & Upload AI Video Shorts with OpenAI, ElevenLabs & Replicate

https://n8nworkflows.xyz/workflows/create---upload-ai-video-shorts-with-openai--elevenlabs---replicate-4009


# Create & Upload AI Video Shorts with OpenAI, ElevenLabs & Replicate

### 1. Workflow Overview

This n8n workflow automates the creation and uploading of AI-generated video shorts by integrating OpenAI (for ideation and script generation), ElevenLabs (for text-to-speech audio conversion), Replicate (for video generation), and cloud/video platforms like Cloudinary and YouTube. It targets entrepreneurs, freelancers, and agencies seeking to scale content production and client deliverables with AI-driven automation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation:** Receives user input via Telegram, checks API key availability, and initiates the ideation process.
- **1.2 Idea Generation & Approval:** Uses OpenAI language models to generate video ideas, manages conversation memory, and handles user approval or rejection of ideas.
- **1.3 Script and Media Generation:** Converts approved ideas into scripts, splits scripts into chunks, generates images and videos, converts scripts to audio, and aggregates media components.
- **1.4 Video Rendering and Approval:** Sends aggregated media to Creatomate for final video render, waits for completion, and handles user approval or decline of the final video.
- **1.5 Upload and Notification:** Converts final video to suitable format, uploads to YouTube and Cloudinary, and notifies the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:**  
This block handles incoming requests from Telegram users, verifies that necessary API keys are set, and triggers the ideation process.

- **Nodes Involved:**  
  - Telegram Trigger  
  - If Message From User  
  - If All API Keys Set  
  - Telegram: API Keys Missing  
  - Missing API Keys (stop and error node)  
  - Input Variables  
  - Set API Keys  

- **Node Details:**  
  - **Telegram Trigger:**  
    - *Type:* Telegram Trigger node  
    - *Role:* Entry point for user messages via Telegram webhook  
    - *Config:* Listens for any incoming message  
    - *Connections:* Outputs to "If Message From User"  
    - *Failures:* Network or webhook misconfiguration, invalid Telegram token  
  - **If Message From User:**  
    - *Type:* If node  
    - *Role:* Checks if the Telegram message is valid for processing (e.g., text message)  
    - *Config:* Expression-based condition on message type  
    - *Connections:* On true, proceeds to "Discuss Ideas 💡"  
  - **If All API Keys Set:**  
    - *Type:* If node  
    - *Role:* Validates that all required API keys (OpenAI, ElevenLabs, Replicate, etc.) are available before proceeding  
    - *Config:* Checks environment variables or workflow variables for presence of keys  
    - *Connections:* True → "Input Variables" to continue workflow; False → "Telegram: API Keys Missing"  
  - **Telegram: API Keys Missing:**  
    - *Type:* Telegram node  
    - *Role:* Sends notification to user if API keys are missing  
    - *Config:* Sends a message warning about missing credentials  
    - *Connections:* Leads to "Missing API Keys" (stop node)  
  - **Missing API Keys:**  
    - *Type:* Stop and Error node  
    - *Role:* Stops the workflow with an error if API keys are missing  
  - **Input Variables:**  
    - *Type:* Set node  
    - *Role:* Prepares and sets input variables for ideation  
    - *Config:* Defines workflow variables like prompt templates, client data, or parameters  
    - *Connections:* Sends data to "Ideator 🧠"  
  - **Set API Keys:**  
    - *Type:* Set node  
    - *Role:* Explicitly sets or refreshes API keys for downstream nodes  
    - *Notes:* Marked with "SET BEFORE STARTING," indicating manual or initial setup  

- **Potential Failures:**  
  - Telegram webhook or token misconfiguration  
  - Missing or invalid API credentials leading to early termination  
  - User sends unsupported message types or empty input  

---

#### 2.2 Idea Generation & Approval

- **Overview:**  
This block uses OpenAI language models to brainstorm video ideas, structures AI responses, manages conversation history, and facilitates user approval or denial of generated ideas.

- **Nodes Involved:**  
  - Discuss Ideas 💡 (Langchain Agent)  
  - OpenAI Chat Model  
  - Structure Model Output (Output Parser Structured)  
  - Track Conversation Memory (Memory Buffer Window)  
  - If No Video Idea (If node)  
  - Telegram: Conversational Response  
  - Telegram: Approve Idea  
  - If Idea Approved (If node)  
  - Idea Denied (Set node)  
  - Telegram: Processing Started  

- **Node Details:**  
  - **Discuss Ideas 💡:**  
    - *Type:* Langchain Agent node with OpenAI integration  
    - *Role:* Generates multiple video ideas or concepts using AI, driven by user input and context  
    - *Config:* Uses OpenAI Chat Model with contextual memory through "Track Conversation Memory"  
    - *Connections:* Outputs structured data to "Structure Model Output"  
    - *Failures:* OpenAI rate limits, malformed prompts, or response parsing errors  
  - **OpenAI Chat Model:**  
    - *Type:* Language model node (OpenAI Chat completion)  
    - *Role:* Provides AI-generated chat completions used by the agent  
  - **Structure Model Output:**  
    - *Type:* Output Parser node  
    - *Role:* Parses AI responses into structured JSON for downstream processing  
  - **Track Conversation Memory:**  
    - *Type:* Memory node  
    - *Role:* Maintains conversational context in a windowed buffer to enable context-aware AI responses  
  - **If No Video Idea:**  
    - *Type:* If node  
    - *Role:* Checks if any viable video ideas were generated  
    - *Connections:* False → Sends conversational response to Telegram; True → Sends "Approve Idea" message  
  - **Telegram: Conversational Response:**  
    - *Type:* Telegram node  
    - *Role:* Sends a message back to the user if no valid idea was generated  
  - **Telegram: Approve Idea:**  
    - *Type:* Telegram node  
    - *Role:* Sends generated ideas to user for approval, triggering their decision  
  - **If Idea Approved:**  
    - *Type:* If node  
    - *Role:* Branches workflow based on user approval of the idea  
    - *Connections:* True → "Telegram: Processing Started" and script generation; False → "Idea Denied"  
  - **Idea Denied:**  
    - *Type:* Set node  
    - *Role:* Resets or adjusts variables to restart ideation or terminate  
  - **Telegram: Processing Started:**  
    - *Type:* Telegram node  
    - *Role:* Notifies user that idea processing and content generation has started  

- **Potential Failures:**  
  - AI generating irrelevant or empty ideas  
  - User ignoring approval requests or sending invalid responses  
  - Telegram message delivery failures  

---

#### 2.3 Script and Media Generation

- **Overview:**  
Converts approved ideas into detailed scripts, splits scripts into manageable chunks, generates images and videos based on prompts, converts scripts to audio, and prepares media for final rendering.

- **Nodes Involved:**  
  - Ideator 🧠 (Langchain OpenAI node)  
  - Script (Set node)  
  - Chunk Script (HTTP Request)  
  - Split Out (Split Out node)  
  - Image Prompter 📷 (Langchain OpenAI)  
  - Aggregate Prompts (Aggregate node)  
  - Request Images (HTTP Request)  
  - Generating Images (Wait node)  
  - Get Images (HTTP Request)  
  - Request Videos (HTTP Request)  
  - Generating Videos (Wait node)  
  - Get Videos (HTTP Request)  
  - Aggregate Videos (Aggregate node)  
  - Convert Script to Audio (HTTP Request)  
  - Upload to Cloudinary (HTTP Request)  
  - Merge Videos and Audio (Merge node)  
  - Set JSON Variable (Set node)  
  - Send to Creatomate (HTTP Request)  

- **Node Details:**  
  - **Ideator 🧠:**  
    - *Type:* OpenAI node generating the detailed video script from approved idea  
  - **Script:**  
    - *Type:* Set node  
    - *Role:* Stores or formats the script text for further processing  
  - **Chunk Script:**  
    - *Type:* HTTP Request  
    - *Role:* Calls an external API or service to chunk the script into smaller parts (useful for TTS or video segments)  
  - **Split Out:**  
    - *Type:* Split Out node  
    - *Role:* Splits chunked script data into individual pieces for parallel processing  
  - **Image Prompter 📷:**  
    - *Type:* OpenAI node  
    - *Role:* Generates image prompts based on script chunks for visual content creation  
  - **Aggregate Prompts:**  
    - *Type:* Aggregate node  
    - *Role:* Combines generated image prompts for batch submission  
  - **Request Images:**  
    - *Type:* HTTP Request  
    - *Role:* Calls image generation API (possibly Replicate or other image generation service)  
  - **Generating Images:**  
    - *Type:* Wait node  
    - *Role:* Waits for asynchronous image generation to complete  
  - **Get Images:**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves generated images from the service  
  - **Request Videos:**  
    - *Type:* HTTP Request  
    - *Role:* Requests video generation based on images and script chunks  
  - **Generating Videos:**  
    - *Type:* Wait node  
    - *Role:* Waits for video generation to finish  
  - **Get Videos:**  
    - *Type:* HTTP Request  
    - *Role:* Fetches generated video files  
  - **Aggregate Videos:**  
    - *Type:* Aggregate node  
    - *Role:* Combines video segments for final merge  
  - **Convert Script to Audio:**  
    - *Type:* HTTP Request  
    - *Role:* Calls ElevenLabs or similar TTS API to generate audio from script  
  - **Upload to Cloudinary:**  
    - *Type:* HTTP Request  
    - *Role:* Uploads audio or media files for CDN or cloud storage  
  - **Merge Videos and Audio:**  
    - *Type:* Merge node  
    - *Role:* Combines audio with video files to create synchronized media  
  - **Set JSON Variable:**  
    - *Type:* Set node  
    - *Role:* Prepares JSON payload for final render request  
  - **Send to Creatomate:**  
    - *Type:* HTTP Request  
    - *Role:* Sends aggregated media and instructions to Creatomate API for rendering final video  

- **Potential Failures:**  
  - API timeouts or failures in external services (image/video generation, TTS)  
  - Incorrect prompt formatting causing poor media output  
  - Network issues causing incomplete media uploads or downloads  
  - Data synchronization errors between media components  

---

#### 2.4 Video Rendering and Approval

- **Overview:**  
Waits for the final video render from Creatomate, retrieves the video, sends it to the user for approval, and handles the approval or decline branches.

- **Nodes Involved:**  
  - Generating Final Video (Wait node)  
  - Get Final Video (HTTP Request)  
  - Merge Video Variables (Merge node)  
  - Telegram: Approve Final Video  
  - If Final Video Approved (If node)  
  - Convert Video to Base64 (HTTP Request)  
  - Telegram: Video Declined  
  - Decode Base64 to File (ConvertToFile node)  

- **Node Details:**  
  - **Generating Final Video:**  
    - *Type:* Wait node  
    - *Role:* Pauses the workflow until the rendering is complete, using webhook or polling approach  
  - **Get Final Video:**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the final rendered video from Creatomate or storage  
  - **Merge Video Variables:**  
    - *Type:* Merge node  
    - *Role:* Combines video metadata and content for Telegram messaging  
  - **Telegram: Approve Final Video:**  
    - *Type:* Telegram node  
    - *Role:* Sends final video to user with approval buttons or instructions  
  - **If Final Video Approved:**  
    - *Type:* If node  
    - *Role:* Branches workflow depending on user's approval response  
    - *True:* Proceeds to video conversion and upload  
    - *False:* Sends decline notification  
  - **Convert Video to Base64:**  
    - *Type:* HTTP Request  
    - *Role:* Converts video file to Base64 encoding for upload or transfer  
  - **Telegram: Video Declined:**  
    - *Type:* Telegram node  
    - *Role:* Notifies user that video was declined  
  - **Decode Base64 to File:**  
    - *Type:* ConvertToFile node  
    - *Role:* Converts Base64 string back to a file format for upload  

- **Potential Failures:**  
  - Failures in receiving render completion webhook  
  - Video retrieval or download errors  
  - Telegram message size or format limitations  
  - User ignoring video approval requests  

---

#### 2.5 Upload and Notification

- **Overview:**  
Uploads the approved final video to YouTube and Cloudinary, and sends notifications to the user confirming upload status.

- **Nodes Involved:**  
  - Upload to YouTube (YouTube node)  
  - Telegram: Video Uploaded  

- **Node Details:**  
  - **Upload to YouTube:**  
    - *Type:* YouTube node  
    - *Role:* Uploads final video file to configured YouTube channel  
    - *Config:* Requires OAuth2 credentials for YouTube API  
  - **Telegram: Video Uploaded:**  
    - *Type:* Telegram node  
    - *Role:* Sends confirmation message with video link or success notification to user  

- **Potential Failures:**  
  - OAuth token expiration or invalidation  
  - YouTube API quota limits or upload errors  
  - Telegram delivery failures  

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                           | Input Node(s)                 | Output Node(s)                      | Sticky Note                      |
|----------------------------|--------------------------------|-----------------------------------------|------------------------------|------------------------------------|----------------------------------|
| Telegram Trigger            | Telegram Trigger               | Entry point for user input via Telegram | —                            | If Message From User                |                                  |
| If Message From User        | If                            | Validates user message type              | Telegram Trigger             | Discuss Ideas 💡                   |                                  |
| If All API Keys Set         | If                            | Checks presence of required API keys    | Set API Keys                 | Input Variables, Telegram: API Keys Missing |                                  |
| Telegram: API Keys Missing  | Telegram                      | Notifies user of missing API keys       | If All API Keys Set          | Missing API Keys                   |                                  |
| Missing API Keys            | Stop and Error                | Stops workflow on missing API keys      | Telegram: API Keys Missing   | —                                  |                                  |
| Set API Keys               | Set                           | Sets/refreshes API keys                  | Telegram: Processing Started | If All API Keys Set                | SET BEFORE STARTING              |
| Input Variables             | Set                           | Prepares input variables for ideation   | If All API Keys Set          | Ideator 🧠                        |                                  |
| Ideator 🧠                 | Langchain OpenAI              | Generates video scripts from ideas      | Input Variables              | Script                           |                                  |
| Script                     | Set                           | Holds generated script text              | Ideator 🧠                  | Convert Script to Audio            |                                  |
| Convert Script to Audio     | HTTP Request                  | Converts script to audio (TTS)           | Script                      | Chunk Script, Upload to Cloudinary |                                  |
| Chunk Script               | HTTP Request                  | Splits script into chunks                 | Convert Script to Audio      | Split Out                        |                                  |
| Split Out                  | Split Out                     | Splits chunked scripts for parallel use  | Chunk Script                | Image Prompter 📷                 |                                  |
| Image Prompter 📷          | Langchain OpenAI              | Generates image prompts from chunks       | Split Out                   | Aggregate Prompts, Request Images |                                  |
| Aggregate Prompts          | Aggregate                     | Aggregates image prompts for batch API   | Image Prompter 📷           | Merge Video Variables (branch 1)  |                                  |
| Request Images             | HTTP Request                  | Calls image generation API                | Aggregate Prompts           | Generating Images                |                                  |
| Generating Images          | Wait                         | Waits for image generation to complete   | Request Images              | Get Images                      |                                  |
| Get Images                 | HTTP Request                  | Retrieves generated images                | Generating Images           | Request Videos                  |                                  |
| Request Videos             | HTTP Request                  | Calls video generation API                | Get Images                  | Generating Videos               |                                  |
| Generating Videos          | Wait                         | Waits for video generation to complete   | Request Videos              | Get Videos                     |                                  |
| Get Videos                 | HTTP Request                  | Retrieves generated videos                | Generating Videos           | Aggregate Videos               |                                  |
| Aggregate Videos           | Aggregate                     | Aggregates video segments                  | Get Videos                  | Merge Videos and Audio          |                                  |
| Upload to Cloudinary       | HTTP Request                  | Uploads audio/media files                  | Convert Script to Audio      | Merge Videos and Audio          |                                  |
| Merge Videos and Audio     | Merge                        | Combines video and audio files             | Aggregate Videos, Upload to Cloudinary | Generate Render JSON          |                                  |
| Generate Render JSON       | HTTP Request                  | Prepares final render JSON payload         | Merge Videos and Audio      | Set JSON Variable              |                                  |
| Set JSON Variable          | Set                          | Holds JSON payload for rendering           | Generate Render JSON        | Send to Creatomate             |                                  |
| Send to Creatomate         | HTTP Request                 | Sends data to Creatomate API for rendering | Set JSON Variable           | Generating Final Video         |                                  |
| Generating Final Video     | Wait                        | Waits for final video render completion    | Send to Creatomate          | Get Final Video                |                                  |
| Get Final Video            | HTTP Request                | Downloads final rendered video              | Generating Final Video      | Merge Video Variables          |                                  |
| Merge Video Variables      | Merge                       | Merges final video data for approval       | Get Final Video, Aggregate Prompts | Telegram: Approve Final Video |                                  |
| Telegram: Approve Final Video | Telegram                 | Sends video for user approval               | Merge Video Variables       | If Final Video Approved        |                                  |
| If Final Video Approved    | If                          | Branches on user approval                     | Telegram: Approve Final Video | Convert Video to Base64, Telegram: Video Declined |                                  |
| Convert Video to Base64    | HTTP Request                | Converts video file to Base64 encoding       | If Final Video Approved     | Decode Base64 to File          |                                  |
| Decode Base64 to File      | ConvertToFile              | Converts Base64 string back to file          | Convert Video to Base64     | Upload to YouTube              |                                  |
| Upload to YouTube          | YouTube                    | Uploads final video to YouTube channel        | Decode Base64 to File       | Telegram: Video Uploaded       |                                  |
| Telegram: Video Uploaded    | Telegram                   | Notifies user of successful upload           | Upload to YouTube           | —                            |                                  |
| Telegram: Video Declined   | Telegram                   | Notifies user video was declined              | If Final Video Approved     | —                            |                                  |
| Discuss Ideas 💡           | Langchain Agent             | Manages ideation and conversation flow       | If Message From User        | If No Video Idea               |                                  |
| If No Video Idea           | If                          | Checks if AI returned any video idea          | Discuss Ideas 💡            | Telegram: Conversational Response, Telegram: Approve Idea |                                  |
| Telegram: Conversational Response | Telegram             | Sends message if no video idea generated      | If No Video Idea            | —                            |                                  |
| Telegram: Approve Idea     | Telegram                   | Sends video idea for user approval             | If No Video Idea            | If Idea Approved              |                                  |
| If Idea Approved           | If                          | Branches based on user idea approval           | Telegram: Approve Idea      | Telegram: Processing Started, Idea Denied |                                  |
| Idea Denied                | Set                         | Handles idea rejection, resets or ends flow    | If Idea Approved            | Discuss Ideas 💡              |                                  |
| Telegram: Processing Started | Telegram                 | Notifies user that processing started          | If Idea Approved            | Set API Keys                  |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**
   - Type: Telegram Trigger
   - Configure with your Telegram Bot Token
   - Set to listen for incoming messages
   - Connect output to "If Message From User"

2. **Create If Node "If Message From User":**
   - Condition: message type equals "text"
   - True branch to "Discuss Ideas 💡"
   - False branch can be left unconnected or handled as needed

3. **Create If Node "If All API Keys Set":**
   - Checks presence of all required API keys as environment variables or workflow variables (OpenAI, ElevenLabs, Replicate, Cloudinary, YouTube)
   - True branch to "Input Variables"
   - False branch to "Telegram: API Keys Missing"

4. **Create Telegram Node "Telegram: API Keys Missing":**
   - Sends message: "API keys missing. Please configure them before proceeding."
   - Connect to "Missing API Keys" stop node

5. **Create Stop and Error Node "Missing API Keys":**
   - Stops workflow execution on error

6. **Create Set Node "Set API Keys":**
   - Define all necessary API keys as workflow variables or parameters
   - Mark as "SET BEFORE STARTING" for manual setup
   - Connect output to "If All API Keys Set"

7. **Create Set Node "Input Variables":**
   - Set initial variables such as user inputs, prompt templates, and client data
   - Connect output to "Ideator 🧠"

8. **Create Langchain OpenAI Node "Ideator 🧠":**
   - Configure to generate detailed video scripts from input variables
   - Use OpenAI credentials
   - Connect output to "Script"

9. **Create Set Node "Script":**
   - Store the generated script text
   - Connect to "Convert Script to Audio"

10. **Create HTTP Request Node "Convert Script to Audio":**
    - Call ElevenLabs or similar TTS API to convert script text to audio
    - Connect two outputs: one to "Chunk Script," another to "Upload to Cloudinary"

11. **Create HTTP Request Node "Chunk Script":**
    - Call service to split script into smaller chunks
    - Connect output to "Split Out"

12. **Create Split Out Node "Split Out":**
    - Splits chunked script into individual pieces
    - Connect to "Image Prompter 📷"

13. **Create Langchain OpenAI Node "Image Prompter 📷":**
    - Generate image prompts from script chunks
    - Connect outputs to "Aggregate Prompts" and "Request Images"

14. **Create Aggregate Node "Aggregate Prompts":**
    - Aggregate image prompts
    - Connect to "Merge Video Variables" (for one branch) and "Request Images" (for processing)

15. **Create HTTP Request Node "Request Images":**
    - Calls image generation API (like Replicate)
    - Connect to "Generating Images"

16. **Create Wait Node "Generating Images":**
    - Waits for image generation completion
    - Connect to "Get Images"

17. **Create HTTP Request Node "Get Images":**
    - Retrieves generated images
    - Connect to "Request Videos"

18. **Create HTTP Request Node "Request Videos":**
    - Calls video generation API with images and script chunks
    - Connect to "Generating Videos"

19. **Create Wait Node "Generating Videos":**
    - Waits for video generation completion
    - Connect to "Get Videos"

20. **Create HTTP Request Node "Get Videos":**
    - Retrieves generated videos
    - Connect to "Aggregate Videos"

21. **Create Aggregate Node "Aggregate Videos":**
    - Aggregates video segments
    - Connect to "Merge Videos and Audio"

22. **Create HTTP Request Node "Upload to Cloudinary":**
    - Upload audio or media files to Cloudinary
    - Connect to "Merge Videos and Audio"

23. **Create Merge Node "Merge Videos and Audio":**
    - Merge aggregated videos and audio streams
    - Connect to "Generate Render JSON"

24. **Create HTTP Request Node "Generate Render JSON":**
    - Prepares JSON for final rendering service (Creatomate)
    - Connect to "Set JSON Variable"

25. **Create Set Node "Set JSON Variable":**
    - Holds JSON payload for Creatomate API
    - Connect to "Send to Creatomate"

26. **Create HTTP Request Node "Send to Creatomate":**
    - Sends render request to Creatomate
    - Connect to "Generating Final Video"

27. **Create Wait Node "Generating Final Video":**
    - Waits for final video render completion webhook or polling
    - Connect to "Get Final Video"

28. **Create HTTP Request Node "Get Final Video":**
    - Downloads final video file
    - Connect to "Merge Video Variables"

29. **Create Merge Node "Merge Video Variables":**
    - Prepares video metadata for Telegram notification
    - Connect to "Telegram: Approve Final Video"

30. **Create Telegram Node "Telegram: Approve Final Video":**
    - Sends video to user for approval
    - Connect to "If Final Video Approved"

31. **Create If Node "If Final Video Approved":**
    - Branches on approval response
    - True → "Convert Video to Base64"
    - False → "Telegram: Video Declined"

32. **Create HTTP Request Node "Convert Video to Base64":**
    - Converts video to Base64 string for upload
    - Connect to "Decode Base64 to File"

33. **Create ConvertToFile Node "Decode Base64 to File":**
    - Converts Base64 back to file
    - Connect to "Upload to YouTube"

34. **Create YouTube Node "Upload to YouTube":**
    - Uploads video to YouTube channel
    - Configure OAuth2 credentials
    - Connect to "Telegram: Video Uploaded"

35. **Create Telegram Node "Telegram: Video Uploaded":**
    - Notifies user of successful upload

36. **Create Telegram Node "Telegram: Video Declined":**
    - Notifies user if video is declined

37. **Create Langchain Agent Node "Discuss Ideas 💡":**
    - Manages ideation flow with OpenAI chat model and memory
    - Connect to "If No Video Idea"

38. **Create If Node "If No Video Idea":**
    - Checks for presence of valid video ideas
    - False → "Telegram: Conversational Response"
    - True → "Telegram: Approve Idea"

39. **Create Telegram Node "Telegram: Conversational Response":**
    - Sends message if no idea is generated

40. **Create Telegram Node "Telegram: Approve Idea":**
    - Sends ideas to user for approval
    - Connect to "If Idea Approved"

41. **Create If Node "If Idea Approved":**
    - Branches on user idea approval
    - True → "Telegram: Processing Started"
    - False → "Idea Denied"

42. **Create Set Node "Idea Denied":**
    - Handles rejection and restarts or ends flow

43. **Create Telegram Node "Telegram: Processing Started":**
    - Notifies user processing started
    - Connect to "Set API Keys"

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow is part of a course titled "How I Built a $13K/Month Agency with Just ONE AI Automation"                            | Course description in workflow metadata                                                                     |
| Integrates OpenAI, ElevenLabs, Replicate, Cloudinary, YouTube, and Telegram for seamless AI-driven video content automation      | Multi-platform AI media pipeline                                                                             |
| Requires configuration of API keys before running; marked clearly in "Set API Keys" node                                         | Critical setup step                                                                                           |
| Uses Creatomate API for video rendering, enabling customizable video templates and automation                                     | https://creatomate.com                                                                                       |
| Telegram nodes use webhooks requiring public URLs or n8n cloud setup for internet accessibility                                   | Telegram Bot API documentation: https://core.telegram.org/bots/api                                          |
| OpenAI usage involves managing conversation memory and structured output parsing for coherent multi-turn interactions             | Langchain integration best practices                                                                         |
| Video approval flow includes user interaction via Telegram for quality control                                                    | Interactive user approval to ensure content relevance                                                       |
| For scalable implementations, consider API rate limits, error handling, and retry mechanisms for each external service           | Refer to respective API docs for error codes and limits                                                     |

---

This document provides a complete, detailed reference and rebuild guide for the "Create & Upload AI Video Shorts with OpenAI, ElevenLabs & Replicate" n8n workflow, enabling understanding, modification, and reproduction without needing the original JSON.