Create Motivational Videos with Ollama AI, fal.ai Images & ElevenLabs Voice

https://n8nworkflows.xyz/workflows/create-motivational-videos-with-ollama-ai--fal-ai-images---elevenlabs-voice-10954


# Create Motivational Videos with Ollama AI, fal.ai Images & ElevenLabs Voice

---

### 1. Workflow Overview

This workflow automates the creation of motivational videos by integrating AI text generation, image synthesis, voice generation, and video editing services. It is designed to produce faceless motivational videos with AI-generated scripts, images, and narration, then combine and caption these into final video files ready for distribution.

The workflow logically divides into the following blocks:

- **1.1 Input Reception and Data Retrieval:** Trigger the workflow and fetch input data (motivational themes/scripts) from a Google Sheet.
- **1.2 AI Text Generation and Parsing:** Use Ollama AI chat and LangChain to generate structured motivational scripts and audio/image prompts.
- **1.3 Image and Audio Generation:** Generate images and voice audio clips using external APIs (fal.ai for images and ElevenLabs for voice).
- **1.4 Video Creation and Editing:** Generate video clips for each segment, trim, combine audio and video, concatenate clips, and add captions.
- **1.5 Output Assembly:** Build final arrays of videos and deliver the combined motivational video.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Retrieval

- **Overview:** This block initiates the workflow manually and fetches motivational script data from Google Sheets for further processing.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get row(s) in sheet (Google Sheets)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution on user demand.  
    - Configuration: Default manual trigger without parameters.  
    - Connections: Outputs to "Get row(s) in sheet".  
    - Failure: None typical; manual trigger relies on user action.

  - **Get row(s) in sheet**  
    - Type: Google Sheets Node  
    - Role: Reads rows from a predefined Google Sheet containing initial motivational content or prompts.  
    - Configuration: Requires Google Sheets credentials; configured to fetch specific rows or the entire sheet.  
    - Inputs: From manual trigger  
    - Outputs: To "Basic LLM Chain" node for text generation.  
    - Failures: Authentication errors, API limits, empty or malformed sheet data.

---

#### 2.2 AI Text Generation and Parsing

- **Overview:** Generates structured motivational scripts and audio/image prompt data using a chained AI language model with Ollama Chat and LangChain. Parses AI-generated output into usable structured data.
- **Nodes Involved:**  
  - Basic LLM Chain (LangChain LLM Chain)  
  - Ollama Chat Model (Ollama AI Chat)  
  - Structured Output Parser (LangChain Structured Output Parser)  
  - Image Scripts (Set)  
  - Audio Scripts (Set)

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: LangChain LLM Chain Node  
    - Role: Orchestrates AI prompt execution and chains multiple output parsers.  
    - Configuration: Linked to Ollama Chat Model as language model and Structured Output Parser as output parser.  
    - Inputs: Data from Google Sheets node  
    - Outputs: To "Image Scripts" and "Audio Scripts" set nodes.  
    - Failure Points: API rate limits, malformed prompt output.

  - **Ollama Chat Model**  
    - Type: Ollama AI Chat Node  
    - Role: Language model backend for text generation.  
    - Configuration: Requires Ollama API credentials; configured for motivational text generation.  
    - Inputs: From "Basic LLM Chain" as language model source.  
    - Outputs: To "Basic LLM Chain" node's AI processing chain.  
    - Failure Points: Network issues, API errors, invalid credentials.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses AI-generated text into structured JSON or objects for downstream use.  
    - Inputs: AI-generated text from Ollama Chat Model via Basic LLM Chain.  
    - Outputs: To "Basic LLM Chain" for further processing.  
    - Failure Points: Parsing errors if AI output does not match expected format.

  - **Image Scripts**  
    - Type: Set Node  
    - Role: Sets variables or parameters specifically for image generation prompts.  
    - Inputs: From "Basic LLM Chain".  
    - Outputs: To "Code3" node for further processing.  
    - Failure: Usually none unless variables are missing.

  - **Audio Scripts**  
    - Type: Set Node  
    - Role: Sets variables or parameters specifically for audio (voice) generation prompts.  
    - Inputs: From "Basic LLM Chain".  
    - Outputs: To "Code2" node for processing.  
    - Failure: None typical.

