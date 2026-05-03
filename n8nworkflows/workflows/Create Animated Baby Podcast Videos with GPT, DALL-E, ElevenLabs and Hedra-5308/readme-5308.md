Create Animated Baby Podcast Videos with GPT, DALL-E, ElevenLabs and Hedra

https://n8nworkflows.xyz/workflows/create-animated-baby-podcast-videos-with-gpt--dall-e--elevenlabs-and-hedra-5308


# Create Animated Baby Podcast Videos with GPT, DALL-E, ElevenLabs and Hedra

---
### 1. Workflow Overview

This workflow automates the creation of animated baby podcast videos by integrating advanced AI tools and multimedia platforms. It takes user input to generate podcast scripts and corresponding AI-generated baby-themed images, converts the script into audio, and then combines these assets to produce animated videos. Finally, it uploads the completed videos to Google Drive for storage and access.

The workflow’s logic is grouped into these main functional blocks:

- **1.1 Input Reception:** Receives podcast parameters via a form trigger.
- **1.2 AI Content Generation:** Generates podcast script text and baby-themed images using GPT (OpenAI) and DALL·E.
- **1.3 Audio Synthesis:** Converts the generated script into audio using ElevenLabs or similar TTS services.
- **1.4 Video Assembly with Hedra:** Uploads image and audio assets to Hedra, a video composition platform, to produce the animated baby podcast video.
- **1.5 Video Retrieval and Storage:** Downloads the final video from Hedra and uploads it to Google Drive for user access.
- **1.6 Data Preparation and Merging:** Handles binary data extraction, merging, and format conversions necessary for smooth asset transfer between services.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures user inputs to initiate the podcast video creation process.

- **Nodes Involved:**  
  - Generate Baby Podcast (Form Trigger)

- **Node Details:**  
  - **Generate Baby Podcast**  
    - Type: Form Trigger  
    - Role: Entry point that receives user parameters for the podcast (e.g., topic, style).  
    - Configuration: Uses a webhook ID to listen for form submissions.  
    - Inputs: External HTTP request (form submission).  
    - Outputs: Triggers the next nodes for image and script generation.  
    - Edge Cases: Missing or malformed form data may lead to incomplete generation downstream.

#### 2.2 AI Content Generation

- **Overview:**  
  Generates the podcast script and baby-themed images using OpenAI's GPT and DALL·E models.

- **Nodes Involved:**  
  - Baby Image Generator (OpenAI Langchain node)  
  - Script (OpenAI Langchain node)  
  - Open AI Generate Image (HTTP Request)  
  - B64 String to File (Convert to File)  

- **Node Details:**  
  - **Baby Image Generator**  
    - Type: OpenAI Langchain Node  
    - Role: Generates image prompts or requests for baby-themed images via OpenAI.  
    - Configuration: Uses GPT/DALL·E credentials; handles prompt input from form trigger.  
    - Outputs: Base64 encoded images.  
    - Edge Cases: API rate limits, invalid prompts, or model failures.  
  - **Script**  
    - Type: OpenAI Langchain Node  
    - Role: Generates podcast script text based on inputs.  
    - Configuration: GPT model with prompt based on user input.  
    - Outputs: Text for audio synthesis.  
    - Edge Cases: API errors, incomplete text generation.  
  - **Open AI Generate Image**  
    - Type: HTTP Request  
    - Role: Requests image generation via OpenAI API using the prompt from Baby Image Generator.  
    - Configuration: HTTP POST with image generation parameters.  
    - Outputs: Base64 image data.  
    - Edge Cases: HTTP errors, authentication issues.  
  - **B64 String to File**  
    - Type: Convert to File  
    - Role: Converts Base64 image string to binary file format for upload.  
    - Inputs: Base64 string from image generation.  
    - Outputs: Binary file suitable for Hedra upload.  
    - Edge Cases: Invalid Base64 data causing conversion failure.

#### 2.3 Audio Synthesis

- **Overview:**  
  Converts the generated podcast script into audio files using TTS services (e.g., ElevenLabs).

- **Nodes Involved:**  
  - Audio Creation (HTTP Request)  
  - Create Audio Asset For Hedra (HTTP Request)  
  - Audio To Hedra (HTTP Request)  
  - Extract And Combine Binary and Array (Code)  
  - Merge1 (Merge)  

