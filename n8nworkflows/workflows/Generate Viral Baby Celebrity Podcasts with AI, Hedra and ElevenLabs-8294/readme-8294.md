Generate Viral Baby Celebrity Podcasts with AI, Hedra and ElevenLabs

https://n8nworkflows.xyz/workflows/generate-viral-baby-celebrity-podcasts-with-ai--hedra-and-elevenlabs-8294


# Generate Viral Baby Celebrity Podcasts with AI, Hedra and ElevenLabs

### 1. Workflow Overview

This workflow automates the creation of viral baby celebrity podcasts by leveraging AI-generated content, multimedia asset creation, and publishing pipelines. It integrates AI language models (OpenAI via LangChain), text-to-speech conversion, image and audio asset generation via the Hedra platform, and storage/upload to Google Drive.

**Target Use Cases:**
- Automated generation of podcast ideas and scripts featuring baby celebrity themes.
- Creation of corresponding audio and visual assets.
- Assembly of multimedia content into a video podcast.
- Scheduling and triggering for periodic content generation.
- Automated publishing and storage of final video assets.

**Logical Blocks:**

- **1.1 Trigger and Idea Generation**  
  Initiates the workflow either manually or via schedule, fetches existing podcast ideas from Google Sheets, processes them, and generates new ideas and prompts using AI agents.

- **1.2 AI Prompting and Content Generation**  
  Uses AI language models to generate audio script prompts and image prompts based on ideas.

- **1.3 Asset Creation and Conversion**  
  Converts AI-generated text to speech, generates audio assets, creates images from prompts, converts assets to files, and uploads them to Hedra.

- **1.4 Asset Merging and Video Creation**  
  Merges the audio and image assets, uploads them, creates a video podcast on Hedra, waits for processing, then retrieves and downloads the generated video.

- **1.5 Video Upload and Finalization**  
  Uploads the downloaded video to Google Drive for storage and potential further distribution.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Idea Generation

- **Overview:**  
  Starts the workflow either manually or on schedule, reads existing podcast ideas from Google Sheets, processes them, and generates new ideas.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Schedule Trigger1  
  - Get Existing Ideas (Google Sheets)  
  - Sheet Process (Code)  
  - Idea Generator (LangChain Agent)  

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual trigger  
    - Role: Allows manual start for testing  
    - Inputs: None  
    - Outputs: Triggers "Get Existing Ideas"  
    - Edge cases: None  
    - Version: v1

  - **Schedule Trigger1**  
    - Type: Scheduled trigger  
    - Role: Periodically triggers the workflow automatically  
    - Inputs: None  
    - Outputs: Triggers "Get Existing Ideas"  
    - Version: v1.2  
    - Edge cases: Scheduling misconfiguration or missed runs

  - **Get Existing Ideas**  
    - Type: Google Sheets node  
    - Role: Reads existing podcast ideas for input to AI  
    - Configuration: Reads a specific sheet with ideas  
    - Inputs: Trigger node outputs  
    - Outputs: Data to "Sheet Process"  
    - Version: v4.5  
    - Failure: Connectivity or permission errors

  - **Sheet Process**  
    - Type: Code node  
    - Role: Processes raw sheet data into structured input for AI  
    - Inputs: From "Get Existing Ideas"  
    - Outputs: Structured data to "Idea Generator"  
    - Edge cases: Data format inconsistencies or parsing errors

  - **Idea Generator**  
    - Type: LangChain Agent (AI)  
    - Role: Generates new podcast ideas and prompts based on processed sheet data  
    - Inputs: Processed data from "Sheet Process"  
    - Outputs: Sends data to "Audio Prompt & Title" and "Image Prompt" nodes  
    - Version: v1.7  
    - Edge cases: AI response failures or API limits  
    - Uses OpenAI Chat Model as sub-model

#### 1.2 AI Prompting and Content Generation

- **Overview:**  
  Generates prompts for audio and image content using OpenAI models, preparing scripts and visual descriptions.

- **Nodes Involved:**  
  - Audio Prompt & Title (LangChain OpenAI)  
  - Image Prompt (LangChain OpenAI)  
  - OpenAI Chat Model (Language Model)  
  - Structured Output Parser (Structured output parsing for Idea Generator)  