---

#### 2.3 Image and Audio Generation

- **Overview:** This block generates the actual images and voice audio clips based on AI-generated scripts, using HTTP API calls to fal.ai for images and ElevenLabs for voice.
- **Nodes Involved:**  
  - Code3 (Code)  
  - Image Gen (HTTP Request)  
  - Image Get (HTTP Request)  
  - Code (Code)  
  - Code2 (Code)  
  - Voice Gen1 (HTTP Request)  
  - Get Voice (HTTP Request)  
  - Code1 (Code)

- **Node Details:**

  - **Code3**  
    - Type: Code Node (JavaScript)  
    - Role: Prepares or transforms data for image generation API calls.  
    - Inputs: From "Image Scripts".  
    - Outputs: To "Image Gen".  
    - Failure: Script errors or unexpected input data structure.

  - **Image Gen**  
    - Type: HTTP Request  
    - Role: Calls fal.ai or equivalent image generation API with prepared prompts.  
    - Configuration: Requires API endpoint URL, authentication (API key), and POST data containing prompt.  
    - Inputs: From "Code3".  
    - Outputs: To "Image Get".  
    - Failure: HTTP errors, authentication failure, API quota limits.

  - **Image Get**  
    - Type: HTTP Request  
    - Role: Fetches the generated image or image metadata from the image service.  
    - Inputs: From "Image Gen".  
    - Outputs: To "Code".  
    - Failure: HTTP errors, missing or delayed image availability.

  - **Code**  
    - Type: Code Node  
    - Role: Processes the image data to a usable format or sets variables for next steps.  
    - Inputs: From "Image Get".  
    - Outputs: To "Merge" node.  
    - Failure: Data processing errors.

  - **Code2**  
    - Type: Code Node  
    - Role: Prepares audio generation data based on audio scripts.  
    - Inputs: From "Audio Scripts".  
    - Outputs: To "Voice Gen1".  
    - Failure: Script errors.

  - **Voice Gen1**  
    - Type: HTTP Request  
    - Role: Calls ElevenLabs or similar text-to-speech API to generate voice audio.  
    - Inputs: From "Code2".  
    - Outputs: To "Get Voice".  
    - Failure: HTTP errors, auth failures, API limits.

  - **Get Voice**  
    - Type: HTTP Request  
    - Role: Retrieves generated voice audio or metadata.  
    - Inputs: From "Voice Gen1".  
    - Outputs: To "Code1".  
    - Failure: HTTP errors or audio not ready.

  - **Code1**  
    - Type: Code Node  
    - Role: Processes voice audio data for further integration.  
    - Inputs: From "Get Voice".  
    - Outputs: To "Merge".  
    - Failure: Data processing errors.

---

#### 2.4 Video Creation and Editing

- **Overview:** This block takes prepared audio and image/video data, generates video clips, trims them, combines audio, concatenates clips, and adds captions to produce the final motivational video.
- **Nodes Involved:**  
  - Merge (Merge)  
  - Set Variables (Set)  
  - Build Faceless Array (Code)  
  - Split Out (Split Out)  
  - Generate Video (HTTP Request)  
  - Create Array with Videos (Code)  
  - Split Items (Split Out)  
  - Get Audio Metadata (HTTP Request)  
  - Trim Video (HTTP Request)  
  - Combine Audio + Video (HTTP Request)  
  - Build Video Array (Code)  
  - Concatenate Video (HTTP Request)  
  - Caption Video (HTTP Request)

