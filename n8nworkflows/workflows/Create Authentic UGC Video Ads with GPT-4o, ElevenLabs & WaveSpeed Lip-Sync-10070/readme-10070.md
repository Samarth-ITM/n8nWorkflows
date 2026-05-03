Create Authentic UGC Video Ads with GPT-4o, ElevenLabs & WaveSpeed Lip-Sync

https://n8nworkflows.xyz/workflows/create-authentic-ugc-video-ads-with-gpt-4o--elevenlabs---wavespeed-lip-sync-10070


# Create Authentic UGC Video Ads with GPT-4o, ElevenLabs & WaveSpeed Lip-Sync

---
### 1. Workflow Overview

This workflow automates the creation of authentic User-Generated Content (UGC) video advertisements by leveraging advanced AI technologies: GPT-4o for text and image understanding, ElevenLabs for voice synthesis, and WaveSpeed for image generation and lip-sync video creation. It receives product images via a Telegram bot, analyzes the image to extract product and character information, generates testimonial scripts and corresponding images, creates natural voice-overs, synchronizes lip movements, and finally delivers a polished testimonial video back to Telegram.

**Target Use Cases:**  
- Social media marketers creating authentic product testimonial videos for platforms like TikTok and Instagram.  
- Brands seeking automated, scalable UGC video ad generation with realistic voice and lip-sync.  
- Content creators automating video content production with AI-driven creativity and synthesis.

**Logical Blocks:**  
- **1.1 Input Reception & Image Download:** Receives images via Telegram, retrieves and downloads files.  
- **1.2 Image Analysis & Content Understanding:** Uses GPT-4o to analyze images and extract product/character data.  
- **1.3 Testimonial Script & Image Prompt Generation:** Generates testimonial text and image prompts using GPT-4o and LangChain.  
- **1.4 Image Generation & Upload:** Calls WaveSpeed API to generate testimonial images, polls status, and uploads to Cloudinary.  
- **1.5 Voice Synthesis:** Generates natural voice-overs via ElevenLabs based on testimonial scripts, branching by gender.  
- **1.6 Lip-Sync Video Creation & Upload:** Uses WaveSpeed to create lip-sync videos combining audio and images, polls status, and uploads results.  
- **1.7 Delivery via Telegram:** Sends generated images and final testimonial video back to Telegram chat.  
- **1.8 Workflow Orchestration & Waits:** Manages timing of asynchronous API calls and handles conditional logic.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Image Download

- **Overview:**  
  This block listens for incoming product images via Telegram, retrieves the file path using Telegram API, and downloads the image for further processing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Set Bot Token  
  - Image Path (getFile)  
  - Download File → Gambar  

- **Node Details:**  

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point, listens to Telegram messages with photos.  
    - *Configuration:* Watches for "message" updates only, uses Telegram API credentials.  
    - *Input:* Incoming Telegram photo message.  
    - *Output:* Message data including photo file_id.  
    - *Failures:* Telegram API downtime, invalid bot token, missing photo in message.

  - **Set Bot Token**  
    - *Type:* Set  
    - *Role:* Stores Telegram Bot Token for API calls.  
    - *Configuration:* Assigns the bot token string to variable `bot id`.  
    - *Input:* Trigger output.  
    - *Output:* Passes bot token downstream.  
    - *Failures:* Misconfigured token disables further Telegram API calls.

  - **Image Path (getFile)**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves the file path for the photo using Telegram API getFile method.  
    - *Configuration:* URL built dynamically using bot token and photo `file_id`.  
    - *Input:* Bot token and photo file_id.  
    - *Output:* JSON containing `file_path` of the image.  
    - *Failures:* Invalid file_id, Telegram API errors, network issues.

  - **Download File → Gambar**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the actual photo file from Telegram servers.  
    - *Configuration:* Downloads from Telegram file URL constructed with bot token and `file_path`.  
    - *Input:* File path from previous node.  
    - *Output:* Binary image data for analysis.  
    - *Failures:* Network errors, file missing or expired on Telegram.

