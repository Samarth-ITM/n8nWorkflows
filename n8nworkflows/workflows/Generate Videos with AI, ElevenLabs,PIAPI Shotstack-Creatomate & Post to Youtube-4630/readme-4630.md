Generate Videos with AI, ElevenLabs,PIAPI Shotstack/Creatomate & Post to Youtube

https://n8nworkflows.xyz/workflows/generate-videos-with-ai--elevenlabs-piapi-shotstack-creatomate---post-to-youtube-4630


# Generate Videos with AI, ElevenLabs,PIAPI Shotstack/Creatomate & Post to Youtube

### 1. Workflow Overview

This n8n workflow automates the entire process of generating AI-driven videos and posting them to YouTube. It integrates AI content generation, multimedia asset creation, video rendering using Shotstack and Creatomate platforms, and final publishing on YouTube.  
The workflow is designed for content creators, marketers, and media producers who want to automate video production from ideation to publishing with AI assistance and cloud services.

The workflow is logically segmented into the following blocks:

- **1.1 Scheduling and Idea Generation**: Periodically triggers the workflow, generates video ideas using AI, and stores them in Airtable and Google Sheets.
- **1.2 Script Generation and Processing**: Uses AI models to generate scripts for the ideas, parses and stores them, updates status.
- **1.3 Scene Extraction and Asset Creation**: Extracts scenes from scripts, generates images (via Text-to-Image), waits for asset readiness, and stores image URLs.
- **1.4 Voice/Narration Generation**: Extracts narration text, generates voice audio via ElevenLabs or similar, stores audio assets.
- **1.5 Music Generation**: Creates AI-generated music prompts, converts them to music tracks, stores music URLs.
- **1.6 Video Assembly and Rendering**: Prepares video asset timelines, invokes Shotstack or Creatomate APIs for rendering, polls for completion, downloads videos.
- **1.7 Video Storage and Publishing**: Stores final videos in Google Drive, manages sharing permissions, posts videos to YouTube, and sends notification emails.
- **1.8 Error Handling and Workflow Controls**: Includes wait nodes, conditional checks, retries, and fallback email notifications for failed video creation.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Idea Generation

**Overview:**  
Starts the workflow on a schedule, generates new video ideas using AI agents, and stores them in Airtable and Google Sheets for record keeping.

**Nodes Involved:**  
- Schedule Trigger  
- Generate Idea  
- Parse Ideas  
- Add Ideas  
- Store in Airtable  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow on a defined schedule (cron or interval)  
  - Config: Default schedule parameters (not explicitly set)  
  - Input: None (trigger)  
  - Output: Triggers "Generate Idea" node  
  - Edge cases: Scheduling misconfiguration; missed triggers if service downtime occurs.

- **Generate Idea**  
  - Type: Langchain Agent  
  - Role: Uses AI to generate video ideas based on input or prompt  
  - Config: Uses OpenAI Chat model as backend (via linked AI model node)  
  - Input: Trigger data from Schedule Trigger  
  - Output: Raw AI-generated ideas  
  - Failure types: API key issues, rate limits, malformed prompt.

- **Parse Ideas**  
  - Type: Code  
  - Role: Parses AI-generated ideas into structured format for storage  
  - Config: Custom JavaScript code (likely parsing JSON or text)  
  - Input: Output from Generate Idea  
  - Output: Structured idea objects  
  - Edge cases: Parsing errors if AI output format changes or is unexpected.

- **Add Ideas**  
  - Type: Google Sheets  
  - Role: Stores parsed ideas into a Google Sheets document for logging  
  - Config: Append mode to a specific sheet  
  - Input: Parsed ideas  
  - Output: Data confirmation  
  - Edge cases: API quota limits, credentials expiration.

- **Store in Airtable**  
  - Type: Airtable  
  - Role: Adds the new ideas to an Airtable base/table for further workflow processing  
  - Config: Table and base IDs configured  
  - Input: Parsed ideas  
  - Output: Airtable record creation confirmation  
  - Edge cases: Airtable API rate limits, credential errors.

---

#### 1.2 Script Generation and Processing

**Overview:**  
Generates detailed video scripts from ideas using AI, parses the output, stores scripts, and updates idea statuses in data tables.

