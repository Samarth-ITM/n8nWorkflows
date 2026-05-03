Convert Websites to Instagram Reels with Gemini Veo, OpenAI TTS, and JsonCut

https://n8nworkflows.xyz/workflows/convert-websites-to-instagram-reels-with-gemini-veo--openai-tts--and-jsoncut-11367


# Convert Websites to Instagram Reels with Gemini Veo, OpenAI TTS, and JsonCut

### 1. Workflow Overview

This workflow automates the conversion of blog articles or web pages into engaging Instagram Reels. It targets content creators and social media managers who want to transform textual web content into short, visually consistent video clips with voiceovers, background music, and captions, then publish them automatically on Instagram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Web Scraping:** Receives URL input via chat trigger, scrapes webpage content using Firecrawl, and generates a concise summary.
- **1.2 AI Content Generation:** Uses AI agents (OpenRouter GPT models) to create video prompts, narration scripts, and social captions based on the summarized content.
- **1.3 Parallel Media Generation:** Generates background videos (Google Gemini Veo), voiceover audio (OpenAI TTS), and downloads Creative Commons background music from OpenVerse.
- **1.4 Media Upload & Aggregation:** Uploads generated videos, audio files, and background music to JsonCut storage and aggregates these assets into a single data structure.
- **1.5 Video Composition:** Uses JsonCut API to compose a final vertical video (9:16) combining clips, voiceover, music, subtitles, and branding overlays.
- **1.6 Publishing:** Uploads the composed video to Instagram as a Reel using Blotato API with the AI-generated caption.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Web Scraping

- **Overview:** Captures a URL input from a chat interface and scrapes the full webpage content. Then, it summarizes the content into a concise, 1,000-character text preserving key facts.
- **Nodes Involved:**  
  - When chat message received  
  - Scrape a url and get its content  
  - Content Summarizer  
  - OpenRouter Chat Model  

- **Node Details:**

  - **When chat message received**  
    - Type: Langchain Chat Trigger (Webhook)  
    - Role: Entry point that fires workflow upon receiving chat input containing a URL.  
    - Configuration: Default webhook settings; listens for chat input JSON with URL in `chatInput` field.  
    - Inputs: External webhook  
    - Outputs: Passes URL to next node.  
    - Edge cases: Missing or malformed URL; webhook not triggered if chat input absent.

  - **Scrape a url and get its content**  
    - Type: Firecrawl Web Scraper  
    - Role: Retrieves webpage content for the provided URL.  
    - Configuration: Scrape operation using Firecrawl API credentials.  
    - Inputs: URL from chat message node.  
    - Outputs: Extracted page content in markdown format under `data.markdown`.  
    - Edge cases: HTTP errors, blocked pages, rate limits, or content extraction failures.

  - **Content Summarizer**  
    - Type: Langchain AI Agent  
    - Role: Summarizes scraped markdown content into ~1,000 characters focusing on clarity and fact preservation.  
    - Configuration: Prompt explicitly instructs a summary with complete sentences and factual accuracy.  
    - Inputs: Markdown content from scraper node.  
    - Outputs: Concise summary text.  
    - Edge cases: AI generation errors, incomplete context, or timeout.

  - **OpenRouter Chat Model**  
    - Type: Langchain LM Chat Model (OpenRouter API)  
    - Role: Underlying AI model used by Content Summarizer agent.  
    - Configuration: Model set to `openai/gpt-5-mini` with default options.  
    - Inputs: Summarizer prompt.  
    - Outputs: Summary text.  
    - Edge cases: API auth errors, rate limits, model unavailability.

#### 2.2 AI Content Generation

- **Overview:** Generates detailed video prompts, a narration script (max 75 words), and a social media caption for creating consistent short videos based on the summarized content.
- **Nodes Involved:**  
  - AI Agent  
  - Structured Output Parser  
  - OpenRouter Chat Model2  

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain AI Agent  
    - Role: Creative director and scriptwriter agent generating 3–4 video prompts (each ~8s), narration text, and caption in JSON format.  
    - Configuration: Detailed prompt specifying style, tone, lighting, camera movements, word limit, and output JSON structure.  
    - Inputs: Summarized content from previous block.  
    - Outputs: JSON with `video_prompts` (array), `full_text` (narration), and `caption` (social caption).  
    - Edge cases: AI response format errors, JSON parse issues, API failures.

  - **Structured Output Parser**  
    - Type: Langchain Output Parser  
    - Role: Parses AI Agent's JSON output, auto-fixes minor errors, and ensures valid structured data for downstream nodes.  
    - Configuration: JSON schema example provided for validation.  
    - Inputs: Raw AI text output.  
    - Outputs: Parsed JSON object with separate fields for video prompts, narration, and caption.  
    - Edge cases: Unfixable JSON errors, missing keys.

  - **OpenRouter Chat Model2**  
    - Type: Langchain LM Chat Model (OpenRouter API)  
    - Role: AI model used by AI Agent for generation (same model as Content Summarizer).  
    - Inputs: AI Agent prompt.  
    - Outputs: AI-generated JSON.  
    - Edge cases: Same as OpenRouter Chat Model.