- **Node Details:**

  - **Audio Prompt & Title**  
    - Type: LangChain OpenAI  
    - Role: Generates audio script and podcast title  
    - Inputs: From "Idea Generator"  
    - Outputs: To "Text to Speech"  
    - Version: v1.8  
    - Edge cases: API timeouts, invalid prompt format

  - **Image Prompt**  
    - Type: LangChain OpenAI  
    - Role: Generates image description prompt for baby celebrity podcast art  
    - Inputs: From "Idea Generator"  
    - Outputs: To "Create The Image"  
    - Version: v1.8  
    - Edge cases: API failures, invalid output

  - **OpenAI Chat Model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Underlying language model for AI prompting nodes  
    - Inputs: From "Structured Output Parser" and triggers "Idea Generator"  
    - Outputs: AI-generated text data  
    - Version: v1.2  
    - Edge cases: API key or quota issues

  - **Structured Output Parser**  
    - Type: LangChain Output Parser Structured  
    - Role: Parses AI outputs into structured format for "Idea Generator"  
    - Inputs: From OpenAI Chat Model  
    - Outputs: To "Idea Generator"  
    - Version: v1.2  
    - Edge cases: Parsing errors, inconsistent AI output format

#### 1.3 Asset Creation and Conversion

- **Overview:**  
  Converts AI-generated text to speech, generates audio and image assets via Hedra API, converts raw data to files, and uploads assets to Hedra.

- **Nodes Involved:**  
  - Text to Speech (HTTP Request)  
  - Create Audio Asset (Hedra) (HTTP Request)  
  - Merge Audio Assets (Merge)  
  - Split Audio Metadata (Code)  
  - Upload Audio (Hedra) (HTTP Request)  
  - Image Prompt (LangChain OpenAI)  
  - Create The Image (HTTP Request)  
  - Convert to File (Convert to File)  
  - Create Image Asset (Hedra) (HTTP Request)  
  - Merge 2 (Merge)  
  - Split Image Metadata (Code)  
  - Upload Image (Hedra) (HTTP Request)  

- **Node Details:**

  - **Text to Speech**  
    - Type: HTTP Request  
    - Role: Converts text script to speech audio using TTS API  
    - Inputs: Audio script from "Audio Prompt & Title"  
    - Outputs: To "Create Audio Asset (Hedra)" and "Merge Audio Assets"  
    - Version: v4.2  
    - Edge cases: TTS service failures, auth errors

  - **Create Audio Asset (Hedra)**  
    - Type: HTTP Request  
    - Role: Creates audio asset on Hedra platform from TTS audio data  
    - Inputs: From "Text to Speech" and "Merge Audio Assets"  
    - Outputs: To "Merge Audio Assets" and "Split Audio Metadata"  
    - Version: v4.2  
    - Edge cases: Hedra API errors, network issues

  - **Merge Audio Assets**  
    - Type: Merge  
    - Role: Combines multiple audio asset data streams into one  
    - Inputs: From "Text to Speech" and "Create Audio Asset (Hedra)"  
    - Outputs: To "Split Audio Metadata"  
    - Version: v3.1  

  - **Split Audio Metadata**  
    - Type: Code  
    - Role: Extracts metadata and asset IDs from merged audio data for upload  
    - Inputs: From "Merge Audio Assets"  
    - Outputs: To "Upload Audio (Hedra)"  
    - Version: v2  
    - Edge cases: Unexpected data format

  - **Upload Audio (Hedra)**  
    - Type: HTTP Request  
    - Role: Uploads processed audio asset to Hedra for storage and further processing  
    - Inputs: From "Split Audio Metadata"  
    - Outputs: To "Merge Assets"  
    - Version: v4.2  
    - Edge cases: Upload failures, API auth issues

  - **Image Prompt** (already analyzed in 1.2)

  - **Create The Image**  
    - Type: HTTP Request  
    - Role: Generates image based on prompt from AI via Hedra API  
    - Inputs: From "Image Prompt"  
    - Outputs: To "Convert to File"  
    - Version: v4.2  
    - Edge cases: API failures, image generation errors

  - **Convert to File**  
    - Type: Convert to File  
    - Role: Converts raw image data to file format accepted by Hedra  
    - Inputs: From "Create The Image"  
    - Outputs: To "Create Image Asset (Hedra)" and "Merge 2"  
    - Version: v1.1  

  - **Create Image Asset (Hedra)**  
    - Type: HTTP Request  
    - Role: Creates image asset on Hedra platform  
    - Inputs: From "Convert to File"  
    - Outputs: To "Merge 2" and "Split Image Metadata"  
    - Version: v4.2  

  - **Merge 2**  
    - Type: Merge  
    - Role: Combines image asset data streams  
    - Inputs: From "Convert to File" and "Create Image Asset (Hedra)"  
    - Outputs: To "Split Image Metadata"  
    - Version: v3.1  

  - **Split Image Metadata**  
    - Type: Code  
    - Role: Extracts image asset metadata and IDs for upload  
    - Inputs: From "Merge 2"  
    - Outputs: To "Upload Image (Hedra)"  
    - Version: v2  

  - **Upload Image (Hedra)**  
    - Type: HTTP Request  
    - Role: Uploads image asset to Hedra platform  
    - Inputs: From "Split Image Metadata"  
    - Outputs: To "Merge Assets"  
    - Version: v4.2  

