Visual Storytelling Content Factory: Gemini & Replicate AI with Human-in-the-loop Publishing

https://n8nworkflows.xyz/workflows/visual-storytelling-content-factory--gemini---replicate-ai-with-human-in-the-loop-publishing-7951


# Visual Storytelling Content Factory: Gemini & Replicate AI with Human-in-the-loop Publishing

### 1. Workflow Overview

This workflow automates the creation, approval, and publishing process of a visual storytelling content series titled **"A Day in the Life"** featuring a serene skeleton character. It leverages AI models (Google Gemini via OpenRouter and Replicate) for narrative generation, image and video creation, human-in-the-loop approval via Slack, and multi-platform social media publishing using Blotato.

The workflow is structured into the following logical blocks:

- **1.1 Story Generation:** Generates a detailed narrative of scenes describing the skeleton’s day.
- **1.2 AI Prompt Conversion:** Converts the narrative scenes into text-to-image and text-to-video prompts.
- **1.3 Image Generation & Approval:** Produces images from prompts, requests approval on Slack, and handles regeneration if declined.
- **1.4 Video Generation & Approval:** Creates videos from generated images and video prompts, requests approval similarly.
- **1.5 Media Processing:** Resizes images and prepares media for publishing.
- **1.6 Social Media Publishing:** Posts approved content to Instagram, Facebook, and TikTok.
- **1.7 Control & Utilities:** Includes manual trigger, optional scheduling, and helper nodes for flow control and debugging.

---

### 2. Block-by-Block Analysis

#### 2.1 Story Generation

