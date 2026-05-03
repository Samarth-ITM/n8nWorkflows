Transform Viral Videos into New Content with AI Voice Cloning & Multi-Platform Publishing

https://n8nworkflows.xyz/workflows/transform-viral-videos-into-new-content-with-ai-voice-cloning---multi-platform-publishing-5471


# Transform Viral Videos into New Content with AI Voice Cloning & Multi-Platform Publishing

---

## 1. Workflow Overview

This workflow automates the transformation of viral video content into new multimedia assets using AI voice cloning and multi-platform publishing. It targets creators and social media managers aiming to repurpose popular videos by generating fresh text content, synthesizing voiceovers, editing videos with text overlays, and publishing across various social platforms.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Content Source Identification:** Trigger reception from WhatsApp or Google Sheets and classification of video source platforms (Instagram, TikTok, YouTube).
- **1.2 Content Retrieval & Preprocessing:** Fetching original video/audio and metadata from source platforms and performing audio filtering and transcription.
- **1.3 AI Text Generation & Processing:** Using OpenAI and Google Gemini models to generate new texts and clean up the content for reuse.
- **1.4 Voice Cloning & Video Generation:** Obtaining speaker voice profiles, generating AI voice-cloned audio, and assembling new videos with text overlays.
- **1.5 Publishing Preparation & Multi-Platform Upload:** Preparing final assets, uploading covers and videos to hosting services, and automating publishing via "Blotato" API endpoints.
- **1.6 Data Persistence & Logging:** Saving intermediate and final data to Google Sheets for tracking and analytics.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Content Source Identification

**Overview:**  
Receives triggers from WhatsApp messages or Google Sheets updates and determines the original platform of the viral video content.

**Nodes Involved:**  
- WhatsApp Trigger  
- Google Sheets Trigger1  
- If1  
- AI Agent  
- Switch  
- Get Post Content from instagram  
- Get Post Content from tiktok  
- Get Post Content from youtube  
- Edit Fields, Edit Fields1, Edit Fields2  
- Google Gemini Chat Model  
- Structured Output Parser

**Node Details:**

- **WhatsApp Trigger**  
  - Type: WhatsApp webhook trigger  
  - Role: Starts workflow upon receiving WhatsApp messages containing video links or commands  
  - Configuration: Webhook with unique ID; listens for incoming messages  
  - Input: WhatsApp incoming messages  
  - Output: Passes data to AI Agent node  
  - Possible Failures: Webhook connection issues, malformed messages  

- **Google Sheets Trigger1**  
  - Type: Google Sheets trigger  
  - Role: Starts workflow when new rows or updates are detected in a configured Google Sheet  
  - Configuration: Spreadsheet ID and worksheet specified  
  - Output: Passes data to If1 node  
  - Edge Cases: Triggering on irrelevant rows; permission errors  

- **If1**  
  - Type: Conditional logic  
  - Role: Decides next steps based on Google Sheets data conditions (e.g., valid input detection)  
  - Output: Routes to Prepare for Publish or halts  

- **AI Agent**  
  - Type: Langchain AI agent  
  - Role: Analyzes incoming message or sheet data to classify content source and intent  
  - Configuration: Linked to Google Gemini Chat Model (as language model) and Structured Output Parser  
  - Input: WhatsApp or sheet data  
  - Output: Routes to Switch node for platform-specific content extraction  
  - Failures: AI model timeout or parsing errors  

- **Google Gemini Chat Model**  
  - Type: Google Gemini conversational AI model  
  - Role: Provides conversational AI power for AI Agent node  
  - Configuration: Requires Google Gemini credentials  
  - Input/Output: AI Agent interface  

- **Structured Output Parser**  
  - Type: Langchain structured parser  
  - Role: Parses AI output into structured JSON for workflow routing  
  - Input: AI Agent output  
  - Output: Parsed data for Switch node  

- **Switch**  
  - Type: Conditional router  
  - Role: Determines which social media platform content is from (Instagram, TikTok, YouTube)  
  - Output: Routes to respective Get Post Content nodes  