#### 2.3 Parallel Media Generation

- **Overview:** Concurrently generates background videos, voiceover audio, and downloads background music.
- **Nodes Involved:**  
  - Split Video Prompts  
  - Generate a video  
  - Upload video  
  - Aggregate video paths  
  - Generate audio  
  - Upload voice  
  - Get List of background Audio  
  - download background audio  
  - Upload background audio  

- **Node Details:**

  - **Split Video Prompts**  
    - Type: SplitOut  
    - Role: Splits array of 3–5 video prompts into separate items for parallel video generation.  
    - Inputs: `video_prompts` array from AI Agent.  
    - Outputs: Individual prompt per item.  
    - Edge cases: Empty array or missing prompts.

  - **Generate a video**  
    - Type: Google Gemini Veo (Langchain Node)  
    - Role: Generates short 9:16 videos (~8 seconds) from detailed prompts.  
    - Configuration: Model `veo-2.0-generate-001`, aspect ratio 9:16, duration 8s.  
    - Inputs: Single video prompt from Split Video Prompts.  
    - Outputs: Video file data.  
    - Credentials: Google Gemini API account.  
    - Edge cases: API quota, generation errors, network issues.

  - **Upload video**  
    - Type: JsonCut Upload  
    - Role: Uploads generated video files to JsonCut storage with 24-hour TTL.  
    - Inputs: Video file from Generate a video node.  
    - Outputs: Storage URL and metadata.  
    - Credentials: JsonCut API.  
    - Edge cases: Upload failures, storage limits.

  - **Aggregate video paths**  
    - Type: Aggregate  
    - Role: Aggregates all uploaded video URLs into a single list under field `videos` for composition.  
    - Inputs: Multiple uploaded video items.  
    - Outputs: Aggregated JSON data.  
    - Edge cases: Missing or incomplete uploads.

  - **Generate audio**  
    - Type: OpenAI TTS (Langchain Audio)  
    - Role: Converts narration text (`full_text`) into speech audio (MP3) with voice `onyx`.  
    - Inputs: Narration text from AI Agent output.  
    - Outputs: MP3 audio file binary.  
    - Credentials: OpenAI API.  
    - Edge cases: TTS failures, size limits.

  - **Upload voice**  
    - Type: JsonCut Upload  
    - Role: Uploads generated voiceover audio to JsonCut storage (24-hour TTL).  
    - Inputs: Audio file from Generate audio node.  
    - Outputs: Storage URL and metadata.  
    - Credentials: JsonCut API.  
    - Edge cases: Upload failures.

  - **Get List of background Audio**  
    - Type: HTTP Request  
    - Role: Queries OpenVerse API for Creative Commons background music matching query `ambiente`.  
    - Inputs: Triggered after content summarization.  
    - Outputs: JSON list of audio tracks.  
    - Edge cases: API downtime, no results returned.

  - **download background audio**  
    - Type: HTTP Request  
    - Role: Downloads a random background audio MP3 from OpenVerse results.  
    - Inputs: URL selected randomly from OpenVerse results.  
    - Outputs: Binary audio data.  
    - Edge cases: Download failures, invalid URLs.

  - **Upload background audio**  
    - Type: JsonCut Upload  
    - Role: Uploads background music audio file to JsonCut storage (24-hour TTL).  
    - Inputs: Downloaded audio file.  
    - Outputs: Storage URL and metadata.  
    - Credentials: JsonCut API.  
    - Edge cases: Upload failures.

#### 2.4 Media Upload & Aggregation

- **Overview:** Merges all uploaded media files (videos, voiceover, background music) into a single data structure preparing for final video composition.
- **Nodes Involved:**  
  - Merge All Files  
  - Aggregate Files  

- **Node Details:**

  - **Merge All Files**  
    - Type: Merge  
    - Role: Merges three inputs (uploaded videos, voiceover audio, background audio) into one combined item.  
    - Inputs: Uploaded voiceover, uploaded background audio, aggregated videos.  
    - Outputs: Merged JSON data.  
    - Edge cases: Missing inputs, partial uploads.

  - **Aggregate Files**  
    - Type: Aggregate  
    - Role: Aggregates all merged item data into a single comprehensive data object for JsonCut composition.  
    - Inputs: Merged data from Merge All Files.  
    - Outputs: Final aggregated data including all media URLs.  
    - Edge cases: Data mismatch, empty inputs.