- **Node Details:**

  - **Merge**  
    - Type: Merge Node  
    - Role: Combines processed image and audio data for synchronized video generation.  
    - Inputs: From "Code" and "Code1".  
    - Outputs: To "Set Variables".  
    - Failure: Mismatched data arrays or timing issues.

  - **Set Variables**  
    - Type: Set Node  
    - Role: Defines variables needed for video generation and array building.  
    - Inputs: From "Merge".  
    - Outputs: To "Build Faceless Array".  
    - Failure: Missing variables or incorrect setup.

  - **Build Faceless Array**  
    - Type: Code Node  
    - Role: Constructs an array of video segments to be generated, each faceless (likely just images + audio).  
    - Inputs: From "Set Variables".  
    - Outputs: To "Split Out".  
    - Failure: Logic errors in array construction.

  - **Split Out**  
    - Type: Split Out Node  
    - Role: Splits array items to process each video segment independently.  
    - Inputs: From "Build Faceless Array".  
    - Outputs: To "Generate Video".  
    - Failure: Empty arrays or split errors.

  - **Generate Video**  
    - Type: HTTP Request  
    - Role: Calls video generation API to create video clips from image and audio inputs.  
    - Inputs: From "Split Out".  
    - Outputs: To "Create Array with Videos".  
    - Failure: API errors, timeouts.

  - **Create Array with Videos**  
    - Type: Code Node  
    - Role: Aggregates generated video clips into an array for further processing.  
    - Inputs: From "Generate Video".  
    - Outputs: To "Split Items".  
    - Failure: Data aggregation errors.

  - **Split Items**  
    - Type: Split Out Node  
    - Role: Splits video items to individually process audio metadata.  
    - Inputs: From "Create Array with Videos".  
    - Outputs: To "Get Audio Metadata".  
    - Failure: Split errors or empty items.

  - **Get Audio Metadata**  
    - Type: HTTP Request  
    - Role: Retrieves metadata such as duration for audio tracks to assist with trimming.  
    - Inputs: From "Split Items".  
    - Outputs: To "Trim Video".  
    - Failure: HTTP errors, missing metadata.

  - **Trim Video**  
    - Type: HTTP Request  
    - Role: Trims video clips based on audio metadata to ensure synchronization.  
    - Inputs: From "Get Audio Metadata".  
    - Outputs: To "Combine Audio + Video".  
    - Failure: API errors, incorrect trimming parameters.

  - **Combine Audio + Video**  
    - Type: HTTP Request  
    - Role: Merges trimmed video clips with respective audio tracks.  
    - Inputs: From "Trim Video".  
    - Outputs: To "Build Video Array".  
    - Failure: Muxing errors, audio/video sync issues.

  - **Build Video Array**  
    - Type: Code Node  
    - Role: Builds the final array of combined video segments.  
    - Inputs: From "Combine Audio + Video".  
    - Outputs: To "Concatenate Video".  
    - Failure: Data aggregation errors.

  - **Concatenate Video**  
    - Type: HTTP Request  
    - Role: Concatenates all video segments into one final video file.  
    - Inputs: From "Build Video Array".  
    - Outputs: To "Caption Video".  
    - Failure: API errors, file size or length limits.

  - **Caption Video**  
    - Type: HTTP Request  
    - Role: Adds captions or subtitles to the final video.  
    - Inputs: From "Concatenate Video".  
    - Outputs: Endpoint for final video output or storage.  
    - Failure: Caption file errors, API failures.

---

#### 2.5 Output Assembly

- **Overview:** Finalizes the video output arrays and prepares them for export or upload.
- **Nodes Involved:**  
  - Build Video Array (Code)  
  - Concatenate Video (HTTP Request)  
  - Caption Video (HTTP Request)

- **Node Details:**  
  (Covered above in Block 2.4 as part of video creation and assembly)

---

### 3. Summary Table