**Nodes Involved:**  
- Generate Script  
- OpenAI Chat Model1  
- Structured Output Parser  
- Parse Script Output  
- Store Script  
- Store Script in Airtable  
- Updated Idea to Scripted  
- Update Status Ideas Table  

**Node Details:**  

- **Generate Script**  
  - Type: Langchain Agent  
  - Role: AI-powered script generation from video ideas  
  - Config: Connected to OpenAI Chat Model1 and Structured Output Parser nodes for language model input and output parsing  
  - Input: Data from "Store in Airtable" (new ideas)  
  - Output: Script text and structured data  
  - Failure: API errors, output parsing errors.

- **OpenAI Chat Model1**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides backend AI model for "Generate Script"  
  - Config: Model parameters (temperature, model name)  
  - Input: Prompt from "Generate Script" node  
  - Output: Text completions  
  - Edge cases: Quota, rate limits, invalid credentials.

- **Structured Output Parser**  
  - Type: Langchain Output Parser  
  - Role: Parses AI output into structured format (JSON) for ease of use  
  - Config: Parser schema defined to extract script details  
  - Input: Raw AI text output  
  - Output: Structured script data  
  - Failure: Parsing errors if AI output deviates from schema.

- **Parse Script Output**  
  - Type: Code  
  - Role: Processes structured script data for downstream storage and usage  
  - Config: Custom JavaScript code  
  - Input: Output from Generate Script  
  - Output: Parsed script ready for storage  

- **Store Script**  
  - Type: Google Sheets  
  - Role: Logs the script text into Google Sheets document  
  - Config: Append mode, specific sheet and columns  
  - Input: Parsed script output  
  - Output: Confirmation data  

- **Store Script in Airtable**  
  - Type: Airtable  
  - Role: Stores script data in Airtable for record and status tracking  
  - Input: Parsed script data  
  - Output: Airtable record confirmation  

- **Updated Idea to Scripted**  
  - Type: Google Sheets  
  - Role: Updates the idea status to "Scripted" in Google Sheets  
  - Input: Script completion trigger  
  - Output: Confirmation  

- **Update Status Ideas Table**  
  - Type: Airtable  
  - Role: Updates idea status to indicate script generation complete  
  - Input: Confirmation from script storage  
  - Output: Updated Airtable record  

---

#### 1.3 Scene Extraction and Asset Creation

**Overview:**  
Extracts individual scenes from scripts, generates images for scenes, waits for asset generation, and stores image URLs.

**Nodes Involved:**  
- Extract Scenes  
- Generate Music Prompt  
- Text-to-Image  
- Wait for 4 Min  
- Get Images  
- Get ImageUrls  
- Scene Image Urls  
- Update ImageUrls  
- Store Image Urls in Airtable  

**Node Details:**  

- **Extract Scenes**  
  - Type: Code  
  - Role: Parses script text into discrete scenes for video segments  
  - Config: Custom JavaScript logic to split script  
  - Input: Updated status from previous block  
  - Output: Scene list array  

- **Generate Music Prompt**  
  - Type: Langchain Agent  
  - Role: Generates prompts for AI music generation based on scenes  
  - Input: Scene data  
  - Output: Music prompt text  

- **Text-to-Image**  
  - Type: HTTP Request  
  - Role: Calls external Text-to-Image API to generate images for scenes  
  - Config: API endpoint, authentication, dynamic scene text input  
  - Input: Scene descriptions  
  - Output: Image generation job IDs or URLs  

- **Wait for 4 Min**  
  - Type: Wait  
  - Role: Waits for image generation to complete  
  - Config: Fixed 4-minute delay  
  - Input: Trigger after Text-to-Image request  

- **Get Images**  
  - Type: HTTP Request  
  - Role: Polls or retrieves generated images from the image generation service  
  - Input: Job IDs or references from previous node  
  - Output: Image URLs  

- **Get ImageUrls**  
  - Type: Code  
  - Role: Extracts and formats image URLs from API response  
  - Input: Raw API response  
  - Output: Clean list of image URLs  

- **Scene Image Urls**  
  - Type: Code  
  - Role: Associates scene data with corresponding image URLs  
  - Output: Merged scene-image data  