---

#### 2.2 Image Analysis & Content Understanding

- **Overview:**  
  Analyzes the downloaded image using GPT-4o to classify if the image is a product, character, or both, and extracts detailed attributes such as brand, color scheme, font style, outfit style, and visual description. This analysis informs subsequent AI content generation.

- **Nodes Involved:**  
  - Analyze YAML  
  - Wait (short pause)  

- **Node Details:**  

  - **Analyze YAML**  
    - *Type:* LangChain OpenAI node using GPT-4o  
    - *Role:* Performs image content analysis and outputs structured YAML data about product or character features.  
    - *Configuration:* Analyzes image URL from Telegram download; returns structured YAML without explanation.  
    - *Key Expressions:* Uses OpenAI chat model `chatgpt-4o-latest`; image URL dynamically set.  
    - *Input:* Image binary converted to accessible URL.  
    - *Output:* YAML with fields like brand_name, color_scheme, character_name, outfit_style.  
    - *Failures:* Image URL inaccessible, OpenAI API limits, parsing errors.  
    - *Version:* Requires GPT-4o support and image analysis enabled.

  - **Wait**  
    - *Type:* Wait node  
    - *Role:* Adds a pause to ensure downstream readiness or rate-limiting compliance.  
    - *Configuration:* Default wait time (unspecified, likely few seconds).  
    - *Input:* Output from Analyze YAML.  
    - *Output:* Passes data downstream.  
    - *Failures:* None typical.

---

#### 2.3 Testimonial Script & Image Prompt Generation

- **Overview:**  
  Generates a professional product testimonial prompt for image creation and a natural, emotive text-to-speech script using GPT-4o and LangChain agents, based on analyzed product/character data.

- **Nodes Involved:**  
  - PROMPT PRODUK REVIEW  
  - Wait5  
  - If4 (conditional on prompt content)  
  - TTS & CAPTION1  
  - If5 (conditional on caption content)  

- **Node Details:**  

  - **PROMPT PRODUK REVIEW**  
    - *Type:* LangChain LLM Chain with GPT-4o  
    - *Role:* Creates a detailed image generation prompt describing a natural, cinematic testimonial photo of the product with a Malaysian character.  
    - *Configuration:* Custom prompt enforcing naturalness, cinematic grain, and product interaction rules; output in English without extra symbols.  
    - *Input:* Product description and analysis from previous block.  
    - *Output:* Image prompt text.  
    - *Failures:* Model errors, prompt parsing failures.

  - **Wait5**  
    - *Type:* Wait node  
    - *Role:* Controls timing before conditional logic.  
    - *Input/Output:* Passes prompt text downstream.

  - **If4**  
    - *Type:* If node  
    - *Role:* Checks if prompt text contains disallowed double quotes (`"`) to ensure prompt validity.  
    - *Input:* Prompt text from PROMPT PRODUK REVIEW.  
    - *Output:*  
      - True: Proceeds to TTS & CAPTION1 (generate text-to-speech).  
      - False: Revert to PROMPT PRODUK REVIEW (retry).  
    - *Failures:* Expression evaluation errors.

  - **TTS & CAPTION1**  
    - *Type:* LangChain Agent  
    - *Role:* Generates a short, natural Malay-English mixed testimonial script for text-to-speech synthesis, max 10 seconds duration.  
    - *Configuration:* System prompt defines style as social media manager, emotive and persuasive speech.  
    - *Input:* Product description content in JSON.  
    - *Output:* Text-to-speech script (`textospeech`) and caption text.  
    - *Failures:* Model timeouts, output parsing errors.

  - **If5**  
    - *Type:* If node  
    - *Role:* Checks generated caption text for disallowed double quotes to validate output.  
    - *Output:*  
      - True: Proceeds to image generation.  
      - False: Loops back to TTS & CAPTION1 for retry.  
    - *Failures:* Expression evaluation errors.

---

#### 2.4 Image Generation & Upload

