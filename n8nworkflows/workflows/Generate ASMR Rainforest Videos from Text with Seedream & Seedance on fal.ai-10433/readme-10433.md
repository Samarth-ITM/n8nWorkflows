Generate ASMR Rainforest Videos from Text with Seedream & Seedance on fal.ai

https://n8nworkflows.xyz/workflows/generate-asmr-rainforest-videos-from-text-with-seedream---seedance-on-fal-ai-10433


# Generate ASMR Rainforest Videos from Text with Seedream & Seedance on fal.ai

### 1. Workflow Overview

This workflow automates the creation of ASMR rainforest-themed videos starting from a text prompt, using the Fal.ai platform's Seedream and Seedance AI models. It is designed for content creators or automation engineers who want to generate cinematic rainforest videos featuring rain and storm effects, without manual intervention in intermediate steps.

The workflow is logically divided into three main functional blocks:

- **1.1 Starter & Prompt Setup:** Receives manual trigger input and defines the textual prompts for image and video generation.
- **1.2 Image Generation & Status Polling:** Sends the prompt to Seedream (Fal.ai text-to-image model), waits, and polls for image generation completion.
- **1.3 Video Generation & Status Polling:** Uses the generated image as input for Seedance (Fal.ai image-to-video model), waits, polls for video completion, and finally downloads the completed video.

Each block manages asynchronous operations and includes wait/poll mechanisms to handle the latency inherent in AI model processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Starter & Prompt Setup

- **Overview:** Initializes the workflow execution via manual trigger and sets detailed prompts for image and video generation. This block provides the starting point and input data.
- **Nodes Involved:**  
  - manual-workflow  
  - prompt  
  - Sticky Note (starter)

- **Node Details:**

  - **manual-workflow**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow manually, enabling user control over execution timing.  
    - *Config:* No parameters; default manual trigger settings.  
    - *Input/Output:* No input; outputs to `prompt` node.  
    - *Edge Cases:* User must trigger manually; no automation trigger here.  
    - *Sticky Note:* “## starter”

  - **prompt**  
    - *Type:* Set Node  
    - *Role:* Defines two string variables: `prompt-teks-image` (the detailed text prompt for image generation) and `prompt-image-video` (a short prompt for video generation).  
    - *Config:*  
      - `prompt-teks-image`: A highly descriptive prompt describing a thunderstorm in a dense forest with cinematic details.  
      - `prompt-image-video`: Set to "animate this" as a minimal prompt for the video model.  
    - *Expressions:* Uses static string assignment; no dynamic expressions.  
    - *Input/Output:* Input from `manual-workflow`; outputs to `create-image`.  
    - *Edge Cases:* No dynamic input; relies on user editing this node to change prompts.

  - **Sticky Note (starter)**  
    - *Content:* “## starter”  
    - *Purpose:* Visual grouping for the starter block.

---

#### 2.2 Image Generation & Status Polling

- **Overview:** Sends the text prompt to the Fal.ai Seedream text-to-image API, waits for processing, polls for completion, and retrieves the generated image URL.
- **Nodes Involved:**  
  - create-image  
  - wait-10s  
  - get-status-image  
  - is-image-complete  
  - link-image  
  - Sticky Note (create image)