- **Update ImageUrls**  
  - Type: Google Sheets  
  - Role: Updates Google Sheets with image URLs for scenes  
  - Input: Scene-image data  
  - Output: Confirmation  

- **Store Image Urls in Airtable**  
  - Type: Airtable  
  - Role: Stores image URL data in Airtable for retrieval and reference  
  - Input: Scene-image data  
  - Output: Airtable record confirmation  

---

#### 1.4 Voice/Narration Generation

**Overview:**  
Extracts narration text from scenes, generates voice audio files using ElevenLabs or similar TTS service, waits for processing, and stores voice URLs.

**Nodes Involved:**  
- Extract Narration  
- Voice Generation  
- Wait 2 Min  
- Store Sound  
- Allow Access  
- Aggregating Voice Urls  
- Update VoiceUrls  
- Store Voice Urls  

**Node Details:**  

- **Extract Narration**  
  - Type: Code  
  - Role: Extracts narration lines from scene data  
  - Input: Scene-image data  
  - Output: Narration scripts  

- **Voice Generation**  
  - Type: HTTP Request  
  - Role: Sends narration text to voice generation API (e.g., ElevenLabs)  
  - Config: API endpoint, auth, voice parameters  
  - Output: Voice generation job ID or audio URL  

- **Wait 2 Min**  
  - Type: Wait  
  - Role: Delay to allow voice generation to complete  
  - Input: After voice generation request  

- **Store Sound**  
  - Type: Google Drive  
  - Role: Uploads generated audio files to Google Drive  
  - Config: Drive folder set, permissions controlled  
  - Output: Drive file metadata  

- **Allow Access**  
  - Type: Google Drive  
  - Role: Sets sharing permissions for audio files (anyone with link)  
  - Output: Sharing link confirmation  

- **Aggregating Voice Urls**  
  - Type: Code  
  - Role: Aggregates all voice URLs into a list for later processing  
  - Output: Aggregated voice URL list  

- **Update VoiceUrls**  
  - Type: Google Sheets  
  - Role: Updates Google Sheets with new voice URLs  
  - Input: Aggregated list  
  - Output: Confirmation  

- **Store Voice Urls**  
  - Type: Airtable  
  - Role: Stores voice URL data in Airtable for reference  
  - Input: Aggregated list  
  - Output: Airtable record confirmation  

---

#### 1.5 Music Generation

**Overview:**  
Generates music prompts with AI, converts prompts into music files, stores music in Google Drive, and saves URLs.

**Nodes Involved:**  
- Parse Music Prompts  
- OpenAI Chat Model2  
- Generate Music Prompt  
- MusicPrompt (Set Node)  
- Limit  
- Text toMusic  
- 2 Min Wait  
- Music Store  
- Access to Anyone with Link  
- Add Aggregate Music URLs  
- Store Music Urls  
- Music Urls  

**Node Details:**  

- **Parse Music Prompts**  
  - Type: Code  
  - Role: Parses AI-generated music prompts for API consumption  
  - Input: Raw AI music prompt text  
  - Output: Structured prompt data  

- **OpenAI Chat Model2**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides AI generation for music prompts  
  - Output: Music prompt text  

- **Generate Music Prompt**  
  - Type: Langchain Agent  
  - Role: Generates detailed music prompt based on scenes or scripts  

- **MusicPrompt**  
  - Type: Set  
  - Role: Stores or formats the music prompt for downstream nodes  

- **Limit**  
  - Type: Limit  
  - Role: Controls concurrency or rate of music generation requests  

- **Text toMusic**  
  - Type: HTTP Request  
  - Role: Sends music prompts to an AI music generation API  

- **2 Min Wait**  
  - Type: Wait  
  - Role: Waits for music generation to finish  

- **Music Store**  
  - Type: Google Drive  
  - Role: Stores generated music files  

- **Access to Anyone with Link**  
  - Type: Google Drive  
  - Role: Sets sharing permissions on stored music files  

- **Add Aggregate Music URLs**  
  - Type: Code  
  - Role: Aggregates music URLs into a list  

- **Store Music Urls**  
  - Type: Airtable  
  - Role: Stores music URLs in Airtable  

