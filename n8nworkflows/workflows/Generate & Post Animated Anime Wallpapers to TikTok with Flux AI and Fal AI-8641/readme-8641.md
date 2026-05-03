Generate & Post Animated Anime Wallpapers to TikTok with Flux AI and Fal AI

https://n8nworkflows.xyz/workflows/generate---post-animated-anime-wallpapers-to-tiktok-with-flux-ai-and-fal-ai-8641


# Generate & Post Animated Anime Wallpapers to TikTok with Flux AI and Fal AI

### 1. Workflow Overview

This workflow automates the generation and posting of animated anime wallpapers to TikTok using a combination of AI models and APIs. It accepts user input via a form, generates a detailed anime-style image prompt with an AI language model, uses an AI image generator to create a wallpaper, converts that wallpaper into an animated video, then posts the video to TikTok automatically. The workflow is structured into three main logical blocks:

- **1.1 Input Reception:** Collects user input on the wallpaper topic and anime style via an n8n form.
- **1.2 Image and Video Generation:** Creates a text prompt, generates the anime wallpaper image, uploads it, converts the image to a looping animated video, and polls for video generation status.
- **1.3 TikTok Posting:** Generates relevant hashtags and posts the final video to TikTok using the GetLate API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures user input about the wallpaper topic and preferred anime style through a web form, triggering the workflow to start.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point, listens for user submissions on a custom n8n form.  
  - Configuration: Form titled "Wallpaper Poster" with two fields: "Topic" (mandatory text input) and "Style Anime" (dropdown with predefined anime artist options such as Studio Ghibli, Hayao Miyazaki, etc.).  
  - Input: User fills out the form.  
  - Output: Passes form data downstream for prompt generation.  
  - Edge Cases: Missing required fields; malformed form data; webhook issues.  
  - Sub-workflow: None.

---

#### 2.2 Image and Video Generation

**Overview:**  
This block generates a detailed anime wallpaper prompt, produces the wallpaper image, uploads it, converts it into an animated video, and monitors video generation status until completion.

**Nodes Involved:**  
- Groq Chat Model  
- Prompt Generator  
- Set URL  
- Generate Image  
- Upload IMG  
- Create Video  
- Wait  
- Get Status  
- If  
- Get Video

**Node Details:**  

- **Groq Chat Model**  
  - Type: AI Language Model (Groq GPT-OSS 120B)  
  - Role: Generates AI prompts using Groq API.  
  - Configuration: Uses Groq API credentials; model set to "openai/gpt-oss-120b".  
  - Input: Receives form data (Topic and Style Anime) for prompt generation and tag generation.  
  - Output: Sends prompt text to the Prompt Generator and Tag Generator nodes.  
  - Edge Cases: API authentication failure; model errors; rate limits.

- **Prompt Generator**  
  - Type: LangChain LLM Chain  
  - Role: Produces a rich text-to-image prompt for anime wallpaper based on input topic and style.  
  - Configuration: Custom prompt instructing the AI to generate detailed, stylistically accurate, single-line anime wallpaper prompts without extra formatting.  
  - Input: Receives topic and style from form submission and Groq Chat Model.  
  - Output: Passes generated prompt text downstream to the Set URL node.  
  - Edge Cases: Expression evaluation failure; incomplete or ambiguous prompt generation.

- **Set URL**  
  - Type: Set Node  
  - Role: Constructs the image generation API URL by inserting the generated prompt into the Pollinations AI Flux model endpoint.  
  - Configuration: Creates a string variable `prompt_url` formatted as:  
    `https://image.pollinations.ai/prompt/{{ prompt_text }}.jpg?width=720&height=1280&model=flux&nologo=true`  
  - Input: Receives prompt text from Prompt Generator.  
  - Output: Passes the URL to Generate Image node.  
  - Edge Cases: URL encoding issues; missing prompt text.

- **Generate Image**  
  - Type: HTTP Request  
  - Role: Calls Pollinations AI Flux model API to generate the anime wallpaper image.  
  - Configuration: GET request to the URL from Set URL node.  
  - Input: Receives constructed URL.  
  - Output: Image data passed to Upload IMG node.  
  - Edge Cases: Request timeouts; API errors; invalid image URL.