- **Node Details:**  
  - **Audio Creation**  
    - Type: HTTP Request  
    - Role: Sends the podcast script text to TTS API to generate audio.  
    - Configuration: HTTP POST with script text and voice parameters.  
    - Outputs: Binary or Base64 audio data.  
    - Edge Cases: API failure, unsupported audio format.  
  - **Create Audio Asset For Hedra**  
    - Type: HTTP Request  
    - Role: Uploads audio binary to Hedra platform as an asset.  
    - Inputs: Audio data from Audio Creation.  
    - Outputs: Asset ID or confirmation from Hedra.  
    - Edge Cases: Network errors, invalid file format.  
  - **Audio To Hedra**  
    - Type: HTTP Request  
    - Role: Possibly a follow-up request to finalize audio asset or link it within Hedra.  
    - Inputs: Processed audio asset info.  
    - Outputs: Confirmation or further asset data.  
    - Edge Cases: Hedra API errors.  
  - **Extract And Combine Binary and Array**  
    - Type: Code  
    - Role: Processes and merges audio data formats for compatibility.  
    - Inputs: Audio data streams.  
    - Outputs: Combined audio binary ready for Hedra.  
    - Edge Cases: Data corruption, script errors.  
  - **Merge1**  
    - Type: Merge  
    - Role: Synchronizes multiple inputs before processing audio data.  
    - Inputs: Audio data and metadata.  
    - Outputs: Unified data stream.  
    - Edge Cases: Mismatched input lengths or timing.

#### 2.4 Video Assembly with Hedra

- **Overview:**  
  Uploads image and audio assets to Hedra, composes the animated video, and retrieves the final video file.

- **Nodes Involved:**  
  - Create Image Asset For Hedra (HTTP Request)  
  - Post Open AI Image To Hedra (HTTP Request)  
  - Merge2 (Merge)  
  - Code2 (Code)  
  - Generate Video Asset Hedra (HTTP Request)  
  - Get Our Baby Video File From Hedra (HTTP Request)  
  - Download Our Video (HTTP Request)  

- **Node Details:**  
  - **Create Image Asset For Hedra**  
    - Type: HTTP Request  
    - Role: Uploads the converted baby image binary to Hedra as a visual asset.  
    - Inputs: File from B64 String to File node.  
    - Outputs: Hedra asset reference.  
    - Edge Cases: Upload failures, format incompatibility.  
  - **Post Open AI Image To Hedra**  
    - Type: HTTP Request  
    - Role: Secondary or confirmation step to link the image asset inside Hedra’s environment.  
    - Inputs: Asset data or IDs.  
    - Outputs: Confirmation or asset metadata.  
    - Edge Cases: API errors, invalid asset references.  
  - **Merge2**  
    - Type: Merge  
    - Role: Combines audio and image asset data streams to prepare for video creation.  
    - Inputs: Audio and image asset information.  
    - Outputs: Unified payload for video generation.  
    - Edge Cases: Mismatched inputs causing failure downstream.  
  - **Code2**  
    - Type: Code  
    - Role: Custom scripting to format or prepare payload for Hedra’s video generation API.  
    - Inputs: Merged asset data.  
    - Outputs: Finalized request structure.  
    - Edge Cases: Script errors, malformed requests.  
  - **Generate Video Asset Hedra**  
    - Type: HTTP Request  
    - Role: Sends the combined asset data to Hedra’s API to trigger video generation.  
    - Outputs: Video generation job ID or confirmation.  
    - Edge Cases: API limits, processing errors.  
  - **Get Our Baby Video File From Hedra**  
    - Type: HTTP Request  
    - Role: Polls or retrieves the completed video file after generation.  
    - Outputs: Video file binary or download URL.  
    - Edge Cases: Timeouts, file not ready.  
  - **Download Our Video**  
    - Type: HTTP Request  
    - Role: Downloads the video file from Hedra for further processing or storage.  
    - Outputs: Binary video file.  
    - Edge Cases: Network errors, incomplete downloads.

#### 2.5 Video Retrieval and Storage

- **Overview:**  
  Uploads the finalized video file to Google Drive for user access and archival.

- **Nodes Involved:**  
  - Upload Bin File To Conver In Google Drive (Google Drive)  
  - Download Video (Google Drive)  

- **Node Details:**  
  - **Upload Bin File To Conver In Google Drive**  
    - Type: Google Drive  
    - Role: Uploads the video binary file to a specified folder on Google Drive.  
    - Configuration: Requires Google Drive OAuth2 credentials.  
    - Inputs: Video binary from Download Our Video node.  
    - Outputs: File metadata including Drive URL.  
    - Edge Cases: Auth errors, quota limits.  
  - **Download Video**  
    - Type: Google Drive  
    - Role: Enables download or sharing of the uploaded video file.  
    - Inputs: Reference to uploaded video file.  
    - Outputs: Downloadable link or binary data.  
    - Edge Cases: Permission errors.