- **Overview:**  
  Uses WaveSpeed AI to generate a testimonial image based on the crafted prompt, polls the generation status, uploads the image to Cloudinary, and analyzes the image to determine character gender for voice selection.

- **Nodes Involved:**  
  - CREATE NEW IMG1  
  - Wait 65s1  
  - GET STATUS NEW IMG1  
  - If1  
  - UPLOAD NEW IMG V1  
  - Analyze IMG NEW2  

- **Node Details:**  

  - **CREATE NEW IMG1**  
    - *Type:* HTTP Request  
    - *Role:* Calls WaveSpeed API to generate an image with 9:16 aspect ratio using the prompt from PROMPT PRODUK REVIEW.  
    - *Configuration:* JSON body includes image URL from Telegram and prompt text; authorization header with bearer token.  
    - *Input:* Prompt text and original image URL.  
    - *Output:* Returns prediction job ID for polling.  
    - *Failures:* API key invalid, rate limits, invalid image URL.

  - **Wait 65s1**  
    - *Type:* Wait node  
    - *Role:* Waits 65 seconds to allow image generation to complete before polling.  
    - *Failures:* None typical.

  - **GET STATUS NEW IMG1**  
    - *Type:* HTTP Request  
    - *Role:* Polls WaveSpeed API for image generation status using job ID.  
    - *Configuration:* Authorization header with bearer token; retries every 3 seconds until status `completed`.  
    - *Input:* Job ID from CREATE NEW IMG1.  
    - *Output:* Image generation results with output URLs.  
    - *Failures:* API timeout, failed job, network errors.

  - **If1**  
    - *Type:* If node  
    - *Role:* Checks if image generation status is `completed`.  
    - *Output:*  
      - True: Proceeds with uploading image.  
      - False: Waits again using Wait 65s1 for retry.  
    - *Failures:* Logic errors.

  - **UPLOAD NEW IMG V1**  
    - *Type:* HTTP Request  
    - *Role:* Uploads generated image to Cloudinary for public hosting.  
    - *Configuration:* Multipart form-data POST with file from generation output and upload preset `Picture`; uses Cloudinary credentials.  
    - *Input:* Generated image file.  
    - *Output:* Cloudinary hosted image URL.  
    - *Failures:* Cloudinary auth errors, file upload failures.

  - **Analyze IMG NEW2**  
    - *Type:* LangChain OpenAI node using GPT-4o  
    - *Role:* Analyzes uploaded image to determine character gender (`lelaki` or `perempuan`) for voice selection.  
    - *Input:* Cloudinary image URL.  
    - *Output:* Gender string in lowercase without extra symbols.  
    - *Failures:* Image URL invalid, OpenAI API errors.

---

#### 2.5 Voice Synthesis

- **Overview:**  
  Generates natural-sounding voice-over audio using ElevenLabs text-to-speech API, selecting male or female voice based on gender analysis.

- **Nodes Involved:**  
  - Switch GENDER  
  - CLONING AUDIO CE (Female)  
  - CLONING AUDIO CO (Male)  
  - Extract ElevenLabs Audio Data (Female)  
  - Extract ElevenLabs Audio Data (Male)  
  - Convert Audio to WAV Format (Female Voice)  
  - Convert Audio to WAV Format (Male Voice)  
  - UPLOAD AUDIO LIPSYNC  

