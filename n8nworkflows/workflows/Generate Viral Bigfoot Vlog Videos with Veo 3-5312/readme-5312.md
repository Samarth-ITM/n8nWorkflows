Generate Viral Bigfoot Vlog Videos with Veo 3

https://n8nworkflows.xyz/workflows/generate-viral-bigfoot-vlog-videos-with-veo-3-5312


# Generate Viral Bigfoot Vlog Videos with Veo 3

### 1. Workflow Overview

This workflow, titled **"The Recap AI - VEO 3 Bigfoot Video Generator"**, automates the creation of viral Bigfoot-themed vlog videos using AI-generated scripts, human approval, and automated video production with Veo 3. It is designed for content creators targeting YouTube or similar platforms who want to generate engaging, character-driven short videos featuring a Bigfoot character named "Sam."

The workflow is logically organized into three main blocks:

- **1.1 Video Script Generation:**  
  Starts with user input via a web form describing a Bigfoot vlog idea, then uses AI (LangChain with Anthropic Claude 4 model) to generate a detailed storyboard with 8 scripted scenes adhering to a strict character and style guide.

- **1.2 Human Approval:**  
  Sends the generated storyboard to a Slack channel for human reviewers to approve or deny the concept before video production proceeds.

- **1.3 Video Generation & Upload:**  
  Once approved, each scene prompt is sent to the Veo 3 AI video generation API. The workflow monitors processing status, downloads completed videos, uploads them to Google Drive, and finally posts a completion message on Slack with links to the generated content.

---

### 2. Block-by-Block Analysis

#### 2.1 Video Script Generation

**Overview:**  
This block captures a user’s Bigfoot video idea, generates a storyboard outline with 8 scripted scenes using a detailed character bible and style guide, and prepares the narrative for approval.

**Nodes Involved:**  
- `form_trigger`  
- `narrative_writer`  
- `claude-4-sonnet` (Anthropic Claude 4 model)  
- `narrative_parser`  
- `split_scenes`  
- `scene_director`  
- `set_scene_prompts`  
- `aggregate_scenes`  
- `send_narrative_msg`

**Node Details:**

- **form_trigger**  
  - *Type:* Web form trigger  
  - *Role:* Entry point; captures user input describing the Bigfoot vlog idea via a simple web form titled "Bigfoot Video Idea" with one required text field.  
  - *Connections:* Outputs to `narrative_writer`.  
  - *Edge Cases:* Missing or malformed input may halt the workflow; form validation requires non-empty input.

- **narrative_writer**  
  - *Type:* LangChain LLM Chain  
  - *Role:* Generates a structured storyboard outline of 8 scenes based on the user input; uses a detailed prompt specifying Bigfoot "Sam" persona, creative mandate, and output format.  
  - *Configuration:* Uses custom prompt text embedding character bible, style guide, and narrative instructions.  
  - *Connections:* Outputs JSON to `split_scenes`.  
  - *Edge Cases:* LLM API failures, rate limits, or parsing errors. Requires Anthropic API credentials.

- **claude-4-sonnet**  
  - *Type:* LangChain LLM Chat (Anthropic Claude 4 Sonnet)  
  - *Role:* Provides language model responses backing the narrative writer node.  
  - *Configuration:* Model set to "claude-sonnet-4-20250514" with Anthropic API credentials.  
  - *Connections:* Feeds into `narrative_writer` (AI language model input).  
  - *Edge Cases:* Authentication failures, API timeouts.

- **narrative_parser**  
  - *Type:* LangChain structured output parser  
  - *Role:* Parses the raw AI output into a validated JSON structure with defined schema for vlog title, concept, character name, and 8 scenes.  
  - *Configuration:* Uses a strict JSON schema to enforce correctness of storyboard output.  
  - *Connections:* Outputs parsed JSON back to `narrative_writer` for further processing.  
  - *Edge Cases:* Parsing failures if output does not comply with schema.

- **split_scenes**  
  - *Type:* Split Out  
  - *Role:* Splits the array of scenes from the parsed storyboard into individual items for detailed processing.  
  - *Connections:* Outputs each scene item to `scene_director`.

