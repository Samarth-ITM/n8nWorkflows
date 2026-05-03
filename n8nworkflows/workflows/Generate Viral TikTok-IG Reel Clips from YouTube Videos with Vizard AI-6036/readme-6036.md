Generate Viral TikTok/IG Reel Clips from YouTube Videos with Vizard AI

https://n8nworkflows.xyz/workflows/generate-viral-tiktok-ig-reel-clips-from-youtube-videos-with-vizard-ai-6036


# Generate Viral TikTok/IG Reel Clips from YouTube Videos with Vizard AI

---

## 1. Workflow Overview

This n8n workflow, titled **"Generate Viral TikTok/IG Reel Clips from YouTube Videos with Vizard AI"**, automates the process of analyzing long-form YouTube videos to generate viral short clips and share the best clips on Slack for review. It is designed primarily for social media content creators, marketing teams, or digital agencies aiming to quickly repurpose YouTube content into viral short videos suitable for platforms like TikTok or Instagram Reels.

The workflow is logically divided into two main blocks:

- **1.1 Analyze YouTube Video & Generate Viral Clips**  
  Handles input reception, submission of the YouTube video URL to the Vizard AI API for clip generation, and polling for processing status until results are available.

- **1.2 Share Best Viral Clips in Slack**  
  Processes the returned clips, filters for high viral scores, and posts selected clips into a Slack channel thread for team review and further action.

---

## 2. Block-by-Block Analysis

### 1.1 Analyze YouTube Video & Generate Viral Clips

**Overview:**  
This block accepts a YouTube video URL via a form trigger, submits it to Vizard AI for clip generation, waits for processing, and checks periodically whether the clips are ready.

**Nodes Involved:**  
- `form_trigger`  
- `submit_video`  
- `wait`  
- `get_clipping_status`  
- `check_status`

---

#### Node: form_trigger

- **Type & Role:**  
  Form Trigger node; entry point that listens for a user-submitted YouTube video URL input through a web form.

- **Configuration:**  
  The form is titled "YouTube Video Clipper" and requires a single mandatory field: "YouTube Video Url" with a placeholder example (`https://www.youtube.com/watch?v=DB9mjd-65gw`).

- **Key Expressions:**  
  Outputs user input as `$json['YouTube Video Url']`.

- **Connections:**  
  Outputs to `submit_video`.

- **Edge Cases:**  
  Invalid or missing URL input will block the workflow since the field is required. URL formatting errors are not validated here.

---

#### Node: submit_video

- **Type & Role:**  
  HTTP Request node; sends a POST request to Vizard AI's API to create a project for clip generation from the provided YouTube video.

- **Configuration:**  
  - URL: `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/create`  
  - Method: POST  
  - Body (JSON): includes language ("en"), video URL from form input, video type (2 for YouTube), max clip number (8), and preferred clip length (0 means default).  
  - Authentication: HTTP Header Auth with Vizard AI credentials.

- **Key Expressions:**  
  Uses `{{$json['YouTube Video Url']}}` to dynamically insert submitted URL.

- **Connections:**  
  Outputs to `wait`.

- **Edge Cases:**  
  - Authentication failures with Vizard API.  
  - API rate limits or downtime.  
  - Malformed URL causing API rejection.

---

#### Node: wait

- **Type & Role:**  
  Wait node; introduces a fixed delay (10 seconds) to allow Vizard AI to process the video.

- **Configuration:**  
  Wait time set to 10 seconds.

- **Connections:**  
  - From `submit_video` (initial wait).  
  - Also connected after `check_status` node for polling retries.

- **Edge Cases:**  
  Fixed delay may be insufficient or excessive depending on video length and Vizard AI processing speed.

---

#### Node: get_clipping_status

- **Type & Role:**  
  HTTP Request node; polls the Vizard AI API for current project status to check if clips are ready.

- **Configuration:**  
  - URL constructed dynamically with project ID from previous response:  
    `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/query/{{ $json.projectId }}`  
  - Method: GET  
  - Authentication: HTTP Header Auth with Vizard AI credentials.

- **Connections:**  
  Outputs to `check_status`.

- **Edge Cases:**  
  - Project ID missing or invalid.  
  - API authentication errors or downtime.  
  - Network timeouts.

---

#### Node: check_status

- **Type & Role:**  
  If node; evaluates if Vizard AI processing is complete by checking a specific status code.

- **Configuration:**  
  - Condition: `$json.code` equals `2000` (indicating processing finished successfully).  
  - If TRUE, proceeds to send initial Slack message (`send_initial_msg`).  
  - If FALSE, loops back to `wait` node to retry polling.

- **Connections:**  
  - TRUE branch to `send_initial_msg`.  
  - FALSE branch to `wait` for another delay before re-polling.

- **Edge Cases:**  
  - Unexpected status codes or API errors.  
  - Infinite looping if status never reaches 2000.