#### 2.6 Data Preparation and Merging

- **Overview:**  
  Contains utility nodes for merging data streams and converting between binary and array formats to ensure compatibility between nodes.

- **Nodes Involved:**  
  - Merge (Merge)  
  - Merge1 (Merge)  
  - Merge2 (Merge)  
  - Extract And Combine Binary and Array (Code)  
  - Extract And Combine Binary and Array1 (Code)  

- **Node Details:**  
  - **Merge, Merge1, Merge2**  
    - Type: Merge  
    - Role: Synchronize and combine multiple input data streams for further processing.  
    - Configuration: Default merge behavior (e.g., Wait for all inputs).  
    - Edge Cases: Missing inputs causing stalled merges.  
  - **Extract And Combine Binary and Array, Extract And Combine Binary and Array1**  
    - Type: Code  
    - Role: Custom code to extract binary data and combine arrays, ensuring data integrity for media uploads.  
    - Edge Cases: Data corruption or unexpected input types.

---

### 3. Summary Table

| Node Name                         | Node Type                 | Functional Role                           | Input Node(s)                      | Output Node(s)                      | Sticky Note                         |
|----------------------------------|---------------------------|-----------------------------------------|----------------------------------|-----------------------------------|-----------------------------------|
| Generate Baby Podcast             | Form Trigger              | Input reception for podcast parameters   | (Webhook trigger)                 | Baby Image Generator, Script       |                                   |
| Baby Image Generator             | OpenAI Langchain          | Generate baby-themed image prompts      | Generate Baby Podcast            | Open AI Generate Image             |                                   |
| Script                          | OpenAI Langchain          | Generate podcast script text             | Generate Baby Podcast            | Audio Creation                    |                                   |
| Open AI Generate Image          | HTTP Request              | Request image generation via OpenAI API | Baby Image Generator             | B64 String to File                |                                   |
| B64 String to File              | Convert to File           | Convert Base64 image to binary file      | Open AI Generate Image           | Merge, Create Image Asset For Hedra |                                   |
| Merge                          | Merge                    | Combine image binary with other data     | B64 String to File, Create Image Asset For Hedra | Extract And Combine Binary and Array1, Create Image Asset For Hedra |                                   |
| Create Image Asset For Hedra    | HTTP Request              | Upload image file to Hedra                | B64 String to File, Merge        | Post Open AI Image To Hedra       |                                   |
| Post Open AI Image To Hedra     | HTTP Request              | Confirm or link image asset in Hedra     | Create Image Asset For Hedra     | Merge2                           |                                   |
| Merge2                         | Merge                    | Combine audio and image assets            | Audio To Hedra, Post Open AI Image To Hedra | Code2                           |                                   |
| Code2                          | Code                     | Prepare payload for video generation      | Merge2                         | Generate Video Asset Hedra        |                                   |
| Generate Video Asset Hedra       | HTTP Request              | Trigger video generation in Hedra         | Code2                          | Get Our Baby Video File From Hedra |                                   |
| Get Our Baby Video File From Hedra | HTTP Request           | Retrieve generated video file             | Generate Video Asset Hedra       | Download Our Video                |                                   |
| Download Our Video              | HTTP Request              | Download video file binary                 | Get Our Baby Video File From Hedra | Upload Bin File To Conver In Google Drive |                                   |
| Upload Bin File To Conver In Google Drive | Google Drive         | Upload video file to Google Drive          | Download Our Video              | Download Video                   |                                   |
| Download Video                 | Google Drive              | Enable download or sharing of video file  | Upload Bin File To Conver In Google Drive |                               |                                   |
| Audio Creation                | HTTP Request              | Convert podcast script text to audio      | Script                         | Create Audio Asset For Hedra, Merge1 |                                   |
| Create Audio Asset For Hedra    | HTTP Request              | Upload audio file to Hedra                | Audio Creation                 | Audio To Hedra                   |                                   |
| Audio To Hedra                 | HTTP Request              | Finalize audio asset in Hedra             | Create Audio Asset For Hedra, Extract And Combine Binary and Array | Merge2                           |                                   |
| Extract And Combine Binary and Array | Code                 | Process and merge audio binary data       | Merge1                         | Audio To Hedra                   |                                   |
| Extract And Combine Binary and Array1 | Code                 | Process and merge image binary data       | Merge                         | Post Open AI Image To Hedra       |                                   |
| Merge1                         | Merge                    | Combine audio data inputs                   | Audio Creation, Create Audio Asset For Hedra | Extract And Combine Binary and Array |                                   |
| Merge                          | Merge                    | Combine image binary inputs                 | B64 String to File, Create Image Asset For Hedra | Extract And Combine Binary and Array1 |                                   |
| Sticky Note                    | Sticky Note               | (Empty, no content)                        | -                              | -                               |                                   |
| Sticky Note1                   | Sticky Note               | (Empty, no content)                        | -                              | -                               |                                   |
| Sticky Note2                   | Sticky Note               | (Empty, no content)                        | -                              | -                               |                                   |
| Sticky Note3                   | Sticky Note               | (Empty, no content)                        | -                              | -                               |                                   |
| Sticky Note5                   | Sticky Note               | (Empty, no content)                        | -                              | -                               |                                   |
| Sticky Note7                   | Sticky Note               | (Empty, no content)                        | -                              | -                               |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named "Generate Baby Podcast" to receive user inputs for the baby podcast (e.g., topic, style). Set up a webhook for form submission.