- **Node Details:**  

  - **Switch GENDER**  
    - *Type:* Switch  
    - *Role:* Routes workflow based on gender string (`perempuan` or `lelaki`).  
    - *Input:* Gender output from Analyze IMG NEW2.  
    - *Output:* Female branch → CLONING AUDIO CE; Male branch → CLONING AUDIO CO.  
    - *Failures:* Unexpected gender values, routing errors.

  - **CLONING AUDIO CE (Female)**  
    - *Type:* HTTP Request  
    - *Role:* Calls ElevenLabs TTS API to generate female voice audio with `eleven_multilingual_v2` model.  
    - *Configuration:* POST JSON with testimonial text; uses female voice model ID and ElevenLabs API key.  
    - *Input:* Text-to-speech script.  
    - *Output:* Binary audio data response.  
    - *Failures:* Invalid API key, rate limits, input text issues.

  - **CLONING AUDIO CO (Male)**  
    - *Type:* HTTP Request  
    - *Role:* Same as above but for male voice.  
    - *Input/Output:* As above.  
    - *Failures:* Same as above.

  - **Extract ElevenLabs Audio Data (Female & Male)**  
    - *Type:* Extract From File  
    - *Role:* Converts HTTP response binary audio into proper property for further processing.  
    - *Input:* HTTP binary response.  
    - *Output:* Audio binary under `data` key.  
    - *Failures:* Binary parsing errors.

  - **Convert Audio to WAV Format (Female & Male Voices)**  
    - *Type:* Convert To File  
    - *Role:* Converts audio data to WAV format suitable for lip-sync API.  
    - *Input:* Audio binary data.  
    - *Output:* Binary WAV audio.  
    - *Failures:* Conversion errors.

  - **UPLOAD AUDIO LIPSYNC**  
    - *Type:* HTTP Request  
    - *Role:* Uploads WAV audio to Cloudinary for WaveSpeed lip-sync video creation.  
    - *Configuration:* Multipart form-data upload with Cloudinary credentials and preset.  
    - *Input:* WAV audio binary.  
    - *Output:* Cloudinary audio URL.  
    - *Failures:* Authentication errors, upload fails.

---

#### 2.6 Lip-Sync Video Creation & Upload

- **Overview:**  
  Sends uploaded audio and image URLs to WaveSpeed API to generate a lip-sync video with natural character movements, polls video generation status, uploads the video to Cloudinary, and delivers it via Telegram.

- **Nodes Involved:**  
  - CREATE LIPSYNC V1  
  - Wait 140s  
  - GET STATUS LIPSYNC V1  
  - If3  
  - UPLOAD VIDEO REVIEW V1  
  - Send a video  

- **Node Details:**  

  - **CREATE LIPSYNC V1**  
    - *Type:* HTTP Request  
    - *Role:* Initiates lip-sync video creation using WaveSpeed with audio and image URLs, plus a prompt describing natural movements.  
    - *Configuration:* JSON body with audio URL, image URL, resolution 480p, seed -1; authorization header with bearer token.  
    - *Input:* URLs from previous uploads.  
    - *Output:* Job ID for video generation.  
    - *Failures:* API key errors, invalid URLs, request limits.

  - **Wait 140s**  
    - *Type:* Wait node  
    - *Role:* Waits sufficient time (~2 minutes 20 seconds) for video generation before polling.  
    - *Failures:* None typical.

  - **GET STATUS LIPSYNC V1**  
    - *Type:* HTTP Request  
    - *Role:* Polls WaveSpeed API for lip-sync video generation status until `completed`.  
    - *Configuration:* Authorization header with bearer token; retries every 3 seconds.  
    - *Input:* Job ID.  
    - *Output:* Video generation result with output video file.  
    - *Failures:* API downtime, failed generation.

  - **If3**  
    - *Type:* If node  
    - *Role:* Checks completion status; if complete, proceeds to upload video; else waits again.  
    - *Failures:* Conditional logic errors.

  - **UPLOAD VIDEO REVIEW V1**  
    - *Type:* HTTP Request  
    - *Role:* Uploads completed lip-sync video to Cloudinary for hosting.  
    - *Configuration:* Multipart form-data with file and preset `Picture`; uses Cloudinary credentials.  
    - *Input:* Video file from WaveSpeed.  
    - *Output:* Public video URL.  
    - *Failures:* Upload failures, auth issues.

  - **Send a video**  
    - *Type:* Telegram node  
    - *Role:* Sends final testimonial video to Telegram chat ID.  
    - *Input:* Cloudinary video URL.  
    - *Output:* Confirmation of message sent.  
    - *Failures:* Telegram API errors, invalid chat ID.

---

#### 2.7 Delivery via Telegram