#### 1.4 Asset Merging and Video Creation

- **Overview:**  
  Combines uploaded audio and image assets, creates a video podcast with Hedra API, waits for processing, and downloads the final video.

- **Nodes Involved:**  
  - Merge Assets (Merge)  
  - Split Image & Audio Assets IDs (Code)  
  - Create Video (Hedra) (HTTP Request)  
  - Wait 5 Mins (Wait)  
  - Get BabyPod Video (Hedra) (HTTP Request)  
  - Wait 10 Secs (Wait)  
  - Download BabyPod Video (Hedra) (HTTP Request)  

- **Node Details:**

  - **Merge Assets**  
    - Type: Merge  
    - Role: Combines audio and image asset upload outputs into a single data stream  
    - Inputs: From "Upload Audio (Hedra)" and "Upload Image (Hedra)"  
    - Outputs: To "Split Image & Audio Assets IDs"  
    - Version: v3.1  

  - **Split Image & Audio Assets IDs**  
    - Type: Code  
    - Role: Extracts asset IDs required for video creation  
    - Inputs: From "Merge Assets"  
    - Outputs: To "Create Video (Hedra)"  
    - Version: v2  

  - **Create Video (Hedra)**  
    - Type: HTTP Request  
    - Role: Initiates video creation on Hedra platform using asset IDs  
    - Inputs: From "Split Image & Audio Assets IDs"  
    - Outputs: To "Wait 5 Mins"  
    - Version: v4.2  
    - Edge cases: API failure or rate limits

  - **Wait 5 Mins**  
    - Type: Wait  
    - Role: Allows time for video processing on Hedra  
    - Inputs: From "Create Video (Hedra)"  
    - Outputs: To "Get BabyPod Video (Hedra)"  
    - Version: v1.1  

  - **Get BabyPod Video (Hedra)**  
    - Type: HTTP Request  
    - Role: Retrieves status or link of created video from Hedra  
    - Inputs: From "Wait 5 Mins"  
    - Outputs: To "Wait 10 Secs"  
    - Version: v4.2  

  - **Wait 10 Secs**  
    - Type: Wait  
    - Role: Short delay to ensure video availability before download  
    - Inputs: From "Get BabyPod Video (Hedra)"  
    - Outputs: To "Download BabyPod Video (Hedra)"  
    - Version: v1.1  

  - **Download BabyPod Video (Hedra)**  
    - Type: HTTP Request  
    - Role: Downloads finalized video file from Hedra  
    - Inputs: From "Wait 10 Secs"  
    - Outputs: To "Upload Video"  
    - Version: v4.2  
    - Edge cases: Download errors, timeouts

#### 1.5 Video Upload and Finalization

- **Overview:**  
  Uploads the downloaded video to Google Drive for storage and potential sharing or further processing.

- **Nodes Involved:**  
  - Upload Video (Google Drive)  
  - Download Video (YouTube) (Google Drive)  