#### 2.5 Video Composition

- **Overview:** Uses JsonCut API to create a final 720x1280 MP4 vertical video combining video clips, voiceover, background music, auto-generated subtitles, transitions, and branding overlays.
- **Nodes Involved:**  
  - Generate media  

- **Node Details:**

  - **Generate media**  
    - Type: JsonCut Video Composition  
    - Role: Combines video clips (with transitions), voiceover (80% volume), background music (20% volume), adds subtitles and branding overlays.  
    - Configuration:  
      - Video dimensions: 720x1280, 25 FPS, MP4 format  
      - Clips duration: ~5 seconds per clip, 4 clips total  
      - Subtitles: Auto-generated with letter-by-letter animation, styled font and outline  
      - Audio tracks: Voiceover and background music with specified volume mix and offsets  
      - Branding: Project icon overlay top-left, source URL text bottom-right  
    - Inputs: Aggregated media URLs and metadata.  
    - Outputs: Final composed video file URL.  
    - Credentials: JsonCut API.  
    - Edge cases: Composition errors, API timeouts, invalid media URLs.

#### 2.6 Publishing

- **Overview:** Uploads the composed video to Instagram as a Reel using Blotato API with the AI-generated caption.
- **Nodes Involved:**  
  - Upload media  
  - Create Instagram post  