- **Overview:**  
  Sends intermediate generated testimonial images and final video testimonial back to the Telegram user/channel.

- **Nodes Involved:**  
  - Send a photo message  
  - Send a video  

- **Node Details:**  

  - **Send a photo message**  
    - *Type:* Telegram node  
    - *Role:* Sends generated testimonial image to Telegram chat.  
    - *Input:* Image URL from Cloudinary upload.  
    - *Output:* Telegram message confirmation.  
    - *Failures:* Invalid chat ID, API errors.

  - **Send a video**  
    - *Type:* Telegram node  
    - *Role:* Sends final testimonial video to Telegram chat.  
    - *Input:* Video URL from Cloudinary upload.  
    - *Output:* Telegram message confirmation.  
    - *Failures:* Same as above.

---

#### 2.8 Workflow Orchestration & Waits

- **Overview:**  
  Manages timing and conditional logic to handle asynchronous API calls, response polling, and retries.

- **Nodes Involved:**  
  - Wait  
  - Wait5  
  - Wait 65s1  
  - Wait 140s  
  - If1  
  - If3  
  - If4  
  - If5  

- **Node Details:**  

  - **Wait nodes**  
    - Used to delay workflow to accommodate API processing times (65s for image generation, 140s for video).  
    - Prevents premature polling and rate-limit issues.

  - **If nodes**  
    - Validate outputs (e.g., prompt formatting, generation status).  
    - Direct workflow based on success/failure or content conditions.  
    - Loop retries or proceed with next steps accordingly.  
    - Critical for reliability and error handling.

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                                       | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                                             |
|-------------------------------|----------------------------------|-----------------------------------------------------|------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger              | Telegram Trigger                 | Entry: Receive product images via Telegram           | -                            | Set Bot Token                  |                                                                                                                                         |
| Set Bot Token                | Set                             | Store Telegram bot token                              | Telegram Trigger             | Image Path (getFile)           | Replace 'YOUR_TELEGRAM_BOT_TOKEN' with your bot token.                                                                                  |
| Image Path (getFile)         | HTTP Request                    | Retrieve Telegram file path                           | Set Bot Token                | Download File → Gambar         |                                                                                                                                         |
| Download File → Gambar       | HTTP Request                    | Download image file from Telegram                     | Image Path (getFile)         | Analyze YAML                  |                                                                                                                                         |
| Analyze YAML                | LangChain OpenAI (GPT-4o)        | Analyze image for product/character attributes       | Download File → Gambar       | Wait                          | Requires OpenAI API key; use GPT-4o with image analysis enabled.                                                                        |
| Wait                        | Wait                           | Pause for readiness                                   | Analyze YAML                | PROMPT PRODUK REVIEW           |                                                                                                                                         |
| PROMPT PRODUK REVIEW        | LangChain LLM Chain (GPT-4o)     | Generate detailed image prompt for testimonial       | Wait                        | Wait5                         |                                                                                                                                         |
| Wait5                       | Wait                           | Short wait before conditional checks                  | PROMPT PRODUK REVIEW         | If4                           |                                                                                                                                         |
| If4                         | If                             | Check prompt text for invalid double quotes           | Wait5                       | TTS & CAPTION1 (true), PROMPT PRODUK REVIEW (false) |                                                                                                                                         |
| TTS & CAPTION1              | LangChain Agent                 | Generate testimonial text-to-speech script            | If4                         | If5                           |                                                                                                                                         |
| If5                         | If                             | Validate caption text for invalid double quotes       | TTS & CAPTION1              | CREATE NEW IMG1 (true), TTS & CAPTION1 (false) |                                                                                                                                         |
| CREATE NEW IMG1             | HTTP Request                    | Request WaveSpeed AI to generate testimonial image    | If5                         | Wait 65s1                    | Requires WaveSpeed API key.                                                                                                             |
| Wait 65s1                   | Wait                           | Wait for image generation processing                   | CREATE NEW IMG1             | GET STATUS NEW IMG1            |                                                                                                                                         |
| GET STATUS NEW IMG1         | HTTP Request                    | Poll WaveSpeed API for image generation status        | Wait 65s1                   | If1                           |                                                                                                                                         |
| If1                         | If                             | Check if image generation is completed                 | GET STATUS NEW IMG1          | UPLOAD NEW IMG V1 (true), Wait 65s1 (false) |                                                                                                                                         |
| UPLOAD NEW IMG V1           | HTTP Request                    | Upload generated image to Cloudinary                   | If1                         | Analyze IMG NEW2               | Requires Cloudinary API key and upload preset 'Picture'.                                                                               |
| Analyze IMG NEW2            | LangChain OpenAI (GPT-4o)        | Analyze image to determine character gender            | UPLOAD NEW IMG V1            | Switch GENDER                  |                                                                                                                                         |
| Switch GENDER               | Switch                         | Route workflow based on gender                          | Analyze IMG NEW2             | CLONING AUDIO CE (female), CLONING AUDIO CO (male) |                                                                                                                                         |
| CLONING AUDIO CE            | HTTP Request                    | ElevenLabs TTS API call for female voice               | Switch GENDER (female)       | Extract ElevenLabs Audio Data  | Replace 'YOUR_ELEVENLABS_API_KEY' with valid key.                                                                                       |
| CLONING AUDIO CO            | HTTP Request                    | ElevenLabs TTS API call for male voice                 | Switch GENDER (male)         | Extract ElevenLabs Audio Data (Male) |                                                                                                                                         |
| Extract ElevenLabs Audio Data | Extract From File              | Extract audio binary data from ElevenLabs response     | CLONING AUDIO CE             | Convert Audio to WAV Format (Female Voice) |                                                                                                                                         |
| Extract ElevenLabs Audio Data (Male) | Extract From File         | Extract audio binary data from ElevenLabs response     | CLONING AUDIO CO             | Convert Audio to WAV Format (Male Voice) |                                                                                                                                         |
| Convert Audio to WAV Format (Female Voice) | Convert To File        | Convert ElevenLabs audio to WAV format                  | Extract ElevenLabs Audio Data | UPLOAD AUDIO LIPSYNC           |                                                                                                                                         |
| Convert Audio to WAV Format (Male Voice) | Convert To File          | Convert ElevenLabs audio to WAV format                  | Extract ElevenLabs Audio Data (Male) | UPLOAD AUDIO LIPSYNC           |                                                                                                                                         |
| UPLOAD AUDIO LIPSYNC        | HTTP Request                    | Upload WAV audio to Cloudinary for lip-sync             | Convert Audio to WAV Format  | CREATE LIPSYNC V1              | Requires Cloudinary credentials and upload preset 'Picture'.                                                                           |
| CREATE LIPSYNC V1           | HTTP Request                    | Request WaveSpeed API to create lip-sync video          | UPLOAD AUDIO LIPSYNC         | Wait 140s                     | Requires WaveSpeed API key.                                                                                                             |
| Wait 140s                   | Wait                           | Wait for lip-sync video generation                       | CREATE LIPSYNC V1            | GET STATUS LIPSYNC V1          |                                                                                                                                         |
| GET STATUS LIPSYNC V1       | HTTP Request                    | Poll WaveSpeed API for lip-sync video generation status | Wait 140s                   | If3                           |                                                                                                                                         |
| If3                         | If                             | Check lip-sync video generation completion              | GET STATUS LIPSYNC V1        | UPLOAD VIDEO REVIEW V1 (true), Wait 140s (false) |                                                                                                                                         |
| UPLOAD VIDEO REVIEW V1      | HTTP Request                    | Upload generated video to Cloudinary                     | If3                         | Send a video                  | Requires Cloudinary credentials and upload preset 'Picture'.                                                                           |
| Send a photo message        | Telegram                        | Send testimonial image to Telegram chat                  | UPLOAD NEW IMG V1            | -                             |                                                                                                                                         |
| Send a video                | Telegram                        | Send final testimonial video to Telegram chat            | UPLOAD VIDEO REVIEW V1       | -                             |                                                                                                                                         |
| Sticky Note7                | Sticky Note                    | Author contact and expertise information                  | -                            | -                             | Muhammad Farooq Iqbal contact details and portfolio.                                                                                   |
| Sticky Note6                | Sticky Note                    | Setup & integration instructions                          | -                            | -                             | Includes all required API keys and setup links for OpenAI, ElevenLabs, WaveSpeed, Cloudinary, and Telegram Bot.                       |
| Sticky Note10               | Sticky Note                    | Workflow summary and tech stack overview                  | -                            | -                             | Summary of workflow purpose, tech used, and expected output.                                                                           |
| Sticky Note13               | Sticky Note                    | Ethical AI content guidelines                              | -                            | -                             | Responsible AI use, compliance checklist, and platform-specific advertising requirements.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates with photos.  
   - Add Telegram API credentials with your bot token.  