- **scene_director**  
  - *Type:* LangChain Chain LLM  
  - *Role:* Takes a simple scene brief and expands it into a fully detailed, production-ready video script for that scene following the character bible and style guide.  
  - *Configuration:* Uses a prompt detailing the series bible, style, technical specs, and a precise output template in markdown.  
  - *Connections:* Outputs detailed scene script text to `set_scene_prompts`.

- **set_scene_prompts**  
  - *Type:* Set  
  - *Role:* Assigns the detailed scene script text to a field named `scene_prompt`.  
  - *Connections:* Outputs to `aggregate_scenes`.

- **aggregate_scenes**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all individual scene prompts into a single array for subsequent steps.  
  - *Connections:* Outputs to `send_narrative_msg`.

- **send_narrative_msg**  
  - *Type:* Slack node (send message and wait for response)  
  - *Role:* Sends the full storyboard text and key video info to a Slack channel for review and approval, pausing the flow until a response is received.  
  - *Configuration:* Uses OAuth2 Slack credentials; posts to channel `C08KC39K8DR` (named `ai-tools-content`).  
  - *Connections:* On approval, outputs to `send_and_wait` node.  
  - *Edge Cases:* Slack API errors, network issues, or no user response.

---

#### 2.2 Human Approval

**Overview:**  
This block manages human review and approval of the generated storyboard before video creation proceeds.

**Nodes Involved:**  
- `send_and_wait`  
- `reset_scene_prompts`  
- `split_scene_prompts`  
- `iterate_prompts`

**Node Details:**

- **send_and_wait**  
  - *Type:* Slack node (send and wait for approval)  
  - *Role:* Sends an approval request message in Slack asking reviewers to approve or deny the storyboard to proceed.  
  - *Configuration:* Double approval required; posts to same Slack channel as `send_narrative_msg`.  
  - *Connections:* On approval, outputs to `reset_scene_prompts`.  
  - *Edge Cases:* Approval timeout, Slack API failures.

- **reset_scene_prompts**  
  - *Type:* Set  
  - *Role:* Resets and prepares aggregated scene prompts into a single field `scene_prompts` (array) for splitting.  
  - *Connections:* Outputs to `split_scene_prompts`.

- **split_scene_prompts**  
  - *Type:* Split Out  
  - *Role:* Splits the aggregated scene prompts array into individual scenes for batch processing.  
  - *Connections:* Outputs to `iterate_prompts`.

- **iterate_prompts**  
  - *Type:* Split In Batches  
  - *Role:* Processes scene prompts in batches (default batch size 1) to sequentially generate videos per scene.  
  - *Connections:* Outputs to `aggregate_video_results` and `set_current_prompt`.

---

#### 2.3 Video Generation & Upload

**Overview:**  
This block handles the generation of videos from scene prompts using Veo 3 API, monitors job status, downloads completed videos, uploads them to Google Drive, and sends a final completion message on Slack.

**Nodes Involved:**  
- `set_current_prompt`  
- `queue_create_video`  
- `wait` (two instances)  
- `fetch_status`  
- `check_status`  
- `fetch_result`  
- `download_result_video`  
- `upload_video`  
- `aggregate_video_results`  
- `send_completion_msg`

**Node Details:**

- **set_current_prompt**  
  - *Type:* Set  
  - *Role:* Prepares the current scene prompt string as a `prompt` field for the video generation API call.  
  - *Connections:* Outputs to `queue_create_video`.

- **queue_create_video**  
  - *Type:* HTTP Request  
  - *Role:* Sends a POST request to Veo 3 API endpoint to queue a video generation job with parameters: prompt text, 16:9 aspect ratio, 8 seconds duration, and audio generation enabled.  
  - *Configuration:* Uses HTTP header authentication credential named `FAL API`.  
  - *Connections:* Outputs to first `wait` node.  
  - *Edge Cases:* API errors, invalid prompt, authentication failure.

- **wait** (positioned after `queue_create_video`)  
  - *Type:* Wait node  
  - *Role:* Pauses the workflow for 10 seconds before checking job status to avoid hammering API.  
  - *Connections:* Outputs to `fetch_status`.