- **Music Urls**  
  - Type: Google Sheets  
  - Role: Maintains music URLs in Google Sheets for tracking  

---

#### 1.6 Video Assembly and Rendering

**Overview:**  
Constructs video timelines, submits rendering jobs to Shotstack or Creatomate, waits for completion, downloads finished videos.

**Nodes Involved:**  
- Prepare Video Assets  
- Build Shotstack Timeline  
- ShotStack Render Video  
- Wait 5 Min  
- Poll Rendered Videos  
- Final Video  
- Final Video Update  
- Download video  
- Store video  
- Youtube Video Created  
- Search for Latest Ready Video  
- If Ready?  
- Download Video  
- Post YouTube  
- Video Creatomate  
- Prepare  Assets For Creatomate (disabled)  
- List Elements  
- Wait for Render  
- Get Final Video  
- Check Video Status  
- Download Binary  
- Final Creatomate Video  
- Final Video Creatomate  
- Creatomate Final Video  

**Node Details:**  

- **Prepare Video Assets**  
  - Type: Code  
  - Role: Organizes all media assets (images, audio, music) into a timeline structure for rendering  
  - Output: Timeline JSON  

- **Build Shotstack Timeline**  
  - Type: Code  
  - Role: Converts assets into Shotstack-specific timeline JSON  

- **ShotStack Render Video**  
  - Type: HTTP Request  
  - Role: Sends video timeline to Shotstack API to start rendering job  

- **Wait 5 Min**  
  - Type: Wait  
  - Role: Initial delay post-submission to allow processing  

- **Poll Rendered Videos**  
  - Type: HTTP Request  
  - Role: Polls Shotstack API for rendering job status  

- **Final Video**  
  - Type: Airtable  
  - Role: Updates Airtable with final video job status and metadata  

- **Final Video Update**  
  - Type: Google Sheets  
  - Role: Logs final video metadata in Google Sheets  

- **Download video**  
  - Type: HTTP Request  
  - Role: Downloads finished video from Shotstack or Creatomate URL  

- **Store video**  
  - Type: Google Drive  
  - Role: Uploads final video file to Google Drive  

- **Youtube Video Created**  
  - Type: Gmail  
  - Role: Sends notification email about successful YouTube video upload  

- **Search for Latest Ready Video**  
  - Type: Airtable  
  - Role: Queries Airtable for videos ready to be downloaded and posted  

- **If Ready?**  
  - Type: If  
  - Role: Checks if video is ready, branches workflow accordingly  

- **Download Video**  
  - Type: HTTP Request  
  - Role: Downloads video for posting  

- **Post YouTube**  
  - Type: YouTube  
  - Role: Uploads video to YouTube channel  

- **Video Creatomate**  
  - Type: HTTP Request  
  - Role: Renders video via Creatomate platform alternative to Shotstack  

- **Prepare  Assets For Creatomate** *(disabled)*  
  - Type: Code  
  - Role: (Disabled) Prepares assets for Creatomate rendering  

- **List Elements**  
  - Type: Code  
  - Role: Lists and formats elements for Creatomate API call  

- **Wait for Render**  
  - Type: Wait  
  - Role: Waits for Creatomate rendering completion  

- **Get Final Video**  
  - Type: HTTP Request  
  - Role: Retrieves final video from Creatomate  

- **Check Video Status**  
  - Type: Switch  
  - Role: Branches based on video render status - success, failure, or pending  

- **Download Binary**  
  - Type: HTTP Request  
  - Role: Downloads video binary data  

- **Final Creatomate Video**  
  - Type: Google Drive  
  - Role: Stores Creatomate rendered video  

- **Final Video Creatomate**  
  - Type: Airtable  
  - Role: Stores metadata for Creatomate final video  

- **Creatomate Final Video**  
  - Type: Google Sheets  
  - Role: Logs Creatomate video info  

---

#### 1.7 Video Storage and Publishing

**Overview:**  
Manages final video storage, sharing, and publishes the video on YouTube, including sending success or failure notification emails.

**Nodes Involved:**  
- Allow Access  
- Youtube Video Created  
- Failed Creation  

**Node Details:**  