- **Upload IMG**  
  - Type: HTTP Request  
  - Role: Uploads generated image to GetLate media storage for later use in video creation and posting.  
  - Configuration: POST multipart-form-data with image binary sent to `https://getlate.dev/api/v1/media`. Uses HTTP Bearer authentication with GetLate API key.  
  - Input: Receives binary image data from Generate Image.  
  - Output: Provides uploaded media URL to Create Video node.  
  - Edge Cases: Authentication failure; upload errors; file size limits.

- **Create Video**  
  - Type: HTTP Request  
  - Role: Initiates video creation on Fal AI using the uploaded image URL and a prompt describing looping, ambient animation effects suitable for a calm, dreamy anime wallpaper.  
  - Configuration: POST JSON body to Fal AI endpoint `https://queue.fal.run/fal-ai/minimax/hailuo-02-fast/image-to-video`. Authenticated with Fal AI API keys.  
  - Input: Uses uploaded image URL from Upload IMG node.  
  - Output: Returns a status URL to poll video generation progress.  
  - Edge Cases: API quota exhaustion; malformed JSON; network errors.

- **Wait**  
  - Type: Wait Node  
  - Role: Pauses workflow for 10 seconds between polling attempts for video status.  
  - Configuration: Fixed 10-second wait time.  
  - Input: Triggered after Create Video or after polling loop.  
  - Output: Triggers Get Status node to check video generation progress.  
  - Edge Cases: Excessive delays; workflow timeouts.

- **Get Status**  
  - Type: HTTP Request  
  - Role: Polls the Fal AI API for the current status of video generation using the status URL.  
  - Configuration: GET request authenticated with Fal AI credentials.  
  - Input: Receives status URL from previous Create Video or Wait node.  
  - Output: Provides status JSON to If node.  
  - Edge Cases: Status URL expiration; API errors.

- **If**  
  - Type: Conditional Node  
  - Role: Checks if video generation status equals "COMPLETED" to proceed or wait more.  
  - Configuration: Condition on JSON field `status == "COMPLETED"`.  
  - Input: Status data from Get Status node.  
  - Output: If true, proceeds to Get Video node; otherwise loops back to Wait node for another polling cycle.  
  - Edge Cases: Unexpected status values; infinite loop risk if status never completes.

- **Get Video**  
  - Type: HTTP Request  
  - Role: Retrieves the final animated video URL from Fal AI once processing is complete.  
  - Configuration: GET request authenticated with Fal AI credentials to the video response URL from Create Video node.  
  - Input: Triggered when video generation completes.  
  - Output: Passes final video URL downstream to Tag Generator.  
  - Edge Cases: Video not found; expired URLs.

---

#### 2.3 TikTok Posting

**Overview:**  
This block generates relevant hashtags for the video post and publishes the animated wallpaper video on TikTok using the GetLate API.

**Nodes Involved:**  
- Tag Generator  
- Tiktok Post

**Node Details:**  

- **Tag Generator**  
  - Type: LangChain LLM Chain  
  - Role: Generates exactly five related hashtags based on the wallpaper topic for use in TikTok post content.  
  - Configuration: Prompt instructs AI to produce 5 related Twitter-style hashtags in a single comma-separated line, no extra text.  
  - Input: Receives topic from form submission.  
  - Output: Passes hashtag string to Tiktok Post node.  
  - Edge Cases: Inappropriate or unrelated hashtags; empty output.

- **Tiktok Post**  
  - Type: HTTP Request  
  - Role: Publishes the generated video to TikTok via the GetLate API using the uploaded video URL and hashtags.  
  - Configuration: POST JSON to `https://getlate.dev/api/v1/posts`. Includes content with generated hashtags plus static text "Epic Anime Wallpaper". Posts to TikTok with privacy and consent settings configured. Authenticated with GetLate API credentials.  
  - Input: Receives hashtags and video URL.  
  - Output: Final step; posts video publicly on TikTok.  
  - Edge Cases: Authentication failure; invalid video URL; API rate limits.

---

### 3. Summary Table