- **fetch_status**  
  - *Type:* HTTP Request  
  - *Role:* Queries the Veo 3 API for current status of the video generation request using the request ID from `queue_create_video`.  
  - *Connections:* Outputs to `check_status`.

- **check_status**  
  - *Type:* If node  
  - *Role:* Checks if the status returned is "COMPLETED".  
  - *Connections:* If true, proceeds to `fetch_result`; else loops back to `wait` to poll again.  
  - *Edge Cases:* Non-standard statuses, request cancellation, API errors.

- **fetch_result**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the completed video details and URLs from the Veo 3 API.  
  - *Connections:* Outputs to `download_result_video`.

- **download_result_video**  
  - *Type:* HTTP Request (file download)  
  - *Role:* Downloads the generated video file as binary data from the provided URL.  
  - *Connections:* Outputs to `upload_video`.  
  - *Edge Cases:* Download failures, broken URLs.

- **upload_video**  
  - *Type:* Google Drive node  
  - *Role:* Uploads the downloaded video file to a specified Google Drive folder with the filename format `scene_{index}.mp4`.  
  - *Configuration:* Uses Google Drive OAuth2 credentials; uploads to folder URL `https://drive.google.com/drive/folders/1Lf84YrGu456naiaclZRp1ARMZdQUKJj8`.  
  - *Connections:* Outputs to `iterate_prompts` (to continue next scene).  
  - *Edge Cases:* Upload permission errors, quota limits.

- **aggregate_video_results**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all uploaded videos' metadata for final reporting.  
  - *Connections:* Outputs to `send_completion_msg`.

- **send_completion_msg**  
  - *Type:* Slack node (send message)  
  - *Role:* Posts a final Slack message signaling video generation completion with vlog title, concept, and Google Drive folder link.  
  - *Connections:* Terminal node.  
  - *Edge Cases:* Slack API failures.

---

### 3. Summary Table

| Node Name             | Node Type                                | Functional Role                         | Input Node(s)                      | Output Node(s)                   | Sticky Note                           |
|-----------------------|----------------------------------------|---------------------------------------|----------------------------------|---------------------------------|-------------------------------------|
| form_trigger          | Form Trigger                           | Capture user Bigfoot video idea       | -                                | narrative_writer                 |                                     |
| narrative_writer      | LangChain Chain LLM                    | Generate 8-scene storyboard outline   | form_trigger                     | split_scenes                    |                                     |
| claude-4-sonnet       | LangChain LM Chat (Anthropic)         | AI language model backend              | narrative_writer (AI model input)| narrative_writer (AI response)  |                                     |
| narrative_parser      | LangChain Output Parser Structured     | Parse storyboard JSON output           | claude-4-sonnet                  | narrative_writer                 |                                     |
| split_scenes          | Split Out                             | Split storyboard into individual scenes| narrative_writer                 | scene_director                  |                                     |
| scene_director        | LangChain Chain LLM                    | Expand scene briefs into detailed scripts| split_scenes                   | set_scene_prompts               | ## 1. Write Video Script            |
| set_scene_prompts     | Set                                   | Assign detailed scene script to field | scene_director                   | aggregate_scenes                |                                     |
| aggregate_scenes      | Aggregate                             | Aggregate all scene scripts            | set_scene_prompts                | send_narrative_msg              |                                     |
| send_narrative_msg    | Slack (send message and wait)          | Send storyboard for human approval     | aggregate_scenes                | send_and_wait                  | ## 2. Get Human Approval            |
| send_and_wait         | Slack (send and wait for approval)     | Request approval to proceed             | send_narrative_msg              | reset_scene_prompts             |                                     |
| reset_scene_prompts   | Set                                   | Prepare aggregated scene prompts array | send_and_wait                   | split_scene_prompts             |                                     |
| split_scene_prompts   | Split Out                             | Split scene prompts for batch processing| reset_scene_prompts             | iterate_prompts                |                                     |
| iterate_prompts       | Split In Batches                      | Process scene prompts batch-wise        | split_scene_prompts             | aggregate_video_results, set_current_prompt |                                     |
| set_current_prompt    | Set                                   | Set current prompt for video generation | iterate_prompts                | queue_create_video             |                                     |
| queue_create_video    | HTTP Request                         | Queue video generation job at Veo 3 API| set_current_prompt             | wait                          | ## 3. Generate Video                |
| wait (after queue)    | Wait                                  | Pause before polling status             | queue_create_video              | fetch_status                   |                                     |
| fetch_status          | HTTP Request                         | Check video generation job status       | wait                          | check_status                   |                                     |
| check_status          | If                                    | Branch based on job completion          | fetch_status                   | fetch_result / wait            |                                     |
| fetch_result          | HTTP Request                         | Fetch completed video info               | check_status                   | download_result_video          |                                     |
| download_result_video | HTTP Request (file)                   | Download generated video file            | fetch_result                   | upload_video                  |                                     |
| upload_video          | Google Drive                         | Upload video to Google Drive             | download_result_video          | iterate_prompts               |                                     |
| aggregate_video_results| Aggregate                            | Aggregate all video uploads              | iterate_prompts                | send_completion_msg           |                                     |
| send_completion_msg   | Slack (send message)                   | Notify completion with links             | aggregate_video_results        | -                             |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create form trigger node (`form_trigger`):**  
   - Type: Form Trigger  
   - Configure form title: "Bigfoot Video Idea"  
   - Add one required text field labeled "Bigfoot Video Idea" with placeholder "Describe the high level idea for your bigfoot vlog"  
   - Save webhook URL for external user submission.