- **Allow Access**  
  - Type: Google Drive  
  - Role: Sets sharing permissions on videos for public access  

- **Youtube Video Created**  
  - Type: Gmail  
  - Role: Sends email notification confirming successful YouTube posting  

- **Failed Creation**  
  - Type: Gmail  
  - Role: Sends alert email if video creation or posting fails  

---

#### 1.8 Error Handling and Workflow Controls

**Overview:**  
Includes wait nodes for timing controls, limit nodes for concurrency control, conditional checks, and sticky notes for documentation.

**Nodes Involved:**  
- Wait for 4 Min  
- Wait 2 Min  
- Wait 5 Min  
- 2 Min Wait  
- Limit  
- If Ready?  
- Sticky Notes (multiple)  

**Node Details:**  

- **Wait Nodes**  
  - Type: Wait  
  - Role: Inserted delays to ensure external processes complete (image generation, voice generation, video rendering)  
  - Configured with fixed wait durations.

- **Limit**  
  - Type: Limit  
  - Role: Controls concurrency of requests to APIs, avoiding rate limits or overloading services  

- **If Ready?**  
  - Type: If  
  - Role: Branches logic based on readiness of video for download or further processing  

- **Sticky Notes**  
  - Type: Sticky Note  
  - Role: Provide documentation and comments within the workflow for user clarity  

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                          | Input Node(s)                   | Output Node(s)                  | Sticky Note                         |
|-----------------------------|---------------------------------|----------------------------------------|--------------------------------|---------------------------------|-----------------------------------|
| Schedule Trigger             | Schedule Trigger                | Start workflow on schedule              |                                | Generate Idea                   |                                   |
| Generate Idea               | Langchain Agent                 | Generate video ideas using AI           | Schedule Trigger               | Parse Ideas                    |                                   |
| Parse Ideas                 | Code                           | Parse AI ideas into structured format   | Generate Idea                 | Add Ideas                     |                                   |
| Add Ideas                   | Google Sheets                  | Store parsed ideas in Google Sheets     | Parse Ideas                   | Store in Airtable              |                                   |
| Store in Airtable           | Airtable                       | Store ideas in Airtable                  | Add Ideas                     | Generate Script               |                                   |
| Generate Script             | Langchain Agent                 | Generate video scripts from ideas       | Store in Airtable             | Parse Script Output           |                                   |
| OpenAI Chat Model1          | Langchain OpenAI Chat Model    | AI model backend for script generation  | Generate Script               | Generate Script               |                                   |
| Structured Output Parser    | Langchain Output Parser        | Parse AI script output into JSON        | Generate Script               | Generate Script               |                                   |
| Parse Script Output         | Code                           | Further parse script data                | Generate Script               | Store Script                 |                                   |
| Store Script               | Google Sheets                  | Store script text                        | Parse Script Output           | Store Script in Airtable      |                                   |
| Store Script in Airtable    | Airtable                       | Store script in Airtable                 | Store Script                 | Updated Idea to Scripted      |                                   |
| Updated Idea to Scripted    | Google Sheets                  | Update idea status to scripted           | Store Script in Airtable      | Update Status Ideas Table     |                                   |
| Update Status Ideas Table   | Airtable                       | Update idea status in Airtable           | Updated Idea to Scripted      | Extract Scenes               |                                   |
| Extract Scenes             | Code                           | Extract scenes from script               | Update Status Ideas Table     | Generate Music Prompt, Text-to-Image |                                   |
| Generate Music Prompt       | Langchain Agent                 | Generate music prompt based on scenes   | Extract Scenes               | MusicPrompt                  |                                   |
| MusicPrompt                | Set                            | Format music prompt                      | Generate Music Prompt         | Limit                       |                                   |
| Limit                      | Limit                          | Control concurrency                      | MusicPrompt                  | Parse Music Prompts          |                                   |
| Parse Music Prompts        | Code                           | Parse music prompts                      | Limit                       | Text toMusic                |                                   |
| Text toMusic               | HTTP Request                   | Generate music from prompt               | Parse Music Prompts           | 2 Min Wait                  |                                   |
| 2 Min Wait                 | Wait                           | Wait for music generation                | Text toMusic                 | Music Store                 |                                   |
| Music Store                | Google Drive                   | Store generated music                    | 2 Min Wait                   | Access to Anyone with Link, Add Aggregate Music URLs |                                   |
| Access to Anyone with Link  | Google Drive                   | Make music publicly accessible           | Music Store                  |                               |                                   |
| Add Aggregate Music URLs   | Code                           | Aggregate music URLs                     | Music Store                  | Store Music Urls, Merge       |                                   |
| Store Music Urls            | Airtable                       | Store music URLs                         | Add Aggregate Music URLs      | Merge                       |                                   |
| Merge                      | Merge                          | Combine image, voice, and music URLs     | Scene Image Urls, Aggregating Voice Urls, Add Aggregate Music URLs | Prepare Video Assets  |                                   |
| Prepare Video Assets        | Code                           | Organize assets into video timeline      | Merge                       | Build Shotstack Timeline     |                                   |
| Build Shotstack Timeline    | Code                           | Prepare Shotstack timeline JSON          | Prepare Video Assets         | ShotStack Render Video       |                                   |
| ShotStack Render Video      | HTTP Request                   | Submit video rendering job to Shotstack  | Build Shotstack Timeline     | Wait 5 Min                  |                                   |
| Wait 5 Min                 | Wait                           | Delay to allow rendering                  | ShotStack Render Video       | Poll Rendered Videos         |                                   |
| Poll Rendered Videos       | HTTP Request                   | Check rendering status                    | Wait 5 Min                  | Final Video                 |                                   |
| Final Video                | Airtable                       | Update Airtable with video status         | Poll Rendered Videos         | Final Video Update           |                                   |
| Final Video Update          | Google Sheets                  | Log video metadata                        | Final Video                 | Download video              |                                   |
| Download video             | HTTP Request                   | Download finished video                   | Final Video Update           | Post YouTube               |                                   |
| Post YouTube               | YouTube                        | Upload video to YouTube                   | Download video              |                               |                                   |
| Search for Latest Ready Video | Airtable                   | Find videos ready for posting             | Schedule Trigger1            | If Ready?                  |                                   |
| If Ready?                  | If                             | Checks if video is ready                  | Search for Latest Ready Video | Download Video, Update Pepduction Table |                                   |
| Update Pepduction Table    | Google Sheets                  | Update production status                   | If Ready?                   |                               |                                   |
| Schedule Trigger1           | Schedule Trigger               | Secondary scheduled trigger               |                             | Search for Latest Ready Video |                                   |
| Extract Narration          | Code                           | Extract narration text from scenes        | Update ImageUrls             | Voice Generation            |                                   |
| Voice Generation           | HTTP Request                   | Generate voice audio from narration       | Extract Narration           | Wait 2 Min                 |                                   |
| Wait 2 Min                 | Wait                           | Wait for voice generation                 | Voice Generation            | Store Sound                |                                   |
| Store Sound                | Google Drive                   | Store audio files                          | Wait 2 Min                  | Allow Access, Aggregating Voice Urls |                                   |
| Allow Access               | Google Drive                   | Set sharing permissions for audio         | Store Sound                 |                             |                                   |
| Aggregating Voice Urls     | Code                           | Aggregate voice URLs                       | Store Sound                 | Update VoiceUrls, Merge      |                                   |
| Update VoiceUrls           | Google Sheets                  | Update voice URLs                          | Aggregating Voice Urls       | Store Voice Urls            |                                   |
| Store Voice Urls           | Airtable                       | Store voice URLs                           | Update VoiceUrls             |                             |                                   |
| Scene Image Urls           | Code                           | Combine scene data with image URLs         | Get ImageUrls               | Update ImageUrls, Merge      |                                   |
| Get ImageUrls              | Code                           | Extract image URLs from API response       | Get Images                  | Scene Image Urls            |                                   |
| Get Images                 | HTTP Request                   | Retrieve generated images                  | Wait for 4 Min              | Get ImageUrls               |                                   |
| Wait for 4 Min             | Wait                           | Wait for image generation                   | Text-to-Image               | Get Images                 |                                   |
| Text-to-Image             | HTTP Request                   | Request images based on scene descriptions | Generate Music Prompt, Extract Scenes | Wait for 4 Min             |                                   |
| Download binary            | HTTP Request                   | Download video binary data                  | Check Video Status          | Final Creatomate Video     |                                   |
| Final Creatomate Video     | Google Drive                   | Store final Creatomate video                | Download Binary             | Youtube Video Created      |                                   |
| Youtube Video Created      | Gmail                          | Notify successful YouTube video creation   | Final Creatomate Video      | Final Video Creatomate     |                                   |
| Failed Creation            | Gmail                          | Notify failure in video creation            | Check Video Status          |                             |                                   |
| Video Creatomate           | HTTP Request                   | Submit video rendering job to Creatomate   | List Elements               | Wait for Render           |                                   |
| Wait for Render            | Wait                           | Wait for Creatomate rendering               | Video Creatomate            | Get Final Video            |                                   |
| Get Final Video            | HTTP Request                   | Retrieve final video from Creatomate        | Wait for Render             | Check Video Status         |                                   |
| List Elements             | Code                           | Build list of elements for Creatomate API   |                             | Video Creatomate           |                                   |
| Final Video Creatomate    | Airtable                       | Store metadata for final Creatomate video   | Youtube Video Created       | Creatomate Final Video     |                                   |
| Creatomate Final Video     | Google Sheets                  | Log final Creatomate video metadata          | Final Video Creatomate      |                             |                                   |
| Update Pepduction Table    | Google Sheets                  | Update production status in sheets          | If Ready?                  |                             |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure the cron or interval schedule to trigger video generation workflow.