- **Node Details:**

  - **Upload Video**  
    - Type: Google Drive  
    - Role: Uploads the final video file to a configured Google Drive folder  
    - Inputs: From "Download BabyPod Video (Hedra)"  
    - Outputs: To "Download Video (YouTube)"  
    - Version: v3  
    - Edge cases: Auth errors, quota exceeded

  - **Download Video (YouTube)**  
    - Type: Google Drive  
    - Role: Downloads video from Google Drive, possibly for verification or further processing  
    - Inputs: From "Upload Video"  
    - Outputs: None (end node)  
    - Version: v3  

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                         | Input Node(s)                          | Output Node(s)                      | Sticky Note            |
|----------------------------|----------------------------------|---------------------------------------|--------------------------------------|-----------------------------------|------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Manual start trigger                   | None                                 | Get Existing Ideas                 |                        |
| Schedule Trigger1           | Schedule Trigger                 | Scheduled start trigger                | None                                 | Get Existing Ideas                 |                        |
| Get Existing Ideas          | Google Sheets                   | Reads existing podcast ideas          | Manual Trigger, Schedule Trigger     | Sheet Process                     |                        |
| Sheet Process              | Code                            | Processes sheet data                   | Get Existing Ideas                   | Idea Generator                   |                        |
| Idea Generator              | LangChain Agent                 | Generates podcast ideas and prompts   | Sheet Process                       | Audio Prompt & Title, Image Prompt |                        |
| OpenAI Chat Model           | LangChain LM Chat OpenAI        | Language model for AI prompts          | Structured Output Parser             | Idea Generator                   |                        |
| Structured Output Parser    | LangChain Output Parser Structured | Parses AI output into structured data | OpenAI Chat Model                   | Idea Generator                   |                        |
| Audio Prompt & Title        | LangChain OpenAI                | Generates audio script and title       | Idea Generator                     | Text to Speech                   |                        |
| Text to Speech              | HTTP Request                   | Converts text to speech audio          | Audio Prompt & Title               | Create Audio Asset, Merge Audio Assets |                        |
| Create Audio Asset (Hedra)  | HTTP Request                   | Creates audio asset on Hedra           | Text to Speech, Merge Audio Assets | Merge Audio Assets, Split Audio Metadata |                        |
| Merge Audio Assets          | Merge                           | Merges audio asset streams             | Text to Speech, Create Audio Asset | Split Audio Metadata             |                        |
| Split Audio Metadata        | Code                            | Extracts audio metadata and IDs        | Merge Audio Assets                 | Upload Audio (Hedra)             |                        |
| Upload Audio (Hedra)        | HTTP Request                   | Uploads audio asset to Hedra            | Split Audio Metadata               | Merge Assets                    |                        |
| Image Prompt                | LangChain OpenAI                | Generates image prompt                  | Idea Generator                     | Create The Image                |                        |
| Create The Image            | HTTP Request                   | Generates image via Hedra               | Image Prompt                     | Convert to File                 |                        |
| Convert to File             | Convert to File                 | Converts image to file format           | Create The Image                 | Create Image Asset, Merge 2      |                        |
| Create Image Asset (Hedra)  | HTTP Request                   | Creates image asset on Hedra            | Convert to File                 | Merge 2, Split Image Metadata   |                        |
| Merge 2                    | Merge                           | Merges image asset streams              | Convert to File, Create Image Asset | Split Image Metadata           |                        |
| Split Image Metadata        | Code                            | Extracts image metadata and IDs         | Merge 2                         | Upload Image (Hedra)            |                        |
| Upload Image (Hedra)        | HTTP Request                   | Uploads image asset to Hedra             | Split Image Metadata            | Merge Assets                   |                        |
| Merge Assets               | Merge                           | Merges audio and image asset uploads    | Upload Audio (Hedra), Upload Image (Hedra) | Split Image & Audio Assets IDs |                        |
| Split Image & Audio Assets IDs | Code                            | Extracts asset IDs for video creation    | Merge Assets                   | Create Video (Hedra)            |                        |
| Create Video (Hedra)        | HTTP Request                   | Creates video podcast on Hedra          | Split Image & Audio Assets IDs   | Wait 5 Mins                   |                        |
| Wait 5 Mins                | Wait                            | Waits for video processing              | Create Video (Hedra)             | Get BabyPod Video (Hedra)      |                        |
| Get BabyPod Video (Hedra)   | HTTP Request                   | Retrieves video status/link              | Wait 5 Mins                    | Wait 10 Secs                  |                        |
| Wait 10 Secs               | Wait                            | Short delay before downloading video    | Get BabyPod Video (Hedra)        | Download BabyPod Video (Hedra) |                        |
| Download BabyPod Video (Hedra) | HTTP Request                   | Downloads final video from Hedra         | Wait 10 Secs                   | Upload Video                  |                        |
| Upload Video               | Google Drive                   | Uploads video to Google Drive            | Download BabyPod Video (Hedra) | Download Video (YouTube)       |                        |
| Download Video (YouTube)    | Google Drive                   | Downloads video from Google Drive        | Upload Video                   | None                         |                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’".  
   - Add a **Schedule Trigger** node named "Schedule Trigger1".

2. **Google Sheets Input:**  
   - Add a **Google Sheets** node "Get Existing Ideas" configured to read the sheet containing existing podcast ideas. Set credentials for Google API access.

3. **Process Sheet Data:**  
   - Add a **Code** node "Sheet Process" connected from "Get Existing Ideas". Implement JavaScript to format and clean the data for AI input.