- **Get Post Content from instagram / tiktok / youtube**  
  - Type: HTTP Request  
  - Role: Fetches metadata and media URLs from the specified platform's API or web endpoints  
  - Configuration: Platform-specific API endpoints and authentication  
  - Output: Passes raw data to Edit Fields nodes  

- **Edit Fields, Edit Fields1, Edit Fields2**  
  - Type: Set/Transform node  
  - Role: Normalizes and prepares fetched data for downstream processing (e.g., extracting URLs, captions)  
  - Output: Data ready for further processing  

---

### 2.2 Content Retrieval & Preprocessing

**Overview:**  
Downloads audio/video content, filters audio sources, and performs transcription via AWS services.

**Nodes Involved:**  
- Switch1  
- Filter Instagram Audio  
- Filter Audio1  
- Filter Youtube Audio  
- Download Audio1  
- AWS S31  
- AWS Transcribe2  
- Wait5  
- AWS Transcribe3  
- Template ID1  
- Save Data1  
- Save New Data

**Node Details:**

- **Switch1**  
  - Type: Router for audio filtering  
  - Role: Routes content to appropriate audio filter based on platform  

- **Filter Instagram Audio / Filter Audio1 / Filter Youtube Audio**  
  - Type: Code node  
  - Role: Filters and selects relevant audio streams or URLs from platform-specific content  
  - Possible Issues: Unexpected data formats, empty audio streams  

- **Download Audio1**  
  - Type: HTTP Request  
  - Role: Downloads the filtered audio file for transcription and voice cloning  
  - Config: Proper URL and authentication headers  
  - Edge Cases: Download failures, slow responses  

- **AWS S31**  
  - Type: AWS S3 Node  
  - Role: Uploads downloaded audio to AWS S3 bucket for transcription processing  
  - Credentials: AWS IAM permission required  
  - Failures: Upload permission denied, network errors  

- **AWS Transcribe2 / AWS Transcribe3**  
  - Type: AWS Transcribe service  
  - Role: Performs speech-to-text transcription on audio files stored in S3  
  - Output: Transcribed text passed downstream  
  - Failures: Transcription errors, unsupported audio format  

- **Wait5**  
  - Type: Wait node (webhook)  
  - Role: Waits for asynchronous transcription completion before proceeding  

- **Template ID1**  
  - Type: Code node  
  - Role: Assigns or generates template identifiers for data tracking  

- **Save Data1**  
  - Type: Google Sheets  
  - Role: Saves intermediate transcription-related data for audit and reference  

- **Save New Data**  
  - Type: Google Sheets  
  - Role: Saves newly processed metadata for video and audio content  

---

### 2.3 AI Text Generation & Processing

**Overview:**  
Generates new text content using OpenAI models, cleans and splits generated text for video subtitles or captions.

**Nodes Involved:**  
- Get Text  
- OpenAI1  
- Switch1 (previously described)  
- Perplexity  
- Clean Up  
- Generate Texts  
- Split  
- Video ID

**Node Details:**

- **Get Text**  
  - Type: OpenAI (Langchain)  
  - Role: Generates initial text content based on input prompts or transcriptions  
  - Configuration: OpenAI credentials, prompt templates  
  - Outputs: Text for further refinement  

- **OpenAI1**  
  - Type: OpenAI (Langchain)  
  - Role: Further processes or classifies generated text  
  - Connected to Switch1 for routing based on content type  

- **Perplexity**  
  - Type: HTTP Request  
  - Role: Queries Perplexity AI API for additional context or validation of generated text  

- **Clean Up**  
  - Type: Code  
  - Role: Cleans generated text removing noise, special characters, or unwanted tokens  

- **Generate Texts**  
  - Type: OpenAI (Langchain)  
  - Role: Generates multiple text variants or segments for use in video content  

- **Split**  
  - Type: Code  
  - Role: Splits long text into smaller chunks suitable for subtitles or scene captions  

- **Video ID**  
  - Type: Code  
  - Role: Generates or assigns unique video identifiers for tracking and storage  