2. **Add Langchain Agent node ("Generate Idea")**  
   - Connect output from Schedule Trigger.  
   - Configure to use OpenAI API credentials and set prompt to generate video ideas.

3. **Add Code node ("Parse Ideas")**  
   - Connect from "Generate Idea".  
   - Write JavaScript to parse AI-generated text into structured idea objects.

4. **Add Google Sheets node ("Add Ideas")**  
   - Connect from "Parse Ideas".  
   - Configure with Google Sheets credentials, select target sheet and append mode.

5. **Add Airtable node ("Store in Airtable")**  
   - Connect from "Add Ideas".  
   - Configure Airtable API credentials, base ID, and table for storing ideas.

6. **Add Langchain Agent node ("Generate Script")**  
   - Connect from "Store in Airtable".  
   - Set up with OpenAI Chat Model1 and Structured Output Parser nodes.  
   - Configure prompt template for script generation.

7. **Add Langchain OpenAI Chat Model node ("OpenAI Chat Model1")**  
   - Provide OpenAI API key and model configuration.

8. **Add Langchain Output Parser node ("Structured Output Parser")**  
   - Define JSON schema to parse script content.

9. **Add Code node ("Parse Script Output")**  
   - Connect from "Generate Script".  
   - Implement parsing logic for structured script data.