| Node Name                   | Node Type                     | Functional Role                                  | Input Node(s)                         | Output Node(s)                   | Sticky Note |
|-----------------------------|-------------------------------|-------------------------------------------------|-------------------------------------|---------------------------------|-------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Starts the workflow manually                      | -                                   | Get row(s) in sheet             |             |
| Get row(s) in sheet          | Google Sheets                 | Retrieves motivational data                       | When clicking ‘Execute workflow’     | Basic LLM Chain                 |             |
| Basic LLM Chain              | LangChain LLM Chain           | AI text generation and chaining                   | Get row(s) in sheet                  | Image Scripts, Audio Scripts    |             |
| Ollama Chat Model            | Ollama AI Chat                | AI language model for text generation              | Basic LLM Chain (LLM role)           | Basic LLM Chain (AI output)     |             |
| Structured Output Parser     | LangChain Output Parser       | Parses AI output into structured format            | Ollama Chat Model                   | Basic LLM Chain                 |             |
| Image Scripts                | Set                          | Sets variables for image generation prompts        | Basic LLM Chain                     | Code3                          |             |
| Audio Scripts                | Set                          | Sets variables for audio generation prompts        | Basic LLM Chain                     | Code2                          |             |
| Code3                       | Code                         | Prepares data for image generation API call        | Image Scripts                      | Image Gen                      |             |
| Image Gen                   | HTTP Request                 | Calls image generation API                           | Code3                             | Image Get                      |             |
| Image Get                   | HTTP Request                 | Retrieves generated image or metadata                | Image Gen                         | Code                          |             |
| Code                        | Code                         | Processes image data                                 | Image Get                         | Merge                         |             |
| Code2                       | Code                         | Prepares data for voice generation API call          | Audio Scripts                     | Voice Gen1                    |             |
| Voice Gen1                  | HTTP Request                 | Calls voice generation API                            | Code2                            | Get Voice                     |             |
| Get Voice                   | HTTP Request                 | Retrieves generated voice audio                        | Voice Gen1                       | Code1                         |             |
| Code1                       | Code                         | Processes voice audio data                            | Get Voice                        | Merge                         |             |
| Merge                       | Merge                        | Combines image and audio data                         | Code, Code1                     | Set Variables                 |             |
| Set Variables               | Set                          | Sets variables for video generation                   | Merge                           | Build Faceless Array          |             |
| Build Faceless Array        | Code                         | Builds array of faceless video segments               | Set Variables                   | Split Out                    |             |
| Split Out                   | Split Out                    | Splits video segments for parallel processing         | Build Faceless Array            | Generate Video               |             |
| Generate Video              | HTTP Request                 | Calls API to generate video clips                      | Split Out                      | Create Array with Videos     |             |
| Create Array with Videos    | Code                         | Aggregates video clips into array                       | Generate Video                 | Split Items                  |             |
| Split Items                 | Split Out                    | Splits video clips for audio metadata retrieval         | Create Array with Videos       | Get Audio Metadata           |             |
| Get Audio Metadata          | HTTP Request                 | Retrieves audio metadata for trimming                    | Split Items                   | Trim Video                  |             |
| Trim Video                 | HTTP Request                 | Trims video clips to sync with audio                      | Get Audio Metadata            | Combine Audio + Video       |             |
| Combine Audio + Video       | HTTP Request                 | Merges trimmed video and audio clips                      | Trim Video                   | Build Video Array           |             |
| Build Video Array           | Code                         | Builds final array of combined video clips                 | Combine Audio + Video         | Concatenate Video           |             |
| Concatenate Video           | HTTP Request                 | Concatenates video clips into single video                  | Build Video Array             | Caption Video               |             |
| Caption Video              | HTTP Request                 | Adds captions to final video                                  | Concatenate Video             | -                           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To allow manual execution start of workflow.

2. **Google Sheets Node: Get row(s) in sheet**  
   - Connect from Manual Trigger node.  
   - Configure with Google Sheets OAuth2 credentials.  
   - Set sheet and range to fetch motivational scripts or prompts.

3. **LangChain Basic LLM Chain Node**  
   - Connect from Google Sheets node.  
   - Configure with Ollama Chat Model as LLM.  
   - Add Structured Output Parser for response parsing.  
   - Configure prompt templates for motivational text generation.

4. **Ollama Chat Model Node**  
   - Connect from Basic LLM Chain as AI language model.  
   - Set Ollama API credentials.  
   - Configure for chat-based generation of scripts.

5. **Structured Output Parser Node**  
   - Connect from Ollama Chat Model output in Basic LLM Chain.  
   - Define output schema to parse motivational scripts and prompts.