---

### 1.2 Share Best Viral Clips in Slack

**Overview:**  
Once clips are ready, this block extracts the clips data, filters for viral clips (viralScore ≥ 9), and posts them as threaded messages to a designated Slack channel.

**Nodes Involved:**  
- `send_initial_msg`  
- `set_videos`  
- `split_videos`  
- `filter_viral_score`  
- `send_video_msg`

---

#### Node: send_initial_msg

- **Type & Role:**  
  Slack node; sends an initial message with a link to the original YouTube video in a Slack channel to start a thread for clip results.

- **Configuration:**  
  - Text: clickable link using Slack markdown to the original YouTube URL from the form input.  
  - Channel: selected by ID (`C08KC39K8DR`).  
  - OAuth2 authentication with Slack.

- **Connections:**  
  Outputs to `set_videos`.

- **Edge Cases:**  
  - Slack API authentication failures.  
  - Invalid channel ID.  
  - Rate limiting on Slack API.

---

#### Node: set_videos

- **Type & Role:**  
  Set node; extracts the `videos` array from the checked status JSON and assigns it to a workflow variable for further processing.

- **Configuration:**  
  - Assignment: `videos = {{ $('check_status').item.json.videos }}`

- **Connections:**  
  Outputs to `split_videos`.

- **Edge Cases:**  
  - Missing or empty `videos` array in API response.

---

#### Node: split_videos

- **Type & Role:**  
  SplitOut node; splits the `videos` array into individual items, each representing one clip.

- **Configuration:**  
  - Field to split out: `videos`.

- **Connections:**  
  Outputs each clip item to `filter_viral_score`.

- **Edge Cases:**  
  - Empty array results in no output items.

---

#### Node: filter_viral_score

- **Type & Role:**  
  Filter node; filters clips to only those with a viral score ≥ 9.

- **Configuration:**  
  - Condition: `$json.viralScore >= 9` (loose type validation enabled).

- **Connections:**  
  Outputs qualifying clips to `send_video_msg`.

- **Edge Cases:**  
  - Clips with missing or non-numeric `viralScore` field may be excluded or cause errors.

---

#### Node: send_video_msg

- **Type & Role:**  
  Slack node; posts each selected viral clip as a message in the Slack thread started by `send_initial_msg`.

- **Configuration:**  
  - Text includes bolded clip title, viral score, and the clip video URL formatted as a code block.  
  - Posts as a thread reply using `thread_ts` from the initial message.  
  - Channel: same as initial message.  
  - OAuth2 Slack authentication.

- **Connections:**  
  Terminal node (no outputs).

- **Edge Cases:**  
  - Slack API limits or failures.  
  - Missing `title`, `viralScore`, or `videoUrl` fields in clip data.

---

### Sticky Notes Overview

Two sticky notes provide helpful high-level context:

- **Sticky Note (top-left):** Explains the entire first block of analyzing YouTube video and polling Vizard AI.

- **Sticky Note1 (below):** Explains the second block of filtering viral clips and sharing to Slack with possible future extensions.

---

## 3. Summary Table

| Node Name          | Node Type           | Functional Role                                   | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                              |
|--------------------|---------------------|--------------------------------------------------|-----------------------|--------------------------|--------------------------------------------------------------------------------------------------------|
| form_trigger       | Form Trigger        | Receives YouTube video URL input from user       | —                     | submit_video             | ## 1. Analyze Long Form YouTube Video & Generate Viral Clips<br>- Provide YouTube Video Urls as Input<br>- Send API request to Vizard AI to submit the video to analyze and generate the clips<br>- Poll the Vizard `/query` endpoint |
| submit_video       | HTTP Request        | Submits video URL to Vizard AI to start clip gen | form_trigger          | wait                     | Same as above                                                                                           |
| wait               | Wait                | Waits 10 seconds before polling status again      | submit_video, check_status | get_clipping_status      | Same as above                                                                                           |
| get_clipping_status | HTTP Request        | Polls Vizard AI API for project clip generation status | wait                  | check_status             | Same as above                                                                                           |
| check_status       | If                  | Checks if clip generation is complete             | get_clipping_status    | send_initial_msg (true), wait (false) | Same as above                                                                                           |
| send_initial_msg   | Slack               | Posts initial Slack message linking original video | check_status          | set_videos               | ## 2. Share Best Viral Clips In Slack<br>- Filter video clip results that have a viral score of at least 9 out of 10<br>- Share video title and link to download in slack thread for further review<br>- This can be extended even further to auto-generate captions using a LLM and auto-posting to social media channels via blotato |
| set_videos         | Set                 | Extracts videos array from API response            | send_initial_msg       | split_videos             | Same as above                                                                                           |
| split_videos       | SplitOut            | Splits videos array into individual clip items    | set_videos             | filter_viral_score       | Same as above                                                                                           |
| filter_viral_score | Filter              | Filters clips with viral score ≥ 9                 | split_videos           | send_video_msg           | Same as above                                                                                           |
| send_video_msg     | Slack               | Posts each viral clip as a threaded Slack message | filter_viral_score     | —                        | Same as above                                                                                           |
| Sticky Note        | Sticky Note         | Describes first block (Input + Vizard AI polling) | —                      | —                        | See first sticky note content                                                                            |
| Sticky Note1       | Sticky Note         | Describes second block (Slack sharing)             | —                      | —                        | See second sticky note content                                                                           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: `form_trigger`  
   - Configure form title as "YouTube Video Clipper"  
   - Add one required form field:  
     - Label: "YouTube Video Url"  
     - Placeholder: `https://www.youtube.com/watch?v=DB9mjd-65gw`  
   - No authentication needed.