| Node Name          | Node Type                | Functional Role                        | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                       |
|--------------------|--------------------------|-------------------------------------|-----------------------|-----------------------|-------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger             | Receives user input to start workflow| None                  | Prompt Generator      | ## START HERE                                                                                   |
| Groq Chat Model     | AI Language Model (Groq) | Generates prompt and hashtags inputs | On form submission    | Prompt Generator, Tag Generator |                                                                                                 |
| Prompt Generator    | LangChain LLM Chain      | Creates detailed anime wallpaper prompt | On form submission, Groq Chat Model | Set URL              | ## 1. IMAGE GENERATOR                                                                           |
| Set URL            | Set Node                 | Builds image generation API URL      | Prompt Generator      | Generate Image        |                                                                                                 |
| Generate Image      | HTTP Request             | Calls AI image generation API        | Set URL               | Upload IMG            |                                                                                                 |
| Upload IMG          | HTTP Request             | Uploads image to GetLate media       | Generate Image        | Create Video          |                                                                                                 |
| Create Video        | HTTP Request             | Requests animated video creation     | Upload IMG            | Wait                  | ## 2. ANIMATED VIDEO GENERATOR                                                                  |
| Wait                | Wait Node                | Waits before polling video status    | Create Video, If (false) | Get Status           |                                                                                                 |
| Get Status          | HTTP Request             | Polls video generation status        | Wait                  | If                    |                                                                                                 |
| If                  | Conditional Node         | Checks if video generation completed | Get Status            | Get Video (true), Wait (false) |                                                                                                 |
| Get Video           | HTTP Request             | Retrieves final video URL             | If                    | Tag Generator         |                                                                                                 |
| Tag Generator       | LangChain LLM Chain      | Generates related hashtags            | Get Video, Groq Chat Model | Tiktok Post          | ## 3. POST TO TIKTOK                                                                            |
| Tiktok Post         | HTTP Request             | Posts video to TikTok via GetLate API | Tag Generator         | None                  |                                                                                                 |
| Sticky Note         | Sticky Note              | Visual section marker                 | None                  | None                  | ## 2. ANIMATED VIDEO GENERATOR                                                                  |
| Sticky Note1        | Sticky Note              | Visual section marker                 | None                  | None                  | ## 1. IMAGE GENERATOR                                                                           |
| Sticky Note2        | Sticky Note              | Visual section marker                 | None                  | None                  | ## 3. POST TO TIKTOK                                                                            |
| Sticky Note3        | Sticky Note              | Visual section marker                 | None                  | None                  | ## START HERE                                                                                   |
| Sticky Note4        | Sticky Note              | Workflow description and overview    | None                  | None                  | ## Animated Anime Wallpaper TikTok Post  \n### This n8n template demonstrates how to generate animated anime wallpapers and automatically post them on TikTok.  \n\nBy default, the workflow creates an anime-themed image based on user input, converts it into an animated video, and then automatically publishes it to TikTok.  \n\n**Possible Customizations:**  \n- Replace the default **Form Trigger** with a **Scheduled Trigger**.  \n- Connect a topics database (e.g., Google Sheets or Airtable) to automatically generate and post animated anime wallpapers on TikTok at regular intervals.  \n\n### How It Works  \n1. The user opens an n8n form and enters the desired anime wallpaper topic and style.  \n2. Based on the input, **OpenAI GPT-OSS (via Groq)** generates a text-to-image prompt.  \n3. The **Flux AI model on Pollination AI** generates an anime wallpaper image from the prompt.  \n4. Using the generated image, the **Minimax Hailuo 02 Fast model on Fal AI** creates an animated video.  \n5. The final video is automatically published to TikTok via the **GetLate API**.  \n\n### Requirements  \n- Groq API Key  \n- Fal AI API Key  \n- GetLate API connected to your TikTok account  \n|
| Sticky Note5        | Sticky Note              | Setup and usage instructions          | None                  | None                  | ## How to Use  \n\n1. Open the **Groq Chat Model** node, add your **Groq API Key**, and select an LLM model.  \n   - By default, this template uses **OpenAI GPT-OSS 120B**.  \n2. Open the **n8n Form** using either the **Test URL** or the **Production URL**.  \n3. Get your API key from [getlate.dev](https://getlate.dev/) and add the credentials in both the **Upload IMG** node and the **TikTok Post** node.  \n4. Get your API key from [Fal.ai](https://fal.ai/dashboard/keys) and make sure to top up credits.  \n5. Add your **Fal AI credentials** to the following nodes: **Create Video**, **Get Status**, and **Get Video**.  \n6. Once everything is set up, copy your **n8n Form URL** and open it in your browser to start using the workflow.  \n|
| Sticky Note6        | Sticky Note              | Documentation links                   | None                  | None                  | ## Documentation\n1. [Minimax Hailuo API on Fal](https://fal.ai/models/fal-ai/minimax/hailuo-02-fast/image-to-video/api)\n2. [Pollination AI](https://github.com/pollinations/pollinations)\n3. [GetLate.dev](https://getlate.dev/docs)\n4. [Groq API](https://console.groq.com/docs/quickstart) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "On form submission"  
   - Configure form titled "Wallpaper Poster" with fields:  
     - "Topic" (Text, Required)  
     - "Style Anime" (Dropdown) with options: Studio Ghibli, Hayao Miyazaki, Makoto Shinkai, Yoshitaka Amano, Akira Toriyama  
   - This node is the workflow entry point.