2. **Add LangChain Chain LLM node (`narrative_writer`):**  
   - Connect from `form_trigger`.  
   - Paste detailed prompt defining Bigfoot persona, creative mandate, output spec for 8 scenes.  
   - Use Anthropic Claude 4 Sonnet model as AI backend.  
   - Set to parse output as JSON.

3. **Add Anthropic Claude 4 Sonnet node (`claude-4-sonnet`):**  
   - Set model to "claude-sonnet-4-20250514".  
   - Provide Anthropic API credentials.

4. **Connect `claude-4-sonnet` AI output to `narrative_writer` as AI model input.**

5. **Add LangChain Output Parser node (`narrative_parser`):**  
   - Configure with JSON schema for storyboard validation (8 scenes, fields like vlogTitle, concept, scenes with sceneNumber, timestamp, narrative).  
   - Connect output of `claude-4-sonnet` to this parser.  
   - Connect parser output back to `narrative_writer`.

6. **Add Split Out node (`split_scenes`):**  
   - Split the `scenes` array from `narrative_writer` output.  
   - Connect from `narrative_writer`.

7. **Add LangChain Chain LLM node (`scene_director`):**  
   - Configure prompt to expand single scene brief into detailed production-ready script using the character bible and style guide, with exact output markdown template.  
   - Connect from `split_scenes`.

8. **Add Set node (`set_scene_prompts`):**  
   - Assign the detailed scene script text to field `scene_prompt`.  
   - Connect from `scene_director`.

9. **Add Aggregate node (`aggregate_scenes`):**  
   - Aggregate all `scene_prompt` fields into array.  
   - Connect from `set_scene_prompts`.

10. **Add Slack node (`send_narrative_msg`):**  
    - Configure to post assembled storyboard text to Slack channel `ai-tools-content` (`C08KC39K8DR`).  
    - Use OAuth2 Slack credentials.  
    - Connect from `aggregate_scenes`.

11. **Add Slack node (`send_and_wait`):**  
    - Configure to send approval request message with double approval option.  
    - Same Slack channel and credentials.  
    - Connect from `send_narrative_msg`.

12. **Add Set node (`reset_scene_prompts`):**  
    - Set `scene_prompts` field to aggregated scene prompts array from previous step.  
    - Connect from `send_and_wait`.

13. **Add Split Out node (`split_scene_prompts`):**  
    - Split `scene_prompts` array into individual prompts.  
    - Connect from `reset_scene_prompts`.

14. **Add Split In Batches node (`iterate_prompts`):**  
    - Configure batch size (default 1).  
    - Connect from `split_scene_prompts`.

15. **Add Set node (`set_current_prompt`):**  
    - Assign current batch prompt string to field `prompt`.  
    - Connect from `iterate_prompts`.

