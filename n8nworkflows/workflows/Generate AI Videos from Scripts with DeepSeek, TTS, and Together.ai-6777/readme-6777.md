Generate AI Videos from Scripts with DeepSeek, TTS, and Together.ai

https://n8nworkflows.xyz/workflows/generate-ai-videos-from-scripts-with-deepseek--tts--and-together-ai-6777


# Generate AI Videos from Scripts with DeepSeek, TTS, and Together.ai

### 1. Workflow Overview

This workflow automates the generation of AI-powered videos based on user-submitted scripts or transcripts. It converts textual ideas into engaging video content by producing a video script, generating text-to-speech (TTS) audio, creating segmented image prompts, rendering video clips from those images, and finally combining clips with captions, audio, and optional music. The workflow is designed for content creators seeking automated video production using AI models and integrates services such as DeepSeek, Together.ai, Google Sheets, Google Drive, and Telegram for notifications.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Initialization**: Collects user input through a form, sets initial parameters, and branches logic based on video type.
- **1.2 Script Generation & Parsing**: Creates or derives the video script either from user idea or transcript using AI language models and structured output parsers.
- **1.3 Text-to-Speech (TTS) Generation**: Generates voice-over audio files and stores them on Google Drive.
- **1.4 Script Segmentation & Scene Creation**: Splits scripts into timed segments, fixes segment durations, and creates scene descriptions.
- **1.5 Image Prompt Generation & Image Creation**: Uses AI to generate image prompts for scenes and calls Together.ai API to produce images, storing them to Google Drive.
- **1.6 Video Clip Creation & Assembly**: Converts images into video clips using a local HTTP service, aggregates clip URLs, and concatenates clips into a full video.
- **1.7 Caption Addition & Final Video Composition**: Adds captions to the video, merges audio and video layers, and optionally adds background music.
- **1.8 Output Storage & Notification**: Updates Google Sheets with video URLs and metadata, downloads final video files, and sends notifications via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

**Overview:**  
Receives user input from a web form, sets key variables, and routes processing depending on the video type selected.

**Nodes Involved:**  
- On form submission  
- Set Idea  
- Switch  

**Node Details:**

- **On form submission**  
  - *Type*: Form Trigger  
  - *Role*: Entry point capturing user inputs (topic, duration, generative style, video type, TTS voice, image provider).  
  - *Configuration*: Defines form fields including text area and dropdowns with predefined options. Ignores bots and disables attribution.  
  - *Connections*: Outputs to Set Idea, Google Sheets3, Google Sheets1, Google Sheets6, Google Sheets8, Google Sheets12, Google Sheets15, Google Sheets17.  
  - *Edge Cases*: Form input validation errors, missing required fields.  
  - *Credentials*: None.  

- **Set Idea**  
  - *Type*: Set Node  
  - *Role*: Assigns variables "User Input" (main topic) and "Script Duration" from form data for downstream use.  
  - *Configuration*: Extracts fields from form JSON.  
  - *Connections*: Outputs to Switch.  
  - *Edge Cases*: Missing or malformed input data.  

- **Switch**  
  - *Type*: Switch Node  
  - *Role*: Directs flow based on "Video Type" field ("From transcript" vs. "From user idea").  
  - *Configuration*: Checks exact string match on Video Type form input.  
  - *Connections*:  
    - "From transcript" to Long form to Script Writier 🧠  
    - "User Idea" to Script Writier 🧠  
  - *Edge Cases*: Unexpected or missing Video Type values could break routing.

---

#### 1.2 Script Generation & Parsing

**Overview:**  
Generates a video script using AI language models based on user input or transcript, then parses the structured output.

**Nodes Involved:**  
- Script Writier 🧠  
- Long form to Script Writier 🧠  
- Output Parser 🛠  
- Structured Output Parser1  
- Combine  
- Format Cleanup  

**Node Details:**