2. **Create Set node "Set Bot Token":**  
   - Type: Set  
   - Assign a string variable `bot id` with your Telegram bot token value.

3. **Add HTTP Request node "Image Path (getFile)":**  
   - Method: GET  
   - URL: `https://api.telegram.org/bot{{ $json["bot id"] }}/getFile?file_id={{ $('Telegram Trigger').item.json.message.photo[0].file_id }}`  
   - No authentication needed.  

4. **Add HTTP Request node "Download File → Gambar":**  
   - Method: GET  
   - URL: `https://api.telegram.org/file/bot{{ $('Set Bot Token').item.json['bot id'] }}/{{ $json.result.file_path }}`  
   - Set response to binary data to download image file.

5. **Add LangChain OpenAI node "Analyze YAML":**  
   - Model: GPT-4o (chatgpt-4o-latest) with image analysis enabled.  
   - Input: Image URL from "Download File → Gambar".  
   - Prompt: Analyze image to detect product/character features and output YAML.  
   - Add OpenAI credentials.

6. **Add Wait node:**  
   - Default wait time to allow for processing.

7. **Add LangChain LLM Chain node "PROMPT PRODUK REVIEW":**  
   - Model: GPT-4o  
   - Prompt: Generate a detailed image prompt for testimonial photo per described instructions.  
   - Input: Product description from Analyze YAML.  
   - Add OpenAI credentials.