2. **Add two OpenAI Langchain nodes:**  
   - "Baby Image Generator": Configure to generate baby-themed image prompts using the OpenAI GPT or DALL·E model with appropriate credentials.  
   - "Script": Configure to generate the podcast script text from the form input using GPT.

3. **Add an HTTP Request node** named "Open AI Generate Image" to call OpenAI’s image generation API with the prompt from "Baby Image Generator". Configure with POST method and necessary headers (API key).

4. **Insert a Convert to File node** named "B64 String to File" to convert the Base64 image response from "Open AI Generate Image" into a binary file.

5. **Add a Merge node** named "Merge" to synchronize the image file and related data.

6. **Add an HTTP Request node** named "Create Image Asset For Hedra" to upload the binary image file to Hedra. Configure with Hedra API endpoint, authentication, and proper headers.

7. **Add an HTTP Request node** named "Post Open AI Image To Hedra" to link or confirm the image asset inside Hedra. Connect the output of "Create Image Asset For Hedra" here.

8. **Add a Merge node** named "Merge2" to combine image and audio asset data before video creation.

9. **Add a Code node** named "Code2" to format the combined payload correctly for Hedra video creation API.

10. **Add an HTTP Request node** named "Generate Video Asset Hedra" to send the combined assets to Hedra and trigger video generation.

11. **Add an HTTP Request node** named "Get Our Baby Video File From Hedra" to poll or retrieve the finished video file from Hedra.

12. **Add an HTTP Request node** named "Download Our Video" to download the actual video binary file.

13. **Add a Google Drive node** named "Upload Bin File To Conver In Google Drive" to upload the downloaded video file to a designated Google Drive folder. Configure OAuth2 credentials accordingly.

14. **Add a Google Drive node** named "Download Video" to facilitate downloading or sharing the uploaded video file.

15. **For audio synthesis:**  
    - Add an HTTP Request node named "Audio Creation", configured to send the "Script" output to an ElevenLabs or similar TTS API to generate audio.  
    - Add an HTTP Request node named "Create Audio Asset For Hedra" to upload the generated audio to Hedra.  
    - Add a Merge node "Merge1" to synchronize audio data streams.  
    - Add a Code node "Extract And Combine Binary and Array" to process audio binary data.  
    - Add an HTTP Request node "Audio To Hedra" to finalize audio asset handling in Hedra.  
    - Connect "Audio To Hedra" output to "Merge2" for final video payload merging.

16. **Add Merge nodes ("Merge", "Merge1", "Merge2") and Code nodes ("Extract And Combine Binary and Array", "Extract And Combine Binary and Array1") as necessary to handle data format conversions and synchronization between image, audio, and video assets.**

17. **Ensure all HTTP Request nodes are properly configured with authentication, endpoints, headers, and expected payload formats.**

18. **Set error handling policies such as "continue on error" for AI nodes to maintain workflow resilience.**

19. **Test the complete workflow end to end with sample inputs to verify the generation of baby podcast videos and their upload to Google Drive.**

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                              |
|--------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow uses Hedra API for advanced video composition integrating AI-generated assets.          | Hedra platform documentation (external resource)             |
| OpenAI GPT and DALL·E are leveraged for script and image generation within Langchain nodes.      | OpenAI API docs: https://platform.openai.com/docs/           |
| ElevenLabs or similar TTS API is used for audio synthesis; credentials must be configured.       | ElevenLabs docs: https://elevenlabs.io/docs                   |
| Google Drive nodes require OAuth2 credentials with appropriate drive file permissions.           | Google Drive API: https://developers.google.com/drive/api    |
| Proper error handling and data merging via Merge and Code nodes ensure robust media file handling.| n8n official docs on Merge node and Code node usage           |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.