- **Script Writier 🧠**  
  - *Type*: LangChain LLM Chain  
  - *Role*: Generates video script from user idea and desired duration.  
  - *Configuration*: Uses prompt instructing to create an engaging YouTube video script with title, description, hook, main script, and CTA.  
  - *Expressions*: Injects "User Input" and "Script Duration" variables.  
  - *Connections*: Outputs to Output Parser 🛠.  
  - *Credentials*: OpenRouter API with DeepSeek model.  
  - *Edge Cases*: API rate limits, malformed input, model misinterpretation.  

- **Long form to Script Writier 🧠**  
  - *Type*: LangChain LLM Chain  
  - *Role*: Generates video script from longer transcript input.  
  - *Configuration*: Similar prompt but optimized for transcript processing.  
  - *Connections*: Outputs to Output Parser 🛠.  
  - *Credentials*: OpenRouter API with DeepSeek model.  

- **Output Parser 🛠**  
  - *Type*: LangChain LLM Chain with Output Parser  
  - *Role*: Parses AI-generated text into structured JSON with required fields (Title, Description, Hook, MainScript, CTA).  
  - *Connections*: Outputs to Combine.  
  - *Edge Cases*: Parsing failure if AI output format differs.  

- **Combine**  
  - *Type*: Set Node  
  - *Role*: Concatenates Hook, MainScript, and CTA into "Main Script" string for TTS input.  
  - *Connections*: Outputs to Format Cleanup.  

- **Format Cleanup**  
  - *Type*: Set Node  
  - *Role*: Cleans script text by removing extra newlines and asterisks.  
  - *Connections*: Outputs final cleaned script for TTS generation and storage.  

---

#### 1.3 Text-to-Speech (TTS) Generation

**Overview:**  
Generates voice-over audio from the script using TTS API, then uploads the audio file to Google Drive.

**Nodes Involved:**  
- Generate TTS  
- Google Drive  
- Google Sheets1  
- Google Sheets14  

**Node Details:**

- **Generate TTS**  
  - *Type*: HTTP Request  
  - *Role*: Calls TTS service with script text, voice selection, and speed parameters. Returns MP3 audio and timestamps.  
  - *Configuration*: POST to local TTS endpoint with JSON body including voice and speed. Response includes audio and timestamps.  
  - *Connections*: Outputs audio file headers to Google Drive.  
  - *Edge Cases*: Network errors, API timeouts, unsupported voice selections.  

- **Google Drive**  
  - *Type*: Google Drive Upload  
  - *Role*: Uploads generated audio file to Google Drive folder "tts". Uses date header as filename.  
  - *Connections*: Outputs file metadata to Google Sheets1 and Google Sheets14.  
  - *Credentials*: Google Drive OAuth2.  
  - *Edge Cases*: Drive API quota limits, permission errors.  

- **Google Sheets1**  
  - *Type*: Google Sheets Append  
  - *Role*: Stores initial video metadata from the form submission for tracking.  
  - *Credentials*: Google Sheets OAuth2.  

- **Google Sheets14**  
  - *Type*: Google Sheets Update  
  - *Role*: Updates sheet with TTS audio web link for use in further processing.  

---

#### 1.4 Script Segmentation & Scene Creation

**Overview:**  
Splits the script into short timed segments (approx. 5-6 seconds), merges short last segment if needed, and prepares scene data for image generation.

**Nodes Involved:**  
- Get Captions  
- Google Drive1  
- Extract from File  
- Aggregate  
- Split Out  
- Fixer  
- Split into 5s Scenes  
- Google Sheets3  
- Google Drive2  

**Node Details:**

- **Get Captions**  
  - *Type*: HTTP Request  
  - *Role*: Downloads timestamps JSON for generated TTS audio from local service.  
  - *Connections*: Outputs to Google Drive1.  

- **Google Drive1**  
  - *Type*: Google Drive Download  
  - *Role*: Downloads the timestamps JSON file for parsing.  
  - *Connections*: Outputs to Extract from File.  