8. **Add Wait node "Wait5":**  
   - Short delay before conditional checks.

9. **Add If node "If4":**  
   - Condition: Check if generated prompt contains double quotes (`"`)  
   - True: Proceed to "TTS & CAPTION1"  
   - False: Loop back to "PROMPT PRODUK REVIEW".

10. **Add LangChain Agent node "TTS & CAPTION1":**  
    - Model: GPT-4o  
    - System prompt: Social media manager generating natural Malay-English testimonial speech.  
    - Input: Product description YAML.  
    - Output: Text-to-speech script (`textospeech`).  
    - Add OpenAI credentials.

11. **Add If node "If5":**  
    - Condition: Check if caption text contains double quotes  
    - True: Proceed to image generation  
    - False: Loop back to "TTS & CAPTION1".

12. **Add HTTP Request node "CREATE NEW IMG1":**  
    - Method: POST  
    - URL: WaveSpeed API endpoint for image generation  
    - Body: JSON with aspect ratio 9:16, original image URL, and prompt text from "PROMPT PRODUK REVIEW".  
    - Headers: Authorization Bearer with WaveSpeed API key.  

13. **Add Wait node "Wait 65s1":**  
    - Wait 65 seconds for image generation.

14. **Add HTTP Request node "GET STATUS NEW IMG1":**  
    - Method: GET  
    - URL: WaveSpeed API status endpoint with prediction job ID.  
    - Headers: Authorization Bearer.  
    - Retry every 3 seconds until status is "completed".

15. **Add If node "If1":**  
    - Condition: Check if status is "completed"  
    - True: Proceed to "UPLOAD NEW IMG V1"  
    - False: Loop back to "Wait 65s1".