- **Overview:** This block generates the creative narrative that defines the day's story of the serene skeleton character. It uses a language model to produce a JSON array of detailed scene descriptions aligned with the project’s theme and mood.

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`  
  - `Creative Director`  
  - `OpenRouter Chat Model` (AI backend for Creative Director)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point to manually start the workflow  
    - *Configuration:* No parameters; triggers downstream nodes when user executes  
    - *Inputs:* None  
    - *Outputs:* Connected to `Creative Director`  
    - *Edge Cases:* Manual trigger requires user action; no errors expected here  

  - **Creative Director**  
    - *Type:* LangChain LLM Chain node  
    - *Role:* Generates a detailed story outline as JSON scenes  
    - *Configuration:*  
      - Prompt instructs to create 5-8 unique scenes for “A Day in the Life” of a serene skeleton, with mood, setting, and narrative arc details  
      - Includes dynamic date insertion with `{{$now.format('dd LLLL')}}` to customize story per day  
      - Output is a structured JSON array of scene objects with keys like `scene_number`, `scene_description`, `time_of_day`, etc.  
    - *Inputs:* Trigger from manual node  
    - *Outputs:* JSON story data forwarded to `Creative Technician Brief`  
    - *Edge Cases:*  
      - Failure if OpenRouter API unavailable or if prompt syntax errors occur  
      - Output format errors if AI returns invalid JSON  

  - **OpenRouter Chat Model**  
    - *Type:* AI Language Model node (Gemini 2.5 flash via OpenRouter)  
    - *Role:* Underlying AI model powering the `Creative Director` node  
    - *Configuration:*  
      - Model: `google/gemini-2.5-flash`  
      - Temperature: 0.9 (creative outputs)  
      - Response format: JSON object  
    - *Credentials:* OpenRouter API key required  
    - *Edge Cases:* API rate limits, auth failures, or unexpected model errors  

---

#### 2.2 AI Prompt Conversion

- **Overview:** Converts each story scene JSON into two AI prompts per scene — one for text-to-image generation and one for text-to-video generation — ensuring visual consistency and style guidelines.

- **Nodes Involved:**  
  - `Creative Technician Brief`  
  - `OpenRouter Chat Model1` (AI backend for Technician)  
  - `Split Out`  

- **Node Details:**

  - **Creative Technician Brief**  
    - *Type:* LangChain LLM Chain node  
    - *Role:* Receives JSON from Creative Director and generates AI prompts for images and videos based on detailed instructions  
    - *Configuration:*  
      - Instruction to produce 2 prompts per scene: detailed static image prompt and a focused video prompt describing subtle animations  
      - Style guide includes digital painting, lighting by time of day, character consistency, and text placement requirements  
      - Outputs JSON array with `scene_number`, `text_to_image_prompt`, and `text_to_video_prompt` per scene  
      - Input is JSON stringified scene array from Creative Director  
    - *Inputs:* JSON from Creative Director  
    - *Outputs:* JSON array sent downstream to `Split Out`  
    - *Edge Cases:* Same as Creative Director node — AI output format errors or API issues  

  - **OpenRouter Chat Model1**  
    - *Type:* AI Language Model node (Gemini 2.5 flash via OpenRouter)  
    - *Role:* AI backend for the Technician prompt generation  
    - *Configuration:* Same as first OpenRouter Chat Model node  
    - *Credentials:* OpenRouter API key as before  
    - *Edge Cases:* Same as above  

  - **Split Out**  
    - *Type:* Split Out node  
    - *Role:* Splits the array of prompt objects, so each scene prompt is handled individually in the downstream loop  
    - *Configuration:* Splits on field `data` which holds the array of scene prompts  
    - *Inputs:* JSON array from Technician Brief  
    - *Outputs:* Each scene prompt as individual item to next node  
    - *Edge Cases:* Failure if input is not an array or empty array  

---

#### 2.3 Image Generation & Approval

- **Overview:** Generates images from the scene image prompts using Replicate’s AI model, then requests human approval via Slack. If rejected, the image generation is retried.

- **Nodes Involved:**  
  - `Limit` (disabled, for testing)  
  - `Loop Over Items`  
  - `Generate an Image`  
  - `Request Approval` (Slack)  
  - `If: Approved`  
  - `Reset Prompt`  
  - `Upload images`  
  - `Set Scene Image`  

- **Node Details:**

  - **Limit**  
    - *Type:* Limit node (disabled)  
    - *Role:* Restricts number of scenes processed (used for testing, disabled in production)  
    - *Configuration:* Max items 3 (inactive)  
    - *Edge Cases:* Can be enabled to avoid API overuse during testing  

  - **Loop Over Items**  
    - *Type:* Split In Batches  
    - *Role:* Processes each scene prompt individually  
    - *Configuration:* Default batch size (usually 1)  
    - *Inputs:* Individual scene prompts from `Split Out`  
    - *Outputs:* Each scene processed sequentially  

  - **Generate an Image**  
    - *Type:* HTTP Request  
    - *Role:* Calls Replicate API to generate static images from prompts  
    - *Configuration:*  
      - POST to `https://api.replicate.com/v1/models/qwen/qwen-image/predictions`  
      - Includes prompt from scene JSON (`text_to_image_prompt`) with parameters for style, guidance, size, and LoRA weights for watercolor style  
      - Authenticated via Replicate HTTP Header Auth credential  
      - Header: `Prefer: wait` to wait for synchronous response  
    - *Inputs:* Scene prompt JSON from loop  
    - *Outputs:* JSON including image URL(s)  
    - *Edge Cases:* API errors, rate limits, network timeouts  

  - **Request Approval**  
    - *Type:* Slack node (send and wait for response)  
    - *Role:* Sends generated image to Slack channel `#content-automation-approvals` requesting double approval  
    - *Configuration:*  
      - Message includes image URL and prompt text  
      - Approval options: double approval required, with “Regenerate” as disapprove label  
      - Uses Slack OAuth2 credentials  
    - *Inputs:* Image generation output  
    - *Outputs:* Approval result with boolean flag `approved`  
    - *Edge Cases:* Slack API failures, missing channel, user not responding  

  - **If: Approved**  
    - *Type:* If node  
    - *Role:* Routes flow based on approval result  
    - *Configuration:* Checks if `data.approved` is true  
    - *Outputs:* If approved -> upload image; if not -> reset prompt to regenerate  

  - **Reset Prompt**  
    - *Type:* Set node  
    - *Role:* Resets the prompt to the original to retry image generation  
    - *Configuration:* Copies prompt string from previous `Generate an Image` request  
    - *Outputs:* Feeds back to `Generate an Image` node  

  - **Upload images**  
    - *Type:* Blotato node  
    - *Role:* Uploads approved image to Blotato media library  
    - *Configuration:* Sends URL from Replicate output  
    - *Credentials:* Blotato API key required  
    - *Outputs:* Uploaded media URL  

  - **Set Scene Image**  
    - *Type:* Set node  
    - *Role:* Enriches data with scene number, image URL, and video prompt for downstream use  
    - *Outputs:* Passes enriched JSON to next loop for video generation  