- **Extract from File**  
  - *Type*: Extract from File  
  - *Role*: Parses downloaded JSON timestamps into usable data.  
  - *Connections*: Outputs to Aggregate.  

- **Aggregate**  
  - *Type*: Aggregate  
  - *Role*: Aggregates all timestamp data into a single "Segments" field.  
  - *Connections*: Outputs to Fixer.  

- **Fixer**  
  - *Type*: Code  
  - *Role*: Merges last segment with the previous if duration under 2 seconds to avoid very short clips.  
  - *Connections*: Outputs to Split Out.  

- **Split into 5s Scenes**  
  - *Type*: Code  
  - *Role*: Splits aggregated words into segments of ~6 seconds each with a small pause buffer.  
  - *Connections*: Outputs scene segments and total duration info.  

- **Google Sheets3**  
  - *Type*: Google Sheets Append  
  - *Role*: Stores segmented script data and total duration info.  
  - *Connections*: Outputs to Google Drive2.  

- **Google Drive2**  
  - *Type*: Google Drive Upload  
  - *Role*: Uploads segmented scene data for archival or further processing.  

---

#### 1.5 Image Prompt Generation & Image Creation

**Overview:**  
Transforms each script segment into an image prompt in the selected generative style, generates images via Together.ai API, and stores images on Google Drive.

**Nodes Involved:**  
- Loop Over Items  
- Image Prompter V2 📷  
- Open Router - Deepseek v3.  
- HTTP - Together.ai  
- Base64 To String  
- Convert String to binary  
- Google Drive3  
- Google Sheets7  
- Wait1  
- Sticky Note11  
- Aggregate3  
- IDs To Array1  
- Google Sheets4  
- Google Sheets6  
- Loop Over Items3  
- Google Sheets9  

**Node Details:**

- **Loop Over Items**  
  - *Type*: Split In Batches  
  - *Role*: Iterates over each scene segment to process individually.  
  - *Connections*: Outputs to Image Prompter V2 📷 and HTTP - Together.ai.  

- **Image Prompter V2 📷**  
  - *Type*: LangChain LLM Chain  
  - *Role*: Generates artistic image prompts based on segment script and generative style.  
  - *Configuration*: Detailed prompt instructing style, composition, lighting, mood, etc.  
  - *Credentials*: OpenRouter API.  

- **HTTP - Together.ai**  
  - *Type*: HTTP Request  
  - *Role*: Calls Together.ai image generation API with prompt, requesting base64-encoded images.  
  - *Connections*: Outputs base64 JSON to Base64 To String.  
  - *Credentials*: Header Auth with Together.ai API key.  
  - *Edge Cases*: API rate limits, slow response times (noted in sticky note).  

- **Base64 To String**  
  - *Type*: Set  
  - *Role*: Extracts base64 image string from API response.  
  - *Connections*: Outputs to Convert String to binary.  

- **Convert String to binary**  
  - *Type*: Move Binary Data  
  - *Role*: Converts base64 string into binary image data with filename image.png.  
  - *Connections*: Outputs to Google Drive3.  

- **Google Drive3**  
  - *Type*: Google Drive Upload  
  - *Role*: Stores generated images in "image" folder on Google Drive.  
  - *Connections*: Metadata sent to Google Sheets7.  
  - *Credentials*: Google Drive OAuth2.  

- **Google Sheets7**  
  - *Type*: Google Sheets Update  
  - *Role*: Updates sheet with image URLs and IDs linked to scene records.  

- **Wait1**  
  - *Type*: Wait  
  - *Role*: Delays processing by 10 seconds to mitigate Together.ai API rate limits.  

- **Aggregate3**  
  - *Type*: Aggregate  
  - *Role*: Collects all generated scene Record_ids for batch processing.  
  - *Connections*: Outputs to IDs To Array1.  