16. **Add HTTP Request node "UPLOAD NEW IMG V1":**  
    - Method: POST multipart/form-data  
    - URL: Cloudinary upload endpoint  
    - Body: Upload generated image file and set upload preset to "Picture".  
    - Add Cloudinary credentials.

17. **Add LangChain OpenAI node "Analyze IMG NEW2":**  
    - Model: GPT-4o  
    - Input: Cloudinary image URL  
    - Prompt: Analyze image gender, output `lelaki` or `perempuan`.  
    - Add OpenAI credentials.

18. **Add Switch node "Switch GENDER":**  
    - Condition: Check gender string from "Analyze IMG NEW2"  
    - Outputs: "perempuan" to female voice branch, "lelaki" to male voice branch.

19. **Female Voice Branch:**  
    a. HTTP Request node "CLONING AUDIO CE": POST ElevenLabs TTS API with female voice ID, text from TTS script.  
    b. Extract From File node "Extract ElevenLabs Audio Data": Extract binary audio.  
    c. Convert To File node "Convert Audio to WAV Format (Female Voice)": Convert extracted audio to WAV.  

20. **Male Voice Branch:**  
    a. HTTP Request node "CLONING AUDIO CO": POST ElevenLabs TTS API with male voice ID.  
    b. Extract From File node "Extract ElevenLabs Audio Data (Male)": Extract binary audio.  
    c. Convert To File node "Convert Audio to WAV Format (Male Voice)": Convert to WAV.

21. **Merge branches to single node "UPLOAD AUDIO LIPSYNC":**  
    - Upload WAV audio to Cloudinary with preset "Picture".

22. **Add HTTP Request node "CREATE LIPSYNC V1":**  
    - Method: POST  
    - URL: WaveSpeed lip-sync video creation endpoint  
    - Body: JSON with audio URL, image URL, prompt for natural character motion, resolution 480p.  
    - Headers: Authorization Bearer.

23. **Add Wait node "Wait 140s":**  
    - Wait 140 seconds for video generation.

24. **Add HTTP Request node "GET STATUS LIPSYNC V1":**  
    - Poll WaveSpeed API for video generation status until completed.

25. **Add If node "If3":**  
    - Condition: Check if video generation status is completed  
    - True: Proceed to upload video  
    - False: Loop back to Wait 140s.

26. **Add HTTP Request node "UPLOAD VIDEO REVIEW V1":**  
    - Method: POST multipart/form-data  
    - Upload generated video to Cloudinary with preset "Picture".

27. **Add Telegram node "Send a photo message":**  
    - Send generated testimonial image to Telegram chat ID.

28. **Add Telegram node "Send a video":**  
    - Send final testimonial video to Telegram chat ID.

29. **Add Sticky Notes:**  
    - Include setup instructions, contact info, ethical guidelines, and workflow summary for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Muhammad Farooq Iqbal is the workflow creator and automation expert. Contact via email mfarooqiqbal143@gmail.com or LinkedIn https://linkedin.com/in/muhammadfarooqiqbal for support or customization.                                 | Sticky Note7                                                                                                     |
| Setup Guide: Requires API keys for OpenAI (for GPT-4o with image analysis), ElevenLabs (voice cloning), WaveSpeed AI (image and video generation), Cloudinary (file hosting), and Telegram Bot token. Setup upload presets accordingly. | Sticky Note6                                                                                                     |
| Workflow creates authentic UGC video ads combining AI-generated testimonial scripts, images, voice synthesis, and lip-sync videos delivered via Telegram. Processing time approx. 3-5 minutes per video.                               | Sticky Note10                                                                                                    |
| Ethical AI Content Generation Guidelines: Clearly disclose synthetic content, respect platform policies, avoid misleading claims, and comply with local ad regulations. Consult legal counsel if unsure.                             | Sticky Note13                                                                                                    |

---

**Disclaimer:** The provided content is derived solely from an n8n automated workflow. It respects all relevant content policies and contains no illegal or offensive elements. All processed data is legal and publicly accessible.