- **Node Details:**

  - **create-image**  
    - *Type:* HTTP Request  
    - *Role:* Posts the image generation request to Fal.ai Seedream v4 text-to-image endpoint.  
    - *Config:*  
      - URL: `https://queue.fal.run/fal-ai/bytedance/seedream/v4/text-to-image`  
      - Method: POST  
      - Body Parameters:  
        - `prompt`: from `prompt-teks-image` variable  
        - `image_size`: fixed as `portrait_16_9`  
      - Authentication: HTTP header with Fal.ai API key credential.  
    - *Input/Output:* Input from `prompt`; outputs to `wait-10s`.  
    - *Edge Cases:*  
      - API key missing or invalid (auth errors)  
      - Network timeout or API errors  
      - API rate limits or service unavailability

  - **wait-10s**  
    - *Type:* Wait  
    - *Role:* Pauses workflow for 10 seconds to allow image generation to start/complete.  
    - *Config:* Fixed 10-second wait.  
    - *Input/Output:* Input from `create-image`; outputs to `get-status-image`.  
    - *Edge Cases:* None significant; timing may be insufficient if image generation is slow.

  - **get-status-image**  
    - *Type:* HTTP Request  
    - *Role:* Polls the status URL returned by the `create-image` node to check if image generation is complete.  
    - *Config:*  
      - URL: dynamically set from `status_url` received in previous node JSON.  
      - Authentication: Same Fal.ai API key credential.  
    - *Input/Output:* Input from `wait-10s`; outputs to `is-image-complete`.  
    - *Edge Cases:*  
      - Status URL may be invalid or expired  
      - API errors or delays  
      - JSON parsing failures if response format changes

  - **is-image-complete**  
    - *Type:* If  
    - *Role:* Checks if the image generation status equals "COMPLETED".  
    - *Config:* Condition: `$json.status` equals `COMPLETED` (case sensitive, strict validation).  
    - *Input/Output:* Input from `get-status-image`; if true outputs to `link-image`; else loops back to `wait-10s`.  
    - *Edge Cases:*  
      - Status other than COMPLETED (e.g., FAILED, PENDING)  
      - Infinite loops if status never changes  
      - Possible need for max retries/timeouts (not implemented)

  - **link-image**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves the final image data from the response URL once image generation completes.  
    - *Config:* URL dynamically set from `response_url` JSON field.  
    - *Authentication:* Fal.ai API key header.  
    - *Input/Output:* Input from `is-image-complete`; outputs to `create-video`.  
    - *Edge Cases:*  
      - URL invalid or expired  
      - Download failures or timeouts

  - **Sticky Note (create image)**  
    - *Content:* “## create image”  
    - *Purpose:* Visual grouping for image generation block.

---

#### 2.3 Video Generation & Status Polling

- **Overview:** Converts the generated image into a video using Fal.ai Seedance image-to-video model, includes wait and polling to confirm video processing, and finally downloads the video file.
- **Nodes Involved:**  
  - create-video  
  - wait-30s  
  - get-status-video  
  - is-video-complete  
  - link-video  
  - download-video  
  - Sticky Note2 (create video)

- **Node Details:**

  - **create-video**  
    - *Type:* HTTP Request  
    - *Role:* Submits the request to Fal.ai Seedance v1 pro image-to-video endpoint.  
    - *Config:*  
      - URL: `https://queue.fal.run/fal-ai/bytedance/seedance/v1/pro/image-to-video`  
      - Method: POST  
      - Body Parameters:  
        - `prompt`: from `prompt-image-video` variable in the `prompt` node  
        - `image_url`: from the downloaded image URL (`$json.images[0].url`)  
        - `aspect_ratio`: fixed `9:16` (portrait video)  
      - Authentication: Fal.ai API key header.  
    - *Input/Output:* Input from `link-image`; outputs to `wait-30s`.  
    - *Edge Cases:*  
      - Invalid image URL or prompt causing API errors  
      - Network or authentication failures

  - **wait-30s**  
    - *Type:* Wait  
    - *Role:* Waits 30 seconds to allow video processing to progress.  
    - *Config:* Fixed 30-second wait.  
    - *Input/Output:* Input from `create-video` or loopback from `is-video-complete` false condition; outputs to `get-status-video`.  
    - *Edge Cases:* None significant; time may require tuning.

  - **get-status-video**  
    - *Type:* HTTP Request  
    - *Role:* Polls the video processing status URL returned by `create-video`.  
    - *Config:*  
      - URL dynamically from `status_url` field.  
      - Authentication: Fal.ai API key header.  
    - *Input/Output:* Input from `wait-30s`; outputs to `is-video-complete`.  
    - *Edge Cases:* Same as image status polling node.

  - **is-video-complete**  
    - *Type:* If  
    - *Role:* Checks if video processing finished with status `COMPLETED`.  
    - *Config:* Condition check `$json.status` equals `COMPLETED`.  
    - *Input/Output:* Input from `get-status-video`; if true outputs to `link-video`, else loops back to `wait-30s`.  
    - *Edge Cases:* Same as image completion check.

  - **link-video**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves the final video file URL from the `response_url` field once processing completes.  
    - *Authentication:* Fal.ai API key header.  
    - *Input/Output:* Input from `is-video-complete`; outputs to `download-video`.  
    - *Edge Cases:* Similar to `link-image`.

  - **download-video**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the actual video file from the provided URL.  
    - *Config:* URL from `$json.video.url`.  
    - *Input/Output:* Input from `link-video`; final output node.  
    - *Edge Cases:* Download failure, URL invalid or expired.

  - **Sticky Note2 (create video)**  
    - *Content:* “## create video”  
    - *Purpose:* Visual grouping for video generation block.