- **Node Details:**

  - **Upload media**  
    - Type: Blotato Media Upload  
    - Role: Uploads the composed video file binary to Blotato for Instagram.  
    - Inputs: Final video from JsonCut composition.  
    - Outputs: Media URL for Instagram post.  
    - Credentials: Blotato API.  
    - Edge cases: Upload failures, invalid media format.

  - **Create Instagram post**  
    - Type: Blotato Instagram Post  
    - Role: Creates an Instagram Reel post with the uploaded media URL and AI-generated caption.  
    - Configuration: Instagram account ID must be set.  
    - Inputs: Media URL from Upload media; Caption from AI Agent output.  
    - Outputs: Confirmation of post creation.  
    - Credentials: Blotato API.  
    - Edge cases: Auth errors, posting limits, caption formatting issues.

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                             | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                                              |
|-------------------------------|-----------------------------------|---------------------------------------------|---------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received     | Langchain Chat Trigger            | Receives URL input via chat                  | —                               | Scrape a url and get its content |                                                                                                                                            |
| Scrape a url and get its content | Firecrawl Web Scraper             | Scrapes webpage content                       | When chat message received       | Content Summarizer             | ### Crawl Page & Summarize Content Chat trigger receives URL → Firecrawl scrapes webpage content → AI condenses it into a focused 1,000-character summary while preserving key facts |
| Content Summarizer             | Langchain AI Agent                | Summarizes scraped content                    | Scrape a url and get its content | Get List of background Audio, AI Agent |                                                                                                                                            |
| Get List of background Audio   | HTTP Request                     | Fetches CC-licensed background music list    | Content Summarizer               | download background audio      |                                                                                                                                            |
| download background audio      | HTTP Request                     | Downloads random background audio             | Get List of background Audio     | Upload background audio        |                                                                                                                                            |
| Upload background audio        | JsonCut Upload                   | Uploads background music to JsonCut           | download background audio         | Merge All Files                |                                                                                                                                            |
| AI Agent                      | Langchain AI Agent                | Generates video prompts, narration, caption  | Content Summarizer               | Split Video Prompts            | ### Generate Script, Voice & Videos AI creates 3-5 video prompts, voiceover script (max 75 words), and social caption. Then parallel generation: Google Gemini Veo produces videos (9:16, 8 sec), OpenAI TTS generates voiceover, and OpenVerse fetches CC-licensed background music |
| Structured Output Parser       | Langchain Output Parser           | Parses AI Agent JSON output                    | OpenRouter GPT Mini              | AI Agent                      |                                                                                                                                            |
| OpenRouter GPT Mini            | Langchain LM Chat Model           | AI model for structured output parsing        | —                               | Structured Output Parser       |                                                                                                                                            |
| OpenRouter Chat Model          | Langchain LM Chat Model           | AI model for content summarization             | —                               | Content Summarizer             |                                                                                                                                            |
| OpenRouter Chat Model2         | Langchain LM Chat Model           | AI model for AI Agent generation               | —                               | AI Agent                      |                                                                                                                                            |
| Split Video Prompts            | SplitOut                        | Splits video prompts array into individual items | AI Agent                      | Generate a video               |                                                                                                                                            |
| Generate a video               | Google Gemini Veo (Langchain)     | Generates short videos from prompts            | Split Video Prompts              | Upload video                  |                                                                                                                                            |
| Upload video                  | JsonCut Upload                   | Uploads generated videos to JsonCut            | Generate a video                 | Aggregate video paths          |                                                                                                                                            |
| Aggregate video paths          | Aggregate                      | Aggregates uploaded video URLs                  | Upload video                    | Merge All Files                |                                                                                                                                            |
| Generate audio                | OpenAI TTS (Langchain Audio)      | Generates narration voiceover audio             | AI Agent                       | Upload voice                  |                                                                                                                                            |
| Upload voice                  | JsonCut Upload                   | Uploads voiceover audio to JsonCut              | Generate audio                  | Merge All Files                |                                                                                                                                            |
| Merge All Files               | Merge                          | Merges uploaded voice, background audio, and videos | Upload voice, Upload background audio, Aggregate video paths | Aggregate Files              | ### Upload & Aggregate Files All generated assets (videos, voiceover, background music) are uploaded to JsonCut storage, then merged into a single data structure for composition |
| Aggregate Files              | Aggregate                      | Aggregates merged media files into one object    | Merge All Files                | Generate media                |                                                                                                                                            |
| Generate media               | JsonCut Video Composition         | Composes final vertical video with all media    | Aggregate Files                | Upload media                  | ### Compose Final Video JsonCut combines all elements: - Sequences video clips with transitions - Layers voiceover (80% volume) + background music (20%) - Adds auto-generated subtitles - Includes source attribution and branding overlay Output: 720×1280 MP4 (9:16) |
| Upload media                | Blotato Media Upload              | Uploads final video to Blotato                   | Generate media                 | Create Instagram post          |                                                                                                                                            |
| Create Instagram post         | Blotato Instagram Post            | Publishes video Reel with caption on Instagram  | Upload media                  | —                            | ### Publish to Instagram Final video uploads to Instagram via Blotato API as a Reel with AI-generated caption                            |
| Sticky Note1                 | Sticky Note                      | Workflow explanation and setup instructions     | —                               | —                            | ## How it works This workflow transforms blog articles into engaging Instagram Reels automatically... (detailed content provided)        |
| Sticky Note2                 | Sticky Note                      | Explains final video composition via JsonCut    | —                               | —                            | ### Compose Final Video JsonCut combines all elements... (see above)                                                                       |
| Sticky Note3                 | Sticky Note                      | Explains AI generation and parallel media creation | —                               | —                            | ### Generate Script, Voice & Videos AI creates 3-5 video prompts, voiceover script... (see above)                                          |
| Sticky Note4                 | Sticky Note                      | Explains crawling and summarization process      | —                               | —                            | ### Crawl Page & Summarize Content Chat trigger receives URL → Firecrawl scrapes webpage content → AI condenses it into summary (see above)|
| Sticky Note6                 | Sticky Note                      | Explains Instagram publishing step                | —                               | —                            | ### Publish to Instagram Final video uploads to Instagram via Blotato API as a Reel with AI-generated caption                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node** (Langchain Chat Trigger)  
   - Configure webhook to receive chat input with a URL in JSON field `chatInput`.

2. **Add "Scrape a url and get its content" node** (Firecrawl)  
   - Set operation to "scrape".  
   - Use Firecrawl API credentials.  
   - Connect from "When chat message received", pass URL from `chatInput`.

3. **Add "Content Summarizer" node** (Langchain AI Agent)  
   - Use prompt to summarize the scraped markdown content into 1,000 characters max, preserving facts.  
   - Connect from "Scrape a url and get its content", input `data.markdown`.

4. **Add "OpenRouter Chat Model" node** (Langchain LM Chat Model)  
   - Set model to `openai/gpt-5-mini`.  
   - Use OpenRouter API credentials.  
   - Connect to "Content Summarizer" as AI model.

5. **Add "Get List of background Audio" node** (HTTP Request)  
   - URL: `https://api.openverse.engineering/v1/audio/`  
   - Query parameters: q=ambiente, license=cc0,by,by-sa, format=json, extension=mp3.  
   - Connect from "Content Summarizer".

6. **Add "download background audio" node** (HTTP Request)  
   - Download a random audio URL from `results` array of previous node.  
   - Connect from "Get List of background Audio".

7. **Add "Upload background audio" node** (JsonCut Upload)  
   - Set category to "audio", resource to "file", TTL 24h.  
   - Use JsonCut API credentials.  
   - Connect from "download background audio".