---

#### 2.4 Video Generation & Approval

- **Overview:** Generates videos using Replicate from the images and text-to-video prompts, requests Slack approval, and handles retries similarly to images.

- **Nodes Involved:**  
  - `Loop Over Images`  
  - `Generate a video`  
  - `Request Approval for Video` (Slack)  
  - `If: Approved2`  
  - `Reset Video Prompt`  
  - `Upload videos`  
  - `Set Scene Video`  

- **Node Details:**

  - **Loop Over Images**  
    - *Type:* Split In Batches  
    - *Role:* Processes each scene’s image/video prompt pair individually  
    - *Inputs:* From previous loop output (`Set Scene Image`)  
    - *Outputs:* One item per scene  

  - **Generate a video**  
    - *Type:* HTTP Request  
    - *Role:* Calls Replicate API to generate short videos with subtle animations from images and video prompts  
    - *Configuration:*  
      - POST to `https://api.replicate.com/v1/models/bytedance/seedance-1-lite/predictions`  
      - Parameters include fps=24, duration=5s, resolution=480p, aspect ratio 3:4  
      - Uses image URL and video prompt text  
      - Authenticated with Replicate HTTP Header Auth  
      - Header: `Prefer: wait` for sync response  
    - *Outputs:* Video URL returned in JSON  
    - *Edge Cases:* Similar to image generation node  

  - **Request Approval for Video**  
    - *Type:* Slack node (send and wait)  
    - *Role:* Sends video for approval in Slack channel with double approval requirement  
    - *Inputs:* Video URL from generation node  
    - *Outputs:* Approval boolean  
    - *Edge Cases:* Same as image approval node  

  - **If: Approved2**  
    - *Type:* If node  
    - *Role:* Routes based on video approval result  
    - *Outputs:* If approved -> upload video; else reset prompt to regenerate  

  - **Reset Video Prompt**  
    - *Type:* Set node  
    - *Role:* Resets video prompt and image for retry  
    - *Outputs:* Feeds back to `Generate a video` node  

  - **Upload videos**  
    - *Type:* Blotato node  
    - *Role:* Uploads approved video to Blotato media library  
    - *Inputs:* Video URL from Replicate  
    - *Credentials:* Blotato API key required  

  - **Set Scene Video**  
    - *Type:* Set node  
    - *Role:* Adds scene number, video URL, and image URL for downstream processing  

---

#### 2.5 Media Processing

- **Overview:** Resizes approved images (adds borders and crops) for TikTok slideshow format and prepares media URLs for publishing.

- **Nodes Involved:**  
  - `Sort`  
  - `Loop Over Items1`  
  - `Get Image`  
  - `Resize Image: Add Borders`  
  - `Resize Image1`  
  - `Upload images1`  
  - `Summarize1`  

- **Node Details:**

  - **Sort**  
    - *Type:* Sort  
    - *Role:* Orders scenes by `scene_number` to maintain narrative sequence  
    - *Configuration:* Sort ascending by `scene_number`  

  - **Loop Over Items1**  
    - *Type:* Split In Batches  
    - *Role:* Processes images one-by-one for resizing and uploading  

  - **Get Image**  
    - *Type:* HTTP Request  
    - *Role:* Downloads image from approved image URL for editing  

  - **Resize Image: Add Borders**  
    - *Type:* Edit Image  
    - *Role:* Adds padding borders (height 230 pixels) to image  

  - **Resize Image1**  
    - *Type:* Edit Image  
    - *Role:* Crops image to 1080x1920 for TikTok vertical format  

  - **Upload images1**  
    - *Type:* Blotato node  
    - *Role:* Uploads resized images to Blotato media library, using binary data (image file)  

  - **Summarize1**  
    - *Type:* Summarize  
    - *Role:* Concatenates all uploaded TikTok image URLs into a single string for the slideshow post  

---

#### 2.6 Social Media Publishing

- **Overview:** Publishes the approved and processed content to Instagram, Facebook, and TikTok platforms using Blotato integration.