- **IDs To Array1**  
  - *Type*: Code  
  - *Role*: Extracts and flattens all Record_ids into an array for retrieval.  
  - *Connections*: Outputs to Google Sheets5.  

- **Loop Over Items3**  
  - *Type*: Split In Batches  
  - *Role*: Iterates over scenes to fetch clip URLs.  
  - *Connections*: Outputs to Google Sheets9.  

- **Google Sheets9**  
  - *Type*: Google Sheets Read  
  - *Role*: Retrieves video clip metadata filtered by scene Record_id.  

---

#### 1.6 Video Clip Creation & Assembly

**Overview:**  
Creates video clips from images, aggregates clip URLs, and concatenates clips into a single video.

**Nodes Involved:**  
- Create Clips  
- Google Sheets10  
- Aggregate1  
- Video url to array  
- Combine Clips  
- Google Sheets13  

**Node Details:**

- **Create Clips**  
  - *Type*: HTTP Request  
  - *Role*: Calls local HTTP video service to transform images into video clips with specified length, frame rate, zoom speed, and ID.  
  - *Connections*: Updates Google Sheets10 with clip URLs.  
  - *Credentials*: Custom HTTP header authentication.  
  - *Edge Cases*: HTTP failures, image URL errors.  

- **Google Sheets10**  
  - *Type*: Google Sheets Update  
  - *Role*: Stores video clip URLs linked to Record_id.  

- **Aggregate1**  
  - *Type*: Aggregate  
  - *Role*: Aggregates all video clip URLs into an array.  
  - *Connections*: Outputs to Video url to array.  

- **Video url to array**  
  - *Type*: Code  
  - *Role*: Transforms aggregated URLs into expected JSON array structure for concatenation API.  
  - *Connections*: Outputs to Combine Clips.  

- **Combine Clips**  
  - *Type*: HTTP Request  
  - *Role*: Calls local HTTP video concatenation service to merge video clips into a single video file.  
  - *Connections*: Updates Google Sheets13 with concatenated video URL.  
  - *Credentials*: Custom HTTP header authentication.  
  - *Edge Cases*: Timeout (configured 50s), HTTP errors.  

- **Google Sheets13**  
  - *Type*: Google Sheets Update  
  - *Role*: Records final concatenated video URL for reference.  

---

#### 1.7 Caption Addition & Final Video Composition

**Overview:**  
Adds captions to the concatenated video, merges audio layers, and optionally integrates background music.

**Nodes Involved:**  
- Create Captions  
- Google Sheets18  
- HTTP Request  
- Combine Clips3  
- Google Sheets16  
- Add Music (disabled)  
- Google Sheets20  
- Google Sheets15  
- Code2  
- Google Drive1  
- Telegram1  
- Telegram  
- Telegram2  

**Node Details:**

- **Create Captions**  
  - *Type*: HTTP Request  
  - *Role*: Invokes local video captioning API with styling parameters to add captions.  
  - *Connections*: Updates Google Sheets18 with captioned video URL.  
  - *Credentials*: Custom HTTP header authentication.  
  - *Edge Cases*: Caption generation timeout (90s), style misconfiguration.  

- **Google Sheets18**  
  - *Type*: Google Sheets Update  
  - *Role*: Stores captioned video URL.  

- **HTTP Request**  
  - *Type*: HTTP Request  
  - *Role*: Downloads captioned video file.  
  - *Connections*: Outputs to Google Drive4.  

- **Combine Clips3**  
  - *Type*: HTTP Request  
  - *Role*: Merges raw video with generated audio layers, controlling volume levels and encoding.  
  - *Connections*: Updates Google Sheets16 with combined video + audio URL.  
  - *Credentials*: Custom HTTP header authentication.  

- **Google Sheets16**  
  - *Type*: Google Sheets Update  
  - *Role*: Stores combined video + audio URL.  