---

### 2.4 Voice Cloning & Video Generation

**Overview:**  
Obtains voice profile from speaker data, generates AI voice-cloned audio, and creates video assets with added text.

**Nodes Involved:**  
- Get Speaker  
- Generate Video  
- Wait  
- Get Video  
- Add Text To Video  
- Wait1  
- Get Final Video  
- Save Final Data

**Node Details:**

- **Get Speaker**  
  - Type: HTTP Request  
  - Role: Retrieves or clones speaker voice profile from external voice cloning API  
  - Possible Failures: API authentication, unavailable voice model  

- **Generate Video**  
  - Type: HTTP Request  
  - Role: Sends request to video generation API to create video with cloned voiceover  
  - Input: Voice audio, video template  

- **Wait**  
  - Type: Wait node (webhook)  
  - Role: Waits asynchronously for video generation completion  

- **Get Video**  
  - Type: HTTP Request  
  - Role: Retrieves generated video asset from external service  

- **Add Text To Video**  
  - Type: HTTP Request  
  - Role: Sends request to add subtitles or text overlays on the video  

- **Wait1**  
  - Type: Wait node (webhook)  
  - Role: Waits for video text overlay processing completion  

- **Get Final Video**  
  - Type: HTTP Request  
  - Role: Retrieves final edited video  

- **Save Final Data**  
  - Type: Google Sheets  
  - Role: Logs final video metadata and publishing information  

---

### 2.5 Publishing Preparation & Multi-Platform Upload

**Overview:**  
Prepares the final assets for publishing and distributes videos across social media platforms using Blotato integration.

**Nodes Involved:**  
- Prepare for Publish  
- Upload to Blotato  
- [Instagram] Publish via Blotato  
- [Facebook] Publish via Blotato  
- [Linkedin] Publish via Blotato  
- [Tiktok] Publish via Blotato  
- [Youtube] Publish via Blotato  
- [Twitter] Publish via Blotato

**Node Details:**

- **Prepare for Publish**  
  - Type: Set node  
  - Role: Sets variables and metadata needed for publishing (e.g., video URLs, captions)  

- **Upload to Blotato**  
  - Type: HTTP Request  
  - Role: Uploads final video and cover images to Blotato platform for distribution  

- **[Platform] Publish via Blotato (Instagram, Facebook, Linkedin, Tiktok, Youtube, Twitter)**  
  - Type: HTTP Request  
  - Role: Uses Blotato API endpoints to publish content to respective social media platforms  
  - Configuration: API keys, access tokens for each platform  
  - Failures: API rate limits, invalid credentials, platform-specific errors  

---

### 2.6 Data Persistence & Logging

**Overview:**  
Stores all relevant metadata and processing information into Google Sheets for auditing, analysis, and tracking.

**Nodes Involved:**  
- Save Data1  
- Save New Data  
- Save Final Data  

**Node Details:**

- **All Google Sheets nodes**  
  - Type: Google Sheets  
  - Role: Append or update rows with transcription, video generation, and publishing data  
  - Configuration: Spreadsheet ID, worksheet, and column mapping  
  - Failures: Permission errors, quota exceeded  

---