6. **Set Nodes for Image Scripts and Audio Scripts**  
   - Create two Set nodes, connect both from Basic LLM Chain output.  
   - Configure variables for image generation prompt and audio script prompt respectively.

7. **Code Nodes for Preprocessing**  
   - Code3 (for image generation): Connect from Image Scripts node.  
   - Code2 (for voice generation): Connect from Audio Scripts node.  
   - Write JavaScript to prepare API call payloads.

8. **HTTP Request Node for Image Generation (Image Gen)**  
   - Connect from Code3.  
   - Configure with fal.ai API endpoint and API key.  
   - POST request with prompt data.

9. **HTTP Request Node for Fetching Image (Image Get)**  
   - Connect from Image Gen.  
   - Fetch the generated image or its metadata.

10. **Code Node (Code)**  
    - Connect from Image Get.  
    - Process image response, prepare data for merge.

11. **HTTP Request Node for Voice Generation (Voice Gen1)**  
    - Connect from Code2.  
    - Configure with ElevenLabs TTS API credentials.  
    - POST request with text-to-speech payload.

12. **HTTP Request Node for Fetching Voice (Get Voice)**  
    - Connect from Voice Gen1.  
    - Retrieve voice audio.

13. **Code Node (Code1)**  
    - Connect from Get Voice.  
    - Process audio data for merge.

14. **Merge Node**  
    - Connect from Code and Code1.  
    - Combine image and audio data into one dataset.

15. **Set Variables Node**  
    - Connect from Merge.  
    - Define variables needed for video clip generation.

16. **Code Node (Build Faceless Array)**  
    - Connect from Set Variables.  
    - Build array of video segments to process faceless videos.

17. **Split Out Node**  
    - Connect from Build Faceless Array.  
    - Split array into separate items for parallel processing.

18. **HTTP Request Node (Generate Video)**  
    - Connect from Split Out.  
    - Configure video generation API endpoint; send image+audio data.

19. **Code Node (Create Array with Videos)**  
    - Connect from Generate Video.  
    - Aggregate generated video segments.

20. **Split Out Node (Split Items)**  
    - Connect from Create Array with Videos.  
    - Prepare each video item for metadata retrieval.

21. **HTTP Request Node (Get Audio Metadata)**  
    - Connect from Split Items.  
    - Retrieve audio metadata like duration.

22. **HTTP Request Node (Trim Video)**  
    - Connect from Get Audio Metadata.  
    - Trim videos according to audio metadata.

23. **HTTP Request Node (Combine Audio + Video)**  
    - Connect from Trim Video.  
    - Merge trimmed video with audio tracks.

24. **Code Node (Build Video Array)**  
    - Connect from Combine Audio + Video.  
    - Aggregate combined video clips.

25. **HTTP Request Node (Concatenate Video)**  
    - Connect from Build Video Array.  
    - Concatenate all video clips into final video.

26. **HTTP Request Node (Caption Video)**  
    - Connect from Concatenate Video.  
    - Add captions or subtitles.

27. **Configure all HTTP Request nodes with proper API URLs, authentication, headers, and data formats as per respective external APIs (fal.ai, ElevenLabs, video editing services).**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                           |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| This workflow integrates multiple AI services, including Ollama AI for text generation, fal.ai for images, and ElevenLabs for voice generation. | Project integration overview                              |
| Video editing APIs must support trimming, concatenation, and captioning functionalities with HTTP endpoints. | Video processing requirements                             |
| Credentials setup for Google Sheets, Ollama AI, fal.ai, and ElevenLabs APIs are mandatory for execution.      | Credential configuration specifics                        |
| Consider API rate limits and error handling for robustness in production environments.                        | Operational considerations                                |
| Sticky Notes in the workflow contain empty content but can be used for inline documentation or future annotations. | Workflow documentation practice                           |

---

**Disclaimer:** The provided analysis is based exclusively on the n8n workflow configuration shared. The workflow respects applicable content policies and processes only legal and public data.

---