2. **Add Groq Chat Model Node**  
   - Type: AI Language Model (Groq Chat)  
   - Name: "Groq Chat Model"  
   - Configure credentials with your Groq API key.  
   - Model: "openai/gpt-oss-120b"  
   - Connect input from "On form submission" node.  
   - This node will feed prompt generation and tag generation nodes.

3. **Add Prompt Generator Node**  
   - Type: LangChain LLM Chain  
   - Name: "Prompt Generator"  
   - Configure prompt to generate detailed anime wallpaper prompt based on inputs: topic, anime style.  
   - Connect input from "On form submission" and "Groq Chat Model".  
   - Connect output to "Set URL" node.

4. **Add Set Node**  
   - Name: "Set URL"  
   - Purpose: Compose image generation URL for Pollinations AI Flux model.  
   - Create a string variable "prompt_url" with expression:  
     `https://image.pollinations.ai/prompt/{{ $('Prompt Generator').item.json.text }}.jpg?width=720&height=1280&model=flux&nologo=true`  
   - Connect input from "Prompt Generator".  
   - Connect output to "Generate Image" node.

5. **Add Generate Image Node**  
   - Type: HTTP Request (GET)  
   - Name: "Generate Image"  
   - URL: Use expression `{{$json.prompt_url}}` from "Set URL" node.  
   - Connect input from "Set URL".  
   - Connect output to "Upload IMG" node.

6. **Add Upload IMG Node**  
   - Type: HTTP Request (POST)  
   - Name: "Upload IMG"  
   - URL: `https://getlate.dev/api/v1/media`  
   - Content-Type: multipart-form-data  
   - Body: Send binary image data from "Generate Image" node output.  
   - Authentication: HTTP Bearer with GetLate API key (credential setup required).  
   - Connect input from "Generate Image".  
   - Connect output to "Create Video" node.

7. **Add Create Video Node**  
   - Type: HTTP Request (POST)  
   - Name: "Create Video"  
   - URL: `https://queue.fal.run/fal-ai/minimax/hailuo-02-fast/image-to-video`  
   - Body (JSON):  
     ```json
     {
       "prompt": "seamless looping video with gentle, ambient motion. Add slow, continuous effects such as drifting particles, subtle light flicker, soft haze, floating dust, rippling water, moving clouds, or faint color shifts. The motion should be cyclical and loopable, with no clear start or end, maintaining a calm and dreamy Lofi vibe suitable for a background visual. Camera movement is static.",
       "image_url": "{{ $json.files[0].url }}",
       "prompt_optimizer": false
     }
     ```  
   - Authentication: HTTP Header Auth with Fal AI API key (credential setup required).  
   - Connect input from "Upload IMG".  
   - Connect output to "Wait" node.

8. **Add Wait Node**  
   - Type: Wait  
   - Name: "Wait"  
   - Duration: 10 seconds  
   - Connect input from "Create Video" and from "If" node (for polling).  
   - Connect output to "Get Status" node.

9. **Add Get Status Node**  
   - Type: HTTP Request (GET)  
   - Name: "Get Status"  
   - URL: Use status URL from "Create Video" or previous status check.  
   - Authentication: HTTP Header Auth with Fal AI API key.  
   - Connect input from "Wait".  
   - Connect output to "If" node.