16. **Add HTTP Request node (`queue_create_video`):**  
    - POST to `https://queue.fal.run/fal-ai/veo3`  
    - JSON body: `{ "prompt": "{{ $node[\"set_current_prompt\"].json.prompt }}", "aspect_ratio": "16:9", "duration": "8s", "generate_audio": true }`  
    - Use HTTP Header Auth credentials named `FAL API`.  
    - Connect from `set_current_prompt`.

17. **Add Wait node (`wait`):**  
    - Pause 10 seconds after queuing video.  
    - Connect from `queue_create_video`.

18. **Add HTTP Request node (`fetch_status`):**  
    - GET `https://queue.fal.run/fal-ai/veo3/requests/{{ $node["queue_create_video"].json.request_id }}/status`  
    - Use same `FAL API` credentials.  
    - Connect from `wait`.

19. **Add If node (`check_status`):**  
    - Condition: JSON status field equals "COMPLETED".  
    - Connect from `fetch_status`.  
    - True branch to `fetch_result`.  
    - False branch loops back to `wait`.

20. **Add HTTP Request node (`fetch_result`):**  
    - GET `https://queue.fal.run/fal-ai/veo3/requests/{{ $node["queue_create_video"].json.request_id }}`  
    - Use `FAL API` credentials.  
    - Connect from `check_status` (true branch).

21. **Add HTTP Request node (`download_result_video`):**  
    - Download video file at URL `{{ $json.video.url }}`  
    - Set response to file/binary.  
    - Connect from `fetch_result`.

22. **Add Google Drive node (`upload_video`):**  
    - Upload binary video file.  
    - File name format: `scene_{{ $runIndex + 1 }}.mp4`  
    - Destination folder: Google Drive folder URL `https://drive.google.com/drive/folders/1Lf84YrGu456naiaclZRp1ARMZdQUKJj8`  
    - Use Google Drive OAuth2 credentials.  
    - Connect from `download_result_video`.

23. **Connect `upload_video` output back to `iterate_prompts` to process next scene.**

24. **Add Aggregate node (`aggregate_video_results`):**  
    - Aggregate all video upload results.  
    - Connect from `iterate_prompts`.

25. **Add Slack node (`send_completion_msg`):**  
    - Send completion message with vlog title, concept, and Google Drive link to Slack channel `ai-tools-content`.  
    - Use OAuth2 Slack credentials.  
    - Connect from `aggregate_video_results`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                            | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The Bigfoot character “Sam” is based on the “Outdoor Boys” YouTube channel style and persona, requiring a jolly, warm voice with simple language and mild PG-rated exasperations.                                                                                                         | Character Bible embedded in prompts                                                                |
| The video scenes must be exactly 8 seconds long, shot in 4K UHD @ 29.97fps with 16:9 horizontal aspect ratio, a 24mm equivalent lens at ƒ/2.8, and 1/60s shutter speed to simulate subtle motion blur and handheld wobble for “found footage” authenticity.                              | Scene Director prompt details                                                                      |
| Avoid any subtitles, captions, or on-screen labels in final videos to maintain the authentic vlog style.                                                                                                                                                                                | Strict instructions in scene director prompt                                                      |
| Slack channel `C08KC39K8DR` named `ai-tools-content` is used for all messaging and approval requests. OAuth2 credentials are required for Slack API access.                                                                                                                             | Slack node configuration                                                                           |
| Veo 3 video generation API is accessed at `https://queue.fal.run/fal-ai/veo3` with HTTP header authentication credentials named `FAL API`.                                                                                                                                               | HTTP Request nodes for video generation                                                           |
| Google Drive folder for video uploads is `https://drive.google.com/drive/folders/1Lf84YrGu456naiaclZRp1ARMZdQUKJj8`. OAuth2 credentials for Google Drive API are required.                                                                                                                  | Google Drive node configuration                                                                   |
| The workflow uses LangChain integrations with Anthropic Claude 4 Sonnet model; valid Anthropic API keys must be configured.                                                                                                                                                             | LangChain node and LLM Chat node details                                                          |

---

**Disclaimer:**  
The text and data processed in this workflow are exclusively from an automated n8n workflow designed to generate Bigfoot-themed video content. All content complies with applicable content policies and contains no illegal or offensive material. Data processed is legal and public.