## 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                              | Input Node(s)               | Output Node(s)                         | Sticky Note                      |
|------------------------------|----------------------------------|----------------------------------------------|-----------------------------|--------------------------------------|---------------------------------|
| WhatsApp Trigger             | WhatsApp Trigger                 | Starting trigger from WhatsApp messages      |                             | AI Agent                             |                                 |
| Google Sheets Trigger1       | Google Sheets Trigger            | Starting trigger from Google Sheets          |                             | If1                                 |                                 |
| If1                         | If                             | Conditional routing based on sheet data      | Google Sheets Trigger1       | Prepare for Publish                  |                                 |
| AI Agent                    | Langchain Agent                 | AI classification and routing                 | WhatsApp Trigger             | Switch                              |                                 |
| Google Gemini Chat Model     | Langchain LM Chat Google Gemini | AI language model for AI Agent                 |                             | AI Agent                            |                                 |
| Structured Output Parser     | Langchain Output Parser Structured | Parses AI Agent output                         |                             | AI Agent                            |                                 |
| Switch                      | Switch                         | Routes to platform-specific content fetch    | AI Agent                    | Get Post Content from instagram / tiktok / youtube |                                 |
| Get Post Content from instagram | HTTP Request                   | Fetches Instagram post content                | Switch                     | Edit Fields                        |                                 |
| Get Post Content from tiktok | HTTP Request                   | Fetches TikTok post content                    | Switch                     | Edit Fields1                       |                                 |
| Get Post Content from youtube | HTTP Request                   | Fetches YouTube post content                  | Switch                     | Edit Fields2                       |                                 |
| Edit Fields                 | Set                            | Normalizes Instagram data                      | Get Post Content from instagram | Get Cover                        |                                 |
| Edit Fields1                | Set                            | Normalizes TikTok data                         | Get Post Content from tiktok | Get Cover                        |                                 |
| Edit Fields2                | Set                            | Normalizes YouTube data                        | Get Post Content from youtube | Get Cover                        |                                 |
| Get Cover                  | HTTP Request                   | Fetches cover images for posts                  | Edit Fields / Edit Fields1 / Edit Fields2 | Upload Cover                  |                                 |
| Upload Cover               | HTTP Request                   | Uploads cover images to hosting                  | Get Cover                   | Get Text                         |                                 |
| Get Text                   | Langchain OpenAI               | Generates base text content                       | Upload Cover                | OpenAI1                         |                                 |
| OpenAI1                    | Langchain OpenAI               | Processes and routes text                          | Get Text                   | Switch1                        |                                 |
| Switch1                    | Switch                         | Routes audio filtering based on content          | OpenAI1                     | Filter Instagram Audio / Filter Audio1 / Filter Youtube Audio |                                 |
| Filter Instagram Audio     | Code                          | Filters Instagram audio streams                    | Switch1                     | Download Audio1                  |                                 |
| Filter Audio1              | Code                          | Filters generic audio streams                      | Switch1                     | Download Audio1                  |                                 |
| Filter Youtube Audio       | Code                          | Filters YouTube audio streams                       | Switch1                     | Download Audio1                  |                                 |
| Download Audio1            | HTTP Request                   | Downloads audio files                              | Filter nodes                | AWS S31 / AWS S31 (loop)         |                                 |
| AWS S31                    | AWS S3                        | Uploads audio files to AWS S3 for transcription    | Download Audio1             | AWS Transcribe2                  |                                 |
| AWS Transcribe2            | AWS Transcribe                | Performs transcription on audio                     | AWS S31                     | Wait5                          |                                 |
| Wait5                      | Wait                          | Waits for transcription completion                  | AWS Transcribe2             | AWS Transcribe3                 |                                 |
| AWS Transcribe3            | AWS Transcribe                | Final transcription step                             | Wait5                       | Template ID1                   |                                 |
| Template ID1               | Code                          | Assigns template IDs for data tracking               | AWS Transcribe3             | Save Data1                     |                                 |
| Save Data1                 | Google Sheets                 | Saves transcription and related data                 | Template ID1                | Perplexity                    |                                 |
| Perplexity                 | HTTP Request                  | Queries Perplexity AI for text validation             | Save Data1                  | Clean Up                      |                                 |
| Clean Up                   | Code                          | Cleans generated text                                 | Perplexity                  | Generate Texts                |                                 |
| Generate Texts             | Langchain OpenAI              | Generates multiple text variants                      | Clean Up                    | Split                        |                                 |
| Split                      | Code                          | Splits text into manageable chunks                     | Generate Texts              | Video ID                     |                                 |
| Video ID                   | Code                          | Assigns unique video IDs                                | Split                      | Save New Data                |                                 |
| Save New Data              | Google Sheets                 | Saves new video metadata                                | Video ID                   | Get Speaker                 |                                 |
| Get Speaker                | HTTP Request                  | Retrieves or clones AI voice profile                     | Save New Data               | Generate Video             |                                 |
| Generate Video             | HTTP Request                  | Generates video with AI voiceover                        | Get Speaker                | Wait                       |                                 |
| Wait                       | Wait                          | Waits for video generation completion                    | Generate Video             | Get Video                  |                                 |
| Get Video                  | HTTP Request                  | Retrieves generated video                                 | Wait                       | Add Text To Video          |                                 |
| Add Text To Video          | HTTP Request                  | Adds subtitles and text overlay to video                   | Get Video                  | Wait1                      |                                 |
| Wait1                      | Wait                          | Waits for text overlay processing completion              | Add Text To Video           | Get Final Video            |                                 |
| Get Final Video            | HTTP Request                  | Retrieves final video asset                                | Wait1                      | Save Final Data            |                                 |
| Save Final Data            | Google Sheets                 | Saves final video publishing data                          | Get Final Video             |                              |                                 |
| Prepare for Publish        | Set                           | Prepares metadata and variables for publishing              | If1                        | Upload to Blotato          |                                 |
| Upload to Blotato          | HTTP Request                  | Uploads final video and cover to Blotato platform           | Prepare for Publish         | [Instagram], [Facebook], [Linkedin], [Tiktok], [Youtube], [Twitter] Publish via Blotato |                                 |
| [Instagram] Publish via Blotato | HTTP Request             | Publishes content to Instagram via Blotato API              | Upload to Blotato           |                              |                                 |
| [Facebook] Publish via Blotato | HTTP Request             | Publishes content to Facebook via Blotato API               | Upload to Blotato           |                              |                                 |
| [Linkedin] Publish via Blotato | HTTP Request             | Publishes content to Linkedin via Blotato API               | Upload to Blotato           |                              |                                 |
| [Tiktok] Publish via Blotato | HTTP Request               | Publishes content to TikTok via Blotato API                 | Upload to Blotato           |                              |                                 |
| [Youtube] Publish via Blotato | HTTP Request              | Publishes content to YouTube via Blotato API                | Upload to Blotato           |                              |                                 |
| [Twitter] Publish via Blotato | HTTP Request              | Publishes content to Twitter via Blotato API                | Upload to Blotato           |                              |                                 |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **WhatsApp Trigger** node with a webhook to listen for incoming messages containing URLs or commands.
   - Add a **Google Sheets Trigger** node configured to trigger on new/updated rows in the tracking spreadsheet.