10. **Add Google Sheets node ("Store Script")**  
    - Append script content to a designated sheet.

11. **Add Airtable node ("Store Script in Airtable")**  
    - Store parsed script data.

12. **Add Google Sheets node ("Updated Idea to Scripted")**  
    - Update idea status.

13. **Add Airtable node ("Update Status Ideas Table")**  
    - Reflect updated status.

14. **Add Code node ("Extract Scenes")**  
    - Parse the script into scenes.

15. **Add Langchain Agent node ("Generate Music Prompt")**  
    - Generate music prompt based on scenes.

16. **Add Set node ("MusicPrompt")**  
    - Store formatted music prompt.

17. **Add Limit node**  
    - Control concurrency of music generation requests.

18. **Add Code node ("Parse Music Prompts")**  
    - Parse music prompts from AI output.

19. **Add HTTP Request node ("Text toMusic")**  
    - Connect to AI music generation API. Configure endpoint and auth.

20. **Add Wait node ("2 Min Wait")**  
    - Wait 2 minutes for music generation.

21. **Add Google Drive node ("Music Store")**  
    - Upload generated music files.

22. **Add Google Drive node ("Access to Anyone with Link")**  
    - Set sharing permissions.

23. **Add Code node ("Add Aggregate Music URLs")**  
    - Aggregate URLs for storage.