- **Add Music** (Disabled)  
  - *Type*: HTTP Request  
  - *Role*: Adds background music to video audio track with looping and volume mix. Disabled in current workflow.  

- **Google Sheets20**  
  - *Type*: Google Sheets Update  
  - *Role*: Would update music-added video URL if enabled.  

- **Google Sheets15**  
  - *Type*: Google Sheets Read  
  - *Role*: Provides metadata for final video processing steps.  

- **Code2**  
  - *Type*: Code  
  - *Role*: Extracts Google Drive file IDs from TTS Audio URLs for downloading.  

- **Google Drive1**  
  - *Type*: Google Drive Download  
  - *Role*: Downloads caption JSON file for internal use.  

- **Telegram1, Telegram, Telegram2**  
  - *Type*: Telegram Node  
  - *Role*: Sends video files and messages to Telegram chat IDs for notification and sharing.  
  - *Credentials*: Telegram API OAuth2 or Bot Token.  
  - *Edge Cases*: Telegram API rate limits, chat ID errors.  

---

#### 1.8 Output Storage & Notification

**Overview:**  
Updates the central Google Sheets document with all relevant output URLs and metadata, stores files on Google Drive, and sends notifications via Telegram.

**Nodes Involved:**  
- Google Sheets (multiple nodes: 1,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,20)  
- Google Drive (multiple nodes: 1,2,3,4,6)  
- Telegram, Telegram1, Telegram2  
- HTTP Request (final file download)  