2. **Add Conditional Logic:**
   - Add an **If** node connected to Google Sheets Trigger to route valid data to processing.

3. **Configure AI Agent:**
   - Add a **Langchain AI Agent** node.
   - Configure the agent to use the **Google Gemini Chat Model** as its language model and the **Structured Output Parser** for output parsing.
   - Connect WhatsApp Trigger output and Google Gemini Chat Model to the AI Agent input ports.
   - The AI Agent routes to a **Switch** node.

4. **Setup Platform Content Fetching:**
   - Add a **Switch** node to distinguish content source (Instagram, TikTok, YouTube).
   - Add three **HTTP Request** nodes to fetch post content from each platform, connected respectively from the Switch outputs.
   - Add **Set** nodes (Edit Fields, Edit Fields1, Edit Fields2) after each fetch node to normalize data.

5. **Retrieve Media Assets:**
   - Add a **Get Cover** HTTP Request node to fetch cover images, connected from all Edit Fields nodes.
   - Add an **Upload Cover** HTTP Request to upload cover images to hosting service.

6. **Generate Text Content:**
   - Add a **Langchain OpenAI** node named **Get Text** to generate base text from the cover upload output.
   - Add a second **OpenAI** node (**OpenAI1**) to further process text.
   - Connect **OpenAI1** to a **Switch1** node which routes audio filtering.

7. **Audio Filtering & Downloading:**
   - Add **Code** nodes for **Filter Instagram Audio**, **Filter Audio1**, and **Filter Youtube Audio**.
   - Connect these to a **Download Audio1** HTTP Request node to fetch audio files.