10. **Add If Node**  
    - Type: Conditional  
    - Name: "If"  
    - Condition: Check if JSON field `status` equals "COMPLETED".  
    - If true, connect to "Get Video" node.  
    - If false, connect back to "Wait" node to continue polling.

11. **Add Get Video Node**  
    - Type: HTTP Request (GET)  
    - Name: "Get Video"  
    - URL: Use response URL from "Create Video" node.  
    - Authentication: HTTP Header Auth with Fal AI API key.  
    - Connect input from "If" node (true branch).  
    - Connect output to "Tag Generator" node.

12. **Add Tag Generator Node**  
    - Type: LangChain LLM Chain  
    - Name: "Tag Generator"  
    - Prompt: Generate exactly 5 related hashtags for the wallpaper topic, formatted as comma-separated Twitter hashtags.  
    - Connect input from "Get Video" and "Groq Chat Model".  
    - Connect output to "Tiktok Post" node.

13. **Add Tiktok Post Node**  
    - Type: HTTP Request (POST)  
    - Name: "Tiktok Post"  
    - URL: `https://getlate.dev/api/v1/posts`  
    - Body (JSON):  
      ```json
      {
        "content": "{{ $('Tag Generator').item.json.text }} Epic Anime Wallpaper",
        "publishNow": true,
        "platforms": [
          {
            "platform": "tiktok",
            "accountId": "xxxxxxxxxxxxxxxxxxxxx",
            "platformSpecificData": {
              "tiktokSettings": {
                "privacy_level": "PUBLIC_TO_EVERYONE",
                "video_made_with_ai": true,
                "allow_comment": true,
                "allow_duet": false,
                "allow_stitch": false,
                "commercial_content_type": "none",
                "content_preview_confirmed": true,
                "express_consent_given": true
              }
            }
          }
        ],
        "mediaItems": [
          { "type": "video", "url": "{{ $('Get Video').first().json.video.url }}" }
        ]
      }
      ```  
    - Authentication: HTTP Bearer with GetLate API key.  
    - Connect input from "Tag Generator".  
    - This is the final node.

14. **Add Sticky Notes** for visual clarity and documentation:  
    - Mark sections: "START HERE", "1. IMAGE GENERATOR", "2. ANIMATED VIDEO GENERATOR", "3. POST TO TIKTOK"  
    - Include workflow description, usage instructions, and documentation links as separate sticky notes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Animated Anime Wallpaper TikTok Post: This workflow generates anime-themed images, creates looping animated videos, and posts them to TikTok automatically. Customizations include replacing the form trigger with scheduled triggers or integrating with databases like Google Sheets or Airtable for automated batch posting.                                                                                                                                                                                         | Workflow description (Sticky Note4)                                                                 |
| How to Use: Configure Groq API key and select model in "Groq Chat Model" node; set GetLate API keys in "Upload IMG" and "Tiktok Post" nodes; add Fal AI credentials in video creation and status nodes; obtain n8n form URL to submit topics.                                                                                                                                                                                                                                                                | Usage instructions (Sticky Note5)                                                                   |
| Documentation Links: 1. [Minimax Hailuo API on Fal](https://fal.ai/models/fal-ai/minimax/hailuo-02-fast/image-to-video/api) 2. [Pollination AI GitHub](https://github.com/pollinations/pollinations) 3. [GetLate.dev Docs](https://getlate.dev/docs) 4. [Groq API Quickstart](https://console.groq.com/docs/quickstart)                                                                                                                                                                                             | Official API and service documentation (Sticky Note6)                                              |
| The prompt generation explicitly instructs the AI to produce vivid, detailed, single-line prompts for anime wallpapers, emphasizing style, mood, and atmosphere, avoiding generic phrases, and excluding image size or aspect ratios.                                                                                                                                                                                                                                                                          | Prompt design best practices                                                                        |
| The Fal AI video creation uses a specific model "minimax/hailuo-02-fast" to generate seamless looping videos with ambient effects, creating a calm, dreamy Lofi vibe suitable for animated wallpapers.                                                                                                                                                                                                                                                                                                      | Video animation model details                                                                       |
| GetLate API is used both for media upload and for posting to TikTok, requiring API keys and account configuration. The TikTok post is configured to be publicly visible, with AI video attribution, and disables duet and stitch features to control content usage.                                                                                                                                                                                                                                       | TikTok posting policy and API usage                                                                |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.