- **Nodes Involved:**  
  - `Create Instagram Post`  
  - `Create Facebook Post`  
  - `Create TikTok Post`  

- **Node Details:**

  - **Create Instagram Post**  
    - *Type:* Blotato node  
    - *Role:* Posts video content with caption from the first scene’s reassuring quote  
    - *Parameters:*  
      - Account: Instagram account ID `12109`  
      - Media URLs: concatenated video URLs from summarization  
      - Caption: First scene’s `life_reassuring_quote`  
    - *Credentials:* Blotato API  

  - **Create Facebook Post**  
    - *Type:* Blotato node  
    - *Role:* Posts video content with caption on specific Facebook page  
    - *Parameters:*  
      - Account ID `8120` and Facebook Page ID `110037418539464`  
      - Caption and media same as Instagram  
    - *Credentials:* Blotato API  

  - **Create TikTok Post**  
    - *Type:* Blotato node  
    - *Role:* Creates TikTok slideshow post from resized images  
    - *Parameters:*  
      - Account ID `13396`  
      - Text: "skeletal life" (static caption)  
      - Media URLs: concatenated TikTok image URLs  
      - Options: auto-add music, disable duet, privacy set to self only, AI-generated flag true  
    - *Credentials:* Blotato API  

---

#### 2.7 Control & Utilities

- **Overview:** Contains helper nodes for flow control, manual or scheduled triggers, and documentation notes.

- **Nodes Involved:**  
  - `Sticky Note` nodes (multiple, documentation)  
  - `Schedule Trigger` (disabled)  