8. **AWS Processing:**
   - Add an **AWS S3** node to upload audio files.
   - Add two **AWS Transcribe** nodes (Transcribe2 and Transcribe3) to transcribe audio.
   - Add a **Wait** node (**Wait5**) to await transcription results.
   - Add a **Code** node (**Template ID1**) to assign template IDs.
   - Add a **Google Sheets** node (**Save Data1**) to save transcription data.

9. **Text Validation and Processing:**
   - Add a **Perplexity** HTTP Request node to validate text.
   - Add a **Code** node (**Clean Up**) to clean text.
   - Add a **Langchain OpenAI** node (**Generate Texts**) to generate multiple text variants.
   - Add a **Code** node (**Split**) to split text.
   - Add a **Code** node (**Video ID**) to assign video IDs.
   - Add a **Google Sheets** node (**Save New Data**) to save processed metadata.

10. **Voice Cloning & Video Generation:**
    - Add a **Get Speaker** HTTP Request node to obtain voice profile.
    - Add a **Generate Video** HTTP Request node to create video with AI voice.
    - Add a **Wait** node to wait for video generation.
    - Add a **Get Video** HTTP Request node to retrieve video.
    - Add an **Add Text To Video** HTTP Request node for subtitles.
    - Add a **Wait** node (**Wait1**) to await text overlay completion.
    - Add a **Get Final Video** HTTP Request node.
    - Add a **Google Sheets** node (**Save Final Data**) to log final video data.

11. **Publishing:**
    - Add a **Set** node (**Prepare for Publish**) to set publishing metadata.
    - Add an **Upload to Blotato** HTTP Request node to upload assets.
    - Add HTTP Request nodes for each platform ([Instagram], [Facebook], [Linkedin], [Tiktok], [Youtube], [Twitter]) connected from Upload to Blotato node for publishing.

12. **Connect All Nodes According to Logical Flow:**
    - Link triggers → AI Agent → Switch → Content fetch → Edit Fields → Get Cover → Upload Cover → Text generation → Audio filtering → Download audio → AWS S3 → AWS Transcribe → Wait → Template ID → Save Data → Perplexity → Clean Up → Generate Texts → Split → Video ID → Save New Data → Get Speaker → Generate Video → Wait → Get Video → Add Text → Wait → Get Final Video → Save Final Data → If → Prepare for Publish → Upload to Blotato → Publish nodes.

13. **Credentials Setup:**
    - Configure WhatsApp webhook credentials.
    - Set Google Sheets API credentials for triggers and data saving.
    - Configure OpenAI API keys for Langchain nodes.
    - Set Google Gemini API access credentials.
    - Configure AWS credentials for S3 and Transcribe services.
    - Set authentication tokens and API keys for Blotato and social platforms.

14. **Default Values and Constraints:**
    - Ensure all API keys are valid and have appropriate scopes.
    - Set wait times in Wait nodes according to expected processing delays.
    - Define prompt templates and text length limits in OpenAI nodes.
    - Validate webhook URLs and Google Sheets IDs.

---

## 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                  |
|------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow leverages Google Gemini AI for advanced conversational parsing.| Google Gemini Chat Model integration node.                                                     |
| Blotato platform used for multi-platform publishing API integration.          | Multi-platform social media publishing via Blotato.                                            |
| AWS Transcribe and S3 services handle audio transcription securely.           | AWS credentials with proper IAM roles required.                                                |
| Workflow includes asynchronous waits for external process completions.       | Wait nodes use webhook IDs to resume processing upon callbacks.                                |
| For voice cloning, external HTTP APIs are used to obtain speaker profiles.    | Ensure API endpoints and credentials are up to date.                                          |
| Video editing and text overlay handled via HTTP API services.                 | Video generation nodes rely on external video processing services.                             |
| Google Sheets used extensively for data persistence and auditing.             | Spreadsheet must have correct columns set up to capture metadata.                             |

---

Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---