**Node Details:**  
- Google Sheets nodes serve to append or update rows with metadata such as script, audio URLs, image URLs, clip URLs, captions URL, final video URL, and scenes array.  
- Google Drive nodes upload and download files to/from designated folders for TTS audios, images, captions, and video clips.  
- Telegram nodes send messages and videos to specified chat IDs for real-time notifications and sharing.  
- HTTP Request nodes manage file downloads from URLs before uploading to Google Drive or sending to Telegram.  

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                              | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                  |
|-----------------------------|--------------------------------|----------------------------------------------|----------------------------|----------------------------|---------------------------------------------------------------------------------------------|
| On form submission          | Form Trigger                   | Entry point to gather user input             | -                          | Set Idea, Google Sheets3... | By: Laki                                                                                   |
| Set Idea                   | Set                           | Assigns user input & duration variables      | On form submission          | Switch                     |                                                                                             |
| Switch                     | Switch                        | Routes flow by video type                     | Set Idea                   | Script Writier 🧠, Long form to Script Writier 🧠 |                                                                                             |
| Script Writier 🧠           | LangChain LLM Chain           | Generates script from user idea               | Switch                     | Output Parser 🛠            |                                                                                             |
| Long form to Script Writier 🧠 | LangChain LLM Chain           | Generates script from transcript              | Switch                     | Output Parser 🛠            |                                                                                             |
| Output Parser 🛠            | LangChain LLM Chain           | Parses AI output into structured JSON         | Script Writier 🧠, Long form to Script Writier 🧠 | Combine                   |                                                                                             |
| Combine                    | Set                           | Concatenate Hook, Script, CTA                 | Output Parser 🛠            | Format Cleanup             |                                                                                             |
| Format Cleanup             | Set                           | Cleans up script formatting                    | Combine                    | Generate TTS               |                                                                                             |
| Generate TTS               | HTTP Request                  | Calls TTS API to generate voice audio         | Format Cleanup, Google Sheets1 | Google Drive                |                                                                                             |
| Google Drive               | Google Drive                  | Uploads TTS audio to Drive                     | Generate TTS               | Google Sheets14            |                                                                                             |
| Google Sheets1             | Google Sheets                 | Stores initial metadata                        | On form submission          | Generate TTS               |                                                                                             |
| Google Sheets14            | Google Sheets                 | Updates sheet with TTS audio link              | Google Drive               | Get Captions               |                                                                                             |
| Get Captions               | HTTP Request                  | Fetches TTS timestamps JSON                    | Google Sheets14            | Google Drive1              |                                                                                             |
| Google Drive1              | Google Drive                  | Downloads timestamps JSON file                 | Get Captions               | Extract from File          |                                                                                             |
| Extract from File          | Extract From File             | Parses JSON timestamps                         | Google Drive1              | Aggregate                  |                                                                                             |
| Aggregate                  | Aggregate                    | Aggregates timestamps data                     | Extract from File          | Fixer                      |                                                                                             |
| Fixer                      | Code                         | Merges short last segment with previous       | Aggregate                  | Split Out                  |                                                                                             |
| Split Out                  | Split Out                    | Splits segments for image prompt generation   | Fixer                      | Image Prompter V2 📷       |                                                                                             |
| Split into 5s Scenes       | Code                         | Splits script into ~6s segments                | Aggregate                  | Google Sheets3             |                                                                                             |
| Google Sheets3             | Google Sheets                 | Stores segmented script & duration             | Split into 5s Scenes       | Google Drive2              |                                                                                             |
| Google Drive2              | Google Drive                  | Uploads scene JSON data                        | Google Sheets3             |                             |                                                                                             |
| Loop Over Items            | Split In Batches             | Iterates over scenes for image generation     | Split Out2                 | HTTP - Together.ai, Image Prompter V2 📷 | ## together.ai FREE - Has rate limits so much slower due to batching                         |
| Image Prompter V2 📷        | LangChain LLM Chain           | Generates image prompts from script segments  | Loop Over Items            | Google Sheets4             |                                                                                             |
| HTTP - Together.ai         | HTTP Request                  | Calls Together.ai to generate images          | Loop Over Items            | Base64 To String           |                                                                                             |
| Base64 To String           | Set                           | Extracts base64 image data                      | HTTP - Together.ai         | Convert String to binary   |                                                                                             |
| Convert String to binary   | Move Binary Data             | Converts base64 string to binary image data   | Base64 To String           | Google Drive3              |                                                                                             |
| Google Drive3              | Google Drive                  | Uploads generated images                        | Convert String to binary   | Google Sheets7             |                                                                                             |
| Google Sheets7             | Google Sheets                 | Updates sheet with image URLs                   | Google Drive3              | Wait1                      |                                                                                             |
| Wait1                      | Wait                         | Delay to handle Together.ai rate limits        | Google Sheets7             | Loop Over Items            |                                                                                             |
| Aggregate3                 | Aggregate                    | Aggregates all scene Record_ids                 | Google Sheets4             | IDs To Array1              |                                                                                             |
| IDs To Array1              | Code                         | Converts scene IDs into array                    | Aggregate3                | Google Sheets5             |                                                                                             |
| Google Sheets5             | Google Sheets                 | Updates sheet with scenes array                  | IDs To Array1             | Loop Over Items3           |                                                                                             |
| Loop Over Items3           | Split In Batches             | Iterates over scenes for clip URL retrieval    | Google Sheets10            | Google Sheets9             |                                                                                             |
| Google Sheets9             | Google Sheets                 | Reads clip URLs filtered by scene IDs           | Loop Over Items3           | Create Clips               |                                                                                             |
| Create Clips               | HTTP Request                  | Creates video clips from images                  | Google Sheets9             | Google Sheets10            | ## Create clips from images                                                                 |
| Google Sheets10            | Google Sheets                 | Updates sheet with clip URLs                      | Create Clips               | Loop Over Items3           |                                                                                             |
| Aggregate1                 | Aggregate                    | Aggregates video clip URLs                        | Google Sheets11            | Video url to array         |                                                                                             |
| Video url to array         | Code                         | Formats clip URLs into JSON array for API       | Aggregate1                 | Combine Clips              |                                                                                             |
| Combine Clips              | HTTP Request                  | Concatenates clips into final video              | Video url to array         | Google Sheets13            | ## Combine clips into 1 video (disabled sticky note)                                        |
| Google Sheets13            | Google Sheets                 | Updates sheet with concatenated video URL        | Combine Clips              | Create Captions            |                                                                                             |
| Create Captions            | HTTP Request                  | Adds captions to video                            | Google Sheets17            | Google Sheets18            | ## Add Captions (disabled sticky note)                                                      |
| Google Sheets18            | Google Sheets                 | Updates sheet with captioned video URL           | Create Captions            | HTTP Request               |                                                                                             |
| HTTP Request              | HTTP Request                  | Downloads captioned video                         | Google Sheets18            | Google Drive4              |                                                                                             |
| Google Drive4              | Google Drive                  | Uploads final video clips                          | HTTP Request               | Telegram1                  |                                                                                             |
| Telegram1                  | Telegram                      | Sends video file link & metadata via Telegram    | Google Drive4              | Google Drive6              |                                                                                             |
| Google Drive6              | Google Drive                  | Downloads final video file                         | Telegram1                 | Telegram, Telegram2        |                                                                                             |
| Telegram                   | Telegram                      | Sends final video via Telegram                     | Google Drive6             |                            |                                                                                             |
| Telegram2                  | Telegram                      | Sends final video to another Telegram chat ID     | Google Drive6             |                            |                                                                                             |
| Combine Clips3             | HTTP Request                  | Composes video and audio layers                    | Code2                      | Google Sheets16            | ## Combine Video & Audio (disabled sticky note)                                            |
| Google Sheets16            | Google Sheets                 | Updates sheet with combined video+audio URL       | Combine Clips3             | Create Captions            |                                                                                             |
| Add Music (disabled)       | HTTP Request                  | Adds background music to video (currently off)   |                           | Google Sheets20            | ## Add BG Music (disabled sticky note)                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("On form submission"):**  
   - Configure form fields: "The Main Topic" (textarea, required), "Duration" (dropdown), "Generative Style" (dropdown), "Video Type" (dropdown, required), "TTS Voice" (dropdown, required), "Image Provider" (dropdown, required).  
   - Set webhook ID if needed.