- **Node Details:**

  - **Sticky Note**  
    - *Role:* Provides user guidance, setup instructions, tutorial links, and process explanations  
    - *Content Highlights:*  
      - Setup instructions for accounts and credentials (Replicate, Blotato, OpenRouter, Slack)  
      - Notes on modifying storylines, Slack channel setup, and social media integration  
      - YouTube tutorial link embedded: `[youtube](F4MWgsftNWE)`  

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger (disabled)  
    - *Role:* Optional automation to start workflow daily at 10 AM if enabled  

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                         | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                               |
|-------------------------|-----------------------------------|---------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Manual start of workflow               | None                             | Creative Director                |                                                                                                          |
| Creative Director        | LangChain LLM Chain               | Generates story JSON scenes            | When clicking ‘Execute workflow’ | Creative Technician Brief        |                                                                                                          |
| OpenRouter Chat Model    | AI Language Model (Gemini)        | AI backend for story generation        | Creative Director (AI backend)   | Creative Director                |                                                                                                          |
| Creative Technician Brief| LangChain LLM Chain               | Converts story to AI prompts            | Creative Director                | Split Out                       |                                                                                                          |
| OpenRouter Chat Model1   | AI Language Model (Gemini)        | AI backend for prompt conversion       | Creative Technician Brief (AI backend) | Creative Technician Brief    |                                                                                                          |
| Split Out               | Split Out                         | Splits prompt array into individual items | Creative Technician Brief        | Limit (disabled)/Loop Over Items |                                                                                                          |
| Limit                  | Limit (disabled)                   | Limits number of items for testing      | Split Out                       | Loop Over Items                 | ## Limit images activate this during testing                                                             |
| Loop Over Items         | Split In Batches                  | Processes each scene prompt individually | Limit / Split Out               | Generate an Image               |                                                                                                          |
| Generate an Image       | HTTP Request                     | Calls Replicate API to create images   | Loop Over Items                 | Request Approval               |                                                                                                          |
| Request Approval        | Slack (Send and Wait)             | Requests human approval for images     | Generate an Image               | If: Approved                   |                                                                                                          |
| If: Approved            | If                               | Routes flow based on approval          | Request Approval                | Upload images / Reset Prompt    |                                                                                                          |
| Reset Prompt            | Set                              | Resets prompt to regenerate image      | If: Approved (disapprove path) | Generate an Image               |                                                                                                          |
| Upload images           | Blotato                         | Uploads approved image to Blotato      | If: Approved                   | Set Scene Image                |                                                                                                          |
| Set Scene Image         | Set                              | Prepares scene data for video generation | Upload images                  | Loop Over Images               |                                                                                                          |
| Loop Over Images        | Split In Batches                  | Processes each scene image/video pair  | Set Scene Image                | Generate a video               |                                                                                                          |
| Generate a video        | HTTP Request                     | Calls Replicate API to create videos   | Loop Over Images               | Request Approval for Video     |                                                                                                          |
| Request Approval for Video | Slack (Send and Wait)             | Requests human approval for videos     | Generate a video               | If: Approved2                 |                                                                                                          |
| If: Approved2           | If                               | Routes flow based on video approval    | Request Approval for Video      | Upload videos / Reset Video Prompt |                                                                                                          |
| Reset Video Prompt      | Set                              | Resets video prompt to regenerate video | If: Approved2 (disapprove path) | Generate a video               |                                                                                                          |
| Upload videos           | Blotato                         | Uploads approved video to Blotato      | If: Approved2                 | Set Scene Video               |                                                                                                          |
| Set Scene Video         | Set                              | Prepares scene data for publishing     | Upload videos                 | Loop Over Images              |                                                                                                          |
| Sort                    | Sort                             | Sorts scenes by scene_number            | Loop Over Images               | Summarize                     |                                                                                                          |
| Summarize               | Summarize                       | Concatenates video and image URLs      | Sort                          | Create Instagram Post, Create Facebook Post |                                                                                                         |
| Create Instagram Post   | Blotato                         | Posts video content to Instagram       | Summarize                    |                                |                                                                                                          |
| Create Facebook Post    | Blotato                         | Posts video content to Facebook        | Summarize                    |                                |                                                                                                          |
| Summarize1              | Summarize                       | Concatenates TikTok image URLs         | Loop Over Items1              | Create TikTok Post            |                                                                                                          |
| Loop Over Items1        | Split In Batches                  | Processes TikTok images for resizing   | Upload images1               | Resize Image: Add Borders       |                                                                                                          |
| Get Image               | HTTP Request                     | Downloads image for processing          | Loop Over Items1             | Resize Image: Add Borders       |                                                                                                          |
| Resize Image: Add Borders | Edit Image                       | Adds borders to image                   | Get Image                   | Resize Image1                 |                                                                                                          |
| Resize Image1           | Edit Image                       | Crops image for TikTok vertical format | Resize Image: Add Borders    | Upload images1               |                                                                                                          |
| Upload images1          | Blotato                         | Uploads resized images to Blotato      | Resize Image1               | Loop Over Items1              |                                                                                                          |
| Create TikTok Post      | Blotato                         | Posts TikTok slideshow                  | Summarize1                  |                               |                                                                                                          |
| Sticky Note             | Sticky Note                      | Documentation and setup notes           | None                         | None                         | ## Generate a Story                                                                                       |
| Sticky Note1            | Sticky Note                      | Documentation note for prompt conversion| None                         | None                         | ## Convert the Story to prompts                                                                           |
| Sticky Note2            | Sticky Note                      | Documentation for image generation block| None                         | None                         | ## Generate and Approve images                                                                             |
| Sticky Note3            | Sticky Note                      | Documentation for video generation block| None                         | None                         | ## Generate and Approve videos                                                                             |
| Sticky Note4            | Sticky Note                      | Documentation for publishing blocks     | None                         | None                         | ## Publish: Instagram, Facebook                                                                             |
| Sticky Note5            | Sticky Note                      | Documentation for TikTok publishing     | None                         | None                         | ## Publish: TikTok slideshow                                                                                |
| Sticky Note6            | Sticky Note                      | Setup instructions and credentials info | None                         | None                         | ## SetUp: 1. Create accounts on [Replicate](https://replicate.com/) and [Blotato](https://tinyurl.com/blotatoapp) 2. Connect LLM and Slack, etc. |
| Sticky Note7            | Sticky Note                      | YouTube tutorial link                    | None                         | None                         | ## YouTube Tutorial: @[youtube](F4MWgsftNWE)                                                              |
| Sticky Note8            | Sticky Note                      | Testing note for image limit             | None                         | None                         | ## Limit images activate this during testing                                                               |
| Schedule Trigger        | Schedule Trigger (disabled)      | Optional scheduled workflow start       | None                         | Creative Director            |                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually  
   - Connect output to `Creative Director`