4. **AI Idea Generation:**  
   - Add a **LangChain Agent** node "Idea Generator" connected from "Sheet Process". Configure with OpenAI API credentials.  
   - Add a **LangChain LM Chat OpenAI** node "OpenAI Chat Model" and connect its output back to "Idea Generator" as AI language model.  
   - Add a **LangChain Output Parser Structured** node "Structured Output Parser" between "OpenAI Chat Model" and "Idea Generator" to parse AI outputs.

5. **Generate Audio and Image Prompts:**  
   - Add **LangChain OpenAI** node "Audio Prompt & Title" connected from "Idea Generator". Configure to generate podcast script and title.  
   - Add **LangChain OpenAI** node "Image Prompt" connected from "Idea Generator". Configure to generate image description prompt.

6. **Text to Speech Conversion:**  
   - Add an **HTTP Request** node "Text to Speech" connected from "Audio Prompt & Title". Configure TTS API (e.g., ElevenLabs) with credentials and parameters to convert text to audio.

7. **Create Audio Asset on Hedra:**  
   - Add an **HTTP Request** node "Create Audio Asset (Hedra)" connected from "Text to Speech". Configure with Hedra API credentials to create an audio asset.

8. **Merge Audio Assets:**  
   - Add a **Merge** node "Merge Audio Assets" to merge results from "Text to Speech" and "Create Audio Asset (Hedra)". Connect appropriately.

9. **Split Audio Metadata:**  
   - Add a **Code** node "Split Audio Metadata" connected from "Merge Audio Assets" to extract audio asset IDs and metadata.

10. **Upload Audio Asset:**  
    - Add an **HTTP Request** node "Upload Audio (Hedra)" connected from "Split Audio Metadata". Configure to upload audio asset to Hedra.

11. **Create Image Asset:**  
    - Add **HTTP Request** node "Create The Image" connected from "Image Prompt". Configure Hedra API image generation.  
    - Add **Convert to File** node connected from "Create The Image" to convert raw image data to file format.  
    - Add **HTTP Request** node "Create Image Asset (Hedra)" connected from "Convert to File". Configure Hedra API for image asset creation.

12. **Merge and Split Image Metadata:**  
    - Add **Merge** node "Merge 2" to merge image asset data streams from "Convert to File" and "Create Image Asset (Hedra)".  
    - Add **Code** node "Split Image Metadata" connected from "Merge 2" to extract image asset IDs.  
    - Add **HTTP Request** node "Upload Image (Hedra)" connected from "Split Image Metadata" to upload image asset.

13. **Merge Audio and Image Assets:**  
    - Add **Merge** node "Merge Assets" to combine outputs from "Upload Audio (Hedra)" and "Upload Image (Hedra)".

14. **Split Asset IDs for Video:**  
    - Add **Code** node "Split Image & Audio Assets IDs" connected from "Merge Assets" to extract all asset IDs needed for video creation.

15. **Create Video Podcast:**  
    - Add **HTTP Request** node "Create Video (Hedra)" connected from "Split Image & Audio Assets IDs". Configure Hedra API to create video from assets.

16. **Wait for Video Processing:**  
    - Add **Wait** node "Wait 5 Mins" connected from "Create Video (Hedra)".  
    - Add **HTTP Request** node "Get BabyPod Video (Hedra)" connected from "Wait 5 Mins" to check video status.  
    - Add **Wait** node "Wait 10 Secs" connected from "Get BabyPod Video (Hedra)".  

17. **Download Final Video:**  
    - Add **HTTP Request** node "Download BabyPod Video (Hedra)" connected from "Wait 10 Secs" to download the video file.

18. **Upload Video to Google Drive:**  
    - Add **Google Drive** node "Upload Video" connected from "Download BabyPod Video (Hedra)". Configure with Google Drive OAuth2 credentials and target folder.

19. **Optionally Download Video from Google Drive:**  
    - Add **Google Drive** node "Download Video (YouTube)" connected from "Upload Video" if further processing or verification is needed.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                        |
|-----------------------------------------------------------------------------------------------|-------------------------------------|
| The workflow uses Hedra APIs for media asset management: audio, image, and video creation.     | Hedra API documentation              |
| OpenAI API is used via LangChain nodes for generating ideas, prompts, and scripts.             | https://openai.com/api/              |
| ElevenLabs or other TTS providers can be used in the "Text to Speech" HTTP Request node.       | ElevenLabs TTS API documentation     |
| Google Drive nodes require OAuth2 credentials configured in n8n for file uploads/downloads.    | Google Drive API OAuth2 setup        |
| The workflow includes wait nodes to allow asynchronous processing on Hedra platform.           | Hedra processing time considerations |
| Error handling should be implemented for API failures and timeouts on HTTP Request nodes.      | n8n best practices                   |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow developed with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.