2. **Add a Set node ("Set Idea"):**  
   - Assign variables "User Input" from form "The Main Topic".  
   - Assign "Script Duration" from form "Duration".

3. **Add a Switch node ("Switch"):**  
   - Add rules to route based on "Video Type" form field:  
     - "From transcript" → "Long form to Script Writier 🧠"  
     - "From user idea" → "Script Writier 🧠"

4. **Create LangChain LLM Chain nodes:**  
   - "Script Writier 🧠": Prompt to generate video script from user idea and duration.  
   - "Long form to Script Writier 🧠": Prompt to generate script from transcript.  
   - Use OpenRouter API credential with DeepSeek model.

5. **Create Output Parser node ("Output Parser 🛠"):**  
   - Define JSON schema for Title, Description, Hook, MainScript, CTA.  
   - Connect outputs of script writers here.

6. **Add Set nodes for combining and cleaning script:**  
   - "Combine": Concatenate Hook, MainScript, CTA into "Main Script".  
   - "Format Cleanup": Remove excessive newlines and asterisks from script text.

7. **Add HTTP Request node for "Generate TTS":**  
   - POST to local TTS endpoint with script, voice selection, speed.  
   - Expect mp3 audio and timestamps.

8. **Add Google Drive node to upload TTS audio:**  
   - Upload to "tts" folder, filename from request header date.

9. **Add Google Sheets nodes:**  
   - Append initial metadata (form data).  
   - Update sheet with TTS audio link.

10. **Add HTTP Request node "Get Captions":**  
    - GET captions timestamps JSON from local service using TTS response header.

11. **Add Google Drive node to download caption timestamps JSON.**

12. **Add Extract From File node to parse timestamps JSON.**