2. **Create Creative Director Node**  
   - Type: LangChain Chain LLM  
   - Configure prompt with detailed instructions to generate 5-8 JSON scenes describing a day in the life of a serene skeleton, including date context with expression `{{$now.format('dd LLLL')}}`  
   - Set batching empty  
   - Connect input from Manual Trigger  
   - Connect output to OpenRouter Chat Model node

3. **Create OpenRouter Chat Model Node (for Creative Director)**  
   - Type: LangChain LLM Chat OpenRouter  
   - Model: `google/gemini-2.5-flash`  
   - Temperature: 0.9  
   - Response format: JSON object  
   - Attach OpenRouter API credential  
   - Connect AI output to Creative Director node  

4. **Create Creative Technician Brief Node**  
   - Type: LangChain Chain LLM  
   - Prompt: Instructions to convert story JSON into two AI prompts per scene (image and video), including style guide and lighting instructions  
   - Input: Output JSON string from Creative Director node  
   - Connect output to OpenRouter Chat Model1 node

5. **Create OpenRouter Chat Model1 Node (for Technician)**  
   - Same settings as first OpenRouter node  
   - Connect AI output to `Split Out` node

6. **Create Split Out Node**  
   - Field to split: `data` (array of prompts)  
   - Connect output to `Limit` node (optional)

7. **Create Limit Node (optional, disabled by default)**  
   - Max Items: 3 (for testing only)  
   - Connect output to `Loop Over Items`

8. **Create Loop Over Items Node**  
   - Type: Split In Batches  
   - Connect each item (scene prompt) to `Generate an Image`

9. **Create Generate an Image Node (HTTP Request)**  
   - POST URL: `https://api.replicate.com/v1/models/qwen/qwen-image/predictions`  
   - Body JSON includes prompt from current item’s `text_to_image_prompt` with parameters for style, guidance, LoRA weights, aspect ratio, etc.  
   - Add HTTP Header Auth credential for Replicate  
   - Header: `Prefer: wait`  
   - Connect output to `Request Approval` node

10. **Create Request Approval Node (Slack)**  
    - Operation: sendAndWait  
    - Channel: `#content-automation-approvals` (adjust to your Slack setup)  
    - Message: Include image URL and prompt text  
    - Approval Type: double approval with disapprove labeled “Regenerate”  
    - Attach Slack OAuth2 credentials  
    - Connect output to `If: Approved`

11. **Create If: Approved Node**  
    - Condition: `{{$json.data.approved}}` is true  
    - If yes: connect to `Upload images`  
    - If no: connect to `Reset Prompt`

12. **Create Reset Prompt Node (Set)**  
    - Reset `text_to_image_prompt` to original prompt from previous `Generate an Image` node request  
    - Connect output back to `Generate an Image` for retry

13. **Create Upload images Node (Blotato)**  
    - Resource: media  
    - Media URL: image URL from Replicate output  
    - Connect output to `Set Scene Image`

14. **Create Set Scene Image Node (Set)**  
    - Assign: scene_number, image URL, and keep `text_to_video_prompt` from current item  
    - Connect output to `Loop Over Images`

15. **Create Loop Over Images Node**  
    - Split In Batches node to process each scene’s image/video pair  
    - Connect output to `Generate a video`

16. **Create Generate a video Node (HTTP Request)**  
    - POST URL: `https://api.replicate.com/v1/models/bytedance/seedance-1-lite/predictions`  
    - Input: fps=24, duration=5, resolution=480p, aspect_ratio=3:4, image URL, prompt from `text_to_video_prompt`  
    - Use Replicate HTTP Header Auth credentials  
    - Header: `Prefer: wait`  
    - Connect output to `Request Approval for Video`

17. **Create Request Approval for Video Node (Slack)**  
    - Similar config as image approval node, message includes video URL  
    - Approval type double with “Regenerate” label  
    - Connect output to `If: Approved2`

18. **Create If: Approved2 Node**  
    - Condition: `{{$json.data.approved}}` is true  
    - If yes: connect to `Upload videos`  
    - If no: connect to `Reset Video Prompt`

19. **Create Reset Video Prompt Node (Set)**  
    - Reset `text_to_video_prompt` and image to original values from `Generate a video` request  
    - Connect output back to `Generate a video`