24. **Add Airtable node ("Store Music Urls")**  
    - Store music URLs.

25. **Add Google Sheets node ("Music Urls")**  
    - Log music URLs.

26. **Add HTTP Request node ("Text-to-Image")**  
    - Use Text-to-Image API to generate images from scenes.

27. **Add Wait node ("Wait for 4 Min")**  
    - Wait 4 minutes for image generation.

28. **Add HTTP Request node ("Get Images")**  
    - Retrieve generated images.

29. **Add Code node ("Get ImageUrls")**  
    - Extract image URLs from response.

30. **Add Code node ("Scene Image Urls")**  
    - Combine scenes with image URLs.

31. **Add Google Sheets node ("Update ImageUrls")**  
    - Update Google Sheets with image URLs.

32. **Add Airtable node ("Store Image Urls in Airtable")**  
    - Store image URLs.

33. **Add Code node ("Extract Narration")**  
    - Extract narration from scenes.

34. **Add HTTP Request node ("Voice Generation")**  
    - Send narration text to voice synthesis API.

35. **Add Wait node ("Wait 2 Min")**  
    - Wait for voice generation.

36. **Add Google Drive node ("Store Sound")**  
    - Upload voice audio files.

37. **Add Google Drive node ("Allow Access")**  
    - Set sharing permissions.

38. **Add Code node ("Aggregating Voice Urls")**  
    - Aggregate voice URLs.

39. **Add Google Sheets node ("Update VoiceUrls")**  
    - Update voice URLs in sheet.

40. **Add Airtable node ("Store Voice Urls")**  
    - Store voice URLs.

41. **Add Code node ("Prepare Video Assets")**  
    - Prepare timeline JSON for video rendering.

42. **Add Code node ("Build Shotstack Timeline")**  
    - Build Shotstack timeline JSON.

43. **Add HTTP Request node ("ShotStack Render Video")**  
    - Submit rendering job to Shotstack API.

44. **Add Wait node ("Wait 5 Min")**  
    - Delay for rendering.

45. **Add HTTP Request node ("Poll Rendered Videos")**  
    - Poll rendering status.

46. **Add Airtable node ("Final Video")**  
    - Update final video status.

47. **Add Google Sheets node ("Final Video Update")**  
    - Log final video info.

48. **Add HTTP Request node ("Download video")**  
    - Download finished video.

49. **Add Google Drive node ("Store video")**  
    - Upload video to Drive.

50. **Add YouTube node ("Post YouTube")**  
    - Upload video to YouTube channel. Configure OAuth2 credentials.

51. **Add Airtable node ("Search for Latest Ready Video")**  
    - Query Airtable for ready videos.

52. **Add If node ("If Ready?")**  
    - Branch on readiness.

53. **Add Google Sheets node ("Update Pepduction Table")**  
    - Update production status.

54. **Add Gmail nodes ("Youtube Video Created", "Failed Creation")**  
    - Send notification emails on success/failure.

55. **Optional: Add Creatomate rendering nodes**  
    - Prepare assets, submit render job, poll status, download, store, and notify as with Shotstack, configured similarly.

56. **Add Wait nodes and Limit nodes as needed**  
    - Control process timing and concurrency.

57. **Add Sticky Notes for documentation inside workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                         |
|-------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| The workflow integrates AI services (OpenAI for text, ElevenLabs for voice), cloud storage (Google Drive), Airtable, Google Sheets. | Project ecosystem overview             |
| Video rendering is performed via Shotstack and Creatomate APIs as alternatives.                                                     | Shotstack: https://shotstack.io/      |
| YouTube node requires OAuth2 credentials configured in n8n for channel access.                                                     | YouTube API docs                      |
| Includes error notification emails to alert failures in video creation or publishing steps.                                        | Gmail node usage                      |
| Workflow uses wait nodes extensively to handle asynchronous external API processing delays.                                        | Timing and rate-limit management      |
| Sticky notes are used throughout the workflow for inline documentation and instructions.                                            | Internal workflow documentation       |
| For best performance, ensure API rate limits and quotas for OpenAI, Airtable, Google Sheets, Google Drive, Shotstack, and Creatomate are respected. | API management best practices         |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created using n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.