13. **Add Aggregate node to aggregate timestamps into segments.**

14. **Add Code node "Fixer" to merge short last segment if <2 seconds.**

15. **Add Split Out node to split segments for image prompt generation.**

16. **Add Code node "Split into 5s Scenes":**  
    - Split segments into ~6-second scenes with pause buffer.  
    - Calculate total duration.

17. **Add Google Sheets node to append segmented scenes and duration info.**

18. **Add Google Drive node to upload scene JSON.**

19. **Add Split In Batches node "Loop Over Items" to iterate over scenes.**

20. **Add LangChain LLM Chain "Image Prompter V2 📷":**  
    - Generate image prompt per scene including generative style.  
    - Use OpenRouter credential.

21. **Add HTTP Request node "HTTP - Together.ai":**  
    - Call Together.ai image generation API with prompt.  
    - Receive base64-encoded images.

22. **Add Set node "Base64 To String" to extract base64 image.**

23. **Add Move Binary Data node to convert base64 string to binary file.**

24. **Add Google Drive node to upload images to "image" folder.**

25. **Add Google Sheets node to update image URLs linked to scenes.**

26. **Add Wait node to delay 10 seconds between batches to avoid rate limits.**

27. **Add Aggregate node to collect all scene Record_ids.**

28. **Add Code node "IDs To Array1" to flatten scene IDs into array.**

29. **Add Google Sheets node to update scene array for references.**

30. **Add Split In Batches node "Loop Over Items3" to iterate over scenes for video clips.**

31. **Add Google Sheets node to read clip info filtered by scene IDs.**

32. **Add HTTP Request node "Create Clips":**  
    - POST to local video service to create clips from images using image URL, duration, frame rate, zoom speed.  
    - Use header auth credential.

33. **Add Google Sheets node to update video clip URLs.**

34. **Add Aggregate node to aggregate clip URLs.**

35. **Add Code node "Video url to array" to format clip URLs for concatenation.**

36. **Add HTTP Request node "Combine Clips":**  
    - POST to local video concatenation service with clip URLs array.  
    - Use header auth credential.

37. **Add Google Sheets node to update concatenated video URL.**

38. **Add HTTP Request node "Create Captions":**  
    - POST to captioning API with video URL and styling options.  
    - Use header auth credential.

39. **Add Google Sheets node to update captioned video URL.**

40. **Add HTTP Request node to download captioned video file.**

41. **Add Google Drive node to upload final video to clips folder.**

42. **Add Telegram nodes to send video files and metadata to preconfigured chat IDs.**

43. **Optional:** Add HTTP Request node "Combine Clips3" to merge video and audio layers with volume control. Update Google Sheets accordingly (disabled in current workflow).

44. **Optional:** Add HTTP Request node "Add Music" to add background music (disabled).**

45. **Set up all Google Sheets and Google Drive credentials with OAuth2.**

46. **Set up OpenRouter and Together.ai HTTP header authentication credentials.**

47. **Ensure local services at host.docker.internal ports 8880, 9090, 9000 are accessible and running.**

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                  |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------|
| together.ai FREE tier has rate limits causing slower image generation; use Wait node to mitigate | Sticky Note5, Loop Over Items                    |
| Video generation uses local HTTP services at host.docker.internal (ports 8880, 9090, 9000)      | Workflow HTTP Requests                           |
| Telegram chat IDs configured for notifications; update with your own chat IDs                 | Telegram nodes                                   |
| Google Sheets document used extensively for data storage and update; requires proper OAuth2   | Google Sheets nodes                              |
| OpenRouter API used for AI language model calls with DeepSeek models                           | Open Router credentials                          |
| All TTS, image, and video files stored in Google Drive folders for centralized access          | Google Drive nodes                               |
| The workflow is designed to produce short-form videos with dynamic scene images and captions   | Overall workflow purpose                          |

---

**Disclaimer:** The text provided is generated exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.