20. **Create Upload videos Node (Blotato)**  
    - Resource: media  
    - Media URL: video URL from Replicate output  
    - Connect output to `Set Scene Video`

21. **Create Set Scene Video Node (Set)**  
    - Assign scene_number, video URL, and image URL  
    - Connect output to `Sort`

22. **Create Sort Node**  
    - Sort ascending by `scene_number`  
    - Connect output to `Summarize`

23. **Create Summarize Node**  
    - Fields to summarize: concatenate all `video` and `image` URLs separated by commas  
    - Connect outputs to `Create Instagram Post` and `Create Facebook Post`

24. **Create Create Instagram Post Node (Blotato)**  
    - Platform: Instagram  
    - Account ID: your Instagram account  
    - Post content text: first scene’s `life_reassuring_quote` from Creative Director output  
    - Media URLs: concatenated video URLs from Summarize  
    - Credentials: Blotato API

25. **Create Create Facebook Post Node (Blotato)**  
    - Platform: Facebook  
    - Account ID and Facebook Page ID set accordingly  
    - Post content and media same as Instagram node  
    - Credentials: Blotato API

26. **Create Loop Over Items1 Node**  
    - Split In Batches to process images for TikTok resizing  
    - Input: from `Upload images`

27. **Create Get Image Node (HTTP Request)**  
    - Downloads image binary from URL for editing  
    - Connect output to `Resize Image: Add Borders`

28. **Create Resize Image: Add Borders Node**  
    - Edit Image node  
    - Add 230 px vertical borders (height only)  
    - Connect output to `Resize Image1`

29. **Create Resize Image1 Node**  
    - Edit Image node  
    - Crop or resize to 1080x1920 (TikTok vertical)  
    - Connect output to `Upload images1`

30. **Create Upload images1 Node (Blotato)**  
    - Upload resized images to Blotato as binary  
    - Connect output to `Loop Over Items1` (to continue processing) and `Summarize1`

31. **Create Summarize1 Node**  
    - Concatenate all TikTok image URLs for slideshow  
    - Connect output to `Create TikTok Post`

32. **Create Create TikTok Post Node (Blotato)**  
    - Platform: TikTok  
    - Account ID: your TikTok account  
    - Post content text: "skeletal life" (static caption)  
    - Media URLs: concatenated TikTok image URLs  
    - Options: auto-add music, disable duet, privacy self only, AI-generated flag true  
    - Credentials: Blotato API

33. **(Optional) Schedule Trigger Node**  
    - Configure for daily run (e.g., 10 AM)  
    - Connect output to `Creative Director`

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Setup instructions: Create accounts on [Replicate](https://replicate.com/) and [Blotato](https://tinyurl.com/blotatoapp). Connect Gemini 2.5 flash via [OpenRouter](https://openrouter.ai/). Create Slack Bot and channel.   | Sticky Note6                                                                                     |
| You can add more social media platforms supported by Blotato and connect multiple accounts per platform. Ensure Slack channels exist and are correctly referenced in Slack nodes.                                           | Sticky Note6                                                                                     |
| Slack approval nodes require a Slack app with proper scopes for chat:write, chat:write.public, and interactive components to handle approvals.                                                                             | General integration note                                                                         |
| The YouTube tutorial for this workflow is available here: https://youtu.be/F4MWgsftNWE                                                                                                                                        | Sticky Note7                                                                                     |
| Use the `Limit` node during development to restrict API calls and speed up testing by limiting processed items.                                                                                                              | Sticky Note8                                                                                     |
| The workflow uses LoRA weights hosted on HuggingFace for watercolor brush effect in image generation. Ensure that the URL is accessible or replace with your preferred LoRA weights.                                         | `Generate an Image` node configuration                                                          |
| All AI-generated text prompts are designed for static scene generation; avoid references to other scenes to maintain prompt independence.                                                                                   | Creative Technician Brief instructions                                                         |
| Slack approvals use double confirmation to reduce errors and maintain quality control.                                                                                                                                      | Slack nodes configuration                                                                       |
| Blotato node requires API credentials and correct account IDs for social media platforms; ensure these are set up before publishing.                                                                                         | Blotato nodes                                                                                   |

---

**Disclaimer:** The content provided is generated from an automated workflow made with n8n, respecting all applicable content policies. All processed data is legal and publicly accessible.