---

### 3. Summary Table

| Node Name         | Node Type         | Functional Role                      | Input Node(s)          | Output Node(s)           | Sticky Note                                                  |
|-------------------|-------------------|------------------------------------|-----------------------|--------------------------|--------------------------------------------------------------|
| manual-workflow    | Manual Trigger    | Start workflow                     | -                     | prompt                   | ## starter                                                   |
| prompt            | Set               | Define text prompts for image & video | manual-workflow       | create-image             |                                                              |
| create-image      | HTTP Request      | Request text-to-image generation   | prompt                 | wait-10s                 | ## create image                                              |
| wait-10s          | Wait              | Pause 10 seconds before polling    | create-image           | get-status-image         | ## create image                                              |
| get-status-image  | HTTP Request      | Poll image generation status       | wait-10s               | is-image-complete        | ## create image                                              |
| is-image-complete | If                | Check if image generation complete | get-status-image       | link-image, wait-10s     | ## create image                                              |
| link-image        | HTTP Request      | Retrieve generated image URL       | is-image-complete      | create-video             | ## create image                                              |
| create-video      | HTTP Request      | Request image-to-video generation  | link-image             | wait-30s                 | ## create video                                              |
| wait-30s          | Wait              | Pause 30 seconds before polling    | create-video, is-video-complete (false) | get-status-video         | ## create video                                              |
| get-status-video  | HTTP Request      | Poll video generation status       | wait-30s               | is-video-complete        | ## create video                                              |
| is-video-complete | If                | Check if video generation complete | get-status-video       | link-video, wait-30s     | ## create video                                              |
| link-video        | HTTP Request      | Retrieve generated video URL       | is-video-complete      | download-video           | ## create video                                              |
| download-video    | HTTP Request      | Download final video file          | link-video             | -                        |                                                              |
| Sticky Note       | Sticky Note       | Visual grouping for create image   | -                      | -                        | ## create image                                              |
| Sticky Note1      | Sticky Note       | Visual grouping for starter block  | -                      | -                        | ## starter                                                   |
| Sticky Note2      | Sticky Note       | Visual grouping for create video   | -                      | -                        | ## create video                                              |
| Sticky Note3      | Sticky Note       | Requirements note                  | -                      | -                        | ## requirements key - api key [fal.ai](https://fal.ai/dashboard/keys) |
| Sticky Note5      | Sticky Note       | Setup instructions                 | -                      | -                        | Step-by-step setup guide including credential and prompt setup |
| Sticky Note6      | Sticky Note       | Workflow overview and future plans | -                      | -                        | Automated visual explanation and roadmap for enhancements  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Credential**  
   - Type: HTTP Header Authentication  
   - Name: `fal` (or any descriptive name)  
   - Header Name: `Authorization`  
   - Header Value: `key your-api-key` (replace `your-api-key` with your actual Fal.ai API key)  
   - Save this credential; it will be used in all HTTP Request nodes interacting with Fal.ai API.

2. **Add Manual Trigger Node**  
   - Name: `manual-workflow`  
   - Use default settings (no parameters).  
   - This node starts the workflow manually.

3. **Add Set Node for Prompts**  
   - Name: `prompt`  
   - Under "Values to Set," add two string fields:  
     - `prompt-teks-image`: paste the detailed rainforest thunderstorm prompt (see overview).  
     - `prompt-image-video`: set to `"animate this"`.  
   - Connect output of `manual-workflow` to this node.