2. **Create HTTP Request Node to Submit Video**  
   - Type: HTTP Request  
   - Name: `submit_video`  
   - Connect input from `form_trigger`  
   - Method: POST  
   - URL: `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/create`  
   - Authentication: HTTP Header Auth with Vizard AI API credentials  
   - Body Type: JSON  
   - Body Content (Use expression to insert form input URL):  
     ```json
     {
       "lang": "en",
       "preferLength": [0],
       "videoUrl": "{{ $json['YouTube Video Url'] }}",
       "videoType": 2,
       "maxClipNumber": 8
     }
     ```

3. **Create Wait Node for Initial Delay**  
   - Type: Wait  
   - Name: `wait`  
   - Set wait time to 10 seconds  
   - Connect input from `submit_video`

4. **Create HTTP Request Node to Poll Status**  
   - Type: HTTP Request  
   - Name: `get_clipping_status`  
   - Connect input from `wait`  
   - Method: GET  
   - URL: Use expression to dynamically insert project ID from previous HTTP response:  
     `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/query/{{ $json.projectId }}`  
   - Authentication: HTTP Header Auth with Vizard AI API credentials

5. **Create If Node to Check Completion Status**  
   - Type: If  
   - Name: `check_status`  
   - Connect input from `get_clipping_status`  
   - Condition:  
     - Check if `$json.code` equals `2000` (number, strict equality)  
   - If TRUE: connect to next block (`send_initial_msg`)  
   - If FALSE: connect back to `wait` node for another delay (loop)

6. **Create Slack Node to Send Initial Message**  
   - Type: Slack  
   - Name: `send_initial_msg`  
   - Connect input from TRUE output of `check_status`  
   - Authenticate using Slack OAuth2 credentials  
   - Channel: select or enter channel ID (e.g., `C08KC39K8DR`)  
   - Message Text (expression):  
     `<{{ $('form_trigger').item.json['YouTube Video Url'] }}|Video Clipper Results>`  
   - Disable "Include Link to Workflow"

7. **Create Set Node to Extract Videos Array**  
   - Type: Set  
   - Name: `set_videos`  
   - Connect input from `send_initial_msg`  
   - Assignment: create new field `videos` with value:  
     `{{ $('check_status').item.json.videos }}`

8. **Create SplitOut Node to Split Videos Array**  
   - Type: SplitOut  
   - Name: `split_videos`  
   - Connect input from `set_videos`  
   - Field to split: `videos`

9. **Create Filter Node to Select Viral Clips**  
   - Type: Filter  
   - Name: `filter_viral_score`  
   - Connect input from `split_videos`  
   - Condition: `$json.viralScore >= 9` (number, greater or equal, loose validation enabled)

10. **Create Slack Node to Post Viral Clips**  
    - Type: Slack  
    - Name: `send_video_msg`  
    - Connect input from `filter_viral_score`  
    - Authenticate using Slack OAuth2 credentials  
    - Channel: same as `send_initial_msg`  
    - Message Text (expression):  
      ```
      =*{{ $json.title }} | ({{ $json.viralScore }} / 10)*
      ```
      ```
      {{ $json.videoUrl }}
      ```
      `---`  
    - Configure message to post as thread reply referencing `thread_ts` from `send_initial_msg` message timestamp.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                 | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow uses the Vizard AI API to analyze YouTube videos and generate viral clips automatically.                                                                                         | Vizard AI API docs: https://docs.vizard.ai (not included in workflow)                             |
| Slack messages are posted to a specific channel and threaded to keep results organized. OAuth2 credentials for Slack must be configured properly.                                              | Slack API info: https://api.slack.com/authentication/oauth-v2                                   |
| The workflow can be extended to include auto-caption generation using large language models and automatic posting to social media platforms like TikTok or Instagram via tools such as blotato. | Sticky Note1 content within the workflow                                                          |
| Workflow uses polling with a fixed 10-second wait; depending on video length and API response times, this might require adjustment or implementing exponential backoff for better efficiency.     | Best practice suggestion for polling mechanisms                                                  |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---