8. **Add "AI Agent" node** (Langchain AI Agent)  
   - Prompt: generate 3-5 video prompts (8s each), narration (max 75 words), and social caption based on summarized text.  
   - Output must be valid JSON with fields: `video_prompts`, `full_text`, `caption`.  
   - Connect from "Content Summarizer".

9. **Add "OpenRouter Chat Model2" node** (Langchain LM Chat Model)  
   - Same config as previous OpenRouter Chat Model.  
   - Connect AI Agent to this model.

10. **Add "Structured Output Parser" node** (Langchain Output Parser)  
    - Provide JSON schema example as per AI Agent output.  
    - Connect from "OpenRouter GPT Mini" (used internally in AI Agent).

11. **Add "Split Video Prompts" node** (SplitOut)  
    - Split field: `output.video_prompts`.  
    - Connect from "AI Agent".

12. **Add "Generate a video" node** (Google Gemini Veo)  
    - Model: `veo-2.0-generate-001`.  
    - Aspect ratio: 9:16, Duration: 8s.  
    - Connect from "Split Video Prompts".  
    - Use Google Gemini API credentials.

13. **Add "Upload video" node** (JsonCut Upload)  
    - Category: video, resource: file, TTL 24h.  
    - Connect from "Generate a video".  
    - Use JsonCut API credentials.

14. **Add "Aggregate video paths" node** (Aggregate)  
    - Aggregate all uploaded video URLs under field `videos`.  
    - Connect from "Upload video".

15. **Add "Generate audio" node** (OpenAI TTS)  
    - Input: `full_text` narration from AI Agent output.  
    - Voice: onyx, response format: mp3.  
    - Use OpenAI API credentials.  
    - Connect from "AI Agent".

16. **Add "Upload voice" node** (JsonCut Upload)  
    - Category: audio, resource: file, TTL 24h.  
    - Connect from "Generate audio".  
    - Use JsonCut API credentials.

17. **Add "Merge All Files" node** (Merge)  
    - Configure to merge 3 inputs: uploaded voice, uploaded background audio, aggregated video paths.  
    - Connect inputs from "Upload voice", "Upload background audio", and "Aggregate video paths".

18. **Add "Aggregate Files" node** (Aggregate)  
    - Aggregate all merged data into a single item structure.  
    - Connect from "Merge All Files".

19. **Add "Generate media" node** (JsonCut Video Composition)  
    - Configure video: 720x1280, 25fps, mp4, clips with videos, overlays, subtitles, transitions.  
    - Audio tracks: voiceover (80% volume), background music (20% volume).  
    - Connect from "Aggregate Files".  
    - Use JsonCut API credentials.

20. **Add "Upload media" node** (Blotato Media Upload)  
    - Upload final video file binary.  
    - Connect from "Generate media".  
    - Use Blotato API credentials.

21. **Add "Create Instagram post" node** (Blotato Instagram Post)  
    - Instagram account ID: set your Instagram account ID.  
    - Post content text: from AI Agent caption output.  
    - Post media URL: from "Upload media" node.  
    - Connect from "Upload media".  
    - Use Blotato API credentials.

22. **Add Sticky Notes** for documentation near relevant nodes explaining each block and steps as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow transforms blog URLs to Instagram Reels with AI-generated video prompts, narration, and captions.    | Full process automated from input URL to Instagram publishing.                                     |
| Required credentials: Firecrawl API, OpenRouter API, OpenAI API, Google Gemini API, JsonCut API, Blotato API.  | See respective official sites for API keys and setup instructions.                                |
| Instagram account ID must be updated in "Create Instagram post" node before publishing.                         |                                                                                                    |
| Video composition uses JsonCut API with layered clips, transitions, auto subtitles, and branding overlays.    | JSON configuration shown in "Generate media" node parameters.                                     |
| AI prompt guides specify video style consistency, narration word count, and output JSON format strictly.      | Helps ensure uniform video aesthetics and accurate AI output parsing.                             |
| OpenVerse API is used to source free Creative Commons background music to enrich videos.                       | https://openverse.engineering                                                                       |
| Blotato API automates publishing videos to Instagram Reels with captions.                                      | https://my.blotato.com/login                                                                        |
| Firecrawl API scrapes web pages reliably for content extraction.                                              | https://www.firecrawl.dev/                                                                          |
| OpenRouter API provides AI language model access for summarization and script generation.                      | https://openrouter.ai/                                                                              |
| Google Gemini Veo API generates short vertical videos from text prompts.                                       | https://aistudio.google.com/api-keys                                                                |

---

**Disclaimer:** The provided content is derived exclusively from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.