4. **Add HTTP Request Node for Image Creation**  
   - Name: `create-image`  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/bytedance/seedream/v4/text-to-image`  
   - Authentication: Use the created HTTP Header Auth credential.  
   - Body Parameters (JSON):  
     - `prompt`: Expression: `{{$json["prompt-teks-image"]}}` from Set node data  
     - `image_size`: `"portrait_16_9"`  
   - Connect output of `prompt` to this node.

5. **Add Wait Node (10s)**  
   - Name: `wait-10s`  
   - Amount: 10 seconds  
   - Connect `create-image` output to this node.

6. **Add HTTP Request Node to Poll Image Status**  
   - Name: `get-status-image`  
   - Method: GET  
   - URL: Expression: `{{$json.status_url}}` (taken from `create-image` response)  
   - Use same Fal.ai HTTP Header Auth credential.  
   - Connect `wait-10s` output to this node.

7. **Add If Node to Check Image Completion**  
   - Name: `is-image-complete`  
   - Condition: `$json.status` equals `COMPLETED` (case sensitive, strict)  
   - Connect `get-status-image` output to this node.

8. **Add HTTP Request Node to Retrieve Image Data**  
   - Name: `link-image`  
   - Method: GET  
   - URL: Expression: `{{$json.response_url}}` (from `get-status-image` JSON)  
   - Authentication: Fal.ai HTTP Header Auth credential.  
   - Connect `is-image-complete` true output to this node.

9. **Looping for Image Status**  
   - Connect `is-image-complete` false output back to `wait-10s` to retry polling.

10. **Add HTTP Request Node for Video Creation**  
    - Name: `create-video`  
    - Method: POST  
    - URL: `https://queue.fal.run/fal-ai/bytedance/seedance/v1/pro/image-to-video`  
    - Authentication: Fal.ai HTTP Header Auth credential.  
    - Body Parameters:  
      - `prompt`: Expression: `{{$node["prompt"].json["prompt-image-video"]}}`  
      - `image_url`: Expression: `{{$json.images[0].url}}` (from `link-image` response)  
      - `aspect_ratio`: `"9:16"`  
    - Connect output of `link-image` to this node.

11. **Add Wait Node (30s)**  
    - Name: `wait-30s`  
    - Amount: 30 seconds  
    - Connect `create-video` output to this node.

12. **Add HTTP Request Node to Poll Video Status**  
    - Name: `get-status-video`  
    - Method: GET  
    - URL: Expression: `{{$json.status_url}}` (from `create-video` response)  
    - Use Fal.ai HTTP Header Auth credential.  
    - Connect `wait-30s` output to this node.

13. **Add If Node to Check Video Completion**  
    - Name: `is-video-complete`  
    - Condition: `$json.status` equals `COMPLETED` (case sensitive, strict)  
    - Connect `get-status-video` output to this node.

14. **Add HTTP Request Node to Retrieve Video Data**  
    - Name: `link-video`  
    - Method: GET  
    - URL: Expression: `{{$json.response_url}}` (from `get-status-video` JSON)  
    - Authentication: Fal.ai HTTP Header Auth credential.  
    - Connect `is-video-complete` true output to this node.

15. **Looping for Video Status**  
    - Connect `is-video-complete` false output back to `wait-30s` to retry polling.

16. **Add HTTP Request Node to Download Video File**  
    - Name: `download-video`  
    - Method: GET  
    - URL: Expression: `{{$json.video.url}}` (from `link-video` JSON)  
    - Connect `link-video` output to this node.

17. **Final Step**  
    - The `download-video` node outputs the final video data, which can be saved or used in subsequent workflows.

18. **Optional: Add Sticky Notes**  
    - Add sticky notes to visually group nodes by function: starter, create image, create video.  
    - Add notes for requirements and setup as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| API key required from Fal.ai: https://fal.ai/dashboard/keys                                                                                                                | Credential setup for all Fal.ai HTTP nodes.    |
| Setup instructions: create credential, set prompts, use Seedream v4 for text-to-image, Seedance v1 pro for image-to-video, run manually.                                  | Sticky Note5 in workflow                        |
| Workflow automates prompt ➔ image ➔ video generation with polling and wait steps for reliable asynchronous processing.                                                     | Sticky Note6 in workflow                        |
| Future enhancement ideas: auto-post results to social media, log URLs to Google Sheets or datatables, send notifications on completion.                                  | Sticky Note6 in workflow                        |
| Fal.ai Seedream: https://fal.ai/models/bytedance/seedream-v4 (model documentation)                                                                                         | Reference for text-to-image model               |
| Fal.ai Seedance: https://fal.ai/models/bytedance/seedance-v1-pro (model documentation)                                                                                     | Reference for image-to-video model              |

---

This documentation fully covers the structure, logic, configuration, and operation of the "Generate ASMR Rainforest Videos from Text with Seedream & Seedance on fal.ai" workflow in n8n. It provides sufficient detail for reproduction, modification, and error anticipation.